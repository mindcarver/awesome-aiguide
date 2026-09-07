# 条件边返回 50 个 Send 之后：LangGraph 并行分支与状态合并

**TL;DR:** `Send` 是 LangGraph 的 map-reduce 原语：条件边返回一列 `Send(节点, 输入)`，运行时为每项起一个并行任务。用它的三个纪律：reducer 合并顺序不保证，去重排序放到后续节点；并行区里的人审要走 id 映射；fan-out 数量要自己设上限。

## 场景：500 份简历打分

一批文档、每个独立打分、最后汇总排名。串行写法 500 次模型调用排队，太慢。Send 的写法：

```python
import operator
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send

class S(TypedDict):
    docs: list[str]
    scores: Annotated[list[int], operator.add]  # 各分支结果往这里合并

def split(state: S) -> list[Send]:
    # 条件边不返回节点名，返回 Send 列表：每项一个并行任务
    return [Send("score", {"doc": d}) for d in state["docs"]]

def score(state: dict) -> dict:
    # 注意：这里的 state 不是全局 S，是 Send 给的局部输入
    return {"scores": [rate(state["doc"])]}

def merge(state: S) -> dict:
    ranked = sorted(state["scores"], reverse=True)
    return {"scores": []}  # 汇总后的落库逻辑略

g = StateGraph(S)
g.add_node("score", score)
g.add_conditional_edges(START, split, ["score"])
g.add_edge("score", "merge")
g.add_edge("merge", END)
app = g.compile()
```

运行时收到 500 个 Send，就同时跑 500 个 `score` 任务。每个任务的输入是 Send 里给的那份局部 state，不是全局状态。全部任务跑完（一个 superstep 结束），500 份增量 `{"scores": [x]}` 一起交给 `operator.add` 合并。map-reduce 的两个阶段对应得非常整齐：Send 是 map，reducer 是 reduce。

## 合并秩序：三件必须知道的事

**顺序不保证。** 500 个并行任务谁先完成不受你控制，合并进列表的顺序和提交顺序无关。需要稳定顺序就在 merge 节点里排序，或者让 Send 的输入带上序号、结果里带回去。别在任何依赖「写入顺序」的逻辑上做假设。

**写入冲突看 reducer。** 并行任务写同一个 key，key 必须挂了 reducer，否则就是 #2 讲过的覆盖语义，最后完成的那个把别人的全擦掉。并行场景的 state 设计先问一句：这个 key 的 reducer 是什么？答不上来就别并行写它。

**fan-out 没有免费的无限并发。** Send 列表长度没有硬性上限，但每个任务都要占运行时资源，LLM 接口有自己的限流。500 个 Send 直接怼上去，多半换来一堆 429。实用做法是把 fan-out 粒度调粗（一次 Send 处理一批文档），或者把限流和重试做进任务内部。`recursion_limit` 默认 25 也值得记住：map 之后如果还有层层嵌套的图，层级和步数都在消耗这个预算。

## 并行区里的人审

并行分支里也有 interrupt 时（比如 50 份审批同时挂起），坑六的问题被放大：恢复值必须和 interrupt 一一对上。做法是给每个 interrupt 的 payload 带全局唯一 id（文档 id + 动作类型），恢复时传一个 `{id: 决策}` 的映射，每个分支节点按自己的 id 取值。这个模式官方文档有对应说明，属于并行人审的标准解法，别试图用「按顺序第几个」来对位。

## 为什么不用并行节点写死

只想要固定数量的并行分支（比如两个评审互相独立），直接从同一个节点拉两条边就够了，不需要 Send。Send 的存在价值是**数量在运行时才知道**：文档数、候选数、待审数是数据决定的。数量固定用静态边，数量动态用 Send，这个界线很清楚。

## 权衡

并行把延迟从「任务数 × 单任务耗时」压到接近「单任务耗时」，收益直观。成本在结果确定性上：合并顺序、失败重试的语义、部分失败的处理（500 个里 3 个失败，整体算成功还是失败）都要你在 merge 节点里显式设计。并行不是加个参数，是把「一致性处理」从框架手里接过来自己写。

## 延伸阅读

- [Graph API 文档](https://docs.langchain.com/oss/python/langgraph/graph-api.md)：Send 与条件边的官方定义
- [Use Subgraphs 指南](https://docs.langchain.com/oss/python/langgraph/use-subgraphs.md)：map-reduce 里的并行与嵌套关系
- [GRAPH_RECURSION_LIMIT 错误说明](https://docs.langchain.com/oss/python/langgraph/errors/GRAPH_RECURSION_LIMIT)：并行嵌套时步数预算
