# 没有评测的 LangGraph Agent 只是 demo：评测与回归实战

**TL;DR:** Agent 应用「demo 阶段永远是对的，上线就开始崩」，原因是没人定义过「对」。评测要分三层：最终答案、执行轨迹、工具调用参数。LangGraph 独有的优势是 checkpoint 回放可以当回归工具用：从同一张快照出发换模型、换 prompt 重跑，diff 出行为变化。工具选型上 LangSmith 最顺但有绑定和账单，DeepEval、Promptfoo 是可用的开源替代。

## 先定义「对」：三层评测对象

「agent 回答错了」没法进 CI。把它拆成可断言的三层：

1. **轨迹层（最重要）。** 执行路径对不对：该调工具的时候调了吗？走的哪个分支？循环了几次？轨迹错但答案碰巧对的用例，是上线后最不可控的那类。
2. **工具参数层。** 调对了工具、传错了参数，等价于错。参数是结构化的，能精确断言，是性价比最高的评测层。
3. **答案层。** 最终输出是否正确、格式是否合规。文本答案的判定需要 LLM 当裁判（LLM-as-judge）或人工抽检，成本最高放最外层。

排序就是投入顺序：先把轨迹和参数断言建起来（纯代码，确定性，跑得快），答案层只对核心场景配裁判模型。

## LangGraph 独有的武器：回放当回归

一般框架做回归，输入相同、执行路径靠运气。LangGraph 的 checkpoint 让「从执行中间的某一点开始重跑」成为原生能力：

```python
# 从历史快照出发，用新版本的图重跑，对比新旧轨迹
config = {"configurable": {"thread_id": "case-17",
                           "checkpoint_id": snapshot_id}}
new_result = new_app.invoke(None, config)
old_result = load_recorded("case-17.json")   # 上次记录的轨迹与答案
diff_traj(old_result, new_result)
```

具体做法：录制阶段，每个评测用例跑一遍，把最终状态和轨迹存档；每次改完 prompt、模型或图结构，从存档快照重跑，对比轨迹差异。模型行为不确定导致答案可能不同，但轨迹差异能告诉你「新版本多绕了一圈」「换了个工具」，这正是回归要抓的。这套机制别的框架给不了，是 LangGraph 用户最该占的便宜。

## 工具选型：绑定与账单要摆在台面上

**LangSmith**：和数据集、trace、线上监控是一体的，做 LLM-as-judge 和轨迹评分的配套最全。两个代价：闭源 SaaS，trace 数据出你的环境；收费（按席位加用量，价格以官方页为准）。团队已经在用它的 trace，评测顺手就做了，是最短路径。

**DeepEval**：开源，agent 评测指标齐全（任务完成、参数正确性），可进 CI。

**Promptfoo**：开源 CLI，适合 prompt 回归、红队用例和 CI 集成，配置是声明式的。

不绑定任何一家的组合也完全成立：录制回放自己写（两百行以内），轨迹断言用 pytest，裁判模型只在答案层用。本仓库的 [评测专题](../../evaluation/README.md) 有模型、RAG、Agent 各维度的评测方法地图，评测协议的坑（裁判不稳定、用例泄漏）在那里有展开，不重复。

## 进 CI 的最小方案

每次提交跑得起的版本：

1. 节点单测 + 路由断言（毫秒级，必过）。
2. 假模型跑核心轨迹用例（秒级，拓扑和分支覆盖）。
3. 固定 20 到 50 条录制用例回放，只断言轨迹和参数（分钟级）。
4. 答案层 LLM 裁判不进每次提交，每天定时跑，输出趋势而不是门禁。

先有第 1、2 条的团队很多，卡在第 3 条：录制用例要维护。省力做法是从真实线上失败里沉淀用例，每次事故转一条录制用例，评测集跟着事故走，半年后它就是你们系统最值钱的资产。

## 权衡

评测体系最大的成本不是工具，是用例的持续维护。轨迹断言写得太死（逐条消息比对），模型一升级全线红；写得太松（只看最终状态），bug 全漏。经验值是断言「关键决策点」：调了哪些工具、走了哪些分支、参数里的关键字段，消息措辞不比。另外回放回归有个盲区：从中间快照重跑时，上游真实世界的状态可能已经变了（库存、汇率），录制环境要能 mock 掉这类外部依赖，否则回归集永远假红。

## 延伸阅读

- [本仓库评测专题](../../evaluation/README.md)：模型 / RAG / Agent 评测方法地图
- [LangSmith Evaluation](https://docs.langchain.com/oss/python/langchain/evals)：trajectory evaluation 官方文档
- [DeepEval Agents](https://deepeval.com/docs/getting-started-agents)：开源 agent 评测指标
- [Use Time Travel 指南](https://docs.langchain.com/oss/python/langgraph/use-time-travel.md)：回放机制的操作细节
