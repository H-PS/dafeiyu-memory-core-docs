# 03 · A/B 一致性、WAL、effects 与恢复机制

解决 F-02/F-03/F-04、双写分叉（E-24/E-25/E-26/E-27/E-28）与崩溃恢复。

## 1. 两层关系定义

| 维度 | A 层（memory-store.jsonl） | B 层（social-v2.json 内三字段） |
|---|---|---|
| 角色 | **事实源**（source of truth） | **投影**（projection / 摘要缓存） |
| 拥有 | memoryId、rev、lifecycle、visibility、全文、向量 | memoryId、rev 镜像、摘要文本、展示用元数据 |
| 可写者 | 仅 WriteQueue | 仅 Reconciler / WriteQueue 的 B-effect |
| 损坏后果 | 数据丢失（严重） | 可从 A 重建（`/api/socialV2/rebuild-b` 触发） |
| 读取方 | 向量检索、控制台 list、审批 | `formatMemoryV2`（提示词注入） |

**禁止**：B 层任何入口写回 A 层；A 层任何入口递归触发 B 独立写（B 变更只能是同一 op 的 effect）。

### 1.1 B 投影条目结构（social-v2.json 升级）

```json
activeTopics[i]   = { "memoryId":"mem_…", "rev":3, "text":"…前200字摘要…", "lastMentionAt":…, "participants":[…], "pendingQuestion":"…" }
pendingThoughts[i]= { "memoryId":"mem_…", "rev":1, "text":"…", "createdAt":…, "expiresAt":…, "motivation":"…" }
memberImpressions["名字"] = { "memoryId":"mem_…", "rev":4, "traits":[…], "interactionCount":…, "lastSeenAt":… }
```

- B 文本 = A 层规范化文本的**前 200 字**（同一 Normalizer 输出，仅展示截断不同）。
- 关联键 = `memoryId`，**不得**以文本相等作长期关联（任务 3 要求）。
- B 层允许与 A 层文本**不同**（B 是摘要）；一致性由 `memoryId + rev` 判定：`B.rev < A.rev` 即投影过期，读时惰性刷新或由 Reconciler 批量修复。

## 2. 统一身份字段（任务 3 交付）

| 字段 | 定义 | 谁写 | 说明 |
|---|---|---|---|
| `memoryId` | ULID（`mem_` 前缀），全局唯一、永不回收 | append 时生成；迁移时补 | A 与 B **共用同一 id** |
| `revision` | 整数，从 1 起，每次成功变更 +1 | WriteQueue | A 持有真值，B 镜像 |
| `factKey` | 见 04 §2（topic/temporary 为空） | append 时按模板生成 | 决策 1/2 |
| `owner` | canonical key，归属主体 | Normalizer | 决定默认 visibility |
| `visibility` | owner_private / group_shared | WriteQueue | 见 05 |
| `category` | activeTopic / pendingThought / memberImpression | Normalizer | V1 三类保留 |
| `lifecycleStatus` | active / pending_delete / expired / evicted / suppressed | WriteQueue | 见 04 |
| `createdAt` / `updatedAt` | 毫秒时间戳 | WriteQueue | — |

## 3. 脱敏、截断、key 规范化的执行位置（任务 3 要求 4/5/6）

| 环节 | 位置 | 规则 |
|---|---|---|
| key 规范化 | **Normalizer 入口**（MS 第一行代码） | `canonicalV2Key`；非法 key 400；A/B/store 全为 canonical（修 F-23） |
| token 脱敏 | **Normalizer 入口** | `redactKnownTokensOnly`（复用 `bridge.js:7554`）→ **原文（未脱敏）永不进入 A 层、B 层、WAL、审计、向量请求日志**（修 F-02 隐私面） |
| 全文截断 | Normalizer | ≤1000 字（与 V1 `appendVectorMemory:150` 上限一致） |
| B 摘要截断 | B-effect 构造时 | 规范化文本前 200 字（`pendingThought` V1 已是 `slice(0,200)`） |
| embed 输入截断 | Embedder | ≤4000 字（`m-v.js:88` 现状保留）；**输入为已脱敏文本** |

