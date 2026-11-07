# 02 · A 层统一写入架构（并发 / 三版本 / 请求顺序）

解决 F-01（含 R1~R8）、F-06、F-10、F-15、F-16、N-01，以及本版新增的 rev 混用（R1）与请求顺序（R5）。

---

# 1. 写操作七种类型与统一管线

| op | 输入 | embedding | 乐观锁字段 | 说明 |
|---|---|---|---|---|
| `append` | key, category, text, extra{subjectId, participants, pendingQuestion, motivation, expiresAtMs} | 是（队列外） | 无（新记录） | 去重命中转 touch（rev+1，contentRev 不变） |
| `edit_text` | memoryId, **expectedContentRev**, newText | 是（对新文本） | **expectedContentRev** | 内容变更 |
| `edit_meta` | memoryId, **expectedMetaRev**, extra{patch 语义} | 否 | **expectedMetaRev** | 只改元数据；`rev+1`、`contentRev` 不变、`metaRev+1` |
| `edit_subject` | memoryId, **expectedMetaRev**, newSubjectId | 否 | **expectedMetaRev** | 见 01-N-03；`metaRev+1` |
| `replace_extra` | memoryId, **expectedRev**, 完整 extra | 否 | **expectedRev** | 整体替换元数据（非 patch），最强操作 |
| `remove` | scope: `{memoryId, expectedContentRev}` \| `{key, category, text}` \| `{key, category, subjectId}` | 否 | **expectedContentRev** | 精确删除 |
| `clear` | key [, category] | 否 | 无 | 全清/分类清（A + P 同步） |
| `share_request` | memoryId, **expectedContentRev**, targetKey | 否 | **expectedContentRev** | 写 SR，不动 A 记录 |
| `share_approve` / `share_reject` | requestId | 可选复用原件向量 | 审批时复查 **expectedContentRevAtRequest** | 生成/删除副本 |
| `share_revoke` | requestId \| copyId | 否 | 复查 | 撤销授权 |

## 1.1 统一管线（P0~P4）

```
[P0 归一化]（队列外，可并行）
  canonicalKey → redactKnownTokensOnly → clampText(1000) → subjectId 校验
  ★ 分配 requestSeq（进程内单调）+ requestedAt
[P1 Embedding]（队列外，不持锁；仅 append / edit_text）
  提交 Embedder；等待 ≤ embedWaitMs（默认 30s，可配）
  超时/失败 → 不中止，携带 {status: pending|failed, forContentRev} 入队
  ★ forContentRev = 本次计算对应的内容版本（防覆盖新正文）
[P2 入队]（FIFO，进程内唯一写者）
  op = {opId, requestSeq, requestedAt, opType, params, embeddingResult, actor}
[P3 队内执行——临界区，串行]
  3.1 WAL append：{opId, requestSeq, requestedAt, opType, targetId,
                    expectedContentRev, targetRev, targetContentRev,
                    status:'pending', effects:[…], actor}
  3.2 重读最新 store（storeVersion 版本化）
  3.3 定位：按 memoryId（绝不用数组索引/文本匹配）
       ├─ append：查重（key+category+规范化 text+lifecycle==active）
       ├─ 其他：按 id；不存在 → 3.X「目标不存在」
  3.4 乐观锁（按 opType 选字段，见 §4.4）：
       ├─ append          → 无此步
       ├─ edit_text/remove/share_request/share_approve
       │                  → record.contentRev === expectedContentRev？
       ├─ edit_meta/edit_subject/suppress/unsuppress
       │                  → record.metaRev === expectedMetaRev？
       ├─ replace_extra/restore/force_restore
       │                  → record.rev === expectedRev？
       ├─ 失配 → 「版本冲突 409 + current{Content,Meta,}Rev」
       └─ tombstone 命中 → 「已删除 410」
  3.5 计算目标态：targetRev = rev+1
       targetContentRev / targetMetaRev 按 §4.1 矩阵分别递增
       （lifecycle/visibility/shareCopies/embedding/suppression 全量定值）
  3.6 Tier-1 审计（remove/clear/share_*/restore/migrate/suppress）
       写失败 → 「审计阻断」，WAL→aborted，业务未发生
  3.7 原子提交（单次生效点，见 §3）：
       store 重写 + tombstones 追加 + projection 重写 + share-requests 更新
       ★ 顺序：T → A → P → SR
  3.8 WAL 更新 status:'committed' + commitSeq + committedAt
  3.9 Tier-2 审计（失败→补偿队列）
[P4 返回]
  {ok, opId, memoryId, rev, contentRev, embedding:{status}, conflict?…}
```

