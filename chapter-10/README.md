# 第十章课后习题：智能体通信协议

> <strong>提示</strong>：部分习题没有标准答案，重点在于培养学习者对智能体通信协议的综合理解和实践能力。

## 习题 10.1：MCP、A2A 与 ANP 的定位和组合

本章介绍了三种智能体通信协议：MCP、A2A 和 ANP。请分析：

- 在 10.1.2 节中对比了三种协议的设计理念。请深入分析：为什么 MCP 强调“上下文共享”，A2A 强调“对话式协作”，而 ANP 强调“网络拓扑”？这些设计理念分别解决了什么核心问题？
- 假设你要构建一个“智能客服系统”，需要以下功能：（1）访问客户数据库和订单系统；（2）多个专业客服智能体协作处理复杂问题；（3）支持大规模并发用户请求。请为每个功能选择最合适的协议，并说明理由。
- 三种协议是否可以组合使用？请设计一个实际应用场景，展示如何同时使用 MCP、A2A 和 ANP 来构建一个完整的智能体系统。画出系统架构图并说明各协议的职责。

### 参考答案

MCP 强调上下文共享，是因为它主要解决“模型应用如何标准化访问外部工具、资源和提示模板”的问题。智能体要完成真实任务，必须读取数据库、调用 API、访问文件、使用企业知识库。MCP 把这些能力封装成可发现、可调用、可治理的服务器能力，让模型客户端以统一方式获得上下文和执行工具。

A2A 强调对话式协作，是因为它主要解决“多个智能体如何围绕同一任务交换消息、分配任务和返回结果”的问题。一个复杂任务往往需要研究员、规划员、执行员、审查员等多个角色协作。A2A 的重点不是访问工具，而是智能体之间如何表达任务、状态、结果、拒绝、协商和完成。

ANP 强调网络拓扑，是因为它面向更大规模的智能体网络。它关注智能体发现、路由、寻址、拓扑结构、负载和容错。当智能体数量从几个扩展到几百个甚至上千个时，问题不再只是两个智能体如何对话，而是如何找到合适智能体、如何转发消息、如何避免中心节点瓶颈。

智能客服系统中，访问客户数据库和订单系统最适合 MCP。数据库查询、订单查询、退款工具、物流工具都可以作为 MCP Tools 暴露，并配合 Resources 提供客户资料或政策文档。多个专业客服智能体协作处理复杂问题适合 A2A，例如退款智能体、物流智能体、投诉处理智能体、人工复核智能体之间通过任务消息协作。大规模并发用户请求更适合 ANP 或类似网络路由层，用于客服智能体发现、负载均衡、故障转移和跨区域路由。

三种协议可以组合。示例是“跨区域企业客服智能体网络”：

```text
用户请求
-> ANP Gateway：发现可用客服集群，按地区和负载路由
-> A2A Coordinator：组织订单智能体、退款智能体、物流智能体协作
-> 各智能体通过 MCP Client 调用工具
   -> Customer DB MCP Server
   -> Order System MCP Server
   -> Logistics MCP Server
   -> Policy Knowledge MCP Server
-> A2A 汇总协作结果
-> ANP 返回给用户所在区域入口
```

职责划分是：ANP 负责网络级发现、路由和容错；A2A 负责智能体之间的任务协作；MCP 负责每个智能体访问外部工具、资源和业务系统。三者解决的是不同层级的问题，不是互斥关系。

## 习题 10.2：扩展 MCP 服务器与远程通信

MCP（Model Context Protocol）是智能体与工具通信的标准协议。基于 10.2 节的内容，请深入思考：

> <strong>提示</strong>：这是一道动手实践题，建议实际操作

