# CMB v0.1 — 安全与隐私审查（11-SECURITY-PRIVACY-AUDIT）

采集：2026-11-06 ｜ 执行：Agent E（只读审查，未修改任何项目文件）
审查对象：`E:\[PROJECT]\CMB-v0.1-prototype`（非 git 工作区）
上游事实源：`E:\[PROJECT]\refs\billion-context` @ `0aace2f`（VERIFIED）
宿主：`D:\qq-bridge`（非 git，按源码）、DSH `app.asar`（**不可读**，见 07 报告 §1.4）
规范源：`C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design`

方法：静态逐行（全部 16 个 `src/*.ts` + 2 个适配器 + 夹具 + CLI）+ **4 个 TEMP 探针文件 / 20+ 用例**（`%TEMP%\cmb-e-probe\probe1..4.mjs`）。

> **本文的三条主结论（详见后文）**
> 1. **日志/报告/Ledger 确实不含正文**（实测 6 个敏感串 × 4 个面 × 2 种模式，0 命中）——这一层是干净的。
> 2. **但"系统提示词"会被当成用户证据写进记忆**：`validator` 拦的是 `content_type === "system"`，而上游的 system 提示词是 `role === "system"` + `content_type === "text"`。实测 5 条 Evidence 里有 1 条是系统提示词，**且它恰好是唯一豁免 speaker 校验的那一类**。
> 3. **`DRY_RUN` / `DISCOVERY` / `OFF` 全部照写不误**：`mode` 在引擎里没有任何分支，实测 6 种模式全部 `evidence_written=1`。

---

## B1 — 日志泄露

### B1.1 全部日志调用点（`this.log.*`，共 10 处）逐个核查

`StructuredLogger`（`observability.ts:17-50`）只写内存数组，**`toJsonl()` 在 `src/` 内 0 个调用者**，无日志文件落盘（grep：`toJsonl` 仅定义处 + 测试）。因此日志泄露面 = ①进程内 `events[]`（可达测试/未来接线）②CLI stdout ③Ledger 文本列。

| # | 调用点 | event | `data` 内容 | 是否可能含正文 |
|---|---|---|---|---|
| 1 | `reconciliation-engine.ts:77` | `run_start` | `{run_id, mode, opts:{sessionId,blockId,recoveryOnly,revokeScan}}` | 否（结构化参数） |
| 2 | `reconciliation-engine.ts:119` | `block_terminal_skip` | `{status}` | 否 |
| 3 | `reconciliation-engine.ts:139` | `state_reset_fail` | `{message, from}` ← `err.message` | **条件性**（见 B1.2） |
| 4 | `reconciliation-engine.ts:187` | `identity_unresolved` | `{message}` ← provider 抛错的 message | 否（现仅含 session_id） |
| 5 | `reconciliation-engine.ts:200` | `validate_fail` | `{message}` ← `ValidationDiagnostic.message` | 否（现仅含码/角色/hash 前缀） |
| 6 | `reconciliation-engine.ts:277` | `evidence_write_fail` | `{message}` ← sink 抛错的 message | **条件性** |
| 7 | `reconciliation-engine.ts:301` | `block_error` | `{message}` ← 任意异常的 message | **条件性** |
| 8 | `reconciliation-engine.ts:309` | `ledger_persist_fail` | `{message}` ← sqlite 异常 message | **条件性** |
| 9 | `reconciliation-engine.ts:354` | `run_end` | `{run_id, evidence_written, failed}` | 否 |
| 10 | `reconciliation-engine.ts:373` | `evidence_revoked` | 无 `data`，仅 `locator` | 否 |

**结论**：**没有任何一处显式传入正文**。风险全部集中在"透传 `err.message`"的 5 处（#3/#4/#6/#7/#8）——当前所有错误文案都是结构化短语（见 B1.2），但 `data` 无白名单、无脱敏层，一旦接真实 sink/宿主，其错误文本若含片段就会原样入日志。

### B1.2 错误消息（`CmbError` 构造点）是否携带正文

全部 26 个 `new CmbError(...)`（`bc-source.ts` 3、`config.ts` 7、`evidence-sink.ts` 10、`host-identity.ts` 6）+ 3 个 `new Error`（`evidence-adapter.ts:46,49`；`engine.ts:255`）逐一核查：

| 位置 | message 内容 | 携带正文？ |
|---|---|---|
| `bc-source.ts:102` | `getBlockHistory unsupported ... (ingest.ts:425) ... (store.ts:1077-1087)` | 否 |
| `bc-source.ts:109` | `no supersede producer exists` | 否 |
| `bc-source.ts:148` | `payload_json is not valid JSON`（details: session_id/block_id） | 否 |
| `config.ts:89-117` | 全部为字段名 + 取值 | 否 |
| `evidence-sink.ts:38-47` | `simulated ...` + `{locator}` | 否 |
| `evidence-sink.ts:53` | `evidence identity collision with different content`（**未附带内容**） | 否（正确） |
| `host-identity.ts:59,62,81,109,123,134` | 均只含 `session_id` / 表项缺失描述 | 否 |
| `integrity-validator.ts:59-104`（诊断） | `tenant_id missing` / `group_id missing` / `speaker unresolvable for ${role}/${content_type}` / `blob sha256 ${actual.slice(0,12)}… != declared ${...}` / `original_text empty or blob missing` | 否（hash 只取前 12 位） |
| `source-resolver.ts:157-171` | `blob missing for message ${locator} (hash=${short(hash)})`；`${n} covered message ids have no persisted row (ids redacted)` | 否（**显式 redacted**） |
| `evidence-adapter.ts:46,49` | `SCHEMA_INVALID: empty original_text at ${locator}` / `missing content_hash at ${locator}` | 否 |

