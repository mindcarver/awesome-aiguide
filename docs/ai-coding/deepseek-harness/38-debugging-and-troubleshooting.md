# dsh 的排查与调试：全插件化 harness 怎么追问题

> dsh 追问题按事实的获取成本分三层：加载时用 `--dump-config` 打印层叠后的插件树，每一行带来源注释；运行时用会话日志这本事件溯源账本重建 agent 看到和做过的一切，用不变量注册表守住事件与数据的契约；开发时用 typecheck、type-equiv、hygiene 三道门禁挡住类型与文档的漂移。
> 这套体系的底气来自两条设计前提：没有特权核心，问题不在你猜不到的内核里；模型可见即可重建，运行时的事实都躺在可打印、可重放的日志里。多数问题不用加一行 print 就能定位。

## 全插件化系统的排查难在哪

单体软件出问题，怀疑对象是那一个进程；微服务出问题，怀疑对象是某台服务。全插件化的 harness 出问题，怀疑对象是"组合"：同样是官方发布的一堆插件，你的 patch 层叠方式、你的 profile、你加载顺序里的某个第三方插件，任何一个都能让行为偏离预期。问题很少藏在你正盯着的那行插件代码里，更多藏在它和邻居的接缝上。

这决定了排查的第一条纪律：先确认事实，再读代码。事实分三类，对应三个时间点。加载时的事实是"这棵插件树到底长什么样"，哪些行在场、每行 config 是谁定的。运行时的事实是"agent 实际看到了什么、做了什么"，每次模型请求的完整构成、每次工具调用的结果。开发时的事实是"源码和它的声明、文档、构建产物是否还一致"。dsh 给三类事实各配了工具，这一篇按这个顺序讲，最后把工具串成两条完整的故障推演。

顺序本身有讲究。加载时的事实最便宜，一条命令、不起进程；运行时的事实要翻账本，但什么都在；开发时的事实最贵，要跑构建和门禁。从便宜往贵查，是省自己时间的路。反过来走，容易在没有事实的情况下猜，猜错了再回头查，一轮就是半小时。

## 第一步永远是 dump-config

出问题的第一个怀疑对象是"你的插件树组合和你以为的不一样"。这不用猜，跑 `dsh --profile web --dump-config`，它组合整棵树但不起进程，把叠完的结果打印出来。相关 flag 两个，分工写在 `apps/cli/src/args.ts` 里：`--dump-default-config` 只打 bundle 层，回答"出厂长什么样"；`--dump-config` 再叠 profile 自己的 patch、home patch 和命令行 `--patch` overlay，回答"我的环境里最终长什么样"。两个 flag 互斥，dump 不接受 app 参数，`--dump-default-config` 也不接受 `--patch`，传了直接报错。两个输出的差，就是这台机器上全部的本地定制。

输出不是裸 YAML，带来源注释。`renderConfigDump`（`packages/boot/app-boot/src/index.ts`）的实现方式是对 base 加前 k 层逐个做快照，再位置对比，同一文件贡献的连续行块前面打一行 `# ==` 注释，写明这些行来自哪个文件、被哪些 patch 层改过。输出仍是可加载的 YAML 文档，`!!js` 表达式按原文打印、不求值。这是排查的证据链，三个典型场景都能从这一份输出里读出答案。

插件没生效。先在 dump 里找它的 id。找不到，是 patch 没写对，id 拼错、insert 没被加载、patch 文件路径不对；找得到但行为不对，看这行的注释，你的 patch 文件名挂在上面就是生效了，没挂就是被更高层盖掉了。patch 指向不存在的 id 时，dump 会在 stderr 报出未匹配目标并带上层标签，这是最便宜的拼写检查，甚至不用刻意去找，看一眼 stderr 有没有报告就行。

config 值不对。dump 显示的是层叠后的最终值，想追哪层改的，顺着注释里列出的文件从低到高看：bundle 的 patch、profile 的 patch、home 的 patch、命令行 overlay。同一个 id 被多层覆盖时，赢的是最后碰它的那层。

字段没出现在 dump 里。不一定是丢了，可能是它走 schema 默认值，运行时才填入。判断依据是 config catalog 里这个插件的字段声明，有默认值的字段不显式设置就不出现。少字段先想默认值，再想丢失。

dump-config 是"没有特权核心"在排查上的兑现：打印出来的就是全部，你不用怀疑内核私藏了一个你不知道的配置层。

