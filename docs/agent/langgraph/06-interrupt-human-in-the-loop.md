# 人审不是加个按钮：LangGraph interrupt 落地的六个坑

**TL;DR:** `interrupt()` 五分钟能跑通 demo，生产化要跨过六个坑：别用 try/except 包它、恢复值按索引匹配、一个节点每轮只能 interrupt 一次、payload 必须可序列化、前置副作用要幂等、并行分支要按 id 映射恢复。六个坑背后是同一条机制：恢复时整个节点从头重跑。外加一个框架不管的问题：人一直不批，谁来兜底。

## 人审是运行时里最难的一种状态

把一个正在执行的流程暂停下来等人，这件事的难度被 demo 严重低估了。暂停一个函数很容易，sleep 就行；难的是暂停之后：进程可能重启，等待可能持续三天，恢复时操作的可能是另一个进程。这意味着「执行到一半的现场」必须存在进程外，而且要能被一个和暂停时完全无关的进程接着跑。所以人审天然绑着持久化：没有存档，就没有真正的暂停，只有一个占着内存的 sleep。

场景：假设你在做采购系统里的付款 agent，要执行一笔 5 万的转账，公司规则是金额超过 5000 必须财务总监点头。这个「点头」不是前端弹个确认框，而是整个流程真正停住，等一个可能几小时后才发生的人工决定，等到了还要从停住的那个位置继续，而不是从头再跑一遍。

先把最小可用的人审立起来：

```python
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

def approve_transfer(state: MessagesState) -> dict:
    decision = interrupt({
        "type": "transfer_approval",
        "amount": 50000,
        "question": "是否批准向供应商 A 转账 5 万元？",
    })
    # 人处理后从这里继续，decision 就是 resume 传进来的值
    if decision == "approve":
        return {"messages": [("assistant", "已批准，执行转账")]}
    return {"messages": [("assistant", "已拒绝，流程终止")]}

g = StateGraph(MessagesState)
g.add_node("approve", approve_transfer)
g.add_edge(START, "approve")
g.add_edge("approve", END)
app = g.compile(checkpointer=InMemorySaver())

config = {"configurable": {"thread_id": "po-1"}}
app.invoke({"messages": [("user", "申请转账")]}, config)  # 暂停在 interrupt
app.invoke(Command(resume="approve"), config)             # 人批准后恢复
```

两个硬前提藏在 demo 里，生产上漏掉任何一个人审就是坏的：图必须挂 checkpointer，因为暂停的本质是存档，不挂 checkpointer，暂停状态无处安放；调用必须带 thread_id，因为审批对象是「某一次具体的运行」，不是这段代码本身，同一个图同时有一百笔转账在等审批，靠 thread_id 分辨谁是谁。

然后是全文最重要的一句话：`interrupt()` 把当前节点挂起，恢复时**整个节点从第一行重跑**，跑到 `interrupt()` 那一行时拿到 resume 值继续往下。

为什么是重跑而不是从暂停处接着走？因为挂起的实现方式决定了这一点。节点函数不是被「冻结在原地」的，而是通过抛出一个特殊异常把控制权交还运行时，运行时把当前状态写进 checkpoint 后返回。等人处理完，运行时拿着同一个 checkpoint 重新进入节点函数，把之前执行过的代码原样再走一遍，走到 `interrupt()` 时不再暂停，而是把存的 resume 值当作这次调用的返回值给出去。

这个设计直接解释了 demo 里看不到的一件事：恢复的调用者和暂停的进程可以是两个完全无关的进程。暂停那晚的进程早就随发版消失了，三天后处理审批的服务拿 thread_id 加 `Command(resume=...)` 发起恢复，照样能接着跑，因为全部现场都在 checkpoint 里，没有一样东西依赖当初那块内存。多副本部署、滚动发版、审批服务与应用服务分离，这些生产架构在人审场景能成立，靠的就是这个把状态彻底放在进程外的设计。

理解了这个机制，读人审代码时要做一个转换：`interrupt()` 之前的每一行，在暂停轮和恢复轮各执行一次；`interrupt()` 之后的每一行，只在恢复轮执行一次。