**唯一隐忧**：`engine.ts:304` 把异常 `message` 原样写入 `block_ledger.failure_reason`，`engine.ts:139/277/301/309` 把它写进日志 `data.message`。这是**唯一会把任意第三方文本带进持久层/日志的通路**（当前输入良构，属"未来接线风险"）。

### B1.3 异常堆栈

- 引擎内 6 个 `catch` 全部只取 `err instanceof Error ? err.message : String(err)`，**从不读 `err.stack`**（`engine.ts:137,185,275,299,307`）。✅
- 唯一打印完整错误对象的点：`cli/cmb.ts:68` `console.error("FATAL", err)` —— 仅在 `main()` 整体 reject 时触发（配置非法、Ledger 打不开等启动期错误）。此时打印的是 `CmbError` 的 `code/message/details` + 调用栈（含文件路径与行号），**不含正文**（因为这些错误对象本身不含正文）。✅

### B1.4 CLI 输出

`cli/cmb.ts:61` `console.log(reportToHuman(report))`。`reportToHuman`（`observability.ts:80-93`）输出：`run_id`、`mode`、`duration`、计数、`latency`、`failure_reasons`（错误码→计数）、`notes`（**恒为空数组**，引擎从不 push）。**无正文**。✅
（`reportToHuman` 已完整实测：见 B1.7。）

### B1.5 Ledger 表里存了什么

| 表 | 列 | 是否含正文 |
|---|---|---|
| `block_ledger`（`bridge-ledger.ts:67-81`） | `bridge_key(=sha256)`、`session_id`、`block_id`、`covered_hash`、`block_version(=sha256)`、`block_status`、时间戳、`evidence_count`、`summary_hash`、`status`、`failure_reason` | **否**（`failure_reason` 为条件性文本列，见 B1.2） |
| `message_ledger`（`:82-92`） | `source_message_key(=session·msgid)`、`session_id`、`message_id`、`ledger_seq`、`source_locator`、`content_hash`、`evidence_id`、`processing_status`、`revoked_status` | **否**（只有 hash 与 ID） |
| `run_ledger`（`:93-105`） | `run_id`、时间、6 个计数、`duration_ms`、`error_summary` | **否**（`error_summary` = `JSON.stringify(failureReasons)`，只有码与计数，`engine.ts:352`） |

**重要旁证**：Ledger **不存任何身份字段**（无 tenant/bot/group/speaker）——隐私上干净，但代价见 07 报告 §2.9（身份不可修正、撤销不可重建）。

### B1.6 `ReconciliationReport` 字段

`observability.ts:52-78`：`run_id / mode / started_at / finished_at / duration_ms / counts{12 个整数} / failure_reasons{码:整数} / latency{avg,p95} / notes[]`。**无任何文本正文**；`notes` 引擎从不写入（死字段）。✅

### B1.7 实测：带敏感文本跑一遍，全输出 grep

探针 `%TEMP%\cmb-e-probe\probe1.mjs`：构造一个块，7 条消息分别携带 6 个唯一敏感串（`SECRET-USER-正文-甲`、`SECRET-SYSTEMPROMPT-乙`、`SECRET-DEVELOPER-丙`、`SECRET-REASONING-丁`、`SECRET-TOOLRESULT-戊`、`SECRET-UNKNOWN-TYPE-己`）+ 1 条空正文；两种 speaker 口径（hint / no-hint）各跑一遍；收集 4 个面：`log.toJsonl()`、`JSON.stringify(report)`、`reportToHuman(report)`、Ledger 全表快照。

```
=== secret leak scan over log/report/ledger ===
NO_SECRET_LEAK_FOUND
```

**结论（VERIFIED_RUNTIME）**：日志 / 报告 / 人类可读报告 / Ledger 四个面**均无正文泄露**，6 个敏感串 0 命中。
**但同一实验暴露了另一件事**：这 6 个串里有 **3 个进了 Evidence**（系统提示词、developer 消息、未知类型消息）——泄露不在日志面，在**收件箱面**。见 B2。

**附带发现（内存，非持久）**：`StructuredLogger.events[]` 无上限（`observability.ts:18,26`），长跑会单调增长；`toJsonl` 无落盘，故当前不构成磁盘泄露，但一旦接线成文件日志，"`data` 无白名单"立刻变成持久风险。

---

## B2 — 消息类型策略矩阵

### B2.1 上游真实取值（先定事实，再谈策略）

