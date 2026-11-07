# SCHEMA · 统一数据模型（唯一权威定义）

> 本文档是全部设计文档中字段、状态、语义的**唯一权威来源**。
> 任何其他文档与本文档冲突，以本文档为准；若需变更，先改本文档。
> 目的：解决 P1-10「统一 Retain、Reflect、Recall 的数据字段和生命周期语义」。

---

## 0. 本版相对上一版的破坏性变更（必须同步到全文档）

| 变更 | 旧 | 新 | 影响 |
|---|---|---|---|
| 可见性模型 | `visibility: owner_private\|group_shared` + `share_review_status` 混用 | 拆为 `scope.{read,originGroupId,contentClass,copies}` + 独立 `review.status` | 02/04/06/07/11/12 |
| 新增状态值 | — | `review.status: not_required` | 取代 `auto_shared`（该值**废弃**） |
| 生命周期 | active/stale/superseded/deleted/proposed | 增加 `quarantined`，并规定 `proposed` 不进召回 | 全文档 |
| 断言模型 | `epistemic` 独立字段 | 收进 `assertion{epistemic,sourceType,assertedBy,confidence}` | 07 |
| 索引一致性 | 索引不领先 A | 索引**可滞后**、不可领先、不可残留已删 | 01/04/12 |
| 版本 | rev/contentRev/metaRev | 保留；明确 rev 不保证连续 | 11 #2 |

---

## 1. A 层记录（`state/memory-store.jsonl`）

一行一条完整 JSON，追加为主，修正走整文件原子重写（受控）。

```jsonc
{
  // ── 身份与版本 ──────────────────────────────────────
  "id": "mem_01J8Z7K3",              // 全局唯一，创建后不变
  "rev": 7,                           // 记录整体版本，任何变更 +1（不保证连续）
  "contentRev": 3,                    // text 变更次数
  "metaRev": 5,                       // 除 text 外的元数据变更次数
  "vecContentRev": 3,                 // 当前 vec 对应哪个 contentRev；必须 <= contentRev

  // ── 归属（决定谁能看到）────────────────────────────
  "key": "group:123456",              // 会话键。召回永远 key 严格相等
  "ownerQq": "[OWNER_QQ]",            // 主人 QQ（单主人模型）

  // ── 来源范围（origin scope）────────────────────────
  "origin": {
    "channel": "qq_group",            // qq_group | qq_private | legacy | manual | reflect
    "groupId": "123456",              // 群来源；私聊为 null
    "messageSeq": 88123,              // 本地 seq，可回溯原消息
    "sender": "[BOT_QQ]",
    "at": 1761900000000,              // 事件发生时间（毫秒）
    "trigger": "retain_rule:commitment"  // 谁产生的
  },

  // ── 读取权限范围（read scope）──────────────────────
  "scope": {
    "read": "origin_group",           // owner_only | origin_group
    "originGroupId": "123456",        // read=origin_group 时必填，且必须等于 origin.groupId
    "contentClass": "public",         // public | sensitive | uncertain
    "classReason": "none",            // none|credential|third_party|address|medical|finance|relationship|ambiguous
    "copies": []                      // 显式跨群共享：{groupId, approvedBy, approvedAt}。默认空，无自动写入路径
  },

  // ── 审核状态（仅"人工审核"语义，与可见性解耦）──────
  "review": {
    "status": "not_required",         // not_required | pending | approved | rejected
    "requestedAt": null,
    "reviewedAt": null,
    "note": null
  },

  // ── 内容 ────────────────────────────────────────────
  "text": "……",                       // 已脱敏，<=1000 字
  "category": "activeTopic",          // activeTopic | pendingThought | memberImpression | topic | preference | summary
  "kind": "fact",                     // fact | observation | preference | summary
  "fingerprint": "a1b2c3d4e5f6a7b8",  // sha256(category + ':' + normalize(text))[:16]

  // ── 断言模型（事实/推断/观点的统一载体）────────────
  "assertion": {
    "epistemic": "fact",              // fact | inference | opinion
    "sourceType": "self_statement",   // self_statement | third_party | ai_inference | ai_opinion | manual
    "assertedBy": "[BOT_QQ]",       // 断言主体（谁说的/谁被说）
    "confidence": 1.0,                // fact=1.0；inference<=0.75；opinion<=0.6
    "evidence": []                    // kind!=fact 时必填，见 §4
  },

  // ── 生命周期 ────────────────────────────────────────
  "lifecycle": "active",              // active|stale|superseded|quarantined|deleted|proposed
  "supersededBy": null,               // lifecycle=superseded 时必填
  "duplicateOf": null,                // 完全重复被合并时指向主记录
  "mergedFrom": [],                   // 反向：本条合并了哪些 id
  "validFrom": null,                  // 双时态：事实开始成立
  "validUntil": null,                 // 双时态：事实不再成立（不删，只失效）

  // ── 附加信息 ────────────────────────────────────────
  "extra": {},                        // 类别专属字段，见 §3
  "originGroupId": "123456",          // 冗余便于索引，必须等于 scope.originGroupId

  // ── 向量与嵌入状态（三态）───────────────────────────
  "vec": [1024 floats],
  "vecModel": "bge-m3",
  "vecDim": 1024,
  "embedding": {
    "status": "ready",                // ready | pending | failed | missing
    "attempts": 1,
    "lastError": null,
    "nextRetryAt": null
  },

  // ── 完整性 ──────────────────────────────────────────
  "_crc": "sha256:..."
}
```

