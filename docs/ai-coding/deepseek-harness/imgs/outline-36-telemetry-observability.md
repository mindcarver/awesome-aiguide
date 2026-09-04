---
title: dsh Telemetry 可观测性：20 张 Notion 风格配图大纲
source: ../36-telemetry-observability.md
style: notion
aspect: 16:9
watermark: false
---

# 配图大纲

1. 会话日志与外部监控的职责分家。
2. 一条事件从捕获到 OTel 接收方去重的旅程。
3. 协调器与 telemetry sink 的小型契约。
4. harness 在 emit() 停止的边界公理。
5. 捕获点与热重载收养扫描如何覆盖会话。
6. FULL、FEEDBACK_ONLY、DISABLED 三种出站模式。
7. feedback-only 只接受真实落盘对象的授权闸门。
8. 默认关闭与环境变量配置正门。
9. 每个 turn/step 只投影首个 assistant/chunk。
10. seq 间隙是压缩投影，不等于丢失。
11. 深拷贝、脱敏插件与 fail-closed 的瀑布。
12. 尽力而为的水位线、崩溃与接收方去重。
13. ledger 与 ops 两个独立的观测通道。
14. shutdown 缺席为何必须结合模式解释。
15. 外发事件中真正可能离开机器的数据。
16. 共享页只披露策略，不承诺投递或留存。
17. OTel SDK 管道、批处理与三秒关机死线。
18. 匿名 user.id 资源身份的生成与重置。
19. 被否决的方案与“不预付复杂度”的取舍。
20. capture、projection、redaction、emit 的整体边界总结。
