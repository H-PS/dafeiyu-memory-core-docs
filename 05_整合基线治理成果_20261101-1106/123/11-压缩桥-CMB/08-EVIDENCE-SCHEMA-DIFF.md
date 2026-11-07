# 08 — Evidence Schema 三方差分（CMB DTO ↔ Memory Core Schema ↔ 权威文档）

> 配套：`06-MEMORY-CORE-CONTRACT-AUDIT.md`（审查全文）、`schema-diff.json`（机器可读）
> 审查性质：**只读**。未修改 Memory Core 任何文件、未修改任何 Schema。
> 生成时间：2026-11-06 08:15（Asia/Shanghai）｜ 证据等级：除特别标注外均为 `VERIFIED_STATIC`

---

## 1. 方法与基线

### 1.1 三方源

| 方 | 精确定位 | 大小 | SHA-256 前16位 | 说明 |
|---|---|---|---|---|
| **CMB DTO** | `E:\[PROJECT]\CMB-v0.1-prototype\src\core\domain.ts:116-134`（`interface EvidenceDTO`） | 15 字段 | `DACD357ED9C0C2AA`（整文件） | 内部 DTO；`domain.ts:110-115` 自述"不宣称通过 evidence.schema.json" |
| **Schema** | `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\contracts\schemas\evidence.schema.json` | 36 行 / 2,943 B | `F4D087C3AADAC774`（16 位） | Draft-07；`additionalProperties:false`；required 7 |
| **权威文档** | `DD-AUTHORITY-V2.md` §3.2（`L208-223`）＋ `13-Evidence与EmotionContext.md` §7.2（`L232-248`） | 13 字段 | `A83916181B8CF75F` / `8657E5204E9D3B6E` | `13 §8.1 L734`：冲突时以 V2 §3.2 为准 |

### 1.2 路径差异（必须记录）

| 项 | 值 |
|---|---|
| 任务书指定 | `C:\Users\[USER]\Desktop\work\R4.1-memory-bank-input` → **不存在** |
| 实际使用 | `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design` |
| 备份（仅比对用） | `Memory-Core-R4.1-Design.backup-20261104-192558` |

**备份比对**：两处 `evidence.schema.json` **逐字节一致**（SHA-256 `F4D087C3AADAC774EC2C7B770D159442660D55EF6213330259C7474E08CA9832`）。
→ **2026-11-04 V2 升级未改动该 schema**，这是"文档已升级、契约未跟进"的直接物证。

### 1.3 字段集差异的结构性事实

| 来源 | 字段数 | 独有字段 |
|---|---|---|
| `evidence.schema.json`（＝`01 §7.8` v1 基线） | 21 | `source_version`、`excerpt`、`observed_at`、`recorded_at`、`actor_id`、`provenance_id`、`source_session_id`、`source_message_id`、`source_subject_id`、`extraction_method`、`model_version`、`visibility_scope`、`tenant_id` |
| `V2 §3.2` / `13 §7.2`（权威） | 13 | `speaker_id`、`timestamp`、`original_text`、`context_before`、`context_after`、`visibility` |
| `CMB EvidenceDTO` | 15 | `bot_id`、`group_id`、`produced_by_run`、`produced_by_blocks`、`visibility`(5值)、`original_text`、`speaker_id` |

**交集仅 8 个**：`evidence_id`、`source_type`、`source_id`、`source_locator`、`content_hash`、`scope`、`retention_policy`、`revoked`。
→ 这**不是"schema 少几个字段"**，而是**两个不同版本的对象模型**。

---

## 2. 差异类型定义（本报告口径）

| 类型 | 含义 |
|---|---|
| `NO_DIFF` | 三方结构兼容，CMB 值可直接通过 schema 校验 |
| `MISSING_IN_SCHEMA` | CMB 或权威文档有该字段，**schema 无**（因 `additionalProperties:false`，携带即被拒） |
| `MISSING_IN_DTO` | 仅 schema 有（多为可选），CMB 不提供 |
| `TYPE_MISMATCH` | 字段名相同但类型/格式不兼容 |
| `REQUIRED_MISMATCH` | 必填性不一致，导致 schema 必填项无法满足 |
| `ENUM_MISMATCH` | 枚举值域不同且无完整映射 |
| `NAMING_MISMATCH` | 同义字段用了不同名字（本报告 0 例；`actor_id`↔`speaker_id` 因语义不完全等价，归 `MISSING_IN_*`） |

