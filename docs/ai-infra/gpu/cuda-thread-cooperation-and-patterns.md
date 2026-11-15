# CUDA 线程协作与计算模式：Shared Memory、Shuffle、Reduction 与矩阵乘法

这页承接 [CUDA C/C++ 入门](cuda-c-cpp-memory-kernel.md) 和 [GPU 执行模型、内存层级与性能分析](cuda-execution-memory-performance.md)，从“一个线程处理一个元素”进一步看线程如何协作：Block 内线程用 Shared Memory 复用数据，Warp 内线程用 Shuffle 交换寄存器值，再把这些机制用于 Reduction 和矩阵乘法。

这里的 **Tiling** 指传统 SIMT Kernel 中的“分块计算”：线程块把 Global Memory 中的一小块数据搬入 Shared Memory，再重复使用。它描述的是一种算法与访存优化方式，和 CUDA 13.3 引入的 CUDA Tile / cuTile 编程接口不是同一个概念。

## 1. Shared Memory 与 Block 内同步

### 1.1 Shared Memory 是什么

在 Kernel 中用 `__shared__` 声明的变量位于 GPU 的片上共享内存。它有三个关键特征：

- **作用范围**：同一个 Thread Block 内的线程可以读写这块内存；其他 Block 有各自独立的 Shared Memory 区域。
- **生命周期**：一次 Block 执行期间有效。
- **用途**：保存线程之间要共享的中间结果，或暂存从 Global Memory 搬入、接下来会重复使用的数据。

例子：

~~~cpp
constexpr int BLOCK_SIZE = 128;

__global__ void stageData(const float* input, float* output, int n) {
    __shared__ float tile[BLOCK_SIZE];

    int tid = threadIdx.x;
    int index = blockIdx.x * blockDim.x + tid;

    tile[tid] = (index < n) ? input[index] : 0.0f;

    __syncthreads();

    // 屏障之后，Block 内线程可以读取已装入的 tile
    if (index < n) {
        output[index] = tile[tid];
    }
}
~~~

上面每个线程先把自己的一个元素放进共享数组。由于线程执行先后不固定，某个线程可能已经写完，而另一个线程还没写完；因此在读取依赖所有线程写入的数据前，需要设置同步点。

### 1.2 `__syncthreads()` 同时建立屏障和内存顺序

`__syncthreads()` 是 **Thread Block 级屏障**：Block 中未退出的线程都到达这一调用后，才一起继续执行。它也保证屏障前相关的内存写入，在参与线程越过屏障后按规定可见。

可以将它读成：

1. “大家先把当前这一步做完”；
2. “确认共享数据已准备好”；
3. “再一起进入下一步”。

它不能让不同 Block 在 Kernel 内直接进行普通的全局同步。需要跨 Block 汇总时，常用分阶段 Kernel：每个 Block 先写出局部结果，后续 Kernel 再处理这些结果。

**易错点：**参与同一屏障的线程必须在一致的控制流中到达它。若部分线程提前 `return` 或因分支跳过屏障，其他线程可能一直等待，程序也可能产生未定义行为。矩阵分块时，即便某些线程对应的矩阵元素越界，也应填入 0 并继续经过屏障，不要提前退出。

若共享数据只由同一 Warp 中的线程写入和读取，在符合 Warp 参与条件的情况下可考虑 `__syncwarp()`；若 Block 中跨 Warp 的线程有数据依赖，使用 Block 级同步。具体选择取决于谁生产、谁消费数据。

## 2. Warp Shuffle：不经过 Shared Memory 交换寄存器值

### 2.1 Warp 内的数据交换

Warp Shuffle Intrinsics 可以让同一 Warp 中的线程从另一个 Lane 读取一个寄存器值，不用先写 Shared Memory 再读回。CUDA Warp 通常包含 32 个 Lane，每个 Lane 对应 Warp 中的一个线程。

两个常用接口：

- `__shfl_sync(mask, value, srcLane)`：从指定的 Lane 读取 value。
- `__shfl_down_sync(mask, value, delta)`：从当前 Lane 编号加上 delta 的 Lane 读取 value。