- 在 10.2.3 节的 MCP 服务器实现中，我们定义了`list_tools`、`call_tool`等核心方法。请扩展这个实现，添加一个新的 MCP 服务器，提供以下工具：（1）数据库查询工具；（2）数据可视化工具；（3）报表生成工具。要求工具之间能够协作完成复杂的数据分析任务。
- MCP 协议支持“资源”（Resources）和“提示”（Prompts）两个重要概念，但本章主要聚焦于“工具”（Tools）。请查阅 MCP 官方文档，了解 Resources 和 Prompts 的设计目的，并设计一个应用场景，展示如何利用这三个核心概念构建更强大的智能体系统。
- MCP 使用 JSON-RPC 2.0 作为底层通信协议，通过 stdio 进行进程间通信。请分析：这种设计有什么优势和局限性？如果需要支持远程 MCP 服务器（通过 HTTP/WebSocket 访问），应该如何扩展当前的实现？

### 参考答案

数据分析 MCP Server 可以提供三个工具：

```text
query_database(sql, parameters)：执行只读 SQL 查询，返回表格数据。
create_chart(data_ref, chart_type, x, y, group_by)：基于查询结果生成图表。
generate_report(query_refs, chart_refs, template)：生成 Markdown/HTML/PDF 报表。
```

协作流程示例：

```text
用户：分析最近 30 天销售额下降原因
-> list_tools 获取可用工具
-> query_database 查询销售额、订单量、渠道、商品类别
-> query_database 查询退款率和库存缺货
-> create_chart 生成销售趋势图、渠道对比图、退款率图
-> generate_report 汇总结论、图表和建议
```

关键安全要求是：数据库工具必须只允许只读查询；SQL 应经过解析器校验，禁止 `DROP`、`DELETE`、`UPDATE`、`INSERT`、危险函数和跨库访问；查询结果应有行数限制和脱敏规则；报表生成只引用当前会话允许访问的数据。

MCP 的 Resources 用于暴露可读取的上下文资源，例如文件、数据库 schema、文档、日志、配置或知识库条目。Prompts 用于暴露可复用的提示模板，例如“生成销售分析报告”“进行 SQL 审查”“解释图表异常”。Tools 用于执行动作，例如查询数据库、生成图表、导出报表。

一个完整场景是“企业经营分析助手”：

```text
Resources:
  - sales_schema：销售数据库表结构
  - metric_definitions：指标口径说明
  - report_templates：公司标准报表模板
Prompts:
  - sales_analysis_prompt：销售分析提示模板
  - anomaly_explanation_prompt：异常解释提示模板
Tools:
  - query_database
  - create_chart
  - generate_report
```

智能体先读取 Resources 理解数据口径，再使用 Prompt 生成规范分析计划，最后调用 Tools 执行查询、绘图和报告生成。这样比单纯工具调用更可靠，因为模型不仅知道“能做什么”，还知道“数据怎么解释”和“报告应该按什么标准写”。

JSON-RPC 2.0 + stdio 的优势是简单、语言无关、容易本地启动、适合桌面应用或本地开发工具和 MCP Server 通信。stdio 不需要开放端口，安全边界相对清晰，调试也方便。局限是主要适合同机进程通信，不适合跨机器访问；连接生命周期依赖本地进程；扩展到多用户、高并发和远程访问时不够方便。

支持远程 MCP Server 时，可以保留 JSON-RPC 消息模型，把传输层替换为 HTTP、Server-Sent Events 或 WebSocket。HTTP 适合请求响应式工具调用，WebSocket 适合长连接、双向通知和流式结果。扩展时必须增加认证、TLS、租户隔离、访问控制、限流、审计和超时重试。

远程架构可以是：

```text
MCP Client
-> HTTPS/WebSocket Transport
-> MCP Gateway
   -> AuthN/AuthZ
   -> Rate Limit
   -> Audit Log
   -> Tool Router
-> Remote MCP Server
```

这样协议层仍然使用 MCP 的 tools/resources/prompts 语义，但传输层变成适合网络环境的实现。

## 习题 10.3：A2A 协作、冲突解决与框架互通

A2A（Agent-to-Agent Protocol）支持智能体间的对话式协作。基于 10.3 节的内容，请完成以下扩展实践：

