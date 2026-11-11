# GPU 执行模型、存储层级与性能分析

这页从 CUDA 编程模型出发，依次看 GPU 如何安排线程、数据经过哪些存储层、一个 kernel 为什么受算力或带宽限制，以及 Occupancy 如何影响延迟隐藏。工厂比喻用于建立直觉，正式概念仍以 CUDA 的执行与资源模型为准。

## 1. CUDA 执行层级：Kernel、Grid、Block、Warp 与 SM

GPU kernel 是在 GPU 上运行的函数。CPU 端启动 kernel 时，会给出执行配置：Grid 有多少个 Block，每个 Block 有多少个 Thread。GPU 调度器再把这些 Block 分配给各个 SM 执行。

| CUDA 概念 | 准确含义 | 工厂比喻 |
| --- | --- | --- |
| GPU | 执行 kernel 的设备，由多个 SM 和设备内存等组成 | 整座工厂 |
| SM（Streaming Multiprocessor） | GPU 上调度并执行线程的处理单元，包含执行单元、Warp 调度器、寄存器和片上数据缓存等 | 一个大车间，里面有工位、工具和调度员 |
| Grid | 一次 kernel 启动所包含的所有 Thread Block | CPU 发起的一张总派工单 |
| Thread Block | 一组协作线程；CUDA 编程模型把一个 Block 的线程放在同一个 SM 上执行 | 派进某个车间的一组工作队 |
| Thread | 执行 kernel 的基本逻辑线程，通常处理一个或多个数据元素 | 一名工人 |
| Warp | CUDA 中由 32 个 Thread 组成的执行分组，按 SIMT 方式推进指令 | 32 名工人组成的小分队 |

一个 Block 的线程由同一个 SM 承载，这使它们可以使用该 SM 上的 Shared Memory 并进行 Block 内同步。**一个 SM 可以同时容纳多个 Block**，具体能容纳多少取决于线程数、寄存器和 Shared Memory 等资源；Grid 中的 Block 也不一定同时全部运行。

Block 的执行顺序以及被分配到哪个 SM 由调度器决定，程序不能依赖某个固定顺序。不同 Block 之间若要交换结果，通常需要通过全局内存、后续 kernel 或专门的协作机制完成。

在工厂比喻里，SM 是车间，Block 是派进车间的工作队；一个工作队不会被拆到多个车间共同执行，而一个车间在资源够用时可以接纳多个工作队。GPU 的 SM 并非完全独立的小电脑，它们还会共享更高层的设备资源，例如 L2 Cache 和显存。

### Warp 与 32 个线程

CUDA 编程模型中的 Warp 含 32 个线程。Warp 执行指令时采用 SIMT 思路：一组线程推进同一段 kernel 指令，但每个线程通常处理不同的数据索引。

例如，一条向量加法指令中，Warp 的不同线程可以分别计算数组中相邻的元素。若线程在条件分支中走不同路径，Warp 可能需要分别处理不同分支，从而降低有效执行效率，这称为 Warp divergence（分支发散）。

## 2. 线程索引与边界检查

Grid 和 Block 定义了要启动的线程数，但业务数据的元素数不一定刚好等于线程数。线程通常根据自己的索引计算负责的数据位置：

~~~cpp
int idx = blockIdx.x * blockDim.x + threadIdx.x;

if (idx < n) {
    C[idx] = A[idx] + B[idx];
}
~~~

假设数据有 1000 个元素，每个 Block 启动 256 个线程，需要启动 4 个 Block，共 1024 个线程位置。最后一个 Block 中只有 232 个线程对应有效索引，剩下 24 个线程的 idx 大于等于 1000。条件判断让这些线程不读写数组边界之外的地址。

这就是比喻里的“最后一队有些工人没有货要搬”：他们应跳过本次数据操作，不能访问相邻数组之外的内容。普通 kernel 中可以用 if 做掩码式边界检查；continue 只适用于循环内部的当前迭代，并不是通用的线程退出写法。

