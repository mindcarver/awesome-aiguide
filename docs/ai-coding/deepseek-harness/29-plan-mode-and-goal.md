# Plan Mode 与 Goal：dsh 怎么管理目标和计划

> `dsh` 管理目标靠两个重量不同的机制。`ctx.planMode` 是软引导：激活时往每个模型请求塞一段部署拥有的提示，模型可以不听，它没有也不想要强制力。`ctx.goals` 是持久的事件溯源目标生命周期：阶段、修订号、轮次预算，全部从会话日志 fold 出来，跨重启恢复。两者共享一条根基：状态记在日志里、用纯 fold 派生，都和沙箱、审批这类硬强制分开。分清楚"引导模型"和"追踪目标"是两件事，再分清楚"目标怎么了"和"现在能不能续跑"又是两件事，这篇的主线就是这两次拆分。

## 为什么目标管理是两个机制

让 agent 管理目标，最朴素的设想是给它一个"当前目标"字段。但"管目标"至少藏着两个层次不同的需求。

第一个是引导模型先规划再动手：改代码之前先产出一份计划给人审。这是一种协作姿态。模型可以不听，但它被引导着先想后做。第二个是追踪一个长期目标的进展：目标是什么、到哪一步了（活跃、暂停、卡住、完成）、跑了几轮、还能跑几轮。这是持久状态，要跨重启恢复，要防并发改坏。

把这两个揉成一个"目标系统"会两头不讨好。引导需要的是轻：一段提示、一个退出工具，装了就用，不装拉倒。追踪需要的是重：生命周期、修订号、预算，一样都不能少。`dsh` 拆成 `ctx.planMode` 和 `ctx.goals` 两个可选包，agent loop 一个都不依赖。

两者还有一个共同的边界声明：都和硬性强制分开。plan mode 明确自称软引导，沙箱和审批各自独立强制，不读也不写 plan 状态。goal 的强制也分两层，持久阶段回答"目标怎么了"，进程内 activation 单独回答"续跑消费者能不能再开一轮"。引导、追踪、强制，三个词三种机制，这篇依次展开。

## Plan Mode：一段提示的重量

`ctx.planMode` 的服务类型是 `PlanModeController`。它贡献三样东西：`plan:policy` 提示段、`exit_plan_mode` 工具、`/plan` 命令。三样加起来，就是 plan mode 的全部。

提示段只在激活时渲染，顺序 50，内容是部署配置的原文。配置校验是严格的：section 缺失、空白、非字符串，或出现任何未知键，都在插件加载时失败，而不是被忽略。一个拼错键名的部署在启动那一刻就被拦下，不会带着一段不知道从哪来的提示跑半个月。

### 引导不冒充强制

文档开篇把定性钉死：plan mode 是软引导。这意味着 plan mode 激活，不等于"agent 不能改文件"。沙箱允不允许写、审批要不要问，和 plan mode 无关，部署分别配置它们。

为什么不干脆让 plan mode 顺手把沙箱也收紧了？设计笔记里记录了被拒绝的方案和拒绝的理由：一个 mode 拥有的沙箱上限，会让用户显式选择的沙箱模式"看起来成功了，实际上什么都没做"。用户明明选了 `workspace-write`，因为 plan mode 激活被悄悄压成只读，界面还告诉他设置成功。这种静默降级比没有还糟。另一个被拒绝的思路是维护一份"计划模式下可变的工具清单"，理由更根本：可变性是每个工具自己的属性（包括未来的和 MCP 来的工具），不是每个 plan 部署都要维护的一份名单。名单一漏，新工具天然钻空子。

所以 plan mode 的安全观很诚实：它是引导，不是安全边界。模型在 plan mode 里坚持动手改文件，拦得住它的是沙箱和审批。引导没被遵守，那是引导的成本，用错了工具去强制才是事故。

### 状态是纯 fold，没有镜像

`plan/mode` 事件载荷就一个布尔值 `{ active }`，纯日志、整值替换、永不进模型 transcript。生效状态永远是会话日志的一个纯 fold：`foldPlanMode(events)` 返回前缀里最后一条记录的值，没有就返回 `false`。没有 live mirror，没有旁路状态。

