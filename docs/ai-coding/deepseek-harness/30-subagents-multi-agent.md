# 子 Agent 与多智能体：dsh 怎么调度另一个 agent

> `ctx.subagents` 是一个命名 provider 注册表，六种后端共存，把委派分成两类：一次式是一个可丢弃的 run，拿一个结果就 dispose；可继续是一个持久子会话，带进程内 Activation，能多轮 FIFO 对话、能冷恢复、能上报给父。可继续那一半有三条硬规矩：Agent inbox 是唯一队列，授权只认活的直接父 Agent 对象，provider 只贡献首次创建的数据、之后的整个生命周期归 continuation manager。这三条规矩合起来，让"和另一个 agent 长期协作"落在和普通会话完全相同的标准机制上。

## 为什么这个接缝是多 provider 共存

bash、workflow 这些接缝，一个 context 一个实现：再注册第二个 executor 直接抛错。这对 bash 是合理的，一台机器跑命令就一种方式。子 agent 不是。同一个父会话里，可能这一刻要把一个便宜的小任务委派给同进程的子 agent，下一刻要驱动一个隔离的进程外 agent 走 ACP 协议，再一会儿要把一段活儿整个交给 Codex 或 Claude Code。这些需求的传输机制、上下文继承、生命周期全都不同，单实现接缝内部就得长满分支。

所以子 agent 接缝学的不是 bash，是 LLM 适配器注册表：多个后端按名字注册到 `ctx.subagents`，共存于一个 context。当前六个 provider：`spawn-in-process`、`fork`、`acp`、`codex`、`claude-code`、`dsh-sdk`。选择是配置驱动的，不是模型驱动的：工具绑定到确切的 provider 名，模型只看到描述和提示框；要同时暴露多种传输，就把工具插件按不同工具名多挂几次。

模型面向的消费者有三个：`tool-subagent` 按 provider 委派，`tool-subagent-control` 提供全局的 `send_message`、`interrupt_agent`、`list_agents`，`tool-subagent-report` 给子 agent 自己一个 `report` 回报通道。

## 两类能力，两种发现方式

provider 的能力分两类，发现方式完全不同，这个差别本身就说清了两类委派的分工。

启动期能力是静态描述符 `SubagentCapabilities`，四个布尔：`outputSchema`（结构化输出）、`depthLimit`（深度上限）、`toolFilter`（工具过滤）、`persona`（人设）。服务在一次式 run 存在之前就检查它：请求需要某个能力而 provider 不支持，用 `UNSUPPORTED_CAPABILITY` 大声拒绝，绝不"接受了然后忽略"。为什么必须是启动期检查而不是运行期方法？因为调用方依赖这些语义构造请求，静默降级会让请求在没有承诺的语义下继续跑。配了结构化输出，provider 其实不支持，返回却当普通文本解析，错在下游才爆。

可继续能力的发现走另一条路：看可选方法 `prepareContinuable` 在不在。方法在，能力就在，靠 TS 类型收窄。不需要单独的标志位，也就不存在标志位和实现漂移的问题。为什么可以这样？因为一次式子 agent 是 provider 自己组装的，能力得提前声明；可继续子 agent 是 continuation manager 组装的，provider 要表态的只剩愿不愿意参与创建。

## 一次式：一个 run，一个结果

一次式委派返回 `SubagentRun` 句柄：一次性的前台委派，带一个结果，绝不是持久的子句柄。消费者 await `result`，然后总是 dispose 达到静默，dispose 幂等。

失败语义值得细看。子 agent 级别的失败不让 `result` reject，而是 resolve 成非 completed 的停止原因；只有无法表示的基础设施故障才 reject。`stopReason` 是可合并扩展的联合：`completed`、`aborted`、`error`、`max-tokens`、`refusal`，遇到未知的值按失败处理。非 completed 意味着 output 可能残缺，消费者把它映射成 isError 工具结果，不把半截输出当成功。`structured` 只在请求了 schema 且成功满足时才出现，请求过不保证有。

结果旁边有个 `diagnostic`，它的边界很讲究：provider 会剥掉工具输入、文件内容、环境变量值、凭证、协议载荷，完整值上限 4096 UTF-8 字节。诊断是给排错看的，不是第二个数据通道。

