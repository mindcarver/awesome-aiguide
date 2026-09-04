---
illustration_id: 42-06
type: framework
style: notion
generator: gpt-image-2
---

ZONES: Dispatcher loop with three listeners; one throws into isolated log; invariant arrow rethrows after loop.
LABELS: 回调异常在 dispatcher 隔离; listener A; listener B throws; log warning; next listener; INVARIANT; rethrow; 一个坏订阅者不饿死后续监听器
COLORS: White, near-black, pastel blue, soft yellow, pale pink.
STYLE: Notion-style event framework, exact readable labels.
ASPECT: 16:9
TAKEAWAY: 普通回调失败隔离，不变量违规必须穿透。
