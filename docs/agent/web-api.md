# 服务接口：FastAPI、Pydantic 与 REST

## 1. FastAPI、Pydantic 与 REST

### FastAPI：Python HTTP 服务框架

FastAPI 用 Python 类型声明组织路由、请求参数和响应模型，结合 Starlette 的 ASGI 服务能力与 Pydantic 数据模型，自动生成 OpenAPI / JSON Schema 文档。它适合以 HTTP JSON API 为边界、需要异步 I/O 的 Python 服务。

### Pydantic：数据边界模型

Pydantic 用字段类型、默认值和约束描述输入输出结构，并在数据进入业务逻辑时解析、校验和序列化。FastAPI 可以根据这些模型生成接口文档和交互式 API 页面。字段校验属于数据边界治理；业务规则、权限、鉴权、访问控制与数据库事务仍由应用负责。

### REST：接口架构风格

REST 是一种网络应用架构风格。RESTful API 通常用 URI 表示资源，用 HTTP 方法表达读取、创建、更新和删除，并遵守统一接口、无状态交互等约束。REST 不是框架，也不是“所有 JSON API”的统称。

FastAPI 是实现 API 的框架；REST 是接口设计风格。FastAPI 可以构建 RESTful API，也可以构建以操作名称为中心的 API。

## 2. HTTP 请求的基本结构

一次 HTTP 请求通常由方法、路径、请求头和可选消息体组成；服务器返回状态码、响应头和消息体。

- **方法**描述操作语义：GET 通常读取，POST 通常提交或创建，PUT/PATCH 通常更新，DELETE 通常删除。
- **路径**标识资源或操作入口，例如 /tasks/123。
- **请求头**携带内容类型、认证信息、追踪标识等元数据。
- **状态码**表达处理结果：2xx 表示成功，4xx 表示请求或权限问题，5xx 表示服务端处理失败。
- **消息体**承载 JSON 等数据，Pydantic 模型用于约束项目中的 JSON 结构。

API 合约要把方法、路径、请求模型、响应模型和错误语义一起说明，客户端才能稳定调用。

## 3. Python 异步基础：协程与事件循环

协程是可以在等待点暂停、之后继续执行的函数，通常以 async def 声明；await 表示等待一个可等待对象完成。事件循环（event loop）负责调度协程：当一个任务等待网络响应时，事件循环可以运行其他已就绪任务。

异步特别适合模型 API、Redis 和网络服务等 I/O 密集工作。它提高的是等待期间的并发利用率，不代表 CPU 密集计算自动并行，也不意味着一个请求一定更快。

在 async 路由里直接调用长时间同步阻塞函数，会占住事件循环，拖慢同一 worker 上其他请求。处理方式包括使用异步客户端、把阻塞调用交给线程池、或把耗时任务交给后台队列。项目应检查具体依赖的调用方式，不能只看函数签名里有没有 async。

EchoMind 的服务路径使用 await 调用异步模型与 Redis；若 ChromaDB 同步调用在异步请求中执行，项目通过 asyncio.to_thread 将该阻塞操作放到线程中。线程化可以释放事件循环继续调度其他协程，但线程池容量和下游延迟仍需考虑。

## 4. ASGI 是什么

ASGI（Asynchronous Server Gateway Interface）是 Python 异步 Web 服务器与 Web 应用之间的接口规范。ASGI server（如 Uvicorn）负责网络连接和协议处理，ASGI application（如 FastAPI/Starlette 应用）负责路由和业务逻辑。

可以把关系理解为：

**客户端 → HTTP → Uvicorn（ASGI server）→ FastAPI（ASGI application）→ 业务代码与依赖服务**

ASGI 支持异步请求处理、长连接等 Web 能力。WSGI 是更早期以同步调用为核心的 Python Web 接口规范。ASGI 本身不提供业务路由或数据校验；它定义服务器和应用如何协作。

## 5. 两个项目中的 API

### 5.1 EchoMind

EchoMind 的 FastAPI 应用在 api/main.py 创建，并在 lifespan 中初始化 Agent、记忆、工具管理器、Chroma 知识库和监控等组件。

主要路由包括：

