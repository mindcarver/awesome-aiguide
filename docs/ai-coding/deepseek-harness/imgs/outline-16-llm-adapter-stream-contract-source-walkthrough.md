---
type: mixed
density: rich
style: notion
generator: gpt-image-2
image_count: 12
---

# LLM 适配器与 stream 契约：配图大纲

1. **适配器边界**：loop 只面对 `ctx.llm` 和中立 `StreamChunk`；图 16-01-framework-adapter-boundary.png。
2. **provider 路由**：provider 是路由键，model 只是建议；图 16-02-flowchart-provider-routing.png。
3. **stream waterfall**：洋葱中间件可放行或短路；图 16-03-framework-stream-waterfall.png。
4. **差异吸收**：序列化、SSE、记账、错误归一都在适配器里；图 16-04-framework-adapter-responsibilities.png。
5. **空闲看门狗**：持续产出与挂起超时、abort 的边界；图 16-05-timeline-idle-watchdog.png。
6. **封闭动作与开放内容**：协议动作封闭、块类型可扩展；图 16-06-comparison-protocol-vocabulary.png。
7. **收尾顺序**：delta 实时，block-end、usage、finish 在 DONE 后按序；图 16-07-flowchart-stream-ordering.png。
8. **三层契约防御**：编译、运行期校验、assembler 防御；图 16-08-framework-contract-defense.png。
9. **双轨消费**：chunk 原样日志与 BlockAssembler 同时进行；图 16-09-flowchart-dual-track-consumption.png。
10. **中断安全前缀**：文本推理保留，工具调用丢弃；图 16-10-comparison-interruption-safety.png。
11. **失败与重试**：两出口归一、策略层指数退避；图 16-11-flowchart-failure-retry.png。
12. **离线重放**：日志重建中立 chunk，私有 replayState 严格隔离；图 16-12-framework-replay-boundary.png。
