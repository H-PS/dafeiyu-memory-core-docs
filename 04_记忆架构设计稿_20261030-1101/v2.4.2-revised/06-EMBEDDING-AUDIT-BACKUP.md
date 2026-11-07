# 06 · Embedding 状态机、审计、备份与恢复

解决 F-08、F-14、F-17、F-22、F-24、F-25、F-26、F-11（恢复面）。本版修订：**M0 三态 + vecContentRev 绑定**、Tier-1 分级、最小恢复校验。

---

# 1. Embedding 状态机

## 1.1 M0：三态（必须）

```
      append/edit_text（正文先落盘）
                │
                ▼
          ┌─────────┐   embed 成功   ┌───────┐
          │ pending │ ─────────────▶ │ ready │
          └─────────┘                └───────┘
                │   embed 失败
                ▼
          ┌────────┐
          │ failed │（终态：正文保留，仅关键词降级）
          └────────┘
```

| 状态 | 语义 | 语义检索 | 降级检索 |
|---|---|---|---|
| `pending` | 已持久化、待嵌入 | ❌ | ✅ |
| `ready` | 向量可用且校验通过 | ✅ | — |
| `failed` | 嵌入失败（终态，正文在） | ❌ | ✅ |

**M0 的关键约束**：
- **正文在 embed 之前持久化** ⇒ `embedding.status='pending'` 的记录已存在 store
- embed 失败 ⇒ `status='failed'`，**不删除记录**
- M0 无自动重试（重试队列是 M1），但提供手动 `POST /api/memory/reembed {memoryId?}`

## 1.2 M1：六态（扩展）

在 M0 三态基础上增加 `processing` / `retry_wait` / `cleared`：

```
pending → processing → ready
                    ↘ failed（本次）
                    ↘ retry_wait（attempts < 5）→ processing
                    ↘ failed（终态，attempts ≥ 5）
suppression(scope含vec) / 模型退役 → cleared
```

| 状态 | 语义 | 语义检索 | 降级检索 |
|---|---|---|---|
| `pending` | 待首次嵌入 | ❌ | ✅ |
| `processing` | 请求在途 | ❌ | ✅ |
| `ready` | 可用且通过校验 | ✅ | — |
| `retry_wait` | 等待重试（退避） | ❌ | ✅ |
| `failed` | 超上限（终态，正文在） | ❌ | ✅ |
| `cleared` | 向量被显式清除 | ❌ | ✅ |

## 1.3 ★ vecContentRev：向量不得覆盖新正文（P0-1 要求 7）

```js
// 写入向量前的强制检查（INo-7）
function canWriteVec(record, forContentRev) {
  return record.contentRev === forContentRev          // 计算的正文仍是当前正文
      && record.vecContentRev < forContentRev;        // 且向量确实过期
}

// 写入后
record.vec = newVec;
record.vecContentRev = forContentRev;
record.embedding.status = 'ready';
record.rev += 1;              // 全量版本 +1
// contentRev 不变
```

**三条防线**：
1. P1 提交 embed 时携带 `forContentRev`
2. P3 写入前 `canWriteVec` 检查
3. 不满足 ⇒ 丢弃该向量（记录标 `pending`，等待新正文的嵌入）

**这解决了原设计的隐患**：edit 提交后，旧正文的迟到向量不会覆盖新正文。

## 1.4 向量校验（F-14）

```js
function validateVec(vec, dim) {
  if (!Array.isArray(vec)) return { ok:false, reason:'not_array' };
  if (vec.length !== dim) return { ok:false, reason:'dim_mismatch' };
  if (!vec.every(Number.isFinite)) return { ok:false, reason:'not_finite' };
  let n2 = 0; for (const v of vec) n2 += v*v;
  if (Math.sqrt(n2) < 1e-9) return { ok:false, reason:'zero_vector' };
  return { ok:true };
}
```

不合格 ⇒ 该次嵌入判 `failed`，不入库（M0）；`retry_wait`（M1）。

## 1.5 M0 / M1 的重试与模型切换

| 项 | M0 | M1 |
|---|---|---|
| 自动重试 | ❌ | ✅（60s 扫描，退避 1m→5m→30m→2h→6h，上限 5） |
| 手动重嵌入 | ✅ `POST /api/memory/reembed` | ✅ 同上 + 批量 |
| 模型切换 | 手动触发全量重嵌入 | 自动批量（M2） |
| `vecModel` 隔离 | 记录但**不强制**过滤 | 检索只召回同模型记录 |

