# 05 · 分享申请、审批、撤销与群隔离

解决 F-09。保留决策 3/8/9/10。本版核心修订：**ShareRequest 绑定 `expectedContentRevAtRequest`**（而非 rev），使多群审批不被元数据变更打断。

---

# 1. ShareRequest 对象

`state/memory-share-requests.jsonl`：

```json
{
  "requestId": "req_01J8…",
  "memoryId": "mem_01J8…",
  "expectedContentRevAtRequest": 3,
  "expectedContentHashAtRequest": "sha256:…",
  "targetKey": "group:999",
  "requestedBy": { "type": "agent", "id": "group:123", "tokenFp": "…" },
  "requestedSeq": 128, "requestedAt": 1730000000000,
  "status": "pending" | "approved" | "rejected" | "revoked" | "superseded",
  "decidedBy": null, "decidedAt": null, "note": "",
  "copyId": null,
  "createdAt": 1730000000000, "updatedAt": 1730000000000,
  "opId": "op_…"
}
```

| 字段 | 说明 |
|---|---|
| `expectedContentRevAtRequest` | **申请时原件 contentRev**（审批时复查）★本版关键 |
| `expectedContentHashAtRequest` | 申请时原件 contentHash（双保险，防 contentRev 被误改） |
| `requestedSeq` / `requestedAt` | 请求顺序（见 02 §5） |

**申请键 = (memoryId, targetKey)**：同一记忆对群 A 与群 B 是两条独立 SR，各自独立批准/拒绝（修 V1 单字段缺陷）。

原件 `shareCopies[]` 是授权登记表，与 SR 双向引用（`requestId` ↔ `copyId`）。

---

# 2. 申请流程（requestShare）

```
模型 → MCP qq_memory_share_request {memoryId, token, targetKey?}
  ├─ 无条件三重鉴权
  ├─ Normalizer：targetKey=canonical
  ├─ 校验源 key 非 private（要求 10：private → 400「私聊记忆不支持共享到群」，
  │   与 V1 bridge.js:6892-6894 行为一致）
  ├─ 入队 share_request {memoryId, expectedContentRev, targetKey}
  └─ P3：
     ① 复查原件：store 有 id？            否 → 404
     ② tombstone 命中？                   是 → 410
     ③ lifecycle==='active'？             否 → 409「目标不可分享（当前状态 X）」
     ④ 源 key 与 target：源 group:X 仅允许 target=group:X（同群自分享）
        （第一版不支持跨群转发 → D-10）
     ⑤ 同 (memoryId, targetKey)：
          pending  → already_pending（幂等）
          approved → already_approved
     ⑥ contentRev 校验：record.contentRev === op.expectedContentRev？
                          否 → 409 + currentContentRev
     ⑦ 写 SR {status:'pending', expectedContentRevAtRequest=record.contentRev,
              expectedContentHashAtRequest=record.contentHash}
     ⑧ Tier-2 审计（失败→补偿）
```

## 2.1 申请必须用 memoryId

MCP 工具签名从 `{key, token, text}` 改为 `{key, token, memoryId}`；模型先经 `qq_memory_query` 获取 id。
`text` 参数保留一个版本周期作兼容校验（与 store 中 text 比对，不一致 → 400 提示改用 id），之后移除（D-20）。

---

# 3. 审批流程（approveShare / rejectShare）