| 维度 | 真实取值 | 证据 |
|---|---|---|
| **role** | `"user" \| "assistant" \| "system" \| "tool"`（**4 值**） | `kernel/src/types.ts:3` |
| **contentType** | `"text" \| "tool-call" \| "tool-result" \| "reasoning"`（**4 值，连字符**） | `kernel/src/types.ts:5-6` |
| **DB `messages.kind`** | `= contentType ?? "message"`（**旧数据可能为 `"message"`**） | `storage/schema.ts:148`（默认 `'message'`）；`storage/ingest.ts:195,341` |
| **`developer`** | **不是 kernel role**，但**是真实 wire role**（codex ≥0.153 会发），由 `compat-roles.ts` 在 400 后改写 | `compat-roles.ts:86`；`src/util.ts:635-640`；`affinity-neutral.ts:111` |
| **多模态/附件** | **没有 contentType**；图片等作为同一消息的 media 字节存在 `content` CAS + `message_media` 表（`(session_id, seq, media_hash)`），摘要里的再访问句柄用 `MEDIA_REFS_MARKER` 段承载 | `schema.ts:267-272`；`kernel/src/media-refs.ts:139-140` |
| **tool 判定** | 官方语义：`contentType === "tool-call" \|\| "tool-result"`（不看 `toolName`） | `kernel/src/message-kind.ts:11-15` |

CMB 侧取值来源（`bc-source.ts:178,192`）：先取 DB `kind`，若 blob 里 `core.contentType` 是字符串则覆盖。→ 实际取值集合 ⊇ `{message, text, tool-call, tool-result, reasoning}`。

### B2.2 CMB 的 Validator 实际策略（`integrity-validator.ts`）

```
:94  if (m.content_type === "reasoning") → UNSUPPORTED_MESSAGE_TYPE   ← 命中
:95  if (m.content_type === "system")    → UNSUPPORTED_MESSAGE_TYPE   ← 永不命中（system 是 role，不是 contentType）
:70  speakerExempt = role∈{tool,system} || content_type∈{toolCall,toolResult}  ← 后两项是驼峰，上游永远是连字符 → 永不命中
:71  if (!speakerExempt && !speaker) → SPEAKER_UNRESOLVED
```

**这两个"驼峰/角色混淆"的判定错误，是整个 B2 的根因。**

### B2.3 策略矩阵（要求的 10 类）

| 类型 | 上游是否存在 | CMB 是否恢复 | 是否允许 Evidence | 是否允许群共享 | 默认处理 | 证据 |
|---|---|---|---|---|---|---|
| **user** | ✅ role=`user`，ct=`text`/`message` | ✅ 恢复正文（blob.text） | ✅ 允许 | ⚠️ 取决于 `visibility`（`owner_private` 或 `group_shared`，单由绑定表决定） | 需 speaker：真实适配器恒 null → **全部 `SPEAKER_UNRESOLVED` 被拒**；Mock/部署表有值时通过 | `types.ts:3`；`bc-source.ts:189`；`validator.ts:69-73`；实测 P6（0 条 user Evidence） |
| **assistant** | ✅ role=`assistant` | ✅ | ✅ | 同上 | 同 user（也需 speaker，同样全拒） | 同上 |
| **system** | ✅ role=`system`，ct=`text` | ✅ 恢复正文 | 🔴 **当前会写成 Evidence**（实测） | 🔴 会，且随会话 visibility | **未拦截**：`validator:95` 查的是 `content_type==="system"`（永不成立）；`speakerExempt` 因 `role==="system"` 为真而豁免 speaker → 唯一通行无阻的类别 | `types.ts:3`；`validator.ts:70,95`；实测 probe1（`SECRET-SYSTEMPROMPT-乙` 落 Evidence，seq=1） |
| **developer** | ⚠️ wire 上有（codex ≥0.153），kernel 类型无；DB `role` 原样存 | ✅（`role` 是自由字符串，`bc-source.ts:204` `String(r.role)`） | 🔴 **当前会写成 Evidence**（实测） | 🔴 会 | **未拦截**：既非 `system` 角色豁免、也无类型规则；有 speaker 则通过。实测有 bound speaker 时确实通过 | `compat-roles.ts:86`；`bc-source.ts:204`；`validator.ts:94-95`；实测 probe1（`SECRET-DEVELOPER-丙` 落 Evidence，seq=2） |
| **tool** | ✅ role=`tool`，ct=`tool-result` | ✅ | ✅ **允许**（工具返回值按设计是证据候选） | 同会话 visibility | 无 speaker 要求（`role==="tool"` 豁免）；实测通过（`SECRET-TOOLRESULT-戊`，seq=5） | `types.ts:3,6`；`validator.ts:70`；实测 probe1 |
| **toolResult（`tool-result`）** | ✅ | ✅ | ✅ 允许 | 同上 | 实际靠 `role==="tool"` 豁免；**`content_type==="toolResult"`（驼峰）分支永不命中**——若某宿主把 tool-result 放在 `role:"assistant"` 上，则会被要求 speaker（意外 fail-closed） | `validator.ts:70`；`message-kind.ts:11-15` |
| **toolCall（`tool-call`）** | ✅ | ✅（`text` 通常是工具调用参数） | ⚠️ 允许（有 speaker 时） | 同上 | ①真实路径：`role=assistant` 且 `content_type==="toolCall"`（驼峰）**不命中豁免** → 被 `SPEAKER_UNRESOLVED` 拒（意外 fail-closed）；②Mock/有 bound speaker 时：`text` 为空 → `CONTENT_MISSING` 拒（实测 seq=4）；`text` 为参数 JSON → **通过并成为证据**（工具参数不是"用户原话"，却会进记忆） | `validator.ts:70,80-81`；`types.ts:6`；实测 probe1（`CONTENT_MISSING:1`） |
| **reasoning** | ✅ | ✅（但解析层先排除） | ❌ 拒绝 | ❌ | **两道拦截**：`source-resolver.ts:113-123` 解析层过滤 + `validator.ts:94` 兜底；且 `bc-source.ts:214` 打 `excluded_reason` | `types.ts:6`；`source-resolver.ts:113-123`；`validator.ts:94`；实测 `REASONING_EXCLUDED:1` |
| **multimodal（图文混排）** | ✅ 存在（media 字节 + `message_media` 表 + `MEDIA_REFS_MARKER` 段） | 🔴 **只恢复文本，媒体被静默丢弃** | ⚠️ 文本部分成为 Evidence | 同上 | **既非拒绝、也非隔离——静默降级**：`bc-source.ts:183-192` 只取 `core.text`，从不 JOIN `message_media`，也不解析 media ref 段；Evidence 里没有任何"此消息原本含 N 个附件"的标记 | `schema.ts:267-272`；`media-refs.ts:139-140`；`bc-source.ts:172-198` |
| **attachment（纯附件/无文本）** | ✅（可推断：`text` 为空、`message_media` 有行） | 🔴 不恢复 | ❌ 实际被拒（`original_text` 空 → `CONTENT_MISSING` → CORRUPTED） | — | **误判为"损坏"**：附件消息被计入 `corrupted`，语义错误（不是损坏，是 CMB 不支持） | `validator.ts:80-81`；`bc-source.ts:175-189` |
| **unknown（任意未知 contentType，如 `mystery-kind`）** | ⚠️ 上游 schema 无约束（`kind` 是自由 TEXT），第三方/新版本可写任意值 | ✅ 原样带入 | 🔴 **静默放行！** | 🔴 会 | **没有 default-deny**：`validator.ts:94-95` 只硬编码了两个字符串，其余一律通过。实测 `content_type="mystery-kind"` 的消息**无任何诊断**直接成为 Evidence（`SECRET-UNKNOWN-TYPE-己`，seq=6） | `validator.ts:93-95`；实测 probe1（`failure_reasons` 里没有该消息的任何码，Evidence 里有） |

