# MCP 工具协议

## 核心概念

MCP（Model Context Protocol）是一套连接 AI 应用与外部能力的协议。它规定 Host、Client 和 Server 如何协作，以及 Server 如何公开可供使用的能力。

- **Host**：承载模型与用户会话的 AI 应用。
- **Client**：Host 内连接一个或多个 MCP Server 的协议客户端。
- **Server**：暴露外部能力的服务。
- **Tools**：由模型决定何时调用的动作，例如搜索、计算或写入。
- **Resources**：由应用选择并提供给模型的上下文数据。
- **Prompts**：可复用的提示模板。

MCP 像一种通用连接约定，让工具提供方不必为每种 AI 应用各写一套集成。它本身不是工具市场、托管平台或业务逻辑，也不自动提供应用需要的访问授权和风险控制。

## 一次 MCP 工具调用如何发生

1. Host 创建 Client 并连接 MCP Server。
2. Client 按协议发现 Server 提供的能力和工具 schema。
3. Host 把可用工具描述交给模型。
4. 模型选择工具并给出参数。
5. Host / Client 按 MCP 消息调用 Server。
6. Server 返回结果，Host 将结果放进后续模型上下文。

MCP 约定发现和调用方式；执行权限、身份认证、敏感操作审批、超时、日志和业务规则仍需要产品实现。

## Enterprise Insight Agent 的实现

项目一提供可选的 MCP 研究 Retriever。MCPClientManager 使用 langchain-mcp-adapters 中的 MultiServerMCPClient，把项目配置转换为多个 MCP Server 连接配置，再获取可用工具。MCPRetriever 采用两阶段思路：先按研究 query 从候选工具中挑选较相关工具，再调用选出的工具进行研究。

该路径依赖配置的 MCP Server。Enterprise Insight Agent 的默认研究来源仍是网页；不能因为代码支持 MCPRetriever，就说所有研究请求都会走 MCP。

本机代码入口：gpt_researcher/mcp/client.py、gpt_researcher/retrievers/mcp/retriever.py，以及 gpt_researcher/skills/researcher.py。

## EchoMind 的工具层实现

EchoMind 的 mcp/tool_manager.py 定义了项目自己的 MCPToolManager 和 Tool 结构。它负责工具注册、schema 参数检查、执行超时、结果缓存、熔断、可选重排和降级；AgentOrchestrator 向模型提供工具 schema，接收 tool_use 并执行 handler。

这套代码采用统一工具注册和调用的设计，但当前源码没有展示 MCP 标准 Client/Server 的协议握手与传输实现。准确描述是：EchoMind 有自定义的工具管理层，名字和设计目标受到 MCP 工具抽象启发；不能把它表述为已完整实现标准 MCP 协议。

项目一和项目二的差异说明：**MCP 是协议，工具管理器是应用内部的运行逻辑。内部有 Tool 类不自动意味着系统就是 MCP Client 或 Server。**

## MCP 与其他模块如何协作

MCP 和 ReAct 处于不同层次：

- ReAct 负责模型什么时候采取行动、看到结果后是否继续。
- MCP 负责工具服务如何被发现、描述和调用。
- RAG 可以作为某个 MCP Tool 的业务实现，也可以是应用内函数。
- 工具执行结果通过 Agent 循环回到模型，作为 Observation。
- API 可能接收用户请求，但 MCP 不替代用户侧 HTTP API。

当只有少量由同一团队控制的内部工具时，直接写函数或 HTTP 客户端通常更简单。多个 AI Host 要共享、动态发现和复用工具服务时，MCP 的标准化更有价值。

## 面试追问与回答

### MCP 是什么，为什么需要它？

> MCP 是 AI Host 与外部工具 / 数据服务之间的标准协议。它统一能力发现和调用的结构，减少每个 Agent 应用对每种工具分别实现适配的工作。它不替代业务授权、工具安全、超时和结果审计。

### MCP、Function Calling 和工具管理器有什么区别？

> Function Calling 是模型返回结构化工具名称和参数的能力；工具管理器是应用执行与治理这些工具的代码；MCP 是连接 Host / Client / Server 并发现、调用外部能力的协议。它们可以组合使用，但不是同一个层次。

### 两个项目里如何讲 MCP？

> Enterprise Insight Agent 有可配置的 MCP Retriever，能通过多服务 MCP Client 获取工具并用于研究；EchoMind 当前的 MCPToolManager 是项目内部自定义工具注册与运行组件，没有标准协议传输链路。面试时我会按实际代码区分“真实 MCP 客户端路径”和“MCP 风格的内部工具抽象”。