# 12 · 变更记录（相对原 V2.4.2）

> 逐项记录改动、原因、影响范围。只记实质变更，不记排版。

---

# R1 · 拆分 `rev` 与 `contentRev`

**问题**（独立报告 §5.3 / §10.1）
原设计 `D-17` 让 `rev` 承担三种语义（内容版本、元数据版本、乐观锁令牌），建议"任何字段变更都 rev+1"。后果：
1. 用户 edit 时后台 embedding 刷新 → rev+1 → edit 的 expectedRev 失配 → **409**
2. 批准群 A 分享（shareCopies+1 → rev+1）→ 群 B 的 pending SR 全部 superseded
3. WAL 重放时，高 rev 的 embedding_refresh 会让 pending edit 被判 superseded → **编辑静默丢失**

**改动**
| 字段 | 语义 |
|---|---|
| `rev` | 全量版本，任何持久化字段变化 +1 |
| **`contentRev`**（新增） | 内容版本，**仅 text 变化 +1** |
| **`vecContentRev`**（新增） | 向量对应的 contentRev |

**影响的文档**：`00 §4`、`02 §1/§4`、`04 §1/§6`、`05 §1/§3`、`06 §1.3`、`09 AC-2.x`、`11 E-01`

**规则改动**
- 乐观锁：`edit_*` / `remove` / `share_request` 用 `expectedContentRev`；`restore` / `suppress` 用 `expectedRev`
- 审批：`expectedContentRevAtRequest`
- WAL superseded：只比 contentRev
- 向量写入：`canWriteVec` 检查 `forContentRev === contentRev && vecContentRev < contentRev`

**安全强度不变**：`share_approve` / `restore` / `force_restore` 仍重新检查 tombstone / lifecycle / visibility / suppression（P0-1 要求 8）

**废弃**：D-17

---

# R2 · memberImpressions 一对多映射

**问题**（独立报告 §3.2 误判②）
原设计 `03 §1.1` 写 `memberImpressions["名字"] = {memoryId, rev, traits}`（1:1）。源码实际（`bridge.js:9258`）是 `{traits[], interactionCount, lastSeenAt}`，且每次 append 都新增一条 A 记录（1:N）。

**改动**
`traits` 元素升级为对象：`{memoryId, contentRev, text, at}`

**影响的文档**：`00 §5.1`、`03 §2.2`、`09 AC-3.8~3.10`

**新增规则**
| 操作 | P 层动作 |
|---|---|
| append | traits.push + interactionCount++ |
| edit_text | 按 memoryId 找 trait → 更新 text/contentRev |
| edit_subject | 跨 key 迁移 trait |
| remove | 按 memoryId 移除；traits 空则保留对象 |

**附加修复**（N-02）：traits 上限两条路径不一致（append 10 / update 20）→ **统一为 10**

**interactionCount 归属**：属 S 语义，M0/M1 暂存 P 层，M2 迁回

---

# R3 · 三层数据所有权（A / P / S）

**问题**（独立报告 §2.2 / §6.2）
原设计说"B 层 = 投影，可从 A 重建"。源码实测：`social-v2.json` 21 个字段中只有 3 个是记忆投影，18 个是运行状态（`recentMessages`/`unread`/`wakeConfig`/`agentToken` 等），**无法从 A 重建**。

**改动**
| 层 | 文件 | 角色 |
|---|---|---|
| A | `memory-store.jsonl` | 事实源 |
| **P**（新增） | **`memory-projection.json`** | 记忆投影（可重建） |
| S | `social-v2.json` | 运行状态（不可重建） |

**影响的文档**：`00 §2/§6`、`03 §1/§2`、`06 §3/§4`、`09 AC-3.x`、`10 §1`

**兼容策略**
- M0：P 新建 + **双写**（A 操作同时更新 P 与 S 三字段）；`formatMemoryProjection` 优先 P，缺失降级 S
- M1：S 层三字段移除
- 39 处运行状态写入**完全不动**（继续写 S）

**新增证据**：N-04（reset 整文件重写）、N-05（formatMemoryV2 读 S）

---

# R4 · Reconciler 并发边界