### B2.4 回答任务书的重点问题：未知类型会怎样？

> **被静默放行，不是拒绝，也不是隔离。**（VERIFIED_RUNTIME）

- 代码层：`integrity-validator.ts:93-95` 是**黑名单**（只列 `reasoning`、`system` 两个字符串），没有 `switch` + `default: reject`，没有"允许集合"白名单。
- 实测层：`content_type="mystery-kind"` 的消息 → `failure_reasons` 无条目、`counts` 无变化、Evidence 正常写出（`evidence: {seq:"6", ...}`）。
- 后果：**上游任何新类型（未来的 `image`、`audio`、`citation`、`file`…）都会默认为"和用户原话等价"，直接进记忆**。这与"未知即拒绝"（fail-closed）相反，与 Memory Core `04 §7.7`「缺失或不可确认 → 拒绝」也不一致（类型可确认性同属"关键字段"）。
- 同一黑名单写法的第二个洞：`content_type === "system"` 这一条**永远不会命中**，因为它把一个 role 当成了 contentType 比较——真正需要拦的 `role === "system"` 反而被 `speakerExempt` 放行（见矩阵 system 行）。

### B2.5 策略缺口汇总（按严重性）

| # | 缺口 | 严重性 | 后果 |
|---|---|---|---|
| 1 | `role=system` 的提示词成为 Evidence | **CRITICAL** | 系统提示词（含角色设定、内部规则、可能的密钥/PII）写入长期记忆库，可被检索/共享 |
| 2 | 未知 content_type 静默放行 | **HIGH** | 上游演进即静默扩大记忆面；无审计 |
| 3 | `role=developer` 成为 Evidence | **HIGH** | 同上，且 developer 常承载系统级指令 |
| 4 | 多模态/附件静默丢弃或误判损坏 | **MEDIUM** | 数据完整性：记忆缺少"图里说了什么"；附件消息被计为 corrupted 污染指标 |
| 5 | `tool-call` 参数会被当证据 | **MEDIUM** | 工具参数（可能含密钥、路径、SQL）进记忆 |
| 6 | 豁免集合用驼峰匹配连字符 | **LOW-MEDIUM** | 行为与注释/文档不符，未来接入即错 |

---

## B3 — 加密与失败关闭

### B3.1 `BILI_ENCRYPTION_KEY` 生效时上游行为（VERIFIED_SOURCE）

`src/persist.ts:535-569`：