还要留意同步点：如果一个 Block 使用 __syncthreads()，不能让一部分线程提前 return 或跳过这个同步点，而其他线程继续等待它们。参与同一 Block 屏障的线程必须按照正确的控制流到达同步位置。

## 3. GPU 存储层级：数据离计算单元有多近

GPU 上的数据存储有不同的容量、可见范围和访问成本。粗略原则是：离 SM 越近的存储通常越快、容量越小；离计算单元越远的存储通常容量越大，但访问延迟更高。实际速度和容量依 GPU 架构及访问方式而变。

| 存储 | 可见范围与用途 | 工厂比喻 |
| --- | --- | --- |
| Register（寄存器） | Thread 私有，由编译器分配，保存当前计算频繁使用的值 | 每位工人随身的小工具包 |
| Shared Memory（共享内存） | 同一 Block 的线程共享，可显式暂存和交换数据，位于 SM 附近 | 一个车间内大家共用的黑板或工作台 |
| L1 Cache | 靠近 SM 的硬件缓存，自动缓存一部分访问 | 车间手边的小本子 |
| L2 Cache | GPU 上多个 SM 可以使用的更高层缓存 | 几个车间附近共用的资料柜 |
| Global Memory（全局内存） | GPU 的设备内存，容量大，Grid 中的线程可访问 | 整座工厂的显存大仓库 |

### Registers：快，但每个 SM 的总量有限

寄存器通常由编译器为每个线程分配，用来保存该线程当前用到的变量。一个线程需要的寄存器越多，同一 SM 可同时驻留的线程或 Block 可能越少。

如果寄存器需求超过可用范围，编译器可能把部分值 spill 到 local memory。名称里的 “local” 指线程私有的逻辑地址空间，不代表它一定和寄存器一样位于片上；spill 访问可能落到设备内存，性能代价会明显增加。

工厂比喻中，寄存器是每位工人随身的工具包；工具带得多，有利于减少往返取物，但车间能同时安排的工人总数会受工具库存约束。

### Shared Memory：Block 内协作的快速暂存区

Shared Memory 位于 SM 附近，同一个 Block 的线程可以读写同一块共享空间。它常用于让线程共享中间结果，或把会重复使用的数据先搬到较近的位置，减少重复访问 Global Memory。

它的容量比 Global Memory 小得多，而且每个 SM 和每个 Block 可用的容量随 GPU 架构变化；“大约 100 KB”只能描述某些设备或配置，不能当成所有 GPU 的固定值。部分架构还会让 Shared Memory 与 L1 Cache 共用片上存储资源，调整其中一种资源会影响另一种可用容量。

如果一个 Block 分配了很多 Shared Memory，同一 SM 可同时驻留的 Block 数可能下降；若访问造成 bank conflict（多个线程竞争同一存储 Bank），Shared Memory 本身也会成为性能瓶颈。它的价值来自数据复用和线程协作，使用前后应测量性能。

### Global Memory 与 Cache

Global Memory 是显卡上的设备内存，例如 GDDR 或 HBM。它负责容纳大量输入、输出和模型数据，容量与带宽都很重要；相较寄存器和片上 Shared Memory，访问延迟更高。

L1 和 L2 Cache 是硬件自动管理的缓存层级。若请求的数据在 Cache 中命中，访问会由相应 Cache 提供，避免这次访问继续到更远的存储层；若未命中，就要从下一层取得数据。Cache 能利用访问局部性，但不会让不规则访问自动变成高效访问。CUDA 不同架构的 Cache 组织、策略和容量各有差异。

工厂比喻里，Global Memory 是大仓库，L1/L2 是近处的资料柜。资料柜里正好有需要的货时，就不用跑到大仓库；如果没有，就仍要继续向远处取货。

## 4. 合并访存：让一个 Warp 少跑几趟仓库