生命周期是成对的。服务铸造 `runId`，发布时发 `subagent/start`，结束时配对的 `subagent/end` 带最终输出或基础设施故障。本地 run 必须在 `start()` 兑现之前把子发布出去，`SubagentRun.id` 就是子会话 id；被拒绝的 start 清理未发布的资源、不发生命周期事件，所以账上永远不会出现半个发布出去的子。远程 run 返回 `localAgent: undefined`，也不进持久枚举。

取消信号的归属在两个阶段都是请求里的 signal：发布前取消会清理部分资源，发布后取消剩余的 turn 工作。一次式的取消从头到尾是一条通道。

把一次式放在真实的 turn 里走一遍。模型决定委派，工具层从模型输入构造请求：prompt 和 parent 是必填，parent 顺手提供了子的 cwd、lineage 和派生深度；模型还要求了结构化输出。服务先查能力表，provider 的 `outputSchema` 是 true，放行。run 启动，`subagent/start` 落账，子 agent 在自己的扁平 scope 里干活。结果回来，`stopReason` 是 `max-tokens`：output 里是截断的半份分析，`structured` 缺席。消费者把这条映射成 isError 工具结果，附上剥过敏感内容的 diagnostic，dispose run 达到静默，`subagent/end` 配对闭合。模型看到失败，决定换个更窄的 prompt 重试。整个循环里没有任何持久状态被制造出来，也就没有任何东西需要清理。

## 可继续：持久会话加 Activation

可继续子 agent 是一个持久子 Session，带至多一个进程内的 Activation，也就是重建出来的子 Agent 驻留的一个时期。结构上：

```text
persisted Session
  -> optional live Activation
       -> one retained AgentHandle
       -> Agent inbox as the only turn FIFO
       -> zero or more owned child Activations
```

Activation 明确不是请求、不是结果、不是取消、也不是 Task。它可以执行很多 FIFO turn，在它创建的后代还在跑的整个时期保持驻留。一个 Session 至多一个活 Activation，所以子会话 id 本身就能认出活子。历史会话在 Activation 销毁后不占任何运行时内存：身份在 Session 里，驻留才在 Activation 里。

三个状态不是维护出来的，是推导出来的。running 意味着 Agent 有活跃的准入、turn、或正在唤醒的 inbox 工作；waiting 意味着静默但还拥有至少一个未完成销毁的子 Activation；settled 意味着静默且所有子都销毁了，此时 manager 销毁 AgentHandle、移除 Activation。推导的原料是 Agent 静默状态加拥有的子集合，没有第二台执行状态机。这条纪律的分量在于：状态机是 bug 温床，两台状态机保持同步是 bug 灾区，能推导就不要存。

`startContinuable` 在 inbox 接受、拿到消息 id 时就 resolve，返回 `{ childId, messageId }`，不等 turn 开始、不等消息进日志。更早的任何失败触发回滚：销毁已建的句柄、移除 Activation、撤销父的归属，然后才拒绝，不留未配对的终态边。

## followup：路由只看 Activation 驻留

`followup` 是唯一的续消息操作，路由只取决于 Activation 状态：

| Activation 状态 | followup 行为 |
|---|---|
| running | 在同一个 Activation 入队 |
| waiting | 唤醒同一个 Activation |
| 没有 Activation | 冷恢复一个新 Activation |

每条续消息变成一个 `Agent.followup()` FIFO turn。被接受的消息有唯一可观察顺序，一条 follow-up 没法重定向一个已经在进行的 turn。

为什么死磕"唯一队列"？设计笔记把替代方案逐个否了。每个 turn 一个 Task？用 Task 做对话投递会造出第二个队列、复制 turn 所有权。在 inbox 旁边放一个 manager FIFO？它和 Agent 已接受的消息之间的顺序是模糊的，而基于 Job 的取消没法精确移除一条排着的请求。两个 FIFO 没有唯一的排序权威，对话顺序就成了运气。一条 inbox、一个顺序，问题从根上没了。

这条性质往下还有一层红利。既然每条续消息就是一个普通的 FIFO turn，那么会话日志里子 agent 的对话记录和任何普通会话长得一模一样：user 消息、assistant 回复、工具调用，按同一次序排布。审计一个多智能体会话不需要懂任何特殊协议，读日志就行；重放它也不需要，因为 turn 的排序权威只有一个，日志里的顺序就是执行过的顺序。

## 授权：活的直接父，不是消息字段

follow-up 的授权来自一个确切的 live Agent 工具上下文。被认证的 Agent 必须是这个持久子 agent 的直接父，记录在 `SessionHeader.parentSession`。

