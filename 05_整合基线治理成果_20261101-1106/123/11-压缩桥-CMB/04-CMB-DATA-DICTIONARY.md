# CMB v0.1 数据字典（04-CMB-DATA-DICTIONARY）

采集：2026-11-06 ｜ 主代理直接从 `src/core/domain.ts` 与上游 `schema.ts` 提取

---

## 1. 字段来源分类说明

| 标记 | 含义 |
|---|---|
| **UPSTREAM** | 上游真实字段，可从源库直接读取 |
| **CMB** | CMB 内部派生/构造 |
| **HOST** | 宿主注入（不在上游库中） |
| **MOCK** | 仅 Mock 夹具存在，真实源无 |
| **DERIVED** | 由其他字段计算得出 |
| **INFERRED** | 推测，未经真实数据验证 |

---

## 2. SourceSession

| 字段 | 类型 | 必填 | 可空 | 来源 | 真实验证 | 备注 |
|---|---|---|---|---|---|---|
| `session_id` | string | ✓ | ✗ | UPSTREAM | 是（schema.ts:109 `sessions.session_id` PK） | 上游唯一真实身份 |
| `tenant_id` | string\|null | ✓ | ✓ | HOST | **否** | `bc-source.ts:39` 恒 null；`src/` 全仓从未读取 `SourceSession.tenant_id` |
| `bot_id` | string\|null | ✓ | ✓ | HOST | **否** | 同上，恒 null |
| `group_id` | string\|null | ✓ | ✓ | HOST/MOCK | **否** | `bc-source.ts:41` 恒 null；Mock 夹具自带 |
| `session_type` | `"group"\|"private"\|"unknown"` | ✓ | ✗ | HOST | **否** | `bc-source.ts:42` **硬编码 `"unknown"`** → 真实源下所有会话不可解析 |
| `source_version` | string | ✓ | ✗ | CMB | 是（常量） | `billion-context/0aace2f/unified-v1` |
| `parent_session_id` | string\|null | ✓ | ✓ | UPSTREAM | 是（`schema.ts:110`） | fork 父会话，CMB **读取但零使用** |
| `fork_point_seq` | number\|null | ✓ | ✓ | UPSTREAM | 是（`schema.ts:111`） | fork 点，CMB **读取但零使用** |
| `created_at` | number\|null | ✓ | ✓ | UPSTREAM | 是（`schema.ts:118`） | epoch ms |

---

## 3. SourceMessage

| 字段 | 类型 | 必填 | 可空 | 来源 | 真实验证 | 备注 |
|---|---|---|---|---|---|---|
| `message_id` | string | ✓ | ✗ | UPSTREAM | 是（`messages.msg_id`） | 可被重铸（#4390/#4491） |
| `session_id` | string | ✓ | ✗ | UPSTREAM | 是 | |
| `ledger_seq` | number | ✓ | ✗ | UPSTREAM | 是（`messages.seq`） | 会话内单调；复活保原 seq |
| `role` | string | ✓ | ✗ | UPSTREAM | 是（`messages.role`） | 上游值：`user\|assistant\|system\|tool` |
| `content_type` | string | ✓ | ✗ | UPSTREAM | 是（`messages.kind`） | 上游 `MessageContentType`：`text\|tool-call\|tool-result\|reasoning`；**无 `"system"`** |
| `original_text` | string\|null | ✓ | ✓ | DERIVED | 部分（见下） | 从 `content.bytes` JSON.parse 取 `text` 字段；**多模态被拆分，图片在 `message_media`，CMB 未 hydrate** |
| `content_hash` | string | ✓ | ✗ | UPSTREAM | 是（`messages.content_hash`） | **是 blob 字节哈希，不是 `original_text` 哈希**（违反 13 号文档 MUST） |
| `timestamp` | number\|null | ✓ | ✓ | UPSTREAM | 是（`messages.ts`） | **语义二义**：首见戳或批采样时刻，无列可区分 → Evidence 恒置 null |
| `tool_name` | string\|null | ✓ | ✓ | DERIVED | 是 | 从 blob JSON 取 `toolName` |
| `tool_call_id` | string\|null | ✓ | ✓ | DERIVED | 是 | 从 blob JSON 取 `toolCallId` |
| `speaker_id` | string\|null | ✓ | ✓ | HOST | **否** | `bc-source.ts:211` 恒 null；上游 `messages` 表**无 speaker 列** |
| `source_locator` | string | ✓ | ✗ | DERIVED | 是 | `${session_id}:${seq}` —— **session_id 含冒号时解析错误** |
| `visibility` | Visibility\|null | ✓ | ✓ | HOST | **否** | Mock 恒 null，由 HostIdentityProvider 决定 |
| `excluded_reason` | string\|null | ✓ | ✓ | CMB | 是 | `reasoning` 时设 `"REASONING_EXCLUDED"` |

