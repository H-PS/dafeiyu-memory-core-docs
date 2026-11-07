# 03 · 三层一致性：A（事实源）/ P（投影）/ S（运行状态）+ WAL

解决 F-02/F-03/F-04、双写分叉（E-24~E-28）、N-01/N-05，以及本版新增的 A/B 所有权划分（R3/R4）。

---

# 1. 三层数据所有权（本版核心变更）

| 维度 | **A 层** `memory-store.jsonl` | **P 层** `memory-projection.json` | **S 层** `social-v2.json` |
|---|---|---|---|
| 角色 | **长期记忆事实源** | **记忆投影**（可重建） | **会话运行状态**（不可重建） |
| 拥有 | id/rev/contentRev/key/category/subjectId/text/vec/lifecycle/visibility/suppression/shareCopies/embedding/origin | 三字段：activeTopics / pendingThoughts / memberImpressions | recentMessages/unread/wakeConfig/lastReadThroughSeq/agentToken/sendTimes/wakeTimes/seenForwardIds/… |
| 唯一写者 | MemoryService WriteQueue | **Reconciler**（含 WriteQueue 的 P-effect） | 43 处运行状态写入 + reset（N-04） |
| 损坏后果 | 数据永久丢失 | 可从 A 重建 | 会话状态丢失（不影响记忆） |
| 重建 | ❌ | ✅（幂等，per-key `convVersion` 驱动） | ❌ |

## 1.1 为什么必须拆 P 层（`SOURCE_CONFIRMED`）

```
源码事实：
  bridge.js 中 saveSocialV2State() 调用 43 处，saveSocialV2StateStrict() 4 处
  其中仅 4 处（strict）涉及记忆三字段，39 处写运行状态
  social-v2.json 实测顶层字段 21 个，记忆三字段占 3 个，运行状态占 18 个

原设计"B 层=投影，可从 A 重建" ⇒ 对 18 个运行状态字段【不成立】
原设计"Reconciler 只改三字段" ⇒ 但 atomicWriteJson 是整文件重写
  ⇒ Reconciler 与 39 处运行状态写入必互相覆盖
```

拆分后：P 与 S 不同文件 ⇒ 零覆盖风险。

## 1.2 S 层字段归属（`SOURCE_CONFIRMED`，实测 `state/social-v2.json`）

| 字段 | 归属 | 可由 A 重建 |
|---|---|---|
| `activeTopics` | **P** | ✅ |
| `pendingThoughts` | **P** | ✅ |
| `memberImpressions`（traits） | **P** | ✅ |
| **`impressionStats`**（★新增）`{name:{interactionCount,lastSeenAt}}` | **S** | ❌ |
| `recentMessages`（100 条） | S | ❌ |
| `unread` | S | ❌ |
| `wakeConfig`（10 键） | S | ❌ |
| `lastReadThroughSeq` / `lastUnreadSeq` | S | ❌ |
| `agentToken` | S | ❌ |
| `sendTimes` / `wakeTimes` / `stickerCollectTimes` | S | ❌ |
| `seenForwardIds` | S | ❌ |
| `lastIncomingAt` / `lastAiReplyAt` / `lastActionAt` | S | ❌ |
| `preSleepWaitSatisfiedAt` / `preSleepWaitObservedAt` / `preSleepWaitAccumMs` | S | ❌ |
| `bootstrapSent` / `lastWakeReason` | S | ❌ |

**18 个运行状态字段，无一可从 A 重建。**

---

# 2. P 层结构与 Reconciler

## 2.1 文件结构

`state/memory-projection.json`：