### 1.1 `rev` 的语义（回应 11 #2）

- `rev` 是**变更计数器**，不是序列号。**不保证连续**（并发下两次提交可能被合并为一次递增，或保留空洞）。
- 唯一约束：同一 `id` 的 `rev` **严格单调递增**，且每次成功提交至少 +1。
- 允许重跳（1 → 3），不允许重复。验收只检查"单调 + 无重复"，不检查连续性。
- `contentRev <= rev`、`metaRev <= rev`、`vecContentRev <= contentRev` 三条恒成立。

---

## 2. 生命周期状态机（回应 P1-10）

```
                    ┌──────────────────────────────────────────┐
                    │                                          │
   [写入]──► active ──┬──► stale ──────(再次被提及)──────────────┘
              │       │
              │       └──► superseded  （完全重复被合并 / 被新观察取代）
              │                 ▲
              │                 │
              ├──► proposed ────┘   （观察/偏好生成时；不进召回）
              │        │
              │        └──► active （自动晋升条件满足 或 人工批准）
              │
              ├──► quarantined      （权限/内容分级未通过；只有主人可审）
              │        ├──► active （主人批准，可提升 scope）
              │        └──► deleted（主人驳回）
              │
              └──► deleted          （用户/管理端删除；正文移入 tombstone 指向）
```

### 2.1 状态语义表（所有文档必须遵循）

| lifecycle | 进召回？ | 进注入？ | 可被 Reflect 处理？ | 备注 |
|---|---|---|---|---|
| `active` | 是 | 是 | 是 | 正常状态 |
| `stale` | 是（降权 0.6） | 是（仅在时间/明确查询时） | 是 | 过时但保留；可复活 |
| `superseded` | **否** | 否 | 否（终态） | 可显式查询（管理端） |
| `deleted` | 否 | 否 | 否 | 正文不在此记录（tombstone 只存元数据） |
| `proposed` | **否**（默认） | **否** | 是（等证据/审核） | 需 `includeProposed` 才可查；见 07 |
| `quarantined` | **否** | 否 | 否 | 只有管理端可见；见 06 §4 |

**这是本版最重要的修订之一**：`proposed` 从"降权注入"改为"默认完全不进召回与注入"。理由见 07 文档 §5.4。

---

## 3. 各类别的 `extra` 字段（封闭集合，不允许自由扩展）

| category | kind | extra 字段 | 说明 |
|---|---|---|---|
| `activeTopic` | fact | `participants[]`, `pendingQuestion`, `lastMentionAt` | 进行中的话题 |
| `pendingThought` | fact | `motivation`, `expiresAt` | 想说未说 |
| `memberImpression` | fact | `target`, `dimension` | 对某人的具体印象（单条，不聚合） |
| `topic` | observation | `clusterKey`, `evidenceCount` | 归纳出的话题观察 |
| `preference` | preference | `dimension`, `polarity`, `negatedFrom`, `conflictWith[]` | 见 07 §6 |
| `summary` | summary | `coveredFrom`, `coveredTo`, `messageCount` | 会话摘要 |

**原则**：新增字段必须先进本文档，再进代码。禁止"先加字段后补文档"。

---

## 4. 证据对象（observation / preference 必备）

```jsonc
"evidence": [
  {
    "memId": "mem_01J8Z7K3",     // 被引用的事实记录 id（必须存在且 lifecycle=active）
    "quote": "我在看那块最小系统板",  // 必须是该记录 text 的**连续子串**
    "at": 1761000000000,         // 该事实的发生时间
    "sourceType": "self_statement" // 该证据本身的性质
  }
]
```

