# dsh 的 Web 客户端：Chat Nodes 与多 agent 协议

> dsh 的 Web 客户端是两半各自完整的 Cordis 插件图：Host 半跑在 Node.js 进程里拥有全部 agent 能力，Client 半跑在浏览器里拥有全部渲染逻辑，中间靠一条 boot 时注入的图（`window.__DSH_BOOT__`）和一个不知道任何 harness 概念的 HTTP carrier 连接。浏览器怎么知道加载哪些插件、代码变更怎么保持一致、两半之间怎么通话，是三个独立的设计问题，各有各的答案：内容寻址的 rev 查询参数、四象限的消息协议、按调用解析的类型安全远程调用。
> 这一篇顺着一次页面加载和一次远程调用走完全程：boot graph 的形状与顺序、Client Modules 的增量扫描、Web Server 的路由纪律、RPC 协议的消息规则、断线之后的折叠重建，最后看 Chat Nodes 和多 agent 场景怎么落在这个架构上。

## 不是后端渲染前端，是两半插件图

传统的 Web 架构里，服务器渲染 HTML，浏览器只是显示。dsh 的 Web 客户端不走这条路：Host 和 Client 是两个独立的插件图，各自有完整的 Cordis 生命周期。

Host 半跑在 Node.js 进程里，是 dsh 的主体：模型适配器、工具、会话、沙箱、审批，全部在服务器或本地进程里激活。Client 半跑在浏览器里，是一组真正的 Cordis 插件：UI 组件、Conversation Node 的定义和渲染器、交互逻辑，在浏览器里加载、激活、销毁。Host 不产出 HTML 模板，Client 不直接调用 agent 能力，中间隔着 API gateway 和消息协议。

为什么拆这么狠？一个工程上的硬约束逼出了这个形状。两半的 TypeScript 各自 declaration-merge 了同一个 Context 接口 key 下的不同 service，如果把两边的源码放进同一个编译单元，类型碰撞会让 `ts.Program` 直接报错。所以仓库维护两个 aggregate（`tsconfig.host.json` 和 `tsconfig.client.json`），两个构建永不进入同一个 program。这不是组织洁癖，是 Context 合并语义的物理边界。

分层再往下看是四层：front 层放协议定义和 fetch 抽象，Node 和浏览器都能 import；assembly 层决定哪些插件挂载、带什么默认值；carrier 层只有 HTTP，对工作区零依赖；最上面是浏览器侧的插件树。方向纪律是单向的：client 包永不 import host runtime，webserver 不依赖 runtime，它拿到的是一个运行时注入的关系而不是包依赖。这个纪律的回报后面会看到：Electron 和 Python SDK 各自换掉运输层，不动上层。

## 浏览器第一眼看到的东西：boot graph

浏览器打开页面，Host 返回的 index.html 的 head 里第一个 script 不是应用代码，是一份清单：`window.__DSH_BOOT__` 指向一个 boot graph。

图的每一行描述一个 client 插件包：包名、bundle 地址（`/plugins/<id>/client.js?rev=<rev>`）、内容哈希 rev，外加两块可选信息。`immediately` 把这行标进第一梯队：module-face boot 期间就预取并执行，但只注册工厂不做别的事。`inject` 是信息性的依赖边，给人看不参与调度。`external` 最有意思，它声明这个包运行时要 require 一批非基线的说明符，也就是其他动态包的导出。

外层的图自己带一个 rev 和一份按序排列的行列表。顺序在这里是语义，不是摆设。浏览器半的模块表是懒执行的 CommonJS 风格，`require` 是同步的：一个包的代码到达并执行时，如果它 require 另一个动态包，那个包的代码必须已经在浏览器里。所以图按模块图顺序排：被 external 需要的包排在需要它的包前面。这个顺序只管代码到达，与 Cordis 的激活顺序无关，激活时机归 fiber 服务管，两件事不混。

页面解析 shell 先于 boot 执行，一个没有有效 manifest 的页面直接不能启动：浏览器侧的 parser 在缺失或畸形的图上大声抛异常，不降级、不空白屏装死。清单里所有插件控制的字符串都过了一遍 `<` 转义，任何包名、任何 URL 都逃不出 script 元素。这份图是 Host 和 Client 之间的单一真相源，两半产出同样形状的行，不存在第二种格式。

