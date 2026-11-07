# Memory Core R4.1 — 完整目标架构与实施基线

> 生成：2026-11-04 ｜ 维护位置：`E:\[PROJECT]\work-交付包\15-MemoryCore-R4.1目标架构与实施基线.md`
> 地位：**目标架构总纲**（文档性质：推荐架构基线 / 待正式裁决）。算法规格以 `14` 为准；与裁决台账（`08`）冲突时以台账为准。**P0–P3 是当前实施授权范围，P4 以后必须单独获得授权**（§23）。
> 章节编号（0～26）沿用原稿。

---

**文档性质：** 推荐架构基线 / 待正式裁决
**适用对象：** Memory Core、DSH、QQ Bridge 及其后续扩展
**核心目标：** 构建模块化、可追溯、可治理、可替换、可持续演进的长期记忆基础设施。

---

# 0. 总体设计结论

## 0.1 系统定位

Memory Core 是一个独立于具体 LLM、Agent 框架和聊天平台的长期记忆基础设施。

它负责：

1. 接收记忆形成所需的输入。
2. 提取候选记忆。
3. 管理记忆对象及其证据来源。
4. 存储、版本化和维护记忆。
5. 整合重复、冲突和关联信息。
6. 根据请求召回相关记忆。
7. 对记忆访问进行权限控制。
8. 构建受 Token 预算约束的记忆上下文。
9. 管理记忆生命周期。
10. 通过评估和审计持续改善系统质量。

它不负责：

- 替代 LLM 的推理能力。
- 替代 DSH 的 Agent 调度。
- 替代 QQ Bridge 的平台通信。
- 把所有聊天记录永久转化为长期记忆。
- 把模型推测直接当作用户事实。
- 将上下文压缩等同于长期记忆。
- 将外部知识库和文档仓库全部复制进记忆库。

## 0.2 核心架构原则

| 原则 | 具体要求 |
|---|---|
| 模块自治 | 每个模块有独立职责、接口和测试 |
| 数据统一 | 所有模块遵守统一对象模型和状态语义 |
| 权威存储 | SQLite 保存权威业务状态 |
| 派生可重建 | FTS5、LanceDB、缓存等不是权威数据源 |
| 读写分离 | 记忆形成与记忆读取采用不同流水线 |
| 权限优先 | 所有读取、写入和派生操作受权限策略约束 |
| 来源可追溯 | 记忆必须能够追溯来源和处理过程 |
| 不确定性保留 | 推断、事实、冲突和未知不得混为一谈 |
| 版本化 | 数据结构、算法、索引空间和接口均有版本 |
| 可替换 | 数据库驱动、Embedding、检索算法和模型可替换 |
| 可恢复 | 索引和缓存损坏后能够从权威数据重建 |
| 可观测 | 关键操作有事件、指标和审计记录 |

## 0.3 总体结构

系统划分为五个逻辑平面：

### A. 记忆处理平面

负责记忆形成、候选生成、证据关联、整合和生命周期维护。

### B. 持久化平面

负责权威数据、版本、事件、事务和派生索引。

### C. 检索与上下文平面

负责查询处理、多路召回、排序、权限终检和上下文构建。

### D. 治理平面

负责身份、权限、隐私、审计、删除和数据质量约束。

### E. 集成与运行平面

负责与 DSH、QQ Bridge、LLM、Embedding 服务和上下文压缩系统协作。

五个平面是逻辑分层，不要求部署成五个独立服务。首版可以全部运行在一个 Node.js 进程中，通过内部接口隔离。

---

# 1. 完整模块体系

## 1.1 核心模块

| 模块 | 名称 | 职责 |
|---|---|---|
| M01 | Memory Storage | 权威存储、事务、版本、Outbox |
| M02 | Retrieval & Context | 检索、排序、权限终检、上下文构建 |
| M03 | Memory Extraction | 候选提取、规范化、证据关联 |
| M04 | Memory Consolidation | 去重、合并、冲突处理、Observation |
| M05 | Governance & Permission | 身份、权限、访问控制、隐私治理 |

## 1.2 协同模块

| 模块 | 职责 |
|---|---|
| Dream | 周期维护任务的调度与编排 |
| Psychology | 维护受约束的模拟心理状态 |
| Persona | 消费记忆视图，为人格表达提供上下文 |
| Session Memory | 当前会话的短期状态 |
| Context Adapter | 向 DSH 提供统一上下文接口 |
| Audit Service | 操作审计和安全事件记录 |
| Evaluation Service | 质量、检索和系统评估 |

协同模块不得绕过 M01、M04、M05 的职责边界。

## 1.3 模块自治要求

每个模块必须拥有：

- 独立接口定义。
- 独立输入输出 Schema。
- 独立配置命名空间。
- 独立单元测试。
- 独立错误码。
- 独立版本号。
- 独立可观测性指标。
- 明确的数据访问范围。
- 明确的依赖清单。

模块之间只能通过公开接口或事件契约协作。

禁止直接读取其他模块的内部数据库表、私有文件或运行时变量。

允许共享底层基础设施，但不允许共享未公开的内部业务逻辑。

---

# 2. 统一记忆本体框架

