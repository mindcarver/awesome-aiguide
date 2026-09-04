---
illustration_id: 13-08
type: flowchart
style: notion
generator: gpt-image-2
---

策略有序，执行可以重叠 - 三段并发图

Layout: three vertical swimlanes read left to right: policy decisions, execution overlap, ordered result submission. Four call tokens traverse the lanes.
STEPS: 提交顺序; 策略串行; 执行并发; 结果有序; 模型历史稳定.
CONNECTIONS: in lane one, tokens pass sequentially; lane two has overlapping horizontal bars; lane three queues results in original order even if completion order differs.
LABELS: Title: 策略有序，执行可以重叠. Labels: 提交顺序, 策略串行, 执行并发, 结果有序, 模型历史稳定. Takeaway: 快结果宁可排队也不乱序.
COLORS: white background, pastel yellow policy lane, pastel blue execution lane, pastel pink result queue accents, near-black outlines.
STYLE: GPT Image 2 Notion-style hand-drawn flowchart, modular cards, abundant white space. Exact readable Chinese labels; no fake text, watermark, brands, people, robot, or dark background.
ASPECT: 16:9, infographic-diagram.
