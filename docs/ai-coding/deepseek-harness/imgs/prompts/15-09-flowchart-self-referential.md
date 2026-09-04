---
illustration_id: 15-09
type: flowchart
style: notion
generator: gpt-image-2
---

agent 改自己的插件树 - Self-referential Flow

Layout: circular two-step process. Current agent step invokes cordis_run, dynamic package registers a new tool into the plugin tree, then next step assembles a changed tool list for the same agent.
STEPS: 当前 step; cordis_run; 动态包; 注册新工具; 下一个 step; 新工具列表.
CONNECTIONS: cycle arrow back to agent, thin memory boundary around process labelled 进程内存.
LABELS: Title: agent 改自己的插件树. Labels: 当前 step, 动态包, 注册新工具, 下一个 step, 进程内存. Takeaway: 能加不能拆，不跨重启.
COLORS: white background, blue agent cards, yellow plugin tree, pink boundary accent, black lines.
STYLE: GPT Image 2 Notion-like educational flowchart, exact readable Chinese text, no pseudo-text, watermark, brands, people, robot, or dark background.
ASPECT: 16:9.
