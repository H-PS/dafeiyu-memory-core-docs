# 00 · V2.4.2 修订版总摘要

- 修订时间：2026-10-31 17:15
- 修订依据：原 V2.4.2 设计（00~09）+ `MEMORY-V2.4.2-INDEPENDENT-REVIEW.md` + `D:\qq-bridge` 真实源码
- 本版本：**设计修订**，未修改任何项目源码/配置/数据，未启动服务，未实施 M0

---

# 1. 本版相对原 V2.4.2 的实质改动

> 第一轮 R1~R9（下表）；第二轮追加 R10~R16（详见 `12-CHANGELOG.md`）：
> R10 P 层 per-key 版本 · R11 A 层 extra + category 校验 · R12 interactionCount/lastSeenAt 归 S ·
> R13 metaRev（元数据并发乐观锁）· R14 M0 双写 effect 协议 · R15 删除版本/traits 淘汰/检索过滤 · R16 源码再核对（N-06/N-07）

| # | 原设计 | 修订后 | 依据 |
|---|---|---|---|
| **R1** | `rev` 单一计数器（内容+元数据+乐观锁混用），D-17 建议"任何字段变更都 rev+1" | **拆 `rev`（全量版本）+ `contentRev`（内容版本）**；乐观锁与分享绑定改用 contentRev | 独立报告 §5.3 / §10.1：embedding 刷新会让 pending edit 被误 superseded |
| **R2** | `memberImpressions["名字"] = {memoryId, rev, traits[…]}`（1:1） | **traits 数组元素升级为对象** `{memoryId, contentRev, text, at}`（1:N），按 memoryId 精确增删改 | 独立报告 §3.2 误判②：源码 `bridge.js:9251-9269` 是数组追加 |
| **R3** | "B 层 = 投影，可全量重建" | **拆三层：A（事实源）/ P（记忆投影，新文件）/ S（运行状态，不可重建）** | 独立报告 §2.2：`saveSocialV2State()` 43 处调用，39 处与记忆无关 |
| **R4** | Reconciler 未定义读-改-写边界 | **P 层独立成文件**，Reconciler 是唯一写者，与 S 层零交集 | 独立报告 §6.2 / §10.4：整文件重写会互相覆盖 |
| **R5** | 无 `requestedAt`/`requestSeq`/`commitSeq` | **新增三字段 + clear 屏障语义（待决策）** | 独立报告 §10.5：embedding 队列外导致提交顺序 ≠ 请求顺序 |
| **R6** | M0 把 F-02/F-08/F-11 留给 M1 | **F-02/F-08（三态）/F-11（最小恢复校验）上移 M0**；M0 关闭 ShareRequest | 独立报告 §9.3：M0 不具备上线条件 |
| **R7** | 心跳在 S4 启动 | **心跳在 S1 lock_acquired 后立即启动**；S4 不重复 | 独立报告 §4.3 缺口①：S2 recovery 超 GRACE 会被误接管 |
| **R8** | writer.lock 声称保护 bridge vs 脚本 | **明确其真实职责：脚本 vs 脚本互斥**；bridge vs 脚本靠强制停服 | 独立报告 §4.2：bridge 全程持锁 ⇒ 脚本必等 bridge 停 |
| **R9** | 队列无界；`drain(5s)` 与 P1(30s) 冲突 | **队列上限 + 背压 + 取消语义 + drain 与 P1 对齐** | 独立报告 §4.1 |

---

# 2. 三层数据所有权（本版核心架构变更）

```
┌─────────────────────────────────────────────────────────────┐
│ A 层  state/memory-store.jsonl                               │
│   角色：长期记忆【事实源】（单点权威）                        │
│   内容：id / rev / contentRev / key / category / subjectId /  │
│         text / vec / lifecycle / visibility / suppression /   │
│         shareCopies / embedding / origin / _crc               │
│   写者：仅 MemoryService WriteQueue                           │
│   重建：不可（丢失即永久丢失）                                │
└──────────────────────┬──────────────────────────────────────┘
                       │ 单向派生（A → P）
┌──────────────────────▼──────────────────────────────────────┐
│ P 层  state/memory-projection.json          ★ 本版新增        │
│   角色：记忆【投影】（可全量重建）                            │
│   内容：{ storeVersion, generatedAt, conversations: {        │
│           "<key>": { activeTopics[], pendingThoughts[],      │
│                      memberImpressions{} } } }                │
│   写者：仅 Reconciler（含 WriteQueue 内的 B-effect）          │
│   重建：可从 A 全量重建（幂等）                               │
│   元数据：storeVersion 用于判断是否过期                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ S 层  state/social-v2.json（保留，结构不变）                  │
│   角色：会话【运行状态】（不可从 A 重建）                     │
│   内容：recentMessages / unread / wakeConfig /               │
│         lastReadThroughSeq / agentToken / sendTimes /        │
│         wakeTimes / seenForwardIds / lastIncomingAt / …      │
│   写者：43 处运行状态写入（保持 V1 行为，不受 MS 管辖）        │
│   重建：不可（这些字段无 A 层对应来源）                       │
│   ★ 三字段（activeTopics/pendingThoughts/memberImpressions）  │
│     在 M1 迁移后从本文件移除，只留 P 层                       │
└─────────────────────────────────────────────────────────────┘
```

