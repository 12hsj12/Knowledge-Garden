# Agent 可观测性：Session、Trajectory、Trace 与 Span

## 先用一次客服请求建立画面

一个用户在 EchoMind 中说“快递显示签收，但我没有收到”。系统先读取会话记忆，识别意图，选择 Agent；模型决定查询物流工具，应用执行查询，再把结果返回模型，最后生成答复。

这段过程可以从四个角度观察：

- **Session（会话）**：把同一个用户的一系列对话轮次归在一起。
- **Trajectory（执行轨迹）**：按时间顺序描述 Agent 如何从输入走到输出，包括决策、工具行动、观察结果和最终回答。
- **Trace（请求追踪）**：围绕一次请求，把各个阶段的执行证据、状态和耗时关联起来，便于开发者排查。
- **Span（操作跨度）**：Trace 中某个具体操作的起止区间，例如一次模型请求、一次检索或一次工具调用。

在常见分布式追踪模型中，一个 Trace 由一个根 Span 和若干子 Span 组成；Span 记录名称、开始/结束时间、状态、属性、事件和父子关系。Trace ID 连接同一请求内的工作，Span ID 标识单项工作。

## Session、Trajectory、Trace 的关系

| 对象 | 主要回答 | 常见边界 | 对 Agent 的意义 |
| --- | --- | --- | --- |
| Session | “这是谁的哪段对话？” | 用户和对话 ID，可跨多个请求轮次 | 记忆隔离与多轮上下文 |
| Trajectory | “Agent 是怎样一步步完成任务的？” | 一个 Agent 任务或一次交互的决策顺序 | 分析规划、工具选择、观察和最终回答 |
| Trace | “这一次请求经过哪些组件，哪里耗时或失败？” | 通常以一次请求或一次研究任务为根 | 定位模型、检索、工具、依赖和生成环节 |
| Span | “某一个操作本身发生了什么？” | Trace 中的一段操作；可有 parent span | 比较操作耗时、错误状态和调用关系 |

项目中往往会为一个 Session 产生多条 Trace：用户每发起一轮新的请求，就产生一次请求执行记录。Trajectory 描述 Agent 的行动顺序；Trace 提供把行动与时间、状态、调用组件关联起来的运行记录。两者能够共享相同的步骤信息，但用途不同。

## ReAct 的 Observation 与 Trace 有什么关系？

**Observation 是 Agent 执行循环的输入**：工具真实返回的结果或错误回到模型上下文，模型据此继续判断。

**Trace 是系统的执行记录**：开发者用它查看模型提出了什么工具调用、应用执行是否成功、花了多久，以及后续是否完成。

例如，知识库工具返回三个相关片段：

1. 这些片段作为 Observation 回到 Agent，影响下一步回答。
2. 检索工具名称、query、状态、耗时、缓存/重排标志可以进入 Trace，帮助排查召回和运行问题。

Observation 关注“Agent 下一步能依据什么”；Trace 关注“系统刚才实际上做了什么”。模型可能只看到经过清理和格式化的工具结果，而开发者 Trace 保留的是受控诊断字段，两者不必完全相同。

## EchoMind 记录的 Trace 内容

EchoMind 在 `agents/agent_orchestrator.py` 中把最近的请求轨迹保存在进程内队列，默认最多保留 200 条；`/trace/tool/{request_id}` 查询单条，`/trace/tools` 查询最近记录。

请求级记录包括：

- `request_id`、时间戳、意图、主 Agent、辅助 Agent、是否升级和总耗时。
- 工具名称和工具列表。
- 每次工具调用的 Agent 类型、工具名、模型给出的 `tool_use_id`、输入参数、调用是否成功、业务结果是否成功、耗时、是否命中缓存、是否重排和错误信息。
- 物流升级候选及其原因。

物流单号在写入工具轨迹前会经过脱敏。工具轨迹能帮助回答：“哪个 Agent 调用了哪个工具？参数有没有过校验？工具慢还是模型慢？知识搜索是否走了重排？错误发生在哪一步？”

这是 EchoMind 的应用级请求 Trace，以 `request_id` 为查询键，当前存放在进程内的有界队列中。

## Enterprise Insight Agent 的 Trace 内容

项目一在 `gpt_researcher/enterprise/trace.py` 中实现了 `ResearchTraceRecorder`。`/api/enterprise/tasks` 创建任务时生成 `task_id`，并把它作为 `run_id`；Trace 同时拥有独立的 `trace_id`。