---

## 3. 逐字段差分总表（32 行）

> 28 个请求字段 + 4 个附录字段（`produced_by_run`、`produced_by_blocks` ＝ CMB 独有；`context_before`、`context_after` ＝ 权威独有）。
> 本表由 `schema-diff.json` **程序生成**，与 JSON 逐项一致（生成脚本读取 JSON 后输出，非手工誊写）。

| # | 字段 | CMB DTO | evidence.schema.json | 权威文档 | 差异类型 | 兼容性 | Blocking | 结论 |
|---|---|---|---|---|---|---|---|---|
| 1 | `evidence_id` | string **必填** | string (uuid) **必填** | string (uuid) **必填** | TYPE_MISMATCH | INCOMPATIBLE | **是** | ADAPTER_WORK_REQUIRED |
| 2 | `source_type` | string literal {chat} **必填** | string {direct_user_statement, assistant_observation, imported_document, tool_result, system_event, approved_inference, explicit_user, conversation, repeated_statement, imported, system_generated, human_reviewed} **必填** | string {chat, file, import, tool, system} **必填** | ENUM_MISMATCH | INCOMPATIBLE | **是** | UPSTREAM_CHANGE_REQUIRED |
| 3 | `source_id` | string **必填** | string\|null **必填** | string **必填** | NO_DIFF | COMPATIBLE | 否 | NO_ACTION |
| 4 | `source_version` | 不存在 | string\|null | 不存在 | MISSING_IN_DTO | COMPATIBLE | 否 | NO_ACTION |
| 5 | `source_locator` | string **必填** | string\|null | string\|null | NO_DIFF | COMPATIBLE | 否 | NO_ACTION |
| 6 | `source_session_id` | 不存在 | string\|null | 不存在 | MISSING_IN_DTO | COMPATIBLE | 否 | NO_ACTION |
| 7 | `source_message_id` | 不存在 | string\|null | 不存在 | MISSING_IN_DTO | COMPATIBLE | 否 | NO_ACTION |
| 8 | `source_subject_id` | 不存在 | string\|null | 不存在 | MISSING_IN_DTO | COMPATIBLE | 否 | NO_ACTION |
| 9 | `excerpt` | 不存在 | string\|null | 不存在 | MISSING_IN_DTO | COMPATIBLE | 否 | NO_ACTION |
| 10 | `original_text` | string **必填** | **不存在** | string **必填** | MISSING_IN_SCHEMA | INCOMPATIBLE | **是** | UPSTREAM_CHANGE_REQUIRED |
| 11 | `content_hash` | string **必填** | string **必填** | sha256 **必填** | NO_DIFF | COMPATIBLE_STRUCTURALLY_ONLY | 否 | USER_DECISION_REQUIRED |
| 12 | `observed_at` | 不存在 | string\|null (date-time) | 不存在 | MISSING_IN_DTO | COMPATIBLE | 否 | NO_ACTION |
| 13 | `recorded_at` | 不存在 | string (date-time) **必填** | 不存在 | REQUIRED_MISMATCH | INCOMPATIBLE | **是** | UPSTREAM_CHANGE_REQUIRED |
| 14 | `timestamp` | number\|null **必填** | **不存在** | datetime\|null | MISSING_IN_SCHEMA | INCOMPATIBLE | **是** | UPSTREAM_CHANGE_REQUIRED |
| 15 | `actor_id` | 不存在 | string\|null | 不存在 | MISSING_IN_DTO | COMPATIBLE | 否 | NO_ACTION |
| 16 | `speaker_id` | string\|null **必填** | **不存在** | string **必填** | MISSING_IN_SCHEMA | INCOMPATIBLE | **是** | UPSTREAM_CHANGE_REQUIRED |
| 17 | `tenant_id` | string **必填** | string **必填** | **不存在** | NO_DIFF | COMPATIBLE | 否 | NO_ACTION |
| 18 | `bot_id` | string **必填** | **不存在** | 不存在 | MISSING_IN_SCHEMA | INCOMPATIBLE | **是** | UPSTREAM_CHANGE_REQUIRED |
| 19 | `group_id` | string\|null **必填** | **不存在** | 不存在 | MISSING_IN_SCHEMA | INCOMPATIBLE | **是** | UPSTREAM_CHANGE_REQUIRED |
| 20 | `scope` | 不存在 | string {owner_private, group_shared, session_only, project_scoped, system_internal} **必填** | enum(v1 五值) **必填** | REQUIRED_MISMATCH | INCOMPATIBLE | **是** | ADAPTER_WORK_REQUIRED |
| 21 | `visibility_scope` | 不存在 | string\|null | 不存在 | MISSING_IN_DTO | COMPATIBLE | 否 | NO_ACTION |
| 22 | `visibility` | enum Visibility {owner_private, group_shared, session_only, project_scoped, system_internal} **必填** | **不存在** | enum §2.10 {owner_private, group_shared, public} **必填** | MISSING_IN_SCHEMA | INCOMPATIBLE | **是** | USER_DECISION_REQUIRED |
| 23 | `provenance_id` | 不存在 | string\|null (uuid) | 不存在 | MISSING_IN_DTO | COMPATIBLE | 否 | NO_ACTION |
| 24 | `extraction_method` | 不存在 | string\|null | 不存在 | MISSING_IN_DTO | COMPATIBLE | 否 | NO_ACTION |
| 25 | `model_version` | 不存在 | string\|null | 不存在 | MISSING_IN_DTO | COMPATIBLE | 否 | NO_ACTION |
| 26 | `retention_policy` | 不存在 | string\|null | string\|null | MISSING_IN_DTO | COMPATIBLE | 否 | INSUFFICIENT_INFO |
| 27 | `revoked` | boolean **必填** | boolean | boolean **必填** | REQUIRED_MISMATCH | COMPATIBLE | 否 | UPSTREAM_CHANGE_REQUIRED |
| 28 | `revoked_at` | 不存在 | 不存在 | 不存在 | MISSING_IN_SCHEMA | COMPATIBLE | 否 | USER_DECISION_REQUIRED |
| 29 | `produced_by_run` | string\|null **必填** | **不存在** | 不存在 | MISSING_IN_SCHEMA | INCOMPATIBLE | **是** | ADAPTER_WORK_REQUIRED |
| 30 | `produced_by_blocks` | string[] **必填** | **不存在** | 不存在 | MISSING_IN_SCHEMA | INCOMPATIBLE | **是** | ADAPTER_WORK_REQUIRED |
| 31 | `context_before` | 不存在 | **不存在** | string\|null | MISSING_IN_SCHEMA | COMPATIBLE | 否 | UPSTREAM_CHANGE_REQUIRED |
| 32 | `context_after` | 不存在 | **不存在** | string\|null | MISSING_IN_SCHEMA | COMPATIBLE | 否 | UPSTREAM_CHANGE_REQUIRED |