## 1.2 失败路径（每条都写 WAL）

| 路径 | WAL 状态 | HTTP | 副作用 |
|---|---|---|---|
| 目标不存在 | `aborted(not_found)` | 404 | 无 |
| 内容版本冲突 | `aborted(content_rev_conflict, currentContentRev)` | 409 | 无 |
| 已删除（tombstone） | `aborted(tombstoned)` | 410 | 无 |
| 审计阻断（Tier-1 失败） | `aborted(audit_required)` | 500 | 无 |
| 提交 IO 失败 | `pending`（保持） | 500 | 进入恢复（03 §5） |
| 幂等命中（同 opId 已 committed） | 不新增 | 返回首次结果 | 无 |
| **被 clear 屏障丢弃**（若 D-21=方案 A） | `aborted(barriered)` | 409 `{reason:'cleared_by_barrier'}` | 无 |

---

# 2. 关键不变量（修订版）

1. **INo-1 单写者**：同一时刻进程内至多一个 op 处于 P3。
2. **INo-2 无旧快照**：P3.2 起每步基于"本次提交前最后已提交状态"；任何 await 之后都不复用旧数组（P1 结果只携带向量 + forContentRev，不携带记录集）。
3. **INo-3 单点生效**：P3.7 之前磁盘无部分变更。
4. **INo-4 三版本单调**：`rev` / `contentRev` / `metaRev` 只增不减；任一 expected 版本失配的操作不可能写盘。
5. **INo-5 opId 幂等**：同 opId 第二次提交不产生第二次 effects。
6. **INo-6 永不越权修改**：定位失败直接终止，绝不退化为索引/文本匹配去改别的记录。
7. **INo-7 向量不追新**（★新增）：`embedding_refresh` 只在 `record.vecContentRev < record.contentRev` 时写入，且写入后 `vecContentRev = 当时的 contentRev`。**禁止用旧正文的向量覆盖新正文**。
8. **INo-8 元数据不阻断内容**（★新增）：`embedding_refresh` / `shareCopies` / `lifecycle` 变更**不得**使 `edit` / `remove` / `share_request` 的 `expectedContentRev` 失效。

---

# 3. 原子提交规范（统一 `atomicWriteLines`）

```
write(file, lines):
  tmp = `${file}.${process.pid}.${crypto.randomBytes(6).toString('hex')}.tmp`
  fd = openSync(tmp, 'w', 0o600)
  try:
    writeSync(fd, lines.join('\n')+'\n')
    fsyncSync(fd)                      // 修复 F-15
  finally:
    closeSync(fd)
  try:
    renameWithRetry(tmp, file)         // 复用 bridge.js:165-179，attempts=4
  catch e:
    try { unlinkSync(tmp) } catch {}   // 无垃圾 tmp
    throw e
```

- 追加型（tombstones / WAL）：`appendFileSync` + **追加前校验末行是完整 JSON**（防多行粘连）；同样在 P3 临界区内。
- **提交顺序固定**：`WAL(pending) → T → A(store) → P(projection) → SR → WAL(committed)`。
- 崩溃恢复按该顺序判定进度（03 §5）。

## 3.1 为什么 P 在 SR 之前

- P 是"记忆可见性"的载体，先落地能让用户尽快看到结果；
- SR 只影响分享授权，晚一步无副作用；
- 若 P 成功、SR 失败 → 记忆可见但授权未登记 → 恢复时补 SR（幂等）。

---

# 4. 三版本：rev / contentRev / metaRev

## 4.1 递增矩阵（与 00 §4.1 一致，此处为实现视角）

