# Memory Core R4.1：五大核心算法实施规格

> 生成：2026-11-04 ｜ 维护位置：`E:\[PROJECT]\work-交付包\14-MemoryCore-R4.1算法实施规格.md`
> 地位：**用户定稿的统一算法实施基线，算法实现以本文为准**。默认值是工程配置，不代表改变既有设计裁决；如果与正式裁决台账（`08-adjudication-ledger.md`）冲突，以台账为准。代码 AI 不得自行改变算法方向或静默降级。
> 说明：章节编号（0～5、十一～十四）沿用原稿；本文不改变 R4.1 的模块职责，也不自动扩大 P0–P3 的授权范围。

这份规格以我们已经确定的架构为基础：SQLite 是权威存储，FTS5 负责全文检索，LanceDB 负责向量检索，FLAT 负责精确向量召回，RRF 负责多路融合，首版使用规则排序，不使用 ANN 和 Cross-Encoder。

必要的实现参数以默认值一并给出，让算法可以落地。

---

## 0. 总体设计与数据流

五个算法不是五个互相独立的功能，而是一个完整的记忆检索生命周期：

1. 多路召回算法：从不同检索通道找出可能相关的记忆。
2. 融合排序算法：合并候选，计算综合排名。
3. 上下文构建算法：经过权限检查，在 Token 预算内选择记忆并组织 Prompt。
4. 索引构建与同步算法：确保 SQLite、FTS5、LanceDB 的数据一致。
5. 检索评估与反馈算法：用可重复的数据和指标验证检索质量，定位问题。

完整算法执行链：

```text
当前消息 + 最近对话上下文
        │
        ▼
算法一：查询处理与多路召回
（SQLite 过滤 · FTS5 · Embedding · LanceDB FLAT）
        │
        ▼
算法二：候选融合与排序
（标准化 · 去重 · RRF · 规则重排 · 筛选）
        │
        ▼
算法三：权限终检与上下文构建
（SQLite 回源 · M05 canRead · Token 预算 · Prompt 组织）
        │
        ▼
最终记忆上下文 ──→ 交给 DSH / 宿主 LLM

算法四：索引构建与同步（SQLite → Outbox → FTS5 / LanceDB）
算法五：检索评估与质量反馈（测试集 → 指标 → 回归测试 → 参数版本管理）
```

所有算法共享三个基础约束：

- SQLite 是记忆内容和业务状态的唯一权威来源。
- FTS5、LanceDB 都是可重建的派生索引。
- 任何记忆在进入最终 Prompt 之前，都必须通过 M05 权限终检。

---

## 算法一：查询处理与多路召回算法

目标：尽可能完整地找到与当前查询相关的记忆，不让某一种检索方式成为唯一入口。

### 1.1 输入与输出

输入：

```text
RetrievalRequest
├── query_id
├── current_message
├── recent_context
├── requester_id
├── conversation_id
├── group_id
├── timestamp
├── scope
└── retrieval_options
```

输出：

```text
RecallResult
├── query_id
├── query_features
├── structured_candidates[]
├── fts_candidates[]
├── vector_candidates[]
├── embedding_profile
└── retrieval_trace
```

所有候选都必须使用稳定的 memory_id 标识。

### 1.2 查询预处理

执行顺序：

```text
原始查询
  ↓
Unicode 规范化
  ↓
空白与标点规范化
  ↓
查询文本分离
  ↓
关键词提取
  ↓
专有名词保护
  ↓
时间与实体条件提取
  ↓
构建检索请求
```

保留两份文本：

- raw_query：原始查询，不修改。
- normalized_query：规范化后的检索文本。

查询特征：

```text
QueryFeatures
├── keywords[]
├── named_entities[]
├── time_constraints[]
├── semantic_query
├── intent
├── scope_constraints[]
└── original_query
```

规则：

1. 关键词提取不得删除原始语义。
2. 专有名词尽量保留完整表达。
3. 时间条件只在明确识别时使用。
4. 无法识别的查询特征不得强制生成过滤条件。
5. 查询意图用于召回优化，不具有权限决策能力。

### 1.3 结构化过滤

使用 SQLite 对记忆范围进行约束。

过滤条件：

- 用户及会话范围。
- 记忆状态。
- 删除状态。
- 生命周期状态。
- 时间范围（用户明确提出时）。
- 记忆类型（用户明确提出时）。
- 向量空间和索引版本（向量通道使用）。

基本形式：

\[ C_s=\{d\in D\mid F(d,q)=true\} \]

这里的 \(C_s\) 是结构化过滤后的候选范围。

必须区分：

- 检索范围过滤。
- 权限判断。

结构化过滤不能替代 M05。

### 1.4 FTS5 关键词召回

执行：

```text
normalized_query
   ↓
关键词与短语构建
   ↓
FTS5 MATCH
   ↓
BM25 排序
   ↓
Top-K
```

推荐初始配置：

| 参数 | 默认值 |
|---|---|
| FTS Top-K | 50 |
| 最小匹配条件 | 至少一个有效查询项 |
| 排序 | BM25 |
| 中文分词 | 可替换分词器 |
| 查询超时 | 配置化 |

FTS5 负责精确词项和表达匹配。

返回：

```text
FTSCandidate
├── memory_id
├── rank
├── bm25_score
├── matched_terms[]
└── index_version
```

