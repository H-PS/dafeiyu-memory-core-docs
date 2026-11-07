# Memory V2.4.2 · 最终设计审核包

- 提取时间：2026-10-31 17:35
- 来源：`D:\v2.4.2-revised\`（14 份修订版文档）+ `D:\qq-bridge` 源码只读核对
- 本报告性质：**只读提取 + 独立交叉核查**，未修改设计、未实施 M0
- 标注约定：`[文档]` = 设计文档结论；`[源码]` = 已核对真实源码；`[推断]` = 设计规则推演；`[缺口]` = 本报告发现的文档缺失

## 阅读指南

| 你想知道 | 看哪节 |
|---|---|
| 架构长什么样 | §1（三层所有权 + 组件职责 + 数据流） |
| 版本怎么管、并发怎么防 | §2（三版本矩阵 + 10 组并发 + 8 类 op 管线） |
| 崩溃了怎么恢复 | §3（5 个崩溃点 + 逐点重放步骤） |
| 删除/分享/屏蔽怎么做 | §4（tombstone 三版本快照 + 审批 9 步 + 级联） |
| M0 交付什么 | §5（20 项缺陷 + 73 条验收 + 6 项已知风险） |
| 我要拍板什么 | §6（D-21/22/23）+ §10（含不做决策的后果） |
| 设计有没有胡说 | §7（10 项抽查对照表） |
| **有什么问题** | **§8（8 项独立核查）+ §9（10 个问题 + 修复优先级）** |

## 读取覆盖声明

| 引用 | 读取范围 |
|---|---|
| 00-SUMMARY | 全文（591 行，含 §4.0/§4.3/§5.1.1/§5.2/§6.3 本版新增节） |
| 02-WRITE-CONCURRENCY | 全文（503 行，含 §4.5/§8.5 本版新增节） |
| 03-AB-CONSISTENCY-WAL | 全文（417 行，含 §2.5.1/§4.1 谓词） |
| 04-LIFECYCLE-TOMBSTONE | 全文（208 行） |
| 05-SHARE-SECURITY | 全文（234 行） |
| 06-EMBEDDING-AUDIT-BACKUP | 全文（291 行） |
| 07-LOCK-AND-STARTUP | 全文（328 行） |
| 08-DECISIONS-REQUIRED | 全文（130 行） |
| 09-ACCEPTANCE-CRITERIA | 全文（223 行） |
| 10-M0-ACCEPTANCE | 全文（244 行） |
| 11-SOURCE-EVIDENCE | 全文（261 行） |
| 12-CHANGELOG | 全文（405 行） |
| FINAL-REVIEW | 全文（340 行） |
| **源码** | `memory-vector.js` 全文；`bridge.js` 定向（锁/端点/appendMemoryV2/memberImpressions/extra 四字段/interactionCount/expiresAt 全部读写点） |

**未读取**：原始 V2.4.2（`Downloads/0*-*.md`）——本报告只以修订版为准，不混入旧版结论。

---

# 1. 整体架构摘要

## 1.1 三层所有权 `[文档 00 §2 / 03 §1]`

| 层 | 文件 | 角色 | 可重建 | 唯一写者 |
|---|---|---|---|---|
| **A** | `state/memory-store.jsonl` | 长期记忆**事实源** | ❌ | MemoryService WriteQueue |
| **P** | `state/memory-projection.json` | 记忆**投影** | ✅（从 A） | Reconciler（仅在写队列内） |
| **S** | `state/social-v2.json` | 会话**运行状态** | ❌ | 43 处运行写入 + reset `[源码]` |

**为什么必须三层** `[源码]`：`social-v2.json` 实测 21 个字段，只有 3 个是记忆投影，**18 个无法从 A 重建**（`recentMessages`/`unread`/`wakeConfig`/`agentToken`/`sendTimes`/`seenForwardIds` 等）。原设计"整个 B 层 = 投影"不成立。

S 层另新增 `impressionStats: {name: {interactionCount, lastSeenAt}}`（不可重建），承载互动统计。

## 1.2 组件职责 `[文档 00 §3 / 02 §1]`

```
HTTP 端点 / MCP 5 工具
  ↓
MemoryService
  ├─ Normalizer    canonicalKey + redact + clampText(1000) + validateExtra
  ├─ Embedder      三态（M0）/ 六态（M1），【队列外】执行
  ├─ WriteQueue    进程内 FIFO，串行 P3 临界区
  ├─ Reconciler    只写 P，per-key convVersion
  ├─ Recovery      启动扫描 WAL 未提交项
  ├─ Auditor       两级审计（Tier-1 阻断 / Tier-2 补偿）
  └─ Backup        envelope + dry-run + 最小恢复校验 CLI（M0）
```

## 1.3 数据流与写入边界

```
写：请求 → P0 归一化 → P1 embed（队列外）→ P2 入队 → P3 临界区
    P3: WAL(pending) → T → A → P → SR → WAL(committed)   [文档 02 §3]
读：P 优先（formatMemoryProjection），P 缺失降级 S 三字段（M0 兼容期）
```

**硬边界** `[文档 03 §2.5]`：P-effect 内禁止 enqueue（INo-9），防 A/P 互相递归。

---

# 2. 三版本与并发规则

## 2.1 版本定义 `[文档 00 §4]`

| 字段 | 语义 | 递增条件 | 用于 |
|---|---|---|---|
| `rev` | 全量版本 | 任何持久化字段变化 | WAL / 审计 / 幂等判断 |
| `contentRev` | 内容版本 | **仅 `text` 变化** | edit_text / remove / share_* |
| `metaRev` | 元数据版本 | `extra` / `lifecycle` / `visibility` / `shareCopies` / `suppression` | edit_meta / edit_subject / suppress |
| `vecContentRev` | 向量对应内容版本 | 向量写入时 = 当时 contentRev | 防旧向量覆盖新正文 |

## 2.2 递增矩阵 `[文档 00 §4.1 / 02 §4.1]`

| 操作 | rev | contentRev | metaRev | vecContentRev |
|---|---|---|---|---|
| append | 1 | 1 | 1 | 1（vec 就绪）/0 |
| edit_text | +1 | **+1** | 不变 | 不变 |
| edit_meta | +1 | 不变 | **+1** | 不变 |
| edit_subject | +1 | 不变 | **+1** | 不变 |
| replace_extra | +1 | 不变 | **+1** | 不变 |
| embedding_refresh | +1 | 不变 | 不变 | **= contentRev** |
| shareCopies 增删 | +1 | 不变 | **+1** | 不变 |
| suppression(scope=vec) | +1 | 不变 | **+1** | 不变 |
| suppression(scope=text/all) | +1 | **+1** | **+1** | 不变 |
| lifecycle 迁移 | +1 | 不变 | **+1** | 不变 |
| remove | +1 | 不变 | 不变 | 不变 |
| restore | +1 | 不变 | **+1** | 不变 |

## 2.3 乐观锁校验 `[文档 00 §4.2 / 02 §4.4]`

| 操作 | 校验字段 | 失配 |
|---|---|---|
| edit_text | `expectedContentRev` | 409 `{currentContentRev}` |
| edit_meta / edit_subject | **`expectedMetaRev`** | 409 `{currentMetaRev}` |
| replace_extra / restore / force_restore | `expectedRev` | 409 `{currentRev}` |
| remove / share_request | `expectedContentRev` | 409 |
| share_approve | `expectedContentRevAtRequest` | SR→superseded + 409 |
| suppress / unsuppress | **`expectedMetaRev`** | 409 |

## 2.4 metadata patch 语义 `[文档 00 §4.3]`

```
edit_meta = patch（只改显式传入字段）
  未传的字段 → 保留当前值
  participants=[] → 显式清空（区别于未传）
