# ReAct 与工具调用

## 核心概念

ReAct 是 Reasoning 与 Acting 的组合思路。模型针对目标判断下一步，必要时执行外部动作，再根据动作返回的 Observation 更新判断。它让语言模型能够把“回答问题”和“执行可验证的操作”放进同一条循环里。

常见循环是：

**目标与上下文 → 判断下一步 → Action → Observation → 再次判断 → 完成或继续**

Action 可以是查订单、查询知识库、调用计算器或请求外部服务。Observation 是工具实际返回的数据、空结果或错误。循环需要有停止条件，例如模型给出最终答案、达到最大调用次数或发生不可恢复的失败。

ReAct 是 Agent 执行范式，不是某一个 Python 框架。项目可以用 LangChain、LangGraph 等工具实现，也可以直接使用模型 API 编写自己的循环。

## 项目中的实现

### EchoMind

EchoMind 在 AgentOrchestrator 中直接维护工具调用循环：

1. 把 system prompt、会话上下文、用户消息和 Agent 可用工具 schema 发给 Anthropic Messages API。
2. 如果模型返回文本，当前 Agent 完成本轮处理。
3. 如果返回 tool_use，系统确认工具属于该 Agent 白名单，并校验必填参数和类型。
4. 执行工具 handler，把结果或错误组织成 tool_result，加入消息历史。
5. 再次请求模型，让它基于工具结果继续处理；单个 Agent 的工具循环最多 3 轮。
6. 记录工具名称、输入、成功状态、耗时和错误等 Trace。

因此项目里的循环不是抽象地“模型在思考”，而是可以在代码里指出模型请求、结构化工具调用、白名单校验、handler 执行、结果回传和下一轮模型请求。

### Enterprise Insight Agent

Enterprise Insight Agent 的主路径是研究工作流：围绕研究目标组织网页检索、页面内容获取、证据筛选、报告生成和证据治理。这个流程具有 Agentic 的规划与行动特点，但当前工作流不应被简化成 EchoMind 的单工具调用循环。若使用配置的 MCPRetriever，工具选择与工具研究执行是另一个可选路径。

## Thought、Action、Observation 如何协作

- **Thought / 判断**：模型选择直接回答、调用哪个工具，或基于结果再行动。应用侧通常只需要观察结构化动作和结果，不要求记录模型完整的内部推理。
- **Action / 行动**：模型输出结构化工具名称与参数，应用执行受控 handler。
- **Observation / 观察**：工具结果或失败信息返回给模型，作为下一轮输入。

外部数据能为回答增加依据，Action 和 Observation 也让运行过程可审查。它们只能降低无依据作答的概率，不能保证消除幻觉：检索内容可能不相关，工具数据可能过时，模型也可能误读结果。

可靠的实现还需要工具白名单、参数校验、超时和错误处理、来源检查、输出约束、Trace 和评测。模型的推理过程不能代替这些工程控制。

## 与其他 Agent 组件的协作

ReAct 回答“模型如何循环行动”。工具注册表描述可调用的能力；MCP 在需要跨应用复用服务时提供标准接入方式；RAG 工具则把检索结果作为一种 Observation 带回模型。Redis 和数据库为工具提供状态或业务数据，但它们不是 ReAct 的组成定义。

EchoMind 一次知识问答可以这样串起来：Agent 收到知识库工具 schema → 模型决定调用 → RAG 在 ChromaDB 中检索 → 检索片段作为工具结果返回 → 模型依据片段组织回答 → Trace 记录调用路径。

## 面试追问与回答

### ReAct 和普通 LLM 调用有什么区别？

普通调用通常是输入上下文、输出一次答案。ReAct 允许模型通过结构化动作访问外部能力，并把执行结果放回上下文进行后续判断。关键差异在于有“工具行动—结果观察—继续决策”的闭环。

### Thought、Action、Observation 会不会消除幻觉？

不会。工具结果提供外部依据，结构化调用限制动作范围，但检索错误、数据过期和模型误读仍会造成错误。还要用证据校验、拒答策略、日志和评测管理风险。

### EchoMind 的 ReAct 在哪段代码里？

在 AgentOrchestrator 的模型调用中：请求携带工具 schema，识别 tool_use 后校验白名单和参数、执行 handler、构造 tool_result 并再次请求模型；循环上限为 3 轮。工具输入、执行状态和耗时进入 Trace。

### 项目使用了哪个 Agent 框架？

EchoMind 的服务用 FastAPI，但 Agent 编排主链是项目自己的 AgentOrchestrator，调用 Anthropic SDK 的 tool_use；不能把 FastAPI 说成 Agent 框架。Enterprise Insight Agent 以 GPT Researcher 的研究能力为基础，再由项目自己的 IntelligenceWorkflow 编排企业研究、证据和报告链路。