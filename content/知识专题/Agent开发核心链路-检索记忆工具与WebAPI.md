# Agent 开发核心链路：ReAct、MCP、RAG、记忆与 Web API

> 这篇是可复用的专题笔记，按“系统请求怎么走 → 每个组件解决什么 → 两个项目怎么落地 → 怎么讲给别人听”展开。项目实现依据是我本机当前工作区源码；项目一是 Enterprise Insight Agent 的 GPT Researcher 定制工作区，项目二是 EchoMind。代码会变化，涉及实现时以当前分支源码为准。

## 0. 先建立全局地图

我现在把一个 Agent 应用看成一条可观察、可约束的请求链：

**用户请求 → Web API 接收与校验 → 读取上下文/记忆 → 识别任务并路由 → Agent 决定是否调用工具 → 工具检索或操作 → 观察工具结果 → 组织回答 → 记忆写回与过程记录。**

几个名词回答的是不同问题：

| 组件 | 它回答的问题 | 在 EchoMind 中的例子 |
|---|---|---|
| FastAPI | 外部请求如何进入 Python 服务？ | /chat、/knowledge/add、/trace/tools |
| Pydantic | 输入输出数据长什么样、字段类型是否符合约定？ | ChatRequest、ChatResponse |
| Agent / ReAct | 模型如何在回答和执行动作之间作选择？ | 模型发出 tool_use，工具结果再回到模型 |
| MCP | 不同 Agent 主机如何用统一协议发现、描述和调用外部能力？ | 项目一有 MCP 客户端；项目二有自定义工具管理器 |
| RAG | 如何先从知识源找相关内容，再把内容交给模型回答？ | ChromaDB 的知识库检索 |
| Redis | 如何快速保存近期会话状态和摘要？ | EchoMind 的工作记忆 |
| ChromaDB | 如何保存向量化文本并按语义相近程度找回？ | FAQ 知识库、情景记忆、用户画像 |
| Trace / Metrics | 请求为什么走这条路径、用了什么工具、运行表现如何？ | EchoMind 工具调用轨迹和 Prometheus 指标 |

这些组件可以组合，但不是彼此替代品。Redis 不是 Agent，MCP 不是 RAG，FastAPI 不是 REST，Embedding 也不是向量数据库。

## 1. 两个项目到底采用了什么“框架”？

“框架”这个词容易把几层东西混为一谈，我分开说：

1. **应用服务框架**：提供 HTTP 接口的是 FastAPI。
2. **Agent 编排方式**：决定谁处理请求、何时调用工具、如何串联结果。
3. **Agent 执行范式**：例如 ReAct，描述模型“判断—行动—观察”的循环。
4. **底层 SDK / 组件**：例如 Anthropic SDK、GPT Researcher、LangChain 的文本切分和检索压缩组件。
5. **外部能力协议**：例如 MCP，用于 Agent 主机与外部工具服务交互。

### 项目一：Enterprise Insight Agent

当前本机工作区在 GPT Researcher 的研究链路上做企业研究流程、任务分类、证据治理和 API 封装。主链可以概括为：

**FastAPI 企业任务接口 → IntelligenceWorkflow → GPT Researcher 网页检索与报告流程 → ContextManager / ContextCompressor 对网页内容做相关性筛选和压缩 → 结构化证据与报告质量检查 → 保存任务结果、诊断和 Trace。**

它不是“用 ChromaDB 建客服 FAQ 知识库”的那条链。Enterprise 当前默认研究来源是网页；代码中也有可选向量库路径，但这和默认网页研究路径是两件事。当前工作区未发现 Redis 被用作这个项目的运行时组件，也未发现 ChromaDB 是默认企业研究检索库。

相关本机代码位置：

- backend/server/enterprise_api.py
- gpt_researcher/enterprise/workflow.py
- gpt_researcher/skills/context_manager.py
- gpt_researcher/context/compression.py
- gpt_researcher/context/source_aware.py
- gpt_researcher/retrievers/mcp/retriever.py
- gpt_researcher/mcp/client.py

### 项目二：EchoMind

EchoMind 是 FastAPI 服务 + 自己编写的 Agent 编排与工具调用代码。主对话链大致是：