> **NOT_VERIFIABLE**：`truncateText`（`md-to-plain.js` 未导出）语义；V2 统一改用自实现的 `clampText(s,n)`，不依赖外部模块。

## 4. B 层生命周期与 A 层联动（任务 3 要求 7/8/9/10）

| 事件 | A 层动作 | B 层动作（同 op effect） |
|---|---|---|
| append 去重命中 | `rev+1`、`updatedAt=now`（topic 再提及） | 同 id 条目 `rev` 同步、`lastMentionAt=now` |
| 超过 20 条 topic | 最旧者 `lifecycle='evicted'` | Reconciler 移除对应 B 条目 |
| pendingThought 到期 | `lifecycle='expired'`（惰性：读时判定 + 后台 60s 扫描双保险） | 读时过滤 + Reconciler 移除 |
| 超过 50 个印象 | 同 evicted | 同上 |
| **A 层 edit** | `rev+1`、text 更新、向量重算（异步） | 同 id 条目 `rev/text` 更新（修 V1 `memory-update` 只改 B 不改 A 的分叉，E-24~E-28 之源） |
| **A 层删除** | tombstone（见 04） | 级联移除同 id 条目 + 级联分享副本 |
| B 层单独修改入口 | **取消**（`/api/socialV2/memory-update` 改为转发到 MS.edit，需 memoryId+expectedRev） | — |

**防过期召回**（任务 3 要求 10）：`ReadAPI.search/list` 过滤条件固定为
`lifecycle === 'active' && (expiresAt == null || now < expiresAt) && !tombstone(id) && (vecModel === 当前模型 || 允许降级)`。
向量层与文本层使用**同一谓词**，杜绝 V1「B 已过期、A 仍召回」（F-04）。

## 5. WAL 恢复（任务 4）

### 5.1 WAL 与业务写入的先后（任务 4 要求 1）

**WAL 先行**：`pending` 在阶段 P3.1 落盘，先于任何业务写入（tombstone/store/B）。理由：崩溃后能知道「曾打算做什么」；业务写是幂等 effect 重放，不需要在 WAL 前发生。

### 5.2 崩溃点矩阵与恢复动作

| 崩溃点 | 磁盘状态 | 恢复动作 |
|---|---|---|
| ①WAL(pending) 后、任何业务写前 | store 旧、WAL pending | effects 未发生 → 直接把 WAL 标 `aborted(reason=crashed_before_apply)`，无副作用 |
| ②tombstones 写后、store 重写前 | tombstone 有、store 仍有记录 | 重放 effect A/T（幂等：按 id+targetRev 判定）→ 重写 store → 续 |
| ③store 写后、B 投影写前 | A 新、B 旧 | 重放 B effect（幂等 set-by-id）→ 续 |
| ④B 写后、WAL(committed) 前 | 全部新、WAL 仍 pending | **无害**——按 effects 已全部就绪，直接标 committed（提交点后移原则，见 §5.4） |

### 5.3 重放幂等与 superseded（任务 4 要求 7/8/9）

重放一条 effect 前判定：

```
canApply(effect):
  T-effect: tombstone 已存在同 id → skip（已生效）
  A-effect: store 中该 id 不存在 且 effect.kind=delete → skip
            store 中该 id 不存在 且 effect.kind=upsert 且 无 tombstone → 直接 upsert（op 曾部分生效）
            store 中该 id 存在 → 仅当 record.rev < effect.rev 才 upsert   ← ★旧 revision 不得覆盖新 revision
  B-effect: B 条目 rev < effect.rev 才写入                                    ← 同上
superseded(op): 存在另一 op 同 targetId、status=committed/或更晚 pending、targetRev 更高
              → 本 op 标 superseded，effects 全部跳过（任务 4 要求 8）
```

**原理**：所有 effect 携带「完整目标态 + 目标 rev」。重放是**设置**（set-to-state）而非「增量再次施加」，因此天然幂等；`rev` 比较防止旧操作覆盖新状态。

### 5.4 提交点与「WAL 写成功但业务失败」（任务 4 要求 2）