### 3.1 汇总

| 指标 | 值 |
|---|---|
| 总字段 | **32** |
| `NO_DIFF` | **4**（`source_id`、`source_locator`、`content_hash`*、`tenant_id`）※`content_hash` 仅结构无差，语义有偏离 |
| **`blocking`** | **12** |
| `MISSING_IN_SCHEMA` | 11 |
| `MISSING_IN_DTO` | 12 |
| `REQUIRED_MISMATCH` | 3 |
| `TYPE_MISMATCH` | 1 |
| `ENUM_MISMATCH` | 1 |
| `NAMING_MISMATCH` | 0 |

**结论分布**：`UPSTREAM_CHANGE_REQUIRED` 10 ／ `NO_ACTION` 14 ／ `ADAPTER_WORK_REQUIRED` 4 ／ `USER_DECISION_REQUIRED` 3 ／ `INSUFFICIENT_INFO` 1 ／ `RUNTIME_EXPERIMENT_REQUIRED` 0。

---

## 4. 12 个 Blocking 字段详解

`blocking` 判据：**该字段导致 CMB `EvidenceDTO` 无法通过 `evidence.schema.json`**（`additionalProperties:false` 或 required 不满足）。`INFERRED`（机制确定，未实跑校验器）

### 4.1 因 `additionalProperties:false` 被拒（CMB 有、schema 无）—— 8 个

