---
illustration_id: 40-14
type: flowchart
style: notion
generator: gpt-image-2
---

ZONES: Client and dsh runtime connected by protected stdout pipe; stderr diagnostics lane; invalid print contaminates protocol; protocol checklist.
LABELS: JSON-RPC stdio 管道纪律; stdout = protocol; stderr = diagnostics; print breaks stream; turn/end; reason.kind; SdkProtocolError; 协议流里不能混日志
COLORS: White background, near-black lines, pastel blue, soft yellow, pale pink.
STYLE: GPT Image 2 Notion-style protocol flowchart with clear red warning, exact labels. The right endpoint is dsh runtime, never MCP server. Do not show MCP, tools, resources, or prompts.
ASPECT: 16:9
TAKEAWAY: stdout 归协议，任何调试输出都应去 stderr。
