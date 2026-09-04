# dsh 的 Conversation Node：给 Web 写一个自定义渲染节点

> Conversation Node 是把一组相关的会话事件折叠成一个有状态视图单元的机制。你写一个 Definition，声明哪些事件属于我、状态怎么从事件里长出来、什么时候发布、发布成什么渲染数据；引擎兜住剩下的全部：回放、分页一致性、乱序归位、Context 生命周期、渲染调度。写好一个 Node 的难度几乎全部压在两件事上：事件族设计得可回放，以及 append 路径不做全窗口扫描。
> 两条最值得记住的纪律：永远不要把一个 update 分配给"最近一个还没结束的"Context，身份必须来自事件自己携带的持久 id；一个 Node 的 key 只能来自 kind 加 id，永远不能来自 seq 或渲染位置，running 变 settled 的瞬间组件不 remount。

## 自己监听事件为什么注定失败

想给对话界面加一张自己的卡片，比如一个代码审查的进度条，最直觉的写法是：开一个事件订阅，把属于这个审查的事件攒进一个自己的 store，store 变了就重渲染。单机演示这样能跑，放进 dsh 的 Web 客户端里它会在三个真实场景下坏掉。

第一个场景是往回翻页。会话历史不是一次性到齐的，客户端从尾部开始一页一页往回要。用户滚到顶上，引擎加载更早的一页，这一页里可能有整条审查的 start 事件，而你已经用后到的 update 事件渲染了半天。自己管状态的代码此时要回答：这些已经显示的进度，要不要推倒重来？按什么顺序重算？

第二个场景是 resync。断线重连或检测到日志间隙时，引擎会把加载的窗口整个重建。你自己攒的那份内存状态没有任何办法跟着重建，因为它不是从日志推出来的，是沿着时间一点点捡来的。

第三个场景是乱序到达。网络让事件的到达顺序和日志顺序不一致是常态。你按到达顺序折叠状态，折叠出来的东西就和日志的真实顺序对不上，而且这种错没有任何报错，它只是安静地错着。

这三个场景指向同一个根因：你把"状态"存在了到达顺序里，而到达顺序不是事实，日志顺序才是。Conversation Node 机制的全部设计就是把状态搬回日志顺序上：引擎持有事件窗口，按 seq 升序回放，你的状态是回放的产物而不是到达的沉积物。这套机制落在 2026-08-09 合入的客户端对话节点组装决策里，Host 侧的会话投影是另一套独立机制（2026-08-19 的投影状态拆分决策），两边不要混。

## 心智模型：账本与视图分离

把会话日志想成一本账本，每行一个事件，每行有全局递增的行号 seq。对话界面不是账本本身，是账本的一个投影。dsh 在这套投影机制里分了两层角色：业务插件在 Host 侧往账本上记账，视图插件在 Client 侧声明"这一族行怎么折叠成一张卡片"。

分层落在职责表上：Session 只负责维护当前连续事件窗口、区分 replace、prepend、append 三种摄入、调度快照通知，不解释任何业务事件；Event Registry 按 Cordis 生命周期保存唯一 kind 的 Definition 和唯一 fallback；Assembler 匹配事件、维护 Context、Location、依赖和发布脏集；Node Definition 定义一个业务对象的身份、State 演进和渲染贡献；View Builder 把最终 Node 增量整理成该视图的快照；React renderer 按 kind 展示，只读自己 Node 所属 Location 的只读业务数据。

折叠的配方就是 Definition。它声明四件事：哪些事件属于这个 Node（match），状态怎么从事件里构建（start 加 update），状态变化什么时候对外发布（publication），发布成什么样的渲染数据（buildViewNode，可选搭配 buildLocationData）。剩下的全是引擎的责任。

有一个前提要先说清楚：引擎只能折叠已经被记到账本上的事件。所以写 Definition 之前，先要在 Host 侧的业务插件里把事件记全。事件的类型通过 `SessionEventMap` 的声明合并注册，在 `declare module '@deepseek-ai/dsh-session/types'` 里给接口加三条，`'review/start'` 对应开始载荷，`'review/progress'` 对应进度载荷，`'review/end'` 对应结束载荷。branded id 类型跟着 producer 走，跨进程边界使用，防止审查的 id 和别的字符串撞在一起还查不出类型错误。

