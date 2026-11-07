# Agent 系统评测：质量、证据与运行链路

## 为什么 Agent 评测要看全链路

Agent 的最终回答由一串步骤共同决定：请求理解、检索、工具选择、工具执行、上下文组装、生成、证据检查和服务返回。只给最终文字打一个分，会把“没找到资料”“找错工具”“工具失败”“生成时编造”等不同故障混在一起。

一次可复现评测需要固定任务集、样本标签、代码版本、模型与参数、检索配置和运行环境；同一组请求分别运行 baseline 与候选版本，保留逐样本输出、Trace 和聚合指标。分类指标由程序依据 gold label 计算；开放式回答质量可由 rubric 评分、LLM-as-Judge 辅助，再抽取人工复核。

## 七个评测维度

| 维度 | 要回答的业务问题 | 常用指标和证据 |
| --- | --- | --- |
| 输出质量 | 回答是否有用、准确、完整，并覆盖用户真正的目标？ | Rubric 分项得分、人工偏好、解决率、事实错误率、拒答质量 |
| RAG 召回完整性 | 必需的证据或知识片段是否被找回？ | Recall@K、Hit Rate@K、MRR、nDCG、Required Unit Evidence Coverage |
| 证据可靠性 | 引用是否支持对应主张，来源是否适合该任务？ | Citation Correctness、Citation Completeness、Unsupported Claim Rate、来源类型/独立性检查 |
| 工具调用正确性 | Agent 是否选对工具、参数正确、结果可用并安全处理失败？ | 工具选择准确率、参数校验通过率、执行成功率、拒绝非法调用数、升级 Precision/Recall |
| 稳定性与异常处理 | 请求是否完成，超时、模型失败或依赖故障时是否受控？ | 完成率、错误率、超时率、fallback 率、P50/P95 延迟、重复请求与恢复结果 |
| 成本与效率 | 质量是否以合理的时间、模型调用和检索成本达成？ | 输入/输出 tokens、单任务估算成本、工具调用数、搜索数、P50/P95 延迟、缓存命中率 |
| 安全与权限 | 是否只访问获准工具和数据，是否保护敏感内容？ | 白名单/越权调用测试、跨用户隔离测试、敏感信息泄漏测试、提示注入和 URL 安全用例 |

七个维度共享一条评测逻辑：先定义真实业务成功条件，再把条件映射到可观测步骤、样本标签和指标。最终报告保留分项结果和失败案例，不把它们压成一个看不出业务取舍的总分。

## 两个项目各自的评测重点

### Enterprise Insight Agent：研究证据与报告主张

项目一使用固定的 12 个研究案例，划分为 8 个 development 和 4 个 holdout。对比同一代码版本下的 baseline（来源权重 0）和 source-aware（来源权重 0.2）；人工审核标注包含 Required Unit 覆盖、引用正确性、引用完整度和无支持主张率。

主要结果：

- Required Unit Evidence Coverage：development 和 holdout 两组中两个版本均为 100%。
- development 的 Citation Correctness：baseline 为 405/482 = 84.02%，source-aware 为 474/506 = 93.68%，提高 9.65 个百分点。
- holdout 的 Citation Correctness：baseline 为 208/219 = 94.98%，source-aware 为 187/209 = 89.47%。开发集的提升没有在 holdout 重现。
- Citation Completeness：source-aware 在 development 为 95.74%，baseline 为 96.63%；holdout 为 95.80%，baseline 为 96.39%。
- Unsupported Claim Rate 越低越好。source-aware 在 development 为 4.26%、holdout 为 4.20%；对应 baseline 为 3.37% 和 3.61%。

业务解释：来源权重能改变引用质量，但“来源更权威”不自动代表“答案里的每个主张都被支持”。报告要同时看覆盖、引用正确性、引用完整度和无支持主张率；dev/holdout 差异说明选权重时要把泛化结果纳入决策。面试时可讲成一轮有明确对照、指标口径和取舍的证据质量消融实验。

本机评测报告位于 `gpt-researcher/benchmarks/results/v1/final_quality_comparison.md`。其中同一组 live runs 还记录搜索次数、耗时、估算费用与失败数；这次 8 案例 development 两组都完成 8/8，4 案例 holdout 两组都完成 4/4。运行耗时和费用是执行记录，报告没有把它们解释成来源权重造成的因果提升。

### EchoMind：意图、路由和物流工具

EchoMind 的 `data/eval/g4_intents_v1.json` 是固定评测集。评测报告保存数据集 SHA-256、样本分布、代码版本、逐样本预测和配置，便于复查同一批样本。分类、路由和物流工具分别统计，不混成一个“准确率”。

G4 记录的关键结果：

| 版本 | 旧意图标签 | 五类物流细标签 | Agent 路由 |
| --- | --- | --- | --- |
| baseline | 31/36 = 86.11%，Macro-F1 0.9099 | 没有细分物流标签 | 37/38 = 97.37% |
| logistics | 33/36 = 91.67%，Macro-F1 0.9186 | 13/15 = 86.67%，Macro-F1 0.8714 | 36/38 = 94.74% |
| local Mapika provider | 33/36 = 91.67%，Macro-F1 0.8788 | 14/15 = 93.33%，Macro-F1 0.9314 | 37/38 = 97.37%，必需 Agent 覆盖 38/38 |

