# 05 · 分享申请、审批、撤销、群隔离与权限验证

解决 F-09。保留决策 3/8/9/10。

## 1. ShareRequest 对象（state/memory-share-requests.jsonl）

```json
{
  "requestId": "req_01J8…",
  "memoryId": "mem_01J8…",
  "expectedRevAtRequest": 3,          // 申请时原件 rev（审批时复查）
  "targetKey": "group:999",           // 每群一条 → 群隔离（任务 6 要求 7）
  "requestedBy": { "type": "agent", "id": "group:123", "tokenFp": "…" },
  "status": "pending" | "approved" | "rejected" | "revoked" | "superseded",
  "decidedBy": null, "decidedAt": null, "note": "",
  "copyId": null,                     // approved 后生成的副本 id
  "createdAt": 1730000000000, "updatedAt": 1730000000000
}
```

- **申请键 = (memoryId, targetKey)**：同一记忆对群 A 与群 B 是两条独立 SR，各自独立批准/拒绝（修 V1 单字段 `share_review_status` 无法分群表达的缺陷）。
- 原件上的 `shareCopies[]` 是授权登记表（审批后写入），与 SR 双向引用（`requestId` ↔ `copyId`）。

## 2. 申请流程（requestShare）

```
模型 → MCP qq_memory_share_request {memoryId, token}
  ├─ 无条件三重鉴权（agentTokenOk/v2SessionAllowed/v2ToolEnabled）
  ├─ Normalizer：targetKey=canonical；校验 **源 key 非 private**（要求 10：
  │    private 记忆走普通分享流程 → 400「私聊记忆不支持共享到群」，与 V1 bridge.js:6892 一致）
  ├─ 入队 share_request {memoryId, expectedRev, targetKey}
  └─ P3：
     ① 复查原件：store 有 id？  否 → 404（要求 2 的申请侧）
     ② tombstone 命中？        是 → 410
     ③ lifecycle==='active'？  否 → 409「目标不可分享（当前状态 X）」
     ④ 源 key 与 target 关系校验：源为 group:X → 仅允许 target=group:X（同群自分享）
        （第一版不支持跨群转发，跨群属未来扩展 → 08 D-10）
     ⑤ 同 (memoryId, targetKey) 已有 pending → 返回 already_pending（幂等）
        已 approved → 返回 already_approved
     ⑥ 写 SR {status:'pending', expectedRevAtRequest=当前 rev}
     ⑦ Tier-2 审计（失败→补偿）
```

**申请必须用 memoryId**（要求 1）：MCP 工具签名从 `{key, token, text}` 改为 `{key, token, memoryId}`；模型先经 `qq_memory_query` 获取 id。`text` 参数保留一个版本周期作为兼容校验（与 store 中 text 比对，不一致 → 400 提示改用 id），之后移除。

## 3. 审批流程（approveShare / rejectShare）

```
控制台 → POST /api/memory/admin {op:'share_approve', requestId, note}
  ├─ 鉴权（consoleToken + 拒绝 x-agent-token，V1 bridge.js:6703-6714 保留）
  ├─ 入队 share_approve {requestId}
  └─ P3（逐项复查，任一失败即明确拒绝——要求 2/3/4/5/6）：
     ① SR 存在且 status==='pending'？        否 → 404 / 409
     ② 原件存在？store 有 memoryId？          否 → SR 标 superseded，404「原件已删除」
     ③ tombstone 命中 memoryId？              是 → SR 标 superseded，410（要求 4）
     ④ 原件 rev === SR.expectedRevAtRequest？ 否 → SR 标 superseded，409「原件已变更（当前 rev=N），请重新申请」（要求 3/6）
     ⑤ 原件 lifecycle==='active'？            否 → 409（要求 5）
     ⑥ 原件 visibility 允许？  owner 的合法持有；private 源拒绝（要求 10）
     ⑦ suppression 为空？                     否 → 409「原件内容已被屏蔽」
     ⑧ Tier-1 审计写入（失败 → SR 不变，整体失败）
     ⑨ 生成副本：
        copy = { id:新ULID, rev:1, key:targetKey, owner:targetKey,
                 category:原件category, text:原件text,
                 origin:{id:原件id, rev:原件rev},          ← 要求 8：可追溯
                 visibility:'group_shared',
                 lifecycle:'active',
                 vec/vecModel/vecDim：按决策 10（见 §5） }
        原件.shareCopies += {targetKey, status:'approved', requestId, copyId}
        原件 rev+1（shareCopies 变更属内容变更）——注意：**该 rev+1 使既有其他 pending SR
        全部 superseded**（它们 expectedRevAtRequest 已过期，需重新申请；
        如不希望此副作用 → 08 D-17）
     ⑩ 原子提交：store（原件+副本）→ SR → B 投影（副本在目标群可见）→ WAL committed
```