### 3.1 `original_text` 恢复链路（关键）

```
content.bytes (BLOB)
  ↓ JSON.parse
CoreMessage 投影（已 splitMediaContent 拆分）
  ↓ 取 .text
original_text
```

**限制**：
1. 图片消息的 `text` 可能只剩 `{"$media":"..."}` 占位符
2. 退役（tombstone）消息的媒体 ref 被立即释放（`store.ts:951-955`）→ 图片永久不可恢复
3. blob 缺失时 `original_text = null`

---

## 4. CompressionBlock

| 字段 | 类型 | 必填 | 可空 | 来源 | 真实验证 | 备注 |
|---|---|---|---|---|---|---|
| `block_id` | string | ✓ | ✗ | UPSTREAM | 是（`blocks.block_id`） | refold 换新 id；编号可复用 |
| `session_id` | string | ✓ | ✗ | UPSTREAM | 是 | |
| `covered_hash` | string\|null | ✓ | ✓ | UPSTREAM | 是（`blocks.covered_hash`） | = `sha256(JSON.stringify({effectiveMessageIds, directMessageIds}))`；**全仓无人校验** |
| `start_seq` | number\|null | ✓ | ✓ | UPSTREAM | 是（`blocks.start_seq`） | `-1` 哨兵 → 适配器转 null；**由 store 层从 live messages 重算，非写入时快照** |
| `end_seq` | number\|null | ✓ | ✓ | UPSTREAM | 是 | 同上 |
| `block_status` | `"active"\|"inactive"` | ✓ | ✗ | DERIVED | 是（`blocks.active`） | 1→active, 0→inactive |
| `direct_message_ids` | string[]\|null | ✓ | ✓ | UPSTREAM | 是（`payload_json.directMessageIds`） | **仅经 payload_json，无独立列** |
| `effective_message_ids` | string[]\|null | ✓ | ✓ | UPSTREAM | 是（`payload_json.effectiveMessageIds`） | 同上 |
| `supersedes_block_id` | string\|null | ✓ | ✓ | UPSTREAM | 是（`blocks.supersedes_block_id`） | **恒 NULL 死列**（`ingest.ts:425` 硬编码） |
| `summary` | string\|null | ✓ | ✓ | UPSTREAM | 部分 | `bc-source.ts:166` 恒 null（summary 在 CAS `summary_hash`，未读取） |
| `created_at` | number\|null | ✓ | ✓ | UPSTREAM | 是（`blocks.created_at`） | |
| `block_version` | string | ✓ | ✗ | DERIVED | 是 | `sha256(payload_json)` —— **与 bridge_key 使用的 covered_hash 不同源** |

---

## 5. EvidenceDTO

| 字段 | 类型 | 必填 | 可空 | 来源 | 对照 Memory Core schema | 备注 |
|---|---|---|---|---|---|---|
| `evidence_id` | string | ✓ | ✗ | DERIVED | ✗ 格式冲突（要求 uuid） | `${session_id}:${ledger_seq}` → **会撞键** |
| `source_type` | `"chat"` | ✓ | ✗ | CMB 常量 | ✗ 枚举冲突（schema 12 值，DOC 5 值） | |
| `source_id` | string | ✓ | ✗ | DERIVED | ✓（schema 允许 null） | = `message_id` |
| `source_locator` | string | ✓ | ✗ | DERIVED | ✓ | |
| `original_text` | string | ✓ | ✗ | DERIVED | **✗ schema 无此字段** | 核心内容，却被 schema 拒绝 |
| `content_hash` | string | ✗ | ✗ | UPSTREAM | ✓ 但语义错 | **应为 sha256(original_text)** |
| `speaker_id` | string\|null | ✓ | ✓ | HOST | **✗ schema 无此字段** | 真实源恒 null |
| `timestamp` | number\|null | ✓ | ✓ | CMB 常量 | ✗ 名称不同（schema 用 observed_at/recorded_at） | **恒为 null** |
| `tenant_id` | string | ✓ | ✗ | HOST | ✓ | 部署常量 |
| `bot_id` | string | ✓ | ✗ | HOST | **✗ schema 无** | |
| `group_id` | string\|null | ✓ | ✓ | HOST | **✗ schema 无** | |
| `visibility` | Visibility | ✓ | ✗ | HOST | ✗（schema 用 visibility_scope 自由字符串） | |
| `revoked` | boolean | ✓ | ✗ | CMB | ✓ | |
| `produced_by_run` | string\|null | ✓ | ✓ | CMB | **✗ schema 无** | 审计用 |
| `produced_by_blocks` | string[] | ✓ | ✗ | CMB | **✗ schema 无** | 审计用 |
| — | — | — | — | — | **✗ schema 要求 `scope`，CMB 无** | required |
| — | — | — | — | — | **✗ schema 要求 `recorded_at`，CMB 无** | required |

