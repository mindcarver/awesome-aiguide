---
article: 29-plan-mode-and-goal.md
type: mixed
density: rich
style: notion
palette: default
image_count: 20
---

# Plan Mode 与 Goal 配图大纲

## Illustration 1
**Position**: 导言之后
**Purpose**: 先区分轻量引导与持久目标。
**Visual Content**: Plan Mode 与 Goal 的双栏重量对比。
**Text Plan**: 标题「目标管理不是一个机制」；标签「软引导」「持久生命周期」「零主干依赖」；提示「先分清引导模型和追踪目标」。
**Filename**: 29-01-two-weights.webp

## Illustration 2
**Position**: 引导不冒充强制
**Purpose**: 说明 Plan Mode 不替代沙箱和审批。
**Visual Content**: 软引导与硬边界的分层。
**Text Plan**: 标题「引导不冒充强制」；标签「Plan Mode」「Sandbox」「Approval」；提示「安全边界不读也不写 plan 状态」。
**Filename**: 29-02-guidance-not-enforcement.webp

## Illustration 3
**Position**: Plan Mode 开头
**Purpose**: 展示四个插件贡献。
**Visual Content**: Plan Mode 四件套环形结构。
**Text Plan**: 标题「Plan Mode 的四样东西」；标签「plan:policy」「exit_plan_mode」「/plan」「plan 投影」；提示「一段提示加三条接缝」。
**Filename**: 29-03-plan-mode-four-parts.webp

## Illustration 4
**Position**: 状态是纯 fold
**Purpose**: 解释日志唯一真相。
**Visual Content**: plan/mode 事件折叠为 active。
**Text Plan**: 标题「状态是纯 fold，没有镜像」；标签「plan/mode」「foldPlanMode」「active」「日志」；提示「恢复、fork、压缩都从日志重建」。
**Filename**: 29-04-plan-pure-fold.webp

## Illustration 5
**Position**: 投影三元状态
**Purpose**: 解释 active、wanted、running。
**Visual Content**: 三个状态卡和 pending。
**Text Plan**: 标题「客户端看到的是可重放的 pending」；标签「active」「wanted」「running」「stateVersion 2」；提示「开关状态不依赖进程镜像」。
**Filename**: 29-05-replayable-pending.webp

## Illustration 6
**Position**: turn 边界 flush
**Purpose**: 解释 queued 的原子落盘时间线。
**Visual Content**: Step 3、queued、Step 4 pre-step、event、prompt。
**Text Plan**: 标题「turn 边界 flush」；标签「queued」「agent/pre-step」「plan/mode」「请求派生」；提示「状态先入日志，再影响请求」。
**Filename**: 29-06-turn-boundary-flush.webp

## Illustration 7
**Position**: set 的四种返回值
**Purpose**: 总结控制器时机。
**Visual Content**: 四个结果卡。
**Text Plan**: 标题「set 的四条出路」；标签「committed」「queued」「cancelled」「noop」；提示「pending 只在原子边界提交」。
**Filename**: 29-07-set-outcomes.webp

## Illustration 8
**Position**: exit_plan_mode
**Purpose**: 说明计划审查和受审退出。
**Visual Content**: markdown plan 到 Review 再到 pending exit。
**Text Plan**: 标题「exit_plan_mode 是受审退出」；标签「Markdown 计划」「Approve」「Keep planning」「pending exit」；提示「批准不立刻撕裂当前工具批」。
**Filename**: 29-08-reviewed-exit.webp

## Illustration 9
**Position**: /plan 命令
**Purpose**: 展示文字、图片和 off 的语义。
**Visual Content**: 三条命令输入路径。
**Text Plan**: 标题「/plan 现在也吃图」；标签「/plan」「/plan <message>」「image-only」「/plan off」；提示「off 不接收图片附件」。
**Filename**: 29-09-plan-command-images.webp

## Illustration 10
**Position**: Goal 开头
**Purpose**: 展示四件套组合。
**Visual Content**: goal、tool、command、driver 四块堆栈。
**Text Plan**: 标题「Goal 是四件套目标栈」；标签「goal」「tool-goal」「command-goal」「goal-round-driver」；提示「每层可拔，agent loop 零依赖」。
**Filename**: 29-10-goal-four-piece-stack.webp