## 一致性锚是 rev，不是 HTTP 缓存

bundle 怎么服务才能保证浏览器永远拿到最新代码？直觉答案是 HTTP 缓存加失效策略。dsh 的答案相反：bundle 全部带 `no-cache` 头服务，HTTP 缓存被明确排除在一致性机制之外。

锚是 rev 查询参数。每一行的 rev 是 bundle 的内容哈希，搭在 URL 上；图的 rev 哈希所有组合后的行，任何一行变了，图的 rev 跟着变。内容一变 URL 就变，浏览器不存在"拿着旧地址拿到新内容"的可能。这比缓存协议可靠得多：缓存失效经过多层代理和启发式，是概率性的；内容寻址的 URL 是确定性的。哈希在这里同时承担版本号和身份证两个角色，改动不需要协商失效，因为地址本身就是版本。

bundle 的服务路由有自己的守门规则：只有 GET 和 HEAD 被接受，其他方法返回 405；未知的包 id 或不可读（通常是没构建）的 bundle 返回 404。后面这条值得停一下：一个没构建出来的包，不会以一个"成功的 JavaScript 响应"出现在浏览器面前，坏的东西在门口就报 404，不会变成一行执行到一半才炸的脚本。

## Client Modules：增量扫描与两种失败姿态

`ctx.clientModules` 是 Node 半的登记处，负责把"哪些包是 client 包"变成上面那张图。一个包通过 `package.json` 里声明 `dsh.client`（`platform: 'web'`、可选的 inject 边、可选的 immediately 标记）加入，并在 `exports["./client"]` 导出构建产物。

扫描是按包增量的，仓库里没有全量重扫的代码路径。每次 Cordis 的 `internal/plugin` 事件（fiber 构造或销毁）把那个 fiber 的条目名标记为脏，一个 microtask flush 把脏名字和 live loader 条目对账。解析锚定在配置树的 baseUrl 上，也就是 cordis.yml 所在目录，这个锚没设置时构造直接抛错，不猜。

首次扫描和稳态走的是同一个实现：激活那一趟把所有当前条目种进同一个脏集合同步 flush。但失败姿态刻意相反。激活时，一个畸形声明或缺失 bundle 会和其他坏包聚合成一个 `AggregateError` 整体报出，fiber 失败，boot 的 fail-loud sweep 接管；稳态时，同一个坏包只 warn，不许毒害邻居。同一个扫描器，启动时宁死不屈，运行时得过且过，因为启动时的坏配置值得拦住整个进程，运行时的一个坏包不该拖垮其他所有 UI。

包的元数据按名缓存且永不过期，包括"这不是 client 包"的否定判定。插件集合的变更在重启时生效，这是用重启换确定性。fiber 重启则复用它的行和 rev，一点不动，bundle 内容的变化只有一条路进图：`rebuilt(id)` 重新哈希，只有 rev 真变了才重组图并通知。两条通知路径，逐 bundle 的 `onRebuilt` 和拉模型的 `onGraphChanged`，都隔离监听器异常，一个抛错的订阅者跳不过后面的订阅者，也杀不死触发这次 flush 的一方。

## Web Server：一个不认识 harness 的 HTTP carrier

`ctx.webServer` 是浏览器侧所有 HTTP 的落点，它刻意薄到极点：一个 `node:http` 插件，提供命名路由注册表、index.html 转换回调和恰好一个 fallback 座位。它不知道任何 harness 概念，不在 agent loop 里，也不是能力接缝。`/api` 桥、插件 bundle、HMR 事件流，每个功能路由都由别的插件注册，Web Server 只提供运输。2026-08-04 之后它还多一个 upgrade 注册点，给 WebSocket 下行用，后面细说。

