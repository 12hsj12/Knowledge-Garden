# 项目实践：EchoMind

## 1. 项目定位

EchoMind 是面向电商售后场景的多 Agent 智能客服系统。FastAPI 接收对话请求后，系统读取会话状态、识别意图、选择 Agent，并通过知识库或业务工具获取信息。回答完成后，近期消息写入 Redis，历史经验和知识存入 ChromaDB，工具轨迹和服务指标支持调试与运行观察。

## 2. 一次对话请求的完整链路

1. POST /chat 接收 ChatRequest，Pydantic 解析 message、user_id 和可选 conv_id。
2. MemoryManager 从 Redis 读取近期会话和摘要，从 ChromaDB 获取语义相关历史和用户画像。
3. IntentRecognizer 融合 LLM、Embedding 和 Pattern 三类信号，形成意图与置信度。
4. 不确定或存在分歧的意图可以选择性送到 OpenJev 仲裁；意图结果进入 AgentOrchestrator 路由。
5. Agent 将受控工具 schema 提供给 Anthropic Messages API。
6. 模型返回文本或 tool_use；应用校验工具白名单和参数，执行相应 handler。
7. 工具结果作为 tool_result 加回消息历史，模型继续生成；单个 Agent 的工具循环最多 3 轮。
8. API 生成 ChatResponse，将用户与助手消息写入 Redis，并异步更新用户画像。
9. 请求 Trace 保留意图、路由、工具和延迟等信息；Prometheus 抓取服务指标。

主要代码入口：

- api/main.py
- agents/agent_orchestrator.py
- core/intent_recognizer.py
- agents/tools.py
- mcp/tool_manager.py
- mcp/knowledge_base.py
- memory/conversation_memory.py
- monitor/performance_monitor.py
- Dockerfile 与 docker-compose.yml

## 3. 意图识别：三路基线与 OpenJev 仲裁

### 3.1 三类基线信号

IntentRecognizer 综合三路信号：LLM 分类理解上下文，Embedding 匹配相似意图表达，Pattern 识别明确关键词、格式与规则特征。开启 Embedding 时融合权重为 0.7、0.2、0.1；LLM 和 Embedding 可并发执行。意图识别的 Embedding 优先使用配置的远端向量接口；远端接口不可用时使用 256 维字符 n-gram 哈希向量作为本地近似匹配。它与 ChromaDB 知识库的文档 Embedding 是两条不同链路。LLM 失败时系统按可用 Embedding 或 Pattern 信号降级。

### 3.2 OpenJev 的位置

OpenJev 是选择性仲裁路径，不是三路基线之外的等权第四票。系统针对 baseline 为 other、综合置信度偏低或信号互相冲突的情况调用配置的 OpenJev 服务；高置信结果可以跳过仲裁。有效候选被采纳，失败或无效结果保持现有基线判断。Trace 和响应保留决策来源、状态及耗时。

意图结果决定 Agent 路由，知识库 RAG 则在进入相关 Agent 后查找支持答案的资料。详细机制见[意图识别与 Agent 路由](../agent/intent-routing.md)。

### 3.3 固定样本评测结果

G5 OpenJev paired 使用 38 条固定案例；通用标签指标覆盖 36 条适用样本，细粒度物流标签覆盖 15 条：

| 指标 | Baseline | OpenJev |
| --- | ---: | ---: |
| 通用意图准确率 | 32/36 = 88.89% | 34/36 = 94.44% |
| 通用意图 Macro-F1 | 0.9065 | 0.9394 |
| 细粒度物流准确率 | 13/15 = 86.67% | 15/15 = 100% |
| 细粒度物流 Macro-F1 | 0.8714 | 1.0000 |
| 端到端延迟 P50 | 1356 ms | 1513 ms |
| 端到端延迟 P95 | 2442 ms | 7150 ms |

本轮 OpenJev 仲裁 6 次并采纳 6 次，32 个样本跳过。结果表明，选择性仲裁改善了这组固定样本的意图识别，同时增加被仲裁请求的尾延迟。

其他版本的固定样本结果需保留各自数据集与配置：

| 评测版本 | 通用意图 | 细粒度物流 | 路由/工具补充 |
| --- | --- | --- | --- |
| G4 baseline g0 | 31/36 = 86.11%，Macro-F1 0.9099 | — | 主路由 37/38 = 97.37% |
| G4 logistics g2 | 33/36 = 91.67%，Macro-F1 0.9186 | 13/15 = 86.67%，Macro-F1 0.8714 | 主路由 36/38 = 94.74%；指定工具成功匹配 5/5；升级候选 3/3 |
| G4 local g3（Mapika GGUF） | 33/36 = 91.67%，Macro-F1 0.8788 | 14/15 = 93.33%，Macro-F1 0.9314 | 主路由 37/38 = 97.37%；必需 Agent 覆盖 38/38 |

