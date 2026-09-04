---
article: 24-jobs-and-workflow-ralph.md
type: mixed
density: rich
style: notion
palette: default
image_count: 18
generator: gpt-image-2
---

# Jobs、Workflow 与 Ralph 插图大纲

1. 两类长活：Jobs 管生命周期，Workflow 管编排逻辑。
2. 可预测 JobId 与 owner 授权边界。
3. JobStart 的 preflight 与不可失败 commit。
4. first-wins 结算与完成最后宣布的次序。
5. 繁忙注入、空闲 followup 唤醒。
6. maxConsecutiveWakes=3 与用户消息回充。
7. controller、并发上限与 owner 清理。
8. list/read/kill 三工具与避免轮询。
9. workflow 的 meta 是数据，不是代码。
10. result 永不 reject，取消有界。
11. fatal 错误与子项 null 的严格分流。
12. 脚本 realm 与宿主 JSON 的物化边界。
13. workflow 事件给快照，不给控制权。
14. workflow 历史以四类日志事件与合法前缀持久化。
15. 结构化子 agent 的 schema 提交硬约束。
16. Ralph 只传不可变目标、工作区与有界交接。
17. Ralph 轮次上限 256、全新 agent 与终端标签。
18. Jobs、Workflow、Ralph 的三层总结。
