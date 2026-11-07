# 02 · A 层统一写入架构（写并发 / revision / 冲突处理）

解决 F-01（含 R1~R8 全部竞态）、F-06、F-10、F-15、F-16。

## 1. 写操作七种类型与统一管线

| op | 输入 | 需要 embedding | expectedRev | 说明 |
|---|---|---|---|---|
| `append` | key, category, text, extra{subjectId, participants, pendingQuestion, motivation, expiresAtMs} | 是（**队列外**） | 无（新记录） | 去重命中时转为 touch（rev+1） |
| `edit` | memoryId, expectedRev, newText | 是（**队列外**，对新文本） | **必填** | 旧文本仅作校验参数 |
| `remove` | scope: `{memoryId, expectedRev}` \| `{key, category, text}` \| `{key, category, subjectId}` | 否 | 单条必填 | 精确删除 |
| `clear` | key [, category] | 否 | 无 | 全清/分类清，**A+B 同步**（修复 V1 只清 B） |
| `share_request` | memoryId, expectedRev, targetKey | 否 | **必填** | 写 SR 文件，不动 A 记录 |
| `share_approve / share_reject` | requestId | 否（或复用原件向量，见 06 §2.6） | 审批时复查 | 生成/删除副本 |
| `share_revoke` | requestId \| copyId | 否 | 复查 | 撤销已批准授权 |

### 1.1 统一管线（所有写操作同一执行序列）

```
[阶段 P0 归一化]（队列外，可并行）
  canonicalKey → redactKnownTokensOnly → 截断(1000) → subjectId 校验
[阶段 P1 Embedding]（队列外，不持任何锁；仅 append/edit）
  提交 Embedder（六态状态机，见 06 §2）；
  等待结果 ≤ embedWaitMs（默认 30s，可配）；
  超时/失败 → 不中止操作，携带 embeddingIntent={text, status:pending|failed} 入队
[阶段 P2 入队]（FIFO，进程内唯一写者）
  op = {opId, opType, params, embeddingResult, actor, createdAt}
[阶段 P3 队内执行——临界区，串行]
  3.1 WAL append：{opId, seq++, opType, targetId, expectedRev, status:'pending', effects:[意图], actor, createdAt}
  3.2 重读最新：从 store 缓存（storeVersion 版本化，每次提交后失效）读取当前记录集
  3.3 定位：按 memoryId 查找（绝不用数组索引/文本匹配定位）
       ├─ append：查重（key+category+规范化 text+lifecycle==active）
       ├─ edit/remove/share_*：按 id；不存在 → 3.X 失败路径「目标不存在」
  3.4 乐观锁校验：record.rev === expectedRev？ 失配 → 失败路径「版本冲突 409」
       （tombstone 命中同 id → 失败路径「已删除 410」）
  3.5 计算目标态：targetRev = record.rev + 1；构造 effects 的完整目标记录
       （lifecycle/visibility/shareCopies/embedding/suppression 全量定值）
  3.6 Tier-1 审计写入（remove/clear/share_*/restore/migrate）；
       写失败 → 失败路径「审计阻断」，WAL→aborted，业务未发生
  3.7 原子提交（单次生效点，见 §3）：
       memory-store.jsonl 重写 + memory-tombstones.jsonl 追加 + share-requests 更新 + B 投影写
  3.8 WAL 更新 status:'committed'（同文件内该 seq 行改写，见 §4）
  3.9 Tier-2 审计（失败 → 补偿队列，不影响结果）
[阶段 P4 返回]
  {ok, opId, memoryId, rev, embedding:{status}, conflict?…}
```

**失败路径**（每条都写 WAL，保证可追溯）：

| 路径 | WAL 状态 | HTTP | 副作用 |
|---|---|---|---|
| 目标不存在 | `aborted` (reason=not_found) | 404 | 无 |
| 版本冲突 | `aborted` (reason=rev_conflict, currentRev) | 409 + currentRev | 无 |
| 已删除（tombstone 命中） | `aborted` (reason=tombstoned) | 410 | 无 |
| 审计阻断（Tier-1 失败） | `aborted` (reason=audit_required) | 500 + 明确提示 | 无（业务未发生） |
| 提交 IO 失败 | `pending`（保持） | 500 | 进入恢复（03 §5） |
| 幂等命中（同 opId 已 committed） | 不新增 WAL | 返回首次结果 | 无 |

## 2. 关键不变量

