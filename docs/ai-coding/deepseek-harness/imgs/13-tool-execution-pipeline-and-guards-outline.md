---
type: mixed
density: rich
style: notion
palette: default
image_count: 10
article: 13-tool-execution-pipeline-and-guards.md
---

# 工具执行管线与守卫：七道关卡 — 配图大纲

## Illustration 1
**Position**: 七道关卡文字流程之后
**Purpose**: 总览一次工具调用从模型到日志的七层生命周期。
**Visual Content**: 七个编号关卡由 tool_call 连到 session log。
**Text Plan**: 标题「工具调用的七道关卡」；标签「前置策略」「审批」「守卫」「执行包装」「结果处置」「归一化 / 冻结」；提示「策略挂在关卡上」。
**Filename**: 13-01-flowchart-seven-gates.png

## Illustration 2
**Position**: 前置策略门段落之后
**Purpose**: 强调前置策略只有放行、拒绝、审批三种表态，不能篡改已记录参数。
**Visual Content**: 已记录参数经过前置策略三分支，改参数分支被禁止。
**Text Plan**: 标题「前置策略只能表态，不能改参数」；标签「放行」「拒绝」「转问审批」「参数已入日志」；提示「想规范入参，只能拒绝后重试」。
**Filename**: 13-02-flowchart-pre-policy-boundary.png

## Illustration 3
**Position**: 审批与守卫段落之后
**Purpose**: 对比审批只有明确批准才放行，与守卫只能拒绝的单向权力。
**Visual Content**: approval 和 guard 两个门，明确批准绿通行、所有不确定红拒绝、guard 无放行按钮。
**Text Plan**: 标题「审批先表态，守卫只会拒绝」；标签「明确批准」「不确定即拒绝」「guard」「不能放行」；提示「拒绝一旦发生，顺序无法翻案」。
**Filename**: 13-03-comparison-approval-guard.png

## Illustration 4
**Position**: 执行包装段落之后
**Purpose**: 将执行包装与工具本体及文件接缝的深层策略区分开。
**Visual Content**: timeout/retry/metric 外层包裹 tool body，文件工具进入 observation/version check。
**Text Plan**: 标题「执行被包装，工具在最内层」；标签「timeout」「retry」「耗时观测」「tool body」「版本核对」；提示「执行包装不是执行决策」。
**Filename**: 13-04-framework-execution-wrapper.png

## Illustration 5
**Position**: 结果处置与归一化段落之后
**Purpose**: 表现结果处置可以改内容但不能改成败，归一化兜住异常。
**Visual Content**: 原始结果经 accept/replace/intercept/context，到 normalization 统一成规范结果；success/failure 锁定。
**Text Plan**: 标题「结果可处置，成败不可翻案」；标签「接受 / 替换」「拦截 / 上下文」「normalization」「规范结果」；提示「异常也收敛为一条结果」。
**Filename**: 13-05-flowchart-result-disposal-normalization.png

## Illustration 6
**Position**: 两个时序撑起真实性段落之后
**Purpose**: 显示调用先落日志、结果终检后冻结的前后真实性约束。
**Visual Content**: 左半时序 tool_call → session log → pipeline；右半 pipeline → final check → frozen result → UI/telemetry/storage。
**Text Plan**: 标题「真实性靠两个时序」；标签「先记日志，再执行」「终检」「冻结结果」「同一个对象」；提示「调用不缺席，观察者不分叉」。
**Filename**: 13-06-timeline-log-freeze-truth.png

## Illustration 7
**Position**: 并发段落之后
**Purpose**: 分离有序策略、重叠执行和有序提交。
**Visual Content**: 三泳道：按提交顺序的 gates、并发 execution、按提交顺序的结果日志。
**Text Plan**: 标题「策略有序，执行重叠」；标签「提交顺序」「并发执行」「有序提交」「独占调用」；提示「快结果宁可排队也不乱序」。
**Filename**: 13-07-framework-concurrency-order.png

## Illustration 8
**Position**: Code Mode 段落之后
**Purpose**: 说明程序内部子调用没有绕过策略。
**Visual Content**: Code Mode 程序块内的子调用仍返回同一七道关卡；程序外直调被拒绝并指路。
**Text Plan**: 标题「Code Mode 不开策略后门」；标签「程序内子调用」「同一条管线」「绑定拒绝」「程序外直调拒绝」；提示「特殊入口也要过同样的门」。
**Filename**: 13-08-flowchart-code-mode-no-bypass.png

## Illustration 9
**Position**: 权衡收益段落之后
**Purpose**: 以纵深防御层展现一个策略失效不会击穿其他层。
**Visual Content**: 多层防御屏障：hook、approval、guard、normalization；漏洞在前层被后层阻挡。
**Text Plan**: 标题「纵深防御：一个策略失效不击穿全局」；标签「hook」「approval」「guard」「normalization」；提示「每层只管一个关注点」。
**Filename**: 13-09-framework-defense-depth.png

## Illustration 10
**Position**: 结论之前
**Purpose**: 汇总七道管线给出的顺序、真实性与安全三项保证。
**Visual Content**: 管线中心，三张保证卡为顺序、真实性、拒绝不可翻案。
**Text Plan**: 标题「七道关卡换来三项保证」；标签「顺序稳定」「调用可追溯」「结果一致」「拒绝不可翻案」；提示「安全不靠自觉」。
**Filename**: 13-10-framework-pipeline-guarantees.png