## 事件族设计：先想清楚再动手

Definition 写得好不好，八成取决于事件族设计得好不好。三条规则，每条都有具体的失败方式。

第一条：每个事件要么携带一个稳定的业务 id，要么能从自己的 payload 推导出来。被明文禁止的反模式是"把 update 分给最近一个还没结束的 Context"。这个写法在单事件顺序到达时完全正确，于是它能通过你所有的手动测试；翻页把两条审查的进度交错加载时，它把 A 的进度记到 B 头上，而且没有任何机制会告诉你错了。稳定 id 意味着每个事件自己就够定位自己的归属，不需要任何"当前正在进行的那个"这类运行期记忆。

交错不是臆造的边角，是常态。用户先对 PR 提了审查 A，问到一半又对另一个文件提了审查 B，日志里就是 A 的 progress、B 的 progress、A 的 end、B 的 progress 这样犬牙交错。按 id 归属，A 和 B 各自的状态在回放下毫无歧义；按"最近的"归属，B 的最后一条进度会被算给已经结束的 A，或者反过来，取决于事件到达的运气。

第二条：每个 `(kind, id)` 最多一个 start 事件，第二个 start 直接失败，id 永不复用。一次代码审查对应一个 `reviewId`，从 start 到 end 用同一个，开新的审查就发新的 id。听起来像废话，但它是回放确定性的前提：如果同一个 id 可以有两次开始，按 seq 回放时状态就有了两种合法读法，整套机制就没有确定答案了。

第三条：增量事件必须在按 seq 升序回放时产生确定性的 State，不能依赖 live-only 的内存状态。设计上更推荐整值 checkpoint：进度事件直接带"当前完成了几项、还剩几项"的完整快照，而不是只带"又完成了一项"的增量。整值 checkpoint 对窗口边界免疫，start 落在已加载窗口之外时，一个带完整状态的 terminal 或 checkpoint 事件照样能撑起渲染。纯增量在此处无解，只能等 start 到达。

以审查 Job 为例，事件族长这样：

| 事件 | 角色 | 必须携带的持久事实 |
|---|---|---|
| `review/start` | 唯一的开始 | `reviewId`、Turn/Step 坐标、标题 |
| `review/progress` | 更新 | 同一个 `reviewId`、坐标、可回放的进度 |
| `review/end` | 更新 | 同一个 `reviewId`、坐标、最终摘要 |

只有一个事件的简单业务可以偷懒：用事件自身的稳定身份（比如 `event.seq`）当 Definition-local id，start 和 update 就是同一个事件。内建的 inbox 就是这个形态，每条 splice 事件自成一个瞬间态 Context，靠 reader 链把它们串起来。这在机制里是合法的捷径，代价是这条事件既是开始也是结束，后续想加第二个事件时 id 体系要重新设计。

还有一条容易踩的协议边界：到达顺序可以反，业务日志顺序不能反。一个 update 事件的 seq 早于它的 start 事件的 seq，回放直接失败并报协议错误。翻译成业务语言：你不能先记账"审查结束了"再记账"审查开始了"，哪怕现实里你是先收到结束通知。事件写入日志之前把顺序摆正，是 producer 的责任，不是引擎能修的。

## Definition 的成员契约

落到代码，一个 Definition 是一个对象，审查 Job 的定义叫 `reviewDefinition`，类型是 `ConversationNodeDefinition` 加上你的 State 类型参数。逐个成员看契约，看的时候记住一个总的分层：match 管身份，start 和 update 管状态，publication 管节奏，buildLocationData 和 buildViewNode 管产出。

`match(event)` 是身份提取器，不是折叠器。它只看当前这一个事件，返回 Definition-local 的 id 加生命周期角色（start 或 update），不属于自己返回 null。它拿不到 Context、拿不到历史、拿不到 Reader，这个刻意的隔离是为了让匹配在任何摄入路径下表现一致：replace 时对每个事件问一遍，append 时只对最新的一个问一遍，答案必须相同。审查 Job 的匹配就三分支：start 事件返回 start 角色，progress 和 end 返回同一个 id 的 update 角色，其余返回 null。同一个事件可以被多个普通 Definition 同时认领：一条 Assistant 事件同时更新 assistant-step 和 turn-tail 两个 Context，互不干扰。每个 Session 还有一个唯一的 fallback Definition（内建 kind 是 `unknown-surface`），只在所有普通 Definition 都返回 null 时被问到，把没人认领的 append 面事件兜成一行 JSON。