`MessageSource` 和 `senderSessionId` 记录谁提供了消息，但不授予任何权威。模型面向的工具用 `CoordinatorMessageSource`，kind 是 `coordinator`、form 是 `relay`，名字就在说明自己只是中继。持久谱系单独也证明不了什么：日志里记着的父，不代表现在还拥有这个子。检查发生在两个关口，最终的 no-await 准入边界，和冷恢复重建之前。祖先、host、团队一律拒绝，直到出现具体的消费者再说。

这条姿态在 `dsh` 里反复出现：jobs 靠 owner 不靠 id 保密，approval 的监听器按 agent 作用域隔离。共同点都是权威绑定在活的对象上，不绑定在可以被伪造、被复读的数据上。

## 信号只在 inbox 接受前拥有

对 `startContinuable` 和 `followup`，调用方的 signal 只在查找、物化、准入阶段有效，到 inbox 接受为止。之后 manager 独立拥有 Activation：后来的调用方取消既不取消已接受的 turn，也不销毁子 agent。接缝不暴露任何 steering 操作。

为什么不做 steering？设计笔记的理由是它需要当前 turn 的控制器状态和一套独立的准入策略，把每条续消息排队就完全避开了这份状态和它的准入竞态。严格的 live-only steering 留给了未来的 host UI，接缝里不做。

还有一个精致的性质：投递和最终销毁竞争时，准入截断恰好一边赢。要么 inbox 接受了这条消息，要么这条消息留给之后的冷恢复。没有中间态，没有"算发出去了又算没发出去"。

## interrupt：唯一的公共停止

`interrupt(targetSessionId, authority)` 是唯一的公共停止。它同步授权，对活目标发 `Agent.cancel(cause, { keepInbox: true })`，不等静默就返回。Activation、未认领的 pending inbox 工作、已发布的后代都不动；已经被认领进被中断 turn 的工作不重新入队。被中断的 driver 空闲后，一条唤醒的 send 会恢复停着的 FIFO 队列。

授权两种：`user` 是人客户端呈现的持久直接父地址，`ancestor` 是一个确切的 live Agent 对象，其记录的 lineage 必须包含调用方。父地址不匹配、或调用方在活谱系之外，`UNAUTHORIZED` 拒绝。过期的祖先对象和自指向在查找之前就拒；目标不存在、或组合里没有 manager，是接受的 no-op。这个区分有意义：前者是安全拒绝，后者是无事可做，混在一起会把"没权限"和"没对象"搅成一笔糊涂账。

## report 与 settled：子给父的两类消息

`reportFrom` 让一个活的可继续子 agent 把选定内容发给它的持久直接父。权威凭证就是子 agent 自己这个活对象，调用方不能指定接收者：manager 从子的 `parentSession` 派生唯一接收者，要求那个父 Agent 活着，把内容框成一条 `subagent-report` user 消息，返回稳定的 `MessageId`。

投递方式由部署策略决定。`quiet` 用 `Agent.inject()`，不唤醒父；`next-step` 用 `Agent.steer()`，父空闲就唤醒，父在跑就并入最近的 step 边界。两种都不结束子的 turn，也没有隐式的最终答复。reporting 是子自己的选择，驻留本身不能悄悄送子一条回话通道。

manager 还单独记自己的账。一个驻留的 Activation 结算时，manager 给子的持久直接父发一条通知，描述这个 epoch 怎么结束、带子最后的 assistant 内容。这条投递对所有 id 被调用方拿到过的子无条件发生，时机在会让父被判 settled 的归属释放之前。父如果已经在拆除中，投递不唤醒它：唤醒一个静默的 Agent 会开启一个 turn，而这里要的是排队。通知的来源 kind 是 `subagent-settled`、form 是 `notice`，和 `subagent-report` 刻意分开，transcript 永远不会把 manager 的运行时陈述呈现成子写的话。审计归审计，发言归发言。

report 和 settled 通知为什么是两套机制而不是一套？因为发起方、时机、语义都不同。report 是子在自己 turn 里主动说话，内容、时机由子的模型输出决定；settled 是 manager 在生命周期边界代为陈述，内容是运行时事实。一个走子的叙事通道，一个走 manager 的通知通道，合并就会出现"子还没说话，transcript 里已经有了以子的名义出现的内容"这类张冠李戴。

## provider 的边界：只贡献首次创建