- WAL pending 成功、业务 IO 失败 → 进程存活时走失败路径（500），WAL 标 `aborted(reason=io_error, error)`；**进程崩溃**时进入恢复（§5.2）。
- 恢复中某 effect 反复失败（如磁盘持续错误）→ 该 op 标 `recovery_blocked`（决策 7）：
  - 服务照常启动，但**该 op 涉及的 memoryId 进入冻结集**（读写均返回 503 `recovery_pending`），防止在不一致状态上叠加新写入；
  - 控制台提供 `POST /api/memory/recovery/resubmit {opId}`（控制台令牌 + Tier-1 审计）重新执行；
  - `POST /api/memory/recovery/discard {opId}` 放弃该 op（标 aborted），供人工裁决。

### 5.5 A 成功 B 失败 / B 成功 A 失败（任务 4 要求 3/4）

在统一管线中**不再出现**：A 与 B 的写是同一次「原子提交」的多个文件写入（§02 §3 顺序：T→A→B→SR→WAL-committed），任何一步失败即整体重试/恢复，恢复按 §5.2 矩阵收敛到「全部生效 + WAL committed」。**端点不再返回 V1 式 `aSynced/bSynced/partial`**，改返回 `opId + 各层计数 + recoveryNote`。

### 5.6 识别「已被后续合法操作替代的旧 effect」（要求 8）

见 §5.3 `superseded(op)`。补充：恢复扫描按 seq 升序处理，天然先旧后新；同 targetId 的高 rev 提交会遮蔽低 rev 的 pending effects。

### 5.7 人工介入（要求 10）

`recovery_blocked` 的三种出路：`resubmit`（重试）/ `discard`（放弃，需 Tier-1 审计 + 控制台令牌）/ 保持冻结。冻结集在 `/api/status` 可见。

## 6. 成功流程与三个失败恢复时序图

### 6.1 ✅ A/B 双写成功（append 完整流程）

```
模型 → MCP qq_memory_append → POST /api/socialV2/memory-append {key, category, content, extra}
P0  Normalizer: key=canonical, text=redact+clamp, subjectId 校验, opId 生成
P1  Embedder.submit(text)  ←—— 队列外，≤30s
P2  WriteQueue.enqueue({opId, append, ...})
P3.1  WAL: {opId, seq:128, append, targetId:null, status:'pending', effects:[A upsert(id,rev1), B project(id,rev1)]}
P3.2  重读 store（storeVersion=v42）
P3.3  查重：无 key+category+text 命中 → 新 id
P3.4  —（append 无 expectedRev）
P3.5  目标态 = 完整记录（embedding.status=ready, vec=…）
P3.6  （append 属 Tier-2，无阻断审计）
P3.7  提交：memory-store.jsonl 追加/重写 → social-v2.json B 条目写入 → storeVersion=v43
P3.8  WAL: status='committed'
P3.9  Tier-2 审计 append {opId, memoryId, rev:1, result:ok}（失败→补偿队列）
P4  响应 {ok:true, opId, memoryId, rev:1, embedding:{status:'ready'}}
```

### 6.2 ❄ 恢复场景 A：崩溃于 ③（store 已写、B 未写）

```
启动 → 状态机 S5 Recovery：
  扫描 WAL：seq:128 pending, effects=[A upsert(mem_W,rev1), B project(mem_W,rev1)]
  判定：
    A-effect: store 无 mem_W、无 tombstone → upsert（写入 mem_W）
    B-effect: social-v2 无 mem_W 条目 → project
  WAL: seq:128 → committed
  继续扫描直至无 pending → 冻结集空 → 进入 S6
终态：A 有 mem_W(rev1)、B 有条目、WAL committed —— 与 6.1 终态一致 ✅
```

### 6.3 ❄ 恢复场景 B：崩溃于 ②（tombstone 已写、store 仍旧）

