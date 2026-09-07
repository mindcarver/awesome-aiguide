# 旧教程全过期了：LangGraph 0.x → v1 迁移实录

**TL;DR:** 2025 年 10 月的 LangGraph 1.0 是分水岭：`create_react_agent` 废弃、预置类型大清洗、Python 3.9 出局，而中文互联网的主流教程和三个翻译站还停在 0.x。这篇给一条五步迁移路径，加上社区实测出来的坑：最疼的不是 API 改名，是 `pip install -U` 之后 import 直接炸。

## 先判断：迁、不迁、还是绕

三种情况。存量项目、还要继续演进：迁，0.x 已进维护尾声，越拖迁移面越大。存量项目、功能冻结：不迁，锁死全家族版本继续跑，把精力省下来。新项目：没有选择，直接 1.x，别抄旧教程起手。

1.0 官方承诺了 1.x 内无破坏性变更，这是迁移窗口的保证：迁到 1.x 之后，至少到 2.0 之前不会再有这一轮规模的折腾。0.x 没有这个承诺。

## 五步迁移法

**第一步：冻结现状，让测试全绿。** 迁移前把当前版本全家族锁死（`langgraph`、`langgraph-checkpoint`、`langgraph-checkpoint-postgres`、`langchain` 各自 pin），补齐核心路径的测试。没有测试的迁移等于盲迁，出问题连回归基线都没有。

**第二步：换掉 create_react_agent。** 高频改动就这两行：

```python
# 0.x（已废弃）
from langgraph.prebuilt import create_react_agent
agent = create_react_agent(model, tools, prompt="你是客服")

# 1.x
from langchain.agents import create_agent
agent = create_agent(model, tools, system_prompt="你是客服")
```

注意两处：包名从 `langgraph.prebuilt` 换到 `langchain.agents`，参数名 `prompt=` 换成 `system_prompt=`。原来靠 `prompt` 参数注入的复杂提示逻辑，1.x 的做法是写 middleware，而不是继续堆字符串。

**第三步：人审相关类型逐个对。** 这是改动最碎的部分，因为类型在两个来源里：`langgraph.types` 里的 interrupt / Command 还在，但 0.x 预置的 `HumanInterrupt`、`HumanInterruptConfig`、`ActionRequest` 被移除，相关能力并入 `langchain.agents` 的 middleware 体系（如 `HumanInTheLoopMiddleware`）。旧代码里凡是 import 了这组名字的，都要重写成人审中间件或裸 `interrupt()`。

**第四步：清扫移除清单。** `AgentState`（含 Pydantic 版，1.0 的口径是不再推荐 Pydantic 做 state）、`ValidationNode`、`MessageGraph` 全部出局。逐个 grep 旧 import，命中即改。

**第五步：Python 版本。** 3.9 支持已移除，运行时和 CI 的矩阵一起升。

## 社区实测的坑

**`pip install -U` 即刻炸 import。** 升级主包会把 LangChain 1.0 一并拉进来，旧的 `langchain` 链式 import 立刻断。社区里「升级完跑不起来」的帖子密度很高，解法只有一条：全家族精确锁版本，升级走变更流程，别用 `-U` 裸升。

**DeprecationWarning 不是安全期。** 0.x 后期的警告意味着 1.0 移除，看到警告就该排期改，而不是压掉警告继续跑。

**多 agent 项目是重灾区。** 单 agent 改两行完事，多 agent 项目里 prebuilt 的 supervisor/swarm 写法、状态传递、人审类型缠在一起，社区有迁移到一半卡住求援的长帖。这类项目的正确姿势是逐个子图迁移、每个子图过完测试再动下一个，中途保持新旧代码能共存（用版本锁把共存环境钉住）。

## 旧教程识别指南

拿到一篇 2025 年之前的教程或一份翻译站内容，看三个信号判断是否已过期：import 里有 `from langgraph.prebuilt import create_react_agent`，0.x 无疑；文档域名是 `langchain-ai.github.io/langgraph`，旧站已停止更新，正身在 `docs.langchain.com`；人审示例用 `interrupt_before` 而不是 `interrupt()`，旧范式，官方已明确不推荐用于 HITL。命中任意一条，只当概念参考，代码不要抄。

## 权衡

迁移的真实成本分布很不均匀：标准用法半天，人审重度项目一两周。评估时按「人审类型引用量 × 多 agent 复杂度」估，不要按代码行数。另一个值得记录的观察：这一轮迁移后官方把「预置能力」从 LangGraph 挪去了 LangChain，意味着以后追 changelog 要看两个仓库，升级 checklist 里应该把 `langchain` 的 release note 加进去，这是 0.x 时代没有的心智负担。

## 延伸阅读

- [LangGraph v1 迁移指南](https://docs.langchain.com/oss/python/migrate/langgraph-v1)：移除清单与替代写法的官方对照
- [LangChain 1.0 发布博客](https://www.langchain.com/blog/langchain-langgraph-1dot0)：架构调整的官方口径
- [LangChain 1.0 版本说明](https://docs.langchain.com/oss/python/releases/langchain-v1)：middleware 与 create_agent 的新体系