`start(context, match)` 在 start 事件上跑一次，是唯一的状态初始化点，返回 undefined 是契约错误。`update(context, match)` 在每个 update 事件上跑，返回新状态，返回 undefined 同样失败。推荐返回新的不可变值，但原地修改后返回同一个对象也有相同的 adoption 语义，引擎采纳的是返回值而不是引用比较。更细的一条：发布不按 State 引用相等做门控，每个被接受的 update 都会递增 Context 的 revision 并重新评估它的 Reader 消费者，哪怕你返回的对象没变。"内容没变就不触发下游"这种优化引擎不提供，需要的话自己在上层做。

`publication(match)` 控制状态变化物化成视图发布的节奏，三档。`immediate` 是默认档，走微任务，给结构性或终止性变化用。`animation-frame` 把同一帧里的多次发布合并到下一帧，给高频可见增量用，assistant 的流式 token 就是这一档，不合并的话每个 token 一次 React 通知，界面会被自己的输出淹没。`none` 完全不发布，给只喂给后续 publication 的中间状态用，inbox 这种不可见容器就是典型。要点是 cadence 只合并视图发布，update 本身永远按日志顺序逐个执行，状态不会被合并，被合并的只有"通知 React 重画"这一下，token 不会丢。一个 Context 在一次 append 里命中多个档位时，最高紧迫度获胜，immediate 压过 animation-frame 压过 none。内建 assistant 的用法更细：chunk 走 animation-frame，final 走 immediate，纯 usage/finish 的分片走 none。

`buildLocationData(context, scope)` 是可选的跨 Node 通道。它把 Definition 拥有的数据发布到引擎拥有的 Turn 或 Step 上，scope 不是 step 或 turn 时返回 null。Assembler 永远先物化 step 再物化 turn，顺序是引擎保证的，所以 Turn 级聚合可以读同一轮刚更新的 Step data。消费方通过受约束的 hook（如 `useTurnData`）读取，拿到的是只读的 `data.get(key)`，没有遍历、没有修改。值类型通过两张 map 的声明合并注册：`ConversationStepDataMap` 管 Step 层，`ConversationTurnDataMap` 管 Turn 层。另一个 Context 想占同一个 Location key 会被拒绝。

`buildViewNode(context)` 产出渲染就绪的 Node，和 `target` 必须成对出现，声明一个 target 拥有的渲染贡献。三条纪律。保留 `context.key` 作为 React 身份，引擎会校验。`anchorSeq` 从持久排序证据里选：start 的 seq，没有就第一个匹配的 seq，再没有就 0，它决定这张卡片在对话流里的锚点位置，所以不能来自"当前渲染到第几行"。一旦 target Node 发布了，就持续返回同一个 key，需要暂时离开可见流时用 `visibility: 'hidden'`，不要返回 null 撤回它，一个已经物化的非 null Node 在增量路径上变回 null 是被引擎拒绝的。

`current` 和 hidden 的区别值得单独说，因为它来自一个真实的实现需求。引擎区分"从未物化过"和"物化过但现在隐藏"两种状态，assistant 用这个区分做重试抑制：一条流式回复被中断后进入重试，重试期间旧卡片隐藏而不是销毁，重试成功旧卡片回来，React 的组件状态（滚动位置、展开的折叠块）全程没丢。如果你的业务也有"暂时不可见但别拆掉"的形态，这是同一条路。

最后一条契约边界：没有通用的 `end()` 生命周期。终态就是一个普通的 update 事件，你的 State 里自己标 `status: 'done'`。引擎不理解"结束"，它只理解"又一个 update"，这让引擎侧少一整个概念，代价是你的状态机自己收尾。

## 三条摄入路径：一次完整的使用过程

引擎从三个路径接收事件，历史可以从尾部往回一页一页要，但每个被接受的页面都先归一成按 seq 升序再做回放。用一个真实的使用时间线把三条路径串起来。