## 2.1 为什么必须拆出 P 层

| 问题 | 拆前 | 拆后 |
|---|---|---|
| Reconciler 与运行状态互相覆盖 | 同一文件，`atomicWriteJson` 整文件重写 ⇒ 后写者赢 | P 与 S 不同文件，零交集 |
| "B 可重建"表述失真 | 声称整个 social-v2.json 可重建（错误） | 只有 P 可重建，S 明确不可 |
| 迁移影响面 | 需改 43 处调用 | 43 处不动，只新增 P 文件 |
| 回滚 | 难（合在一起） | 只需删 P 文件，S 完全不受影响 |

## 2.2 兼容性代价

- `formatMemoryV2()` 需改为从 P 层读三字段（P 为空时降级到 S 层旧字段，兼容迁移期）
- `memory-projection.json` 是新增文件，M0 迁移时生成一次
- S 层的三字段在 M1 才移除（M0 保留双写兼容）

---

# 3. 核心设计原则（修订版）

1. **单一写路径**：A 层写只能经 MemoryService；P 层写只能经 Reconciler（含 WriteQueue 的 B-effect）；S 层不受 MS 管辖。
2. **A 是事实源，P 是投影，S 是运行状态**：三者不可混称"B 层"。
3. **写前重读**：写队列内每步从最新 store 定位，绝不复用请求进入时的数组/索引。
4. **三版本**：`rev` 管全量，`contentRev` 管内容，`metaRev` 管元数据；**乐观锁按操作类型各用对应版本**。
5. **旧 contentRev 不得覆盖新 contentRev**；元数据变更不阻断内容操作。
6. **恢复不得绕过 tombstone**；备份恢复默认跳过已删。
7. **A/P 不得互相递归触发写入**：P 更新只能是 A 操作的 effect。
8. **正文持久化与向量可用性解耦**：`embedding.status ∈ {pending, ready, failed}`，正文优先落盘。
9. **分级审计**：Tier-1（删除/授权/恢复）失败即拒绝；Tier-2 走补偿。
10. **不引入新存储引擎**：JSONL + 文件锁。
11. **请求顺序可追溯**：`requestSeq` / `commitSeq` / `requestedAt` / `committedAt` 四字段。
12. **M0 必须可上线**：隐私（脱敏）、正文不丢、恢复不绕墓碑，三者缺一不可。

---

# 4. A 层记录 Schema（V2 修订版）

```json
{
  "id": "mem_01J8Z…",                  // ULID，全局唯一，永不回收
  "rev": 7,                             // 全量版本：任何持久化字段变化 +1
  "contentRev": 3,                      // 内容版本：仅 text 变化 +1
  "metaRev": 4,                         // 元数据版本：extra/lifecycle/visibility/shareCopies/suppression ★本版新增
  "key": "group:123456",                // canonical
  "category": "activeTopic",            // activeTopic|pendingThought|memberImpression
  "subjectId": null,                    // memberImpression 必填
  "factKey": null,                      // M2 启用，M0/M1 为 null
  "text": "…（脱敏 + 截断 ≤1000 字）",
  "extra": {                            // ★本版新增（原先缺失）
    "participants": ["甲", "乙"],        // 仅 activeTopic；≤10 项，每项 ≤32 字
    "pendingQuestion": "",              // 仅 activeTopic；≤200 字
    "motivation": "curiosity"           // 仅 pendingThought；≤50 字
  },
  "owner": "group:123456",
  "visibility": "owner_private",
  "lifecycle": "active",                // active|expired|evicted|suppressed|pending_delete
  "vec": [ …1024 floats ],
  "vecModel": "bge-m3",
  "vecDim": 1024,
  "vecContentRev": 3,                   // 向量对应的 contentRev（防覆盖新正文）
  "embedding": { "status": "ready", "attempts": 0, "nextRetryAt": null, "lastError": null },
  "origin": null,                       // 分享副本：{ "id": "mem_原件", "contentRev": 2 }
  "expiresAt": null,                    // pendingThought 专用（顶层，不从 extra 派生）
  "shareCopies": [ { "targetKey": "group:999", "status": "approved", "requestId": "req_…", "copyId": "mem_…" } ],
  "suppression": null,                  // { scope:'text'|'vec'|'all', at, by, reason }
  "createdAt": 1730…, "updatedAt": 1730…, "statusChangedAt": 1730…,
  "migratedFrom": null,
  "opId": "op_…",                       // 产生本记录的操作
  "_crc": "sha256:…"
}
```

