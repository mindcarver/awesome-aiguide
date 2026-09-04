---
type: mixed
density: rich
style: notion
palette: default
generator: gpt-image-2
image_count: 10
---

# 工具执行管线与守卫：配图大纲

## Illustration 1

**Position**: 开篇导语之后
**Purpose**: 建立“策略挂在固定管线、工具不承载策略”的总览心智模型。
**Visual Content**: 从模型调用到会话日志的七段横向管线，策略卡片附着在对应关卡。
**Text Plan**: 标题「七道关卡」；标签「前置策略门」「审批」「守卫」「执行包装」「结果处置」「归一化」「终检冻结」；提示「策略挂在关卡上」。
**Filename**: 13-01-flowchart-seven-gates.webp

## Illustration 2

**Position**: “策略不能长在工具里”末尾
**Purpose**: 对比散落策略与机制策略分离。
**Visual Content**: 左侧工具内部纠缠的策略线团，右侧工具外部固定管线和可插拔监听器。
**Text Plan**: 标题「机制和策略分离」；标签「工具」「固定管线」「监听器」「跨工具家族」；提示「机制不认识具体策略」。
**Filename**: 13-02-comparison-policy-separation.webp

## Illustration 3

**Position**: “执行前的三道门”开头
**Purpose**: 解释执行前依次经过前置策略、一次性审批、守卫。
**Visual Content**: 三道从左到右的门，调用卡通过或被拒，守卫门明确只有拒绝出口。
**Text Plan**: 标题「执行前的三道门」；标签「钩子 权限 沙箱」「一次性审批」「守卫」「放行」「拒绝」「转问审批」；提示「不确定，一律拒绝」。
**Filename**: 13-03-flowchart-pre-execution-gates.webp

## Illustration 4

**Position**: 审批与守卫规则说明之后
**Purpose**: 强调审批 fail-closed 与守卫只拒不放的不可翻案边界。
**Visual Content**: 两列决策卡对比：审批的明确批准与缺席/拒绝；守卫的拒绝或不表态，红线挡住宽松监听器。
**Text Plan**: 标题「审批与守卫的权力边界」；标签「明确批准」「审批缺席」「拒绝」「只拒不放」「部署方红线」；提示「守卫拒绝，后续无法翻案」。
**Filename**: 13-04-comparison-approval-guard.webp

## Illustration 5

**Position**: “执行与处置”开头
**Purpose**: 区分执行包装、工具本体和结果处置的职责。
**Visual Content**: 同心包装图：最内工具本体，外圈超时/重试/耗时观测，输出端是结果处置。
**Text Plan**: 标题「怎么跑，结果谁说了算」；标签「工具本体」「超时」「重试」「耗时观测」「结果处置」；提示「处置能改结果，改不了成败」。
**Filename**: 13-05-framework-execution-result.webp

## Illustration 6

**Position**: “归一化”段落之后
**Purpose**: 展示任何异常都会被收敛为一条规范失败结果。
**Visual Content**: 多种故障输入（策略崩溃、包装崩溃、工具异常、不可序列化值）汇聚到归一化漏斗，再输出结构化失败结果。
**Text Plan**: 标题「归一化兜住一切崩溃」；标签「策略崩溃」「工具异常」「不可序列化」「结构化失败」；提示「每个调用恰好一条规范结果」。
**Filename**: 13-06-infographic-normalization.webp

## Illustration 7

**Position**: “两个时序撑起真实性”末尾
**Purpose**: 使调用先记日志、结果冻结后再被观察的时序可见。
**Visual Content**: 上下两条时间线，分别对应调用先落日志再执行、结果终检冻结后分发给界面/遥测/持久化。
**Text Plan**: 标题「两条时序，撑起真实性」；标签「调用先落日志」「再执行」「终检」「结果冻结」「界面 遥测 持久化」；提示「所有观察者看同一个对象」。
**Filename**: 13-07-timeline-truthfulness.webp

## Illustration 8

**Position**: “并发：策略有序，执行重叠”开头
**Purpose**: 区分有序策略段、并发执行段和按提交顺序结果段。
**Visual Content**: 三泳道：策略逐项判定、执行调用重叠、结果按提交序归队。
**Text Plan**: 标题「策略有序，执行可以重叠」；标签「提交顺序」「策略串行」「执行并发」「结果有序」；提示「快结果宁可排队也不乱序」。
**Filename**: 13-08-flowchart-concurrency-segments.webp

## Illustration 9

**Position**: 四调用并发示例之后
**Purpose**: 让读者一眼看到读 A/读 B 的并发、写 A 的独占和读 C 的等待。
**Visual Content**: 甘特式时序卡，读文件 A/B 重叠，写文件 A 独占，读文件 C 最后运行，底部按 A/B/写 A/C 提交。
**Text Plan**: 标题「四个调用如何排队」；标签「读文件 A」「读文件 B」「写文件 A」「读文件 C」「独占」；提示「写操作排干并发后单独运行」。
**Filename**: 13-09-timeline-call-scheduling.webp

## Illustration 10

**Position**: “特殊入口不开后门”之后
**Purpose**: 说明 Code Mode 子调用与文件接缝仍使用同一管线。
**Visual Content**: Code Mode 程序和文件改动接缝从不同入口汇入同一七道关卡，外部直调工具被拒绝。
**Text Plan**: 标题「特殊入口不开后门」；标签「Code Mode」「子调用」「文件接缝」「同一条管线」「直调工具拒绝」；提示「所有入口复用同一套策略」。
**Filename**: 13-10-framework-no-backdoor.webp