例如，Warp 中所有 Lane 都调用下面这条指令时，每个 Lane 都能拿到 Lane 0 的值：

~~~cpp
unsigned mask = 0xffffffffu;
float laneZeroValue = __shfl_sync(mask, value, 0);
~~~

Shuffle 适合 Warp 内归约、前缀计算和邻居数据交换。它省去显式 Shared Memory 暂存，也不需要 Block 范围的 `__syncthreads()`；不过它仅在 Warp 内交换寄存器值，不能代替 Shared Memory 的跨 Warp 通信。

### 2.2 `_sync` 版本的 mask 约束

`mask` 的每一位对应一个 Lane。调用者自己的位必须被设为 1；mask 中列出的未退出线程必须在同一程序位置使用相同 mask 调用对应 Intrinsic。并且，读取来源 Lane 必须是有效参与者，否则取回的值没有定义。

- 对完整 Warp 的教学示例，若 32 个 Lane 都参与，可用 `0xffffffffu`。
- 若只让一部分 Lane 参与，应构造准确的参与 mask，并保证来源 Lane 也参与。
- `__shfl_sync` 的同步作用限于 Warp 参与线程的这次操作；它**不提供 Shared/Global Memory 的通用内存屏障**。

因此，不能把 “带 sync 后缀”理解为“所有 GPU 线程或所有内存访问都已经同步”。它表达的是 Warp 参与者在这一 Warp 操作上的配合约束。

## 3. Reduction：从线程局部值合并成总和

**Reduction（归约）**把许多输入值按某个运算逐步合成更少的结果，例如求和、最大值或最小值。求和可用树形结构并行执行：每一轮让线程与一个距离逐渐缩小的伙伴合并，最后由 Warp 的 Lane 0 得到该 Warp 的总和。

### 3.1 一个完整 Warp 的并行求和

假设一个 Warp 的 32 个 Lane 各有一个局部值，先按 16、8、4、2、1 的距离逐轮相加：

~~~cpp
__device__ float warpReduceSum(float value) {
    const unsigned mask = 0xffffffffu;

    for (int offset = 16; offset > 0; offset >>= 1) {
        value += __shfl_down_sync(mask, value, offset);
    }

    // 只有 Lane 0 的 value 是完整 Warp 的总和
    return value;
}
~~~

每轮可以直观地看成把可用的部分和合并一层：

~~~text
第 1 轮：间隔 16
第 2 轮：间隔 8
第 3 轮：间隔 4
第 4 轮：间隔 2
第 5 轮：间隔 1
结果：Lane 0 持有 32 个输入值的总和
~~~

每个 Lane 都会执行 Shuffle，但归约完成后应由 Lane 0 使用完整结果。这个例子的前提是一个 Warp 的 32 个 Lane 都参加每一轮；数组尾部的无效元素要先填 0，而不是让部分 Lane 在 Shuffle 之前提前退出。

### 3.2 从 Warp 归约扩展到一个 Block

一个 Block 通常包含多个 Warp。做法是先让每个 Warp 各自归约，再让每个 Warp 的 Lane 0 把局部总和写进 Shared Memory。经过一次 Block 屏障后，Warp 0 读取这些局部总和，再做一轮 Warp 归约。

下面以 256 个线程（8 个完整 Warp）为例。Kernel 为每个 Block 产生一个部分和：

~~~cpp
constexpr int BLOCK_SIZE = 256;
constexpr int WARP_SIZE = 32;

__global__ void reduceBlockSum(const float* input,
                               float* blockSums,
                               int n) {
    __shared__ float warpSums[BLOCK_SIZE / WARP_SIZE];

    int tid = threadIdx.x;
    int lane = tid % WARP_SIZE;
    int warp = tid / WARP_SIZE;
    int index = blockIdx.x * blockDim.x + tid;

    // 超出输入范围的线程用加法单位元 0 填充，仍继续参与同步
    float value = (index < n) ? input[index] : 0.0f;

    // 每个 Warp 各自归约，Lane 0 得到该 Warp 的局部总和
    for (int offset = WARP_SIZE / 2; offset > 0; offset >>= 1) {
        value += __shfl_down_sync(0xffffffffu, value, offset);
    }

    if (lane == 0) {
        warpSums[warp] = value;
    }

    // 等待所有 Warp 写完自己的局部总和
    __syncthreads();

    // Warp 0 读取所有 Warp 的局部总和并完成 Block 级归约
    if (warp == 0) {
        float blockValue = (lane < BLOCK_SIZE / WARP_SIZE)
                               ? warpSums[lane]
                               : 0.0f;

        for (int offset = WARP_SIZE / 2; offset > 0; offset >>= 1) {
            blockValue +=
                __shfl_down_sync(0xffffffffu, blockValue, offset);
        }

        if (lane == 0) {
            blockSums[blockIdx.x] = blockValue;
        }
    }
}
~~~