| 字段 | CMB 值来源 | schema 状态 | 权威文档是否支持 CMB | 判定 |
|---|---|---|---|---|
| `original_text` | `domain.ts:122`；`evidence-adapter.ts:57` | 无该属性（`:7,9-35`） | ✅ **V2 §3.2 L217 必填** | **权威站在 CMB 一边，schema 错** |
| `timestamp` | `domain.ts:125`；恒 `null`（`adapter.ts:60`） | 无 | ✅ V2 §3.2 L216 可选 | 同上（另注：CMB 恒 null 与权威 "未知为 null" 一致） |
| `speaker_id` | `domain.ts:124` | 无 | ✅ V2 §3.2 L215 **必填** | 同上（但 CMB 可 null，权威要求必填 → 另有冲突） |
| `bot_id` | `domain.ts:127` | 无 | ❌ 权威亦无 | 需上游新增承载位 |
| `group_id` | `domain.ts:128` | 无 | ❌ 权威亦无 | 需上游新增承载位 |
| `visibility` | `domain.ts:129`（5 值） | 无（仅自由串 `visibility_scope`） | ✅ V2 §3.2 L219 **必填**（但 3 值） | 权威站在 CMB 一边；枚举需裁决 |
| `produced_by_run` | `domain.ts:132` | 无 | ❌ 权威亦无 | 适配器剥离/映射 |
| `produced_by_blocks` | `domain.ts:133` | 无 | ❌ 权威亦无 | 适配器剥离/映射 |

### 4.2 因 required 不满足被拒（schema 必填、CMB 无）—— 2 个

| 字段 | schema | CMB | 权威文档 | 判定 |
|---|---|---|---|---|
| `recorded_at` | **必填**（`:8,25`） | 无 | **权威亦无** | **schema 单方面发明必填字段** → 上游必须删必填或补权威定义 |
| `scope` | **必填**（`:8,28`）；enum 5 值 | 无 | ✅ V2 §3.2 L220 **必填** | 权威与 schema 一致；CMB 可投影（值集完全相同，见 4.3） |

### 4.3 因值不合法被拒 —— 2 个

| 字段 | CMB 值 | schema 约束 | 判定 |
|---|---|---|---|
| `evidence_id` | `"<session_id>:<ledger_seq>"`（`adapter.ts:53`） | `format: uuid`（`:10`） | schema＋权威一致要求 uuid → **CMB 改**（可用 UUIDv5 从 `(content_hash, source_locator)` 派生，保留 ADR-003 幂等语义） |
| `source_type` | `"chat"`（`domain.ts:119`） | enum 12 值**不含 `chat`**（`:13`） | CMB 需改为 `"conversation"`（CMB 已在 `adapter.ts:22` 预留该映射），但根因是 schema 枚举错（见 §5） |

### 4.4 `scope` 可无损投影的依据

CMB `Visibility`（`domain.ts:29-34`）与 schema `scope` 枚举（`evidence.schema.json:28`）**逐值相同**：

```
owner_private, group_shared, session_only, project_scoped, system_internal
```

→ 适配器可直接 `evidence.scope = dto.visibility`。**这是 12 个 blocking 里最容易解的一个。**
（注意：投影后 `visibility` 字段仍然缺失 → 见 §4.1，AI-2 缺陷独立存在。）

---

## 5. `source_type` 枚举专项（关键陷阱）

### 5.1 三套枚举并存（文档只说两套）