用户打开一个有审查记录的会话，Replace 路径跑起来。它清理全部 Context、索引和依赖，把窗口内的事件按 seq 排好，每个事件对每个 Definition 匹配一次，回放每个已启动的 Context，最后让视图构建器整体 replace。触发 Replace 的不只是初次打开：断线 resync、日志间隙修复、注册表变化（热重载了一个 Definition）都会走这条路。它的语义是"不信任之前的一切，从窗口重推"。

用户往上滚，加载更早的一页，Prepend 路径跑。它先去掉和已加载窗口重叠的 seq，只匹配真正新增的更早事件，按 `(kind, id)` 合并进现有 Context，已有 keyed Node 的身份全部保留，只回放受影响的 Context 和它们的依赖，最后视图构建器做增量 upsert。这里藏着 pending Context 的复活机制：如果之前窗口尾部只有审查的 progress 没有 start，assembler 早就为它建了一个 pending Context，事件都收着、状态不构建；更早的页面把 start 送来，这个 Context 立刻激活，把收集的 update 按 seq 升序一次性回放，产出的结果和一开始就完整加载一模一样。这也是验证机制正确性的核心命题：相等的事件窗口，不管从哪条路径进来，产出相等的状态和 Node。

live 事件到达，Append 路径跑。它只接受紧贴窗口尾部的连续事件，对每个 Definition 的 match 调一次，按 key 常数时间找到匹配的 Context，只更新那一个，不扫描任何现有 Context。一条审查的 progress 到达，全窗口可能有一万个事件、二十个 Context，这次 append 的成本是 D 次匹配加一次哈希查找，和窗口大小无关。

Prepend 还负责解决一个微妙的依赖问题。`reader.previous(kind)` 返回的是 start seq 之前最近已启动的 Context，而"之前最近"这个判断依赖当前加载的窗口：也许更早的页面里还有一个更近的前驱。所以依赖查询的 miss 分两种。窗口还有更多页没加载（hasMore 为 true）时，miss 是暂时的，依赖被标记为 provisional；一个空页把 hasMore 翻成 false，之前的暂时 miss 被正式解析为"确实没有前驱"。如果后来一个更早的 prepend 供上了更近的前驱、关闭了一个之前不知道的间隙、或者修改了前驱的状态，引擎从 start 重跑依赖的 Context，按 seq 升序回放。依赖永远指向严格更早的 Context，时间上单向，传递重放不可能绕成环。依赖的登记跟着调用走：start 里每调一次 previous，之前登记的依赖就被替换成这一次的，不会累积出一份历史调用的旧账。

事件到坐标的映射也有兜底阶梯。一个 Context 的位置取 start 事件的 Location，没有 start 就取第一个匹配事件的 Location，都没有就落成 unresolved。位置一共四种形态：session 级、turn 级、step 级、unresolved，各带 open、closed、unknown 的状态。窗口往前翻页会重建这些位置事实，但没变的 Turn 和 Step 引用原样保留，不因为一次 prepend 就让下游的引用比对全部失效。业务侧的含义很实际：审查卡片挂在哪个 Turn 下面，由事件自己携带的坐标决定，坐标缺失时引擎给出明确的 unresolved 而不是一个猜出来的位置，宁可标"不知道"，不编一个像样的答案。

## 性能不变量：append 路径的硬约束

Definition 代码必须守住一条量化的不变量：注册了 D 个 Definition 时，一个事件的到来做 D 次当前事件匹配，匹配后做常数时间的 Context-key 查找。append 的成本是 O(D) 加 O(1)，和窗口里有多少事件、多少 Context、多少已渲染 Node 全部无关。

这条不变量翻译成禁令就是四不准：在 append 路径上不准遍历完整的事件窗口，不准遍历所有 Context，不准遍历 `context.matches`，不准遍历已渲染的 Node 集合。违反任何一条，append 从 O(D+1) 退化到 O(N)，一个长会话越用越卡，而且卡得毫无必要。

替代方案机制都给了。累积事实放进 State，不要每次都从头数事件。同 Turn 或同 Step 的共享走 Location data，消费方 `data.get(key)` 一次拿到，不要互相扫描。前驱查询用 `reader.previous(kind)`，它是索引化的一次查询，引擎替你记账依赖，比"扫一遍找上一个"又快又正确。

