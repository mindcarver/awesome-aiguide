# 读懂 Reducer，才算读懂 LangGraph：节点、状态与增量更新

**TL;DR:** LangGraph 的状态不是共享变量，而是一组「每个 key 带合并规则的通道」。节点永远只返回增量更新，合并方式由 reducer 决定。理解了这一点，superstep 执行、checkpoint 边界、并行合并乱序这些现象都能自然推导出来；不理解，就只能靠背。

## 先看一个所有人都踩过的坑

两个节点先后写同一个 key，直觉是「后写的赢」。跑一下：

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    items: list[str]

def node_a(state: State) -> dict:
    return {"items": ["A"]}  # 我贡献了 A

def node_b(state: State) -> dict:
    return {"items": ["B"]}  # 我贡献了 B

g = StateGraph(State)
g.add_node("a", node_a)
g.add_node("b", node_b)
g.add_edge(START, "a")
g.add_edge("a", "b")
g.add_edge("b", END)
app = g.compile()

print(app.invoke({"items": []}))
# {'items': ['B']}
```

没有报错，但 `node_a` 的结果消失了。对于没有声明合并规则的 key，LangGraph 的默认行为是**覆盖**：节点返回什么，这个 key 就变成什么。你写的不是「我新增了 A」，而是「items 现在是 A」。节点 B 一返回，A 就没了。

想让它变成累加，给 key 挂一个 reducer：

```python
from typing import Annotated, TypedDict

def merge(left: list[str], right: list[str]) -> list[str]:
    return left + right

class State(TypedDict):
    items: Annotated[list[str], merge]
```

其他代码不变，这次输出 `{'items': ['A', 'B']}`。`Annotated` 的第二个参数告诉 LangGraph：当有新值写入这个 key 时，拿旧值和新值调用这个函数，结果才是新状态。

## reducer 是状态的一部分，不是工具函数

这是最容易被忽略的认知：reducer 声明在 State 定义里，而不是写在节点代码里。它回答的问题是「当两个写入冲突时听谁的」，这是状态定义层面的事。

常用内置的三种：

1. 默认覆盖：不写 `Annotated`，后写覆盖先写。适合只有一个节点会写的 key。
2. `add_messages`：消息列表专用 reducer，按消息 id 去重合并，还支持 `RemoveMessage` 删除语义。
3. 自定义函数：自己定义合并逻辑，比如按分数去重、按时间排序。

`MessagesState` 这个预置类型就是「一个挂了 `add_messages` 的 messages key」，聊天类应用直接用它就够了。

一个反直觉的点：reducer 管的是「新值怎么并进来」，不管「怎么避免重复写入」。如果两个并行节点返回了内容相同的两条消息，`add_messages` 因为 id 不同会都保留。去重是你的业务逻辑，不是 reducer 的。

## superstep：所有事都发生在边界上

LangGraph 的执行模型来自 Pregel：图按 **superstep**（超步）推进。每个 superstep 里，当前所有就绪的节点拿着**同一份状态**并行执行；全部执行完后，各自的增量更新经过 reducer 合并，生成下一份状态，进入下一个 superstep。

这意味着两件事：

**并行节点互相看不见。** 同一 superstep 里的两个节点，拿到的输入状态完全相同。A 在执行中写的东西，B 看不到，要等下一个 superstep。如果你的两个节点有「先查询再汇总」的依赖，它们必须在不同的 superstep 里，也就是用边连起来。

**checkpoint 只在 superstep 边界生成。** 快照是每个边界上的完整状态，节点执行到一半挂掉，恢复时整个节点从头再跑。所以官方反复强调：不要把五个步骤塞进一个大节点，每个步骤的失败都会连累其余四个步骤跟着重跑。反过来也不用担心拆太细，checkpoint 写入是异步的，节点粒度细不构成性能问题。

并行分支的合并顺序也要注意。两个节点在同一个 superstep 里写同一个挂了 reducer 的 key，合并能保证发生，但**先后顺序不保证**。`["A", "B"]` 和 `["B", "A"]` 都可能出现。对顺序敏感的场景，要么串行，要么在合并后统一排序。

## 为什么是「增量」：为了可重放

强制节点返回增量而不是直接改状态，代价是写起来多想一步，收益是整个执行变成可重放的：状态完全由「初始输入 + 逐边界的更新序列」推导出来。checkpoint 里存的其实就是这套推导的中间结果，所以才能拿着历史 checkpoint 重跑（回放）、或者从历史某点分叉出新执行。

节点想真正清空一个 key 怎么办？返回 `Overwrite(None)` 这类包装值，绕过 reducer 直接覆盖。这是少数几个「我就是不要合并」的合法出口。

还有两个高频小坑，都和这套模型直接相关：

- 静态边和 `Command` 动态路由不要同时给一个节点用。两者都会生效，流量走两遍。
- `recursion_limit`（默认 25）写在 invoke 的 config 里，不是 `compile()` 参数：`app.invoke(inputs, config={"recursion_limit": 50})`。搜到的很多旧教程写错了位置。

## 权衡与建议

reducer 模型的成本是显性的：你得想清楚每个 key 的合并语义，状态里放原始数据、在节点里按需格式化（这是官方文档给出的原则，直接抄就好）。成本换来的是并行、恢复、回放这些能力不需要你写一行同步代码。

我的建议：新项目从 `MessagesState` 起步，业务状态只加必要的 key，每个 key 都能一句话说清「合并规则是什么」。说不清的 key 大概率不该存在，它对应的数据可能属于 Store 或者干脆属于节点内部局部变量。

## 延伸阅读

- [Graph API 概念文档](https://docs.langchain.com/oss/python/langgraph/graph-api)：State、reducer、边的权威说明
- [Thinking in LangGraph](https://docs.langchain.com/oss/python/langgraph/thinking-in-langgraph)：官方写的设计原则，含「状态存原始数据」的出处
- [GRAPH_RECURSION_LIMIT 错误说明](https://docs.langchain.com/oss/python/langgraph/errors/GRAPH_RECURSION_LIMIT)：默认 25 与调参位置
