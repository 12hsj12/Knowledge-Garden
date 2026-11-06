# 服务接口：FastAPI、Pydantic 与 REST

## 核心概念

### FastAPI：Python HTTP 服务框架

FastAPI 用 Python 类型声明组织 HTTP 路由、请求参数和响应模型。它和 Starlette 的 ASGI 服务能力、Pydantic 数据模型和 OpenAPI 文档生成紧密结合，适合构建以 JSON API 为主的异步服务。

async 路由有助于在等待模型 API、Redis、数据库或网络 I/O 时并发处理其他请求。它不会让阻塞代码自动变成异步：同步阻塞操作若在事件循环里运行，仍会拖慢其他请求，需要异步客户端、线程池或任务队列等合适的执行方式。

### Pydantic：数据边界模型

Pydantic 模型用字段类型、默认值和约束描述数据结构，并在数据进入业务代码时完成解析、校验和序列化。FastAPI 可以把这些模型用于请求体与响应模型，生成 OpenAPI / JSON Schema，并提供交互式 API 文档。

Pydantic 校验的是数据结构与声明约束；业务规则、权限鉴别、访问授权和数据库事务仍由应用负责。

### REST：接口架构风格

REST 是一种网络应用架构风格。常见 RESTful API 用 URI 表示资源，用 HTTP 方法表达对资源的操作，强调统一接口和无状态交互等约束。REST 不是 Python 框架，也不等于所有 HTTP JSON API。

FastAPI 是实现服务的框架；REST 是接口的设计风格。FastAPI 项目可以实现 RESTful API，也可以暴露动作型 RPC 风格接口。

## 两个项目中的服务接口

### EchoMind

EchoMind 使用 FastAPI 创建 ASGI 应用，在应用 lifespan 中初始化 Agent、记忆、工具管理器、Chroma 知识库、监控和评测组件。

主要路由包括：

- **POST /chat**：请求体使用 ChatRequest，返回 ChatResponse。
- **GET /health**：健康检查。
- **GET /trace/tool/{request_id}** 与 **GET /trace/tools**：查询工具调用轨迹。
- **GET /monitor** 与 **GET /metrics**：读取监控摘要和 Prometheus 指标。
- **POST /knowledge/add**：批量导入知识库文档。
- **POST /search**：展示查询改写、检索和重排链路。

ChatRequest 和 ChatResponse 使用 Pydantic。/chat 的服务流程包含 await 的模型与 Redis 调用；ChromaDB 的同步客户端调用在若干路径通过 asyncio.to_thread 执行，避免直接堵塞事件循环。

### Enterprise Insight Agent

项目一的 Enterprise API 在 backend/server/enterprise_api.py 中创建 FastAPI APIRouter。**POST /api/enterprise/tasks** 创建并执行研究任务；**GET /api/enterprise/tasks** 列出任务；**GET /api/enterprise/tasks/{task_id}** 查询单个任务；/health 和 /ready 检查服务与存储可用性。

IntelligenceRequest、IntelligenceResult 和 TaskRecord 由 Pydantic 约束字段结构；API 把请求交给 IntelligenceWorkflow，并保存任务状态、结果、诊断和 Trace。

## REST 风格如何判断

Enterprise API 用 /tasks 和 /tasks/{task_id} 表示任务集合与单个任务，并通过 GET 和 POST 表达读取与创建，属于资源导向设计。

EchoMind 的 /chat、/search、/eval/run 用动作名称表达操作，是动作型 HTTP API。它仍然是 HTTP JSON API，但不需要把每一个路由都称作严格 REST。

设计接口时，更重要的是请求语义一致：读取请求不应产生意外写入；创建、更新与查询的状态码、请求体、响应体和错误结构要清楚；不同用户与会话的数据访问必须受到服务端校验。

## 为什么两个项目都选择 FastAPI

两个项目都以 Python 为主，用户侧通过 HTTP JSON 与服务交互，且请求结构和响应结构明确。FastAPI 能用类型模型表达接口约定，自动生成 OpenAPI 文档，并支持异步调用 LLM 和 I/O 服务，适合当前工程体量。

Flask 可以构建同样的 API，但项目需要自行补上更多模型校验和文档约定。Django 提供更完整的 Web 体系，适合需要 ORM、管理后台和整套站点能力的场景；当前服务主要是 Agent API，因此 FastAPI 更贴合需求。

## 与 Agent 执行链路的协作

FastAPI 和 Pydantic 位于模型运行之前和之后：入口校验请求，调用应用服务，并把内部结果组织成稳定响应。REST 影响客户端如何理解接口资源和操作。Agent 编排、MCP、RAG 与记忆都属于 API handler 所调用的内部能力。

一个完整请求需要分清：

1. HTTP 边界：路径、方法、请求头、状态码。
2. 数据边界：Pydantic 请求和响应模型。
3. 应用逻辑：Agent 调度和业务规则。
4. 外部依赖：模型、Redis、ChromaDB、数据库和 MCP 服务。
5. 可观测性：Trace、日志、指标和错误信息。

## 面试追问与回答

### FastAPI 与 REST 是什么关系？

> FastAPI 是 Python Web 框架，用来实现 HTTP API；REST 是一种接口架构风格，指导资源 URI、HTTP 方法和无状态交互等设计。FastAPI 可以实现 REST API，也可以实现动作型 API。

### Pydantic 只负责参数校验吗？

> 它用模型声明数据结构、字段类型、默认值和约束，帮助解析输入、校验边界并序列化输出。FastAPI 还能利用这些模型生成 OpenAPI 文档。业务规则、权限和安全校验仍由应用实现。

### FastAPI 的 async 一定比同步快吗？

> 不一定。异步适合等待 I/O 时释放事件循环；若在 async 路由里直接执行长时间同步阻塞工作，仍会拖慢并发请求。需要检查数据库、模型 SDK 和文件处理的实际调用方式。

### 为什么项目一和 EchoMind 都用 FastAPI？

> 两个项目都需要以 HTTP JSON 暴露 Agent 能力，且请求与响应结构清楚。FastAPI 的类型模型、OpenAPI 文档和异步路由让 API 边界易于说明，并适合调用异步模型与 I/O 服务。