| # | 枚举 | 值 | 出处 | 语义 |
|---|---|---|---|---|
| A | `MemoryUnit.source_type` | 5：`USER_EXPLICIT`/`USER_INSTRUCTION`/`ASSISTANT_INFERRED`/`EXTERNAL_SOURCE`/`SYSTEM_GENERATED` | `memory-unit.schema.json:116`；权威 `V2 §2.9 L121-128` | 来源**性质** |
| B | `Evidence.source_type`（权威） | 5：`chat`/`file`/`import`/`tool`/`system` | `V2 §3.2 L212`；`13 §7.2 L237`；`13 §7.6.3 L404` | 证据**形态** |
| C | `Evidence.source_type`（**schema 实际**） | **12**：A 的 6 个 v1 名 + 6 个别名 | `evidence.schema.json:13` | ⚠ **B 与 A 语义混装** |
| D | CMB | 1：`chat` | `domain.ts:119` | 形态（对齐 B） |

### 5.2 权威原文（`13 §7.6.3 L399-406`）

> **§7.6.3 辨析：两个 `source_type` 字段（MUST NOT 混淆）**
> | 字段 | 所属对象 | 枚举 | 语义 |
> |---|---|---|---|
> | `MemoryUnit.source_type` / `Evidence`（V2 §2.9 语境）/ `EmotionContext.source_type` | 记忆 / 来源性质 / 情绪 | **5 值**：`USER_EXPLICIT` / `USER_INSTRUCTION` / `ASSISTANT_INFERRED` / `EXTERNAL_SOURCE` / `SYSTEM_GENERATED` | 来源**性质**（谁/怎么产生） |
> | `Evidence.source_type`（V2 §3.2 字段表） | Evidence 自身 | **5 值**：`chat` / `file` / `import` / `tool` / `system` | 证据**形态**（从哪种载体来） |
>
> **MUST**：二者是**不同字段、不同枚举**；文档与契约 **MUST NOT** 混用、互相覆盖或用同一参数 ID；分别登记 15 号参数注册表。

`13 §7.2 L237` 另加注："**注意**：此处是证据形态 5 值，与 V2 §2.9 的 source_type 5 值**是两个不同字段的两个枚举**"。

### 5.3 schema 12 值枚举的真实来源

`01 §7.8` 六规范值（`L430-439`）＋ 6 个登记别名（`L441`）：

| 规范值（01:L434-439） | 别名（01:L441） |
|---|---|
| `direct_user_statement` | `explicit_user`≡ 本值；`conversation`/`repeated_statement`→ 本值 + `extraction_method` 区分 |
| `assistant_observation` | — |
| `imported_document` | `imported`≡ 本值 |
| `tool_result` | — |
| `system_event` | `system_generated`≡ 本值 |
| `approved_inference` | `human_reviewed`→ 保留为**审核标记而非来源类型** |

→ schema 把**规范值 + 别名**全部塞进一个 enum = 12 值。**这是把"迁移映射表"误当成"枚举定义"的典型错误。**

### 5.4 能否建立映射？——不能（完整）

**（a）schema 12 值 → 权威形态 5 值**

| schema 值 | → 形态 | 结果 |
|---|---|---|
| `direct_user_statement` / `explicit_user` / `conversation` / `repeated_statement` | `chat` | ✅ |
| `imported_document` / `imported` | `file` | ✅ |
| `tool_result` | `tool` | ✅ |
| `system_event` / `system_generated` | `system` | ✅ |
| `assistant_observation` | — | ❌ **NO_MAPPING**（"性质"非"形态"） |
| `approved_inference` | — | ❌ **NO_MAPPING**（"性质"非"形态"） |
| `human_reviewed` | — | ❌ **NO_MAPPING**（审核标记，`01:L441` 自己说"非来源类型"） |

**9 可映射 / 3 不可映射**。另：权威枚举内 `file` 与 `import` 本身二义（导入文档算哪个，未定义）。

**（b）schema 12 值 → MemoryUnit 5 值**

