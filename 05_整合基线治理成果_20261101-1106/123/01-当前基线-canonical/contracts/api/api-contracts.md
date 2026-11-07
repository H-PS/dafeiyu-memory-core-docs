# Memory Core R4.1 — API 契约总集（M01–M05 / Context Adapter / Database Adapter）

> 文件：`contracts/api/api-contracts.md` ｜ 版本 1.0 ｜ 状态：结构 APPROVED（依据 m.md §15 + sf 后文 §3.1.4/§4.2/§12.1；名称级细节标 PROPOSED）
> 通用约定：所有接口 MUST 满足 sf 后文 §12.1 十三项（名称/目的/输入/输出/必填/可选/类型/默认值/错误码/幂等/权限/版本/实施范围）。响应统一信封：

```json
{
  "request_id": "uuid",
  "status": "success | failed | partial",
  "data": {},
  "meta": { "schema_version": "1.0", "algorithm_version": "1.0" },
  "errors": [ { "code": "...", "message": "...", "trace_id": "..." } ]
}
```

统一请求上下文（m.md §15.3，全部接口必带）：`request_id, tenant_id, actor{type,id}, subject_id, session_id, scope[], timestamp, idempotency_key`。身份 MUST 由可信 Adapter 构造。

**实施范围标记**：`[P1]`=P0–P3 内实施；`[CONTRACT]`=P0–P3 仅预留契约（默认拒绝/占位）；`[FUTURE]`=设计先行不实施。

---

## 1. REST API（Memory API 对 Host Adapter，m.md §15.2）[P1 起逐步开放]

### 1.1 POST /api/v1/memories/retain
- 目的：提交记忆候选或授权来源。**不等于直接成为 active**（进入 candidate/pending_review）。
- 输入：`{ actor_context, items: [ { content, type, content_kind, scope, subject_id, evidence_refs[], source_refs[], extraction_method } ], provenance }`
- 输出：`{ accepted: [candidate_id...], rejected: [{item_index, code, reason}] }`
- 幂等：idempotency_key 必填。权限：M05 canWrite（契约期=contract_deny 语义校验）。错误：VALIDATION_ERROR, PERMISSION_DENIED, STORAGE_ERROR。
- M03 完整自动提取=[FUTURE]；P1 仅接受显式 items + 手工证据。

### 1.2 POST /api/v1/memories/recall
- 目的：检索授权记忆（M02 全流水线）。输入=`retrieval-request.schema.json`；输出=`retrieval-response.schema.json` + `context-response.schema.json`（两段可一次返回或分步）。
- 幂等：可选（读操作默认不强制）。权限：M05 canRead 逐条终检。错误：VALIDATION_ERROR, RETRIEVAL_FAILED, INDEX_ERROR, EMBEDDING_ERROR, TIMEOUT, PERMISSION_DENIED。
- 超时：fts/query_timeout_ms、embedding timeout_ms 各自生效；整体默认 10s [EXP-DEFAULT]。

### 1.3 POST /api/v1/memories/confirm
- 目的：确认待审核记忆。**[FUTURE]**——confirmed 仅允许 M04 审核通道写入（台账 D-AP-05）；M04 未实现前该端点返回 PERMISSION_DENIED。
- 输入：`{ memory_id, expected_version, evidence_refs[], reviewer }`。

### 1.4 POST /api/v1/memories/revise
- 目的：修订已有记忆。输入：`{ memory_id, expected_version, patch{content?,valid_from?,valid_to?,importance?}, reason }`；产生新版本 + provenance；禁止静默覆盖。错误：VERSION_CONFLICT, VALIDATION_ERROR, PERMISSION_DENIED。

### 1.5 POST /api/v1/memories/revoke
- 目的：撤销。**语义必须二选一显式声明**（OI-03-01，TBD 待用户裁决默认=撤销可用性）：`mode: "availability" | "authorization"`。
  - availability：availability_state→suspended（可逆）。
  - authorization：删除/撤权路径（触发 SM-7 删除传播，不可逆）。
- 权限：所属主体或授权治理流程。

### 1.6 GET /api/v1/memories/{memory_id}
- 目的：读取权威记忆（含版本、状态、source_refs；**不含**无权正文字段）。权限：M05 canRead。错误：NOT_FOUND, PERMISSION_DENIED。支持 `?version=n`（getMemoryVersion）。

### 1.7 GET /api/v1/tasks/{task_id}
- 目的：查询异步任务状态（SM-5 五态；cancelled≠succeeded）。错误：TASK_NOT_FOUND。

### 1.8 GET /api/v1/health
- 目的：服务状态（sqlite/fts5/lancedb/embedding/m05_mode/worker/outbox_backlog/outbox_oldest_age）。**MUST NOT 返回任何私人记忆内容**。降级态返回 200 + degraded 标志。

---

## 2. M01 Storage API（模块内部）[P1]