整体替换 → 用 replace_extra（校验 expectedRev）
```

## 2.5 两个并发 metadata 案例 `[文档 02 §8.5]`

**案例 A：edit_meta ∥ edit_meta**

| t | A（participants） | B（pendingQuestion） | 结果 |
|---|---|---|---|
| t0 | M: contentRev=2, metaRev=3 | — | — |
| t1 | 入队 eMR=3 | — | — |
| t2 | 提交：metaRev→4 | — | — |
| t3 | — | 入队 eMR=3 | — |
| t4 | — | 校验 4≠3 → **409** | B 显式失败 |

**无 metaRev 时**：两者 contentRev 均为 2 → 都通过 → 若 B 是整体替换 → A 的修改被覆盖。

**案例 B：edit_meta ∥ edit_text** → 互不干扰，两者都成功（一个用 metaRev，一个用 contentRev）。

## 2.6 正文编辑与删除 `[文档 02 §7.1]`

| 排列 | 结果 |
|---|---|
| remove 先提交 | edit_text 定位失败（tombstone）→ 410 ✅ |
| edit_text 先提交 | remove 的 expectedContentRev 若为旧值 → 409；重读后重试 |

## 2.7 embedding 与正文更新 `[文档 02 §7.7 / INo-7]`

```
t1  旧 embed 完成，携带 forContentRev=2
t2  入队：检查 canWriteVec(record, 2) = (contentRev===2 && vecContentRev<2)
t3  若此时 contentRev 已被 edit 改成 3 → canWriteVec 失败 → 丢弃该向量 ✅
```

**关键不变量**：`embedding_refresh` 只在 `vecContentRev < contentRev` 且 `forContentRev === contentRev` 时写入。

## 2.8 八类操作管线 `[文档 02 §1]`

| op | embedding | 乐观锁 | 备注 |
|---|---|---|---|
| append | ✅ 队列外 | 无 | 去重命中转 touch（rev+1，contentRev 不变） |
| edit_text | ✅ | expectedContentRev | 内容变更 |
| edit_meta | ❌ | expectedMetaRev | patch |
| edit_subject | ❌ | expectedMetaRev | 跨 P 层 key 迁移 |
| replace_extra | ❌ | expectedRev | 整体替换 |
| remove / clear | ❌ | contentRev / 无 | A+P 同步 |
| share_request / approve / revoke | 可选复用向量 | contentRev / 复查 | 见 §4.4 |

## 2.9 八类操作的完整写入管线（逐 op 展开）

> 统一骨架：`P0 归一化 → P1 embed（仅两类）→ P2 入队 → P3 临界区（WAL→校验→目标态→审计→原子提交→WAL committed）→ P4 返回`

### 2.9.1 append

```
P0  canonicalKey；redact；clampText(1000)；validateExtra(category, extra)；分配 requestSeq
P1  embed(text)  ≤30s；失败 → 携带 {status:'pending'|'failed', forContentRev:1}
P2  enqueue({opId, opType:'append', params, embeddingResult})
P3.1  WAL: {opId, targetId:null, status:'pending', effects:[A insert, P project, S_stats merge, S_compat project]}
P3.2  重读 store（storeVersion）
P3.3  查重：key + category + 规范化 text + lifecycle==='active'
      ├─ 命中 → 转 touch：rev+1、updatedAt=now；contentRev/metaRev **不变**
      └─ 未命中 → 生成新 ULID
P3.4  （无乐观锁）
P3.5  目标态：rev=1, contentRev=1, metaRev=1, vecContentRev=(vec 就绪?1:0)
P3.6  （Tier-2，不阻断）
P3.7  原子提交：T(无) → A → P → S_stats → S_compat
P3.8  WAL committed + commitSeq
P3.9  Tier-2 审计
P4   {ok, opId, memoryId, rev:1, contentRev:1, embedding:{status}}
```

### 2.9.2 edit_text

```
P0/P1  同上（embed 用 newText）
P3.3   按 memoryId 定位（**不用索引、不用文本匹配**）
P3.4   record.contentRev === expectedContentRev？ 失配 → 409
P3.5   targetContentRev = contentRev+1；targetRev = rev+1；metaRev **不变**；vecContentRev 不变（待重算）
P3.6   （Tier-2）
P3.7   提交：A（text 变、向量旧）→ P（text 更新）
       ★ 此时向量落后正文 → vectorSearchable 为假 → 该记录暂不可语义检索（预期行为）
P3.9   Tier-2 审计
```

### 2.9.3 edit_meta（patch）

```
P0     validateExtra(category, patch)；**只保留显式传入的字段**
P1     （无 embed）
P3.3   按 memoryId 定位
P3.4   record.metaRev === expectedMetaRev？ 失配 → 409 {currentMetaRev}
P3.5   合并：current.extra ⊕ patch（未传字段保留；[] / "" 为显式清空）
       targetMetaRev = metaRev+1；rev+1；contentRev **不变**
P3.7   提交：A + P
★ 若 patch 同时含 text → 应走 edit_text（不允许混用，否则版本语义混乱）
```

### 2.9.4 edit_subject

```
P3.4   record.metaRev === expectedMetaRev？
P3.5   subjectId 更新；metaRev+1；rev+1
P3.7   提交：A + P（**P 层跨 key 迁移**：从旧 subject 的 traits 移除 → 插入新 subject 的 traits；
       旧 subject traits 空且 interactionCount 归零 → 删对象）
★ S_stats 也随之迁移 key（同一个 effect 内）
```

### 2.9.5 replace_extra

```
P3.4   record.rev === expectedRev？（最强操作，用全量版本）
P3.5   extra 整体替换；metaRev+1；rev+1
★ 与 edit_meta 的区别：不做 merge，直接覆盖
```

### 2.9.6 remove

```
P0     定位参数：{memoryId, expectedContentRev} | {key, category, text} | {key, category, subjectId}
P3.2   重读；按 id 或 按条件筛选（批量）
P3.3   contentRev 逐条校验（批量时任一失配 → 整体 409）
       ★ 不校验 metaRev（元数据变化不阻止删除）
       ★ 若 record.metaRev ≠ 调用方观察值 → 收集 warnings（不阻止）
P3.4   处理关联：
       - pending SR 命中 → SR.status='superseded'
       - approved 副本（cascade=true）→ 副本 tombstone(reason='cascade')
P3.5   写 tombstone：{id, prevRev, prevContentRev, prevMetaRev, contentHash}
P3.6   **Tier-1 审计（失败 → aborted，业务不发生）**
P3.7   提交：T → A（移出）→ P（移除）→ SR（revoked）
P3.8   WAL committed
P4     {ok, aRemoved, pRemoved, partial, warnings?}
```

### 2.9.7 share_request / share_approve / share_revoke

```
share_request:
  P3.3  复查原件存在 / 无 tombstone / lifecycle=active / 同 (memoryId,targetKey) 幂等
  P3.4  record.contentRev === expectedContentRev？
  P3.7  写 SR 文件（**不动 A 记录**）
  → SR.status='pending'，expectedContentRevAtRequest=当前 contentRev

share_approve（Tier-1）:
  P3.3  SR 存在且 pending
  P3.4  record.contentRev === sr.expectedContentRevAtRequest？ 失配 → SR=superseded + 409
       record.contentHash === sr.expectedContentHashAtRequest？（双保险）
  复核  tombstone / lifecycle / visibility / suppression **四项独立检查**
  P3.5  生成副本（新 ULID，contentRev=1）+ 原件 shareCopies 追加
       原件 rev+1、**metaRev+1**、contentRev **不变**
  P3.7  提交：A（原件+副本）→ P（副本在目标群可见）→ SR（approved + copyId）

share_revoke（Tier-1）:
  P3.5  副本 tombstone(reason='share_revoke')；原件 shareCopies status='revoked'
  P3.7  提交：T → A → P（移除副本）→ SR（revoked）
```

### 2.9.8 clear

```
P3.2  重读该 key 的全部记录（可按 category 过滤）
P3.3  逐条 contentRev 校验（无 expected → 删当前全部）
P3.5  批量 tombstone（reason='clear'）
P3.6  **Tier-1 审计**
P3.7  提交：T（批量）→ A（移除）→ P（**该 key 三字段置空** + convVersion+1）
      ★ 注：clear 后 P 层该 key 是"置空"还是"删除条目"，文档未明确 → 见 §8 核查 3（S-3）
P3.8  WAL committed
```

---

# 3. A/P/S 与 WAL 一致性

## 3.1 WAL 状态转换 `[文档 02 §1.2 / 03 §5]`

```
created → running → {committed | partial | failed_retriable | recovery_blocked | cancelled}
partial → {running(重试) | committed | failed_exhausted | cancelled}
```

| 路径 | WAL 状态 | HTTP | 副作用 |
|---|---|---|---|
| 目标不存在 | `aborted(not_found)` | 404 | 无 |
| 内容版本冲突 | `aborted(content_rev_conflict)` | 409 | 无 |
| 已删除 | `aborted(tombstoned)` | 410 | 无 |
| Tier-1 审计失败 | `aborted(audit_required)` | 500 | 无（业务未发生） |
| 提交 IO 失败 | 保持 `pending` | 500 | 进恢复 |
| 同 opId 已 committed | 不新增 | 首次结果 | 无 |
| 被 clear 屏障丢弃 | `aborted(barriered)` | 409 | 无（D-21 方案 A 时才可能） |

## 3.2 superseded 判定 `[文档 02 §4.3]`

```
按 op 类型分别比较：
  影响 text 的 op  → 比 contentRev
  只改元数据的 op  → 比 metaRev
