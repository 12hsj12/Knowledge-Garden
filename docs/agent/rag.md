# RAG 与语义检索

## 核心概念

RAG（Retrieval-Augmented Generation，检索增强生成）先从外部语料里取回与问题相关的内容，再把内容作为上下文交给模型生成答案。它解决的是模型参数知识之外的资料接入与依据提供问题。

RAG 是一条系统链路，不等于向量数据库：

**资料加载 → 清理和切块 → Embedding → 建立索引 → 查询向量化 → 召回候选 → 元数据过滤 / 重排 → 组装上下文 → 模型生成**

知识库可以是 FAQ、产品手册、项目文档或网页。检索可以使用关键词、向量或混合方式；向量库只是保存向量和做近邻查询的一种方案。

## Embedding 与语义匹配

Embedding 模型把文本编码为固定维度的向量。文档块和用户 query 使用兼容的 Embedding 模型映射到向量空间，向量检索再按配置的距离度量找近邻。

语义近邻表示模型认为文本表达或主题相似，不代表事实正确，也不保证完全回答用户的问题。检索质量受语料质量、语言和领域覆盖、块大小、重叠、Embedding 模型、距离度量、Top-K 和后续重排影响。

常见的两阶段召回：

1. **召回阶段**：用关键词或向量搜索，从大量文本中取出较小的候选集合。
2. **排序阶段**：使用更强的相关性模型、规则或 LLM 对候选重新排序，再把少量高相关内容放进上下文。

Top-K 是候选数量，不是相关性阈值。若没有最低分数或拒答逻辑，低相关结果也可能被送入模型，因此要通过检索样本评测决定阈值和参数。

## EchoMind 中的 RAG 知识库

知识库实现位于 mcp/knowledge_base.py，API 与工具注册在 api/main.py、agents/tools.py 和 mcp/tool_manager.py。

### 文档写入

1. 文档以 title 和 content 输入。
2. 长文本按中文句号与换行切分，目标长度约 500 个字符。
3. 每个 chunk 保存 title、chunk_index、total_chunks 元数据。
4. ChromaDB collection 接收 documents，并由 collection 使用的默认 Embedding function 生成向量。
5. 写入使用稳定 ID 和 upsert，使重复导入可以更新相同记录。

该切分器是句子累计式实现，不使用 overlap，也不是按 token 数控制。若一整句超过目标长度，单块可能超过 500 个字符。切块策略要结合真实资料结构与召回评测迭代。

### 查询与回答

1. Agent 使用 knowledge_search 工具检索知识库。
2. ChromaDB 将 query 向量化，在 knowledge_base collection 做近邻检索，默认 Top-K 为 5。
3. EchoMind 的搜索链路可以把问题改写成多个角度的子查询，并行召回后合并去重。
4. 合并结果可由 LLM reranker 再排序，取 Top-K 返回 Agent。
5. 工具结果作为 Observation 返回模型，模型基于检索内容生成回复。

查询改写扩大可能的召回角度；重排优化候选先后顺序。二者都不能让错误语料变正确，也需要监控检索命中与最终答复。

### 距离度量与分数解释

EchoMind 在创建知识库 collection 时没有显式设置距离度量。Chroma 新建 collection 默认使用 L2 距离；项目代码注释写“余弦相似度”，并计算 1 - distance 作为 score。L2 距离不等同余弦距离，因此该 score 不能直接理解为经过标定的 0 到 1 相似度或置信度。

如果分数要用于阈值判断，应显式确定 collection 距离度量，并使用与它一致的分数解释；也可以保留原始距离，只把它当成距离值。Collection 已建好后切换索引距离通常需要新建或迁移 collection，不能仅改一行注释。

项目依赖中使用 Chroma 默认 all-MiniLM-L6-v2 Embedding。客服语料以中文为主，是否适配要用中文 query 和标注答案集评测，不能只因为 collection 成功建成就认定语义效果足够好。

## Enterprise Insight Agent 中的检索

Enterprise Insight Agent 的主流程与 FAQ RAG 不同。它默认从网页搜索和页面抓取开始，把网页内容压缩成可用研究上下文，并保留 URL 等证据来源。

当前任务感知检索链路为：

1. ResearchTaskClassifier 根据问题中的确定性信号识别研究任务类别。
2. 系统从固定 EvidencePolicy 中取该类别的证据策略，并确定来源权威性权重。
3. ContextCompressor 将网页候选按语义相关性筛选。长页面约按 1000 字切块、100 字重叠；默认回退相关性门槛为 0.35。
4. SourceAwareScorer 将语义相似度与来源权威分数组合：最终分数 = (1 - w) × 语义相似度 + w × 来源权威分。当前 w 不超过 0.30，因此语义相关性仍然占主要权重。
5. 筛出的页面形成带 URL 的结构化证据，进入报告和证据检查。

因此项目一的“任务感知”主要调节来源可靠性在排序中的影响，不是换成一套 Chroma 知识库。GPT Researcher 代码也支持可选向量库路径，那个路径采用约 1000 字 chunk、200 字 overlap；它与默认企业网页研究路径需要分开说明。

## 与 Agent、记忆和工具的协作

RAG 负责准备与当前问题有关的外部上下文；ReAct 决定是否调用检索工具以及检索后还要做什么；工具层负责执行与错误处理；MCP 可能提供跨服务的检索工具接入；Trace 记录查询、返回结果和执行耗时。

EchoMind 的短期工作记忆不是 RAG：Redis 按 session key 读取最近消息。ChromaDB 的 episodic collection 用 query 检索历史摘要；knowledge_base collection 则检索客服知识。两者都可能把文本召回模型，但语料和用途不同。

## 面试追问与回答

### RAG 一次查询经历哪些步骤？

> 原始资料先清理、切块并保存必要元数据，再使用 Embedding 编码并写进索引。查询时对问题做同类向量编码，先召回候选片段，再做元数据过滤或重排，最后把有用上下文交给模型生成答案。检索结果的相关性和最终回答都需要评测。

### Embedding、ChromaDB 和 RAG 的关系是什么？

> Embedding 模型负责把文本转为向量；ChromaDB 保存文档、向量和元数据并提供近邻查询；RAG 是把这些能力接入回答生成的整体流程。任何一个组件都不能单独代表完整 RAG。

### 项目二怎么做语义匹配？

> 知识库写入 Chroma 的 documents，collection 的默认 Embedding function 生成文档向量；查询时 Chroma 对 query 做同一类向量编码并按 collection 的距离度量召回 Top-K。之后工具管理层可以做多查询改写、并行召回、去重和 LLM 重排。当前实现还需要核实距离分数的解释，检索效果也要用中文客服问题集评估。

### 项目一的任务感知检索体现在哪里？

> 它先把研究任务分到不同类别，再按任务选择来源权威性策略；网页块仍先按 Embedding 相关度筛选，之后再把相关性和来源权威性合并排序。它不是默认使用 Redis 或 Chroma 的客服知识库检索链路。