**问题**（独立报告 §10.4）
原设计未定义 Reconciler 的读-改-写边界。若它整文件重写 social-v2.json，会回退 39 处运行状态。

**改动**
- Reconciler **只在 WriteQueue 内执行**（P3 的 P-effect）
- P-effect 只写 `memory-projection.json`（与 S 零交集）
- 新增 `INo-9`：P-effect 内禁止 enqueue

**影响的文档**：`02 §2`、`03 §2.3/§2.4`

**三种模式**：增量更新（正常）/ 会话重建（P 损坏）/ 全量重建（迁移/显式）

---

# R5 · 请求顺序四字段 + clear 屏障

**问题**（独立报告 §4.1 / §10.5）
原设计 P1 在队列外并行 embed ⇒ 提交顺序 = embed 完成顺序 ≠ 请求顺序。设计中**完全没有** `requestedAt`/`requestSeq`/`commitSeq` 字段。

**改动**
| 字段 | 分配点 |
|---|---|
| `requestSeq` | P0（通过鉴权后） |
| `commitSeq` | P3.1（写 WAL） |
| `requestedAt` | 随 requestSeq |
| `committedAt` | 随 commitSeq |

**影响的文档**：`02 §1/§5`、`03 §3.1`、`08 D-21`

**新增机制**：clear 屏障（D-21，待批准）
- 方案 A（推荐）：`op.requestSeq < clear.requestSeq` 且未提交 ⇒ 丢弃
- 方案 B（M0 默认）：V1 行为，不屏障

**不强制排序的理由**：按 requestSeq 串行 embed 会让单个慢请求阻塞全部（`D-24` 待批准）

---

# R6 · 重新划分 M0/M1

**问题**（独立报告 §9.3）
原 M0 把 F-02（隐私）/ F-08（丢正文）/ F-11（恢复绕过墓碑）留给 M1，导致 M0 不具备上线条件。

**改动**
| 缺陷 | 原 | 现 |
|---|---|---|
| F-02 A 层脱敏 | M1 | **M0** |
| F-08 embedding | M1（六态） | **M0（三态）** + M1（六态） |
| F-11 恢复 | M1 | **M0（CLI 校验函数）** + M1（UI） |
| N-01 memory-update | 未列 | **M0 关闭端点** |

**影响的文档**：`00 §8`、`01 §3/§4`、`09`（全部标 M0/M1）、`10`（新建）

**M0 新增能力**：P 层 + 最小 Reconciler（因为投影是新的，迁移必须生成）

---

# R7 · bridge.lock 心跳与 GRACE

**问题**（独立报告 §4.3 缺口①②）
1. 心跳在 S4 启动 ⇒ S1~S3（尤其 S2 recovery）无心跳 ⇒ `age = now - startedAt` ⇒ recovery 超 90s 会被误接管
2. B2a "最多等 6 次" = 30s，但 GRACE = 90s ⇒ 30s 后行为未定义

**改动**
- 心跳在 **S1 lock_acquired 后立即启动**；S4 不重复
- `maxAttempts = ceil(GRACE/5000)`（90s → 18 次）；循环内**重读锁**，非固定睡眠
- 循环结束仍未决 → exit(3)「启动超时」

**影响的文档**：`07 §2.1/§3/§4.1`

**新增验收**：AC-9.2（A 在 S2 耗时 120s 但心跳正常 → B 不得接管）

---

# R8 · writer.lock 职责重新定义

**问题**（独立报告 §4.2）
原设计称 writer.lock 保护"bridge vs 脚本"。实际上 bridge 全程持有该锁 ⇒ 脚本必须等 bridge 停 ⇒ "bridge vs 脚本"由强制停服规则保证，**不是由 writer.lock**。

**改动**
| 机制 | 真实职责 |
|---|---|
| bridge.lock | Bridge 实例唯一性 |
| 进程内 FIFO | 同进程写入顺序 |
| **writer.lock** | **脚本 vs 脚本**互斥 |
| 强制停服（D-08） | bridge vs 脚本隔离 |

**影响的文档**：`07 §1.1`、`02 §8`

---