## 1.6 查询降级（M0 带标记，M1 完整）

```json
{
  "ok": true, "key": "…",
  "projection": { …P 层三字段摘要… },
  "vector": {
    "status": "ready|degraded|unavailable",
    "reason": "model_mismatch|embedder_down|no_pending_vec",
    "hits": [ … ],
    "degradedHits": [ …文本匹配… ]
  }
}
```

- P 层结果**永远**返回（M0 起）
- A 层失败只降级不整段报错（M1 完整；M0 返回 `status:'unavailable'` + 保留 P 结果）

**M0 的最小实现**：P 结果始终返回；A 失败时 `vector.status='unavailable'`，不再整段 500。

---

# 2. 审计系统

## 2.1 事件结构（`state/memory-audit.log`，JSONL）

```json
{ "at": "2026-11-01T00:00:00.000Z",
  "opId": "op_01J8…", "opType": "remove",
  "actor": { "type": "console|agent|system", "id": "group:123|console", "tokenFp": "a1b2c3d4" },
  "memoryId": "mem_01J8…",
  "revBefore": 3, "revAfter": null,
  "contentRevBefore": 2, "contentRevAfter": null,
  "result": "ok|failed|blocked|recovery_blocked",
  "reason": "…", "contentHash": "sha256:前16", "textLen": 42 }
```

- **不存正文**（修 F-17）：仅 `contentHash` + 长度
- `tokenFp` = sha256(令牌) 前 8 位（操作者弱标识，不存明文）
- **新增 `contentRevBefore/After`**：便于追溯内容版本

## 2.2 顺序（不变）

```
WAL(pending) → [Tier-1 审计] → 原子提交(T→A→P→SR) → WAL(committed) → [Tier-2 审计]
                    ↑ 阻断点                                ↑ 补偿点
```

| Tier | 操作 | 审计失败时 |
|---|---|---|
| **Tier-1** | remove、clear、share_approve、share_reject、share_revoke、suppress、unsuppress、restore、force_restore、migrate、recovery resubmit/discard、tombstone 白名单复活 | **操作整体失败**（WAL→aborted，业务未发生）。500「审计不可用，操作已取消」 |
| **Tier-2** | append、edit_*、share_request、embedding 状态变更、备份 | 业务成功；入 `memory-audit-pending.jsonl` 补偿队列；后台 30s 重试；队列也失败 → 内存环形缓冲 + `/api/status` 告警 + bridge.log ERROR |

**明确否决**"审计失败只告警然后继续"用于 Tier-1。理由：删除/授权不可逆，无审计等于无追责。

## 2.3 轮转与损坏

| 项 | 规则 |
|---|---|
| 轮转 | 单文件 10MB × 10 份（`.log` → `.1` … `.9`） |
| 事件数兜底 | 20 万条 |
| 损坏 | 坏行移入 `memory-audit.log.quarantine`（原文保留）+ 告警 + `AUDIT_QUARANTINE` 系统事件 |

---

# 3. 备份

## 3.1 envelope（`backups/memory-backup-YYYYMMDD-HHMMSS.json`）

```json
{
  "schemaVersion": 2,
  "createdAt": 1730000000000,
  "tool": "memory-service/2.4.2-rev",
  "cause": "daily_first_write|pre_delete|pre_restore|manual|migration",
  "storeVersion": 143,
  "counts": { "records": 98, "tombstones": 3, "shareRequests": 2, "walPending": 0 },
  "records": [ …A 层记录（含 _crc）… ],
  "tombstones": [ … ],
  "shareRequests": [ … ],
  "projection": { "group:123": {…三字段…} },
  "digest": { "algo": "sha256", "records": "…", "tombstones": "…", "total": "…" }
}
```

- `digest`：各数组按序拼接的 sha256（加载后重算比对）
- **A/P 一致性快照**：备份含 P 快照与 `storeVersion`，恢复后可对账
- WAL：仅存 `walPending`

## 3.2 原子性与触发

- 写法：完整 envelope → 临时文件 → fsync → renameWithRetry（02 §3）。**不用 copyFileSync**
- 失败：**不 catch{} 静默**。日备失败 → Tier-2 审计 + 告警；**删除/恢复前的备份失败 → 阻断该操作**（Tier-1 前置）
- 触发点：每日首次写、每次 remove/clear/restore 前、手动
- 文件名含时间戳（修 F-26）
- 保留：日备 14 + 周备 8（D-07）