## 2.1 记忆分类

采用多维分类，不把所有记忆类型塞进一个互斥枚举。

### 内容类型

| 类型 | 含义 |
|---|---|
| Semantic | 稳定事实、属性、偏好 |
| Episodic | 具体经历、事件、互动 |
| Procedural | 经验、流程、方法 |
| Preference | 表达、行为、交互偏好 |
| Intent | 目标、计划、意图 |
| Relationship | 人物、项目、实体之间的关系 |
| Observation | 从多条来源信息形成的派生观察 |

Preference 可以作为 Semantic 的子类，也可以独立分类；推荐保留独立分类标签，以便制定特殊提取和生命周期规则。

### 信息性质

每条记忆还必须区分：

- Fact：明确事实。
- Opinion：观点。
- Preference：偏好。
- Intent：意图。
- Hypothesis：假设。
- Inference：推断。
- Event：事件。
- Procedure：方法。

不能仅凭文本形式判断信息性质。

## 2.2 记忆范围

每条记忆必须绑定明确的 Scope。

建议首版支持：

| Scope | 含义 |
|---|---|
| owner_private | 指定主体的私人记忆 |
| group_shared | 指定群聊内共享 |
| session_only | 当前会话有效 |
| project_scoped | 指定项目范围 |
| system_internal | 系统运行内部数据 |

Scope 必须与 subject_id、tenant_id、channel_id 等身份字段共同确定。

不得仅依靠文件夹、群聊 ID 或数据库分区位置推断访问权限。

## 2.3 统一 Memory Object

建议采用如下逻辑结构：

```json
{
  "memory_id": "stable-unique-id",
  "schema_version": "1",
  "subject_id": "subject-id",
  "tenant_id": "tenant-id",
  "scope": "owner_private",

  "type": "semantic",
  "subtype": "preference",
  "content": "用户偏好逐步讲解技术知识",

  "validation_state": "confirmed",
  "lifecycle_state": "active",
  "availability_state": "available",

  "confidence": null,
  "importance": null,
  "sensitivity": "private",

  "valid_time": {
    "from": null,
    "to": null
  },

  "source_refs": [],
  "relation_refs": [],

  "version": 1,
  "created_at": null,
  "updated_at": null,
  "expires_at": null
}
```

这是逻辑 Schema，不是要求所有字段首版都实现。

具体要求：

1. memory_id 使用稳定唯一 ID，不使用 SQLite rowid 作为业务标识。
2. 所有记忆必须明确主体和 Scope。
3. 内容、验证状态、生命周期状态、可用性状态分别存储。
4. 时间信息支持事实有效时间和系统记录时间。
5. 来源通过 Evidence 引用关联。
6. 记忆更新必须产生新版本或可审计的状态变更。
7. 置信度、重要度和检索分数不得混为一谈。

## 2.4 三类状态必须分离

### Validation State

```text
unverified
confirmed
disputed
rejected
```

### Lifecycle State

```text
candidate
pending_review
active
superseded
archived
deleted
```

### Availability State

```text
available
restricted
suspended
```

例如，一条记忆可以是：

```text
validation = confirmed
lifecycle = active
availability = restricted
```

表示内容已经确认、仍然有效，但当前访问受到限制。

状态转换必须通过受控操作完成，不允许任意模块直接修改状态字段。

---

# 3. Evidence 与 Provenance 框架

## 3.1 设计目标

Evidence 负责保存或引用记忆的原始依据。

Provenance 负责说明记忆是如何形成的。

二者不能合并成一个简单的 source 字符串。

## 3.2 Evidence 来源类型

| 类型 | 说明 |
|---|---|
| explicit_user | 用户明确表达 |
| conversation | 对话中提取 |
| repeated_statement | 多次表达 |
| imported | 外部导入 |
| system_generated | 系统生成 |
| human_reviewed | 经人工审核 |

## 3.3 Evidence 对象

建议包含：

- evidence_id
- source_type
- source_ref
- source_session_id
- source_message_id
- source_subject_id
- captured_at
- content_hash
- content_excerpt 或受控内容引用
- extraction_method
- model_version
- visibility_scope
- retention_policy

原始对话正文不一定全部复制进 Evidence。应优先引用已有会话记录，只有在确有保存必要且权限允许时才保存内容副本。

## 3.4 Provenance 链

```text
Source Message
    ↓
Evidence
    ↓
Memory Candidate
    ↓
Memory Object
    ↓
Observation / Relation / Summary
    ↓
Retrieval Result
    ↓
Context Injection
```

每一层都要保存来源引用。

任何派生结果都不能丢失来源权限。

## 3.5 三种可信度必须独立

| 属性 | 回答的问题 |
|---|---|
| Confidence | 提取或推断有多确定？ |
| Authority | 来源是否有权威性？ |
| Validity | 该信息现在是否仍有效？ |

不允许把三个值简单平均成一个"可信度"。

用户明确陈述可以有较高来源权威性，但其内容仍可能过时。

---

# 4. M01 — Memory Storage

## 4.1 定位

M01 是唯一权威业务存储入口。

它负责数据持久化，不负责内容判断。

## 4.2 技术基线