```json
{
  "schemaVersion": 1,
  "globalStoreVersion": 143,          // 仅用于外部写入侦测/备份比对，不判定 staleness
  "generatedAt": 1730000000000,
  "conversations": {
    "group:123456": {
      "convVersion": 42,               // ★per-key 投影版本
      "generatedAt": 1730000000000,
      "activeTopics": [
        { "memoryId": "mem_A", "contentRev": 2, "text": "…≤200字…",
          "lastMentionAt": 1730…, "participants": ["甲"], "pendingQuestion": "" }
      ],
      "pendingThoughts": [
        { "memoryId": "mem_B", "contentRev": 1, "text": "…",
          "createdAt": 1730…, "expiresAt": 1730…, "motivation": "curiosity" }
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

## 2.2 A → P 的映射规则（P0-2 核心）

| 规则 | 说明 |
|---|---|
| **activeTopic** | 一条 A 记录 ↔ P 层该会话 `activeTopics[]` 一个元素 |
| **pendingThought** | 一条 A 记录 ↔ P 层该会话 `pendingThoughts[]` 一个元素 |
| **memberImpression** | **一条 A 记录 ↔ P 层该 subject 的 `traits[]` 一个元素**（携带自己的 memoryId/contentRev） |
| 文本 | P.text = A.text 前 200 字（同一 Normalizer 输出，仅展示截断不同） |
| 关联键 | **memoryId**（不得用文本相等） |
| 排序 | activeTopics/pendingThoughts 按 `lastMentionAt`/`createdAt` 升序；traits 按 `at` 升序 |
| 去重 | 同 memoryId 只保留一条 |
| 数量上限 | activeTopics 20 / pendingThoughts 20 / memberImpressions 50 个名字 / **traits 10 条** |
| 超限处理 | 投影层裁剪（移除最旧），**A 层记录不变**（M0）；M1 才做 A 层 lifecycle 迁移 |

### 2.2.1 memberImpressions 的增删改（P0-2 要求 2/3/4）

| 操作 | P 层动作 |
|---|---|
| append（新印象） | 若 subject 不存在 → 新建对象 `{interactionCount:1, lastSeenAt:now, traits:[{memoryId, contentRev, text, at}]}`；存在 → `traits.push(...)` + `interactionCount++` + `lastSeenAt=now` |
| edit_text | 按 memoryId 找 trait → 更新 `text`/`contentRev`；找不到 → 忽略（A 未变则 P 不应有） |
| edit_subject（改名） | 从旧 key 的 traits 移除该 memoryId → 插入新 key 的 traits；旧 key traits 空且 interactionCount 归零 → 删除对象 |
| remove | 按 memoryId 从 traits 移除；traits 空 → **保留对象**（interactionCount/lastSeenAt 仍在，属 S 语义） |
| traits 超 10 | 移除 `at` 最旧的 |
| 名字超 50 | 移除 `lastSeenAt` 最旧的 |

### 2.2.2 interactionCount / lastSeenAt 的归属（P0-2 要求 5）

**结论：迁到 S 层（`impressionStats`），不放 P 层。**

| 理由 | 说明 |
|---|---|
| A 层无对应语义 | A 记录只表达"一条印象"，不表达"互动了几次" |
| P 层是可重建的 | 若 P 全量重建时清零 `interactionCount`，则"互动 5 次"的事实丢失 |
| S 层不可重建 | 语义正确：互动统计属于运行状态，不属于记忆内容 |
| 源码证据 | `SOURCE_CONFIRMED`：`interactionCount` 仅 3 处（`:9260` +1 / `:6817` 可改 / `:9199` 读），`lastSeenAt` 仅 3 处（`:9261` / `:6824` / `:9265`） |

**S 层结构**：
```json
"impressionStats": { "某群友": { "interactionCount": 5, "lastSeenAt": 1730000000000 } }
```

**写入点**：`append memberImpression` 时同时写 P（traits）与 S（stats），两者都是 required effect（见 `00 §6.3`）。

**P 层角色**：只维护 `traits[]`（可重建），不含任何计数。

## 2.3 Reconciler 的三种模式

| 模式 | 触发 | 行为 |
|---|---|---|
| **增量更新**（正常路径） | 每次 A 提交后 | 只更新受影响的会话/条目 |
| **会话重建**（P 损坏/过期） | 读时 `projection.conversations[k].convVersion !== keyVersion.current(k)` | 从 A 重建**该会话**三字段；`convVersion` 置为当前 keyVersion |
| **全量重建**（初始化/迁移） | 显式 `POST /api/memory/rebuild-projection` | 从 A 重建全部；`interactionCount` 置 0（并在审计中记录） |

## 2.4 Reconciler 的并发边界（P0-4 核心）

```
★ Reconciler 只在 WriteQueue 内执行（P3 的 P-effect），不独立并发

即：P 层的每一次写入都是某个 A 操作的 effect：
  append/edit/remove/clear/share_* 的 P3.7 阶段
    → 构造 P 的目标态（仅该会话）
    → 读当前 P（同一个 store 读周期内）
    → 合并（保留其他会话 + 其他字段）
    → 原子写 P

因此：
  · 不需要独立的 P 锁（WriteQueue 已串行）
  · P 层不存在"Reconciler 与业务写入并发"（同一队列）
  · P 层与 S 层无交集（不同文件）⇒ 39 处运行状态写入不会覆盖 P