> <strong>提示</strong>：这是一道动手实践题，建议实际操作

- 在 10.3.4 节的“研究团队”案例中，研究员和撰写员通过 A2A 协议协作完成论文写作。请扩展这个案例，添加第三个智能体“审稿人”（Reviewer），它能够评审论文质量并提出修改建议。设计三个智能体之间的协作流程，并实现完整的代码。
- A2A 协议定义了`task`、`task_result`等消息类型。请分析：如果协作过程中出现冲突（如两个智能体对同一问题有不同意见），应该如何设计冲突解决机制？请扩展 A2A 协议，添加“协商”（negotiation）和“投票”（voting）等消息类型。
- 对比 A2A 协议与第六章介绍的 AutoGen、CAMEL 等多智能体框架：A2A 作为标准协议，与这些框架的关系是什么？它们能否互相替代？请设计一个方案，让基于 A2A 协议的智能体能够与 AutoGen 框架中的智能体进行通信。

### 参考答案

三智能体论文写作流程可以是：

```text
Coordinator 发起任务
-> Researcher：收集资料、列出观点和证据
-> Writer：根据研究结果写论文草稿
-> Reviewer：按结构、论证、证据、引用和表达评审
   -> 如果通过：返回 final_accept
   -> 如果不通过：返回 revision_request
-> Writer 根据审稿意见修改
-> Researcher 补充缺失证据
-> Reviewer 复审
-> 完成
```

消息结构示例：

```python
class A2AMessage(BaseModel):
    message_id: str
    conversation_id: str
    sender: str
    receiver: str
    type: str
    content: dict
    reply_to: str | None = None
```

核心代码骨架：

```python
class Agent:
    name: str
    def handle(self, msg: A2AMessage) -> A2AMessage:
        raise NotImplementedError

class Researcher(Agent):
    name = "researcher"
    def handle(self, msg):
        return A2AMessage(
            message_id=new_id(),
            conversation_id=msg.conversation_id,
            sender=self.name,
            receiver="writer",
            type="task_result",
            content={"outline": "...", "evidence": ["source A", "source B"]},
            reply_to=msg.message_id,
        )

class Writer(Agent):
    name = "writer"
    def handle(self, msg):
        if msg.type in ["task_result", "revision_request"]:
            draft = write_or_revise(msg.content)
            return A2AMessage(
                message_id=new_id(),
                conversation_id=msg.conversation_id,
                sender=self.name,
                receiver="reviewer",
                type="task_result",
                content={"draft": draft},
                reply_to=msg.message_id,
            )

class Reviewer(Agent):
    name = "reviewer"
    def handle(self, msg):
        review = review_draft(msg.content["draft"])
        if review["score"] >= 8:
            msg_type = "final_accept"
        else:
            msg_type = "revision_request"
        return A2AMessage(
            message_id=new_id(),
            conversation_id=msg.conversation_id,
            sender=self.name,
            receiver="writer" if msg_type == "revision_request" else "coordinator",
            type=msg_type,
            content=review,
            reply_to=msg.message_id,
        )
```

冲突解决可以扩展消息类型：

```text
negotiation_request：请求协商，说明冲突点和各方立场。
negotiation_proposal：提出折中方案或证据。
vote_request：发起投票，列出候选方案和投票规则。
vote_cast：提交投票。
vote_result：公布结果。
arbitration_request：请求仲裁智能体或人工介入。
```

处理流程是：先要求冲突双方提交证据和理由；如果能形成折中方案，则继续执行；如果不能，发起投票或由仲裁者根据预设规则决策。高风险任务不应简单少数服从多数，而应由更高权限的专家或人类确认。

A2A 是通信协议，AutoGen、CAMEL 是多智能体框架。协议定义“消息如何表达和传输”，框架定义“智能体如何运行、调度和协作”。它们不能完全互相替代，但可以互补。AutoGen 可以把 A2A 作为外部通信层，A2A 智能体也可以通过适配器接入 AutoGen 群聊。