**关键新增字段**：
- `contentRev`：内容版本，乐观锁与分享绑定
- `vecContentRev`：向量对应的内容版本 ⇒ **embedding 结果不得覆盖更新的正文**
- **`metaRev`**：元数据版本 ⇒ **并发编辑元数据必须检出冲突**（见 §4.3）
- **`extra`**：承载 `participants` / `pendingQuestion` / `motivation`（原设计缺失，见 §4.0）

## 4.0 `extra` 字段与 category 校验（本版新增）

**源码事实**（`SOURCE_CONFIRMED`）：B 层有三个字段在 A 层无对应位置——`participants`（`bridge.js:9221-9230`、`:6794`）、`pendingQuestion`（`:9225-9231`、`:6793`）、`motivation`（`:9241-9247`、`:6799`）。原设计 A Schema 缺这三个字段，导致"B 完全由 A 投影"不可行。

### 校验矩阵（append 与 edit 共用 `validateExtra(category, extra)`）

| 字段 | activeTopic | pendingThought | memberImpression | 约束 |
|---|---|---|---|---|
| `participants` | ✅ | ❌ 拒绝 | ❌ 拒绝 | 去重 → `slice(0,10)` → 每项 `slice(0,32)` + `redact` |
| `pendingQuestion` | ✅ | ❌ 拒绝 | ❌ 拒绝 | `redact` → `slice(0,200)` |
| `motivation` | ❌ 拒绝 | ✅ | ❌ 拒绝 | `redact` → `slice(0,50)`，默认 `'curiosity'` |
| `expiresAtMs` | ❌ 拒绝 | ✅ | ❌ 拒绝 | 正整数；默认 2h；落到顶层 `expiresAt` |
| `target` | ❌ 拒绝 | ❌ 拒绝 | ✅ | `slice(0,64)`，同时作为 `subjectId` |

**拒绝语义**：传了该 category 不允许的字段 → **400 `EXTRA_NOT_ALLOWED`**（不静默忽略，避免调用方误以为生效）。

### 统一约束的原因（新证据 N-06）

`SOURCE_CONFIRMED` 的三处路径不一致：

| 字段 | append 路径 | update 路径 | 差异 |
|---|---|---|---|
| `participants` | `:9223` `[...set].slice(0,10)` | `:6794` **直接赋值，无上限** | update 可写任意多条 |
| `pendingQuestion` | `:9225` `truncateText(...,200)` | `:6793` **`String(...)` 无截断** | 可写任意长 |
| `motivation` | `:9241` `.slice(0,50)` | `:6799` **`String(...)` 无截断** | 同上 |

⇒ 统一由 `validateExtra` 处理，两条路径调用同一函数。

## 4.1 版本递增矩阵（三版本，`[已确定]`）

| 操作 | rev | contentRev | metaRev | vecContentRev | 说明 |
|---|---|---|---|---|---|
| `append` | 1 | 1 | 1 | 1（若 vec 就绪）/0 | 新记录 |
| `edit_text`（text 变） | +1 | **+1** | 不变 | 不变（待重算） | 内容变更 |
| `edit_meta`（extra 变） | +1 | **不变** | **+1** | 不变 | 元数据变更 |
| `edit_subject`（subjectId 变） | +1 | 不变 | **+1** | 不变 | 元数据变更 |
| `embedding_refresh` | +1 | 不变 | 不变 | **= contentRev** | 只在 `vecContentRev < contentRev` 时 |
| `shareCopies` 增删 | +1 | 不变 | **+1** | 不变 | 授权登记属元数据 |
| `share_approve`（原件） | +1 | 不变 | **+1** | 不变 | 同上 |
| `suppression`（scope=vec） | +1 | 不变 | **+1** | 不变 | 元数据 |
| `suppression`（scope=text/all） | +1 | **+1** | **+1** | 不变 | 正文变了 |
| `lifecycle` 迁移 | +1 | 不变 | **+1** | 不变 | 状态变更 |
| `remove` | +1 | 不变 | 不变 | 不变 | 记录移出 store |
| `restore` | +1 | 不变 | **+1** | 不变 | 状态变更 |

## 4.2 乐观锁检查矩阵（三版本，`[已确定]`）