绝不用 rev 比较来决定是否丢弃内容操作
```

## 3.3 崩溃点矩阵 `[文档 03 §5.2]`

| 崩溃点 | 磁盘状态 | 恢复动作 |
|---|---|---|
| ① WAL(pending) 后 | store 旧 | 标 `aborted(crashed_before_apply)` |
| ② T 写后、store 前 | tombstone 有 | 重放 A/T（按 id+contentRev 幂等） |
| ③ store 后、P 前 | A 新、P 旧 | 重放 P-effect |
| ④ P 后、SR 前 | A/P 新、SR 旧 | 重放 SR-effect |
| ⑤ SR 后、WAL(committed) 前 | 全部新 | 直接标 committed |

### 3.3.1 逐崩溃点的重放步骤（以 append 为例，A/P/S_stats/S_compat 四 effect）

**① 崩溃于 WAL(pending) 之后、任何业务写之前**

```
磁盘：store 无新记录；P 无；S 无；WAL 有 pending
恢复：
  1. 读 WAL，取 seq 最小的 pending op
  2. 逐 effect 验证：
     - A-effect：查 store 是否有该 memoryId？无 → 未生效
     - P-effect：查 P 是否有该 memoryId？无 → 未生效
     - S_stats：查 stats 是否有该 memoryId？无 → 未生效
  3. 全部未生效 + 无任何副作用 → 标 aborted(reason=crashed_before_apply)
  4. 不重放（因为 op 可能已被调用方放弃）
  ⚠️ 注意：也可选择重放（幂等 insert）。设计选择 aborted，理由："曾打算做但没做"
     用户可能需要重新提交 —— 这会导致"用户以为写了但实际没写"
     ★ 本报告认为应改为【重放】而非 aborted（见 §8 核查 6 的补充意见）
```

**② 崩溃于 tombstone 写后、store 重写前（remove 场景）**

```
磁盘：T 有该 id；store 仍有记录；P 仍有；WAL pending
恢复：
  1. 逐 effect 判定
     - T-effect：tombstone 已存在同 id → skip（已生效）
     - A-effect：store 有该 id（rev=2）→ 执行 delete（写入目标态=无该 id）
     - P-effect：移除同 id 条目
     - SR-effect：级联相关副本
  2. 标 committed
  ★ 关键：此时"记录已不在检索中"（因为 tombstone 已生效），即使 store 还有
     → 用户不会看到已删记录 → 数据安全
```

**③ 崩溃于 store 写后、P 写前**

```
磁盘：A 新（含新记录/已删）；P 旧；WAL pending
恢复：
  1. A-effect verify：store 状态 == effects.set 的目标态？是 → done
  2. P-effect：P 的 convVersion < targetConvVersion → 重放 project
  3. 标 committed
  ★ 若 P 重放失败（磁盘错误）→ recovery_blocked + 冻结集
```

**④ 崩溃于 P 写后、SR 写前（share_approve 场景）**

```
磁盘：A 新（含副本）；P 新（副本可见）；SR 旧（仍 pending）；WAL pending
恢复：
  1. A/P verify 通过
  2. SR-effect：SR.status 仍 pending？是 → 写 approved + copyId
  ★ 若跳过 SR 修复 → 副本可见但 SR 是 pending → 检索时 projectionValid 检查
    发现 SR 不是 approved → 副本不可见 → 不一致
```

**⑤ 崩溃于 SR 写后、WAL(committed) 前**

```
磁盘：全部新；WAL 仍 pending
恢复：
  1. 四个 effect 全部 verify 通过
  2. 直接标 committed（不需要重放任何东西）
  ★ 幂等：即使重放，也是 set-to-state，无副作用
```

### 3.3.2 恢复的扫描顺序与并发

```
启动时（S2 recovering 阶段，bridge.lock 已持有、writer.lock 已持有、端口未监听）：
  1. 读 MANIFEST（WAL generation）
  2. 读全部状态为非终态的 op
  3. 按 requestSeq 升序排序（不是 commitSeq，理由见 02 §5）
  4. 逐个执行 3.3.1 的重放步骤
  5. 期间若某 op 重放失败 3 次 → recovery_blocked + 该 memoryId 入冻结集
  6. 全部处理完 → 建立 keyVersion（从 P）→ 开放服务（S3 listening）
★ 恢复期间不监听端口 → 不会有新请求进来 → 无并发
```


## 3.4 Reconciler 的 per-key convVersion `[文档 00 §5.2 / 03 §2.3]`

```js
isStale(key) = !projection.conversations[key]
            || projection.conversations[key].convVersion !== keyVersion.current(key)
```

| 项 | 设计 |
|---|---|
| 内存结构 | `keyVersion: Map<key, number>` |
| 递增 | 仅该 key 的三字段变化时 |
| 启动恢复 | 从 P 的 `convVersion` 装入 |
| P 缺失 | 从 A 重建该 key → convVersion=1 |
| WAL 关联 | P-effect 带 `targetConvVersion`；重放时 P 已 ≥ 目标值则跳过 |

**keyVersion 的启动恢复（易被忽略的细节）**：

```
启动 S2 阶段：
  1. 读 memory-projection.json（若存在）
  2. for each [key, conv] of projection.conversations:
       keyVersion.set(key, conv.convVersion)
  3. 对 store 中存在但 P 中不存在的 key：
       keyVersion.set(key, 0)      ← 保证 isStale 为真，首次读会重建
  4. 对 P 中存在但 store 中不存在的 key（A 层已全删）：
       keyVersion.delete(key)
       projection.conversations[key] 保留（等下次 clear 或 rebuild 清理）
  ★ 第 4 步是 S-3（clear 后语义）的另一面：A 层记录全删了，P 层会话是否保留？
```

**重建三粒度**：

| 粒度 | 触发 | 代价 |
|---|---|---|
| 增量 | 每次 A 提交 | O(1) |
| 单会话 | `isStale(key)` | O(该会话记录数) |
| 全量 | 显式端点 / P 损坏 | O(全部) |

## 3.5 M0 双写的 effect 等级 `[文档 00 §6.3]`

| 数据 | 等级 | 失败后果 |
|---|---|---|
| A 记录（e1） | required | 整单失败 |
| P 三字段（e2） | required | partial；**读时 isStale 自愈** |
| S-stats（e3） | required | partial；**不可自愈** |
| S-compat 三字段（e4） | **optional** | committed + warn |
| 审计（e5） | optional | 补偿队列 |

**committed 条件**：e1 + e2 + e3 全部 done 且 verified。

---

# 4. 生命周期和安全机制

## 4.1 tombstone `[文档 04 §3]`

```json
{ "id":"mem_…", "prevRev":3, "prevContentRev":2, "prevMetaRev":4,
  "deletedAt":…, "deletedBy":{…}, "reason":"manual_remove|clear|cascade|retention|share_revoke",
  "contentHash":"sha256:…前16", "originCascade":bool, "relatedRequests":[…] }