G4 与 G5 使用不同冻结样本和配置，应该按评测版本分别陈述。准确率回答“总体预测命中多少”；Macro-F1 让每个类别等权参与平均；路由准确率检查意图最终是否进入期望 Agent；工具匹配检查对应动作是否执行到位。

## 4. RAG：知识库的构建、召回和意图识别协作

### 4.1 文档与向量

知识导入时，文本按中文句号和换行累计成目标约 500 字符的 chunk，并保留标题和 chunk_index 等 metadata。ChromaDB collection 的默认 Embedding function 为文档生成向量，稳定 ID 与 upsert 支持重复导入更新。

查询时，工具层可改写成多个 query 并行召回，合并候选、去重，再由 LLM reranker 重新排序，选择有限证据返回 Agent。意图识别先让系统选对业务路径，RAG 再让该路径拿到相关资料。

collection 没有显式指定距离度量时，Chroma 默认使用 L2。项目代码用 1 - distance 作为展示 score；它是距离的变换值，不是校准后的置信度。项目默认 Embedding 是 all-MiniLM-L6-v2；中文检索质量应按固定中文客服样本的召回与证据命中结果解释。

### 4.2 召回质量的业务意义

客服问句经常口语化、简短且缺少产品术语。多查询改写帮助覆盖同义表达和问题子面；并行召回降低串行等待；去重避免同一内容反复占据 Top-K；LLM 重排让真正回答问题的片段优先进入模型上下文。可以用 Recall@K、证据覆盖率、重复率、最终回答正确性和延迟一起观察链路。

更多原理见[RAG 与语义检索](../agent/rag.md)。

## 5. Redis 与 ChromaDB：工作记忆和长期召回

### Redis

工作记忆 key 为 wm:{user_id}:{conv_id}。新消息以 LPUSH 写入 List，读取后反转为时间顺序；写入刷新 24 小时 TTL。消息达到 15 条压缩阈值后，旧消息交给模型压缩为摘要，工作列表保留最近 5 条，摘要 key 也设置 24 小时 TTL。

### ChromaDB

episodic collection 保存压缩的历史摘要并按 query 语义检索；user_profile 保存用户偏好和实体；knowledge_base 保存客服文档。Redis 用会话 key 取得当前窗口，ChromaDB 用向量相似度找到历史或业务知识。完整对比见[记忆系统：Redis 与 ChromaDB](../agent/memory.md)。

## 6. 工具、ReAct 与 MCP

EchoMind 的 AgentOrchestrator 使用 Anthropic SDK tool_use 管理 Action—Observation 循环。模型提出工具名和参数后，程序检查 Agent 白名单及 schema，执行 handler，将 tool_result 返回模型。FastAPI 是服务框架，不是 Agent 编排框架。

mcp/tool_manager.py 中的 MCPToolManager 是自定义工具管理层，负责工具注册、参数治理、超时、缓存、熔断、查询改写、并行召回、去重、重排和降级。当前这条实现按自定义工具管理器描述；MCP 标准协议客户端/服务端是另一层互操作协议概念。

## 7. Trace 与 Prometheus

### 7.1 工具 Trace

每次请求有 request_id 与时间戳，轨迹记录意图、主 Agent、辅助 Agent、工具列表、工具调用、升级情况和总耗时。每条工具调用保留 agent_type、tool_name、tool_use_id、输入摘要（物流单号会脱敏）、成功状态、耗时、缓存/重排状态、错误和升级原因。最近轨迹由有界 deque 保存，并可通过 /trace/tool/{request_id} 和 /trace/tools 查询。

这些字段可以回答：系统识别了什么意图、路由到谁、用了什么工具、工具输入输出状态如何、延迟发生在哪里。Trace 是请求级执行记录，不是 Prometheus 时间序列。ReAct 的 Observation 是返回模型继续决策的工具结果；Trace 保存它的诊断摘要，供开发者事后复盘。

### 7.2 Prometheus 指标

FastAPI 的 /metrics 暴露 Prometheus 格式指标。当前监控组件定义 Agent 成功率 Gauge、Agent 延迟 Histogram、工具成功率 Gauge，以及 requests_total Counter。PerformanceMonitor 定期更新成功率类 Gauge 和平均耗时 Histogram；requests_total 当前已注册，但采集循环没有请求递增埋点。agent_latency_ms 记录的是采集周期读取到的平均延迟，不是逐请求原始耗时分布。Prometheus 以 10 秒间隔抓取 echomind:8000/metrics，适合观察已更新指标随时间的变化。

Trace 用来追踪单次请求的详细路径；Prometheus 用聚合时间序列回答整体成功率、延迟和趋势问题。项目部署中的 scrape 配置和指标类型见[Prometheus 指标监控](../agent/prometheus.md)。

## 8. Docker：项目如何打包和运行

Docker 把应用运行时、依赖和启动命令构建成镜像，再以容器运行。EchoMind 使用多阶段 Dockerfile：依赖阶段安装 Python 包并预下载 ONNX 模型；生产阶段复制依赖与应用代码，以非 root 用户运行 Uvicorn，并提供 /health 健康检查；development 阶段增加热重载。

