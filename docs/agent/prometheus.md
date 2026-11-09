# Prometheus：把服务运行情况变成看得见的数字

## 先用日常例子理解

想象你经营一个客服柜台。你想知道今天来了多少人、现在排队多少人、每个人平均等多久。只看某一个顾客的服务过程，回答不了全天总体情况；只看全天平均数，也不知道某一位顾客为什么等了很久。

Prometheus 用来持续记录和查询这类运行数字。它通常按配置好的间隔，去应用提供的 /metrics 页面读取一份数字清单，并把每次读取到的数值按时间保存下来。它更像一个会定时抄表、保存历史并支持查询的监控系统；它不会自动猜出应用里所有业务指标，应用需要自己把关心的数字统计好并暴露出来。

**应用统计数字 → /metrics 页面公开数字 → Prometheus 定时读取 → 查询变化趋势或触发告警**

## 通过一个 Agent 请求看流程

假设用户向 EchoMind 发了一条请求。应用可以统计：

- 请求数量增加了 1。
- 本次请求处理了多少秒。
- 本次调用 Agent 是否成功。
- 使用了哪个工具，以及工具是否成功。

应用把当前统计结果放在 /metrics。Prometheus 每隔一段时间读取它。例如某次读取看到请求累计数是 100，下一次看到 125，就可以推算这段时间增加了多少请求、平均每秒增加多少。

因此 /metrics 是应用提供的“数字窗口”；Prometheus 是按时间读取并保存这些数字的服务；PromQL 是查询历史数据的语言；Prometheus 自带的页面可以直接运行查询并画出趋势。

## 四种常见指标，用柜台比喻记

| 指标 | 日常比喻 | Agent 服务里的例子 |
| --- | --- | --- |
| Counter（计数器） | 汽车里程表，只累计增加 | 累计请求数、累计错误数、工具调用次数 |
| Gauge（仪表读数） | 温度计，会升也会降 | 当前并发请求数、当前队列长度 |
| Histogram（直方图） | 把每次等待时间放进不同区间 | 请求耗时、模型调用耗时、检索耗时 |
| Summary（摘要） | 在应用端计算一段时间内的观测摘要 | 特定服务实例上的延迟摘要 |

如果只想判断“最近请求变多了吗”，看 Counter 的变化；如果想判断“现在有多少请求正在处理”，看 Gauge；如果想判断“慢请求占比有没有增加”，看 Histogram 的耗时分布。

Histogram 像把每次耗时放入一组区间，例如小于 0.5 秒、1 秒、2 秒。P95 可以理解为：从耗时分布看，约 95% 的请求不超过这个时间。它帮助观察慢请求的尾部表现，不只看容易掩盖问题的平均值。

## 一个极简埋点例子

下面的代码说明应用如何统计请求次数和耗时。它是通用示例，名字和埋点位置不代表 EchoMind 当前代码里的具体实现。

~~~python
from prometheus_client import Counter, Histogram

chat_requests = Counter(
    "chat_requests_total",
    "完成的聊天请求数",
    ["status"],
)
chat_duration = Histogram(
    "chat_request_duration_seconds",
    "聊天请求处理时长",
)

def handle_chat():
    with chat_duration.time():
        # 在这里执行一次聊天请求
        result = "回答完成"

    chat_requests.labels(status="success").inc()
    return result
~~~

Prometheus 抓取时会读到 Counter 当前累计值和 Histogram 当前的分布。可以在 Prometheus 查询页面输入：

~~~promql
rate(chat_requests_total[5m])
~~~

它表示查看最近 5 分钟请求计数的平均增长速度，结果按“每秒请求数”理解。对 Histogram 也可以查询 P95；要理解这类查询，先记住它是根据已记录的耗时区间估计尾部延迟。

## Prometheus 和 Trace 怎么配合

可以把一群请求想成一天走进柜台的所有顾客：

- **Metrics（指标）**回答总体情况：“过去 5 分钟请求数是多少？”“物流工具最近是不是普遍变慢了？”
- **Trace（链路追踪）**回答单次过程：“这位顾客的请求先经过意图识别、再检索、然后调用模型；究竟哪一步花了 8 秒？”
- **Logs（日志）**提供离散事件的文字细节，例如这次工具调用返回了超时异常。

一个典型排查过程是：

1. Prometheus 图表显示最近的请求 P95 上升。
2. 工程师从同一时间段找一条慢请求的 Trace。
3. Trace 显示该次请求卡在 Redis、检索、模型或工具调用中的某一步。
4. 再看对应日志中的错误上下文，定位原因。