1. **INo-1 单写者**：同一时刻进程内至多一个 op 处于阶段 P3。
2. **INo-2 无旧快照**：P3.2 起的每一步都基于「本次提交前最后已提交状态」；任何 await 之后都不复用旧数组（阶段 P1 的结果只携带向量，不携带记录集）。
3. **INo-3 单点生效**：阶段 3.7 之前磁盘上不存在部分变更；3.7 是唯一的生效点。
4. **INo-4 rev 单调**：`record.rev` 只增不减；`expectedRev` 校验失败的操作不可能写盘。
5. **INo-5 opId 幂等**：同一 opId 的第二次提交不产生第二次 WAL effects。
6. **INo-6 永不越权修改**：定位失败（3.3）直接终止，**绝不**退化为索引/文本匹配去改别的记录（修复 R2「改错记录」）。

## 3. 原子提交规范（统一 `atomicWriteLines`）

```
write(file, lines):
  tmp = `${file}.${process.pid}.${crypto.randomBytes(6).toString('hex')}.tmp`
  try:
    fs.writeFileSync(tmp, lines.join('\n')+'\n', {encoding:'utf8', mode:0o600})
    fs.fsyncSync(fd)                        // 修复 F-15
    renameWithRetry(tmp, file)              // 复用 bridge.js:165-179，attempts=4
  catch e:
    try { fs.unlinkSync(tmp) } catch {}     // 无垃圾 tmp
    throw e
```

- tombstones / WAL 为**追加型**：`appendFileSync` + 追加前校验文件尾行为完整 JSON（防 F-18 类多行粘连）；追加同样在 P3 临界区内（串行保证无交错）。
- 提交顺序固定：**WAL(pending) → tombstones → store → B 投影 → share-requests → WAL(committed)**。崩溃恢复按该顺序判定进度（03 §5）。

## 4. WAL 条目与 seq

```json
{
  "opId": "op_01J8…", "seq": 128,
  "opType": "edit",
  "targetId": "mem_01J8…", "expectedRev": 2, "targetRev": 3,
  "status": "pending" | "committed" | "aborted" | "recovery_blocked" | "superseded",
  "effects": [ {"layer":"A","kind":"upsert","id":"mem_…","rev":3,"record":{…完整目标态…}},
               {"layer":"B","kind":"project","id":"mem_…","rev":3},
               {"layer":"T","kind":"tombstone","id":"mem_…","prevRev":2} ],
  "actor": {"type":"agent","id":"group:123","tokenFp":"a1b2c3d4"},
  "createdAt":1730000000000, "updatedAt":1730000000000, "error": null
}
```

- `seq`：进程内单调；跨进程重启后从 WAL 末行 +1。
- 状态改写：WAL 采用「追加新行、同 opId 旧行标记 stale」的 append-only 风格（避免原地改写破坏追加日志性质）；checkpoint 时压缩只留终态行（M2）。
- **superseded 判定**（03 §5.3）：恢复或重放时，若存在同 targetId 且 `targetRev` 更高的已提交 op，则本 op 标 `superseded`，其 effects 不执行。

## 5. 六组并发对的执行时序与终态

> 全部经同一 FIFO 队列，物理上不可能交错；下表展示**入队顺序**决定的两种排列。

### 5.1 edit ∥ 精确 remove（V1 的 R2，曾致「改错记录」）

| 时刻 | edit(e) | remove(r) | 说明 |
|---|---|---|---|
| t0 | P0/P1：embed 新文本（队列外，≤30s） | P0：无嵌入 | 两者并行 |
| t1 | | 入队 r{opId_r, id:M, expectedRev:2} | e 尚未入队 |
| t2 | | P3：定位 M → rev=2 匹配 → effects=[A upsert rev3=删除, T tombstone, B project] → Tier-1 审计 → 提交 → WAL committed | M 已删除 |
| t3 | 入队 e{opId_e, id:M, expectedRev:2} | | |
| t4 | P3.3 定位 M → **store 无此 id**（tombstone 命中）→ 3.X 失败路径 | | WAL: e→aborted(reason=tombstoned) |
| 终态 | e 返回 **410 已删除**；r 返回成功；磁盘=M 已删 + 墓碑 | | **无改错记录、无复活** ✅ |

反向排列（e 先入队提交，r 后入队）：r 的 expectedRev 若为 1 → 409 冲突，调用方重读后用 rev=2 重试 → 正常删除。均明确。

### 5.2 edit ∥ clear（V1 的 R1，曾致「复活」）

| 时刻 | edit | clear(全) | 终态 |
|---|---|---|---|
| t1 | embed 完成入队 | — | |
| t2 | P3 提交：M.rev 2→3，文本更新 | — | |
| t3 | — | 入队；P3 重读（含 t2 结果）→ 对该 key 全部 N 条生成 tombstone effects（M rev=3）→ 提交 | **N 条全删**，含刚改过的 M ✅ |

反向（clear 先提交）：edit 入队后定位失败（410）。**不存在旧快照覆盖路径** ✅。