六个 provider 最核心的差异是创建子 agent 的方式，尤其是否继承父对话。`inheritsParentContext` 描述的只是对话 seeding，fork 为 true，spawn 和 acp 为 false；它不暗示继承工具、服务或权威。

`spawn-in-process` 在同进程里起全新子 agent，每个子拿一个新的扁平 scope，不继承父的注册，深度为父深度加一。`fork` 同进程，但通过 `CreateAgentOptions.seed` 把父日志的一个平衡的已完成 turn 前缀喂给子会话：seed 从 seq 0 连续、无损 JSON、可被不变量重放接受。进行中的 turn 被排除，因为它里面的子 agent 调用还没有结果，构不成合法的重放历史。`acp` 走 ACP 协议和外部 agent 进程通信，只读 cwd，且仅当没配部署 cwd 覆盖时。`codex` 和 `claude-code` 是产品级 provider，`dsh-sdk` 走 dsh SDK。

provider 只参与首次创建：一次式的 `start`，或可继续的 `prepareContinuable`。后者返回的 `ContinuableCreateSpec` 只带 detached 的创建输入，当前就是可选的父历史 seed。文档的原话是，它是数据，不是能力：不带 Agent、不带句柄、不带投递、不带结果、不带销毁、不带 resume。可继续子 agent 的冷恢复根本不分发给 provider，manager fold 通用的 descriptor，通过 activation-owner 的 scope 调 `ctx.agents.resume()`，提交等待的 turn。

为什么把 provider 隔离在创建之后？设计笔记反问了一句：给 provider 任何后续的句柄、run 或消息所有权，等于保留 provider 的所有权，却没有任何已交付的行为证明这有必要。fork 和 spawn 的差别只在 seed，让它们做两个 provider 而不是一个请求开关，也是同样的洁癖：每个后端的契约保持干净。

### descriptor：子的持久身份

子 agent 的持久身份是一个 mode 判别的 descriptor。两种 mode 都带 provider 名；一次式可选带 label；可继续要求一段 description，并把 provider、model、可选的 persona 和 toolFilter 快照进去。两个刻意的省略值得单独讲：不带 `subagentDepth`，因为头部的 `delegationDepth` 是单调下限，快照反而给了它降级的机会；不带 `outputSchema`，因为结构化输出是一次式 run 的请求参数，不是持久身份。

descriptor 是纯日志事件，永不进模型历史，压缩也保留它。头部还有一个 `seedLength` 字段标记 fork 谱系边界：这个子从父日志吃了多长的前缀，一查便知。枚举和冷恢复 fold 的就是这个 descriptor，上一节的三级缓存阶梯缓存的是它的 fold 结果。

## codex 与 claude-code：产品级的两条路

这两个 provider 的存在理由是给真实产品开官方通道。需求很具体：交出去一个自包含的任务，在父的工作区里干，返回一个答案或明确的失败、取消，并且不留任何被管理的产品进程在后台。

被拒绝的路线值得看。直接协议调用（裸模型 HTTP、`codex exec`、手写 Claude CLI 协议）绕过了产品的官方扩展面，于是两个 provider 分别走官方集成面：Codex 用 `app-server --stdio`，Claude Code 用 Agent SDK 的 `query()`。共享一个产品进程 helper 包也被拒绝，它只会复制所有权而不消灭任何一个私有的产品适配器。产品可用性、实例配置、认证是部署事实，所以工具绑定走 Profile，不做模型可见的动态选择。

版本是钉死的（写作时 Codex 0.147.0，Agent SDK 0.3.220 对应 Claude Code 2.1.220），产品或端点基线一变就要重新验证。能力边界文档写得直白：不恢复会话、不流式进度、不接受新的人工交互、不回滚工具或文件副作用、不施加墙上时钟超时。无人工值守歧义的请求、任何未知的服务器请求，直接让 run 失败，而不是挂着等一个不存在的 UI。每次委派都付一个新鲜产品进程加独立模型上下文的成本。原始 stderr 永不进诊断，只有匹配过的有界签名进。

## 深度、工具过滤、人设

委派深度复用持久词汇：`SessionHeader.delegationDepth` 加可合并扩展的运行时字段 `AgentOptions.subagentDepth`，缺省表示顶层深度零，较大的存在值权威。进程内子持久化父深度加一，冷恢复不能降低它，start 拒绝落在安全整数域外或超过 `maxDepth` cap 的派生深度。深度只能增不能降，防的是子 agent 把自己降回顶层绕过递归限制。in-process 后端强制深度和工具过滤；ACP 两个能力都广告为关，直接拒这类请求。