| op | rev | contentRev | metaRev | vecContentRev | 触发重嵌入 |
|---|---|---|---|---|---|
| append | 1 | 1 | 1 | 1（若 vec 就绪）/ 0（pending） | — |
| edit_text | +1 | **+1** | 不变 | 不变 | ✅（异步） |
| edit_meta | +1 | 不变 | **+1** | 不变 | ❌ |
| edit_subject | +1 | 不变 | **+1** | 不变 | ❌ |
| replace_extra | +1 | 不变 | **+1** | 不变 | ❌ |
| embedding_refresh | +1 | 不变 | 不变 | **= contentRev** | — |
| shareCopies 增删 | +1 | 不变 | **+1** | 不变 | ❌ |
| suppression(scope=vec) | +1 | 不变 | **+1** | 不变（vec 清空） | ❌ |
| suppression(scope=text/all) | +1 | **+1** | **+1** | 不变 | ❌ |
| lifecycle 迁移 | +1 | 不变 | **+1** | 不变 | ❌ |
| remove | +1（记录移出前） | 不变 | 不变 | 不变 | ❌ |
| restore | +1 | 不变 | **+1** | 不变 | ❌ |

**判定口诀**：`text 变 → contentRev+1`；`extra/visibility/lifecycle/shareCopies/suppression 变 → metaRev+1`；`rev 恒 +1`。

## 4.2 为什么必须多版本

原设计 `D-17` 建议"任何字段变更都 rev+1"（简单），代价是三处静默错误：

| 场景 | 单版本（原设计） | 三版本（本版） |
|---|---|---|
| 用户 edit_text 时后台刷新向量 | rev 2→3 ⇒ edit 的 expectedRev=2 失配 ⇒ **409** | contentRev 未变 ⇒ **edit 正常执行** |
| 批准群 A 分享（shareCopies+1） | 群 B 的 pending SR 全部 superseded | contentRev 未变 ⇒ **群 B 的 SR 保持有效** |
| WAL 重放 pending edit | 高 rev 的 embedding_refresh 先提交 ⇒ edit 被判 superseded ⇒ **编辑静默丢失** | superseded 只比 contentRev ⇒ **edit 正常重放** |
| **两个并发 edit_meta**（本版新增场景） | contentRev 都不变 ⇒ 两者都通过 ⇒ **后写覆盖前写** | metaRev 冲突 ⇒ 后到者 **409** |
| **edit_meta ∥ edit_text** | 若都用 contentRev ⇒ edit_meta 被 text 变更打断 | 各用各的版本 ⇒ **互不干扰** |

## 4.3 WAL 的 superseded 判定（修订）

```
superseded(op):
  // 按 op 类型分别比较对应版本
  if (op 影响 text):
     若存在同 targetId、contentRev 更高、状态 ∈ {committed, 更晚 pending}
       → 该 op 的 text 相关 effect 标 superseded
       → 该 op 的元数据 effect 仍可重放（幂等 set-to-state）
  else if (op 只改元数据):
     若存在同 targetId、metaRev 更高、状态 ∈ {committed, 更晚 pending}
       → 该 op 的元数据 effect 标 superseded
  else:
     正常重放

★ 绝不用 rev 比较来决定是否丢弃内容操作
★ 绝不用 contentRev 来决定是否丢弃元数据操作（反之亦然）
```

## 4.4 乐观锁检查矩阵（实现细节）

| op | 检查字段 | 失配返回 |
|---|---|---|
| `edit_text` | `record.contentRev === op.expectedContentRev` | 409 `{currentContentRev}` |
| `edit_meta` | **`record.metaRev === op.expectedMetaRev`** | 409 `{currentMetaRev}` |
| `edit_subject` | **`record.metaRev === op.expectedMetaRev`** | 409 `{currentMetaRev}` |
| `replace_extra` | `record.rev === op.expectedRev` | 409 `{currentRev}` |
| `remove` | `record.contentRev === op.expectedContentRev` | 409 `{currentContentRev}` |
| `share_request` | `record.contentRev === op.expectedContentRev` | 409 `{currentContentRev}` |
| `share_approve` | `record.contentRev === sr.expectedContentRevAtRequest` | SR→superseded，409 |
| `restore` / `force_restore` | `record.rev === op.expectedRev` | 409 `{currentRev}` |
| `suppress` / `unsuppress` | **`record.metaRev === op.expectedMetaRev`** | 409 `{currentMetaRev}` |

**安全属性重新核验**（不变）：无论用哪个版本字段，`share_approve` / `restore` / `force_restore` 都必须**重新检查**：
`tombstone` 命中？`lifecycle==='active'`？`visibility` 允许？`suppression` 为空？——这四项与版本检查独立，不可省略。

## 4.5 删除的版本规则（★本版明确）

`remove` 用 `expectedContentRev`，但**删除是不可逆的整条移除**，需要额外规则：