`reader.previous` 只在 `start` 里可用，返回只读数据，不授予修改权，也不暴露业务特定的查询方法。这两条限制是一对的：给你一个通用的"最近前驱"，你就没有理由要求"上上一个"或"指定 id 的那个"，引擎也就不用维护一族谁都用不上两次的查询接口。

## keyed renderer：React 组件的接入

渲染侧是一个 keyed React 组件，通过 `ChatNodeViewProps` 接收 Node 数据。类型参数把组件钉死在一个 kind 上：`ChatNodeViewProps<'review-job'>` 的组件只服务 review-job 这一种 Node。审查卡片的实现就是拿 `node.data` 里的标题、完成度、摘要拼一张卡，没有任何别的输入源。

组件的消费面被刻意收窄：只消费 `node.data` 和受约束的 Location hook。它不接收 Session 事件、不扫描 Context 集合、不访问其他 Chat Node。这个收窄和 Definition 侧的性能不变量是同一堵墙的两面：引擎保证匹配是常数级，组件保证渲染不绕过引擎自己找数据，两边一起守住"事件量增长不拖慢界面"。`useTurnData()` 是收窄常见读取方式，不是权限沙箱：session-scoped renderer 仍持有标准 `useSession`，全窗口统计可以显式用 Session snapshot，只是不能把它伪装成"当前 Node 的 Turn data"。

注册分三步，写在 client 插件的 `apply` 里。先声明依赖，导出 `inject` 数组包含 `conversationEvents` 和 `slots`。再调 `ctx.conversationEvents.register(reviewDefinition)`，把 Definition 注册进引擎，kind 就是注册名，重复的 kind 会被注册表拒绝。最后调 `ctx.slots.inject('conversation.chat.node', ...)`，在注入回调里用带 key 的插槽注册把组件挂上，key 就用 `'review-job'`。

数据类型同样走声明合并：在 `declare module '@deepseek-ai/dsh-client-ui-conversation/client'` 里给 `ChatNodeDataMap` 加一条 `'review-job'` 对应你的渲染数据类型。这让每个 key 有精确的值类型，组件消费的数据形状和 Definition 产出不一致时，TypeScript 在编译期就报错，不用等运行期渲染出一张空卡。

三条摄入路径对 Node 身份的处理有一条共同结果值得点名：ChatNodeSeat 在同一个父列表里保持自己的 Context key，running 变 settled 不 remount。用户正在展开的折叠块、正 hover 的按钮，不会因为状态从"进行中"翻成"已完成"而闪一下重置。稳定性来自 key 只由 kind 加 id 派生，`conversationContextKey` 保证跨 kind 不碰撞，和渲染位置、和 seq、和可见性全部无关。要和业务主动撤显区分开：业务把已发布 Node 改成 hidden，它退出可见流，恢复 visible 时会重新 mount，这是明确的撤显语义，与 running 到 settled 的稳定 Seat 保证是两回事。

视图层的快照构建器（ChatSnapshotBuilder）维护几个切片：排序、按 key 索引的 Node、位置索引、时间线。判断"这次变化是不是结构性的"有明确标准：出现新 key，或者 anchorSeq、可见性、Location 身份变化，才算结构变化；只是 data 变了就走值更新。这个标准把 React 的重渲染成本和业务语义对齐了，多数 progress 事件只改数据不动结构，界面做一次值更新就完，列表不重排、组件不重建。

## 内建 Definition 全家福

看一遍内建实现能学到所有形态。截至 2026-08 的 Chat 注册表里有十一种业务 Definition 加一个 fallback，各自展示了 id 与生命周期的不同选法。

