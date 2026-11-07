# 10 · M0 独立验收清单

> 本文件只覆盖 **M0**。M1 的验收见 `09-ACCEPTANCE-CRITERIA.md` 中标注 `[M1]` 的条目。
> M0 的定义见 `00 §8`。

---

# 1. M0 范围确认

## 1.1 M0 包含（20 项缺陷 + 3 项本版新增能力）

| 类别 | 内容 |
|---|---|
| 写路径 | MemoryService + WriteQueue（上限 64、背压、取消） |
| 身份与版本 | memoryId（ULID）+ `rev` + **`contentRev`** + `vecContentRev` |
| 持久化 | WAL（pending/committed/aborted/recovery_blocked/superseded）+ 原子写（随机 tmp + fsync + renameWithRetry） |
| 删除 | tombstone（永久）+ 读谓词过滤 + **最小恢复校验 CLI** |
| **投影** | **P 层 `memory-projection.json` + 最小 Reconciler**（增量 + 会话重建 + 全量重建） |
| Embedding | **三态**（pending/ready/failed）+ 正文优先持久化 + 手动 reembed |
| 审计 | 两级（Tier-1 阻断 / Tier-2 补偿）+ contentHash + 轮转 |
| 锁 | bridge.lock 状态机（state + instanceId + **S1 心跳** + GRACE 等待）+ writer.lock |
| 迁移 | V1 → V2（含 P 层生成 + 对账 + 失败回退） |
| 安全 | A 层脱敏 + 统一截断 1000 + canonical key + 无条件三重鉴权 |
| 兼容 | S 层三字段双写（P 优先，S 降级） |

## 1.2 M0 明确不含

| 不含 | M0 的替代 |
|---|---|
| 分享功能（ShareRequest） | 端点返回 501 |
| `memory-update` 端点 | 返回 501（N-01，避免继续制造 A/B 分叉） |
| embedding 六态 + 自动重试 | 三态 + 手动 reembed |
| vecModel 强制隔离 | 记录 vecModel，不强制过滤 |
| 写时淘汰（evicted/expired 迁移） | 读谓词过滤（`lifecycle` + `expiresAt`） |
| suppression | 不实现（M1） |
| 备份 envelope 周备/自动清理 | 保留全部日备 |
| 图形化恢复 UI | CLI `memory-restore-check.mjs` |
| `interactionCount` 迁回 S 层 | 暂存 P 层 |

---

# 2. M0 验收条目（62 条，从 09 筛选）

## 2.1 数据正确性（14 条）

| ID | 场景 | 通过标准 |
|---|---|---|
| AC-1.1 | edit ∥ remove | 后到者 410/409；无错改 |
| AC-1.2 | edit ∥ clear | 0 条复活 |
| AC-1.3 | edit ∥ append | 两者都成功 |
| AC-1.4 | edit ∥ edit | 后者 409 + currentContentRev |
| AC-1.5 | append ∥ clear | 符合 D-21 决策（M0 按方案 B） |
| AC-1.7 | opId 幂等 | 条数不变 |
| AC-1.8 | append 去重 | 1 条，rev=2，contentRev 不变 |
| AC-1.9 | 原子写失败注入 | 第 4 次成功；无 tmp 残留 |
| AC-1.10 | 崩溃点 ①②③④⑤ | 五方对账一致 |
| AC-1.11 | 旧 contentRev 重放 | 不覆盖新内容 |
| AC-1.12 | 队列满 | 429 |
| AC-2.1 | edit_meta 后 edit_text | 成功（不被元数据阻断） |
| AC-2.2 | refresh ∥ edit_text | edit 成功 |
| AC-2.3 | 旧向量迟到 | 丢弃 |

## 2.2 三版本与投影（12 条）

| ID | 场景 | 通过标准 |
|---|---|---|
| AC-2.6 | 版本字段一致性 | `contentRev ≤ rev` 且 `vecContentRev ≤ contentRev` |
| AC-3.1 | 含令牌内容 | 四方 grep 无明文 |
| AC-3.2 | >200 字 | P=前200，A≤1000，remove 可删 |
| AC-3.3 | A 少删 | partial:true + 明细 |
| AC-3.4 | 前导零 key | 统一为 `group:123` |
| AC-3.5 | P 损坏 | 会话重建后对齐 |
| AC-3.6 | S 层不被 P 影响 | recentMessages/unread 完整 |
| AC-3.7 | P 重建保留 interactionCount | 从旧 P 合并 |
| AC-3.8 | memberImpressions 删 1 条 | traits 剩 2 且正确 |
| AC-3.9 | memberImpressions 编辑 | 对应 trait 更新 |
| AC-3.10 | traits 上限 10 | 两条路径一致 |
| AC-3.11 | A/P 不互相触发 | P-effect 无 enqueue |
| AC-3.12 | memory-update → 501 | 返回 501 |

## 2.3 WAL 与恢复（6 条）

AC-4.1 ~ AC-4.6（同 09）

## 2.4 删除与防复活（6 条）

AC-5.1、AC-5.3、AC-5.4、AC-5.5、AC-5.7 + **AC-10.6（CLI 恢复校验）**

## 2.5 Embedding（6 条）

AC-7.1、AC-7.2、AC-7.4、AC-7.6、AC-7.8 + 三态完整性

## 2.6 审计（7 条）

AC-8.1 ~ AC-8.7

## 2.7 锁与启动（7 条）

AC-9.1 ~ AC-9.7

