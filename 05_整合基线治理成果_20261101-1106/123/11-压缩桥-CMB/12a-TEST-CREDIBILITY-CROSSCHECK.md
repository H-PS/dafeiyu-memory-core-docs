# 12a — 测试可信度与性能数字「独立交叉复核」

> 复核对象：主代理代写的 `12-TEST-CREDIBILITY-AUDIT.md`、`13-PERFORMANCE-REVIEW.md`、`test-inventory.json`
> 复核人：Agent D（测试可信度 / 故障恢复 / 性能审查）
> 时间：2026-11-06 ｜ 工作目录：`E:\[PROJECT]\CMB-v0.1-prototype` ｜ Node v24.21.0
> 纪律：**未修改原型任何 `.ts` 文件**；所有探针放在 `E:\[PROJECT]\_agentD_probe\`，所有复跑在隔离副本 `_agentD_perf_tmp*` 内进行。

**证据等级图例**：`VERIFIED_RUNTIME`=我实测 ｜ `VERIFIED_STATIC`=读码可证 ｜ `MOCK_ONLY` ｜ `INFERRED`

### 副作用声明（必读）

1. 按任务要求我运行了 `node tests/performance/perf-run.ts small`，该脚本会**无条件覆写** `reports/perf-results.json`（`perf-run.ts:210`）。原文件中的 Large 记录因此被我的 Small 记录替换。Large 的原始数字仍完整保留在 `docs/reports/PERF-REPORT.md:52-66`，下文所有关于 Large 的推算均引用该文档。
2. 我创建的临时目录：`_agentD_probe/`（探针）、`_agentD_perf_tmp/`、`_agentD_perf_tmp2/`（原型 src+tests 的只读副本，用于纯净 DB 复跑）。原型目录内仅 `data/test-only/*.db` 与 `reports/perf-results.json` 发生变化。

---

## 一、测试数量独立复核

### 1.1 实测（VERIFIED_RUNTIME）

```
node tests/runner.ts
→ 78 passed, 0 failed, 9 files, 84ms          exit=0

node tests/runner.ts tests/unit            → 35 passed, 0 failed, 5 files, 37ms   exit=0
node tests/runner.ts tests/integration     → 20 passed, 0 failed, 2 files, 59ms   exit=0
node tests/runner.ts tests/fault-injection → 11 passed, 0 failed, 1 files, 43ms   exit=0
node tests/runner.ts tests/security        → 12 passed, 0 failed, 1 files, 35ms   exit=0
```

连跑 3 次（稳定性）：

```
RUN 1 : exitcode=0 | 78 passed, 0 failed, 9 files, 88ms
RUN 2 : exitcode=0 | 78 passed, 0 failed, 9 files, 86ms
RUN 3 : exitcode=0 | 78 passed, 0 failed, 9 files, 89ms
```

> 主代理的 78 / 35 / 20 / 11 / 12 与 exit=0 —— **逐项确认**。耗时有 ±10ms 抖动，无实质差异。

### 1.2 三种互不依赖的方法算 78

| 方法 | 做法 | 结果 |
|---|---|---|
| ① 静态调用点 | 正则 `^\s*test\(` 逐文件计数 | **75** 处 |
| ② 运行时实例 | 75 处中 1 处在 `for` 循环内（`phase6-regression.test.ts:33`，4 元素）→ 75−1+4 | **78** |
| ③ 实测输出 | 从 runner 输出按 `ok  <file> ::` 分组计数 | **78** |

方法③ 的逐文件明细（这是唯一无歧义的口径）：

| 文件 | 运行时用例数 |
|---|---|
| `tests/fault-injection/fi.test.ts` | 11 |
| `tests/integration/phase6-regression.test.ts` | **11** |
| `tests/integration/pipeline.test.ts` | 9 |
| `tests/security/security.test.ts` | 12 |
| `tests/unit/domain.test.ts` | 7 |
| `tests/unit/evidence.test.ts` | 9 |
| `tests/unit/ledger.test.ts` | 5 |
| `tests/unit/resolver.test.ts` | 7 |
| `tests/unit/validator.test.ts` | 7 |
| **合计** | **78** |

### 1.3 判定：总数 78 正确，但主代理的「74 + 4」分解有两处口径错误

**D-1（纠正，低危）**：`12` §1 表格把 `phase6-regression.test.ts` 记为 **7**，`test-inventory.json` 记为 `"count": 7`。
实测该文件 **8 个 `test()` 调用点 / 11 个运行时实例**（7 个循环外 + 1 个循环内 ×4）。理由：7 + 9 + 5 + 7 + 7 + 9 + 11 + 11 + 12 = 78；若按 7 计则只有 74。
→ 主代理的「74 = 各文件之和」与 runner 的「78 = 实际执行数」之间的差 4，其实来自 **phase6 少记了 4 个实例**，不是「74 个静态点之外另有 4 个」。「74 + 4 = 78」的算式能对上，但**语义是错的**：真实静态调用点是 **75**（含循环内那 1 处），75 − 1 + 4 = 78。

**D-2（纠正，低危）**：`test-inventory.json` 内部不自洽 —— `files[].count` 之和 = **74**，而 `total_reported` = **78**、`summary.by_category` 之和 = **78**。同一个文件里两个口径打架。建议 `phase6` 条目的 `count` 改为 11（或把字段改名为 `non_loop_call_sites`）。

**确认**：`test-inventory.json` 的 9 个 `files[]` 条目与磁盘上的 9 个 `.test.ts` **一一对应，路径全部存在，无虚构、无遗漏**（`runner.ts` 的默认根目录正好收集这 9 个）。

### 1.4 ESM 模块缓存：主代理提出的风险，实测结果

实测：

```
node tests/runner.ts tests/unit tests/unit
→ 35 passed, 0 failed, 10 files, 37ms
```

**结论（VERIFIED_RUNTIME）**：
- 同一 specifier 第二次 `import()` 命中 ESM 缓存**不重新执行模块**，`registry` 不再收到注册 → 第二次的 5 个文件**静默跑了 0 条**。但计数行仍显示 `10 files`，**文件数会骗人**。
- 因此主代理「ESM 缓存可能让文件被跳过」的担心**方向正确**，但当前套件不受影响：每个文件只被导入 1 次，`registry.length = 0` 在 import 之前执行，顶层 `test()` 同步注册 —— 这条链路是干净的。**确认主代理 §3.5「无模块级可变状态」的结论成立**（我额外检查了 `src/`，无模块级 `let`/可变单例；`DEFAULT_CONFIG` 未冻结但所有测试都是展开拷贝，未发生就地改写）。
- **D-3（新增，中危）**：`runner.ts:67` 的 `await import(spec)` **不在任何 try/catch 内**，`main()` 的 rejection 直接 `process.exit(2)`。任何单个测试文件的顶层语法错误/导入失败 → **整轮 0 报告、退出码 2、已跑过的结果全部丢失**。作为「唯一测试入口」，这是可用性缺陷：CI 里会表现为"没有输出"而不是"某文件坏了"。

---

## 二、测试有效性独立质疑

### 2.1 主代理已指出的三条 —— 逐条独立验证

| 主代理主张 | 我的复核 | 判定 |
|---|---|---|
| `resolver.test.ts:76-80` 是占位测试 | 原文：`test("BC adapter getBlockHistory is UNSUPPORTED ...", async () => { assert(true); void CmbError; })`。全仓正则扫描 `assert(true|1|"|!false|null)` **仅命中这一处** | **确认** |
| `SEC-7` 用了上游不存在的 `contentType='system'` | `05-BILLION-CONTEXT-AUDIT.md:1073` 明确：「**没有 `"system"`、没有 `"tool"`** —— system/tool 的区分在 `role` 上，不在 `contentType` 上」。测试确实构造了一个不可达形态 | **确认，且后果比主代理写的严重得多（见 2.2）** |
| 全部用 `:memory:` | `new BridgeLedger(":memory:")` 共 **16 处**，文件路径 0 处（`perf-run.ts` 除外，它不在套件内） | **确认** |

### 2.2 主代理漏掉的关键一条：SEC-7 不是「弱测试」，是「假安全」

探针（`_agentD_probe/probe2.ts`，直接驱动真实引擎，`VERIFIED_RUNTIME`）：

```
role=system    contentType=system   evidence_written=0  reasons={"UNSUPPORTED_MESSAGE_TYPE":1}  SYSTEM_PROMPT_LEAKED=false
role=system    contentType=text     evidence_written=1  reasons={}                               SYSTEM_PROMPT_LEAKED=true
role=system    contentType=message  evidence_written=1  reasons={}                               SYSTEM_PROMPT_LEAKED=true
role=assistant contentType=system   evidence_written=0  reasons={"UNSUPPORTED_MESSAGE_TYPE":1}  SYSTEM_PROMPT_LEAKED=false
```

成因链（`VERIFIED_STATIC`）：
1. `source-resolver.ts:113-123` 只过滤 `content_type === "reasoning"`，**不看 `role`**；
2. `integrity-validator.ts:94-95` 只排除 `content_type === "reasoning" | "system"`，**不看 `role`**；
3. 真实适配器 `bc-source.ts:214-215` 的 `excluded_reason` 也**只对 `contentType === "reasoning"` 置位**，`role: String(r.role ?? "")` 读进来后无任何下游使用；
4. 上游 `role` 的取值域是 `user|assistant|system|tool`（`05:1039`），`kind` 取值域是 `text|toolCall|toolResult|reasoning`（`05:1073`）。

→ **真实的 system prompt（`role='system'`、`content_type='text'`）在当前链路里会一路通过校验并写成 Evidence。** SEC-7 之所以"通过"，只是因为它恰好构造了唯一会被拦下的那个组合。

这是对 §7-D「System Prompt 永不成为 evidence」的**实质反例**，且属于测试给出了错误信心（false assurance），比"测试写得弱"更严重。**建议：`SEC-7` 必须改为 `role="system"` + `contentType="text"`，并以当前代码预期失败（即该测试应当 RED）。**

**D-4（新增，高危）**：主代理 §2 未发现此点，仅判定「contentType 不存在」。需要升级为独立缺陷项（代码缺陷 + 测试缺陷双重）。

### 2.3 恒真 / 弱断言扫描（全量，不止主代理列的 1 条）

- 恒真：**仅 1 处** `assert(true)`（确认主代理）。`assert(1)`/`assert("x")`/`assert(!false)` **0 处**。
- **D-5（新增，中低危）**：`phase6-regression.test.ts:49`
  ```ts
  assert(r.counts.failed === 0 || r.counts.duplicate_hits >= 1, "recovery via dedupe, not failure");
  ```
  这是**析取式**断言：`failed=5 且 duplicate_hits=1` 也通过。而该场景 `duplicate_hits` 恒为 1，故该行**在任何部分失败下都会绿**。属弱断言，不是恒真但接近。
- **D-6（新增，中低危）**：`security.test.ts:84-93`（SEC-5）
  ```ts
  for (const ev of sink.snapshot()) { assert(ev.tenant_id !== null ...); }
  ```
  **没有前置非空断言**。若回归导致 evidence 数为 0，循环体一次不执行，测试**空集通过**（vacuous truth）。实测当前该场景写入 1 条，所以断言确实执行过 —— 但测试本身不设防。
- 宽松阈值：`pipeline.test.ts:43` `assert(report.counts.evidence_written >= 8, ...)`、`:77` `blocks_scanned >= 5`、`:97` `skipped >= 4` —— 用下界代替精确值，会漏掉"多写/少写若干条"的回归。属可接受但不严格。
- 自证（从实现反推期望）：`fi.test.ts:86` `assertEq(r1.failure_reasons["SINK_WRITE_FAIL"], 3)`、`:148` `assertEq(r2.counts.duplicate_hits, 2)`、`validator.test.ts:38/46` 用 `JSON.stringify({id,role,text})` 现造 blob 再断言哈希 —— 均为「夹具与断言同源」。其中 **validator 的 blob 格式与上游真实字节不符**（`05:3818` 段：真实 `content.bytes` 经过 `splitMediaContent()`，`imageBase64` 被替换为 `{"$media":"<sha256>"}`），见 D-9。

### 2.4 四条「:memory: 结论」到底证明了什么

| 测试 | 名称声称 | 在 `:memory:` 下实际证明 | 未证明 |
|---|---|---|---|
| `FI-3(c)` `fi.test.ts:190-201` | 「Evidence 已写、Ledger 未更新」的中断/补账 | 同进程内新建空 ledger + **复用同一个 `MockEvidenceSink` 实例** → sink 查重生效、message_ledger 被回填 | 跨进程：进程死了 `MockEvidenceSink` 的内存 Map 一起消失，**没有任何一层能证明重启后不重复写**。真实 sink 的重启去重能力 = **UNVERIFIED** |
| `FI-4` `fi.test.ts:215-242` | 「三连扫描一致」 | 同一 `engine` 实例、同一内存 ledger、同一内存 sink 连续调用 3 次 | 三次调用之间无持久化边界；flush/重开/崩溃窗口全未触及 |
| `RG-B1` `phase6-regression.test.ts:33-51` | 「崩溃残留恢复」 | **手工 `ledger.upsertBlock({...row, status: residue})`** 制造一行残留，再跑一轮 | 不是崩溃：没有半写状态、没有 WAL、没有事务中断。它证明的是「状态机复位路径正确」，这点价值成立（我的探针独立复现了 4 种残留都能收敛 COMMITTED） |
| 集成 C `pipeline.test.ts:146-162` | 「ledger-loss recovery」 | 新建空 ledger + 复用 `first.sink` → 全量重扫、写入全部命中 duplicate | 同上：ledger 丢失是**构造出来的空实例**，不是文件被删/损坏；sink 未重启 |

**结论（与主代理一致，但我要补一句更准确的表述）**：这 4 条**不是无效测试**——它们有效验证了「复位逻辑 / 补账逻辑 / 幂等逻辑」这三段**纯逻辑**；真正没被验证的是「**持久化边界**」，而 `BridgeLedger` 的持久化恰好**是好的**（见 2.5）。所以缺口在"测试没测"，不在"代码坏了"，但对外主张「崩溃恢复已验证」**不成立**。

### 2.5 主代理 §2.1「文件 ledger 重启后 allBlocks=1」—— 我用两个独立进程确认

```
=== 进程 A（写文件 ledger）===
PID process A wrote: evidence=1, blocks=1

=== 进程 B（全新 node 进程读同一文件）===
process B (fresh) sees block_ledger rows = 1 [{"block_id":"x1","status":"COMMITTED","evidence_count":1}]
process B sees run_ledger rows = 1
```

**确认主代理结论**：`BridgeLedger`（`node:sqlite` `DatabaseSync`）的落盘与重开是**真实可用**的；缺的是测试从未用文件路径跑过 `ReconciliationEngine`。`VERIFIED_RUNTIME`。

### 2.6 其它检查项

| 检查项 | 结果 |
|---|---|
| 测试间共享模块级可变状态 | **无**（确认）。唯一的全局副作用是 `security.test.ts:152-164` 改 `process.env.BILI_ENCRYPTION_KEY`，已 `try/finally` 还原 |
| 依赖执行顺序 | **无**（每 test 自建 fixture；同文件内亦无跨 test 依赖） |
| `Math.random()` 在测试套件内 | **0 处**（确认主代理）；仅 `perf-run.ts:105` 与 `evidence-sink.ts:46`（`failRate>0` 时才触发，测试中 `failRate` 恒 0） |
| skip / todo 测试 | **0 处**（无 `test.skip`/`.todo`/`xit`/`describe`） |
| 未被 runner 调用的测试文件 | `tests/performance/perf-run.ts` 不是 `.test.ts` 且默认根不含 `tests/performance` → **永不参与 `npm test`**（确认；且它 `writeFileSync` 覆写报告，本就不该混进套件） |
| 断言总量 | 283 条 `assert/assertEq/assertThrows` |

---

## 三、B-1 修复彻底性独立判断

### 3.1 状态机转移矩阵（`_agentD_probe/probe.ts`，逐对实跑 `transition()`，`VERIFIED_RUNTIME`）

```
PARTIAL     -> FAILED : LEGAL        PARTIAL     -> DISCOVERED : ILLEGAL
READY       -> FAILED : LEGAL        READY       -> DISCOVERED : ILLEGAL
RESOLVING   -> FAILED : LEGAL        RESOLVING   -> DISCOVERED : ILLEGAL
VALIDATING  -> FAILED : LEGAL        VALIDATING  -> DISCOVERED : ILLEGAL
UNSUPPORTED -> FAILED : ILLEGAL      UNSUPPORTED -> DISCOVERED : LEGAL
COMMITTED   -> FAILED : ILLEGAL      COMMITTED   -> DISCOVERED : ILLEGAL
MISSING     -> FAILED : LEGAL        MISSING     -> DISCOVERED : LEGAL
CORRUPTED   -> FAILED : LEGAL        CORRUPTED   -> DISCOVERED : LEGAL
FAILED      -> FAILED : ILLEGAL      FAILED      -> DISCOVERED : LEGAL
```

对照 `domain.ts:157-170`：**完全一致**。

### 3.2 复位段（`reconciliation-engine.ts:128-144`）逐状态推演

条件：`if (status !== "FAILED" && status !== "MISSING" && status !== "CORRUPTED") status = transition(status,"FAILED")`，随后 `transition(status,"DISCOVERED")`。

| 进入复位段的状态 | 走 `→FAILED` 吗 | 合法性 | 结果 | 判定 |
|---|---|---|---|---|
| PARTIAL | 是 | PARTIAL→FAILED ✓ | FAILED→DISCOVERED → 重扫 | **已修** ✓ |
| READY | 是 | ✓ | 恢复 | **已修** ✓ |
| RESOLVING | 是 | ✓ | 恢复 | **已修** ✓ |
| VALIDATING | 是 | ✓ | 恢复 | **已修** ✓ |
| FAILED | 否（被排除） | — | FAILED→DISCOVERED ✓ | **正确**（排除是对的：直连合法） ✓ |
| MISSING | 否 | — | MISSING→DISCOVERED ✓ | **正确** ✓ |
| CORRUPTED | 否 | — | CORRUPTED→DISCOVERED ✓ | **正确** ✓ |
| **UNSUPPORTED** | **是** | **`UNSUPPORTED→FAILED` 非法** | 抛错 → catch → `STATE_RESET_FAIL` + `failed++` + `continue` | **未修（B-1 不彻底）** ✗ |
| **COMMITTED（version 漂移）** | **是** | **`COMMITTED→FAILED` 非法** | 同上 | **未修** ✗ |

> 关于任务书里那句提示「`status !== "FAILED" && !== "MISSING" && !== "CORRUPTED"` 跳过了哪些状态的 FAILED 中转，会不会漏」——**这里主代理与我的判断一致：排除这三态是正确的，不是漏**。因为 `MISSING/CORRUPTED/FAILED` 的 `→DISCOVERED` 直连本来就合法，多一跳反而会因 `FAILED→FAILED` 非法而崩。真正漏的是**不在排除名单里、但也不允许 `→FAILED` 的 `UNSUPPORTED` 和 `COMMITTED`**。

### 3.3 探针实测（原文）

```
=== 1. UNSUPPORTED 残留（FI-7 之后第二次扫描）===
  run1: unsupported=1 failed=0 status=UNSUPPORTED
  run2: unsupported=0 failed=1 status=UNSUPPORTED reasons={"STATE_RESET_FAIL":1}
  run3: failed=1 reasons={"STATE_RESET_FAIL":1}
  >>> UNSUPPORTED 块在 3 次扫描后的台账状态 = UNSUPPORTED（未复位、未重扫、每轮多计 failed）

=== 2. COMMITTED 残留 + block_version 漂移 ===
  run2: failed=1 skipped=0 reasons={"STATE_RESET_FAIL":1}
  >>> 台账状态仍 = COMMITTED（version 漂移块永久卡死，且每轮计 failed）

=== 3. MISSING / CORRUPTED / FAILED 残留（应为合法直连复位）===
  MISSING 残留    → 台账 COMMITTED, failed=0, reasons={}
  CORRUPTED 残留  → 台账 COMMITTED, failed=0, reasons={}
  FAILED 残留     → 台账 COMMITTED, failed=0, reasons={}

=== 4. 永久失败块的重试上限 ===
  5 轮扫描中该块被重复处理 5 次，台账 evidence_count=0，无退避/无上限
```

### 3.4 对主代理 §4 的判定

| 主代理结论 | 我的独立判定 |
|---|---|
| PARTIAL/READY/RESOLVING/VALIDATING 已修 | **确认（VERIFIED_RUNTIME）** |
| 复位失败不再击穿 `run()` | **确认**：run2/run3 均正常返回 |
| UNSUPPORTED 未修、永久卡死 | **确认（VERIFIED_RUNTIME）**，并补充：卡死时**台账不落任何痕迹**（`catch` 分支只 `continue`，不调 `persistBlockRow`），运维只能从 `failure_reasons.STATE_RESET_FAIL` 反推 |
| N-1 `counts.unsupported` 被污染 | **确认**：run1 `unsupported=1` → run2 `unsupported=0, failed=1`。同一块在两轮里的"身份"变了 |
| N-2 永久失败无退避/无上限 | **确认（VERIFIED_RUNTIME）**：5 轮全重试；`cfg.retry.max_attempts=3` 全仓零引用（`VERIFIED_STATIC`，grep 仅命中 `config.ts`） |
| （主代理未在 12 展开）COMMITTED 漂移死锁 | **D-7（补充）**：`reconciliation-engine.ts:112` 的幂等跳过要求 `status==="COMMITTED" && block_version===block.block_version`。**version 变了就掉进复位段 → `COMMITTED→FAILED` 非法 → 死锁**。触发条件：`covered_hash` 不变但 `payload` 变（`block_version=sha256(payload)` 与 `covered_hash` 上游是两列，可不同步）。若 `covered_hash` 也变，则 `bridge_key` 变 → `ledgerRow===null` → 正常走 DISCOVERED，不死锁。**这是窄窗口但真实存在，且每轮白烧一次 `failed`** |

### 3.5 「12 条回归测试」的核对

**实际是 11 条**（`phase6-regression.test.ts`，运行时 11 个实例）：

| 来源 | 条数 | 覆盖 |
|---|---|---|
| RG-B1（循环 ×4） | 4 | PARTIAL / READY / RESOLVING / VALIDATING 残留 → COMMITTED ✓ |
| RG-B1b | 1 | REVOKED 终态短路 ✓ |
| RG-M1 / M1b / M1c | 3 | 群绑定欺骗 / 一致通过 / 无绑定拒绝 ✓ |
| RG-M5 / M5b | 2 | speaker 强制 / tool 豁免 ✓ |
| RG-E2 | 1 | blob 读不到 → HASH_MISMATCH ✓ |
| **合计** | **11** | **B-1 只覆盖 4 种状态，UNSUPPORTED / COMMITTED 漂移均无测试** |

`PHASE6-CHANGES.md:44` 自己写的是「55 基线 + 11 新增 = 66」→ **11 是对的，任务书口径里的"12 条"不成立**。66 + 后面新增的 `security.test.ts`(12) = 78 ✓ 与实测吻合。

**D-8（纠正，低危）**：主代理 §1 表中 phase6 记 7 条、正文又称"12 条回归"，两个数都不对；正确值是 **11**。

---

## 四、性能数字独立复核

### 4.1 我的复跑（三种状态，全部贴原文）

**(a) 原型目录直接跑（DB 已存在）** —— 这是任何"第二次跑"的人会看到的结果：

```
=== PERF small: 10 sessions × 100 msgs ===
fixture build: 6ms, rerun: 15ms (written=0 skipped=50)
{"spec":"small","sessions":10,"messages":1000,"blocks":53,
 "evidence_written":0,"duplicate_hits":0,"skipped":50,
 "failed":0,"missing":3,"corrupted":0,"unsupported":0,
 "duration_ms":23,"throughput_msgs_per_s":43478,
 "avg_block_ms":0.0476,"p95_block_ms":0.0997,
 "peak_rss_mb":61,"second_run_skipped":50,"second_run_written":0}
```

**(b) 隔离副本 + 纯净 DB（第一次）**：

```
fixture build: 7ms, rerun: 9ms (written=0 skipped=50)
{"blocks":51,"evidence_written":340,"skipped":0,"failed":0,"missing":1,
 "duration_ms":930,"throughput_msgs_per_s":1075,
 "avg_block_ms":18.398,"p95_block_ms":22.474,"peak_rss_mb":63}
```

**(c) 隔离副本 + 纯净 DB（第二次，另一个随机种子）**：

```
fixture build: 9ms, rerun: 15ms (written=0 skipped=50)
{"blocks":53,"evidence_written":340,"skipped":0,"failed":0,"missing":3,
 "duration_ms":899,"throughput_msgs_per_s":1112,
 "avg_block_ms":17.698,"p95_block_ms":22.148,"peak_rss_mb":63}
```

**(d) 同一干净目录连跑第二次**：`duration_ms:12, throughput_msgs_per_s:83333, evidence_written:0`。

### 4.2 与 PERF-REPORT.md 的差异：`evidence_written` 对不上，且不可能对上

| 指标 | PERF-REPORT Small | 我的纯净复跑 | 差异 |
|---|---|---|---|
| blocks | 51 | 51 / 53 | 随机，见 4.6 |
| **evidence_written** | **500** | **340 / 340** | **−32%** |
| duration_ms | 1,180 | 930 / 899 | 更快 |
| throughput | 847 | 1,075 / 1,112 | 更高 |
| missing | 1 | 1 / 3 | 随机 |
| peak_rss_mb | 64 | 63 / 63 | ≈ |

**340 不是抖动，可以精确推导（`VERIFIED_STATIC` 算术 + 实测吻合）**：
- 覆盖消息数 = 5 个块 × 10 条 × 10 会话 = **500**；
- 其中 `role === "assistant"` 的消息：每个覆盖区间 10 条里 `mi%3===1` → 区间 0-9:3、20-29:3、40-49:**4**、60-69:3、80-89:3 = **16/会话 → 160 条**；
- `perf-run.ts:78`：`speaker_id: role === "user" ? PERF-U… : null` → **assistant 消息 speaker 恒为 null**；
- M-5 修复（`integrity-validator.ts:69-73`）后 speaker 对 user/assistant **无条件强制** → 这 160 条全部 `SPEAKER_UNRESOLVED` 被拒；
- **500 − 160 = 340**，与两次实测**逐位一致**。

→ **`PERF-REPORT.md` 的 Small/Medium/Large evidence 数字（500 / 50,000 / 500,000）是在 M-5 修复之前采集的，Phase 6 之后从未重跑。**

**D-9（高危，纠正）**：`13-PERFORMANCE-REVIEW.md:118-132` §7 声称「主代理实跑 Small」并给出
`evidence_written:500, duration_ms:1180, throughput:847, avg_block_ms:23.38, p95:26.18`
—— 这 5 个数字与 `PERF-REPORT.md` **逐字相同**，而在当前代码下 **evidence_written=500 不可达（上限 340）**。同段还写着「与报告一致。**可复现 ✅**」，紧接着又说「严格说性能数据不可完全复现」—— 自相矛盾。
**判定：该段不是本次复跑的真实输出，属照抄旧报告。**（若确有其事，唯一可能是运行在 Phase 6 修复前的代码快照上，但那与"复跑当前 HEAD"的表述不符。）

### 4.3 吞吐口径（主代理方向对，但辩解错了一半）

- `msgCount = sessions × messagesPerSession` = **总消息数**，不是处理数 —— **确认**（`perf-run.ts:150,187`）。
- 块覆盖 ≈ 50%，因此按"实际成为候选的消息"计，吞吐高估约 **2 倍** —— **确认**。
- **D-10（纠正）**：主代理 §2 的缓和论据「若把吞吐理解为'单位时间扫描的消息量'（含跳过），报告口径是合理的（它确实遍历了全部消息找覆盖）」—— **不成立**。`SourceResolver.locateMessages`（`source-resolver.ts:145-148`）在 `start_seq/end_seq` 非空时只 `getMessages(startSeq, endSeq)`，**未被任何块覆盖的那 50% 消息从未被读取**。正确写法：`1000/0.899 = 1112 "msgs/s"` 实际对应 **500 条被解析消息 + 340 条被写入**，即真实解析吞吐 ≈ **556 msg/s**。

### 4.4 「幂等重扫快 194 倍」—— 主代理说"语义误导"是对的，但事实前提错了

主代理 §3 写：「全部 20,000 块命中 `COMMITTED` → `continue`；**不调用 `listBlocks` 之外任何 store 方法**」。

用 `PERF-REPORT.md` 自己的数字即可证伪：**总块数 21,763，二次扫描 `skipped=20,000`** → 有 **1,763 个块没有跳过**，而 `missing` 恰好 **= 1,763**（`PERF-REPORT.md:56-58,65`）。MISSING 不是 COMMITTED，不满足 `reconciliation-engine.ts:112` 的跳过条件 → 这 1,763 块**每轮都完整重跑** `listBlocks → resolveBlock → getMessages(0..MAX) → validate → persist`。

定量：若按 Medium 的 `2,000 块/9ms`（≈4.5µs/块）推算，20,000 次跳过 ≈ **90ms**；剩下 **13,691ms / 1,763 块 ≈ 7.8ms/块** —— 正好是一次"全会话 1,000 条消息扫描"的量级。**13.8s 的重扫耗时几乎全部来自这 8% 的永久失败块，而不是"台账查询"**。

**这条结构性事实在 Medium 上更好验证，且暴露出 PERF-REPORT 自身的矛盾**：Medium 总块数 2,136、`skipped=2,000`、`missing=136`（`PERF-REPORT.md:38,44`），同样满足 `2,136 − 2,000 = 136 = missing`。而报告对**同一次二次扫描**给出了两个互斥的耗时：

- `:44` 「二次扫描 … 9 ms 内完成重扫」
- `:49` 「幂等跳过路径（二次扫描）比首次扫描快 3 个数量级（**405ms** vs 128s）」

`perf-run.ts:194` 只打印**一个** `rerun: ${rerunMs}ms`，不可能同时是 9ms 和 405ms。136 个未跳过块各做一次全会话扫描，量级上只能是 **405ms** 那一边；**「9 ms」是错误值**（很可能是把某次空跑或 warm 缓存的跳过路径当成了完整重扫）。**D-15（新增，中危）**：13 号 §3 直接采信了「9 ms」叙事（"重扫 ≈ 遍历块列表 + 查 20,000 次 ledger"），因此其"只做了一次台账查询"的心智模型从根上就不对。

**因此"194 倍"有两重问题**：
1. 分子分母不同质（完整对账 vs "92% 跳过 + 8% 全量重做"）；
2. 它掩盖了一个**真实的性能缺陷**：永久失败块在 1M 规模下每轮固定消耗 ≈13.7s，永不收敛（与 §3.4 的"无重试上限"是同一个病）。

**D-11（纠正，中危）**：主代理 §3 的事实陈述错误（"全部 20,000 块"、"不调用其他 store 方法"）。结论（"数值正确、语义误导"）保留，但理由需替换。

### 4.5 峰值 RSS

- `process.memoryUsage.rss()` 在 `perf-run.ts` 中**恰好出现 2 次**（`:164` 前、`:168` 后）—— **确认主代理"2 次采样"**。
- 且 `rssBefore` 是在 `buildPerfFixture()` **之后**取的，1M 消息对象已在基线里；`Math.max(before, after)` 在 GC 回收后只会取到"更大的那个"，**真实峰值未知**。→ **确认为下限估计**。
- 1,160MB 主要是夹具常驻内存（`MOCK_ONLY`），不代表引擎开销。

### 4.6 `batch_size` / `concurrency` / `retry` 是否生效

全仓 grep（`src/**/*.ts`）：三者**只在 `config.ts` 出现**（类型、默认值、`validateConfig`），引擎 `reconciliation-engine.ts` **零引用**。

→ **确认主代理 §6**：`perf-run.ts:162` 传的 `batch_size: 500` 是**装饰性参数**，对性能数字没有任何影响；`concurrency` 恒 1（串行 `for...of`）；`retry` 未实现。**D-12（补充）**：这意味着 perf 报告里的"批量/并发"想象空间全部不成立，性能曲线是**纯单线程**的。

### 4.7 `Math.random()` → 可复现性

`perf-run.ts:105`：`if (bi % 2 === 0 && Math.random() < spec.nestedRatio && ...)` 生成嵌套父块。实测块数：**51 / 53 / 53**（三次运行）。计划值是每会话 5 块 ×10 = 50，多的就是随机嵌套父块。

→ **确认主代理**："每次运行 fixture 不同"。补一句更硬的：**连"块总数"这个一阶指标都不稳定（±4%）**，且它直接进入 `ops/块`、`P95` 等派生指标 → 跨运行对比必须说明种子，而代码**没有固定种子**。

另：`fixture build` 耗时被单独计时（**不**计入 `runMs`），但 1M 消息的构造成本（Large 3,283ms）是真实存在的；报告未把它计入总耗时，属口径选择（可接受，但需说明）。

### 4.8 `perf-run.ts` 复用旧 DB —— 主代理完全没提（D-13，新增，高危）

`perf-run.ts:161`：`new BridgeLedger(\`data/test-only/perf-${spec.name}.db\`)` —— **固定路径，运行前不删除**。

实测后果见 4.1(a)：在已有 DB 的目录里直接跑，`evidence_written=0, skipped=50, duration=23ms, throughput=43,478 msgs/s`。也就是说：
- 任何人"复跑"这个基准，第二次起测的都是**纯跳过路径**，数字虚高 **40–80 倍**；
- 这也解释了为什么主代理要拿到 500 必须先有干净 DB —— 而他的报告里没有说明这一点；
- 同一个缺陷还让 `reports/perf-results.json` 被我的一次运行静默覆写（见副作用声明），**基准产物不可追溯**。

**建议**：`perf-run.ts` 应在构造 ledger 前 `rm -f` 目标文件，或在文件名里带时间戳/run_id，并把 `fixture_seed` 写进结果 JSON。

### 4.9 机器配置（主代理 §8 判定过强）

主代理称 `Get-CimInstance Win32_Processor` 被拒 → "CPU 型号、核心数、RAM 容量无法获取"。
我复现：CIM 同样 `PermissionDenied 0x80041003`（**确认该路不通**）。但另有两条路可拿到部分信息：

```
NUMBER_OF_PROCESSORS=20
PROCESSOR_IDENTIFIER=Intel64 Family 6 Model 154 Stepping 3, GenuineIntel
HKLM:\HARDWARE\DESCRIPTION\System\CentralProcessor\0\ProcessorNameString
  → 12th Gen Intel(R) Core(TM) i9-12900H
```

**D-14（纠正，低危）**：应记为「CPU: i9-12900H / 20 逻辑处理器；RAM 容量未能采集」，而不是「无法获取」。这对性能数字的可比性有实际意义（单核性能档位已知）。

### 4.10 性能可信度：一句话

**这些数字是"内存 Map 遍历 + SQLite ledger 写入"的真实测量，不是"真实 SQLite 查询 + CAS blob 读取"的性能**；方向上可看复杂度趋势，数值上既不可比也不可复现（随机夹具 + 复用 DB + 口径高估 2 倍），**不能用于容量规划，也不能作为"支持 100 万消息"的验收证据。**

---

## 五、「78 条测试逃过 6 个 BLOCKER」—— 我的判断

**结论：约 2/3 是「测试写得不好」（本可避免），1/3 是「当前阶段只能这样」（环境所限）。不是"只能这样"。**

**属于"写得不好"（本可避免，改动量极小）**：
1. `RG-B1` 的循环数组加一个 `"UNSUPPORTED"`（**改 1 个词**）就能抓住 B-1 残留缺口；
2. `SEC-7` 把 `contentType:"system"` 改成 `role:"system" + contentType:"text"`（**改 1 行**）就能抓住 system prompt 泄漏 —— 这是本次复核里最刺眼的一条：测试名与测试内容**指向不同的东西**，于是给出了反向的安全保证；
3. 任意一条测试用**文件路径 ledger + 两个进程**（约 15 行）就能覆盖"跨进程幂等"，而这是本项目 ADR-007 的核心主张；
4. `perf-run.ts` 加 `rm -f` + 固定随机种子（2 行）就能让基准可复现。

**属于"只能这样"（结构性/环境限制）**：
1. 真实 `billion-context` 上游库不可得 → 所有上游集成只能 `MOCK_ONLY`；
2. 真实 Memory Core sink 契约（B-1 schema 冲突）未解决 → `MemoryCoreEvidenceAdapter` 故意全抛 `SINK_UNSUPPORTED`，无法测真写；
3. 真实跨进程崩溃（kill -9 / WAL 半写）在单测框架里本就难做，但**"两个进程 + 文件 ledger"这条中间路线是可行的且没做**。

**根因不是"用例太少"，而是"夹具与实现同源"**：`MockSourceStore`/`fi-fixture`/`fv` 的字段语义是**照着实现写的**，于是"实现怎么想，夹具就怎么造"，测试只能验证内部自洽。真正能抓 BLOCKER 的是**异源数据**（上游真实 blob 字节、上游真实 role/kind 组合、真实文件库、真实第二进程），而这四样里有两样其实**现在就能做**。

**量化**：78 条中 0 条跨进程、0 条真实 SQLite 文件、0 条真实 CAS、1 条占位（`assert(true)`）、≥3 条弱断言（析取式/空集量化/宽阈值）、1 条构造了不可达输入并因此静默放行真实泄漏。**强有效用例下界 ≈ 73。**

---

## 六、与主代理 12 / 13 的差异清单（共 15 项）

| # | 位置 | 主代理说法 | 我的独立结论 | 判定 |
|---|---|---|---|---|
| D-1 | 12 §1 / inventory | 74 静态 + 4 循环 = 78；phase6 = 7 | 静态 **75**（含循环内 1 处）；phase6 = **8 点/11 实例**；78 正确但分解错 | 纠正（低） |
| D-2 | inventory | `files[].count` 合计 = 78 | 实际合计 **74**，与 `total`/`summary` 冲突 | 纠正（低） |
| D-3 | 12 §3.5 | ESM 缓存"可能跳过文件"，但结论是"无问题" | 方向对；另发现 `runner.ts:67` import 在 try 外，单文件坏 → 全体 0 报告/exit 2 | 补充（中） |
| D-4 | 12 §2 | SEC-7 只是"用了不存在的 contentType" | 升级为**真实泄漏**：`role=system + contentType=text` → evidence_written=1（实测） | 新增（高） |
| D-5 | 12 §3 | 只有 1 条无效测试 | 另有 `RG-B1:49` 析取式断言（部分失败也绿） | 补充（中低） |
| D-6 | 12 §3 | （未提） | `SEC-5` 无前置非空断言 → 空集通过 | 补充（中低） |
| D-7 | 12 §4 | （未在 12 展开） | `COMMITTED + block_version 漂移` → `STATE_RESET_FAIL` 死锁，机制与实测均有 | 补充（中） |
| D-8 | 12 §1 | 正文称"12 条回归测试" | 实际 **11** 条（`PHASE6-CHANGES.md:44` 也是 11） | 纠正（低） |
| D-9 | 12 §2 表 | `validator.test.ts` "锁定了错误的 content_hash 语义（违反 13 §7.2-4）" | 与 `05` 号审计「`content_hash` = sha256(`content.bytes`)」**一致**，测试语义正确；真正的弱点是**夹具 blob 格式与上游字节不符**（缺 `$media` 替换），不是语义写反 | 纠正（中） |
| D-10 | 13 §7 | 实跑 Small：500/1180/847/23.38/26.18，"与报告一致 ✅ 可复现" | 当前代码 **evidence 上限 340**（实测 340/340）；该段与旧报告逐字相同，**不是本次复跑输出** | 纠正（高） |
| D-11 | 13 §2 | "若理解为扫描消息量则口径合理（遍历了全部消息）" | 未覆盖的 50% 消息**从未被读取**，不存在"遍历" | 纠正（中） |
| D-12 | 13 §3 | "全部 20,000 块跳过，不调用其他 store 方法" | 21,763 块中 **1,763 块（=missing）每轮全量重跑**，占重扫耗时 ≈99% | 纠正（中） |
| D-13 | 13 全文 | （未提） | `perf-run.ts:161` **固定 DB 路径且不清理** → 第二次跑测的是纯跳过（实测 0 written / 43,478 msgs/s） | 新增（高） |
| D-14 | 13 §8 | "CPU/核心/RAM 无法采集" | CPU = **i9-12900H / 20 逻辑处理器**可获取（注册表+环境变量）；仅 RAM 拿不到 | 纠正（低） |
| D-15 | 13 §3（采信 PERF-REPORT:44） | 重扫「9 ms 完成」 | 同一报告 `:49` 同一次重扫写的是 **405ms**；136 个非跳过块决定了只能是 405ms 一侧 → "9ms" 是错误值 | 纠正（中） |

### 确认项（主代理对的地方，我不改）

- 78 / 0 failed / exit=0，三个分类数 35+20+11+12，连跑 3 次稳定 —— **逐项确认**。
- 全部测试用 `:memory:`（16 处），0 条真实 SQLite 文件 / 0 条真实 CAS / 0 条真实上游 —— **确认**。
- `BridgeLedger` 持久化实现本身正确，缺口在"测试从不用文件" —— **我用两个独立进程复现确认**。
- `assert(true)` 占位 1 处、`Math.random` 只在 `perf-run.ts`、无 skip/todo、无模块级共享状态 —— **确认**。
- B-1 对 PARTIAL/READY/RESOLVING/VALIDATING 已修、对 UNSUPPORTED 未修、复位失败不击穿 run() —— **逐条实测确认**。
- 无重试上限/退避、`retry` 零引用 —— **确认**。
- 峰值 RSS 仅 2 次采样、是下限 —— **确认**。
- `batch_size`/`concurrency`/`retry` 对引擎无效 —— **确认**。
- "194 倍"语义误导、吞吐口径高估约 2 倍、真实 SQLite+CAS 性能未测 —— **结论确认**（理由需替换，见 D-11/D-12）。

---

## 七、总体裁定

| 维度 | 12 号 | 我的裁定 |
|---|---|---|
| 数量与稳定性 | 78/0/exit 0 | **PASS（确认）**，分解口径需修（D-1/D-2/D-8） |
| 测试有效性 | 1 占位 → 77 有效 | **PASS_WITH_GAP**：77 是上界；另有 3 条弱断言；且有 1 条**假安全**测试（SEC-7，D-4） |
| 真实持久化覆盖 | 0 条 | **FAIL（确认）**，但底层实现是好的（跨进程已另证） |
| B-1 彻底性 | PASS_WITH_LIMITATION（UNSUPPORTED 遗漏） | **同意，并补 COMMITTED 漂移死锁**；准确表述应为**「B-1 的原始崩溃路径已消除，但复位段仍有两个不可达分支会永久卡死」** |
| 性能数字 | 真实但不代表真实系统 | **同结论，但 13 号 §7 的"复跑"不可信（D-10）**；另新增"复用 DB"这一可复现性硬伤（D-13） |

### 给主代理的返修建议（按优先级）

1. **P0** — 修 `SEC-7` 为 `role:"system"`，并把它标成当前 RED（它暴露的是真缺陷，不是测试问题）。
2. **P0** — 13 号 §7 换成真实复跑输出（340 / 899–930ms / 1,075–1,112），或明确写"引用旧报告，未复跑"。
3. **P1** — 12 号 §4 补 `COMMITTED + block_version` 分支；`RG-B1` 加 `UNSUPPORTED`。
4. **P1** — `perf-run.ts` 加 DB 清理 + 固定种子；13 号补"重扫 = 92% 跳过 + 8% 全量重做"的事实。
5. **P2** — 修 `test-inventory.json` 的 `count` 口径；12 号 §1 的 74→75、7→11。
6. **P2** — `runner.ts` 把 `await import(spec)` 包进 try/catch（单文件坏了也要有汇总）。