| 项 | 规则 |
|---|---|
| **校验字段** | `expectedContentRev`（用户视角："删这条内容"） |
| **不校验 metaRev 的理由** | 元数据变化（如别人刚改过 participants）不应阻止删除；删除是压倒性操作 |
| **但必须提示** | 若 `record.metaRev !== op.observedMetaRev`（调用方读到的 metaRev），响应附 `warnings:['元数据已变更（metaRev X→Y）']`，**不阻止** |
| **批量删除** | 逐条校验 contentRev；任一失配 → **整体 409**，不部分执行 |
| **删除时的版本快照** | tombstone 记录 `prevRev` + `prevContentRev` + `prevMetaRev` 三值（供恢复审计） |
| **删除后可恢复性** | 物理移出 store；tombstone 保留三版本号；`force_restore` 校验 `expectedRev` |

**并发场景**：`remove ∥ edit_meta`

```
t1  remove(M, expectedContentRev=2) 入队
t2  edit_meta(M, expectedMetaRev=3) 入队
排列 1（remove 先）：M 被删 → edit_meta 定位失败 → 404 ✅
排列 2（edit_meta 先）：metaRev 3→4，contentRev 仍 2 → remove 校验 2 ✅ 通过 → 删除
       → 响应附 warning「元数据已变更」，但删除成功
两者都不产生"删错"或"改错" ✅
```

---

# 5. 请求顺序：requestSeq / commitSeq

## 5.1 字段语义

| 字段 | 分配点 | 单调性 | 重启恢复 |
|---|---|---|---|
| `requestSeq` | P0 阶段（通过鉴权后立即） | 进程内单调 | 从 `state/memory-request-seq` + WAL 最大值 |
| `commitSeq` | P3.1（写 WAL 时） | 进程内单调（= WAL seq） | 从 WAL 末行 +1 |
| `requestedAt` | 随 requestSeq | — | — |
| `committedAt` | 随 commitSeq | — | — |

## 5.2 为什么要记录（而不是强制排序）

**必须记录**：
- 崩溃恢复后要判断"哪些请求先到"
- 审计要能回答"这条为什么被 clear 掉了"
- 屏障语义（D-21 方案 A）依赖 requestSeq

**不强制排序**：
- P1 队列外设计的初衷是并行 embed
- 若按 requestSeq 严格排序，一个 30s 的 embed 会阻塞后续所有 op
- 折中：记录 + 可选屏障（只对 clear 生效）

## 5.3 clear 屏障语义（D-21，**待用户批准**）

### 方案 A（推荐）：屏障

```
clear.requestSeq = R
所有 op 满足 op.requestSeq < R 且 status='pending'（未提交）
  → clear 提交时：这些 op 的 effects 全部丢弃，WAL 标 aborted(reason=barriered)
  → 返回 409 {reason:'cleared_by_barrier', clearedBy: clearOpId}
op.requestSeq > R → 正常执行（clear 之后的请求不受影响）
```

**语义**：clear 之前的请求 = 清理范围内；clear 之后的请求 = 新内容。

### 方案 B（V1 语义）：不屏障

```
clear 只删除"提交时刻已存在的记录"
clear 之后提交的 append（即使请求更早）→ 记录存在
```

### 对比

| 场景 | 方案 A | 方案 B |
|---|---|---|
| 用户调 clear，之前有个 append 卡在 embed | append **被丢弃** | append **写入成功**（用户看到"清空后还有一条"） |
| 用户预期 | 符合（清了就是清了） | 不符（觉得没清干净） |
| 实现复杂度 | 需按 requestSeq 过滤队列 + WAL 记 barriered | 简单（V1 现状） |
| 风险 | clear 会丢弃用户的合法写入（若用户本意是"清旧留新"） | 用户困惑 |

**本版推荐方案 A**，但**必须由用户批准**（涉及产品语义）。在批准前，M0 按**方案 B** 实现（保守，不改 V1 行为），并在响应中提示"clear 不阻止清空后到达的写入"。

## 5.4 其余顺序问题回答

