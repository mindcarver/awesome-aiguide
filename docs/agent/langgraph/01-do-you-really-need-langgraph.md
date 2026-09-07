# 先问一句：你的 Agent 真的需要 LangGraph 吗

**TL;DR:** LangGraph 是面向长时运行、有状态 Agent 的编排引擎，不是 AI 应用的默认起点。选不选它，四个问题就够：任务要不要断点续跑、高风险动作要不要人审、要不要多角色编排、出问题要不要回放追责。四个都答否，几十行手写循环更干净；任何一个答是，它才开始划算。

## 先看一个不需要任何框架的 Agent

很多人在项目第一天就把 LangGraph 写进了 requirements.txt。我们先不加框架，直接用模型官方 SDK 写一个能自主调用工具的 agent：

```python
# pip install openai==1.102.0
# export OPENAI_API_KEY=sk-...
import json

from openai import OpenAI

client = OpenAI()
MODEL = "gpt-4o-mini"
MAX_STEPS = 8


def get_weather(city: str) -> str:
    return f"{city}：晴，26°C"  # 演示用假数据，替换成真实 API 即可


TOOLS = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "查询一个城市的天气",
        "parameters": {
            "type": "object",
            "properties": {"city": {"type": "string"}},
            "required": ["city"],
        },
    },
}]


def run(question: str) -> str:
    messages = [{"role": "user", "content": question}]
    for _ in range(MAX_STEPS):
        resp = client.chat.completions.create(
            model=MODEL, messages=messages, tools=TOOLS
        )
        msg = resp.choices[0].message
        messages.append(msg)
        if not msg.tool_calls:
            return msg.content or ""
        for call in msg.tool_calls:
            args = json.loads(call.function.arguments)
            result = get_weather(**args)
            messages.append({
                "role": "tool",
                "tool_call_id": call.id,
                "content": result,
            })
    return "达到最大步数，停止"


print(run("北京和上海今天哪个更热？"))
```

大约 40 行，这是一个完整的 agent：模型自己决定什么时候调工具，工具结果拼回上下文，循环到给出最终答案。换模型、加工具、改步数上限，它都能适应。相当一部分"智能体需求"，到这一步就已经完成了。

## 三个时刻，才轮到框架

手写循环的舒适区会被三个时刻打破。

**时刻一：任务跑到一半，进程没了。** 审批流挂了三天，重启后用户希望从第三步继续，而不是从头再来。这意味着每一步之后都要有快照，快照还要能恢复。

**时刻二：执行到高风险动作，需要有人点头。** 转账、删数据、群发邮件之前停下来等人确认，确认完从暂停点继续，而不是重跑整个流程。

**时刻三：出了问题，要能回到历史某一步看现场。** 客户说上周三那次跑错了参数，你需要重放那次的执行路径。

三个时刻对应的正是持久化、human-in-the-loop、时间旅行。这三件事自己实现都要踩很多坑（快照粒度、恢复语义、幂等），LangGraph 把它们做成了标准能力。注意前提：是这三个时刻真的出现了才需要，不是"以后可能有"就提前上。

## LangGraph 到底是什么

截至 2026 年 9 月，LangGraph 最新版本是 1.2.11，1.0 发布于 2025 年 10 月，官方承诺 1.x 内不做破坏性变更。项目 README 的自我定位很直白：low-level orchestration framework for building stateful agents，面向长时运行、有状态 agent 的低层编排框架。

1.0 还带来一个容易被忽视的结构变化：旧的 `langgraph.prebuilt.create_react_agent` 被废弃，高层封装 `create_agent` 移到了 `langchain.agents`，而它运行在 LangGraph 的运行时之上，再往上是 deepagents 这类 agent harness。分层倒过来了：LangGraph 不再是"带图 API 的全家桶"，而是底下那台引擎，上面可以套不同的壳。

它给你的核心能力就四件：

1. 快照：图执行每跨过一个边界就存一次状态，挂在某个 thread_id 下。
2. 叫停与恢复：`interrupt()` 在任意位置暂停存档，人处理完用 `Command(resume=...)` 从断点继续。
3. 回放：拿着历史 checkpoint 重跑或分叉，调试和追责都靠它。
4. 编排：一个图里挂多个有独立状态的子角色，supervisor 或对等移交都行。

Uber、LinkedIn、Klarna、Replit 都在生产使用它。这是 LangChain 官方口径，参考时记得打折。

## 一条判断线：四个问题

把三个时刻加一条编排需求，就是完整的判断线。对着你的项目问：

1. 任务会不会跑到一半中断，之后需要从断点继续？
2. 有没有高风险动作需要在执行前经过人工确认？
3. 是否真的需要多个角色分工，而不是一个配了好工具的单角色能解决？
4. 出了问题，是否需要回放历史执行路径来定位或追责？

任何一个答"是"，LangGraph 值得认真评估。四个全答"否"，手写循环、裸 SDK 或者 Pydantic AI 这类轻量封装是更便宜的选择。