工具过滤在创建窗口里做 scoped `tools.restrict()`：被过滤的工具从子的提示里消失、也拒绝执行，一次可见性，两处同步。未知的工具名大声校验失败，不静默忽略。人设注册为 scoped 的 `deployment:persona` 段，只为这个子遮蔽部署的人设，插值用严格的 `{{…}}` 语法。子 agent 看到的世界和它的父刻意不同，这三样就是刻度的旋钮。

## 枚举：不唤醒任何人的读

`listChildren(parentSessionId)` 合并 live 会话存储和可选的持久化（live-preferred），候选项是持久头部带 `origin: 'subagent'` 的直接子。纯读：不加载也不 resume 任何 Agent，不用 query service。这个 origin 标记只做枚举分类和粗粒度的通用路由拒绝，不能建立有效的 descriptor。

每个子的 mode 和 label 来自注册的 `subagent` 投影单元，走三级阶梯：注册表的水位缓存（活子，零日志读）、可选的投影检查点缓存（冷子，own-suffix seq 闸门证明值晚于 fork seed）、否则一次持久化 inspect 经注册表 fold。缓存是纯加速器，缺失或失败静默退回权威的 refold。fold 是 last-wins：子自己的 descriptor 覆盖 fork seed 里祖先的；畸形或未知版本的载荷 fold 成可序列化的 null 哨兵，当无值处理。

结果按 createdAt 再按 id 排序，三种条目：`child`（mode 加活动状态）、`corrupt`（已结算、无可服务身份，缺失、畸形、不支持刻意不区分）、`unavailable`（冷检视失败，下次列再试）。跑着但没有身份的候选被省略。`listDescendants` 加上 `parentId` 和 `depth`，稳定前序遍历，普通会话和一次式子仍是遍历节点，它们下面的可继续后代能被找到。两个前置错误码在读取之前检查：没有投影注册表、没有会话存储。枚举不参考 Activation、Agent 注册表或 provider，`send_message` 在投递时刻仍是权威。

## 崩溃恢复与销毁

崩溃之后只有落了日志的消息活下来。inbox 接受本身不提供重启保证，接受了但没进日志的消息不会被重放；一条之后被授权的 follow-up 仍然能从持久 Session 冷恢复这个子，把对话续上。孤儿 Activation 被结构性预防：父在拥有的子集合非空时不能 settle 或销毁，归属图的边把逃跑路线全堵住了。

把一次崩溃按时间排开。父 agent 起了一个可继续子，子跑完第一轮 turn，静默，进入 waiting（它自己还有个孙在跑）。此刻进程被 kill。重启后：子的 Session 还在，descriptor、对话、深度都在日志里；Activation 没了，它本来就只在进程里。孙的 Activation 同样没了，父的 waiting 状态随之失效，因为那本来就是从活的后代集合推导出来的。用户回来对着子发一条 follow-up：没有活 Activation，路由到冷恢复，manager fold descriptor、经 activation-owner scope 调 `ctx.agents.resume()`、提交等待的 turn，一个新的 Activation 驻留开始。对话从日志停下的地方继续，唯一真正丢掉的是崩溃前接受了但还没落日志的那条消息，而这条丢失是有界的、可重发的。

销毁走 drain。manager 卸载跑一个全 manager 的 drain，关准入、销毁每棵活森林；`drainContinuableDescendants` 限定到确切的活父；`drainContinuableChildren` 释放选定的直接子，父不对或身份过期就 `UNAUTHORIZED`。释放句柄一律子优先，并 await 每个选中的分支，个别失败不中断整体。持久 Session 活过拆除。结算 flush 会等 `ctx.sessions.flush(session)` 但忽略它的参与布尔，理由很硬：因为留着一个子会把它的祖先永久钉在 waiting 里。flush 被拒只记日志，不让 Activation 失败。

## 一个多轮协作的完整样本

把可继续的机制放进一个真实工作流。父 agent 在重构一个模块，fork 出一个子 agent，seed 带父已完成的六个 turn，工具过滤掉文件写入类工具，人设是一段"只做只读分析"的描述。子拿到父的上下文和自己的限制，第一轮 turn 干完，`reportFrom` 用 `next-step` 把结论 steer 给父，父在最近的 step 边界收到一条 `subagent-report`。