适配方案：

```text
A2A Agent
-> A2A-AutoGen Adapter
   -> 将 A2A task 转换为 AutoGen message
   -> 将 AutoGen agent reply 转换为 A2A task_result
-> AutoGen GroupChat
```

适配器需要维护会话 ID、角色映射、消息类型映射和错误处理。例如 A2A 的 `task` 映射为 AutoGen 用户消息，A2A 的 `task_result` 映射为某个 AutoGen AssistantAgent 的回复，AutoGen 的终止信号映射为 A2A 的 `final_result`。

## 习题 10.4：ANP 拓扑、路由与容错

ANP（Agent Network Protocol）支持大规模智能体网络。基于 10.4 节的内容，请深入分析：

- 在 10.4.2 节中介绍了 ANP 的网络拓扑设计，包括星型、网状、分层等结构。请分析：在什么场景下应该选择哪种拓扑结构？如果网络规模从 10 个智能体扩展到 1000 个智能体，拓扑结构应该如何演进？
- ANP 协议支持“路由”（routing）和“发现”（discovery）机制，让智能体能够动态找到合适的协作伙伴。请设计一个“智能路由算法”：根据任务类型、智能体能力、网络负载等因素，自动选择最优的消息路由路径。
- 在 10.4.4 节的“智能城市”案例中，多个智能体协作管理城市系统。请思考：如果某个关键智能体（如交通管理智能体）出现故障，整个系统应该如何应对？请设计一个“容错机制”，包括故障检测、备份切换、状态恢复等功能。

### 参考答案

星型拓扑适合规模小、控制中心明确的系统，例如 10 个以内的企业内部智能体由一个协调器统一调度。优点是简单、易审计、权限集中；缺点是中心节点容易成为瓶颈和单点故障。

网状拓扑适合智能体之间频繁点对点协作、中心不明显的场景，例如研究智能体社区或去中心化服务发现。优点是灵活、无单点中心；缺点是路由复杂、权限和一致性难管理。

分层拓扑适合大规模系统，例如智能城市、跨区域客服网络、企业多部门智能体网络。顶层负责全局路由和治理，中层负责领域协调，底层负责具体任务执行。它能在可扩展性和可管理性之间取得平衡。

从 10 个智能体扩展到 1000 个智能体时，拓扑应从简单星型演进为分层加局部网状。早期一个 coordinator 足够；中期按业务域拆成多个 coordinator；大规模时需要服务注册中心、能力目录、区域网关、负载均衡、健康检查和故障转移。局部领域内可以网状协作，跨领域通过上级路由。

智能路由算法可以为候选智能体计算综合得分：

```text
score =
0.35 * capability_match +
0.20 * historical_success_rate +
0.15 * current_load_score +
0.10 * latency_score +
0.10 * trust_score +
0.10 * cost_score
```

路由流程：

```text
1. 解析任务类型和所需能力。
2. 从能力目录发现候选智能体。
3. 过滤无权限、不可用、负载过高的智能体。
4. 根据综合得分排序。
5. 选择 top-k，必要时并行请求或主备调用。
6. 记录结果，更新成功率、延迟、负载和信任分。
```

对于复杂任务，可以先路由到领域 coordinator，再由 coordinator 分解给具体智能体。例如“优化城市早高峰交通”先路由到交通域，再分派给信号灯智能体、公交调度智能体、事故检测智能体。

智能城市中的关键智能体故障需要容错机制。故障检测可以通过心跳、响应超时、错误率、状态滞后和外部监控判断。备份切换可以采用主备模式或多副本模式：交通管理智能体有热备实例，主节点故障后由路由层把流量切到备节点。状态恢复需要将关键状态写入共享存储或事件日志，例如道路拥堵状态、信号灯策略、事故事件、当前调度计划。

容错流程：