BM25 分数只在 FTS 通道内部使用，不直接与向量相似度比较。

中文分词应支持替换实现。默认 unicode61 只能作为基础 tokenizer 方案，不能假定其已经解决中文分词问题。

### 1.5 Embedding 查询向量生成

通过统一接口：

```text
EmbeddingService
├── embedQuery(text)
├── embedDocument(text)
├── getProfile()
└── validateCompatibility()
```

查询向量：

\[ q=Embedding(query) \]

向量空间必须包含：

```text
EmbeddingProfile
├── profile_id
├── provider
├── model_id
├── model_version
├── dimension
├── metric
├── normalization
├── preprocessing_version
└── embedding_space_id
```

禁止不同 Embedding 空间的向量直接混合比较。

### 1.6 LanceDB FLAT 向量召回

**采用 LanceDB 是独立的向量检索层。**

执行：

```text
查询向量
  ↓
验证 embedding_space_id
  ↓
LanceDB 查询
  ↓
FLAT 精确距离计算
  ↓
按距离排序
  ↓
Top-K
```

相似度计算按 Embedding Profile 指定的距离度量执行。

如果使用余弦相似度：

\[ sim(q,d)=\frac{q\cdot d}{\|q\|\|d\|} \]

如果使用 L2 或 Dot Product，则必须使用对应的距离排序方向，不能混用。

推荐初始配置：

| 参数 | 默认值 |
|---|---|
| Vector Top-K | 50 |
| 检索方式 | FLAT 精确检索 |
| ANN | 禁止 |
| Cross-Encoder | 禁止 |
| 空间隔离 | 必须 |
| 查询维度校验 | 必须 |

**需要由实现验证当前 LanceDB SDK 的实际查询执行路径。若无法保证首版所要求的精确检索，停止并报告，不得偷偷切换 ANN 或 SQLite 代替。**

### 1.7 三路召回结果合并

形成：

\[ C=C_s\cup C_f\cup C_v \]

其中：

- \(C_s\)：结构化候选。
- \(C_f\)：FTS5 候选。
- \(C_v\)：LanceDB 候选。

此处只统一候选身份，不做最终排序。

每条候选必须保留：

```text
RecallCandidate
├── memory_id
├── source_channels[]
├── source_ranks{}
├── source_scores{}
├── embedding_space_id
└── index_versions{}
```

### 1.8 算法一伪代码

```text
function retrieve(request):

    query = normalize(request.current_message)

    features = extractFeatures(
        query,
        request.recent_context
    )

    scope = buildRetrievalScope(
        request,
        features
    )

    structured = sqlite.filter(scope)

    fts = fts5.search(
        features.keywords,
        scope,
        topK = 50
    )

    profile = embedding.getActiveProfile()

    vector = embedding.embedQuery(
        features.semantic_query,
        profile
    )

    validateVector(vector, profile)

    semantic = lancedb.searchExact(
        vector,
        profile.embedding_space_id,
        scope,
        topK = 50
    )

    return normalizeCandidates(
        structured,
        fts,
        semantic
    )
```

---

## 算法二：多路融合与规则排序算法

目标：把多个检索通道的候选统一成一个可靠的优先级序列。

### 2.1 候选身份统一

同一条记忆可能同时出现在：

- SQLite 结构化过滤结果。
- FTS5 结果。
- LanceDB 结果。

以 memory_id 作为唯一身份。

去重后保留所有来源排名：

```text
M001
├── FTS5 rank = 2
├── LanceDB rank = 1
└── Structured = matched
```

不能在去重时丢失通道证据。

### 2.2 RRF 融合（核心融合算法）

采用 Reciprocal Rank Fusion。

\[ RRF(d)=\sum_{r\in R}\frac{w_r}{k+rank_r(d)} \]

其中：

- \(d\)：候选记忆。
- \(R\)：参与融合的检索通道。
- \(rank_r(d)\)：候选在通道中的排名。
- \(w_r\)：通道权重。
- \(k\)：平滑参数。

首版基线：

| 参数 | 初始值 |
|---|---|
| RRF k | 60 |
| FTS 权重 | 1.0 |
| Vector 权重 | 1.0 |
| 未命中通道 | 不贡献分数 |
| 排名起点 | 1 |

结构化过滤默认只承担范围约束，不把它当作普通排名通道重复加权，避免所有符合范围的记录获得不合理优势。

示例：

| 记忆 | FTS 排名 | Vector 排名 |
|---|---|---|
| M001 | 1 | 3 |
| M002 | 2 | 未命中 |
| M003 | 未命中 | 1 |

M001 同时命中两路，获得两路排名贡献。

### 2.3 时间相关性计算

时间因素不能简单地等同于"越新越重要"。

建议使用可解释的分段衰减函数：

\[ T(d)=e^{-\lambda_t\Delta t} \]

其中：

- \(\Delta t\)：当前时间与记忆时间的差值。
- \(\lambda_t\)：由记忆类型对应的时间衰减参数决定。

但不同类型记忆必须采用不同的时间策略：

| 记忆类型 | 时间处理 |
|---|---|
| 稳定事实 | 不进行常规时间衰减 |
| 历史事件 | 保留历史价值，不因时间久远自动淘汰 |
| 当前计划 | 结合计划状态与有效期 |
| 临时状态 | 较强时间衰减 |
| 过期信息 | 根据生命周期状态排除或降权 |

