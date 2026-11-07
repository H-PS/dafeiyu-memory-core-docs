# 06 · Embedding 状态机、审计、备份与恢复

解决 F-08、F-14、F-24、F-26、F-17、F-22、F-53、F-25、F-11（恢复面）。

## 1. Part A · Embedding 状态机（六态）

### 1.1 状态定义与转换

```
             submit(text)
                │
                ▼
            ┌─────────┐  开始请求  ┌───────────┐
            │ pending │ ─────────▶ │ processing│
            └─────────┘            └───────────┘
                ▲                      │      │
        重试到期 │                 成功  │      │ 失败(网络/超时/校验)
   ┌──────────────┐                    ▼      ▼
   │ retry_wait   │◀── attempts<N ─── ready   failed(本次)
   └──────────────┘                    │      │ attempts≥N
        │ 超过重试上限                  │      ▼
        └────────────────────────────▶ │  ┌────────┐
                                        │  │ failed │（终态：正文保留，仅关键词降级）
                                        │  └────────┘
   suppression(scope含vec) / 模型退役 ──▶ ┌─────────┐
                                          │ cleared │（向量清空，正文在）
                                          └─────────┘
```

| 状态 | 语义 | 记录可检索性（语义） | 可检索性（降级） |
|---|---|---|---|
| `pending` | 已持久化、待首次嵌入 | 否 | **是**（若降级开启） |
| `processing` | 请求在途 | 否 | 是 |
| `ready` | 向量可用且通过校验 | 是 | — |
| `retry_wait` | 等待下次重试（退避） | 否 | 是 |
| `failed` | 超过重试上限（终态，保留正文） | 否 | 是 |
| `cleared` | 向量被显式清除（suppression/模型退役） | 否 | 是 |

### 1.2 十项设计要点逐条（任务 7 要求）

| # | 要求 | 设计 |
|---|---|---|
| 1 | 失败时正文是否先持久化 | **是**。append 管线（02 §1.1）正文先入 store（status=pending），嵌入在队列外异步。**embedding 失败 ≠ 记忆写入失败** |
| 2 | 是否允许关键词降级检索 | 允许，默认 **开启**（08 D-04）。实现 V1 `m-v.js:10-11` 声明的降级契约：`searchVectorMemory` 当 (a) 查询向量不可用，或 (b) 记录 status≠ready，或 (b) vecModel 不匹配 → 对应记录走「规范化包含 + 分类前缀」文本匹配；结果带 `degraded:true` 与命中方式标注 |
| 3 | 如何安排重试 | 后台调度器（60s 扫描）：收集 `status∈{retry_wait}` 且 `nextRetryAt<=now` 的记录 → 提交 Embedder（并发上限 2，防止压垮本机 Ollama） |
| 4 | 如何限制重试次数 | `embedding.attempts`，上限 5；退避 1min→5min→30min→2h→6h；超限 → `failed`（终态）。**不自动放弃正文** |
| 5 | 如何处理模型切换 | 记录 `vecModel`；Embedder 配置热更新（修 F-25）。模型变更事件 → 所有 `ready` 且 vecModel≠新值的记录批量转 `pending`（reason='model_change'，M2 批量任务；M1 手动触发端点 `POST /api/memory/reembed {key?}`） |
| 6 | 如何验证向量维度 | 写入校验：`Array.isArray && length===vecDim`（默认 1024，配置项）；不满足 → 该次嵌入判 failed（重试） |
| 7 | 如何验证数值有效性 | 全部 `Number.isFinite`、`v.length>0`；L2 范数 > 1e-9（拒零向量）；任一不满足 → failed |
| 8 | 如何区分旧/新模型向量 | `vecModel` 字段 + 检索谓词 `record.vecModel === currentModel`（旧模型记录仅降级可召回，修 F-14） |
| 9 | 如何处理 embedding 超时 | 30s（沿用 `m-v.js:92`）计入一次 attempt；processing 超时 → retry_wait |
| 10 | 失败不影响正文持久化 | 见 #1；`failed`/`retry_wait` 记录在 list/search(降级)/审计中均可见其状态 |

### 1.3 Embedder 接口

```js
Embedder.submit(text) → {promise, cancel}
  promise: resolve({vec, vecModel, vecDim}) | reject({code:'timeout'|'invalid'|'unreachable'|'http', message})
配置: {endpoint, model, dim, timeoutMs, maxConcurrent, maxAttempts, backoff[]}
热更新: cfg 变更事件 → Embedder.config 替换（下一次 submit 生效）
```

