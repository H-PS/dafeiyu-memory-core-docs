# 删除、撤销与数据生命周期（10-LIFECYCLE-REVOCATION-DELETION）

采集：2026-11-06 ｜ 主代理读上游源码 + CMB 代码 + 实测

---

## 1. 上游生命周期事件的真实语义

| 事件 | 上游实现 | 是否可逆 | 证据 |
|---|---|---|---|
| **消息 tombstone** | `UPDATE messages SET retired_at = NOW()` | **可逆**（`SET retired_at = NULL`） | `store.ts:949` / `store.ts:925` |
| **媒体随 tombstone 释放** | `DELETE FROM message_media` + `releaseContent` | **不可逆**（refcount 归零即物理删） | `store.ts:951-955` |
| **Block 失效** | `active=0` | 可逆（`active=1`） | `blocks.active` 列 |
| **Block 删除** | 物理 `DELETE FROM blocks` | **不可逆** | `store.ts:1084` |
| **Block refold** | 新 block_id，旧块可能被删 | 不可逆 | `ingest.ts:402` |
| **Session 删除** | 物理删除全部 messages/blocks/ccr | **完全不可逆** | `store.ts:1694-1696` |
| **Blob 释放** | refcount ≤ 0 时 `DELETE FROM content` | **不可逆** | `store.ts:806` |

---

## 2. CMB 对每个事件的行为

| 上游事件 | CMB 是否感知 | 感知方式 | 滞后 | CMB 行为 |
|---|---|---|---|---|
| 消息 tombstone | ✅ | `getMessages` 过滤 `retired_at IS NULL` | 一个扫描周期 | 块判 MISSING / 消息计 SOURCE_MISSING |
| 消息复活 | ❌ | — | — | **SINK_CONFLICT**（见 §3） |
| 媒体释放 | ⚠️ 部分 | `getBlob` 返回 null | 一个周期 | CONTENT_MISSING → CORRUPTED |
| Block active→inactive | ⚠️ 部分 | `block_status` 字段变化 | 一个周期 | 无状态转移（引擎不按 active 过滤） |
| Block 物理删除 | ❌ | — | — | **旧 ledger 行永久 COMMITTED** |
| Block refold（覆盖集变） | ✅ | `covered_hash` 变 → 新 bridge_key | 一个周期 | 新块全新处理，旧块残留 |
| Block refold（覆盖集不变） | ⚠️ | `block_version` 变 | 一个周期 | **COMMITTED 锁死**（X-2） |
| Session 删除 | ❌ | — | — | ledger 行残留，无清理 |
| 数据库重建 | ❌ | — | — | 全部 bridge_key 重算 → 大规模重复 Evidence |
| Blob 释放 | ⚠️ | `getBlob` → null | 一个周期 | CONTENT_MISSING |

---

## 3. 撤销是单向棘轮（不可逆）—— 已实测

### 3.1 代码事实

`MockEvidenceSink.write()`（`evidence-sink.ts:50-58`）：
```ts
const existing = this.store.get(key);
if (existing) {
  if (existing.original_text !== ev.original_text || existing.revoked !== ev.revoked) {
    throw new CmbError("SINK_CONFLICT", "evidence identity collision with different content", ...);
  }
  return { duplicate: true };
}
```

`REVOKED` 在 `ALLOWED_TRANSITIONS`（`domain.ts:168`）中无出边。

### 3.2 复活场景推演

```
t0: 消息可见 → CMB 写 Evidence (revoked=false)
t1: 用户撤回 → revokeScan → Evidence.revoked = true
t2: 用户撤销撤回（上游 retired_at = NULL，消息复活）
t3: CMB 下一轮扫描 → 消息可见 → 构造 Evidence (revoked=false)
t4: sink.write() → existing.revoked(true) !== ev.revoked(false) → SINK_CONFLICT
```

**结果**：
- 块永久 PARTIAL/FAILED
- `counts.failed` 每轮 +1
- 无法 un-revoke（无此 API）
- 无法重写（SINK_CONFLICT）

### 3.3 为什么这是设计缺陷而非保护

保护意图（"不静默复活"）是合理的，但**没有配套的"合法复活"路径**：
- 上游 `retired_at` 明确支持复活（`store.ts:925` 的 UPDATE）
- CMB 的表达能力无法覆盖上游的合法状态转换

→ **CMB 的状态机比上游的状态空间小**，必然有状态无法表达。

---

## 4. 生命周期矩阵（任务书 §12 要求）