```

**崩溃恢复**：
- P 写失败 → WAL `partial`，重放 P-effect（幂等 set-by-state）
- P 文件损坏 → `convVersion` 缺失/不符 → 下次读触发**该会话**重建（不影响其他会话）
- `globalStoreVersion` 仅用于外部写入侦测与备份比对，不参与 staleness 判定

## 2.5 防止 A/P 互相触发（要求 6）

```
INo-9（★新增）：P-effect 不得产生新的 A 写
  · Reconciler 只读 A，只写 P
  · P-effect 内部禁止调用任何 enqueue
  · 代码审查 + 单测：P-effect 函数内不得出现 enqueue/append/edit 调用
```

## 2.5.1 ★ traits 裁剪与真实淘汰的语义区分（本版明确）

P 层 traits 上限 10，但这**不等于** A 层只保留 10 条。必须区分两类动作：

| 动作 | 触发 | 作用层 | A 层记录 | 用户可见 | 里程碑 |
|---|---|---|---|---|---|
| **投影裁剪** | P 层某 subject 的 traits > 10 | **P** | **不动**（仍全部存在） | 提醒词里只显示 10 条 | M0 |
| **真实淘汰** | 同上，但升级为 `lifecycle='evicted'` | P + **A** | 最旧的改 `evicted`（读谓词过滤） | 检索/列表都不再返回 | M1 |

**M0 的语义**：
```
A 层可能有 15 条 memberImpression（同 subjectId），P 层 traits 只显示最新 10 条。
→ 检索（向量）仍能命中全部 15 条（因为 visible() 不看 traits 长度）
→ 提醒词（formatMemoryProjection）只显示 10 条
→ `qq_memory_list` 能看到全部 15 条
这是"展示裁剪"，不是"数据丢失"。
```

**M1 的语义**：
```
同 subjectId 超过 10 条 → 最旧的 5 条 lifecycle='evicted'
→ visible() 过滤 → 检索也命中不到
→ 需要人工或"再提及"才恢复（evicted → active）
```

**为什么要 M1 才做真实淘汰**：真实淘汰会改变 A 层状态（不可逆地降级），需要 WAL + 审计 + 恢复路径全部就绪（M1 才完整）。M0 只做无害的展示裁剪。

**用户知情项**（写入 M0 交付说明）：
> 「同一群友的印象超过 10 条时，提醒词只显示最近 10 条；用 qq_memory_list 可查看全部。M1 会提供自动淘汰策略。」

## 2.6 `formatMemoryV2` 的改造（N-05）

```
现状（SOURCE_CONFIRMED）：bridge.js:8945-8975 读 st.activeTopics / st.pendingThoughts / st.memberImpressions
改造：
  formatMemoryProjection(projConv, sConv):
    const src = projConv ?? sConv;     // M0 双写期：P 优先，缺失降级 S
    读 src 三字段（结构兼容：M0 的 P 与 S 三字段同构）
  
  调用点改造（3 处）：
    :6805 memory GET
    :6829 memory-update
    :6692 memory-append
```

**M0 双写策略**：A 操作同时更新 P 与 S 三字段 ⇒ 两个来源都可用，降级安全。

---

# 3. WAL 结构（修订）

## 3.1 条目

```json
{
  "opId": "op_01J8…",
  "requestSeq": 128, "commitSeq": 127,
  "requestedAt": 1730000000000, "committedAt": 1730000000050,
  "opType": "edit_text",
  "targetId": "mem_01J8…",
  "expectedContentRev": 2, "targetContentRev": 3, "targetRev": 4,
  "status": "pending" | "committed" | "aborted" | "recovery_blocked" | "superseded",
  "effects": [
    { "layer": "A", "kind": "upsert", "id": "mem_…", "rev": 4, "contentRev": 3, "record": { …完整目标态… } },
    { "layer": "P", "kind": "project", "convKey": "group:123", "id": "mem_…", "contentRev": 3 },
    { "layer": "T", "kind": "tombstone", "id": "mem_…", "prevRev": 3 }
  ],
  "actor": { "type": "agent|console|system", "id": "group:123", "tokenFp": "a1b2c3d4" },
  "error": null
}
```

- `commitSeq`：跨进程重启从 WAL 末行 +1
- `requestSeq`：从 `state/memory-request-seq` + WAL 最大值恢复
- 状态改写：append-only（追加新行、同 opId 旧行标 stale）；checkpoint 压缩留终态（M2）

## 3.2 superseded 判定（修订，见 02 §4.3）

```
superseded(op):
  只比较 contentRev。存在同 targetId、contentRev 更高、状态 ∈ {committed, 更晚 pending}
    → 该 op 的 text 相关 effect 标 superseded
    → rev/元数据 effect 仍可重放