时间衰减不能直接覆盖 M04 的冲突处理结果。

### 2.4 规则排序

RRF 负责多路融合，规则排序负责处理记忆业务属性。

推荐采用可解释的排序特征：

```text
RankingFeatures
├── rrf_score
├── exact_term_match
├── entity_match
├── time_relevance
├── memory_type_relevance
├── lifecycle_validity
├── source_quality
└── duplicate_penalty
```

为避免不同量纲混用，先将每项转换成约定范围，再计算排序分数。

建议的首版排序公式：

\[ Score(d)= \alpha R(d) +\beta E(d) +\gamma N(d) +\delta T(d) +\epsilon S(d) -\zeta P(d) \]

其中：

| 符号 | 含义 |
|---|---|
| \(R\) | 归一化 RRF 得分 |
| \(E\) | 精确关键词或实体匹配 |
| \(N\) | 查询与记忆类型的适配程度 |
| \(T\) | 时间相关性 |
| \(S\) | 来源质量 |
| \(P\) | 重复或冗余惩罚 |

首版建议的初始权重：

| 权重 | 初始值 |
|---|---|
| \(\alpha\) | 0.55 |
| \(\beta\) | 0.15 |
| \(\gamma\) | 0.10 |
| \(\delta\) | 0.10 |
| \(\epsilon\) | 0.10 |
| \(\zeta\) | 0.10 |

实现时必须将正向得分与惩罚项分开归一化，避免重复惩罚。

这些权重是可运行的初始基线，后续由算法五的评估集校准，不允许在代码里散落硬编码。

### 2.5 冲突与重复处理

这里必须明确 M02 和 M04 的边界：

- M02 可以识别重复候选、聚合相同记忆的检索证据。
- M02 不负责判定两条记忆哪条是真的。
- M02 不负责合并权威记忆。
- M02 不得擅自覆盖 M04 已经维护的冲突关系。

如果两条候选存在已记录的冲突关系：

1. 保留冲突标识。
2. 优先遵循权威记忆状态与有效性。
3. 不自行选定其中一条为真实事实。
4. 必要时同时输出冲突信息，交给上层模型谨慎处理。

### 2.6 最终排序与截断

执行：

```text
候选集合
  ↓
memory_id 去重
  ↓
RRF 融合
  ↓
业务规则评分
  ↓
状态检查
  ↓
冲突标识
  ↓
稳定排序
  ↓
候选 Top-K
```

稳定排序的最后兜底条件使用固定字段，例如 memory_id，保证相同输入和相同数据版本得到可复现结果。

建议排序后保留 Top 20，交给上下文构建算法进行权限与预算筛选。

### 2.7 算法二伪代码

```text
function rankCandidates(candidates, query):

    unique = deduplicateByMemoryId(candidates)

    for candidate in unique:

        candidate.rrfScore = calculateRRF(
            candidate.sourceRanks,
            k = 60,
            weights = {
                fts: 1.0,
                vector: 1.0
            }
        )

        features = calculateRankingFeatures(
            candidate,
            query
        )

        candidate.finalScore = weightedScore(
            features
        )

        candidate.conflicts = readConflictMetadata(
            candidate.memoryId
        )

    ranked = stableSort(
        unique,
        by = finalScore DESC,
        then = memoryId ASC
    )

    return ranked.slice(0, 20)
```

---

## 算法三：权限终检与上下文构建算法

目标：从排序后的候选中，选出真正有用、允许读取、能够放进上下文的记忆。

这个算法不是简单地取排名前几名，而是一个受权限约束的预算分配过程。

### 3.1 输入与输出

输入：

```text
ContextBuildRequest
├── query
├── ranked_candidates[]
├── requester
├── conversation_context
├── token_budget
└── context_policy
```

输出：

```text
MemoryContext
├── query_id
├── selected_memories[]
├── excluded_memories[]
├── total_tokens
├── remaining_tokens
├── permission_trace
├── selection_trace
└── algorithm_version
```

### 3.2 权威数据回源

排序阶段拿到的候选只是索引召回结果。

进入上下文构建前，必须重新从 SQLite 获取最新记录：

```text
memory_id
   ↓
SQLite SELECT
   ↓
检查是否存在
   ↓
检查版本
   ↓
检查删除状态
   ↓
检查生命周期状态
   ↓
取得最新正文与元数据
```

处理规则：

| 情况 | 处理 |
|---|---|
| 记录不存在 | 排除 |
| 已删除 | 排除 |
| 已失效且不允许读取 | 排除 |
| 版本已变化 | 使用最新版本，并重新评估 |
| 索引与权威记录不一致 | 记录一致性异常 |
| 数据读取失败 | 排除并记录错误 |

**禁止将 LanceDB 的副本正文直接作为最终上下文。**

### 3.3 M05 权限终检（强制安全边界）

每条候选都必须调用 M05：

```text
canRead(
    requester,
    memory,
    context
)
```

结果只有三种业务状态：

```text
ALLOW
DENY
INDETERMINATE
```

其中异常、超时和未知统一归入 INDETERMINATE。

处理：

```text
ALLOW          → 继续
DENY           → 排除
INDETERMINATE  → 排除
```

必须遵守：