| 问题 | 答案 |
|---|---|
| edit 先请求、edit 后请求，embed 完成顺序相反 | 两 edit 的 expectedContentRev 相同 ⇒ 先提交者赢，后者 409。**"先提交" = 先完成 embed**，与请求顺序无关。这是 P1 队列外的固有语义，已记录 requestSeq 供审计。若业务必须按请求顺序，需 D-24（是否开启串行 embed）|
| 服务重启后顺序语义 | requestSeq 从文件 + WAL 恢复；未提交 op 按 requestSeq 升序重放（03 §5.6） |
| WAL 记录方式 | 每条含 requestSeq / commitSeq / requestedAt / committedAt |
| 是否需要 committedAt 之外的时序 | 不需要；commitSeq 已是严格顺序 |

---

# 6. 进程内队列：背压、取消、drain

## 6.1 实现（修订）

```js
class WriteQueue {
  #chain = Promise.resolve();
  #depth = 0;
  static MAX_DEPTH = 64;          // ★新增上限

  async enqueue(op) {
    if (this.#depth >= WriteQueue.MAX_DEPTH) {
      throw Object.assign(new Error('写队列已满'), { code: 'QUEUE_FULL', depth: this.#depth });
    }
    this.#depth++;
    const p = this.#chain.then(() => {
      if (op.cancelled) return { ok: false, code: 'CANCELLED' };   // ★入口处检查取消
      return this.#execute(op);
    });
    this.#chain = p.catch(() => {});
    p.finally(() => { this.#depth--; });
    return p;
  }

  async drain(timeoutMs) { … }     // 见 6.3
}
```

## 6.2 背压与取消

| 项 | 规则 |
|---|---|
| 队列上限 | **64 个待执行 op**；超出 → HTTP 429 `{code:'QUEUE_FULL', depth}` |
| 取消（排队中） | 调用方置 `op.cancelled = true`；P3 入口检查，直接返回 `CANCELLED`（WAL 记 `aborted(cancelled)`） |
| 取消（执行中） | **不可取消**（P3 是同步 IO，中断会留半成品）；只能等其结束 |
| 深度可观测 | `/api/status` 暴露 `writeQueueDepth` |

**为什么上限 64**：单 op 最长 30s（embed 在队列外，P3 通常 < 50ms），64 个排队意味着最坏 3.2s 延迟。超过则应拒绝而非堆积。

## 6.3 drain 与 P1 的关系（修订）

```
问题：P1 embed 最长 30s，但 drain 只等 5s ⇒ 排队中的 op 会被标 recovery_blocked
⇒ 正常退出也会产生大量 recovery_blocked（污染恢复流程）

修订：
  ① drain 分为两段：
     a. 等待 P3 执行中的 op 结束（通常 < 1s）
     b. 等待排队中的 op 在 drainTimeoutMs 内进入 P3 并完成
  ② drainTimeoutMs 默认 = 5000，但若队列非空且最近 op 处于 P1（embed），
     则延长至 min(30s, 队列剩余 × 平均 P3 时间)
  ③ 超时仍未执行的排队 op → 标 aborted(reason='shutdown')，不是 recovery_blocked
     ★ recovery_blocked 只用于"P3 中途 IO 失败"，不用于"没来得及执行"
```

**修正原设计的错误**：原 `07 §5` 说"drain 超时的 op 标 recovery_blocked"——过度，会让每次正常退出都产生需人工裁决的 op。正确做法是 `aborted(shutdown)`（未执行的 op 无副作用）。

---

# 7. 八组并发对的执行时序（原 6 组 + 新增 2 组）

> 全部经同一 FIFO 队列，物理上不可能交错。下表展示**入队顺序**决定的排列。

## 7.1 edit_text ∥ remove（原 R2「改错记录」）

| 时刻 | edit(e) | remove(r) | 说明 |
|---|---|---|---|
| t0 | P1：embed 新文本 | P0：无嵌入 | 并行 |
| t1 | | 入队 r{id:M, expectedContentRev:2} | |
| t2 | | P3：contentRev=2 匹配 → effects=[A delete, T tombstone, P remove] → 审计 → 提交 | M 已删 |
| t3 | 入队 e{id:M, expectedContentRev:2} | | |
| t4 | P3.3 定位 M → **store 无此 id**（tombstone 命中）→ aborted(tombstoned) | | |
| 终态 | e 返回 **410**；r 成功；磁盘=M 已删 + 墓碑 | | **无改错、无复活** ✅ |

反向（e 先提交，r 后）：r 的 expectedContentRev 若为 1 → 409，重读后重试。

## 7.2 edit_text ∥ clear（原 R1「复活」）

