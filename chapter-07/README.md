# 第七章课后习题：构建你的 Agent 框架

## 习题 7.1：分析 HelloAgents 框架设计

本章构建了 `HelloAgents` 框架，并阐述了“为何需要自建Agent框架”。请分析：

- 在7.1.1节中提到了当前主流框架的四个主要局限性。结合你在[第六章习题](../chapter-06/README.md)或实际项目中使用过的某个框架的实际经验，说明这些问题是如何影响开发效率的。
- `HelloAgents` 提出了“万物皆为工具”的设计理念，将 `Memory`、`RAG`、`MCP` 等模块都抽象为工具。这种设计有什么优势？是否存在局限性？请举例说明。
- 对比第四章从零实现的智能体代码和本章的框架化实现，框架化带来了哪些具体的改进？如果让你设计一个框架，你会优先考虑哪些设计原则？

### 参考答案

主流框架常见局限包括抽象复杂、定制困难、调试链路不透明、工程集成成本高。以 `AutoGen` 为例，它很适合多智能体对话协作，但当任务需要严格流程控制时，开发者往往要额外写调度器、终止条件和异常处理逻辑；如果对话跑偏或陷入循环，排查问题需要回看大量消息历史。以 `LangGraph` 为例，它对流程控制很强，但开发者必须先把状态、节点、边和条件设计清楚，简单任务的开发成本会显得偏高。这些问题都会影响开发效率：原型阶段可能被框架概念拖慢，生产阶段又要为权限、日志、重试、测试和监控补很多工程代码。

“万物皆为工具”的优势是统一调用接口。无论是普通函数、记忆系统、RAG 检索、MCP 服务还是外部 API，都可以被智能体看作可调用工具。这样做可以降低框架复杂度：Agent 不需要理解每种能力的内部实现，只需要根据工具名称、描述和参数 schema 调用。它还便于权限控制、日志记录、失败重试和工具组合。例如 `RAGTool` 负责检索知识库，`MemoryTool` 负责读写记忆，`MCPTool` 负责调用外部 MCP Server，它们都可以暴露统一的 `execute()` 接口。

局限在于并非所有模块都天然适合被简化成一次工具调用。记忆系统可能需要长期状态管理、遗忘、压缩和冲突合并；RAG 可能包含文档解析、分块、召回、重排和引用校验；MCP 工具可能有权限、连接生命周期和流式返回。如果全部抽象为普通工具，可能掩盖模块内部的复杂性。因此更合理的做法是外部接口统一，内部实现保留专门模块。例如 `MemoryTool.execute()` 可以是统一入口，但内部仍由 `MemoryStore`、`Consolidator`、`ForgetPolicy` 组成。

相比第四章从零实现的智能体，本章框架化实现带来三类改进。第一，抽象更清晰，`Agent`、`Message`、`Tool`、`Config`、`LLM` 各自承担明确职责。第二，可扩展性更好，新增模型供应商、工具或 Agent 范式不需要复制大量代码。第三，工程能力更完整，可以统一处理配置、日志、错误、异步执行和结构化消息。如果我设计框架，会优先考虑：接口稳定、最小核心、工具协议统一、状态可观测、错误可恢复、权限可控、方便测试，并避免过早引入复杂抽象。

## 习题 7.2：扩展 HelloAgentsLLM 与本地模型方案

在7.2节中，我们扩展了 `HelloAgentsLLM` 以支持多模型供应商和本地模型调用。

> <strong>提示</strong>：这是一道实践题，建议实际操作

- 参考7.2.1节的示例，尝试为 `HelloAgentsLLM` 添加一个新模型供应商的支持（如`Gemini`、`Anthropic`、`Kim`）。要求通过继承方式实现，并能够自动检测该提供商的环境变量。
- 在7.2.3节中介绍了自动检测机制的三个优先级。请分析：如果同时设置了 `OPENAI_API_KEY` 和 `LLM_BASE_URL="http://localhost:11434/v1"`，框架最后会选择哪个提供商？这种优先级设计是否合理？
- 除了本章介绍的 `VLLM` 和 `Ollama`，还有 `SGLang` 等其他本地模型部署方案。请先搜索并了解 `SGLang` 的基本信息和特点，然后对比 `VLLM`、`SGLang` 和 `Ollama` 这三者在易用性、资源占用、推理速度、推理精度等方面的优劣。