**POST /chat → Pydantic 校验 ChatRequest → 读取 Redis/Chroma 记忆 → 意图识别 → Agent 路由 → 模型可能发起工具调用 → 工具执行并把结果返回模型 → 整理 ChatResponse → 将对话写回工作记忆，异步更新画像。**

相关本机代码位置：

- api/main.py
- agents/agent_orchestrator.py
- agents/tools.py
- mcp/tool_manager.py
- mcp/knowledge_base.py
- memory/conversation_memory.py

EchoMind 的 Agent 编排不是“因为用了 FastAPI 就自动有 Agent”，也不是在这条主链中直接调用 LangChain 或 LangGraph。代码用 Anthropic SDK 的 tool_use 能力，并由项目自己的 AgentOrchestrator 管理白名单、参数校验、执行、返回结果和 Trace。

## 2. Redis：EchoMind 为什么用它？

### 2.1 Redis 是什么？

Redis 可以理解成一个通过网络访问的内存数据结构存储服务。它不只是简单的字符串键值缓存，还提供 List、Hash、Set、Sorted Set、Stream 等结构，并支持过期时间、原子操作和可配置的持久化。

“数据主要在内存中，所以读写快”是理解 Redis 的好入口；但不要把 Redis 说成“数据绝对不落盘”或“只能做缓存”。Redis 能做数据库、缓存、消息和流处理等用途，具体可靠性取决于部署与持久化设置。

### 2.2 它在 EchoMind 里做什么？

EchoMind 把不同时间尺度的记忆放在不同地方：

- **工作记忆：Redis。** 每个 user_id / conv_id 有单独的 List key，写入新消息时用 LPUSH；读取时取最近消息并恢复时间顺序。工作记忆和摘要都设置了 24 小时 TTL。
- **会话摘要：Redis。** 工作消息达到压缩阈值后，调用 LLM 总结较早对话，摘要与旧摘要合并后写入 Redis；消息列表只保留最近 5 条。
- **情景记忆：ChromaDB。** 压缩后的摘要作为文档写入 episodic collection，之后根据当前问题做语义检索。
- **用户画像：ChromaDB。** LLM 从对话中提炼偏好和实体，更新 user_profile collection。
- **FAQ 知识库：ChromaDB。** knowledge_base collection 与记忆集合分开，供知识问答检索。

所以 Redis 在这个项目里负责的是**短期、频繁更新、有过期时间的会话状态**；它不是负责语义搜索的那个库。

### 2.3 为什么是 Redis，而不是全放在别处？

这是一种按数据访问方式做的组件分工：

| 选择 | 适合做什么 | 对 EchoMind 当前工作记忆的取舍 |
|---|---|---|
| Redis | 高频读写、List 操作、TTL、共享会话状态 | 适合保留最近对话和短期摘要；需要关注内存、过期、持久化和恢复策略 |
| 进程内 dict | 单进程原型里的临时状态 | 重启会丢；多进程 / 多实例不共享 |
| 关系数据库 | 长期、可审计、结构化、事务性数据 | 更适合永久记录、复杂查询和事实数据；把每次热上下文读取都走关系查询可能增加负担 |
| Memcached | 简单缓存 | 适合简单键值缓存；Redis 的 List 等数据结构更贴合“最近 N 条对话”这种操作 |
| ChromaDB | 向量化文本的相似检索 | 适合跨会话语义召回，不负责替代 Redis 的短期消息列表 |

面试里不要只说“Redis 快”。更完整的理由是：当前工作记忆按 user_id 和 conv_id 隔离，要求高频追加、读取最近消息、过阈值压缩以及 TTL 自动清理；Redis List 和过期能力正好匹配。较长期的语义记忆放到 ChromaDB，是因为它要按问题找相近的历史内容，而不是只按 key 取回。

## 3. ReAct：模型怎样边想边做、再根据结果调整？

### 3.1 概念

ReAct 是 Reasoning + Acting 的思路：模型先根据用户目标判断下一步，再决定是否采取动作（如调工具）；得到工具返回的 Observation 后，再结合结果继续决定，直到能回答或达到停止条件。

可以用这条循环记忆：

**目标 / 上下文 → 判断下一步 → Action（调用工具）→ Observation（工具结果或错误）→ 再判断 → 回答 / 继续行动**

ReAct 是一种 Agent 执行范式，不等同于 LangChain、LangGraph 之类的框架。一个项目可以用框架实现 ReAct，也可以像 EchoMind 一样自己写循环。