记录分成两类：

### 1. 带序号的结构化事件

- 研究开始和最终完成状态。
- 证据选择的任务类别、候选与选中数量、来源元数据覆盖情况、权重和相似度阈值。
- 每条被选证据的相似度、来源权威分和最终排序分。
- Claim Gate 的主张风险、证据强度要求、门控决策。
- 生成主张的数量和 ID。
- Grounding Validation 的状态、发现代码、引用数量和是否需要修复。
- Repair 的动作类型和关联证据 ID。

### 2. 运行阶段诊断

`stage("research")`、`stage("report")`、`stage("search")`、`stage("retrieval")` 记录阶段完成/失败、异常类型和耗时。Trace 还会统计搜索调用数、搜索失败数、检索次数、选中证据数和被丢弃的事件数。

完整记录可以导出为 `outputs/traces/{trace_id}.json`，任务 API 会暴露 `trace_id` 和 Trace 文件引用。Trace 设计只保留有界诊断数据和结构化 ID，不把完整提示词、网页证据正文、URL 或异常消息复制进事件。

这对报告业务的好处是：当某个结论引用不可靠或没有证据支持时，可以从报告主张回看证据选择、来源分数、Claim Gate、生成和 Grounding Validation 的步骤。

## 项目里的阶段记录与 OpenTelemetry Span

OpenTelemetry 的 Span 通常包含操作名称、Trace ID、Span ID、父 Span、开始/结束时间、状态、属性和事件；上下游服务通过 context propagation 传递 Trace/Span 上下文，形成跨进程调用链。

Enterprise Insight Agent 的自定义 Trace 已记录 trace_id、run_id、事件序号、时间、状态、评分和阶段耗时。它的结构化事件与阶段诊断可按 Span 的概念理解，但 `stage()` 当前保存的是阶段级运行事件，并没有建立 OpenTelemetry 式的 parent-child Span 树或通过 OTLP exporter 发到追踪后端。EchoMind 则是 request_id 关联的请求级工具轨迹。

这个区分帮助准确描述工程实现：可以说项目有结构化 Trace 和阶段耗时；只有接入 Trace/Span SDK、上下文传播和 exporter 后，才描述为 OpenTelemetry 分布式追踪。

## Trace 为什么有用

- **定位延迟**：把总耗时拆成模型、检索、工具与生成阶段。
- **定位质量问题**：检查检索候选、来源权重、工具 Observation 和最终主张之间的链路。
- **复现失败**：通过 request_id、run_id、版本与输入引用找到对应执行。
- **支持评测分析**：把失败样本与 Trace 关联，区分意图错误、召回漏项、工具异常和生成错误。
- **辅助业务审计**：研究报告中的主张可以回到证据选择与支持检查记录；客服则能定位实际调用的本地物流样本或知识库工具。

记录 Trace 时按诊断目的选择字段。模型输入输出、个人信息、密钥与完整网页内容需要单独的访问控制和保留策略；Trace 的核心价值是保存能定位问题的结构化事实。

## 面试回答

### Session、Trajectory、Trace 分别是什么？

> Session 按用户与会话 ID 组织多轮交互；Trajectory 是 Agent 完成一个任务时按顺序发生的判断、行动、工具观察和最终输出；Trace 是面向运行排障的请求记录，串起组件、状态和耗时。Span 则是 Trace 中一个有起止时间的操作。

### EchoMind 的 Trace 记录了什么？

> EchoMind 用 request_id 查询一次请求的意图、主辅 Agent、工具列表、每个 tool_use 的名称、参数、成功状态、耗时、缓存和重排信息，以及错误或升级信息。它用有界进程内队列保存最近记录，并通过两个 Trace API 查询。

### Enterprise 项目的 Trace 有什么不同？

> 企业研究 Trace 围绕 run_id/trace_id 记录任务策略、证据排序分项、主张门控、生成、grounding 检查、修复与最终状态，并导出 JSON 供任务诊断。它能把报告里的主张审查和前面的证据选择过程联系起来。

### ReAct 的 Observation 是 Trace 吗？

> Observation 是工具结果送回模型的执行输入；Trace 是系统留给开发和运维的诊断记录。工具结果可以被格式化成 Observation，同时把工具名、状态和耗时写入 Trace，但概念和消费方不同。

## 参考资料

- [OpenTelemetry: Traces](https://opentelemetry.io/docs/concepts/signals/traces/)
- [OpenTelemetry: Context propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