还有一个容易被忽略的细节：节点没有执行到 return，它内部做过的修改就一律没有提交。state 是按节点为边界合并的，节点中途暂停，这轮里算出来的局部变量、改过的临时结构全部消失，恢复轮从函数第一行重新开始算。所以 interrupt 之前的代码要按「会执行两遍、且两遍之间不共享任何局部状态」来写，贵的计算能挪到 interrupt 之后挪，挪不了的要接受重算。这不只是性能问题，它和坑五的幂等纪律是同一条根。

## 恢复时的世界，已经不是暂停时的世界

暂停三天再恢复，变的可能不只是流程，还有流程依赖的一切。价格调了，库存没了，那张采购单在 ERP 里被人关掉了，甚至审批人本人已经没有这笔金额的审批权限。恢复轮的代码如果拿着暂停前的假设直接往下走，就是在用三天的旧地图走新路。

所以恢复轮的第一件事不是继续执行，而是重新校验业务前提：单据状态还有效吗，金额有没有变，审批人还该不该审这一单。校验不通过就走异常路径，把 thread 标成需要人工重审，而不是硬着头皮执行一个前提已经塌掉的动作。把这条写进人审节点的设计模板里，它值得占模板的第一行。

反过来，暂停时长本身也是信息。checkpoint 带时间戳，审批发生在三天前还是三秒后，对风控的意义完全不同。把「发起时间、暂停时间、恢复时间」记进审计，出事复盘时这三段时间线是第一手证据，而这些 checkpoint 里本来就有，缺的只是你把它查出来用。第 03 篇拆解 checkpoint 里到底存了什么，那里面的东西比多数人用到的多。

接下来是六个坑。前五个是「恢复等于重跑」这条机制的直接推论，第六个是它在并行场景的放大版。

## 坑一：用 try/except 把 interrupt 包住了

最常见的写法是想给节点加「统一错误处理」：

```python
# 错误示范
def approve_transfer(state: MessagesState) -> dict:
    try:
        decision = interrupt({"question": "是否批准？"})
        return do_transfer(decision)
    except Exception as e:
        return {"messages": [("assistant", f"执行失败：{e}")]}
```

跑起来不报错，行为却完全不对：人审要么静默失效（异常被吞掉，流程「正常」走完），要么节点行为错乱。根源在上一节讲的机制：`interrupt()` 暂停的实现就是抛一个特殊异常（`GraphInterrupt`）把控制权交还运行时。你的 `except Exception` 不知道这个约定，把它当成普通错误捕获了，运行时永远收不到「该暂停了」的信号。

修复有两条路，按顺序考虑。第一，错误处理的范围别覆盖 interrupt 调用处，把 `interrupt()` 和它之后的决策逻辑放在 try 块外面，try 只包真正可能抛业务异常的外部调用。第二，确实要包整段逻辑（比如接了统一的节点级 error_handler），捕获之后判断异常类型，`GraphInterrupt` 原样 re-raise，只处理真正的异常。第 12 篇讲的节点级错误处理和重试策略也属于这一类：任何「统一兜住所有异常」的机制都是这个坑的变体，接入之前先确认它对 interrupt 特殊处理了。

判断一个团队的人审代码是否踩了这坑，有个快速办法：把审批人换成永远不会点批准的测试账号，流程如果还是跑到了终点，说明暂停信号在半路被吃了。

## 坑二：恢复值按索引匹配，顺序一变就对错位

一个流程里有多处 interrupt 时，resume 值的对齐方式是按**发生的顺序**依次消费，不是按你给 interrupt 起的名字，也没有任何参数让你指定「这个 resume 值给哪个 interrupt」。假设一个采购流程有两处人审：先部门经理批预算，再财务批付款，代码里写了两处 `interrupt()`。正常情况下，第一次 `Command(resume="ok")` 给第一处，第二次给第二处，一切正常。

事故发生在流程变更之后：有人在最前面加了一个「供应商黑名单校验不通过时额外人工确认」的分支。这个分支一个月只触发一两次，但只要它触发，后面所有审批的答案就整体错位一格：黑名单确认拿到了部门经理的「同意」，部门经理节点拿到了财务的「同意」，财务节点拿到了空。没有报错，钱按错误的确认流转。这是社区 issue 里最高频的人审事故类型，而且它的恶心之处在于测试环境几乎复现不出来：测试数据不触发新加的分支。

