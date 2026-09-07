# 自部署还是买平台：LangGraph 部署形态与成本账

**TL;DR:** 先纠正名词：社区说的「LangGraph Platform」在官方语境里已经演进为 Agent Server（运行时）和 LangSmith Deployment（托管产品），选型文章里还在用旧名词的多半内容也旧了。四种部署形态从轻到重：开源库自写 API（免费）、Agent Server 自托管（自带 PG/Redis）、LangSmith Cloud（按席位）、企业自管控面（Enterprise）。成本上最容易被低估的不是许可费，是自托管的运维清单。

## 名词先对齐

选型前先统一语言，2025 到 2026 年这组名词换过一轮：

- **LangGraph（开源库）**：pip 装的那个，本文所有文章讲的东西，免费 MIT。
- **Agent Server**：统一运行时，提供 assistants / threads / runs 这套 API，`langgraph dev` 本地起的就是它。它让 Studio、前端 SDK 有了标准对接面。
- **LangSmith Deployment**：托管产品（旧名 LangGraph Platform），跑在 LangChain 管的云上或你的 VPC 里。

中文社区至今大量文章把三者混着叫，引用时先看作者说的到底是哪一层。

## 四种形态，各自成立的前提

**形态一：开源库 + 自写 FastAPI。** `langgraph` 是普通库，编译出来的图在任意 web 框架里 invoke，Postgres 自己接。免费、代码全在自己手里、没有额外 API 面。适合流程简单、不需要线程级 API 和 Studio 的团队。缺点：threads 的并发管理、流的对接协议都要自己写，等于放弃生态现成件。

**形态二：Agent Server 自托管。** 官方 Docker 镜像跑在你的 K8s 里，自带 Postgres 和 Redis 依赖，你的图变成标准的 threads/runs API，Studio 可以直接连上来调试。这是「要生态但数据不出门」的主答案。注意许可：standalone 部署形态需要 LangSmith license，别以为带 Docker 就是免费自托管。

**形态三：LangSmith Deployment Cloud。** 官方托管，不用管任何运维，控制台和 trace 无缝。前提是 LangSmith Plus 以上的席位，数据进 LangChain 的云。

**形态四：self-hosted control plane / hybrid。** 控制面和数据面拆开，跑在企业自己的 VPC，合规数据不出门。Enterprise 合同，小团队不用看。

## 成本账，把两列都算上

**显性成本**（2026 年 9 月官方定价页口径，随时会变，决策前自己核对）：LangSmith 按席位订阅（开发者席约 39 美元/月/席），trace 用量另计，百万条 trace 量级在数千美元/月；Deployment Cloud 包含在相应套餐里；自托管的 standalone 形态要 license。

**隐性成本**（自托管形态的真实大头）：Postgres 和 Redis 的运维（备份、扩容、监控，checkpoint 表的清理策略要有人管）、版本升级的回归测试、故障时没有官方 SLA。形态二省下的许可费，通常以半个工程师的方式花出去。云端形态反过来：钱花在明处，运维归零，但 trace 和运行数据都在别人云上。

一个务实的算例口径：5 人团队、日活千级的应用，形态一 + 自建监控的现金成本最低；加要 Studio 调试体验，形态二的 license + 运维成本往往超过直接上形态三的席位费，除非合规硬性要求。

## 决策线

按顺序问：

1. 需要标准 threads/runs API 和 Studio 吗？不需要，形态一，到此结束。
2. 数据能不能出你的环境？不能，形态二（有 license 预算）或形态四（Enterprise）。
3. 有没有运维人力？没有，形态三，把隐性成本变成显性订阅费。

本地开发不分形态：`langgraph dev` 起本地 Agent Server，Studio 免费连，这条链路对四种最终形态都是一致的，前期不存在锁死。

## 权衡

最大的选型风险不是选贵了，是形态一的隐藏工作量被低估：团队说「我们就自己写个 API」，三个月后发现自己实现了半个 threads API（断线重连、并发 run、流协议），这时候迁移到形态二或三，checkpoint 数据和调用方都要动。反过来的风险是形态三的粘性：trace、评测、部署全在一家的云上，商业关系变化时的退出成本要认真评估。给中间态留一条路：形态一起步，但业务代码只用公开 API 面（不 hack 内部结构），保证哪天迁去 Agent Server 时代码不动。

## 延伸阅读

- [Deployment 文档](https://docs.langchain.com/langsmith/deployment)：四种形态的官方说明
- [Standalone Server 部署指南](https://docs.langchain.com/langsmith/deploy-standalone-server)：形态二的具体要求
- [LangSmith 定价页](https://www.langchain.com/pricing)：当前价格，决策前核对
