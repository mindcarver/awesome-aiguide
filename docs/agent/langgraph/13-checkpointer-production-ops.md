# LangGraph Checkpoint 不是免费的：一次生产事故复盘

**TL;DR:** checkpointer 上线三个月后的三类典型事故：checkpoint 表膨胀拖垮数据库、子包 patch 升级夹带破坏性变更、state schema 演进后旧快照反序列化失败。这篇按事故复盘的格式把它们拆开，数据来自公开 issue 和工程博客，结论部分给出对应的生产纪律。

> 素材说明：以下三个事故样本来自 LangGraph 仓库公开 issue 和工程师博客（链接在文末），按复盘格式整理，非作者亲历项目。

## 事故一：表膨胀，Postgres 磁盘告警

**现象。** 多轮对话 agent 上线三个月，checkpoint 相关表占掉数百 GB，数据库整体性能下滑，慢查询告警。

**机制。** checkpointer 的写入模型是「每个 superstep 存一张完整快照」。一个 30 轮的对话线程，每轮 2 到 4 个 superstep，就是 60 到 120 张快照，每张包含全量消息历史。消息越聊越长，快照越大，存储是平方级增长。社区 issue #7714 里的实测口径是：序列化膨胀让存储增加约 85%，每轮模型调用的 token 开销额外增加约 37.8%（作者声称值，未见官方复测，但方向与快照模型一致）。

**修复。** 两层：给 thread 定生命周期（结束后清理或归档 checkpoints），运行中的 thread 设置保留窗口；消息侧做裁剪和摘要（checkpointer 存多少取决于 state 有多大，state 小快照就小）。1.2 版本的 DeltaChannel（beta）把快照从全量改为增量存储，是框架层面的缓解，可以关注但生产采用要自己评估 beta 风险。

**教训。** 上线方案评审时，「checkpoint 数据怎么清理」应该和「日志怎么轮转」同等级别地被问到。没有答案，就是给三个月后的磁盘告警签了字。

## 事故二：patch 版本升级，序列化打挂

**现象。** 例行依赖升级，`langgraph-checkpoint-postgres` 从 2.0.21 升到 2.0.22（一个 patch 版本），metadata 序列化行为变更，存量快照读写报错，服务启动即炸。

**机制。** LangGraph 的包拆得很细：`langgraph`、`langgraph-checkpoint`、`langgraph-checkpoint-postgres`、`langgraph-sdk` 各自独立发版。主包的版本纪律好（1.0 起承诺无破坏变更），但子包的 patch 版本曾经夹进过行为变更。锁了主版本、放开子版本的项目，一次 `pip install -U` 就中招。

**修复。** 降级回 2.0.21 恢复服务，然后把所有 langgraph 系子包全部锁死精确版本。

**防复发。** 三条纪律：langgraph 全家族包锁精确版本；升级当作变更管理走灰度（先在预发环境用生产快照的副本验证读写）；订阅仓库 release，升级前看子包 changelog 而不是只看主包。

## 事故三：state schema 改了，旧快照全废

**现象。** 给 state 加了新字段、删了一个旧字段，部署新版本后，恢复历史会话的请求全部报反序列化错误。

**机制。** 快照里的 state 按 schema 序列化存储，schema 演进没有官方迁移工具。这是官方文档明确承认的限制：新代码读旧快照没有版本化保障。删字段、改类型是最危险的操作；加带默认值的字段通常安全，但序列化格式变更同样可能出问题。

**缓解。** 治本是接受「checkpoint 是短期资产」这个定位：长期存活的 thread 才需要 schema 兼容纪律，把 thread 生命周期控制住（见事故一），schema 演进的暴露面自然小。工程上再加两条：state 里放一个自写的 `schema_version` 字段，恢复节点读快照时先检查版本做兼容处理；字段只加不删，废弃字段先停止写入、下个版本再删。

## 复盘总结

三起事故指向同一个根因：把 checkpointer 当成了透明的魔法，而它实际是一个跟着你的业务一起演化的数据库。每次评审问三个问题就够：表多大、多快、谁清理；哪个包锁版本、怎么灰度；schema 怎么演进、旧快照怎么办。答不上来任何一条，checkpointer 就是下个季度的故障储备。

## 延伸阅读

- [Scaling LangGraph's Postgres Checkpointer](https://tadeodonegana.com/posts/scaling-langgraph-postgres-checkpointer/)：表增长实测与 TTL 方案
- [Checkpointing Is Not Free](https://pub.towardsai.net/langgraph-checkpointing-is-not-free-a-production-postmortem-398bc86861f4)：同类复盘，生产视角
- [issue #7714](https://github.com/langchain-ai/langgraph/issues/7714)：序列化膨胀的实测数据与讨论
- [Checkpointers 文档](https://docs.langchain.com/oss/python/langgraph/checkpointers.md)：快照结构、DeltaChannel 与清理接口