修复模式是让 resume 值自带身份，不依赖顺序。每个 interrupt 的 payload 里放业务唯一标识（审批单号、动作类型），前端或工单系统把标识原样带回 resume 值里，节点内校验对上了才消费，对不上就报错让人工介入：

```python
decision = interrupt({"type": "payment_approval", "po_id": "PO-2024-0891"})
# resume 值要原样带回 type 和 po_id，两个都对上才消费
if (decision.get("type") != "payment_approval"
        or decision.get("po_id") != "PO-2024-0891"):
    raise ValueError(f"审批答复错位：收到 {decision.get('type')}")
```

看起来多写了三行，但它把「静默错位」变成了「当场报错」。审批系统里，当场报错是好消息。注意这里的校验是双向的：payload 里的标识供审批端回显和核对，resume 里的标识供节点校验，两头都对得上，顺序才变得无关紧要。

## 坑三：一个节点在一轮里反复调 interrupt

有人的写法是在循环里逐项请求确认：

```python
# 错误示范
for item in state["items"]:
    if not interrupt({"item": item}):   # 循环里调 interrupt
        cancel(item)
```

直觉上这是「每项问一次」，实际会掉进重放陷阱。恢复时节点从头重跑，循环从头再进，第一项的 `interrupt()` 会再次进入等待状态等待新的 resume 值，而 `cancel(item)` 因为上一轮没恢复到永远没执行。重放路径随着循环长度指数膨胀，官方文档明确把这条列为反模式。

修复是把「问几次」压缩成「问一次」：一次 interrupt 收齐全部决策，payload 是列表，人一次性处理完，节点内遍历结果。这个改法不只是绕坑，交互上也更好：审批人在一个界面里勾完五十项，比被五十个弹窗依次打断体验好得多。如果决策之间存在依赖（第二项要不要取消取决于第一项的结果），那说明这不是一次审批而是一段流程，应该拆成多个节点，每个节点至多一处 interrupt，让图的拓扑表达依赖关系，而不是在一个节点里用循环硬凑。

## 把两级审批拆成两个节点

用「部门经理批预算、财务批付款」这个例子，把坑二和坑三的纪律合起来看。拆成两个节点，每个节点一处 interrupt，payload 和 resume 值都带标识；「审批答复错位」和「审批被拒绝」要分开处理：前者是 bug，抛异常；后者是业务结果，走条件边路由到终止：

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt
from langgraph.checkpoint.memory import InMemorySaver

class POState(TypedDict):
    po_id: str
    rejected: bool

def manager_approve(state: POState) -> dict:
    decision = interrupt({"type": "budget_approval", "po_id": state["po_id"]})
    if decision.get("type") != "budget_approval":
        raise ValueError("审批答复错位")          # 错位是程序 bug，报错没商量
    return {"rejected": not decision.get("ok")}   # 拒绝是业务结果，交给路由

def route_after_manager(state: POState) -> str:
    return END if state["rejected"] else "finance"

def finance_approve(state: POState) -> dict:
    # 恢复轮重跑会再次走到这里：先重验前提，再消费审批结果
    decision = interrupt({"type": "payment_approval", "po_id": state["po_id"]})
    if decision.get("type") != "payment_approval":
        raise ValueError("审批答复错位")