| 能力 | 推荐方案 |
|---|---|
| 权威数据库 | SQLite |
| 事务 | SQLite Transaction |
| 全文索引 | SQLite FTS5 |
| 向量索引 | LanceDB |
| 异步同步 | Outbox |
| 版本 | 单调递增版本号 |
| ID | 稳定 UUID/ULID 类 ID |
| 备份 | SQLite 一致性备份 + LanceDB 备份/重建方案 |

SQLite 是权威存储；LanceDB 是独立向量检索层。

FTS5 和 LanceDB 均为派生检索数据，不能反向覆盖权威记忆。

## 4.3 M01 核心职责

1. Memory CRUD。
2. Evidence CRUD。
3. 关系数据持久化。
4. 事务管理。
5. 版本管理。
6. Outbox 事件写入。
7. 删除与状态变更。
8. 数据导入导出接口。
9. 备份与恢复。
10. 数据完整性校验。

## 4.4 建议数据表

```text
subjects
memory_objects
memory_versions
evidence
memory_evidence_links
memory_relations
observations
outbox_events
event_ledger
index_registry
embedding_spaces
schema_migrations
audit_metadata
```

说明：

- memory_objects：当前权威状态。
- memory_versions：记忆历史版本。
- evidence：证据记录。
- memory_evidence_links：记忆与证据的关联。
- memory_relations：关系对象。
- observations：派生观察。
- outbox_events：索引同步任务。
- event_ledger：业务事件历史。
- index_registry：派生索引状态。
- embedding_spaces：向量空间定义。
- schema_migrations：数据库迁移记录。
- audit_metadata：不包含不必要私人正文的审计元数据。

首版可以根据正式 Schema 缩减表数量，但不得把不同概念混在一个无约束 JSON 字段里。

## 4.5 事务规则

一次权威记忆更新应在同一事务中完成：

1. 校验输入。
2. 校验版本。
3. 写入或更新 Memory Object。
4. 写入版本记录。
5. 写入业务事件。
6. 写入 Outbox。
7. 提交事务。

索引同步在事务提交后异步执行。

## 4.6 Outbox 规则

每个 Outbox Event 必须包含：

- event_id
- aggregate_id
- aggregate_version
- operation
- payload_ref
- created_at
- retry_count
- status
- last_error

处理要求：

- 幂等。
- 支持重试。
- 支持失败记录。
- 支持版本校验。
- 支持删除传播。
- 支持死信处理。
- 支持索引重建。

不允许旧版本的异步任务覆盖新版本索引。

---

# 5. M02 — Retrieval & Context

## 5.1 定位

M02 负责读取记忆和构建授权后的上下文。

不负责修改权威记忆。

## 5.2 检索总流程

```text
Query
  ↓
Query Normalization
  ↓
Intent / Scope Parsing
  ↓
Metadata Filtering
  ↓
Parallel Retrieval
  ├── SQLite Structured Filter
  ├── FTS5 Keyword Retrieval
  └── LanceDB Vector Retrieval
  ↓
Candidate Deduplication
  ↓
RRF Fusion
  ↓
Rule-based Ranking
  ↓
SQLite Authoritative Fetch
  ↓
M05 Permission Check
  ↓
Context Budgeting
  ↓
Context Assembly
  ↓
DSH
```

## 5.3 多路召回

### A. 结构化检索

处理：

- subject_id
- scope
- type
- lifecycle_state
- availability_state
- time range
- sensitivity
- project/channel filters

### B. FTS5

负责关键词和文本匹配。

支持：

- 精确词项
- 关键短语
- 中文分词适配
- 标题与正文权重
- 类型和时间过滤

中文分词器通过接口隔离，不写死在检索核心。

### C. LanceDB

负责语义向量检索。

首版采用 FLAT 精确检索路线。

Embedding 通过独立接口调用，不在 M02 内绑定具体模型。

不同 Embedding 模型或不同向量空间不得混用。

## 5.4 RRF 融合

采用 Reciprocal Rank Fusion：

\[
RRF(d)=\sum_{r \in R}\frac{w_r}{k+rank_r(d)}
\]

其中：

- d：候选记忆。
- R：检索来源。
- w_r：检索来源权重。
- k：RRF 平滑参数。
- rank_r(d)：候选在该路检索中的排名。

建议初始配置：

```yaml
retrieval:
  fts_top_k: 50
  vector_top_k: 50
  rrf_k: 60
  fts_weight: 1.0
  vector_weight: 1.0
```

这些是实验起点，不是已完成裁决的固定参数。

## 5.5 规则排序

RRF 后进行规则排序。

建议考虑：

- 语义相关性。
- 关键词相关性。
- 记忆类型匹配。
- 时间有效性。
- 来源质量。
- 记忆重要度。
- 状态。
- 重复惩罚。
- 用户明确指定的偏好。

排序特征必须归一化，不能直接混合不同量纲的原始分数。

首版不引入 Cross-Encoder。

排序权重通过 Benchmark 验证，不把主观设置伪装成最优值。

## 5.6 权威回源与权限终检

召回结果只作为候选。

最终输出前必须：

