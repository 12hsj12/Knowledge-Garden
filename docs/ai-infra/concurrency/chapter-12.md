# 并发基础：第 12 章 异步通信

第 11 章介绍了事件循环怎样等待 I/O 事件并调用回调。本章把这种执行方式组织成更自然的程序流程：**协程在等待时主动让出执行权，事件循环运行其他任务，结果就绪后再恢复协程。**

主线是：**异步调用 → 协作式多任务 → 协程 → Future/Task → Event loop**。

## 1. 异步调用为什么有用

一个请求处理过程可能依次访问网络服务、数据库和文件系统。若每次等待都占住一个线程，服务要么创建很多线程，要么受线程池容量限制。

**异步调用（asynchronous procedure call）**让调用方发起操作后不必一直占住当前执行资源。等待期间，运行时可以安排其他工作；操作完成后，再把结果交还给发起任务。

异步首先解决“等待时如何让其他任务运行”，不表示代码自动在另一个 CPU 核心并行执行。CPU 密集计算仍需要多进程、线程池、GPU 等计算资源。

## 2. Cooperative multitasking：任务主动让出控制权

**协作式多任务（cooperative multitasking）**由当前任务在合适的时机主动让出控制权。网络读取、定时器等待等 await 点就是常见交接位置：

~~~text
任务 A 发起网络请求
→ await 等待响应，暂时挂起
→ 事件循环运行任务 B
→ B 也等待数据库，循环继续处理其他任务
→ A 的网络结果到达，任务 A 从等待点恢复
~~~

若协程长时间不等待、不主动让出，它会占着事件循环线程，其他协程就无法及时获得执行机会。

## 3. Coroutine：可以暂停并恢复的计算过程

**协程（coroutine）**是可以暂停并从暂停点恢复的计算过程。Python 使用 async def 声明协程函数，调用后得到协程对象；仅仅调用函数并不会自动执行它。

~~~python
import asyncio

async def fetch_data():
    await asyncio.sleep(1)
    return "完成"

async def main():
    result = await fetch_data()
    print(result)

asyncio.run(main())
~~~

async def 声明协程函数，调用 fetch_data() 得到协程对象，await 等待结果。若其内部需要等待，当前协程会挂起；asyncio.run() 运行顶层协程和事件循环。

