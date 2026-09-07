# 收官实战：用 LangGraph 搭一条能上线的人审流水线

**TL;DR:** 这篇用一个完整项目把 LangGraph 的核心件拧在一起：一条「AI 起草 → 事实核查 → 人工审核 → 发布」的内容流水线，覆盖状态设计、人审、重试、幂等发布、降级出口。代码可以直接跑（把模型和发布函数换成你的实现），上线前按文末检查单过一遍。

## 为什么收官选这条流水线

系列写到现在，每个机制都单篇讲过了，收官要回答的是另一个问题：这些件拼在一起长什么样，拼的时候哪些决定在文档里找不到答案。样本的选择标准是「小，但五脏俱全」：流程要短到一屏能画下，又要覆盖四个关键要素：模型调用（会瞬时失败）、错误处理（错误是数据还是异常）、人审（会长时间等待）、不可逆动作（发布出去收不回）。

拿第 01 篇的四个问题给这条流水线打分：断点续跑，是（审批经常跨天）；人审，是（发布前必须人点头）；回放追责，是（发布内容出问题要能查当时哪版草稿、谁批的）；多角色，否（一个流程里的不同工序不等于多角色，第 09 篇讲过这个区别）。三个「是」，这是 LangGraph 的舒适区，也是大多数「AI 内容生产 + 审核」类需求的标准形状。如果你的项目打分比它低，先回第 01 篇重做判断线。

先看图：

```
        ┌──────────┐     事实存疑      ┌──────────┐
START → │ 起草      │ → │ 事实核查     │ ─┐
        └──────────┘     └──────────┘  │ 通过
             ↑      修改意见（≤2 次）    ↓
             └──────────────────── 人工审核 ── 批准 → 发布 → END
                                     │
                                     └─ 拒绝 → END（归档）
```

设计取舍先说清：核查不通过不自动重写（避免无人监督的自我循环），带着问题交给人；人工修改意见回流起草节点，最多两轮，防止无限返工；发布是唯一不可逆动作，幂等键保护。这三个「不」比代码本身更值得读，每一条背后都是一类生产事故，后面逐个拆。

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

## 读图的三个决定

代码逐行读之前，先看图里三个「为什么这样连」，这是文档不会替你做的部分。

**为什么「核查」和「人审」是两个节点，而不是一个节点里先查再问？** 因为它们的失败模式和节奏完全不同：核查是机器调用，秒级返回，可能瞬时失败，需要重试语义；人审是人工操作，小时到天级，需要挂起语义。塞进同一个节点，重试策略没法只作用于核查，interrupt 的恢复重跑也会把核查再跑一遍。节点是部署和故障的基本单位，节奏不同的工作不要共享节点。

**为什么路由用条件边，不写在节点内部？** `fact_check` 查出问题走人审、没问题直发，这个分支逻辑放在条件边里，节点本身保持「只做一件事、只返回数据」。好处在调试时显现：第 14 篇讲过循环图的观测模型，条件边的路由函数是独立的、可以单测的纯函数，给定 state 就能断言走向；写在节点内部，想测「分支对不对」就得把整个节点连同模型调用跑起来。

**为什么拒绝是一个独立的 archive 节点，而不是让路由直接结束？** 因为拒绝也是流程的一等公民：它要留下记录（decision 是 archived 而不是凭空消失）、未来可能挂通知或归档动作。给终态留节点，是给「流程结束后还要发生的事」留位置。

还有一个藏在函数签名里的小决定：两处 `add_conditional_edges` 都显式传了去向列表（`["human_review", "publish"]`、`["publish", "write_draft", "archive"]`）。这不只是参数补全，它是一份可校验的声明：路由函数声称只会去这几个地方，写错了去向名，建图时就报错，而不是等某次路由真的返回那个名字才炸。路由逻辑越复杂，这份声明的价值越大，它是路由函数的单元测试之外的第二道保险，也让人在画流程图时有据可查。

## 状态设计：state 是对外的合同

`PostState` 一共六个字段，这个克制是刻意的。state 里的每个字段都会进每一条 checkpoint（第 03 篇），字段越多，快照越大，schema 演进的包袱越重。更重要的视角：state 是流程对外的合同，恢复流程、排查问题、写测试的人都靠它理解「这个流程的现场是什么」，字段即文档，多余字段即误导。