- **key 来源**：**仅环境变量**（`:540` `process.env.BILI_ENCRYPTION_KEY`；注释 `:535-539` 明说 "env-only storage policy … a key file next to the data sits on the same untrusted filesystem"），非法值**启动即抛**（fail fast，不静默降级）。
- **key 生效时**：会话文件用 AES-256-GCM 加密（`:542-543`，日志 `:547`）。
- **对 CMB 关键的一条**：`:562-569`
  ```
  // Encryption is a hard opt-out: index.db holds the ingest payload in
  // cleartext, so the sidecar refuses to open while BILI_ENCRYPTION_KEY
  // is active and the store stays legacy-only.
  const unifiedBlockedByEncryption = storageMode !== "legacy" && key !== null;
  if (unifiedBlockedByEncryption) log("warn", "[storage] unified sidecar DISABLED ...");
  ```
  → **加密模式下 `index.db`（CMB 唯一数据源）不再写入**。CMB 若此时读它，读到的是**过时快照**（或根本不存在）。

### B3.2 CMB 的自检逻辑（`src/config/config.ts:100-108`）

```ts
if (cfg.source.adapter === "billion-context-readonly" && cfg.source.encryption_compatibility) {
  if (process.env.BILI_ENCRYPTION_KEY) {
    throw new CmbError("ENCRYPTION_MODE_UNSUPPORTED", "...");
  }
}
```

- m-3 修复已生效：`validateConfig` 不再只在 CLI 调用，**引擎构造期强制**（`reconciliation-engine.ts:60`，实测 `security.test.ts:167-173`）。
- 默认值安全：`DEFAULT_CONFIG.source.encryption_compatibility = true`（`:60`）。

### B3.3 能否绕过？（逐条）

| 绕过方式 | 是否可行 | 说明 |
|---|---|---|
| **设 `encryption_compatibility=false`** | ✅ **可行，且这是唯一开关** | 自检整段被跳过，**没有任何替代检查**（不验 `index.db` 的 mtime、不验 `-wal` 是否存在、不验会话数）。结果是：安静地读一个"上游已停写"的库 → 报告 `sessions=0 / evidence_written=0` 或"扫了一遍旧数据" —— **两种都是"成功"退出**。这是任务书点名的"绕过"，**未修复** |
| 用 `mock` 适配器 | ✅ 不受检 | 合理（不读真库）；但也意味着"配置成 mock 却以为在读真库"没有任何护栏 |
| 换进程/换用户跑 CMB | ✅ **误判** | 检测依据是**本进程的环境变量**，不是库的属性。上游（bili/DSH）在 A 进程带 key 跑，CMB 在 B 进程读同一个 `index.db` → CMB 看不到 key → 自检通过 → 读到陈旧明文库。**这是比"改开关"更隐蔽的绕过，且无需敌意** |
| 直接 `new BillionContextSourceAdapter(path)` | ✅ | 适配器本身没有任何加密/mtime 检查（`bc-source.ts:28-31` 只 `readOnly:true`） |

### B3.4 `DRY_RUN` / `DISCOVERY` 模式下是否真的不会写？

> **会写。全部 6 种模式行为完全一致。**（VERIFIED_RUNTIME，`probe4.mjs`）

```
mode=OFF             sink_count=1 evidence_written=1 ledger_blocks=1 report.mode=OFF
mode=DISCOVERY       sink_count=1 evidence_written=1 ledger_blocks=1 report.mode=DISCOVERY
mode=DRY_RUN         sink_count=1 evidence_written=1 ledger_blocks=1 report.mode=DRY_RUN
mode=EVIDENCE_ONLY   sink_count=1 evidence_written=1 ledger_blocks=1 report.mode=EVIDENCE_ONLY
mode=RECONCILIATION  sink_count=1 evidence_written=1 ledger_blocks=1 report.mode=RECONCILIATION
mode=DIAGNOSTIC      sink_count=1 evidence_written=1 ledger_blocks=1 report.mode=DIAGNOSTIC
```

代码依据：`cfg.mode` 在引擎里只出现两次——`engine.ts:77`（写进日志）和 `engine.ts:333`（写进报告）。**没有任何分支读取它**。`DEFAULT_CONFIG.mode = "OFF"` 也照写。

附带的"构造即落盘"：`BridgeLedger` 构造函数无条件 `mkdirSync(dirname(dbPath), {recursive:true})` + 打开/建表（`bridge-ledger.ts:59-63`）——实测**仅构造**就创建了 `...\nested\ledger.db`。默认 `ledger.db_path = "data/test-only/ledger.db"` 是**相对路径**，随 CWD 落到任意目录。

### B3.5 `DEFAULT_CONFIG` 逐字段评估（`config.ts:55-82`）

