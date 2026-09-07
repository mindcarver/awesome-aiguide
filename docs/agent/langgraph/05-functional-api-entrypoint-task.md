# 不画图写 LangGraph：@entrypoint、@task 与两种范式的选择

**TL;DR:** LangGraph 有两套写法：StateGraph 画图，functional API 写普通函数。后者用 `@entrypoint` 标入口、`@task` 标步骤，靠「重放 + 跳过已完成 task」实现持久化。选型标准不是新旧，是控制流形态：分支路由复杂、需要可视化调试选图；步骤序列为主、想按普通代码方式推理选函数式。两者可以混用，不是二选一。

## 同一件事的两种写法

先感受差异。「取订单、生成周报」这个两步任务，StateGraph 版本长这样：

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class S(TypedDict):
    user_id: str
    orders: list
    report: str

def fetch(state: S) -> dict:
    return {"orders": query_orders(state["user_id"])}

def report(state: S) -> dict:
    return {"report": make_report(state["orders"])}

g = StateGraph(S)
g.add_node("fetch", fetch)
g.add_node("report", report)
g.add_edge(START, "fetch")
g.add_edge("fetch", "report")
g.add_edge("report", END)
graph_app = g.compile()
```

函数式版本：

```python
from langgraph.func import entrypoint, task
from langgraph.checkpoint.memory import InMemorySaver

@task
def fetch_orders(user_id: str) -> list:
    return query_orders(user_id)

@task
def make_report(orders: list) -> str:
    return summarize(orders)

@entrypoint(checkpointer=InMemorySaver())
def weekly(user_id: str) -> str:
    orders = fetch_orders(user_id).result()
    return make_report(orders).result()

weekly.invoke("u42", config={"configurable": {"thread_id": "w-1"}})
```

注意 `@task` 装饰的函数调用后返回的不是结果，是一个 future，要 `.result()` 才拿值。这是函数式 API 最重要的语法细节，也是它实现持久化的钥匙。

## 重放：函数式的持久化语义

图版本的恢复靠 superstep 边界快照；函数式没有图，它的恢复机制是**重放**：`@entrypoint` 挂了 checkpointer 之后，每次执行会把每个 task 的输入输出记档。中断后重新 invoke 同一个 thread，函数体从头执行，但已经跑完的 task 不再真的执行，直接取上次记录的结果，只有没跑到的地方才真正继续。

这套语义直接推出一条铁律：**副作用必须放进 `@task`**。写在 task 外面的代码，重放时会原样再跑一遍，请求发两次、邮件收两封。反过来，把非确定值（随机数、当前时间、外部 API 响应）也放进 task，恢复后的执行才和正常执行一致。

```python
@entrypoint(checkpointer=InMemorySaver())
def weekly(user_id: str) -> str:
    ts = datetime.now()          # 错：重放时会变成恢复时刻的时间
    orders = fetch_orders(user_id).result()
    ts2 = now().result()         # 对：时间也作为 task 结果被记录
    ...
```

## 怎么选

三条判断，按顺序问：

1. **控制流是数据依赖的还是步骤序列的？** 有大量条件分支、动态路由、并行扇出，图的模型更贴合，`add_conditional_edges` 和 Send 就是干这个的。线性步骤为主，函数式更省事，不用为三个步骤定义 State schema。
2. **要不要图可视化？** functional API 不支持图可视化和 Studio 的图视图。调试靠日志。团队需要「给非工程同事看流程图」，选图。
3. **团队熟悉度？** 函数式是普通 Python，新人上手快；图的 superstep、reducer 心智模型要先学。

灰色场景存在：主流程线性，中间一段是复杂子流程。两者可以混用，`@entrypoint` 里可以正常 invoke 一个编译好的图，图节点里也可以调 task 逻辑。按边界拆，别硬翻译成一种范式。

## 权衡

函数式换来的代价：调试工具少（没有图视图）、生态集成少（大量教程和组件默认图形态）、恢复语义更依赖你对 task 粒度的把握（task 切太粗，重放代价大；太细，记录开销大）。它适合的范围比图窄，但在适合的范围里，代码量的差距是压倒性的。

我的用法：把 functional API 当成「LangGraph 提供的持久化函数库」，用在明确线性的任务上，比如批处理、报表、单文件 ETL；任何开始长出分支的需求，直接换 StateGraph，别等它腐烂。

## 延伸阅读

- [Functional API 文档](https://docs.langchain.com/oss/python/langgraph/functional-api)：entrypoint/task 语义与重放机制
- [Choosing your approach](https://docs.langchain.com/oss/python/langgraph/choosing-apis)：官方的选型建议
- [Graph API 文档](https://docs.langchain.com/oss/python/langgraph/graph-api)：对照阅读两套模型的差异
