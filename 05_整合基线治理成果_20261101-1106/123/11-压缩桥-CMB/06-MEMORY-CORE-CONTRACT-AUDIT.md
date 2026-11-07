# 06 — Memory Core Evidence 契约审查（Agent C）

> 审查对象：CMB v0.1 ↔ Memory Core R4.1-Design 的 **Evidence 契约**
> 审查性质：**只读**。未修改 Memory Core 任何文件、未修改任何 Schema、未修改 CMB 任何文件。
> 配套交付：`08-EVIDENCE-SCHEMA-DIFF.md`（逐字段差分）、`schema-diff.json`（机器可读）
> 审查时间：2026-11-06 08:15（Asia/Shanghai）
> 证据等级标注：`VERIFIED_STATIC`（静态读文件可确认）／`VERIFIED_RUNTIME`（实跑确认）／`INFERRED`（推断，非直读）／`UNVERIFIED`（未能核实）

---

## 0. 路径差异记录（任务书与实际不符，必须登记）

| 项 | 值 | 状态 |
|---|---|---|
| 任务书指定路径 | `C:\Users\[USER]\Desktop\work\R4.1-memory-bank-input` | **不存在**（`Test-Path` = False）`VERIFIED_STATIC` |
| 实际使用路径 | `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design` | 存在 `VERIFIED_STATIC` |
| 备份目录 | `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design.backup-20261104-192558` | 存在；**仅用于版本比对**，未作为审查输入 |
| 仓库类型 | 非 git（无 `.git`） | `VERIFIED_STATIC` |
| 目录规模 | 顶层 5 目录（`contracts`/`diagrams`/`evidence`/`reviews`/`test-specs`）+ 35 个顶层 md/json | `VERIFIED_STATIC` |

**备份比对结果（有价值的发现）**：`contracts/schemas/evidence.schema.json` 在备份目录与现目录**逐字节一致**
（两者 SHA-256 = `F4D087C3AADAC774EC2C7B770D159442660D55EF6213330259C7474E08CA9832`）。
→ 即 **2026-11-04 的 V2 架构升级改写了 13/01/00 等文档，但没有改 `evidence.schema.json` 一个字节**。
这是 B-1 冲突的成因证据（文档已升级到 V2，机器契约仍停在 v1）。

**哈希复核（与 `21-REPOSITORY-BASELINES.md` 对照）**：

| 文件 | 本轮 SHA-256 前16位 | 基线的前16位 | 一致 | 基线行数 | 本轮实测行数 | 结论 |
|---|---|---|---|---|---|---|
| `contracts/schemas/evidence.schema.json` | `F4D087C3AADAC774` | `F4D087C3AADAC774` | ✅ | 36 | 36 | 一致 |
| `contracts/schemas/memory-unit.schema.json` | `A204CE745D31878C` | `A204CE745D31878C` | ✅ | 293 | 293 | 一致 |
| `DD-AUTHORITY-V2.md` | `A83916181B8CF75F` | `A83916181B8CF75F` | ✅ | 400 | **465** | ⚠ 哈希同、**行数基线错** |
| `13-Evidence与EmotionContext.md` | `8657E5204E9D3B6E` | `8657E5204E9D3B6E` | ✅ | 819 | **1034** | ⚠ 哈希同、**行数基线错** |
| `04-权限与治理规范.md` | `FD06CBCD3F1971B8` | `FD06CBCD3F1971B8` | ✅ | 330 | **451** | ⚠ 哈希同、**行数基线错** |

> **证据纪律提示**：`21-REPOSITORY-BASELINES.md` §4 的哈希可信，但**行数列不可信**（系统性偏低，疑为采集工具的行计数方式差异）。
> 本报告所有 `文件:行号` 均以本轮直接读取（`read` 工具 / .NET `ReadAllText` 分割）为准，并用 `Select-String`/`grep` 交叉验证。

---

## 1. 执行摘要

1. **B-1 冲突成立，且比 CMB 侧既有描述更严重。** CMB 侧描述为「schema 缺 `original_text`/`speaker_id`」；本轮核实为**整个字段集是两个不同版本的产物**：`evidence.schema.json` 实现的是 `01 §7.8`（v1 基线，21 字段），而 `13 §8.1` 已明确把 `01 §7.8` **降级**、声明 `V2 §3.2` 为权威。二者是**不同的 21 字段集合**，不是「少几个字段」。
2. **机器契约与权威文档的权威链断裂。** `DD-AUTHORITY-V2.md:5` 自称"唯一枚举/字段/接口定义来源"，`13 §8.1` 也重申 V2 §3.2 优先，但 `evidence.schema.json` 未跟随（备份比对证明未被改动）。这是**权威链断点**，不是字段级笔误。
3. **`MemoryUnit` 与 `Evidence` 两个 schema 自相矛盾。** `memory-unit.schema.json:50,166` 两处要求"原句 MUST 存于 `Evidence.original_text`"，但 `evidence.schema.json` 无该属性且 `additionalProperties:false` → **任何满足 memory-unit 约束的 Evidence 都无法落盘**。
4. **`source_type` 存在三套枚举，且 schema 那套语义自相矛盾。** 权威文档声明只有两套（`13 §7.6.3`），但 schema 里有第三套 12 值枚举，混合了"证据形态"与"来源性质"两类语义。CMB 用的 `"chat"` **在 schema 12 值中不存在**。
5. **M03/M04 均为设计规划，无任何实现代码。** 仓库内不存在 M03/M04 的源码；`api-contracts.md:89,93` 明确标 `[FUTURE — 契约预留]`。"Evidence 如何进入 M03"**没有可执行答案**。
6. **`additionalProperties:false` 使 CMB 侧 12 个字段结构性不可写入**（`blocking = 12`）。
7. **该冲突在 Memory Core 侧从未被登记**（`DOCUMENT-CONFLICT-REGISTER-V2.md`、`OPEN-ISSUES.md` 均无），且内部一致性审查**误判为 PASS**（`reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md:39`——它只比了 `01 §7.8` ↔ schema，没比 V2 §3.2）。

---

## A. Schema Authority Chain

### A-1. `contracts/schemas/evidence.schema.json` 完整内容（逐字段）

文件共 36 行，Draft-07，`additionalProperties: false`（`:7`），`required` 7 项（`:8`）。`VERIFIED_STATIC`

**必填集（`:8`）**：`evidence_id`, `source_type`, `source_id`, `tenant_id`, `scope`, `content_hash`, `recorded_at`

