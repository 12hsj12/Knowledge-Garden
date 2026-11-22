# AI Infra 学习主线

AI Infra 关注为 AI 工作负载提供和管理运行能力：计算资源如何组织，模型如何部署和服务，任务如何调度，数据如何流动，以及系统如何在负载增加时保持性能与稳定性。

它和 Agent 开发是不同的学习方向：

- **Agent** 主要关注应用层：模型如何理解任务、选择工具、检索知识并组织结果。
- **AI Infra** 主要关注运行基础设施：计算、存储、网络、模型服务、并发调度、资源效率与系统扩展。

两个方向会在“模型服务”这样的接口处相遇，但问题视角不同。本板块按 AI Infra 的基础概念单独整理，不把它并入 Agent 章节。

## 当前学习内容

### 并发与计算机体系结构基础

并发是理解服务如何同时应对多个请求、使用 CPU/GPU 资源、控制延迟和提高吞吐的基础。计算机体系结构则帮助理解指令、缓存和不同处理器如何影响程序表现。

- [并发基础：第 1 章 引入并发](concurrency/chapter-01.md)：并发为何重要，性能如何衡量，系统如何扩展，并发可以出现在哪些层次。
- [并发基础：第 2 章 串行与并行执行](concurrency/chapter-02.md)：串行、顺序、并行、任务独立性、硬件支持、Amdahl 定律、Gustafson 定律。
- [计算机如何工作：第 3 章](concurrency/chapter-03.md)：处理器、缓存、CPU 执行周期、运行时系统、并行硬件层次、SMP、Flynn 分类与 CPU/GPU 分工。
- [并发程序的基本构件：第 4 章](concurrency/chapter-04.md)：并发程序设计步骤、进程和线程的内部结构与状态、多进程通信、线程调度，以及 Python 服务中的进程、线程和 asyncio。
- [进程间通信：第 5 章](concurrency/chapter-05.md)：共享内存、消息传递、管道与队列、线程池，以及并发任务的拆分、执行和结果汇总。
- [多任务：第 6 章](concurrency/chapter-06.md)：CPU 与 I/O 瓶颈、单核时间共享、抢占式调度、上下文切换、任务隔离与 Python 执行模型。
- [分解问题与并发模式：第 7 章](concurrency/chapter-07.md)：依赖分析、任务/数据分解、Pipeline、Map、Fork/Join、MapReduce 与粒度取舍。
- [竞态条件与同步：第 8 章](concurrency/chapter-08.md)：共享状态、丢失更新、临界区、Lock、Semaphore、Condition 与原子操作。
- [死锁、活锁与饥饿：第 9 章](concurrency/chapter-09.md)：哲学家就餐、死锁预防、进度公平性、生产者—消费者与读者—写者。
- [非阻塞 I/O：第 10 章](concurrency/chapter-10.md)：客户端—服务器、Socket、C10K 问题、阻塞与非阻塞 I/O。
- [事件驱动并发：第 11 章](concurrency/chapter-11.md)：I/O 多路复用、事件循环、回调、Reactor 与 I/O 模型。
- [异步通信：第 12 章](concurrency/chapter-12.md)：协作式多任务、协程、Future、Task 与 asyncio 执行方式。
- [编写并发应用：第 13 章](concurrency/chapter-13.md)：Foster 方法、矩阵乘法、分布式词频统计与全书主线。

### GPU 基础

从 GPU 的执行层级、存储层级和性能约束入门，理解计算任务怎样映射到硬件，数据搬运怎样影响性能。

- [CUDA C/C++：编译、Kernel 启动与内存管理](gpu/cuda-c-cpp-memory-kernel.md)：从 C/C++ 指针与编译工具链，串起 Kernel 启动、线程索引、Host/Device 数据流和两种内存管理方式。
- [CUDA 线程协作与计算模式](gpu/cuda-thread-cooperation-and-patterns.md)：Shared Memory、Block 同步、Warp Shuffle、并行 Reduction、Tiling 与矩阵乘法 Kernel。
- [GPU 执行模型、内存层级与性能分析](gpu/cuda-execution-memory-performance.md)：CUDA 的 Grid/Block/Thread/Warp/SM、寄存器与显存、合并访存、运算强度和 Occupancy。

## 建立 Infra 视角

阅读新概念时，可以依次问：

1. 系统里有哪些工作单元？它们能不能独立推进？
2. 当前瓶颈在计算、内存、存储、网络，还是等待外部服务？
3. 关注的是单个请求更快（延迟），还是单位时间处理更多工作（吞吐）？
4. 可以通过增加单机资源扩展，还是需要增加机器并分摊负载？
5. 增加并发资源后，协调、通信、排队和数据合并会增加多少成本？

这些问题把概念连接到 AI 推理服务、数据处理和分布式系统的运行方式。后续笔记会随着实际学习逐步扩展。
