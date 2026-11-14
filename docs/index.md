# Knowledge Garden

这是我的 AI 技术知识库，分为 **Agent 应用开发** 与 **AI Infra 基础** 两个独立学习方向。笔记按主题组织，逐步讲清概念、运行过程、系统关系和实际使用场景。

## 学习入口

### Agent 应用开发

- [Agent 系统主线](agent/index.md)：理解 Agent 应用从请求进入到工具执行、回答、记忆和观测的整体链路。
- [ReAct 与工具调用](agent/react.md)：模型如何在判断、调用工具和观察结果之间循环。
- [RAG 与语义检索](agent/rag.md)：从文档切分、Embedding、向量召回到重排和生成。
- [MCP 工具协议](agent/mcp.md)：理解 Host、Client、Server，以及项目里的工具接入方式。
- [记忆系统：Redis 与 ChromaDB](agent/memory.md)：区分短期会话状态、情景记忆、用户画像和知识库。
- [项目实践](projects/enterprise-insight-agent.md)：沿项目请求链路查看概念如何落地。

### AI Infra

- [AI Infra 学习主线](ai-infra/index.md)：从计算资源、任务执行和系统性能等基础概念入门。
- [并发基础：第 1 章 引入并发](ai-infra/concurrency/chapter-01.md)：并发的价值、延迟与吞吐、系统扩展和并发层次。
- [并发基础：第 2 章 串行与并行执行](ai-infra/concurrency/chapter-02.md)：任务依赖、并行条件、Amdahl 与 Gustafson 定律。
- [计算机如何工作：第 3 章](ai-infra/concurrency/chapter-03.md)：处理器、缓存、CPU 执行周期、运行时系统、并行硬件、SMP、Flynn 分类和 CPU/GPU 分工。
- [并发程序的基本构件：第 4 章](ai-infra/concurrency/chapter-04.md)：并发设计步骤、进程与线程的状态和实现，以及 Python 服务中的调度选择。
- [进程间通信：第 5 章](ai-infra/concurrency/chapter-05.md)：共享内存、消息传递、线程池与任务结果汇总。
- [CUDA C/C++：编译、Kernel 启动与内存管理](ai-infra/gpu/cuda-c-cpp-memory-kernel.md)：C/C++ 基础、Kernel 启动与 CPU/GPU 数据流。
- [GPU 执行模型、内存层级与性能分析](ai-infra/gpu/cuda-execution-memory-performance.md)：从 CUDA 执行层级、访存到 Occupancy 理解 GPU 性能。

## 阅读方式

左侧导航将 Agent 与 AI Infra 分开。Agent 笔记以应用系统链路和项目实现为主；AI Infra 笔记从基础概念出发，连接计算资源、调度与性能。学习过程中的临时记录不会作为按日期排列的正文页。
