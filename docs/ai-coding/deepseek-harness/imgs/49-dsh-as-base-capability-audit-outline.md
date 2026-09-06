---
type: mixed
density: rich
style: notion
generator: gpt-image-2
image_count: 21
language: zh
---

## Illustration 01

**Position**: 文章开头、总判定之后
**Purpose**: 以三层同心结构呈现 dsh 引擎、产品外壳与用户交付边界
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「当基座的能力审计」；标签「dsh 引擎」「产品外壳」「用户体系」「单用户能盖」「多用户自己包一层」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 01-framework-base-capability-map.webp

## Illustration 02

**Position**: 五个进入面章节开头
**Purpose**: 从控制面、扩展面、界面三种身份连接 Python SDK、headless、ACP、Web 插槽、工具插件
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「五个进入面」；标签「控制面」「扩展面」「界面」「Python SDK」「ACP」「工具插件」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 02-framework-five-entry-surfaces.webp

## Illustration 03

**Position**: Python SDK 小节后
**Purpose**: 父 Python 服务启动 dsh 子进程，以 JSON-RPC 驱动会话并显式隔离 DSH_HOME
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「Python SDK 进程树」；标签「Python 服务」「dsh 子进程」「JSON-RPC」「dsh_home」「RunResult」「会话隔离」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 03-flowchart-python-sdk-process-tree.webp

## Illustration 04

**Position**: headless 与 sdk profile 小节后
**Purpose**: 左右对比一次性命令执行和常驻 JSON-RPC 控制
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「headless 与 sdk profile」；标签「headless」「跑完退出」「纯文本 stdout」「sdk profile」「JSON-RPC」「结构化结果」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 04-comparison-headless-versus-sdk.webp

## Illustration 05

**Position**: ACP 小节后
**Purpose**: 编辑器客户端经 ACP 驱动会话、权限与取消；私有聊天卡片不在 wire 上
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「ACP 的控制边界」；标签「ACP」「会话控制」「权限询问」「取消」「automation-only」「无私有展示数据」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 05-framework-acp-control-boundary.webp

## Illustration 06

**Position**: Web 插槽小节后
**Purpose**: 左侧可扩展聊天卡片和工具卡，右侧独立页面需要 fork 官方布局
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「Web 插槽的边界」；标签「聊天卡片」「工具卡片」「插槽树」「独立页面」「fork 布局」「勉强」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 06-comparison-web-slots-boundary.webp

## Illustration 07

**Position**: 工具插件与 MCP 小节后
**Purpose**: Cordis 插件的 allow deny ask 策略钩子与 MCP 工具桥接并列，强调不可绕过的 guard
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「工具插件与 MCP」；标签「Cordis 插件」「MCP client」「allow」「deny」「ask」「单调 guard」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 07-framework-tool-plugin-mcp-guard.webp

## Illustration 08

**Position**: 能力底座段落中 ASCII 架构图后
**Purpose**: 交易研究服务、SDK 实例池、工具插件、MCP 与 Web 卡片的系统架构
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「交易工作台拼装图」；标签「外部 cron」「研究服务」「SDK 实例池」「行情 MCP」「下单风控」「Web 卡片」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 08-framework-trading-workbench-architecture.webp

## Illustration 09

**Position**: 自动化段落后
**Purpose**: jobs、workflow、schedule 按人是否在场与会话生命周期比较
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「自动化的三块拼图」；标签「jobs」「workflow」「schedule」「会话内」「最小 300 秒」「外部 cron」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 09-comparison-automation-three-parts.webp

## Illustration 10

**Position**: 状态与分发段落后
**Purpose**: JSONL 事件流、凭证引用和 preset 整装配置形成可审计可分发底座
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「状态、凭证与配置分发」；标签「JSONL v2」「fork 与恢复」「凭证引用」「脱敏视图」「preset」「三步装机」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 10-framework-state-credentials-distribution.webp

## Illustration 11

**Position**: 红线一小节开头
**Purpose**: 本机回环 Web 与多用户公网服务之间的安全边界，红色警示
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「红线一：不肯当服务器」；标签「127.0.0.1」「无 TLS」「无认证」「单用户本机」「多租户」「自己包服务层」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 11-infographic-red-line-no-server.webp

