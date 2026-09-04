---
illustration_id: 20-08
type: architecture
style: notion
generator: gpt-image-2
---

Notion 风格分层图，标题 `read-before-edit 是策略，不是 provider 内置`。底层卡 `Provider: 原子读写编辑` 保持纯净；上层透明策略卡 `dsh-fs-observation-policy` 监听 `fs/*` 事件；顶层是编辑工具。画出策略层可以替换，provider 不变。标签：`事件`、`观察记录`、`写入前校验`。白底手绘风，无人物。