| 时刻 | edit | clear(all) | 终态 |
|---|---|---|---|
| t1 | embed 完成入队 | — | |
| t2 | P3 提交：M.contentRev 2→3 | — | |
| t3 | — | 入队；P3 重读（含 t2）→ 该 key 全部 N 条生成 tombstone → 提交 | **N 条全删**，含刚改的 M ✅ |

反向（clear 先提交）：edit 定位失败（410）。**不存在旧快照覆盖** ✅。

## 7.3 edit_text ∥ append（原 R3「新记忆丢失」）

| 时刻 | edit | append(a) | 终态 |
|---|---|---|---|
| t1 | embed 中 | embed 中（并行） | |
| t2 | 入队提交（M.contentRev 2→3） | | |
| t3 | | 入队；重读（含 M 新文本）→ 查重未命中 → 新记录 W 提交 | **M 已改 + W 已写** ✅ |

反向：e 的 expectedContentRev 失配 → 409，不覆盖 W ✅。

## 7.4 edit_text ∥ edit_text（原 R4「丢更新」）

| 时刻 | edit1(M, eCR=2, T1) | edit2(M, eCR=2, T2) | 终态 |
|---|---|---|---|
| t1 | 入队、提交 M.contentRev=3, T1 | | |
| t2 | | 入队、校验 2≠3 → **409** | T1 生效；T2 明确失败 + currentContentRev=3 ✅ |

## 7.5 append ∥ clear（原 R7「清空后残留」）

| 排列 | 时序 | 终态 |
|---|---|---|
| clear 先提交 | append 入队后重读为空 → 查重未命中 → W 写入 | clear 前记录全删；之后写入的 W 存在（**D-21 方案 B 语义**） |
| append 先提交 | clear 入队后重读含 W → W 一并 tombstone | W 已删 |

若 D-21 = 方案 A：clear 提交时按 requestSeq 丢弃未提交的 append ⇒ **两种排列都不留残留**。

## 7.6 share 操作 ∥ edit_text（原 §5.6）

| 时刻 | share_request(M, eCR=2) | edit_text(M) | 终态 |
|---|---|---|---|
| t1 | 入队提交：SR={memoryId:M, expectedContentRevAtRequest:2, pending}（不动 M） | | |
| t2 | | 入队提交 M.contentRev 2→3 | |
| t3 | 审批 share_approve：复查 contentRev=3 ≠ 2 → **SR 标 superseded**，409 | | 旧审批不生效 ✅ |

**关键**：若 t2 是 `embedding_refresh`（rev+1、contentRev 不变）⇒ 审批**仍然有效** ✅（这是多版本的核心收益）。

## 7.7 ★ 新增：embedding_refresh ∥ edit_text

| 时刻 | embedding_refresh(M) | edit_text(M, eCR=2) | 终态 |
|---|---|---|---|
| t1 | 旧 embed 完成，携带 forContentRev=2 | — | |
| t2 | 入队：检查 M.vecContentRev(1) < M.contentRev(2) → 可写 | | |
| t3 | 提交：vecContentRev=2，rev+1 | | |
| t4 | — | 入队：contentRev=2 匹配 → 提交 contentRev=3 | ✅ 编辑成功 |
| t5 | — | P1 重新 embed 新文本，携带 forContentRev=3 | |
| t6 | — | 提交：检查 vecContentRev(2) < contentRev(3) → 可写 → vecContentRev=3 | 向量与新正文一致 ✅ |

**反向（edit 先提交，旧 embed 后到）**：
| 时刻 | edit_text | 迟到的 embedding_refresh(forContentRev=2) |
|---|---|---|
| t1 | 提交 contentRev=3 | — |
| t2 | — | 入队：检查 `forContentRev(2) < M.contentRev(3)` → **丢弃**（INo-7） ✅ |

**这解决了原设计的隐患**：旧向量不会覆盖新正文。

## 7.8 ★ 新增：shareCopies 变更 ∥ 其他群的 pending SR

| 时刻 | 批准群 A | 群 B 的 pending SR |
|---|---|---|
| t1 | 入队：原件 shareCopies += {group:A}；**rev+1，contentRev 不变** | — |
| t2 | 提交 | — |
| t3 | — | 审批群 B：校验 `expectedContentRevAtRequest === 原件.contentRev` → **匹配** → 审批通过 ✅ |