inbox-next-turn 和 inbox-next-step 用 splice 事件的 seq 当 id，每条事件自成一个 start-only 瞬间态 Context，靠 `reader.previous(ownKind)` 链成连续状态，claimed 集合供 message 消费。input-message 用消息 id，start 是 `user/message`，根据 source 或最近的 next-step inbox 把消息分类成 user、steering 或 context。assistant-step 用 `turn:step` 当 id，聚合 blocks、usage、首 token 时间和重试隐藏状态。tool-call 用根 call id，聚合 root、children 和 parent Map，Code Dispatch 的事件用 `rootCallId` 精确路由。command 聚合 `command/run` 到 `command/done` 的结果。compaction 在缺 start 时可用 checkpoint 做 fallback Node。model-retry 聚合同一 retryId 的 `llm/retry` 与 `llm/retry-started`。turn-error 只认 error 型的 `turn/end`，同一个 turn 的重试链在它旁边渲染，绝不隐藏。turn-max-tokens 是 2026-08-12 之后新增的，match 的是 reason.kind 为 max-tokens 的 `turn/end`，给被输出上限截断的轮次一行提示。turn-tail 读各 Step 的 assistant data 算出 closing 与指标，发 Turn data。deliverables 聚合一个 Turn 内成功的变更路径并发布 Turn data，自己不生成 Node。unknown-surface 兜底没人认领的 append 面事件。

这张表里每一行都是"稳定 id 加正序回放"的一种解。写自己的 Node 之前对着它找最像的形态照抄骨架，比从零设计快而且稳。

## 被拒绝的替代方案

这套机制不是没考虑过更直的路。架构笔记里记录了一串被明确拒绝的做法，看懂拒绝理由比记住结论有用。

最诱人的方案是让 Session 自己做集中折叠，提供一堆查询助手给业务用。被拒的理由是职责倒挂：业务怎么折叠是业务的事，折进核心意味着每加一种卡片都要动 Session，一个折叠 bug 污染所有卡片共用的核心路径。现在的分层里 Session 只管窗口和调度，折叠逻辑全在业务插件里，坏也只坏自己一家。

第二条被拒的路是让 React 组件直接扫描事件窗口自己算。这在原型期最快，但渲染成本随窗口线性增长，且回放逻辑在每个组件里重写一遍，replace 和 prepend 的语义没人保证。等于把本篇开头"自己监听事件"的三个失败场景原样搬进渲染层。

第三条是允许 matcher 读 Context。听起来无害，实际会引入路径依赖：replace 时 Context 状态和 append 时不同，matcher 一旦读 Context，同一个事件在不同路径下就匹配出不同结果，"相等窗口产出相等状态"这条核心不变量直接作废。match 只看当前事件，不是性能优化的选择，是正确性的选择。

第四条是给引擎加通用的 `end()` 生命周期，让引擎理解"结束"。它被拒的理由：引擎多一个概念，所有 Definition 多一个要不要实现的选择，而绝大多数业务的"结束"只是 State 里一个标记，引擎根本不需要知道。类似的还有把 inbox 提成一级概念、给 reader 加业务特定的查询方法、允许组件直接读别的 Definition 的 State、为历史反扫定义逆向 State fold、在同一个 Definition 里为 Chat 和 Trajectory 两个 target 分支，拒绝逻辑同出一源：每一个都是往引擎里塞只有个别业务用得上的东西，塞进去的每一样都是所有业务要共同承担的复杂度。

## 验证清单

写完一个 Conversation Node，用这六条验证，每条对应一类真实故障：

1. 完整窗口 replace 产生预期的最终 State、Location data、Node payload 和 anchorSeq。这条验的是折叠逻辑本身。
2. 只有 update 的尾部保持 pending；prepend 进唯一的 start 后，产出和完整 replace 完全一致。这条验的是分页复活。
3. 初始历史后 live append，产出和把合并窗口整体回放一致。这条验的是 live 与历史同构。
4. prepend 更早的页面只添加更早的行，不替换数据未变的已有 keyed Node。这条验的是身份稳定。
5. 重复的可见 delta 保留 `context.key`，请求 cadence 下每帧最多发布一次。这条验的是合并与身份。
6. keyed renderer 只消费 `node.data` 和受约束的 Location hook，不扫描事件窗口、Context 或 Chat Node。这条验的是消费面收窄。

仓库里有三个现成的参考实现，各代表一种形态。`assistant.ts` 做流式和中断，是 animation-frame 合并和 current 隐藏语义的出处。`inbox.ts` 加 `message.ts` 做前驱查询，是 reader.previous 的实战样本。`ui-deliverables` 是一个发布 Turn 数据但不创建自己 Node 的 Definition，示范 Location data 通道的独立用法。三条路径六条检查都过，你的 Node 在翻页、resync、乱序、live 四类场景下才有站得住的理由。

