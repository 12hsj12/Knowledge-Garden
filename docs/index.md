# Knowledge Garden

这是我的 Agent 开发知识库。内容以稳定的知识章节组织，围绕概念、代码实现、系统协作和面试表达逐步展开。

## 学习入口

- [Agent 系统主线](agent/index.md)：先理解 Agent 应用从请求进入到工具执行、回答和记忆写回的整体链路。
- [ReAct 与工具调用](agent/react.md)：理解模型如何在判断、调用工具和观察结果之间循环。
- [RAG 与语义检索](agent/rag.md)：从文档切分、Embedding、向量召回到重排和生成。
- [MCP 工具协议](agent/mcp.md)：理解 Host、Client、Server，以及两个项目里的工具接入方式。
- [记忆系统：Redis 与 ChromaDB](agent/memory.md)：区分短期会话状态、情景记忆、用户画像和知识库。
- [服务接口：FastAPI、Pydantic 与 REST](agent/web-api.md)：理解服务框架、数据边界模型和接口风格。
- [项目实践](projects/enterprise-insight-agent.md)：从项目请求链路查看这些概念如何落地。

## 阅读方式

左侧导航按“Agent → 技术主题”展开；每个主题依次说明核心概念、项目实现、与其他模块的协作关系，以及面试追问和回答。项目章节用于把分散在各主题中的实现还原成完整系统。

笔记以经过整理的知识正文呈现。学习过程中的临时记录不会作为按日期排列的正文页。