## Illustration 12

**Position**: 红线二小节开头
**Purpose**: 8 月 19 日到 9 月 4 日十个 release 的快节奏，标出持久化 API、SQLite、v2 格式变化
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「红线二：地基每几天动一次」；标签「8月19日」「9月4日」「10 个 release」「兼容性破坏」「JSONL v2」「每周 diff」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 12-timeline-red-line-fast-moving-foundation.webp

## Illustration 13

**Position**: 红线三小节开头
**Purpose**: 左侧 Python SDK 控制面，右侧 TypeScript code-runtime 执行面，指出 Python 后端仍实验性
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「红线三：执行面只会一种语言」；标签「Python SDK」「控制面」「TypeScript」「worker thread」「实验性 Python 后端」「MCP 包 Python」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 13-comparison-red-line-one-runtime-language.webp

## Illustration 14

**Position**: 常见误解章节后
**Purpose**: 三张并列误解卡，分别校正 SDK、Web 多会话与 ACP 数据源的推断
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「三个常见误解」；标签「SDK 不等于执行 Python」「多会话不等于多租户」「ACP 不等于全量 UI 数据」「控制面」「单用户」「私有展示数据」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 14-comparison-three-common-misconceptions.webp

## Illustration 15

**Position**: 从审计到动工小节后
**Purpose**: Day 1 到 Day 10 的可回滚验证路线，从 SDK、MCP、风控到并发压测
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「两周验证切片」；标签「Day 1 SDK」「Day 2 MCP」「Day 4 风控」「Day 6 headless」「Day 8 preset」「Day 10 并发」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 15-timeline-two-week-validation-slice.webp

## Illustration 16

**Position**: 判定表之后
**Purpose**: 绿黄红三列映射单用户桌面、单机自动化、编辑器、工作台、多用户 SaaS
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「产品能不能盖」；标签「绿灯 今天能盖」「黄灯 自己补」「红灯 改赛道」「单用户」「外部 cron」「多用户 SaaS」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 16-comparison-product-fit-decision.webp

## Illustration 17

**Position**: 判定表解释段落后
**Purpose**: 浅层能力与深层 UI、多用户能力呈阶梯，越深升级税越高
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「数你的层」；标签「工具挂架」「审批门」「凭证」「会话持久化」「整页界面」「多用户」「升级成本」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 17-framework-count-your-layers.webp

## Illustration 18

**Position**: Python SDK 隔离承诺段落后
**Purpose**: 两个 dsh_home 家目录隔离 CLI 与产品实例，不共享会话、配置、凭证
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「DSH_HOME 隔离」；标签「本机 CLI」「产品实例」「显式 dsh_home」「配置隔离」「凭证隔离」「会话隔离」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 18-flowchart-sdk-home-isolation.webp

## Illustration 19

**Position**: MCP 边界段落后
**Purpose**: stdio MCP 子进程启动时过滤 KEY PASSWORD SECRET TOKEN 与 DSH_* 环境变量
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「MCP 密钥清洗」；标签「stdio MCP」「环境变量」「KEY」「SECRET」「TOKEN」「过滤后启动」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 19-flowchart-mcp-secret-filter.webp

## Illustration 20

**Position**: 红线三资金敏感段落后
**Purpose**: 下单工具依次穿过金额阈值 ask、黑名单 deny、单调 guard 执行门
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「交易工具风控门」；标签「下单工具」「金额超限 ask」「黑名单 deny」「单调 guard」「模拟盘优先」「人工审批」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 20-flowchart-risk-control-gates.webp

## Illustration 21

**Position**: 边界章节后
**Purpose**: 版本变化触发：复查官方依据、压测隔离 workspace/session、更新判定表
**Visual Content**: Notion-style knowledge cards with clear connectors and article-specific Chinese labels.
**Text Plan**: 标题「审计刷新闭环」；标签「pre-1.0」「重查官方文档」「独立 workspace」「独立 session id」「压测」「更新判定表」；提示「按文章结论判断，不把生成图当证据」。
**Filename**: 21-flowchart-refresh-the-audit.webp