```

- 永久保留，不参与备份清理，不被 checkpoint 压缩
- **不存正文**，只有 contentHash
- 记录三版本快照（供恢复审计）

## 4.2 删除的版本检查关系 `[文档 02 §4.5 / 04 §3.2]`

| 项 | 规则 |
|---|---|
| 校验字段 | `expectedContentRev`（**不校验 metaRev**） |
| 理由 | 元数据变化（如刚改过 participants）不应阻止删除；删除是收敛操作 |
| 但附警告 | 若 `record.metaRev ≠ 调用方观察值` → `warnings:['元数据已变更(metaRev X→Y)']`，不阻止 |
| 批量删除 | 逐条校验；任一失配 → 整体 409 |
| tombstone | 记三版本号 |

## 4.3 安全属性二次核验 `[文档 02 §4.4 / 05 §3]`

无论用哪个版本字段，`share_approve` / `restore` / `force_restore` **必须重新检查四项**：

```
① tombstone 命中？
② lifecycle === 'active'？
③ visibility 允许？
④ suppression 为空？
```

这四项与版本检查**独立**，不可省略。

## 4.4 分享审批与副本级联 `[文档 05 §3/§4 / 04 §5]`

**审批 9 步**（逐项失败语义）：

| # | 检查 | 失败返回 |
|---|---|---|
| ① | SR 存在且 `status==='pending'` | 404 / 409 |
| ② | 原件在 store 中存在 | SR→superseded，404 |
| ③ | 原件无 tombstone | SR→superseded，410 |
| ④ | `record.contentRev === sr.expectedContentRevAtRequest` | SR→superseded，409 `{currentContentRev}` |
| ⑤ | `record.contentHash === sr.expectedContentHashAtRequest` | 同上（双保险，防 contentRev 被误改） |
| ⑥ | `record.lifecycle === 'active'` | 409 |
| ⑦ | visibility 允许（private 源拒绝） | 403 |
| ⑧ | `record.suppression === null` | 409「原件内容已被屏蔽」 |
| ⑨ | Tier-1 审计写入成功 | 整体失败，SR 不变 |

通过后：生成副本 + 原件 `shareCopies` 追加（**rev+1、metaRev+1、contentRev 不变**）。

**第 ④ 步是本版核心**：用 contentRev 而非 rev，使得"批准群 A 的 shareCopies 变更"不会让群 B 的 pending SR 失效。

**关键收益**：批准群 A 不会让群 B 的 pending SR 失效（contentRev 未变）。

**级联**：原件删除 → 所有 `status='approved'` 副本逐条 tombstone（`originCascade=true`）+ SR→revoked。

## 4.5 检索与投影可见性谓词 `[文档 03 §4.1]`

```js
visible(record) =
     record.lifecycle === 'active'
  && (record.expiresAt == null || now < record.expiresAt)
  && !tombstone(record.id)
  && (record.contentPurged !== true)
  && (M1: record.vecModel === currentModel || 允许降级)

vectorSearchable(record) =          // 语义检索额外要求
     visible(record)
  && record.embedding.status === 'ready'
  && Array.isArray(vec) && vec.length === vecDim
  && record.vecContentRev === record.contentRev      // 向量不落后正文
  && (M1: vecModel === currentModel)

projectionVisible(entry) =           // P 层展示额外要求
     entry.contentRev === A 记录的 contentRev         // 投影不落后
  && 对应 A 记录 visible()
```

---

# 5. M0 实施边界与验收

## 5.1 M0 必须实现 `[文档 00 §8.1 / 10 §1.1]`

| 能力 | 关闭缺陷 |
|---|---|
| 写队列（上限 64/背压/取消）+ memoryId + 三版本 | F-01 |
| **A 层脱敏 + clampText(1000) + canonical key** | F-02 / F-23 |
| 两级审计（Tier-1 阻断） | F-05 / F-17 |
| 原子写（随机 tmp + fsync + renameWithRetry） | F-06 / F-15 |
| **embedding 三态 + 正文优先持久化** | F-08 |
| **tombstone + 最小恢复校验 CLI** | F-11 |
| 读失败抛错（不返回空） | F-13 |
| **P 层 + 最小 Reconciler（per-key）** | 本版新增 |
| bridge.lock 状态机（**心跳 S1** + GRACE） | F-07 |
| `aRemoved` 上报 + partial | F-03 |
| 坏行隔离 + storeVersion 缓存 | F-12 / F-16 |
| 无条件三重鉴权 + 备份时间戳 | F-20 / F-26 |

## 5.2 M0 明确不含 `[文档 00 §8.2]`

| 不含 | M0 替代 |
|---|---|
| **分享功能（ShareRequest）** | 端点返回 501 |
| **`memory-update` 端点** | 返回 501（N-01，防第 4 条分叉路径） |
| embedding 六态 + 自动重试 | 三态 + 手动 reembed |
| vecModel 强制隔离 | 记录但不强制过滤 |
| 写时淘汰（evicted/expired） | 读谓词过滤 |
| suppression | 不实现 |
| 备份周备/自动清理 | 保留全部日备 |
| 图形化恢复 | CLI 校验 |
| B 层淘汰联动 A（F-04） | 读时过滤补偿 |

## 5.3 允许降级 vs 必须失败 `[文档 00 §6.3 / 09 §8]`

| 类别 | 失败时 | 例 |
|---|---|---|
| **必须失败** | 操作整体拒绝，WAL aborted | A 写失败；Tier-1 审计失败；原子写失败 |
| **允许 partial** | 已做的保留，未做的重试 | P 写失败（读时自愈）；S-stats 失败（重试）；SR 写失败 |
| **允许降级** | 业务成功 + warn | S-compat 三字段失败；Tier-2 审计失败 |

## 5.4 M0 验收 `[文档 10 §2]`

**62 → 73 条**（本版新增 11 条），分 10 组：数据正确性 14 / 三版本与投影 12 / WAL 6 / 删除防复活 6 / Embedding 6 / 审计 7 / 锁 7 / 备份 3 / 迁移 7 / 鉴权 3。

**5 条红线**（对应 09 §13）：迁移对账不平 / 五方对账不一致 / 已删记录可见 / A 层有未脱敏正文 / Tier-1 失败仍提交 / 人工恢复绕过墓碑 / 双实例 / embed 失败丢正文（实为 8 条，文档标"5 条"为笔误 → 见 §9 S-10）。

### 5.4.1 73 条 M0 验收的分组明细

| 组 | 条数 | 代表项 |
|---|---|---|
| 数据正确性（并发） | 12 | AC-1.1~1.12（edit∥remove、edit∥clear、崩溃点、幂等、队列满） |
| 三版本与 extra | 11 | AC-2.1~2.11（edit_meta∥edit_meta、patch、EXTRA_NOT_ALLOWED、长度统一） |
| P 层与双写 | 6 | AC-2.12~2.17（per-key、interactionCount、traits 裁剪、双写等级、自愈） |
| A/P/S 一致性 | 12 | AC-3.1~3.12（脱敏、截断、partial、canonical、P 重建、S 隔离、1:N、上限） |
| WAL 与恢复 | 6 | AC-4.1~4.6（五个崩溃点、recovery_blocked、requestSeq 顺序） |
| 删除与防复活 | 6 | AC-5.1/5.3/5.4/5.5/5.7 + AC-10.6 |
| Embedding | 6 | AC-7.1/7.2/7.4/7.6/7.8 + 三态完整性 |
| 审计 | 7 | AC-8.1~8.7（Tier-1 阻断、Tier-2 补偿、零审计补齐、contentHash） |
| 锁与启动 | 7 | AC-9.1~9.7（GRACE 等待、S2 慢启动、drain、外部写侦测） |
| 备份 | 3 | AC-10.1/10.2/10.5 |
| 迁移 | 7 | AC-11.1~11.7（对账、回退、P 生成、orphan、v1-approved） |
| 鉴权 | 3 | AC-12.1~12.3 |

**M0 上线前 Checklist**（`10 §3`，5 组 20 项）：
```
数据安全 8 项：备份/对账/.v1.bak/tombstone/WAL/request-seq/P 生成/CLI 演练
写入路径 5 项：脱敏/截断/canonical/关闭 4 级匹配/无条件鉴权
可靠性 6 项：崩溃点×5/原子写失败/Tier-1 失败/Tier-2 失败/队列满/drain
锁 4 项：双进程/慢启动/kill -9/legacy
Embedding 4 项：停机 append/reembed/坏向量/旧向量不覆盖
```

### 5.4.2 M0 的已知风险（不阻塞上线，但需知情）

| # | 风险 | 影响 | 缓解 |
|---|---|---|---|
| R-1 | A 层记录只增不减（M0 无 evicted） | 磁盘年增 <10^4 条 | 读时过滤；M1 补 |
| R-2 | traits 超 10 只裁剪不淘汰 | 用户看不到超出部分 | `qq_memory_list` 可查全部 |
| R-3 | 无分享功能 | 群内记忆不能跨群 | 群内检索正常 |
| R-4 | 无 B 层编辑 | 不能改记忆文本 | remove + append |
| R-5 | 换模型需手动 reembed | 无自动批量 | 提供端点 |
| R-6 | interactionCount 可能偏差 | 见 §8 核查 1 | 无（需用户决定） |


---

# 6. D-21 ~ D-23 决策详情

## D-21 · clear 屏障语义（**阻止 M0 吗？否，有默认**）

| 项 | 内容 |
|---|---|
| 背景 | clear 之前已请求、但 embed 未完成的 append/edit，是否允许在 clear 后提交？ |
| 方案 A（推荐） | **屏障**：`op.requestSeq < clear.requestSeq` 且未提交 ⇒ 丢弃（WAL `aborted(barriered)`，409 `cleared_by_barrier`） |
| 方案 B（V1 语义） | **不屏障**：只删"提交时刻已存在"的记录；之后提交的 append 正常写入 |
| 影响 | A 符合"清了就是清了"直觉；B 会让用户看到"清空后还有一条" |
| 当前推荐 | A |
| **M0 默认** | **B**（不改 V1 行为，响应提示"clear 不阻止清空后到达的写入"） |
| 是否阻止 M0 | **否**（有默认），但影响产品语义表达 |

## D-22 · `subjectId` 是否可编辑（**影响 M0 编码**）

| 项 | 内容 |
|---|---|
| 背景 | V1 `memory-update` 支持 memberImpression 改名（`[源码] bridge.js:6812-6819`） |
| 方案 A（推荐） | **允许**，走 `edit_subject`（`expectedMetaRev`），P 层 key 原子迁移 |
| 方案 B | 禁止（改名 = 删旧 + 建新两条操作） |
| 影响 | A 保留 V1 能力；B 简单但需两次操作且可能跨事务 |
| 是否阻止 M0 | **是**（决定 `edit_subject` 是否实现）。未批准则 M0 关闭该 op |

## D-23 · `reset` 是否清 A 层（**影响 M0 编码**）

| 项 | 内容 |
|---|---|
| 背景 | V1 `[源码] bridge.js:4624` `atomicWriteJson(SOCIAL_V2_FILE, {conversations:{}})` 清空整个 social-v2.json（含记忆三字段） |
| 方案 A（推荐） | **不清 A/P**：reset 只重置 S 层会话状态，记忆保留 |
| 方案 B | 连 A/P 一起清（等于"重置一切"） |
| 影响 | A 符合"reset 是重置会话"；B 会让用户丢记忆 |
| 是否阻止 M0 | **是**（决定 reset 端点是否改）。未批准则 M0 保守：不清 A |

## 其他未批准的用户产品决策 `[文档 08]`

| # | 决策 | 本版推荐 | 未批准时默认 |
|---|---|---|---|
| D-09 | force_restore 是否提供 `ignoreTombstone` | **不提供** | 不实现（force 永不复活墓碑） |
| D-15 | 是否提供"不级联删副本"选项 | **不提供** | 强制级联 |
| D-14b | suppression(scope=text) 是否保留正文 | **保留** | 保留（M1 才用） |

**三个决策的关联影响**：

```
D-22（subjectId 可编辑）× D-23（reset 是否清 A）
  → 若允许 edit_subject + reset 不清 A：改名后的记忆在 reset 后仍存在，P 层 key 已迁移
  → 若禁止 edit_subject + reset 清 A：改名需删旧建新，reset 后全没
  建议组合：D-22 方案 A（允许）+ D-23 方案 A（不清 A），语义最一致

