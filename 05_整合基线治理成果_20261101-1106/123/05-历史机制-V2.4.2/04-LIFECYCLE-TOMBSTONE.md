# 04 · memoryId、生命周期、删除、级联与防复活（tombstone）

解决 F-01（复活面）、F-04、F-11。保留决策 4/5/11/12。

## 1. 记录身份与生命周期状态机

```
                    append
                      │
                      ▼
                  ┌────────┐  到期(expiresAt)  ┌─────────┐
                  │ active │ ───────────────▶ │ expired │
                  └────────┘                  └─────────┘
                    │   ▲▲                       │
   淘汰(容量上限)    │   ││ 再提及/重置          │ 任一终态 → delete 可发生
                    ▼   ││
                 ┌─────────┐                    ┌──────────────┐
                 │ evicted │ ──复活策略见 §4.3──▶ │    active    │
                 └─────────┘                    └──────────────┘
                    │
   suppression(独立) ▼
              ┌────────────┐
              │ suppressed │ ◀── 解除 suppression → active（正文需可恢复，见 §6.4）
              └────────────┘
   delete（任一状态） ▼
              ┌────────────────┐      ┌───────────────┐
              │ pending_delete │ ───▶ │ （物理移出store │
              └────────────────┘      │  + tombstone） │
                                      └───────────────┘
```

| 状态 | 含义 | 可被检索 | 可被编辑 | 可被分享 |
|---|---|---|---|---|
| `active` | 正常 | 是 | 是 | 是 |
| `expired` | pendingThought 到期 | 否 | 是（重置 expiresAt→active） | 否 |
| `evicted` | 容量淘汰（20/50） | 否 | 是（重提及时→active） | 否 |
| `suppressed` | 内容屏蔽（独立管理） | 否（按 scope） | 否（先解除） | 否 |
| `pending_delete` | 删除事务进行中（仅崩溃窗口可见） | 否 | 否 | 否 |
| （已删） | 已 tombstone | 否 | 否（410） | 否（申请级联失效） |

## 2. factKey（保留决策 1/2/6）

| 类目 | factKey 模板 | 进 factKey 的内容 |
|---|---|---|
| activeTopic / pendingThought（topic、temporary） | **null**（不生成） | 决策 1：不进入 factKey |
| fact | `fact:{owner}:{slug(text 规范化后哈希前 12)}` | 稳定事实 |
| preference | `preference:{owner}:{subjectId\|slug}` | 偏好 |
| relationship | `relationship:{owner}:{subjectId}:{type}` | type ∈ 固定七种（决策 6） |
| bot_persona | `bot_persona:{owner}` | 更新必审核（决策 3/9） |

- 第一版（M2）实现模板引擎；M0/M1 中 `factKey` 字段**存在但可为 null**，不参与任何逻辑（避免过度设计）。
- `subjectId`：memberImpression **必填**（Normalizer 强制校验，修 V1 `bridge.js:6932` 无法按 target 定位 A 层的问题）。

## 3. tombstone（决策 11：永久保留）

### 3.1 结构（state/memory-tombstones.jsonl，只追加）

```json
{ "id":"mem_01J8…", "prevRev":3, "deletedAt":1730000000000,
  "deletedBy":{"type":"console","tokenFp":"a1b2c3d4"},
  "reason":"manual_remove|clear|cascade|retention|force_restore_cascade",
  "contentHash":"sha256:…前16", "originCascade":true|false,
  "relatedRequests":["req_…","req_…"] }
```

- **永久保留，永不清理**（不参与备份保留策略、不被 checkpoint 压缩）。
- 不存正文；`contentHash` 用于审计比对与恢复去重。
- 体积评估：纯身份行 ≈200B/条；个人场景年删除量级 <10^4 → 文件 <2MB，可接受（列入 08 D-03 复核）。

### 3.2 删除事务完整流程（任务 5）

```
remove(memoryId | key+category+text | key+category+subjectId, expectedRev, opts{cascade=true})
P3.1  WAL pending, effects=[T{ids…}, A{delete ids…}, B{remove ids…}, SR{cascade…}]
P3.2  重读：定位目标（可能 1 条或 N 条）
P3.3  rev 校验（批量删除时逐条校验；任一失配 → 整体 409，不部分执行）
P3.4  待审批 ShareRequest 处理：
        targetId ∈ 本次删除集合 → SR.status='superseded'（不产生副本）
      已批准副本处理（决策 4 默认级联）：
        cascade=true  → 副本逐条 tombstone（reason='cascade'），SR.status='revoked'
        cascade=false → **必须**先经确认端点（见 §5.3 警告流程），SR 保留但标记 orphan
P3.5  Tier-1 审计（失败 → aborted，业务不发生）
P3.6  原子提交：tombstones 追加 → store 重写（移除目标）→ B 级联 → SR 更新 → WAL committed
```

- **向量处理**：A 层整条物理删除（含 vec 数组），不留向量。
- **内容清除与身份删除分离**：身份进 tombstone（无正文）；正文随记录消失。若只需屏蔽正文 → 走 content_suppression（§6），不是删除。

### 3.3 删除遇未完成 WAL

若目标 memoryId 处于冻结集（recovery_blocked）→ 删除返回 503 `recovery_pending`，提示先 `resubmit/discard`。**不允许**在未决 op 之上叠加删除（避免 effect 乱序）。

### 3.4 删除后恢复备份如何识别已删记录（任务 5）

见 06 §5.4：恢复时逐条比对 tombstone `id` 集合 → 命中即跳过并在报告中列明。

### 3.5 缓存防「重新发布已删除记忆」（任务 5）

