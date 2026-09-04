---
article: 32-mcp-in-dsh-and-mcp-memory.md
type: mixed
density: rich
style: notion
palette: default
image_count: 14
---

# MCP 客户端与记忆服务器配图大纲

1. MCP 把 N 乘 M 的适配变成 N 加 M。
2. 一个桥接插件实例对应一个外部服务器。
3. 公开工具名是稳定的确定性纯函数。
4. tools/list 分页发现与两阶段代际切换。
5. 同步队列和 isCurrent 栅栏避免代际交错。
6. 指数退避、稳定窗口与重连预算。
7. MCP 内容解析、图片双条件和规范值边界。
8. stdio 环境清洗与 HTTP 凭证边界。
9. 一次 mcp 工具调用的完整往返。
10. 保旧与归零两条代际失败时间线。
11. stdio 与 streamable-http 的信任账。
12. mcp-memory 三份钉死版本配置。
13. 跨会话写入、检索、使用的验证闭环。
14. MCP 桥接的边界与工程纪律总结。