路由模型是一条纪律。路径分 exact 和 prefix 两类，匹配顺序固定：先查 exact 表，再取最长匹配的 prefix，最后落到 fallback。注册顺序不携带任何请求语义，因为命名路由被组合成互不相交的：两个插件声明了重叠的路径模式，抛错，这是组合层的配置错误，不是运行时谁先谁后的运气。这和按注册顺序匹配的常见做法是两种世界观，后者的行为取决于插件加载时序，而时序不是产品语义。

fallback 只有一个座位，第二个注册者抛错。座位被认领之前，未匹配的请求得到 404。出厂的 Web 组合把这个座位给了 SPA 静态文件服务，语义锁死：非 GET/HEAD 返 405，dist root 之外的路径遍历返 403，存在的文件直接服务，缺失或非文件的目标给空的 404，不认识的扩展名按 octet-stream 发。SPA 路由的落点是 index.html 返 200。

index 渲染是两层的。结构化的注入行通过一次 `webserver/index-inject` 事件收集，每次调用都是新行，订阅者读到的是 emit 时刻的活状态；Client Modules 就是在这里回答 boot manifest 行的。原始的 `tapIndex` HTML 到 HTML 变换是逃生舱，按注册顺序应用，给结构化行表达不了的 markup 用。逃生舱排第二位，是姿态也是顺序。

配置小到一眼看完。`host` 只接受两个值：`127.0.0.1`（默认，loopback）和 `0.0.0.0`（刻意的网络暴露）。`port` 传 0 让操作系统分配，运行时能读到实际监听的端口。没有 TLS、没有认证、没有 origin policy，这三样缺失意味着 `0.0.0.0` 绑定是把服务器交给你所在的网络。安全姿态是 loopback 默认加显式豁免，不是内置一套半吊子的鉴权。出错的行为也定了性：一个抛错的 handler（畸形转义、客户端中途断开）被 warn 加 400 回答，headers 已发出的情况直接毁掉 socket，进程永不退出。监听失败（比如端口占用）则拒绝初始化，失败的 fiber 在 boot 时报告。销毁时 `close()` 配上强制关闭所有连接，因为 SSE 型的 handler 自己不会结束，不强关的话拆卸会挂死。

## 两半之间的消息协议：四象限，不是 JSON-RPC

浏览器怎么调 Host 的能力？先把一个常见误会清掉：这条通道不是 JSON-RPC。设计笔记明确否决了复用 JSON-RPC 2.0，理由有两条：错误码在通用协议里会降级，契约会在两个副本之间漂移。

dsh 用的是一个四象限消息协议。两个轴：谁发起（客户端或服务器），消息性质（请求或响应）。组合出四种消息：客户端请求走 `POST /api/<method>`，服务器响应对应它的 HTTP 200 体；服务器请求（session 事件、审批与提问）从下行流推过来，客户端响应回传到 `POST /api/respond`，回填原来的 rpcId。整个消息集是一个四成员的判别联合，`switch (message.type)` 窄化，加一个新成员就是加一个具名分支。

rpcId 是协议的铸造纪律：谁发起谁铸造，响应回显、永不铸造，业务代码任何时候不铸造 id。服务器的请求精确分两类，静态按帧的 method 区分，不设第三种 kind。可应答的帧（审批请求、提问）带稳定的 rpcId，受理时铸造一次、基线回放原样复用；纯推送的帧（session 事件等）每次用新的。这条区分直接决定了断线重放的语义，后面会用到。

错误模型是一张映射表：每个错误码一行，`details` 字段必填，新增一个错误码等于加一行映射加一个 schema 分支，两处不改完编译不过。传输层失败是 carrier 异常，永远不混进业务错误，两套故障走两套处理路径。方法签名是唯一事实源：请求和响应的类型全部从接口方法签名派生，禁止手写一份同名的类型副本，第二份名字就是第二个可以漂移的事实。

取消有明确的位置：普通 unary 调用默认挂一个 30 秒的 `AbortSignal.timeout`，构造参数可调；由用户掌控节奏的操作（选目录、执行命令）不设这个时限，但保留调用方和连接取消，因为给长 handler 上硬传输时限会杀死合法的慢操作。流不设时限。

