---
type: mixed
density: rich
style: notion
palette: default
generator: gpt-image-2
image_count: 10
---

# 系统提示组装与动态 Cordis：配图大纲

## Illustration 1

**Position**: 开篇导语之后
**Purpose**: 总览每个 step 现场组装系统提示与工具 schema 的过程。
**Visual Content**: 注册表贡献汇入组装、排序、waterfall、插值，再到模型请求。
**Text Plan**: 标题「系统提示是现场组装的投影」；标签「注册表」「order」「waterfall」「renderPrompt」「工具 schema」；提示「插件树决定下一个 step 看到的世界」。
**Filename**: 15-01-flowchart-prompt-assembly.png

## Illustration 2

**Position**: “提示是投影，不是文件”中四类贡献之后
**Purpose**: 说明事实所有者注册对应贡献。
**Visual Content**: 四张贡献卡围绕 ctx.systemPrompt 注册表。
**Text Plan**: 标题「谁拥有事实，谁注册贡献」；标签「段落」「上下文」「工具 schema」「变量」「ctx.systemPrompt」；提示「中心文本不再转述」。
**Filename**: 15-02-framework-four-contributions.png

## Illustration 3

**Position**: 全局层与 scope 层说明之后
**Purpose**: 展示 scoped 同名贡献覆盖 global 而不改全局。
**Visual Content**: 两层堆叠的注册表和一个子 agent scope 覆盖卡。
**Text Plan**: 标题「scope 覆盖 global」；标签「全局层」「scope 层」「同名覆盖」「子 agent」；提示「换人格、换工具集，不动全局」。
**Filename**: 15-03-comparison-global-scope.png

## Illustration 4

**Position**: step 组装文本块之后
**Purpose**: 把完整组装顺序可视化。
**Visual Content**: 从贡献、合并、排序、waterfall、complete、render 到请求的横向流程。
**Text Plan**: 标题「跟着一个 step 组装一次」；标签「合并」「按 order 排序」「assemble waterfall」「complete」「插值渲染」；提示「工具 schema 与 system 文本同次组装」。
**Filename**: 15-04-flowchart-step-assembly.png

## Illustration 5

**Position**: 动态上下文段落之后
**Purpose**: 区分 system 前缀与追加到日志的 user 快照。
**Visual Content**: 左侧静态 system 前缀，右侧随变化追加的 user context snapshots。
**Text Plan**: 标题「动态上下文不进 system 前缀」；标签「user 快照」「内容变化」「会话日志」「当前无运行时上下文」；提示「变化对模型是追加，不是前缀改写」。
**Filename**: 15-05-comparison-context-snapshot.png

## Illustration 6

**Position**: “每步重拼，前缀稳定”开头
**Purpose**: 解释相同字节可复用 KV cache、任一变动从首个 token 失效。
**Visual Content**: 两条请求前缀对比：稳定复用与某处改变后的 cache miss。
**Text Plan**: 标题「每步重拼，前缀稳定」；标签「相同字节」「KV cache」「插件加载顺序无关」「第一个改动 token」；提示「不变不付代价」。
**Filename**: 15-06-comparison-prefix-cache.png

## Illustration 7

**Position**: 动态 Cordis 的五工具说明之后
**Purpose**: 给 cordis_inspect / define / run / stop / undefine 一个生命周期地图。
**Visual Content**: 五个动词构成定义、运行、停止、忘记的循环，inspect 旁路观察。
**Text Plan**: 标题「动态 Cordis 的五个动词」；标签「cordis_inspect」「cordis_define」「cordis_run」「cordis_stop」「cordis_undefine」；提示「define 不运行，stop 保留定义」。
**Filename**: 15-07-flowchart-cordis-tools.png

## Illustration 8

**Position**: run 两种形状说明之后
**Purpose**: 对比 host-only vm 运行与带浏览器半边的人工批准往返。
**Visual Content**: 左右两路径：Host 半边进入 vm sandbox，浏览器半边发请求并等待页面批准。
**Text Plan**: 标题「cordis_run 的两种形状」；标签「host 半边」「vm 沙箱」「浏览器半边」「页面批准」「取消信号」；提示「无头部署会一直等待」。
**Filename**: 15-08-comparison-host-browser-run.png

## Illustration 9

**Position**: 自指与运行时边界段落之后
**Purpose**: 展示动态包注册新工具后，下一 step 的模型工具列表立即变化。
**Visual Content**: 当前 step 的 agent 调 cordis_run、插件树新增工具、下一 step request 工具列表改变。
**Text Plan**: 标题「agent 改自己的插件树」；标签「当前 step」「动态包」「注册新工具」「下一个 step」「进程内存」；提示「能加不能拆，不跨重启」。
**Filename**: 15-09-flowchart-self-referential.png

## Illustration 10

**Position**: “撤销是结构保证”之后
**Purpose**: 说明 fiber 把注册记账并在 stop 时按序冲销，及 bash 级信任边界。
**Visual Content**: fiber 生命周期账本记录工具、监听器、服务、定时器，stop 触发回滚到静止；外圈标注 bash trust。
**Text Plan**: 标题「撤销是结构保证」；标签「fiber」「可逆副作用」「工具 监听器 服务 定时器」「cordis_stop」「bash 级信任」；提示「挂载是实验，卸载是常态」。
**Filename**: 15-10-framework-reversible-fiber.png