### 3.2 EchoMind 的代码怎样体现这条循环？

在 AgentOrchestrator 的 Agent 调用流程里：

1. 把系统说明、上下文、用户问题以及该 Agent 可用的工具 schema 发给模型。
2. 模型若返回普通文本，就作为本轮 Agent 结果返回。
3. 模型若返回 tool_use，代码检查工具是否在该 Agent 的白名单里，并校验必填参数和类型。
4. 执行 handler，把结果或错误打包为 tool_result，再加回对话消息。
5. 再调用模型，让它根据 Observation 继续处理；每个 Agent 最多循环 3 轮。
6. 工具名、参数、成功状态、耗时、错误等信息进入工具 Trace。

这是一个清楚的 ReAct / tool-use 循环实现。它减少无依据回答的方式不是“让模型思考就不会幻觉”，而是让模型能查询项目数据或知识库，并把工具返回值作为后续上下文；同时用工具白名单和参数校验限制可执行动作。

### 3.3 Thought、Action、Observation 为什么有帮助？有什么边界？

- **Thought / 判断**：模型根据目标和已有上下文选下一步。应用不一定拿到或保存模型完整的内部推理。
- **Action / 动作**：按结构化 schema 调具体工具，而不是只靠自然语言声称“我已经查过”。
- **Observation / 观察**：把真实工具输出、错误、空结果返回模型，帮助它修正下一步或说明限制。

它能让回答多一个外部证据来源，并能在工具失败时让 Agent 看见错误。不过它不能保证零幻觉：检索可能不相关，工具数据可能过时，模型也可能误读或忽略结果。更可靠的工程措施还包括来源/证据校验、拒绝无依据结论、输出约束、日志 Trace 和离线评测。

面试时把 Thought 讲成“完整公开模型思维链”不合适。更准确的表达是：应用在模型与工具之间维护受控的多轮交互，把可审计的动作与结果记录下来。

## 4. MCP：它是统一的“外部调用平台”吗？

### 4.1 正确定义

MCP 是 Model Context Protocol，是一套让 AI 应用主机与外部能力提供方用共同方式交互的协议。主机内的 MCP Client 连接 MCP Server；Server 可以暴露：

- **Tools**：由模型决定调用的动作。
- **Resources**：由主机选择并提供给模型的上下文数据。
- **Prompts**：可复用的提示模板。

因此 MCP 更像“通用插头和交互约定”，不是一个托管所有工具的在线平台，也不自动替你实现鉴权、权限审查、业务逻辑和安全隔离。客户端与服务器仍要通过协议规定的消息和传输方式通信。

### 4.2 两个项目中的差别很重要

**Enterprise Insight Agent：有可选的真实 MCP 客户端链路。** 项目依赖 MCP 适配组件，MCPClientManager 将配置转换为多服务客户端，并获取配置服务器的工具；MCPRetriever 先让模型从候选工具中选择较相关的工具，再用选出的工具做研究。它需要提供 MCP Server 配置才会走这条路径；企业工作流默认的网页研究并不意味着每次都在调用 MCP。

**EchoMind：当前的 MCPToolManager 是项目内部自定义工具管理器。** 它有 Tool 定义（name、description、handler、schema、timeout 等），并负责注册、参数校验、缓存、熔断、重排和降级；AgentOrchestrator 负责将工具 schema 提供给模型并执行调用。但在当前本机源码中没有看到 MCP Client/Server 的标准协议握手和传输实现。因此准确说法是“借用了统一工具注册与调用的设计思路，提供 MCP 风格的内部管理层”，不能把它说成已经完整实现 MCP 标准协议。

### 4.3 为什么需要 MCP，什么时候自己写接口也够？

- 自己控制一个应用、只有少数固定工具时，内部函数 / 普通 HTTP API 往往足够，依赖少、容易调试。
- 需要不同 Agent 主机复用相同工具服务、希望工具可发现且 schema/交互方式一致时，MCP 的互操作价值更明显。
- MCP 不会自动让工具更可靠；还要处理权限、超时、错误、返回数据可信度、敏感信息和审计。

## 5. RAG：从知识内容到可用上下文

### 5.1 RAG 解决什么问题？

