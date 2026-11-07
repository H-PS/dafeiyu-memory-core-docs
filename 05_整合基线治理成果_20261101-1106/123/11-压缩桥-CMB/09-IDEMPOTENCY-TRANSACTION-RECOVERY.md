# 幂等、事务与故障恢复审查（09-IDEMPOTENCY-TRANSACTION-RECOVERY）

采集：2026-11-06 ｜ 主代理直接读码 + 实测探针

---

## 1. 三层幂等键的数学定义

| 层 | 定义 | 位置 | 分隔符 | 碰撞风险 |
|---|---|---|---|---|
| **Block** | `sha256(session_id \| block_id \| covered_hash)` | `bridge-ledger.ts:273` | `\|` | 低（sha256 定长） |
| **Message** | `(session_id, message_id)` 拼接为 `session_id\|message_id` | `bridge-ledger.ts:281` | `\|` | **中**：若 session_id 或 message_id 含 `\|` 则歧义 |
| **Evidence** | `(content_hash, source_locator)` 拼接为 `content_hash\|source_locator` | `evidence-adapter.ts:75` | `\|` | **高**（见 §2） |

---

## 2. Evidence 幂等键的三个致命问题

### 2.1 问题一：分隔符歧义（未验证但存在）

`evidenceIdentity = ${content_hash}|${source_locator}`

- `content_hash` 是 64 位十六进制 → **不可能含 `|`**
- `source_locator` = `${session_id}:${seq}` → 若 `session_id` 含 `|`（不太可能但无约束）

**判定**：低风险，但无输入校验。

### 2.2 问题二：source_locator 撞键（**已实测确认**）

```
session_id = "SESS-fi-private", seq = 0
→ locator = "SESS-fi-private:0"

（会话重建后）
session_id 仍为 "SESS-fi-private", seq 又从 0 开始
→ locator = "SESS-fi-private:0"    ← 相同！
```

但由于 `content_hash` 不同（内容不同），`(content_hash, locator)` **不相等** → sink 接受为新 Evidence。

**实测**：
```
after run1, sink: SESS-fi-private:0=旧内容A
run2: {"written":1,"dup":0,"failed":0}
final sink: SESS-fi-private:0=旧内容A, SESS-fi-private:0=新内容B
```

→ **同一 locator 下两条 Evidence**，`evidence_id` 也相同（都是 `SESS-fi-private:0`）。

**影响**：
- Evidence 身份不再唯一
- `revokeScan` 用 `(content_hash, locator)` 只能找到其中一条
- 下游若以 `evidence_id` 为键会覆盖

### 2.3 问题三：content_hash 语义错误（**已确认**）

按权威文档 `13-Evidence与EmotionContext.md:255`：
> `content_hash` **MUST** 为 `original_text` 的 sha256

CMB 用的是 blob 哈希。后果：
- **去重失效**：同一句话在不同 session 下 blob 不同 → 不认为是同一内容
- **校验失效**：无法用 content_hash 验证 original_text

---

## 3. 写入与 Ledger 的一致性

### 3.1 是否同一事务？

**否。** Evidence Sink 与 Bridge Ledger 是两个完全独立的组件：
- Sink：`MockEvidenceSink`（进程内 Map）或 `MemoryCoreEvidenceAdapter`（抛异常）
- Ledger：独立的 SQLite 文件/`:memory:`

代码顺序（`reconciliation-engine.ts:244-279`）：
```ts
const res = await this.sink.write(ev);              // ① 写 sink
if (res.duplicate) duplicates++;
else written++;
const back = await this.sink.getByIdentity(...);    // ② 验证
if (back === null) throw new Error("VERIFY_FAIL");  // ③ 验证失败抛错
this.ledger.upsertMessage({...});                   // ④ 写 ledger
if (!res.duplicate) counts.evidence_written++;      // ⑤ 计数
```

**四个操作不在任何事务内**，任一步抛错都会中断后续。

### 3.2 不一致窗口分析