当一个 Warp 的多个线程访问 Global Memory 时，硬件会把这些请求组合成尽可能少的内存事务，这称为 Coalescing（合并访存）。

对齐的连续数组访问是常见的高效模式：Warp 中相邻线程读取相邻元素，硬件可以用较少的事务取回这些相邻数据。若每个线程访问相距很远的位置，同一条指令可能需要更多内存事务，传输的数据中也会有更多未被使用的部分。

例如，在适用的 NVIDIA GPU 上，32 个线程各读取一个连续的 4-byte float，总有效数据是 128 bytes；按 32-byte 事务计算，需要 4 个事务。若访问地址分散到很多不同的 32-byte 区间，事务数就可能增加。具体合并规则取决于设备架构、数据大小和地址分布。

工厂比喻是让连号工人拿连号货物，管理员可以把相邻货物一起运回；若每个工人都要去不同角落取货，运输次数就会增加。设计数组布局和线程索引时，经常会让相邻线程处理相邻元素。

## 5. GPU 为什么擅长大规模计算：吞吐与延迟隐藏

GPU 的优势不只是“有很多核心”，还在于它围绕大规模并行工作设计了执行方式：把大量线程分给多个 SM；一个 Warp 等待数据或依赖结果时，调度器可以安排其他就绪 Warp 继续执行。GPU 不会让那次慢访存本身变快，而是尽量让其他工作填满等待空档。

### Clock Cycle：时钟周期只是时间刻度

时钟频率表示处理器每秒经历多少个时钟周期。假设某个处理单元以 2 GHz 工作：

~~~text
每秒 2,000,000,000 个周期
一个周期时长 = 1 / 2 GHz = 0.5 ns
~~~

时钟周期不是“每周期完成一条指令”的保证。指令的延迟可能跨越多个周期；流水线也可能在一条指令尚未完全结束时继续接收其他指令。因此要区分：

- **Latency（延迟）**：一条依赖链或一次数据请求从发出到结果可用要等多久。
- **Throughput（吞吐）**：处理单元在一段时间里能完成多少独立工作。

### 不同存储的延迟数量级

不同 GPU、缓存命中情况、访问模式和时钟频率会改变真实延迟。下表只帮助建立数量级，不是任意 GPU 的固定规格：

| 存储位置 | 参考数量级 | 直观理解 |
| --- | --- | --- |
| Register | 最靠近执行单元，访问通常非常快 | 工人手里的工具 |
| L1 / Shared Memory | NASA HECC 汇总的片上访问参考值约 20–30 cycles（受架构与测量条件影响） | 车间内的工作台或近处资料 |
| L2 Cache | NASA HECC 对 A100 的参考约 200 cycles | 多个车间共享的资料柜 |
| Global Memory（未命中缓存时） | NASA HECC 对 A100 给出的参考约 200–1000 cycles；NVIDIA 文档也说明未缓存访问可能需要数百个周期 | 去显存大仓库取货 |

如果仅为换算而假设频率是 2 GHz，那么 20 cycles 约为 10 ns，500 cycles 约为 250 ns。实际 GPU 频率会动态变化，数据请求也可能命中 L1/L2，因此不能把这些换算当成固定实测，更不能直接据此推算整个程序会慢多少倍。

单次访存延迟高，不等于显存吞吐低。GPU 的显存可以提供很高的总带宽；当许多访问能够并行发出、地址访问有效时，GPU 可以在较长的单次延迟期间持续搬运大量数据。**延迟描述一次请求要等多久，带宽描述单位时间能搬多少数据。**

### GPU 为什么适合大量并行工作

GPU 上有许多 SM，每个 SM 可以安排多个 Warp。若一个 Warp 因为 Global Memory 访问而等待，SM 的调度器可以选择另一个当前已准备好执行的 Warp。这个过程叫延迟隐藏。