# R9 · 队列背压与 drain

**问题**（独立报告 §4.1）
1. 队列无界（内存风险）
2. `drain(5s)` 与 P1(30s) 冲突 ⇒ 正常退出会产生大量 `recovery_blocked`（污染恢复流程）

**改动**
- 队列上限 **64**，超出 → 429 `QUEUE_FULL`
- 取消：排队中可取消（P3 入口检查）；执行中不可取消
- drain 分两段；超时未执行的排队 op → **`aborted(shutdown)`**（不是 recovery_blocked）
- `recovery_blocked` 只用于"P3 中途 IO 失败"

**影响的文档**：`02 §6`、`07 §5.1`

---

# 第二轮追加修订（R10 ~ R16）

> 本段由用户第二轮反馈驱动，逐项对应 7 个要求。

## R10 · P 层全局 storeVersion → per-key convVersion

**问题**：前一稿 `isProjectionStale = projection.storeVersion !== store.storeVersion` 用**全局**版本。任意一条记忆写入都会让 storeVersion+1 ⇒ **所有会话**的投影被判过期 ⇒ 下次读触发全量重建。

**改动**：
| 项 | 前一稿 | 本版 |
|---|---|---|
| 判定字段 | `projection.storeVersion`（全局） | `conversations[key].convVersion`（per-key） |
| 递增范围 | 全局 | 仅该 key |
| 全局字段用途 | staleness 判定 | 仅外部写入侦测/备份比对 |

**影响文档**：`00 §5.2/§5.3`、`03 §1/§2.3/§2.4`

**新增机制**：内存 `keyVersion: Map<key,number>`，启动时从 P 的 convVersion 装入；WAL 的 P-effect 携带 `targetConvVersion`。

**验收**：AC-2.12

---

## R11 · A 层 `extra` 字段 + category 校验

**问题**：原设计的 A Schema 没有 `participants` / `pendingQuestion` / `motivation`，但 B 层有（`SOURCE_CONFIRMED`：`bridge.js:9227-9247`）。"B 完全由 A 投影"不可行。

**补充证据 N-06**：这三个字段在 `update` 路径**缺少 append 路径的约束**：
- `participants`：append `slice(0,10)`（:9223）vs update **无上限**（:6794）
- `pendingQuestion`：append `truncateText 200`（:9225）vs update **无截断**（:6793）
- `motivation`：append `slice(0,50)`（:9241）vs update **无截断**（:6799）

**改动**：
- A Schema 增加 `extra: {participants?, pendingQuestion?, motivation?}`
- 新增 `validateExtra(category, extra)` 统一函数，append 与 edit 共用
- 新增校验矩阵（`00 §4.0`）：每个 category 只允许特定字段，越界 → 400 `EXTRA_NOT_ALLOWED`
- `expiresAt` 保持 A 层顶层（不从 extra 派生）

**影响文档**：`00 §4/§4.0`、`02 §1`、`09 AC-2.10/2.11`、`11 N-06/N-07`

---

## R12 · `interactionCount` / `lastSeenAt` 从 P 层分离

**问题**：前一稿说"暂存 P 层，M2 迁回"。但 **P 层是可重建的** ⇒ 全量重建会清零 `interactionCount` ⇒ "互动 5 次"的事实丢失。

**补充证据**：两字段各仅 3 处读写（`SOURCE_CONFIRMED`），无任何"记忆内容"语义。

**改动**：
- 从 P 层**移除**（`memberImpressions[name]` 只剩 `traits[]`）
- 迁到 S 层新增 `impressionStats: {name: {interactionCount, lastSeenAt}}`
- append memberImpression 同时写 P（traits）与 S（stats），**两者都是 required effect**

**影响文档**：`00 §5.1.1/§6.1/§6.2/§6.3`、`03 §2.2.2`、`09 AC-2.13`

**语义澄清**：
| 数据 | 归属 | 可重建 |
|---|---|---|
| `traits[]` | P | ✅ |
| `interactionCount` | S | ❌ |
| `lastSeenAt` | S | ❌ |

---

## R13 · 新增 `metaRev`（元数据并发乐观锁）