Prometheus 不需要 Trace 才能采集或计算指标；Trace 也不负责生成 Prometheus 的指标图表。它们可以在仪表盘和排查流程里互相补充。request_id 适合放在 Trace 或日志中，用来定位某一次请求；不要把每个 request_id 都作为 Prometheus 标签，否则每个用户请求都会产生新的时间序列。

## EchoMind 当前的监控链路

### Prometheus 服务怎么读到 EchoMind 的指标

- 配置文件在 config/prometheus.yml。
- 全局采集间隔是 15 秒，EchoMind job 单独设置为每 10 秒抓取。
- 抓取目标是 Compose 网络中的 echomind:8000，读取路径为 /metrics。
- FastAPI 的 GET /metrics 使用 prometheus_client.generate_latest() 输出可采集的数据。
- Prometheus 页面从宿主机的 127.0.0.1:9090 访问。

这里的 echomind 是 Compose 服务名。Prometheus 和 EchoMind 位于同一 Compose 网络时，可以用服务名互相寻找，不需要填容器的动态 IP。

### PerformanceMonitor 做什么

EchoMind 还有一段应用内的 PerformanceMonitor。它每 10 秒从 Orchestrator 和 ToolManager 读取运行统计，做阈值告警、滑动窗口 Z-score 异常检测和建议生成，并可能把路由惩罚分数写回 Orchestrator。

这和 Prometheus 有分工：

- PerformanceMonitor 读取应用统计并参与应用内部的监控判断和路由反馈。
- Prometheus 按时间抓取 /metrics，保存可查询的历史指标。
- /monitor 返回应用当前监控摘要；/metrics 提供 Prometheus 要读取的指标格式。

因此 Prometheus 不是直接控制 Agent 路由的模块；项目里的路由反馈由 PerformanceMonitor 相关代码完成。

### 当前定义的几项指标

PerformanceMonitor 在 PROMETHEUS_PORT 非零时注册应用自定义指标：

- agent_success_rate{agent}：Gauge，记录每类 Agent 的成功率。
- agent_latency_ms{agent}：Histogram，记录 Monitor 每轮采集到的 Agent 平均延迟。
- tool_success_rate{tool}：Gauge，记录工具成功率。
- requests_total：Counter 已注册；当前采集循环里没有递增它的代码。

这里有一个理解指标时要留意的细节：agent_latency_ms 当前记录的是 Monitor 每轮读取到的平均延迟，并非每个用户请求的原始耗时。因此它可以反映各次采集得到的延迟读数，但不能直接当成完整的单请求延迟分布来解释 P95。若要观察真实请求 P95，应在每次请求或工具调用结束时记录实际耗时。

## 指标标签怎么选

标签用来把一类指标再按固定维度拆开。例如按 agent、tool、status 或少数固定的 route 分类，就能比较不同 Agent 和工具的情况。

标签的取值最好有限而稳定。user_id、conv_id、完整 query、订单号和 request_id 都可能不断出现新值；逐个放入标签会产生大量独立时间序列，并且可能暴露用户信息。单次请求的细节交给 Trace 和日志记录更合适。

## 面试回答

### Prometheus 是什么？

> Prometheus 是一个开源监控系统。应用把自己关心的运行数字通过 /metrics 暴露出来，Prometheus 按配置周期读取并保存成带时间的数据，再用 PromQL 查询趋势或配置告警。它适合看一段时间内服务整体的请求量、成功率和耗时分布。

### Prometheus 和 Trace 的关系是什么？

> Prometheus 看很多请求汇总后的趋势，例如请求率和 P95 延迟；Trace 还原一条请求经过哪些步骤、每一步花了多久。Prometheus 的指标发现“最近普遍变慢”，Trace 帮我定位某次慢请求卡在哪一步，两者从整体趋势和单次执行过程互补。

### EchoMind 里怎么用 Prometheus？

> FastAPI 通过 /metrics 暴露 prometheus_client 指标，Compose 中的 Prometheus 每 10 秒抓取 echomind:8000/metrics。应用的 PerformanceMonitor 维护 Agent 成功率、平均延迟和工具成功率等统计，并通过应用内部逻辑做阈值判断、异常检测和路由反馈；Prometheus 负责保存和查询这些指标随时间的变化。

## 参考资料

- [Prometheus First Steps：采集目标、周期和查询](https://prometheus.io/docs/introduction/first_steps/)
- [Prometheus 指标类型：Counter、Gauge、Histogram、Summary](https://prometheus.io/docs/concepts/metric_types/)
- [Prometheus Overview](https://prometheus.io/docs/introduction/overview/)