灰色地带也存在。比如你要的其实只是 durable execution（任务在机器重启后自动恢复），并不需要 agent 语义和图编排，那 Temporal 这类工作流引擎比 LangGraph 更对口。下面的案例部分有一个现成的迁移故事。

## 用它的代价：即使四个问题都答"是"

这是官方文档不会放在首页的部分。

**调试体验会变差。** 循环图和主流 APM 的 span 树模型对不上，报错堆栈经常在节点边界断掉。社区里"没有 LangSmith 就活不下去"的说法反复出现，而 LangSmith 收费，开源替代都得自己搭。

**多一个有状态组件要运维。** checkpointer 要接真数据库（生产禁用内存版），checkpoint 表持续膨胀需要清理策略，state schema 演进没有官方版本化迁移工具。2025 年还出过 patch 版本夹带破坏性变更的事故：`langgraph-checkpoint-postgres` 从 2.0.21 升到 2.0.22，metadata 序列化被打挂，跟着升级的服务直接报错。

**学习成本在抽象层。** reducer、superstep、Command 这些概念要先建立心智模型，才能写出不出错的代码。state 里放什么、什么时候返回增量、并行分支怎么合并，都是新坑。

**教程时效风险。** 0.x 时代的中文教程和翻译站大量存在且已过期，照着旧教程写会直接用到废弃 API。

## 三个真实案例：两正一反

**Uber（正面）。** 面向 5000 多名内部工程师的 AI 开发工具，用 LangGraph 编排代码修复、单测生成等专用 agent，官方口径称节省 21000 多工程师小时。工具型、流程长、需要审计，这正是判断线全答"是"的场景。

**Klarna（正反两面）。** 客服 assistant 上线首月处理 230 万次对话，消化约三分之二的客服会话，解决时长下降约 80%。这些数字被广泛引用，常被省略的是后半句：Klarna 后来公开承认过度自动化损害了服务质量，重新招回了人工客服。框架没有问题，问题在于把"能自动化"当成了"该自动化"。

**Grid Dynamics（反面）。** 给一家财富 500 制造商做深度研究 agent，技术栈是 LangGraph 加 Redis、Kafka。他们踩到状态过期 bug 难以复现、人审等待需要自建重试、Kafka 扩容竞态，最后整体迁到 Temporal，删掉了数千行自建的重试和错误处理代码。关键结论不是"LangGraph 不行"，而是他们要的核心其实是 durable execution：声明式重试、无状态 worker、自动恢复。checkpoint 提供的是快照存取，不等于这套执行保障。

还有一个常被引用的弃用故事：Octomind 写过《Why we no longer use LangChain》，核心论点是框架在需求变具体之后就崩了，Hacker News 上有几百条讨论，LangChain 的 CEO 也亲自回应。公平起见，反方声音同样真实：不少从零手写的团队，在重试、断点恢复、并发控制上写出几千行补丁之后，又回到了框架。两边都是真实经验，区别只在你的任务落在判断线哪一侧。

## 替代品速查

| 你的情况 | 更顺手的起点 |
| --- | --- |
| 单趟任务、简单线性链 | 裸 SDK，手写循环 |
| 只需要长任务断点恢复，不需要图 | Temporal、Restate 这类 durable execution 引擎 |
| 想要开箱即用的编码 agent | Claude Code、Claude Agent SDK |
| 原型期多角色协作，先跑通再说 | CrewAI |
| 要 agent 但想避开重抽象 | Pydantic AI、smolagents |

这份表不是排名，每一行的成立条件就是左边那句话。

## 结论

回到标题的问题。我的建议按顺序：

1. 先手写。100 行以内的循环加两三个工具，能覆盖大多数需求，还能让你真正理解 agent 在干什么。
2. 撞到判断线里的任何一个"是"（断点续跑、人审、多角色、回放），再认真评估 LangGraph，并且接受它的运维成本。
3. 如果发现要的其实只是任务恢复而不是编排，去看 Temporal 一类的工作流引擎，别拿 agent 框架硬凑。

提前给项目装上 LangGraph 不算错误，但抽象税从第一天就开始付，收益要等你真的撞上那三个时刻才兑现。

## 延伸阅读

- [LangGraph 官方文档](https://docs.langchain.com/oss/python/langgraph/)：概念与 API 的权威来源，旧域名 langchain-ai.github.io 已停止更新
- [LangChain 1.0 发布博客](https://www.langchain.com/blog/langchain-langgraph-1dot0)：分层变化的官方说明
- [Octomind: Why we no longer use LangChain 与 Hacker News 讨论](https://news.ycombinator.com/item?id=40739982)：弃用叙事与官方回应
- [Grid Dynamics 迁移 Temporal 案例实录](https://temporal.io/blog/prototype-to-prod-ready-agentic-ai-grid-dynamics)：checkpoint 与 durable execution 边界的一手素材
- [Hacker News: Agent design is still hard](https://news.ycombinator.com/item?id=46013935)：手写路线的复杂度反噬