## 会话日志：飞行记录仪

加载对了，问题还在，就要看运行时。dsh 的会话日志是一个事件溯源的账本：原始对话从不原地修改，LLM 消息历史、UI 对话、telemetry，全部是从已提交日志投影出来的视图。这条设计有一条运行时不变量盯着（`packages/core/agent-loop/src/invariant.ts`）：loop 构建的请求若在日志里没有对应的 `request/header` 事件，直接失败。对排查来说这句话可以翻译成：agent 看到的、做过的一切，都在日志里，一个不缺。

翻这本账本时，几类事件各答一个问题。

`request/header` 回答"这次请求到底发了什么"。它记录请求的完整构成：provider、model、路由配置，追加时机分 initial、resume、change 三种。模型行为不对时，把 request/header 和你的预期并排对比，是最高效的一步，因为模型的行为完全由它收到的请求决定，请求里没有的信息模型不可能知道。工具没被调用，先看工具 schema 在不在请求头里，不在就是工具面的问题，在就是模型选择的问题，两个方向的排查路径完全不同。举个对比的用法：agent 一直用 bash 硬啃一个本该走 LSP 跳定义的任务，把请求头里的工具清单打出来，发现 LSP 工具的 schema 根本不在这次请求里，问题瞬间从"模型不会用"变成"工具为什么没注册上"，后者用 dump-config 一查就是一个 disabled 行或者一个没启用的 tool 行。请求头对比的产出经常就是这样一次病灶搬家：把对模型的怀疑换成对一个配置事实的怀疑。

`assistant/chunk` 回答"流到了哪"。有部分 chunk 然后中断，是流中途断了；一个 chunk 都没有，是请求从没开始或立刻失败。这一类读法把"没响应"这个模糊症状拆成了两个可区分的故障。

`tool/result` 回答"工具实际返回了什么"。每个工具调用的结果都带 `isError` 标志落在日志里。agent 做了奇怪的决定，追它的工具调用历史通常能找到原因，最常见的模式是某个工具静默失败了，agent 拿着一份错误结果继续推理，行为看起来荒谬，输入其实已经被污染。

`turn/end` 的 reason 回答"这轮为什么结束"。reason 的 `kind` 为 error 时携带错误消息与失败分类，是定位请求失败的入口；kind 为 aborted 时携带取消原因。

`turn/start`、`step/start`、`step/end`、`todo/write` 这些是 trace 类数据，不影响模型可见的内容，但能还原 agent 的行动序列。怀疑 agent 在无意义循环时，看 step 序列的时间戳和内容，循环会在序列里直接显形。

账本还有两个排查友好的性质。它可以 fork：出问题的会话，复制一个副本在副本上实验，原始会话不动。它可以 resume：想重现一个问题时，从出问题前的位置恢复，用同样的上下文重放。重现难度是排障成本的大头，账本把"当时 agent 脑子里有什么"变成了可重放的事实。

## 运行时不变量：机器自带的报警器

会话日志是被动的事实，不变量是主动的守卫。`ctx.invariants` 是一个可配置的注册表服务，官方子系统文档对它的定位说得直白：support 组的包，不是三角色的能力接缝，也不在 agent loop 主干上。它在运行时检查各包自己声明的契约，出问题时不等你去翻日志，直接抛错。

机制是这样。每个 workspace 包发布一个 `./invariant` companion 插件，用自己精确的 npm 包名注册检查。检查的内容限定在该包拥有的可观测事件或可变数据关系上；一个包确实没有可检查的关系时，导出一个空 installer，文件开头用 `No runtime invariant:` 注释解释为什么，解释必须针对这个包，套话过不了机械校验。检查发现违规，调用 `fail(message)`，抛出 `InvariantError`：错误带稳定的 `code: 'INVARIANT'` 和归属的 `packageName`，消息前缀是 `invariant violated by "<package>":` 加具体内容。违规天然可归因，而且注册表不需要 import 任何产品包，诊断服务对被诊断对象零依赖。

有一条职责边界刻得很死：不变量只能断言权威事件流或可变数据，不能断言 service 或方法是否存在。service 和方法的存在归类型系统管，编译期回答的问题不该在运行时再问一遍。官方文档同样写明这些断言刻意不做合成（deliberately not synthetic），不变量守的是包作者自己声明的关系，不是某种自动推导的全局一致性。覆盖面靠两条兜着：每个包要么有有意义的检查要么有解释过的空缺，`pnpm run verify-package-invariants` 做机械校验，拒绝"生成文件"标记、未解释的空 installer、忽略 reporter 的非空 installer、错误的注册名、不完整的接线。