工厂比喻里，GPU 是一座有很多车间和工作队的工厂：一队人去仓库取料时，调度员可以先让另一队继续加工。仓库里的这一趟并没有变快，但工厂整体不必完全停工。

这使 GPU 擅长**可拆成大量相对独立、结构规律的工作**，例如大批量向量计算、矩阵运算和神经网络张量计算。CPU 通常更适合控制流程复杂、分支不规则或可并行部分很少的工作。比较时要看工作负载：GPU 强项通常是更高的总体吞吐，不代表每个单独任务的响应延迟都比 CPU 低。

延迟隐藏也不是自动成功：SM 需要有足够多的就绪 Warp；若所有 Warp 都在等待同一数据，或任务依赖没有可重叠的工作，计算单元仍会空等。这与 Occupancy 有关系，但 Occupancy 只是驻留 Warp 数的比例，不能直接说明 Warp 此刻是否就绪或在做有效工作。

### Shared Memory 如何加速：把重复用的数据留在车间

Shared Memory 位于 SM 片上，由程序显式安排，能让同一个 Block 的线程协作使用一份数据。适合它的关键条件是：**一次搬入的数据会在之后被多次复用**。

矩阵乘法是典型例子。一个 Block 负责计算输出矩阵的一块 Tile：

1. Block 中的线程按协作方式，从 Global Memory 合并读取输入矩阵中需要的 Tile。
2. 把 Tile 放入 Shared Memory；需要跨线程读取时，在正确位置执行 Block 同步。
3. 多个线程从 Shared Memory 反复读取这批数据，完成多次乘加。
4. 得到输出 Tile 后再写回 Global Memory。

如果没有这次显式复用，多个线程可能反复请求相同的矩阵元素；用 Shared Memory 暂存后，一批数据可以服务更多计算。主要收益是**减少重复的 Global Memory 流量并提高数据复用**，同时片上 Shared Memory 的访问延迟通常低于 Global Memory。

但直接读取 Global Memory 可能命中 L1 或 L2 Cache，所以不能把每一次重复读都当成一次完整的显存往返。Shared Memory 的作用是让开发者明确控制 Block 内的数据放置、复用和访问布局；它不是“只要用就更快”的替代缓存。NVIDIA 的矩阵乘法示例也展示了通过 Tile 将输入数据搬入 Shared Memory 并反复用于计算的优化方式。

### Shared Memory 的收益要扣除使用成本

放入 Shared Memory 之前，线程要先从 Global Memory 加载数据；跨线程依赖时还要同步。Shared Memory 容量有限，用量过大可能减少每个 SM 可同时驻留的 Block；访问还可能遇到 Bank Conflict。若数据只读一次、没有跨线程复用，搬入和同步的成本可能抵消收益。

因此可以按下面的判断来理解：

- **向量加法**：每个输入元素通常只参与一次加法。直接做合并的 Global Memory 读写通常更合适；专门搬进 Shared Memory 往往没有足够复用收益。
- **矩阵乘法**：输入 Tile 会参与多次乘加，适合放入 Shared Memory 让线程块反复复用。
- **不规则访问**：如果 Shared Memory 能把读取重排为更合并的 Global Memory 访问，也可能有帮助；仍要结合同步、Bank Conflict 和实际性能测量判断。

所以 Shared Memory 不是越多越好。判断重点是数据能复用多少、减少了多少外部访存，以及搬运、同步和资源占用增加了多少成本。

## 6. 运算强度与 GPU 性能瓶颈

### Arithmetic Intensity（运算强度）

运算强度通常表示每从内存搬运一个 Byte，完成了多少浮点运算：

~~~text
Arithmetic Intensity = FLOPs / Bytes transferred
~~~

它描述计算工作与数据搬运之间的比例。用于 Roofline 分析时，需要明确 Bytes 指哪个存储层级的流量；经典模型通常关心 Global Memory/DRAM 流量。

