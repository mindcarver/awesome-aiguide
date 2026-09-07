# 子图一嵌，状态就丢？LangGraph Subgraph 的隔离与透传

**TL;DR:** 子图（subgraph）有独立的 checkpoint namespace，父图默认看不到它的内部状态，这不是 bug 是设计。真正会出事的是三处透传：同名 key 的写入要走 reducer 合并、人审恢复值要能穿过父子两层、streaming token 要显式开 `subgraphs=True`。三个都有对应开关，关键是知道隔离默认存在。

## 先立现象：日志里有，状态里没有

典型报障：子图节点日志显示检索正常完成，父图后续节点拿到的检索结果却是空。排查方向不是「数据在哪丢了」，而是先确认一件事：**父图 schema 里有没有同名 key**。子图不是函数调用，它是一个有自己状态空间的图实例；它的内部状态存在自己名下（checkpoint namespace 形如 `node:uuid`），父图只通过「两边 schema 的同名 key」和它交换数据。

父图 state 有 `docs` 这个 key，子图 state 也有 `docs`，子图写入的值经过 reducer 合并后才会出现在父图。父图没定义这个 key，数据就留在子图里，父图永远看不见。

## 三种接法，适用场景不同

**接法一：编译后的子图直接当节点。**

```python
parent = StateGraph(ParentState)
parent.add_node("research", research_subgraph)  # research_subgraph 是 compile() 的产物
```

同名 key 自动透传，最常用。适合「子流程有自己的内部结构，但对父图只暴露输入输出」的场景。

**接法二：用函数包一层，显式做字段映射。**

```python
def call_research(state: ParentState) -> dict:
    result = research_subgraph.invoke({"query": state["query"]})
    return {"findings": result["summary"]}
```

子图 schema 和父图完全解耦，靠这层函数翻译。字段名不一致、要做裁剪清洗时用这种，代价是子图的中间过程对父图完全不可见，checkpointer 断点恢复只覆盖函数边界之外。

**接法三：子图节点直接指挥父图跳转。** 节点返回 `Command` 时可以指定作用域是父图：

```python
from langgraph.types import Command

def handoff(state) -> Command:
    return Command(goto="reviewer", graph=Command.PARENT,
                   update={"current": "review"})
```

swarm 式移交靠它：子 agent 决定「这事归谁」，直接让父图路由，不用先退回父节点再判断。

## 隔离对人审和回放的影响

子图挂了自己的 checkpointer 状态（namespace 隔离），两个实际后果：

**人审要穿两层。** 子图里的 `interrupt()` 触发后，父图也停在对应节点。恢复时 resume 值要能从父图传进子图的 interrupt 处。父图和子图都要挂 checkpointer，这是容易漏的配置，漏了子图的暂停状态无处安放，恢复行为不可预期。

**时间旅行有过历史 bug。** 旧版本在「interrupt + subgraph」组合下回放，可能复用残留的 RESUME 值，1.1 版本修复了这个问题。用旧版本的团队在调试人审回放时如果遇到「恢复值莫名其妙」，先升版本再查自己的代码。

## streaming 的透传开关

前端联调时最常见的现象：`stream_mode="messages"` 下，顶层图的 LLM token 能推到前端，子图里 LLM 的 token 却不出现。因为流式输出默认只在顶层流动，嵌套一层就断了。修复是一个参数：

```python
for chunk in app.stream(inputs, config, subgraphs=True, stream_mode="messages"):
    ...
```

`subgraphs=True` 让内部子图的流也冒泡上来。代价是事件量变大，配合 metadata 里的来源信息过滤出你要的那层。前端「为什么有些回答没有流式效果」的工单，十有八九是这一条。

## 什么时候不拆子图

拆子图的收益是状态隔离和团队边界，成本全在透传上。判断标准：如果拆完之后两个图的同名 key 超过三分之一，说明它们本来就是同一个状态域，拆开只会让每个读写都过两层合并；如果子图只是想「让代码短一点」，用普通函数节点就够了，函数节点没有 namespace、没有恢复语义的复杂性。真正值得拆的信号是：不同部分需要不同的工具集或权限，或者不同团队维护各自的流程。

## 延伸阅读

- [Use Subgraphs 指南](https://docs.langchain.com/oss/python/langgraph/use-subgraphs.md)：三种接法与 namespace 说明
- [LangGraph 仓库](https://github.com/langchain-ai/langgraph)：issue 区可按 subgraph 关键字搜到真实案例
- [Streaming 文档](https://docs.langchain.com/oss/python/langgraph/streaming.md)：subgraphs=True 与流式语义
