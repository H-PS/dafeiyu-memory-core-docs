# 10 · 文件清单 · 迁移 · 兼容性（修订版）

> 模块编号统一为 M01~M14（定义见 `01-ARCHITECTURE.md` §8）。本版新增 M04 权限模块，更新行数总量为 4,470，改动量 790/-75。

---

## 1. 新增源码（14 个，约 4,470 行）

| # | 文件 | 行数 | 里程碑 | 职责 |
|---|---|---|---|---|
| M01 | `src/memory-store.js` | 750 | M0 | A 层读写、WAL、tombstone、三版本、per-key 锁、审计 |
| M02 | `src/memory-projection.js` | 380 | M0 | 投影构建、per-key `convVersion`、重建差异 |
| M03 | `src/memory-stats.js` | 150 | M0 | S-1 独立统计文件 |
| M04 | `src/memory-permission.js` | 220 | M0 | `classify()` / `canRead()` / `detectContentClass()` / 五维断言 |
| M05 | `src/memory-index.js` | 520 | M0 写 / M1 查 | docId、generation、coveredRev、尾部补偿 |
| M06 | `src/tokenize.js` | 90 | M1 | 分词 + `estimateTokens`（全项目唯一实现） |
| M07 | `src/memory-recall.js` | 560 | M1 | 注册表、RRF、归一化、两遍预算、断言、降级 |
| M08 | `src/memory-channels/semantic.js` | 120 | M1 | 向量通道 |
| M09 | `src/memory-channels/keyword.js` | 130 | M1 | BM25 通道 |
| M10 | `src/memory-channels/temporal.js` | 140 | M1 | 时间通道（Asia/Shanghai 固定） |
| M11 | `src/prompt-ledger.js` | 170 | M1 | 块账本 |
| M12 | `src/memory-retain.js` | 380 | M2 | 信号、四闸、三池、补算队列 |
| M13 | `src/memory-reflect.js` | 540 | M2 | 三档去重、过时、偏好、观察、`verifyObservation` |
| M14 | `scripts/memory-maintenance.mjs` | 320 | M0/M1 | rebuild-index / diff-projection / verify-crc / repair-stats |

---

## 2. 修改源码（3 个，+790 / -75）

### 2.1 `src/memory-vector.js`（380 行 → +70 / -15）

| 位置 | 改动 |
|---|---|
| `:149` `appendVectorMemory` | 写路径委托给 `memory-store.js`；原函数改为 `throw new Error('已迁移')`（暴露漏网调用） |
| `:157` 默认值 | `share_review_status:'not_requested'` → 调用 `classify()` 推导 `scope` |
| `:171-178` `searchVectorMemory` | 过滤条件改用 `canRead()`；删除硬编码的 `visibility/approved` 判断 |
| `:217` `listVectorMemory` | 过滤器改调 `canRead()`（admin caller） |
| `:44` `loadRows` | 坏行从静默 skip 改为 `{rows, errors}` 上报 |
| `:361` `editVectorMemory` | await 前的快照改为 await 后重读定位（消除竞态） |
| `:376` `atomicRewrite` | tmp 名加 `pid + rand` |

### 2.2 `src/bridge.js`（12,482 行 → +600）

| 区域 | 改动 |
|---|---|
| `buildWakePromptV2`（`:10669`） | 改为 Ledger 驱动（+150）；凭据块从每轮改首轮（-1 行逻辑 +10） |
| `formatMemoryV2`（`:9173`） | 加字符预算；`interactionCount` 改读 stats（+40） |
| `ensureSession`（`:8461`） | `not-found` 分支清 ledger（+5） |
| 会话轮转 | `rotationTx` 状态机 + 排空 + 恢复（+115） |
| 新增端点 | `/memory-recall`（+60）、`/memory/health`、`/rebuild-projection`、`/diff-projection`（+120）、`/agent/handshake`（+60） |
| 消息链路 | 挂 `retainScan`（+20） |
| 空闲调度 | 挂 `reflectBatch`（+60） |
| 权限接线 | 写路径统一走 `canRead` / `classify`（+80） |

### 2.3 `src/mcp-snowluma-safe.js`（1,465 行 → +120 / -60）

| 区域 | 改动 |
|---|---|
| 新增工具 | `qq_memory_recall`（+60） |
| `qq_memory_query`（`:981`） | topic 分支改调 `/memory-recall`（+20） |
| **凭据目标态** | 移除 40+ schema 的 `token` 参数（-60）；按 key 解析凭据 + 缓存（+40） |

**注**：凭据改动依赖 `12-RISKS-PERMISSIONS.md` §6 F-1 核查结论；未核查前只做过渡态（bridge 侧），MCP 侧不动。

---

## 3. 新增数据文件

```
state/
├── memory-store.jsonl              A 层（格式微扩，见 SCHEMA）
├── memory-projection.json          P 层（加 digest / recallCache）
├── social-v2.json                  S 层（移除 interactionCount；加 ledger/rotationTx/clearedKeys/reviewQueue/pendingEmbed）
├── memory-impression-stats.json    M0·S-1
├── memory-quarantine.jsonl         M0 建结构 / M2 写入
├── memory-candidates.jsonl         M2
├── memory-index/                   M0 建 / M1 查
│   ├── meta.json  ids.json  vec.f32  bm25.json  temporal.json
└── wal/{pending,committed,aborted}.jsonl
```

体积估算（10k 条）：`vec.f32` ≈ 40MB、`bm25.json` ≈ 5MB、`temporal.json` ≈ 0.5MB、WAL 每次 op ≈ 1KB（定期归档）。