**硬约束（可程序化校验）**：

1. `evidence.length >= 3`（观察）或 `>= 3`（偏好）。
2. `quote` 必须是 `mem.text` 的连续子串（`text.includes(quote) === true`）。
3. **独立来源**：`evidence` 中 `memId` 互不相同，且 `new Set(evidence.map(e => new Date(e.at).toDateString())).size >= 2`（跨 ≥2 个自然日）。
4. **主体一致**：所有 `memId` 指向的记录 `key` 相同。

第 3 条是"独立证据"的可执行定义：同一段对话里连说三句不算三个来源。

---

## 5. S 层新增结构（`state/social-v2.json`）

```jsonc
{
  // 已有
  "agentToken": "…",                  // 保留字段；**不再进入 prompt**（见 05 §3）
  "recentMessages": [...], "unread": [...], "activeTopics": [...], "pendingThoughts": [...],
  "memberImpressions": {...},         // 移除 interactionCount（见 S-1）

  // 新增
  "promptLedger": {                   // 块账本
    "group:123456": {
      "sessionId": "sess_abc", "epoch": 3,
      "blocks": { "persona": {"hash":"a1b2","sentTurn":1,"sentAt":1761900000000} },
      "turnsSincePersona": 1,
      "injectedMemo": [{ "fp": "m_9f2a", "turn": 12, "at": 1761900000000 }]
    }
  },
  "sessionMeta": {
    "group:123456": { "sessionId":"sess_abc", "createdAt":0, "turns":12, "estHistoryTokens":8400 }
  },
  "rotationTx": {                     // 轮转事务日志（P0-5 原子性）
    "group:123456": { "phase":"prepare", "startedAt":0, "fromSessionId":"sess_abc", "attempt":1, "digest":"…" }
  },
  "clearedKeys": { "group:123456": { "clearedAt": 0, "clearEpoch": 88 } },  // 见 §7
  "reviewQueue": [ { "id":"rev_1", "memId":"mem_x", "type":"observation|quarantine|conflict", "at":0 } ],
  "reflectCursor": { "group:123456": { "lastId":"mem_x", "lastRunAt":0 } }
}
```

---

## 6. P 层结构（`state/memory-projection.json`）

```jsonc
{
  "globalRev": 12,
  "keys": {
    "group:123456": {
      "convVersion": 88,             // per-key 会话版本
      "fromARev": 412,               // 构建时读到的 A 层最大 rev（依赖声明）
      "bLayer": { "activeTopics": [], "pendingThoughts": [], "memberImpressions": {} },
      "digest": { "text":"…", "coveredFrom":0, "coveredTo":88123, "generatedAt":0, "dirty":false },
      "recallCache": { "queryHash":"…", "resultIds":[], "at":0 },
      "builtAt": 0
    }
  }
}
```

### 6.1 P 的依赖声明（回应 P0-4「P 重建规则必须与实际依赖数据一致」）

| P 字段 | 依赖 | 说明 |
|---|---|---|
| `bLayer.*` | **A 层**（同 key，lifecycle=active，kind=fact） | 从 A 层投影而来，不读 S 层 B 段 |
| `digest` | A 层（同 key）+ S 层 `recentMessages`（同 key） | 唯一同时依赖两者的字段 |
| `recallCache` | 无（纯缓存） | 可随时丢弃 |
| `convVersion` | S 层 `clearedKeys` + 自身 | 见 §7 |

**明确**：P 层不读别的 key 的任何数据。重建某 key 只需该 key 的 A 层 + 该 key 的 S 段。

---

## 7. 版本与 epoch（统一并发语义）

| 名称 | 位置 | 语义 | 谁递增 |
|---|---|---|---|
| `rev` | A 记录 | 记录变更次数 | 每次成功提交 |
| `convVersion` | P 层 per key | 投影重建代数 | 投影重建 / clear |
| `clearEpoch` | S 层 per key | 清空次数（fencing token） | `clear` / `reset --purge` |
| `globalRev` | P 层 | 投影整体代数 | 任何 key 重建 |
| `sessionEpoch` | bridge 内存 + S 层 | 会话代数（防复活） | 会话轮转 / reset |
| `indexGeneration` | 索引 meta | 索引构建代数 | 全量重建 |

**`clearEpoch` 的用法（回应 P0-4「统一提交语义」）**：