| 字段 | 默认值 | 评估 | 判定 |
|---|---|---|---|
| `mode` | `"OFF"` | **语义上安全，实现上无效**（见 B3.4） | 🔴 **名不副实** |
| `source.adapter` | `"mock"` | 安全默认（不接真库） | 🟢 |
| `source.index_db_path` | `null` | 安全；配 `bc` 但缺路径会抛（`config.ts:97-99`） | 🟢 |
| `source.encryption_compatibility` | `true` | 安全默认；但**可一键关闭且无替代检查** | 🟡 |
| `evidence_sink.adapter` | `"mock"` | 安全；`memory-core` 直接抛 `SINK_UNSUPPORTED`（`:109-114`），诚实 | 🟢 |
| `evidence_sink.simulate_failure_rate` | `0` | 安全；取值白名单校验（`:115-118`） | 🟢 |
| `evidence_sink.simulate_timeout_ms` | `null` | 未接线（`MockEvidenceSink` 不读它） | 🟡 死配置 |
| `ledger.db_path` | `"data/test-only/ledger.db"` | **相对路径**：随 CWD 落到任意位置；构造即落盘 | 🟡 |
| `batch_size` | `200` | **未接线**（引擎从不读，一次性全量扫） | 🟡 死配置 |
| `concurrency` | `1` | **未接线** | 🟡 死配置 |
| `retry` | `{3, 50ms}` | **未接线** | 🟡 死配置 |
| `hash_algorithm` | `"sha256"` | 校验为唯一值（`:94-96`） | 🟢 |
| `identity_requirements.require_tenant` | `true` | 检查的是"常量非空"，不是"数据归属"（07 报告 §2.2） | 🟡 有检查无效果 |
| `identity_requirements.require_bot` | `true` | 同上 | 🟡 |
| `identity_requirements.require_group` | `false` | 且代码判据 `session.session_type==="group"`（`validator.ts:63`）在真实适配器下恒 `"unknown"` → **该检查永不执行** | 🔴 |
| `identity_requirements.require_speaker` | `false` | **与实现矛盾**：M-5 已改为"非 tool/system 无条件强制"（`validator.ts:67-73`），配置说可选、代码说必填 | 🔴 契约不一致 |
| `source_retention.cache_originals` | `false` | 安全（不缓存原始 blob）；但**该字段从未被读取**（无缓存实现） | 🟡 死配置 |
| `log_level` | `"info"` | 安全（debug 也无正文，见 B1） | 🟢 |

**统计**：19 个叶子字段中，**5 个纯死配置**（`simulate_timeout_ms`、`batch_size`、`concurrency`、`retry`、`cache_originals`）、**3 个语义失效**（`mode`、`require_group`、`require_speaker`）。即"配置面"向运维承诺的控制力，约 **42% 不存在**。

---

## B4 — 删除与失效传播

### B4.1 上游 tombstone / `retired_at` 语义（VERIFIED_SOURCE）

| 事实 | 证据 |
|---|---|
| 行不会物理删除，只打 `retired_at = Date.now()`（"historian sweep"） | `store.ts:930-950` |
| 文本 CAS 字节**保留**（"retired rows still own their bytes"）；**媒体 ref 会被释放** | `store.ts:941-955` |
| `getMessages` 默认过滤 `retired_at IS NULL`，可 `includeRetired` 取全量 | `store.ts:2983-2999` |
| **复活是设计内的**：同 `(msg_id, identity)` 再次出现 → `retired_at = NULL`（rewind/undo 场景） | `store.ts:919-928`（"#4390 rewind-redo: the identity came back — reactivate the tombstone at its original seq"）；另 `:1399-1435` 亦有 NULL 复位 |
| `retired_at` 语义是"**历史中不在当前视图**"，**不是**"用户要求删除" | `store.ts:930-940` 注释 |
| fork 会**复制** `retired_at` 到子会话 | `store.ts:2881-2883` |

→ **上游没有"用户删除"原语**，`retired_at` 同时承载"回退/撤销"和"视图缺席"两种语义。CMB 把它当作"删除/撤销"使用，**语义比上游更窄**。

### B4.2 CMB 的 `revokeScan`（`engine.ts:360-380`）—— "单次采样即撤销"问题

```ts
for (const row of msgs) {
  if (row.revoked_status === 1) continue;            // :365 已撤销不复检
  const m = await this.store.getMessage(s.session_id, row.ledger_seq);  // :366 单次读
  if (m === null) {                                   // :367 一次为 null 即认定"源已不可见"
    const ev = await this.sink.getByIdentity(row.content_hash, row.source_locator);
    if (ev !== null && !ev.revoked) { await this.sink.revoke(ev.evidence_id); ... }  // :370-374
  }
}
```

**确认存在"单次采样即撤销"**，且比"采样"更糟：

| 问题 | 说明 |
|---|---|
| ① **单次 null 即终局** | 没有 N 次确认、没有 `observed_at`、没有 `revoke_reason`、没有宽限期 |
| ② **null 语义混淆** | `getMessage` 返回 null 同时代表：tombstone / 行被物理删除 / 会话消失 / 读失败被适配器吞成 null（Mock 的 `getMessages` 对 retired 返回 []，`mock-source.ts:45-55`；真适配器 SQL 过滤，`bc-source.ts:81`）。CMB **无法区分**"上游说它没了"与"我读不到它" |
| ③ **只覆盖有账的消息** | 必须先有 `message_ledger` 行（只有成功写过 Evidence 才有）。块级撤销没有任何入口 |
| ④ **不复核身份** | 不比对当前行的 `message_id`/`content_hash` 与账本是否一致 → `ledger_seq` 被复用时（07 报告 §2.5）会撤销**错误的 Evidence** 或漏撤 |
| ⑤ **默认关闭** | `revokeScan` 只在 `opts.revokeScan === true` 时执行（`engine.ts:317-319`）；CLI **不传**（`cli/cmb.ts:59` `engine.run()` 无参）→ **v0.1 CLI 从不撤销任何东西** |

### B4.3 Evidence 撤销后能否复活？（上游 retired 变回 visible 时）