1. 默认拒绝。
2. 权限服务失败不能默认放行。
3. 不能用检索范围过滤替代权限检查。
4. 不能因为用户曾经创建过记忆，就推定当前场景可以读取。
5. 不能因为某条记忆已进入检索结果，就认为它允许注入。
6. 权限终检必须发生在最终 Prompt 注入之前。

如果权限终检后，记忆又发生了版本或权限变更，必须重新读取和检查，不能继续使用旧检查结果。

### 3.4 Token 计算

每条候选计算实际 Token 成本：

\[ Cost(d)=TokenCount(Serialize(d)) \]

注意，计算对象不是只有记忆正文，而是最终注入时真正会出现的完整表示，包括必要的：

- 标题。
- 正文或摘要。
- 时间。
- 来源标记。
- 记忆类型标记。
- 上下文分隔符。

Token 预算：

\[ \sum_{d\in S}Cost(d)\leq B \]

其中：

- \(S\)：最终选择的记忆集合。
- \(B\)：本次请求允许使用的 Token 预算。

如果宿主没有提供精确 Tokenizer，可以使用明确标记的估算器，但日志必须区分 estimated 和 exact。

### 3.5 候选价值计算

排序算法已经计算相关性，但上下文构建还要考虑内容冗余。

例如：

- 两条记忆都描述用户同一个学习计划。
- 一条记忆是计划的完整版本，另一条只是重复表达。
- 两条记忆分别包含不同的补充信息。

不能仅因为它们排名靠前，就全部注入。

建议使用边际价值：

\[ U(d\mid S)=Relevance(d)-\lambda Redundancy(d,S) \]

其中：

- \(S\)：已经选择的记忆集合。
- \(Relevance(d)\)：候选相关性。
- \(Redundancy(d,S)\)：与已选记忆的内容冗余程度。
- \(\lambda\)：冗余惩罚参数。

这里的冗余检测只影响本次上下文选择，不修改权威记忆，也不执行 M04 合并。

### 3.6 预算选择算法

首版采用确定性的贪心选择：

1. 对候选按最终相关性排序。
2. 读取完整权威内容。
3. 执行权限终检。
4. 计算最终序列化 Token 成本。
5. 计算边际价值。
6. 检查剩余预算。
7. 选择符合预算的候选。
8. 对剩余候选继续处理，直到达到候选上限或预算无法容纳更多内容。

核心公式：

\[ S^* \approx GreedySelect(C,B) \]

选择时必须遵守：

- 优先保留高相关性内容。
- 避免注入高度重复内容。
- 不因预算不足而截断成语义不完整的句子。
- 有经过验证的摘要时，可使用摘要表示。
- 没有合适摘要时，宁可跳过，也不随意改写原文。
- 不得为了凑满预算而注入低相关性记忆。

首版建议配置：

| 参数 | 默认值 |
|---|---|
| 排序候选数 | 20 |
| 上下文预算 | 1200 Tokens |
| 预算策略 | Greedy |
| 冗余检测 | 开启 |
| 自动截断原文 | 关闭 |
| 未授权内容 | 绝不注入 |

这里的 1200 Tokens 是可配置初始值，不是固定不可变的系统限制。

### 3.7 上下文组织

最终输出采用有序分区：

```text
MemoryContext
├── DirectlyRelevant
│   └── 当前问题直接相关记忆
├── StableFacts
│   └── 长期稳定事实
├── HistoricalEvents
│   └── 相关历史事件
├── ActivePlans
│   └── 当前仍有效的计划
└── RelevantBackground
    └── 必要背景信息
```

组织规则：

1. 当前问题直接相关内容优先。
2. 稳定事实不因时间久远而自动降级。
3. 历史事件保持时间和来源信息。
4. 计划必须保留当前状态，不能将已完成计划描述成正在执行。
5. 冲突记忆保留冲突标记，不擅自生成一个统一结论。
6. 只输出对当前查询有帮助的内容。

### 3.8 算法三伪代码

```text
function buildContext(ranked, request):

    budget = request.tokenBudget
    selected = []
    excluded = []

    for candidate in ranked:

        memory = sqlite.getLatest(
            candidate.memoryId
        )

        if memory == null:
            excluded.push("NOT_FOUND")
            continue

        if memory.deleted:
            excluded.push("DELETED")
            continue

        if !isEligible(memory):
            excluded.push("INELIGIBLE")
            continue

        permission = M05.canRead(
            request.requester,
            memory,
            request.context
        )

        if permission != ALLOW:
            excluded.push("PERMISSION_DENIED")
            continue

        representation = chooseRepresentation(
            memory,
            request.contextPolicy
        )

        cost = tokenizer.count(
            serialize(representation)
        )

        if cost > budget:
            excluded.push("BUDGET_EXCEEDED")
            continue

        utility = calculateMarginalUtility(
            candidate,
            representation,
            selected
        )

        if utility <= 0:
            excluded.push("REDUNDANT")
            continue

        selected.push(representation)
        budget -= cost

    return assembleContext(selected)
```

### 3.9 必须记录的追踪信息

每条记忆至少记录：

```text
ContextTrace
├── memory_id
├── retrieved
├── ranked
├── authoritative_read
├── permission_result
├── selected
├── token_cost
└── exclusion_reason
```

这样才能回答：