D-21（clear 屏障）× D-23（reset 是否清 A）
  → 两者的共同点：都是"清空"类操作
  → 若 D-21 选方案 A（屏障）但 D-23 选"不清 A"，会出现"clear 严格、reset 宽松"的不一致
  → 建议：要么都严格（A + B），要么都宽松（B + A）
  推荐：D-21 方案 A（严格）+ D-23 方案 A（reset 不清 A，因为 reset 语义不同）
  理由：clear 明确是"清记忆"，reset 是"清会话状态"——两者不该用同一把尺
```

**已废弃**：D-16（合并入 D-21）、D-17（三版本已解决）。

**未决定但不阻塞**：D-01~D-08、D-10~D-14、D-18~D-20、D-24（技术参数，可 M1 前定）。

---

# 7. 源码证据（本版新增部分）

## 7.1 E-46a~d：extra 字段的路径不一致 `[源码]`

| 编号 | 结论 | 行号 | 代码 |
|---|---|---|---|
| **E-46a** | `participants` 两路径上限不一致 | `bridge.js:9223` vs `:6794` | append：`[...set].slice(0,10)`<br>update：`= cleanNewExtra.participants`（**无上限**） |
| **E-46b** | `pendingQuestion` 截断不一致 | `:9225` vs `:6793` | append：`truncateText(...,200)`<br>update：`String(...)`（**无截断**） |
| **E-46c** | `motivation` 截断不一致 | `:9241` vs `:6799` | append：`.slice(0,50)`<br>update：`String(...)`（**无截断**） |
| **E-46d** | **A 层无 extra 载体** | `memory-vector.js:155-158` | 写入 row 只有 `key/text/time/vec/share_review_status` |

## 7.2 E-47a~c：计数与过期 `[源码]`

| 编号 | 结论 | 行号 |
|---|---|---|
| **E-47a** | `interactionCount` 仅 3 处 | `:9260`（append +1）/ `:6817`（update 可改）/ `:9199`（显示） |
| **E-47b** | `lastSeenAt` 仅 3 处 | `:9261`（append 设 now）/ `:6824` / `:9265`（淘汰排序）；**update 不更新** |
| **E-47c** | `expiresAt` 4 处过滤 | `:6842` / `:9185` / `:9211` / `:11124` |

## 7.3 N-06 / N-07 `[源码]`

- **N-06**：`extra` 三字段在 update 路径**缺少 append 的约束**（同 E-46a~c）
- **N-07**：A 层缺少 `extra` 载体（同 E-46d）

**统一处置** `[文档 00 §4.0]`：新增 `validateExtra(category, extra)`，append 与 edit 共用；越界 → 400 `EXTRA_NOT_ALLOWED`。

## 7.3.1 设计声称 vs 源码事实（抽查 10 项）`[源码]`

| 设计声称 | 源码事实 | 一致？ |
|---|---|---|
| F-01 edit 用 await 前快照 | `memory-vector.js:362` loadRows → `:367` await embed → `:372` atomicRewrite(rows) | ✅ |
| F-06 固定 tmp 名 | `:377` `const tmp = VEC_FILE + '.tmp'` | ✅ |
| F-12 坏行静默跳过 | `:53` `try{rows.push(JSON.parse(line))}catch{}` | ✅ |
| F-13 读异常返回 [] | `:57` `catch { return []; }` | ✅ |
| F-05 审计在业务后 | `bridge.js:6731` 删除 → `:6733` 审计（无 try/catch） | ✅ |
| F-11 无 tombstone/恢复函数 | 全仓 0 命中 | ✅ |
| N-01 memory-update 只改 B | `bridge.js:6765-6831` 全程无 appendVectorMemory/editVectorMemory | ✅ |
| N-02 traits 上限 10 vs 20 | `:9257` slice(0,10) / `:6815` slice(0,20) | ✅ |
| N-06 extra 三字段缺约束 | `:6794/6793/6799` 无 slice/truncate | ✅ |
| N-07 A 层无 extra | `memory-vector.js:155-158` 只写 5 字段 | ✅ |

**未抽查的**：F-02/F-03/F-04/F-07/F-08/F-09/F-10/F-14/F-16/F-17/F-19/F-20/F-22/F-23/F-24/F-25/F-26
—— 均已在前一轮核验（`11 §2/§3`），行号见该文档。

**结论**：抽查 10/10 一致，证据可信。

## 7.4 证据等级分布 `[文档 11]`

| 等级 | 数量 | 说明 |
|---|---|---|
| `SOURCE_CONFIRMED` | 74 条 | 有文件:行号 |
| `DESIGN_INFERENCE` | 5 条 | 有推演前提（D-1~D-5） |
| `NOT_VERIFIABLE` | 9 条 | 说明缺什么证据 |

**重要区分**：`SOURCE_CONFIRMED` 证明"V1 当前行为如此"，**不证明"V2 已修复"**——M0 未实施，所有修复均为设计承诺。

---

# 8. 八项独立核查结果

> 本节为**独立判断**，不重复文档结论。每项给出：核查方法 → 结论 → 证据 → 严重程度。

## 核查 1：S-stats 失败后是否真能可靠恢复？

**结论：⚠️ 不能完全可靠恢复，存在永久计数偏差风险（P1）**

| 环节 | 判断 |
|---|---|
| e3（stats）失败 → retry | ✅ 设计已定（`00 §6.3`） |
| 重试成功 | ✅ 恢复 |
| **重试耗尽** | ❌ **永久少 1 次计数**，且 `00 §6.3` 提到"M1 用 `repair-stats` 修"但**该端点未定义** |
| **重试被覆盖** | ❌ **S 层被 43 处其他写入整文件重写**（`[源码]` `saveSocialV2State` × 43）→ 重试写入的 stats 可能被下一次 `atomicWriteJson` 覆盖 |

**根因**：`interactionCount` 存在 `social-v2.json` 里，而该文件有 43 个写入者，全部是**整文件重写**。任何一次重写都会用"它读到的快照"覆盖，若那个快照早于 stats 更新，stats 被回退。

**文档缺口**：
- `00 §6.3` 说 e3 是 required，但**没说 S 层的并发写如何协调**
- `03 §2.4` 说 Reconciler 只在 WriteQueue 内，但 **P 是独立文件，不涉及 S**；S 的统计写入在队列内，却与其他 39 处 S 写入（不在队列内）竞争

**建议**（不修改设计，仅提出问题）：
1. stats 单独存一个文件（`memory-impression-stats.json`），与 social-v2.json 解耦
2. 或：所有 S 层写入也纳入 WriteQueue（代价大）
3. 或：明确声明"interactionCount 允许偏差 ±1，不保证精确"

**如何验证本问题**（M0 实施期）：
```
① 造 2 个并发：append memberImpression（写入 stats）+ 另一处 saveSocialV2State（运行状态）
   → 检查最终 stats 是否被回退