> **不能。撤销是单向棘轮（one-way ratchet）。**（VERIFIED_RUNTIME，`probe3.mjs`）

实测序列：

| 步骤 | 结果 |
|---|---|
| ① 首扫（2 条消息可见） | `evidence: [{seq:"0",revoked:false},{seq:"1",revoked:false}]` |
| ② seq0 变 tombstone + `revokeScan:true` | `revoked=1` ✅；`message_ledger: [{seq:0,revoked:1,st:"REVOKED"}]` |
| ③ **上游复活**（retired 清除）+ `revokeScan:true` | `revoked=0`（本次没有新撤销）；**Evidence 仍 `revoked:true`**；`message_ledger` 仍 `revoked_status=1` |
| ④ `REVIVED` 判定 | **`false`** |

机制原因（三处硬堵）：
1. `engine.ts:365` `if (row.revoked_status === 1) continue;` —— 已撤销行**永不复查**；
2. `MockEvidenceSink.revoke` 只写 `revoked:true`（`evidence-sink.ts:71-79`），**`EvidenceSink` 接口里根本没有 unrevoke**（`:8-17`）；
3. 块已 `COMMITTED` 且 `block_version` 未变 → `engine.ts:112-116` 直接 skip → **连重写的机会都没有**；
4. 即使强行重写，`MockEvidenceSink.write` 对"同身份但 `revoked` 不同"抛 `SINK_CONFLICT`（`:51-57`）——**冲突而不是覆盖**（这一点是好的：不静默复活，但也没有合法复活路径）。

**后果**：与上游语义直接冲突。上游 `store.ts:923-927` 明确把"identity 回来"当作**正常事件**并复位 `retired_at`；CMB 侧同一条消息的 Evidence 永久停在 `revoked`。用户 rewind/undo 后，记忆库会**永久少一条**，且没有任何告警。

**反向风险同样存在**：撤销是"删除成功"的强断言，而 CMB 的删除依据只有一次读 —— 一次瞬时读失败即可永久抹掉一条证据。

### B4.4 已确认 MemoryUnit 如何感知来源失效？

> **完全感知不到——没有任何通道。**

| 规范要求（Memory Core） | CMB 的对应实现 | 结论 |
|---|---|---|
| `01-记忆本体与数据模型.md:1011` E01-11：证据被撤销但其记忆仍 `active` → 记忆 MUST 降级 `disputed`/`restricted`（P0-P3 至少告警） | 无。CMB 只调 `sink.revoke(evidence_id)`，不产生任何"受影响记忆"清单 | ❌ 未实现 |
| `02-核心算法规范.md:741` §5.5：来源删除/修改/权限收紧/证据撤销 ⇒ 依赖 Observation MUST 置 `needs_recompute`/`invalidated`/`restricted` + `invalidated_reason` | 无。`revoked` 只是 Evidence 的一个布尔位 | ❌ 未实现 |
| `01`：`memory_evidence_links` 的删除 MUST 随证据撤销传播 | 无。CMB 不持有任何 link | ❌ 未实现 |
| `04-权限与治理规范.md:82`：tombstone MUST 阻止已删内容经备份恢复/索引重建复活 | CMB 的 `revoked` 是单向的（B4.3），方向上满足"阻止复活"，但它拦的是**上游合法撤销的回滚**，不是恶意恢复 | ⚠️ 语义错位 |

- 结构原因：`MemoryCoreEvidenceAdapter` **全方法抛 `SINK_UNSUPPORTED`**（`evidence-sink.ts:92-110`），`config.ts:109-114` 在配 `memory-core` 时直接拒绝启动。→ v0.1 **不存在真实 Memory Core 写入**，也就不存在传播。
- 但即便 adapter 实现了，"撤销传播"也**无处触发**：`EvidenceSink` 接口（`evidence-sink.ts:8-17`）只有 `write/getByIdentity/count/revoke/injectFailure`，**没有事件、没有 outbox、没有 `onRevoked` 钩子**。`revoke` 返回 `boolean`，谁都不消费。
- 另：`SummaryArtifactStore.invalidateBySession`（`summary-artifact.ts:70-84`）是**唯一**实现了"失效标记"的地方，但它①无调用者（07 报告：M09 是孤岛）②失效方式是把 `:INVALIDATED` 无限追加到 `generator_version` 字符串上（幂等性缺陷，历史 m-4，本轮未修）。

---

## 5. 风险清单（本报告范围内，按严重性）