### 5.3 edit ∥ append（V1 的 R3，曾致「新记忆丢失」）

| 时刻 | edit | append(a) | 终态 |
|---|---|---|---|
| t1 | embed 新文本中 | embed 新记忆中（并行，队列外） | |
| t2 | 入队；P3 提交（M rev2→3） | | |
| t3 | | 入队；P3 重读（含 M 新文本）→ 查重未命中 → 新记录 W(id, rev=1) 提交 | **M 已改 + W 已写**，两者都在 ✅ |

反向（a 先提交，e 后提交）：e 的 expectedRev 校验基于最新 rev，若失配 → 409，不覆盖 W ✅。

### 5.4 edit ∥ edit（V1 的 R4，曾条件性丢更新）

| 时刻 | edit1(M, eRev=2, T1) | edit2(M, eRev=2, T2) | 终态 |
|---|---|---|---|
| t1 | 入队、P3 提交 M.rev=3,text=T1 | | |
| t2 | | 入队、P3：expectedRev=2 ≠ 当前 3 → **409** | T1 生效；T2 明确失败并得到 currentRev=3，由调用方决定重试 ✅ |

**语义**：先到先赢，后到者显式冲突。这是任务书要求的「目标不存在或版本冲突时不得静默修改其他记录」的确定性实现。

### 5.5 append ∥ clear（V1 的 R7，曾致「清空后残留」）

| 排列 | 时序 | 终态 |
|---|---|---|
| clear 先提交 | append 入队后 P3 重读为空 → 查重未命中 → W 写入（active） | clear 时刻之前的记录全删；**之后**写入的 W 存在（语义正确：clear 不是「未来屏障」） |
| append 先提交 | clear 入队后 P3 重读含 W → W 一并被 tombstone | W 已删 |

两种排列都是**明确语义**，无「悄悄残留」也无「悄悄复活」✅。（若业务要求 clear 屏障后续写入，需 UI 层串行化，见 08 D-16。）

### 5.6 share 操作 ∥ edit

| 时刻 | share_request(M, eRev=2) | edit(M) | 终态 |
|---|---|---|---|
| t1 | 入队提交：SR={requestId, memoryId:M, expectedRevAtRequest:2, status:pending}（不动 M） | | |
| t2 | | 入队提交 M.rev 2→3 | |
| t3 | 审批 share_approve(requestId)：P3 复查 → M.rev=3 ≠ 2 → **SR 标 `superseded`**，返回「原件已变更，请重新申请」 | | 旧审批**不得生效** ✅（任务 6 要求 6） |

## 6. 进程内队列 vs writer.lock 的职责边界

| 维度 | 进程内 WriteQueue | 文件锁 writer.lock |
|---|---|---|
| 解决的问题 | 同进程操作**时序正确性**（谁先谁后、原子生效） | **写入者唯一性**（bridge vs 迁移/恢复/维护脚本） |
| 粒度 | 每操作 | 每进程（bridge 启动获取、退出释放） |
| 保证强度 | 强（语言级串行） | 弱（advisory，遵守者有效） |
| 不可替代性 | 无法用文件锁替代（每操作 flock 在 Windows 语义不可靠） | 无法用队列替代（跨进程） |

**对不遵守锁的外部写入者的缓解**（非保证）：store 文件首行写入 `{writerInstanceId, storeVersion, lastCommitAt}` 头；bridge 周期性核对 mtime 与 `lastCommitAt`，发现「writer.lock 空闲但 store 被外部改动」→ 强制全量重载 + Tier-1 审计告警。**NOT_VERIFIABLE**：无法防御恶意绕写。

## 7. 队列实现要点

```js
class WriteQueue {
  #chain = Promise.resolve();
  enqueue(op) {
    const p = this.#chain.then(() => this.#execute(op));
    this.#chain = p.catch(() => {});          // 失败不断链
    return p;
  }
  async drain(timeoutMs = 5000) { … }          // 退出时等待（07 §4）
}
```

- 队列无界但每 op 有 opId，调用方可取消（标记 canceled 的排队 op 在 P3 入口直接 aborted）。
- 队列深度暴露到 `/api/status`（可观测）。
- **禁止**在 P3 内调用任何会再次 enqueue 的函数（INo-1 由代码审查 + 单测保证）。

## 8. 验收要点（详见 09）

1. 上述 6 组并发对（含两种排列）共 12 个场景，全部得到「明确成功或明确失败」，终态可预测。
2. 注入 `renameSync` 失败 / Tier-1 审计失败 / 嵌入超时，各场景下 store 与 WAL 终态一致。
3. 同 opId 重放 N 次不产生 N 份记录。
