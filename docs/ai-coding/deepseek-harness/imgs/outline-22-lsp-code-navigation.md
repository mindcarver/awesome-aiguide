---
article: 22-lsp-code-navigation.md
type: mixed
density: rich
style: notion
palette: default
image_count: 16
generator: gpt-image-2
---

# LSP 接缝插图大纲

## Illustration 1
**Position**: grep 的天花板
**Purpose**: 对比文本命中与语义导航。
**Visual Content**: 同名字符串噪声与精确符号引用。
**Text Plan**: 标题「grep 的天花板」；标签「文本命中」「语义引用」「定义」「注释」；提示「代码要按结构理解」。
**Filename**: 22-01-grep-vs-semantic-navigation.png

## Illustration 2
**Position**: grep 的天花板结尾
**Purpose**: 展示接缝吸收的协议复杂度。
**Visual Content**: 生命周期、文档同步、能力协商被压入 provider。
**Text Plan**: 标题「极薄的 LSP 接缝」；标签「生命周期」「文档同步」「能力协商」「Provider」；提示「模型只问导航问题」。
**Filename**: 22-02-lsp-seam-hides-protocol.png

## Illustration 3
**Position**: 四个操作，封闭联合
**Purpose**: 说明四个统一查询操作。
**Visual Content**: 四张操作卡围绕共享请求 schema。
**Text Plan**: 标题「四个操作，封闭联合」；标签「跳定义」「查引用」「跳实现」「悬停」；提示「加一个操作，三层一起改」。
**Filename**: 22-03-four-closed-operations.png

## Illustration 4
**Position**: 工具定位
**Purpose**: 说明 LSP 是精度辅助而非日常入口。
**Visual Content**: search/read 与 lsp 的分流决策。
**Text Plan**: 标题「LSP 是精度辅助」；标签「search」「read」「lsp」「语义歧义」；提示「先普通导航，必要时再精确查询」。
**Filename**: 22-04-lsp-precision-assist.png

## Illustration 5
**Position**: 无逃生舱
**Purpose**: 表现封闭调用面带来的安全边界。
**Visual Content**: 只读四操作通过闸门，任意 JSON-RPC 被拒绝。
**Text Plan**: 标题「没有协议逃生舱」；标签「只读导航」「JSON-RPC」「拒收 workspace edit」「不执行命令」；提示「审过的才是全部」。
**Filename**: 22-05-no-escape-hatch.png

## Illustration 6
**Position**: 注册与选择
**Purpose**: 展示按扩展名、顺序无关的路由。
**Visual Content**: .ts/.rs/.py 路由至独立 provider。
**Text Plan**: 标题「按扩展名选 Provider」；标签「.ts」「.rs」「.py」「与顺序无关」；提示「选择是部署的事」。
**Filename**: 22-06-extension-routing.png

## Illustration 7
**Position**: 注册原子性
**Purpose**: 说明冲突时整体不发布。
**Visual Content**: 注册事务成功与 .ts 冲突回滚对照。
**Text Plan**: 标题「注册原子，冲突不发布」；标签「branded id」「扩展名映射」「冲突」「rollback」；提示「没有半注册状态」。
**Filename**: 22-07-atomic-registration.png

## Illustration 8
**Position**: stdio provider 部署
**Purpose**: 呈现配置显式、按需懒启动与资源边界。
**Visual Content**: servers 配置到懒启动进程的流程。
**Text Plan**: 标题「语言服务器显式部署」；标签「argv 直传」「16MiB」「4MiB」「懒启动」；提示「不自动发现，不猜参数」。
**Filename**: 22-08-explicit-server-deployment.png

## Illustration 9
**Position**: provider 拿到的是加过工的请求
**Purpose**: 解释完整请求与派生 languageId。
**Visual Content**: 4 个必填字段进入接缝后补语言 id。
**Text Plan**: 标题「请求从诞生就是完备的」；标签「operation」「file path」「position」「workspace root」「languageId」；提示「无需 resolve」。
**Filename**: 22-09-complete-derived-request.png

## Illustration 10
**Position**: 坐标
**Purpose**: 解释模型友好坐标与协议坐标转换。
**Visual Content**: 一基行列转换为零基 UTF-16。
**Text Plan**: 标题「工具层转换坐标」；标签「一基」「零基」「UTF-16」「半开区间」；提示「emoji 附近要小心列号」。
**Filename**: 22-10-coordinate-conversion.png

## Illustration 11
**Position**: 规范工作区 URI
**Purpose**: 展示内外工作区路径渲染边界。
**Visual Content**: provider canonical URI 分开工作区内外位置。
**Text Plan**: 标题「用规范工作区 URI 定坐标」；标签「file: URI」「工作区内」「工作区外」「远程沙箱」；提示「不要拿宿主路径规则猜」。
**Filename**: 22-11-canonical-workspace-uri.png

## Illustration 12
**Position**: 一个查询的一生
**Purpose**: 总览从模型调用到归一化结果的链路。
**Visual Content**: 端到端的查询流水线。
**Text Plan**: 标题「一次 LSP 查询的一生」；标签「ctx.lsp.query」「stdio provider」「didOpen」「JSON-RPC」「归一化结果」；提示「模型看不到协议细节」。
**Filename**: 22-12-query-lifecycle.png

## Illustration 13
**Position**: 瞬态打开
**Purpose**: 对比瞬态同步与长期文档状态机。
**Visual Content**: 每次读取—打开—查询—关闭对比持续同步。
**Text Plan**: 标题「瞬态打开，查完即弃」；标签「read current text」「didOpen」「query」「didClose」；提示「事实源是磁盘」。
**Filename**: 22-13-transient-document-sync.png

## Illustration 14
**Position**: 实例池与串行
**Purpose**: 表达单工作区串行、跨工作区并行。
**Visual Content**: workspace A 队列与 workspace B 并行进程。
**Text Plan**: 标题「同工作区串行，不同工作区并行」；标签「实例池」「queue」「Workspace A」「Workspace B」；提示「把终止爆炸半径限在一次查询」。
**Filename**: 22-14-instance-pool-serialization.png

## Illustration 15
**Position**: 结果与错误
**Purpose**: 说明结果联合、null 与稳定错误码。
**Visual Content**: 位置结果、hover null 与异常码分支。
**Text Plan**: 标题「结果与错误都是封闭词表」；标签「locations」「hover: null」「LSP_UNAVAILABLE」「畸形响应」；提示「按码路由，不解析消息」。
**Filename**: 22-15-results-and-errors.png

## Illustration 16
**Position**: 结论
**Purpose**: 总结 ctx.lsp 的克制边界与价值。
**Visual Content**: 四操作、封闭接口、可替换 provider 的总结图。
**Text Plan**: 标题「可靠的 LSP 接缝」；标签「四个只读操作」「封闭调用面」「可替换 Provider」「跨语言稳定」；提示「把协议细节压回 Provider」。
**Filename**: 22-16-lsp-seam-summary.png