协议的能力边界也钉死了：Remote 严格是一次请求一次结果的一元调用。会话事件、分页、增量折叠、实体子流，这些流式需求走独立的数据协议（`events.mux`、`events.host` 两个流方法，不进 unary 的方法表），禁止伪装成 Remote 方法混进调用描述符，哪怕它们复用同一条 Connection。混进去的后果是调用语义被稀释，超时、取消、错误路由都要为"永不返回的调用"开例外。线上的验证是两级的：信封先过一遍 parse，载荷再过一遍，两层各有各的 schema。JSON 线上还有一个被显式承认的不完美：缺省和 undefined 在线上无法区分，`Wire<T>` 类型为此做了深层的可选放宽，这是把 TypeScript 的严格可选性和 JSON 的物理现实对齐的代价，不是疏忽。

### 下行从 SSE 换成了 WebSocket

截至 2026-08-04，浏览器真实载体对两条下行流各开一条独立的 WebSocket：`/api/events.mux` 只发 MuxFrame，`/api/events.host` 只发 HostFrame，每条文本消息是一份完整的 ServerRequest JSON。动机写得很具体：HTTP/1.1 浏览器对每个来源只给大约六条并发连接，SSE 时代每个页面长期占住两条，同源多标签页、插件资源和普通 RPC 会争抢连接槽，达到上限后不是降速而是排队阻塞。两条 WebSocket 不占这个配额。

边界也刻得清楚。WebSocket 只承担 host 到 browser 的下行；所有 client 到 host 的 unary 调用和对 server request 的 respond 继续走 `POST /api/*`，WebSocket 上不收任何客户端业务消息。upgrade 之前复用 `/api` 的 Host/Origin 信任栅栏，不受信的 authority 或跨来源 Origin 在事件流启动前就被拒绝。两条流保持独立生命周期、无跨流顺序保证，任一条中途失败都让整个连接代际失败，按既有退避策略重建；readiness 握手仍要等 mux、host 都 open 且 `host.describe` 这个 HTTP 调用成功，才发布 connected。网络上对 `/api/events.*` 的 GET 只返回 upgrade required，不保留 SSE 回退，官方的理由是双载体erness会让生产路径因代理差异静默分叉。进程内实现保留 SSE 编解码，用来验证协议的通道无关性。开发期的 HMR 事件走的是另一条独立的 SSE 通道 `/plugins/events`，和这两条 WebSocket 无关。

carrier 侧的抽象收得很紧。所有协议不变量住在 `AbstractApiClient` 里，平台差异只有两个切面：传输的 `doFetch` 和可覆写的 `onEnvelope` 观察钩子。同一套协议因此有四个实现：浏览器用 fetch 上行加 WebSocket 下行，Electron 走 IPC，测试用 Fixture（一个自洽的假服务器，帧的 rpcId 由它自己 mint），还有一个进程内实现跑完整的真协议但不占端口。平台差异是继承切面不是构造参数，这个选择否决的是工厂函数式的配置化：配置项堆出来的平台分支很快会烂在参数表里。观察钩子是实例私有的 microtask 批缓冲，监听器抛错被隔离。

## 一次远程调用的走查

看一个具体调用怎么过河。业务 service 在 Host 上用 `@Remote` 或 `@RemoteScope` 声明可调用方法，没标记的方法进不了生成的客户端类型，也调不到。要协作取消，Host 签名最后一个参数得是全局类型的 `signal: AbortSignal`，它记录在描述符里而不进 args，客户端生成的方法则接受最后一个可选的 AbortSignal。

客户端代码写 `await ctx.remote.goals.create(agentId, { objective: 'ship it' })`。这个调用走 Connection 的 `/api` 路由，变成 `connection.rpc.call('/api', 'goals/create', { args }, signal)`，HTTP carrier 把它映射成 `POST /api/goals/create`，载荷里只有一个具名的 args 对象。注意这是自定义信封，不是 JSON-RPC 的方法参数数组。