**问题**：前一稿用 `expectedContentRev` 校验 `edit_meta`。两个并发 edit_meta 的 contentRev 都不变 ⇒ 都通过 ⇒ **后写覆盖前写**（若实现为整体替换 extra）。

**改动**：
- 新增第三版本 `metaRev`（extra/lifecycle/visibility/shareCopies/suppression 变化时 +1）
- `edit_meta` / `edit_subject` / `suppress` / `unsuppress` 改用 `expectedMetaRev`
- `edit_meta` 明确 **patch 语义**（只改显式传入字段；`[]`/`""` 表示显式清空）
- 新增 `replace_extra` op（整体替换，校验 `expectedRev`）

**三版本分工**：
| 版本 | 递增条件 | 用于 |
|---|---|---|
| `rev` | 任何字段 | WAL / 审计 / 幂等 |
| `contentRev` | text | edit_text / remove / share_* |
| `metaRev` | extra/状态/授权 | edit_meta / edit_subject / suppress |

**影响文档**：`00 §4/§4.1/§4.2/§4.3`、`02 §1/§4/§8.5`、`09 AC-2.7/2.8/2.9`

**新增并发案例**：`02 §8.5.1`（edit_meta∥edit_meta）、`02 §8.5.2`（edit_meta∥edit_text）

---

## R14 · M0 的 A/P/S 双写 effect 与失败补偿

**问题**：前一稿说 M0"双写"，但没定义 effect 结构、required/optional 等级、失败补偿路径。

**改动**（`00 §6.3`）：
| 数据 | M0 地位 | effect 等级 |
|---|---|---|
| A 层记录 | 权威 | `required` |
| P 层三字段 | **权威**（读优先） | `required` |
| S 层三字段 | **兼容副本**（降级读） | **`optional`** |
| S 层 `impressionStats` | **唯一来源** | **`required`** |

**committed 条件**：A + P + S-stats 全部 done 且 verified。

**失败补偿矩阵**：
| 失败组合 | 状态 | 后续 |
|---|---|---|
| A 失败 | `failed_retriable` | 重试 A，P/S 不执行 |
| A 成功、P 失败 | `partial` | 重试 P；**读时 `isStale` 自愈** |
| A/P 成功、S-stats 失败 | `partial` | 重试 stats；**不可自愈**（无副本） |
| S-compat 失败 | `committed` + warn | 不重试（P 为准） |

**影响文档**：`00 §6.2/§6.3`、`09 AC-2.15/2.16/2.17`、`02 §9`

---

## R15 · 删除版本、traits 淘汰、检索过滤

### R15.1 删除的版本规则（`02 §4.5`）

| 项 | 规则 |
|---|---|
| 校验字段 | `expectedContentRev`（不校验 metaRev） |
| 元数据已变 | 响应附 `warnings`，**不阻止**删除 |
| 批量删除 | 逐条校验；任一失配 → 整体 409 |
| tombstone | 记 `prevRev` + `prevContentRev` + **`prevMetaRev`** |

### R15.2 traits 裁剪 vs 真实淘汰（`03 §2.5.1`）

| 动作 | 作用层 | A 层记录 | 里程碑 |
|---|---|---|---|
| **投影裁剪**（traits > 10） | P | **不动** | M0 |
| **真实淘汰**（`lifecycle='evicted'`） | P + A | 最旧改 evicted | M1 |

M0 期间：A 层可能有 15 条，P 只显示 10 条，但检索仍能命中 15 条，`qq_memory_list` 能看到 15 条。

### R15.3 检索过滤（`03 §4.1`）

新增两个谓词：

```js
vectorSearchable(record) =            // 语义检索额外要求
     visible(record)
  && record.embedding.status === 'ready'
  && vec.length === vecDim
  && record.vecContentRev === record.contentRev   // ★向量不得落后正文

projectionVisible(entry) =            // P 层展示额外要求
     entry.contentRev === A 记录的 contentRev     // 投影不落后
  && 对应 A 记录 visible()
```

**影响文档**：`02 §4.5`、`03 §2.5.1/§4.1`、`04 §3.1/§3.2`、`09 AC-2.14`