### 参考答案

新增供应商时，应让新类继承 `HelloAgentsLLM` 或框架中的供应商基类，并实现统一的 `chat()`、`complete()` 或 `generate()` 方法。以 Gemini 为例，可以设计为：

```python
class GeminiLLM(HelloAgentsLLM):
    provider = "gemini"

    @classmethod
    def is_available(cls) -> bool:
        return bool(os.getenv("GEMINI_API_KEY"))

    def __init__(self, model: str = "gemini-1.5-pro", **kwargs):
        self.api_key = os.getenv("GEMINI_API_KEY")
        self.model = model
        self.client = build_gemini_client(api_key=self.api_key)

    def chat(self, messages: list[Message], **kwargs) -> Message:
        payload = self._convert_messages(messages)
        response = self.client.generate_content(model=self.model, contents=payload, **kwargs)
        return Message(role="assistant", content=self._extract_text(response))
```

自动检测时，注册表可以按优先级遍历供应商：

```python
PROVIDERS = [LocalOpenAICompatibleLLM, OpenAILLM, GeminiLLM, AnthropicLLM]

def auto_detect_llm():
    for provider in PROVIDERS:
        if provider.is_available():
            return provider()
    raise RuntimeError("No available LLM provider found")
```

如果框架的优先级是“显式配置优先、本地模型优先、云端 API 兜底”，那么同时设置 `OPENAI_API_KEY` 和 `LLM_BASE_URL="http://localhost:11434/v1"` 时，应选择本地 OpenAI-compatible provider，即 Ollama 的 OpenAI 兼容接口。理由是 `LLM_BASE_URL` 更像用户显式指定的模型服务地址，优先级应高于仅存在的云端 API Key。这个设计总体合理，因为它避免在已经配置本地模型时误调用云端服务，降低隐私和成本风险。但最好允许用户通过 `LLM_PROVIDER=openai` 或 `LLM_PROVIDER=ollama` 显式覆盖，避免自动检测带来歧义。

`vLLM`、`SGLang` 和 `Ollama` 的定位不同。`Ollama` 最易用，适合个人电脑、本地开发和快速试验。它安装简单、模型管理方便，但高并发服务、复杂调度和极致吞吐不是它的主要优势。

`vLLM` 更偏生产推理服务，特点是高吞吐、连续批处理、PagedAttention、OpenAI 兼容 API、适合多用户并发推理。它适合服务端部署和高并发场景，但部署、显存规划和参数调优比 Ollama 复杂。

`SGLang` 是面向大语言模型和多模态模型的高性能推理与编程框架，强调后端推理加速和前端结构化生成编排，支持 RadixAttention、连续批处理、约束解码、工具/函数调用相关能力等。它适合需要复杂推理流程、高性能服务和结构化生成控制的团队。资源占用和部署复杂度通常高于 Ollama，和 vLLM 一样更偏工程化服务端。

推理精度主要由模型权重、量化方式、上下文长度和采样参数决定，而不是单纯由推理框架决定。在相同模型和参数下，三个系统的结果应接近；差异更多来自量化格式、默认参数、KV 缓存策略和并发压力。选型建议是：个人开发和课程实验用 Ollama；企业高并发 API 服务优先 vLLM；需要复杂结构化生成、Agentic workflow 推理优化或高性能编排时考虑 SGLang。

## 习题 7.3：Message、Agent 与 Config 设计

在7.3节中，我们实现了 `Message` 类、`Config` 类和 `Agent` 基类。请分析：

- `Message` 类使用了 `Pydantic` 的 `BaseModel` 进行数据验证。这种设计在实际应用中有哪些优势？
- `Agent` 基类定义了 `run` 和 `_execute` 两个方法，其中 `run` 是公开接口，`_execute` 是抽象方法。这种设计模式叫什么？有什么好处？
- 在 `Config` 类中，我们使用了单例模式。请解释什么是单例模式，为什么配置管理需要使用单例模式？如果不使用单例会导致什么问题？

### 参考答案

`Message` 使用 `Pydantic BaseModel` 的优势是可以把消息结构从普通字典提升为有类型、有校验、有序列化能力的数据对象。实际应用中，智能体消息可能包含 `role`、`content`、`name`、`metadata`、`tool_calls`、`timestamp` 等字段。如果用普通字典，字段缺失、类型错误、拼写错误很难在早期发现。使用 Pydantic 后，可以在创建消息时校验角色是否合法、内容是否为空、元数据是否为字典，还可以方便地转 JSON、做日志记录和 API 传输。