RAG（Retrieval-Augmented Generation，检索增强生成）是先从外部知识中找出与问题有关的内容，再把检索结果连同问题提供给 LLM 生成答案。它让模型可以依据项目知识或新资料回答，降低只靠参数记忆回答的风险，并能附带来源依据。

RAG 不等于“用了向量数据库”。最小链路是：

**准备语料 → 切块并保留元数据 → Embedding 编码 → 写入检索系统 → 用户提问也做 Embedding → 找候选片段 → 可选过滤 / 重排 → 把片段放入提示词 → 根据证据生成回答。**

### 5.2 Embedding 与语义匹配

Embedding 模型把文本编码成固定维度的向量，使语义或主题相关的文本通常落在向量空间的相近区域。检索时把 query 转成向量，与已存文档块的向量按所选距离度量找近邻。

“语义相关”不是模型真的理解了知识，也不是命中内容必然正确。词面、语言、领域、文本长度、切块方式、Embedding 模型和距离度量都会影响排序。通常先用向量相似度取候选，再用过滤、关键词 / 混合检索或 reranker 改善结果；RAG prompt 应让模型基于检索内容作答，必要时给出来源或承认资料不足。

### 5.3 EchoMind 的知识库 RAG

代码链路是：

1. 文档导入到 ChromaDB 的 knowledge_base collection。
2. 每篇长文本按中文句号 / 换行切分，目标块大小约 500 个字符；保存 title、chunk_index、total_chunks 元数据。
3. 插入时 Chroma 根据 collection 的 embedding function 生成文档向量；查询时对 query 使用同一个 collection 的 embedding function 编码，再取 Top-K，默认 K=5。
4. knowledge_search 被注册为 Agent 可调用工具之一；工具结果可经过查询改写、多路并行检索和 LLM 重排，再回到 Agent 上下文。
5. Agent 据此形成回答。

这里的“500”是字符目标，不是 token 数，也不是严格上限。当前分块器没有 overlap；若单句本身超过 500 字，单句块也可能超过目标长度。切块是很简单的句子累计方式，没有按标题层级或 token 长度做更细的结构化切分。

### 5.4 一处需要在项目里核实 / 修正的相关性分数

EchoMind 的知识库创建 collection 时没有显式指定向量距离。Chroma 文档说明新 collection 默认距离是 L2；项目代码的注释却写成余弦相似度，并用 1 - distance 生成 score。对于 L2 距离，这个值不是规范化的余弦相似度，甚至可能为负，不能直接当作 0 到 1 的置信度或通用阈值。

复习时我应该这样说：**当前实现用 Chroma 做基于向量近邻的语义召回，Top-K 是主要候选选择；代码把距离转为展示分数的做法需要按实际距离度量核实。** 如果要把分数用于阈值判断或跨查询比较，应显式设置距离度量，并根据该度量定义正确转换；或者直接保留原始距离并清楚命名。换 metric 也需要重建 / 迁移现有 collection，而不是假设旧索引会自动改变。

另外，项目依赖说明使用 Chroma 默认 all-MiniLM-L6-v2 Embedding。是否适合中文客服问答要用实际问题集评估，不能仅凭“默认可用”就断言检索效果好。

### 5.5 EchoMind 的对话记忆也是检索，但用途不同

- 当前会话中最近的消息：Redis List 精确读取，不做语义相似检索。
- 被压缩的跨轮 / 跨会话摘要：写入 Chroma episodic collection，基于 query 做向量检索，并用 user_id / conv_id 元数据过滤；当前对话命中不足时会扩展到该用户的其他会话。
- 用户画像：单独放在 Chroma user_profile collection，LLM 从对话中抽取偏好和实体。
- FAQ 知识：单独放在 Chroma knowledge_base collection。

这几类内容共享 Chroma 服务 / 持久化路径，但 collection 分开，语料含义和检索目的不同。可把它们理解为“当前工作状态、个人经历、用户偏好、领域知识”几类不同的上下文来源。代码中对情景记忆查询使用 Top-K 和 metadata filter，没有显式设一个语义相关度最低阈值；所以召回结果仍需评测和检查。

### 5.6 Enterprise Insight Agent 的任务感知证据检索

项目一的默认链路是网页搜索 / 页面抓取和证据筛选，不是 EchoMind 那种 Chroma FAQ 知识库。其任务感知主要做这些事：

