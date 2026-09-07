# 从 LangGraph 迁去 Temporal 的人在想什么：框架选型的真实账

**TL;DR:** Grid Dynamics 把一个 LangGraph 深度研究 agent 整体迁到了 Temporal，删掉数千行自建的重试和恢复代码。这个案例的答案不是「LangGraph 不行」，而是他们的系统真正需要的是 durable execution，checkpoint 给不了。选型的正确姿势由此推出：先分清你要的是编排语义还是执行保障，再谈框架；可靠性横评证明各框架差异不大，真正该比的是开发体验、运维成本和退出成本。

## 一个迁出案例的完整拆解

Grid Dynamics 给一家财富 500 制造商做深度研究 agent（多步检索、交叉验证、长流程），技术栈是 LangGraph 加 Redis 做状态、Kafka 做分发。上线过程中撞到三个问题：

1. **状态过期 bug 难以复现。** agent 拿着 Redis 里的状态继续执行时，外部世界的状态（工单、数据快照）已经变了，bug 出现的时机随机，复现要重建当时的全部环境。
2. **人审等待靠自建重试硬扛。** 深度研究流程里有人审环节，等待几小时是常态。LangGraph 的 interrupt 语义是「存档后停」，恢复靠外部再调一次；谁来调、调不到怎么办（服务重启、任务丢失），全部自己写，结果是一套手写的重试与补偿代码。
3. **Kafka 扩容竞态。** 分发层的扩容让同一任务可能被重复投递，下游没有幂等保护，偶发重复执行。

迁移到 Temporal 后：工作流代码变成声明式的（RetryPolicy 配置化），worker 无状态可随意扩缩，人审等待用运行时原生的定时器表达，上面那套手写重试删掉数千行。他们的结论值得原样记住：要的不是「图编排」，是长流程的执行保障。

## checkpoint 和 durable execution 的边界

这两个词经常被混着用，语义差异是这次选型问题的根：

| 能力 | LangGraph checkpoint | Temporal 类 durable execution |
| --- | --- | --- |
| 状态存取 | 每 superstep 快照，可回放 | 每活动步骤持久化，可恢复 |
| 重试 | 节点内 RetryPolicy，进程内 | 平台级，进程崩了换机器续跑 |
| 等待/定时 | interrupt 挂起，靠外部恢复恢复 | 定时器是运行时原语，等一周不占资源 |
| 失败语义 | 图内处理，图外自建 | 自动重投、幂等键、事件历史可查 |
| 心智模型 | 编排：流程是图 | 执行：流程是函数 |

选型判断浓缩成一句话：**流程的结构复杂度决定要不要图，流程的执行保障要求决定要不要 durable execution 引擎。** 两者都要的场景（多角色 + 跨天恢复）确实存在，那时是 LangGraph 嵌进 Temporal activity 里用，而不是二选一。

## 其余替代品的真实定位

社区讨论里反复出现的分工，去掉了厂商宣传的部分：

- **CrewAI**：角色抽象和上手速度是真实优势，规模化后控制粒度不足。原型期好用，精细流程会被迫下沉。
- **AutoGen / AG2**：多 agent 对话的交互设计最好，0.2 到 0.4 的重写加社区分裂让长期投入者犹豫。
- **OpenAI Agents SDK**：极简，单 agent 快速交付利器，生态绑定 OpenAI，缺持久化与人审。
- **Claude Agent SDK**：agentic coding 场景天生契合，循环是黑盒，可见性是交换来的代价。
- **smolagents / Pydantic AI**：反抽象路线，核心小、行为可预测，适合「要 agent 语义但不想扛框架」的团队。
- **手写循环**：50 行起步毫无负担，重试、恢复、并发的复杂度会随需求逐步还回来，Grid Dynamics 迁移前的状态就是终点形态。

## 选型该比什么

社区有团队做过大量同任务跨框架的可靠性对比，一个稳定结论是：任务成功率在成熟框架之间没有显著差异。这很反直觉，但想通了就释然：框架的执行核心都是「模型加工具循环」，差异在皮不在骨。真正拉开长期成本的是三件事：

1. **开发体验。** 改一个分支逻辑要动几处？调试要看几层数据？新人几天能产出？
2. **运维账单。** 自托管要养哪些组件？SaaS 的席位和用量怎么涨？
3. **退出成本。** 业务代码和框架 API 的耦合面积有多大？checkpoint 数据迁得走吗？

## 两个方向都是真的

公平地收尾：迁出 LangGraph 的故事是真的，迁进去的故事也是真的。Octomind 因为「需求变具体后框架崩」离开了 LangChain 系；同样多的团队在手写重试和恢复写到第三个月时回到了框架。Klarna 的案例则提醒第三种失败：框架选对了，把「能自动化」当成「该自动化」照样翻车。工具选型决定不了项目成败，它只决定你付成本的方式：抽象税、运维税，或者手写税。

## 延伸阅读

- [Grid Dynamics：从原型到生产级 agent（Temporal 案例）](https://temporal.io/blog/prototype-to-prod-ready-agentic-ai-grid-dynamics)：本文主要案例的一手记录
- [Checkpoints Are Not Durable Execution（Diagrid）](https://www.diagrid.io/blog/checkpoints-are-not-durable-execution-why-langgraph-crewai-google-adk-and-others-fall-short-for-production-agent-workflows)：语义边界的批判视角
- [Octomind: Why we no longer use LangChain（HN 讨论）](https://news.ycombinator.com/item?id=40739982)：迁出叙事与 LangChain CEO 回应
- [Hacker News: Agent design is still hard](https://news.ycombinator.com/item?id=46013935)：手写路线的复杂度反噬
