# Knowledge Garden

个人 Agent 开发知识库。知识按主题和概念层级组织，通过项目代码、系统链路和面试追问把概念讲透。

## 在线阅读

文档站使用 MkDocs Material，包含左侧章节导航、页内目录和全文搜索。GitHub Pages 发布地址：

[打开 Knowledge Garden 文档站](https://12hsj12.github.io/Knowledge-Garden/)

首次发布前，需要在仓库 Settings → Pages 将 Build and deployment 的 Source 设为 **GitHub Actions**。之后每次更新 docs/ 下的内容，GitHub Actions 会构建并发布站点。

## 知识结构

- **Agent**：系统主线、ReAct 与工具调用、RAG、MCP、记忆系统、服务接口。
- **项目实践**：Enterprise Insight Agent 与 EchoMind 的架构、代码落点和面试表达。

每个技术主题依次说明概念原理、两个项目中的真实实现、模块如何协作，以及面试追问与回答。笔记是整理后的知识正文，不按日期生成学习日志。

## 项目主线

1. [Enterprise Insight Agent](https://github.com/12hsj12/Enterprise-Insight-Agent)
2. [多 Agent 智能客服（EchoMind）](https://github.com/12hsj12/multi-agent-customer-support-logistics)
3. 第三个项目启动后再加入，不预设技术方案。

## 本地文档源

- [docs/](docs/)：文档站内容。
- [mkdocs.yml](mkdocs.yml)：站点主题与左侧导航树。
- [.github/workflows/deploy-docs.yml](.github/workflows/deploy-docs.yml)：自动构建与发布流程。

本仓库为公开仓库，只记录适合公开分享的内容。