1. 对用户研究问题做确定性规则分类，例如事实核验、技术能力分析、竞品比较、趋势分析、可信度冲突处理或决策建议。
2. 按分类选择一份固定的 EvidencePolicy，得到来源权威性权重。
3. 对搜索页内容做语义相关性筛选：ContextCompressor 用 Embedding 相似度过滤；长页面会切成约 1000 字块，overlap 100 字，默认相似度门槛由配置决定（代码默认回落值为 0.35）。
4. 对保留下来的候选，用“语义相关度 + 来源权威性先验”做排序。代码公式是：(1 - w) × 语义相似度 + w × 来源权威分；w 最大为 0.30，所以设计上让语义相关度仍占主要比重。
5. 把检索结果组织为有来源 URL 的证据，再进入报告和证据治理流程。

因此“任务感知”不是语义向量本身变得更聪明，也不等于分类器会自动换检索数据库。它在当前实现中主要通过任务类别调节来源权威性排序权重；分类器的回退、策略约束和证据 Trace 也需要一起理解。项目还有可选的向量库组件，分块参数为 1000 字、重叠 200 字，但它不是该企业网页研究默认链路的同义说法。

## 6. FastAPI、Pydantic、REST API 分别是什么？

### 6.1 FastAPI

FastAPI 是 Python Web 框架，适合用类型标注构建 HTTP API。它能把 Pydantic 数据模型接到请求体 / 响应模型校验和 OpenAPI 文档上，也支持 async 路由；它的性能优势与 ASGI / Starlette 生态及正确使用异步 I/O 有关，并不意味着任何业务代码都会自动变快。阻塞式调用仍需避免卡住事件循环。

EchoMind 里 /chat、/knowledge/add 等是 FastAPI 路由；项目启动时的 lifespan 初始化 Agent、记忆、知识库和监控对象。同步 Chroma 调用的若干路径会通过 asyncio.to_thread 放到线程中执行。

项目一的 /api/enterprise/tasks 提供创建和查询研究任务接口；/health、/ready 暴露健康状态。

**为什么选 FastAPI？** 两个项目主要是 Python JSON API，需要异步等待模型 / Redis / I/O，且请求与响应结构明确。FastAPI 的类型模型、自动文档、依赖整合和异步路由适合这个目标。Flask 也能实现，但许多 schema / 文档约定要自己补；Django 更偏完整 Web 框架，若不需要 ORM、Admin 等整套功能可能显得更重。选型理由应联系项目需求，不要只说“它快”。

### 6.2 Pydantic

Pydantic 通过 Python 模型声明字段类型、默认值和约束，并在数据进入业务逻辑时解析 / 校验；也能把模型数据序列化。FastAPI 利用这些模型识别请求体、生成 JSON Schema / OpenAPI 文档，并按 response_model 约束响应形状。

所以“Pydantic 只是参数格式校验”不够完整，但方向对。它主要解决数据边界的结构化和类型校验；不能代替业务规则、权限鉴别、数据库约束或安全审查。

### 6.3 REST API

REST 是一种网络应用架构风格，不是某种 Python 框架，也不只是“统一写 API 的语法”。常见的 RESTful API 用 URI 表示资源，使用 HTTP 方法表达对资源的操作，强调统一接口、无状态交互等约束。

- **GET /api/enterprise/tasks/{task_id}**：按资源 URI 查询一个任务，比较 REST 风格。
- **POST /api/enterprise/tasks**：创建 / 执行研究任务，也是资源导向的写操作。
- EchoMind 的 **POST /chat**、**POST /search** 更像动作型接口；系统整体是 HTTP/JSON API，但不必宣称每个路由都是严格 REST。

FastAPI 是“用什么工具写服务”；REST 是“接口遵循哪种架构风格”。一个 FastAPI 项目可以设计 REST API，也可以设计 RPC 风格 API。

## 7. 服务可观测性：为什么要留痕？

Agent 的最终文本不够用来排查问题。要回答“它为什么选这个 Agent、调了哪个工具、工具是否报错、RAG 命中什么、请求慢在哪里”，系统需要保留结构化 Trace 和运行指标。

EchoMind 有 Agent 工具调用轨迹，API 可查询单次或最近工具 Trace；/monitor 返回系统监控摘要，/metrics 暴露 Prometheus 指标。Enterprise 的研究链路也记录任务阶段、证据筛选诊断和任务结果。