## Illustration 11
**Position**: 会话日志就是数据库
**Purpose**: 解释 full snapshot 和 strict fold。
**Visual Content**: goal/change 日志到严格验证再到视图。
**Text Plan**: 标题「会话日志就是目标数据库」；标签「goal/change」「完整快照」「strict fold」「clear 墓碑」；提示「坏记录停在入口，不被静默修复」。
**Filename**: 29-11-goal-log-database.webp

## Illustration 12
**Position**: 修订号
**Purpose**: 展示 compare-and-set 的拒绝路径。
**Visual Content**: 两个写手竞争同一 revision。
**Text Plan**: 标题「改目标先对表」；标签「GoalRef」「revision 7」「CAS」「stale 拒绝」；提示「先读、再改、冲突后重试」。
**Filename**: 29-12-cas-revision.webp

## Illustration 13
**Position**: 两层答案
**Purpose**: 分开持久阶段和进程 activation。
**Visual Content**: persistent phase 与 ephemeral activation 对照。
**Text Plan**: 标题「目标怎么了，不等于现在能续跑」；标签「active / paused / blocked / complete」「armed」「disarmed」「session-start」；提示「重启默认 disarm，不会意外自启」。
**Filename**: 29-13-phase-and-activation.webp

## Illustration 14
**Position**: 轮次预算
**Purpose**: 区分人类 turn 与 goal round。
**Visual Content**: 两类消息进入预算计数器。
**Text Plan**: 标题「轮次预算只数 goal 的账」；标签「human turn」「goal round」「roundsStarted」「maxGoalRounds 256」；提示「人类澄清不烧预算」。
**Filename**: 29-14-goal-round-budget.webp

## Illustration 15
**Position**: /goal 命令
**Purpose**: 呈现人类可控入口。
**Visual Content**: 读取、创建、编辑、暂停、恢复、清除。
**Text Plan**: 标题「/goal 是人的入口」；标签「create」「edit」「pause」「resume」「clear」；提示「不静默清掉再建目标」。
**Filename**: 29-15-goal-command-entry.webp

## Illustration 16
**Position**: 模型工具与权限
**Purpose**: 说明三个工具、两把钥匙。
**Visual Content**: 工具卡通向两种授权钥匙。
**Text Plan**: 标题「三个工具和两把钥匙」；标签「get_goal」「create_goal」「update_goal」「直接人类授权」「当前 goal 轮次」；提示「自主轮次只能报告完成或阻塞」。
**Filename**: 29-16-goal-tools-two-keys.webp

## Illustration 17
**Position**: 续跑驱动器
**Purpose**: 呈现 Goal 到 Step 层级。
**Visual Content**: 四层嵌套层级图。
**Text Plan**: 标题「Goal Round 不是普通 Turn」；标签「Goal」「Goal Round」「Turn」「Step」；提示「只有归属 goal 的 turn 推进轮次」。
**Filename**: 29-17-goal-round-hierarchy.webp

## Illustration 18
**Position**: 预留、接纳、结算
**Purpose**: 解释调度准入与失败映射。
**Visual Content**: reserve、pre-step gate、settle 三段流程。
**Text Plan**: 标题「续跑：预留、接纳、结算」；标签「reserve」「agent/pre-step」「accept」「blocked」；提示「异常不自动重试」。
**Filename**: 29-18-driver-reserve-accept-settle.webp

## Illustration 19
**Position**: 崩溃恢复
**Purpose**: 展示哪些状态跨重启。
**Visual Content**: crash、replay、disarmed、resume 的时间线。
**Text Plan**: 标题「崩溃后恢复，工作不会自启」；标签「revision 12」「round 37/256」「日志重放」「disarmed」「/goal resume」；提示「日志恢复进展，人类重新授权续跑」。
**Filename**: 29-19-crash-recovery.webp

## Illustration 20
**Position**: 结论之前
**Purpose**: 总结 Plan 与 Goal 的协作序列。
**Visual Content**: plan review 到 goal execution 到 blocked/resume。
**Text Plan**: 标题「先规划，再持续执行」；标签「/plan」「审查退出」「/goal」「round-limit」「resume」；提示「引导、追踪、强制必须分离」。
**Filename**: 29-20-plan-goal-workflow.webp