Python 官方文档区分协程函数与协程对象；要让协程运行，需要 await 它或将它交给 Task 调度。参见 [Coroutines and Tasks 文档](https://docs.python.org/3.14/library/asyncio-task.html)。

## 4. Event loop：协程任务的调度者

事件循环运行就绪任务、等待 I/O 或定时器，并在结果准备好后恢复对应任务。一个事件循环线程通常一次运行一个 Task；Task 在 await 等待时让出，循环才能调度其他就绪 Task。

~~~text
事件循环取出就绪任务
→ 运行协程，直到结束或遇到需要等待的 await
→ 登记等待条件，切换到其他就绪任务
→ I/O 或定时器完成后，将 Task 标记为可继续
→ 再次运行该 Task
~~~

这种调度是协作式的，不会在任意指令处自动抢占当前协程。同步网络请求、长时间 CPU 循环或阻塞文件操作若运行在事件循环线程中，会挡住其他任务。

## 5. Future 与 Task：表示工作和未来的结果

**Future**代表一个尚未完成、之后会产生结果或异常的操作。程序可以等待它；操作完成后，Future 保存结果或异常。

Python 的 **Task** 将协程交给事件循环调度，也承载它最终的执行结果：

- **Coroutine**描述要执行的异步流程；
- **Task**表示这个协程已被安排执行；
- **Future**表示将来会完成的结果。

创建协程对象本身不等于开始并发执行。要让多个协程交错推进，可以同时等待它们，或使用 asyncio.create_task() 创建 Task。现代 Python 还提供 TaskGroup 管理一组相关任务的生命周期和异常。

## 6. 多个 I/O 请求在等待期间交错推进

例如服务要请求多个独立数据源，再汇总结果：

~~~python
import asyncio

async def fetch_one(source):
    return await request_source(source)

async def main():
    sources = ["catalog", "profile", "inventory"]
    results = await asyncio.gather(
        *(fetch_one(source) for source in sources)
    )
    return merge(results)

asyncio.run(main())
~~~

每个协程等待各自 I/O 时，事件循环可以推进其他协程。它们是否同时在多个 CPU 核心执行，是另一个问题；asyncio 主要通过重叠 I/O 等待提高并发效率。

如果 request_source 内部调用阻塞版网络库，事件循环仍会被阻塞。完整异步链路要使用异步 I/O 接口，或把阻塞工作交给线程池等执行器。

## 7. 异步披萨店：保留顺序流程，同时释放等待时间

披萨订单提交给厨房后，服务员无需一直站着等。厨房制作期间，服务员可以接收其他订单；披萨完成后再回到原订单通知顾客。

~~~python
async def serve_order(order):
    ticket = await send_to_kitchen(order)
    pizza = await wait_for_ready(ticket)
    await notify_customer(order.customer, pizza)
~~~

协程把流程写成接近顺序逻辑的样子；在 wait_for_ready 等待期间，事件循环可以运行其他订单。底层仍依赖事件循环处理任务挂起、I/O 完成和恢复。

## 8. 异步模型的边界和工程习惯

- **await 点提供协作机会**：长时间不等待的计算会占住循环。
- **区分等待与计算**：异步 I/O 提高等待时的资源利用率，不自动加速 CPU 密集任务。
- **调用链要避免阻塞**：在异步函数中调用同步网络库，会把阻塞带回事件循环。
- **明确任务生命周期**：谁创建、谁等待、异常如何传播、取消时如何清理资源，都要有约定。
- **管理任务与异常**：未跟踪的后台任务可能让错误难以及时发现。
- **控制并发边界**：同时创建过多任务会压满内存、连接和下游服务；可用信号量、队列或批次限制并发。

## 9. 本章小结

- 异步通信让任务发起 I/O 后先处理其他工作，完成后再继续。
- 协程可以暂停并恢复，await 是异步控制流的重要交接点。
- 事件循环在一个线程中运行就绪 Task；协程等待时主动让出执行权。
- Future 表示未来结果，Task 把协程交给事件循环调度。
- I/O 等待可以重叠推进，这提供并发，但不等于 CPU 多核并行。
- 阻塞调用会占住事件循环；异步程序还要管理并发上限、错误和取消。

### 面试表达：async/await 怎样提高 I/O 并发能力？

> async/await 把异步流程写成可暂停和恢复的协程。协程遇到网络等待等 await 点时会挂起并把控制权交给事件循环；循环可以运行其他就绪任务。I/O 完成后，原协程再从等待点继续。因此少量线程可以管理许多 I/O 等待中的任务，减少为每个请求分配线程的需要。它主要提升 I/O 密集场景的并发效率；阻塞调用会卡住事件循环，CPU 密集计算应交给适合的并行执行资源。

### 复习时可以自问

1. 调用 async 函数后，协程是否一定立即执行？
2. Coroutine、Task、Future 分别表示什么？
3. 事件循环在一个线程里怎样推进多个 Task？
4. await 为什么能让其他任务先运行？
5. 多个 I/O 等待可以重叠，为什么不等于多核并行？
6. 同步数据库调用放在 async 函数里会发生什么？
7. 为什么要设置并发上限并管理任务异常和取消？

### 参考资料

- Kirill Bobrov, [Grokking Concurrency：第 12 章大纲](https://livebook.manning.com/learn/grokking-concurrency)
- Python 文档：[Coroutines and Tasks](https://docs.python.org/3.14/library/asyncio-task.html)
- [上一章：事件驱动并发](chapter-11.md) · [下一章：编写并发应用](chapter-13.md)