| 方法 | 签名 | 事务 | 幂等 | 权限 | 错误 |
|---|---|---|---|---|---|
| createMemory | (input:MemoryObjectCreate)→MemoryObject | 同事务写 Outbox | idempotency_key | canWrite | VALIDATION_ERROR/PERMISSION_DENIED/STORAGE_ERROR/DUPLICATE_REQUEST |
| getMemory | (memoryId)→MemoryObject | 读 | — | canRead（契约） | NOT_FOUND/PERMISSION_DENIED |
| getMemoryVersion | (memoryId, version)→MemoryObjectVersion | 读 | — | canRead | NOT_FOUND |
| queryMemories | (query:StructuredQuery)→MemoryObject[] | 读 | — | 范围过滤（不替代 M05） | VALIDATION_ERROR |
| updateMemory | (memoryId, patch, expectedVersion)→MemoryObject | 同事务；乐观并发 | expectedVersion | canUpdate | VERSION_CONFLICT/VALIDATION_ERROR/STORAGE_ERROR |
| deleteMemory | (memoryId, context)→DeletionHandle | 同事务+tombstone+Outbox delete | tombstone | canDelete | NOT_FOUND/PERMISSION_DENIED |
| listMemoryVersions | (memoryId)→Version[] | 读 | — | canRead | NOT_FOUND |
| createEvidence | (input:Evidence)→Evidence | 同事务 | content_hash 去重 | canWrite | VALIDATION_ERROR |
| getEvidence | (evidenceId)→Evidence | 读 | — | canRead | NOT_FOUND |
| createRelation | (input:Relation)→Relation | 同事务；校验两端 Scope 交集 | 端点对+类型幂等 | canWrite+两端 canRead | VALIDATION_ERROR/PERMISSION_DENIED |
| appendObservation | (input:Observation)→Observation | 同事务 | derivation+sources 哈希 | canWrite | VALIDATION_ERROR |
| getVersion | (memoryId)→int | 读 | — | — | NOT_FOUND |
| verifyIntegrity | (scope?)→IntegrityReport | 读 | — | 维护角色（不含私人正文输出） | STORAGE_ERROR |

通用要求：所有输入 Schema 校验；禁止任意 SQL；调用方禁止直连内部表；所有失败返回结构化错误（error.schema.json）。

## 3. M02 Retrieval API（模块内部）[P2]

| 方法 | 输入 | 输出 | 说明 |
|---|---|---|---|
| retrieve | RetrievalRequest | RecallResult（retrieval-response.schema） | 算法一：规范化→structured 约束→fts/vector 双通道召回→RRF 合并。通道失败按批准降级；未批准=failed |
| rankCandidates | RecallResult | RankedCandidates[]（Top20） | 算法二：RRF(k,weights)→六特征规则排序（正/罚分归一化）→稳定排序 |
| buildContext | RankedCandidates+ContextBuildRequest | MemoryContext（context-response.schema） | 算法三：回源→canRead 终检→GreedySelect→分区组装 |
| invalidateCaches | (memoryIds[])→void | — | 权限变更传播第 3 步调用点 |

## 4. M03 API [FUTURE — 契约预留]

`submitSource(input)→candidates[]`；`runQualityCheck(candidate)→quality_flags`；`linkEvidence(candidate, evidence)→candidate`。M03 不得直接写 active；候选仅入队（SM-1 状态 1）。J7：情绪标注仅预留接口字段（affect_tags/affect_source），不实现 LLM 标注（台账 §3e）。

## 5. M04 API [FUTURE — 契约预留]

`reviewCandidate(candidate)→decision(active|reject|merge|pending)`；`detectDuplicates(scope)→DuplicateGroup[]`；`classifyConflict(memoryA,memoryB)→conflict_type`；`proposeMerge(group)→MergeProposal`（须授权执行）；`recomputeObservation(observationId)→Observation`。**confirmed 写入权唯一归 M04**（台账 D-AP-05）。

## 6. M05 Permission API [CONTRACT — P0–P3 契约默认拒绝]

```
canRead / canWrite / canUpdate / canDelete / canShare / canExport
  (subject: PermissionContext, resource: {memory_id|scope|resource_type}, context) →
  { allowed: boolean, reason_code: string, policy_version: string, decision_id: string }
```
- 结果三态业务语义：ALLOW / DENY / INDETERMINATE（异常、超时、未知 ⇒ INDETERMINATE ⇒ 排除）。
- **m05_mode=contract_deny 时恒返回 `{allowed:false, reason_code:"M05_NOT_IMPLEMENTED"}`**（生产默认拒绝；测试替身仅 test 环境）。
- C40 权限引擎不可插拔（台账 D-AP-07）；派生数据继承最严格交集；无法确认=拒绝。

## 7. Context Adapter API（宿主面）[P2 契约 / P9 接入]

`buildMemoryContext(hostRequest)→context-response.schema.json`；`submitRetainTask(authorizedSources)→task_id`。宿主 MUST 提供 user_id/group_id/session_id/bot_id/tenant_id 稳定映射；返回八字段（context_id/request_id/memory_refs/versions/token_usage/retrieval_status/permission_status/adapter_version）。L3：注入成功发 `context.assembled` + `prompt_injection_event_id`（只证明进入 Prompt）。

## 8. Database Adapter API（基础设施）[P0 VERIFY]

```
open(config) / beginTransaction() / commit() / rollback()
run(sql:ParameterizedStatement)   // 禁止任意 SQL 拼接
migrate(script) / backup(target) / verifyIntegrity()
```
驱动候选 node:sqlite / better-sqlite3 / sqlite3 —— P0 真实验证后裁决并记录；未验证不得声称满足全部要求。**禁止实现 SQLite Vector Adapter 替代 LanceDB**（sf 后文 §16.5；`fallback_to_sqlite_vector` 恒 false）。

## 9. 版本兼容

- API 版本：`/api/v1/`；破坏性变更必须升版本，禁止改旧版字段语义（m.md §20.3）。
- 接口 MUST 声明：输入/输出 Schema 版本、错误码、幂等要求、权限要求、超时、重试、可观测字段（sf 后文 §15.3）。
- 字段修改必须同步：Schema/API/Event/Tests/Migration/Documentation（sf 后文 §12）。