---

## R16 · 源码再核对（新增 N-06 / N-07）

**核对范围**：`extra` 四字段全部读写点、`interactionCount`/`lastSeenAt` 全部读写点、`expiresAt` 全部过滤点。

**新增发现**：
| # | 发现 | 证据 |
|---|---|---|
| N-06 | extra 三字段在 update 路径缺约束 | `:6793/6794/6799` vs `:9223/9225/9241` |
| N-07 | A 层无 extra 载体 | `memory-vector.js:155-158` |

**新增证据编号**：E-46a~E-46d、E-47a~E-47c（`11 §2.4`）

**影响文档**：`11`

---

# 附：新增源码发现（N-01 ~ N-05）

| # | 发现 | 影响的文档 | 处置 |
|---|---|---|---|
| N-01 | `memory-update` 只改 B 层不碰 A 层（`6765-6831`） | `01 §3`、`03 §4` | M0 关闭端点（501）；M1 转发 `MS.edit_meta` |
| N-02 | traits 上限 10 vs 20 不一致 | `03 §2.2`、`09 AC-3.10` | 统一为 10 |
| N-03 | `memory-update` 支持 memberImpression 改名 | `08 D-22` | 允许，走 `edit_subject`（待批准） |
| N-04 | `reset` 整文件重写 social-v2.json（`4624`） | `08 D-23` | 只清 S 层，不清 A/P（待批准） |
| N-05 | `formatMemoryV2` 读 S 层三字段 | `03 §2.6` | 改为 `formatMemoryProjection`（P 优先，S 降级） |

---

# 文档变更清单

| 文档 | 变更 |
|---|---|
| `00-SUMMARY.md` | **重写**（三层架构、三版本、M0/M1 重划） |
| `01-MAPPING.md` | **重写**（F-01~F-26 更新 + N-01~N-05 新增 + 里程碑重划） |
| `02-WRITE-CONCURRENCY.md` | **重写**（三版本、请求顺序、10 组并发、背压） |
| `03-AB-CONSISTENCY-WAL.md` | **重写**（三层所有权、P 层、五档崩溃矩阵） |
| `04-LIFECYCLE-TOMBSTONE.md` | 修订（contentRev 与 suppression 关系、M0 简化态、D-09/D-15 待批） |
| `05-SHARE-SECURITY.md` | **重写**（contentRev 绑定、多群审批、M0 关闭） |
| `06-EMBEDDING-AUDIT-BACKUP.md` | 修订（三态/六态、vecContentRev、M0 CLI 恢复） |
| `07-LOCK-AND-STARTUP.md` | 修订（心跳 S1、GRACE 匹配、writer.lock 职责、drain 修正） |
| `08-DECISIONS-REQUIRED.md` | **重写**（D-01~D-24：2 废弃、2 修改、4 新增） |
| `09-ACCEPTANCE-CRITERIA.md` | **重写**（71 → 84 条，全部标 M0/M1） |
| `10-M0-ACCEPTANCE.md` | **新建** |
| `11-SOURCE-EVIDENCE.md` | **新建**（67 条证据 + 5 条推演 + 9 条不可验证） |
| `12-CHANGELOG.md` | **新建**（本文件） |
| `FINAL-REVIEW.md` | **新建** |

---

# 数字对比

| 项 | 原 V2.4.2 | 本版 |
|---|---|---|
| A 层字段 | 22 | **28**（+contentRev, +metaRev, +vecContentRev, +extra, +statusChangedAt） |
| 数据层数 | 2（A/B） | **3**（A/P/S） |
| 版本字段 | 1（rev） | **3**（rev, contentRev, metaRev）+ vecContentRev |
| WAL 崩溃点 | 4 | **5**（+P 层） |
| 并发场景 | 6 组 | **10 组**（+edit_meta∥edit_meta、edit_meta∥edit_text） |
| 验收条目 | 71 | **95** |
| 待决策 | 20 | **24**（2 废弃、4 新增） |
| M0 缺陷数 | 15 | **20** |
| 新增源码发现 | 0 | **7**（N-01~N-07） |