`run` 作为公开接口、`_execute` 作为抽象方法，属于模板方法模式。基类在 `run` 中定义通用流程，例如输入标准化、日志记录、异常捕获、回调通知、耗时统计和输出包装；子类只需要实现 `_execute` 中的具体智能体逻辑。好处是所有 Agent 对外表现一致，同时不同范式可以保留自己的内部实现。调用者只需要调用 `agent.run(input)`，不需要关心它是 ReAct、Reflection 还是 Plan-and-Solve。

单例模式是指一个类在整个进程中只创建一个实例，并提供全局访问点。配置管理适合使用单例，因为 API Key、模型名称、超时时间、日志级别、代理地址等配置应在全局保持一致。如果每个模块都创建自己的配置对象，可能出现不同模块读到不同值的问题。例如一个 Agent 使用 `gpt-4`，另一个工具读取到 `gpt-3.5`；一个模块开启调试日志，另一个没有开启；环境变量更新后部分实例未同步。这会导致行为不一致和难以排查的 bug。

不过单例也要谨慎使用。它会引入全局状态，测试时容易相互污染。更好的做法是默认提供全局单例，但允许在测试或高级用法中显式传入独立配置对象。

## 习题 7.4：框架化 Agent 范式扩展

在7.4节中，我们动手进行了四种 `Agent` 范式的框架化实现。

> <strong>提示</strong>：这是一道实践题，建议实际操作

- 对比第四章从零实现的 `ReActAgent` 和本章框架化的 `ReActAgent`，列举3个具体的改进点，并说明这些改进如何提升了代码的可维护性和可扩展性。
- `ReflectionAgent` 实现了“执行-反思-优化”循环。请扩展这个实现，添加一个“质量评分”机制：在每次反思后，让 `LLM` 对当前版本的输出打分，只有分数低于阈值时才继续优化，否则提前终止。
- 请设计并实现一个新的 `Agent` 范式 `Tree-of-Thought Agent`，要求继承 `Agent` 基类，它能够在每一步生成多个可能的思考路径，然后选择最优路径继续。

### 参考答案

框架化 `ReActAgent` 至少有三个改进点。第一，消息对象标准化，输入输出都使用 `Message`，减少字符串拼接和字段混乱。第二，工具系统统一，工具通过 `BaseTool` 或工具注册表管理，新增工具不需要修改 Agent 主流程。第三，LLM 调用被封装为统一接口，可以切换 OpenAI、本地模型或其他供应商，而不影响 ReAct 逻辑。除此之外，框架化实现通常还会加入日志、异常处理、最大迭代次数、工具调用校验和配置管理，使代码更适合扩展和生产调试。

`ReflectionAgent` 的质量评分机制可以设计为每轮生成后先反思，再评分。如果分数达到阈值，则提前结束；如果低于阈值，则根据反思意见继续优化。伪代码如下：

```python
class ScoredReflectionAgent(Agent):
    def __init__(self, llm, threshold: float = 8.0, max_iters: int = 3):
        self.llm = llm
        self.threshold = threshold
        self.max_iters = max_iters

    def _execute(self, task: str) -> Message:
        draft = self.llm.chat([Message(role="user", content=task)]).content

        for i in range(self.max_iters):
            feedback = self.llm.chat([
                Message(role="system", content="请严格审查当前输出，指出可改进点。"),
                Message(role="user", content=draft),
            ]).content

            score_result = self.llm.chat([
                Message(role="system", content="请只输出 JSON：{\"score\": 0-10, \"reason\": \"...\"}"),
                Message(role="user", content=f"任务：{task}\n输出：{draft}\n反馈：{feedback}"),
            ]).content

            score = parse_score(score_result)
            if score >= self.threshold:
                break

            draft = self.llm.chat([
                Message(role="system", content="请根据反馈改进输出，保持事实准确。"),
                Message(role="user", content=f"原任务：{task}\n当前版本：{draft}\n反馈：{feedback}"),
            ]).content

        return Message(role="assistant", content=draft)
```

