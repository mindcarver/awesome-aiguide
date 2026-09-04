---
illustration_id: 40-12
type: flowchart
style: notion
generator: gpt-image-2
---

ZONES: Runtime explicit-config gate; SDK condition checks; injected default cordis.yml path; override disables injection.
LABELS: 配置注入不是回退; DSH_CORDIS_CONFIG; argv config; default cordis.yml; SDK wrapper; runtime_bin override; no config = exit; 显式传递，责任分层
COLORS: White background, near-black lines, pastel blue, soft yellow, pale pink.
STYLE: GPT Image 2 Notion-style configuration gate flowchart, exact labels.
ASPECT: 16:9
TAKEAWAY: runtime 从不回退，SDK 只在严格条件下显式注入。
