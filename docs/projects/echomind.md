# 项目实践：EchoMind

## 项目定位

EchoMind 是面向电商售后场景的多 Agent 智能客服系统。它由 FastAPI 服务接收消息，识别用户意图并路由到适合的 Agent；Agent 可通过受控工具查询知识库或业务数据，随后返回客服回答。系统还管理近期会话、历史情景记忆、用户画像、工具 Trace 和 Prometheus 指标。

## 一次对话请求如何运行

1. POST /chat 接收 ChatRequest。Pydantic 检查 message、user_id 和可选 conv_id 等字段。
2. MemoryManager 从 Redis 读取近期工作记忆和会话摘要，并从 ChromaDB 获取相关历史和用户画像。
3. AgentOrchestrator 结合历史与当前消息识别意图并确定主 Agent / 辅助 Agent。
4. Agent 向模型提供可用工具 schema；模型可以直接回答，也可以返回 tool_use。
5. 代码校验工具白名单和参数，执行 handler；知识库检索返回 ChromaDB 结果，物流工具查询项目本地样本等。
6. 工具结果作为 tool_result 加回消息历史，模型继续生成最终回答；单个 Agent 最多执行 3 轮工具调用。
7. API 将响应建模为 ChatResponse，把用户与助手消息写入 Redis，并异步更新用户画像。
8. 工具执行轨迹可按 request_id 查询，系统监控通过 /monitor 和 /metrics 暴露。

主要代码入口：

- api/main.py
- agents/agent_orchestrator.py
- agents/tools.py
- mcp/tool_manager.py
- mcp/knowledge_base.py
- memory/conversation_memory.py
- monitor/performance_monitor.py

## 技术主题在项目里的落点

### Agent 与 ReAct

EchoMind 自己实现 AgentOrchestrator，并调用 Anthropic SDK 的 tool_use。模型判断是否用工具；应用对工具调用进行白名单和 schema 校验，运行结果返回模型形成下一轮输入。这条显式循环体现 ReAct 的行动—观察方式。FastAPI 负责服务入口，不是 Agent 编排框架。

### Redis 与 ChromaDB

Redis List 保存当前会话的近期消息；摘要也在 Redis 中，消息数到阈值后由 LLM 压缩，工作窗口保留最近 5 条并设置 24 小时 TTL。

ChromaDB 有 episodic、user_profile 和 knowledge_base 三类 collection。episodic 找与当前问题相关的对话摘要；user_profile 保存提取出的偏好与实体；knowledge_base 保存客服 FAQ 文档。它们共享向量检索基础设施，但按用途隔离。

### 知识库 RAG

知识导入时长文本按中文句号和换行切成目标约 500 字符的 chunk，附上标题和 chunk 索引后写入 ChromaDB。查询默认召回 Top-K=5；工具层还支持 query 改写、多查询并行召回、去重和 LLM 重排，再将结果交给 Agent 作为上下文。

Chroma collection 未显式指定距离度量，默认 L2；代码将 1 - distance 作为 score 的处理需要进一步核实。文档依赖使用 Chroma 默认 all-MiniLM-L6-v2，中文问答质量需通过实际召回样本评估。

### MCP 与工具管理层

mcp/tool_manager.py 的 MCPToolManager 是 EchoMind 自己实现的工具注册、参数检查、超时、缓存、熔断、重排和降级管理层。AgentOrchestrator 直接使用它支持工具调用，但当前源码没有 MCP 标准协议客户端/服务器传输。项目面试中应称为自定义工具管理器或 MCP 风格工具层，而不是完整 MCP 实现。

### FastAPI、Pydantic 与 REST

FastAPI 提供聊天、知识库、Trace、监控和评测接口；Pydantic 描述请求和响应。/chat 和 /search 是动作型 API，/trace/tool/{request_id} 和 /knowledge/stats 等更偏资源读取。项目是 HTTP JSON API，其中部分路由有 REST 风格，不必声称所有路由都是严格 REST。

## 面试回答

### 约一分钟项目介绍

> 我做了一个面向电商售后场景的多 Agent 客服系统。FastAPI 接收对话请求后，系统先读取 Redis 中的近期会话和摘要，并从 ChromaDB 查询相关历史与用户画像；意图识别结果再决定由哪个 Agent 处理。Agent 使用 Anthropic tool_use 做受控工具调用，工具输入经过白名单与 schema 校验，结果返回模型形成下一轮上下文。客服知识库用 ChromaDB 做 RAG 召回，物流查询走项目提供的样本数据。最后系统写回会话记忆、后台更新画像，并记录工具 Trace 和 Prometheus 指标。

### Redis 和 ChromaDB 为什么分开？

> Redis 适合按会话 key 快速追加和读取近期消息，并用 TTL 管理工作记忆；ChromaDB 用向量相似度检索跨轮历史、画像和客服知识。它们的数据访问模式不同，因此用不同存储并分开 collection。

### 项目使用了 MCP 吗？

> 当前源码的 MCPToolManager 是自定义内部工具层，负责注册、schema 校验、执行治理和检索优化，但我没有把它描述成完整标准 MCP。工具调用循环和模型 tool_use 是 Agent 执行能力，MCP 互操作协议是另一层概念。

### RAG 的语义召回怎么做？

> 文档先按句子切成目标约 500 字符的 chunk，Chroma 默认 Embedding function 为 chunk 生成向量。查询也向量化后取 Top-K，再由工具层做多 query 召回、合并去重和 LLM 重排，结果返回 Agent。当前距离度量与 score 的转换需要按 Chroma 默认 L2 核实，中文模型效果也要用真实问题集评价。

### Agent 调工具为什么不直接执行模型返回内容？

> 工具调用是外部动作，应用必须检查工具是否属于当前 Agent 白名单、参数是否满足 schema，并处理超时、失败和降级。模型负责提出动作，程序负责授权范围和真实执行。