启动时要让 Block 大小与示例约定一致：

~~~cpp
int blocks = (n + BLOCK_SIZE - 1) / BLOCK_SIZE;
reduceBlockSum<<<blocks, BLOCK_SIZE>>>(d_input, d_blockSums, n);
~~~

Kernel 输出的是“每个 Block 一个部分和”。若输入跨越多个 Block，还需要第二个归约阶段继续合并这些部分和；也可以使用合适的归约库或原子操作实现最终合并。实际工程可优先评估 CUB 等成熟归约原语，它们处理参与掩码和硬件差异更完整；手写 Shuffle 版本有助于理解归约的线程协作过程。

**数值注意：**浮点加法受舍入影响，不满足严格结合律。并行归约改变了加法顺序，因此结果可能与 CPU 串行求和有细微差异。验证时应采用与精度要求相匹配的容差。

## 4. Tiling：把 Global Memory 数据块搬近并重复使用

### 4.1 为什么分块

直接矩阵乘法会让多个线程反复从 Global Memory 读取相同输入。例如，输出矩阵中的一块 C Tile 会重复使用 A 的部分行和 B 的部分列。若每个线程都独立从 Global Memory 读取，很多加载会重复发生。

**Tiling**把较大的矩阵划成小块。一个 Block 协作加载输入 Tile 到 Shared Memory，所有线程等待数据就绪，再从片上重复读取并计算。目标是让一次 Global Memory 读取服务更多乘加运算，从而提高数据复用、减少重复的 Global Memory 流量。

### 4.2 一个输出 Tile 如何计算

对于 C = A × B：

- A 的形状是 M × K；
- B 的形状是 K × N；
- C 的形状是 M × N；
- 一个 Thread Block 负责 C 中的一个 TILE × TILE 区域；
- 当 K 大于 TILE 时，沿 K 维分多轮加载输入块。

每轮的流程：

1. Block 内线程合作，从 Global Memory 加载 A、B 的两个 Tile。
2. 对超出矩阵边界的位置填 0。
3. 调用 `__syncthreads()`，确保所有需要的数据已写入 Shared Memory。
4. 每个线程从两个 Tile 读取数值，执行乘加并累积到自己的寄存器。
5. 再次同步，确保本轮读取完成后再覆盖 Shared Memory，进入下一轮 K 分块。

对 TILE = 16 的 FP32 矩阵乘法，每个 Block 暂存两个 16 × 16 的 Tile，共需 2 × 16 × 16 × 4 = 2048 bytes Shared Memory（不含其他资源）。每个 A Tile 元素会被多个输出列复用，每个 B Tile 元素会被多个输出行复用。

## 5. CUDA 矩阵乘法：二维线程索引与一维地址

### 5.1 从二维矩阵映射到一维内存

C/C++ 中的普通二维矩阵经常以**行优先（row-major）**连续存放。若矩阵有 cols 列，第 row 行、第 col 列的元素地址偏移为：

~~~text
row * cols + col
~~~

因此：

- A 是 M × K：A[row][col] → A[row * K + col]
- B 是 K × N：B[row][col] → B[row * N + col]
- C 是 M × N：C[row][col] → C[row * N + col]

索引计算中的列数必须使用对应矩阵的列数。A、B、C 的列数分别是 K、N、N，不能把它们混用。

### 5.2 二维 Grid/Block 与输出元素

使用二维 Thread Block 表示输出 Tile 的二维坐标：