哪些包的检查生效，由配置决定。`enabled` 是全局开关，默认开。`package_allowlist` 和 `package_blocklist` 都是正则源，前者匹配包名放行（空表示全部允许），后者在 allowlist 之后应用，blocklist 匹配压过 allowlist 匹配。正则用 `new RegExp(source)` 编译，匹配是 unanchored 的，除非源里自己写了 `^` 和 `$`；`/pattern/flags` 这种字面量形式不解析。无效的、空白的、重复的条目在服务启动时抛异常，不会被静默跳过。

这里有一个排查时值得知道的解耦：名字预约和过滤是两回事。一个包注册时名字就被预约，哪怕过滤配置让它的 installer 处于不活跃状态，预约依然持有，两个插件永远无法认领同一个名字。过滤器在服务生命周期内固定，一条合法正则可以匹配不到任何当前加载的包，这换来的是后续加载和 HMR 的确定性。推论是：改过滤配置不是热生效的运行时旋钮，要让新的 allowlist 或 blocklist 生效，得让 invariants 服务走一次自己的生命周期。installer 本身失败时，子 fiber 销毁、名字预约原子性释放，companion 可以重载后用同一个名字重新注册，不残留状态。

unanchored 匹配配得上一个具体警告。allowlist 里写 `dsh-session`，匹配到的不只是官方的 `@deepseek-ai/dsh-session`，任何名字里含这串字符的包都会被放行，比如你自己的 `my-dsh-session-wrapper`。想精确匹配一个包名，锚点要自己写全：`^@deepseek-ai/dsh-session$`。反过来利用也可以，写 `^@deepseek-ai/` 就选中全部官方包。过滤配置的排查症状很有辨识度：某个包的不变量"该响没响"，先看它是不是被一条想当然的 substring 正则意外挡在外面，再看全局开关。

一个最小配置长这样（正则源按 JavaScript 规则编译）：

```yaml
- id: invariants
  config:
    enabled: true
    package_allowlist:
      - '^@deepseek-ai/'
    package_blocklist:
      - '-experimental$'
```

排查遇到 `InvariantError` 时读法固定：先看 packageName 定位到包，再看消息里被违反的契约。两种可能，要么你的组合方式让这个包的契约不成立（比如没加载它 Requires 的 service provider），要么这个包的实现有 bug。前者的修法在配置树里，后者的修法在源码里，错误消息会把你放到正确的那一侧。

## 模型请求失败：request-error waterfall

模型请求会失败，网络断、超时、响应畸形。agent loop 对失败的处理不走硬编码的 if-else，而是派发一个 `agent/request-error` waterfall：请求以 error 或 aborted 收场时，监听器被依次叫到，一个监听器可以返回 `{ kind: 'retry' }`，让 loop 在同一份持久历史上开启新的编号轮次重试；没有任何监听器认领的失败是终态。

重试策略是扩展点不是内置逻辑。默认策略由 `dsh-llm-retry` 提供，认五个稳定错误码：`EMPTY_RESPONSE`、`RATE_LIMIT`、`SERVER`、`TIMEOUT`、`TRANSPORT`，默认重试五次，有界指数退避从 500 毫秒到 10 秒，带 10% jitter。每个 provider 适配器可以带自己的 `retryPolicy`（normal 可调预算、合格码与退避；always 无上限），策略在路由注册时捕获，换路由不影响进行中的失败。你可以挂自己的监听器改策略，换退避曲线、失败时切 provider、把失败记进告警系统。对排查更重要的两点：一是策略只认错误码不读报错原文，所以失败归类的正确性取决于适配器有没有把错误归一到正确的码，怀疑重试行为不对时先查错误码归类，再查策略本身；二是一次适配器调用等于一次 provider 尝试，适配器自己包的 HTTP 库要关掉内部重试，两层重试叠加会让重试次数和退避节奏都不可预期。每次调度重试前，插件会追加一条不进表层的 `llm/retry` 事件（带共享 retryId、provider、策略 key、失败与延迟），日志里能直接数出重试链。