## 权衡

这套机制的第一条代价写在前提里：事件必须可回放。业务逻辑依赖 live-only 的内存状态，Definition 就会出错，所有 State 必须在按 seq 升序回放时确定性地重建。这不是建议，是硬约束，协议错误会在回放失败时直接抛出来。连带的一条是 start 之前不能渲染：窗口里只有 update 没有 start，Context 保持 pending，产品必须在 start 加载前就显示的话，需要一个携带完整 fallback 状态的 checkpoint 事件，纯增量没有绕路的办法。

Node 之间不共享状态。一个 Node 不能直接读另一个 Node 的 State，需要共享时用 Location data 发布到 Turn 或 Step 层，消费方拿到的是只读快照。这个限制换来的是每个 Context 独立回放：任何一个 Definition 的重算不需要排序、不需要锁、不需要关心别的 Definition 在不在半路上。如果 Node 之间可以互读，一次 prepend 就得决定重算的拓扑顺序，机制复杂度直接上一个量级。

引擎不理解"结束"。没有 end() 回调，终态是你 State 里的一个标记。想让卡片在结束后收起来、变灰、可折叠，全是渲染层读自己 State 的事，引擎帮不上也不需要帮。反过来，任何依赖"引擎知道这事完了"的设想（比如自动清理 Context）都不成立，Context 的生命由窗口决定，不由业务决定。完成的 Context 继续存在于当前窗口，这是特性不是泄漏：它既提供稳定渲染 identity，也作为后续 Reader 的前序证据。

Definition 作者的学习成本是实打实的。官方决策文档把代价列得很直白：要理解稳定 ID、唯一 start、正序 replay、Step 到 Turn 的发布顺序、只读 Reader 和 Node 不撤回规则，哪一条理解偏了都会在分页场景炸出协议错误。换来的才是"新增一种卡片不改 Session 的任何代码"这个扩展性质。业务包要么吃下这套心智模型，要么退回"让 Host 侧直接发渲染就绪事件"的偷懒做法，后者把分页、乱序、resync 的问题原样留给自己。

当前教程的 target 固定为 chat。Trajectory 是另一个 target，它针对同一个 Assembler 和 Session 事件窗口注册自己的 Definition 和 Builder，保留 stage 式读模型；共享定义跨 target 分支的做法在架构上被明确拒绝过，每个 target 有自己的贡献声明。

## 结论

Conversation Node 的分工线划得很清楚：你声明事件身份、状态构建和渲染数据，引擎兜住回放、分页一致性、乱序归位和常数级性能。写 Definition 时守住两条就够了。事件族可回放：稳定业务 id，每个 `(kind, id)` 至多一个 start，delta 按 seq 升序能确定性重建，需要 start 前渲染就发带完整状态的 checkpoint。append 路径不扫描：累积进 State、共享走 Location、前驱用 reader.previous。做到这两条，翻页、resync、乱序到达、live 追加就都是引擎要处理的情况，不是你要处理的。

## 延伸阅读

- [添加 Web Client Conversation Node 官方教程](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/cookbook/adding-a-conversation-node.md)：事件族设计到 renderer 注册的完整代码样例
- [Client Conversation Node 组装决策](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-09-client-conversation-node-assembly.md)：引擎模型、三条摄入路径与全部被拒方案
- [assistant.ts 参考实现](https://github.com/deepseek-ai/deepseek-harness/blob/master/packages/client/ui-conversation/src/client/conversation-nodes/assistant.ts)：流式合并与重试隐藏的出处
- [ui-deliverables（无 Node 的 Definition 参考）](https://github.com/deepseek-ai/deepseek-harness/tree/master/packages/client/ui-deliverables)

上一篇：[dsh 的排查与调试：全插件化 harness 怎么追问题](./38-debugging-and-troubleshooting.md)
下一篇：[dsh 的 Python SDK、Headless 与 JSON-RPC：把 agent 编进流水线](./40-python-sdk-headless-jsonrpc.md)
