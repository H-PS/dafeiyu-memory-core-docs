# 04 · 生命周期、tombstone、删除级联与内容屏蔽

解决 F-01（复活面）、F-04、F-11。保留决策 4/5/11/12。本版新增：contentRev 与 suppression 的关系、force_restore 的版本校验。

---

# 1. 生命周期状态机

```
                    append
                      │
                      ▼
                  ┌────────┐   到期(expiresAt)   ┌─────────┐
                  │ active │ ─────────────────▶ │ expired │
                  └────────┘                    └─────────┘
                    │   ▲                            │
   淘汰(容量上限)    │   │  再提及/重置                │
                    ▼   │                            │
                 ┌─────────┐                         │
                 │ evicted │ ──复活(M1)──────────────┘
                 └─────────┘
                    │
   suppression(独立) ▼
              ┌────────────┐
              │ suppressed │ ◀── 解除 → active
              └────────────┘
   delete（任一状态）▼
              ┌────────────────┐      ┌──────────────┐
              │ pending_delete │ ───▶ │ 物理移出 store│
              └────────────────┘      │ + tombstone  │
                                      └──────────────┘
```

| 状态 | 含义 | 可检索 | 可编辑 | 可分享 | M0 是否产生 |
|---|---|---|---|---|---|
| `active` | 正常 | ✅ | ✅ | ✅ | ✅ |
| `expired` | pendingThought 到期 | ❌ | ✅（重置→active） | ❌ | **M0 不产生**（读时过滤） |
| `evicted` | 容量淘汰 | ❌ | ✅（重提及→active） | ❌ | **M0 不产生**（投影裁剪） |
| `suppressed` | 内容屏蔽 | ❌（按 scope） | ❌（先解除） | ❌ | M1 |
| `pending_delete` | 删除事务中（仅崩溃窗口） | ❌ | ❌ | ❌ | ✅ |
| （已删） | 已 tombstone | ❌ | ❌（410） | ❌ | ✅ |

**M0 的简化**：只用 `active` / `pending_delete` / 已删三态；`expired`/`evicted`/`suppressed` 在 M1 启用。M0 通过**读时过滤 `expiresAt`** 达到"过期不召回"的效果（部分修复 F-04）。

---

# 2. factKey（保留决策 1/2/6，M2 启用）

| 类目 | factKey 模板 | 进 factKey |
|---|---|---|
| activeTopic / pendingThought | **null** | 决策 1：不进入 |
| fact | `fact:{owner}:{slug(text hash 前 12)}` | 稳定事实 |
| preference | `preference:{owner}:{subjectId\|slug}` | 偏好 |
| relationship | `relationship:{owner}:{subjectId}:{type}` | type ∈ 固定七种 |
| bot_persona | `bot_persona:{owner}` | 更新必审核 |

M0/M1 中 `factKey` 字段**存在但恒为 null**，不参与任何逻辑。

`subjectId`：memberImpression **必填**（Normalizer 强制），修 V1 `bridge.js:6932` 无法按 target 定位 A 层的问题。

---

# 3. tombstone

## 3.1 结构（`state/memory-tombstones.jsonl`，只追加）

```json
{ "id": "mem_01J8…", "prevRev": 3, "prevContentRev": 2, "prevMetaRev": 4,
  "deletedAt": 1730000000000,
  "deletedBy": { "type": "console", "tokenFp": "a1b2c3d4" },
  "reason": "manual_remove|clear|cascade|retention|share_revoke",
  "contentHash": "sha256:…前16",
  "originCascade": true|false,
  "relatedRequests": ["req_…"] }
```

- **永久保留**，不参与备份保留策略、不被 checkpoint 压缩
- **记录三版本快照**（`prevRev` / `prevContentRev` / `prevMetaRev`）：供恢复审计与 `force_restore` 的版本比对
- **不存正文**；`contentHash` 只用于审计比对与恢复去重
- 体积：≈200B/条；年删除量 <10^4 → <2MB（D-03 复核）

## 3.2 删除事务完整流程

```
remove(memoryId | key+category+text | key+category+subjectId, expectedContentRev, opts{cascade=true})
P3.1  WAL pending, effects=[T{ids}, A{delete ids}, P{remove ids}, SR{cascade}]
P3.2  重读：定位目标（1 条或 N 条）
P3.3  **contentRev 校验**（批量时逐条；任一失配 → 整体 409，不部分执行）
      ★ 删除只校验 contentRev，不校验 metaRev（元数据变更不阻止删除）
      ★ 但若 record.metaRev 与调用方观察值不同 → 响应附 warnings（不阻止，见 02 §4.5）
P3.4  待审批 SR：targetId ∈ 删除集合 → SR.status='superseded'（不产生副本）
      已批准副本（默认级联）：
        cascade=true  → 副本逐条 tombstone(reason='cascade')，SR='revoked'
        cascade=false → **必须先经确认端点**（§5.3 警告），SR 保留但标 orphan
P3.5  Tier-1 审计（失败 → aborted，业务不发生）
P3.6  原子提交：T 追加 → store 重写（移除目标）→ P 移除 → SR 更新 → WAL committed
```

- **向量处理**：A 层整条物理删除（含 vec），不留向量
- **内容清除与身份删除分离**：身份进 tombstone（无正文）；正文随记录消失。若只需屏蔽正文 → `content_suppression`（§6）

## 3.3 删除遇未完成 WAL

目标 memoryId 在冻结集（`recovery_blocked`）→ 返回 503 `recovery_pending`，提示先 `resubmit`/`discard`。
**不允许**在未决 op 之上叠加删除（避免 effect 乱序）。

## 3.4 删除后恢复备份如何识别已删