```
启动 Recovery：
  WAL: seq:130 remove(mem_M, expectedRev:2) pending
       effects=[T tombstone(mem_M,prevRev:2), A delete(mem_M), B remove(mem_M), SR cascade(2副本)]
  判定：
    T: tombstone 已存在 → skip（已生效）
    A: store 仍有 mem_M(rev2) → delete（写入目标态=无 mem_M）
    B: 移除同 id 条目
    SR: 两个已批准副本（mem_C1/mem_C2）→ 级联 tombstone（决策 4）
  WAL → committed
终态：M 及其 2 个副本全部删除且墓碑在案 ✅（无复活）
```

### 6.4 ❄ 恢复场景 C：磁盘持续故障 → recovery_blocked

```
启动 Recovery：
  WAL: seq:132 edit(mem_X, targetRev:5) pending
  A-effect 写 store 抛 EIO ×3 → 重放失败
  → WAL: seq:132 status='recovery_blocked', error='EIO…'
  → mem_X 入冻结集；其余 op 继续恢复
  → /api/status: {recovery:{blocked:[{opId, seq:132, memoryId:mem_X, reason}…]}}
人工（控制台令牌）：
  修复磁盘后 POST /api/memory/recovery/resubmit {opId}
   → 重新入队执行 P3.2~P3.8（expectedRev 仍按原值；若期间 mem_X 已被其他 op 改到 rev>5
     → 该 op 标 superseded，返回「已被替代」）
终态：人工裁决路径，全程审计（Tier-1）✅
```

## 7. V1 → V2 迁移（任务书 §八 Q5~Q7）

**前置条件**：服务停止 + writer.lock 获取 + `node scripts/migrate-v1-to-v2.mjs`（控制台令牌）。

```
步骤 1  全量备份（envelope，见 06 §5）
步骤 2  dry-run：
        读 state/memory-vectors.jsonl（逐行）
        ├─ 解析失败行 → 收集到 badlines（不丢，见 F-12）
        ├─ 每行 → {id:新ULID, rev:1, key:canonical(旧key), category:推断(text 前缀),
        │           subjectId:null, text:redact+clamp(旧text), lifecycle:'active',
        │           embedding: 有vec?'ready':'failed', vecModel:当前配置,
        │           expiresAt:null, migratedFrom:'v1', _crc}
        ├─ 去重：同 key+category+text → 合并（保留最早 time 为 createdAt，计数报告）
        └─ 输出报告 {inputRows, badLines, duplicates, outputRecords, orphanCandidates}
步骤 3  人工确认（dry-run 令牌 + 报告签字）
步骤 4  写入 state/memory-store.jsonl.new（临时文件，原子 rename 生效）
        旧文件改名 memory-vectors.jsonl.v1.bak（只读保留，永不删除）
        tombstones 初始化为空；WAL 初始化为空
步骤 5  B 层投影：social-v2.json 三字段逐条尝试匹配（key+text 前缀）：
        匹配 → 注入 memoryId/rev=1
        未匹配 → 导出 state/social-v2-orphans.json（人工决定，不自动入 A、不自动丢弃）
步骤 6  对账：outputRecords == inputRows - badLines - duplicates；任一不等 → 整体回退
        （删除 .new，恢复 .bak 原名，退出码非 0）
步骤 7  审计（Tier-1）：MIGRATE {rows, duplicates, badlines, orphans}
```

**防丢失**：步骤 1 备份 + 步骤 6 对账 + 临时文件原子生效 + 失败整体回退。
**防复活**：迁移只读**当前** store 文件；**不触碰** `backups/`、`.pre-*.bak`、`qq-bridge备份-*`；V1 为物理删除，当前文件不含已删记录；历史备份若日后被人工恢复，由 V2 恢复流程的 tombstone 拦截（但 V1 时期删除的记录在 V2 tombstone 中无记录 → 边界见 08 D-12）。

## 8. 验收要点

1. 在 ①②③④ 四个崩溃点分别注入终止，恢复后终态与 6.1 等价（对账脚本逐条比较 store/B/WAL/tombstone）。
2. 旧 revision 重放不覆盖新状态（构造 rev=3 committed 后重放 rev=2 pending，store 保持 rev=3）。
3. 迁移对账失败注入 → 回退且原文件未被修改。