这个选择的收益在恢复路径上兑现：resume、fork、compaction 都直接恢复 plan 状态，因为状态本来就是从日志算的。消费方各自取用：TUI 和 Web 客户端通过 `session/event` 观察已提交的翻转来渲染开关，提示组装在读到的状态激活时渲染 policy 段，`exit_plan_mode` 用它判断调用是否合法。所有消费者读到的是同一个持久事实，没有谁比谁更权威。被拒绝的替代方案是让某个展示传输层拥有 plan 状态，那样一来浏览器刷新一次状态就没了，fork 出来的会话还继承不继承、怎么继承，全是新问题。

### turn 边界 flush：pending 的四条出路

plan mode 最精巧的部分是选择的落盘时机。因为每个会话事件都被 turn 包住，一个用户选择会保持 pending，直到下一个被接受的 in-turn pre-step 在请求派生之前追加它。一个选择永远不强制 continuation：turn 最后一个被接受的 pre-step 之后做的选择，会在更晚的某个 turn 里追加。

为什么这么讲究？因为 plan 状态影响每个请求的提示段。想象相反的设计：请求派生到一半改状态，这个请求的提示段用了新状态，而它的请求头还描述着旧状态，日志里就出现一条"一半旧一半新"的撕裂记录。把追加点固定在"下一个被接受的 in-turn pre-step、请求派生之前"，并让测试证明每条已提交的 `plan/mode` 都先于它影响的请求头，撕裂在结构上就不可能出现。

`set(agent, active)` 的四种结果对应四种时机。`committed`：turn 之间调用，立即追加，因为下一个 prompt 之前不会有 in-turn pre-step 跑。`queued`：turn 开着，选择 pending，等下一个被接受的 in-turn pre-step。`cancelled`：一个相反的 pending 选择被清掉了，已记录的状态本来就和目标一致。`noop`：已经在这个状态，或已经 pending 在这个状态，无事可做。`get(agent)` 返回 `{ active, pending? }`：已记录的、用于组装当前 step 的状态，加上等着被追加的已选状态。

运行期的唯一追加点是一个 prepend 的 `agent/pre-step` 监听器。它观察每个被提议的请求 step（包括 turn 1 step 1 和请求恢复重试），先调下游监听器，只在它们接受 step 之后才追加。prompt 录入发生在 turn 之前，没法追加 `plan/mode`，所以 prompt 时做的选择由那个 turn 的第一个被接受的 in-turn pre-step 追加。追加失败不能阻塞 turn：选择保持 pending，等下一个边界。这里有个容易误判的点：turn 结束后的检查点期间，会话状态还是 `running`，判断"turn 是否开着"靠的是 open-turn fold，不是状态字段的表面值。

把一次 queued 的落盘按时间排开看更清楚。假设 turn 开着，第 3 步在跑，用户在界面上点了开关：`set` 返回 `queued`，选择进入 pending。第 3 步的请求早已派发，带着旧状态走了，不受影响。模型回话，提议第 4 步。prepend 的 pre-step 监听器看到这个提议，先把下游监听器问一遍，全部接受之后，追加 `plan/mode` 事件，然后第 4 步的请求派生读到新状态，请求头也描述新状态。日志里这两条记录的先后顺序，就是原子性的证明材料：任何人重放这段日志，都看到状态变更先于它影响的请求。反过来，下游要是拒绝了第 4 步，或者追加本身失败，选择就继续 pending，turn 照常走完，没有任何半个状态留在账上。

一个已追加的用户选择还会记一条 plugin 来源的 `user/message` 通知，但只在上次记录的请求头描述的是另一个状态时才记。模型被通知的时机精确到"上下文真的变了"，绝不重复唠叨。

代价也有，文档直接承认：turn 最后一个被接受的 pre-step 之后做的选择是进程本地的，进程在下一个被接受的 in-turn pre-step 之前退出，选择就丢了。这是 turn 边界 flush 的对价：选择不是立即落盘，要等一个能保证原子性的时机。

