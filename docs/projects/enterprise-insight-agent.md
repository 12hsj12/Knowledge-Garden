# 项目实践：Enterprise Insight Agent

## 项目定位

Enterprise Insight Agent 面向企业研究与竞品分析。它接收结构化研究目标，将研究任务交给网页研究流程，汇总带来源的证据，生成研究报告，并保存任务状态和执行诊断。

当前项目学习工作区位于本机 GPT Researcher 定制代码。这里描述的是核对过的当前工作区路径；项目分支和远端版本可能随时间变化，面试时以实际提交的公开代码为准。

## 从请求到报告的主链

1. FastAPI 企业 API 接收 IntelligenceRequest，并通过 Pydantic 校验任务字段。
2. IntelligenceWorkflow 根据请求构造 GPT Researcher，并调用网页研究流程。
3. 研究器获取网页结果；ContextManager 和 ContextCompressor 对候选页面做内容压缩与语义筛选。
4. 当前启用任务感知证据选择时，规则分类器判断研究任务类别，选择对应 EvidencePolicy。
5. SourceAwareScorer 根据语义相关性与来源权威性先验对候选证据排序。
6. 研究结果形成带 URL 的结构化证据，进入报告生成、主张支持检查和诊断 Trace。
7. API 保存任务状态、结果和诊断，客户端可按任务 ID 查询。

主要代码入口：

- backend/server/enterprise_api.py
- gpt_researcher/enterprise/workflow.py
- gpt_researcher/enterprise/task_policy.py
- gpt_researcher/skills/context_manager.py
- gpt_researcher/context/compression.py
- gpt_researcher/context/source_aware.py

## 技术主题在项目里的落点

### Agent 与研究工作流

主路径由项目的 IntelligenceWorkflow 编排，它把企业任务接入 GPT Researcher 的网页研究和报告能力。研究步骤中有规划、搜索、内容获取、证据筛选和报告生成。该路径可用 Agentic 研究来描述，但不应说成只有一段固定的 ReAct 工具循环。

### RAG 与检索

默认研究资料来自网页搜索和页面抓取。Embedding 用于对长页面切块并按 query 相关性过滤，随后保留结构化来源信息进入证据链。它属于检索增强的研究生成路径，但不是 EchoMind 通过 ChromaDB collection 实现的 FAQ 知识库 RAG。

GPT Researcher 代码还包含可选向量库路径。Enterprise 默认工作流不以 ChromaDB 或 Redis 为检索底座。将“项目里有向量库代码”说成“默认流程使用 ChromaDB”会混淆可选能力与主链。

### 任务感知证据选择

任务分类器按问题信号进行确定性分类，再选择一份固定证据策略。ContextCompressor 仍先依据语义相关性筛选候选；之后 SourceAwareScorer 将语义相似度和来源权威性组合排序。当前策略把来源权威性权重限制在 0.30 内，让语义相关性保持主要影响。

因此项目的任务感知体现在“不同研究问题对来源权威性的偏好不同”，不是把向量匹配替换成类别分类，也不是自动切换数据库。

### MCP

MCP 是可选研究接入能力。MCPClientManager 使用 MultiServerMCPClient 获取配置 MCP Server 的工具；MCPRetriever 可先做工具选择，再调用适合当前研究 query 的工具。默认网页研究路径与 MCP Retriever 要分开讲。

### FastAPI 与 Pydantic

FastAPI 提供 /api/enterprise/tasks 的创建、列表与查询接口；Pydantic 模型描述研究请求、任务记录与结果。API 将 workflow 的执行状态与诊断形成可查询的 JSON 响应。

## 面试回答

### 约一分钟项目介绍

> 我做的 Enterprise Insight Agent 面向企业研究和竞品分析。用户提交研究目标和分析维度后，FastAPI 接收结构化任务，Pydantic 校验请求，再由 IntelligenceWorkflow 调用 GPT Researcher 完成网页研究。网页候选先经过 Embedding 相关性筛选，任务分类会选出来源策略，最后把语义相关性和来源权威性结合排序，形成可追溯证据进入报告生成和证据检查。任务状态和执行诊断通过 API 查询。这个项目的重点是让研究任务、来源选择、报告主张和证据依据形成可检查的链路。

### 为什么做任务感知检索？

> 不同研究任务对来源可靠性的要求不同。事实核验或技术能力确认更需要权威来源；趋势和竞品分析也要保持语义相关，并结合来源质量。系统先分类研究意图，再调整来源权威性在排序中的权重，但语义相关性仍然是主要信号。

### 项目有没有用 Redis 和 ChromaDB？

> 当前 Enterprise 工作区的默认网页研究路径没有把 Redis 或 ChromaDB 作为运行时检索底座。GPT Researcher 代码有可选向量库能力，但这不是默认企业网页研究流程。我会把默认路径与可选组件分开说明。

### MCP 怎么接入？

> MCP Retriever 使用配置的多个 MCP Server，通过 MultiServerMCPClient 获取工具；然后先筛出与研究问题相关的工具，再使用这些工具执行研究。它是可配置的检索路径，不代表每个请求都必然经过 MCP。

### 研究链路如何控制幻觉？

> 系统把网页结果转换成带来源的证据，按语义相关性和来源权威性筛选排序，并检查报告主张与证据之间的支持关系。这样可以减少没有依据的报告内容并追溯问题来源，但不能保证网页数据无误或模型完全不误读，因此仍要呈现限制并做人工或评测检查。