| 上游事件 | CMB 当前行为 | Evidence 行为 | Memory Core 行为 | 缺失契约 |
|---|---|---|---|---|
| 消息 tombstone | 块 MISSING，消息 SOURCE_MISSING | 保持原状（不撤销） | — | 需明确"tombstone 是否等于撤销" |
| 消息 tombstone + revokeScan | 撤销 Evidence | `revoked=true` | — | 无下游传播 |
| 消息复活 | SINK_CONFLICT → 块 PARTIAL | 保持 revoked=true | — | **无 un-revoke 接口** |
| Block active→inactive | 无变化（引擎不按 active 过滤） | 不变 | — | 无"块失效是否影响证据"定义 |
| Block 物理删除 | 无感知 | 不变 | — | **无删除传播机制** |
| Block refold（覆盖集变） | 新 bridge_key → 新块 | 旧 Evidence 保留，新 Evidence 新增 | — | 无 supersede 关系（上游 `supersedes_block_id` 恒 null） |
| Block refold（覆盖集不变） | COMMITTED 锁死 | 不变 | — | 键设计缺陷 |
| Session 删除 | 无感知 | 不变 | — | **无会话级撤销** |
| Blob 释放 | CONTENT_MISSING → CORRUPTED | 不变 | — | 无"证据完整性降级"传播 |
| 数据库重建 | 全部重新处理 | **重复 Evidence 风险** | — | 无库世代标识 |

---

## 5. CMB 有权做什么 / 无权做什么

| 操作 | 是否有权 | 依据 |
|---|---|---|
| 创建 Evidence | ✅ | 设计目标 |
| 标记 Evidence revoked | ✅ | `revokeScan` |
| 删除 Evidence | ❌ | Sink 无 delete 接口 |
| 复活 Evidence（un-revoke） | ❌ | Sink 无此接口 |
| 修改 Evidence 内容 | ❌ | SINK_CONFLICT |
| 删除 MemoryUnit | ❌ | 无接口，且任务书禁止 |
| 修改 confirmed MemoryUnit | ❌ | 无接口 |
| 标记 MemoryUnit 来源失效 | ❌ | 无接口（Memory Core 未实现） |

**任务书 §12 要求：「不得让 CMB 擅自删除 Memory Core 的 confirmed MemoryUnit」→ 当前实现满足（无接口）。**

但反过来，「已确认 MemoryUnit 如何感知其来源 Evidence 失效」→ **完全无路径**。

---

## 6. 保留窗口与不可恢复区

| 数据类型 | 保留期 | 超期后 |
|---|---|---|
| 文本 `content.bytes` | 只要 `messages` 行存在 | `deleteSession` 后永久丢失 |
| 媒体 blob | 只要 `message_media` 行存在 | tombstone 后立即释放 |
| Block 行 | 只要在当轮 `state.blocks` 中 | 下一轮 refold 即物理删 |
| `blocks.summary` | 随 block 行 | 永久丢失 |
| `orphans_json` | 会话生命周期内 | 文本摘要，非完整历史 |

**关键判断**：
- **文本消息**：可长期恢复（除非 session 删除）
- **多模态**：tombstone 后立即不可恢复
- **Block 历史**：**不可恢复**（无墓碑机制）
- **Block 覆盖关系变化史**：**不可恢复**

→ CMB 若错过某个 Block 的存在窗口（例如停机超过 N 轮压缩），**该块覆盖的历史永不可重建**。

---

## 7. 结论

| 项 | 判定 |
|---|---|
| 消息 tombstone 感知 | **PASS** |
| 消息复活处理 | **FAIL**（SINK_CONFLICT，不可逆） |
| Block 删除感知 | **FAIL**（完全无感知） |
| Block 历史可恢复性 | **FAIL**（无墓碑，物理删除） |
| Session 删除感知 | **FAIL**（无感知） |
| 数据库重建处理 | **FAIL**（重复 Evidence 风险） |
| Evidence 撤销 | **PASS_WITH_LIMITATION**（单向、不可逆） |
| 下游传播（Evidence→MemoryUnit） | **UPSTREAM_CHANGE_REQUIRED**（Memory Core 未实现） |
| CMB 不越权删除 MemoryUnit | **PASS**（无接口） |

### 一句话结论

**CMB 是"只进不退"的：能创建、能标记撤销，但不能删除、不能复活、不能感知上游的删除；Block 生命周期的历史完全不可恢复，错过窗口即永久丢失。**
