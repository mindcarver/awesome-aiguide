---
illustration_id: 42-11
type: flowchart
style: notion
generator: gpt-image-2
---

ZONES: Model request failure feeds waterfall; retry listener; numbered attempts with exponential delay; final error path.
LABELS: agent/request-error waterfall; EMPTY_RESPONSE; RATE_LIMIT; SERVER_ERROR; TIMEOUT; TRANSPORT; 5 retries; 500ms → 10s; 没人认领才是终态
COLORS: White, near-black, pastel blue, soft yellow, pale pink.
STYLE: Notion-style retry process diagram, exact readable labels.
ASPECT: 16:9
TAKEAWAY: 可插拔策略在同一份持久历史上开启新轮次。
