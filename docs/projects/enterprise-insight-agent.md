# 项目实践：Enterprise Insight Agent

## 1. 项目定位

Enterprise Insight Agent 面向企业研究和竞品分析。用户提交结构化研究目标，系统通过实时网页搜索与抓取获取资料，再经过任务感知证据筛选、来源排序、报告生成和主张审查，形成可追溯的研究结果。

当前项目工作区基于 GPT Researcher 定制代码。主链由 FastAPI 企业 API 和自定义 IntelligenceWorkflow 组织；默认检索资料来自网页，不是预先维护的固定知识库。

## 2. 从研究请求到可审查报告

1. **接收任务**：FastAPI 接收 IntelligenceRequest，Pydantic 约束主题、目标和研究参数。
2. **识别任务类型**：ResearchTaskClassifier 按确定性规则把任务归到事实核验、技术能力分析、竞品比较、趋势研究、冲突可信度判断或企业决策建议等类别。
3. **选定证据策略**：类别映射到 EvidencePolicy，定义偏好的来源类型和来源权威性权重。
4. **搜索和抓取网页**：GPT Researcher 搜集页面，并保留 URL 等来源信息。
5. **切块与语义筛选**：ContextCompressor 以约 1000 字符切块、100 字符 overlap，再使用 EmbeddingsFilter 依据查询相关性筛选候选；默认阈值为 0.35。
6. **来源感知排序**：SourceAwareScorer 计算 `(1 - w) × similarity + w × authority`。权重随任务类别在 0.08 到 0.30 间变化；分类不可用时使用 0.20。语义相关性保持主导，来源质量在相关候选间影响排序。
7. **生成研究报告**：研究器将筛选后的证据与研究任务交给生成流程。
8. **审查报告主张**：启用 V2 evidence execution 时，流程围绕 Writer 草稿规划 claim，整合 claim 与 evidence，经 Claim Gate 和 Grounding Validator 检查支持关系，再根据 findings 形成修复计划并输出审查后的报告。
9. **保存运行记录**：任务状态、来源选择、证据审查和 Trace 随任务保存，客户端可通过 task API 查询。

主要实现位置：

- backend/server/enterprise_api.py
- gpt_researcher/enterprise/workflow.py
- gpt_researcher/enterprise/task_policy.py
- gpt_researcher/context/compression.py
- gpt_researcher/context/source_aware.py
- gpt_researcher/enterprise/trace.py

## 3. 核心设计：任务感知检索与来源重排

同一条网页对不同任务的价值可能不同。事实核验和技术能力分析应更看重官方文档、标准或直接证据；竞品比较还需要保留可比信息；趋势研究则需要兼顾时效和多来源线索。

系统先用研究任务决定来源策略，再按查询语义筛出候选，最后融合相关性与来源权威性。分类改变的是排序策略，不是检索数据库；权威来源也只有在语义筛选通过后才参与排序。公式中的 w 是可解释的来源先验权重，最高为 0.30，因此与 query 的语义相关性仍是主排序信号。

这条设计把“任务类型—来源偏好—候选证据—报告引用”连接起来，也能在 Trace 中复盘分类结果和排序选择。

## 4. Claim-Evidence 与结论审查

生成报告中的结论应能追溯到具体证据。V2 evidence path 将研究证据整理成带来源标识的结构对象，再把报告中的主张与证据建立联系。Claim Gate 负责审查支持关系；Grounding Validator 输出 finding code，修复计划根据这些发现处理缺失支持或证据冲突。

这一流程的价值有三点：

- 报告结论可以定位到支持它的网页证据。
- 审查器可以识别没有充分证据支撑的 claim，减少“引用存在但没有证明结论”的情况。
- Trace 能记录 claim ID、evidence ID、审查状态和修复动作，让调试从最终文本追到具体证据选择。

来源 URL 仍然重要：自动化审查判断报告与采集证据之间的关系，原始网页则供读者确认事实、时间与上下文。

## 5. 项目 Trace 记录什么

ResearchTraceRecorder 将执行记录组织为有界、带类型的事件，并关联 trace_id、run_id 和时间。常见事件包括：

- 研究任务类别、采用的 EvidencePolicy 和来源权重；
- 搜索及检索阶段的状态、耗时和候选/筛选数量；
- 证据的标识与选择分数；
- 报告生成时涉及的 claim ID；
- Claim Gate 和 Grounding Validator 的状态、finding code 与修复信息；
- 研究与报告阶段的结果状态和耗时。

Trace 导出为 outputs/traces/{trace_id}.json。企业项目的 ResearchTrace 主要用结构化事件描述阶段和关系；RunTrace 还记录 research、report、search、retrieval 等阶段耗时。它是项目级诊断轨迹，不应与已经实现完整父子关系的 OpenTelemetry Span 直接等同。概念对照见[Trace、Span 与 Agent 轨迹](../agent/trace-observability.md)。

## 6. 项目评测：业务问题、指标与结果

评测从业务目标开始：研究报告既要覆盖问题，也要让结论可靠、来源正确、执行稳定且成本可接受。项目采用固定的 12 个研究案例，划分为 8 个开发案例和 4 个留出案例；比较来源权重 0.0 的 baseline 与权重 0.2 的 source-aware 版本，结果由人工复核标注支持。

### 6.1 七个维度如何拆解