```

---

# 4. B 层生命周期与 A 层联动（P 层）

| 事件 | A 层动作 | P 层动作（同 op effect） |
|---|---|---|
| append 去重命中 | rev+1，updatedAt=now（**contentRev 不变**） | 同 id 条目 `lastMentionAt=now` |
| 超过 20 条 topic | **M0：不动 A**；M1：最旧 `lifecycle='evicted'` | 投影裁剪（移除最旧） |
| pendingThought 到期 | **M0：不动 A**（读时过滤）；M1：`lifecycle='expired'` | 读时过滤 + 投影移除 |
| 超过 50 个印象 | 同 topic | 投影裁剪 |
| A 层 edit_text | rev+1，contentRev+1，向量异步重算 | 同 id 条目 `text`/`contentRev` 更新 |
| A 层 edit_meta | rev+1，contentRev 不变 | 同 id 条目元数据更新 |
| A 层删除 | tombstone | 同 id 条目移除 + 级联副本 |
| B（P）单独修改入口 | **取消**（`memory-update` 改为转发 `MS.edit_meta`/`edit_text`，N-01） | — |

## 4.1 读谓词（防过期召回，M0 即生效）

```js
visible(record) =
     record.lifecycle === 'active'
  && (record.expiresAt == null || now < record.expiresAt)
  && !tombstone(record.id)
  && (record.contentPurged !== true)              // 语义见 04 §6
  && (M1: record.vecModel === currentModel || 允许降级)

// ★ 语义检索（向量）额外要求（本版新增）
vectorSearchable(record) =
     visible(record)
  && record.embedding.status === 'ready'
  && Array.isArray(record.vec) && record.vec.length === record.vecDim
  && record.vecContentRev === record.contentRev  // ★向量不得落后于正文
  && (M1: record.vecModel === currentModel)

// ★ P 层展示额外要求
projectionVisible(entry) =
     entry.contentRev === (A 层该 memoryId 的 contentRev)   // 投影不落后
  && 对应 A 记录 visible()
```

**M0 的效果**：虽然 A 层不做 evicted/expired 迁移，但读时过滤 `expiresAt` ⇒ **过期的 pendingThought 不会被向量召回**（部分修复 F-04）。

## 4.2 M0 双写与 M1 切换

| 阶段 | S 层三字段 | P 层 |
|---|---|---|
| M0 | 保留并继续写（兼容） | 新建，**双写**（A 操作同时更新 P 与 S） |
| M0 读 | `formatMemoryProjection` 优先 P，缺失降级 S | — |
| M1 迁移 | **移除三字段** | 唯一来源 |
| M1 之后 | 不含三字段 | 唯一来源 |

---

# 5. WAL 恢复

## 5.1 WAL 先行

`pending` 在 P3.1 落盘，先于任何业务写。理由：崩溃后能知道"曾打算做什么"；业务写是幂等 set-to-state，不需在 WAL 前发生。

## 5.2 崩溃点矩阵（四档，含 P 层后为五档）

| 崩溃点 | 磁盘状态 | 恢复动作 |
|---|---|---|
| ① WAL(pending) 后、任何业务写前 | store 旧、WAL pending | effects 未发生 → 标 `aborted(crashed_before_apply)`，无副作用 |
| ② T 写后、store 前 | tombstone 有、store 仍有记录 | 重放 A/T（按 id + targetContentRev 幂等）→ 重写 store → 续 |
| ③ store 写后、P 前 | A 新、P 旧 | 重放 P-effect（幂等 set-by-id）→ 续 |
| ④ **P 写后、SR 前** | A/P 新、SR 旧 | 重放 SR-effect → 续 |
| ⑤ SR 写后、WAL(committed) 前 | 全部新、WAL pending | 按 effects 已就绪 → 直接标 committed |

## 5.3 重放幂等与 superseded

```
canApply(effect):
  T-effect: tombstone 已存在同 id → skip
  A-effect: store 无该 id 且 kind=delete → skip
            store 无该 id 且 kind=upsert 且无 tombstone → upsert
            store 有该 id → 仅当 record.contentRev < effect.contentRev 才 upsert（内容层面）
                           元数据 effect（rev 更高）→ 仅当 record.rev < effect.rev 才应用
  P-effect: P 条目 contentRev < effect.contentRev 才写
  SR-effect: SR 状态非终态才写

