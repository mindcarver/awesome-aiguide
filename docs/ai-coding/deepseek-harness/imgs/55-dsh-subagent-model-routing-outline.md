---
type: mixed
density: rich
style: notion
generator: gpt-image-2
image_count: 16
language: zh
---

## Illustration 01

**Position**: 开头约束段
**Purpose**: 后台子代理没有可见审批界面，因此将审批策略固定为 never，决定全部前置
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「子代理为什么不许问」；标签「子代理」「审批 never」「隐形阻塞」「确定性拒绝」「权限快照」「不重试」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-01-framework-approval-never-constraint.webp

## Illustration 02

**Position**: 三十秒模型
**Purpose**: 授权、快照、执行三层加选择层传送带的整体模型
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「子代理模型路由三层」；标签「授权层」「快照层」「选择层」「执行层」「精确白名单」「调用前复核」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-02-framework-three-layer-routing.webp

## Illustration 03

**Position**: 授权层小节
**Purpose**: Host 设置中逐条授权 provider 加 model 对，拒绝前缀和通配扩权
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「授权层：精确白名单」；标签「Host 设置」「provider + model」「精确匹配」「无通配符」「用户同意」「新适配器不扩权」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-03-framework-authorization-whitelist.webp

## Illustration 04

**Position**: 授权层设置界面段
**Purpose**: 开关与路由表一次版本栅栏提交，禁用保留列表、启用至少一条
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「设置保存的原子性」；标签「开关」「路由表」「原子提交」「版本栅栏」「禁用保留」「启用至少一条」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-04-flowchart-atomic-settings-commit.webp

## Illustration 05

**Position**: 快照层小节
**Purpose**: 新会话将授权表写成持久 subagent/model-selection-policy 事件，子会话继承、旧会话恢复旧表
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「快照层：会话事实」；标签「新会话」「持久事件」「子会话继承」「恢复旧表」「JSONL 日志」「历史可复现」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-05-flowchart-session-policy-snapshot.webp

## Illustration 06

**Position**: 选择层小节
**Purpose**: 静态默认、部署配置、调用参数三来源按优先级合并，provider 和 model 成对出现
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「选择层：参数合并」；标签「静态默认」「配置覆盖」「调用参数」「provider + model」「effort 清除」「新模型默认」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-06-flowchart-selection-parameter-precedence.webp

## Illustration 07

**Position**: 选择层发现工具段
**Purpose**: list_subagent_models 只展示授权快照与活适配器目录的交集
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「发现工具的交集」；标签「list_subagent_models」「授权快照」「活目录」「交集」「目录咨询性」「授权硬闸」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-07-framework-discovery-authorized-intersection.webp

## Illustration 08

**Position**: 执行层小节
**Purpose**: 执行器在创建子代理前独立复核白名单，并处理取消与 HMR 适配器变更
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「执行层：贴着副作用复核」；标签「工具参数」「独立复核」「取消检查」「provider 仍注册」「HMR」「创建子代理前」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-08-flowchart-executor-defensive-recheck.webp

## Illustration 09

**Position**: 能力门小节
**Purpose**: 支持动态选择的 spawn fork SDK 与明确拒绝的 ACP Codex Claude Code 两列对比
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「能力门：支持与拒绝」；标签「spawn」「fork」「DSH SDK」「ACP」「Codex」「Claude Code」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-09-comparison-capability-gate-providers.webp

## Illustration 10

**Position**: 能力门解释段
**Purpose**: 部署时 provider 固定 model 字段与委派参数运行时选择的区别
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「固定模型不等于动态选择」；标签「固定 model」「部署常量」「动态选择」「委派参数」「挂载失败」「不静默降级」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-10-comparison-fixed-versus-dynamic-model.webp

## Illustration 11

**Position**: fork 小节
**Purpose**: 复制前缀可复用 KV 缓存；换 provider/model 导致完整重算，节省被吞掉
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「fork 为什么禁用选择」；标签「subagent_fork」「已完成前缀」「KV 缓存」「换模型失效」「重算成本」「禁用选择」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-11-framework-fork-kv-cache-economics.webp

## Illustration 12

**Position**: 贯穿案例
**Purpose**: 从周一授权到周五恢复的时间线，包含越界拒绝与新会话生效
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「运维台的一周」；标签「周一授权」「周二快照」「周三拒绝」「周四扩表」「周五恢复」「旧会话不热更新」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-12-timeline-ops-week-routing-story.webp

## Illustration 13

**Position**: 常见误解
**Purpose**: 并排校正授权范围、快照热更新、固定模型、通配授权和开关语义
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「五个常见误解」；标签「只管子代理」「快照不热更新」「固定不等于动态」「无通配」「关闭保留列表」「精确授权」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-13-comparison-common-misconceptions.webp

## Illustration 14

**Position**: 相邻机制
**Purpose**: 模型白名单、沙箱快照、默认模型和 SDK 起点的职责地图
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「和相邻机制的分工」；标签「模型白名单」「沙箱继承」「agent-default-model」「SDK 初始化」「成本范围」「权限范围」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-14-framework-adjacent-mechanisms.webp

## Illustration 15

**Position**: 权衡与边界
**Purpose**: 左侧可审计、可复现、确定性；右侧冻结授权、降低灵活性、需用户确认
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「确定性换来什么」；标签「可审计」「可复现」「确定性」「授权冻结」「换路由要改设置」「用户确认」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-15-comparison-tradeoff-ledger.webp

## Illustration 16

**Position**: 结尾边界
**Purpose**: 当前三层稳定结构与可能变化的字段、事件名、未来可见审批通道
**Visual Content**: Notion-style knowledge cards and hand-drawn connectors.
**Text Plan**: 标题「pre-1.0 的边界」；标签「v0.1.3-alpha.1」「pre-1.0」「字段可能调整」「授权进日志」「校验贴近副作用」「未来可见审批」；提示「按授权边界路由，不把生成图当证据」。
**Filename**: 55-16-timeline-future-boundary.webp