- **写队列与 Embedder 解耦**：队列绝不 await 嵌入（除 append/edit 的可选 `awaitEmbed:true` 且 ≤30s 等待，超时转 pending 不失败）。
- 嵌入成功后回写向量：作为 `embedding_refresh` op 入队（Tier-2，effects=[A upsert(仅 vec/embedding 字段, rev+1)]）——**注意**：该 rev+1 会使 pending SR superseded，若不接受 → 08 D-17。

### 1.4 查询降级（修 F-22）

`qq_memory_query` 返回结构：
```json
{ "ok":true, "key":"…", "b":{…B 层摘要…},
  "vector":{ "status":"ready|degraded|unavailable", "reason":"model_mismatch|embedder_down|…",
             "hits":[…], "degradedHits":[…文本匹配…] } }
```
B 层结果**永远**返回；A 层失败只降级不整段报错。

## 2. Part B · 审计系统（两级制）

### 2.1 审计事件结构（state/memory-audit.log，JSONL）

```json
{ "at":"2026-11-01T00:00:00.000Z",
  "opId":"op_01J8…", "opType":"remove",
  "actor":{"type":"console|agent|system","id":"group:123|console","tokenFp":"a1b2c3d4"},
  "memoryId":"mem_01J8…", "revBefore":3, "revAfter":null,
  "result":"ok|failed|blocked|recovery_blocked",
  "reason":"…", "contentHash":"sha256:前16", "textLen":42 }
```

- **不存正文**（修 F-17）：仅 `contentHash` + 长度。
- `tokenFp` = sha256(令牌) 前 8 位（操作者弱标识，不存明文）。

### 2.2 顺序与业务提交的关系（任务 8 要求 1/2/3/4）

```
WAL(pending) → [Tier-1 审计写入] → 原子提交(04/02 §3) → WAL(committed) → [Tier-2 审计]
                     ↑ 阻断点                          ↑ 补偿点
```

| Tier | 操作 | 审计失败时 |
|---|---|---|
| **Tier-1（必须成功）** | remove、clear、share_approve、share_reject、share_revoke、suppress、unsuppress、restore、force_restore、migrate、recovery resubmit/discard、tombstone 白名单复活 | **操作整体失败**（WAL→aborted，业务未发生）。返回 500「审计不可用，操作已取消」。**不接受**「审计失败但业务照做」 |
| **Tier-2（可补偿）** | append、edit、share_request、embedding 状态变更、备份 | 业务成功；事件入 `memory-audit-pending.jsonl` 补偿队列；后台 30s 重试；队列写入也失败 → 事件缓存在内存环形缓冲 + `/api/status` 告警位 + bridge.log ERROR |

> 明确否决「审计失败只告警然后继续」用于 Tier-1 的做法（任务 8 要求）。理由：删除/授权类操作一旦发生即不可逆，无审计等于无追责；宁可拒绝服务。

### 2.3 其余要点（要求 5/6/7/8）

| 要求 | 设计 |
|---|---|
| 5 不泄露正文 | 见 §2.1 contentHash；旧 `memory-admin.log`（含 80 字正文）归档停用 |
| 6 分享审批与删除 | 均为 Tier-1，事件含 requestId/copyId/级联明细 |
| 7 限制增长 | 大小轮转：单文件 10MB，保留 10 份（`memory-audit.log` → `.1` … `.9`，rename 轮转，复用 V1 `createCappedLogger` 思路但按字节）；另设事件数兜底 20 万条 |
| 8 文件损坏 | 启动时逐行校验：坏行移入 `memory-audit.log.quarantine`（原文保留）+ 告警 + 审计 `AUDIT_QUARANTINE` 系统事件；**不静默丢弃**（与 F-12 同口径） |

## 3. Part C · 备份

### 3.1 备份 envelope（backups/memory-backup-YYYYMMDD-HHMMSS.json）

```json
{ "schemaVersion": 2,
  "createdAt": 1730000000000, "tool": "memory-service/2.4.2",
  "cause": "daily_first_write|pre_delete|pre_restore|manual|migration",
  "storeVersion": 143,
  "counts": { "records": 98, "tombstones": 3, "shareRequests": 2, "walPending": 0 },
  "records": [ …A 层记录（含 _crc）… ],
  "tombstones": [ …墓碑… ],
  "shareRequests": [ …SR… ],
  "bProjection": { "group:123": {…三字段快照…}, … },
  "digest": { "algo": "sha256", "records": "…", "tombstones": "…", "total": "…" } }
```