```
控制台 → POST /api/memory/admin {op:'share_approve', requestId, note}
  ├─ 鉴权（consoleToken + 拒绝 x-agent-token，V1 bridge.js:6707 保留）
  ├─ 入队 share_approve {requestId}
  └─ P3（逐项复查，任一失败即明确拒绝）：
     ① SR 存在且 status==='pending'？           否 → 404 / 409
     ② 原件存在？store 有 memoryId？             否 → SR=superseded，404
     ③ tombstone 命中？                          是 → SR=superseded，410
     ④ **record.contentRev === sr.expectedContentRevAtRequest？**
                                                 否 → SR=superseded，409「原件内容已变更（当前 contentRev=N），请重新申请」
     ⑤ **record.contentHash === sr.expectedContentHashAtRequest？**
                                                 否 → 同上（双保险）
     ⑥ lifecycle==='active'？                    否 → 409
     ⑦ visibility 允许？private 源拒绝            否 → 403
     ⑧ suppression 为空？                        否 → 409「原件内容已被屏蔽」
     ⑨ Tier-1 审计写入（失败 → SR 不变，整体失败）
     ⑩ 生成副本：
        copy = { id:新ULID, rev:1, contentRev:1,
                 key:targetKey, owner:targetKey, category:原件category,
                 subjectId:原件subjectId,        ← 保留主体，便于群里归因
                 text:原件text,                   ← 完整文本（不是 200 字摘要）
                 origin:{id:原件id, contentRev:原件.contentRev},
                 visibility:'group_shared', lifecycle:'active',
                 vec/vecModel/vecDim/vecContentRev：按 §5 }
        原件.shareCopies += {targetKey, status:'approved', requestId, copyId}
        ★ 原件 rev+1（contentRev **不变**）—— 这是本版与 V2.4.2 的关键差异
     ⑪ 原子提交：store（原件+副本）→ P（副本在目标群可见）→ SR → WAL committed
```

## 3.1 为什么原件 rev+1 但 contentRev 不变

| 版本 | 是否变 | 影响 |
|---|---|---|
| `rev` | +1 | 记录内容（shareCopies 数组）变了，WAL/审计需要反映 |
| `contentRev` | **不变** | **其他群的 pending SR 仍有效**（这正是本版修复的） |

原 V2.4.2 用 rev 作审批绑定 ⇒ 批准群 A 会让群 B 的 SR 失效（`05 §3 ⑨` 自述）。
本版用 contentRev ⇒ 多群可依次审批，不再互相打断。

**例外**：若原件 text 在审批期间被改（contentRev 变了）⇒ SR 失效（正确，因为内容变了）。

## 3.2 reject 流程

仅涉及 SR 与审计：`SR.status='rejected'`，不动原件、不生成副本。

---

# 4. 撤销流程（revoke）

```
触发：
  ① 控制台主动撤销（requestId 或 copyId）
  ② 原件删除级联（决策 4）
  ③ 原件内容变更（contentRev 变）→ 仅使 pending superseded，
     已批准副本不受影响（副本携带自己的 text，见 §6）
入队 share_revoke：
  P3：定位副本 → tombstone(reason 'share_revoke'|'cascade')
      → 原件.shareCopies[status='revoked']（原件 rev+1，contentRev 不变）
      → P 移除副本 → SR.status='revoked' → Tier-1 审计 → 提交
```

---

# 5. 副本向量策略（决策 10）

| 条件 | 动作 |
|---|---|
| 原件 `embedding.status==='ready'` **且** `vecModel===当前模型` **且** `vecDim` 一致 **且** `vecContentRev===原件.contentRev` | 直接复用 `vec`（副本 `vecContentRev = 副本.contentRev = 1`） |
| 原件 ready 但模型不一致 | 副本 `embedding.status='pending'`，进重算队列（06 §2.5） |
| 原件 pending/failed/retry_wait | 副本同状态，进重试队列（退避计数独立） |
| 原件 `vecContentRev < contentRev`（向量过期） | **不复用**，副本 `pending`（避免把旧向量带到副本） |
| 原件 suppression.scope 含 vec/all | 拒绝审批（§3 ⑧ 已拦截） |

**本版新增第五行**：原件向量若比正文旧，不能复用 —— 否则副本从一开始就"向量与文本不符"。

---

# 6. 原件编辑与副本的关系

- 副本是**独立记录**（自己的 id/rev/contentRev/text），审批时刻的快照
- 原件后续 edit_text **不自动改副本**（副本在目标群有独立可见性）
- 控制台 `POST /api/memory/admin {op:'share_sync', requestId}` 主动同步（把原件当前 text 推到副本，副本 `contentRev+1`，Tier-1 审计）—— **可选实现（M2）**
- 原件删除 → 级联（决策 4，见 04 §5）
- **原件 contentRev 变化时**：已批准副本不受影响（保留旧快照）；SR 的 `expectedContentRevAtRequest` 已过期，但 SR 已是 approved 终态，不再校验