| # | 字段 | 类型 | 必填 | 约束 / 枚举 | 行号 |
|---|---|---|---|---|---|
| 1 | `evidence_id` | string | ✅ | `format: uuid` | `:10` |
| 2 | `source_type` | string | ✅ | **enum 12 值**（见下） | `:11-15` |
| 3 | `source_id` | string \| null | ✅ | `maxLength 256` | `:16` |
| 4 | `source_version` | string \| null | — | `maxLength 64` | `:17` |
| 5 | `source_locator` | string \| null | — | `maxLength 512`（`session:msg:seq`） | `:18` |
| 6 | `source_session_id` | string \| null | — | `maxLength 128` | `:19` |
| 7 | `source_message_id` | string \| null | — | `maxLength 128` | `:20` |
| 8 | `source_subject_id` | string \| null | — | `maxLength 128` | `:21` |
| 9 | `excerpt` | string \| null | — | `maxLength 2000` | `:22` |
| 10 | `content_hash` | string | ✅ | `pattern ^[0-9a-fA-F]{64}$` | `:23` |
| 11 | `observed_at` | string \| null | — | `format: date-time` | `:24` |
| 12 | `recorded_at` | string | ✅ | `format: date-time` | `:25` |
| 13 | `actor_id` | string \| null | — | `maxLength 128` | `:26` |
| 14 | `tenant_id` | string | ✅ | `minLength 1`, `maxLength 128` | `:27` |
| 15 | `scope` | string | ✅ | **enum 5 值**：`owner_private`/`group_shared`/`session_only`/`project_scoped`/`system_internal` | `:28` |
| 16 | `provenance_id` | string \| null | — | `format: uuid` | `:29` |
| 17 | `extraction_method` | string \| null | — | `maxLength 128` | `:30` |
| 18 | `model_version` | string \| null | — | `maxLength 128` | `:31` |
| 19 | `visibility_scope` | string \| null | — | `maxLength 128`（**自由字符串，非枚举**） | `:32` |
| 20 | `retention_policy` | string \| null | — | `maxLength 128` | `:33` |
| 21 | `revoked` | boolean | — | `default: false`（**未进 required**） | `:34` |

**`source_type` 枚举 12 值（`:13`）**：
`direct_user_statement`, `assistant_observation`, `imported_document`, `tool_result`, `system_event`, `approved_inference`, `explicit_user`, `conversation`, `repeated_statement`, `imported`, `system_generated`, `human_reviewed`

**该 schema 内两处缺陷**：
- `:14` 描述称"`direct_user_statement`/`explicit_user` 为同义主用名（见 **error-codes.md 术语映射**）"——但 `contracts/error-codes.md`（58 行，全文已读）**不含任何 source_type 术语映射**，该引用**悬空**。真实的别名登记在 `01 §7.8:441`。`VERIFIED_STATIC`
- `:32` `visibility_scope` 是自由字符串（受控词表"实施期确定"），**不可机器校验**；且全 schema **没有 `visibility` 枚举字段**。

### A-2. `contracts/schemas/memory-unit.schema.json` 的 `source_type` 与 `evidence_refs`

| 项 | 内容 | 行号 |
|---|---|---|
| `source_type` 枚举 | **5 值**：`USER_EXPLICIT`, `USER_INSTRUCTION`, `ASSISTANT_INFERRED`, `EXTERNAL_SOURCE`, `SYSTEM_GENERATED` | `:116` |
| 权威依据 | `DD-AUTHORITY-V2.md` §2.9（`L121-128`）逐字 | `:117` 描述自述 |
| `evidence_refs` 类型 | `array<string format:uuid>`，引用 `Evidence.evidence_id` | `:163-166` |
| `evidence_refs` 强制 | 正式记忆 MUST 非空；唯一例外 `source_type=SYSTEM_GENERATED` | `:166`, `allOf:219-229` |
| 交叉矛盾 | `:50` 与 `:166` 均要求"原句 MUST 存于 **`Evidence.original_text`**"，但 `evidence.schema.json` 无该属性 | `:50`, `:166` |

> **A-2 关键结论**：`memory-unit.schema.json` 期望 `Evidence.evidence_id` 是 **uuid**（`:160-161,165`），与 `evidence.schema.json:10` 的 `format:uuid` 一致；但 CMB 的 `evidence_id` 是 `session:seq` 字符串（见 B）。同时 `memory-unit` 要求存在 `Evidence.original_text`，与 `evidence.schema.json` **直接互斥**。

### A-3. `DD-AUTHORITY-V2.md` 与 `DD-AUTHORITY.md` 的关系

| 项 | `DD-AUTHORITY.md` (v1) | `DD-AUTHORITY-V2.md` (v2.0) |
|---|---|---|
| 版本/状态 | 1.0｜**APPROVED**（`DD-AUTHORITY.md:3`） | 2.0｜2026-11-04｜**APPROVED**（`V2:3`） |
| 自我定位 | "本次任务的**单一事实来源**"；其他 Agent 不得修改（`:4`） | "**唯一**枚举/字段/接口定义来源"（`:5`） |
| 对彼此效力 | 未提及 V2 | `:4`：**"与 DD-AUTHORITY v1 冲突时以本文为准"**；"**本文不推翻正式裁决台账**" |
| 是否作废 | 无废止声明 | 无"V1 作废"表述；且 `§4`（`L275-295`）明确**保留 v1 语义映射** |

**判定**：
1. **V2 是权威，但不是"V1 作废"关系。** V2 只声明"冲突时以本文为准"，属**优先适用（precedence）**，非**废止（repeal）**。V1 仍有效（其 v1 语义经 V2 §4 映射保留）。
2. **这是双权威并存结构**，冲突消解完全依赖"V2 优先"这条解释规则，**没有任何登记表把"v1 哪些条款被 V2 取代"逐条落账**。`DOCUMENT-CONFLICT-REGISTER-V2.md` 的 `SC-01…SC-12` 只登记"架构级结构性冲突"（`TABLE:13-24`），`CR-01…CR-07` 只登记"台账级冲突"（`:30-38`），`DC-01…DC-10` 只登记"文档级待统一项"（`:44-55`）——**没有一条覆盖 "DD-AUTHORITY v1 中 Evidence 定义是否被 V2 取代"**。
3. 实务后果：`01 §7.8`（v1 Evidence 21 字段）与 `13 §7.2`（V2 Evidence 13 字段）**同时存在于同一交付包**，且 `13 §8.1:734` 只说"冲突时以 V2 §3.2 为准"，没说 `01 §7.8` 表要不要删/改。**schema 选择了 v1 那一侧**，于是权威链在此断裂。

### A-4. `DOCUMENT-CONFLICT-REGISTER-V2.md` 是否已登记 Evidence schema 冲突

**未登记。** `VERIFIED_STATIC`（全文 77 行已读）

| 可能相关的条目 | 实际内容 | 是否覆盖本冲突 |
|---|---|---|
| `SC-07`（`:19`） | "证据/原句追溯：Evidence 存在但未定义原句反向追溯 → 采用 V2，13 号承载" | ❌ 只声明"采用 V2"，**未登记 schema 未跟随** |
| `CR-06`（`:37`） | 原句长期保存 vs 数据最小化/导出边界（台账 K-2、D-AP-09） | ❌ 只涉及保留期与导出 |
| `DC-02`（`:47`） | v1 `source_type` 六值 vs V2 五值 → 逐条映射 | ❌ 只处理 **MemoryUnit** 的 source_type，未涉及 Evidence 的 12 值枚举 |
| `DC-01`（`:46`） | scope 五值 vs visibility 三值 | ❌ 只处理 MemoryUnit 侧 |

**`OPEN-ISSUES.md` 亦无**（全文检索 `Evidence` 仅命中 2 条：`OI-015` A 层迁移、`OI-030` 原句保留/导出边界）。
**`PENDING-USER-DECISIONS.md` 无对应裁决项。**
→ **B-1 是一个"存在于交付物中、但不在任何登记表里"的冲突。**

**为什么会漏（根因证据）**：Memory Core 自己的跨文档一致性审查把该项判为 **PASS**：

> `reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md:39`
> `| Evidence 字段与来源类型 | 01 §7.8；evidence.schema.json | PASS | 21 字段一致；六规范来源类型 + 6 别名映射登记一致 |`

