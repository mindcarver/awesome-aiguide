# 人审不是加个按钮：LangGraph interrupt 落地的六个坑

**TL;DR:** `interrupt()` 五分钟能跑通 demo，生产化要跨过六个坑：别用 try/except 包它、恢复值按索引匹配、一个节点每轮只能 interrupt 一次、payload 必须可序列化、前置副作用要幂等、并行分支要按 id 映射恢复。外加一个框架不管的问题：人一直不批，谁来兜底。

## 先把最小可用的人审立起来

场景：agent 要执行一笔 5 万的转账，停下来等人点头。

```python
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

def approve_transfer(state: MessagesState) -> dict:
    decision = interrupt({
        "type": "transfer_approval",
        "amount": 50000,
        "question": "是否批准向供应商 A 转账 5 万元？",
    })
    # 人处理后从这里继续，decision 就是 resume 传进来的值
    if decision == "approve":
        return {"messages": [("assistant", "已批准，执行转账")]}
    return {"messages": [("assistant", "已拒绝，流程终止")]}

g = StateGraph(MessagesState)
g.add_node("approve", approve_transfer)
g.add_edge(START, "approve")
g.add_edge("approve", END)
app = g.compile(checkpointer=InMemorySaver())

config = {"configurable": {"thread_id": "po-1"}}
app.invoke({"messages": [("user", "申请转账")]}, config)  # 暂停在 interrupt
app.invoke(Command(resume="approve"), config)             # 人批准后恢复
```

两个硬前提：图必须挂 checkpointer（暂停的本质是存档），调用必须带 thread_id。`interrupt()` 把当前节点挂起，恢复时**整个节点从第一行重跑**，跑到 `interrupt()` 时拿到 resume 值继续往下。理解了「恢复等于重跑」，六个坑里有一半能自己推导出来。

## 坑一：用 try/except 把 interrupt 包住了

`interrupt()` 在实现上通过抛出特殊异常把执行流交还运行时。包一层 `try: ... except Exception:` 想做「统一错误处理」，结果是这个特殊异常也被吞了，人审静默失效，或者节点行为错乱。修复：错误处理别覆盖 interrupt 调用处；确实要包，捕获后把 `GraphInterrupt` 原样 re-raise。

## 坑二：恢复值按索引匹配，顺序一变就对错位

一个流程里有多处 interrupt 时，resume 值按**发生的顺序**依次对齐，不是按你起的名字。前面分支加一个 interrupt，后面所有审批人的答案全部错位。这是社区 issue 里最高频的人审事故。修复：每个 interrupt 的 payload 里带业务唯一标识（审批单号、动作类型），前端提交回来的 resume 值也带标识，节点内自行校验对上再消费。别依赖顺序。

## 坑三：一个节点在一轮里反复调 interrupt

有人的写法是在循环里逐项请求确认：

```python
# 错误示范
for item in state["items"]:
    if not interrupt({"item": item}):   # 循环里调 interrupt
        cancel(item)
```

每次恢复，节点从头重跑，循环里的 interrupt 会再次进入等待，重放路径指数膨胀，官方文档明确把这条列为反模式。修复：一次 interrupt 收齐全部决策，payload 是列表，人一次性处理完，节点内遍历结果。

## 坑四：payload 不可序列化

interrupt 的参数和 resume 值都要写进 checkpoint，随数据库落盘。传 lambda、数据库连接、文件句柄，存档那一步就炸。payload 用 JSON 能表达的东西：dict、str、数字、列表。需要富对象就传 id，恢复后重新查库。

## 坑五：前置副作用不幂等

「恢复等于重跑」的推论。interrupt 之前如果调了扣库存接口，人审通过恢复执行，扣库存发生两次。写节点时的纪律：要么把副作用放在 interrupt 之后，要么给外部调用带幂等键。用 deepagents 的 `interrupt_on` 做工具级审批时同理，工具实现要经得起重复调用。

## 坑六：并行分支的 interrupt 恢复错乱

多个分支并行执行、各自 interrupt，resume 时多个值要和对的 interrupt 对上。官方给出的模式是用 interrupt id 构建映射：payload 里带唯一 id，前端恢复时传 `{id: 决策}` 的字典，节点内按 id 取。和坑二同源，只是并行把顺序彻底打乱了，索引彻底不可用。

## 框架不管的事：人一直不批怎么办

LangGraph 没有内置的 interrupt 超时，一个 thread 停在等待状态，理论上是永久的。生产要自己补：一个后台任务定期扫描「停在等待中的 thread」，超过 SLA 就走两个出口之一，用 `update_state` 写入默认决策（自动拒绝比自动批准安全）后 resume，或者标记流程过期并通知发起人。这段代码不难，但要有人写，评审方案时把它当成必答项。

前端和工单系统对接的思路也顺带说清：interrupt 触发时把 payload、thread_id、checkpoint 信息投给你的工单系统；审批人在界面点批准，后端拿 thread_id 调 `Command(resume=...)`。官方的 Agent Chat UI 已内置 interrupt 展示和恢复，可以参考它的交互协议。

## 何时不该做人审

人审很贵：它把同步流程变成异步流程，引入超时、提醒、代批这些一整套机制。低风险高频操作（查询、格式转换）做人审是浪费，用权限分级和 dry-run 替代。真正值得的是不可逆、有资金、有对外影响的动作。

## 延伸阅读

- [Human-in-the-loop 概念文档](https://docs.langchain.com/oss/python/langgraph/interrupts.md)：含官方列出的 interrupt 使用注意事项
- [interrupt API 参考](https://reference.langchain.com/python/langgraph/types/interrupt)：恢复匹配语义
- [Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui)：内置 interrupt 交互的开源前端
