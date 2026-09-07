# Agent 跑一半挂了：LangGraph 错误分类、重试与补偿设计

**TL;DR:** 错误处理的常见死法是两个极端：裸奔（一个异常整个图崩）或者一把抓（全部 try/except 吞掉）。正确起手式是先分类。LangGraph 的设计把错误分成四类，每类有不同的处理位置：瞬时错误给 RetryPolicy，模型可恢复的写回状态回环，需要人的走 interrupt，耗尽重试的给 error_handler 做补偿。

## 先分类：四类错误，四个处理位置

拿到一个异常，先问它会落到哪一类：

1. **瞬时错误。** 网络抖动、接口限流、超时。重试大概率成功，不值得让模型知道。
2. **模型可恢复的错误。** 工具返回了业务报错、输出格式不合法。把错误信息写回状态让模型重试，它能自己修正。
3. **用户可修复的错误。** 参数缺失、权限不足、余额不够。模型重试一百次也没用，要暂停问人，这是 `interrupt()` 的活。
4. **不可恢复的错误。** 重试耗尽、外部系统宕机。框架层面要的是优雅收场：记录、补偿、通知，而不是把异常抛穿整个进程。

分类的价值在于让异常出现在正确的层。最常见的生产 bug 是层级错位：本该重试的瞬时错误直接炸了图，本该问人的业务错误被自动重试刷了一百次。

## 瞬时错误：RetryPolicy 和 timeout

```python
from langgraph.pregel import RetryPolicy

g.add_node(
    "call_payment_api", call_payment_api,
    retry_policy=RetryPolicy(max_attempts=3, initial_interval=1.0,
                             backoff_factor=2.0, jitter=True),
)
```

RetryPolicy 挂在 `add_node` 上，带指数退避和抖动。重试期间节点的写入会被丢弃重来，checkpoint 语义保证这个重试是干净的：上一次尝试的半成品状态不会漏进快照。

1.2 版本给 `add_node` 加了 `timeout` 和 `error_handler` 参数。一个必须知道的限制：**timeout 只支持 async 节点**，同步节点不生效。超时触发 `NodeTimeoutError`，已产生的写入被清理，然后交给 retry policy 决定要不要重试。整张图的默认值可以用 `set_node_defaults` 统一设，不用每个节点重复写。

## 模型可恢复：写回状态回环

工具报错不抛异常，作为数据返回。校验节点发现格式不合法，也走同一条路：把错误写进状态，条件边把流程导回上一个节点，让模型带着报错重试。

```python
def validate(state: S) -> dict:
    problems = check(state["draft"])
    if problems:
        return {"errors": problems}   # 错误是数据，不是异常
    return {"errors": []}

def route(state: S) -> str:
    return "revise" if state["errors"] else "publish"

g.add_conditional_edges("validate", route)
g.add_edge("revise", "validate")      # 回环
```

这是 agent 循环的本质：把「失败」变成「上下文里的一条反馈」。注意回环要带退出条件（重试计数、错误次数上限），否则和 `recursion_limit`（默认 25）撞上就是 `GraphRecursionError`。

## 需要人的：interrupt，不要用异常模拟

权限不足、需要业务确认这类错误，正确动作是暂停等人，也就是上一批文章里讲过的人审原语。反模式是抛异常让上游接口返回 500，用户刷新重试，状态全丢。错误分类的第三类和第四类的分界线就在这：人能修的走 interrupt，人都修不了的才走补偿。

## 耗尽之后：error_handler 与补偿

重试次数用完、外部系统确认宕机，此时图挂掉之前还能做最后一件事：补偿。1.2 的 `error_handler` 参数给节点挂一个收尾函数，做三件事之一——写一条「降级结果」进状态让流程继续（比如返回缓存值）、执行反向操作（删除刚创建的资源）、把现场（thread_id、checkpoint、错误详情）投给监控系统再失败。

官方设计文档里有一条原则值得刻在脑子里：**don't catch what you can't handle**。不能处理的异常就让它炸，炸在 checkpoint 边界意味着恢复后从干净状态重跑；吞掉它才会留下半损坏的状态。try/except 的合法覆盖范围和你能为这个异常写出的恢复动作一一对应，写不出恢复动作就别捕获。

## 权衡

这套机制的运维含义：错误策略是图定义的一部分，跟着代码走版本管理。改重试策略要过评审、要回归测试，和改业务逻辑同级。它换来的是错误行为可预期、可测试、可回放。另一个提醒：补偿逻辑本身也会失败，写它的时候按「尽力而为」设计，补偿失败只能靠人来对账，别设计成递归补偿。

## 延伸阅读

- [Fault Tolerance 文档](https://docs.langchain.com/oss/python/langgraph/fault-tolerance.md)：RetryPolicy、timeout、error_handler 的官方说明
- [Fault Tolerance in LangGraph（官方博客）](https://www.langchain.com/blog/fault-tolerance-in-langgraph)：1.2 容错机制的设计动机
- [Human-in-the-loop 文档](https://docs.langchain.com/oss/python/langgraph/interrupts.md)：第三类错误的处理原语
