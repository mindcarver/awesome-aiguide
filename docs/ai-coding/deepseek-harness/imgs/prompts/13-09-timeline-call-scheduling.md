---
illustration_id: 13-09
type: timeline
style: notion
generator: gpt-image-2
---

四个调用如何排队 - 调度时序图

DIRECTION: left-to-right Gantt-style timeline on four horizontal rows.
EVENTS: 读文件 A and 读文件 B start together and overlap; 写文件 A starts only after both finish and occupies the lane alone; 读文件 C starts after the write. Bottom row submits results in A, B, 写 A, C order.
MARKERS: blue parallel bars for reads, single pink exclusive bar for write, a small queue at bottom.
LABELS: Title: 四个调用如何排队. Labels: 读文件 A, 读文件 B, 写文件 A, 读文件 C, 独占. Takeaway: 写操作排干并发后单独运行.
COLORS: white background, pastel blue reads, pastel pink write, pastel yellow queue, black outlines.
STYLE: GPT Image 2 polished Notion-like hand-drawn timeline with clean spacing and modular cards. Exact legible Chinese labels only; no pseudo-text, watermark, brands, people, robot, or dark background.
ASPECT: 16:9, productivity-visual.