两个容易被问的点。`fact_issues` 既被核查节点用来存问题、又被 `human_review` 用来存修改意见、又被 `publish` 用来存 URL，一个字段三种用途，是偷懒吗？是小流程的合理妥协：它的语义统一为「带给下一个环节的待办事项」，但生产化时建议拆开，三种用途的生命周期不同，混在一个字段里，排查时会互相干扰。`schema_version` 看起来多余，它是给未来的：这个流程一旦上线，数据库里就躺着中断中的 thread，半年后你改 state 定义时，恢复旧 thread 的代码要靠这个字段判断「这是哪个版本的现场」。第 13 篇讲过 schema 演进没有官方迁移工具，自记版本是最便宜的自保。

还有一条没有出现在代码里、比任何字段都重要的纪律：**state 里没有 secrets**。凭证走 config 注入，不进快照。快照会进数据库、会被回放、会被导出调试，凭证进了 state，等于把它复制到所有这些地方。

## 错误是数据，不是异常

`fact_check` 的行为值得单独一节：它发现事实问题时不抛异常，而是把问题列表写进 state，交给条件边路由。这个设计的因果要讲清楚，因为它和 RetryPolicy 的分工最容易混。

异常适合「这一步没做成」：调用超时、返回不是 JSON、网络断了。这类失败的共性是重试有意义，成功率随次数上升，所以挂 `RetryPolicy(max_attempts=3, initial_interval=1.0)`，三次之后才向上抛。数据适合「这一步做成了，结果是内容有问题」：草稿里有事实错误，这个发现本身就是正常产出，重试一百次也不会让错误消失，把它当异常抛出去，流程中断，问题反而没人处理了。正确动作是把问题变成路由信号：有问题走人审，让看得懂的人决定。

两种机制的分界线一句话：**异常管「做不成」，数据管「做成了但有毛病」。** 混用的代价是具体的：把「内容有问题」实现成异常，你会看到重试白白烧三次模型调用，然后整个 thread 以失败告终，审批人永远看不到那份草稿；把「接口超时」实现成数据（issues 里塞一条「调用失败」），核查的失败就被静默降级成「没有发现问题」，错误草稿直奔发布。两种混用方向，两种生产事故。

顺带说 RetryPolicy 挂在哪：示例只给 `fact_check` 挂了，`write_draft` 同样是模型调用，生产里同样该挂；反过来，`human_review` 和 `publish` 不该挂无脑重试，前者的等待是业务节奏不是故障，后者的重试必须先有幂等保护（见下一节）。「挂在哪些节点」本身就是设计决策，不是统一加上的装饰。

## 人审节点：payload 即产品

`human_review` 是整个图里唯一调用 `interrupt()` 的地方，它的写法浓缩了第 06 篇的教训。

payload 里带了四样东西：类型标识（`content_review`）、修订轮次、草稿全文、事实问题清单。类型标识是给恢复端对账用的：第 06 篇讲过恢复值按发生顺序匹配的风险，payload 自带业务标识，恢复端提交的 dict 里也有标识，节点内校验对上再消费，顺序错位就变成可以防住的问题。草稿和问题清单是给审批人看的：审批界面直接渲染 payload，人看到的是「第二版草稿 + 三个待核实问题」，而不是一个光秃秃的「批准吗」。人审的效率和质量，一半取决于你递过去的信息全不全，payload 就是递东西的托盘。

恢复值约定为 `{"decision": ..., "notes": ...}` 的 dict，同样自带结构。`route_review` 读 `decision` 做三路分发：批准直发；要求修改且轮次未满，回起草；拒绝或轮次用尽，归档。注意「修改意见回流起草」的实现方式：notes 被放进 `fact_issues`，`write_draft` 读到它就把修改要求拼进 prompt。修改和核查错误复用同一条回流通道，图就不用画第二条回边。

`MAX_REVISIONS = 2` 这行值得多看一眼。它是业务层面的返工上限，和 `recursion_limit`（运行时的步数兜底）是两层不同的保险：前者表达「这条业务流程不允许无限返工」，后者防止图本身写出死循环。指望 recursion_limit 兼任返工上限，超限时报出来的是一条运行时异常，审批人视角是「流程崩了」；用 revision 上限，超限走的是 archive 出口，审批人视角是「修改两轮没过，自动归档了」。前者是事故，后者是产品行为。

## 幂等发布：唯一不可逆动作的待遇

`publish` 是全图唯一的不可逆节点，它的幂等键不是可选项。原因藏在上篇讲过的恢复语义里：`interrupt()` 恢复时，整个节点从第一行重跑。设想这个时序：审批人点了批准，恢复触发，`publish` 执行到一半进程崩了。恢复之后，`publish` 从头再跑，如果发布调用没有幂等保护，内容就发出去两份。

