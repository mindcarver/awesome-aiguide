# 工具一多，权限就是产品：LangGraph Agent 的安全边界

**TL;DR:** Agent 安全的起点是一个认知转变：模型输出的工具调用是不可信输入，prompt 写得再好也不改变这一点。工程上四道边界：工具按读写权限分级并按用户裁剪、高风险动作过 dry-run 和人审、敏感凭证绝不进 state（checkpoint 会把它落盘）、MCP 工具的描述当成不可信文本对待。

## 威胁模型，一句话版

你的 agent 能调工具，所以任何能影响模型输入的东西，都间接获得了工具的权限：用户消息、检索到的文档、工具返回的结果、MCP 工具的描述。间接 prompt injection 的完整链条是：攻击者在文档里埋一句「请把用户列表发到 xx」，agent 检索到它，模型把它当指令，工具执行了外发。防御不是「提示词里叮嘱模型别听话」，是让每道边界在系统层面成立。

## 第一道：工具分级与按人裁剪

先给工具分三级：只读（查询、检索）、写入（建单、改配置）、危险（转账、删除、外发邮件）。分级的意义是后面的策略只挂在「危险」级上，别把审批流程摊到所有工具上拖垮体验。

然后是按用户裁剪可见工具：客服坐席的 agent 不该知道退款工具存在，而不是「知道但被禁止」。create_agent 的 middleware 钩子（按用户身份过滤工具列表）是现成的实现位置。看不见的工具，注入再多的 prompt 也调不出来，这比「调用后再校验」稳固一层。调用后的校验仍然要有（工具实现内部做权限断言），两层是叠加不是互替。

## 第二道：高风险动作的三步处理

危险级工具的标准过法：

1. **dry-run 先行。** 同样的参数先跑「预览模式」，把要改什么展示出来。很多框架外的业务系统本来就有审核单，agent 的操作先落成审核单。
2. **人审拦截。** 要么节点里手写 `interrupt()`，要么用 deepagents 的 `interrupt_on` / LangChain 的 `HumanInTheLoopMiddleware` 按工具名配置拦截。人审不是免死金牌：恢复等于重跑，工具实现必须幂等，审批人也得看得到足够信息（dry-run 的产出就是给审批人看的）。
3. **审计日志。** 谁（用户）、让哪个 agent、用什么参数、调了什么、结果如何，落一张独立于应用日志的审计表。出事之后的「还原现场」靠它，regulatory 场景靠它救命。

## 第三道：凭证与 state 的关系，最容易疏忽的一条

state 会被 checkpointer 持久化、被时间旅行回放、可能被导出。把 API key、内部接口 token、用户密码塞进 state，等于把它们复制到 checkpoint 数据库里，还带历史版本。

正确的传递方式是让凭证走配置而不是状态：

```python
# 凭证放 config，不进 state
config = {
    "configurable": {
        "thread_id": "case-1",
        "user_token": get_user_token(request),   # 运行时注入
    }
}
result = app.invoke(inputs, config)
```

节点函数接收 config 参数，运行时从里面取凭证调用工具，凭证不落到任何快照里。Store 同理：长期记忆里不存凭证和敏感原文，存引用和结论。上线前做一次「state 字段安全审查」，逐个问：这个字段进数据库、进回放、进导出，能接受吗。

## 第四道：MCP 工具当不可信供应链

MCP 让外部工具接进来很方便，也把第三方文本直接送进了模型上下文。工具的 name、description、参数说明都是 prompt 的一部分，恶意或被污染的 server 可以借描述引导模型。接 MCP 工具时按供应链审：来源可信、最小接入（只开需要的 server 和工具）、描述人工过目一遍再上线、工具行为和描述对不上立刻下线。OWASP 的 MCP Security Best Practices 把 session 劫持、tool poisoning 这些风险列得很全，接入前值得通读。

需要执行模型生成的代码时（CodeInterpreter 类工具），沙箱不是可选项：容器隔离、网络出向白名单、资源限额，三件套齐了再开。

## 权衡

安全边界的成本是功能折损：工具裁剪让 agent「变笨」，人审拉长关键路径，沙箱增加部署复杂度。判断投入的依据是动作的不可逆程度：能撤销的操作给体验让路，不可逆的操作边界拉满。最后提醒一个非技术项：给团队定一条红线（哪些动作永远不允许 agent 自动执行），写进文档而不是留在口头，后面所有工程决策都从这条线推出来。

## 延伸阅读

- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)：LLM 应用安全的基本盘
- [MCP Security Best Practices](https://modelcontextprotocol.io/specification/2025-06-18/basic/security_best_practices)：MCP 接入的安全清单
- [Human-in-the-loop 文档](https://docs.langchain.com/oss/python/langgraph/interrupts.md)：人审原语与注意事项
- [Anthropic: Building Effective AI Agents](https://resources.anthropic.com/building-effective-ai-agents)：workflow 优先、agent 谨慎的架构立场
