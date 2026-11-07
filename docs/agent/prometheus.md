# Prometheus：Agent 服务的指标采集与运行监控

## Prometheus 做什么

Prometheus 是以时间序列方式采集和查询指标的监控系统。应用通过客户端库注册指标，并暴露一个可抓取的 HTTP endpoint；Prometheus 按配置周期拉取指标，按时间保存样本，再用 PromQL 查询趋势、速率和分位数。

典型链路：

**应用埋点 → /metrics 暴露 → Prometheus 定时 scrape → 时间序列查询/告警 → Dashboard 或告警接收器**

Prometheus 指标回答的是“总体运行表现如何”。若要解释某一个请求为什么失败，通常还需要 Trace 和日志。

## 四种常见指标

| 指标类型 | 适合记录 | Agent 服务示例 |
| --- | --- | --- |
| Counter | 累计增加的事件数，进程重启后归零 | 请求总数、错误总数、工具调用次数 |
| Gauge | 可以升也可以降的当前值 | 活跃请求数、队列长度、当前 Agent 成功率 |
| Histogram | 观测值分布，常用可配置桶聚合 | 请求耗时、模型调用时长、检索耗时 |
| Summary | 客户端计算的滑动窗口分位数及总量 | 特定实例内的延迟摘要 |

Counter 通常用 PromQL `rate(counter[5m])` 看单位时间增长速度。Histogram 可通过 `histogram_quantile` 估算 P50/P95 等分位数。服务延迟通常选 Histogram，便于对多个实例聚合。

## EchoMind 里的 Prometheus 链路

配置位于 `config/prometheus.yml`：

- Prometheus 每 15 秒全局采集；EchoMind job 单独配置为每 10 秒采集一次。
- 目标是 Compose 网络内的 `echomind:8000`，metrics path 为 `/metrics`。
- Compose 将 Prometheus UI 暴露在宿主机 `127.0.0.1:9090`。
- FastAPI 的 `GET /metrics` 用 `prometheus_client.generate_latest()` 输出指标格式。
- `PerformanceMonitor` 只有在 `PROMETHEUS_PORT` 为非零时才注册应用自定义指标。

Monitor 初始化自定义指标时定义了：

- `agent_success_rate{agent}`：Gauge，采集每类 Agent 成功率。
- `agent_latency_ms{agent}`：Histogram，采集 Monitor 每轮读取到的 Agent 平均延迟值。
- `tool_success_rate{tool}`：Gauge，采集工具成功率。
- `requests_total`：Counter，在初始化中注册；当前采集循环没有递增调用。

其中 `agent_latency_ms` 当前由每轮 Monitor 的 `avg_ms` 调用 `observe()` 更新，因此直方图的观测粒度是采集周期的平均值。若目标是得到单请求延迟分布，埋点应放在每次请求/工具调用完成时记录原始耗时。

## PerformanceMonitor 和 Prometheus 的分工

EchoMind 的 `PerformanceMonitor` 每 10 秒从 Orchestrator 和 ToolManager 读取运行统计，执行阈值告警、滑动窗口 Z-score 检测、建议生成，并将路由惩罚分数写回 Orchestrator。Agent 成功率低或平均延迟高时，后续路由分数可以降低。

Prometheus 负责把指标暴露并保存成时间序列，便于查看过去的变化和跨实例聚合。它本身不做 Agent 路由决策。`/monitor` 返回应用当前摘要；`/metrics` 返回 Prometheus 可采集的数值样本。

## 标签设计与隐私

Prometheus 每个 label 组合都会产生一个时间序列。适合使用低基数标签，例如 `agent`、`tool`、`status` 和有限枚举的 `route`。不要把 `user_id`、`conv_id`、原始 query、订单号或完整 URL 放进 label；这些值数量会随请求增长，也可能包含敏感数据。

Trace 适合关联单次请求；Metrics 适合聚合很多请求。可在应用日志和 Trace 中保留 request_id 供排查，但不要将 request_id 作为 Prometheus label。

## 与其他观测信号的关系

- **Metrics**：聚合趋势，例如成功率、QPS、延迟分布。
- **Trace**：请求经过的组件、操作层级、每一步的耗时和状态。
- **Logs**：离散事件和错误上下文。
- **Evaluation**：固定样本上的质量表现，例如 Accuracy、Evidence Coverage、Citation Correctness。

可以由指标发现“物流工具 P95 突然上升”，再用 Trace 找到慢在哪个调用，然后检查日志中的依赖错误，最后用固定回归集确认修复没有降低功能质量。

## 面试回答

### Trace 和 Prometheus 都是监控吗？

> 它们都服务于可观测性，但数据粒度不同。Prometheus 记录跨请求聚合的时间序列，比如 Agent 成功率和延迟分布；Trace 展示某一条请求的执行路径和每个操作的状态。EchoMind 的 /monitor 负责读取应用统计并反馈路由权重，/metrics 提供指标样本给 Prometheus 抓取。

### EchoMind 如何暴露 Prometheus 指标？

> FastAPI 的 /metrics endpoint 用 prometheus_client 输出默认 registry；Docker Compose 的 Prometheus job 每 10 秒抓取 echomind:8000/metrics。我在 PerformanceMonitor 中定义 Agent 成功率 Gauge、Agent 延迟 Histogram、工具成功率 Gauge 和请求 Counter，再由采集循环更新 Agent 与工具指标。

### 怎么设计 Agent 延迟监控？

> 将一次用户请求拆成意图识别、检索、模型、工具和报告阶段，分别按请求实际耗时记录 Histogram；用 histogram_quantile 查询 P50/P95，再按稳定的低基数标签查看 agent、工具或路由类型。聚合 P95 能显示尾部变慢，再通过 Trace 定位具体请求路径。

## 参考资料

- [Prometheus metric types](https://prometheus.io/docs/concepts/metric_types/)
- [Prometheus instrumentation practices](https://prometheus.io/docs/practices/instrumentation/)