- 缓存失效由 `storeVersion` 计数器驱动（F-16 方案）：每次成功提交 `storeVersion+1`，读缓存以 `(storeVersion, key)` 命中。
- 读谓词固定包含 `!tombstone(id)`（双保险，即使 store 出现脏数据也不会发布）。
- 检索结果在返回前再过滤一次 lifecycle（防御性）。

## 4. 与 B 层 / 生命周期的联动（任务 3 要求 7/9）

| 触发 | A 层 | B 层 |
|---|---|---|
| 到期 | `lifecycle='expired'`（惰性 + 60s 后台扫描双保险） | 读时过滤 + Reconciler 移除 |
| 容量淘汰（20 topic / 20 thought / 50 impression） | 最旧（按 `updatedAt`/`lastSeenAt`）`lifecycle='evicted'` | 同 id 移除 |
| 再提及被淘汰的条目 | `lifecycle='active'`，`rev+1`，`updatedAt=now`（**复活策略**：evicted 可复活，expired 需显式重置，tombstoned 永不复活） | 同步恢复条目 |
| A 删除 | tombstone | 同 id 移除 + 级联副本 |
| **B 层单独删除** | **入口取消**（V1 `memory-remove` 的 B-only 路径全部改为走 A 层删除） | — |

## 5. 级联与用户选择（决策 4/5）

### 5.1 默认：级联删除副本

原件 tombstone 时，其 `shareCopies` 中所有 `status='approved'` 的副本逐条 tombstone（`reason='cascade'`，`originCascade=true`），对应 SR 标 `revoked`。

### 5.2 不级联的后果（必须写入警告文案）

> 「你选择保留分享副本。原件已删除后，这些副本将成为**孤儿**：
> ① 无法再追溯原件（origin 指向已 tombstone 的 id）；
> ② 原件的修订/撤回永远无法再影响它们；
> ③ 若原件因隐私原因删除，副本中的相同正文仍会在目标群可见。
> 此操作不可撤销。」

### 5.3 执行顺序（不级联路径）

```
① 控制台发起 remove {cascade:false} → 返回 409 REQUIRE_CONFIRM + 影响清单（副本数、目标群、contentHash）
② 控制台二次提交 remove {cascade:false, confirmToken:<一次性令牌>} → 按 §3.2 执行，副本标记 orphan
③ Tier-1 审计记录 WARN_ORPHAN_COPIES 事件
```

## 6. content_suppression（决策 12：独立管理）

### 6.1 与 tombstone 的区别（任务 5 特别要求）

| 维度 | tombstone（删除） | content_suppression（内容屏蔽） |
|---|---|---|
| 目的 | 身份消失，防复活 | 正文不可用，但**身份/结构/关联保留** |
| 记录是否还在 store | 否（物理移出） | 是（仍在，`lifecycle='suppressed'`） |
| 正文保留 | 否 | 按 scope：`text`→替换为 `[suppressed]`；`vec`→清空；`all`→两者 |
| 可逆 | 不可逆（永久墓碑） | 可逆（解除后恢复，见 §6.4） |
| 关联（B/SR/副本） | 级联清除 | **保留**（B 条目显示 `[已屏蔽]`，SR 冻结不流转） |
| 备份恢复 | 拦截（跳过） | 恢复后保持 suppressed 状态 |
| 适用场景 | 用户要求"忘掉这条" | 合规下架正文、临时封禁内容、待审计取证 |
| 审计 | Tier-1 | Tier-1 |

### 6.2 结构

`record.suppression = { scope:'text'|'vec'|'all', at, by:{type,tokenFp}, reason }`

### 6.3 操作入口（独立于删除流程）

`POST /api/memory/suppress {memoryId, scope, reason}` / `POST /api/memory/unsuppress {memoryId}` —— 均为控制台令牌 + Tier-1 审计 + 写队列。

### 6.4 可逆性实现

scope=text 时原正文移入 `state/memory-suppressed-text.jsonl`（加密可选，08 D-14b）以支持解除；scope=vec/all 不可逆（向量重算即可，无正文损失）。**若不启用正文保留（D-14b 拒绝），text 屏蔽同样不可逆，需在警告中明示。**

## 7. force_restore（任务 5）

| 项 | 设计 |
|---|---|
| 权限 | 控制台令牌（Tier-1 审计）+ dry-run 确认 + 显式 `force:true` |
| 执行顺序 | ①取 writer.lock → ②当前状态完整备份 → ③dry-run 报告（含将跳过的 tombstone 数）→ ④确认令牌 → ⑤按 tombstone 跳过导入 → ⑥冲突（同 id 存在且 rev 更高）跳过并列出 → ⑦WAL 记 restore op → ⑧审计 |
| 与 tombstone 的关系 | **默认 force 也不复活**（force 只跳过「版本冲突」，不跳过墓碑）。若运维确需导入墓碑记录（如误删恢复），必须走 ⑦ 中显式 `ignoreTombstone:[id…]` 白名单（逐条列出 + 严重警告 + 审计 `RESURRECT` 事件）→ 08 D-09 |

## 8. 验收要点

1. 删除后 6 类读取路径（search/list/B 投影/分享审批/恢复导入/缓存）均不返回该 id。
2. 备份恢复后 tombstoned id 不复活；`ignoreTombstone` 白名单逐条审计。
3. 级联删除后副本 tombstone 且 `originCascade=true`；不级联路径必须二次确认且警告文案完整。
4. suppression 的 text scope 在启用正文保留时解除可恢复；vec scope 屏蔽后向量不可检索。
