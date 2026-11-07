# LangChain 与 LangGraph：组件集成和工作流编排

## 一句话区分

- **LangChain** 提供模型、工具、提示模板、检索器等集成与 Agent 抽象，也提供 `create_agent` 这类较高层的 Agent harness。
- **LangGraph** 提供显式的有状态流程编排运行时：把状态、节点和边组成图，管理分支、循环、暂停、恢复和持久化。
- **LangChain Agent 可以建立在 LangGraph 的运行机制上；LangGraph 也可以独立使用。**

“LangChain 是工具箱，LangGraph 是把工具箱装成框架”可以帮助记忆，但工程上更准确的说法是：LangChain 偏向组件和 Agent harness，LangGraph 偏向控制流与状态运行时；二者可以组合，也可以按需求分别使用。

## LangChain 解决什么问题

Agent 通常需要接模型、提示词、工具、检索器、文档处理、输出解析和回调。LangChain 把许多通用接入封装成可组合的 Python 对象，减少每种模型和工具都写一套适配代码。

当前 LangChain Agent API 的思路是把模型、工具、提示和中间件组合成一个可配置的执行器。它适合：

- 快速接入不同模型供应商。
- 将函数、API 或查询能力封装成 Agent 工具。
- 组合 Retriever、Document、Prompt、Output Parser 等 RAG/Agent 组件。
- 从较高层的 Agent 循环开始，再按需要扩展行为。

LangChain 并不替业务方定义订单规则、权限、数据库 schema 或评测标准；这些仍然属于应用设计。

## LangGraph 解决什么问题

当工作流出现分支、循环、状态要跨步骤保存、故障后恢复、需要人工审核或长任务执行时，把流程显式建模为图更容易检查和控制。

基本构件：

- **State**：节点共享的任务状态，例如用户目标、检索结果、证据、报告草稿与审核状态。
- **Node**：读取一部分 state，执行模型调用、检索、规则校验或业务函数，并返回状态更新。
- **Edge**：定义节点后的下一步；可以是固定边，也可以依据当前状态做条件跳转。
- **Checkpoint / persistence**：保存图的进度和状态，支持后续恢复或继续。
- **Interrupt / human-in-the-loop**：在指定节点暂停，等待人工输入后继续。
- **Streaming**：执行时逐步输出消息、状态更新或事件。

例如一个研究 Agent 可以将 `classify → search → rank evidence → draft report → claim check` 作为节点；如果覆盖不足，条件边返回 search；如果关键主张需要复核，流程暂停并等待人工检查。每个节点可单独测量和检查状态。

## 两者如何组合

可以这样理解一次组合执行：

1. LangChain 的模型集成连接模型服务。
2. LangChain 的 Tool 抽象描述 Agent 可用的函数或服务。
3. LangGraph 的节点调用模型、工具、检索器或业务函数。
4. LangGraph 的边决定下一个执行步骤。
5. LangGraph 的 checkpoint 保存状态，Trace/观测工具记录节点运行。

LangGraph 不等于“把所有 LangChain 工具打包起来”。它管理的是执行状态与控制流；具体模型、工具或向量库可以来自 LangChain，也可以是自定义代码或其他库。

MCP 与它们处在不同层次：MCP 规定工具服务之间如何发现、描述和调用工具；LangChain/LangGraph 可以使用 MCP 适配器把 MCP 工具接入应用，但使用 LangChain 并不会自动启用 MCP。

## 如何选择

| 场景 | 合适的起点 | 理由 |
| --- | --- | --- |
| 单次模型请求、少量固定工具、简单的有限循环 | 直接用模型 SDK 和自定义代码 | 控制路径直观，依赖少，边界容易讲清楚 |
| 需要多个模型供应商、Retriever、Tools 和 Agent 抽象 | LangChain | 复用集成，快速组合常见组件 |
| 多分支、多轮循环、状态持久化、暂停审核、恢复执行 | LangGraph | 显式表达状态与控制流 |
| 已有自建系统，核心流程稳定 | 保留自建编排，按具体接入需求引入组件 | 避免为框架迁移而迁移 |

## 两个项目实际使用了什么

### EchoMind

EchoMind 的 Agent 主链使用自建 `AgentOrchestrator` 和 Anthropic SDK 的 `tool_use`。项目显式执行工具 schema 下发、白名单和参数校验、handler 调用、`tool_result` 回填，并限制单 Agent 最多 3 轮。Agent 主链没有依赖 LangChain 或 LangGraph。

这条实现路径适合在面试中讲清楚 Agent 控制权：模型提出结构化工具调用，应用代码决定授权范围并执行。

### Enterprise Insight Agent

项目主链由自建 `IntelligenceWorkflow` 调用 GPT Researcher，负责研究任务、网页检索、证据整理、报告生成和主张审查。当前核心企业工作流不是 LangGraph 图。

仓库在若干组件中使用 LangChain 生态类型和工具，例如 `langchain_core.documents.Document`、文本切分器和可选向量存储路径。描述时把“使用了部分 LangChain 组件”与“用 LangChain/LangGraph 编排主 Agent”分开。

## 面试回答

### LangChain 和 LangGraph 的关系是什么？

> LangChain 给我模型、工具、检索器等集成和较高层 Agent 抽象；LangGraph 把多步流程建模成共享状态上的节点和边，适合需要分支、循环、持久化、暂停和恢复的 Agent。LangChain Agent 可以使用 LangGraph 运行机制，LangGraph 也可以独立接自定义模型和工具。它们是可组合的两个层次。

### 为什么 EchoMind 没用 LangChain 或 LangGraph？

> EchoMind 的核心目标是让我看清工具调用闭环和执行控制。自建 Orchestrator 直接调用 Anthropic tool_use，明确处理 schema、权限、参数、工具返回和最大轮数。当前控制流规模适合自建；当流程需要复杂 checkpoint、人工中断和持久化恢复时，LangGraph 会成为合适的编排选项。

### 如果以后把 Enterprise Agent 迁到 LangGraph，先映射什么？

> 先定义共享状态，例如任务目标、任务类别、候选证据、已选证据、报告草稿、Claim Gate 结果和最终报告；再将分类、研究、来源排序、报告生成、主张检查映射为节点；用条件边表达证据不足回检索、主张待审核暂停和完成路径。最后为状态版本、checkpoint、Trace 和节点级评测建立契约。

## 参考资料

- [LangChain Python overview](https://docs.langchain.com/oss/python/langchain/overview)
- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview)
- [LangChain learning resources](https://docs.langchain.com/oss/python/learn)
