# 从 create_agent 到 deepagents：LangGraph 高层抽象省了什么、藏了什么

**TL;DR:** v1 之后 LangGraph 的分层是：底层引擎（StateGraph）→ LangChain 1.0 的 `create_agent`（预置 agent 循环 + middleware 扩展）→ deepagents（虚拟文件系统、子 agent、任务规划的完整 harness）。高层省掉的是重复劳动，藏掉的是控制点。判断标准就一条：你要改的行为，middleware 钩子够不够用；不够，老老实实下沉到图。

## 先看分层地图

```
┌─────────────────────────────────────────────┐
│  deepagents        agent harness，开箱即用    │
│  （虚拟文件系统 / 子 agent / write_todos）     │
├─────────────────────────────────────────────┤
│  create_agent + middleware   预置 agent 循环  │
│  （LangChain 1.0，可插拔改行为）               │
├─────────────────────────────────────────────┤
│  StateGraph / functional API  编排引擎本体     │
│  （LangGraph，一切的地基）                     │
└─────────────────────────────────────────────┘
```

这个分层是 2025 年 10 月 1.0 版本刚调整的，中文互联网大量内容还没跟上：`langgraph.prebuilt.create_react_agent` 已废弃，替代者 `create_agent` 不在 LangGraph 包里，在 `langchain.agents` 里。它和旧版的区别不是改名，是架构：agent 循环的行为全部通过 middleware 插拔，不再靠往函数里塞参数。

## create_agent + middleware：省掉的劳动

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    SummarizationMiddleware, HumanInTheLoopMiddleware,
)

agent = create_agent(
    model="openai:gpt-4o",
    tools=[search, send_email],
    system_prompt="你是客服助手，高风险操作先请示",
    middleware=[
        SummarizationMiddleware(max_tokens=4000),      # 上下文自动摘要
        HumanInTheLoopMiddleware(interrupt_on=["send_email"]),  # 工具级人审
    ],
)
```

middleware 提供的钩子覆盖了 agent 循环的各环节：改 system prompt（按状态动态生成）、过滤工具（按用户权限裁剪可见工具）、消息管理（摘要、裁剪）、工具执行前后拦截、输出校验。自己用 StateGraph 实现同样的效果，每个钩子都是一个节点加一堆条件边，几百行起步。对于「标准 ReAct 循环 + 少量定制」的应用，这一层的代码量优势是数量级的。

`HumanInTheLoopMiddleware` 值得单独说：它把人审从「你在节点里手写 interrupt」变成一行配置。底下的机制就是 interrupt 原语，坑（恢复重跑、幂等）一样存在，只是换了个地方踩。

## deepagents：藏起来的完整 harness

再往上是 deepagents（`create_deep_agent` 入口），它默认给你一套 agent harness 的完整件：

- 虚拟文件系统：agent 在隔离的文件空间里读写，可以配权限规则，不碰真实文件
- 子 agent：一个 `task` 工具，把子任务派给独立上下文的无状态子 agent，主上下文不被污染
- 任务规划：`write_todos` 工具，让 agent 自己维护任务清单
- 审批：`interrupt_on` 指定哪些工具执行前要人审
- 记忆：跨会话的文件式记忆（AGENTS.md 风格）

它解决的问题是「通用 agent 的默认形态」：检索、编码、多步任务这类 open-ended 工作流。代价是你接受了它对 agent 行为的全部默认设计。改它的内部循环比改 create_agent 更难，因为它藏的不只是钩子，是整个执行策略。

## 藏掉了什么：三个丢失的控制点

用高层抽象，下面这些东西从「你写的」变成「它决定的」：

1. **状态结构。** create_agent 的 state 是固定的消息流加少量元数据。你要结构化业务字段（订单号、审批链）挂在状态里，middleware 钩子够不着的地方就得绕。
2. **控制流。** 图的分支、并行、子图编排，在 create_agent 里没有位置。它的控制流是写死的 agent 循环。需要「先并行调研再汇合」的结构，它表达不了。
3. **恢复粒度。** 整个 agent 是图里一个节点的产物，checkpoint 的粒度由它内部决定，你想在特定业务步骤边界存档，做不到。

这三个点就是「什么时候下沉」的判据。命中任意一个，直接用 StateGraph 写；都可以接受，高层抽象随便用。

## 我的用法

新项目从 create_agent 起步，成本最低，middleware 能满足的定制（prompt 动态化、工具裁剪、摘要、人审）占实际需求的八成。遇到结构化状态或非标准控制流，下沉到 StateGraph，把 create_agent 当成图里的一个节点用（它编译出来就是一张可以嵌的图）。deepagents 留给真正 open-ended 的通用助手类需求，业务流程明确的系统用它是拿确定性问题换不确定的 harness 行为。

## 延伸阅读

- [LangChain 1.0 发布博客](https://www.langchain.com/blog/langchain-langgraph-1dot0)：分层调整的官方说明
- [Middleware 概览](https://docs.langchain.com/oss/python/langchain/middleware/overview)：钩子模型与执行顺序
- [Prebuilt middleware](https://docs.langchain.com/oss/python/langchain/middleware/built-in)：现成中间件清单
- [deepagents 仓库](https://github.com/langchain-ai/deepagents)：harness 的默认设计