### exit_plan_mode：一次受审的退出

`exit_plan_mode` 工具在 plan mode 不激活时也保持注册。进出 plan mode 只改提示段，永不改请求的工具目录。这条规矩的收益是稳定性：模型学会的工具集合不随模式切换抖动，工具 schema 的缓存也不因模式切换失效。在 plan mode 外执行它，调用失败。

在 plan mode 里调用它，要求一份以 `#` 标题开头的完整 markdown 计划，通过 user-questions 接缝呈现给人审。批准的返回是 `{ approved: true }`，同时记一个静默（不叙述）的 pending 退出，在下一个被接受的 in-turn pre-step 追加。于是 plan 引导在 assistant 当前工具批的剩余部分仍然激活，退出这件事由工具结果自己向模型报告。"继续规划"是一次失败的调用，带着用户的反馈回来，模型修改后重新呈现。审查期间缺交互通道、或插件被卸载，调用同样失败，fail-closed，手工逃生通道是 `/plan off`。任何不构成"恰好一次干净批准"的审查回答都维持规划状态并返回纠正性反馈，没有模糊地带。

为什么审查不走现成的审批接缝？设计笔记给的理由有两层。其一，计划审查不是权限决策：审批接缝回答"这个动作能不能放行"，计划审查回答"这份方案好不好"，两者的语义、粒度、消费方都不同。其二，退出必须是一次被日志记录的工具调用，作为结构化的状态转换。审批的事件和工具调用的事件是两套账，混用会让重放时无法重建"模型请求退出、人批准退出"这条链。

### 计划本身存哪儿

被拒绝的方案有两个，都值得看一眼。把计划放进模型上下文的 surface（比如作为一条 assistant 消息持久化）会花双份模型上下文：压缩一次，surface 翻转一次，同一份计划占两处。把计划写成文件（比如固定目录下一个 plan 文档）会造出第二个持久之家：会话日志说状态是 A，文件说状态是 B，哪个是真的？计划作为 `exit_plan_mode` 的工具调用参数留在日志里，一处持久，一处派生，这个问题就不存在。

### /plan 命令

`/plan` 命令在 `ctx.commands` 组合进来时注册，三种用法。裸 `/plan` 选 plan mode。`/plan <message>` 选 plan mode，然后用 `agent.steer()` 把文本提交为下一步的普通已记录用户消息，让它在 plan 引导下进入对话。`/plan off` 选不激活，同时取消一个还没追加、还没对请求可见的 pending 进入，这是前述 fail-closed 逃生通道的入口。

### 为什么没有统一的模式注册表

看到 plan mode、沙箱、审批三个"模式"，很自然想抽一个统一的 mode 注册表，让它们共享基类、共享预设、共享配置。设计笔记明确拒绝了这个抽象，理由有两条。

第一条是没有第二个生产消费者。抽象的成本要在真实的第二个用例出现时才看得清，为一个消费者预付抽象，抽出来的接口只会精确拟合这一个用例。笔记的原话是，未来的协作状态可以从两个具体案例里长出正确的共享接缝，凭空设计长不出来。

第二条是三个轴根本没有共同的所有权结构。plan mode 的提示段归部署，沙箱模式归执行世界，审批策略归交互层，它们的生命周期、配置入口、消费方各不相同。硬捏一个共享基类或注册表，等于强迫三个独立演化的子系统共用一件尺码不对的衣服。宁可三个机制各自直白，也不要一个谁都要迁就的抽象。

## Goal：一个持久的目标生命周期

`ctx.goals` 的服务类型是 `GoalService`，解决的是"追踪一个长期目标的持久进展"。这件事的难点不在记录一个目标字符串，在它周围的三个现实：进程会崩，目标不能跟着消失；多个写手会并发出手，改坏要能被拦住；续跑要有闸门，重启之后不能自己狂跑。GoalService 的重量就花在这三个地方：事件溯源的存储对付崩溃，比较并交换的修订号对付并发，两层的续跑授权对付失控。

### 会话日志就是目标数据库