该检查项的**受检双方是 `01 §7.8` 与 schema**——恰好是被 `13 §8.1` 降级的那一侧。**V2 §3.2 从未进入这次比对**，因此 PASS 是"v1 与 v1 自洽"的 PASS，属**假绿（false green）**。这解释了冲突为何能穿过审查。

### A-5. 其他规范文档里定义的 Evidence 字段

**（a）`13-Evidence与EmotionContext.md` §7.2（`L232-248`）——"V2 §3.2 逐字段，逐字" 13 字段**

| 字段 | 类型 | 必填 | 行号 |
|---|---|---|---|
| `evidence_id` | string（uuid） | ✅ | `L236` |
| `source_type` | enum(chat/file/import/tool/system) | ✅ | `L237` |
| `source_id` | string | ✅ | `L238` |
| `source_locator` | string\|null | — | `L239` |
| `speaker_id` | string | ✅ | `L240` |
| `timestamp` | datetime\|null | — | `L241` |
| `original_text` | string | ✅ | `L242` |
| `context_before` / `context_after` | string\|null | — | `L243` |
| `visibility` | enum §2.10（3 值） | ✅ | `L244` |
| `scope` | enum(v1 五值) | ✅ | `L245` |
| `content_hash` | sha256 | ✅ | `L246` |
| `retention_policy` | string\|null | — | `L247` |
| `revoked` | boolean | ✅ | `L248` |

字段纪律 7 条（`L250-258`），其中关键：`original_text` MUST 原样保存（`L252`）、`content_hash` MUST 为 `original_text` 的 sha256（`L255`）、`speaker_id` MUST 是可信身份映射（`L254`）、`visibility`/`scope` MUST 一致或更严格（`L256`）。

**（b）`04-权限与治理规范.md`——不定义 Evidence 字段，只定义其权限语义与治理约束** `VERIFIED_STATIC`

| 内容 | 行号 |
|---|---|
| V2 升级头注：可见性对外 3 值、`scope` 五值为内部细分；**证据权限继承**：记忆与证据一致或更严格；`related_evidence` 不得绕过 | `:3` |
| 资源类型 `evidence` 是六资源之一 | `:123` |
| 权限矩阵：`evidence` 行——读"同 memory 继承"、写"来源主体" | `:134` |
| Scope 五类语义与隔离禁令（不作为记忆类型） | `:152-163` |
| 派生对象 11 类含 **Evidence** | `:259` |
| 继承规则：派生数据 MUST 继承来源权限、MUST NOT 扩大、多来源取**最严格交集**、无法计算交集→**默认拒绝** | `:263-265` |
| 权限变更传播七步（含派生对象按交集重算） | `:270-278` |

> **A-5 关键结论**：`04` 号把 Evidence 当作"派生对象"要求 `visibility(evidence) ≤ visibility(memory)` 且 `scope` 取最严格交集（`L263-265`、`13 §7.4.4 PERM-1`），**但 `evidence.schema.json` 里根本没有 `visibility` 枚举字段**（只有自由字符串 `visibility_scope`）。→ **`04` 号的红线在机器契约层不可执行**。这是 A-5 最重要的发现。

**（c）`01-记忆本体与数据模型.md` §7.8（`L402-428`）——21 字段 v1 基线**（被 `13 §8.1` 降级）

字段：`evidence_id`, `source_type`(六规范值 `L430-439`), `source_id`, `source_version`, `source_locator`, `excerpt`, `content_hash`, `observed_at`, `recorded_at`, `actor_id`, `tenant_id`, `scope`, `provenance_id`, `source_session_id`, `source_message_id`, `source_subject_id`, `extraction_method`, `model_version`, `visibility_scope`, `retention_policy`, `revoked`。
别名登记 6 个在 `L441`（**schema 12 值枚举的真实来源**）。SQLite `evidence` 表列定义在 `L668`（与 schema 一致的 v1 列集）。

**（d）四处定义的差异一览**

| 来源 | 字段数 | 代表字段 | 行号 |
|---|---|---|---|
| `evidence.schema.json` | 21 | `actor_id`/`observed_at`/`recorded_at`/`tenant_id`/`provenance_id`/`visibility_scope` | `:9-35` |
| `01 §7.8` | 21 | 同上（schema 与之一致） | `L406-428` |
| `V2 §3.2` / `13 §7.2` | 13 | `speaker_id`/`timestamp`/`original_text`/`context_before`/`context_after`/`visibility` | `V2:L209-223` / `13:L234-248` |
| `CMB EvidenceDTO` | 15 | `source_type:"chat"`/`original_text`/`speaker_id`/`bot_id`/`group_id`/`visibility`/`produced_by_*` | `domain.ts:116-134` |

### A-6. 权威关系表

| 来源 | 版本 | 定义内容 | 被谁使用 | 权威证据 | 当前状态 |
|---|---|---|---|---|---|
| `DD-AUTHORITY-V2.md` §3.2 | 2.0（2026-11-04, APPROVED） | Evidence **13 字段**权威表 | `13 §7.2`/`§8.1` 逐字复制；`01` 头注称"以 V2 为准"；`memory-unit.schema.json:5` 引 V2 为字段来源 | `V2:4` 冲突时以本文为准；`13:734` 冲突时以 V2 §3.2 为准 | **声明权威 ✅ / 被 schema 实现 ❌** |
| `DD-AUTHORITY.md`（v1） | 1.0（APPROVED） | v1 单一事实来源（含 v1 Evidence 基线） | `01 §7.8` 的字段集 | `DD-AUTHORITY.md:3-4` | **未作废，但被 V2 优先** |
| `13-Evidence与EmotionContext.md` §7.2/§8.1 | 随 V2 | 13 字段逐字 + 7 条字段纪律 | 设计/测试规格（`§12` T-13-15/16） | `:734` 自述以 V2 §3.2 为准 | **有效（权威转述）** |
| `01-记忆本体与数据模型.md` §7.8 | v1 基线 | 21 字段 + 六规范值 + 6 别名 + SQLite 列 | **`evidence.schema.json` 实际实现的就是这一套** | `13:734` **明确降级**："01 §7.8 为 v1 基线…冲突时以 V2 §3.2 为准" | **⚠ 已被降级但仍被机器契约采用** |
| `contracts/schemas/evidence.schema.json` | Draft-07（无版本号） | 21 字段 + 12 值枚举 | CMB 适配器目标；`01 §8.1:765` 登记其 required | `memory-unit.schema.json:5` 指向 V2，但本文件未跟随 | **❌ 实现的是被降级版本（B-1）** |
| `04-权限与治理规范.md` | 随 V2 | 不定义字段；定义 `visibility(evidence) ≤ visibility(memory)`、`scope` 最严格交集 | M05 权限门禁设计 | `:3`, `:263-265` | **有效 / 但机器契约无承载字段 ❌** |
| `DOCUMENT-CONFLICT-REGISTER-V2.md` | V2 | 冲突登记 | 全交付包 | `:13-24`, `:30-38`, `:44-55` | **未登记 B-1 ❌** |
| `reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md` | 第2轮 | 跨文档一致性审查 | 交付质量门 | `:39` | **误判 PASS（只比 v1↔v1）❌** |