以 float32 向量加法 C[i] = A[i] + B[i] 为例，理想情况下每个元素需要读取 A、B 各 4 bytes，写回 C 4 bytes，完成 1 次浮点加法。因此算术强度约为 1/12 FLOP/Byte。数据搬运量相对计算量大，这类操作常表现为 Memory-Bound（带宽受限）。

矩阵乘法能让一块数据被多个计算重复使用，运算次数相对于从显存搬入的数据量显著增加；当数据复用和实现都足够好时，它更可能成为 Compute-Bound（算力受限）。

工厂比喻里，运算强度就是每搬来 1 Byte 的货，计算工能做多少次计算。比值低时，搬货通道容易先忙满；比值高时，计算单元更可能先到达处理能力上限。

### 带宽和算力

- **Memory Bandwidth（内存带宽）**：单位时间内能传输多少数据，常用 GB/s 表示。
- **Compute Throughput（计算吞吐/算力）**：单位时间内能完成多少运算。浮点计算峰值常用 TFLOP/s 表示，具体数值取决于运算精度和 GPU 对应的计算单元。

硬件宣传的峰值算力是理论上限。真实 kernel 还受访存、指令、并行度、数据类型和调度等因素影响，通常需要用实际运行数据分析。

### Roofline：把算力和带宽放在一张图上

Roofline 模型用运算强度、内存带宽上限和计算峰值来估计 kernel 的性能上限：

~~~text
可达到的计算性能 ≤ min（峰值算力，内存带宽 × 运算强度）
~~~

如果运算强度低，带宽上限更可能成为“天花板”，优化重点放在减少或合并数据搬运、提高数据复用；如果运算强度高，可能更接近计算峰值，优化重点转向计算指令与计算单元利用率。

因此先判断 Memory-Bound 还是 Compute-Bound，再决定优化方向，通常比盲目增加线程或 Shared Memory 更有效。

## 7. Occupancy（占用率）：SM 上有多少 Warp 可驻留

CUDA 中的 Occupancy 通常定义为：一个 SM 上当前驻留的 Active Warps 数量，占该 SM 支持的最大 Active Warps 数量的比例。

~~~text
Occupancy = Active Warps / Maximum Active Warps
~~~

可以把一个 SM 想成有固定工位、有限工具包和有限黑板的车间。每个 Block 进来都要占用一定线程位置、寄存器和 Shared Memory。某种资源先耗尽，新的 Block 就不能再同时驻留。除此之外，SM 支持的最大线程数、最大 Warp 数和最大 Block 数也会限制驻留数量。

例如，假设某个 GPU 的一个 SM 最多支持 64 个 Active Warps；若当前驻留 32 个 Warp，占用率就是 50%。这是一个假设例子，具体上限取决于 GPU 架构。

Occupancy 受多种资源共同限制：

- 每个 SM 可用的寄存器总量，以及每个线程的寄存器需求。
- 每个 SM 与每个 Block 可用的 Shared Memory，以及每个 Block 的分配量。
- 每个 SM 支持的最大线程数、Warp 数和驻留 Block 数。
- Block 的线程数量和 GPU 的资源分配粒度。

如果 Shared Memory 用量大到每个 SM 只能同时容纳少数 Block，驻留 Warp 数会减少；寄存器用量也可能限制驻留线程数。这就是“木桶效应”：某类资源先达到上限，就先限制并发驻留量。

### 为什么 Occupancy 高不等于 Kernel 一定快

较高 Occupancy 可以让 SM 在某些 Warp 等待内存或其他依赖时，切换去执行另一个已经准备好的 Warp，从而隐藏一部分延迟。但 Occupancy 是“有多少 Warp 驻留”的比例，不等同于这些 Warp 此刻都在执行有效计算。