docker-compose.yml 将 EchoMind、Redis、ChromaDB、OpenJev、Prometheus 和 Nginx 放入同一服务网络。容器间通过服务名连接，例如 redis、chromadb、openjev；Redis、ChromaDB 和 Prometheus 使用 volume 保存运行数据；Compose 健康检查和服务依赖顺序控制启动条件。

常见工程要点包括：容器内 localhost 指向容器自身，容器间使用 Compose 服务名；端口映射区分宿主机和容器端口；密钥放在环境配置并由 .dockerignore 排除构建上下文；持久化数据放入 volumes；健康检查用于服务就绪状态；镜像 tag 与依赖锁定影响可复现性。完整概念见[Docker 容器化部署](../agent/docker.md)。

## 9. Agent 评测：固定样本与完整链路

项目评测不能只汇报一个分类准确率。固定标注样本先回答意图识别、路由、Agent 覆盖和工具结果是否正确；对话评测器还支持 intent Accuracy、Macro-F1，以及 relevance、accuracy、completeness、helpfulness 四项 LLM-as-Judge 评价，默认通过阈值 0.75 并可与基线比较。

完整 Agent 评测再从七个角度组织：输出质量、RAG 召回完整性、证据可靠性、工具调用校验、稳定性与异常处理、成本效率、安全与权限。每项都要定义样本、判定方式和业务影响。项目报告时应按评测版本准确说明数据集、样本数和指标，避免把分类评测直接说成端到端质量，也避免把不同数据哈希的运行数字拼成一个结果。

完整问题拆解见[Agent 系统评测](../agent/evaluation.md)。

## 10. 服务 API

FastAPI 提供 /chat、/health、知识库、检索、Trace、监控与评测等接口。Pydantic 描述请求和响应结构。/chat 和 /search 是动作型 HTTP API；项目也提供按 request_id 查询 Trace 的读取路由。关于事件循环、HTTP、ASGI 和接口设计见[服务接口：FastAPI、Pydantic 与 REST](../agent/web-api.md)。

## 11. 面试回答

### 约一分钟项目介绍

> 我做了一个面向电商售后的多 Agent 智能客服。FastAPI 接收请求后，系统从 Redis 读取近期会话，并从 ChromaDB 取回相关历史和用户画像。意图识别融合 LLM、Embedding 和 Pattern 信号；边界案例可以选择性由 OpenJev 仲裁，然后 AgentOrchestrator 决定负责的 Agent。模型通过 Anthropic tool_use 提出结构化工具调用，程序校验白名单与参数后执行，并把结果作为下一轮输入。客服知识库用 ChromaDB 做多查询并行召回、去重与重排。请求结束后写回记忆，Trace 记录执行路径，Prometheus 提供服务指标，Docker Compose 组织依赖服务。

### 为什么用 Redis 而不是把消息存在进程内？

> Redis 让不同 worker 通过同一用户和会话 key 读取近期消息，也能用 List 操作管理消息窗口和用 TTL 控制状态过期。进程内字典只在当前 worker 可见，无法可靠共享会话。

### OpenJev 在识别链路中的价值是什么？

> LLM、Embedding 和 Pattern 先给出基线候选；OpenJev 只处理低置信、冲突或 other 等边界案例。固定 G5 样本里通用意图从 32/36 提升到 34/36，细分物流从 13/15 提升到 15/15，同时端到端 P95 从约 2.44 秒增加到 7.15 秒。这个数字把识别效果和选择性仲裁的延迟成本一起呈现。

### 如何解释简历里的准确率？

> 我先说明是哪一版固定样本，再讲标签范围和分母。例如 G4 logistics 版本通用标签是 33/36，即 91.67%，细粒度物流是 13/15，即 86.67%；G4 local provider 的细粒度物流是 14/15，即 93.33%；G5 OpenJev paired 的细粒度物流是 15/15。它们来自不同评测配置，不能合并成一个总体提升。Macro-F1 补充体现类别均衡表现，路由和工具匹配指标则检查预测是否转化为正确动作。

### Trace 和 Prometheus 有什么区别？

> Trace 关联一次请求里的意图、Agent、工具调用和阶段耗时，适合复盘单次执行；Prometheus 以固定时间间隔抓取聚合指标，适合观察一段时间内的成功率与延迟趋势。Trace 回答“这个请求怎么走”，指标回答“这段时间整体表现如何”。

### Docker 在项目里解决什么问题？

> Dockerfile 固化 Python 运行时、依赖和应用启动方式；Compose 将 EchoMind 与 Redis、ChromaDB、OpenJev、Prometheus 和 Nginx 放到同一网络并配置数据卷与健康检查。这样开发、部署和排错时使用相同的服务拓扑；配置、端口、持久化与服务就绪仍要按部署环境正确设置。