`V2 §4.2 L292` 已给官方逐值映射（`assistant_observation`/`approved_inference`→`ASSISTANT_INFERRED`；`imported*`/`tool_result`→`EXTERNAL_SOURCE`；`system_*`→`SYSTEM_GENERATED`；`human_reviewed`→`USER_EXPLICIT`+审计；其余→`USER_EXPLICIT`）。

⚠ **该映射被明令禁止用于 Evidence 字段**（`13 §7.6.3`）。CMB 亦已自我约束：`evidence-adapter.ts:17-18` 声明
`internal_to_memory_unit = "FORBIDDEN_IN_BRIDGE"`，并在 `:17-18` 注明"两套枚举不可互换"。
→ 该映射**只能**用于 MemoryUnit 的 v1→v2 迁移。

**（c）根因：schema 的 12 值枚举语义自相矛盾**

同一 enum 同时承载：
- **载体形态**：`conversation`、`imported_document`、`imported`、`tool_result`、`system_event`
- **来源性质**：`assistant_observation`、`approved_inference`
- **审核状态**：`human_reviewed`

→ `Evidence.source_type` **一个字段无法同时表达"形态"与"性质"**。这正是 `13 §7.6.3` 要防的错误，schema 已经犯下。
→ **不能靠"补映射表"解决**，必须（a）schema 改回权威 5 值形态枚举，或（b）拆成两字段。

### 5.5 CMB 的桥接可行性

| CMB 值 | → 权威形态 | → schema 12 值 | 说明 |
|---|---|---|---|
| `"chat"` | `chat` ✅ | `"conversation"` ✅（需转换） | CMB 已在 `evidence-adapter.ts:20-24` 预留 `internal_evidence_chat_to_v1_schema: "conversation"` |
| `"chat"` 原样投喂 schema | — | ❌ 被 enum 拒（无 `chat` 值） | **当前 `EvidenceDTO` 若直接校验必然失败** |

---

## 6. 两处非字段级结构性缺陷

| ID | 缺陷 | 证据 | 后果 |
|---|---|---|---|
| **AI-1** | `memory-unit.schema.json` 两处要求"原句 MUST 存于 `Evidence.original_text`"，而 `evidence.schema.json` 无该属性且 `additionalProperties:false` | `memory-unit:50`（"原句 MUST 存于 Evidence.original_text，两处禁止互相覆盖"）、`memory-unit:166`、`evidence:7,9-35` | **两个 schema 互斥**：满足 MemoryUnit 约束的 Evidence 无法落盘；反之满足 Evidence schema 的对象无法支撑 MemoryUnit 的原句追溯要求。**不可能同时通过两个 schema** |
| **AI-2** | `evidence.schema.json` 无 `visibility` 枚举字段（仅有自由字符串 `visibility_scope:32`） | `04:263-265`（"派生数据 MUST 继承来源权限，MUST NOT 扩大"、多来源取最严格交集、无法计算→拒绝）、`13 §7.4.4 L322` PERM-1（`visibility(evidence) ≤ visibility(memory)`） | **`04` 号的权限红线在机器契约层不可执行**：无枚举即无法机器比较"更严格/更宽"；`visibility_scope` 是自由串（受控词表"实施期确定"），无法校验 |

---

## 7. `schema-diff.json` 规格与自检

### 7.1 结构

```
{
  "generated_at": "2026-11-06T08:15:45+08:00",
  "generated_by": "...", "audit_id": "B-1",
  "sources": { "cmb_dto": {...}, "schema": {...}, "authority_doc": {...},
               "path_discrepancy": {...} },
  "fields": [ { "name", "cmb_dto":{present,type,required,...ref},
                "schema":{...}, "authority_doc":{...},
                "diff_type", "compat", "blocking", "conclusion", "note" } × 32 ],
  "enum_analysis": { "evidence_source_type_schema": {...},
                     "memory_unit_source_type_schema": {...},
                     "evidence_source_type_authority_doc": {...},
                     "cmb_evidence_source_type": {...},
                     "resolvable": false,
                     "mapping_schema12_to_doc5_form": {...},
                     "mapping_schema12_to_memoryunit5": {...} },
  "summary": { "total":32, "no_diff":4, "blocking":12,
               "by_diff_type":{...}, "by_conclusion":{...},
               "blocking_fields":[...12项...], "blocking_basis":"...",
               "appendix_fields":[...], "note":"..." },
  "b1_definition": "..."
}
```