- 为什么没有找到这条记忆？
- 为什么找到了却没排上去？
- 为什么排序靠前却没有注入？
- 为什么被权限系统拒绝？
- 为什么被 Token 预算排除？

**特别注意：最终注入日志只证明记忆进入了 Prompt，不证明模型实际使用了它。**

---

## 算法四：记忆索引构建、同步与重建算法

目标：保证 SQLite 权威记忆、FTS5 全文索引和 LanceDB 向量索引始终能够正确对应。

这是检索系统的基础设施算法。它的可靠性直接决定前三个算法能不能得到可信数据。

### 4.1 数据分层

```text
SQLite：权威记忆数据库
（Memory · Version · Metadata · Outbox）
        │
        ▼
Outbox Worker
（事件消费 · 版本检查 · 重试 · 幂等）
        │
   ┌────┴────┐
   ▼         ▼
 FTS5     LanceDB
（关键词索引）（向量索引）
```

两个索引均可由 SQLite 中的权威记忆重新构建。

### 4.2 Search Document 构建

每条记忆在进入检索索引之前，先转换成统一的 Search Document。

```text
SearchDocument
├── memory_id
├── memory_version
├── title
├── content
├── summary
├── tags[]
├── memory_type
├── lifecycle_state
├── visibility_metadata
├── created_at
├── updated_at
├── embedding_profile_id
└── document_hash
```

FTS5 和 LanceDB 应共享相同的内容版本和文档身份。

但需要区分：

- FTS5 存储可检索文本。
- LanceDB 存储向量及必要元数据。
- SQLite 保存完整权威记录。
- 权限终检使用 SQLite 和 M05，不依赖向量库元数据。

### 4.3 SQLite 事务与 Outbox

每次权威记忆修改：

```text
BEGIN TRANSACTION
1. 写入 memory
2. 更新 memory_version
3. 更新 document_hash
4. 写入 outbox event
COMMIT
```

事务成功后，后台 Worker 消费 Outbox。

这样可以避免：

- 记忆写入成功但索引更新事件丢失。
- Outbox 事件成功但权威记忆写入失败。

### 4.4 Outbox 事件结构

```text
OutboxEvent
├── event_id
├── memory_id
├── memory_version
├── event_type
├── document_hash
├── embedding_profile_id
├── created_at
├── status
├── retry_count
└── last_error
```

事件类型：

```text
CREATE
UPDATE
DELETE
REINDEX
```

每个事件都必须有唯一标识，保证幂等消费。

### 4.5 同步流程

```text
Outbox Worker
   ↓
读取待处理事件
   ↓
获取 SQLite 最新记录
   ↓
比较 memory_version
   ↓
构建 Search Document
   ↓
更新 FTS5
   ↓
生成 Embedding
   ↓
更新 LanceDB
   ↓
校验两个索引版本
   ↓
标记事件完成
```

注意：FTS5 和 LanceDB 并不共享一个跨数据库事务。因此需要依靠幂等写入、版本记录和重试机制保证最终一致性。

事件处理顺序：

1. 先确认权威记录当前版本。
2. 再构建对应 Search Document。
3. 更新 FTS5。
4. 更新 LanceDB。
5. 验证两个索引对应的版本和文档哈希。
6. 最后确认事件完成。

如果中途失败，不能把事件标记为全部完成。

### 4.6 版本控制与过期事件

例如：

```text
M001 version 1
M001 version 2
M001 version 3
```

如果 version 2 的事件晚于 version 3 到达，不能让旧版本覆盖新版本。

必须执行：

\[ v_{event}<v_{current} \Rightarrow SkipOrSupersede \]

删除事件也必须具有版本保护。

删除操作不能因为旧的 UPDATE 重试而恢复已经删除的记忆。

### 4.7 幂等机制

对于同一事件重复执行：

\[ Process(event_n)=Process(event_n) \]

最终索引状态应保持一致。

实现要求：

- 唯一 event_id。
- 稳定 memory_id。
- 版本比较。
- Upsert 幂等。
- Delete 幂等。
- 重复事件可安全跳过。
- 失败事件记录错误和重试次数。

### 4.8 Embedding 生成与写入

Embedding 生成可能耗时，因此应与 SQLite 权威事务分离。

执行：

```text
SearchDocument
   ↓
EmbeddingService
   ↓
validateVector
   ↓
LanceDB upsert
   ↓
verifyVersion
```

必须验证：

- 维度正确。
- 向量值合法。
- Embedding Profile 一致。
- 文档版本一致。
- 文档哈希一致。
- 归一化规则符合 Profile。

Embedding 生成失败时：

- 记录失败。
- 保留 Outbox 待重试状态。
- 不允许错误向量进入索引。
- 不得把失败事件当成同步完成。

### 4.9 删除传播

删除是一个特殊事件。

流程：

```text
SQLite 删除或失效
   ↓
Outbox DELETE
   ↓
FTS5 删除
   ↓
LanceDB 删除
   ↓
验证索引不存在
   ↓
事件完成
```

在派生索引删除完成之前，检索层仍必须依靠 SQLite 权威状态排除已删除记录。

**不能依赖异步删除保证实时安全。**

### 4.10 全量重建与恢复

重建过程：

