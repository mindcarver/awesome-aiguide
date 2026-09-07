# LangGraph 生产实战系列

> 不是第 N 个入门教程，是中文第一套 LangGraph 生产向系列。20 篇文章从「要不要用」的判断线开始，经引擎心智模型、人审与多 agent 控制，一路写到部署成本、事故复盘和框架选型。连载中，按批次发布。

本系列面向已经能调 LLM API、写过简单 agent，现在要把系统推上生产的后端与 AI 应用开发者。它不重复讲「StateGraph 怎么定义节点」，而是回答一个具体问题：**把 LangGraph 用在生产上，哪些坑是绕不过去的、代价是什么、什么情况下不该用它。**

## 系列定位

- **生产向**：入门 demo 中文已经严重过剩，本系列只写官方文档没讲透、社区踩过坑的部分。
- **时效性**：全部内容基于 LangGraph 1.x（`create_react_agent` 已废弃、LangGraph Platform 已改名 LangSmith Deployment），并对网上大量 0.x 内容给出迁移路径。
- **有判断**：每篇给出「什么场景不该这么做」，不做框架软文。第一篇就劝退一部分读者。

## 与本仓库其他内容的关系

- [智能体设计模式系列](../agent-design-patterns/README.md)：讲模式本身（ReAct、supervisor、反思），本系列讲 LangGraph 这个框架怎么落地这些模式。
- [Agent 与工作流教程](../ai-app-tutorials/agent-workflow/)：概念层（workflow 基础、多 agent 架构、HITL），本系列是框架层的深入。
- [Agentic RAG](../ai-app-tutorials/rag/agentic-rag.md)：已含一段 LangGraph StateGraph 实现，本系列不重复 RAG 专题。

## 阅读路径

只关心「用不用」：读 #1、#19。
要上生产：按 #1-#4 打地基，#6（人审）和 #13（checkpoint 运维）是必读核心。
带团队选型：#1、#9、#17、#19。

## 系列目录

> 共 20 篇，连载中。已发布篇目带链接，其余为规划条目。

### 第一部分 · 地基

| # | 文章 | 重点 | 状态 |
|---|------|------|------|
| 01 | [先问一句：你的 Agent 真的需要 LangGraph 吗](./01-do-you-really-need-langgraph.md) | 引擎定位、v1 分层倒转、「不该用」判断线 | 已发布 |
| 02 | 读懂 Reducer，才算读懂 LangGraph：节点、状态与增量更新 | superstep、reducer、Overwrite、recursion_limit | 规划中 |
| 03 | LangGraph 持久化拆解：Checkpointer 到底帮你存了什么 | thread_id、快照边界、replay/fork、durability | 规划中 |
| 04 | Agent 越聊越贵：LangGraph 三套记忆的边界与 token 账单 | checkpointer vs Store vs messages 管理 | 规划中 |
| 05 | 不画图写 LangGraph：@entrypoint、@task 与两种范式的选择 | 函数式 API、replay 恢复、选型 | 规划中 |

### 第二部分 · 控制

| # | 文章 | 重点 | 状态 |
|---|------|------|------|
| 06 | 人审不是加个按钮：LangGraph interrupt 落地的六个坑 | interrupt/Command(resume)、幂等、超时兜底 | 规划中 |
| 07 | 子图一嵌，状态就丢？LangGraph Subgraph 的隔离与透传 | checkpoint namespace、streaming 不透传 | 规划中 |
| 08 | 条件边返回 50 个 Send 之后：LangGraph 并行分支与状态合并 | Send、map-reduce、合并秩序 | 规划中 |
| 09 | 别为了多 Agent 而多 Agent：LangGraph Supervisor、Swarm 与放弃的时刻 | 多 agent 模式选型与量化对比 | 规划中 |
| 10 | 前端拿不到 token？LangGraph Streaming 七种模式与前端联调 | stream_mode、v2 StreamPart、useStream | 规划中 |
| 11 | 从 create_agent 到 deepagents：LangGraph 高层抽象省了什么、藏了什么 | middleware、agent harness、降级到底层 | 规划中 |

### 第三部分 · 生产

| # | 文章 | 重点 | 状态 |
|---|------|------|------|
| 12 | Agent 跑一半挂了：LangGraph 错误分类、重试与补偿设计 | 四类错误、RetryPolicy、error_handler | 规划中 |
| 13 | LangGraph Checkpoint 不是免费的：一次生产事故复盘 | 表膨胀、TTL、schema 迁移、DeltaChannel | 规划中 |
| 14 | 循环图为什么难调试：LangGraph 可观测与测试实战 | tracing 模型、节点单测、Studio | 规划中 |
| 15 | 没有评测的 LangGraph Agent 只是 demo：评测与回归实战 | 轨迹评测、replay 回归、CI | 规划中 |
| 16 | 工具一多，权限就是产品：LangGraph Agent 的安全边界 | 权限分级、tool 审批、secrets 不进 state | 规划中 |
| 17 | 自部署还是买平台：LangGraph 部署形态与成本账 | 开源自部署、Agent Server、定价 | 规划中 |

### 番外与收官

| # | 文章 | 重点 | 状态 |
|---|------|------|------|
| 18 | 旧教程全过期了：LangGraph 0.x → v1 迁移实录 | create_agent、middleware、迁移坑 | 规划中 |
| 19 | 从 LangGraph 迁去 Temporal 的人在想什么：框架选型的真实账 | 竞品对比、迁出案例、durable execution | 规划中 |
| 20 | 收官实战：用 LangGraph 搭一条能上线的人审流水线 | 贯穿项目、设计决策回链 | 规划中 |

## 取舍说明

- **不写入门 demo**。StateGraph 计数器、create_react_agent 跑通文、Supervisor demo 中文已经过剩，本系列只保留必要的最小代码量。
- **单 agent 与多 agent 的量化对比**只在 #9 做一次，其余篇目引用结论。
- **RAG、评测方法论、MCP 通识**不在此系列展开，链接到仓库对应专题。