---

# 7. 群隔离矩阵（要求 7）

| 场景 | 行为 |
|---|---|
| 群 A 批准、群 B 拒绝 | 两条 SR 独立；A 可见 B 不可见 |
| 群 A 撤销后群 B 仍有效 | 独立 SR/copy，互不影响 |
| 原件删除 | 默认两级联（A、B 副本都删） |
| 一条记忆同时被多群申请 | 多条 pending，逐一审批；**批准 A 不影响 B 的 pending**（contentRev 未变） |
| 副本再分享 | ❌ 禁止（副本 owner 已是 group:X，不能再分享到 group:Y） |

---

# 8. 端到端流程

```
[申请]  agent ──(memoryId+token)──▶ SR(pending, eCR=3, target=group:999)
[审批]  console ──(requestId)──▶ 复查①~⑨
        ──▶ 生成 copy(mem_C, origin=mem_M@contentRev3)
        ──▶ mem_M.shareCopies += {group:999, approved, copyId}；mem_M.rev+1（contentRev 不变）
        ──▶ 群 999 检索可见 ✅
[多群]  群 888 的 pending SR 不受影响（contentRev 仍=3）✅
[撤销]  console ──(copyId)──▶ mem_C tombstone(share_revoke) ──▶ 群 999 不再可见
[原件删除] console ──(memoryId, cascade=true)──▶
        pending SR(req_2) → superseded
        mem_C(rev=1)      → tombstone(cascade, originCascade=true)
        mem_M             → tombstone
        P 同步移除
[原件编辑] edit_text(mem_M, eCR=3→4) ──▶ req_2 等 pending 全部 superseded（内容变了，需重新申请）
                                        已批准 mem_C 不受影响（独立快照）
```

---

# 9. bot_persona 类目（决策 3/9，M1）

`category='bot_persona'`（或 `factKey='bot_persona:*'`）的记忆：
- append/edit 一律进入 `pending` 审核状态，不直接 active
- 控制台批准后方 active
- 复用 ShareRequest 基础设施，但语义为"内容审核"而非"跨群授权"

---

# 10. ShareSuggestion（决策 8）

第一版**不实现**。数据结构预留 `SR.suggestion` 字段（M2 填充）。

---

# 11. M0 期间的分享（本版新增约束）

**M0 关闭分享功能**：
```
qq_memory_share_request → 501 { code:'SHARE_DISABLED_IN_M0',
                                message:'分享功能在 M0 暂未开放' }
/api/memory/admin op=share_approve/reject → 501
```

**理由**：V1 的分享依赖四级模糊匹配（F-09），在 M0 改造为 memoryId 之前，开放分享等于保留一个"可能批错记忆"的授权通道。

**M0 的 P/SR 文件**：`memory-share-requests.jsonl` 初始化为空；迁移时若 V1 有 `share_review_status='approved'` 的记录，**不自动生成副本**（因为无 SR 对象），而是：
- 在迁移报告中列出（`social-v2-orphans.json` + `v1-approved-shared.json`）
- 由人工决定是否在 M1 重新申请

---

# 12. 验收要点

1. 错误 memoryId 申请 → 404；审批时原件 contentRev 已变 → superseded + 明确提示；原件已删 → 410。
2. 两群独立授权矩阵（§7 七种组合）行为符合。
3. **批准群 A 后，群 B 的 pending SR 仍可成功审批**（本版关键，原设计会失败）。
4. 副本可追溯：`copy.origin.id` + `origin.contentRev` 反查原件。
5. private 源申请 → 400（与 V1 `bridge.js:6892` 一致）。
6. 撤销与级联后，目标群 search 不再返回该副本。
7. M0 期间 share_request → 501。