Host 侧，gateway 在这次调用里现解析描述符和活服务，不缓存业务对象。先做统一的信任检查，再校验 args 字段与描述符严格一致（多了少了都拒），用 codec 验证线上值，然后解析身份。身份解析是这套协议里最见功底的部分：复杂 Host 对象不过线。一个 `Agent` 参数在线上叫 `agentId`，是一个普通字符串；gateway 在调用前通过注册的 lookup provider 把它解析回 Host 的活对象。没注册 provider，调用以 `lookup-unavailable` 失败；配置可以按 key 覆盖解析策略而不动参数名和线字段。`agent` 和 `session` 两个 key 的解析器有完整语义：复用活的 Agent，冷的 session 自动恢复并对并发恢复去重，子 agent 路由拥有的身份被拒绝。然后才真正调到业务方法，返回值再过一遍验证，写进响应。

类型怎么到的浏览器？构建期。Host 构建先跑，Typert 生成器在这个阶段以 Host aggregate 为唯一的 program 种子运行，产出 strict 描述符、schema、运行时 codec 和客户端声明合并；Client 构建消费这些产物，不再启动生成器。两个构建永不进同一个 program，这正是前面两个 aggregate 的用途。类型安全是端到端的：Host 方法签名改了，Client 侧的 TypeScript 在下一次构建后报错，不是运行时才炸。

开发态有一条弱化通道。用 tsx 直跑 Host 时跳过 Typert 编译器插件，装饰器把方法名和调用模式记进一个模块私有的 WeakMap，gateway 据此拼一个临时描述符，不用起 program。这个 SRC 通道解析参数名、套 JSON 安全检查，但没有类型、schema、可选推断。浏览器那边对应的态度是拒绝：客户端不 mount 缺 strict codec 的 SRC 描述符，类型永远来自最近一次生成的产物。开发快了，验证没松。

## 断线之后：折叠与重建

会话历史在这个架构里是客户端的一次折叠：服务器不物化快照，客户端把收到的事件流折成当前视图，分页边界对齐消息边界。这个选择决定了重连语义。

断线重连不是续传，是重建。协议里没有恢复游标（`mux` 的 `since` 是个保留座位，传了忽略），重连后客户端比较订阅基线的 lastSeq 和 history 尾部 seq，有背隙就一次性补拉，然后重新折叠整个历史。听起来重，实际上把一堆难缠的状态机干掉了：没有游标就没有游标丢失的对账，没有增量会话就没有半新半旧的视图。审批和提问是这条纪律里的精确例外：首个回答获胜，Host 内存里的 pending 表是唯一裁判，被请求的帧在连接重开后重放，带着原来那个稳定的 rpcId，客户端不会因为断线漏掉一次审批，也不会因为重放重复应答；durable 侧另有 `approval/asked`、`approval/decided` 审计事件落日志，帧是 live 控制面，事件是 durable 审计，两层分开。

回执的语义同样精确。carrier 的回执是一个只有 `accepted` 布尔值的对象，文档明说它不是 RpcMessage，不参与消息协议。迟到的应答拿到 `not-pending`：一个答案在 pending 表里找不到自己的席位，得到的是一个明确的"你晚了"，不是异常，不是静默丢弃。未知的方法在信封 parse 时就大声失败，没有 not-implemented 的兜底分支。协议没有版本协商，客户端和 Host 绑定发布，这对一个同仓构建的两半是诚实的选择，比一套永远不会用到的版本握手便宜。

升级路径也留了口子：Connection 拥有传输、rpcId、响应信封和取消，gateway 只拥有 Remote 数据协议和业务分发。2026-08-04 的下行换载体就是这条口子的兑现：WebSocket 换掉 SSE，描述符、客户端 API、两条流的帧 schema 一行没动。

## 开发时的热更新：HMR 的边界

开发时 `dsh-client-hmr` 是 registry 的 watch driver，生产图完全省略 HMR 行，module host 自己从不 watch 文件，生产行为里没有一行文件监视代码。

Node 半用 stat-poll（不是 fs.watch，避开跨平台行为差异）盯每个图行的 bundle，默认 500 毫秒一趟（`pollIntervalMs` 配置项），基线在启动时同步捕获。检测到变化就调 `rebuilt(id)`，这是内容进图的唯一入口，前面说过它只在 rev 真变时才动图。rev 变更通过 `/plugins/events` 这条 SSE 通道推到浏览器半，浏览器拿到新 bundle 换上，不刷新页面。