1. **输出结果质量**：报告结论是否回答研究目标，主张是否清楚、完整、可读。
2. **检索召回完整性**：重要证据是否进入候选和报告上下文；当前基准记录 Evidence Coverage。
3. **证据与来源可靠性**：引用是否指向正确来源，来源内容是否支持对应主张；观察 Citation Correctness、Citation Completeness 和 Unsupported Rate。
4. **工具调用与执行校验**：网页搜索、抓取和研究步骤是否成功，输入输出是否满足预期。
5. **稳定性与异常处理**：批量运行失败、超时、缺少结构化证据等情况是否可识别、是否能形成任务级结果。
6. **成本与效率**：单任务耗时、模型/检索调用成本及其与质量收益的关系。
7. **安全与权限管控**：请求字段、访问范围、密钥和输出内容是否符合服务边界；通过授权、隔离、敏感信息与越权测试检查。

这七维度是完整 Agent 评测的组织框架。已保存的这轮基准主要量化证据覆盖、引用正确性/完整性、无支持引用率、运行失败数、延迟和成本；结果不把未纳入本轮基准的指标写成已测数值。

### 6.2 固定样本上的实际结果

| 指标 | Dev baseline | Dev source-aware | Holdout baseline | Holdout source-aware |
| --- | ---: | ---: | ---: | ---: |
| Evidence Coverage | 100.00% | 100.00% | 100.00% | 100.00% |
| Citation Correctness | 405/482 = 84.02% | 474/506 = 93.68% | 208/219 = 94.98% | 187/209 = 89.47% |
| Citation Completeness | 96.63% | 95.74% | 96.39% | 95.80% |
| Unsupported Rate | 3.37% | 4.26% | 3.61% | 4.20% |
| Operational failures | 0/8 | 0/8 | 0/4 | 0/4 |

开发集上的引用正确率提升 9.65 个百分点；留出集上 source-aware 版本低 5.50 个百分点。这说明来源重排的价值要结合未见案例复测，并按任务类别观察收益，而不能只依据开发集结果下总体结论。该运行也保存了每个案例的延迟和成本，例如 dev 平均耗时由 250.08 秒到 231.58 秒，平均成本由 $4.204980 到 $3.895348；这些是该次模型运行的观测数据。

### 6.3 面试时如何讲评测逻辑

先说明业务风险，再介绍指标：企业研究的核心风险不是“页面有没有搜到”，而是结论是否有证据、引用是否真的支持结论。于是将召回覆盖、引用正确性、完整性和无支持率作为质量主指标，再辅以执行失败、延迟和成本。开发集用于比较策略，留出集观察泛化；分项指标可以揭示整体分数掩盖的变化。

完整七维度题库和答题框架见[Agent 系统评测](../agent/evaluation.md)。

## 7. MCP、API 与项目边界

### MCP

项目可配置 MCPRetriever。MCPClientManager 通过 MultiServerMCPClient 获取已配置 MCP Server 的工具，再由 MCP Retriever 选择相关工具并执行研究。它是可配置的工具接入路径；默认网页研究链路仍需按自身实际工具路径解释。

### FastAPI 与 Pydantic

Enterprise API 在 /api/enterprise/tasks 提供创建、列表和查询接口。FastAPI 暴露 HTTP 服务，Pydantic 模型定义研究请求、结果和任务状态，IntelligenceWorkflow 承担应用级编排。

### RAG、Redis 与 ChromaDB

项目默认研究通过实时网页搜索和内容抓取获得证据。GPT Researcher 代码包含可选向量库路径；默认企业网页主链不是 Redis 会话记忆或 EchoMind 的 Chroma FAQ collection。相关检索细节见[RAG 与语义检索](../agent/rag.md)。

## 8. 面试回答

### 约一分钟项目介绍

> 我做的 Enterprise Insight Agent 面向企业研究和竞品分析。用户提交研究目标后，FastAPI 接收结构化任务，IntelligenceWorkflow 调用 GPT Researcher 做实时网页研究。系统先根据任务类别选择 EvidencePolicy，再对网页块做 Embedding 相关性筛选，并将语义相关性和来源权威性融合排序。选中的来源进入报告生成；V2 evidence path 还会把报告 claim 与 evidence 建立关系，经 Claim Gate 和 Grounding Validator 审查。任务状态、证据选择和审查事件进入 Trace，便于复盘具体结论如何产生。

### 为什么要做任务感知检索？

> 企业研究任务的证据要求并不相同。事实核验需要直接、权威的来源；竞品比较要保留可比信息；趋势研究要考虑时效和来源多样性。系统先分类任务，再选择来源权重和类型偏好，语义相关性筛选仍然是前置条件。这样能把研究目的转成可配置、可追踪的证据策略。

### 如何解释来源权重和评测结果？

> 排序使用 `(1-w) × similarity + w × authority`，w 按任务策略设置为 0.08 到 0.30，fallback 为 0.20。固定 12 案例的对照中，开发集引用正确率从 84.02% 到 93.68%，留出集从 94.98% 到 89.47%。我会同时报告两组结果，并说明开发集收益没有在这组留出样本上复现，因此评测要按任务和独立样本继续分析。

### Claim Gate 解决什么问题？

> 普通报告只看文本是否完整；Claim Gate 进一步检查具体主张是否有对应证据，以及证据是否支持主张。Finding 和修复计划能定位问题 claim，审查结果也进入 Trace。它将报告质量从整体阅读扩展到 claim-evidence 粒度。

### 项目如何评估 Agent？

> 我先从研究业务风险拆指标：证据有没有覆盖、引用是否支持结论、报告是否有无依据主张，再看网页工具执行、失败率、耗时和成本。固定样本按 dev 与 holdout 拆分，策略变体在同一案例集上对照。完整评测还要覆盖异常、权限和安全输入；七个维度的细分指标在系统评测章节里有完整设计。
