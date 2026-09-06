# Asset Plan

Content type: technical architecture and policy analysis. Every planned image is an original explanatory illustration; article links remain the evidence source.

## Asset 01: 子代理为什么不许问

- Position: 开头约束段
- Type: original framework knowledge-card illustration
- Purpose: 后台子代理没有可见审批界面，因此将审批策略固定为 never，决定全部前置
- Source facts: 子代理；审批 never；隐形阻塞；确定性拒绝；权限快照；不重试
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-01-framework-approval-never-constraint.webp

## Asset 02: 子代理模型路由三层

- Position: 三十秒模型
- Type: original framework knowledge-card illustration
- Purpose: 授权、快照、执行三层加选择层传送带的整体模型
- Source facts: 授权层；快照层；选择层；执行层；精确白名单；调用前复核
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-02-framework-three-layer-routing.webp

## Asset 03: 授权层：精确白名单

- Position: 授权层小节
- Type: original framework knowledge-card illustration
- Purpose: Host 设置中逐条授权 provider 加 model 对，拒绝前缀和通配扩权
- Source facts: Host 设置；provider + model；精确匹配；无通配符；用户同意；新适配器不扩权
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-03-framework-authorization-whitelist.webp

## Asset 04: 设置保存的原子性

- Position: 授权层设置界面段
- Type: original flowchart knowledge-card illustration
- Purpose: 开关与路由表一次版本栅栏提交，禁用保留列表、启用至少一条
- Source facts: 开关；路由表；原子提交；版本栅栏；禁用保留；启用至少一条
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-04-flowchart-atomic-settings-commit.webp

## Asset 05: 快照层：会话事实

- Position: 快照层小节
- Type: original flowchart knowledge-card illustration
- Purpose: 新会话将授权表写成持久 subagent/model-selection-policy 事件，子会话继承、旧会话恢复旧表
- Source facts: 新会话；持久事件；子会话继承；恢复旧表；JSONL 日志；历史可复现
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-05-flowchart-session-policy-snapshot.webp

## Asset 06: 选择层：参数合并

- Position: 选择层小节
- Type: original flowchart knowledge-card illustration
- Purpose: 静态默认、部署配置、调用参数三来源按优先级合并，provider 和 model 成对出现
- Source facts: 静态默认；配置覆盖；调用参数；provider + model；effort 清除；新模型默认
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-06-flowchart-selection-parameter-precedence.webp

## Asset 07: 发现工具的交集

- Position: 选择层发现工具段
- Type: original framework knowledge-card illustration
- Purpose: list_subagent_models 只展示授权快照与活适配器目录的交集
- Source facts: list_subagent_models；授权快照；活目录；交集；目录咨询性；授权硬闸
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-07-framework-discovery-authorized-intersection.webp

## Asset 08: 执行层：贴着副作用复核

- Position: 执行层小节
- Type: original flowchart knowledge-card illustration
- Purpose: 执行器在创建子代理前独立复核白名单，并处理取消与 HMR 适配器变更
- Source facts: 工具参数；独立复核；取消检查；provider 仍注册；HMR；创建子代理前
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-08-flowchart-executor-defensive-recheck.webp

## Asset 09: 能力门：支持与拒绝

- Position: 能力门小节
- Type: original comparison knowledge-card illustration
- Purpose: 支持动态选择的 spawn fork SDK 与明确拒绝的 ACP Codex Claude Code 两列对比
- Source facts: spawn；fork；DSH SDK；ACP；Codex；Claude Code
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-09-comparison-capability-gate-providers.webp

## Asset 10: 固定模型不等于动态选择

- Position: 能力门解释段
- Type: original comparison knowledge-card illustration
- Purpose: 部署时 provider 固定 model 字段与委派参数运行时选择的区别
- Source facts: 固定 model；部署常量；动态选择；委派参数；挂载失败；不静默降级
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-10-comparison-fixed-versus-dynamic-model.webp

## Asset 11: fork 为什么禁用选择

- Position: fork 小节
- Type: original framework knowledge-card illustration
- Purpose: 复制前缀可复用 KV 缓存；换 provider/model 导致完整重算，节省被吞掉
- Source facts: subagent_fork；已完成前缀；KV 缓存；换模型失效；重算成本；禁用选择
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-11-framework-fork-kv-cache-economics.webp

## Asset 12: 运维台的一周

- Position: 贯穿案例
- Type: original timeline knowledge-card illustration
- Purpose: 从周一授权到周五恢复的时间线，包含越界拒绝与新会话生效
- Source facts: 周一授权；周二快照；周三拒绝；周四扩表；周五恢复；旧会话不热更新
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-12-timeline-ops-week-routing-story.webp

## Asset 13: 五个常见误解

- Position: 常见误解
- Type: original comparison knowledge-card illustration
- Purpose: 并排校正授权范围、快照热更新、固定模型、通配授权和开关语义
- Source facts: 只管子代理；快照不热更新；固定不等于动态；无通配；关闭保留列表；精确授权
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-13-comparison-common-misconceptions.webp

## Asset 14: 和相邻机制的分工

- Position: 相邻机制
- Type: original framework knowledge-card illustration
- Purpose: 模型白名单、沙箱快照、默认模型和 SDK 起点的职责地图
- Source facts: 模型白名单；沙箱继承；agent-default-model；SDK 初始化；成本范围；权限范围
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-14-framework-adjacent-mechanisms.webp

## Asset 15: 确定性换来什么

- Position: 权衡与边界
- Type: original comparison knowledge-card illustration
- Purpose: 左侧可审计、可复现、确定性；右侧冻结授权、降低灵活性、需用户确认
- Source facts: 可审计；可复现；确定性；授权冻结；换路由要改设置；用户确认
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-15-comparison-tradeoff-ledger.webp

## Asset 16: pre-1.0 的边界

- Position: 结尾边界
- Type: original timeline knowledge-card illustration
- Purpose: 当前三层稳定结构与可能变化的字段、事件名、未来可见审批通道
- Source facts: v0.1.3-alpha.1；pre-1.0；字段可能调整；授权进日志；校验贴近副作用；未来可见审批
- Capture/generation method: GPT Image 2
- Copyright boundary: original generated conceptual visual; no claimed product screenshot or test evidence
- Output file: imgs/55-16-timeline-future-boundary.webp
