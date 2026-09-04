---
article: 26-context-budget-compaction-and-spill.md
type: mixed
density: rich
style: notion
palette: default
image_count: 20
generator: gpt-image-2
---

# Context Budget、Compaction 与 Spill 插图大纲

1. Compaction 是 harness 被动机制，不是模型工具。
2. 请求前压力触发与请求失败溢出触发。
3. 先确定性修剪，后昂贵总结。
4. 头中尾修剪和 Unicode 码点边界。
5. Token Meter 的重放快照与两种锚点。
6. 0.8 阈值、0.16 保留尾巴、8192 总结上限与逐 step 检查。
7. 不可拆开放尾 step 暂不压缩。
8. start / summary / end 三事件的日志锁。
9. 孤儿锁与会话种子边界。
10. 总结调用以已路由前缀复用 KV 缓存。
11. 自动压缩命中缓存、手动中段压缩正确但不承诺命中。
12. 压缩区域必须保持工具调用/结果配对。
13. 溢出恢复只看表面是否推进。
14. Compaction 与 Spill 的正交分工。
15. spillStore：内容、定位符、提示与字节数。
16. 本地 spill 安全写入 session 私有文件。
17. 结果入上下文前的 spill 策略 waterfall。
18. 为避免循环跳过 read，但 code dispatch 日志不跳。
19. fork 继承定位符引用、工件不复制。
20. 三机制总结：token meter + compaction + spill。
