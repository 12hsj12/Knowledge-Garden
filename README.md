# Knowledge Garden

个人 AI 技术知识库，分别整理 Agent 应用开发与 AI Infra 基础。内容按主题和概念层级组织，通过系统链路、代码实现和实际场景把知识讲清楚。

## 在线阅读

文档站使用 MkDocs Material，包含左侧章节导航、页内目录和全文搜索。GitHub Pages 发布地址：

[打开 Knowledge Garden 文档站](https://12hsj12.github.io/Knowledge-Garden/)

每次更新 docs/ 下的内容，GitHub Actions 会构建并发布站点。

## 知识结构

- **Agent**：系统主线、ReAct 与工具调用、RAG、MCP、记忆系统、服务接口、评测与可观测性。
- **AI Infra**：独立整理 GPU/CUDA 执行与性能、并发基础，以及后续学习中接触的基础设施主题。
- **项目实践**：Enterprise Insight Agent 与 EchoMind 的架构、代码落点和面试表达。
- **学习路线规划**：暂存个人规划材料，作为记录参考。

Agent 主题围绕概念、项目实现和系统协作展开；AI Infra 主题围绕计算、调度、性能和系统扩展等基础问题逐步建立。

## 项目主线

1. [Enterprise Insight Agent](https://github.com/12hsj12/Enterprise-Insight-Agent)
2. [多 Agent 智能客服（EchoMind）](https://github.com/12hsj12/multi-agent-customer-support-logistics)
3. 第三个项目启动后再加入，不预设技术方案。

## 本地文档源

- [docs/](docs/)：文档站内容。
- [mkdocs.yml](mkdocs.yml)：站点主题与左侧导航树。
- [.github/workflows/deploy-docs.yml](.github/workflows/deploy-docs.yml)：自动构建与发布流程。

本仓库为公开仓库，只记录适合公开分享的内容。