排查请求失败的日志路径是三站：`request/header` 确认请求发出去了、配置如预期；`assistant/chunk` 看流走到了哪，是断在半路还是压根没开始；`turn/end` 的 reason 拿到错误归类。三站走完，一次失败的时间点和类型就定位了。配合 defensive patterns 里的一条设计读日志更准：运行时把模型请求的失败归一成终态的 finish chunk，`LlmRuntime.stream()` 只通过终止型 finish 分片暴露模型请求失败，消费者不用猜一个异常是 provider 抛的、包装库抛的还是拼装层抛的，日志里看到的终态就是归一后的结果，归一前的分层细节在适配器那一侧才有意义。

## Telemetry：把事件流导出来看

会话日志在进程内。要把 agent 运行状态导到外部系统做分析，用 session telemetry 子系统。它有 FULL、FEEDBACK_ONLY、DISABLED 三种模式，默认 DISABLED（2026-08-10 起改为默认关闭，并按会话逐个征得同意），排查时关心的是 FULL：ledger 类记录镜像会话事件，经过 chunk 投影和 `session-telemetry/record` 脱敏 waterfall 后交给后端，后端接原样配置的 OpenTelemetry JS SDK，再往 Grafana、Jaeger 这类工具送。chunk 投影的含义要知道：每个 `(turn, step)` 只发第一条 `assistant/chunk` 当"流已开始"信号，其余分片在捕获时丢弃，所以导出流里的 seq 有缺口是常态，不是数据丢失。

脱敏 waterfall 在导出前把敏感值拦下，密钥类内容不会流进 exporter，这意味着导出数据可以放心交给外部观测栈，同时意味着涉及凭证内容的排查只能回头看进程内的会话日志，导出流里那段是脱敏后的。投递是尽力而为：游标标记"已交接"而非"已送达"，记录可能丢也可能重复，接收端按 `(session.id, event.seq)` 去重。默认关闭意味着排障数据要事前布网，出问题再想起它，那段数据已经不在了。这是它和会话日志最大的实用差异：后者默认就在记。

导出后有两个筛选习惯很有用。severity 是捕获时预映射的：error 级对应工具结果的 `isError`、turn 结束的错误原因、`agent-error` 操作记录，先只看 error 级能最快圈出失败点。`agent-error` 和 `shutdown` 是仅有的两个 ops 通道信号，它们没有日志归宿，也刻意不带 seq 式身份：一个会话的导出流里没有 shutdown 标记，这个会话可能是崩掉的而不是正常结束的，拿这个特征扫一段时间的导出数据，能捞出所有非正常退出的会话。

## 开发时的门禁

改 dsh 源码或写插件时，另一类问题（类型漂移、文档漂移、包质量）靠门禁挡在 push 之前。

typecheck 是第一道。`pnpm run typecheck` 先完成 Host 侧的库阶段，包括 Typert 合约生成，再跑 Client 侧的 TypeScript 检查；pre-push hook 自动跑它。完整的 `pnpm run build` 顺序是 Host 编译、Host 打包、Client 编译、Client 打包、web 构建，Typert 在 Host 打包那一步以 `tsconfig.host.json` 为种子运行。一个容易踩的坑写在开发文档里：写全仓库的 ts.Program 脚本时，必须播种 Host 或 Client 的聚合配置，绝不能用根 solution，两边的 cordis `Context` 声明合并会相撞，报出来的错和你的脚本逻辑无关，看起来像类型系统坏了。

type-equiv 防"文档里贴的类型和源码不一致"。文档里粘贴源码声明的围栏写 `ts type-equiv` 而不是普通 `ts`，并在 `scripts/type-equiv.manifest.json` 里登记文档、符号、源文件三者的对应。`pnpm run verify-type-equiv` 用 TypeScript parser 从源码提取该符号的声明和 JSDoc，断言文档里的粘贴一致（比对忽略空白和非 JSDoc 注释，但要求保留每条原始 JSDoc）。类可以用 `public-api` 投影，省略实现体和私有成员但保留全部 JSDoc。中文文档的 `.zh.md` 兄弟文件复用条目有前提：整段被追踪的围栏序列逐字节一致且顺序一致。改了源码没改文档，或增删了围栏没改 manifest，门禁都会失败，逼迫同一次提交里把两边对齐。排查时它的价值是信任：文档里的类型就是源码里的类型，1:1，不会领着你走弯路。

hygiene 检查包质量，`pnpm run hygiene` 包含 publint（用构建出的 `lib/*.js` 校验包入口）和 verify-node-next-types（用临时 NodeNext 消费方校验构建后的声明）。这些检查消费构建产物，所以本地跑出奇怪的报错时先 `pnpm run build` 把 lib 刷新到最新再判断；普通提交和推送不触发它们，除非你的检查链里用到了构建输出。