---

## 6. Bridge Ledger 三表

### 6.1 block_ledger

| 字段 | 类型 | 来源 | 备注 |
|---|---|---|---|
| `bridge_key` | TEXT PK | DERIVED | `sha256(session_id\|block_id\|covered_hash)` |
| `session_id` | TEXT | UPSTREAM | |
| `block_id` | TEXT | UPSTREAM | |
| `covered_hash` | TEXT | UPSTREAM | |
| `block_version` | TEXT | DERIVED | `sha256(payload_json)` |
| `block_status` | TEXT | UPSTREAM | active/inactive |
| `first_seen_at` | INTEGER | CMB | epoch ms |
| `last_checked_at` | INTEGER | CMB | |
| `bridged_at` | INTEGER\|NULL | CMB | COALESCE 保留首次 |
| `evidence_count` | INTEGER | CMB | |
| `summary_hash` | TEXT\|NULL | — | **恒 null**（引擎不存摘要哈希） |
| `status` | TEXT | CMB | 12 种 BridgeStatus |
| `failure_reason` | TEXT\|NULL | CMB | |

### 6.2 message_ledger

| 字段 | 类型 | 来源 | 备注 |
|---|---|---|---|
| `source_message_key` | TEXT PK | DERIVED | `${session_id}\|${message_id}` |
| `session_id` | TEXT | **可能被 locator 解析污染** | |
| `message_id` | TEXT | UPSTREAM | |
| `ledger_seq` | INTEGER | **可能被 locator 解析污染** | |
| `source_locator` | TEXT | DERIVED | |
| `content_hash` | TEXT | UPSTREAM | |
| `evidence_id` | TEXT\|NULL | DERIVED | COALESCE 保留 |
| `processing_status` | TEXT | CMB | |
| `revoked_status` | INTEGER | CMB | 0/1 |

### 6.3 run_ledger

| 字段 | 类型 | 备注 |
|---|---|---|
| `run_id` | TEXT PK | uuid |
| `start_time` | INTEGER | |
| `end_time` | INTEGER\|NULL | **崩溃时永久 NULL** |
| `scan_count` / `success_count` / `skipped_count` / `failed_count` / `pending_count` / `revoked_count` | INTEGER | |
| `duration_ms` | INTEGER\|NULL | |
| `error_summary` | TEXT\|NULL | failure_reasons 的 JSON |

---

## 7. 关键派生字段的算法

| 字段 | 算法 | 位置 |
|---|---|---|
| `source_locator` | `` `${session_id}:${seq}` `` | `bc-source.ts:187` |
| `bridge_key` | `sha256(session_id + "\|" + block_id + "\|" + (covered_hash ?? ""))` | `bridge-ledger.ts:273` |
| `block_version` | `sha256(payload_json)` | `bc-source.ts:168` |
| `evidence_id` | `` `${session_id}:${ledger_seq}` `` | `evidence-adapter.ts:53` |
| `evidenceIdentity` | `` `${content_hash}\|${source_locator}` `` | `evidence-adapter.ts:75` |
| `messageKey` | `` `${session_id}\|${message_id}` `` | `bridge-ledger.ts:281` |
| `content_hash`（上游） | `sha256(splitMediaContent(JSON.stringify(CoreMessage−ts)).content)` | `ingest.ts:194-195` |
| `identity`（上游） | `sha256(JSON minus id/wireIndex/ledgerSeq)` | `identity.ts:86-98` |

---

## 8. 字段可信度总结

| 类别 | 字段数 | 说明 |
|---|---|---|
| 上游真实可读 | 12 | session_id, parent_session_id, fork_point_seq, created_at, message_id, ledger_seq, role, content_type, content_hash, timestamp, block_id, covered_hash, start_seq, end_seq, active, created_at, supersedes_block_id |
| 派生可信 | 6 | source_locator(有缺陷), block_version, bridge_key, evidence_id(有缺陷), messageKey, excluded_reason |
| 宿主注入（未验证） | 5 | tenant_id, bot_id, group_id, speaker_id, visibility |
| 恒空/常量 | 3 | timestamp(Evidence), summary, summary_hash |
| schema 冲突 | 6 | original_text, speaker_id, bot_id, group_id, produced_by_run, produced_by_blocks |
