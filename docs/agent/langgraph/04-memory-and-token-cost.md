# Agent 越聊越贵：LangGraph 三套记忆的边界与 token 账单

**TL;DR:** 「越聊越贵」的机制很朴素：checkpointer 让历史消息全部留存，而模型每次都看全量 messages，成本随轮数线性涨、响应随长度变慢。解法不是换更大数据库，是分清三套东西的边界：checkpointer 管短期状态，Store 管跨会话长期记忆，上下文工程管每次到底给模型看多少。

## 账单是怎么涨起来的

上一轮讲过：挂了 checkpointer 之后，thread 里的历史全部留存，后续每次 invoke 模型都能看到。这正是问题所在。做一个粗略测算（数字是假设，量级是真实的）：一个客服 agent 平均一轮对话新增 2000 token，30 轮之后每轮请求携带约 60000 token。前 30 轮总共产生了约 93 万 token 的输入计费，其中大量是重复发送的旧历史。延迟同样在涨，首 token 时间随上下文长度明显变长。

存档免费，重发才是成本。搞清楚这一点，方案就清晰了：**数据库里存全量，模型面前做裁剪**。这两件事是分开的。

## 三套东西各管一段

社区里大量混淆来自把三个概念都叫「记忆」：

1. **checkpointer**（短期记忆）：thread 内的执行状态，作用域是一个业务流程。有了它，多轮对话和断点恢复才成立。它不该被清理得太积极，它是恢复的依据。
2. **Store**（长期记忆）：跨 thread 的键值存储，按 namespace 组织，可以配 embedding 做语义检索。用户偏好、历史结论放这里。
3. **上下文工程**（messages 管理）：每次调用时决定给模型看多少。它不删数据库里的东西，只裁剪输入。

一句话版：checkpointer 负责「不忘事」，Store 负责「跨会话记得」，上下文工程负责「每次只看该看的」。三者经常被指望互相替代，结果哪头都做不好。

## 管住 messages 的三个工具

**裁剪**：`trim_messages` 按 token 预算从后往前保留：

```python
from langchain_core.messages import trim_messages

trimmed = trim_messages(
    state["messages"],
    max_tokens=3000,
    token_counter=len,  # 按条数近似，生产建议换真实 tokenizer
    strategy="last",
    start_on="human",   # 裁剪后的开头必须是 human 消息
)
```

**删除**：消息带 id，可以显式移除。LangGraph 的惯用组合是「全量移除再回填保留集」：

```python
from langchain_core.messages import RemoveMessage
from langgraph.graph.message import REMOVE_ALL_MESSAGES

def compact(state: MessagesState) -> dict:
    kept = trim_messages(state["messages"], max_tokens=3000,
                         token_counter=len, strategy="last")
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES), *kept]}
```

`RemoveMessage` 走的是 `add_messages` reducer 的删除语义，所以这个节点返回的是合法增量。

**摘要**：把旧历史压缩成一条摘要消息，保留关键结论。自己写逻辑（摘要节点 + 上面两个原语）可行；用 LangChain 1.0 的 `create_agent` 时，现成的 `SummarizationMiddleware` 在接近阈值时自动摘要，一行接入。

## Store 的最小用法

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
store.put(("users", "u42"), "preferences", {"theme": "dark", "tone": "简洁"})
app = g.compile(checkpointer=checkpointer, store=store)
```

namespace 是元组，层级任意，`("users", "u42")` 下可以放多组 key。节点里通过注入的 store 参数读写和检索。生产用 Postgres 实现。要注意的边界：Store 不是向量库的替代品，它适合放「小而关键的结论」，整本文档还是该走 RAG。

## 什么不该进记忆

三类东西写进去就是负债：一次性上下文（当前页面的临时参数，该放请求里）；敏感凭证（state 和 Store 都会被持久化、被回放，API key 进去就等于落盘）；原始日志（该在数据仓库里，不该在 agent 记忆里）。

## 权衡

压缩有信息损失。摘要丢细节，裁剪丢旧轮次，有些任务就是需要第 3 轮的精确数字。稳妥的做法是把「关键结论」主动写入 Store，再放心裁剪 messages，让长期记忆兜底。另外上下文工程直接影响输出质量，压缩策略变了要重跑评测，别只看账单变便宜。

## 延伸阅读

- [Add Memory 指南](https://docs.langchain.com/oss/python/langgraph/add-memory)：短期与长期记忆的官方划分
- [Prebuilt middleware 文档](https://docs.langchain.com/oss/python/langchain/middleware/built-in)：SummarizationMiddleware 的参数与行为
- [Memory 概念](https://docs.langchain.com/oss/python/concepts/memory)：Store 的 namespace 与语义检索