**权威链结论（一句话）**：权威声明（V2 §3.2）→ 转述文档（13 §7.2）链条完整且一致，但**最后一步"权威 → 机器契约"断裂**：`evidence.schema.json` 实现的是被 `13 §8.1` 降级的 `01 §7.8`，且由于备份比对显示该 schema 在 V2 升级中**零改动**，可确认这是"升级遗漏"而非"有意选择"。

---

## B. 三方差分（核心交付）

完整 32 行逐字段表见 **`08-EVIDENCE-SCHEMA-DIFF.md` §3**，机器可读版见 **`schema-diff.json`**。此处给汇总。

**三方**：
- CMB：`E:\[PROJECT]\CMB-v0.1-prototype\src\core\domain.ts:116-134`（`EvidenceDTO`，15 字段）
- Schema：`contracts/schemas/evidence.schema.json:9-35`（21 字段）
- 权威文档：`DD-AUTHORITY-V2.md` §3.2 `L208-223` + `13 §7.2 L234-248`（13 字段）

**汇总统计**（`schema-diff.json` 已用解析器校验，键值自洽）：

| 指标 | 值 |
|---|---|
| 字段总数（28 请求字段 + 4 附录字段） | **32** |
| `NO_DIFF` | **4** |
| **`blocking`（CMB DTO 无法通过 schema）** | **12** |
| `MISSING_IN_SCHEMA` | 11 |
| `MISSING_IN_DTO` | 12 |
| `TYPE_MISMATCH` | 1 |
| `REQUIRED_MISMATCH` | 3 |
| `ENUM_MISMATCH` | 1 |
| `NAMING_MISMATCH` | 0 |

**12 个 blocking 字段**：`evidence_id`、`source_type`、`original_text`、`recorded_at`、`timestamp`、`speaker_id`、`bot_id`、`group_id`、`scope`、`visibility`、`produced_by_run`、`produced_by_blocks`。

**blocking 成因分三类**：
1. **CMB 有、schema 无**（`additionalProperties:false` 直接拒）：`original_text`、`timestamp`、`speaker_id`、`bot_id`、`group_id`、`visibility`、`produced_by_run`、`produced_by_blocks` —— 8 个。
2. **schema 必填、CMB 无**：`recorded_at`、`scope` —— 2 个。
3. **值不合法**：`evidence_id`（非 uuid）、`source_type`（`"chat"` 不在 12 值内）—— 2 个。

**两个必须单独点出的"非字段级"缺陷**：

| ID | 缺陷 | 证据 |
|---|---|---|
| **AI-1** | `memory-unit.schema.json` 要求原句存于 `Evidence.original_text`，但 `evidence.schema.json` 无该属性 + `additionalProperties:false` ⇒ **两个 schema 互斥，不可同时满足** | `memory-unit:50,166` vs `evidence:7,9-35` |
| **AI-2** | schema 无 `visibility` 枚举字段 ⇒ `04` 号"证据权限 MUST 不宽于记忆"红线**不可机器校验** | `evidence:32`（仅 `visibility_scope` 自由串）vs `04:263-265`、`13:322` |

---

## C. `source_type` 枚举冲突（关键陷阱）

### C-1. `Evidence.source_type` 在 schema 里的枚举全集（12 值）

`contracts/schemas/evidence.schema.json:13`

```
direct_user_statement, assistant_observation, imported_document, tool_result,
system_event, approved_inference, explicit_user, conversation,
repeated_statement, imported, system_generated, human_reviewed
```

**来源可追溯**：`01 §7.8` 六规范值（`L430-439`：前 6 个中的 `direct_user_statement`/`assistant_observation`/`imported_document`/`tool_result`/`system_event`/`approved_inference`）+ `01 §7.8:441` 登记的 6 个别名（`explicit_user`/`conversation`/`repeated_statement`/`imported`/`system_generated`/`human_reviewed`）= **12**。即 schema 把"规范值 + 别名"全塞进同一个 enum。`VERIFIED_STATIC`

### C-2. `MemoryUnit.source_type` 在 schema 里的枚举全集（5 值）

`contracts/schemas/memory-unit.schema.json:116`

```
USER_EXPLICIT, USER_INSTRUCTION, ASSISTANT_INFERRED, EXTERNAL_SOURCE, SYSTEM_GENERATED
```

与 `DD-AUTHORITY-V2.md` §2.9（`L121-128`）逐字一致。`VERIFIED_STATIC`

### C-3. 权威文档 13 号 §7.6.3 关于"两个 source_type 不可混淆"的原文

> `13-Evidence与EmotionContext.md:399-406`
> **§7.6.3 辨析：两个 `source_type` 字段（MUST NOT 混淆）**
>
> | 字段 | 所属对象 | 枚举 | 语义 |
> |---|---|---|---|
> | `MemoryUnit.source_type` / `Evidence`（V2 §2.9 语境）/ `EmotionContext.source_type` | 记忆 / 来源性质 / 情绪 | **5 值**：`USER_EXPLICIT` / `USER_INSTRUCTION` / `ASSISTANT_INFERRED` / `EXTERNAL_SOURCE` / `SYSTEM_GENERATED` | 来源**性质**（谁/怎么产生） |
> | `Evidence.source_type`（V2 §3.2 字段表） | Evidence 自身 | **5 值**：`chat` / `file` / `import` / `tool` / `system` | 证据**形态**（从哪种载体来） |
>
> **MUST**：二者是**不同字段、不同枚举**；文档与契约 **MUST NOT** 混用、互相覆盖或用同一参数 ID；分别登记 15 号参数注册表。

同一表格亦见 `V2 §3.2 L212`（`enum(chat/file/import/tool/system)`）与 `13 §7.2 L237`（并特别加注"**注意**：此处是证据形态 5 值，与 V2 §2.9 的 source_type 5 值是两个不同字段的两个枚举"）。

### C-4. 这两套枚举能否建立映射

**结论：不能建立完整双向映射。** 存在**三套**枚举（不是文档说的两套），schema 那套与权威那套不等价。

**（a）schema 12 值 → 权威「证据形态」5 值**（`chat/file/import/tool/system`）

| schema 12 值 | → 证据形态 | 可映射？ |
|---|---|---|
| `direct_user_statement` | `chat` | ✅ |
| `explicit_user` | `chat` | ✅ |
| `conversation` | `chat` | ✅ |
| `repeated_statement` | `chat` | ✅ |
| `imported_document` | `file` | ✅ |
| `imported` | `file` | ✅ |
| `tool_result` | `tool` | ✅ |
| `system_event` | `system` | ✅ |
| `system_generated` | `system` | ✅ |
| `assistant_observation` | — | ❌ **NO_MAPPING**（这是"来源性质"，不是载体形态） |
| `approved_inference` | — | ❌ **NO_MAPPING**（同上） |
| `human_reviewed` | — | ❌ **NO_MAPPING**（这是审核标记，不是形态） |

→ **9 值可映射，3 值不可映射**。且 `file` 与 `import` 在权威枚举内本身二义（导入文档算 file 还是 import 未定义）。

**（b）schema 12 值 → `MemoryUnit.source_type` 5 值**

`V2 §4.2 L292` **官方已给出逐值映射**（`direct_user_statement→USER_EXPLICIT`、`imported*/tool_result→EXTERNAL_SOURCE`、`system_*→SYSTEM_GENERATED`、`human_reviewed→USER_EXPLICIT(+审计)`、`assistant_observation/approved_inference→ASSISTANT_INFERRED`）。