---

## 4. 配置追加（只追加，不改既有键）

```json
{
  "recall": { "enabled": true, "perChannelLimit": 30,
              "fusion": { "rrfK": 60, "weights": { "semantic": 1.0, "keyword": 0.9, "temporal": 0.6 } },
              "rerank": { "mode": "rule",
                          "weights": { "relevance": 0.55, "recency": 0.20, "lifecycle": 0.15, "contentClass": 0.10 },
                          "recencyHalfLifeDays": 30 },
              "budget": { "injectTokens": 600, "maxItems": 8, "toolReturnTokens": 1200 },
              "channelTimeoutMs": 300, "totalTimeoutMs": 450, "injectPathBudgetMs": 500,
              "dedup": { "windowTurns": 3, "bLayerOverlap": 0.8 } },

  "context": { "blockLedger": { "enabled": true, "personaAnchorEveryTurns": 12 },
               "credential": { "mode": "session_once", "ttlMs": 1800000 },
               "personaMaxChars": 3000, "bMemoryMaxChars": 800, "maxPromptTokens": 6000,
               "rotateAfterTurns": 60, "rotateAfterTokens": 40000, "rotateAfterIdleMs": 43200000,
               "rotateDrainTimeoutMs": 5000, "rotateMaxAttempts": 3,
               "digestMaxChars": 600, "digestMode": "deterministic" },

  "retain": { "enabled": true,
              "gates": { "noveltyThreshold": 0.92, "minValueScore": 0.55 },
              "pools": { "candidateTtlDays": 7, "candidateReevaluateAtSeen": 3,
                         "quarantineTtlDays": 0, "autoPromote": false },
              "embedding": { "retryMax": 5, "retryBaseMs": 60000, "retryFactor": 2 } },

  "reflect": { "enabled": true, "idleScheduler": true,
               "dedup": { "mode": "exact_only", "exactCosMin": 0.99 },
               "synthesis": { "minEvidence": 3, "minTimeSpanDays": 3, "llmSynthesis": false,
                              "verifyBigrams": true, "autoActivateAfterProofs": 5 },
               "preference": { "minOccurrences": 3, "minSpanDays": 7, "negationMinGapDays": 3 } }
}
```

**机制类配置不允许改**：`retain.pools.autoPromote`（固定 false）、`retain.async`（固定 true）、`reflect.dedup.mode`（只接受 `exact_only`）、`context.maxPromptTokens` 的砍序（固定）。

---

## 5. 迁移

当前记忆库 **0 条**（2026-10-31 冻结时清空），本次是"建结构"：

| # | 事项 | 动作 |
|---|---|---|
| 1 | `social-v2.json` 残留 `interactionCount` | 抄到 stats 文件，旧字段置 null |
| 2 | `memory-tombstones.jsonl` | 保留不动 |
| 3 | `memory-vectors.jsonl`（旧 A 层） | 若存在走一次性导入器：Schema 转换 + `review.status` 原映射 + `contentClass='uncertain'`（保守），导入后改名 `.imported` 保留 30 天 |

**不批量升级历史 `contentClass`**：历史数据可能是私聊内容，批量标 `public` 会造成越权。

### 5.1 回滚

| 阶段 | 回滚 |
|---|---|
| M0 前 | 删空文件即可 |
| M0 后 | 保留 store + tombstone；Schema 是加法，旧代码可读 |
| M1 后 | 删 `memory-index/`（召回降级但可用） |
| 凭据后 | `credential.mode` 调回 `session_once`（代码保留两条路径） |
| M2 后 | `kind in (observation,preference)` 批量置 `lifecycle='deleted'`（不物理删） |

没有一步需要"恢复备份"——加法式设计的收益。

---

## 6. 兼容性

| 场景 | 结果 |
|---|---|
| 新代码 + 旧数据 | 兼容（缺字段有保守默认） |
| 旧代码 + 新数据 | **部分**：旧 `searchVectorMemory` 不认识 `scope`，群聊过滤退化为"查不到"（不会越权，只是功能退化） |
| 索引损坏/缺失 | 兼容：重建或降级 |
| Ollama 不可用 | 兼容：语义路降级，写入仍成功（`embedding.status='pending'`） |
| 双实例 | **不支持**：`bridge.lock` 保证单实例 |

---

## 7. 部署检查清单

```
[ ] config.json 新字段已加；机制类配置未被改动
[ ] state/memory-index/ 目录已建
[ ] memory-maintenance.mjs rebuild-index 跑通
[ ] /api/memory/health 三者一致
[ ] 记录当前记忆条数作为基线
[ ] state 目录已快照
[ ] 回滚脚本就绪（删索引 / 降 lifecycle / credential.mode 回退）
[ ] 单写者 grep 通过
[ ] 崩溃注入 12 场景通过
[ ] eval-privacy --stage M0 = 0
```

---

## 8. 文件 ↔ 验收映射

| 文件 | 覆盖验收 |
|---|---|
| M01 | M0-01~11、#1、#2 |
| M02 | M0-12、M0-04 |
| M03 | M0-06（衍生） |
| M04 | M0-16、#12 全部 |
| M05 | M0-14、#6 |
| M06/M07/M08/M09/M10 | M1-R1~R12、#3~#7 |
| M11 | M1-C1~C8、#11 |
| M12 | M2-R1~R14、#8 |
| M13 | M2-F1~F16、#9、#10 |
| `bridge.js`（轮转） | #10 |