② 注入 stats 写失败 → 重试 → 同时触发 3 次其他 S 写入 → 检查最终值
③ 若 ② 复现"被覆盖" → 本核查成立 → 需按建议 1 改造
```

## 核查 2：删除只校验 contentRev 是否安全？

**结论：✅ 安全，但需补充"提交时重新核验"的明确条款（P2）**

**论证**：
```
删除是收敛操作（删掉即更安全），不存在"该删没删导致泄漏"的场景。
元数据变化（visibility 从 private 改 shared）不阻止删除 —— 删除更保守。
反向风险（删除后元数据又被改）—— 不可能，记录已移出 store。
```

**但两处需明确** `[文档 04 §3.2]`：
1. **tombstone 重复命中**：同一 id 二次删除 → 应返回幂等成功（不是 500）。文档说"tombstone 命中 → 410"，但**删除已删记录**应是 204/200 幂等，而不是 410（410 是"编辑已删记录"的语义）
2. **提交时重新核验 tombstone**：P3.4 校验 contentRev 后、P3.7 提交前，需再查一次 tombstone（防并发删除同一 id）→ `[文档 02 §1.1]` 管线里**未列这一步**

**严重程度 P2**：不会导致数据错误，但会返回错误码不一致。

**补充：删除时的安全属性是否需要二次核验？**

```
不需要，但要区分两种情况：
  ① 删除一条 lifecycle='suppressed' 的记录 → 应允许（删除比屏蔽更强）
  ② 删除一条 visibility='group_shared' 的记录 → 应允许，但需**级联删除副本**
     （否则出现"原件删了、副本还在群里"的泄漏）
★ 文档 04 §3.2 已覆盖 ②（cascade=true 默认），但未明确 ① 。
建议：文档补一句"删除不检查 lifecycle（除 pending_delete 外），也不检查 visibility，
     因为删除总是更保守；但必须级联副本"。
```

**如何验证**：
```
① 构造 lifecycle='suppressed' 的记录 → remove → 应成功（而非 409）
② 构造有 2 个 approved 副本的记录 → remove（cascade=true）→ 检查副本全部 tombstone
③ 并发：remove 与 edit_meta（改 visibility）同时 → 两者之一成功，终态一致
④ 重复 remove 同一 id → 应幂等成功（不是 410/500）
```

## 核查 3：per-key convVersion 的竞态？

**结论：⚠️ 三处未定义（P1）**

| # | 场景 | 文档现状 | 风险 |
|---|---|---|---|
| **C-1** | clear 某 key 后，P 层该会话是删除还是置空？ | **`[缺口]` 三个文档均未定义** | 若删除 `conversations[key]`，则 `isStale` 为真 → 下次读重建 → 又建出空会话（无害但脏）；若置空，需明确 `convVersion+1` |
| **C-2** | 全量重建（`rebuild-projection`）时，正在进行的写 | `[文档 00 §5.3]` 只说"所有会话置 1" | 若无队列保护，重建可能覆盖刚写的 P → 但文档说 Reconciler 在队列内，**需明确全量重建也走队列** |
| **C-3** | P 文件损坏但 convVersion 恰好与内存 keyVersion 相等 | 未定义 | 不会触发重建 → 用损坏数据。需加 `generatedAt` / checksum 校验 |

**C-1 最实际**：clear 是高频操作，每次 clear 后 P 层语义不明确会导致"清了但提醒词还有空壳"。

**如何验证 C-1**（M0 实施期）：
```
① 某 key 有 3 条记忆 → clear（无 category）
② 立即读该 key 的提醒词（formatMemoryProjection）
③ 若 P 层会话被删除 → isStale 为真 → 触发重建 → 重建后又建出空会话（traits=[]）
④ 若 P 层会话置空 → 提醒词为空（正确），convVersion 应 +1
⑤ 连续 clear 3 次 → 观察 P 文件大小是否增长（空会话堆积）
```

**如何验证 C-3**（P 文件损坏但 convVersion 匹配）：
```
① 手工把 P 文件截断一半（保留前半 + 一个伪造的 convVersion）
② 启动服务 → 读该 key
③ 若 convVersion 恰好与 keyVersion 相等 → 不触发重建 → 用损坏数据
④ 修复：P 文件加 checksum 或 generatedAt 校验
```

## 核查 4：metaRev 是否覆盖所有 metadata 入口？

**结论：✅ 覆盖完整（9 个入口），但需声明扩展规则（P3）**

| metadata 变更点 | 是否 +metaRev | 文档位置 |
|---|---|---|
| extra（participants/pendingQuestion/motivation） | ✅ | `00 §4.1` edit_meta |
| subjectId | ✅ | edit_subject |
| visibility | ✅ | share_approve/reject |
| lifecycle | ✅ | lifecycle 迁移 / restore |
| shareCopies | ✅ | share_approve/revoke |
| suppression | ✅ | suppress/unsuppress |
| expiresAt（经 edit_meta 的 expiresAtMs） | ✅ | 归 extra 变更 |
| origin（副本） | 一次性，无需 | — |
| factKey（M2） | 未定 | 需在 M2 评估 |
| migratedFrom | 迁移时一次性 | — |

**建议补充**：文档应声明"新增 A 层字段时，必须判定是否属 metadata（决定 metaRev 是否 +1）"，否则未来加字段会漏。

**如何验证覆盖完整性**：
```
① 遍历 A Schema 的每个字段，标注"属 content / metadata / 不可变"
   content:  text
   metadata: extra, visibility, lifecycle, shareCopies, suppression, expiresAt, subjectId
   不可变:   id, createdAt, migratedFrom, origin
   独立版本: vec, vecModel, vecDim（用 vecContentRev）
  → 核对 02 §4.1 的递增矩阵是否每个 metadata 字段都有 +metaRev 的场景
② 单测：对每个 metadata 操作，验证 metaRev 确实 +1 且 contentRev 不变
③ 反向：对每个 content 操作，验证 contentRev +1 且 metaRev 不变（除 suppression(text) 两者都 +1）
```

## 核查 5：P 层是否真能从 A 完整重建？

**结论：⚠️ 部分满足（P2）**

| P 层字段 | 能否从 A 重建 | 说明 |
|---|---|---|
| `activeTopics[].text/contentRev/lastMentionAt/participants/pendingQuestion` | ✅ | lastMentionAt 用 A.updatedAt 近似（语义轻微漂移） |
| `pendingThoughts[].text/createdAt/expiresAt/motivation` | ✅ | createdAt 用 A.createdAt |
| `memberImpressions[].traits[]` | ✅ | 从 A 记录按 subjectId 分组 |
| `memberImpressions[].interactionCount/lastSeenAt` | ❌ | **已迁 S 层，不在 P**（正确） |
| **空 traits 的占位对象** | ❌ | 删光某人的印象后 P 保留空对象（`interactionCount` 仍在）；重建时 A 无记录 → **该名字消失** |

**空占位丢失的影响**：无。因为 `interactionCount` 在 S 层，提示词显示时可从 stats 取；P 只负责 traits。

**结论**：P 层"按设计意图"可重建，但**重建后与增量维护的 P 会有细微差异**（空占位、lastMentionAt 近似）。文档应明确"重建后允许的差异"。

**如何验证**：
```
① 造 5 个会话共 50 条记忆 → 记录增量维护后的 P 快照 A
② 删除 P 文件 → 触发全量重建 → 得到 P 快照 B
③ 逐字段 diff A vs B：
   - traits 内容应完全一致
   - lastMentionAt 可能与 A 差（重建用 A.updatedAt）
   - 空 traits 对象在 B 中消失