- `threadIdx.x` 表示线程在当前 Block 的 Tile 中负责哪一列；
- `threadIdx.y` 表示线程负责哪一行；
- `blockIdx.x` 选择输出 Tile 的列；
- `blockIdx.y` 选择输出 Tile 的行。

对应的输出元素位置为：

~~~cpp
int row = blockIdx.y * TILE + threadIdx.y;
int col = blockIdx.x * TILE + threadIdx.x;
~~~

让 x 方向相邻线程访问连续列，有助于 Global Memory 合并访存。Grid 的 x 方向覆盖输出矩阵列，y 方向覆盖输出矩阵行。

### 5.3 使用 Shared Memory Tile 的完整 Kernel

~~~cpp
constexpr int TILE = 16;

__global__ void matmulTiled(const float* A,
                            const float* B,
                            float* C,
                            int M,
                            int N,
                            int K) {
    __shared__ float tileA[TILE][TILE];
    __shared__ float tileB[TILE][TILE];

    int tx = threadIdx.x;
    int ty = threadIdx.y;

    int row = blockIdx.y * TILE + ty;
    int col = blockIdx.x * TILE + tx;

    float sum = 0.0f;
    int tileCount = (K + TILE - 1) / TILE;

    for (int t = 0; t < tileCount; ++t) {
        int aCol = t * TILE + tx;
        int bRow = t * TILE + ty;

        // A 的 TILE：Block 内线程合作加载，边界之外用 0 填充
        tileA[ty][tx] =
            (row < M && aCol < K) ? A[row * K + aCol] : 0.0f;

        // B 的 TILE：x 方向线程读取连续列
        tileB[ty][tx] =
            (bRow < K && col < N) ? B[bRow * N + col] : 0.0f;

        // 等待整个 Block 完成 Tile 加载
        __syncthreads();

        // 当前线程计算自己负责的一个 C[row, col]
        for (int k = 0; k < TILE; ++k) {
            sum += tileA[ty][k] * tileB[k][tx];
        }

        // 确保所有线程已读完当前 Tile，再进入下一轮覆盖它们
        __syncthreads();
    }

    if (row < M && col < N) {
        C[row * N + col] = sum;
    }
}
~~~

Host 端的启动配置应与 Kernel 中的 TILE 对应：

~~~cpp
dim3 block(TILE, TILE);
dim3 grid((N + TILE - 1) / TILE,
          (M + TILE - 1) / TILE);

matmulTiled<<<grid, block>>>(d_A, d_B, d_C, M, N, K);
~~~

其中 A、B、C 指针指向 Device 可访问的内存。该 Kernel 处理 M、N、K 不能整除 TILE 的情况：越界加载填 0，输出越界线程不写回。每个线程都执行循环内的两次 Block 屏障，因此没有线程因边界判断而提前跳过同步点。

### 5.4 把矩阵乘法执行过程串起来

一个 Block 算出 C 的一个 TILE × TILE 输出块。Block 中每个线程对应一个输出坐标并在寄存器里保存部分和；整个 Block 沿 K 维循环，重复加载 A、B 子块并让 Shared Memory 中的数据参与多个乘加。最后线程将自己的累积值写到 C 的对应位置。

~~~text
Grid 的一个 Block
  └── 负责 C 中的一块输出 Tile
       ├── 协作加载 A Tile 和 B Tile
       ├── __syncthreads：等待 Tile 到齐
       ├── 从 Shared Memory 读取，线程各自累积乘加
       ├── __syncthreads：读完后再覆盖下一轮 Tile
       └── 输出各线程负责的 C 元素
~~~

这样同时用到了二维索引、行优先线性地址、Block 协作、Shared Memory 复用、线程同步和边界检查。

## 6. 四种机制之间的分工

| 机制 | 协作范围 | 数据在哪里交换或暂存 | 典型用途 |
| --- | --- | --- | --- |
| Shared Memory | 一个 Thread Block | Block 共享的片上内存 | 跨 Warp 协作、Tile 缓存与重复数据复用 |
| `__syncthreads()` | 一个 Thread Block | 建立执行屏障和共享内存可见顺序 | 等待协作加载完成，或等待共享数据读完 |
| Warp Shuffle | 一个 Warp | 在线程寄存器值之间交换 | Warp 归约、广播、邻居数据交换 |
| Reduction | 由算法决定，可从 Warp 扩到 Block/多 Block | Shuffle、Shared Memory 或库原语 | 求和、最大值、最小值等聚合 |
| Tiling | 一般以 Block 作为协作单元 | Global Memory → Shared Memory → 寄存器 | 增强数据复用，减少重复的 Global Memory 访问 |