g = StateGraph(POState)
g.add_node("manager", manager_approve)
g.add_node("finance", finance_approve)
g.add_edge(START, "manager")
g.add_conditional_edges("manager", route_after_manager, ["finance", END])
g.add_edge("finance", END)
app = g.compile(checkpointer=InMemorySaver())
```

resume 值还是按发生顺序消费，这个规则没有变；变的是每个节点只认自己的标识，任何一处错位都会当场炸出来，而不是把钱批给错误的确认。拒绝走条件边而不是抛异常，也有实际的好处：thread 的终态是「正常结束」，审计里能看清「谁在哪一环拒的」，后续重发起也是干净的新 thread，而不是带着异常残留的半成品。

## 坑四：payload 不可序列化

interrupt 的参数和 resume 值都要写进 checkpoint，随数据库落盘，暂停可能持续几天，内存里什么都留不住。传 lambda、数据库连接、文件句柄、自定义类的实例，存档那一步就炸，或者更糟：用内存版 checkpointer 测试时一切正常，换到 Postgres 版才炸，问题看起来像「数据库有 bug」。

纪律很简单：payload 和 resume 值只用 JSON 能表达的东西，dict、str、数字、列表。需要富对象就传 id，恢复之后重新查库拿最新数据。这个限制反过来还有个好处：payload 里能放什么，天然就是你该给审批人看什么。数据库连接对象对审批人毫无意义，订单号、金额、事由才是。

还有两个时间维度的问题要提前想。一是 resume 值同样受限：前端传回来的东西也要过序列化这一关，别指望审批界面回传一个嵌了对象的复杂结构。二是结构演进：一个 thread 可能带着旧版 payload 在库里等了两周，期间你的代码发了新版，payload 多了个字段、改了个名字，恢复时跑的却是新代码。所以人审节点的代码要按「可能消费一个月前的旧 payload」来写，读字段给默认值，缺关键字段就走人工重审，别直接下标取值。这也是第 13 篇 state schema 演进问题在人审场景的具体形态。

## 坑五：前置副作用不幂等

「恢复等于重跑」最贵的推论。假设审批节点里 interrupt 之前先调了扣库存接口：

```python
def approve_and_deduct(state) -> dict:
    deduct_stock(state["sku"], state["qty"])        # 副作用在 interrupt 之前
    decision = interrupt({"question": "是否确认扣款？"})
    ...
```

第一轮执行到 interrupt，库存已经扣了一次；人审通过，恢复，节点从第一行重跑，库存再扣一次。财务对不上账的时候，日志里两次扣减都「合规」。写节点时的纪律因此是硬性的：要么把副作用全部放在 interrupt 之后，要么给外部调用带幂等键，让重复调用被服务端去重。检查方法也直接：把节点里 interrupt 之前的行全部盖住，剩下的部分如果是纯函数（只读 state、只算东西），这个节点就是安全的。

用 deepagents 的 `interrupt_on` 做工具级审批时同理：被审批的工具本身要经得起重复调用，因为恢复之后工具可能带着同样的参数再执行一遍。工具层的幂等设计和节点层的纪律是同一件事，别指望框架替你去重。

幂等键不需要什么复杂设计，业务里现成的唯一标识就是：单号加动作类型，转账用转账流水号。调用外部接口时把它作为幂等键带上，服务端见过同样的键就直接返回上次的结果。如果外部接口不支持幂等键（不少老系统不支持），那就把副作用挪到 interrupt 之后、且挪到单独的节点里，让「审批通过」和「执行动作」变成两个节点，重跑永远只发生在没有副作用的那一段。这其实是更好的结构：审批节点只做决定，执行节点只做动作，各管各的，出问题时日志也分得清是「决定错了」还是「执行错了」。

## 坑六：并行分支的 interrupt 恢复错乱

多个分支并行执行、各自有 interrupt（比如五十份报销同时挂起等审），恢复时多个 resume 值要和对应的 interrupt 一一对上。坑二讲的顺序对齐在这里彻底失效：并行分支的完成顺序本来就不保证，「第几个发生的」这个问题在并行区没有稳定答案。

官方给出的模式是用 interrupt id 构建映射：payload 里带全局唯一 id，恢复时传一个 `{id: 决策}` 的字典，每个分支节点按自己的 id 取值。id 怎么造？用业务上天然唯一的东西：文档 id 加动作类型，比如 `"exp-2024-0891:finance"`。这个模式和第 08 篇并行分支的合并纪律是配套的，那一篇里有完整的 map-reduce 场景。别试图用「按顺序第几个」来对位，并行区里它连一个月的太平都保证不了。

并行区的人审还要多想一步「部分恢复」：五十份报销，审批人先批了三十份就下班了，剩下二十份要不要继续等？映射模式天然支持这种节奏，resume 字典里只有三十个键，没拿到决策的分支继续挂着等下一批恢复。但你的业务代码要明确这个语义：merge 节点是等全部决策齐了再汇总，还是凑够一批处理一批。这个选择没有标准答案，但它必须是个明确的设计决定，而不是恢复机制恰好表现出什么就是什么。

## 六个坑收敛成一张检查表

六个坑看着零散，其实全部挂在「恢复等于重跑」这一条机制上。这不难记：凡是「暂停轮做过、恢复轮还会再做」的事都要问一遍会不会出错，凡是「靠发生顺序」的假设都要问一遍会不会变。收敛成评审时的五问：

| 检查项 | 对应的坑 | 不合格的后果 |
| --- | --- | --- |
| interrupt 调用处是否被 try/except 或统一错误处理覆盖 | 坑一 | 人审静默失效 |
| resume 值是否自带业务标识并校验 | 坑二、坑六 | 审批答复错位，无报错 |
| 一个节点一轮内是否只有一处 interrupt | 坑三 | 重放膨胀，行为不可预期 |
| payload 和 resume 值是否 JSON 可表达 | 坑四 | 换真数据库才炸 |
| interrupt 之前是否有副作用，有的话是否幂等 | 坑五 | 副作用执行两次 |

评审人审方案时把这五问过一遍，大部分生产事故在写代码之前就能拦住。它也是排障时的排查顺序：线上人审出问题，先确认暂停有没有发生（坑一，信号被吃），再确认恢复值对不对位（坑二、坑六，错位），然后查恢复轮的副作用（坑五，重复执行），最后才怀疑框架本身。实际案例里，走到最后一步的极少，绝大多数在前三步就找到了主人。剩下拦不住的，是框架根本不管的那件事。

## 框架不管的事：人一直不批怎么办

LangGraph 没有内置的 interrupt 超时。一个 thread 停在等待状态，理论上是永久的，框架不会催你，也不会替你做决定。demo 里无所谓，生产上这是最常见的人审故障：审批人休假了，流程挂在等待里，发起人以为在处理，三天后发现谁都没看。积压还有第二个代价：每个等待中的 thread 都是 checkpoint 表里的一行活数据，几千个无人处理的审批挂上几个月，存储曲线就跟着翘头，这是第 13 篇表膨胀事故里最容易被忽视的一种来源。

生产要自己补一层兜底：一个后台任务定期扫描「停在等待中的 thread」，超过 SLA 就走两个出口之一。出口一，用 `update_state` 写入默认决策后 resume，让流程按超时策略走完；出口二，标记流程过期，通知发起人重新发起。骨架大概长这样：

```python
# 独立于图的后台任务，比如每小时跑一次
def sweep_expired_approvals():
    for t in find_threads_waiting_longer_than(SLA):   # 按你的 checkpointer 存储实现查询
        # SLA 前半段先提醒，后半段升级到上级，这里只演示超时出口
        app.update_state(
            {"configurable": {"thread_id": t.thread_id}},
            {"messages": [("user", "审批超时，系统自动拒绝")]},
        )
        app.invoke(
            Command(resume={"type": t.waiting_type, "ok": False, "timeout": True}),
            {"configurable": {"thread_id": t.thread_id}},
        )
        notify(t.initiator, f"审批超时未处理，{t.po_id} 已自动拒绝")
