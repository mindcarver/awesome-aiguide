# LangGraph 持久化拆解：Checkpointer 到底帮你存了什么

**TL;DR:** checkpointer 在每个 superstep 边界给整张图的状态拍快照，按 thread_id 归档。它给你四件事：断点续跑、时间旅行、分叉、以及人审（interrupt）的底层支撑。同时它有明确边界：只存图状态，不存节点局部变量，快照粒度意味着节点级重跑，副作用必须自己保证幂等。

## 五分钟版本：快照怎么工作

给图挂一个 checkpointer，编译，然后用 thread_id 调用：

```python
from langgraph.graph import StateGraph, MessagesState, START
from langgraph.checkpoint.memory import InMemorySaver

def chatbot(state: MessagesState) -> dict:
    return {"messages": [("assistant", "收到：" + state["messages"][-1].content)]}

g = StateGraph(MessagesState)
g.add_node("chatbot", chatbot)
g.add_edge(START, "chatbot")
app = g.compile(checkpointer=InMemorySaver())

config = {"configurable": {"thread_id": "user-42"}}
app.invoke({"messages": [("user", "你好")]}, config)
app.invoke({"messages": [("user", "我刚才说了什么？")]}, config)

print(app.get_state(config).values["messages"])
# 两次调用的消息都在，第二次执行时模型看得到第一轮内容
```

第二次 `invoke` 没有传历史消息，模型却能「记得」第一轮，因为 checkpointer 按 thread_id 找到上次的快照，新输入被追加进去。对话记忆在最简形态下就是这么来的，不玄。

每跨过一个 superstep 边界，LangGraph 把当时的通道状态、待写入、下一步该跑哪些节点一起存下来。快照是完整的，不是增量日志，所以任何一张快照都足以从中间恢复。

## thread_id：一条会话的身份证

同一个编译好的图，用不同 thread_id 调用就是互不干扰的会话。生产里 thread_id 通常对应「一个用户的一个业务流程」，比如一个订单的审批会话。

两个实操细节：thread_id 长度上限 255 字符；`InMemorySaver` 只用于开发和测试，进程一重启数据全没，生产必须换 SqliteSaver、PostgresSaver 这类落盘实现，Postgres 版首次使用前要跑一次 `checkpointer.setup()` 建表。

## 四件事的用法

**恢复**：进程重启后，拿着同一个 thread_id 再 invoke，从最后一张快照继续。上面代码把两次 invoke 换成两次部署之间的两次调用，效果一样。

**回放**：快照历史可以列出来，每张都有 checkpoint_id 和当时待执行的节点：

```python
for snap in app.get_state_history(config):
    print(snap.config["configurable"]["checkpoint_id"], snap.next)
```

**时间旅行**：从历史快照重新执行。把 input 传 `None`、指定 checkpoint_id，图从那个点继续跑：

```python
states = list(app.get_state_history(config))
old = states[-1].config["configurable"]["checkpoint_id"]
app.invoke(None, {"configurable": {"thread_id": "user-42", "checkpoint_id": old}})
```

调试「上周三那次为什么跑错」靠的就是这个。

**分叉**：从历史快照出发换一个输入重新执行，生成一条新的 checkpoint 链，原历史不受影响。想做「同一个请求换个提示词再跑一遍，对比结果」时用它。

另外 0.2.34 之后持久化写入有 durability 三档：`exit`（默认，superstep 结束写）、`async`（异步写，崩溃可能丢最后一步）、`sync`（同步写，最稳最慢）。大多数场景默认档就够，金融类流程才需要动它。

## 它没存什么，这才是容易出事的

**节点局部变量不进快照。** 快照只包含图状态。节点函数里定义的中间变量、数据库连接、打开的文件句柄，恢复后都不在。节点必须设计成「从 state 出发、把结果写回 state」的纯函数风格，恢复才有意义。

**快照粒度是节点，副作用在里面。** 节点中途挂掉，恢复时整个节点重跑。节点里调了外部 API、发了邮件，恢复后就发两次。这条没有框架能替你解决，副作用要幂等，或者干脆把副作用挪出节点（比如放进队列，由带去重键的消费者执行）。用 `interrupt()` 做人审时同理：interrupt 之前的副作用要当作「可能已经发生过」来写。

**state 里有什么，快照里就有什么。** 你把 API key 临时塞进 state，它就跟着每一张快照落到数据库里。状态设计时先想一句：这些字段被持久化、被回放、被导出，我能接受吗。

## 权衡

checkpointer 引入了一个必须运维的有状态组件：数据库连接、表膨胀、备份都跟着来。开发期图省事用内存版，上生产忘了换，是最常见的翻车方式之一。反过来，如果整个应用只跑单轮调用、没有多步流程，checkpointer 带来的只有成本没有收益，别挂。

## 延伸阅读

- [Persistence 概念文档](https://docs.langchain.com/oss/python/langgraph/persistence)：threads、快照结构与 durability
- [Checkpointers 参考](https://docs.langchain.com/oss/python/langgraph/checkpointers)：内置实现与自定义 checkpointer
- [Use Time Travel](https://docs.langchain.com/oss/python/langgraph/use-time-travel)：回放与分叉的官方指南