```text
Health Monitor 检测交通智能体无响应
-> 标记为 degraded
-> ANP Router 停止向故障节点路由
-> Standby Traffic Agent 接管
-> 从状态存储加载最近快照和事件日志
-> 重放未处理事件
-> 通知相关智能体：公交、应急、导航系统
-> 故障节点恢复后进入只读同步，验证一致后再加入集群
```

对关键系统还应设计降级策略。如果智能体集群整体不可用，交通信号灯应回退到预设安全策略，而不是完全停止运行。

## 习题 10.5：通信协议安全、加密与信任评估

智能体通信协议的安全性和隐私保护是实际应用中的关键问题。请思考：

- 在 10.2.4 节的 MCP 客户端实现中，智能体可以调用 MCP 服务器提供的任何工具。请分析：这种设计存在什么安全风险？如果 MCP 服务器提供了危险操作（如删除文件、执行系统命令），应该如何设计权限控制机制？
- A2A 和 ANP 协议涉及多个智能体之间的通信，可能包含敏感信息（如用户隐私数据、商业机密）。请设计一个“端到端加密”方案：确保消息在传输过程中不被窃听或篡改，同时支持智能体身份认证和访问控制。
- 在大规模智能体网络中，恶意智能体可能会发送虚假信息、发起拒绝服务攻击或窃取其他智能体的数据。请设计一个“信任评估系统”：根据智能体的历史行为、协作质量、社区评价等因素，动态评估每个智能体的可信度，并据此调整通信策略。

### 参考答案

如果 MCP 客户端可以调用服务器上的任何工具，风险包括：模型被提示注入诱导删除文件、执行系统命令、读取敏感数据、调用高成本 API、越权访问其他用户资源、泄露业务数据。危险工具不能只靠提示词约束，必须在协议和服务端实现权限控制。

权限控制机制应包括：

```text
工具分级：read_only、write、destructive、external_network、sensitive_data。
身份认证：每个客户端和用户都有身份。
授权策略：按用户、角色、租户、工具、参数范围授权。
参数校验：路径白名单、SQL 只读、命令白名单、金额上限。
人工审批：删除文件、付款、发邮件、提交代码等高风险操作需要确认。
审计日志：记录调用者、工具、参数摘要、结果和时间。
沙箱执行：危险工具在隔离环境中运行。
```

端到端加密方案可以为每个智能体分配公私钥和证书。智能体注册时由可信 CA 或网络治理节点签发身份证书。发送消息时，发送方用接收方公钥加密会话密钥，再用会话密钥加密消息正文；发送方用自己的私钥对消息摘要签名。接收方验证签名确认身份和完整性，再解密消息。

消息结构可以包含：

```json
{
  "sender_id": "agent-a",
  "receiver_id": "agent-b",
  "message_id": "msg-001",
  "timestamp": "2026-07-07T10:00:00Z",
  "ciphertext": "...",
  "encrypted_key": "...",
  "signature": "...",
  "scope": ["order.read"]
}
```

访问控制在解密前后都要执行：路由层检查发送方是否有权联系接收方；接收方检查消息 scope 是否允许请求对应任务；敏感字段可以按最小必要原则脱敏。为防重放攻击，应检查时间戳、nonce 和 message_id。

信任评估系统可以为每个智能体维护动态信任分：

```text
trust_score =
0.30 * historical_success_rate +
0.20 * result_quality +
0.15 * protocol_compliance +
0.15 * peer_reputation +
0.10 * security_record +
0.10 * freshness
```

负面事件包括：发送虚假结果、频繁超时、违反协议格式、请求越权数据、异常高频请求、被多个智能体举报。信任分影响通信策略：高信任智能体可直接协作；中等信任智能体需要更多验证；低信任智能体限流、隔离或只允许访问低风险资源；恶意智能体加入黑名单。

为了避免误伤，应提供申诉和人工复核机制；为了防止刷信誉，应降低短期大量好评权重，优先使用可验证行为，例如任务成功率、结果一致性、审计记录和安全事件。大规模网络中，信任评估应与路由系统联动，路由时优先选择高信任、低负载、能力匹配的智能体。