④ 若 diff 超出"允许差异"清单 → 说明重建不完整
```

## 核查 6：WAL 是否覆盖 A/P/S 每种部分成功组合？

**结论：⚠️ 结构性缺陷（P1）**

**问题**：`[文档 00 §6.3]` 的 append 有 5 个 effect：
```
e1 A → e2 P → e3 S-stats → e4 S-compat → e5 audit
```

**e3 与 e4 都写 `social-v2.json`**，但被拆成两个 effect：
1. 会触发**两次整文件重写**（性能与风险翻倍）
2. 若 e3 成功、e4 失败 → 文件已写一次，第二次失败 → 状态是什么？（文档说 e4 失败 `committed+warn`，但**文件已被 e3 写过**，e4 的失败意味着"兼容副本没写"，可接受）
3. **更严重**：若 e3、e4 之间崩溃 → S 文件处于"stats 已写、compat 未写"的中间态 → 恢复时按 effect 重放 e3（幂等 merge）→ **可能重复 +1**

**修复建议**（不实施）：
- 把 e3 + e4 **合并为单个 S-effect**（一次文件重写同时写 stats 与 compat）
- 或：stats 独立文件（同核查 1 的建议）

**组合覆盖**：16 种理论组合中，按顺序执行只会出现"前缀成功"4 种 + 崩溃点 5 个，文档的矩阵覆盖了这些。**但 e3/e4 同文件是额外的结构问题**。

**补充：崩溃点 ① 的语义争议（本报告意见）**

```
设计（03 §5.2）：崩溃于 WAL(pending) 后、业务写前 → 标 aborted(crashed_before_apply)
含义：不重放，"曾打算做但没做"

问题：用户调用 append 后进程崩溃 → 用户没收到响应 → 用户可能重试
  → 若 aborted 不重放：用户重试时创建新记录（新 opId）→ 正常，无重复 ✅
  → 若用户不重试：这条记忆永久丢失（用户以为写了）

风险评级：低（用户会重试），但语义上"aborted"暗示"操作被取消"，而实际是"未执行"。

两种处理都合理：
  a) aborted（当前设计）—— 依赖用户重试
  b) 重放（upsert 幂等）—— 保证"调用了就写入"

建议：明确写入文档"崩溃于①时不重放，需调用方重试"，避免实施者误以为会自动完成。
```

## 核查 7：M0 验收是否覆盖上述风险？

**结论：⚠️ 3 处缺口（P2）**

| 核查项 | 对应 AC | 状态 |
|---|---|---|
| S-stats 失败重试 | AC-2.17 | ✅ 有，但**未测"重试被 43 处 S 写入覆盖"** |
| 删除只校 contentRev | — | ❌ **无专 AC**（AC-5.x 测的是"删除后不可见"，没测"元数据变更不阻止删除"） |
| clear → P 层语义 | — | ❌ **无 AC**（C-1 未定义，自然无验收） |
| per-key 版本 | AC-2.12 | ✅ 有，但只测"写 A 不影响 B"，未测"clear 后重建" |
| metaRev 覆盖 | AC-2.7/2.8/2.9 | ✅ 完整 |
| P 可重建 | AC-3.5 / AC-2.13 | ✅ 有，未测"空占位丢失" |
| WAL 崩溃点 | AC-1.10 | ✅ 完整 |

**建议**（不实施）：补 3 条 AC：
```
AC-5.9   删除时元数据已变更（metaRev 不同）→ 删除成功 + warnings（不是 409）
AC-3.13  clear 后 P 层该 key 三字段为空且 convVersion+1；连续 clear 不堆积空会话
AC-2.18  stats 写失败 → 重试期间触发 3 次其他 S 写入 → 最终 stats 值正确
```

**如何验证现有 AC 的覆盖度**：
```
① 对 §8 的 8 个核查项，逐项找对应 AC
② 无对应 AC 的 → 标缺口
③ 有 AC 但只测"正常路径"的 → 标"部分覆盖"
（本报告已完成此映射，见 7 处缺口标注）
```

## 核查 8：声称修复的问题是否都有源码证据？

**结论：✅ 证据充分，但需严格区分"V1 行为已证"与"V2 修复已验"（P3，表述问题）**

| 类 | 数量 | 是否可验证 |
|---|---|---|
| F-01~F-26 的 V1 行为 | 26 项 | ✅ 全部有 `[源码]` 行号 |
| N-01~N-07 的新发现 | 7 项 | ✅ 全部有行号 |
| V2 修复效果 | — | ❌ **M0 未实施，无一可验证** |

**关键提醒**：文档中"关闭缺陷"的说法应理解为"设计已覆盖该缺陷的成因"，而非"已修复"。`11 §10` 已声明这一点，但 `01-MAPPING` 的"状态"列容易误读为已修复。

**如何验证证据质量**：
```
① 抽查 10 条 SOURCE_CONFIRMED：读源码对应行号，确认代码与结论一致
② 检查是否有"设计推断被写成源码事实"（如"V1 会复活"是推断，需标 DESIGN_INFERENCE）
③ 检查行号是否随代码变动失效（本项目无 Git，行号基准是 2026-10-31 的工作区）
④ 抽查结果：本报告已核对 E-46a~d / E-47a~c / N-06 / N-07 共 9 条，全部一致 ✅
```

---

# 9. 尚未解决的问题及严重程度

| # | 问题 | 严重度 | 位置 | 影响 |
|---|---|---|---|---|
| **S-1** | S-stats 与 43 处 S 层写入竞争，重试可能被覆盖 | **P1** | `00 §6.3`、`03 §2.4` | interactionCount 永久偏差 |
| **S-2** | e3/e4 写同一文件却是两个 effect | **P1** | `00 §6.3` | 双写风险 + 崩溃中间态 + 重放重复 +1 |
| **S-3** | clear 后 P 层会话语义未定义 | **P1** | 三文档均缺 | 空会话残留 / 反复重建 |
| **S-4** | 删除已删记录返回 410 而非幂等成功 | **P2** | `04 §3.2` | 调用方误判 |
| **S-5** | 提交前未明确"再查 tombstone" | **P2** | `02 §1.1` | 并发删除同一 id 的可能 |
| **S-6** | P 重建后的允许差异未声明 | **P2** | `03 §2.3` | 重建 vs 增量维护结果不一致 |
| **S-7** | 3 条 M0 验收缺失 | **P2** | `09` | 风险未覆盖 |
| **S-8** | metaRev 扩展规则未声明 | **P3** | `00 §4.1` | 未来加字段可能漏 |
| **S-9** | `repair-stats` 端点被提及但未定义 | **P3** | `00 §6.3` | M1 承诺悬空 |
| **S-10** | `09 §13` 称"5 条红线"实为 8 条 | **P3** | `09 §13` | 文档笔误，易漏检 |

### 9.1 修复优先级建议（不改设计，仅建议顺序）

```
第 1 优先（同批解决，互相独立）
  S-1 + S-2：stats 拆独立文件（memory-impression-stats.json）
    → 一次改动同时解决：统计被覆盖、e3/e4 同文件双写
    → 代价：新增 1 个 state 文件；append memberImpression 多一次原子写
  S-3：明确 clear → P 层该 key 三字段置空 + convVersion+1
    → 纯文档补充，无代码复杂度

第 2 优先（可与其他改动同批）
  S-4：删除已删记录返回 200/204 幂等（而非 410）
  S-5：提交前补一次 tombstone 复查（1 行代码）
  S-7：补 3 条 AC