父看完报告，用 `tool-subagent-control` 的 `send_message` 追问一个细节。子的 Activation 还在 running，followup 直接入同一个队列，第二轮 turn 开跑。中间父发现方向不对，调 `interrupt_agent`，授权走 `ancestor`：父的 lineage 里确实包含调用方。`Agent.cancel` 带着 `keepInbox: true` 发出去，正在跑的 turn 被打断，排队着的追问原封不动，子空闲后 FIFO 恢复。

追问答完，子没有后代要等，settle。manager 在归属释放之前给父投一条 `subagent-settled` 通知，带 epoch 的结束描述和子最后的 assistant 内容。父的 transcript 里，子自己发的 `subagent-report` 和 manager 代发的 `subagent-settled` 是两种来源，谁说的话记在谁头上。整个过程：一条队列、一个授权对象、一次创建时的 provider 参与，其余全部是标准会话机制在跑。

## 权衡：这套接缝的成本和回报

成本主要压在可继续那一半。Activation 驻留、冷恢复、归属图、子优先销毁，这套机制比一次式 run 重得多；要的只是简单委派拿结果，一次式 spawn 就够。provider 在首次创建之后被隔离在外，没法在子的后续 turn 里插手，这是生命周期集中到 manager 换来的代价。授权只认活的直接父，安全，但祖父得通过父中转。深度只能增不能降，防绕过，代价是一个设得太高的子没法事后收紧。codex 和 claude-code 把工作交给不受 dsh 控制的产品，能力描述基于 dsh 侧的契约，外部实际行为以产品为准。

回报是同一个接缝覆盖了完整的委派谱系。同进程一次式、继承父历史的 fork、跨协议的 ACP、跨产品的 codex 和 claude-code、dsh SDK，全部用同一套请求、结果、授权、枚举词汇。可继续那一半虽然重，但多轮对话、冷恢复、上报全落在标准机制上：队列就是 Agent inbox，恢复就是 `ctx.agents.resume()`，消息就是普通的 user message。设计一个和普通会话行为一致的子 agent，比设计一套特殊的"多智能体协议"便宜，也好懂得多。

选型上可以给几条直接的判断。并行拆解一批互相独立的小任务，spawn 一次式最合算：起得快、拿结果、不留状态。子必须理解父已经聊过的上下文（比如接着父的分析继续深挖），fork 是唯一选择，代价是父历史在子的上下文里再付一遍 token。需要进程级隔离、或对面本来就是一个独立的 agent 进程，走 ACP。任务的专长恰好落在某个产品的能力圈里（Codex 或 Claude Code 各自的强项），用对应的产品级 provider，同时接受它的边界：每次一个新鲜进程、没人机交互、不能续会话。判断的主轴就一条：需要的是结果、是上下文、是隔离，还是是产品能力；四者分别指向不同的 provider，混着需要时拆成多次委派比找一个全能后端更可靠。

## 结论

`ctx.subagents` 是多 provider 共存的命名注册表，委派分一次式和可继续两类。一次式轻：一个 run、一个结果、失败 resolve 不 reject、dispose 幂等。可继续重：持久 Session 定身份，进程内 Activation 定驻留，状态从静默和子集合推导而非存储。可继续那一半的三条硬规矩是全文的落点：Agent inbox 唯一队列，让对话顺序有唯一权威；授权认活的直接父对象，让伪造消息字段拿不到权；provider 只贡献首次创建的数据，让整个生命周期归 manager，冷恢复绕过一切 provider。委派谱系从进程内一路铺到外部产品，词汇始终是同一套。

## 延伸阅读

- [Subagent 官方文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/subagent.md)：本文主要依据，含一次式与可继续的全部契约
- [Subagent capability seam 笔记](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-06-21-subagent-capability-seam.md)：接缝设计理由
- [Continuable subagent conversations 笔记](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-07-28-continuable-subagent-conversations.md)：可继续子 agent
- [Codex and Claude Code subagent backends 笔记](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/feature/2026-08-04-claude-code-and-codex-subagent-backends.md)：产品级 provider 理由
- [Scoped Agent Registration](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/12-scoped-agent-registration.md)：scoped 注册与 lineage

上一篇：[Plan Mode 与 Goal：dsh 怎么管理目标和计划](./29-plan-mode-and-goal.md)
下一篇：[web-schedule：dsh 会话内的定时、提醒与自动化](./31-web-schedule-timer-automation.md)