每个 `field` 均含 `ref` 字段指回 `文件:行号`，可逐项复核。

### 7.2 自检结果（解析器实跑）

| 检查项 | 结果 |
|---|---|
| `JSON.parse` 合法性 | ✅ 通过（Node v24.21.0） |
| `fields` 行数 | 32 ✅ |
| 每行必备键 `name`/`cmb_dto`/`schema`/`authority_doc`/`diff_type`/`compat` | ✅ 全部存在 |
| `summary.blocking` vs 实际 `blocking:true` 计数 | 12 = 12 ✅ |
| `summary.no_diff` vs `diff_type==="NO_DIFF"` 计数 | 4 = 4 ✅ |
| `summary.by_diff_type` 逐类 vs 实际计数 | 7 类全部相等 ✅ |
| `summary.by_conclusion` 逐类 vs 实际计数 | 6 类全部相等 ✅ |
| `summary.blocking_fields` vs 实际 blocking 字段名序列 | 完全一致 ✅ |

> 说明：`by_diff_type` / `by_conclusion` 中保留了计数为 0 的键（`NAMING_MISMATCH:0`、`RUNTIME_EXPERIMENT_REQUIRED:0`），故直接字符串比对会出现"多键"，逐类数值比对全部相等。

### 7.3 附录字段与计数口径

- 请求清单 28 字段：覆盖完整，无一遗漏。
- 附录 4 字段：`produced_by_run`、`produced_by_blocks`（CMB 独有，会触发 `additionalProperties:false`）、`context_before`、`context_after`（权威独有，schema 缺）。
- `summary.total = 32` **含附录**；`appendix_fields` 字段显式标注，便于按 28 口径过滤。

---

## 8. B-1 冲突的确切定义

> **`evidence.schema.json` 所实现的 Evidence 字段集，是 `13 §8.1:734` 已明确降级为"v1 基线"的 `01 §7.8` 版本，而非被声明为权威的 `DD-AUTHORITY-V2 §3.2`；两者字段集不等（schema 缺 `original_text`/`speaker_id`/`timestamp`/`visibility` 等权威必填项，却含 `actor_id`/`observed_at`/`recorded_at`/`tenant_id`/`provenance_id`/`visibility_scope` 等权威未定义项），叠加 `additionalProperties:false`，使任何携带原句的 CMB Evidence 必然被拒；且该冲突未在任何登记表登记、未裁决。**

**三句话拆解**：
1. **写错了版本**：schema 跟随的是被降级的 v1（`01 §7.8`），不是权威 V2（`V2 §3.2`）。
2. **排斥原句**：`additionalProperties:false` + 无 `original_text` ⇒ 带原句的 Evidence 结构性不可写入（CMB 的核心产出正是原句）。
3. **未登记未裁决**：`DOCUMENT-CONFLICT-REGISTER-V2.md`、`OPEN-ISSUES.md` 均无此项；内部一致性审查误判 PASS（只比 v1↔v1）。

---

## 9. 完成状态

| 交付物 | 路径 | 状态 |
|---|---|---|
| 三方差分报告（本文件） | `E:\[PROJECT]\CMB-v0.1-review-pack\08-EVIDENCE-SCHEMA-DIFF.md` | ✅ |
| 机器可读差分 | `E:\[PROJECT]\CMB-v0.1-review-pack\schema-diff.json` | ✅ 合法 JSON，32 字段，blocking = **12** |
| 契约审查全文 | `E:\[PROJECT]\CMB-v0.1-review-pack\06-MEMORY-CORE-CONTRACT-AUDIT.md` | ✅ |

**只读确认**：Memory Core 目录 0 次写入；`contracts/schemas/` 0 次修改；CMB 原型目录 0 次写入。