HMR 的边界要划清：它处理 bundle 内容变化，不处理 manifest 结构变化。新增一个 client 插件改变了图的行集合，这个变化要重新加载页面才能拿到，因为 boot graph 是页面加载时注入一次的。改代码不刷新、加插件要刷新，这条线在实践中很清晰。

类型又是另一条线。改 Host 的 `@Remote` 方法签名，要重跑 Host 构建客户端才能看到类型变化，只有实现体的改动不需要再生成。`typecheck` 和 CI 用同一套顺序，本地能过的 CI 也能过。

## 浏览器侧的插件纪律

client 包不是一种东西，是三种。纯库（slots、primitives、loader 内核）没有任何插件声明，被别人 import。静态到达的包（connection、runtime、theme、i18n、HMR）不带 `dsh.client` 键，由 shell 直接打进基线。fetch 到达的插件（布局、侧栏、会话视图）才是 boot graph 的住户：双入口，Node 侧的 apply 是空的，实现在 `src/client/` 下走 `./client` 子路径。

三种包之间有一条硬线：跨插件 import 值是构建错误，tsdown 的纯度门禁当场拦下。插件之间要协作，值走 Cordis 服务，不走模块图。这条线把"浏览器里的插件"和"npm 包"区分开：npm 包的依赖图是构建期的，Cordis 插件的依赖图是运行期的、可逆的，两者混起来热重载就没有干净的边界。命名上每个包带组前缀（`dsh-host-runtime`、`dsh-client-ui`），在扁平的 npm 空间里前缀就是归属声明，代价是每个包都要在 tsconfig paths 里显式登记含 `/client` 子路径的映射。

## Chat Nodes 的分工与多 agent 场景

Conversation Node 在这个架构里是 Client 半的插件。Node 的 Definition 和 React 渲染器都在 client 包里，通过 client 侧的注册服务挂进浏览器插件图。Host 半只负责产生 session 事件，match、start、update、buildViewNode 的逻辑全在浏览器里跑。

分工一句话：Host 拥有事件真相，Client 拥有渲染逻辑。一个 client 包要贡献 Chat Node，三步：`package.json` 声明 `dsh.client` 进扫描，`exports["./client"]` 导出 bundle，bundle 的 apply 里注册 Definition 和渲染器。之后 Client Modules 把它组装进 boot graph，浏览器加载它，它注册自己的 Node。整个过程不改 Host 一行代码，UI 的演进不惊动 agent 组合。

多 agent 是这套架构的常态而不是附加题。Host 同时持有 root agent、subagent、后台 job，每个有独立的 session 和 scope。客户端通过 API gateway 拿到所有 agent 的状态和事件，Conversation Node 的 scope 机制让不同 agent 的对话渲染在不同的视图区域。scoped 调用有语法糖：`agentCtx.remote.goals.create({ objective: 'ship it' })`，agent 身份从参数里消失，由 scope 携带，同一个方法在 `ctx.remote` 上是全参版本、在 `agentCtx.remote` 上是省参版本，两个视图一个事实。namespace 是注册为 `remote.<namespace>` 的可追踪 Cordis 子服务，最后一个方法撤回时它随即卸载，依赖声明归实际调用方所有。

仓库里有一个特殊的包值得单独点名。`api-remotes` 是全仓库唯一一个 split Host 和 Client tsconfig 的包：它的 Host 条目必须参与 Host Typert 图，它的 client 条目又依赖 Host 构建先生成的 `/remote` 声明。它同时是一份警告：官方文档明说不要因为别的包也有 `src/index.ts` 加 `src/client/index.ts` 就复制这个 split。普通 client 插件老老实实做单项目。forwarded events 的 allowlist 是这个包的单点控制：哪些 Host 事件原样转发给消费者，一张数组说了算，类型投影、消费端 key、转发循环全部从它派生，两半读同一份声明而不是两份可能漂移的副本。当前组合挂载的 Remote 贡献是 goal 和只读的 Host 插件清单（`pluginInventory/list`）。