每个 goal 变更都是一个持久的 `goal/change` 会话事件，载荷要么是完整的变更后快照，要么是一个 clear 墓碑。快照变体带版本号、操作名、完整的 `GoalSnapshot`，加上 `roundsStarted`、`createdAt`、`updatedAt`。墓碑变体记录被清除的引用和清除时间。

设计笔记对存储选择的表述很干脆：会话日志是唯一的持久真相源，持久化和 fork 不需要另一个数据库或头字段就继承了 goal 记录。被拒绝的方案是第二个存储，理由是它引入原子性和谱系问题：goal 变更和会话事件分属两个存储，一个事务写一半崩溃，两边就对不上了。事件载荷用完整快照而非 delta，也是刻意的：完整快照让检视、严格重放和 last-wins 投影都变简单，和投影层的整值事件规矩同源。

严格 fold 只从这些事件派生生命周期状态，inbox 的变更不影响 goal 状态。重放是严格的：拒绝非正轮次、轮次间隙、stale revision、停了的阶段、cap 溢出。一条损坏的记录让重放失败而不是被忽略或修复，增量重放停在第一个损坏事件的位置，故障被一致地报告。

拿一个具体场景掂量这条纪律的分量。假设有人用外部脚本手改日志，把某条 `goal/change` 的 revision 从 7 改成 9。重放走到前一条 revision 6、这一条 9 的位置，发现跳号，停下来，报告位置。操作者看到的是一条明确的故障地址，不是一个被"修复"过的目标状态。如果重放选择宽容（跳过坏的、继续 fold），用户会拿到一个看似正常实则来源不明的 goal，往下每一步都建立在被污染的状态上。严格拒绝把污染挡在了入口，代价是坏日志必须人工处理，这个交换在目标这种要跨天跨重启的状态上是划算的。

### 修订号：改目标先对表

`GoalId` 是 branded id，`GoalRef` 是 id 加一个正数 revision。调用方通过 `GoalRef` 修改一个确切的修订，每个被接受的持久变更让 revision 加一。这是比较并交换（compare-and-set）语义：你要改目标，得带上你以为是当前的 revision；别人的修改先落地了，你的 revision 就 stale 了，被拒。

防的不只是并发。设计笔记说这套校验加新鲜 id 和转换检查，能在早期拒绝被篡改的、写了一半的、生产者不一致的 goal 记录。一个被外部工具改坏的日志，重放时会停在第一个不一致的事件上，而不是把坏数据继续往前传播。调用方的代价是要处理拒绝：stale 了就重新读、重新改。

### 两个问题，两层答案

goal 设计最重要的切分：持久阶段回答"目标怎么了"，进程内 activation 单独回答"续跑消费者能不能再开一轮"。

持久阶段 `GoalPhase` 有四个值：`active`、`paused`、`blocked`、`complete`。`blocked` 是唯一一个"因为一个问题停了"的持久状态，它的 reason 带一个稳定的小写 kebab-case code（给路由用）和一个非空的自由文本说明（给人也给模型看）。策略层可以把 token 上限、时间上限这类别的限制映射成 blocked 状态，code 保住机器可路由的分类。

activation 是进程内的，永不持久化。`disarm` 移除进程内的续跑授权，不改持久目标阶段或 revision。生命周期拥有者在卸载一个 driver 前调用它；之后一个人授权的 `resume` 记录新的 activation 边。`GoalView` 里的 `activation` 字段和 `roundsStarted`、`createdAt` 这些从日志派生的值并列，但性质不同：派生值跨重启一致，activation 不跨。

为什么分两层？想象合并的世界：目标 active 就等于可以续跑。进程崩了重启，重放日志发现目标 active，续跑消费者自动开一轮，agent 在没人看的时候自己跑起来了。设计笔记的原话是，用户打开一个会话时工作静默开始是令人惊讶的。分层之后，重启的默认姿态是 disarm：日志仍然权威，目标状态原样恢复，但没有任何工作自启。想续跑，人来 `resume`。

### 轮次预算：只数 goal 的账