1. 回 SQLite 读取权威记录。
2. 校验版本和状态。
3. 调用 M05 canRead。
4. 拒绝无权限记录。
5. 对派生数据执行权限继承检查。
6. 再进入上下文构建。

M05 尚未完整实现时，接口默认拒绝；仅隔离测试环境允许显式配置测试授权。

不得使用默认 allow-all 作为生产回退。

## 5.7 Context Builder

Context Builder 不是检索器。

它负责：

- 去重。
- 组织内容。
- 分配 Token 预算。
- 保留来源引用。
- 控制不同记忆类型的比例。
- 防止无关记忆占据上下文。
- 生成结构化输出。

建议初始配置：

```yaml
context:
  total_memory_token_budget: 1200
  max_items: 20
  include_source_refs: true
  preserve_scope: true
  deduplicate: true
```

1200 是实验起点，不是永久限制。

输出建议分为：

```text
USER_PROFILE
RELEVANT_EPISODES
ACTIVE_INTENTS
RELEVANT_PROCEDURES
IMPORTANT_RELATIONSHIPS
OBSERVATIONS
SOURCE_REFERENCES
```

具体分区可以根据 Token 预算为空，不要求每次都注入所有分区。

---

# 6. M03 — Memory Extraction

## 6.1 定位

M03 将输入转化为候选记忆。

它不直接决定最终记忆是否成立。

## 6.2 输入

- 对话消息。
- 会话摘要。
- 用户明确保存请求。
- 外部导入内容。
- 经过授权的系统事件。

## 6.3 处理步骤

```text
Input
  ↓
Source Validation
  ↓
Information Segmentation
  ↓
Memory Type Classification
  ↓
Candidate Extraction
  ↓
Normalization
  ↓
Evidence Linking
  ↓
Confidence Estimation
  ↓
Candidate Deduplication
  ↓
Candidate Queue
```

## 6.4 候选类型

- Fact Candidate
- Preference Candidate
- Event Candidate
- Intent Candidate
- Relationship Candidate
- Procedure Candidate
- Observation Candidate

## 6.5 候选质量检查

每个候选至少检查：

1. 是否有来源。
2. 是否能够独立理解。
3. 是否存在主体。
4. 是否具有明确范围。
5. 是否有时间信息。
6. 是否存在推断成分。
7. 是否与已有记忆重复。
8. 是否可能属于临时内容。
9. 是否涉及敏感信息。
10. 是否超出当前授权范围。

## 6.6 提取策略

建议分三种：

| 策略 | 处理方式 |
|---|---|
| Explicit Save | 用户明确要求保存 |
| Automatic Extraction | 系统提取候选 |
| Human/Owner Review | 需要进一步确认 |

明确保存请求也不能跳过身份、权限和数据校验。

自动提取不代表自动确认。

## 6.7 禁止行为

- 不得把推测转成事实。
- 不得丢失 Evidence。
- 不得覆盖已有记忆。
- 不得绕过 M04 的整合规则。
- 不得绕过 M05 的写入授权。
- E14/B04 reinforce 在正式授权前不得写入实现。

---

# 7. M04 — Memory Consolidation

## 7.1 定位

M04 负责对候选记忆和现有记忆进行整合。

## 7.2 核心能力

1. 候选匹配。
2. 重复识别。
3. 近似内容聚类。
4. 记忆合并。
5. 冲突检测。
6. 冲突分类。
7. Observation 生成。
8. 关系维护。
9. 过期记忆识别。
10. 派生视图更新。

## 7.3 重复与冲突分类

| 类型 | 处理 |
|---|---|
| 完全重复 | 去重 |
| 语义重复 | 评估合并 |
| 信息补充 | 补充或关联 |
| 时间变化 | 保留历史并更新当前状态 |
| 明确矛盾 | 标记 disputed，进入处理流程 |
| 证据不足 | 保留候选，不强行合并 |
| 无法判断 | 维持并存，等待更多证据 |

## 7.4 冲突处理原则

不能简单使用"最新内容覆盖旧内容"。

需要综合：

- 来源权威性。
- 时间有效性。
- 用户是否明确纠正。
- 证据完整性。
- 信息是否属于不同时间段。
- 信息是否针对不同主体或范围。

旧版本应保留可审计的历史关系。

## 7.5 Observation

Observation 是派生结论，不是原始事实。

每条 Observation 应保存：

- observation_id
- content
- source_memory_ids
- source_evidence_ids
- derivation_method
- confidence
- created_at
- version
- validation_state
- scope
- availability_state

当来源记忆发生变化、被删除或权限收紧时，相关 Observation 必须被标记为待重算、失效或受限。

---

# 8. M05 — Governance & Permission

## 8.1 定位

M05 是整个系统唯一的权限策略决策入口。

其他模块不得自行复制一套权限判断逻辑。

## 8.2 权限检查接口

建议：

```ts
canRead(subject, resource, context)
canWrite(subject, resource, context)
canUpdate(subject, resource, context)
canDelete(subject, resource, context)
canShare(subject, resource, targetScope, context)
canExport(subject, resource, context)
```

统一返回：

```ts
{
  allowed: boolean,
  reason_code: string,
  policy_version: string,
  decision_id: string
}
```