| 操作 | 校验字段 | 说明 |
|---|---|---|
| `edit_text` | **`expectedContentRev`** | 只关心内容不被并发改 |
| `edit_meta` | **`expectedMetaRev`** | ★并发改元数据必须检出冲突（§4.3） |
| `edit_subject` | **`expectedMetaRev`** | 同上 |
| `replace_extra` | `expectedRev` | 整体替换元数据，最强操作 |
| `remove` | **`expectedContentRev`** | 用户视角是"删这条内容" |
| `share_request` | **`expectedContentRev`** | 绑定内容版本 |
| `share_approve` | **`expectedContentRevAtRequest`** | 原件内容必须未变 |
| `restore` / `force_restore` | `expectedRev` | 整条记录级强操作 |
| `suppress` / `unsuppress` | **`expectedMetaRev`** | 安全属性属元数据 |

**安全属性（visibility/lifecycle/suppression/tombstone）在审批与恢复时**一律重新核验**，不因采用 contentRev 而降低强度。

## 4.3 为什么必须有 `metaRev`（并发编辑元数据）

用 `expectedContentRev` 校验 `edit_meta` 会**静默丢更新**：

```
t0  M: contentRev=2, metaRev=3, participants=["甲"]
t1  edit_meta A（participants → ["甲","乙"]），expectedContentRev=2
t2  edit_meta B（pendingQuestion → "要不要试试"），expectedContentRev=2
t3  A 提交：participants=["甲","乙"]，contentRev 仍=2，metaRev→4
t4  B 提交：contentRev 仍=2 匹配 → 通过
      若 B 做"整体替换 extra" → A 的 participants 被覆盖 ❌
      若 B 做"字段级 patch" → 保留 A 的 participants ✅（但依赖实现纪律）
```

**两个修法叠加**：
1. **`metaRev` 版本**：两个并发 edit_meta 基于同一 metaRev → 后到者 **409**，冲突显式暴露
2. **patch 语义**：`edit_meta` 只改**显式传入**的字段；未传的保留；传 `[]`/`""` 表示显式清空

| 语义 | 正确 | 错误 |
|---|---|---|
| 只传 participants | 只改 participants | 用 `{}` 覆盖整个 extra |
| 传 participants=[] | 清空 participants | 视为"未传" |

**整体替换**：用独立 op `replace_extra`（校验 `expectedRev`）。

# 5. P 层结构（记忆投影）

`state/memory-projection.json`：

```json
{
  "schemaVersion": 1,
  "globalStoreVersion": 143,        // 仅用于外部写入侦测/备份比对，不用于 staleness 判定
  "generatedAt": 1730000000000,
  "conversations": {
    "group:123456": {
      "convVersion": 42,             // ★该会话的投影版本（per-key，见 §5.2）
      "generatedAt": 1730000000000,
      "activeTopics": [
        { "memoryId": "mem_A", "contentRev": 2, "text": "…前200字…",
          "lastMentionAt": 1730…, "participants": ["甲"], "pendingQuestion": "" }
      ],
      "pendingThoughts": [
        { "memoryId": "mem_B", "contentRev": 1, "text": "…", "createdAt": 1730…,
          "expiresAt": 1730…, "motivation": "curiosity" }
      ],
      "memberImpressions": {
        "某群友": {
          "traits": [
            { "memoryId": "mem_C", "contentRev": 1, "text": "喜欢吃辣", "at": 1730… },
            { "memoryId": "mem_D", "contentRev": 1, "text": "怕冷", "at": 1730… }
          ]
        }
      }
    }
  }
}
```

**★ 与前一稿的差异**：`interactionCount` / `lastSeenAt` 已**移出 P 层**（见 §5.1），P 层成员印象只剩 `traits[]`。

## 5.1 memberImpressions 一对多映射（P0-2 核心）

| 维度 | 规则 |
|---|---|
| **A→P** | 每条 `category=memberImpression` 的 A 记录 → P 层该 subject 的 `traits[]` 一个元素（携带 memoryId/contentRev） |
| **删除** | 删 A 记录 → 按 `memoryId` 从 traits 移除；traits 空则**保留空对象**（便于后续 append 续用） |
| **编辑** | 改 A 记录 text → 按 `memoryId` 更新 `traits[i].text` + `contentRev` |
| **排序** | traits 按 `at` 升序（稳定） |
| **去重** | 同 `memoryId` 只保留一条（幂等投影） |
| **traits 上限** | **10 条**（统一 V1 `:9257` 的 10 与 `:6815` 的 20，取小值防膨胀） |
| **traits 超限** | **P 层裁剪**（移除 `at` 最旧的），**A 层记录不动**；M1 才做 A 层 `lifecycle='evicted'` |
| **对象上限** | 50 个名字，超限按 S 层 `lastSeenAt` 淘汰（保持 V1 行为） |

## 5.1.1 ★ `interactionCount` / `lastSeenAt` 的最终归源（本版修订）

**结论：迁到 S 层，不放 P 层。**