`Tree-of-Thought Agent` 的思路是每一步不只生成一个思考路径，而是生成多个候选路径，对候选进行评分，然后选择得分最高的路径继续展开。简化实现如下：

```python
class TreeOfThoughtAgent(Agent):
    def __init__(self, llm, breadth: int = 3, depth: int = 3):
        self.llm = llm
        self.breadth = breadth
        self.depth = depth

    def _execute(self, task: str) -> Message:
        paths = [""]

        for step in range(self.depth):
            candidates = []
            for path in paths:
                prompt = (
                    f"任务：{task}\n"
                    f"已有思考路径：{path}\n"
                    f"请生成 {self.breadth} 个不同的下一步思考方向。"
                )
                thoughts = parse_list(self.llm.chat([Message(role="user", content=prompt)]).content)
                for thought in thoughts:
                    new_path = path + "\n" + thought
                    score = self._score_path(task, new_path)
                    candidates.append((score, new_path))

            candidates.sort(key=lambda x: x[0], reverse=True)
            paths = [path for _, path in candidates[: self.breadth]]

        final_prompt = f"请基于最优思考路径回答任务。\n任务：{task}\n路径：{paths[0]}"
        return self.llm.chat([Message(role="user", content=final_prompt)])

    def _score_path(self, task: str, path: str) -> float:
        result = self.llm.chat([
            Message(role="system", content="请给思考路径打 0-10 分，只输出数字。"),
            Message(role="user", content=f"任务：{task}\n路径：{path}"),
        ]).content
        return float(result.strip())
```

真实实现中还需要限制成本，避免组合爆炸；评分应使用结构化 JSON；必要时可保留 top-k 路径而不是只保留一个，以降低早期误判的影响。

## 习题 7.5：工具系统与工具链

在7.5节中，我们构建了工具系统。请思考以下问题：

- `BaseTool` 类定义了 `execute` 抽象方法，所有工具都必须实现这个方法。请解释为什么要强制所有工具实现统一的接口？如果某个工具需要返回多个值（如搜索工具返回标题、摘要、链接），应该如何设计？
- 在7.5.3节中实现了工具链（`ToolChain`）。请设计一个实际的应用场景，需要串联至少3个工具，并画出工具链的执行流程图。
- 异步工具执行器（`AsyncToolExecutor`）使用了线程池来并行执行工具。请分析：在什么情况下并行执行工具能带来性能提升？

### 参考答案

强制所有工具实现统一的 `execute` 接口，可以让 Agent、工具执行器和工具链不依赖具体工具类型。调用方只需要知道工具名称、参数 schema 和 `execute()`，不需要关心工具内部是 HTTP 请求、数据库查询、文件解析还是本地函数。这带来可替换性和可组合性：新增工具时只要继承 `BaseTool`，就能被注册、调用、日志记录和权限控制。

如果工具需要返回多个值，不应返回难以解析的自然语言字符串，而应返回结构化对象。例如搜索工具可以返回：

```python
class SearchResult(BaseModel):
    title: str
    summary: str
    url: str
    source: str | None = None
    published_at: str | None = None

class SearchToolOutput(BaseModel):
    query: str
    results: list[SearchResult]
```

`execute()` 可以返回字典、Pydantic 对象或统一的 `ToolResult`：

```python
ToolResult(
    success=True,
    data={"results": [...]},
    error=None,
    metadata={"latency_ms": 320}
)
```

实际工具链场景可以是“自动生成竞品分析报告”，串联至少三个工具：

```text
输入：竞品名称和分析目标
-> SearchTool：搜索竞品官网、新闻和产品介绍
-> WebPageReaderTool：抓取候选网页正文
-> SummarizeTool：提取产品功能、价格、目标用户
-> CompareTool：和本公司产品做差异比较
-> ReportWriterTool：生成结构化竞品分析报告
```

流程图：

```text
User Input
-> SearchTool
-> WebPageReaderTool
-> SummarizeTool
-> CompareTool
-> ReportWriterTool
-> Final Report
```

并行执行工具能提升性能的前提是多个工具调用彼此独立，且主要耗时在 I/O 等待而不是 CPU 计算。例如同时查询天气、地图、日历、邮件、多个搜索引擎或多个数据库只读接口，线程池可以把等待时间重叠起来。并行不适合有强依赖的工具链，例如必须先搜索再抓网页，也不适合对同一资源有写冲突的操作。对于 CPU 密集型任务，Python 线程池还会受到 GIL 影响，可能需要进程池、异步 I/O 或外部服务。