幂等键的设计示例用了 `topic:revision`，生产里建议换成更强的键（业务单号或 thread_id 加 revision）：`topic:revision` 在「同一个主题发起两次独立任务」的场景下会误伤，第二次任务的发布会被第一次的记录挡住。幂等键的通用要求就两条：同一逻辑操作恒定不变，不同逻辑操作绝不相同。前面半句防重复发送，后面半句防内容发不出去，两个方向都要测：把 publish 节点手动跑两遍，断言线上只有一份内容；再断言两个不同 thread 的发布互不影响。

同样的逻辑反过来也成立：`archive` 不需要幂等键，因为归档多少次结果都一样。幂等保护只给不可逆动作，不给所有动作，这也是「把复杂度花在刀刃上」。

## 三条正常路径的 state 走查

故障之前，先把正常路径走熟。图的所有行为都体现在 state 的字段变化上，三种结局对应三条走查。

**直发（核查没挑出问题）。** 第一次 invoke：起草产出 draft、revision 变 1；核查返回空列表，`fact_issues` 是空；条件边看到空列表直接路由 publish；发布返回 URL 塞进 `fact_issues`，`decision` 是 published，走到 END。整个过程没有 interrupt，一次 invoke 跑完，这是流水线里唯一「无人工参与」的路径。它也解释了条件边存在的意义：核查这个关卡通过与否，决定流程要不要在人这里停一下，而停不停是路由的事，不是节点的事。

**修改一轮后通过。** 第一次 invoke：起草 revision 1，核查挑出两个问题，路由进人审，图停在 interrupt，此时数据库里的现场是「第一版草稿加两个问题」。审批人提交 `{"decision": "revise", "notes": "续航的说法改掉"}`，恢复触发，human_review 节点从第一行重跑拿到恢复值，`decision` 是 revise、修改意见进了 `fact_issues`；route_review 看到 revise 且 revision 1 小于上限 2，回 write_draft；prompt 这次带着修改意见，产出第二版，revision 变 2；再核查，假设这次干净了，条件边直发 publish，幂等键此时是 topic 加 revision 2。注意这轮全程是恢复触发的那一次 invoke 里连续跑完的：interrupt 恢复后，图从人审节点一直跑到终态才返回，中间不再需要人。

**拒绝归档。** 恢复值是 reject 时，route_review 直接给 archive，`decision` 是 archived，END。thread 的最终快照里留着被拒绝的草稿和拒绝意见，回放时能看到「为什么没发」，这是第 15 篇评测和第 16 篇审计都要用的材料。

三条走查里反复出现的模式值得记住：字段的写入者只有一个（每个字段每轮只有一个节点动它），字段的读者是路由函数和下一个节点。这份「谁写谁读」的清单，就是这张图全部的业务逻辑，也是 code review 时最该盯的部分。

## 从演示到生产：替换清单

这份代码离生产差着的不是功能，是六个占位点。逐个换掉：

1. **`fact_check` 接真实比对源。** 示例让模型裸判断事实，生产里接检索或知识库，把「什么算事实错误」的判定标准写成可审计的规则或至少可复查的引用。模型裸判的核查，错了都不知道错在哪条规则上。
2. **`do_publish` 接真实发布 API，幂等键换业务单号。** 前文说过 `topic:revision` 的误伤场景，换成发号系统给的单号，幂等键的恒定性和唯一性都有保障。
3. **`InMemorySaver` 换 `PostgresSaver`，跑 `setup()` 建表。** 这是跨进程、跨天恢复的前提，剧本一的「从断点继续」全靠它。
4. **凭证确认走 config 注入。** 全文搜一遍 state 字段和 payload 字段，确认没有任何密钥类内容混进去，这是检查单第 5 条的前置。
5. **thread_id 换业务单号。** 示例用了 `post-101` 这种演示编号，生产里 thread_id 就该是业务系统里的单号或内容 ID：运营问「那篇耳机的稿子卡在哪」，答案要能直接查出来，而不是先反查映射表。
6. **模型名和上限参数化。** `MODEL` 和 `MAX_REVISIONS` 写死在代码里，进生产后它们属于配置：换模型是运营决策，返工上限是业务规则，都不该靠改代码发布。

换完这六处，代码的形状没变，但每一步都站在生产的地基上了。检查单的六条管的是「跑起来之后」，这份清单管的是「跑起来之前」，两份合起来才是完整的上线动作。

## 两个生产变体