⚠ **但这个映射被禁止用于 Evidence 字段**：`13 §7.6.3` 明令"文档与契约 MUST NOT 混用"；CMB 亦已自我约束（`evidence-adapter.ts:17-18`：`SOURCE_TYPE_FUTURE_MAP.internal_to_memory_unit = "FORBIDDEN_IN_BRIDGE"`）。该映射只适用于 **MemoryUnit 的 v1→v2 迁移**。

**（c）schema 12 值枚举的语义自相矛盾（最关键的陷阱）**

同一个 enum 里同时装了两类东西：
- **证据形态**（载体）：`conversation`, `imported_document`, `tool_result`, `system_event`, `imported`
- **来源性质 / 审核状态**：`assistant_observation`, `approved_inference`, `human_reviewed`

→ 因此 `Evidence.source_type` 这一个字段**不可能同时满足"形态"和"性质"两种语义**。这正是 `13 §7.6.3` 试图禁止、但 schema 已经犯下的错误。**B-1 中 `source_type` 一项不能靠"补映射表"解决，必须改 schema 或在适配器层拆成两个字段。**

**（d）CMB 现状**

`EvidenceDTO.source_type = "chat"`（单值字面量，`domain.ts:119`）。CMB 侧注释已正确理解两套枚举不可互换（`domain.ts:118`、`evidence-adapter.ts:12-24`），并假定 `"chat"`（v2 §3.2）对应 v1 schema 的 `"conversation"`（`evidence-adapter.ts:13,22`）。
⚠ **但这个假定未在真机验证过**，且 schema 里确实没有 `"chat"`——若直接投喂 `"chat"` 会被 enum 拒绝。CMB 的 `internal_evidence_chat_to_v1_schema: "conversation"` 是**唯一可行的桥接值**。

---

## D. M03 / M04 写入路径真相

### D-1. M03、M04 是已实现代码还是设计规划？

**结论：均为设计文档中的规划，仓库内不存在任何 M03/M04 实现代码。** `VERIFIED_STATIC`

**证据 1 —— 文档自述**：

| 出处 | 原文 |
|---|---|
| `09-记忆摄取Pipeline.md:21` | `| 目标阶段 | P3（M03 + Ingestion Pipeline），**需阶段授权**；P0–P2 仅消费本文件的接口契约 |` |
| `09-记忆摄取Pipeline.md:70` | `| M03 内部实现代码与算法库选型 | FUTURE（P3 待授权） | 本文件只定义规格与契约，不定义源码结构 |` |
| `09-记忆摄取Pipeline.md:71` | `| M04 整合裁决逻辑 | FUTURE（P4 待授权） | …**不定义裁决** |` |
| `DD-AUTHORITY-V2.md:410` | `| P3 | **M03 + Ingestion Pipeline**：… | **新增，需阶段授权** |` |
| `DD-AUTHORITY-V2.md:411` | `| P4 | M04：人物画像、整合、冲突… | 新增，需阶段授权 |` |
| `DD-AUTHORITY-V2.md:34` | "**实施授权仍按阶段授予**，历史裁决'当前 P0–P3 = M01+M02'保持不变" |
| `contracts/api/api-contracts.md:89` | `## 4. M03 API [FUTURE — 契约预留]` |
| `contracts/api/api-contracts.md:93` | `## 5. M04 API [FUTURE — 契约预留]` |
| `api-contracts.md:18` | 标记体系定义：`[FUTURE]`=设计先行**不实施** |

**证据 2 —— 全仓扫描无源码** `VERIFIED_STATIC`
排除 `node_modules`/`pydeps` 后，`.py/.ts/.js/.cjs/.mjs` 文件仅 17 个，全部为：
- 文档维护脚本：`_patch_v2_headers.py`、`sync_ledgers.py`、`sync_stale_refs.py`、`targeted_sync.py`、`update_issue_manifest.py`、`update_outbox_report.py`、`update_reports.py`、`update_status.py`、`index_reports.py`
- schema 校验：`validate_schemas.py`、`validate_schema_fixtures.py`
- **M01/M02 范围的验证 harness**（非 M03/M04）：`sqlite-smoke.cjs`、`sqlite-file-tests.cjs`、`sqlite-recovery-harness.cjs`、`lancedb-harness.cjs`、`outbox-simulation.cjs`

→ **无任何文件实现抽取、拆分、分类、锚点分配、EvidenceBinder、冲突裁决、画像整合**。
→ 目录结构里也没有 `src/`、`lib/`、`app/` 等实现目录（仅 `contracts`/`diagrams`/`evidence`/`reviews`/`test-specs`）。

### D-2. 若未实现，"Evidence 如何进入 M03"是什么状态？

**状态：未定义可执行路径（设计已描述，实现无入口）。**

- 设计上：`09 §7.10 EvidenceBinder`（阶段 10）规定"记忆单元 ↔ 原始证据绑定"，输出 `evidence_refs[]`+`span`+`链接表`（`09:247`），但明确"**证据持久化只能由 M01 在阶段 16 执行**"（`09:572`）。
- 实现上：M03 无代码 → 无入口；M01 `createEvidence` 只有契约签名、无实现代码（见 D-4）。
- 因此该问题当前**不可回答为"有路径"**，只能答"设计有 16 阶段编排，实现为 0"。

### D-3. 定义 M03/M04 职责边界的文档

| 出处 | 内容 |
|---|---|
| `09-记忆摄取Pipeline.md:186` **P-02** | **"MUST NOT 取代 M03"**：候选抽取、分类、拆分、锚点判定必须由 M03 组件执行；协调器只编排（判定逻辑散入协调层 ⇒ 不可测） |
| `09-记忆摄取Pipeline.md:114` **D-AP-02** | **M04 §6.2 为唯一 LLM Gateway 入口**；Pipeline MUST NOT 自建通道 |
| `09-记忆摄取Pipeline.md:117` **D-AP-05** | **confirmed 仅允许 M04 审核通道写入**；`09:199`、`§7.24` 禁止 Pipeline 写 `validation_state=confirmed` |
| `09-记忆摄取Pipeline.md:179` | Pipeline 是**协调层**：负责"顺序、门禁、留痕、重试、降级"，**不负责业务判定** |
| `09-记忆摄取Pipeline.md:194-196` | 模块职责表：**M03**=语义判定组件（规则/价值/抽取/拆分/分类/时间/实体/锚点/情绪/去重/冲突检测）；**M04**=整合裁决 + `confirmed` 唯一写入者 |
| `09-记忆摄取Pipeline.md:205-210` | 调用边界表：Pipeline→M03、Pipeline→M04、Pipeline→M01（**不直接调用**）、Pipeline→LLM（只经 M04 Gateway） |
| `DD-AUTHORITY-V2.md:329` | 阶段 16：`M03/M04 → M01 Commit → Outbox → FTS5/LanceDB` |
| `DD-AUTHORITY-V2.md:331` | "Pipeline 职责：协调，**不直接写 SQLite、不取代 M03、不绕过 M05**" |
| `api-contracts.md:91` | M03 三函数：`submitSource`/`runQualityCheck`/`linkEvidence`；"**M03 不得直接写 active**；候选仅入队" |
| `api-contracts.md:95` | M04 五函数：`reviewCandidate`/`detectDuplicates`/`classifyConflict`/`proposeMerge`/`recomputeObservation`；"**confirmed 写入权唯一归 M04**" |

