---
illustration_id: 43-15
type: flowchart
style: notion
generator: gpt-image-2
---

ZONES: 100000 reasoning-delta events accumulate in session then markFrameDirty limits React publication to one frame.
LABELS: web-stress 十万分片; 100,000 reasoning-delta; acceptLiveEvent; markFrameDirty; requestAnimationFrame; React once per frame; 250ms; 不丢原始事件，不减慢生产方
COLORS: White, near-black, pastel blue, soft yellow, pale pink.
STYLE: Notion-style streaming flowchart, exact readable labels. Do not render prompt field names such as ZONES, LABELS, COLORS, STYLE, ASPECT or explanatory English sentences; only render the supplied labels and Chinese takeaway.
ASPECT: 16:9
TAKEAWAY: 性能边界在会话接收与 React 发布之间。
