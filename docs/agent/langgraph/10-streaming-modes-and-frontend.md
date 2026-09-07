# 前端拿不到 token？LangGraph Streaming 七种模式与前端联调

**TL;DR:** LangGraph 的 `stream_mode` 有七种，前端打字机效果用的是 `messages`；「子图 token 推不出来」要开 `subgraphs=True`；进度条用 `custom` 模式自己推。前端侧用官方 SDK 的 `useStream`，断线靠按 run 重连。2026 年的新项目还应该知道 1.1 引入的类型化流式事件。

## 七种模式，先分清要什么

| stream_mode | 推给你什么 | 典型用途 |
| --- | --- | --- |
| `values` | 每个 superstep 后的完整状态 | 调试看全局 |
| `updates` | 每个节点的增量更新 | 看哪个节点改了什么 |
| `messages` | (消息块, 元数据) 元组 | 前端打字机效果 |
| `custom` | 你自己写的任意事件 | 进度条、业务事件 |
| `checkpoints` | 快照元数据 | 深度调试持久化 |
| `tasks` | 任务开始/结束事件 | 追踪并行任务进度 |
| `debug` | 运行时最详细事件 | 框架级排查 |

最常混用错的是 `values` 和 `updates`：前者是「到这一步为止的全貌」，后者是「这一步改了什么」。UI 上展示流程进度用 `updates`（节点粒度），导出完整执行记录用 `values`。`debug` 模式事件量巨大，只在排查运行时问题时短开。

## messages 模式：打字机效果的三个细节

```python
for chunk in app.stream(inputs, config, stream_mode="messages"):
    msg, metadata = chunk                      # 元组，不是裸消息
    if msg.content and metadata["langgraph_node"] == "writer":
        print(msg.content, end="", flush=True)
```

细节一：输出是元组，第二个元素 `metadata` 带 `langgraph_node` 等来源信息。想只显示某个节点的输出（比如过滤掉内部审核 agent 的思考），就在这里判。

细节二：**嵌套子图默认不透传**。主图里的 LLM token 能推出来，子图里 LLM 的 token 推不出来，加 `subgraphs=True`：

```python
app.stream(inputs, config, subgraphs=True, stream_mode="messages")
```

这是「前端为什么时有时无流式效果」的第一嫌疑人。

细节三：不想暴露的内部 LLM 调用（比如压缩摘要的那次）可以打上 `nostream` 标签，运行时会跳过它的事件。

## custom 模式：进度不只能靠 token

长任务（批量检索、多文件处理）里，token 流是断续的，用户界面需要的是业务进度。节点里拿一个 writer，想推什么推什么：

```python
from langgraph.config import get_stream_writer

def crawl(state: dict) -> dict:
    writer = get_stream_writer()
    for i, url in enumerate(state["urls"]):
        writer({"progress": f"{i + 1}/{len(state['urls'])}", "url": url})
        do_fetch(url)
    return {"done": True}
```

`stream_mode="custom"` 时这些字典原样出现在客户端，配合 messages 模式一起开（`stream_mode=["messages", "custom"]`），前端既有打字机又有进度条。

## 前端联调的现成方案

不用从裸 WebSocket 撸起。官方 JS/TS SDK（`@langchain/langgraph-sdk`）提供 `useStream` React hook，接管了 token 流、interrupt 展示、提交恢复值这些协议细节；配合 Agent Chat UI（开源 Next.js 项目）可以更快拿到一个完整参考实现。真正要自己处理的坑是**断线重连**：长流程跑几分钟，网络闪断后流没了，任务还在服务端跑。SDK 提供 `joinStream`（按 run_id 重新挂上事件流），前端要在重连后先查 run 状态再决定是重新订阅还是拿最终结果。这块没有全家桶，值得在联调阶段专门测一遍。

## 1.1 之后：类型化事件

传统流式的输出是无 schema 的字典，`interrupt` 信息要从状态里挖 `__interrupt__` 字段。1.1 起可以 `stream(..., version="v2")` 拿到类型化的 `StreamPart` 事件，invoke 返回 `GraphOutput`（`.value` / `.interrupts`），前面那个挖字段的动作变成读属性。官方还推出了更激进的内容块中心事件流（v3，beta）。存量代码没必要急着迁，新项目值得直接从 v2 起步，省掉一次事件格式的迁移。

## 权衡

流式的真实成本在服务端事件量和前端的消费逻辑。全模式全开（`debug` + `subgraphs=True`）在多 agent 应用里每秒能产生几千个事件，前端渲染会先于网络崩掉。建议：生产只开业务需要的模式，`debug` 和 `checkpoints` 留给排查；前端对事件做合帧（每 50ms 合并一次再渲染）；把「事件太多」当成和「响应太慢」同级的性能问题对待。

## 延伸阅读

- [Streaming 文档](https://docs.langchain.com/oss/python/langgraph/streaming.md)：七种模式与 v2 事件
- [LangGraph SDK (JS)](https://github.com/langchain-ai/langgraph-sdk)：useStream 与 joinStream
- [Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui)：流式 + interrupt 的完整前端参考