**变体一：多级审批。** 真实内容团队常有初审和终审两级，比如编辑初审、法务终审。实现冲动是在 human_review 节点里先问一审再问二审，这里第 06 篇的坑三直接适用：一个节点在一轮里只能进一次 interrupt，循环式逐级提问会让重放路径膨胀，是官方点名的反模式。正确拆法是把每一级做成独立节点，各级之间用条件边决定要不要升级：编辑驳回落 archive，编辑通过进法终审，终审通过才 publish。每级一个 interrupt，恢复值按节点隔离，顺序错位问题从根上消失。级别多的团队再把「审批链」整体下沉为子图（第 07 篇），主图只看到「审完了」一个结果。

**变体二：审核分级。** 和多级审批相反方向的需求：低风险内容想跳过人审。做法在条件边的路由函数里加一条规则，核查干净且命中低风险特征（例行政务更新、模板化内容）的直发，其余进人审。代码三行，责任重大：这条规则让一部分内容在无人参与的情况下发布，所以它必须是独立函数、可单测、命中记录可审计，「哪些内容跳过了人审、依据什么」要永远答得上来，第 16 篇的权限分级讲的就是这条纪律。分级规则变更要走和人审流程同级的评审，因为它变更的其实是责任边界。

顺带回应一个第 02 篇读者会问的问题：`revision` 为什么用普通 int 的默认覆盖、而不是挂一个累加 reducer？因为 revision 的语义是「当前是第几版」，是状态量，不是贡献量：回写起草节点重跑时，正确行为就是用返回值覆盖旧值，累加反而会把重跑算成两次。reducer 的选择跟着字段语义走，这张图里没有一个是「多方贡献需合并」的字段，所以一个 reducer 都不需要，这不是没用到第 02 篇，是按第 02 篇的方法判断之后确认用不上。

## 三个故障剧本

代码在 demo 里跑通不说明什么，把它放进三个故障剧本里走一遍，才知道设计决策各就各位没有。

**剧本一：核查服务超时。** `fact_check` 的模型调用连续失败，RetryPolicy 试满三次，节点抛异常，图的这次 invoke 以错误结束。因为有 checkpointer，thread 的现场停在核查之前的边界上，重启服务后重新 invoke 同一个 thread_id，流程从断点继续而不是从起草重来。注意这次「从断点继续」的前提是 checkpointer 接了真数据库：InMemorySaver 的现场随进程死掉，剧本一就直接退化成从头再来。

**剧本二：审批人休假三天。** thread 停在 `human_review` 的 interrupt 上，payload 在数据库里躺着。没有兜底的话它躺多久都行，审核 SLA 就是一句空话。兜底是图外的一个扫描任务：找出停在人审超过 SLA 的 thread，用 `Command(resume=...)` 自动注入拒绝决定并通知，走的是 archive 出口。两个细节：自动注入的是拒绝不是批准，宁可不发不能错发；扫描任务的 resume 值也要带业务标识，让它和真人审批在节点内走同一条校验路径。

**剧本三：发布服务在审批通过后宕机。** 恢复触发了，`publish` 跑了一半，进程没了。进程回来后重新恢复该 thread，publish 从头执行，幂等键保证第二次调用被发布服务识别为重复，返回第一次的 URL，内容只有一份。这个剧本是幂等键存在的理由，也是「不可逆动作必须有幂等保护」的完整演示：重试不可避免（恢复就是重跑），可重复的后果才是要消灭的东西。

三个剧本对应三类机制：RetryPolicy 对抗瞬时失败，超时兜底对抗无限等待，幂等键对抗重复执行。一份流水线代码里三类都有，才算「能上线」。

**剧本四：两个人同时点了批准。** 审批人 A 点批准的同时，超时扫描任务也判定这条超 SLA 注入了拒绝，两个 `Command(resume=...)` 先后到达同一个 thread。第一个恢复生效，节点消费决策走完流程到 END；第二个恢复到达时，thread 已经没有等待中的 interrupt，恢复调用要么报错要么被拒，流程状态不会被二次修改，这是 thread 语义给的隔离。要在应用层做的是把这类冲突变成可理解的提示：恢复前先查 thread 状态，已终态的返回「该单已处理」，并把第二个操作记录进审计日志。并发恢复不值得恐惧，值得的是想清楚「第二个人会看到什么」。

## 上线前检查单

代码之外，这六条逐条过。