| # | 风险 | 级别 | 可利用性 | 证据 |
|---|---|---|---|---|
| **S-1** | **系统提示词被写入记忆**（`role=system` 未被拦截，且是唯一豁免 speaker 校验的类别） | **CRITICAL** | 当前即触发（真实路径下**它是唯一能进的类别**） | B2.3；probe1/P6 |
| **S-2** | **所有 `mode` 都不阻断写入**（OFF/DRY_RUN/DISCOVERY 照写 Evidence + Ledger） | **HIGH** | 当前即触发 | B3.4；probe4 |
| **S-3** | **未知消息类型静默放行**（黑名单而非白名单，无 default-deny） | **HIGH** | 当前即可（取决于上游出现新类型） | B2.4；probe1 |
| **S-4** | **加密自检只查本进程 env**：换进程即可绕过；`encryption_compatibility=false` 一键关闭且无替代检查 → 静默读陈旧明文库 | **HIGH** | 无需敌意即可发生 | B3.1-B3.3 |
| **S-5** | **撤销单向不可逆**：上游合法的 rewind/undo 复活无法表达 → 永久丢记忆；反向"单次采样即撤销"可能误删 | **HIGH** | 当前即触发（只要开 revokeScan） | B4.2/B4.3；probe3 |
| **S-6** | `developer` 角色消息成为 Evidence | **MEDIUM** | 上游 codex 车道即产生 | B2.3；probe1 |
| **S-7** | 多模态/附件静默丢弃；纯附件消息被误判 `CORRUPTED` | **MEDIUM** | 当前即触发（图文场景） | B2.3 |
| **S-8** | `data.message` 透传 `err.message`（5 处）+ `failure_reason` 持久化任意异常文本 | **MEDIUM**（潜伏） | 接真实 sink 后 | B1.1/B1.2 |
| **S-9** | 配置面 42% 死字段/语义失效（含 `require_group`/`require_speaker` 与实现矛盾） | **MEDIUM** | 运维误判 | B3.5 |
| **S-10** | `Ledger` 构造即落盘 + 默认相对路径 | **LOW** | — | B3.4 |
| **S-11** | `StructuredLogger.events[]` 无上限 | **LOW**（内存） | 长跑 | B1.7 |

**未发现的问题（应明确记录为"通过"）**：日志/报告/Ledger 无正文泄露（实测）；错误消息只含码/locator/hash 前缀；`reasoning` 两道拦截；`memory-core` sink 诚实拒绝而非假装可用；上游连接只读（`bc-source.ts:30`，上一轮已运行时验证 readOnly 拒绝 DDL）。

---

## 6. 实测附录

探针位置：`%TEMP%\cmb-e-probe\{probe1,probe2,probe3,probe4}.mjs`（**只写系统 TEMP，未触碰项目目录**）。
运行方式：`node probeN.mjs`（Node v24.21.0，原生 type-stripping 直接 import 项目 `.ts`）。

| 探针 | 覆盖 | 关键输出 |
|---|---|---|
| `probe1` | 6 类消息 × 2 种 speaker 口径 × 4 个泄露面 | `NO_SECRET_LEAK_FOUND`；Evidence 含 system/developer/unknown 三类（`seq 1/2/6`）；hint 覆盖 bound（`speaker=U-mock-0`） |
| `probe2` | 真实口径、sessionTypeMap 写反、M-1 交叉校验、unknown session_type | P6：真实口径下**只有 system 消息**进 Evidence；P7a/P7b：visibility/group 静默互换；P8a：M-1 拒绝 ✅；P8b：DeploymentConstant 忽略源自称值 ✅；P10：unknown → 块 MISSING ✅ |
| `probe3` | 撤销/复活、session 重用 | 撤销成功（`revoked=1`）；复活后 `REVIVED: false`；内容改写被 `block_version` skip（0 新 Evidence） |
| `probe4` | 6 种 mode 的写入行为；Ledger 构造副作用 | 6 种模式 `sink_count=1`；`ledger file created by constructor alone? true` |

---

## 7. 证据等级汇总

| 级别 | 条目 |
|---|---|
| **VERIFIED_SOURCE** | 全部日志（10 处）与 `CmbError`（26 处）调用点（grep）；`observability.ts` 全文；`bridge-ledger.ts:67-108`；`config.ts` 全文；`persist.ts:535-569`；`store.ts:919-956,1399-1435,2881-2885,2983-2999`；`schema.ts:141-155,267-272`；`kernel/src/types.ts:3-6`；`kernel/src/message-kind.ts:11-15`；`kernel/src/media-refs.ts:139-140`；`ingest.ts:193-195,340-341`；`compat-roles.ts:86`；`integrity-validator.ts:60-105`；`evidence-sink.ts` 全文；`summary-artifact.ts:70-84`；Memory Core `04:82,152-196`、`01:1011`、`02:741` |
| **STATIC_INFERENCE** | "换进程读陈旧库"的具体触发条件；接真实 sink 后 `err.message` 的泄露概率；多模态消息在 CMB 侧的确切丢失形态（需真实图文会话验证） |
| **VERIFIED_RUNTIME**（本轮实测） | B1.7 泄露扫描（0 命中）；B2.3 system/developer/unknown 三类进 Evidence；B2.3 reasoning 被拦；B3.4 六种 mode 全部写入；B4.3 撤销不可逆；B3.4 Ledger 构造即落盘 |
| **UNVERIFIED / 不可读** | DSH `app.asar`（213 MB，仅 `node_modules` 解包）；真实 Memory Core 写入（无实现）；真实图文/附件会话；`BILI_ENCRYPTION_KEY` 下的 bili 实机行为（仅静态阅读 `persist.ts`）；WAL 只读并发 |
| **UNSUPPORTED** | QQ Bridge 无 git 历史；Memory Core 无实现代码（仅设计规范） |

---

*本报告为只读审查产出；所有运行时验证均在 `%TEMP%\cmb-e-probe\` 内进行，未修改被审对象的任何文件。*