Trace 本身不阻止模型胡说，但它让团队能定位“没检索到、工具报错、检索错了、模型误解了”是哪一步，再用评测和修复改善系统。

## 8. 可以用自己的话这样讲

### 8.1 约一分钟介绍 EchoMind 的一次请求

> 用户把问题发到 FastAPI 的 /chat 接口，Pydantic 先把 message、user_id 和可选 conv_id 解析成结构化请求。服务从 Redis 取当前会话最近消息和摘要，再从 ChromaDB 找相关历史、画像等上下文，然后识别意图并路由到合适的 Agent。Agent 通过模型的 tool_use 决定是否查知识库或调用其他白名单工具；工具执行前有参数 schema 校验，结果或错误会回到模型作为下一轮 Observation。最后系统返回结构化响应、写回 Redis 工作记忆，并异步更新用户画像。工具调用会记录 Trace，监控指标由 Prometheus 暴露。这里 Redis 负责短期会话状态，ChromaDB 负责向量语义检索，两者不是同一种存储。

### 8.2 一句话解释两种检索

> EchoMind 的 RAG 是“把 FAQ 切片、向量化并存入 Chroma，问题向量化后检索相关片段交给客服 Agent”；Enterprise 的任务感知检索则是“网页研究结果先按语义相关性筛选，再按研究任务类别调节来源权威性权重”，不是同一套数据库方案。

### 8.3 简短说明 ReAct 与 MCP

> ReAct 描述 Agent 如何在判断、工具动作和观察结果之间循环；MCP 则约定 Agent 主机如何与提供工具 / 资源的服务互通。EchoMind 有明确的模型 tool_use 循环和自定义工具管理层；项目一有可配置的 MCP 客户端路径。两者解决的问题不同。

## 9. 主动回忆题

先遮住上面的回答，试着用 1 到 3 句话讲清楚：

1. EchoMind 为什么用 Redis 保存工作记忆，同时又用 ChromaDB？
2. Redis List、TTL 和会话记忆压缩分别解决什么问题？
3. Embedding 与向量数据库分别做什么？为什么有向量库还要评测检索质量？
4. EchoMind 的知识库如何从文档走到 Agent 回答？chunk 大小、top_k、rerank 在哪一段？
5. 当前 Chroma 距离和 score 的实现有什么需要核实的地方？
6. ReAct 的 Thought、Action、Observation 在 EchoMind 代码中各对应什么？它为什么只能降低而不能消灭幻觉？
7. MCP 的 Client、Server、Host、Tools 各是什么？EchoMind 的 MCPToolManager 为什么不能直接说成完整 MCP？
8. Enterprise Insight Agent 的“任务感知”具体改变了哪一项排序信号？
9. FastAPI、Pydantic、REST API 分别处在哪一层？
10. FastAPI 路由声明为 async 时，为什么阻塞式工作仍可能拖慢服务？
11. 对项目经历做回答时，怎么区分“代码已实现”“可以扩展”和“未来计划”？

## 10. 本篇来源与代码依据

### 官方资料

- [Redis 文档：数据结构存储入门](https://redis.io/docs/latest/develop/get-started/data-store/)
- [Redis 文档：常见应用场景](https://redis.io/docs/latest/develop/use-cases/)
- [MCP Python SDK：Host、Client、Server 和三类原语](https://py.sdk.modelcontextprotocol.io/get-started/first-steps/)
- [Chroma 官方仓库文档：默认向量距离及如何设置距离](https://github.com/chroma-core/docs/blob/main/docs/usage-guide.md)
- [FastAPI：请求体、Pydantic 模型与 OpenAPI](https://fastapi.tiangolo.com/tutorial/body/)
- [Roy Fielding 博士论文：REST 架构风格](https://ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf)

### 项目源码

- [Enterprise Insight Agent 仓库](https://github.com/12hsj12/Enterprise-Insight-Agent)
- [EchoMind 仓库](https://github.com/12hsj12/multi-agent-customer-support-logistics)

以上项目实现描述来自本机当前工作区代码核对。尤其要记住：EchoMind 当前自定义工具管理层不等同于标准 MCP 协议实现；知识库距离分数有实现核实项；Enterprise 当前网页研究路径不使用 Redis / Chroma 作为默认检索底座。
