---
article: 23-code-runtime-and-code-mode.md
type: mixed
density: rich
style: notion
palette: default
image_count: 16
generator: gpt-image-2
---

# Code Runtime 与 Code Mode 插图大纲

## Illustration 1
**Position**: 为什么 agent 要写代码并执行
**Purpose**: 对比逐次 tool call 与一次程序编排。
**Visual Content**: 30 文件任务由逐步调用压缩为循环程序。
**Text Plan**: 标题「从工具往返到程序编排」；标签「30 个文件」「循环」「条件」「一次执行」；提示「中间状态住在变量里」。
**Filename**: 23-01-program-orchestration.png

## Illustration 2
**Position**: 分层解法
**Purpose**: 展示执行接缝与 Code Mode 的职责分离。
**Visual Content**: ctx.codeRuntime、Code Mode 与工具策略管线三层。
**Text Plan**: 标题「两层，各管一件事」；标签「ctx.codeRuntime」「Code Mode」「tools」「策略管线」；提示「子调用照样过关卡」。
**Filename**: 23-02-runtime-mode-layers.png

## Illustration 3
**Position**: 底层接缝
**Purpose**: 介绍 run、language、isolation 三成员。
**Visual Content**: 三张接口卡及 isolation 非安全评级提示。
**Text Plan**: 标题「ctx.codeRuntime 只有三个成员」；标签「run(request)」「language」「isolation」「worker-thread」；提示「隔离是诊断标签，不是安全承诺」。
**Filename**: 23-03-runtime-contract.png

## Illustration 4
**Position**: 请求与结果
**Purpose**: 说明请求完整、异步函数体和可中止性。
**Visual Content**: program、bindings、signal 流入 async 函数体。
**Text Plan**: 标题「请求带一切」；标签「program」「bindings」「signal」「await」「return」；提示「预算在配置，不藏在请求里」。
**Filename**: 23-04-run-request.png

## Illustration 5
**Position**: 错误是字段
**Purpose**: 解释 resolve-on-failure 和 undefined/null 区分。
**Visual Content**: CodeRunResult 统一承载 value/logs/error。
**Text Plan**: 标题「错误是字段，不是异常」；标签「value」「logs」「error」「undefined」「null」；提示「失败也作为结果返回」。
**Filename**: 23-05-result-not-exception.png

## Illustration 6
**Position**: 预算与内存线
**Purpose**: 区分 compute、wall、output、heap 四条界限。
**Visual Content**: 四种仪表盘分别指向不同资源。
**Text Plan**: 标题「三条预算加一条内存线」；标签「60 秒 compute」「600 秒 wall」「64 MiB output」「512 MiB heap」；提示「每条只管一个维度」。
**Filename**: 23-06-resource-budgets.png

## Illustration 7
**Position**: 绑定
**Purpose**: 描绘宿主工具如何成为程序全局 tools。
**Visual Content**: 宿主 binding 到 null-prototype tools 对象。
**Text Plan**: 标题「宿主函数变成程序全局」；标签「tools」「async 函数」「无损 JSON」「__proto__」；提示「程序输入按不可信处理」。
**Filename**: 23-07-safe-bindings.png

## Illustration 8
**Position**: 失败分类
**Purpose**: 归纳六种正交失败及后续动作。
**Visual Content**: 六张失败卡和恢复动作。
**Text Plan**: 标题「六种正交结果」；标签「exception」「timeout」「abort」「worker-exit」「invalid-output」「output-limit」；提示「不同失败，走不同恢复路径」。
**Filename**: 23-08-six-failure-kinds.png

## Illustration 9
**Position**: Code Mode
**Purpose**: 展示 native/code/both 三态开关。
**Visual Content**: 三个工具可见性模式对照。
**Text Plan**: 标题「Code Mode 改变工具入口」；标签「native」「code」「both」「run_code」；提示「模式选择可审计」。
**Filename**: 23-09-code-mode-switch.png

## Illustration 10
**Position**: 语言分发 SDK
**Purpose**: 表现按运行时语言生成一致 SDK 与传输描述。
**Visual Content**: TypeScript/Python 分发到对应 SDK。
**Text Plan**: 标题「SDK 跟着运行时语言走」；标签「TypeScript」「Python」「SDK renderer」「run_code」；提示「语言与传输描述必须一致」。
**Filename**: 23-10-language-sdk-dispatch.png

## Illustration 11
**Position**: 子调用完整管线
**Purpose**: 展示 run_code 和每项子调用均受守卫审批。
**Visual Content**: 外层 run_code 和子调用穿过同一管线。
**Text Plan**: 标题「每个子调用都走完整管线」；标签「pre-execute」「guard」「审批」「post-execute」「父 token」；提示「程序不是策略后门」。
**Filename**: 23-11-subcall-full-pipeline.png

## Illustration 12
**Position**: ToolCallError
**Purpose**: 说明策略拒绝可被程序捕获并继续。
**Visual Content**: 工具拒绝转为 catch 的 ToolCallError。
**Text Plan**: 标题「拒绝变成可 catch 的事件」；标签「ToolCallError」「toolName」「catch」「失败清单」；提示「策略说不，不必杀掉程序」。
**Filename**: 23-12-tool-call-error.png

## Illustration 13
**Position**: 并发分发
**Purpose**: 展示并行安全调用与独占屏障。
**Visual Content**: Promise.all 并行池和独占调用排干屏障。
**Text Plan**: 标题「并发复用原生约定」；标签「Promise.all」「maxParallelSubCalls=10」「并行」「独占屏障」；提示「只有工具体真正并发」。
**Filename**: 23-13-parallel-subcalls.png

## Illustration 14
**Position**: spill
**Purpose**: 展示大子调用日志的持久化副本溢出。
**Visual Content**: 大结果经 tools/code-dispatch-log 写入 spill。
**Text Plan**: 标题「子调用日志也要过 spill」；标签「maxInlineBytes」「preview」「dispatch」「完整日志」；提示「模型结果和持久化副本分开」。
**Filename**: 23-14-dispatch-log-spill.png

## Illustration 15
**Position**: TypeScript worker
**Purpose**: 展示每次新 worker、敌意端口验证、终止等待。
**Visual Content**: source strip → fresh worker → hostile port validation。
**Text Plan**: 标题「TypeScript 每次运行一个新 Worker」；标签「stripTypeScriptTypes」「512 MiB」「fresh Worker」「端口验证」；提示「不池化，避免跨运行串状态」。
**Filename**: 23-15-fresh-worker-runtime.png

## Illustration 16
**Position**: 安全边界与结论
**Purpose**: 强调 worker 不是强制边界，审批和沙箱才是。
**Visual Content**: worker containment 内层，approval/sandbox 外层强制边界。
**Text Plan**: 标题「安全边界不在 Worker」；标签「worker 是遏制」「审批」「沙箱」「container」；提示「硬隔离要让 code 和 bash 一起换」。
**Filename**: 23-16-security-boundary-summary.png