**源码事实**（`SOURCE_CONFIRMED`）：
```
interactionCount 仅 3 处：:9260（append 时 +1）、:6817（update 可改）、:9199（formatMemoryV2 读显示）
lastSeenAt       仅 3 处：:9261（append 设为 now）、:6824 / :9265（淘汰排序）
```
两字段**没有任何"记忆内容"语义**，纯粹是互动统计。

**归属决定**：

| 字段 | 归属 | 理由 |
|---|---|---|
| `traits[]`（含 memoryId/contentRev/text/at） | **P 层** | 是记忆内容的投影，可从 A 重建 |
| `interactionCount` | **S 层** | 运行统计，**不可从 A 重建**（A 无此语义） |
| `lastSeenAt` | **S 层** | 同上 |

**S 层新增子字段**（放在每个会话的运行状态区）：

```json
"impressionStats": {
  "某群友": { "interactionCount": 5, "lastSeenAt": 1730000000000 }
}
```

**为什么不能留在 P 层**：P 层是可重建的。若 P 全量重建（`rebuild-projection`），`interactionCount` 会被清零——而它本应保留（用户互动 5 次的事实不因投影重建而消失）。把它放进"不可重建"的 S 层，语义才正确。

**迁移影响**：
| 阶段 | interactionCount 位置 |
|---|---|
| M0 迁移 | 从 V1 `social-v2.json` 的 `memberImpressions[name].interactionCount` **原位保留**（S 层内），P 层不复制 |
| M0 双写 | append 时：P 写 traits + S 写 stats（两处） |
| M1 | 同上，S 层三字段移除后，stats 仍在 S |

**写入点改造**（M0 必须）：
```
append memberImpression:
  ① P 层：traits.push({memoryId, contentRev, text, at})
  ② S 层：impressionStats[name].interactionCount += 1; .lastSeenAt = now
  ★ 两处写在同一 WAL op 的两个 effect（A/P 为 required，S-stats 为 required）
     —— 与"三字段兼容副本"不同，stats 是唯一来源，故为 required
```

**`formatMemoryV2` 显示"互动 N 次"时**：从 S 层 stats 读，不读 P。

## 5.2 P 层过期判定（★改为 per-key）

**前一稿的问题**：用全局 `storeVersion` 判定

```js
// ❌ 前一稿：一个 key 更新 → 全局 storeVersion+1 → 所有 key 都被判 stale
isProjectionStale = (projection.storeVersion !== store.storeVersion)
```

后果：任意一条记忆写入，都会让**所有会话**的投影被判为过期 → 下次读触发全量重建（O(N) 重建所有会话）→ 与"增量更新"的初衷矛盾。

**修订：per-key `convVersion`**

```js
// ✅ 本版：每个会话独立版本
isStale(key) =
  !projection.conversations[key]
  || projection.conversations[key].convVersion !== keyVersion.current(key)
```

**`keyVersion` 的来源与持久化**：

| 项 | 设计 |
|---|---|
| 内存结构 | `Map<key, number>`，Reconciler 维护 |
| 递增时机 | 该 key 的三字段发生任何变化（append/edit/remove/clear/share_* 的 P-effect 完成时） |
| 启动恢复 | 从 P 文件的 `conversations[key].convVersion` 装入 Map（P 是权威） |
| P 缺失时 | 从 A 层重建该 key → `convVersion = 1` |
| WAL 关联 | P-effect 携带 `targetConvVersion`；重放时若 P 的 convVersion 已 ≥ 目标值则跳过 |

**全局 `globalStoreVersion` 的用途**（不用于 staleness）：
- 备份 envelope 的 `storeVersion` 字段（比对恢复前后）
- 外部写入侦测（`writer.lock` 空闲但 store 变化 → 触发全量重载）
- `/api/status` 的可观测指标

**效果对比**：

| 场景 | 前一稿（全局） | 本版（per-key） |
|---|---|---|
| 群 A 写入 1 条 | 所有会话投影判 stale | 只有群 A 的 convVersion+1 |
| 下次读群 B | 触发群 B 全量重建（浪费） | 不重建（convVersion 未变） |
| 10 个会话，每 5 分钟各写 1 条 | 每次写入后所有会话都要重建 | 各自只重建自己的 |

## 5.3 P 层重建的三种粒度

| 粒度 | 触发 | 代价 | 使用场景 |
|---|---|---|---|
| **增量更新** | 每次 A 提交 | O(1) | 正常路径 |
| **单会话重建** | `isStale(key)` 为真 | O(该会话记录数) | 读时惰性刷新 |
| **全量重建** | 显式 `POST /api/memory/rebuild-projection` 或 P 文件损坏 | O(全部) | 初始化/迁移/修复 |

**全量重建时 `convVersion` 的处理**：所有会话置为 `1`，并重置内存 `keyVersion`（与 P 一致）。

---

# 6. S 层（social-v2.json）职责边界