见 `06 §4`：恢复时逐条比对 tombstone `id` 集合 → 命中即跳过并在报告列明。

## 3.5 缓存防"重新发布已删除记忆"

- 缓存失效由 `storeVersion` 驱动（F-16）
- 读谓词固定含 `!tombstone(id)`（双保险）
- 检索结果返回前再过滤一次 lifecycle

---

# 4. 与 P 层的联动

| 触发 | A 层 | P 层 |
|---|---|---|
| 到期 | M0：不动（读时过滤）；M1：`expired` | 读时过滤 + 移除 |
| 容量淘汰 | M0：不动；M1：`evicted` | 投影裁剪 |
| 再提及被淘汰的条目 | `active`，`rev+1`（contentRev 不变） | 恢复条目 |
| A 删除 | tombstone | 同 id 移除 + 级联副本 |
| **P 单独删除** | **入口取消**（V1 `memory-remove` 的 B-only 路径全部走 A 层删除） | — |

---

# 5. 级联与用户选择（决策 4/5，**待批准 D-15**）

## 5.1 默认：级联删除副本

原件 tombstone 时，`shareCopies` 中所有 `status='approved'` 的副本逐条 tombstone（`reason='cascade'`，`originCascade=true`），对应 SR 标 `revoked`。

## 5.2 不级联的后果（警告文案，逐字）

> 「你选择保留分享副本。原件已删除后，这些副本将成为**孤儿**：
> ① 无法再追溯原件（origin 指向已 tombstone 的 id）；
> ② 原件的修订/撤回永远无法再影响它们；
> ③ 若原件因隐私原因删除，副本中的相同正文仍会在目标群可见。
> 此操作不可撤销。」

## 5.3 不级联的执行顺序

```
① 控制台 remove {cascade:false} → 409 REQUIRE_CONFIRM + 影响清单（副本数/目标群/contentHash）
② 二次提交 {cascade:false, confirmToken:<一次性令牌>} → 按 §3.2 执行，副本标 orphan
③ Tier-1 审计记录 WARN_ORPHAN_COPIES
```

**注意**：D-15（是否提供不级联选项）**待用户批准**。在批准前，M0/M1 只实现 `cascade=true`（强制级联），不提供 `cascade=false` 分支。

---

# 6. content_suppression（决策 12，M1）

## 6.1 与 tombstone 的区别

| 维度 | tombstone（删除） | content_suppression（屏蔽） |
|---|---|---|
| 目的 | 身份消失，防复活 | 正文不可用，身份/结构/关联保留 |
| 记录是否在 store | ❌ 物理移出 | ✅ 仍在，`lifecycle='suppressed'` |
| 正文 | ❌ | 按 scope：text→`[suppressed]`；vec→清空；all→两者 |
| **contentRev** | 不变 | **scope 含 text/all → contentRev+1**（正文变了）；scope=vec → 不变 |
| 可逆 | ❌ 永久墓碑 | ✅（解除后恢复，见 §6.4） |
| 关联（P/SR/副本） | 级联清除 | **保留**（P 显示 `[已屏蔽]`，SR 冻结） |
| 备份恢复 | 拦截 | 恢复后保持 suppressed |
| 适用 | "忘掉这条" | 下架正文、临时封禁、取证 |

## 6.2 结构

`record.suppression = { scope:'text'|'vec'|'all', at, by:{type,tokenFp}, reason }`

## 6.3 操作入口

`POST /api/memory/suppress {memoryId, expectedRev, scope, reason}` / `POST /api/memory/unsuppress {memoryId, expectedRev}`
—— 控制台令牌 + Tier-1 审计 + 写队列。**用 `expectedRev`（非 contentRev）**：安全操作保守。

## 6.4 可逆性

- `scope=text` 时原正文移入 `state/memory-suppressed-text.jsonl`（加密可选，D-14b）以支持解除
- `scope=vec/all` 不可逆（向量重算即可，无正文损失）
- **若不启用正文保留（D-14b 拒绝），text 屏蔽同样不可逆，需在警告中明示**

---

# 7. force_restore（**待批准 D-09**）

| 项 | 设计 |
|---|---|
| 权限 | 控制台令牌（Tier-1） + dry-run + 显式 `force:true` |
| 版本校验 | `expectedRev`（整条记录级强操作） |
| 执行顺序 | ① 取 writer.lock → ② 完整备份 → ③ dry-run（含将跳过的 tombstone 数）→ ④ confirmToken → ⑤ 按 tombstone 跳过 → ⑥ 冲突（同 id 且 rev 更高）跳过并列出 → ⑦ WAL 记 restore op → ⑧ 审计 |
| 与 tombstone | **默认 force 也不复活**（force 只跳过版本冲突，不跳墓碑）。若确需导入墓碑记录，必须走 `ignoreTombstone:[id…]` 白名单（逐条 + 严重警告 + 审计 `RESURRECT`） |

**D-09 待批准**：是否提供 `ignoreTombstone`。在批准前，M0/M1 **不实现**该白名单（即 force 永远不复活墓碑记录）。

---

# 8. 验收要点

1. 删除后 6 类读取路径（search/list/P 投影/分享审批/恢复导入/缓存）均不返回该 id。
2. 备份恢复后 tombstoned id 不复活；`ignoreTombstone` 白名单（若实现）逐条审计。
3. 级联删除后副本 tombstone 且 `originCascade=true`。
4. suppression 的 text scope 在启用正文保留时解除可恢复；vec scope 屏蔽后不可检索。
5. **suppression(scope=text) 后 contentRev+1 ⇒ 该记录的 pending SR 失效**（正文变了，授权应失效）。