```text
SQLite 权威数据库
   ↓
读取有效记忆
   ↓
构建 Search Documents
   ↓
重建 FTS5
   ↓
生成 Embedding
   ↓
重建 LanceDB
   ↓
版本与哈希校验
   ↓
检索一致性测试
   ↓
切换索引
```

必须支持：

- 单条记忆重建。
- 指定范围重建。
- 全量重建。
- Embedding 空间重建。
- 删除记录清理。
- 重建失败回滚。

SQLite 备份和恢复不能只验证数据库文件能打开，还必须验证恢复后能够重建 FTS5 和 LanceDB。

### 4.11 一致性状态

建议维护：

```text
IndexStatus
├── memory_id
├── memory_version
├── fts_version
├── vector_version
├── document_hash
├── embedding_profile_id
├── fts_synced
├── vector_synced
└── last_sync_at
```

定义：

\[ Consistent(d)= (V_{fts}=V_{db}) \land (V_{vector}=V_{db}) \land (Hash_{index}=Hash_{db}) \]

向量空间不同的情况不能仅比较维度，还必须比较 Profile 和空间身份。

### 4.12 算法四伪代码

```text
function processOutbox(event):

    if eventAlreadyCompleted(event.id):
        return SUCCESS

    memory = sqlite.getLatest(event.memoryId)

    if memory == null or memory.deleted:
        fts5.delete(event.memoryId)
        lancedb.delete(event.memoryId)
        verifyDeleted(event.memoryId)
        return markCompleted(event)

    if event.version < memory.version:
        return markSuperseded(event)

    document = buildSearchDocument(memory)

    fts5.upsert(document)

    vector = embedding.embedDocument(
        document.content,
        document.embeddingProfile
    )

    validateVector(vector)

    lancedb.upsert(
        document,
        vector
    )

    verifyIndexConsistency(
        document.memoryId,
        document.version,
        document.hash
    )

    return markCompleted(event)
```

这里的 markCompleted 只能在所有必需索引更新及一致性检查成功后执行。

---

## 算法五：检索质量评估与反馈算法

目标：让检索系统能够被客观测试、持续比较和迭代，而不是只凭主观感觉判断效果。

这一算法不负责直接修改记忆，而是负责测量前四个算法的实际表现。

### 5.1 评估体系

分成六个评估维度：

| 维度 | 主要问题 | 指标 |
|---|---|---|
| 召回质量 | 应该找到的记忆有没有找到？ | Recall@K、Hit Rate |
| 精确性 | 找到的内容有多少真正相关？ | Precision@K |
| 排序质量 | 最相关的记忆是否排在前面？ | MRR、NDCG@K |
| 上下文质量 | 最终注入的记忆是否有用？ | Context Precision、Coverage |
| 性能 | 检索是否足够快？ | P50、P95、P99 |
| 资源效率 | 系统消耗是否合理？ | Token、CPU、内存、Embedding 成本 |

### 5.2 建立固定评估数据集

建立独立的 Retrieval Benchmark：

```text
benchmark/
├── queries.jsonl
├── memories.jsonl
├── relevance.jsonl
├── permission_cases.jsonl
├── conflict_cases.jsonl
└── expected_results.jsonl
```

每条测试查询包含：

```text
BenchmarkQuery
├── query_id
├── query_text
├── query_type
├── relevant_memory_ids[]
├── expected_exclusions[]
├── expected_top_memories[]
└── expected_context_constraints
```

测试类型必须覆盖：

1. 精确关键词查询。
2. 语义相似查询。
3. 同义表达查询。
4. 专有名词查询。
5. 时间限定查询。
6. 历史事件查询。
7. 长期事实查询。
8. 计划状态查询。
9. 多条相似记忆。
10. 冲突记忆。
11. 删除记忆。
12. 权限隔离记忆。
13. Token 超预算。
14. 不同 Embedding 空间。
15. 索引同步延迟。

测试集应固定版本，避免每次修改算法后使用不同测试数据，导致结果无法比较。

### 5.3 召回质量

Recall@K：

\[ Recall@K= \frac{|Relevant\cap Retrieved_K|} {|Relevant|} \]

它回答：应该找到的记忆，有多少被系统找到了？

Hit Rate@K：

\[ HitRate@K= \frac{\text{至少找到一条相关记忆的查询数}} {\text{全部有效查询数}} \]

必须分别测试：

- FTS5 单独召回。
- LanceDB 单独召回。
- 多路融合召回。

这样才能知道融合算法是否真的带来了提升。

### 5.4 排序质量

MRR：

\[ MRR=\frac{1}{|Q|} \sum_{q\in Q}\frac{1}{rank_q} \]

它主要衡量第一条相关记忆出现的位置。

NDCG@K：

\[ DCG@K= \sum_{i=1}^{K} \frac{2^{rel_i}-1}{\log_2(i+1)} \]

\[ NDCG@K=\frac{DCG@K}{IDCG@K} \]

它可以使用分级相关性标签：

| 等级 | 含义 |
|---|---|
| 3 | 高度相关，直接回答查询 |
| 2 | 明确相关，能提供重要信息 |
| 1 | 有一定背景价值 |
| 0 | 不相关 |

这比简单的命中 / 未命中更适合评估长期记忆检索。

### 5.5 上下文质量

只测召回是不够的。

真正交给模型的是经过权限与预算筛选的最终上下文。

需要单独测量：