superseded(op): 见 02 §4.3（只比 contentRev）
```

## 5.4 提交点与"WAL 写成功但业务失败"

- WAL pending 成功、业务 IO 失败 → 进程存活走失败路径（500），WAL 标 `aborted(io_error)`
- 进程崩溃 → 进入恢复（§5.2）
- 某 effect 反复失败（磁盘持续错误）→ `recovery_blocked`：
  - 服务可启动，但**该 op 涉及的 memoryId 进入冻结集**（读写返回 503 `recovery_pending`）
  - 控制台 `POST /api/memory/recovery/resubmit {opId}` / `discard {opId}`（Tier-1 审计）

## 5.5 A 成功 P 失败 / 反之

统一管线中**不作为终态暴露**：任何一步失败即重试/恢复，按 §5.2 收敛到"全部生效 + WAL committed"。
端点不返回 V1 式 `aSynced/bSynced/partial`，改返回 `opId + 各层计数 + recoveryNote`。

**表述修正**（原 `03 §5.5` 说"不再出现"过强）：应说"不作为**终态**暴露"——中间态在崩溃窗口内确实存在（③④），但恢复会收敛。

## 5.6 恢复扫描顺序（修订）

```
按 requestSeq 升序（不是 commitSeq）重放
理由：requestSeq 反映请求先后，commitSeq 反映提交先后；
      恢复时按请求顺序重放，语义更符合用户预期
若 requestSeq 相同（不应出现）→ 按 commitSeq
```

---

# 6. V1 → V2 迁移（修订，含 P 层）

**前置**：服务停止 + writer.lock + `node scripts/migrate-v1-to-v2.mjs`（控制台令牌）

```
步骤 1  全量备份（envelope，06 §5）
步骤 2  dry-run：
        读 state/memory-vectors.jsonl 逐行
        ├─ 解析失败 → badlines（不丢）
        ├─ 每行 → {id:新ULID, rev:1, contentRev:1, key:canonical,
        │           category:推断(text前缀), subjectId:null,
        │           text:redact+clamp(1000), lifecycle:'active',
        │           embedding: 有vec?'ready':'failed', vecModel:当前,
        │           vecContentRev: 有vec?1:0, expiresAt:null,
        │           migratedFrom:'v1', _crc}
        ├─ 去重：同 key+category+text 合并（保留最早 time 为 createdAt）
        └─ 报告 {inputRows, badLines, duplicates, outputRecords, orphanCandidates}
步骤 3  人工确认（dry-run 令牌 + 报告签字）
步骤 4  写 memory-store.jsonl.new（临时 + 原子 rename）
        旧文件改名 memory-vectors.jsonl.v1.bak（只读保留）
        tombstones / WAL / request-seq 初始化为空
步骤 5  ★ 生成 P 层：
        从新 store 按 key 分组 → 构造 memory-projection.json
        S 层三字段逐条尝试匹配（key + text 前 200 字）：
          匹配 → 注入 memoryId/contentRev
          未匹配 → 导出 social-v2-orphans.json（人工决定）
步骤 6  对账：outputRecords == inputRows - badLines - duplicates；任一不等 → 整体回退
步骤 7  审计（Tier-1）：MIGRATE {rows, duplicates, badlines, orphans, projectionEntries}
```

**防丢失**：备份 + 对账 + 临时文件原子生效 + 失败回退。
**防复活**：迁移只读**当前** store；不触碰 `backups/`、`.pre-*.bak`、`qq-bridge备份-*`；tombstone 初始为空 ⇒ **V1 时期删除的记录无墓碑**（边界见 D-12）。

## 6.1 M0 的迁移特殊要求

M0 必须能独立完成迁移（因为 M0 已含 P 层）：
- 步骤 5 的 P 生成是 **M0 必做**（否则 `formatMemoryProjection` 无数据）
- S 层三字段在 M0 **不移除**（双写兼容）

---

# 7. 验收要点

1. ①②③④⑤ 五个崩溃点注入终止，恢复后终态与正常提交等价（store/P/S/WAL/T 五方对账）。
2. 旧 contentRev 重放不覆盖新内容（构造 contentRev=3 committed 后重放 contentRev=2 pending）。
3. 迁移对账失败注入 → 回退且原文件字节级未变。
4. P 层损坏 → 会话重建后与 A 一致。
5. 39 处运行状态写入与 P 层互不干扰（P 独立文件）。