代码里三个标记按紧急度分级，grep 它们能摸清一片代码的已知坑：FIXME 是应该阻塞新发布的问题，发布不该带着未解决的 FIXME，除非 reviewer 明确同意；TODO 是有资源就该尽快修的；XXX 是某天可能修的，最低优先级，无承诺。

构建这一层还有一个排查点：`pnpm run build` 会把调用方精确的 `DSH_CLIENT_*` 环境记进一份 gitignored 的构建记录，把这些值与 Vite 输出及动态 client bundle 绑定；release 打包和 built web 测试会拒绝缺失记录的产物，也拒绝被后续部分构建改过的产物。症状是发布流程莫名拒绝一个"看起来没问题"的构建，原因多半是中间又跑过一次部分构建把产物弄脏了，重新完整 build 一次让记录和产物重新对齐。

代码防御模式里有几条对读日志直接有用的纪律，值得带进排查直觉。进程结束的几个独立事实（`timedOut`、`signal`、`exitCode`）是各自独立上报的，绝不把一个标志的报告嵌进另一个的分支，不然调用方会把一次被截短的运行读成干净的成功，看 subprocess 类日志时按这个结构读。等待一个永远不可能到来的转换会挂死，所以"没有东西可等"这个分支被显式处理，日志里一个永远不落地的等待通常意味着前置状态没达成，而不是日志系统坏了。异步完成上的竞态会留下静默的坏状态，不完整的拆卸会留孤儿进程，一个抛错的监听器可能饿死排在它后面的监听器，这四类是读 dsh 行为异常时值得默认怀疑的模式。

开发文档里还有几条环境级的排障条目值得记。缓存恢复的安装可能丢 Git hooks，跑 `node scripts/install-lefthook.mjs` 重装，wrapper 拒绝现有 Git 配置或报告过期锁时照它的诊断走，别手改 worktree 元数据，移动过 checkout 之后要重跑。翻译配对的合并冲突用 `pnpm run resolve-translation-pairing-conflicts`，它把所有能安全生成的配对记录 stage 起来，仍有需要手工的冲突时以失败退出；配对合并驱动的运行时不在时，Git 会落一份纯文本结果并打印恢复路径。真实 API 的端到端套件在读不到 `DEEPSEEK_API_KEY` 时自动跳过，所以"测试全绿"可能意味着"真实链路根本没跑"，看套件输出里的跳过计数，别把跳过当通过。

## 一条完整的故障推演

把工具串起来走一个典型故障：你改了自己插件的 config，HMR 之后 agent 的行为开始诡异，某个能力时有时无。

第一步 dump-config。看你的插件行还在不在、注释里挂的是不是你刚编辑的那份 patch、config 是不是你写的值。假设都对，加载时的事实清白，问题在运行时。

第二步翻会话日志。找行为异常的那几个 turn，看 `request/header`：诡异行为发生时的请求里，你的插件贡献的工具 schema 一会儿在一会儿不在。这个模式很有信息量：加载是稳定的，贡献是漂的，指向插件实例在 HMR 中没有正确换血。

第三步对着不变量和源码查。你的插件用 `ctx.effect()` 注册副作用了吗？如果监听是手动挂的、没走 effect，HMR 卸载旧实例时不会自动撤销，新实例又挂了一份，两份监听交替响应，行为自然时有时无。修法是把注册搬进 effect，让可逆副作用这条 Cordis 纪律替你做清理。修完再 HMR 一次，dump 确认树上还是那一行，日志里确认请求头稳定了，收工。

换一个类型完全不同的故障再看一遍顺序的用法：agent 跑着跑着开始频繁报模型请求失败，重试一阵后 turn 以错误收场。

还是先 dump。这次要看的是 llm 路由行：provider 是你想要的那个，模型 id 没被哪层 patch 换掉，`apiKeyEnv` 指向的环境变量名拼写正确。假设都对，加载时清白。然后翻日志走三站。`request/header` 在，说明请求发出去了，配置如预期。`assistant/chunk` 一段都没有，说明不是流断在半路，是请求压根没得到响应。`turn/end` 的 reason 归类是 TRANSPORT。三站读完，故障画像变成一句可行动的话：provider 端点在那段时间不可达，被适配器归一成传输失败码，默认重试五次都没救回来。接下来不用猜了，去查网络和网关在那个时间窗的状态；如果失败归类是上下文超长，方向就完全不同，该去查压缩为什么没触发。同样的"请求失败"症状，错误码这一站把它劈成了不同的后续路径，这就是归一化错误码在排查里的价值。