权限失败即拒绝。

## 8.3 主体模型

至少区分：

- Memory Owner
- Requesting User
- Group Member
- Group Administrator
- Bot Owner
- System Service
- Maintenance Operator

角色不等于数据所有权。

群管理员不自动获得成员私人记忆访问权。

Bot Owner 的维护权限不自动包含私人记忆正文读取权限。

## 8.4 派生数据继承

以下数据必须继承来源权限：

- Evidence
- Observation
- Relation
- Cue
- Summary
- FTS5 Index
- LanceDB Vector
- Recall Cache
- Context Cache
- Backup
- Export

如果一个派生对象引用多个来源，默认使用所有来源权限的最严格交集。

如果无法确认权限，拒绝访问。

## 8.5 权限变更传播

当记忆权限变化时：

1. 更新权威状态。
2. 写入事件。
3. 使相关缓存失效。
4. 更新或删除派生索引。
5. 更新 Observation 和关系的可用状态。
6. 记录审计。
7. 检查未完成任务。

权限收紧不能只影响下一次数据库查询，还必须传播到已有派生数据。

---

# 9. 生命周期与记忆衰减框架

## 9.1 生命周期

```text
Candidate
   ↓
Pending Review
   ↓
Active
   ├── Updated
   ├── Superseded
   ├── Disputed
   ├── Archived
   └── Deleted
```

具体状态转换必须由对应模块授权。

## 9.2 生命周期策略按类型区分

| 类型 | 建议策略 |
|---|---|
| 稳定事实 | 低频更新，长期保存 |
| 偏好 | 随新表达修正 |
| 事件 | 按时间保存 |
| 意图 | 需要完成、取消或过期状态 |
| 临时计划 | 设置有效期 |
| 程序性经验 | 根据实际有效性维护 |
| Observation | 来源变化时重算 |

## 9.3 衰减与重要度

不要让所有记忆都采用同一个时间衰减公式。

建议区分：

- Importance：内容本身的重要性。
- Recency：时间新近程度。
- Relevance：当前请求相关性。
- Utility：历史使用价值。
- Validity：当前有效性。

检索频次不等于真实性。

被频繁召回不意味着记忆就更真实。

用户明确要求保存的内容，不应因为低召回频次自动删除。

首版不实施复杂自动遗忘。先实现状态、过期识别和人工可控的归档策略。

---

# 10. Dream — 周期维护框架

## 10.1 定位

Dream 是调度器和维护编排器，不是一个拥有任意数据库写权限的超级模块。

## 10.2 任务类型

- 待处理候选检查。
- 重复候选整理。
- 冲突检测。
- Observation 重算。
- 过期意图检查。
- 索引一致性检查。
- Outbox 积压处理。
- 数据完整性检查。
- 维护报告生成。

## 10.3 执行模型

```text
Scheduler
   ↓
Task Planner
   ↓
Permission Check
   ↓
Module Invocation
   ↓
Result Validation
   ↓
Authorized Commit
   ↓
Audit Event
```

Dream 不允许：

- 直接修改 M01 内部表。
- 绕过 M04 合并记忆。
- 绕过 M05 权限。
- 将推断自动提升为事实。
- 擅自启用未授权的 reinforce。
- 对私人记忆执行无边界批处理。

---

# 11. Session Memory 与上下文管理

## 11.1 三层记忆结构

| 层次 | 负责内容 |
|---|---|
| Working Memory | 当前推理请求需要的临时上下文 |
| Session Memory | 当前会话状态和近期摘要 |
| Long-term Memory | 可跨会话持续使用的记忆 |

三层可以相互引用，但不能混用存储职责。

## 11.2 Context Assembly

最终 Prompt 由上下文组装器构建。

输入来源包括：

- System Prompt。
- Persona。
- Session Memory。
- Long-term Memory。
- 当前对话。
- 工具结果。
- 当前任务状态。

Memory Core 只提供经过授权的记忆内容及其来源信息。

不负责决定所有 Prompt 的最终顺序。

## 11.3 Context Compression 独立

上下文压缩属于外部协作能力。

它与 Memory Core 的关系：

```text
Memory Core
   ↓
Authorized Memory Context
   ↓
Context Manager
   ↓
Compression / Compaction
   ↓
DSH Prompt
```

压缩结果不能自动成为长期记忆。

如果压缩结果需要进入长期记忆，必须重新经过 M03、M04 和 M05 的受控流程。

---

# 12. Psychology 与 Persona

## 12.1 Psychology

Psychology 是模拟状态层，不是用户事实数据库。

可以维护：

- 模拟情绪状态。
- 互动倾向。
- 状态变化记录。
- 有限时间范围的上下文信号。

必须区分：

- 模拟状态。
- 用户明确表达。
- 系统推断。
- 长期记忆事实。

Psychology 不得修改用户记忆的真实性或权限。

## 12.2 Persona

Persona 负责角色表达，不负责权威记忆管理。

其输入可以包括：

- 用户允许读取的稳定偏好。
- 当前会话状态。
- 已授权关系信息。
- 经过筛选的历史事件。
- Psychology 提供的模拟状态。

Persona 不得直接访问 SQLite 或 LanceDB 内部表。