| 场景 | 结果 | 可恢复？ |
|---|---|---|
| ①成功，②失败 | sink 有数据，抛错进 catch，`writeFailures++` | 下一轮：sink 返回 duplicate，补写 ledger ✅ |
| ①成功，②成功，④失败 | sink 有数据，ledger 无行 | 下一轮：sink duplicate，ledger 补写 ✅ |
| ①成功，②成功，④成功，⑤前崩溃 | sink + ledger 都有，但 `evidence_written` 未计数 | 报告数字偏小（无害） |
| ①成功，进程崩溃 | sink 有数据，无 ledger | 下一轮 duplicate → 补账 ✅ |
| ①抛错 | 无写入 | 下一轮重试 ✅ |

**结论**：设计上是 **at-least-once + 幂等消费**，逻辑正确。

### 3.3 但"幂等消费"依赖 Sink 能力

`MockEvidenceSink.write()` 用内存 Map 去重。
**真实 Memory Core 是否有幂等写入能力？** —— `MemoryCoreEvidenceAdapter` 全抛异常，**无法验证**。

→ **跨系统幂等从未被证明**。

---

## 4. 崩溃恢复的真实性

### 4.1 全部测试用 `:memory:`（**已确认**）

16 处 `new BridgeLedger(":memory:")`，**0 处**使用文件路径。

**实测**：
```js
const run1 = new BridgeLedger(":memory:");
run1.upsertBlock({...}); run1.close();
const run2 = new BridgeLedger(":memory:");
run2.allBlocks().length  // => 0
```

→ 所谓"崩溃恢复"测试，实际是在**同一进程内**创建第二个空实例。

### 4.2 `BridgeLedger` 本身持久化正确

文件版实测：
```
文件 ledger 重启后 allBlocks: 1  ✅
```

→ 实现没问题，**测试方法有问题**。

### 4.3 真实崩溃会留下什么

`run()` 无 try/finally：
- `:73` `startRun` 写入 `end_time=NULL`
- 崩溃 → `:296` `finishRun` 永不执行
- → `run_ledger` 中永久存在 `end_time=NULL` 的行

**无任何机制清理或识别这些半开行。**

### 4.4 `listSessions` / `listBlocks` 无 try

`:66` `const sessions = await this.store.listSessions();` 在 try 之外。
源库异常（SQLITE_BUSY、损坏）→ `run()` 直接 reject → 无报告、`run_ledger` 半开。

---

## 5. 重试机制的缺失

`cfg.retry = { max_attempts: 3, backoff_ms: 50 }` 定义在 `config.ts`。
**全 `src/` 零引用。**

- 无自动重试
- 无退避
- 无重试上限
- 永久失败的块每轮都重试（含 `getBlob` 等 I/O）

**Large 基准实证**：21,763 块，若 1% 永久失败 = 217 块/轮无效重试。

---

## 6. 永久卡死状态汇总（已实测）

| 状态 | 卡死原因 | 后果 | 可恢复？ |
|---|---|---|---|
| `UNSUPPORTED` | 复位尝试 `UNSUPPORTED→FAILED` 非法 | 永不重评 | ❌ |
| `COMMITTED` + payload 变 | `COMMITTED→FAILED` 非法 | 块静默死亡 | ❌ |
| `REVOKED` | 终态无出边 | 永不复活 | ❌（设计） |
| `SUPERSEDED` | 终态无出边 | 永不复活 | ❌（设计） |
| `run_ledger.end_time=NULL` | 崩溃 | 永久半开 | ❌ |

---

## 7. 结论

| 项 | 判定 |
|---|---|
| Block 幂等键 | **PASS_WITH_LIMITATION**（covered_hash 会漂移） |
| Message 幂等键 | **PASS**（(session_id, message_id) 正确） |
| Evidence 幂等键 | **FAIL**（会撞键 + content_hash 语义错） |
| 写入顺序设计 | **PASS**（at-least-once + 幂等消费逻辑正确） |
| 跨系统幂等 | **UNVERIFIED**（真实 Sink 不可用） |
| 崩溃恢复 | **UNVERIFIED**（测试全用 :memory:） |
| 重试机制 | **FAIL**（cfg.retry 零引用） |
| 永久卡死 | **失败 4 处**（见 §6） |

### 一句话结论

**幂等设计在纸面上正确，但从未在跨进程、跨系统条件下被验证；且 4 条永久卡死路径已实测确认。**