### D-4. 是否存在任何可调用的 Evidence 写入入口（API 契约层）

**契约层存在唯一签名；实现层不存在。** `VERIFIED_STATIC`

| 层 | 入口 | 状态 | 证据 |
|---|---|---|---|
| **M01 Storage API** | `createEvidence(input:Evidence) → Evidence` | `[P1]` 契约已定义；**仓库内无实现代码** | `api-contracts.md:71` |
| M01 读 | `getEvidence(evidenceId) → Evidence` | `[P1]` 契约 | `api-contracts.md:72` |
| **M03（FUTURE）** | `linkEvidence(candidate, evidence) → candidate` | `[FUTURE]` 仅签名，无实现 | `api-contracts.md:89-91` |
| **REST** | `POST /api/v1/memories/retain` | 接收 `evidence_refs[]` **引用**，不创建 Evidence；且"P1 仅接受显式 items + 手工证据"、"M03 完整自动提取=[FUTURE]" | `api-contracts.md:24-29`（`:29`） |
| CMB 侧 | `EvidenceSink.write(ev)` | Mock 可用；**`MemoryCoreEvidenceAdapter` 全部方法抛 `SINK_UNSUPPORTED`** | `evidence-sink.ts:92-111`（`:94-98`） |

> **D-4 结论**：SSOT 意义上的 Evidence 写入入口是 **M01 `createEvidence`**，权限 `canWrite`，幂等键 `content_hash 去重`（`api-contracts.md:71`）。
> 但 ①无实现代码；②即便有，`evidence.schema.json` 的 `additionalProperties:false` 会拒绝任何携带原句的 Evidence（B-1）；③M03 的 `linkEvidence` 属 FUTURE。
> → **"可调用的 Evidence 写入入口"当前为 0 个（契约 1 个、实现 0 个）**。

### D-5. Memory API 的 9 个接口定义

权威定义在 `DD-AUTHORITY-V2.md` §7（`L348-362`），`14-ChatLLM读写API.md:195-203` 逐字复制，`:210-218` 给出阶段表。`VERIFIED_STATIC`

| # | 接口 | 用途 | read_mode 支持 | 实施阶段 | 行号（14号） |
|---|---|---|---|---|---|
| 1 | `memory.search(query, opts)` | 混合检索（Anchor 路由 + FTS5/LanceDB + RRF） | 4 值全支持 | **P2** | `:210` |
| 2 | `memory.getProfile(subject_id, sections?)` | 人物画像快速读取（人物理解优先） | `summary` | **P5**（画像对象 P4） | `:211` |
| 3 | `memory.getByAnchor(anchor[], subject_id?)` | 指定锚点读取 | 4 值全支持 | **P2**（Anchor Router） | `:212` |
| 4 | `memory.getEvent(event_id)` | 事件（事实） | `summary`/`memory` | **P5**（Event 对象 P1） | `:213` |
| 5 | `memory.getEvidence(memory_id \| evidence_id, mode)` | 原句证据（`related_evidence` 反向） | `evidence`/`related_evidence` | **P2**（证据读取） | `:214` |
| 6 | `memory.getRelated(memory_id)` | 关联记忆/关系/事件 | `summary`/`memory` | **P2** | `:215` |
| 7 | `memory.getEmotionContext(event_id \| memory_id)` | 情绪关联（**默认不注入**） | `summary`/`memory` | **P5**（CR-02 默认关闭） | `:216` |
| 8 | `memory.getMemoryById(memory_id, version?)` | 指定记忆直读（权威回源） | 4 值全支持 | **P2** | `:217` |
| 9 | `memory.propose(candidate)` | **提议写入**（候选，不直接写库） | 不适用（写） | **P5**（M03=P3、M04=P4） | `:218` |

统一返回 MUST 含 9 字段：`memory_id, content, semantic_type, anchor, source_type, confidence, evidence_refs, lifecycle(tier+status), visibility`（`V2:361`、`14:390`）。
**每次读取 MUST 经 M05；`memory.propose` 只产生候选，M01 为唯一权威写入执行者**（`V2:362`、`14:206`）。

> **与 Evidence 的关系**：9 个接口中**只有 `getEvidence` 读 Evidence，没有任何一个写 Evidence**。写入只能走 `memory.propose`→候选→M04→M01 `createEvidence`。Evidence 本身**没有对外写 API**。

---

## E. 撤销与生命周期

### E-1. Evidence 有 `revoked` 字段吗？类型？

**有。boolean。**

| 来源 | 类型 | 必填 | 默认 | 行号 |
|---|---|---|---|---|
| `evidence.schema.json` | `boolean` | **否**（未进 required） | `false` | `:34` |
| 权威 `V2 §3.2` / `13 §7.2` | `boolean` | **是 ✅** | false | `V2:L223` / `13:L248` |
| CMB `EvidenceDTO` | `boolean` | 是（恒显式写 `false`） | — | `domain.ts:130`；`evidence-adapter.ts:65` |

语义（`13 §7.2 L248`、`V2 L223`）：失效标记；`revoked=true` ⇒ 关联记忆 **MUST** 标记"证据缺失"；置 `true` **MUST** 记录 provenance/audit（`13:L258`）。

⚠ **schema 与权威在此不一致**：权威要求 `revoked` **必填**，schema 未列入 required → schema 弱于权威（`REQUIRED_MISMATCH`，因 CMB 恒显式提供故不 blocking）。

### E-2. 有没有 `revoked_at` / `superseded_by`？

**两者均不存在。** `VERIFIED_STATIC`（全仓 grep）

| 字段 | 检索结果 |
|---|---|
| `revoked_at` | **0 命中**（全部 `.md`/`.json`，排除 `node_modules`/`pydeps`/`.lance`） |
| `superseded_by` | 仅 2 命中，且**与 Evidence 无关**：`contracts/state-machines/state-machines.md:82`、`03-系统运行与协作协议.md:455`——均为 Outbox 的 `last_error=superseded_by_version` |

**Evidence 的"被取代"靠约定而非字段**：`13 §7.4.3 L313` 规定"如确需订正，**MUST 新建 Evidence**（新 `evidence_id`、新 `content_hash`、旧条 `revoked=true` 且保留）"。即：
- **无 `superseded_by` 指针** → 新旧证据之间**没有显式链**，只能靠 `revoked=true` + 人工追溯。
- **无 `revoked_at`** → **无法回答"何时失效"**。`13 §7.2` 的"Evidence 七问"（`L443-453`）中 Q6 只要求 `revoked` + provenance，即失效时间只能间接从审计记录反推。
- 撤销**有状态、无时间戳、无指向**，是生命周期的完整度缺口（登记为 `USER_DECISION_REQUIRED`）。

### E-3. 已确认的 MemoryUnit 如何感知其来源 Evidence 失效？（反向传播接口）

**存在规范要求，不存在可调用的接口；仅有"读取侧标记"+"FUTURE 重算"。** `VERIFIED_STATIC`

**规范要求（3 处，均为 MUST）**：