Shared Memory 回答“Block 之间怎样共享并复用数据”；Shuffle 回答“同一个 Warp 的 Lane 怎样交换寄存器值”；Reduction 是把多个局部结果合成更少结果的算法；Tiling 则把共享内存和线程协作组织成数据分块的计算方案。

## 7. 易错点与性能检查

1. **把 Shared Memory 当成全 Grid 可共享**：它按 Block 隔离，其他 Block 不能直接读取本 Block 的共享数组。
2. **漏掉生产者与消费者之间的屏障**：写入共享 Tile 后，要在其他线程读取之前同步。
3. **不一致地到达 `__syncthreads()`**：越界线程也要通过填充值参与屏障，不要在屏障前提前退出。
4. **把 Shuffle 当成内存屏障**：Shuffle 交换 Warp Lane 的寄存器值，不会为 Shared/Global Memory 提供通用的内存顺序。
5. **Mask 与实际参与者不一致**：mask 中的 Lane 必须按同一调用点参与；来源 Lane 也必须有效。
6. **假设所有线程都有有效数据**：归约尾部元素要用运算单位元补齐，例如求和用 0、乘积用 1、最大值用负无穷或对应类型最小值。
7. **只做 Warp 归约却认为全局结果已经完成**：多个 Warp 需要先汇总局部值，多个 Block 还要用第二阶段等方式继续汇总。
8. **忽略浮点计算顺序**：并行归约改变相加顺序，结果可能与串行计算有舍入差异。
9. **把 Tiling 当成必然加速开关**：Tile 加载和同步有成本；Shared Memory 占用也会影响每个 SM 的并发 Block 数。要结合矩阵规模、访问模式和 profiler 结果判断。
10. **矩阵线性偏移使用错误列数**：A 使用 K，B 和 C 使用 N。
11. **边界检查只包住同步调用**：线程分支不一致地跳过屏障会造成程序错误；让所有线程执行共同的屏障，再对有效输出做条件写回。
12. **只看减少了多少 Global Memory 读取**：还要检查 Global Memory 是否合并访问、Shared Memory 是否有 Bank Conflict、寄存器与共享内存占用是否压低 Occupancy。

## 8. 面试表达：Shared Memory、Shuffle 与矩阵乘法优化

> 对一个 CUDA Block 内会重复使用的数据，我可以让线程协作从 Global Memory 加载到 Shared Memory，并通过 __syncthreads() 确保所有数据就绪后再计算。矩阵乘法是典型场景：一个 Block 负责输出矩阵中的一个 Tile，线程用二维坐标映射到输出位置，沿 K 维分块加载 A、B，重复做乘加，从而让每次 Global Memory 读取服务多个计算。Warp 内的局部结果可以用 __shfl_down_sync 在 Lane 间交换寄存器值并做树形 Reduction；多个 Warp 的结果再写到 Shared Memory 汇总。性能是否提升还要结合边界处理、访存合并、同步成本、资源占用和实际测量判断。

## 参考资料

- NVIDIA, [CUDA Programming Guide：Shared Memory 与 Thread Block 同步](https://docs.nvidia.com/cuda/cuda-programming-guide/02-basics/writing-cuda-kernels.html)
- NVIDIA, [CUDA Programming Guide：`__shared__` 与 `__shfl*_sync` Intrinsics](https://docs.nvidia.com/cuda/cuda-programming-guide/05-appendices/cpp-language-extensions.html)
- NVIDIA, [CUDA C++ Best Practices Guide：Shared Memory in Matrix Multiplication](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/)
- NVIDIA, [CUDA Programming Guide：Writing Tile Kernels](https://docs.nvidia.com/cuda/cuda-programming-guide/02-basics/writing-tile-kernels.html)