第 3 优先（文档补充）
  S-6：声明 P 重建的允许差异
  S-8：声明 metadata 字段判定规则
  S-9：定义或删除 repair-stats 承诺
  S-10：修正"5 条红线"为 8 条
```

**为什么 S-1/S-2 优先**：它会直接导致 `interactionCount` 不可靠，而该字段会进入提示词（`bridge.js:9199` 显示"互动 N 次"），影响角色表现。用户能直接感知。

**S-3 为什么算 P1**：clear 是高频操作。若语义不定，实现者可能选择"删除会话条目"，导致每次 clear 后重新建出空会话，P 文件持续增长，且提醒词可能出现"某群友：（暂无记录）"这种空壳。

### 9.2 严重度定级说明

```
P1 = 会导致用户可感知的错误结果（计数偏差、空壳提醒词、重复写入）
P2 = 会让调用方误判或返回错误码不一致（不影响数据正确性）
P3 = 文档表述问题（笔误、承诺悬空），不影响实施
```

---

# 10. 需要用户确认的事项

## 10.1 阻塞 M0 编码（2 项）

| # | 决策 | 未批准时默认 |
|---|---|---|
| **D-22** | `subjectId` 是否可编辑 | 禁止（M0 关闭 `edit_subject`） |
| **D-23** | `reset` 是否清 A 层 | 不清（保守） |

## 10.2 不阻塞但有产品语义（1 项）

| # | 决策 | 未批准时默认 |
|---|---|---|
| **D-21** | clear 屏障语义 | 方案 B（V1 行为） |

## 10.3 M1 前需定（3 项）

D-09（ignoreTombstone）、D-15（不级联选项）、D-14b（suppression 正文保留）

### 10.3.1 不做决策的后果（影响分析）

| 决策 | 若不定 | 具体后果 |
|---|---|---|
| **D-22**（subjectId 可编辑） | M0 关闭 `edit_subject` | 用户无法改群友印象的名字；只能用"删旧+建新"两次操作，且中间可能被其他写插入 |
| **D-23**（reset 是否清 A） | M0 保守不清 A | reset 后记忆仍在 → 用户可能以为"重置了一切"其实记忆还在（需在 UI 明确文案） |
| **D-21**（clear 屏障） | M0 用方案 B | clear 后可能新出现 1~N 条（embed 慢的 append 后到）→ 用户困惑"没清干净" |
| D-09（ignoreTombstone） | 不实现 | 误删的记录无法通过 UI 找回（只能手工改文件） |
| D-15（不级联选项） | 强制级联 | 用户删原件时，群里副本必然一起消失（不能选"只删我这边"） |
| D-14b（suppression 正文保留） | 保留 | 存储多一份正文副本（磁盘 + 隐私面） |

## 10.4 本报告建议用户额外决定（新增，非原设计决策）

| # | 问题 | 选项 |
|---|---|---|
| **X-1** | interactionCount 精度要求 | a) 必须精确（需 stats 独立文件）<br>b) 允许 ±1 偏差（当前设计） |
| **X-2** | clear 后 P 层会话 | a) 保留空会话 + convVersion+1<br>b) 删除会话条目 |
| **X-3** | P 重建允许的差异 | a) 接受 lastMentionAt 近似 + 空占位丢失<br>b) 要求完全一致（需 A 层补字段） |

---

# 11. 原文文档章节索引

| 主题 | 文档位置 |
|---|---|
| 三层所有权 | `00 §2`、`03 §1` |
| S 层 18 字段归属 | `00 §6.1`、`03 §1.2` |
| A 层 Schema（含 extra） | `00 §4` |
| extra 校验矩阵 | `00 §4.0` |
| 版本递增矩阵 | `00 §4.1`、`02 §4.1` |
| 乐观锁矩阵 | `00 §4.2`、`02 §4.4` |
| metaRev 论证 | `00 §4.3` |
| P 层结构（per-key） | `00 §5`、`03 §2.1` |
| memberImpressions 1:N | `00 §5.1`、`03 §2.2` |
| interactionCount 归 S | `00 §5.1.1`、`03 §2.2.2` |
| per-key convVersion | `00 §5.2`、`03 §2.3` |
| 重建三粒度 | `00 §5.3` |
| M0 双写协议 | `00 §6.3` |
| 请求顺序四字段 | `00 §7`、`02 §5` |
| M0/M1 划分 | `00 §8`、`10 §1` |
| 八类 op 与管线 | `02 §1` |
| 不变量 INo-1~9 | `02 §2` |
| 原子写规范 | `02 §3` |
| superseded 判定 | `02 §4.3` |
| 删除版本规则 | `02 §4.5`、`04 §3.2` |
| 并发案例（10 组） | `02 §7`、`02 §8.5` |
| 队列背压/drain | `02 §6` |
| WAL 条目 | `03 §3` |
| 崩溃点矩阵 | `03 §5.2` |
| 恢复扫描顺序 | `03 §5.6` |
| 迁移（含 P 生成） | `03 §6` |
| tombstone 结构 | `04 §3.1` |
| 级联与不级联 | `04 §5` |
| suppression | `04 §6` |
| force_restore | `04 §7` |
| 分享审批 9 步 | `05 §3` |
| 副本向量策略 | `05 §5` |
| embedding 三态/六态 | `06 §1` |
| 两级审计 | `06 §2` |
| 备份 envelope | `06 §3` |
| 恢复 R0~R2 | `06 §4.1` |
| M0 最小恢复 CLI | `06 §4.2` |
| 三锁职责 | `07 §1.1` |
| 启动六态 + 心跳 S1 | `07 §3` |
| stale 判定 + GRACE | `07 §4` |
| drain 修正 | `07 §5.1` |
| D-01~D-24 | `08` |
| 95 条验收 | `09` |
| M0 独立验收（73 条） | `10` |
| 源码证据（74 条） | `11` |
| 变更记录 R1~R16 | `12` |

---

# 12. 审核结论

## 12.1 设计成熟度

```
架构：三层划分清晰，边界可执行
并发：三版本 + 写队列，覆盖 10 组并发对
WAL：5 个崩溃点 + 幂等重放 + superseded 判定完整
删除：tombstone 永久 + 三版本快照 + 11 条可见性路径
M0：20 项缺陷 + 3 项新能力，73 条验收 + 5 条红线
证据：74 条 SOURCE_CONFIRMED，全部有行号
```

## 12.2 存在的实质问题（本报告独立发现）

```
P1（3 项）：S-stats 竞争覆盖、e3/e4 同文件双写、clear 后 P 层语义未定义
P2（4 项）：删除幂等语义、提交前再查 tombstone、P 重建差异未声明、3 条验收缺口
P3（3 项）：metaRev 扩展规则、repair-stats 未定义、"5 条红线"笔误
```

## 12.3 是否具备进入实施评审

**具备，但建议先解决 S-1/S-2/S-3（3 个 P1）**。

理由：
- S-1/S-2 会导致 `interactionCount` 不可靠 → 影响提示词质量
- S-3 会导致 clear 后 P 层反复重建 → 性能与一致性

**这 3 项不需要重新设计架构，只需**：
- S-1/S-2：把 stats 拆出独立文件，或合并 e3/e4 为单一 S-effect
- S-3：明确"clear → 该 key 三字段置空 + convVersion+1（保留会话条目）"

## 12.3.1 若只改这 3 处，M0 可进入实施

```
① stats 独立文件
   - 新增 state/memory-impression-stats.json
   - append memberImpression 时：A（insert）→ P（traits）→ Stats（count+1）
   - 三个 effect 都 required
   - 撤销 S-stats 与 S-compat 的双写冲突

② clear 语义明确
   - clear 某 key → P 层 conversations[key] 三字段置空 + convVersion+1
   - 不删除会话条目（保留 interactionCount 关联）
   - 补 AC-3.13

③ 删除幂等
   - 已 tombstone 的 id → remove 返回 {ok:true, alreadyDeleted:true}（200）
   - 而非 410（410 留给"编辑已删记录"）
   - 补 AC-5.9
```

**这三处都不改变架构**，只是：
- 加 1 个文件
- 明确 1 条规则
- 改 1 个返回码

## 12.4 必须重申

```
本报告为只读提取 + 交叉核查。
未修改任何设计文档、项目源码、配置、state 数据。
未实施 M0，未启动服务。
本任务不构成 M0 实施授权。
```

---

*报告结束。等待用户审核。*