1. **人审超时兜底。** 就是剧本二的扫描任务，确认它存在、有 SLA 配置、注入的是拒绝。没有这一条，审核积压会在第一个长假集中爆发。
2. **checkpoint 清理。** thread 结束后快照还在表里，日积月累撑爆数据库（第 13 篇是一次真实事故复盘）。确认归档或删除策略存在且执行过。
3. **版本锁定。** langgraph 全家族包 pin 死，升级走预发验证（第 13 篇的 patch 事故和第 18 篇的 `-U` 事故是同一条纪律的两个案例）。
4. **评测基线。** 录 20 条历史用例，改 prompt 后回放对比轨迹（第 15 篇的方法）。人审能拦住坏草稿，但拦不住「起草质量悄悄变差」，质量漂移只有回放对比能发现。
5. **权限核对。** 发布工具只对授权用户可见（第 16 篇的权限分级），审批操作有审计日志且独立落表。「谁批准了这条内容」将来一定会被问，答案不能只在应用日志里。
6. **监控接好。** superstep 事件或 LangSmith trace 至少接一个（第 14 篇）。盲跑等于裸奔：流程停在 interrupt 是正常，停在异常是事故，没有观测手段时分不清这两者。

## 扩展方向

业务长起来之后的三个常见演进，都是在现有图上加件，不用推翻结构。

**多内容形态。** 长文、短视频脚本、社媒帖子各有各的核查和发布规则，每种形态一个独立子图，主图按形态路由（第 07 篇的子图隔离与透传）。子图的好处是各形态的 state 互不污染，代价是透传配置要想清楚。

**审核分级。** 低风险内容（比如已发布内容的例行更新）免人审直发，规则写在条件边的路由函数里。这里有个值得坚持的原则：降级的判定规则要独立成函数、可单测、有审计记录，「什么内容跳过了人审」必须永远答得上来，这是第 16 篇安全边界的要求。

**批量生产。** 上百个主题一次跑，条件边返回 Send 列表并行起草（第 08 篇的 map-reduce），人审集中在汇总节点一次收齐全部决策、按 id 映射恢复（第 06 篇的坑六）。批量化的最大风险是把「一次性批准一百条」做成「批准一百次」，前者的审批界面要按批聚合风险，这个设计决定比并行本身的代码难。

最后一个方向是「不扩展」。上面三个方向都以业务量增长为前提，量没到的时候提前做，得到的不是能力是负担：子图加一层透传配置，分级规则多一套要维护的审计，批量并行多一整类合并 bug。这份代码的可扩展性体现在「加件不用推翻」，不代表「件要提前加上」。判断标准和第 01 篇一脉相承：等需求出现再加，图的结构不会因为晚加而变贵。

## 这份代码和系列的对照

收官篇的另一个用处：把系列二十篇折回这一份代码里，复习时按图索骥。用法建议反过来用更好：先自己指着代码问「这一行对应哪篇讲的什么机制」，答不上来的再回去翻，比顺序重读快得多，也诚实地暴露出哪些篇当时就没读透。

- `State` 与字段合并的规则：第 02 篇（本例全部用默认覆盖，没有增量需求，正是「简单流程不欠 reducer 债」的示范）
- `InMemorySaver` 与 thread：第 03 篇；换 PostgresSaver 后的运维：第 13 篇
- `interrupt` 与恢复值：第 06 篇，payload 带标识即坑二的解法
- `RetryPolicy` 与错误分类：第 12 篇，「错误是数据」即四类错误的归属示范
- 条件边与路由：第 14 篇的可单测路由；批量后的 Send：第 08 篇
- 检查单的权限与审计：第 16 篇；评测回放：第 15 篇；部署形态：第 17 篇；版本锁定：第 18 篇
- 「要不要这条流水线」的判断本身：第 01 篇和第 19 篇

对照里最值得注意的是第 02 篇那一行：一份能上线的人审流水线，一个自定义 reducer 都没用上。框架的核心概念不等于必选清单，按需取用才说明你真懂了每一件东西的适用边界。

这条流水线不复杂，复杂的是让它稳定跑一年的那圈工程：真数据库、清理策略、超时兜底、幂等纪律、评测基线。框架把最难写的恢复语义给了你，剩下这一圈是任何框架都替代不了的部分，也是这个系列真正想交出去的东西。二十篇讲的是 LangGraph，最后落在代码外的这半页纸，如果你读完只记住一件事，记住这个分配：图里的逻辑交给框架，图外的责任留给自己，两边都别指望对方兜底。

## 延伸阅读

- [Human-in-the-loop 文档](https://docs.langchain.com/oss/python/langgraph/interrupts.md)：interrupt 的完整语义
- [Persistence 文档](https://docs.langchain.com/oss/python/langgraph/persistence.md)：PostgresSaver 与 thread 管理
- [Fault Tolerance 文档](https://docs.langchain.com/oss/python/langgraph/fault-tolerance.md)：RetryPolicy 与补偿
- [Graph API 文档](https://docs.langchain.com/oss/python/langgraph/graph-api.md)：条件边与 Send
