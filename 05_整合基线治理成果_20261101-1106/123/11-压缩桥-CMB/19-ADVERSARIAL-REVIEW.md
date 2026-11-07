# 19 — 反方审查（Agent G · 独立质疑）

> 角色：独立反方审查者。只读审查，未修改被审对象的任何文件。
> 阶段：**阶段一（独立判断，未阅读本轮其他 Agent 的报告）**。阶段二在收到同行报告路径后追加。
> 审查对象：
> - `E:\[PROJECT]\CMB-v0.1-prototype`（src/ 17 文件、tests/ 9 文件 + 2 fixture、docs/）
> - `E:\[PROJECT]\refs\billion-context` @ `0aace2f203b2da9065fd53c63f4be0c0ab03c90f`（storage/*、ingest、canonical、identity、schema、kernel/src/types.ts、kernel/src/compress.ts）
> - `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design`（`contracts/schemas/evidence.schema.json`、`DD-AUTHORITY-V2.md`、`13-Evidence与EmotionContext.md`）
> - `E:\[PROJECT]\第三轮专项技术审查报告.md`（作为被引证据源复核）

## 方法与限制（先声明，避免把静态推断冒充运行时事实）

1. **我没有执行测试**。`node tests/runner.ts` 会写 `data/test-only/*.db`（`perf-run.ts:160-161`、`cmb.ts:53-56`），属"修改文件"，与只读授权冲突。所有测试相关结论均为**静态读取**（逐文件数 `test(` 注册点、逐条读断言体）。
2. **本机没有真实上游 `index.db`**（只搜到 `C:\Users\[USER]\.pnpm\store\v11\index.db`，是 pnpm 的）。因此凡涉及"真实库上会怎样"的判断，最高只能到 `VERIFIED_STATIC + INFERRED`，**不得**标 `VERIFIED_RUNTIME`。
3. 我复核了 CMB 引用的上游行号：`ingest.ts:425`（supersedes 硬编码 null）、`store.ts:1031-1038`（batchTs）、`store.ts:1077-1087`（块物理删除）、`store.ts:1105`（payload 解析静默吞）、`store.ts:3013`（`?? ""`）——**全部对得上**。CMB 的静态引用纪律总体是好的，这一点我先认。
4. 区分标记：**【确认缺陷】**＝代码可证、路径可达；**【怀疑】**＝逻辑上成立但缺一个运行时前提（真实库形态、媒体消息是否存在、漂移频率）。

---

## 0. 结论摘要

**最站不住脚的 3 条结论：**

1. **"原文恢复 VERIFIED_RUNTIME / 可完整还原"** —— 等级标错，且在最关键的一个字段上是**结构性不可得**：上游 `messages` 表根本没有"发言者"列（`schema.ts:141-155`），适配器把 `speaker_id` 恒置 `null`（`bc-source.ts:211`），而 Validator 对 user/assistant 无条件强制 speaker（`integrity-validator.ts:69-73`）。**真实源下每一条普通消息都会被拒，一条 Evidence 都产不出来**；78 条测试全绿是因为夹具自己把 `speaker_id` 造好了（`demo-fixture.ts:53`）。这不是"未接入"的边界问题，是**已写死的必然失败**。
2. **"三层幂等 / 三连扫零重复"** —— 幂等的实际承担者是一个进程内 `Map`（`evidence-sink.ts:20`）。换进程即失效；`MemoryCoreEvidenceAdapter` 全部抛 `UNSUPPORTED`（`evidence-sink.ts:92-111`）。所谓跨重启幂等实际只有"台账说 COMMITTED 就跳过"（`engine.ts:112-116`），**它不验证 Evidence 是否存在**——demo 用持久台账文件（`cmb.ts:55`）而 sink 每次进程新建，第二次运行会得到"已桥接 N 块 / 0 条新证据 / sink 为空"的自相矛盾结果。
3. **"B-1 契约冲突未裁决所以不能接 Memory Core"** —— 权威序其实已经判了：`DD-AUTHORITY-V2.md:5` 自称"所有文档、**契约**、状态机、测试规格的唯一枚举/字段/接口定义来源"，`13-Evidence与EmotionContext.md:1028` 明写"与 V2 §3.2 冲突时以 V2 为准"。按 V2，`source_type="chat"` 是**对的**，过时的是 schema 文件。但更糟的是：**CMB 的 DTO 按 V2 也不合规**——`content_hash` 必须是 `sha256(original_text)`（`13-…md:246,255` MUST），CMB 用的是整条 blob 的哈希，而且**有测试把这个违规锁定成"正确"**（`validator.test.ts:44-52`）。所以 B-1 不是"等裁决"，是"CMB 侧设计选错了"。

**反例计数：22 条**（确认缺陷 15 条 / 怀疑未证实 7 条），见 §2。

---

## 1. 逐条质疑

### 质疑 1：「原始消息可完整恢复」真的成立吗？

| 质疑 | 主流结论 | 反方证据 | 我的判断 | 证据等级 |
|---|---|---|---|---|
| 1a 真实源下能否产出 Evidence | 交付报告 §三："原文恢复（含 tool 身份字段）VERIFIED_RUNTIME" | `schema.ts:141-155` messages 表无 speaker/actor/user 列；`bc-source.ts:211` `speaker_id: null`；`integrity-validator.ts:69-73` user/assistant 无 speaker 即 `SPEAKER_UNRESOLVED`；`engine.ts:204-208` 该诊断归入 `missing` | **致命**。`HostIdentityProvider` 接口只收 `(session, speakerHint)`（`host-identity.ts:27`），拿不到 seq/消息内容，**任何实现都无法按消息归属发言者**。真实链路必然 0 证据。 | 确认缺陷（静态可证；运行时后果需真库确认，但逻辑闭合） |
| 1b `SPEAKER_UNRESOLVED` 归入哪个状态 | 报告称"缺失即拒绝" | `engine.ts:202-208`：`CONTENT_MISSING/HASH_MISMATCH` → corrupted，`IDENTITY_MISSING/VISIBILITY/SPEAKER` → missing；`engine.ts:228-234` 全挂时状态取 `CORRUPTED`/`MISSING`/`FAILED` | 分类本身没错，但它让**永久性身份缺失（1a）伪装成"暂时找不到"**，于是每轮重扫。 | 确认缺陷 |
| 1c "blob 在但正文缺失" | 报告称已堵住 `?? ""` | `bc-source.ts:97` 显式 `return null`（不再空串）；`integrity-validator.ts:80-81` 空/缺 → `CONTENT_MISSING`；`bc-source.ts:139-152` payload 解析失败 → 抛 `BLOCK_PAYLOAD_CORRUPT` | **这条我无法反驳：`getBlob` 的 `?? ""` 路径确实被堵住了。** 但同一句 `?? ""` 在 `bc-source.ts:135`（`String(r.payload_json ?? "")`）仍存在，且后续走向是 UNSUPPORTED（见 4d）。 | 主流结论成立（这一条不反驳） |
| 1d "blob 不在但消息行在" | 未讨论 | `store.ts:790-808` `releaseContent` 在 `refcount<=0` 时 **DELETE 物理删除** content 行；`store.ts:806`。上游自身读会话用单事务快照（`store.ts:3070-3081` `db.transaction`），CMB **没有**：`bc-source.ts:75-86` 读 message+content，到 `engine.ts:195` 再单独 `getBlob` | 两次读不是一个快照 ⇒ **TOCTOU**：中间若 content 行被回收，`getBlob→null` 而 `original_text` 非空 ⇒ `integrity-validator.ts:90` 走"HASH_UNVERIFIABLE"拒收，块落 CORRUPTED。**上游自己都开事务读，CMB 没开**，这是可指名的设计缺口。 | 确认缺陷（窗口窄）；后果等级 怀疑 |
| 1e 老版本序列化 / 字段变过 | 未讨论 | `ingest.ts:155-175` `serializeForStore` 会 `delete ts`；`identity.ts:40-77` `splitMediaContent` 会把 `imageBase64` 与 ≥2048B 的 data-URL **替换成 `{"$media":hash}`** 再哈希。`bc-source.ts:183-196` 只认 `core.text`（string），否则 `null` | **两条反例**：① 真实 `content.bytes` 对多模态消息**不是** `JSON.stringify(CoreMessage)` 的逐字副本（`bc-source.ts:181` 注释失准），少了 `ts`、被改写了媒体槽——要"完整还原"必须再跳一次 CAS 取 `$media`，CMB 从不跳；② 纯图片/纯工具调用消息 `text` 缺失 → `originalText=null` → `CONTENT_MISSING` → 该消息**永久**丢弃。好消息：JSON.parse 成功后字段缺失会变 null 并**被拒**，不是静默 undefined——这一点 CMB 做对了。 | 确认缺陷（媒体形态）/ 静态证实 |
| 1f 部分丢失会被发现吗 | 报告称"如实记录 failed" | `engine.ts:228` 只在 `validated.length===0` 时定性；只要块里有 ≥1 条过校验，就走到 `engine.ts:281-282`，`writeFailures===0` 即 `COMMITTED` | **块级 COMMITTED ≠ 覆盖完整**。此后 `engine.ts:112-116` 因 `block_version` 未变而永久跳过重扫 ⇒ **静默永久丢消息**，只在当轮 `failure_reasons` 留痕。这是"可完整恢复"最实质的反例。 | 确认缺陷 |
| 1g 有效性边界 | 报告 §五已声明"真实 index.db 端到端 UNVERIFIED" | `cmb.ts:44-45` CLI **只装配 MockSourceStore**，`bc-source.ts` 在 CLI 中不可达，且**零测试覆盖**（全 tests 目录仅 2 处出现 adapter 字符串，都是 config 校验：`security.test.ts:158`、`domain.test.ts:56`） | 声明是诚实的，但"原文恢复"被放在 §三"已验证能力"里与 VERIFIED_RUNTIME 并列，属**等级误标**。 | 确认（等级错标） |

**小结**：`?? ""` 这条堵住了（不反驳）；但"可完整恢复"仍被三个独立路径打穿：无 speaker 列、媒体分片不还原、部分丢失后 COMMITTED 冻结。

---

### 质疑 2：「幂等」这个词被用对了吗？

| 质疑 | 主流结论 | 反方证据 | 我的判断 | 证据等级 |
|---|---|---|---|---|
| 2a 幂等靠什么 | ADR-003 三层幂等 + "三连扫零重复"（交付报告 §三） | `evidence-sink.ts:19-20` `MockEvidenceSink` = `new Map()`；`evidence-sink.ts:33-61` 去重键 = `${content_hash}|${source_locator}` | 三层里**只有账本层是真持久的**（`bridge-ledger.ts:61` 真 sqlite）。Evidence 层幂等 = "被测组件自己说自己幂等"。 | 确认缺陷 |
| 2b 进程重启后 | "崩溃残留可恢复" | `engine.ts:112-116`：`status==="COMMITTED" && block_version 相同` → `continue`，**不查 Sink**；`cmb.ts:55` demo 用**持久** `data/test-only/demo-ledger.db`（注释却写"一次性临时"，与 `data/` 里实际存在的 `demo-ledger.db` 矛盾）；`cmb.ts:52` 每次进程新建内存 Sink | **可直接反驳**：第二次 `demo` 会输出 `skipped=N, evidence_written=0`，而 `sink.count()===0`。"已桥接"是台账的信仰，不是事实。这就是"幂等"实际证明的全部内容。 | 确认缺陷（代码路径确定，未运行） |
| 2c Sink 换成真实 Memory Core 后键还成立吗 | 三层身份"成立" | `evidence-sink.ts:92-111` 真实适配器**所有方法抛 `SINK_UNSUPPORTED`**；`config.ts:109-114` 直接拒绝装配 | 键是否成立**无法验证**，因为真实 Sink 不存在。在 `content_hash` 语义被权威文档判为违规（见 5c）的前提下，`(content_hash, locator)` 作为跨库幂等键本身就悬空。 | 怀疑（未证实，无真实 Sink） |
| 2d session 删除重建后撞键 | 未讨论 | `evidence-adapter.ts:53` `evidence_id = ${session}:${seq}`；`evidence-sink.ts:50-57` 键命中且文本相同 → 返回 `duplicate`（**不产生新 Evidence、不更新 provenance**）；若旧证据已 `revoked` 且新事件 `revoked=false` → **抛 `SINK_CONFLICT`**（`evidence-sink.ts:52-56`）→ `engine.ts:273-279` 记 writeFailure → `engine.ts:287-290` 块 FAILED → 每次扫描重试、永远冲突 | **确认有撞键，且是双向坏**：撞上未撤销的 → 新事实被吞成"重复"；撞上已撤销的 → **永久失败环**。上游 `msg_id` 是内容派生（`identity.ts:5-21` 注释"A real edit changes the content-derived id"），重建同名会话会重出同一 id+hash，撞键不是理论。 | 确认缺陷（路径）/ 触发频率 怀疑 |
| 2e 同进程内多块重叠 | "嵌套去重" | `engine.ts:254` 写后立刻 `getByIdentity` 回读验证；`engine.ts:244-279` 逐条 try | 同进程内有效。但"写后验证"对真 Sink 无意义（异步/最终一致时必然 `VERIFY_FAIL` → PARTIAL → 重写风暴）。该验证**只对内存 Map 成立**。 | 确认（等级 MOCK_ONLY 被写成通用保证） |

---

### 质疑 3：「78 条测试全绿」说明了什么？

先纠正一个我原本要打的靶子：**78 这个数字是对的**。`phase6-regression.test.ts:33` 用 `for (const residue of [PARTIAL,READY,RESOLVING,VALIDATING])` 注册了 4 条，静态 8 处 + 循环 3 处 = 11，合计 35+9+11+11+12 = **78**。我差点误报，记录在此以免后来者重复。

| 质疑 | 主流结论 | 反方证据 | 我的判断 | 证据等级 |
|---|---|---|---|---|
| 3a 有多少"Mock 验证 Mock" | TEST-REPORT 标称 `模拟` 但交付报告把能力列进 VERIFIED_RUNTIME | 数据面 100% 走 `MockSourceStore`（`perf-run.ts:5`、`phase6:8`、`cmb.ts:13`）；**`bc-source.ts` 无任何测试**；Sink 100% `MockEvidenceSink`；Identity 100% `MockHostIdentityProvider` | 分母 78 里，**被验证的只有"内存对象图上的编排逻辑"**。唯一的真实数据组件（适配器）零覆盖——而 1a/1e 的缺陷恰好全在适配器与真数据的接缝上。 | 确认缺陷 |
| 3b "断言一个没实现的行为" | — | ① `security.test.ts` SEC-7 造 `contentType:"system"` 断言"system prompt 不成为证据"；上游 `MessageContentType = "text"|"tool-call"|"tool-result"|"reasoning"`（`kernel/src/types.ts:5-6`）**根本没有 `system` 这个内容类型**；真实的系统提示是 `role="system"` + `kind="text"`，而 `integrity-validator.ts:95` 判的是 `content_type`，**永远不会命中** ⇒ 真实源下**系统提示会进 Evidence**。② `integrity-validator.ts:70` 判 `"toolCall"/"toolResult"`（驼峰），上游是 `"tool-call"/"tool-result"`（连字符），豁免分支对真实数据永不命中。 | 这不是"断言未实现的行为"，是更坏的：**测试用上游不存在的枚举值伪造了一个安全保证**，并写进交付报告"system 排除 VERIFIED_RUNTIME"。属安全口径误标。 | 确认缺陷 |
| 3c 自证循环 | — | `validator.test.ts:44-52` 标题"hash over full blob, not just original_text (CAS semantics)"，断言 `sha256("hello")` **必须不匹配**。而权威文档 `13-…md:246,255` 写死 "content_hash **MUST** 为 original_text 的 sha256 … MUST NOT 用规范化后文本"。 | **典型自证循环**：把实现对上游 CAS 的观察固化成规范，再用测试锁死，于是"与 Memory Core 契约不符"这件事在测试维度**不可见**。 | 确认缺陷（最重要的一条测试问题） |
| 3d B-1 回归测试的覆盖缺口 | "补 12 条回归"（交付报告 §四）/ TEST-REPORT §3.7 "11 通过" | `phase6-regression.test.ts:33` 只遍历 `PARTIAL/READY/RESOLVING/VALIDATING` 四种残留；**没有** `UNSUPPORTED`、**没有** `COMMITTED+漂移`——恰恰是状态机里唯二会崩的两个（见 4b/4c）。`fi.test.ts` FI-7 造出 UNSUPPORTED 块但**只跑一次 run()**，不断言第二次扫描。 | 测试**从修复的happy path反推**，不是从状态机反推（`domain.ts:157-170` 12 个状态、有效边 21 条，回归只覆盖 4 态）。B-1 是"按症状修"，测试也按症状测。 | 确认缺陷 |
| 3e 性能数字 | 交付报告 §二："10⁶ 消息 → 50 万 Evidence，373–847 msgs/s" | `perf-run.ts:5` `import { MockSourceStore }`；夹具字段 `contentType: "toolResult"`（`perf-run.ts:57`）同样是不存在的枚举 | 数字**只度量内存 Map 遍历**，与 sqlite 读、JSON.parse、sha256、WAL 并发全无关。而 `bc-source.ts:145-148` 在 `start/end=-1→null`（N-4 哨兵）时会 `getMessages(0, MAX_SAFE_INTEGER)` **整会话扫描**，且每个 blob 都 parse——真实形态下这是 O(块×会话)。 | 确认（口径问题）/ 真实性能 怀疑 |

---

### 质疑 4：B-1 修复是不是只把问题挪了位置？

**先把 lead 点名要的那个问题钉死**（`reconciliation-engine.ts:128-143` + `domain.ts:157-170`）：

```
if (ledgerRow !== null && status !== "DISCOVERED") {
  if (status !== "FAILED" && status !== "MISSING" && status !== "CORRUPTED") {
    status = transition(status, "FAILED", ...);   // ← 这里
  }
  status = transition(status, "DISCOVERED", ...); // ← 然后这里
}
```

| 查询 | 结果 | 依据 |
|---|---|---|
| `MISSING → DISCOVERED` 合法吗 | **合法** | `domain.ts:164` `MISSING: ["DISCOVERED","FAILED","REVOKED"]` |
| `CORRUPTED → DISCOVERED` 合法吗 | **合法** | `domain.ts:165` `CORRUPTED: ["DISCOVERED","FAILED"]` |
| `FAILED → DISCOVERED` 合法吗 | **合法** | `domain.ts:167` `FAILED: ["DISCOVERED"]` |
| 排除 `MISSING/CORRUPTED` 会不会导致它们无法前进 | **不会**。这两个状态本身就允许直达 DISCOVERED，排除它们是对的（若先转 FAILED 反而要多一跳） | 同上 |
| `UNSUPPORTED → FAILED` 合法吗 | **不合法** | `domain.ts:166` `UNSUPPORTED: ["DISCOVERED"]` |
| `COMMITTED → FAILED` 合法吗 | **不合法** | `domain.ts:162` `COMMITTED: ["REVOKED","SUPERSEDED"]` |

| 质疑 | 主流结论 | 反方证据 | 我的判断 | 证据等级 |
|---|---|---|---|---|
| 4a B-1 修复本身 | "单块瞬态故障不再击穿整轮 run()" | `engine.ts:129-143` 复位段已在 try 内 | **这条成立，不反驳。** 击穿整轮的问题确实修了。 | 主流结论成立 |
| 4b `UNSUPPORTED` 二次扫描 | 未讨论 | `engine.ts:156-160` 把块落为 `UNSUPPORTED`；下次扫描 `engine.ts:130` 因不在排除名单先转 FAILED ⇒ `transition("UNSUPPORTED","FAILED")` 抛 `BridgeStateError`（`errors.ts:30-34`）⇒ `engine.ts:134-143` catch，`STATE_RESET_FAIL++`、`counts.failed++`、`continue`。**且不落账**（无 persist），台账仍显示 UNSUPPORTED、`failure_reason` 不更新 | **确认：UNSUPPORTED 块永远卡死，且每轮污染失败计数。** 也解释了为什么 `recoveryOnly`（`engine.ts:95-100`）救不了它——白名单里没有 UNSUPPORTED。 | 确认缺陷（代码路径确定）；真实源可达性 怀疑（见 4d） |
| 4c `COMMITTED` + payload 漂移 | "bridge_key 含 covered_hash，漂移会自然产生新 key 走全流程"（`engine.ts:111` 注释） | **注释与代码相反**：`bc-source.ts:168` `block_version = sha256(payload_json)`；`bridge_key = sha256(session|block|covered_hash)`（`engine.ts:106`），`covered_hash` 只覆盖 `{effectiveMessageIds, directMessageIds}`（`ingest.ts:406,428`）。而 `payload_json` 是整个块（`ingest.ts:429`），含 `summary/survivedCount/compressedTokens/active/durationMs/runId`（`kernel/src/types.ts:68-110`）。上游会**原地改块**：`types.ts:89-95` refold-in-place "compress() updates such a block IN PLACE (same id, replaced summary)"；`compress.ts:1730-1743` applyRefolds；`active` 翻转同理 | **确认：覆盖集不变、payload 变 = 常态**（refold/摘要重写/计数更新/失活）。此时 `engine.ts:112` 的跳过条件不成立 → 进入复位段 → `COMMITTED→FAILED` 非法 → `STATE_RESET_FAIL` → **块从此不再被处理，也不会被报告为"漏"**。B-1 的修复在这里把"整轮崩溃"换成了"单块静默死亡"。 | 确认缺陷（代码+上游语义双证）；触发频率 高（怀疑上限） |
| 4d 无限重试 / 性能退化 | 未讨论 | `config.ts:13-16,72` 定义 `retry.max_attempts=3, backoff_ms=50`，**全仓无任何读取点**（grep 仅命中 config.ts 自身）；`engine.ts` 无限循环无计数；`engine.ts:112` 只跳过 COMMITTED+同版本 | **没有上限、没有退避、没有断路器**。永久失败块每轮全量重算（解析所有 blob + 每条 sha256），失败块集合单调不减 ⇒ 单轮成本随历史失败线性增长。配置里那个 `retry` 是装饰品，比没有更糟（读者会以为有限流）。 | 确认缺陷 |
| 4e `STATE_RESET_FAIL` 之后块在哪 | 未讨论 | `engine.ts:134-143`：只 `log.error` + 计数，**没有 `persistBlockRow`**（对比 `engine.ts:291,304,159,234` 都落账） | 块停在**旧状态**（UNSUPPORTED / COMMITTED），台账看不出"它已经死了"，`failure_reason` 为空、`last_checked_at` 不刷新。**运维面完全不可观测**，只能靠当轮日志。下次扫描必然复现同一错误（状态没变），永不可恢复。 | 确认缺陷 |
| 4f `direct=[]` 的块 | "direct 优先，缺失则 effective 恢复"（`source-resolver.ts:7-8`） | `source-resolver.ts:62-63`：`direct_message_ids !== null` 就**直接用**，空数组也算"有" ⇒ `exclusiveIds=[]` ⇒ 无消息 ⇒ `engine.ts:228-234` 落 `FAILED`（不是 MISSING/UNSUPPORTED）；上游 `covered_hash`/`payload_json` 稳定不变 ⇒ 每轮重复同样失败。而上游**允许** `directMessageIds: []`：`compress.ts:1719-1722` 的活锁保护只拦 `!isBlockBoundary`，块边界合并（bN..bM）会以 `filteredIds=[]`、`consumedBlockIds>0` 创建块（`compress.ts:1813-1821`） | **可疑且后果重**：若 tier-2/3 合并块真是 `direct=[] + effective 非空`，则这些块覆盖的消息在 CMB 侧永远拿不到 Evidence，且块永久 FAILED。我**没能证实**真实库里这类块的占比（无真实库）——这是 4 条里唯一标"怀疑"的。 | 怀疑（逻辑可达，缺真实数据佐证） |
| 4g 报告指标 | `counts.pending` | `engine.ts:322-325` 统计 DISCOVERED/RESOLVING/VALIDATING/READY；但 `persistBlockRow` 只在终态/失败时调用（`159,234,291,304`），中间态**从不落账** | `pending` 恒为 0，指标失真（无害，但说明该字段没有实际观测意义）。 | 确认缺陷（轻微） |

---

### 质疑 5：Evidence 契约冲突（B-1）是谁的责任？

| 质疑 | 主流结论 | 反方证据 | 我的判断 | 证据等级 |
|---|---|---|---|---|
| 5a 谁权威 | "冲突未裁决"（交付报告 §五/§六，`domain.ts:110-115`） | `DD-AUTHORITY-V2.md:3` 状态 **APPROVED**；`:5` "所有文档（00–15）、**契约**、状态机、测试规格的**唯一**枚举/字段/接口定义来源"；`13-Evidence与EmotionContext.md:1028` "与 V2 §3.2 冲突时以 V2 为准"；`:237` `source_type` = 5 值 `chat/file/import/tool/system` | **权威序已经明确：V2 §3.2 > schema 文件。** 所以 `source_type="chat"`（`domain.ts:119`）是**对的**，`evidence.schema.json:13` 的 `"conversation"` 是**过时 v1 残留**。B-1 的"未裁决"描述把一件已决的事重新打开，掩盖了真正的问题（5c）。 | 确认（文档序明确） |
| 5b 那 schema 是 bug 吗 | — | `evidence.schema.json:8` 要求 `recorded_at`（V2 §3.2 无此字段）；`:13` 枚举无 `chat`；`:7` `additionalProperties:false`（V2 §3.2 无此约束） | **是，是 Memory-Core 侧的 bug**：V2 APPROVED 后契约文件未再生。责任划分：**Memory-Core 30%（陈旧契约）/ CMB 70%（见 5c）**。 | 确认 |
| 5c 换个权威就合规了吗 | 暗示"裁决后即可对齐" | 按 V2 §3.2 逐字对 CMB 的 `EvidenceDTO`（`domain.ts:116-134`）：<br>① `content_hash` **MUST** 是 `original_text` 的 sha256（`13-…md:246,255`）——CMB 是整 blob 哈希（`integrity-validator.ts:83`、`bc-source.ts:168`）→ **违规**；<br>② `evidence_id` **MUST** uuid（`13-…md:236`）——CMB 是 `${session}:${seq}`（`evidence-adapter.ts:53`）→ **违规**；<br>③ `scope` 必填（`13-…md:245`）——DTO 无此字段 → **缺**；<br>④ `speaker_id` 必填非空（`13-…md:240`）——DTO 可空（`domain.ts:124`）→ **违规**；<br>⑤ 多出 `tenant_id/bot_id/group_id/produced_by_run/produced_by_blocks`（V2 未列，v1 schema `additionalProperties:false` 直接拒） | **决定性反例：无论哪个文档赢，CMB 的 DTO 都不合规。** 所以 B-1 的真实性质不是"外部裁决阻塞"，而是"CMB 侧字段语义选错 + 缺字段"，可以在今天改而不必等任何人。CMB 把 ①（哈希语义）解释成"上游 CAS 事实"（`integrity-validator.ts:32-36` 注释），这恰恰是**把上游实现当契约**的越界。 | 确认缺陷（最重要的一条） |
| 5d 有没有第三份权威 | 只提两份 | `13-…md:232` 标题即"**V2 §3.2 逐字段，逐字**"；`DD-AUTHORITY-V2.md:5` 覆盖"契约" | 三份文件实际同源，不存在"表决歧义"，只存在"未同步"。 | 确认 |

---

### 质疑 6：时间戳恒为 null —— 保守，还是放弃治疗？

| 质疑 | 主流结论 | 反方证据 | 我的判断 | 证据等级 |
|---|---|---|---|---|
| 6a 上游 ts 真的二义吗 | `domain.ts:70-74`、ADR-003："首见戳 或 批采样时刻，无标记列可区分" | `store.ts:1029-1038`：`batchTs=Date.now()` 每批一次；`typeof m.ts === "number" ? m.ts : batchTs`。`kernel/src/types.ts:43-54`：`ts` 是"stamped **ONCE at first sight** and never changed"。 | **主流结论方向对，措辞过头**：`m.ts` 存在时就是首见戳（语义确定）；只有缺 `m.ts` 的新行才落批采样时刻（同批同一秒，是首见时刻的"近似"，不是随机噪声）。所以不是"无法区分"，而是"**部分确定、部分 ±1 批**"。判 null 是**过度保守但可辩护**。 | 主流基本成立 / 措辞 反驳 |
| 6b 还有别的证据来源吗 | 未讨论 | ① `blocks.created_at`（`ingest.ts:427`，块生成时刻）是覆盖消息的**时间上界**；② `sessions.created_at` 是**下界**（`schema.ts:119`）；③ `messages.seq` 严格单调（`schema.ts:153` PK）给出全序；④ `<acp>` 标签在**线上**渲染 `time="yyyy-mm-dd hh:mm:ss"`（`kernel/src/types.ts:43-46`），但**入库时被删除**（`ingest.ts:155-160`） | 存在可用的**区间包络**（[session.created_at, block.created_at]）与全序，足以支撑"同会话内先后/同批次"，不足以支撑跨会话时钟对齐。CMB 一条都没用——`blocks.created_at` 甚至没进 `EvidenceDTO`。 | 确认（可用信息被丢弃） |
| 6c 代价是什么 | "时间戳可空（D-7 待用户裁决）" | `evidence-adapter.ts:60` 硬编码 `timestamp: null`；`SourceMessage.timestamp` 已被保留（`domain.ts:74`、`bc-source.ts:208`）却**在适配器边界丢弃**；`13-…md:253` "timestamp 未知 MUST 为 null，MUST NOT 填推测时间" | **对下游 M03/M04 的后果被严重低估**：`13-…md` §9 的情绪衰减/ΔE、Event 的 `time_start`、画像时效都需要时间轴；全 null 后只能退化为"会话内序号"。**且不可逆**：Evidence 一旦写成 null，日后想补必须**重扫全部上游并重建 Evidence 库**。交付报告把 D-7 列为"待裁决"，实际是"已造成不可逆数据损失"。 | 确认缺陷（等级：影响评估错误） |
| 6d 我是否认为应该改 | — | 反方也要公平：v1 schema `additionalProperties:false` 禁止加 `time_source` 之类标记位，V2 §3.2 的 `timestamp` 又是 `datetime\|null` 单字段 | **在现有契约下"填 null"是唯一不撒谎的选项**，这一点我不反驳。可批的是：① 不该把可用信息一并从 DTO 抹掉；② 影响评估不该降级为"待裁决小事"。 | 主流结论部分成立 |

---

### 质疑 7：真的"不修改上游、不在热路径写"吗？

| 质疑 | 主流结论 | 反方证据 | 我的判断 | 证据等级 |
|---|---|---|---|---|
| 7a 有无写路径指向上游 | "零修改上游" | 全仓 grep `INSERT/UPDATE/DELETE/writeFile`：只命中 `bridge-ledger.ts:67-93,119,181,230,237`（自有台账库）与 `summary-artifact.ts:30-78`（自有库）；`bc-source.ts:30` `new DatabaseSync(indexDbPath, { readOnly: true })` | **成立，不反驳。** 没有发现任何向上游库写入的语句。 | 主流结论成立 |
| 7b `readOnly:true` 真生效吗 | `bc-source.ts:29` 注释已标"未运行验证 —— UNVERIFIED 运行时项" | ① 只读打开 **WAL 库**在 `-shm` 不存在/无写权限时会失败（只读连接仍需访问 shm）；② 跨进程并发读的 `SQLITE_BUSY` 行为未测；③ 只读连接**不会**修改数据页，但可能读写 `-shm` 文件（属文件副作用，非数据修改） | 静态选项传对了；运行时是否可用**确实未验证**，CMB 的自我标注是诚实的。我只能补一句：`capabilities.concurrentRead: "UNVERIFIED"`（`bc-source.ts:127`）应与"不修改上游"的结论**强绑定**——在只读并发未验证前，"可安全并行部署"不能成立。 | 主流结论成立（边界提示） |
| 7c "不在热路径" | 读侧对账，不在请求路径 | `engine.ts` 是独立脚本；CLI 单独进程（`cmb.ts:34-63`） | 成立。但"离线"不等于"便宜"：见 4d/3e 的 O(块×会话) 与无上限重扫。 | 主流结论成立（性能保留意见） |

---

### 质疑 8：架构上有根本问题吗？

| 质疑 | 主流结论 | 反方证据 | 我的判断 | 证据等级 |
|---|---|---|---|---|
| 8a 读侧对账 + 物理删除 = 错过即永失 | RISK-REGISTER 有"块删除窗口"（D-8），定位为**已知限制** | `store.ts:1077-1087`：每次 ingest 把不在 `state.blocks` 里的块**物理 DELETE**；`store.ts:1643,1694-1696,2066` 另有删除点。CMB `engine.ts:103` 只遍历 `listBlocks` 的**现存**块 | **成立，而且比"限制"更严重：它不可审计。** 错过的块在 CMB 侧**连一行痕迹都没有**（无 ledger 行、无报告项、无 `SOURCE_MISSING`），所以"完整性"是**不可证伪**的断言。 | 确认缺陷（架构级） |
| 8b 能否用 Adapter 解决 | 未讨论 | 正面证据：上游确实有可用的**水印/单调性**——`sessions.last_ingested_seq`（`schema.ts:129`）、`messages.seq` 单调、`ingested_seq` 每事务写入（`store.ts:1012`） | **部分可解，不必先改上游**：以 `(session, seq)` 为坐标维护"我扫到哪个 seq"的高水位，与 `last_ingested_seq` 比对，可在**消息层**给出可观测的 gap 统计（"本会话 seq∈[a,b] 未被任何块覆盖"）。这不能找回块，但能把"不可审计"变成"可审计"。CMB 有 `sessions.last_ingested_seq` 却完全没用。真正的**不可解**部分只有"块级删除历史"（上游无审计列/无 tombstone），那部分确需上游改。 | 部分反驳主流（"只能等上游"不成立） |
| 8c 更根本的问题 | — | 见 1a：**身份维度在源侧不存在**。这不是窗口问题、不是契约问题，是**数据源不提供**。若产品要求"谁说的"（QQ Bridge 场景下的跨群/跨人记忆一定是硬要求），则 CMB 整条链路**在架构上不可成立**，任何 Adapter 都无法补救。 | 这比"块删除窗口"更根本，且**当前文档没有把它列为阻塞项**（只写"session_type 注入未实现 M-4"）。 | 确认缺陷（架构级，最高优先级） |

---

## 2. 反例清单（编号可被后续复核）

**A. 我确认的缺陷（15）**

| # | 反例 | 关键证据 | 后果 |
|---|---|---|---|
| X-1 | 上游 `messages` 表无发言者列，适配器恒 `speaker_id:null`，Validator 无条件强制 | `schema.ts:141-155`；`bc-source.ts:211`；`integrity-validator.ts:69-73`；`host-identity.ts:27` | 真实源 0 证据；块永久 MISSING 重扫 |
| X-2 | system prompt 排除判据用错维度 | `integrity-validator.ts:95` vs `kernel/src/types.ts:5-6`、`bc-source.ts:192,204` | 系统提示进入长期记忆（权限/隐私风险） |
| X-3 | tool 豁免判据命名不匹配 | `integrity-validator.ts:70`（`toolCall`）vs `types.ts:5-6`（`tool-call`） | 真实 assistant 工具调用消息被强制 speaker → 又叠加 X-1 |
| X-4 | `content_hash` 语义违反权威 MUST | `13-…md:246,255` vs `integrity-validator.ts:83`、`bc-source.ts:168` | 与 Memory Core 去重/校验不兼容 |
| X-5 | 测试把 X-4 固化为"正确" | `validator.test.ts:44-52` | 契约偏差在测试维度不可见 |
| X-6 | `evidence_id` 非 uuid 且可复用 | `evidence-adapter.ts:53`；`13-…md:236` | 与 §3.2 冲突；跨会话撞 id |
| X-7 | `scope` 缺失、DTO 多余字段 | `domain.ts:116-134`；`13-…md:245`；`evidence.schema.json:7` | 两个权威都不合规（B-1 的真相） |
| X-8 | 部分消息失败但块 COMMITTED 并冻结 | `engine.ts:228,281-282,112-116` | 静默永久丢消息 |
| X-9 | `COMMITTED` + payload 漂移 → `STATE_RESET_FAIL` 永久卡死、不落账 | `engine.ts:128-143`；`domain.ts:162`；`ingest.ts:429`；`kernel types.ts:89-95`、`compress.ts:1730-1743` | 块静默死亡，不可观测 |
| X-10 | `UNSUPPORTED` 二次扫描 → `STATE_RESET_FAIL`（`UNSUPPORTED` 只允许 →DISCOVERED） | `engine.ts:156-160,130`；`domain.ts:166` | 永久卡死 + 每轮污染 failed |
| X-11 | 无重试上限/退避；`retry` 配置是死字段 | `config.ts:13-16,72`（无读取点）；`engine.ts` 无计数 | 成本随失败数线性增长 |
| X-12 | 读侧无事务快照（message 与 blob 两次读） | `engine.ts:195`；`bc-source.ts:75-86`；对照上游 `store.ts:3070-3081` | TOCTOU：正常消息被判 HASH 不可验证 |
| X-13 | 幂等 = 内存 Map；台账 COMMITTED 不验证 Sink | `evidence-sink.ts:20,92-111`；`engine.ts:112-116`；`cmb.ts:52,55` | 重启后"已桥接 N / 证据 0"自相矛盾 |
| X-14 | 媒体分片未还原，纯媒体消息必被拒 | `identity.ts:40-77`；`ingest.ts:195`；`bc-source.ts:181-196` | "完整恢复"对多模态不成立 |
| X-15 | 块删除无审计；`sessions.last_ingested_seq` 有而不用 | `store.ts:1077-1087`；`schema.ts:129`；`engine.ts:103` | 完整性不可审计、不可证伪 |

**B. 我只是怀疑，但没证据（7）**

| # | 怀疑 | 缺什么证据 |
|---|---|---|
| S-1 | `directMessageIds: []` 的 tier-2/3 合并块在真实库中占比不低，导致大批块永久 FAILED | 需要真实 `index.db`：`SELECT count(*) FROM blocks WHERE json_extract(payload_json,'$.directMessageIds')='[]'` |
| S-2 | 块 payload 漂移（refold/失活/计数更新）在生产会话里是高频而非罕见，X-9 的实际触发面很大 | 需要同一 blockId 的两次 ingest 快照对比 |
| S-3 | 会话删除重建 + 已 revoke 的 Evidence 撞键 → 永久 `SINK_CONFLICT` 环在实际运维中会触发 | 需要真实 Sink 与删除/重建事件流 |
| S-4 | 真 Sink 的"写后立即回读"验证会因最终一致而常态化失败 | 需要真实 Sink 的一致性模型 |
| S-5 | 只读打开 WAL 库在并发写入下会 `SQLITE_BUSY`/shm 权限失败（`readOnly` 未必够） | 需要实验 E-1/E-3/E-9/E-10（CMB 自认未跑） |
| S-6 | `start_seq/end_seq = -1 → null` 的块触发整会话扫描，在 10⁶ 级会话上是主要开销 | 需要真实库 + 性能剖面 |
| S-7 | 上游 block 的 `payload_json` 在二次 ingest 后字节序是否稳定（影响 `block_version` 抖动） | 需要同一块两次 `JSON.stringify` 的字节对比 |

---

## 3. 我无法反驳的地方（明确列出，避免"为反而反"）

1. **`getBlob` 的 `?? ""` 静默空串路径**：`bc-source.ts:97` 明确 `return null`，`integrity-validator.ts:80-81,87-91` 对缺失/不可验证都拒绝。**这条修得对**，比上游 `store.ts:3013` 好。
2. **MISSING/CORRUPTED → DISCOVERED 的转移**：`domain.ts:164-165` 明确合法，复位排除这两个状态是**正确的**（先转 FAILED 反而多一跳）。lead 担心的"这两态会不会又崩"——**不会**；会崩的是 `UNSUPPORTED` 与 `COMMITTED`（X-9/X-10）。
3. **`supersedes_block_id` 恒 null、块会被物理删除、`payload_json` 解析失败被上游静默吞**：CMB 对这三条上游事实的静态引用**全部准确**（`ingest.ts:425`、`store.ts:1077-1087`、`store.ts:1105`），并据此把 `getBlockHistory/getSupersededBlocks` 显式抛 `UNSUPPORTED` 而不是假装可用（`bc-source.ts:101-113`）。这个"拒绝假装"的纪律应当被表扬。
4. **"不修改上游"**：全仓无任何写向上游的语句，`readOnly:true` 传参正确。7a/7c 成立。
5. **测试数 78**：经更正后确实成立（见质疑 3 开头），不得作为攻击点。
6. **`?? ""` / 空 blob / 哈希不符都走显式拒绝**：CMB 在"不静默"这一点上总体优于上游。

---

## 4. 阶段二：交叉审查（同行报告已读）

> 已读：`01`、`02`、`03`、`05`（Agent B）、`06`/`08`/`schema-diff.json`（Agent C）、`07`（Agent E）、`12`、`13`、`15`、`evidence/LEAD-CROSS-VERIFICATION-LOG{,-2,-3}.md`。
> 立场不变：只读、不为了让谁难堪而编造。凡我判同行错，给出 `文件:行`；凡我判自己对，同样给证据；凡我阶段一说错，逐条更正（§4.0）。

### 4.0 先更正我自己（三处）

| # | 阶段一原说法 | 更正 | 依据 |
|---|---|---|---|
| C-1 | **X-12**「读侧无事务快照 → content 行可能被回收而消息行仍在」定为**确认缺陷** | **降级为条件缺陷（INFERRED-LOW）**。上游每个 index.db 打开点都执行 `PRAGMA foreign_keys = ON`，`messages.content_hash REFERENCES content(hash)` 因此是真约束：**被消息引用的 content 行删不掉**，"行在 blob 无"只在（a）FK 未开启时期留下的历史脏库、（b）外部手工手术、（c）整会话删除 三种情况出现。剩下的真实窗口只有"两次 SELECT 之间的块级变更"，比阶段一估计窄得多 | `schema.ts:88-95`（`PRAGMA foreign_keys = ON` 在 `storagePragmas` 里，注释"Connection pragmas for every index.db opener"）；`schema.ts:149`；`store.ts:799-808` |
| C-2 | 我阶段一用的 `X-1…X-15` 编号 | **与 `LOG-3` 的编号不是同一套**，已被混用。映射：LOG-3 `X-1`=本文件 `X-1`；LOG-3 `X-2`=本文件 **`X-9`**；LOG-3 `X-3`=本文件 §3-①；LOG-3 `X-4`=本文件「质疑 3」；LOG-3 `X-5`=本文件「5a」；LOG-3 `X-6`=本文件 **`X-4`**。→ 证据链已出现编号串台，见 §4.7 的稳定编号表（建议全pack 改用 `D-n`） | `LOG-3:10-17` vs 本文件 §2 |
| C-3 | （未表态） | 采信 `12` 的 78=74+4 解释，与我独立得出的结论一致（我数静态 75 是我漏了 `phase6:34` 模板串；正确算法是 74 call site + 3 个循环实例）。**两处独立得出同结论，可互证** | `12:39-55`；本文件「质疑 3」开头 |

### 4.1 冲突矩阵（10 处）

| # | 冲突点 | 甲方主张 | 乙方主张 | 我的裁决 | 依据 |
|---|---|---|---|---|---|
| **K-1** | 「blob 可完整恢复」 | 第三轮报告 `L16`「正文确实逐字存于 `content.bytes`，可完整重建 CoreMessage」；交付报告 §三「原文恢复 VERIFIED_RUNTIME」；`02:155`「Mock 源 → Evidence ✅ **完整**可运行」 | **Agent B `05:411-441`**：`content.bytes` = `JSON.stringify(CoreMessage−ts)` **再经 `splitMediaContent()` 媒体外置**；含图消息是 `{"$media":…}` 占位；`hash` 覆盖**拆分后**字节；`byte_len` 是拆分后长度 | **Agent B 对，甲方错**。这是本轮最重要的一处纠正。"完整恢复"对纯文本消息成立，对含图消息**不成立**（少一次 CAS 跳转），且 CMB 从不解 `$media`。另注：Lead 在 `LOG-3:160` 已采纳 B 的表述，但 `01/02/第三轮` 未回改 → **纠正未传播** | `identity.ts:40-77`、`ingest.ts:191-205`、`store.ts:3095-3128` vs `bc-source.ts:180-196` |
| **K-2** | M-2（部分覆盖判 COMMITTED）该不该修 | `PHASE6-CHANGES:25`「裁决记录（**不修**，留待正式接入）」 | `01:102-111` R-A2「MAJOR，**未修复**」；我 `X-8`「静默永久丢消息」 | **"不修"这个裁决在证据上站不住**：它没有记录后果（块 COMMITTED 后 `engine.ts:112` 永不重扫 ⇒ 丢的消息**永久找不回**）。同一事实三种表述（裁决不修 / MAJOR 未修 / BLOCKER 级数据丢失），说明裁决时**没有量化影响** | `engine.ts:224-226`、`:228-234`、`:281-282`、`:112-116` |
| **K-3** | `COMMITTED` 会不会进复位段 | `03:84`「COMMITTED｜**被幂等跳过（不复位）**｜设计意图」 | `LOG-3:88-112` 实测「payload 变 → `STATE_RESET_FAIL` × 2 轮，块永久 COMMITTED」 | **LOG-3 对，03 错**（03 写于核验之前）。但 `LOG-3` 未回改 `03`，两份文档现在**互相矛盾**且都在交付包里 | `engine.ts:112`（跳过条件含 `block_version`）、`:128-143`；`domain.ts:162` |
| **K-4** | 漂移的触发维度 | `03:240`「block payload 变化（**covered_hash 变**）→ bridge_key 变 → 新 key」 | `LOG-3:92-99`「covered_hash **不变**、payload 变」 | **LOG-3 对**。`03` 只考虑了"覆盖集变"这一半，漏掉的正是会死人的另一半。补充：上游 refold-in-place 明确"same id, replaced summary"（`kernel/src/types.ts:89-95`、`compress.ts:1730-1743`） | 同 K-3 |
| **K-5** | `content_hash` 的严重度 | `08:74,103,504`：`NO_DIFF / COMPATIBLE_STRUCTURALLY_ONLY / **USER_DECISION_REQUIRED** / blocking=**否**`；`06:504` 同桶 | `06:160` 自己引了 `13:L255` 的 MUST；`LOG-3:139-212` 定 **BLOCKER**；我 `X-4` 同 | **LOG-3/我 对，Agent C 分类过轻。** 关键补证（C 方没引用到的）：`06:392` 记载 Memory Core 的**幂等键就是 `content_hash` 去重**（`api-contracts.md:71`）——这使偏离从"语义瑕疵"变成"功能失效"：同句不同 msg_id 永不命中去重、且无法验原句。→ 该项应从 `USER_DECISION_REQUIRED`（无需动作）改判 **`ADAPTER_WORK_REQUIRED` + blocking**，blocking 总数应为 **13** 而非 12 | `13:246,255`；`06:392,160`；`08:74` |
| **K-6** | `source_type` 该改哪边 | `08:65`「schema 12 值不含 chat → **UPSTREAM_CHANGE_REQUIRED**」 | `08:144`「**CMB 需改为 `conversation`**」（同一文件 §4.3） | **两条互斥，前者对**。按权威序（`DD:5`、`13:1028`）CMB 的 `"chat"` 正确，schema 的 12 值枚举是把"迁移别名表"误当枚举（`08:182-195` 已证）。→ `08:144` 应改为"保持 chat，待 schema 修正" | `DD-AUTHORITY-V2.md:5,212`；`13:237,404`；`evidence.schema.json:13` |
| **K-7** | `visibility` 是"最容易解"吗 | `08:128`（§4.1）「权威文档**支持** CMB」；`08:154`「12 个 blocking 里**最容易解**的一个，`evidence.scope = dto.visibility` 即可」 | `08:85`（同文件 §3 行 22）「权威 §2.10 = {owner_private, group_shared, public}」；`DD-AUTHORITY-V2.md:130`「visibility（**3 值**）」 | **08 §3 对、§4 错**。CMB 的 5 值 `visibility` 与 v1 **`scope`** 枚举逐值相同（`08:148-153` 自己承认）⇒ 它其实是 `scope` 放错了槽。§3.2 同时要求 `visibility`(3 值) **与** `scope`(5 值)：一次投影只能填一个，另一个必须按 `DD:296-302` 的映射表从 scope **派生**。→ 不是"最容易"，是"1 字段拆 2 字段 + 一次三值派生"；结论应从 `USER_DECISION_REQUIRED` 改 **`ADAPTER_WORK_REQUIRED`**（映射已被权威文档给定，不需要裁决） | `DD-AUTHORITY-V2.md:130-136,296-302`；`domain.ts:29-34`；`evidence.schema.json:28` |
| **K-8** | 上游 messages 会不会被删 | `05:387` B7「✅ 通过（附『**仅 blocks，messages 不删**』补充）」 | 上游存在多处消息物理删除：重复行清理 `store.ts:1405-1411`、`deleteSession` `store.ts:1694`、`pid` 重写 `:1411` 邻域 | **`05` 的措辞只在"权威集清扫只删 blocks"这一狭义上成立**；作为一般陈述会误导（读者会以为消息行永存）。建议改为「权威集清扫只删 blocks；messages 另有删除路径（去重清理/删会话）」 | `store.ts:1077-1087` vs `:1405-1411`、`:1694` |
| **K-9** | Phase6 回归到底几条 | `03:115`「**12 条** Phase6 回归测试未覆盖 UNSUPPORTED」；交付报告 §四「补 **12 条**回归」 | `12:18`「integration 共 20 → pipeline 9 + phase6 11」 | **11 条**（20−9）。数字错误被复制到两处文档。不影响"漏 UNSUPPORTED"的结论（两条都成立），但需更正 | `12:18`；`phase6:33`（4 实例循环）+ 7 个独立 = 11 |
| **K-10** | AI-1 是否"两个 JSON Schema 机器互斥" | `08:244`「**不可能同时通过两个 schema**」；`06:100`「`:50` 与 `:166` **两处**均要求原句存于 Evidence.original_text」 | 我复核：`memory-unit.schema.json` 全文只有 **L6**（`$comment` 的错误示范）与 **L50**（`content.description`）提到原句；**L166 是 `evidence_refs` 的描述，不含该句**（我 grep `original` 全文件仅命中 L6/L50，文件 293 行） | **AI-1 的方向对，机制描述过强，且引用了不存在的第二处。** 精确表述应为：「`evidence.schema.json` 无 `original_text` 属性，而 MemoryUnit 的 content 描述与错误示范要求原句存于 Evidence ⇒ **契约层无法表达原句**」。机器上两个 schema **可以**同时通过——只要不携带原句（那就不再是证据）。所以这不是"二选一裁决"，而是"schema 少字段，必须补"。见 §4.5 | `memory-unit.schema.json:6,50`；`evidence.schema.json:7,9-35` |

### 4.2 证据等级误标审计

| # | 被标为 | 实际等级 | 位置 | 说明 |
|---|---|---|---|---|
| G-1 | 「原文恢复 / 幂等 / 故障自愈 / security 边界」= `VERIFIED_RUNTIME`（交付报告 §三） | **MOCK_ONLY**（原文恢复还叠加 K-1 的"含图不成立"） | `CMB-v0.1-交付报告.md:27-34` | 唯一真实组件 `bc-source.ts` 零执行、零测试（`cmb.ts:44-45` 只装配 Mock） |
| G-2 | 「Mock 源 → Evidence ✅ **完整**可运行（78 测试）」 | **MOCK_ONLY**；"完整"应删 | `02:155` | 与 `02:156` 自己写的"真实 index.db：适配器已写、**从未运行**"并列，同一表格内等级不一致 |
| G-3 | `05` 通篇 `VERIFIED_STATIC`、"实测行为" | 大部分合规；但 `05:265`「实测行为=保留 `{$media:…}` 字面量」若为真测，应标 `VERIFIED_RUNTIME(副本)`；`05:470`「**没有任何代码校验 covered_hash**」是**静态否定命题**，等级只能是 `VERIFIED_STATIC(全仓 grep)`，不能升 RUNTIME | `05:265,470` | 等级标注颗粒度不一致（同文件混用），不影响结论 |
| G-4 | `08:5`「除特别标注外均为 `VERIFIED_STATIC`」 | 结论列的 `UPSTREAM_CHANGE_REQUIRED / USER_DECISION_REQUIRED` 是**判断**不是静态事实；`08:117` 已自注 blocking 判据为 `INFERRED` | `08:5,117` | 建议：结论列另标 `JUDGMENT`，不要被表头一并承认为 STATIC |
| G-5 | `07` 的 `VERIFIED_*` 探针 | **MOCK_ONLY 上的 RUNTIME**（探针只跑 Mock Provider） | `07:10,150` | `07:150` 已诚实区分"真实路径恒 null"，无实质误标；但外部读者易把 `VERIFIED_RUNTIME` 当作真实链路验证，建议加"环境：Mock Provider"前缀 |
| G-6 | `13:52`「吞吐高估约 2 倍」 | 正确自我更正，`13:44-45` 又撤回一半 | — | 属**良好实践**（自我纠偏），列在此仅为对称 |
| G-7 | `12:8-13`「78 passed … VERIFIED_RUNTIME」 | **成立**（确实执行过） | — | 但"运行通过"≠"验证有效"；同一文件 §2 已把 0 条真实持久化讲清楚，属**等级正确、结论克制**的样板 |

**小结**：误标**主要不在同行报告里**（`12`/`13`/`07` 的自我约束总体良好），而在 **CMB 自身文档**（交付报告 §三），并被 `01/02` 的摘要行继承。这一点必须公平地说：本轮 Agent 的等级纪律**优于**被审对象。

### 4.3 群体共识点复核（"三个都说对"≠ 事实）

| # | 共识 | 谁在说 | 独立证据？ | 我的复核 |
|---|---|---|---|---|
| Z-1 | **"blocking 字段 = 12 个"** | `06:230`、`08:103`、`schema-diff.json.summary`（**三份产物**） | **无独立来源**：三份都是同一 Agent 从同一个 JSON 生成的，JSON 又由同一次人工分类生成 | 共识是**单一来源的三次渲染**。按 K-5/K-7 两项改判后应为 **13**（content_hash 入列）且 `visibility` 结论应改。→ 这个数字**不能**作为"三方复核通过"引用 |
| Z-2 | **"78 条测试全绿"** | `01:178`、`02:155`、`12:8-13`、交付报告 | `12` 实跑过（真证据）；其余是转述 | 数字真，但共识掩盖了"77 条有效 + 0 条真实持久化 + 0 条真实上游 + 5 个真实缺陷全部逃逸"。**"全绿"与"可信"是两件事**，pack 里只有 `12` §2 把这点讲透 |
| Z-3 | **"块删除窗口 D-8 是已知限制，可接受"** | `03:241`、`05:387`（B7）、第三轮 `N-6`、交付报告 §六 | 三方引同一行 `store.ts:1077-1087` | **事实对，结论未经检验**：没有任何报告检查过"CMB 是否至少能**发现**自己漏了块"。我复检：`engine.run()` 只遍历现存块，**无 gap 检测、无覆盖率断言**（`engine.ts:103`），因此"接受该限制"缺少前提——它不是"少收一点"，而是"漏了也不知道"。补：可用 `sessions.last_ingested_seq`（`schema.ts:129`）做消息层 gap 审计，成本极低，三方都没想到 |
| Z-4 | **"B-1 权威序：V2 §3.2 优先"** | `08:17,304`、`06`、`LOG-3:§3`、我 5a | **有**：`DD:5`、`13:1028`、`13:237,404` 三处独立文本 | **真共识，可采信**。这条是本轮唯一"多方独立 + 原文直引"的强共识 |
| Z-5 | **"MR-2/覆盖关系静态可证"** | `05:381-392` 12 条 B 项、`01:10-26` 模块表 | `05` 给了 `文件:行`，可复核；`01` 的模块表是转述 | `05` 的 B 表**质量最高**（逐条给行号 + 标"通过/需替换"）；`01` 的模块完整性表属转述，不可作独立证据。→ 引用上游事实时**只引 05**，不要引 01 的复述 |

### 4.4 遗漏的异常路径（pack 全体合计仍缺）

> 说明：`15-REAL-DATA-EXPERIMENT-PLAN.md` 的实验设计**相当完备**（E-1…E-12 覆盖结构/编码/哈希/缺失/损坏/加密/版本/WAL/时间/重铸/生命周期/契约/完整性/撤销）。因此下述缺口**不是"少做了实验"，而是"已知上游事实没有接到本仓库代码路径上"**——即分析链断点。

| # | 遗漏路径 | 为什么没人发现 | 建议（最小成本） |
|---|---|---|---|
| M-1 | **媒体外置 → CMB 的 `original_text` 提取**：`bc-source.ts:183-196` 只认 `core.text`，含图/纯工具消息 `text` 缺失 → `CONTENT_MISSING`。`05` 把机制讲透了，但**没有一份报告把它接到 CMB**；`E-1/E-2` 只验证"能不能读出来" | 上游审查与 CMB 审查之间缺一步"事实→代码路径"映射 | 在 E-1 加一格：统计 `text` 缺失的消息占比（直接决定 CMB 的丢弃率） |
| M-2 | **`directMessageIds: []` 的 tier-2/3 合并块**：`source-resolver.ts:62-63` 把空数组当"有"，⇒ 0 消息 ⇒ 块永久 FAILED 且每轮重扫。上游允许此形态（`compress.ts:1719-1723` 的活锁保护只拦 `!isBlockBoundary`；`:1813-1819` 以 `filteredIds=[]` 建块） | 无人把"块可以有空的 direct"与"resolver 用 `!== null` 判断"连起来 | E-9 加一条 SQL：`json_array_length(payload_json,'$.directMessageIds')=0 AND json_array_length(...effectiveMessageIds)>0` 的块数 |
| M-3 | **`covered_hash` 全仓无人校验**（`05:470` 的关键发现）却**被 CMB 当作幂等根键**（`bridge-ledger.ts:272-274`） | `05` 提了但止于上游；`01/03` 重复了"三层幂等"却没问"根键可信吗" | 明确写进风险：bridge_key 的完整性前提 = 上游写入正确，而该前提无代码保证（仅 `ingest.ts:428` 一处写入点） |
| M-4 | **`EvidenceDTO.timestamp` 恒 null → 下游 M03/M04 的不可逆损失**：`06` 只把它当字段缺失；`E-4` 只测"ts 语义"，没测"下游还能做什么" | 桥接与 Memory Core 的时间轴需求之间无人接力 | 在 `06` 增一节：无时间戳时 Event.`time_start`/情绪衰减/画像时效的可行域（我阶段一 6c 已给初判） |
| M-5 | **`counts.pending` 恒为 0**：中间态从不落账（`persistBlockRow` 只在终态调用） | 指标层无人复核 | 从报告契约里删掉该字段或补中间态落账 |
| M-6 | **`evidence_id` 除"会话重建"外还有"seq 复用"路径**：上游 `retired_at` 复活/rewind 会把行按新 seq 重放（`store.ts:1399-1412`），旧 seq 可能被复用 → 同 session 内撞 id | `01` R-A4 只测了会话重建 | E-5 加探针：同一 session 内 `(msg_id, seq)` 是否出现"旧 seq 换 msg_id" |
| M-7 | **真 Sink 最终一致 ⇒ `engine.ts:254` 写后立即回读必失败** ⇒ 常态化 PARTIAL + 重写风暴 | 该验证只在内存 Map 上成立，无人评估真实 Sink 语义 | 在证据契约里要求 Sink 提供 read-your-writes；否则删掉这个"Verify"步骤 |
| M-8 | **历史脏库的 FK 缺失风险**（我 C-1 的更正带来）：若某库建于 `PRAGMA foreign_keys=ON` 之前，"行在 blob 无"仍可达 | 无人看 pragma 历史 | E-2 加"孤儿 message 行"计数 SQL |

### 4.5 AI-1 / AI-2 对 CMB 的影响（任务 5）

**AI-1（原句无处安放）**

- 精确化（见 K-10）：`evidence.schema.json` 无 `original_text` 属性 + `additionalProperties:false`；`memory-unit.schema.json:6,50` 要求"原句 MUST 存于 `Evidence.original_text`"。**机器层不互斥**（不带原句就能同时过），**语义层互斥**（不带原句就不是证据）。
- 对 CMB 的影响：
  1. CMB 的**唯一核心产出就是 `original_text`**（`domain.ts:122`、`evidence-adapter.ts:57`）。它正好是唯一能满足 MemoryUnit 原句追溯的形态，却被落盘契约拒收 ⇒ **CMB 的产物在当前契约下无处可去**，与"是否接 Memory Core"无关。
  2. 因此 `MemoryCoreEvidenceAdapter` 全方法抛 `SINK_UNSUPPORTED`（`evidence-sink.ts:92-111`）**是当前唯一正确行为**——这点必须给 CMB 记正分，它没有假装可用。
  3. 但 CMB 的**理由陈述不完备**：它说"schema 冲突（B-1）未裁决"，实际是"契约无法表达原句"（AI-1）+ "字段集是 v1"（B-1）。前者**不是裁决问题**，是上游必须改 schema；把它写成"待用户裁决"会让人误以为存在"保持现状"的选项。**没有**。
- 结论：AI-1 是 CMB 停止线的**充分理由**，且**责任 100% 在 Memory Core 侧**（补 `original_text`，并删除 `recorded_at` 这个权威文档里不存在的必填项）。

**AI-2（权限不可机器校验）**

- 事实：schema 无 `visibility` 枚举（仅自由串 `visibility_scope:32`）；权威 §2.10 是 3 值 `owner_private/group_shared/public`；CMB 的 5 值实际是 v1 `scope` 枚举（K-7）。
- 对 CMB 的影响（三点，比 `06:§A-5` 更进一步）：
  1. **CMB 的安全叙事建立在"自造字段"上**：`SEC-1…SEC-5`（`security.test.ts`）全部断言 `visibility` 取值正确，但该字段**在两个契约里都不存在**（schema 无；权威叫 §2.10 三值且 CMB 没实现）。→ 交付报告 §三"私聊/群隔离 fail-closed VERIFIED_RUNTIME"的证据等级最多 **MOCK_ONLY + 不可校验**。
  2. **`04` 的 PERM-1（`visibility(evidence) ≤ visibility(memory)`）在机器层不可执行**（`06:174,245`、`08:245`）⇒ CMB 无法**证明**（只能自称）其证据权限不宽于记忆。任何下游复核都只能靠人读 JSON。
  3. **可修但没人写清修法**：`DD:296-302` 已给 5 值 `scope` → 3 值 `visibility` 的映射（`owner_private/session_only/project_scoped → owner_private`，`public` 禁用）。所以 K-7 的修法是**确定的**，不需要裁决。CMB 应：`visibility` 改名 `scope`，新增派生 `visibility`。
- 结论：AI-2 对 CMB 的影响是"**权限保证不可验证**"，不是"零证据"级别（比 AI-1 轻），但它直接削弱了 CMB 最想展示的安全卖点。

### 4.6 pack 级交付质量观察（顺带的反方意见）

| # | 现象 | 影响 | 建议 |
|---|---|---|---|
| Q-1 | **纠正不回写**：`LOG-3` 已推翻 `01/02/03` 的若干结论，但原文未改 | 读者按目录读 `01→03` 会拿到**已被推翻**的结论（K-1/K-3/K-4） | 在 `01/02/03` 头部加一行"本文已有后轮更正，见 LOG-3/19" 或直接修订 |
| Q-2 | **编号串台**（C-2） | 证据链断裂，跨文档引用会指错缺陷 | 见 §4.7 统一 `D-n` |
| Q-3 | `03:115` 的 12 vs 11（K-9） | 轻微，但会被下游复制 | 改 11 |
| Q-4 | `08` 内部自相矛盾（K-6/K-7） | 同一文件给出相反修法，实施者会选错 | §4.3 改为"保持 chat"；§4.4 改为"拆 2 字段 + 派生 3 值" |
| Q-5 | `06:100` 引用 `memory-unit:166` 不存在（K-10） | 削弱 AI-1 的证据强度（结论仍成立） | 改为 `:6` 与 `:50` 两处**文档级**依据，并声明"非机器约束" |

### 4.7 稳定缺陷编号表（建议全 pack 采用）

| ID | 缺陷 | 等级 | 归属 | 首次提出 | 核验状态 |
|---|---|---|---|---|---|
| D-01 | 真实源无发言者列 + 适配器恒 null + Validator 无条件强制 ⇒ 零 Evidence | **BLOCKER** | 上游+HOST 改 | 19 号 X-1 | 实测 `LOG-3 §1` |
| D-02 | `content_hash` 违反 §3.2 MUST（应 sha256(original_text)），且测试锁定违规 | **BLOCKER** | CMB | 19 号 X-4 | 三方一致 `LOG-3 §3` |
| D-03 | `COMMITTED`+payload 漂移 → `STATE_RESET_FAIL` 永久锁死 | **BLOCKER** | CMB | 19 号 X-9 | 实测 `LOG-3 §2` |
| D-04 | 部分消息失败仍判 `COMMITTED` 并永久跳过（M-2） | **BLOCKER**（原 MAJOR） | CMB | 19 号 X-8 / `01` R-A2 | 实测 `01:110`；`PHASE6:25` 裁决"不修"未量化后果 |
| D-05 | `UNSUPPORTED` 二次扫描 `STATE_RESET_FAIL` 永久卡死 | MAJOR | CMB | 19 号 X-10 / `01` R-A3 | 实测 `03:105-110` |
| D-06 | `source_locator.split(":")` → 含冒号 session_id 撤销失效 | MAJOR | CMB | `01` R-A1 | 实测 `LOG-1 L-3` |
| D-07 | `evidence_id` 非 uuid 且撞键 | MAJOR | CMB | 19 号 X-6 / `01` R-A4 | 实测 `01:127` |
| D-08 | system prompt 未排除（判 `content_type` 而非 `role`）+ SEC-7 用不存在的枚举伪造保证 | **BLOCKER（安全）** | CMB | 19 号 X-2 | 静态确证（`types.ts:5-6`） |
| D-09 | `visibility` 槽位装的是 `scope` 值；`scope` 缺字段；§2.10 三值未实现 | MAJOR | CMB | 19 号 + K-7 | 静态确证 |
| D-10 | 幂等仅 MOCK_ONLY；台账 COMMITTED 不验 Sink | MAJOR | CMB | 19 号 X-13 | 静态确证 |
| D-11 | 无重试上限/退避；`retry/batch_size/concurrency` 死配置 | MAJOR | CMB | 19 号 X-11 / `13:110-112` | 静态确证 |
| D-12 | 媒体外置不还原；纯媒体消息必被拒 | MAJOR | CMB(可修) | 19 号 X-14 / `05` B2 | 静态确证 |
| D-13 | 块删除无审计、无 gap 检测 ⇒ 完整性不可证伪 | 架构级 | CMB(可部分修) | 19 号 §8a | 静态确证 |
| D-14 | 契约无法表达原句（AI-1） | BLOCKER（外部） | Memory Core | `06`/`08` → 本文件 §4.5 | 修正后成立 |
| D-15 | 权限不可机器校验（AI-2） | MAJOR（外部） | Memory Core | `06`/`08` → 本文件 §4.5 | 成立 |
| D-16 | 直连读无事务快照（条件缺陷，见 C-1） | MINOR（条件） | CMB | 19 号 X-12（**已降级**） | FK ON 大幅收窄 |

### 4.8 阶段二结论（三句话）

1. **同行报告的质量分布是"上游 > 测试 > 契约 > CMB 本体"**：`05`（Agent B）的本体事实质量最高（逐条带行号且敢推翻主代理）；`12/13` 的自我约束良好；**最弱的一环是"上游事实 → 本仓库代码路径"的接线**（M-1…M-8 八条缺口全在这一步），这也是我阶段一 22 条反例的主要来源。
2. **主代理的自我更正（LOG-3）方向正确但未回写**，导致 pack 内同时存在"已被推翻的结论"和"推翻它的证据"（K-1/K-3/K-4）。这不是学术瑕疵，是**交付风险**：实施者按 `01/02/03` 做事会踩已确认的坑。
3. **B-1 的性质被全体低估了一次，又高估了一次**：低估在 `content_hash`（K-5，应入 blocking）；高估在 AI-1 的机制描述（K-10，"两 schema 机器互斥"不成立）。净结论不变：按 `08` 的 32 字段口径，**blocking 应为 13 条（12 + `content_hash`），剩余 19 条中 4 条 `NO_DIFF`、15 条为"缺失/多余但非阻断"**；其中 3 条（哈希语义 K-5、权限双字段 K-7、时间戳策略 6c）必须在写第一行真实接入代码之前定案。

### 4.9 需要 Lead 裁决的三件事

1. **D-04（M-2）的"不修"必须重开**：`PHASE6-CHANGES:25` 的裁决没有记录"永久静默丢消息"这一后果。建议改为"已知数据丢失，接受条件 = 报告必须输出 `declaredCount` vs `evidenceCount` 差值"。否则 v0.1 的"完整性"报告不可信。
2. **blocking 计数改 13 并同步三份产物**（`06:230`、`08:103`、`schema-diff.json.summary`），否则 pack 内部数字不一致，且 `content_hash` 这个 BLOCKER 会被 `USER_DECISION_REQUIRED` 的标签埋葬。
3. **`01/02/03` 加"更正指针"或直接修订**（Q-1），并把全 pack 缺陷编号统一为 `D-n`（§4.7），否则本轮的交叉核验成果无法被下一轮引用。