Context Precision——最终注入的记忆中，有多少是真正相关的。

\[ CP= \frac{\text{相关且被注入的记忆数}} {\text{被注入的记忆总数}} \]

Context Coverage——应该注入的关键记忆，有多少最终进入上下文。

\[ CC= \frac{\text{被正确注入的相关记忆数}} {\text{应该注入的相关记忆数}} \]

Token Efficiency：

\[ TE= \frac{\text{有效相关信息量}} {\text{实际使用Token数}} \]

这里的有效信息量需要基于标注数据集或明确的评估规则，不要直接把模型生成的主观评价当作客观指标。

### 5.6 性能评估

每个阶段分别记录：

```text
LatencyTrace
├── query_normalization_ms
├── embedding_ms
├── sqlite_filter_ms
├── fts5_search_ms
├── lancedb_search_ms
├── rrf_ms
├── rerank_ms
├── permission_ms
├── tokenization_ms
└── context_build_ms
```

需要测量：

- P50：典型请求延迟。
- P95：较慢请求延迟。
- P99：尾部请求延迟。
- 每次查询的 Embedding 消耗。
- LanceDB 查询资源消耗。
- Outbox 积压数量。
- 索引同步延迟。

首版应先建立实测基线，再制定正式性能门槛，不要随意编造延迟验收值。

### 5.7 消融实验（非常重要）

要证明每个算法模块确实有价值。

建议固定测试集，对比：

| 实验 | 检索配置 |
|---|---|
| A | 仅 FTS5 |
| B | 仅 LanceDB |
| C | FTS5 + LanceDB |
| D | FTS5 + LanceDB + RRF |
| E | 多路召回 + RRF + 规则排序 |
| F | 完整算法 + 权限终检 + Token 预算 |

比较 Recall@K、NDCG@K、Context Precision、Token 使用量和延迟。

这可以回答：

- LanceDB 是否弥补了关键词检索的不足？
- FTS5 是否提高了专有名词检索质量？
- RRF 是否提升了整体排序？
- 规则排序是否改善了最终结果？
- Token 预算是否造成重要记忆遗漏？

### 5.8 反馈与参数更新

反馈分为三种：

| 类型 | 用途 |
|---|---|
| 离线 Benchmark | 验证算法变化 |
| 运行时日志 | 发现实际问题 |
| 人工标注 | 判断真实相关性 |

运行时日志可以记录：

- 是否召回。
- 最终排名。
- 是否注入。
- 权限排除原因。
- Token 消耗。
- 索引版本。
- 算法版本。

但是：

**不能将模型是否提及某条记忆，直接当作该记忆是否正确或有价值的标签。**

参数更新必须经过评估：

```text
提出参数变更
   ↓
保存当前基线
   ↓
运行 Benchmark
   ↓
比较质量指标
   ↓
检查性能和安全回归
   ↓
生成评估报告
   ↓
批准后更新参数版本
```

**禁止根据单次对话自动修改核心排序权重。**

### 5.9 算法五伪代码

```text
function evaluateAlgorithm(version, benchmark):

    results = []

    for testCase in benchmark:

        output = memoryCore.retrieve(
            testCase.request
        )

        metrics = calculateMetrics(
            output,
            testCase.expected
        )

        security = verifyPermissionIsolation(
            output,
            testCase.permissionConstraints
        )

        performance = collectLatency(
            output.trace
        )

        results.push({
            queryId: testCase.id,
            metrics,
            security,
            performance
        })

    report = aggregate(results)

    compareWithBaseline(
        report,
        version.baseline
    )

    return report
```

---

## 十一、五个算法之间的统一接口

这部分是给工程实现用的。五个算法必须有统一的数据契约，否则各模块各自实现一套结构，最后很难联调。

| 算法 | 输入 | 输出 | 主要模块 |
|---|---|---|---|
| 算法一：多路召回 | RetrievalRequest | RecallResult | M02 |
| 算法二：融合排序 | RecallResult | RankedCandidates | M02 |
| 算法三：上下文构建 | RankedCandidates | MemoryContext | M02 + M05 |
| 算法四：索引同步 | OutboxEvent | IndexSyncResult | M01 + Adapters |
| 算法五：评估反馈 | Benchmark + Trace | EvaluationReport | 测试与评估层 |

建议统一使用以下核心数据结构：

```text
MemoryRecord
├── memory_id
├── version
├── memory_type
├── content
├── summary
├── source
├── lifecycle_state
├── visibility
├── created_at
└── updated_at

SearchDocument
├── memory_id
├── memory_version
├── title
├── content
├── tags
├── document_hash
└── embedding_profile_id

RecallCandidate
├── memory_id
├── source_ranks
├── source_scores
├── rrf_score
└── retrieval_trace

RankedCandidate
├── memory_id
├── final_score
├── ranking_features
├── conflict_metadata
└── explanation

ContextItem
├── memory_id
├── version
├── representation
├── token_cost
├── permission_result
└── section

EvaluationReport
├── algorithm_version
├── benchmark_version
├── recall_metrics
├── ranking_metrics
├── context_metrics
├── performance_metrics
└── regression_results
```

---

## 十二、最终实施配置

以下是一套可以直接作为原型初始配置的参数集合。配置集中管理，不能散落在业务代码中。