## 6.1 字段归属表（`SOURCE_CONFIRMED`，基于 `state/social-v2.json` 实测）

| 字段 | 归属 | 可由 A 重建 | 写者 |
|---|---|---|---|
| `activeTopics` | **P 层**（M1 后从 S 移除） | ✅ | Reconciler |
| `pendingThoughts` | **P 层**（M1 后从 S 移除） | ✅ | Reconciler |
| `memberImpressions`（traits 部分） | **P 层**（M1 后从 S 移除） | ✅ | Reconciler |
| **`impressionStats`**（★本版新增）<br>`{name: {interactionCount, lastSeenAt}}` | **S 层**（不可重建） | ❌ | append 路径 + update 路径 |
| `recentMessages` | S 层 | ❌ | 消息循环 |
| `unread` | S 层 | ❌ | 收消息/已读 |
| `wakeConfig` | S 层 | ❌ | `/api/socialV2/wake` |
| `lastReadThroughSeq` / `lastUnreadSeq` | S 层 | ❌ | 已读水位 |
| `agentToken` | S 层 | ❌ | 会话创建 |
| `sendTimes` / `wakeTimes` / `stickerCollectTimes` | S 层 | ❌ | 限流 |
| `seenForwardIds` | S 层 | ❌ | 转发去重 |
| `lastIncomingAt` / `lastAiReplyAt` / `lastActionAt` | S 层 | ❌ | 时序 |
| `preSleepWait*` | S 层 | ❌ | 睡前观察 |
| `bootstrapSent` / `lastWakeReason` | S 层 | ❌ | 唤醒 |

**结论**：S 层承载 **19 类**字段（原 18 + 新增 `impressionStats`），**无一个可从 A 层重建**。

## 6.2 M0/M1 的过渡策略

| 阶段 | S 层三字段（activeTopics/pendingThoughts/memberImpressions） | S 层 `impressionStats` | P 层 |
|---|---|---|---|
| M0 | 保留（**兼容副本**，可选写） | **唯一来源**（required effect） | 新建并写（**权威**） |
| M1 迁移 | **移除** | 保留（永久） | 唯一来源 |
| M1 之后 | 不含 | 保留 | 唯一来源 |

**关键区分**（M0 双写的两种地位）：

| 数据 | M0 的地位 | WAL effect 等级 |
|---|---|---|
| P 层三字段 | **权威**（`formatMemoryProjection` 优先读） | `required` |
| S 层三字段 | **兼容副本**（P 缺失时才降级读） | **`optional`**（失败仅告警） |
| S 层 `impressionStats` | **唯一来源**（无 P 副本） | **`required`** |

**为什么 S 三字段是 optional 而 stats 是 required**：
- 三字段在 P 有权威副本 ⇒ S 写失败不会丢数据 ⇒ 可降级
- stats 只有 S 一处 ⇒ S 写失败即丢数据 ⇒ 必须 required

## 6.3 M0 双写的完整协议（★本版新增）

M0 的每次记忆写操作，effects 按以下结构构造（以 `append memberImpression` 为例）：

```json
{
  "opId": "op_append_1",
  "opType": "append",
  "effects": [
    { "effectId": "e1", "layer": "A", "kind": "insert", "id": "mem_new",
      "requirement": "required", "set": { "…完整记录…" } },
    { "effectId": "e2", "layer": "P", "kind": "project",
      "convKey": "group:123", "targetConvVersion": 43,
      "requirement": "required",
      "set": { "traits": [ … ] } },
    { "effectId": "e3", "layer": "S_stats", "kind": "merge",
      "convKey": "group:123", "targetKey": "某群友",
      "requirement": "required",
      "set": { "interactionCount": "+1", "lastSeenAt": "now" } },
    { "effectId": "e4", "layer": "S_compat", "kind": "project",
      "convKey": "group:123", "targetKey": "某群友",
      "requirement": "optional",
      "set": { "traits": [ … ], "interactionCount": 5, "lastSeenAt": … } },
    { "effectId": "e5", "layer": "audit", "requirement": "optional" }
  ]
}
```

**committed 条件**：`e1 + e2 + e3` 全部 `done` 且 verified（`e4`/`e5` 失败不影响）。

**失败补偿协议**：

| 失败组合 | 状态 | 行为 | 用户可见 |
|---|---|---|---|
| e1 失败（A 未写） | `failed_retriable` | 重试 e1；e2/e3 不执行 | 500，记忆未写 |
| e1 成功、e2 失败（P 未写） | `partial` | 重试 e2（e1 幂等跳过） | 500 partial；**A 有记录但 P 无投影** ⇒ 读时 `isStale` 触发重建 ⇒ 自愈 |
| e1+e2 成功、e3 失败（stats 未写） | `partial` | 重试 e3 | 500 partial；互动计数少 1（可在 M1 用 `repair-stats` 修） |
| e1+e2+e3 成功、e4 失败 | `committed` + `warn` | 记 warn，不重试 | 200 normal（S 兼容副本过期，P 为准） |
| e5（审计）失败 | `committed` + `auditWarn` | 进补偿队列 | 200 normal |