Logistics 变体还用 5 个本地 mock 工具样例验证物流查询成功/失败条件，期望结果匹配率为 5/5；其中 3 个升级候选样例的候选判定为 3/3。它验证的是定义好的本地规则和工具契约。

之后的 OpenJev paired run 是另一组模型对比，不能和 G4 的 Mapika local provider 混称同一个版本。在保存的 `openjev_paired_20261026T024116143097Z.json` 中：

- common-label scope：baseline 32/36 = 88.89%，OpenJev 34/36 = 94.44%；Macro-F1 从 0.9065 到 0.9394。
- 五类物流细标签：baseline 13/15 = 86.67%，OpenJev 15/15 = 100%；Macro-F1 从 0.8714 到 1.0000。
- 固定数据集共 38 个样例。运行中 OpenJev 被调用 6 次，其中 6 次采用其决策、32 次按门控跳过、fallback 为 0。
- baseline 的 P95 意图延迟为 2442 ms，OpenJev 增强版本为 7150 ms；决策调用自身 P95 为 5636 ms。

这组结果表达的是：在当前固定样例里，OpenJev 改善了两个标签空间的分类结果，同时对触发决策的请求增加了延迟。报告文件保存逐条结果和数据集哈希，准确率应连同样本范围和模型版本一起描述。

## 如何把评测接到研发闭环

1. **定义任务成功**：例如物流纠纷不只要求意图正确，还要路由到 LogisticsAgent、正确抽取运单号、遵守升级条件。
2. **冻结样本和 gold label**：保存数据集版本或 SHA；按风险和业务分层，覆盖普通、歧义、复合、缺失信息、未知请求与异常依赖。
3. **确定 baseline**：同一批输入、同一运行参数，记录运行前后的逐样本行为。
4. **采集全链路证据**：保存路由决策、检索结果 ID、工具输入输出状态、时延、异常状态和最终答案评分。
5. **按维度计算指标**：Accuracy/Macro-F1 衡量分类；Recall@K 衡量召回；Citation Correctness 衡量引用支持；P95 衡量尾延迟。
6. **分析失败样本**：找到错误发生在意图、召回、来源排序、工具执行、证据绑定还是生成阶段。
7. **验证改动收益和代价**：在 development 上提出改进，在 holdout 上确认表现；把质量、延迟、成本与稳定性并列比较。

## 面试回答：你怎么评测 Agent 项目？

> 我按输出质量、检索覆盖、证据可靠性、工具正确性、稳定性、成本效率和安全权限七个维度组织评测。EchoMind 用固定意图集对 baseline、物流扩展和决策增强版本做同集比较，记录 Accuracy、Macro-F1、路由覆盖、工具规则结果和 P50/P95；OpenJev 的配对报告还记录调用与回退状态。Enterprise Insight Agent 用 12 个固定研究案例比较来源权重消融，分别检查证据覆盖、引用正确性、引用完整度和无支持主张率，并按 development/holdout 观察差异。我的判断方式是先看业务目标是否达到，再通过 Trace 找到指标变化对应的链路环节，最后一起权衡质量、延迟和成本。

### 面试追问：为什么同时看 Accuracy 和 Macro-F1？

> Accuracy 表示全部样本里预测正确的比例；Macro-F1 先分别计算每个类别的 F1，再对类别平均。类别数量差异大时，Macro-F1 能显示少数类是否被模型忽略。EchoMind 的类别样本很少且不均衡，所以我会同时报告样本数、Accuracy、Macro-F1 和逐类结果。

### 面试追问：LLM-as-Judge 可以代替人工标注吗？

> LLM Judge 适合按固定 rubric 对开放式回答做规模化初筛，例如相关性、准确性、完整性和帮助性。意图 Accuracy、引用是否支持主张等需要明确的 gold label 或审核规则，最终由可复现脚本或人工审核确认。把 Judge 版本、prompt 和抽样复核结果一起保存，才能解释评分变化。

### 面试追问：样本只有几十条，能说明什么？

> 它能说明系统在这组固定业务用例上的行为，并支持开发版本间的逐样本对照。结论要带上样本范围、标签分布和数据版本；扩大样本、增加时间切片和真实用户案例，可以提升对更多使用场景的覆盖。

## 指标公式速记

- `Accuracy = 正确预测数 / 总样本数`
- `Precision = TP / (TP + FP)`
- `Recall = TP / (TP + FN)`
- `F1 = 2 × Precision × Recall / (Precision + Recall)`
- `Macro-F1 = 每个类别 F1 的算术平均`
- `Recall@K = Top-K 命中的相关项数 / 所有相关项数`
- `Citation Correctness = 支持性引用数 / 被评估引用数`
- `Citation Completeness = 有证据的事实主张数 / 全部事实主张数`
- `Unsupported Claim Rate = 无支持事实主张数 / 全部事实主张数`

一个指标只回答它的定义范围。检索命中率不能代表最终回答正确率，工具成功率也不能代表业务任务已经解决。

## 参考资料

- [LangSmith: Agent evaluation concepts](https://docs.langchain.com/langsmith/evaluation-concepts)
- [Prometheus: Instrumentation practices](https://prometheus.io/docs/practices/instrumentation/)