这两条推演里每个工具都只用了一下，但顺序不可互换：dump 排除加载问题只要十秒，直接跳进日志会多翻几十个事件，跳过日志直接读源码则会在没有事实的情况下猜。排查的效率不在于工具多熟，在于按"加载时、运行时、源码"这个从便宜到贵的顺序最快到达事实。

## 权衡

不变量的覆盖不是系统自动保证的。断言刻意不做合成，守什么契约由包作者声明，机械校验只保证"要么有检查要么有解释"，不保证检查本身有质量。一个关键契约没被声明成不变量，违规就只能靠日志事后追，报警器不会响。

会话日志记的是"到达模型和 harness 的东西"。外部世界的副作用（文件改没改、进程杀没杀）以工具结果的形式间接在账，工具结果说谎或被截断，账本也跟着失真。日志是 agent 视角的飞行记录仪，不是系统级的审计器，后者要靠沙箱、审批那套子系统自己的事件。

telemetry 默认关闭意味着排障数据要事前布网。FULL 模式有成本，常开不现实，于是"出问题的那段没有导出数据"是常态而不是意外。实用的折衷是把开 FULL 的 profile 放在手边，复现类问题在开了导出的环境里重放。就算开了，脱敏 waterfall 也在，凡是被判敏感的值，导出流里查不到原文。

门禁挡的是漂移不是逻辑错误。typecheck、type-equiv、hygiene 保证类型对、文档对、包结构对，一段类型全对、文档同步的逻辑照样可以是错的。门禁的价值是把"错误"的范围收窄到真正的逻辑层，让读代码的时候不用分心怀疑声明和实现已经分叉。代价是维护税：每个类型改动都要同步 manifest 登记的粘贴块，这税是设计者有意收的，宁可同一次提交多改一个文件，也不要留下"文档可能是旧的"这个怀疑成本。

dump-config 有一条能力边界写在它的实现注释里：dump 是 boot-free 的，它不运行 app 的命令行 provider，所以 app 参数会决定的那部分配置不会出现在 dump 里。排查时如果症状依赖某个运行时 flag，dump 给不了答案，要么看 provider 自己的 `--help`，要么直接翻运行时日志。

## 结论

dsh 的排查体系可以压成一张按生命周期排的表。加载时，`--dump-config` 带来源注释打印层叠结果，未匹配的 patch 目标走 stderr，先跑它。运行时，会话日志这本事件溯源账本回答"agent 看到了什么、做了什么"，`request/header`、`assistant/chunk`、`tool/result`、`turn/end` 四类事件各答一个问题；不变量注册表在契约被违反时抛带包名和稳定码的错误，读法是先 packageName 后消息；请求失败看 `agent/request-error` waterfall 的错误码归类，数 `llm/retry` 事件能还原重试链。开发时，typecheck 在 push 前拦类型错，type-equiv 逼文档和源码同进退，hygiene 需要 fresh build。

使用这个体系的顺序纪律比工具本身更重要：从 dump 到日志到源码，从便宜的事实到贵的事实。它的可信度建立在两条前提上：没有特权核心，所以打印出来的就是全部；模型可见即可重建，所以账本里没有缺口。绝大多数"很玄"的 agent 行为问题，沿着这条顺序走到第二站就有了事实基础，第三站往往是收尾而不是开始。

## 延伸阅读

- [运行时不变式子系统文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/invariants.md)：选择逻辑、安装器契约、配套插件约定
- [开发指南](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/development.md)：typecheck 顺序、TODO 标记分级、环境级排障条目
- [防御性模式](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/defensive-patterns.md)：正交结果独立上报等六条从真实缺陷来的规则
- [会话子系统文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session.md)：事件类型与账本契约
- [遥测子系统文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/session-telemetry.md)：记录通道、severity 预映射与尽力而为投递

上一篇：[dsh 的配置实战：用 patch 改行为，用 preset 做分发组合](./37-config-practice-patch-and-preset.md)
下一篇：[dsh 的 Conversation Node：给 Web 写一个自定义渲染节点](./39-write-a-conversation-node.md)
