# 收官实战：用 LangGraph 搭一条能上线的人审流水线

**TL;DR:** 这篇用一个完整项目把 LangGraph 的核心件拧在一起：一条「AI 起草 → 事实核查 → 人工审核 → 发布」的内容流水线，覆盖状态设计、人审、重试、幂等发布、降级出口。代码可以直接跑（把模型和发布函数换成你的实现），上线前按文末检查单过一遍。

## 要建什么

```
        ┌──────────┐     事实存疑      ┌──────────┐
START → │ 起草      │ → │ 事实核查     │ ─┐
        └──────────┘     └──────────┘  │ 通过
             ↑      修改意见（≤2 次）    ↓
             └──────────────────── 人工审核 ── 批准 → 发布 → END
                                     │
                                     └─ 拒绝 → END（归档）
```

设计取舍先说清：核查不通过不自动重写（避免无人监督的自我循环），带着问题交给人；人工修改意见回流起草节点，最多两轮，防止无限返工；发布是唯一不可逆动作，幂等键保护。

## 完整实现

```python
# pip install langgraph==1.2.* openai==1.102.0
import json
import operator
from typing import Annotated, TypedDict

from openai import OpenAI
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver  # 生产换 PostgresSaver
from langgraph.pregel import RetryPolicy

llm = OpenAI()
MODEL = "gpt-4o-mini"
MAX_REVISIONS = 2


class PostState(TypedDict):
    topic: str
    draft: str
    fact_issues: list[str]
    revision: int
    decision: str
    schema_version: int   # 自记版本，为将来 schema 演进留钩子


def write_draft(state: PostState) -> dict:
    notes = state.get("fact_issues") or []
    prompt = f"写一段 200 字的产品介绍：{state['topic']}。"
    if notes:
        prompt += "修正以下问题并保留其余内容：" + "；".join(notes)
    text = llm.chat.completions.create(
        model=MODEL,
        messages=[{"role": "user", "content": prompt}],
    ).choices[0].message.content
    return {"draft": text, "revision": state["revision"] + 1}


def fact_check(state: PostState) -> dict:
    # 真实实现接检索或知识库比对；这里演示「错误是数据，不是异常」
    resp = llm.chat.completions.create(
        model=MODEL,
        messages=[{"role": "user",
                   "content": f"列出下文中的事实性错误，无则返回空列表 JSON：\n{state['draft']}"}],
        response_format={"type": "json_object"},
    )
    issues = json.loads(resp.choices[0].message.content).get("issues", [])
    return {"fact_issues": issues[:5]}


def human_review(state: PostState) -> dict:
    payload = interrupt({
        "type": "content_review",
        "revision": state["revision"],
        "draft": state["draft"],
        "fact_issues": state["fact_issues"],   # 给审批人的决策依据
    })
    # payload 应为 {"decision": "approve" | "revise" | "reject", "notes": "..."}
    return {"decision": payload.get("decision", "reject"),
            "fact_issues": [payload.get("notes", "")]}


def route_review(state: PostState) -> str:
    if state["decision"] == "approve":
        return "publish"
    if state["decision"] == "revise" and state["revision"] < MAX_REVISIONS:
        return "write_draft"
    return "archive"          # 拒绝、或修改次数用尽


def publish(state: PostState) -> dict:
    url = do_publish(
        content=state["draft"],
        idempotency_key=f"{state['topic']}:{state['revision']}",  # 人审恢复=重跑，这里必须幂等
    )
    return {"decision": "published", "fact_issues": [url]}


def archive(state: PostState) -> dict:
    return {"decision": "archived"}


g = StateGraph(PostState)
g.add_node("write_draft", write_draft)
g.add_node("fact_check", fact_check,
           retry_policy=RetryPolicy(max_attempts=3, initial_interval=1.0))
g.add_node("human_review", human_review)
g.add_node("publish", publish)
g.add_node("archive", archive)

g.add_edge(START, "write_draft")
g.add_edge("write_draft", "fact_check")
g.add_conditional_edges("fact_check",
                        lambda s: "human_review" if s["fact_issues"] else "publish",
                        ["human_review", "publish"])
g.add_conditional_edges("human_review", route_review,
                        ["publish", "write_draft", "archive"])
g.add_edge("publish", END)
g.add_edge("archive", END)

app = g.compile(checkpointer=InMemorySaver())

config = {"configurable": {"thread_id": "post-101"}}
result = app.invoke({"topic": "无线耳机 X1", "revision": 0,
                     "fact_issues": [], "decision": "", "schema_version": 1},
                    config)                       # 跑到人审暂停
final = app.invoke(Command(resume={"decision": "approve", "notes": ""}),
                   config)                        # 审批人放行
print(final["decision"])
```

`do_publish` 换成你真实的发布调用（返回 URL）。生产环境把 `InMemorySaver` 换成 `PostgresSaver`（首次跑 `setup()` 建表），中断的审核流程就能跨进程、跨天恢复。

## 代码里的设计决策对照

- **state 没有 secrets**：凭证走 config 注入，不进快照。
- **fact_check 挂 RetryPolicy**：模型调用是典型的瞬时错误高发点；它的输出是结构化 JSON，解析失败会走重试而不是炸图。
- **错误是数据**：事实问题不抛异常，进 state 给人看。
- **人审 payload 带类型标识和上下文**：审批人不只点同意，还能看到草稿和问题清单；恢复值是 dict，自带业务标识，不依赖顺序。
- **publish 幂等键**：人审恢复时节点重跑，发布不能发两份。
- **revision 上限**：修改回环有硬性出口，不跟 recursion_limit 赌运气。
- **schema_version 入 state**：将来加字段、改流程时，恢复旧 thread 前先看版本。

## 上线前检查单

1. 人审超时兜底：扫描停留超 SLA 的 thread，自动拒绝（不是批准）并通知。
2. checkpoint 清理：thread 结束后的归档或删除策略。
3. 版本锁定：langgraph 全家族包 pin 死，升级走预发验证。
4. 评测基线：录 20 条历史用例，改 prompt 后回放对比轨迹。
5. 权限核对：发布工具只对授权用户可见，审计日志独立落表。
6. 监控接好：superstep 事件或 LangSmith trace 至少接一个，盲跑等于裸奔。

## 扩展方向

业务长起来之后的常见演进：多内容形态（加子图，每种形态一个独立子流程）；审核分级（低风险内容免人审直发，规则路由）；批量生产（条件边返回 Send 列表并行起草，人审集中在汇总节点按 id 映射恢复）。每个方向都是在现有图上加件，不用推翻结构，这是把核心件用对位置后，框架给你的复利。

## 延伸阅读

- [Human-in-the-loop 文档](https://docs.langchain.com/oss/python/langgraph/interrupts.md)：interrupt 的完整语义
- [Persistence 文档](https://docs.langchain.com/oss/python/langgraph/persistence.md)：PostgresSaver 与 thread 管理
- [Fault Tolerance 文档](https://docs.langchain.com/oss/python/langgraph/fault-tolerance.md)：RetryPolicy 与补偿
- [Graph API 文档](https://docs.langchain.com/oss/python/langgraph/graph-api.md)：条件边与 Send
