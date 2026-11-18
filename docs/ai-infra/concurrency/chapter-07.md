# 并发基础：第 7 章 分解问题与并发模式

前几章分别介绍了进程和线程、任务间通信，以及操作系统怎样调度任务。本章开始从**问题本身**出发：一个程序有哪些工作可以拆开？这些工作之间有哪些依赖？适合按功能分，还是按数据分？拆成多大的任务，才能让执行资源有足够工作，又不被调度和通信成本拖慢？

本章的顺序可以概括为：**依赖分析 → 任务/数据分解 → 选择并发模式 → 调整粒度 → 评估协调成本**。[本章目录与学习目标](https://livebook.manning.com/book/grokking-concurrency/chapter-7/16)

## 1. 先做依赖分析：找出哪些工作能独立推进

并发设计从工作和依赖关系开始，而不是先决定开多少线程。

可以把程序中的主要工作列成节点，并用箭头表示“后一步需要前一步的结果”：

~~~text
读取文件 A → 解析 A → 统计 A ─┐
读取文件 B → 解析 B → 统计 B ─┼→ 合并统计结果 → 输出报告
读取文件 C → 解析 C → 统计 C ─┘
~~~

从这个图可以看出：

- A、B、C 文件各自的读取、解析和局部统计互不依赖，可以并发推进。
- 对同一个文件，解析要等读取完成，局部统计要等解析完成。
- 全局报告依赖所有文件的局部统计结果，需要在汇总阶段合并。

依赖分析要回答三个问题：

1. **每项工作需要什么输入？**输入由谁产生？
2. **它产生什么结果？**哪个后续工作会使用这个结果？
3. **哪些任务可以同时执行？**哪些必须等前置任务完成？

任务有依赖，不代表整个程序不能并发。只要某些任务已经满足前置条件，它们就可以先运行；调度器之后再安排依赖它们的工作。相反，如果后一项工作必须使用前一项结果，就不能仅靠增加线程消除这条依赖。

## 2. 两类基本分解：按功能分，或按数据分

### 2.1 Task decomposition：按工作职责拆分

**任务分解（task decomposition）**把一个问题拆成不同职责或阶段。例如：

- 一个任务下载数据，另一个任务解析数据，再由第三个任务保存结果；
- 一个任务监听请求，另一个任务处理请求；
- 对每个文件执行同一组步骤，但每个任务负责不同文件。

任务分解关注的是“程序由哪些工作组成”。不同任务可能执行不同函数，也可能串成一条有前后关系的处理链。

### 2.2 Data decomposition：把数据分块处理

**数据分解（data decomposition）**把数据集合切成多个独立部分，然后对每部分执行相同或相似的计算。例如：

- 每个工作单元搜索一部分文件；
- 多个 Worker 分别统计不同文本片段；
- 图像处理时，每个任务负责一个区域；
- CUDA 线程块各自计算输出矩阵中的一个 Tile。

数据分解关注的是“同一种工作如何分配到不同数据块”。若每块可以独立处理，结果最后再合并，就有机会并发执行。

两种分解可以组合：先把工作按阶段分成解析、变换、存储，再把每一阶段的输入数据分成多个块分别处理。

## 3. Pipeline：分阶段处理，让不同数据项重叠推进

**Pipeline（流水线）**将一个任务拆成有先后顺序的多个阶段，并让不同工作项同时处于不同阶段。

用洗衣流程举例：

~~~text
衣物 A：洗涤 → 烘干 → 折叠
衣物 B：       洗涤 → 烘干 → 折叠
衣物 C：              洗涤 → 烘干 → 折叠
~~~

处理单件衣物时，步骤仍然要按顺序完成；流水线的并发来自不同衣物之间的重叠：烘干 A 时可以洗涤 B，折叠 A 时可以烘干 B、洗涤 C。

在程序中，阶段之间通常通过队列或缓冲区传递工作项：

~~~text
输入 → 阶段 1（解析）→ 队列 → 阶段 2（转换）→ 队列 → 阶段 3（写入）
~~~

流水线要留意：

- **阶段耗时是否失衡**：最慢的阶段会限制整体吞吐，其他阶段可能因等待输入或输出而空闲。
- **缓冲区是否合适**：缓冲区可以吸收短暂的速度差，但过度积压会增加内存占用和排队延迟。
- **阶段之间是否共享状态**：若多个 Worker 更新同一资源，需要同步或重新设计数据所有权。
- **吞吐与单项延迟不同**：流水线有机会提高连续处理多个输入的吞吐；单个输入仍要经过所有阶段，端到端延迟不会自动变成某一个阶段的耗时。

流水线适合工作天然有序、每个阶段职责清晰、多个数据项可以同时处于不同阶段的场景。[Manning 对 Pipeline 模式的说明](https://livebook.manning.com/learn/concepts/workflow--pattern/grokking-concurrency)

## 4. Loop-level parallelism 与 Map：把独立迭代分配出去

### 4.1 循环迭代什么时候能并发

循环的迭代之间如果互不依赖，就可以把不同迭代分配给不同执行单元：

~~~cpp
for (int i = 0; i < n; ++i) {
    output[i] = transform(input[i]);
}
~~~

每一轮只读取 input[i]、写入 output[i]，并不需要另一轮的结果，因此这些迭代可以并发处理。这种机会称为**循环级并行（loop-level parallelism）**。

如果迭代之间存在循环携带依赖，就要先处理依赖。例如：

~~~cpp
total = total + input[i];
~~~

这里每轮都读写同一个 total，直接让多线程同时执行会产生竞争。对于求和，可以把数据分块并计算局部和，再把局部和归约；但这改变了合并顺序，浮点结果可能有舍入差异。不是每个有依赖的循环都能简单改写为并行版本。

### 4.2 Map：对每个元素应用同一函数

**Map 模式**把“对每个输入独立应用同一变换”表达成一个操作：

~~~text
输入集合：[x1, x2, x3, ...]
函数：    f(x)
Map 输出：[f(x1), f(x2), f(x3), ...]
~~~

例如，对一组文件分别判断是否含有指定文本，每个文件的搜索结果不依赖其他文件。把文件路径列表交给 Map，每个工作项调用同一个 search 函数，最终产生与输入对应的一组结果。

Python 的 concurrent.futures 提供了 map 接口：

~~~python
from concurrent.futures import ThreadPoolExecutor

def square(value):
    return value * value

values = range(10)

with ThreadPoolExecutor(max_workers=4) as pool:
    results = list(pool.map(square, values))
~~~

这里 Map 描述的是**数据变换模式**；ThreadPoolExecutor 描述的是一种执行方式。算法的独立性来自 square 对每个输入独立计算，不是因为使用了线程池。对于实际应用，还要根据 I/O 或 CPU 工作负载选择线程、进程或其他执行资源；并发执行也不保证结果按完成顺序返回。Python 的 Executor.map 默认按输入顺序产出结果。[Python concurrent.futures 文档](https://docs.python.org/3.14/library/concurrent.futures.html)

## 5. Fork/Join：拆开子任务，完成后汇合

**Fork/Join**先把一个问题拆成若干子任务（fork），让它们并发执行；当子任务完成后，再等待并合并结果（join）。

以大数组求和为例：

~~~text
数组
  ├── 子数组 A → 局部和 A ─┐
  ├── 子数组 B → 局部和 B ─┼→ 等待完成并合并 → 总和
  └── 子数组 C → 局部和 C ─┘
~~~

伪代码的结构如下：

~~~python
futures = [
    executor.submit(sum_chunk, chunk_a),
    executor.submit(sum_chunk, chunk_b),
    executor.submit(sum_chunk, chunk_c),
]

partial_sums = [future.result() for future in futures]
total = sum(partial_sums)
~~~

调用 submit 就是 fork；读取各个 Future 的结果并汇总就是 join。实际设计中，join 可能等待最慢的子任务，因此要关注工作量是否均匀。Fork/Join 可用于递归拆分、分治算法或多块数据处理。

## 6. MapReduce：Map 产生中间结果，Reduce 聚合结果

**MapReduce**把数据处理组织为 Map 与 Reduce 两个主要阶段，适合许多数据分区可以独立处理、随后按键汇总的任务。

用词频统计说明：

1. **Map**：每个文本分区单独处理，输出词和局部计数，例如 `("cuda", 1)`。
2. **按键分组（group/shuffle）**：把相同词的计数放到一起，例如 `("cuda", [1, 1, 1])`。
3. **Reduce**：将同一个词对应的局部计数相加，输出 `("cuda", 3)`。

~~~text
分区 A → Map → ("cuda", 1), ("kernel", 1) ─┐
分区 B → Map → ("cuda", 1), ("memory", 1) ─┼→ 按词分组 → Reduce → 全局词频
分区 C → Map → ("cuda", 1), ("kernel", 1) ─┘
~~~

Map 阶段的不同分区通常可以独立并发；Reduce 阶段按键合并相应中间结果。MapReduce 是一种组织计算的模式，不等于某个特定框架，也不要求一定跨机器运行。跨机器时，框架还要负责数据分区、传输、失败重试和任务调度等工程问题。[Manning 对 MapReduce 模式的说明](https://livebook.manning.com/learn/concepts/mapreduce--pattern/grokking-concurrency)

MapReduce 与 Fork/Join 都会拆分工作并汇总结果；MapReduce 更明确地规定了“按输入分区产生键值中间结果，再按键聚合”的数据流程，Fork/Join 则是更通用的分支执行与汇合结构。

## 7. Granularity：每个任务分多大

**粒度（granularity）**描述任务被拆分的细或粗，可从任务的数量与每个任务包含的工作量理解。

| 粒度 | 任务特点 | 好处 | 代价 |
| --- | --- | --- | --- |
| 粗粒度 | 任务数量少、每个任务工作多 | 调度、通信和合并成本低 | 任务大小不均时容易出现负载不平衡；可用并发任务也可能不足 |
| 细粒度 | 任务数量多、每个任务工作少 | 调度器选择多，工作更容易分配给空闲资源 | 创建、排队、通信和汇总成本增加 |

目标是暴露足够多的独立工作，让可用资源有事可做，同时不让任务管理成本超过实际计算收益。不存在适用于所有问题的固定任务大小：

- 文件大小差异较大时，可以把文件拆成多个块，或使用动态分配减轻长尾等待。
- 每个循环迭代工作极少时，可以把一批迭代合并成一个任务，减少提交开销。
- 任务需要频繁交换大型中间数据时，过细分解会增加通信。
- 有些任务耗时变化很大时，增加可调度任务数可为负载均衡提供更多选择。

把多个小任务合并为较大的工作单元，减少任务创建、通信或协调成本，称为**任务合并（agglomeration）**。合并也有边界：如果把任务合得过粗，处理器可能没有足够工作，或者某个大任务拖慢整体完成时间。要根据任务耗时、数据传输和负载差异调整后测量。[Manning 对粒度的讲解](https://livebook.manning.com/learn/concepts/granularity--domain-concept/grokking-concurrency)

## 8. 并发模式对照：同一问题可以有多种切法

| 模式 | 主要切分方式 | 结果怎样继续流动 | 典型例子 |
| --- | --- | --- | --- |
| Pipeline | 按连续处理阶段分工 | 每个阶段把结果交给下一阶段 | 读取 → 解析 → 转换 → 写入 |
| Map | 按数据元素或分区分工 | 每个输入独立产生一个对应输出 | 对一组文件并发做内容检查 |
| Fork/Join | 一个问题拆成子任务 | 等子任务完成，再合并结果 | 数组分块求和 |
| MapReduce | 按输入分区并进行键值聚合 | Map 产生中间键值对，Reduce 按键合并 | 多文件词频统计 |

它们不是只能单独使用的互斥方案：一个数据处理服务可以让 Map 负责分区内的处理，再把结果放入 Pipeline 的下一阶段；MapReduce 的 Map 阶段也可以并发执行多个任务，最终再 Join 或 Reduce。

## 9. 联系前面章节与 CUDA 矩阵乘法

### 从任务到执行资源

- 第 4 章回答：进程和线程等执行单元是什么。
- 第 5 章回答：执行单元如何传递任务和结果。
- 第 6 章回答：操作系统如何安排多个任务推进。
- 第 7 章回答：问题本身如何拆成适合并发的任务。

并发设计先找到独立工作、表达依赖，再选择执行模型。先选线程数、进程数，却没有检查任务是否可并发，通常解决不了根本问题。

### CUDA 矩阵乘法是多种分解方式的组合

在已有的 [CUDA 线程协作与矩阵乘法笔记](../gpu/cuda-thread-cooperation-and-patterns.md)中：

- 输出矩阵 C 被分成多个 Tile，Tile 之间可以独立计算，这是按数据分解。
- 一个 Thread Block 负责一个输出 Tile，Block 内线程负责不同输出元素。
- 每个输出元素沿 K 维循环执行乘加，循环的输入 Tile 通过 Shared Memory 复用。
- 所有输出 Tile 完成后，结果已经处在统一的 C 矩阵中；这类计算不一定需要像词频统计那样按键聚合。

因此，一个真实算法可以同时包含数据分解、循环级并行和 Block 内合作；具体模式名称是理解结构的工具，关键是能说清楚任务、数据、依赖和汇总关系。

## 10. 常见设计问题与检查顺序

1. **只看代码行数分任务**：应先找到有意义的独立工作，而不是把每一行都切成任务。
2. **忽略输出依赖**：多个任务同时写共享状态会产生竞态或锁竞争；可优先让任务产生私有结果，最后再汇总。
3. **把流水线理解为单个工作项更快**：流水线主要让多个工作项重叠推进，常见收益是吞吐和资源利用。
4. **把 Map 误解成“用了 map API 就自动并发”**：Map 表达独立映射，具体并发由执行器、运行时或框架决定。
5. **忘记 Reduce 的合并成本**：数据分区结束后还需汇总；传输和聚合也可能成为瓶颈。
6. **任务拆得过粗**：并行任务数不足或任务耗时差异大时，部分处理资源会空闲。
7. **任务拆得过细**：排队、调度、同步和传输成本会变得显著。
8. **把不同模式当成互斥标签**：Pipeline、Map、Fork/Join 和 MapReduce可以组合，用来说明不同层面的结构。
9. **不测量就断言加速**：分解能暴露并行机会，但总体性能还取决于可用资源、数据移动、依赖和调度成本。

实际设计可以按这个顺序检查：

~~~text
描述目标
  → 画出任务及其依赖
  → 标记可并行任务
  → 选择按功能、阶段还是数据切分
  → 选一个清晰的并发模式
  → 设定任务大小并估计通信/汇总成本
  → 在目标运行环境测量与调整
~~~

## 11. 本章记忆卡

- 依赖分析先于并发实现：找出任务输入、输出、前置条件以及可同时推进的部分。
- 任务分解按职责或阶段拆工作；数据分解把数据切成独立分区并执行相同操作。
- Pipeline 按阶段传递数据，让多个工作项在不同阶段重叠处理。
- Map 对各元素独立应用同一函数；循环迭代只有在依赖关系允许时才能并行。
- Fork/Join 把一个问题拆成多个子任务，等待它们完成后合并结果。
- MapReduce 通过 Map 生成中间键值对，再按键分组并 Reduce 聚合。
- 粒度平衡并行机会与任务管理成本；agglomeration 合并过小任务，也要避免任务合并过粗。
- 一个算法可以组合多种分解方式与并发模式；要讲清任务如何切分、依赖如何满足、结果如何汇总。

### 面试表达：怎样把一个顺序程序改造成并发程序？

> 我会先做依赖分析，列出每个步骤的输入、输出和前置条件，确认哪些任务可以独立执行。接着选择分解方式：如果工作由不同处理阶段组成，可以用 Pipeline；如果同一函数可以独立处理许多元素，可以用数据分解和 Map；如果一个大问题可以拆成子任务再合并，可以用 Fork/Join；如果每个分区产生键值中间结果并需要按键聚合，可以用 MapReduce。之后再选择任务粒度：任务太少会限制并行度，太小又会增加调度和通信开销。最后结合负载均衡、结果汇总和实际测量确定实现。

### 复习时可以自问

1. 依赖图中的节点和边分别表示什么？哪些任务已经满足运行条件？
2. 任务分解与数据分解分别适合什么类型的问题？
3. Pipeline 为什么能提高多项工作连续处理时的吞吐？它对单项延迟有什么影响？
4. 循环迭代何时能并行？累加循环为什么需要先算局部结果再归约？
5. Map 与 MapReduce 的关系是什么？Reduce 阶段解决了什么问题？
6. Fork/Join 的 Join 为什么可能受最慢的子任务拖累？
7. 粒度太细和太粗分别有什么运行代价？Agglomeration 解决什么问题？
8. CUDA 矩阵乘法中的输出 Tile、Thread Block 和 K 维循环分别对应哪些分解层次？

## 参考资料

- Kirill Bobrov, [Grokking Concurrency, Chapter 7: Decomposition](https://livebook.manning.com/book/grokking-concurrency/chapter-7/16)
- Manning, [What you will learn in Grokking Concurrency](https://livebook.manning.com/learn/grokking-concurrency)
- Manning, [Decomposition in Grokking Concurrency](https://livebook.manning.com/learn/concepts/task-decomposition--technique/grokking-concurrency)
- Manning, [Pipeline in Grokking Concurrency](https://livebook.manning.com/learn/concepts/workflow--pattern/grokking-concurrency)
- Manning, [MapReduce in Grokking Concurrency](https://livebook.manning.com/learn/concepts/mapreduce--pattern/grokking-concurrency)
- Manning, [Granularity in Grokking Concurrency](https://livebook.manning.com/learn/concepts/granularity--domain-concept/grokking-concurrency)
- Python 3.14 documentation, [concurrent.futures — Launching parallel tasks](https://docs.python.org/3.14/library/concurrent.futures.html)


---

**继续阅读：** [第 8 章：竞态条件与同步](chapter-08.md)。