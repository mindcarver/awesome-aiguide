# 循环图为什么难调试：LangGraph 可观测与测试实战

**TL;DR:** 循环图难调试的根源是模型错配：主流 tracing 工具假设执行是一棵 span 树，LangGraph 的执行是一个会回到同一节点的环，报错堆栈在节点边界断掉。对策分三层：排查用 debug 流和快照历史，日常测试用节点级单测加假模型，运行期用步数计数做优雅降级而不是等 `GraphRecursionError`。

## 根源：环和树对不上

一次 LLM 应用的调用链，在 APM 里通常长成一棵树：请求在顶层，模型调用、工具调用是子 span。LangGraph 的执行不是树：同一个节点在一次 invoke 里可能进入五次，工具调用节点的下游可能回到它自己。span 树模型遇到环，只能展开成「每次进入都算一个新 span」，调用链一长，你在树里找不到「第几圈」的语义。堆栈也帮不上忙：异常在节点之间传递时经过框架调度，业务堆栈在边界处断开，你看到的是调度器帧，不是业务调用链。

承认这个根源，就不会浪费时间找一个「完美的 tracing」把它变回普通应用。下面三层是务实的替代。

## 排查层：三件现成的武器

**debug 流。** `stream_mode="debug"` 输出运行时最详细的事件（任务调度、写入、中断），配合 `tasks` 模式能看清并行分支里哪个任务卡住。排查时短开，平时不开。

**快照历史。** 出过事的 thread 用 `get_state_history` 做事后取证：每张快照都有完整状态和「下一步要跑什么」，出错的现场被持久化保留着，不用复现。

**Studio 与 LangSmith。** Studio 的图视图加时间旅行，适合本地开发期拖动回放；LangSmith 把每步 trace、输入输出、token 计费都记下来，是目前体验最完整的方案。它的位置要说清楚：商业 SaaS，免费额度有限，生产全量接要算钱。不买的替代是自建 OTel 上报（1.x 支持 telemetry 导出），或者退回 debug 流加日志。社区「没有 LangSmith 活不下去」的说法，反映的是默认可观测确实薄，不是必须买。

## 测试层：把 LLM 从回路里拿掉

LLM 调用又慢又不确定，直接测整张图等于买彩票。可行的分层：

1. **节点单测。** 节点是普通函数，输入 state 输出增量，直接调用断言。路由函数（条件边）同样直接测。这一层能覆盖绝大部分业务逻辑，毫秒级。
2. **假模型集成测试。** 用 fake model 返回预设的工具调用序列，图的拓扑、回环、状态合并都能测。LangChain 的测试工具提供 GenericFakeChatModel 这类替身。
3. **少量真实冒烟。** 十几条核心场景用真模型，定时跑而不是每次提交跑，容忍偶发失败但盯趋势。

一个测试设计上的坑：状态是逐节点变异的，节点 2 的 bug 常常到节点 5 才暴露。单测时除了断言本节点的输出，把「给定输入 state 的合法性」也断言上，失败定位会快很多。

## 运行层：优雅降级，别等递归爆炸

`recursion_limit` 默认 25，超限抛 `GraphRecursionError`。等它炸不是方案，用户看到的是 500。正确做法是让图自己知道还剩几步，提前走进降级分支：

```python
def route(state: S) -> str:
    if state["steps"] >= MAX_STEPS - 1:
        return "graceful_exit"     # 带着已有结果优雅收场
    return "continue"
```

自己维护步数计数（回环节点里自增）；用 `create_agent` 时它的 state 自带 `remaining_steps` 字段，官方文档建议直接检查它而不是捕获递归异常。降级出口要有内容：把已经拿到的部分结果返回给用户，比一句「出错了」体面得多。

## 权衡

这套分层不是免费的：节点要写成可独立调用的纯函数，假模型要维护，降级分支要设计。换来的确定性是：绝大多数失败在提交前被单测拦住，漏网的失败有快照可取证。另外提醒一句团队成本：图的可视化（Studio 图视图）对沟通很有用，但它验证的是拓扑，不是行为，别把「图长得对」当成「图跑得对」的证据。

## 延伸阅读

- [Test 指南](https://docs.langchain.com/oss/python/langgraph/test.md)：官方测试建议
- [Streaming 文档](https://docs.langchain.com/oss/python/langgraph/streaming.md)：debug 与 tasks 模式
- [Use Time Travel 指南](https://docs.langchain.com/oss/python/langgraph/use-time-travel.md)：快照取证的操作细节
