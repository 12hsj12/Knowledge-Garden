# Agent 系统主线

Agent 应用不是一次大模型调用。它把请求理解、上下文、工具、外部知识、会话状态、服务接口与运行观测组合成一条可运行、可评测的链路。先从一次请求看整体，再沿不同子章节拆解每个组件。

## 一次客服请求的系统路径

以 EchoMind 的售后对话为例：

1. **HTTP 入口**：FastAPI 接收 JSON，Pydantic 解析请求字段。
2. **加载上下文**：Redis 读取当前会话近期消息与摘要；ChromaDB 搜索相关历史和用户画像。
3. **意图识别**：LLM、Embedding、Pattern 提供多路分类信号；OpenJev 可对边界案例选择性仲裁。
4. **Agent 决策**：编排器根据意图和上下文选择主 Agent，并提供可用工具 schema。
5. **Action**：模型可直接答复，也可返回结构化工具调用；应用校验工具白名单和参数后执行。
6. **Observation**：工具结果或错误返回模型，作为下一步决策输入。
7. **检索增强**：RAG 从 ChromaDB 召回知识；多查询、并行召回、去重与 LLM 重排优化候选。
8. **答复与记忆写回**：服务生成响应，将消息写入 Redis，后台更新用户画像。
9. **观察和复盘**：Trace 记录单次请求，Prometheus 汇总服务指标；固定样本评测检查路由和回答质量。

这条链可以概括为：

**请求 → 记忆与意图 → Agent 决策 → 工具行动 → 结果观察 → 回答 → 状态写回与评测**

## 两个项目的系统重心

### Enterprise Insight Agent

企业研究从实时网页搜索和抓取开始。任务分类器选择 EvidencePolicy，网页块按语义相关性筛选，再把相关性与来源权威性融合排序。选中证据进入报告；V2 evidence path 进一步将 claim 与 evidence 关联，执行 Claim Gate 审查。Trace 保留任务策略、证据选择与主张检查信息。

### EchoMind

EchoMind 面向电商售后对话。它把意图识别、Agent 路由、工具调用、客服知识库 RAG、Redis 工作记忆、ChromaDB 情景记忆、OpenJev 仲裁、Trace、Prometheus 和 Docker Compose 连接为可运行服务。项目评测以固定标注样本检查分类和路由表现。

## 章节地图

- [ReAct 与工具调用](react.md)：模型如何在判断、Action 与 Observation 之间循环。
- [意图识别与 Agent 路由](intent-routing.md)：三路基线、OpenJev 选择性仲裁与固定样本结果。
- [RAG 与语义检索](rag.md)：切块、Embedding、向量召回、并行查询、重排与证据审查。
- [MCP 工具协议](mcp.md)：如何通过标准协议连接外部工具服务。
- [记忆系统：Redis 与 ChromaDB](memory.md)：工作记忆、摘要、情景记忆、画像与知识库的分工。
- [服务接口：FastAPI、Pydantic 与 REST](web-api.md)：HTTP、协程、事件循环、ASGI 与项目 API。
- [Agent 系统评测](evaluation.md)：七维度评测框架、固定样本设计与项目结果。
- [Trace、Span 与 Agent 轨迹](trace-observability.md)：一次请求如何连接多个执行阶段，以及 Observation 的区别。
- [LangChain 与 LangGraph](langchain-langgraph.md)：Agent 开发框架的组件边界与适用方式。
- [Prometheus 指标监控](prometheus.md)：Counter、Gauge、Histogram 与 EchoMind 指标路径。
- [Docker 容器化部署](docker.md)：镜像、容器、Compose、网络、数据卷和项目部署。

## 面试回答：怎样讲清楚 Agent 系统

> 用户请求先进入服务接口并经过数据校验，系统读取会话上下文，再判断任务意图和负责的 Agent。模型依据当前信息和工具 schema 决定直接回答或发起结构化工具调用；应用校验动作后执行，把真实结果作为 Observation 返回。RAG 为回答提供外部证据，记忆提供对话连续性，API 组织服务边界。Trace 复盘单次请求，Prometheus 汇总运行指标，固定样本评测检查结果和全流程。这样每个组件都对应一类明确问题。