---

# 13. 关系与知识结构框架

首版不建设复杂知识图谱，但应保留关系对象接口。

建议关系类型：

```text
RELATED_TO
DERIVED_FROM
CONFIRMS
CONTRADICTS
SUPERSEDES
BUILDS_ON
PART_OF
ABOUT
```

关系对象至少保存：

- relation_id
- source_memory_id
- target_memory_id
- relation_type
- scope
- provenance_refs
- created_at
- version
- availability_state

关系必须经过权限检查。

当关系任一端不可访问时，不得通过关系边泄露受限对象信息。

未来只有当多跳关系检索、实体消歧和跨事件推理有明确需求时，才考虑图数据库或专门的图索引。

---

# 14. 搜索文档与索引框架

## 14.1 Search Document

所有检索内容使用统一的 Search Document 规范。

建议字段：

```text
document_id
memory_id
memory_version
schema_version
embedding_space_id
title
content
summary
tags
type
subject_id
scope
sensitivity
lifecycle_state
availability_state
source_refs
created_at
updated_at
```

FTS5 和 LanceDB 必须使用相同的权威版本来源。

## 14.2 Embedding Space

每个向量空间必须明确：

- embedding_space_id
- model_name
- model_version
- dimension
- distance_metric
- normalized
- chunker_version
- text_projection_version
- created_at

同维度不代表同一语义空间。

不同模型、不同预处理版本不得未经验证混合检索。

## 14.3 索引同步

统一采用：

```text
SQLite Transaction
   ↓
Outbox Event
   ↓
Index Worker
   ├── FTS5 Update
   └── LanceDB Upsert/Delete
   ↓
Index Version Check
   ↓
Mark Complete
```

失败必须可以重试。

索引落后时，系统必须能识别其版本差异。

不得静默把 LanceDB 失败替换成 SQLite 向量检索。

---

# 15. 事件与接口协议

## 15.1 事件模型

建议统一事件信封：

```json
{
  "event_id": "unique-id",
  "event_type": "memory.updated",
  "aggregate_id": "memory-id",
  "aggregate_version": 2,
  "schema_version": "1",
  "occurred_at": null,
  "producer": "M01",
  "correlation_id": "request-id",
  "causation_id": "previous-event-id",
  "payload_ref": "reference",
  "scope": "owner_private"
}
```

## 15.2 事件命名

建议：

```text
memory.candidate.created
memory.created
memory.updated
memory.superseded
memory.archived
memory.deleted

evidence.created
observation.created
observation.invalidated

index.upsert.requested
index.delete.requested
index.sync.completed
index.sync.failed

permission.changed
permission.denied

retrieval.started
retrieval.completed
context.assembled

dream.task.started
dream.task.completed
dream.task.failed
```

## 15.3 接口稳定性

接口至少包含：

- 输入 Schema。
- 输出 Schema。
- 版本。
- 错误码。
- 幂等要求。
- 权限要求。
- 超时要求。
- 重试要求。
- 可观测字段。

不得只定义 TypeScript 函数签名而没有行为契约。

---

# 16. 统一错误处理框架

错误分为：

| 类别 | 示例 | 处理 |
|---|---|---|
| Validation | 输入无效 | 拒绝并返回原因 |
| Permission | 无访问权限 | 立即拒绝 |
| Not Found | 记录不存在 | 返回明确状态 |
| Conflict | 版本冲突 | 重新读取并处理 |
| Storage | SQLite 失败 | 事务回滚 |
| Index | LanceDB 失败 | 进入重试 |
| Embedding | 模型不可用 | 标记任务失败或受控降级 |
| Timeout | 操作超时 | 依照幂等策略重试 |
| Internal | 未预期错误 | 记录并隔离 |

所有错误使用稳定的 error_code。

禁止用空数组、默认值或虚假的成功状态掩盖失败。

---

# 17. 安全、隐私与删除框架

## 17.1 安全原则

- 默认拒绝。
- 最小权限。
- 身份显式传递。
- 访问范围显式传递。
- 关键操作审计。
- 私人内容与共享内容隔离。
- 不可信输入不作为系统指令。
- 工具调用不自动获得记忆权限。

## 17.2 删除流程

删除必须区分：

1. 逻辑删除。
2. 检索不可见。
3. 派生索引清理。
4. 缓存失效。
5. 备份保留策略。
6. 物理清除。

删除操作需要有状态和完成判据。

不能只删除 SQLite 主记录而保留可检索的向量。

也不能为了可恢复性而违反已经确认的数据清除要求。

## 17.3 审计

记录：

- 操作主体。
- 操作类型。
- 目标对象 ID。
- 时间。
- 决策结果。
- policy_version。
- request_id。
- 结果状态。

审计日志不应无必要地复制私人正文。

---

# 18. 可靠性、备份与恢复

## 18.1 数据恢复目标

系统需要支持：

- SQLite 备份恢复。
- FTS5 重建。
- LanceDB 重建。
- Outbox 续跑。
- 版本一致性校验。
- 删除传播补偿。
- Embedding 迁移恢复。

## 18.2 恢复流程

