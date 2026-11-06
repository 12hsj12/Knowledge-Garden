# 记忆系统：Redis 与 ChromaDB

## 核心概念

Agent 的“记忆”不是单一的数据表。按访问方式和时间尺度，可以区分：

- **工作记忆**：当前会话中模型马上要用的少量近期消息。
- **摘要记忆**：把较早对话压缩成较短摘要，避免上下文不断变长。
- **情景记忆**：过去发生过的对话或事件，按当前问题语义召回。
- **用户画像**：跨会话相对稳定的偏好、实体和长期约束。
- **领域知识**：客服政策、产品手册、业务 FAQ 等可复用资料。

这些层的持久时间、读写模式和检索方式不同，可以放在不同存储中。

## Redis 在 EchoMind 中的角色

Redis 是网络化的内存数据结构存储，提供 List、Hash、Set、Sorted Set、Stream 等结构和 TTL 等能力。EchoMind 把 Redis 用作近期工作记忆和会话摘要存储。

### 工作记忆写入与读取

1. Key 按 user_id 和 conv_id 区分会话，形式为 wm:user_id:conv_id。
2. 新消息用 LPUSH 写入 Redis List，使最新消息位于列表前端。
3. 读取时按范围取最近消息，并反转回时间顺序后交给上下文组装。
4. 工作记忆 key 每次写入后刷新 24 小时 TTL。
5. 到达压缩阈值后，旧消息由 LLM 压缩为摘要；摘要和工作记忆都设 24 小时 TTL，列表只保留最近 5 条。

Redis List 适合当前这种追加消息、获取最近 N 条并清理过期会话的访问模式。共享 Redis 也让多进程服务读取同一会话状态；如果只放进单个进程的 dict，重启会丢失，多个服务实例也无法共享。

Redis 支持持久化选项，但应用代码设置 TTL 并不等于业务永久保留记录。是否能从 Redis 故障中恢复，还取决于部署持久化和数据恢复策略。

## ChromaDB 在 EchoMind 中的角色

ChromaDB 提供文档、Embedding、metadata 和相似向量检索。EchoMind 使用不同 collection 保存不同用途的向量数据：

- **episodic**：保存压缩后的对话摘要，按当前 query 找相似历史。
- **user_profile**：保存 LLM 提炼的用户偏好和实体。
- **knowledge_base**：保存客服 FAQ 和知识文档 chunk。

情景记忆查询使用 query 和 user_id / conv_id 元数据筛选，取最多 5 条。当前会话范围结果不足时，代码会在同一 user_id 的其他会话中继续检索。画像以用户为单位单独保存。FAQ collection 则由知识库工具负责查询。

ChromaDB 承担向量召回；Redis 承担近期对话的顺序状态。向量检索不适合替代对话列表的精确读取，Redis key 查询也不提供跨会话语义相似搜索。

## 为什么采用 Redis + ChromaDB

EchoMind 的工作记忆要求低延迟追加、读取最近消息、设置过期时间并按条数压缩；Redis 的 List 与 TTL 正好对应这些操作。情景记忆和知识库要在用户换一种说法时仍能找到语义相近内容；Embedding 加 ChromaDB 更适合这类近邻检索。

关系数据库更适合长期、可追溯、事务性业务记录；如果产品需要保存完整永久对话，应当把数据库作为 durable source of truth，再把近期上下文缓存到 Redis。单进程字典可用于小型原型，但不适合多实例共享状态。Memcached 可以保存简单缓存，但没有 Redis List 这种和消息窗口相配的数据结构。

Redis 与向量数据库不是二选一的同类竞品；它们在本项目中分别处理时间窗口状态与语义召回。

## Agent 一轮对话如何组装记忆

在 EchoMind 的 /chat 请求里：

1. MemoryManager 从 Redis 取当前会话消息与摘要。
2. ChromaDB 按 query 检索情景记忆，按 user_id 获取用户画像。
3. MemoryContext 把摘要、相关历史、画像和近期消息组织成模型可用文本。
4. Agent 根据上下文路由并执行对话。
5. 用户和助手新消息写入 Redis；后台任务更新画像。
6. 工作记忆达到阈值时，旧消息压缩并存入 ChromaDB 情景记忆。

这使当前轮次可以兼顾“刚刚说过什么”和“过去类似情况发生过什么”。

## 与 RAG、Embedding 和可观测性的协作

- Redis 中的工作记忆按会话 key 顺序读取，不是 RAG。
- Chroma episodic 和 knowledge_base 都会做向量召回，但一个找用户历史，一个找领域资料。
- Embedding 负责向量表示，ChromaDB 负责存储与近邻查询。
- Agent 决定要用哪些上下文和工具；Trace 用于检查实际取回了什么、调用是否成功。
- 记忆检索结果是上下文候选，不等于真实事实；要检查来源、时间和用户隔离。

## 面试追问与回答

### 为什么同时用 Redis 和 ChromaDB？

> 当前对话近期消息需要按会话精确读写、控制窗口并自动过期，所以用 Redis List 和 TTL；历史经历和知识库要按语义相关性召回，所以用 ChromaDB collection。两个系统按访问模式分工，不互相替代。

### EchoMind 如何避免上下文无限变长？

> 工作记忆有压缩阈值。达到阈值后，较早消息由 LLM 压缩成摘要写入 Redis，并存一份到 ChromaDB 情景记忆，Redis List 只留最近 5 条。下一轮读取时组合摘要、近期消息和语义相关历史。

### Redis 是不是永久记忆？

> 不等同。项目工作记忆和摘要设置了 24 小时 TTL，它承担短期状态。Redis 支持持久化配置，但业务能否长期保留要看部署策略；长期可审计的完整对话应使用可靠的持久化存储。

### ChromaDB 里的三个 collection 有什么区别？

> episodic 存压缩对话摘要，user_profile 存结构化偏好和实体，knowledge_base 存客服知识 chunk。它们共享向量检索基础设施，但数据用途、metadata 过滤和更新方式不同。