```yaml
memory_core:
  algorithm_version: "R4.1-V1"

  storage:
    authority: sqlite

  retrieval:
    structured:
      enabled: true

    fts:
      enabled: true
      top_k: 50
      ranking: bm25
      tokenizer: configurable

    vector:
      enabled: true
      store: lancedb
      search_mode: flat
      top_k: 50
      ann_enabled: false
      cross_encoder_enabled: false

  embedding:
    provider: configurable
    model: configurable
    dimension: 1024
    profile_id: configurable
    normalization: configurable

  fusion:
    algorithm: rrf
    k: 60
    weights:
      fts: 1.0
      vector: 1.0

  reranking:
    algorithm: rule_based
    top_k: 20
    weights:
      rrf: 0.55
      exact_match: 0.15
      type_relevance: 0.10
      time_relevance: 0.10
      source_quality: 0.10
    duplicate_penalty: 0.10

  context:
    token_budget: 1200
    selection: greedy
    redundancy_check: true
    truncate_original_content: false
    permission_fail_policy: deny

  indexing:
    outbox:
      enabled: true
      poll_interval_ms: 1000
      retry: true
      idempotent: true
    consistency_check: true
    rebuild_supported: true

  evaluation:
    benchmark_enabled: true
    metrics:
      - recall_at_k
      - hit_rate
      - precision_at_k
      - mrr
      - ndcg
      - context_precision
      - context_coverage
      - token_efficiency
      - p50_latency
      - p95_latency
```

配置中的 Embedding 维度 1024 是当前原型参考值，必须与实际加载的模型一致，**不能因为配置写了 1024 就强制把其他模型的向量改成 1024 维**。

此外，time_relevance 和 duplicate_penalty 必须通过统一规则实现，不能把上面的权重简单套在任意范围的原始数值上。各特征需要先有明确的归一化定义。

---

## 十三、代码实施顺序与验收门槛

五个算法在设计上是完整体系，但原型开发不应一次把所有功能都塞进 M01/M02。

结合此前的 P0–P3 范围，实施顺序如下：

| 阶段 | 实施内容 | 验收结果 |
|---|---|---|
| P0 | 环境、SQLite、LanceDB、FTS5、Embedding 接口、配置和 Schema | 基础能力可用 |
| P1 | 算法四的权威存储与 Outbox 基础 | M01 独立测试通过 |
| P2 | 算法一、算法二、算法三 | M02 独立检索闭环通过 |
| P3 | 算法五、端到端验证、故障恢复与回归 | 完整原型验收 |
| P4 | M03、M04、M05 的正式实现和接入 | 另行授权 |

其中，P0–P3 可以使用 Mock Embedding 和隔离的权限测试替身验证算法流程，但**必须清楚标记模拟范围，不能冒充真实模型效果或真实 M05 权限验收**。

M05 尚未正式实现时，M02 必须通过权限接口契约工作，**生产默认拒绝**；只有显式隔离的测试环境才允许使用测试替身。

### P3 最终验收清单

```text
[ ] SQLite 权威存储正确。
[ ] FTS5 中文检索可用。
[ ] LanceDB 向量检索可用。
[ ] FLAT 精确检索路径得到验证。
[ ] Embedding Profile 隔离正确。
[ ] 三路召回结果可以统一。
[ ] RRF 排名可复现。
[ ] 规则排序可解释。
[ ] 重复候选可以处理。
[ ] 冲突信息不会被擅自裁决。
[ ] SQLite 权威数据回源正确。
[ ] M05 权限终检失败时默认拒绝。
[ ] Token 预算计算正确。
[ ] 最终 Prompt 不超过预算。
[ ] Outbox 幂等、重试和删除传播正常。
[ ] FTS5 和 LanceDB 可以从 SQLite 重建。
[ ] Benchmark 可以重复执行。
[ ] 测试结果和算法版本可追溯。
[ ] 没有未经授权的设计变更。
```

---

## 十四、交给代码 AI 时必须遵守的总原则

1. 不允许把五个算法混写成一个检索函数，必须保持模块边界。
2. 不允许用 LanceDB 取代 SQLite 权威存储。
3. 不允许用 ANN 替代 FLAT。
4. 不允许用 Cross-Encoder 替代首版规则排序。
5. 不允许直接比较 BM25 分数与向量相似度，必须使用 RRF 等融合方式。
6. 不允许让 M02 自己决定权限。
7. 不允许在权限检查失败时继续注入。
8. 不允许用索引副本替代 SQLite 权威数据。
9. 不允许在不同 Embedding 空间中混合向量。
10. 不允许自动把重复候选合并成权威记忆。
11. 不允许自动根据运行时反馈修改算法权重。
12. 不允许把模拟测试写成真实验收结果。
13. 不允许擅自修改正式裁决、接口契约和验收标准。
14. 每次修改前必须独立备份并校验 SHA256，备份失败立即停止。
15. 每个阶段必须保留测试报告、设计偏差和可追溯日志。

最终定位：算法一到三构成检索执行核心，算法四保障数据与索引正确性，算法五保障算法质量可测量、可比较、可迭代。

这五部分合起来才是完整的 Memory Core 检索算法体系，而不是单独一个向量搜索功能。

这份规格可以作为后续代码实施的统一算法基线。它不改变 R4.1 的模块职责，也不自动扩大 P0–P3 的授权范围。