**为什么 e2 失败能在读时自愈**：`isStale(key)` 检查 `convVersion`，P 未更新则 convVersion 落后 → 下次读触发单会话重建 → 从 A 补齐。

**为什么 e3 失败不能自愈**：stats 无 P 副本，A 也不记录互动次数 → 只能重试或人工修复。这是 stats 必须 `required` 的原因。

---

# 7. 请求顺序与提交顺序（P0-5）

## 7.1 四字段定义（`[已确定]`）

| 字段 | 分配时机 | 持久化位置 | 用途 |
|---|---|---|---|
| `requestSeq` | 请求进入 HTTP 端点、通过鉴权后立即分配（进程内单调） | WAL 条目 | 确定请求先后 |
| `commitSeq` | P3 提交时分配（= WAL seq） | WAL 条目 | 确定提交先后 |
| `requestedAt` | 随 `requestSeq` 一起记录 | WAL 条目 | 人类可读时间 |
| `committedAt` | 随 `commitSeq` 一起记录 | WAL 条目 | 人类可读时间 |

**`requestSeq` 的重启恢复**：存 `state/memory-request-seq`（单行数字），启动时读取并 +1；WAL 中更大的 `requestSeq` 优先。

## 7.2 顺序语义回答

| 问题 | 答案 |
|---|---|
| append 先请求、clear 后请求，但 append 的 embed 后完成 | **取决于 clear 屏障决策（D-21，待批准）**：<br>· 方案 A（屏障）：append.requestSeq < clear.requestSeq ⇒ clear 提交后 append 丢弃（WAL `aborted(reason=barriered)`）<br>· 方案 B（V1 语义）：append 正常提交，clear 不阻止 |
| edit 先请求、edit 后请求，embed 完成顺序相反 | **两 edit 的 expectedContentRev 相同 ⇒ 先提交者赢，后者 409**。但"先提交"= 先完成 embed，与请求顺序无关 ⇒ **这是当前设计的固有语义**。若需按请求顺序，须在 P2 入队时按 `requestSeq` 排序（会牺牲并行度） |
| clear 是否是请求顺序屏障 | **待决策 D-21**（推荐方案 A） |
| clear 之前已请求但未完成 embedding 的操作能否在 clear 后提交 | 同 D-21 |
| 服务重启后如何保留顺序语义 | `requestSeq` 从 `state/memory-request-seq` + WAL 最大值恢复 |
| WAL 如何记录 | 每条 WAL 含 `requestSeq` / `commitSeq` / `requestedAt` / `committedAt` |

## 7.3 为什么必须记录但不能强制排序

- **记录**是必须的：否则崩溃恢复后无法判断"哪个请求先到"，也无法审计。
- **强制按 requestSeq 排序**会牺牲并行度（一个慢 embed 阻塞后续所有 op），与 P1 队列外设计的初衷冲突。
- 因此本版：**记录顺序 + 提供屏障机制（可选）**，不强制全局排序。

---

# 8. M0 / M1 / M2 重新划分（P0-6）

## 8.1 M0 必须包含（不可再降级）

| 能力 | 缺陷 | 理由 |
|---|---|---|
| 写队列 + memoryId + rev/contentRev | F-01 | 数据正确性地基 |
| **A 层脱敏 + 统一截断 1000** | F-02 | **隐私**：未脱敏原文不得入库 |
| 两级审计（Tier-1 阻断） | F-05 | 不可逆操作必须可追责 |
| 原子写（随机 tmp + fsync + renameWithRetry） | F-06/F-15 | 数据完整性 |
| **embedding 三态（pending/ready/failed）+ 正文优先持久化** | F-08 | **正文不得因 embed 失败丢失** |
| **tombstone + 最小恢复校验** | F-11 | **人工恢复不得绕过墓碑** |
| 读失败抛错（不返回空） | F-13 | 防止静默清空 |
| canonical key | F-23 | 一致性 |
| `aRemoved` 上报 + partial 标记 | F-03 | 一致性可观测 |
| 坏行隔离到 `badlines` | F-12 | 防静默丢 |
| storeVersion 缓存失效 | F-16 | 防读旧 |
| 审计脱敏（contentHash）+ 轮转 | F-17 | 合规 |
| 记忆端点无条件三重鉴权 | F-20 | 纵深 |
| 备份文件名带时间戳 | F-26 | 恢复纵深 |
| **P 层（memory-projection.json）+ 最小 Reconciler** | 本版新增 | 三层划分的前提 |
| **bridge.lock 状态机（含心跳 + GRACE）** | F-07 | 双实例防护 |