一个会话里可以混着人的澄清提问、随手检查、和 goal 无关的杂活。轮次预算只算 goal 的账：续跑消费者给每个被接受的 user-message turn 归属一个正的、顺序的轮次号和当前 revision，只有这些被归属的 `user/message` 事件推进 `roundsStarted`。人的唠叨不烧预算，agent 的续跑才烧。

区分两类 turn 最直观的办法是问"这条消息是谁因为什么发的"。用户中途问一句"刚才那步为什么失败"，这是人的澄清，是个被接受的 user-message turn，但没有被 goal 归属，`roundsStarted` 不动。goal driver 判断还有活没干完，自动开下一轮续跑，这才是被归属的轮次，计数加一。一轮的代价（模型调用、工具执行、token 花销）全部记在 goal 头上，预算烧没烧、烧在哪儿，日志里逐条可查。

轮数不能超过 `maxGoalRounds`，默认 256。调用方在 create 时省略 cap，由服务配置内部解析默认值。超出预算就 block，blocked 快照带着策略给的 code，把"为什么停"机器可读地留在了日志里。

### 一次崩溃恢复，从头到尾

把两层的分工放进一次真实的崩溃里走一遍。某个 goal 处于 active，revision 12，`roundsStarted` 37，`maxGoalRounds` 256，续跑消费者 armed，第 38 轮刚提议。进程崩了。

重启之后发生的第一件事是 session-start 边触发：activation 缓存被 disarm。然后日志重放，`goal/change` 事件序列逐条 fold：create（revision 1）、几次 edit 和 pause/resume（revision 一路涨到 12）、37 条被归属的 `user/message`。fold 的输出是 active、revision 12、`roundsStarted` 37。所有跨重启该活下来的东西都活下来了，因为它们本来就从日志来。唯一没活下来的是 activation：它从不持久化，此刻是 disarm 的。

于是用户打开会话看到的是：目标还在，进展还在，agent 安静。续跑消费者看着 disarm 的 activation，不会自己开第 38 轮。用户看完状态，决定 resume：一个新的人授权的 activation 边被记录，预算还剩 219 轮，续跑开跑。整个恢复过程没有一个数字需要"同步"回来，也没有一行代码需要在重启时猜测"上次跑到哪了"；哪些值来自日志、哪些来自进程，边界从头到尾清楚。

### 操作一览

`GoalService` 的操作各自有明确的边界。`get` 返回新鲜视图或 undefined，且强制确切的 live agent 身份，不是活实例直接抛错。`create` 有一条替换规则：完成的 goal 可以被直接替换，其他当前阶段得先 clear 或 resume。`edit` 改目标和/或轮次上限，不改阶段，运行时校验要求至少改一个字段。`pause` 暂停一个活跃目标并解除自动续跑。`resume` 在轮次预算还有容量时 arm 一个停了的目标，或在 session-start 边后 rearm 一个活跃目标。`complete` 标记完成并解除。`block` 标记 blocked 并解除，带上 reason。`clear` 清除当前目标但保留持久墓碑和历史，返回的 ref 比被清快照的 revision 大一。

每次变更发出隔离的 `goal/changed` 通知，带被接受的操作和确切 revision，clear 时省略 goal 字段。通知的分发是 scope 过滤的：agent 作用域的监听器只收到那个 agent 的事件。监听器失败被包含在内，不会反过来影响已经提交的事实：`goal/change` 会话事件先落账，通知只是广播。操作里还有一个 `remoteExportCreate`，给远程边界上的创建用，Web 客户端创建 goal 走的就是它；`get` 和 `disarm` 强制确切的 live agent 实例，对着一个已被替换的旧实例调用直接抛 `GoalError`，防止调用方拿着过期的服务句柄改错对象。

## 并排看，再串一遍

| | Plan Mode | Goal |
|---|---|---|
| 重量 | 轻，一个提示段 | 重，完整生命周期 |
| 强制 | 软引导，无强制 | 持久阶段加进程内 activation 两层 |
| 状态来源 | `plan/mode` 事件纯 fold | `goal/change` 事件严格 fold |
| 防并发 | 无（布尔状态） | compare-and-set revision |
| 轮次预算 | 无 | `maxGoalRounds` 默认 256 |
| 进模型 transcript | 不进 | 派生值可给模型看 |
| 与硬强制关系 | 明确分开 | activation 与持久阶段分开 |