- POST /chat：接收 ChatRequest，返回 ChatResponse。
- GET /health：健康检查。
- GET /trace/tool/{request_id} 与 GET /trace/tools：查询工具执行轨迹。
- GET /monitor 与 GET /metrics：读取监控摘要和 Prometheus 指标。
- POST /knowledge/add：导入知识库文档。
- POST /search：展示查询改写、检索和重排链路。

ChatRequest、ChatResponse 使用 Pydantic 定义。API 把 HTTP 请求转换为 Agent 调用，再将 Agent 结果整理成稳定的响应模型。

### 5.2 Enterprise Insight Agent

Enterprise API 在 backend/server/enterprise_api.py 中注册。POST /api/enterprise/tasks 创建并执行研究任务；GET /api/enterprise/tasks 列出任务；GET /api/enterprise/tasks/{task_id} 查询任务；/health 和 /ready 提供健康检查。

IntelligenceRequest、IntelligenceResult 和 TaskRecord 等模型约束请求、执行结果与任务状态。API 将请求交给 IntelligenceWorkflow，再把任务进度、研究结果、诊断和 Trace 暴露给客户端。

## 6. 项目接口的 REST 风格

Enterprise API 用 /tasks 表示任务集合、/tasks/{task_id} 表示单个任务，并以 GET 查询、POST 创建，符合资源导向的 REST 风格。

EchoMind 的 /chat、/search、/eval/run 是动作型 API；它们是标准 HTTP JSON 接口，其中部分路由围绕操作命名。面试时按实际接口风格描述，比把所有 HTTP API 都称作严格 REST 更准确。

接口设计时需保证读取与写入语义清楚、状态码一致、响应模型稳定，并在服务端验证用户和会话范围，不能仅依赖前端传来的 user_id。

## 7. 为什么两个项目都采用 FastAPI

两个项目都是 Python Agent 服务，使用 HTTP JSON 接收任务和返回结构化结果。FastAPI 的 Pydantic 集成让边界模型明确，自动生成 OpenAPI 文档，异步路由适合编排模型 API、Redis 和网络 I/O。因此它符合当前项目形态，也便于本地部署与服务集成。

选择框架取决于服务需求：Flask 能构建轻量 HTTP API；Django 提供更完整的站点、ORM 和管理后台。当前服务以 Agent API 为主，FastAPI 的类型模型和异步支持更贴合工作流。

## 8. 服务、Agent 与部署的边界

FastAPI 负责 HTTP 入口、请求解析、依赖装配和响应输出；Agent 编排负责意图识别、工具调用和研究决策；Redis、ChromaDB、模型服务和 MCP server 是外部依赖。Docker 将运行时环境与服务依赖组织成可部署容器；Trace 与 Prometheus 分别帮助检查请求路径和服务指标。

详细说明见[Trace、Span 与 Agent 轨迹](trace-observability.md)、[Prometheus 指标监控](prometheus.md)和[Docker 容器化部署](docker.md)。

## 9. 面试追问与回答

### FastAPI 与 REST 是什么关系？

> FastAPI 是 Python Web 框架，用来实现 HTTP 服务；REST 是组织资源 URI、HTTP 方法和无状态交互的架构风格。FastAPI 能实现 RESTful API，也能实现动作型 API。

### Pydantic 除了参数格式校验还做什么？

> Pydantic 模型描述字段类型、默认值和约束，帮助解析输入、校验数据边界、序列化输出，并让 FastAPI 生成 OpenAPI 文档。业务规则、权限和数据访问控制仍由应用实现。

### FastAPI 的 async 一定比同步快吗？

> 异步对大量 I/O 等待任务有帮助，因为协程等待网络响应时事件循环可以调度其他请求；CPU 密集工作不会因此自动并行，阻塞调用也会占住事件循环。要结合客户端是否异步以及同步工作如何隔离来判断。

### asyncio、ASGI 与 FastAPI 如何串起来？

> async def 定义协程，await 在等待点把控制权交给事件循环；Uvicorn 作为 ASGI server 接收 HTTP 并调用 FastAPI 这个 ASGI application。FastAPI 再解析请求并调用业务服务。

### EchoMind 怎么避免同步 Chroma 调用卡住事件循环？

> Chroma 客户端在相关路径是同步调用，异步服务通过 asyncio.to_thread 把阻塞操作交给线程运行，让事件循环继续调度其他协程。实际吞吐还取决于线程池和向量库响应时间。