reject 流程：仅 ①~⑧ 中涉及 SR 与审计的步骤 + `SR.status='rejected'`，不动原件、不生成副本。

## 4. 撤销流程（revoke）

```
触发：①控制台主动撤销（requestId 或 copyId） ②原件删除级联（决策 4） ③原件编辑导致
     rev 失配（仅使 pending superseded，不影响已批准副本——已批准副本携带自己的文本，见 §6）
入队 share_revoke：
  P3：定位副本 → tombstone(原因 'share_revoke'|'cascade') → 原件.shareCopies[status='revoked']
      → B 投影移除 → SR.status='revoked' → Tier-1 审计 → 提交
```

## 5. 副本向量策略（决策 10）

| 条件 | 动作 |
|---|---|
| 原件 `embedding.status==='ready'` **且** `vecModel===当前模型` **且** `vecDim` 一致 | 直接复用 `vec`（副本与原件同文本，语义等价；省一次嵌入） |
| 原件 ready 但模型不一致 | 副本 `embedding.status='pending'`，进重算队列（06 §2.5） |
| 原件 pending/failed/retry_wait | 副本同状态，进重试队列（与原件共用退避计数独立） |
| 原件 suppression.scope 含 vec/all | 拒绝审批（§3 ⑦ 已拦截） |

## 6. 原件编辑与副本的关系

- 副本是**独立记录**（有自己的 id/rev/text），审批时刻的快照。原件后续 edit 不自动改副本（副本在目标群有独立可见性）。
- 控制台提供 `POST /api/memory/admin {op:'share_sync', requestId}` 主动同步（把原件当前 text 推到副本，副本 rev+1，Tier-1 审计）——**可选实现**（M2），避免 A/B 递归触发。
- 原件删除 → 级联（决策 4，见 04 §5）。

## 7. 群隔离矩阵（要求 7）

| 场景 | 行为 |
|---|---|
| 群 A 批准、群 B 拒绝 | 两条 SR 独立；群 A 检索可见、群 B 不可见 |
| 群 A 撤销后群 B 仍有效 | 独立 SR/copy，互不影响 |
| 原件删除 | 默认两级联（A、B 的副本都删） |
| 一条记忆同时被多群申请 | 多条 pending，逐一审批 |

## 8. 端到端流程图（申请→审批→撤销→原件删除）

```
[申请]  agent ──(memoryId+token)──▶ SR(pending, eRev=3, target=group:999)
[审批]  console ──(requestId)──▶ 复查①~⑧ ──▶ 生成 copy(mem_C, origin=mem_M@rev3)
                                   mem_M.shareCopies += {group:999, approved, copyId}
                                   群 999 检索可见 ✅
[撤销]  console ──(copyId)──▶ mem_C tombstone(share_revoke) ──▶ 群 999 不再可见
[原件删除] console ──(memoryId, cascade=true)──▶
        pending SR(req_2) → superseded
        mem_C(rev=1)      → tombstone(cascade, originCascade=true)
        mem_M             → tombstone
        B 投影同步移除
[原件编辑] edit(mem_M, eRev=3→4) ──▶ req_2 等 pending 全部 superseded（需重新申请）
                                   已批准 mem_C 不受影响（独立快照）
```

## 9. bot_persona 类目（决策 3/9）

`category='bot_persona'`（或 `factKey='bot_persona:*'`）的记忆：
- append/edit 一律进入 `pending` 审核状态（等同 SR 流程），不直接 active；
- 控制台批准后方 active；
- M1 实现（与 ShareRequest 复用审批基础设施，但语义为「内容审核」而非「跨群授权」）。

## 10. ShareSuggestion（决策 8）

第一版**不实现**。数据结构预留：`SR.suggestion` 字段（M2 填充）。

## 11. 验收要点

1. 用错误 memoryId 申请 → 404；审批时原件已编辑 → superseded + 明确提示；审批时原件已删 → 410。
2. 两群独立授权矩阵（7 种组合）行为符合 §7 表。
3. 副本可追溯：`copy.origin.id` 反查到原件（或其 tombstone）。
4. private 源 key 申请 → 400，与 V1 `bridge.js:6892` 行为一致。
5. 撤销与级联后，目标群 search 不再返回该副本。
