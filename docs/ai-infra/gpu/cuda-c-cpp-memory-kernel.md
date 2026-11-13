# CUDA C/C++ 入门：编译、Kernel 启动与内存管理

这页把 C/C++ 基础语法与 CUDA 编程模型连接起来，重点看一段向量加法程序怎样从 CPU Host 发起工作、把数据交给 GPU Device、并行执行 Kernel，再把结果交回 CPU。

学习与动手主线来自《大规模并行处理器程序设计（原书第 4 版）》第 2 章的向量加法与设备全局内存/数据传输，以及《CUDA C 编程权威指南》第 2 章有关内存管理和 Kernel 启动的内容。向量加法已在 RTX 4090 上通过托管内存和显式显存管理两种方式运行。

## 1. CUDA 程序用到的 C/C++ 基础

### 1.1 数组、地址和指针

数组是一段连续排列的同类型元素；地址表示数据在内存中的位置；指针变量保存一个地址。

~~~cpp
int value = 7;
int* p = &value;   // p 保存 value 的地址
*p = 9;            // 通过地址访问并修改 value
~~~

- **&value** 取变量的地址。
- **int* p** 声明一个指向 int 的指针。
- ***p** 解引用指针，访问指针指向的 int。
- 指针加 1 会移动一个所指类型的元素大小。例如 float* 加 1，地址前进 sizeof(float) 个字节。

数组在很多表达式中会转换为指向首元素的指针。因此，数组传给函数时常见的参数形式 float* data 实际上接收的是首元素地址；函数还需要另外接收数组长度。

### 1.2 函数值传递、指针传递和引用传递

~~~cpp
void changeValue(int x) {
    x = 10;           // 修改的是形参副本
}

void changeByPointer(int* x) {
    if (x != nullptr) {
        *x = 10;       // 通过复制过来的地址修改原变量
    }
}

void changeByReference(int& x) {
    x = 10;            // x 是原变量的别名
}
~~~

| 写法 | 函数收到什么 | 能否修改调用方的数据 |
| --- | --- | --- |
| void f(int x) | int 的值副本 | 修改 x 不影响调用方变量 |
| void f(int* x) | 指针地址的副本 | 可以用 *x 修改该地址指向的数据 |
| void f(int& x) | 原变量的引用别名 | 可以直接修改引用指向的变量 |

指针参数本身也是按值传递：函数拿到的是地址的一份副本。它可以通过这个地址修改数据；若要让函数改写调用方的指针变量本身，通常要传入指针的地址（例如 T**），或使用 C++ 的指针引用（例如 T*&）。

这个区别会出现在 CUDA 分配接口里：cudaMalloc 需要把新分配的 Device 地址写回调用方的指针变量，所以接口接收的是 void**。

### 1.3 vector、malloc/free 与 new/delete

| 方式 | 管理什么 | 主要特点 |
| --- | --- | --- |
| std::vector<T> | C++ 动态数组 | 自动管理生命周期，可用 .data() 取得连续元素的首地址 |
| malloc / free | 原始字节内存 | C 风格分配，不负责调用 C++ 对象的构造和析构 |
| new / delete | C++ 对象 | 创建对象并调用构造函数，释放时调用析构函数 |
| new[] / delete[] | C++ 对象数组 | 数组分配和释放必须成对使用 |

Host 上常用 std::vector<float> hostData(n) 来管理数组。它的数据位于 CPU 主存；.data() 只返回这段 Host 内存的地址，**不会自动把数据传到 GPU**。

资源释放要与分配方式匹配：malloc 配 free，new 配 delete，new[] 配 delete[]，cudaMalloc 或 cudaMallocManaged 配 cudaFree。这些资源属于不同的管理接口，不能混用释放函数。

## 2. Linux 编译工具链：gcc、g++ 和 nvcc

| 命令 | 常见输入 | 用途 |
| --- | --- | --- |
| gcc | .c | 调用 C 编译流程生成可执行程序或目标文件 |
| g++ | .cpp、.cc | 调用 C++ 编译流程，并链接 C++ 标准库 |
| nvcc | .cu | CUDA 编译驱动，处理 Host 代码和 Device 代码，生成可在目标 GPU 上运行的程序 |

常见命令：