| 出处 | 要求 |
|---|---|
| `13 §7.4.5 L331` | `revoked=true` **MUST 触发**：所有经 `memory_evidence_links` 关联到该证据的 MemoryUnit **MUST 标记"证据缺失"**（`evidence_missing`） |
| `13 §7.4.5 L333` | 若一条 MemoryUnit 的**全部**证据均 revoked：**MUST NOT** 作为已验证事实返回；**SHOULD** 降为 `unverified`/`disputed`，保留来源链与审计 |
| `13 §7.4.5 L334` | 证据失效 **MUST NOT** 自动删除 MemoryUnit（须走 M04+M05 通道） |
| `13 §7.4.5 L335` | **MUST** 同步使派生视图（PersonProfile/Observation）失效或重算 |
| `02-核心算法规范.md:742`（§7.9.5-5.5） | 来源被删/改/权限收紧/证据撤销 ⇒ 依赖的 Observation **MUST** 置 `needs_recompute`/`invalidated`/`restricted`，记 `invalidated_reason`；Relation **MUST** 降级 `availability_state` |
| `12-人物画像整合.md:387` | 每条记录 MUST 携带 `source_evidence_ids`；若其证据 `revoked=true`，**MUST** 在 `evidence_missing` 中标记 |

**实现路径（两条，均不构成"反向传播接口"）**：

| 路径 | 机制 | 状态 |
|---|---|---|
| **读取侧标记** | `memory.getEvidence` 在 `related_evidence` 流程**第 5 步**逐条校验 `revoked`，`revoked=true` ⇒ 标"证据缺失"、**MUST NOT** 返回原句正文（`13 §7.4.1 L298`）；响应结构含 `evidence_missing[]`（`13:L751`、`12:L377`） | 设计完整；依赖 M02（P2），**无实现代码** |
| **写入侧重算** | 唯一相关接口 = M04 `recomputeObservation(observationId)`（`api-contracts.md:95`），仅覆盖 Observation，**不覆盖 MemoryUnit 本身** | **`[FUTURE]`** |

**缺口清单**：
1. **不存在 `markEvidenceMissing(evidence_id)` 或 `propagateRevocation(...)` 之类的接口签名**——全仓 `api-contracts.md` 12 个 M01 方法（`L64-76`）+ 5 个 M04 方法（`L95`）中**没有**任何一个承担"Evidence 撤销 → 反向标记 MemoryUnit"。
2. `memory_evidence_links` 表（`01:L669`：`memory_id`/`evidence_id`/`support_kind`/`created_at`）是唯一可反查的结构，但**只有表，没有消费该表做传播的接口**。
3. **`evidence_missing` 不是持久化状态**——它在响应/派生视图里是**瞬态标记**；`MemoryUnit.schema` 中**没有** `evidence_missing` 字段（`memory-unit.schema.json:39-218` 的 28 字段无此项）。因此"标记必须显式、可读取"（`13:L332`）在**存储层无承载**，每次读取都要重算。
4. 反向传播**跨 3 个未实现模块**（M02 读取 + M03/M04 重算 + M01 事务），当前**链路全空**。

→ **结论**：`E-3` 的答案是"规范要求齐全、机制设计可行、**接口不存在、存储无承载、实现为零**"。

### E-4. 有无 `retention_policy` 定义？

**有字段、有纪律要求，但无取值域。** `VERIFIED_STATIC`

| 层 | 内容 | 行号 |
|---|---|---|
| Schema | `retention_policy`: string\|null, maxLength 128, **非必填** | `evidence.schema.json:33` |
| 权威表 | `retention_policy`: string\|null, 非必填 | `V2 §3.2 L222` / `13 §7.2 L247` |
| 纪律 D-03 | **MUST 为受控词表**（实施期确定，登记 15 号参数注册表）；**MUST NOT** 由 LLM 自由生成；导出边界仍 **BLOCKED** | `13 §7.3 L270` |
| 纪律（§7.2-6） | MUST 与 K-2 导出边界分离：只描述**保留**，MUST NOT 被解释为导出已解禁 | `13 §7.2 L257` |
| CR-06 | 原句长期保存 vs 数据最小化/导出边界未定；**不得声称"记忆文本不出本机"** | `V2:450`、`DOCUMENT-CONFLICT-REGISTER-V2:37`、`OPEN-ISSUES:90`(OI-030) |
| CMB | 无该字段 | `domain.ts:116-134` |

→ **词表未定义、取值域未定、导出边界 BLOCKED** ⇒ 判为 `INSUFFICIENT_INFO`（不是"适配器改一下就行"，也不是"Memory Core 改一下就行"，而是**前置裁决未完成**）。

---

## F. 契约结论分类

按任务书 6 类，对 32 个字段差异逐项归类（机器可读见 `schema-diff.json.field[].conclusion`）。

| 分类 | 数量 | 字段 |
|---|---|---|
| **UPSTREAM_CHANGE_REQUIRED**（必须改 Memory Core） | **10** | `source_type`、`original_text`、`recorded_at`、`timestamp`、`speaker_id`、`bot_id`、`group_id`、`revoked`、`context_before`、`context_after` |
| **ADAPTER_WORK_REQUIRED**（CMB 改适配器可解决） | **4** | `evidence_id`、`scope`、`produced_by_run`、`produced_by_blocks` |
| **USER_DECISION_REQUIRED**（需用户裁决） | **3** | `content_hash`、`visibility`、`revoked_at` |
| **INSUFFICIENT_INFO**（前置信息不足） | **1** | `retention_policy` |
| **RUNTIME_EXPERIMENT_REQUIRED**（需跑实验才能定） | **0** | —（字段级为 0；见下方说明） |
| **NO_ACTION**（兼容，无需动作） | **14** | `source_id`、`source_version`、`source_locator`、`source_session_id`、`source_message_id`、`source_subject_id`、`excerpt`、`observed_at`、`actor_id`、`tenant_id`、`visibility_scope`、`provenance_id`、`extraction_method`、`model_version` |

**关键归类说明**：

| 字段 | 为何归此类 |
|---|---|
| `original_text` | schema 与权威文档**都同意它必须存在**（`13:L242`、"MUST 原样保存"），只有 schema 错了；CMB 无法通过改适配器绕过（`additionalProperties:false`），**必须升 schema** |
| `speaker_id` / `timestamp` | 同上：权威必填/可选字段，schema 缺失。CMB 无法兼容 |
| `bot_id` / `group_id` | schema 与权威**都没有**；需 Memory Core 决定承载位（并入 `scope`/新增字段/明确不支持）——属上游变更 |
| `recorded_at` | schema **单方面新增**的必填字段，权威与 CMB 均无 ⇒ 必须由上游决定删必填或补权威定义 |
| `evidence_id` | schema 与权威**一致**要求 uuid，错在 CMB ⇒ 适配器改（可用 UUIDv5 从 `(content_hash, source_locator)` 派生，保留 ADR-003 幂等语义） |
| `scope` | schema 与权威一致必填；CMB 的 `Visibility` 五值与 `scope` 枚举**逐值相同** ⇒ 适配器直接投影即可 |
| `produced_by_run` / `produced_by_blocks` | CMB 审计字段，Memory Core 无对等概念 ⇒ 适配器层剥离或映射到 `provenance_id` |
| `content_hash` | 结构能过（64 hex），但语义违反 `13 §7.2-4`（须 sha256(original_text)）。**双字段并存会改变 Evidence 身份与去重语义**（ADR-003:94）⇒ 业务取舍须用户裁决 |
| `visibility` | 5 值→3 值映射有 3 个值（`session_only`/`project_scoped`/`system_internal`）**无处安放**；且 schema 无该字段（AI-2）⇒ 需裁决表示法 |
| `revoked_at` | 三方皆无；是否需要"撤销时间"是业务裁决 |
| `retention_policy` | 受控词表未定 + 导出边界 BLOCKED（CR-06/OI-030）⇒ 信息不足 |