## 8.2 M0 明确不含（M1）

| 能力 | 缺陷 | M0 的补偿 |
|---|---|---|
| B 淘汰联动 A（evicted） | F-04 | M0 在**读时**过滤 `lifecycle != active` + `expiresAt`，**不在写时淘汰** ⇒ 过期记录不再被召回（部分修复） |
| ShareRequest 化分享 | F-09 | **M0 关闭分享功能**（端点返回 501），避免用 V1 四级模糊匹配授权 |
| vecModel 强制隔离 | F-14 | M0 记录 vecModel，检索**校验维度**但不强制模型隔离（降级语义 M1） |
| 分层返回（B 成功 + A 降级） | F-22 | M0 保持 V1 行为（A 失败整段报错），但响应带 `degraded:true` |
| 备份 envelope + dry-run 恢复 UI | F-24 | M0 提供**最小恢复校验函数**（CLI），无 UI |
| 配置热更新 | F-25 | M0 需重启生效 |

## 8.3 M0 期间的用户知情项（必须写入交付说明）

```
① 分享功能关闭（qq_memory_share_request 返回 501）
② A/B 语义尚未完全统一（M0 双写兼容，P 为准；S 三字段是兼容副本）
③ 无图形化恢复流程，恢复需 CLI + 手工确认
④ 过期/淘汰在读时过滤，A 层记录不会主动删除（磁盘会缓慢增长）
⑤ 换 embedding 模型需手动触发全量重嵌入（无自动批量）
```

## 8.4 M2（后续）

factKey 完整模板、关系七型、副本批量重嵌入、ShareSuggestion、SQLite、WAL checkpoint 压缩、interactionCount 迁回 S 层。

---

# 9. 对十个最终问题的回答

**Q1 rev 混用是否解决** → ✅ 见 §4.1/§4.2/§4.3（rev + contentRev + metaRev 三版本，乐观锁按 opType 选字段）

**Q2 memberImpressions 一对多是否正确表达** → ✅ 见 §5.1（traits 元素携带 memoryId/contentRev）

**Q3 是否区分 P 层投影与 S 层运行状态** → ✅ 见 §2/§6（三层所有权表，18 字段无一可从 A 重建）

**Q4 是否避免 Reconciler 覆盖其他模块写入** → ✅ 见 §2.1（P 独立文件，与 S 零交集）

**Q5 是否定义请求顺序和提交顺序** → ✅ 见 §7（四字段 + 屏障机制）

**Q6 是否定义 clear 与 pending embedding 的关系** → ⚠️ **待决策 D-21**（推荐屏障语义）

**Q7 是否解决 M0 隐私与正文持久化风险** → ✅ F-02/F-08 上移 M0

**Q8 是否确保 M0 恢复不绕过 tombstone** → ✅ 最小恢复校验函数纳入 M0

**Q9 是否修复 bridge.lock 心跳和 GRACE** → ✅ 心跳 S1 启动；等待次数 = ceil(GRACE/5s)

**Q10 是否仍存在阻止实施的 P0** → ⚠️ 见 `FINAL-REVIEW.md`（D-21 待决策 + 2 项技术验证缺口）

---

# 10. 文档索引

| 文档 | 内容 |
|---|---|
| `01-MAPPING.md` | F-01~F-26 逐条映射（修订版）+ 本版新增发现 |
| `02-WRITE-CONCURRENCY.md` | 8 类 op、管线、三版本、请求顺序、10 组并发 |
| `03-AB-CONSISTENCY-WAL.md` | 三层所有权、P 层、WAL、恢复矩阵 |
| `04-LIFECYCLE-TOMBSTONE.md` | 生命周期、墓碑、级联、suppression |
| `05-SHARE-SECURITY.md` | SR、contentRev 绑定、审批、撤销 |
| `06-EMBEDDING-AUDIT-BACKUP.md` | 三态/六态、审计、备份、恢复 |
| `07-LOCK-AND-STARTUP.md` | 三锁职责（修订）、启动状态机、心跳 |
| `08-DECISIONS-REQUIRED.md` | D-01~D-21（含新增 D-21） |
| `09-ACCEPTANCE-CRITERIA.md` | 78 条验收标准 |
| `10-M0-ACCEPTANCE.md` | M0 独立验收清单 |
| `11-SOURCE-EVIDENCE.md` | 源码证据索引（文件:行号） |
| `12-CHANGELOG.md` | 逐项改动记录 |
| `FINAL-REVIEW.md` | 总体审核、未解决问题、是否可进入实施 |

---

*本版为设计修订。未修改任何项目源码/配置/数据；未启动服务；未实施 M0。*