```

resume 值照旧带上 type 标识和 `timeout: True`，让流程里的节点能区分「人拒绝的」和「超时拒绝的」，两者的后续动作（是否允许重新发起、是否要向上级通报）通常不一样。怎么找到「停在等待中的 thread」取决于你的 checkpointer 存储，Postgres 版就是查 checkpoint 表里处于 interrupt 状态的 thread，这部分第 13 篇的运维内容能直接复用。

设计这个兜底时有一个原则性判断：**自动拒绝比自动批准安全**。自动批准一个被遗忘的 5 万转账，出错时是资金损失；自动拒绝，最多是流程重新走一遍。所以除非业务明确要求「超时视为同意」，默认方向都是拒绝或转人工。SLA 的设计也别一刀切：小额高频的审批 SLA 定短些，大额低风险的可以放宽，审批人离职造成的死锁要靠「审批人节假日自动升级代批」这类组织层面的规则解决，那是工单系统的职责，不是图的职责。

这段代码不难，难的是它不在任何框架教程的路径上，没人写就一直没人写。把「人审超时谁兜底」当成方案评审的必答项，答不上来的方案不进生产。

## 前端和工单系统怎么接

人审的另一半工程在框架外面。interrupt 触发时，后端要做三件事：把 payload、thread_id、checkpoint 信息投给你的工单或审批系统；审批人在界面里看到待办、点批准或驳回；后端拿 thread_id 构造 `Command(resume=...)` 恢复流程。这套链路里 thread_id 是唯一贯穿的钥匙，前端不需要理解图结构，只需要老老实实把它带回来。

官方的 Agent Chat UI 已内置 interrupt 展示和恢复，自己搭前端之前先看它的交互协议：它把 interrupt 当成一种特殊的消息类型推给前端，前端回复时带上 resume 值。自研界面照着这个协议设计，能少走很多弯路。要注意的只有一点：界面上展示给审批人的信息来自 payload，所以坑四的纪律在这里再一次生效，payload 的质量就是审批界面的质量。

安全这条线要单独划出来。thread_id 是恢复流程的唯一钥匙，但它是标识，不是凭证：知道 thread_id 不等于有权审批这一单。后端收到恢复请求时，要校验「当前操作人对这笔单据是否有审批权」，校验依据是你的业务系统（角色、金额权限、单据归属），不是 thread_id 本身。审批动作和 resume 调用之间加这一层业务鉴权，才不会出现「拿到 id 的实习生批了总监的单」这种事故。审计同样落在业务侧：谁、什么时候、基于什么理由批的，这三个字段要在工单系统里留全，checkpoint 时间线只负责对账。

## 人审怎么测

人审流程的测试比普通节点多一个维度：同一个节点要测「暂停轮」和「恢复轮」两种行为。做法是固定的三步：第一步 invoke 到暂停，断言流程确实停住（比如通过 get_state 检查状态里出现了等待中的 interrupt）；第二步用 `Command(resume=...)` 恢复，传构造好的决策值；第三步断言恢复轮的输出和副作用。批准和拒绝两条路径各来一遍，坑二的错位问题也可以在这里主动测：故意传错位的 resume 值，断言节点报错而不是消费。

这类测试放进 CI 有一个额外的好处：人审代码是最容易被后续需求改坏的代码。有人在节点里加了一行 try/except，有人动了两处 interrupt 的顺序，单元测试不暂停不恢复根本看不出来，而「暂停、恢复、断言」的三步测试每一条变更都会跑。第 15 篇的回归体系里，人审流程的轨迹测试应该单独列为一类用例，不要混在普通功能测试里。

测试里传的 resume 值要带业务标识、payload 保持 JSON 可表达，和生产代码走同一套纪律。测试环境用 InMemorySaver 完全可以，但至少留一条用真数据库 checkpointer 跑的集成测试，坑四那种「换库才炸」的问题只有在真序列化路径上才暴露得出来。

## 何时不该做人审

人审很贵，贵的地方不只是等一个人点按钮。它把同步流程变成异步流程，超时、提醒、代批、审批人不在岗的一整套机制随之而来；它让每一次测试都要模拟「等人」这个动作；它还引入了本篇最大的风险面：恢复语义。所以人审是最后手段，不是安全感的来源。

低风险高频操作（查询、格式转换、内容打标）做人审是纯粹的浪费，正确的替代是权限分级加 dry-run：让 agent 先跑，产物标记为草稿，人看草稿而不是卡流程。真正值得人审的是三类动作：不可逆的（删除、对外发送）、动钱的（转账、下单）、有合规要求的（出账、备案）。判断口诀和第 01 篇判断线的第二问一致：动了外部世界且覆水难收的，才配得起一个 interrupt。

夹在中间的地带用聚合审：单笔 30 块的报销不该逐笔 interrupt，但每天一批、汇总成一个 payload 让人一次看完，频率降下来，控制力没丢。人审的粒度是设计出来的，不是 agent 每碰到一个动作就本能地停一次。反过来的教训也有：把审批拆得过碎，审批人会疲劳，疲劳的人看见弹窗就点同意，人审形同虚设还拖慢了流程。审批节点的数量本身要做减法，能合并的决策合并，能降级成草稿的降级，留给 interrupt 的应该是那几个真正要人负责的瞬间。

## 延伸阅读

- [Human-in-the-loop 概念文档](https://docs.langchain.com/oss/python/langgraph/interrupts.md)：含官方列出的 interrupt 使用注意事项
- [interrupt API 参考](https://reference.langchain.com/python/langgraph/types/interrupt)：恢复匹配语义
- [Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui)：内置 interrupt 交互的开源前端