**关于 `RUNTIME_EXPERIMENT_REQUIRED = 0` 的诚实说明**：字段级归零不代表"不需要实验"。有 **1 项机制级**前置实验，但它不改变任一字段的分类：
- **EXP-1（建议）**：若采纳"`content_hash` 双字段并存"（上游哈希 + `sha256(original_text)`），需实测其对 **Evidence 身份键与去重行为**的影响（ADR-003 §5 指出会改变身份、影响去重；`RISK-REGISTER.md:218` 已建议"接入前与 Memory Core 侧确认"）。该实验的结论决定 `content_hash` 最终归 `ADAPTER_WORK_REQUIRED` 还是保持 `USER_DECISION_REQUIRED`，故当前仍按裁决类记账。

---

## G. 证据纪律与未验证项

**已直接读取并核实（`VERIFIED_STATIC`）**：`evidence.schema.json`（全文 36 行）、`memory-unit.schema.json`（293 行）、`DD-AUTHORITY-V2.md`（465 行）、`DD-AUTHORITY.md`（前 60 行 + 定向检索）、`DOCUMENT-CONFLICT-REGISTER-V2.md`（全文 77 行）、`13`（§7.2/§7.3/§7.4/§7.6/§8.1）、`04`（全文检索 + §7.11）、`01`（§7.8/§7.9 + 表结构）、`09`（检索 + 阶段表）、`api-contracts.md`（全文 125 行）、`error-codes.md`（全文 58 行）、`14`（检索 + 9 接口表）、`12`（§7.7.1）、`02`（检索 §7.9.5）、`reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md`（§1 前 30 行）、CMB `domain.ts`（全文 196 行）、`evidence-adapter.ts`（全文 76 行）、`evidence-sink.ts`（全文 111 行）、`ADR-003`（全文 115 行）。

**明确未做/未验证**：

| 项 | 状态 | 说明 |
|---|---|---|
| Memory Core 侧任何**运行时**验证 | **未做** | 仓库无 M01–M05 实现代码，无服务可跑；`evidence/second-round/*-result.json` 是 M01/M02 范围的 harness 输出，与 Evidence 契约无关 |
| `evidence.schema.json` 是否能拒绝/接受真实 CMB DTO | **未实跑** | 未执行 JSON Schema 校验器（属"读取审查"范围外）；结论为静态推导（`additionalProperties:false` + required 差集），标 `INFERRED`-但机制确定 |
| `pendingBuilds`/`15-参数注册表` 中 source_type 参数 ID | **未核对** | `13 §7.6.3` 要求"分别登记 15 号参数注册表"，本轮未打开 15 号逐条比对 |
| DSH 宿主侧 Evidence 语义 | **未涉及** | 不在本次范围 |
| `evidence/second-round/pydeps/*` 下 12 个目录 | **无法读取** | `Get-ChildItem` 返回 `PermissionDenied`（ACL），与 Evidence 契约无关（Python 依赖副本） |
| `candidate.schema.json` 的 evidence 相关字段 | **未审** | 与 B-1 无直接关系，留待其他 Agent |

---

## H. 建议（供 Lead 汇总裁决）

> 本节仅为建议，**未修改任何文件**。按任务书分类。

| # | 建议 | 分类 | 依据 |
|---|---|---|---|
| 1 | **B-1 裁决前，禁止实现 `MemoryCoreEvidenceAdapter`**（现状已正确：全部抛 `SINK_UNSUPPORTED`），保持 `evidence-sink.ts:92-111` 不变 | USER_DECISION_REQUIRED | `INTEGRATION-READINESS.md:203,276`、`ADR-006:65` |
| 2 | Memory Core 侧应**升 `evidence.schema.json` 至 V2 §3.2 字段集**（补 `original_text`/`speaker_id`/`timestamp`/`visibility`/`context_*`；裁决 `actor_id`↔`speaker_id`、`observed_at`+`recorded_at`↔`timestamp`、`tenant_id`/`bot_id`/`group_id` 承载），并同步 `memory-unit.schema.json:50,166` 的自指 | UPSTREAM_CHANGE_REQUIRED | 本报告 A-1/A-6/AI-1 |
| 3 | 将 B-1 登记进 `DOCUMENT-CONFLICT-REGISTER-V2.md`（建议编号 `CR-08`）与 `OPEN-ISSUES.md`（`OI-032`），并**重开** `CROSS-DOCUMENT-CONSISTENCY-REPORT.md:39` 的 PASS 判定（其比对范围漏了 V2 §3.2） | UPSTREAM_CHANGE_REQUIRED | 本报告 A-4 |
| 4 | `source_type`：Evidence 侧必须**二选一**——(a) schema 改回权威 5 值（`chat/file/import/tool/system`）；(b) 拆成两个字段（形态 + 性质）并各自受控。**不得**靠"补映射表"掩盖（12 值中 3 值无形态映射） | UPSTREAM_CHANGE_REQUIRED + USER_DECISION | 本报告 C-4 |
| 5 | 裁决 `content_hash` 语义：接受上游 blob 哈希（偏离 `13 §7.2-4`，需改权威文档）或双字段并存（需改 Evidence 身份，跑 EXP-1） | USER_DECISION_REQUIRED | `ADR-003:82-96` |
| 6 | 裁决 `visibility` 5→3 映射中 `session_only`/`project_scoped`/`system_internal` 的归宿；并把 `visibility` 加进 `evidence.schema.json`（否则 `04` 号红线不可执行） | USER_DECISION_REQUIRED | 本报告 AI-2 |
| 7 | 若 M03/M04 要进入实施，需先补 **Evidence 写入入口**（M01 `createEvidence` 实现）与 **撤销反向传播接口**（当前 0 个签名） | UPSTREAM_CHANGE_REQUIRED | 本报告 D-4/E-3 |
| 8 | `21-REPOSITORY-BASELINES.md` §4 的**行数列**应更正（DD-AUTHORITY-V2 465 行、13 号 1034 行、04 号 451 行），哈希列可保留 | 文档修正 | 本报告 §0 |

---

## I. 完成状态

| 交付物 | 路径 | 状态 |
|---|---|---|
| 本报告 | `E:\[PROJECT]\CMB-v0.1-review-pack\06-MEMORY-CORE-CONTRACT-AUDIT.md` | ✅ |
| 逐字段差分报告 | `E:\[PROJECT]\CMB-v0.1-review-pack\08-EVIDENCE-SCHEMA-DIFF.md` | ✅ |
| 机器可读差分 | `E:\[PROJECT]\CMB-v0.1-review-pack\schema-diff.json` | ✅（`JSON.parse` 通过；32 字段 / blocking 12；summary 各计数经解析器逐类复核一致） |

**未修改清单（只读确认）**：Memory Core 目录下 0 次写入；`contracts/schemas/*` 0 次修改（审查后复查 SHA-256 仍为 `F4D087C3AADAC774` / `A204CE745D31878C`）；CMB 原型目录 0 次写入。