```text
Stop Writers
   ↓
Restore SQLite
   ↓
Validate Schema / Integrity
   ↓
Read Index Registry
   ↓
Rebuild or Reconcile FTS5
   ↓
Rebuild or Reconcile LanceDB
   ↓
Verify Versions
   ↓
Process Pending Outbox
   ↓
Run Acceptance Checks
   ↓
Resume Service
```

如果一致性检查失败，不得把系统标记为完全恢复。

## 18.3 备份安全

每次修改核心文件前：

1. 生成独立备份。
2. 计算 SHA256。
3. 校验备份可读。
4. 校验哈希一致。
5. 才允许修改。
6. 修改后重新检查文件和测试结果。

备份失败即停止修改。

---

# 19. 可观测性与评估框架

## 19.1 统一 Trace

每次请求应有：

```text
request_id
session_id
subject_id
retrieval_id
memory_ids
permission_decisions
context_build_id
prompt_injection_event_id
```

敏感身份信息应按最小必要原则记录和保护。

## 19.2 核心指标

### M03

- Candidate Count
- Candidate Acceptance Rate
- Evidence Coverage
- Extraction Error Rate

### M04

- Duplicate Rate
- Merge Rate
- Conflict Rate
- Observation Validity

### M02

- Recall@K
- Precision@K
- MRR
- NDCG
- Context Precision
- Context Coverage

### M01

- Transaction Failure Rate
- Outbox Lag
- Index Sync Failure
- Rebuild Duration

### M05

- Permission Denial Rate
- Unauthorized Access Attempts
- Derived Permission Propagation Errors

### 系统整体

- Token Efficiency
- End-to-End Latency
- Embedding Cost
- Storage Growth
- Recovery Time
- Data Integrity Errors

## 19.3 评估原则

必须区分：

- 检索到了正确记忆。
- 权限正确。
- 正确记忆进入上下文。
- 模型正确使用记忆。
- 最终回答正确。

这些不是同一个指标。

---

# 20. 版本、迁移与可扩展性

## 20.1 版本维度

至少分别维护：

```text
database_schema_version
memory_schema_version
event_schema_version
embedding_space_version
search_algorithm_version
permission_policy_version
context_contract_version
module_version
```

不要用一个总版本号代替所有版本。

## 20.2 迁移原则

- 先兼容，再迁移。
- 先构建新索引，再切换。
- 迁移期间保证写入一致。
- 保留回滚条件。
- 明确迁移完成判据。
- 不允许未经验证混合不同向量空间。

## 20.3 插件化扩展

可替换接口包括：

```text
StorageAdapter
VectorStoreAdapter
EmbeddingAdapter
TokenizerAdapter
RetrievalStrategy
RankingStrategy
ConsolidationStrategy
PermissionProvider
SchedulerProvider
ContextAdapter
```

每个 Adapter 必须有一致的契约测试。

---

# 21. 配置管理

建议统一配置分层：

```text
core
storage
retrieval
embedding
indexing
extraction
consolidation
permission
lifecycle
dream
context
observability
backup
integration
```

配置要求：

- 有默认值。
- 有 Schema 校验。
- 有版本。
- 有环境隔离。
- 不在代码中散落重复配置。
- 安全相关配置不能由普通业务配置静默覆盖。
- 生产环境与测试环境严格分离。

测试授权必须显式启用，不能成为默认配置。

---

# 22. 测试体系

## 22.1 单元测试

每个模块独立验证内部规则。

## 22.2 契约测试

验证模块之间的输入输出、版本、权限和错误行为。

## 22.3 集成测试

验证：

- SQLite + Outbox。
- FTS5 同步。
- LanceDB 同步。
- M02 多路召回。
- M05 权限终检。
- Context Builder。
- 删除传播。

## 22.4 故障测试

验证：

- SQLite 写入失败。
- LanceDB 不可用。
- Embedding 超时。
- Outbox 重复消费。
- 旧版本事件晚到。
- 权限发生变化。
- 索引损坏。
- 恢复中断。
- 重复删除。
- 并发更新冲突。

## 22.5 端到端测试

至少覆盖：

1. 用户明确要求保存。
2. 自动提取候选。
3. 重复记忆。
4. 新信息补充。
5. 冲突信息。
6. 过期意图。
7. 私人记忆检索。
8. 群聊隔离。
9. 权限拒绝。
10. 删除后不可召回。
11. 索引重建。
12. 上下文预算控制。

---

# 23. 实施路线图

## P0 — 工程环境与基础契约

目标：

- 确认 Node.js 版本。
- 确认 SQLite 驱动。
- 验证 LanceDB Node SDK 的 FLAT 精确检索能力。
- 建立独立原型目录。
- 建立统一配置。
- 建立 Schema v1。
- 建立基础接口。
- 建立备份和哈希流程。

交付：

- 环境报告。
- Schema。
- 接口契约。
- 数据库驱动裁决记录。
- LanceDB 能力验证。
- 目录结构。

## P1 — M01

目标：

- SQLite 权威存储。
- CRUD。
- 事务。
- 版本。
- Evidence 基础结构。
- Outbox。
- 备份恢复。
- 数据完整性检查。

## P2 — M02

目标：