~~~bash
gcc hello.c -o hello
g++ -std=c++17 hello.cpp -o hello
nvcc vector_add.cu -o vector_add
./vector_add
~~~

nvcc 不只是一个“GPU 版 g++”。它识别 .cu 中的 CUDA 扩展，把 Device 代码编译成 GPU 可执行代码，同时将 Host 部分交给受支持的 C++ Host 编译器处理，再把结果组织成一个应用程序。[NVIDIA 对 nvcc 编译流程的说明](https://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/)

编译程序还要留意 CUDA Toolkit、GPU 架构和 Host 编译器版本之间的兼容关系。出现 nvcc 报错时，应先看错误发生在 CUDA Device 编译阶段，还是转交给 Host 编译器的阶段。

## 3. CUDA 函数：代码在哪执行、从哪里调用

CUDA 用执行空间标记说明函数面向 Host 还是 Device：

| 标记 | 函数在哪里执行 | 通常从哪里调用 | 用途 |
| --- | --- | --- | --- |
| 默认或 __host__ | CPU Host | Host | 普通主机函数；不写标记时通常默认是 Host 函数 |
| __device__ | GPU Device | Device 函数或 Kernel | GPU 上运行的辅助函数 |
| __global__ | GPU Device | 常由 Host 通过 Kernel 启动语法调用 | GPU Kernel 入口函数 |
| __host__ __device__ | 可分别编译为 Host 和 Device 版本 | Host 或 Device | 一份函数定义可供两侧调用，代码需满足两侧约束 |

常规 __global__ Kernel 返回类型为 void，由 CPU 端配置 Grid 和 Block 后发起。Kernel 启动通常相对于 CPU Host 异步返回：Host 线程提交 GPU 工作后可以继续执行，GPU 仍在后台处理已提交的 Kernel。[CUDA C++ 函数执行空间说明](https://docs.nvidia.com/cuda/cuda-programming-guide/02-basics/intro-to-cuda-cpp.html)

## 4. Kernel 启动：Grid、Block 和 Thread 索引

启动语法：

~~~cpp
kernelName<<<gridSize, blockSize>>>(arguments);
~~~

- gridSize：Grid 中 Block 的数量或维度。
- blockSize：每个 Block 中 Thread 的数量或维度。
- arguments：传给 Kernel 的参数，例如数组指针和元素个数。
- 常用一维配置 <<<blocks, threads>>>；还可用额外参数配置动态 Shared Memory 和 CUDA Stream。

下面是一个一维向量加法 Kernel：

~~~cpp
__global__ void vectorAdd(const float* a,
                          const float* b,
                          float* c,
                          int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;

    if (i < n) {
        c[i] = a[i] + b[i];
    }
}
~~~

所有 Thread 执行同一段 Kernel 代码，但内置索引值不同：

- threadIdx.x：Thread 在本 Block 内的索引。
- blockIdx.x：Block 在 Grid 内的索引。
- blockDim.x：一个 Block 在 x 方向包含的 Thread 数。
- i = blockIdx.x * blockDim.x + threadIdx.x：把 Block 内局部索引换算成整个 Grid 的全局元素索引。

例如，Block 大小为 256，数据有 1000 个元素，则 Block 数为 4，启动的线程槽位总数为 1024。最后一个 Block 中只有 232 个 Thread 对应有效元素，其余 Thread 的全局索引超过 999。if (i < n) 是边界检查，保证它们不会越界读写。

通常用向上取整计算 Block 数：

~~~cpp
int threads = 256;
int blocks = (n + threads - 1) / threads;
vectorAdd<<<blocks, threads>>>(d_a, d_b, d_c, n);
~~~

对 n=1000，计算结果是 ceil(1000 / 256) = 4。实际代码需要保证 n 大于 0，避免生成 0 个 Block 的无效配置。

## 5. CPU 与 GPU 的数据流：一次向量加法怎样跑完

采用独立 Host 内存和 Device 显存时，典型数据流如下：

~~~text
CPU Host 上的输入数组
        │ cudaMemcpyHostToDevice
        ▼
GPU Device 上的输入数组
        │ Kernel 读取、并行计算
        ▼
GPU Device 上的输出数组
        │ cudaMemcpyDeviceToHost
        ▼
CPU Host 上读取结果
~~~

按程序步骤看：

1. CPU 创建和初始化 Host 输入数据。
2. 通过 cudaMalloc 在 GPU 显存申请输入和输出缓冲区。
3. 通过 HostToDevice 拷贝把输入送到 GPU。
4. CPU 发起 Kernel，Grid 中的 Thread 分摊数组元素。
5. GPU 将计算结果写入 Device 输出缓冲区。
6. 等 GPU 工作完成后，把结果 DeviceToHost 拷贝回 Host。
7. CPU 验证结果，最后分别释放 Host 与 Device 资源。

传给 Kernel 的 d_a、d_b、d_c 是指针值。Kernel 接收这些地址后，在 GPU 上访问对应的 Device 内存；启动参数传递的是地址等小量参数，**并不会自动把指针背后的整块数组从 Host 拷到 Device**。

## 6. 显式设备内存：cudaMalloc、cudaMemcpy、cudaFree

显式管理时，Host 数组和 GPU 显存分开分配，数据移动由程序明确控制。

~~~cpp
#include <cuda_runtime.h>
#include <cstdio>
#include <cstdlib>
#include <vector>

void checkCuda(cudaError_t error, const char* operation) {
    if (error != cudaSuccess) {
        std::fprintf(stderr, "%s failed: %s", operation,
                     cudaGetErrorString(error));
        std::exit(EXIT_FAILURE);
    }
}

__global__ void vectorAdd(const float* a,
                          const float* b,
                          float* c,
                          int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) {
        c[i] = a[i] + b[i];
    }
}

int main() {
    const int n = 1000;
    const size_t bytes = static_cast<size_t>(n) * sizeof(float);

    // std::vector 管理 Host 端 CPU 内存
    std::vector<float> h_a(n, 1.0f);
    std::vector<float> h_b(n, 2.0f);
    std::vector<float> h_c(n);

    // 这些指针最终指向 GPU Device 显存
    float* d_a = nullptr;
    float* d_b = nullptr;
    float* d_c = nullptr;

    checkCuda(cudaMalloc(reinterpret_cast<void**>(&d_a), bytes), "cudaMalloc d_a");
    checkCuda(cudaMalloc(reinterpret_cast<void**>(&d_b), bytes), "cudaMalloc d_b");
    checkCuda(cudaMalloc(reinterpret_cast<void**>(&d_c), bytes), "cudaMalloc d_c");

    checkCuda(cudaMemcpy(d_a, h_a.data(), bytes, cudaMemcpyHostToDevice),
              "copy h_a to d_a");
    checkCuda(cudaMemcpy(d_b, h_b.data(), bytes, cudaMemcpyHostToDevice),
              "copy h_b to d_b");

    const int threads = 256;
    const int blocks = (n + threads - 1) / threads;
    vectorAdd<<<blocks, threads>>>(d_a, d_b, d_c, n);

    // 检查 Kernel 是否成功提交，以及异步执行是否成功完成
    checkCuda(cudaGetLastError(), "vectorAdd launch");
    checkCuda(cudaDeviceSynchronize(), "vectorAdd execution");

    checkCuda(cudaMemcpy(h_c.data(), d_c, bytes, cudaMemcpyDeviceToHost),
              "copy d_c to h_c");

    std::printf("h_c[0] = %.1f", h_c[0]); // 3.0

    checkCuda(cudaFree(d_a), "cudaFree d_a");
    checkCuda(cudaFree(d_b), "cudaFree d_b");
    checkCuda(cudaFree(d_c), "cudaFree d_c");
    return 0;
}
~~~

几个 API 的职责：

- cudaMalloc(&devicePtr, bytes)：向 CUDA Runtime 请求 GPU 显存，将 Device 地址写入指针变量。
- cudaMemcpy(dst, src, bytes, kind)：按指定方向复制指定字节数。
- cudaMemcpyHostToDevice：CPU 主存 → GPU 显存。
- cudaMemcpyDeviceToHost：GPU 显存 → CPU 主存。
- cudaFree(devicePtr)：释放由 CUDA 显存分配接口获得的资源。
- reinterpret_cast<void**>(&d_a)：把 float** 转成 cudaMalloc 参数要求的 void**，使 API 能把分配得到的地址写回 d_a。

显式管理的优点是数据何时分配、何时复制、何时释放都很明确，便于掌握传输和性能；代价是代码需要正确维护 Host 与 Device 两侧的指针和数据生命周期。[CUDA Runtime 内存管理 API](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__MEMORY.html)

## 7. 托管内存：cudaMallocManaged

cudaMallocManaged 分配 Unified Memory（统一内存）管理的内存区域。Host 和 Device 代码可以使用同一个指针；CUDA Runtime 与驱动负责管理数据在 CPU 与 GPU 之间的可访问性和迁移。它让入门代码少写显式拷贝，但数据仍要在合适的硬件位置可用，执行过程中可能发生迁移或访问协调，并不意味着 CPU 和 GPU 永远在同一物理内存位置上直接读写。[NVIDIA Unified Memory 文档](https://docs.nvidia.com/cuda/cuda-programming-guide/04-special-topics/unified-memory.html)

向量加法使用托管内存时，核心步骤可以写成：

~~~cpp
float* a = nullptr;
float* b = nullptr;
float* c = nullptr;

checkCuda(cudaMallocManaged(reinterpret_cast<void**>(&a), bytes),
          "cudaMallocManaged a");
checkCuda(cudaMallocManaged(reinterpret_cast<void**>(&b), bytes),
          "cudaMallocManaged b");
checkCuda(cudaMallocManaged(reinterpret_cast<void**>(&c), bytes),
          "cudaMallocManaged c");

// CPU 可以初始化托管数组
for (int i = 0; i < n; ++i) {
    a[i] = 1.0f;
    b[i] = 2.0f;
}

vectorAdd<<<blocks, threads>>>(a, b, c, n);
checkCuda(cudaGetLastError(), "vectorAdd launch");
checkCuda(cudaDeviceSynchronize(), "vectorAdd execution");

// 同步之后 CPU 再读取 GPU 生成的结果
std::printf("c[0] = %.1f", c[0]);

checkCuda(cudaFree(a), "cudaFree a");
checkCuda(cudaFree(b), "cudaFree b");
checkCuda(cudaFree(c), "cudaFree c");
~~~

两种方法的核心差别：

| 方式 | 指针与内存 | CPU/GPU 数据移动 | 适合的学习阶段 |
| --- | --- | --- | --- |
| cudaMalloc + cudaMemcpy | Host 与 Device 使用不同内存分配和指针 | 程序明确复制 | 学清数据流、分析传输和控制资源 |
| cudaMallocManaged | Host 与 Device 使用同一个托管指针 | Runtime/驱动管理可访问性和迁移 | 快速验证 Kernel、降低入门代码复杂度 |

Managed Memory 省掉的是显式拷贝代码，不代表数据移动成本消失。理解基础后，仍要查看何时发生迁移，以及数据访问是否让 CPU/GPU 来回移动同一批页面。

## 8. Kernel 启动与同步：异步提交不等于算完

Kernel launch 通常是异步的：CPU 执行到 vectorAdd<<<...>>>(...) 时，把工作提交给 CUDA Runtime/Driver 后即可继续执行；GPU 随后执行这些 Thread。

因此，Kernel launch 下一行 CPU 代码不一定等 GPU 计算结束。如果 CPU 立刻读取 Device 输出或 Managed Memory 输出，必须先建立正确的执行顺序和同步。

~~~cpp
vectorAdd<<<blocks, threads>>>(d_a, d_b, d_c, n);

cudaError_t launchError = cudaGetLastError(); // 检查启动配置或提交错误
cudaError_t runError = cudaDeviceSynchronize(); // 等待此前 GPU 工作完成
~~~

cudaDeviceSynchronize() 在 Host 侧等待当前设备上此前提交的 CUDA 工作完成，并可报告此前异步任务中的错误。[CUDA 异步执行与同步说明](https://docs.nvidia.com/cuda/cuda-programming-guide/02-basics/asynchronous-execution.html)

它和 Kernel 内的 __syncthreads() 不同：

- cudaDeviceSynchronize()：Host 等待设备工作完成，范围是设备此前的工作。
- __syncthreads()：Device 上同一个 Block 内的线程到达屏障后继续执行，不能跨 Block 建立普通的全局屏障。

教学代码中使用设备级同步可以明确看清 CPU/GPU 的先后关系。实际程序中，过多同步会让 CPU 等 GPU、GPU 等 CPU，压缩可并行重叠的空间；后续会学习使用 Stream、Event 和异步拷贝表达更细粒度的依赖。

## 9. Hello CUDA：从一个 Kernel 看执行路径

~~~cpp
#include <cstdio>
#include <cuda_runtime.h>

__global__ void helloCuda() {
    printf("Hello from block %d, thread %d", blockIdx.x, threadIdx.x);
}

int main() {
    helloCuda<<<1, 4>>>();
    cudaDeviceSynchronize();
    return 0;
}
~~~

主机程序调用 helloCuda<<<1, 4>>()，启动一个 Block、其中有 4 个 Thread。四个 Device Thread 都会运行 Kernel 函数，因此会各自执行一次打印。线程调度顺序不固定，输出先后顺序不应作为程序逻辑的一部分。

这个例子把 CUDA 的基本边界显示出来：main 在 CPU Host 上执行，helloCuda 在 GPU Device 上并行执行；三尖括号描述工作配置，而不是普通函数参数。

## 10. 易错点与检查清单

1. **把地址当数据**：float* 是地址，不代表它指向的数组已经被复制到 GPU。
2. **把 Host 指针直接交给普通 Device 内存 Kernel**：std::vector.data() 指向 Host 内存；显式显存模式需先 cudaMemcpyHostToDevice。托管指针是另一种由 CUDA 管理的内存。
3. **拷贝长度写错**：cudaMemcpy 的长度单位是字节，应使用 n * sizeof(T)，不是只传元素个数。
4. **拷贝方向写反**：Host→Device 用 cudaMemcpyHostToDevice；Device→Host 用 cudaMemcpyDeviceToHost。
5. **释放器不匹配**：cudaMalloc 与 cudaMallocManaged 都用 cudaFree，不能用 free 或 delete[]。
6. **Kernel 缺少边界检查**：Grid 的线程槽位数经常大于数据长度，访问数组前检查 i < n。
7. **把 Kernel launch 当成同步调用**：launch 后 GPU 可能仍在计算；Host 读取结果前应等相应工作完成。
8. **混淆两种同步**：cudaDeviceSynchronize 是 Host 等设备工作；__syncthreads 是同一个 Block 内线程同步。
9. **忘记分配输出缓冲区**：Device 输出指针也必须先分配足够字节数。
10. **只检查 launch 错误**：cudaGetLastError 检查启动配置与提交阶段；异步执行错误通常要在同步点检查。
11. **把托管内存理解为“没有传输”**：Unified Memory 简化地址和管理方式，仍需理解页面迁移、访问局部性和同步。
12. **把 cudaDeviceSynchronize 放在所有操作之间**：它会等待设备此前所有 Stream 中的命令，先用它验证正确性，优化时再按依赖选择更细的同步方式。

## 11. 一句话串起 CUDA 向量加法

> CPU 使用 C/C++ 管理输入与流程，通过 nvcc 编译 CUDA 源码；__global__ 定义 GPU Kernel，<<<Grid, Block>>> 指定并行工作规模。Kernel 中每个 Thread 用全局索引负责一部分数组元素，并通过边界检查保证安全。数据可以由 cudaMalloc 和 cudaMemcpy 明确分配、搬运，也可以用 cudaMallocManaged 交给 Unified Memory 管理；Kernel launch 后的同步保证 CPU 在结果可用时再读取。

## 参考资料

- [NVIDIA CUDA C++ Programming Guide：CUDA C++ 入门与 Kernel 启动](https://docs.nvidia.com/cuda/cuda-programming-guide/02-basics/intro-to-cuda-cpp.html)
- [NVIDIA CUDA C++ Programming Guide：异步执行与同步](https://docs.nvidia.com/cuda/cuda-programming-guide/02-basics/asynchronous-execution.html)
- [NVIDIA CUDA Runtime API：内存管理](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__MEMORY.html)
- [NVIDIA CUDA Compiler Driver nvcc：编译流程](https://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/)
- [机械工业出版社：《大规模并行处理器程序设计（原书第 4 版）》目录](https://www.cmpedu.com/books/book/5610306.htm)
- 《CUDA C 编程权威指南》第 2 章：内存管理与 Kernel 启动相关内容
