# 别为了多 Agent 而多 Agent：LangGraph Supervisor、Swarm 与放弃的时刻

**TL;DR:** LangGraph 的多 agent 有两种主流拓扑：supervisor（中心调度，子 agent 当工具用）和 swarm（对等移交，谁接手谁主导）。先说结论：多 agent 不是能力升级，是用延迟和成本换上下文隔离。需要「不同工具权限、不同上下文、不同角色」三者之一才值得上；为了架构图好看而拆的多 agent，结果几乎都是更慢更贵更难调。

## 两种拓扑长什么样

**Supervisor：hub-and-spoke。** 一个调度 agent 接收任务，决定派给哪个子 agent，收结果、再调度，直到完成。子 agent 之间不直接说话。适合任务可以清晰分解、有主次之分的场景，也是大多数人想象中的「多 agent」。

```python
# pip install langgraph-supervisor
from langchain.agents import create_agent
from langgraph_supervisor import create_supervisor

researcher = create_agent(
    model, tools=[web_search],
    system_prompt="你只负责检索和整理资料",
)
writer = create_agent(
    model, tools=[write_file],
    system_prompt="你只负责根据资料写稿",
)

supervisor = create_supervisor(
    agents=[researcher, writer],
    model=model,
    prompt="把任务分给合适的助手，汇总结果",
).compile()

supervisor.invoke({"messages": [("user", "调研 LangGraph 并写一篇短文")]})
```

**Swarm：对等移交。** 没有 boss，每个 agent 处理完自己的部分后，通过 `Command(goto=..., graph=Command.PARENT)` 把控制权直接移交给下一个。适合流程天然分段、每段的专业工具完全不同的场景（客服系统里「售前 → 下单 → 退款」各归各的专家）。官方维护的 `langgraph-swarm` 库封装了这个模式。

选型的直觉：任务像「项目」用 supervisor，任务像「接力」用 swarm。拿不准就 supervisor，中心调度的可观测性和可控性都更好。

## 成本账：多 agent 的代价写在调用手里

每多一个 agent 参与，就是多一轮完整的 LLM 调用，外加把必要上下文复述给它。supervisor 模式下还有一份持续的调度开销：每个子 agent 返回后，supervisor 都要重新看一遍全局状态再决定下一步。同样一个任务，单 agent 3 次调用能完成的，supervisor 加 writer 的结构轻松翻到 8 到 10 次，token 账单按倍数涨，延迟按串行链路最长的分支算。

社区做过不少可靠性对比试验，一个被反复复现的结论是：单 agent 和多 agent 在任务成功率上并无稳定显著差异。可靠性不构成上多 agent 的理由，这条很反直觉，但和上面那笔账放在一起看很合理：你多花的钱买的是隔离和权限边界，不是更高的成功率。

## 值得上多 agent 的三个信号

1. **权限必须分开。** 能读代码的 agent 不能碰生产配置，能发邮件的 agent 不能动数据库。权限隔离是多 agent 最硬的理由，工具集不同，拆开是唯一干净的做法。
2. **上下文真的塞不下。** 子任务的中间产物（几十万 token 的检索结果）会污染主流程的上下文，拆出去让它在自己的 state 里消化，只回传结论。
3. **角色间需要交叉验证。** 研究员和批评者、作者和审稿人，相互独立的视角有真实价值时。

三个信号都没有，就一个 agent 配几组工具。还有一个折中多数人不知道：工具本身可以藏复杂度，一个「search_and_summarize」工具内部干再多事，agent 眼里只是一次调用。很多「需要多 agent」的判断，实际是「需要一个更厚的工具」。

## 放弃多 agent 的时刻

出现这些情况，拆开的结构就是在亏损：子 agent 之间传的是「半成品上下文」，谁都得把别人的结论重新解释一遍；supervisor 频繁派错任务，人工看着调度来回折腾；调试时一次失败要翻三个 agent 的轨迹。遇到这些，合回去。图结构改起来不贵，僵在错误结构里的运维才贵。

## 权衡

多 agent 在 LangGraph 里没有一等公民抽象（1.x 的 multi-agent 文档并进了 LangChain 的 agent 章节），`langgraph-supervisor` 和 `langgraph-swarm` 是两个独立维护的封装库，选型时要看它们和你的 LangGraph 版本的匹配度。也可以不依赖这两个库，自己用 Command 移交搭，代码量不大，控制力最强，适合拓扑特殊的团队。

## 延伸阅读

- [Workflows and agents](https://docs.langchain.com/oss/python/langgraph/workflows-agents.md)：官方对编排模式的定位
- [langgraph-supervisor](https://github.com/langchain-ai/langgraph-supervisor)：supervisor 模式封装库
- [LangGraph 仓库](https://github.com/langchain-ai/langgraph)：multi-agent 相关 issue 与讨论