## 习题 7.6：扩展 HelloAgents 的流式输出、对话管理与插件系统

框架的可扩展性是设计的重要考量因素之一。你现在要扩展 `HelloAgents` 框架，为其实现一些有趣的新功能和特性。

- 首先为 `HelloAgents` 添加一个“流式输出”功能，使得 `Agent` 在生成响应时能够实时返回中间结果（类似 `ChatGPT` 用户界面的打字效果）。请设计这个功能的实现方案，说明需要修改哪些类和方法。
- 然后为框架添加“多轮对话管理”功能，能够自动管理对话历史、支持对话分支和回溯，你会如何设计？需要新增哪些类？如何与现有的 `Message` 系统集成？
- 最后请为 `HelloAgents` 设计一个“插件系统”，允许第三方开发者通过插件的方式扩展框架功能（如添加新的 `Agent` 类型、新的工具类型等），而无需修改框架核心代码。要求画出插件系统的架构图并说明关键接口。

### 参考答案

流式输出需要从 LLM 接口开始支持。可以在 `HelloAgentsLLM` 中新增 `stream_chat(messages)` 方法，返回 token 或事件迭代器；在 `Agent` 基类中新增 `stream_run(input)` 公开接口；在具体 Agent 中允许边执行边产生事件，例如 `thought`、`tool_call`、`tool_result`、`token`、`final`。

接口设计示例：

```python
class StreamEvent(BaseModel):
    type: Literal["token", "thought", "tool_call", "tool_result", "final", "error"]
    content: str
    metadata: dict = {}

class HelloAgentsLLM:
    def stream_chat(self, messages: list[Message]) -> Iterator[StreamEvent]:
        ...

class Agent:
    def stream_run(self, input: str) -> Iterator[StreamEvent]:
        yield from self._stream_execute(input)
```

对于普通 Agent，`stream_run()` 可以直接透传 LLM token；对于 ReAct Agent，应该在工具调用前后输出事件，让前端显示“正在查询天气”“工具返回结果”等状态。最终仍要聚合出完整 `Message`，便于记录历史。

多轮对话管理可以新增 `Conversation`、`ConversationStore` 和 `ConversationManager`。`Conversation` 保存一棵消息树，而不是简单列表，以支持分支和回溯。每条 `Message` 增加 `id`、`parent_id`、`conversation_id`、`created_at` 和 `metadata`。当用户从历史某一轮重新提问时，新消息挂到该节点下，形成新分支。

设计如下：

```text
ConversationManager
-> append_message(conversation_id, parent_id, message)
-> get_context(conversation_id, branch_id, max_tokens)
-> fork(conversation_id, message_id)
-> rollback(conversation_id, message_id)
```

上下文构建时，`ConversationManager` 沿当前分支从根节点取消息，按 token 预算裁剪；过长历史可以调用摘要工具压缩。这样既能支持普通多轮对话，也能支持“回到上一版回答重新生成”的场景。

插件系统应采用注册表和入口点机制。第三方插件声明自己提供哪些 Agent、Tool、LLM Provider 或 Callback，框架启动时扫描插件并注册。

架构图：

```text
HelloAgents Core
-> PluginManager
   -> load plugins
   -> validate manifest
   -> register extension points
-> Registries
   -> AgentRegistry
   -> ToolRegistry
   -> LLMProviderRegistry
   -> CallbackRegistry
-> Third-party Plugins
   -> CustomAgent
   -> CustomTool
   -> CustomLLMProvider
```

插件清单可以包含：

```yaml
name: helloagents-web-search
version: 0.1.0
entrypoint: helloagents_web_search.plugin:register
permissions:
  - network
provides:
  tools:
    - web_search
```

关键接口：

```python
class Plugin(Protocol):
    def register(self, registry: PluginRegistry) -> None:
        ...

class PluginRegistry:
    def register_tool(self, name: str, tool_cls: type[BaseTool]) -> None:
        ...

    def register_agent(self, name: str, agent_cls: type[Agent]) -> None:
        ...
```

为了安全，插件系统必须支持版本约束、权限声明、依赖检查、沙箱或最小权限运行、禁用插件、插件加载失败隔离和审计日志。这样第三方扩展不会破坏框架核心。