- 高 Occupancy 的 kernel 仍可能被低效访存、分支发散或指令依赖拖慢。
- 较低 Occupancy 的 kernel 如果每个 Warp 有足够独立工作、数据访问高效，也可能表现很好。
- 为了追求更高 Occupancy 而减少寄存器或 Shared Memory，有时会让 spilling 或数据重用变差，整体更慢。

所以 Occupancy 是诊断和调优的线索，不是最终目标。要结合实际延迟、吞吐、内存带宽利用、计算利用和 Warp stall 等指标判断。可以用 Nsight Compute 的 Occupancy Calculator 与 profiler 辅助查看。

## 8. 一张图景把概念串起来

以向量加法为例：

1. CPU Host 启动一个 Kernel，配置 Grid 与 Block 的大小。
2. GPU 调度器把 Grid 中的 Block 分配到可用 SM；每个 Block 的线程由同一个 SM 承载。
3. 每个 Block 内的线程组成 32-thread Warp，各线程根据索引处理不同数组元素。
4. 线程读取 A、B，经过 Cache 或从 Global Memory 获取数据；线程进行加法并写回 C。
5. 连续线程访问连续元素时，Global Memory 请求可合并成较少事务。
6. 算术强度很低时，性能容易受内存带宽限制。
7. Block 的寄存器和 Shared Memory 用量影响每个 SM 能同时驻留多少 Block 和 Warp，进而影响隐藏访存延迟的能力。
8. 边界线程通过 idx < n 保护数组访问；若存在 Block 屏障，还要保证线程按正确路径到达同步点。

这样从“任务如何分配”连接到“数据怎样移动”，再连接到“性能瓶颈与资源占用”。

## 9. 本章记忆卡

- GPU 上的执行层级：Kernel → Grid → Block → Thread；Block 由 SM 承载，Thread 以 32 个一组组成 Warp。
- Block 内线程可以共享 Shared Memory 并同步；不同 Block 不应依赖固定调度顺序。
- Register 是 Thread 私有；Shared Memory 面向 Block 内协作；Global Memory 容量大但比片上存储访问更远；L1/L2 自动缓存部分访问。
- 合并访存的目标是用较少的内存事务满足 Warp 的访问请求。
- 运算强度是 FLOPs/Bytes；低运算强度的向量加法常受带宽限制，数据复用充分的矩阵乘法常更接近算力限制。
- Occupancy 是 Active Warps 占 SM 最大支持 Warps 的比例；资源限制其数值，但高 Occupancy 本身不保证高性能。
- 边界保护要避免无效读写；有 Block 屏障时要保证线程正确参与同步。

### 面试表达

> CUDA 会把一次 Kernel 启动组织成 Grid，Grid 由多个 Block 组成，Block 的线程在同一个 SM 上执行，线程又按 32 个组成 Warp。性能分析时我会同时看执行资源和数据流：寄存器、Shared Memory、Global Memory 与 Cache 的访问范围不同；Warp 访问显存时尽量合并请求。再用运算强度判断 kernel 更可能受带宽还是算力限制，并把 Occupancy 当作隐藏延迟的一个指标，而不是单纯追求 100%。

## 参考资料

- [CUDA Programming Guide：编程模型与 Thread/Block/Grid/SM](https://docs.nvidia.com/cuda/cuda-programming-guide/01-introduction/programming-model.html)
- [CUDA Programming Guide：CUDA Kernel、边界检查、Shared Memory 与 Coalescing](https://docs.nvidia.com/cuda/cuda-programming-guide/02-basics/writing-cuda-kernels.html)
- [CUDA C++ Best Practices Guide：合并访存与 Occupancy](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/)
- [Nsight Compute Profiling Guide：Roofline 与 Arithmetic Intensity](https://docs.nvidia.com/nsight-compute/ProfilingGuide/)
- [NASA HECC：A100/H100 GPU 架构与存储延迟参考](https://www.nas.nasa.gov/hecc/support/kb/basics-on-nvidia-gpu-hardware-architecture_704.html)