## 2.8 备份（3 条）

AC-10.1、AC-10.2、AC-10.5

## 2.9 迁移（7 条）

AC-11.1 ~ AC-11.7

## 2.10 鉴权（3 条）

AC-12.1 ~ AC-12.3

---

# 3. M0 上线前 Checklist（必须全部打勾）

## 3.1 数据安全

```
□ 迁移前已完成 envelope 全量备份，且 digest 校验通过
□ 迁移对账通过（output == input - badLines - duplicates）
□ 旧文件 memory-vectors.jsonl.v1.bak 存在且只读
□ tombstone 文件已初始化（哪怕为空）
□ WAL 目录已初始化
□ memory-request-seq 已初始化
□ memory-projection.json 已生成且与 store 对齐
□ CLI 恢复校验演练成功（用含已删记录的备份 dry-run，确认全部被跳过）
```

## 3.2 写入路径

```
□ A 层脱敏生效（构造含会话令牌的 content，grep store 无明文）
□ 截断 1000 生效
□ canonical key 生效（group:0123 → group:123）
□ 无 4 级模糊匹配入口（memory-update 已关闭）
□ 所有记忆端点无条件三重鉴权
```

## 3.3 可靠性

```
□ 崩溃点 ①②③④⑤ 各注入一次，恢复后五方对账一致
□ 原子写失败注入（EPERM×3）→ 第 4 次成功
□ Tier-1 审计失败注入 → 操作被拒绝
□ Tier-2 审计失败注入 → 业务成功 + 补偿队列补齐
□ 队列满 → 429
□ drain 超时 → aborted(shutdown) 而非 recovery_blocked
```

## 3.4 锁

```
□ 双进程启动：B 等待，A ready 后 B exit(2)
□ A 在 S2 慢启动（>GRACE）但心跳正常 → B 不接管
□ kill -9 后重启可恢复
□ legacy 锁（纯 PID）可升级
```

## 3.5 Embedding

```
□ Ollama 停机时 append 成功且正文在
□ 手动 reembed 后 ready
□ 坏向量（NaN/零向量/错维度）被拒
□ 旧向量不覆盖新正文（INo-7）
```

---

# 4. M0 交付时必须同步给用户的知情项（逐条，不可省略）

```
① 【分享功能关闭】qq_memory_share_request 与 admin 的 share_approve/reject 返回 501。
   原因：M0 尚未改为 memoryId 授权，继续开放等于保留"可能批错记忆"的通道。

② 【记忆编辑入口关闭】/api/socialV2/memory-update 返回 501。
   原因：它是 V1 第 4 条 A/B 分叉路径（只改 B 不改 A），M1 才改为转发 MS.edit。

③ 【A/B 语义未完全统一】M0 期间 P 层与 S 层三字段双写，以 P 为准。
   P 缺失时降级读 S（兼容）。M1 迁移后 S 层三字段移除。

④ 【过期记录不主动删除】M0 只在读时过滤 expiresAt，A 层记录不会主动清理。
   磁盘会随时间缓慢增长（当前 0 条，年增长量 < 10^4 条 ≈ 数 MB）。

⑤ 【无图形化恢复】恢复需 CLI + 手工确认；M1 提供 dry-run UI。

⑥ 【换模型需手动重嵌入】M0 无自动批量；需逐个 reembed 或写脚本。

⑦ 【interactionCount 语义】M0/M1 暂存 P 层，P 全量重建时置 0（增量更新不受影响）。

⑧ 【clear 语义】M0 采用方案 B（V1 行为）：clear 不阻止清空后到达的写入。
   若需屏障语义（方案 A），需 D-21 批准后实施。
```

---

# 5. M0 不通过的条件（红线）

任一条成立即 **M0 不可上线**：

```
✗ 迁移对账不平
✗ 崩溃恢复后五方（store/P/S/WAL/T）对账不一致
✗ 已删除记录可通过任何读路径返回
✗ A 层存在未脱敏正文
✗ Tier-1 审计失败仍允许业务提交
✗ 人工 copy 备份可绕过 tombstone
✗ 双实例可同时运行
✗ embed 失败导致正文丢失
```

---

# 6. M0 的已知限制（不是缺陷，是范围）

| 限制 | 影响 | 缓解 |
|---|---|---|
| 无分享 | 群内记忆不能共享到其他群 | 群内记忆仍可用（同 key 检索） |
| 无 B 层编辑 | 不能在控制台改记忆文本 | 用 remove + append 代替 |
| 无过期清理 | A 层记录累积 | 读时过滤保证不召回；M1 补迁移 |
| 无 suppression | 只能删不能屏蔽 | M1 补 |
| P 全量重建丢 interactionCount | 重建后计数归零 | 审计记录；M1 增量维护 |
| 无模型隔离 | 换模型后旧向量不召回（降级） | 手动 reembed |

---

# 7. 验收签字模板

```
M0 验收人：__________  日期：__________

数据安全  □ 通过  □ 不通过  备注：____________
写入路径  □ 通过  □ 不通过  备注：____________
可靠性    □ 通过  □ 不通过  备注：____________
锁        □ 通过  □ 不通过  备注：____________
Embedding □ 通过  □ 不通过  备注：____________

知情项已阅读并确认：□ 是
已知限制已确认：    □ 是
待决策项（D-21/22/23/09/15/14b）已明确：□ 是

结论：□ 批准上线  □ 有条件批准（条件：______）  □ 不批准
```