非浏览器入口不走这条路。Electron 用 `file://` 加载构建产物，fetch 走 IPC bridge，Web Server 整个不出场。Python SDK 走 JSON-RPC stdio 和 runtime 通话，那条是另一个协议（headless 通道），和这里的四象限协议是两回事。Web 给人用，程序化入口各有各的 carrier，Host 组合是共享的那一半。

## 权衡

两半分离有价签。贡献一个完整的 client 功能要同时理解两半的插件图、boot 协议、RPC 机制，跨进程的类型同步和构建顺序依赖都是真实成本。类型安全是构建期买断的：签名改了不重建，客户端就在旧类型上编译通过，运行时才对不上。

boot 协议是页面级的。图在加载时注入一次，运行中组合结构变了要重载页面。HMR 覆盖内容不覆盖结构，这条边界前面说过，这里再标一次它的方向：开发时的便利不承诺生产时的语义。

Web Server 没有安全层。loopback 是默认姿态，`0.0.0.0` 是显式豁免，非 loopback 部署需要自己在前排放反向代理补 TLS、认证和 origin 校验。host 配置只接受两个值的设计把这个决定变成了一个显式的、看一眼配置就知道风险级别的选择。

重建式重连在大历史上会有可感的重建成本。分页对齐消息边界缓解了渲染压力，但折叠本身每次重连都要做。这是拿性能换正确性：没有快照就没有快照不一致，代价是客户端多做功。

下行双 WebSocket 是连接预算的解，不是免费的。webserver 多一个 upgrade 注册面，connection 包多一项浏览器 WebSocket 实现依赖，进程内与浏览器两种物理编解码要分别维护；共享同一份 frame schema 和 `IApiClient` 语义是防止它长成第二套业务协议的绳子。stat-poll 有几百毫秒的检测延迟，这是跨平台一致性的价格。开发时感知是"保存之后一小会儿浏览器才更新"，生产无感，因为生产里根本没有轮询。

## 结论

Web 客户端的核心不是某个组件，是三条各管一段的纪律。代码一致性靠 rev 查询参数：内容哈希上 URL，`no-cache` 服务，缓存协议被整体排除出一致性机制。两半通话靠四象限消息协议：谁发起谁铸造 rpcId，业务代码不铸造，错误码一行一分支，传输失败和业务错误永不相混，JSON-RPC 因错误码降级和契约漂移被否决；下行自 2026-08-04 起走两条独立 WebSocket，上行保持 HTTP。能力暴露靠构建期生成的类型安全远程调用：复杂对象不过线、身份用字符串过河再在 Host 侧解析回活对象、每次调用现解析描述符不缓存业务对象。

Host 拥有事件真相，Client 拥有渲染逻辑，Chat Node 的贡献不动 Host 一行代码。断线是重建不是续传，审批靠稳定 rpcId 的重放保证不漏不重。代价是贡献者要跨两半的构建与类型链、boot 是页面级的、Web Server 裸奔在 loopback 默认之后。评估这套架构值不值得抄，先问自己的 UI 演进频率和 agent 组合演进频率是不是真的解耦：是，这份两半图的税交得值；不是，单体前端的账单便宜得多。

## 延伸阅读

- [Client Modules 子系统文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/client-modules.md)：扫描、boot graph、失败姿态的完整定义
- [HTTP Server 子系统文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/web-server.md)：路由模型、fallback 座位、config 与安全姿态
- [API Gateway 文档](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/api-gateway.md)：@Remote 模型、身份解析、错误语义
- [GUI Layering and RPC Protocol Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-07-19-gui-layering-and-rpc-protocol.md)：四象限协议、重连语义、被否决的替代方案
- [浏览器下行 WebSocket 载体 Agent Note](https://github.com/deepseek-ai/deepseek-harness/blob/master/.agents/notes/implemented/architecture/2026-08-04-websocket-downlink-carrier.md)：SSE 换 WebSocket 的动机与边界

上一篇：[dsh 的 Python SDK、Headless 与 JSON-RPC：把 agent 编进流水线](./40-python-sdk-headless-jsonrpc.md)
下一篇：[dsh 的错误处理与容错：这个 harness 怎么不崩](./42-error-handling-fault-tolerance-philosophy.md)
