# ReAct 与工具调用

## 1. 核心概念

ReAct（Reasoning and Acting）描述模型围绕目标进行判断、调用外部能力、读取执行结果并继续决策的循环：

**目标与上下文 → 决定下一步 → Action → Observation → 再次决策 → 完成或继续**

模型可以选择直接回答，也可以提出结构化工具调用。应用执行工具后，将真实返回值作为 Observation 放回后续上下文。循环需要明确停止条件，例如给出最终答案、达到调用轮数上限或处理不可恢复的错误。

ReAct 是 Agent 执行范式，不是特定 Python 框架。项目可以通过模型 SDK 自己实现，也可以用 LangChain 或 LangGraph 等框架组织。

## 2. 两个项目里的 Agent 执行

### 2.1 EchoMind

EchoMind 在 AgentOrchestrator 中直接维护工具调用循环：

1. 将系统说明、会话上下文、当前消息和工具 schema 发送给 Anthropic Messages API。
2. 模型返回文本时，Agent 可以结束本轮；返回 tool_use 时，应用读取工具名称与参数。
3. 代码检查当前 Agent 的工具白名单、必填参数和参数类型。
4. 执行对应 handler，将结果或错误封装为 tool_result。
5. 把工具结果加入消息历史，再次请求模型；单个 Agent 的工具循环最多 3 轮。
6. 记录工具调用名称、部分输入、成功状态、耗时、缓存/重排状态和错误等运行信息。

因此项目中的 ReAct 落在可定位的模型请求、结构化 tool_use、应用校验、handler 执行、tool_result 回传和循环停止逻辑里。

### 2.2 Enterprise Insight Agent

项目一以 IntelligenceWorkflow 编排企业研究：分析任务、网页搜索、页面获取、证据筛选、报告生成和证据审查。它具有 Agentic 的分步行动特征，面试中应按这条研究工作流说明，不要把它缩成 EchoMind 式的单一工具循环。

Enterprise 的证据策略与报告检查见[项目实践：Enterprise Insight Agent](../projects/enterprise-insight-agent.md)。

## 3. Thought、Action、Observation 如何协作

- **Thought / 判断**：模型根据任务、上下文和工具描述决定下一步。工程系统关注可执行决策及其依据；不要求把模型完整内部推理写入日志。
- **Action / 行动**：模型提出结构化工具名和参数；应用在授权范围内执行对应操作。
- **Observation / 观察**：工具的实际数据、空结果或错误返回模型，帮助其决定接下来怎么做。

外部信息可以为回答提供依据，结构化行动也让执行步骤能够检查。它们能降低只靠参数知识生成的无依据回答，但不能保证消除幻觉：检索可能不相关，外部数据可能过期，模型也可能误解工具结果。可靠性还来自来源检查、权限控制、输入校验、超时和错误处理、Trace 与评测。

## 4. Observation 与 Trace 的区别

Observation 是 Agent 决策循环中的一个运行时输入：工具执行后，结果返回给模型，影响下一步行动或最终回答。

Trace 是事后观察整次请求的记录。它把一次请求中的多个阶段和工具调用关联起来，帮助开发者回答“这轮实际发生了什么、在哪里慢或失败”。Trace 可以记录 Observation 的摘要、工具名、成功状态和耗时；但它服务于诊断，不自动成为模型下一步的输入。

可以这样记：

- Observation 面向“模型接下来要依据什么行动”。
- Trace 面向“工程师之后如何复盘这次执行”。
- Observation 是决策闭环里的数据；Trace 是贯穿执行过程的观测记录。

EchoMind 工具轨迹与项目一 ResearchTrace 的内容见[Trace、Span 与 Agent 轨迹](trace-observability.md)。

## 5. 与 RAG、MCP、记忆的协作

- **RAG**把检索片段作为工具结果或上下文提供给模型。
- **MCP**标准化客户端如何发现并调用外部工具服务。
- **记忆**提供近期对话、摘要和用户画像，让模型理解当前会话背景。
- **ReAct**决定何时使用这些能力，并根据结果继续行动。
- **Trace**记录系统实际执行的步骤，**评测**检查输出质量与全流程行为。

EchoMind 的知识问答可串成：模型收到知识库工具 schema → 决定调用 → ChromaDB 多查询召回和重排 → 工具结果作为 Observation 返回 → 模型整理答复 → Trace 留下执行轨迹。相关页面：[RAG 与语义检索](rag.md)、[MCP 工具协议](mcp.md)、[记忆系统](memory.md)、[Agent 系统评测](evaluation.md)。

## 6. 面试追问与回答

### ReAct 和普通 LLM 调用有什么区别？

> 普通调用通常是输入上下文后返回一段回答；ReAct 把模型决策、工具行动、执行结果观察和后续判断连成循环。项目里能明确指出模型提出工具调用、应用校验并执行、结果如何返回以及循环何时结束。

### Thought、Action、Observation 会不会消除幻觉？

> 它们提供结构化的外部行动和真实工具结果，让部分回答有可检查依据；系统仍需验证检索来源、工具结果和最终主张。检索错误、过期数据和模型误读仍可能造成错误，因此还要结合证据检查和评测。

### EchoMind 的 ReAct 在哪里？

> AgentOrchestrator 调用 Anthropic API 并提供工具 schema。模型返回 tool_use 后，程序校验白名单与参数、运行 handler、返回 tool_result 并继续请求模型，最多循环 3 轮。Trace 记录工具调用和执行结果。

### 项目使用了哪个 Agent 框架？

> EchoMind 的 HTTP 服务使用 FastAPI，但 Agent 编排主链由项目自己的 AgentOrchestrator 管理，底层调用 Anthropic SDK 的 tool_use。Enterprise 使用自定义 IntelligenceWorkflow 编排 GPT Researcher 的研究能力。LangChain 和 LangGraph 是我单独整理的 Agent 框架知识，不是这两个项目主链采用的框架。