## 3.3 M0 的最小要求

M0 必须实现：
- envelope 格式 + digest
- 原子写
- 失败告警（不静默）
- 文件名时间戳

M0 可暂不实现：周备、自动清理策略（保留全部日备，磁盘增长可接受）。

---

# 4. 恢复流程

## 4.1 R0~R2 三阶段（M1 完整；M0 最小）

```
POST /api/memory/restore {file, mode:'dry-run'|'execute', confirmToken?, force?}
[R0 预检]  控制台令牌；文件存在；envelope 解析；schemaVersion 兼容；digest 校验
[R1 dry-run（默认且必经）]
  逐条比对当前 store：
    id ∈ tombstone          → willSkip(tombstoned)
    id 存在且 rev ≥ 备份 rev → willSkip(newer)
    id 存在且 rev < 备份 rev → willUpdate（默认跳过并列出 → D-18）
    id 不存在               → willImport
    embedding 非 ready      → needReembed
  输出报告 + 一次性 confirmToken（15 分钟）
[R2 execute（需 confirmToken）]
  ① writer.lock → ② pre_restore 备份 → ③ Tier-1 审计 →
  ④ 构造新 store（当前 ⊕ 备份，按 R1）→ ⑤ 原子写 →
  ⑥ P 重建/对账 → ⑦ SR 合并 → ⑧ WAL 记 restore op → ⑨ 审计 ok
[失败回滚] ④~⑧ 任一步失败 → 用 ② 还原 store/P/SR → 审计 failed → 500
[force]  force:true 跳过 willSkip(newer)（版本降级），**仍跳过 tombstoned**；
         ignoreTombstone 白名单（04 §7，D-09 未批准前不实现）
```

## 4.2 ★ M0 的最小恢复校验函数（本版上移）

M0 无 UI，但**必须有 CLI 校验**，否则人工 copy 备份会绕过 tombstone：

```
node scripts/memory-restore-check.mjs --file <backup.json> [--apply]

行为：
  ① 读 envelope，校验 schemaVersion + digest
  ② 读当前 store 与 tombstones
  ③ 输出报告：
     - 将导入 N 条 / 将跳过 M 条（其中 tombstoned K 条、newer J 条）
     - 冲突列表（id / 备份 rev / 当前 rev）
  ④ 无 --apply 时仅报告（dry-run）
  ⑤ 有 --apply 时：
     - 要求 writer.lock（bridge 必须停止，见 07 §6）
     - 先写 pre_restore 备份
     - 按报告构造新 store（跳过 tombstoned/newer）
     - 原子写 + P 重建
     - 写审计 `RESTORE {imported, skipped}`（Tier-1）
     - 打印最终计数对账
```

**关键**：即使 M0 没有 UI，这个函数**保证"恢复不会复活已删记录"**（F-11 的核心）。

## 4.3 恢复与 P 层

恢复后 P 层必须重建：
- 若备份含 `projection` → 用它（但需 `storeVersion` 匹配，否则重建）
- 否则从恢复后的 store 全量重建 P

---

# 5. M0 的关键缺口清单（本版已闭合）

| 原 M0 缺口 | 本版处置 |
|---|---|
| 无恢复流程 ⇒ 人工 copy 绕过墓碑 | ✅ 最小恢复校验函数（§4.2）纳入 M0 |
| 无 embedding 状态 ⇒ 正文丢失 | ✅ 三态 + 正文优先持久化（§1.1） |
| A 层存未脱敏原文 | ✅ F-02 上移（MS 入口唯一规范化） |
| B 层损坏无法修复 | ✅ P 层可从 A 重建（最小 Reconciler） |

---

# 6. 验收要点

1. Ollama 停机时 append 成功（`embedding:'pending'`）且**重启后正文仍在**。
2. Ollama 恢复后手动 `reembed` → `ready`。
3. 嵌入返回 NaN/零向量/错维度 → 判 `failed`，不入库。
4. **edit 后旧向量迟到 → 不覆盖新正文**（INo-7，构造 forContentRev < contentRev）。
5. Tier-1 审计写失败 → remove/clear/approve 被拒绝且 WAL aborted；Tier-2 失败 → 业务成功 + 补偿最终补齐。
6. 备份文件篡改 1 字节 → digest 校验失败拒绝加载。
7. **M0 恢复校验：用含已删记录的备份执行 --apply → 已删 id 不出现**。
8. 审计文件 grep 不到任何记忆正文片段。