- FTS5。
- Embedding Adapter。
- LanceDB FLAT。
- RRF。
- 规则排序。
- SQLite 权威回源。
- 权限终检接口。
- Token Budget。
- Context Builder。

## P3 — 最小闭环验收

目标：

验证：

```text
写入
  ↓
SQLite
  ↓
Outbox
  ↓
FTS5 + LanceDB
  ↓
多路检索
  ↓
RRF
  ↓
SQLite 回源
  ↓
权限终检接口
  ↓
Token Budget
  ↓
Context Output
```

使用真实原型数据和实际运行结果。

Mock 只能用于接口测试，不能冒充真实验收。

## P4 — M03

候选提取与证据关联。

## P5 — M04

记忆整合、冲突处理与 Observation。

## P6 — M05

完整权限治理与派生数据权限传播。

## P7 — Dream

周期维护与调度。

## P8 — Session / Psychology / Persona

外部协作能力接入。

## P9 — DSH / QQ Bridge 集成

在独立原型通过验收后接入真实宿主。

## P10 — 长期优化

- 检索参数优化。
- 生命周期策略优化。
- Embedding 迁移。
- 关系能力扩展。
- Context Compression 协作。
- 性能和成本优化。

**P0–P3 是当前实施授权范围。P4 以后必须单独获得授权。**

---

# 24. 最终推荐的目录结构

```text
memory-core/
│
├── docs/
│   ├── 00-architecture/
│   ├── 01-domain-model/
│   ├── 02-algorithms/
│   ├── 03-module-contracts/
│   ├── 04-governance/
│   ├── 05-reliability/
│   ├── 06-evaluation/
│   ├── 07-migration/
│   └── 08-integration/
│
├── src/
│   ├── core/
│   ├── contracts/
│   ├── modules/
│   │   ├── m01-storage/
│   │   ├── m02-retrieval/
│   │   ├── m03-extraction/
│   │   ├── m04-consolidation/
│   │   └── m05-governance/
│   │
│   ├── services/
│   │   ├── evidence/
│   │   ├── lifecycle/
│   │   ├── audit/
│   │   ├── evaluation/
│   │   └── context/
│   │
│   ├── adapters/
│   │   ├── sqlite/
│   │   ├── lancedb/
│   │   ├── embedding/
│   │   ├── tokenizer/
│   │   └── dsh/
│   │
│   ├── workers/
│   │   ├── outbox/
│   │   ├── indexing/
│   │   └── dream/
│   │
│   └── shared/
│
├── schemas/
├── migrations/
├── configs/
├── tests/
│   ├── unit/
│   ├── contract/
│   ├── integration/
│   ├── failure/
│   └── e2e/
│
├── scripts/
├── data/
├── backups/
└── PROTOTYPE-PROGRESS.md
```

目录是推荐组织方式，不要求首轮一次创建所有空模块。

---

# 25. 最终架构验收原则

一个完整的 Memory Core 必须同时满足：

1. 记忆有明确身份、范围和类型。
2. 记忆有来源和证据链。
3. 候选与已确认记忆明确区分。
4. 权威数据与派生索引分离。
5. 写入、更新和删除可追溯。
6. 检索结果必须回权威存储。
7. 权限终检不能被绕过。
8. 派生数据继承来源权限。
9. 冲突不会被静默覆盖。
10. 生命周期状态可解释。
11. 索引可以重建。
12. 模块可以独立测试。
13. 接口可以版本化。
14. 模型与数据库可以替换。
15. 故障不会伪装成成功。
16. 系统能够量化评估效果。
17. 上下文压缩与长期记忆保持职责分离。
18. 未授权功能不会因为架构预留而自动启用。

---

# 26. 最终架构结论

Memory Core 的目标不是"存储更多聊天内容"，而是建立一套可持续积累、检索、治理和修正的信息系统。

推荐的整体结构为：

```text
                 DSH / Agent Runtime
                         │
                 Context Adapter
                         │
              Context Assembly Layer
                         │
                 M02 Retrieval
                         │
                   M05 Permission
                         │
              SQLite Authoritative Data
                         │
        ┌────────────────┼────────────────┐
        │                │                │
      M03              M04              M01
   Extraction      Consolidation      Storage
        │                │                │
        └────────────────┼────────────────┘
                         │
                  Evidence Layer
                         │
                 Event / Outbox
                         │
                 Index Workers
                    /       \
                  FTS5     LanceDB

       Dream / Audit / Evaluation / Lifecycle
                    横向治理
```

最终坚持四个判断：

**第一，权威数据只有一个中心。** SQLite 管理权威业务状态，LanceDB 和 FTS5 负责检索，不互相替代。

**第二，模块可以独立演进，但必须共享统一契约。** 模块化的价值在于降低变化的影响范围，而不是允许每个模块自行定义数据语义。

**第三，记忆不是事实的简单集合，而是带有来源、范围、状态、时间和权限的对象集合。**

**第四，先建立最小真实闭环，再逐步增加智能能力。** 不以架构复杂度衡量成熟度，以真实数据、可靠性、权限正确性和实际使用效果衡量成熟度。

这就是我推荐的 Memory Core R4.1 完整目标架构。