写入事务在 `prepare` 阶段记录 `observedEpoch = st.clearEpoch`；在 `commit` 前**必须**校验 `st.clearEpoch === observedEpoch`，不等则 abort（`reason: 'cleared_during_write'`）。

这不是"写前重读文件"——`clearEpoch` 在内存中由 per-key 互斥锁保护，校验是一次内存比较。真正的串行化机制见 01 §5。

---

## 8. 索引结构（`state/memory-index/`）

```jsonc
// meta.json
{
  "generation": 7,                    // 每次全量重建 +1
  "dim": 1024,
  "docCount": 8123,
  "coveredRev": 412,                  // 已入库的 A 层最大 rev（滞后标记）
  "builtAt": 1761900000000,
  "model": "bge-m3",
  "crc": "sha256:..."
}
```

```
ids.json       ["mem_a","mem_b",...]           docId = 数组下标（0..n-1）
vec.f32        Float32Array(docCount * dim)    行对齐 ids
bm25.json      { postings, docLen, avgdl, idfCache }
temporal.json  { buckets: { "2026-10-31": ["mem_a", ...] } }
```

**docId 规则（回应 P1-4）**：
- `docId` 是**构建期分配的连续整数**（数组下标），不是持久 id。持久身份始终是 `mem.id`。
- 追加新文档时 `docId = docCount`，写入 `vec.f32` 末尾 + `ids.json` 末尾（追加，不重排）。
- 删除文档时**不立刻压缩**（避免重排），标记 `tombstoned`；当 `tombstoned / docCount > 0.3` 时触发全量重建（retire generation）。
- 每次索引写操作必须 bump `meta.generation`（单调），用于检测"读到旧索引"。

---

## 9. 权限判定（唯一函数，所有模块共用）

```js
// src/memory-permission.js
// 返回 boolean；所有模块**必须**用它，不得各自写过滤条件
function canRead(record, ctx) {
  // ctx: { key, caller: 'agent'|'admin'|'reflect', groupId|null }
  if (record.lifecycle === 'deleted' || record.lifecycle === 'quarantined') return false;
  if (record.lifecycle === 'proposed' && ctx.caller !== 'admin') return false;

  // 1) 调用方必须是该记录所属会话（禁止跨 key）
  if (record.key !== ctx.key) {
    // 唯一例外：显式 copies（当前无写入路径，恒为空）
    const copied = (record.scope?.copies || []).some(c => c.groupId === ctx.groupId);
    if (!copied) return false;
  }

  // 2) 群聊场景：必须 origin_group + 同群 + 内容为 public
  if (String(ctx.key).startsWith('group:')) {
    if (record.scope?.read !== 'origin_group') return false;
    if (record.scope?.originGroupId !== ctx.groupId) return false;
    if (record.scope?.contentClass !== 'public') return false;
  }

  // 3) 私聊场景：必须是主人本人
  if (String(ctx.key).startsWith('private:')) {
    if (ctx.key !== `private:${record.ownerQq}`) return false;
  }

  return true;
}
```

**调用约定**：`semantic/keyword/temporal` 三通道在**打分前**调用 `canRead`（P6：通道内过滤）。融合后、注入前再调用一次（INo-13 断言）。

---

## 10. 字段所有权（谁能写哪个字段）

| 字段 | 写入者 | 说明 |
|---|---|---|
| `rev/contentRev/metaRev` | `memory-store.js` | 唯一 |
| `text` | `memory-store.js`（edit 路径） | 只有显式 edit 能改 |
| `scope.read/originGroupId` | `memory-permission.js` 的 `classify()` | 由 origin + contentClass 推导 |
| `scope.contentClass` | `memory-permission.js` 的分类器 | 只降不升（自动流程不能把 sensitive 改回 public） |
| `scope.copies` | 管理端端点（当前无写入路径） | 唯一允许跨群的能力 |
| `review.status` | 管理端端点 | 自动流程只能置 `pending`，不能置 `approved` |
| `lifecycle` | `memory-store.js` / `memory-reflect.js` | Reflect 只能 → `stale`/`superseded`/`proposed`，**不能** → `deleted` |
| `assertion.*` | Retain/Reflect（生成时）/ 管理端 | 变更只 `metaRev++` |
| `vec/vecContentRev` | `memory-index.js` 的向量管道 | 见 04 |

**关键约束**：`memory-reflect.js` **没有删除权**。它只能改变 lifecycle 到非删除态。这是"归纳不覆盖事实"在权限层的实现。