一个真实场景把两者串起来。用户要做一个跨好几天的重构，先敲 `/plan`，plan mode 激活，每个请求都带着部署那段"先出计划"的引导。agent 产出一份 markdown 计划，调 `exit_plan_mode`，人审通过，pending 退出在下一个 turn 边界追加，plan 引导退场。整个过程里两个机制互不知晓：plan mode 不关心有没有 goal，goal 也不读 plan 状态，把它们串成一条工作流的是用户和部署，框架只保证各自的那一段不出错。随后用户创建一个 goal，给了 40 轮的预算。agent 开始干活，每个被归属的续跑 turn 推进 `roundsStarted`，中途遇到依赖版本冲突，block 成 `blocked`，reason 的 code 是 `dependency-conflict`。用户解决冲突后 resume，goal 重新 arm。第二天进程重启，用户打开会话：goal 状态原样（从日志 fold 出来），activation 是 disarm 的，agent 安静地等着。用户 resume，工作继续，预算还剩多少，日志里数得清清楚楚。

## 权衡

plan mode 的软是刻意的，代价也摆在明面上。它没有强制力，模型不听引导时能兜底的是沙箱和审批，用 plan mode 当安全边界是用错了东西。pending 选择的落盘要等 turn 边界的合适时机，进程在下一个被接受的 in-turn pre-step 之前退出，选择就丢了；交互通道缺失时 `exit_plan_mode` fail-closed，用户得记得 `/plan off` 这条手工通道。这些限制文档里写得明明白白，不是踩坑之后才发现的暗礁。

goal 的重同样有成本。每次修改都要带 revision，stale 被拒后调用方得处理读改重试。长目标的事件持续累积，fold 成本随日志增长，靠持久化投影缓解冷读。它还有几条明示的边界：一个会话一个 goal；信任边界在进程，能写日志的恶意写手可以伪造记录，系统能检测不一致但不修复；没有调度器、没有评估器、不承诺版本迁移。

两个包都不在 agent-loop 主干里。最小化的 headless 部署可以两者都不装，loop 照常跑。这是"一切皆插件"在目标管理上的落点：连"agent 该有什么目标"本身都是一个可插拔的问题。

## 结论

`ctx.planMode` 是软引导：激活时每个请求多一段部署拥有的提示，状态是 `plan/mode` 事件的纯 fold，选择在 turn 边界追加以保证请求的原子性，退出走一次被人审的日志工具调用。`ctx.goals` 是持久的事件溯源生命周期：revision 防并发、四阶段记状态、两层授权管续跑、轮次预算只数 goal 的账，全部从 `goal/change` 事件 fold，日志之外没有第二个真相源。

贯穿两者的纪律有两条。状态从日志派生，所以恢复和 fork 不需要额外协议；引导、追踪、强制三者分离，所以软的不会被误当成硬的，硬的也不会被软的架空。用户选的沙箱模式永远生效，agent 的目标状态永远可重建，进程重启后永远没有 surprise。

## 延伸阅读

- [Plan Mode 官方文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/plan.md)：本文主要依据之一，软引导与 turn 边界 flush
- [Same-session goals 官方文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/goal.md)：本文另一主要依据，事件溯源目标生命周期
- [Plan-specific collaboration state 笔记](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/simplification/2026-07-22-plan-specific-collaboration-state.md)：plan mode 设计理由
- [Persisted same-session goal domain 笔记](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-07-19-persisted-same-session-goal-domain.md)：goal 持久化与激活决策
- [Sandbox](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/sandbox.md) / [Approval](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/approval.md)：plan mode 明确分开的硬性强制

上一篇：[dsh 的跨会话记忆：session-query / projection / reference](./28-cross-session-memory-query-projection-reference.md)
下一篇：[子 Agent 与多智能体：dsh 怎么调度另一个 agent](./30-subagents-multi-agent.md)