**原设计**：rev+1 ⇒ 群 B 的 SR superseded ⇒ 必须重新申请 ❌
**本版**：contentRev 不变 ⇒ 多群可依次审批 ✅

---

# 8. 队列与进程外写入的关系

| 维度 | 进程内 WriteQueue | 文件锁 writer.lock |
|---|---|---|
| 解决 | 同进程操作**时序正确性** | **脚本 vs 脚本**的写入者唯一性 |
| 粒度 | 每操作 | 每进程 |
| 强度 | 强（语言级串行） | 弱（advisory） |
| 不可替代 | 无法用文件锁替代（Windows flock 不可靠） | 无法用队列替代（跨进程） |

**修订说明（R8）**：原设计称 writer.lock 保护"bridge vs 脚本"。实际上 bridge 在启动时获取并**全程持有** writer.lock ⇒ 脚本必须等 bridge 停止才能获取 ⇒ **"bridge vs 脚本"的隔离由 D-08「强制停服」保证，不是由 writer.lock**。writer.lock 的真实职责是**多个维护脚本之间**的互斥。

详见 `07 §1`。

---

# 8.5 ★ 新增：元数据并发（metaRev 的两个案例）

## 8.5.1 edit_meta ∥ edit_meta（本版新增，原设计会静默覆盖）

| 时刻 | edit_meta A（participants） | edit_meta B（pendingQuestion） | 终态 |
|---|---|---|---|
| t0 | M: contentRev=2, metaRev=3, participants=["甲"] | — | — |
| t1 | 入队 A{eMR=3, patch:{participants:["甲","乙"]}} | — | — |
| t2 | P3.2 重读 → metaRev=3 匹配 → 提交：participants=["甲","乙"]，metaRev=4 | | |
| t3 | — | 入队 B{eMR=3, patch:{pendingQuestion:"要不要试试"}} | |
| t4 | — | P3.4 校验：metaRev 实际 4 ≠ 3 → **409** `{currentMetaRev:4}` | |
| 终态 | B 明确失败，调用方重读后以 eMR=4 重试 → patch 只改 pendingQuestion → **保留 A 的 participants** ✅ | | |

**若没有 metaRev**（用 contentRev）：
```
t1 A 提交：metaRev 无概念，contentRev 仍 2
t2 B 提交：contentRev=2 匹配 → 通过
   若 B 实现为"整体替换 extra" → participants 被回退为 ["甲"] ❌
   若 B 实现为"patch" → 侥幸正确（依赖实现纪律，不可靠）
```

## 8.5.2 edit_meta ∥ edit_text（互不干扰）

| 时刻 | edit_meta（participants） | edit_text（新正文） | 终态 |
|---|---|---|---|
| t1 | 入队，eMR=3 | embed 中 | |
| t2 | 提交：metaRev=4，contentRev 不变(2) | | |
| t3 | | 入队，eCR=2 | |
| t4 | | 校验 contentRev=2 匹配 → 提交：contentRev=3，metaRev 不变(4) | **两者都成功** ✅ |

**反向（edit_text 先提交）**：
```
edit_text 提交 contentRev=3，metaRev 不变(3)
edit_meta 校验 metaRev=3 仍匹配 → 提交 metaRev=4
→ 两者都成功 ✅
```

**这就是三版本的核心价值**：内容操作与元数据操作**互不成为对方的乐观锁障碍**。

---

# 9. 验收要点（详见 09 / 10）

1. 八组并发对（含两种排列）共 16 个场景，全部"明确成功或明确失败"，终态可预测。
2. **新增 2 组元数据并发**（§8.5）：edit_meta∥edit_meta 后者 409；edit_meta∥edit_text 两者都成功。
3. 注入 renameSync 失败 / Tier-1 审计失败 / 嵌入超时 / 旧向量迟到，各场景 store 与 WAL 终态一致。
4. 同 opId 重放 N 次不产生 N 份记录。
5. `embedding_refresh` 不得覆盖更高 contentRev 的正文（INo-7）。
6. `shareCopies` 变更后，其他群的 pending SR 仍可审批（INo-8）。
7. **M0 双写**：A/P 为 required、S-stats 为 required、S-compat 为 optional（见 `00 §6.3`）。
8. **P 层 per-key 版本**：写群 A 不得使群 B 的投影被判 stale（见 `00 §5.2`）。