- `digest`：各数组按序拼接的 sha256（加载后重算比对，防备份文件损坏/篡改）。
- **A/B 一致性快照**：备份包含 B 投影快照与 `storeVersion`，恢复后可对账。
- WAL：仅存 `walPending` 条目（committed 部分无需备份——可从 store 推导）。

### 3.2 备份原子性与触发（修 F-24/F-26）

- 写法：构造完整 envelope → 临时文件 → fsync → renameWithRetry（02 §3 统一规范）。**不用** copyFileSync。
- 失败：**不再 catch{} 静默**。daily 备份失败 → Tier-2 审计 + 告警；**删除/恢复前的备份失败 → 阻断该操作**（Tier-1 前置条件）。
- 触发点：每日首次写（沿用 V1 语义）、每次 remove/clear/restore 前（替代 V1 `bridge.js:6716` 的同名覆盖）、手动。
- 文件名含时间戳（修 F-26）。
- 保留策略：日备 14 份 + 周备 8 份（按文件名日期归类清理；tombstone 相关文件不参与清理）→ 08 D-07。

## 4. 恢复流程（任务 10）

```
POST /api/memory/restore {file, mode:'dry-run'|'execute', confirmToken?, force?}
[阶段 R0 预检]
  控制台令牌；文件存在；envelope 解析；schemaVersion 兼容；digest 校验
[阶段 R1 dry-run（默认且必经）]
  逐条比对当前 store：
    - id ∈ tombstone 集合        → 计入 willSkip(tombstoned)
    - id 存在且 rev ≥ 备份 rev    → 计入 willSkip(newer)
    - id 存在且 rev < 备份 rev    → 计入 willUpdate（降级更新？默认跳过并列出 → 08 D-18）
    - id 不存在                   → 计入 willImport
    - embedding 状态非 ready      → 计入 needReembed
  输出报告 + 一次性 confirmToken（15 分钟有效）
[阶段 R2 execute（需 confirmToken）]
  ① 取 writer.lock → ② 当前状态完整备份（pre_restore）→ ③ Tier-1 审计 →
  ④ 构造新 store（当前状态 ⊕ 备份，按 R1 报告）→ ⑤ 原子写 → ⑥ B 投影对账/重建 →
  ⑦ SR 合并（pending 保留当前，备份中 approved 且副本被导入的保留）→ ⑧ WAL 记 restore op →
  ⑨ 审计 result:ok
[失败回滚]
  ④~⑧ 任一步失败 → 用 ② 的 pre_restore 备份还原 store/B/SR → 审计 result:failed → 返回 500
[force_restore]
  force:true 允许跳过 willSkip(newer)（版本降级），**仍然跳过 tombstoned**；
  ignoreTombstone:[id…] 白名单逐条复活（04 §7，审计 RESURRECT）
```

| 任务 10 要求 | 落点 |
|---|---|
| 格式/schemaVersion | §3.1 |
| 完整性校验 | digest + 每记录 `_crc` |
| 原子性 | §3.2 |
| 保留策略 | §3.2 |
| A/B 一致性快照 | §3.1 bProjection |
| WAL 参与恢复 | R2 ⑧ + 03 §5（启动恢复先于 restore 执行） |
| tombstone 参与恢复 | R1 比对 + 默认跳过 |
| 预览 / dry-run | R1（强制必经） |
| 二次确认 | confirmToken |
| 失败回滚 | pre_restore |
| 默认防复活 | willSkip(tombstoned) |

## 5. 验收要点

1. Ollama 停机：append 成功（`embedding:'pending'`）；`qq_memory_query` 返回 B + 降级命中；恢复后 60s 内转 ready。
2. 坏向量（NaN/零向量/错维度）注入 → 嵌入判 failed，不入库。
3. Tier-1 审计磁盘写失败注入 → remove/clear/approve 均被拒绝且 WAL aborted；Tier-2 失败 → 业务成功 + 补偿队列最终补齐。
4. 备份文件手工篡改 1 字节 → digest 校验失败拒绝恢复。
5. 恢复 dry-run 报告与执行结果一致；已删除 id 不复活。
