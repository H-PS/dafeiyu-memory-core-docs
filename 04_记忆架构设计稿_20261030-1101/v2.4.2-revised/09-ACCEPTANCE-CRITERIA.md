# 09 · 验收标准（完整清单）

> 本清单是**实施阶段的验收基线**，本阶段未执行任何测试。
> 运行环境需专用测试夹具（`state/` 重定向到临时目录），**禁止对生产 state/ 执行**。
> 标注：`[M0]` = M0 必须通过；`[M1]` = M1 通过即可。

---

# 1. 写并发（对应 02 §7）

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-1.1 | edit_text ∥ remove（r 先提交） | 后到者 410/409；磁盘无 M；WAL 两条均有终态；**M 未被错误改写** | `[M0]` |
| AC-1.2 | edit_text ∥ clear | clear 提交后 edit 到达 → 410；**0 条复活** | `[M0]` |
| AC-1.3 | edit_text ∥ append | 两者都成功；store 含 M(改)+W(新) | `[M0]` |
| AC-1.4 | edit_text ∥ edit_text | 先者成功；后者 409 + `currentContentRev` | `[M0]` |
| AC-1.5 | append ∥ clear（两种排列） | 语义符合 D-21 决策；无"悄悄残留/复活" | `[M0]`（按方案 B） |
| AC-1.6 | share ∥ edit_text | SR=superseded，无副本（M1 才有分享） | `[M1]` |
| AC-1.7 | opId 幂等 | 同 opId 提交 3 次 → A 层条数不变，返回首次结果 | `[M0]` |
| AC-1.8 | append 去重 | 已存在同文本 → 1 条，`rev=2`，**`contentRev` 不变**，响应 `deduplicated:true` | `[M0]` |
| AC-1.9 | 原子写注入失败 | mock renameSync EPERM×3 → 第 4 次成功；无 `.tmp` 残留 | `[M0]` |
| AC-1.10 | 崩溃点 ①②③④⑤ | 各点注入终止后重启 → 终态与正常提交一致（store/P/S/WAL/T 五方对账） | `[M0]` |
| AC-1.11 | 旧 contentRev 重放 | store contentRev=3 committed → 重放 contentRev=2 pending → 保持 3 | `[M0]` |
| AC-1.12 | 队列满 | 64 个待执行 op 时第 65 个 → 429 `QUEUE_FULL` | `[M0]` |

---

# 2. 三版本与 extra（本版新增）

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-2.1 | edit_meta 后 edit_text | edit_meta 只 +rev；随后 edit_text 用相同 expectedContentRev → **成功**（不被元数据变更阻断） | `[M0]` |
| AC-2.2 | embedding_refresh ∥ edit_text | refresh 只 +rev；edit 的 expectedContentRev 仍匹配 → 成功 | `[M0]` |
| AC-2.3 | 旧向量迟到 | edit 提交 contentRev=3 后，forContentRev=2 的 refresh 到达 → **丢弃**（vecContentRev 不变） | `[M0]` |
| AC-2.4 | shareCopies 变更不影响同群 pending SR | 批准群 A（原件 rev+1、contentRev 不变）→ 群 B 的 pending SR 仍可审批 | `[M1]` |
| AC-2.5 | contentRev 真变则 SR 失效 | edit_text 后审批 → SR=superseded + 409 | `[M1]` |
| AC-2.6 | 版本字段一致性 | 任一记录满足 `contentRev ≤ rev` 且 `vecContentRev ≤ contentRev` | `[M0]` |

## 2.7 本版追加（三版本 / P 层版本 / extra）

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-2.7 | **edit_meta ∥ edit_meta** | 两个都基于 metaRev=3 → 先者成功（metaRev→4），后者 **409 + currentMetaRev**；A 的修改不被覆盖 | `[M0]` |
| AC-2.8 | **edit_meta ∥ edit_text** | 各用各的版本 → **两者都成功**（互不干扰） | `[M0]` |
| AC-2.9 | edit_meta 的 patch 语义 | 只传 participants → pendingQuestion 保留；传 `[]` → 显式清空 | `[M0]` |
| AC-2.10 | **extra category 校验** | activeTopic 传 motivation → **400 `EXTRA_NOT_ALLOWED`**；memberImpression 传 participants → 400 | `[M0]` |
| AC-2.11 | **extra 长度统一** | participants > 10 项 → 截 10；每项 > 32 字 → 截 32；pendingQuestion > 200 → 截 200；motivation > 50 → 截 50 | `[M0]` |
| AC-2.12 | **P 层 per-key 版本** | 写群 A → 只有群 A 的 convVersion+1；读群 B 不触发重建 | `[M0]` |
| AC-2.13 | **interactionCount 归 S** | 删除 P 文件后全量重建 → traits 恢复，`interactionCount` **仍保留**（来自 S） | `[M0]` |
| AC-2.14 | **traits 裁剪不删 A** | 同 subject 15 条 A 记录 → P 只显示 10 条，但 `qq_memory_list` 能看到 15 条 | `[M0]` |
| AC-2.15 | **M0 双写 effect 等级** | A/P/S-stats 为 required；S-compat 失败 → committed + warn | `[M0]` |
| AC-2.16 | **P 写失败自愈** | A 成功、P 失败 → 下次读 `isStale` 触发该会话重建 → 与 A 一致 | `[M0]` |
| AC-2.17 | **stats 写失败** | A/P 成功、S-stats 失败 → partial（不静默）；重试补齐 | `[M0]` |

---

# 3. A/P/S 一致性与投影（本版重写）

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-3.1 | 写入含会话令牌的 content | A/P/WAL/审计四方 grep 不到令牌明文；`contentHash` 一致 | `[M0]` |
| AC-3.2 | content >200 字 | P.text === A.text 前 200 字；A.text ≤1000；remove 用同一 content 可删 A | `[M0]` |
| AC-3.3 | remove 构造 A 少删 1 条 | 响应 `partial:true` + 残留明细 | `[M0]` |
| AC-3.4 | `group:0123` 写入 | store 与 P 均 `group:123` | `[M0]` |
| AC-3.5 | P 层损坏 | 会话重建后 P 与 A 按 memoryId+contentRev 对齐 | `[M0]` |
| AC-3.6 | **S 层 39 处写入不被 P 影响** | 并发执行 wake/unread 写入 + 记忆写入 → S 层的 recentMessages/unread 完整无损，P 层三字段正确 | `[M0]` |
| AC-3.7 | **P 层重建保留 interactionCount** | 删除 P 文件后触发会话重建 → traits 正确；`interactionCount` 从旧 P 合并（若存在），否则置 0 并审计 | `[M0]` |
| AC-3.8 | **memberImpressions 1:N 删除** | 同一 subject 有 3 条 A 记录 → 删其中 1 条 → P 层 traits 只剩 2 条，且被删的是正确那条 | `[M0]` |
| AC-3.9 | **memberImpressions 1:N 编辑** | 改 1 条 A 记录 text → P 层对应 trait.text 更新，其余不变 | `[M0]` |
| AC-3.10 | traits 上限统一为 10 | append 与 update 两条路径都截断到 10（修 N-02 的 10/20 不一致） | `[M0]` |
| AC-3.11 | A/P 不互相触发 | P-effect 内不出现 enqueue 调用（代码审查 + 单测） | `[M0]` |
| AC-3.12 | `memory-update` 端点处置 | M0：返回 501（关闭）；M1：转发 `MS.edit_meta`（N-01） | `[M0]` 501 / `[M1]` 转发 |

---

# 4. WAL 与恢复

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-4.1 | 崩溃于 WAL(pending) 后 | op 标 `aborted(crashed_before_apply)`，无副作用 | `[M0]` |
| AC-4.2 | 崩溃于 T 后 store 前 | 重放收敛：记录删除 + 墓碑 + P 同步 | `[M0]` |
| AC-4.3 | 崩溃于 store 后 P 前 | 重放收敛：P 补齐 | `[M0]` |
| AC-4.4 | 恢复中 IO 持续失败 | `recovery_blocked` + 冻结集；`/api/status` 可见；resubmit 后收敛或 superseded | `[M0]` |
| AC-4.5 | 双 op 同 target 高低 contentRev | 低 contentRev pending 被高 contentRev committed 遮蔽为 superseded | `[M0]` |
| AC-4.6 | 恢复扫描按 requestSeq 升序 | 乱序 requestSeq 的 pending op 按请求顺序重放 | `[M0]` |

---

# 5. 删除与防复活

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-5.1 | 删除后 6 类读路径 | search/list/P/审批/恢复导入/缓存均不返回该 id | `[M0]` |
| AC-5.2 | 删除级联（默认） | 2 个已批准副本均 tombstone（`originCascade=true`）；SR=revoked | `[M1]` |
| AC-5.3 | 不级联路径 | 未批准 D-15 前**不提供**；调用 `cascade:false` → 400 `NOT_SUPPORTED` | `[M0]` |
| AC-5.4 | 备份恢复（含已删记录） | 跳过 tombstoned；导入后无复活 | `[M0]` |
| AC-5.5 | ignoreTombstone 白名单 | 未批准 D-09 前**不实现**；请求 → 400 `NOT_SUPPORTED` | `[M0]` |
| AC-5.6 | suppression(text, 保留正文) | 屏蔽后不可检索/分享；解除后恢复（D-14b=保留时） | `[M1]` |
| AC-5.7 | 删除遇冻结 id | 503 `recovery_pending`，不叠加删除 | `[M0]` |
| AC-5.8 | **suppression(scope=text) 使 contentRev+1** | 该记录的 pending SR 失效（正文变了） | `[M1]` |

---

# 6. 分享安全

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-6.1 | M0 期间 share_request | 501 `SHARE_DISABLED_IN_M0` | `[M0]` |
| AC-6.2 | 错误 memoryId 申请 | 404 | `[M1]` |
| AC-6.3 | 审批时原件 contentRev 已变 | SR=superseded + 提示当前 contentRev | `[M1]` |
| AC-6.4 | 审批时原件已删 | SR=superseded + 410 | `[M1]` |
| AC-6.5 | 群 A 批准 / 群 B 拒绝 | 两 SR 独立；A 可见 B 不可见 | `[M1]` |
| AC-6.6 | **多群依次审批** | 批准 A → 群 B 的 pending SR 仍成功（contentRev 未变） | `[M1]` |
| AC-6.7 | 副本追溯 | `copy.origin.id` + `origin.contentRev` 反查原件 | `[M1]` |
| AC-6.8 | private 源申请 | 400（与 V1 `bridge.js:6892` 一致） | `[M1]` |
| AC-6.9 | 副本向量（原件 ready 同模型同 contentRev） | 直接复用，无嵌入请求（计数验证） | `[M1]` |
| AC-6.10 | 副本向量（原件 vecContentRev < contentRev） | **不复用**，副本 `pending` | `[M1]` |
| AC-6.11 | 撤销后 | 目标群 search 不返回该副本；SR=revoked | `[M1]` |
| AC-6.12 | 副本再分享 | 400 `COPY_CANNOT_RESHARE` | `[M1]` |

---

# 7. Embedding

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-7.1 | Ollama 停机时 append | 返回 ok + `embedding:'pending'`；**正文已持久化**（重启后仍在） | `[M0]` |
| AC-7.2 | 手动 reembed | `POST /api/memory/reembed` → `ready` | `[M0]` |
| AC-7.3 | 超过重试上限（M1） | `failed` 终态；正文仍在；降级检索可召回 | `[M1]` |
| AC-7.4 | 坏向量注入（NaN/零向量/长度≠dim） | 判 failed，不入库 | `[M0]` |
| AC-7.5 | 模型切换（M1） | 旧 vecModel 记录不再语义召回，降级命中 + `degraded:true` | `[M1]` |
| AC-7.6 | qq_memory_query（Ollama 停机） | P 层结果返回 + `vector.status='unavailable'`（M0）/ `degraded`（M1） | `[M0]` |
| AC-7.7 | 运行期改 topK/threshold | 下次检索生效（M0 需重启） | `[M1]` |
| AC-7.8 | vecContentRev 边界 | `vecContentRev === contentRev` 时 embedding_refresh 不再写入（已最新） | `[M0]` |

---

# 8. 审计

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-8.1 | Tier-1 写失败（remove/clear） | 操作**被拒绝**；WAL=aborted；业务未发生 | `[M0]` |
| AC-8.2 | Tier-2 写失败（append/edit） | 业务成功；补偿队列最终补齐 | `[M0]` |
| AC-8.3 | clear / memory-remove 审计 | 均产生完整事件（对比 V1 零审计） | `[M0]` |
| AC-8.4 | 审计 grep 正文 | 0 命中；contentHash 存在 | `[M0]` |
| AC-8.5 | 审计文件损坏 | 坏行入 quarantine + 告警 | `[M0]` |
| AC-8.6 | 轮转 | 单文件 >10MB 触发；保留 10 份 | `[M0]` |
| AC-8.7 | **审计含 contentRevBefore/After** | 事件完整记录内容版本变化 | `[M0]` |

---

# 9. 锁与启动

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-9.1 | B 在 A 的 GRACE 窗口内启动 | B 等待重试，**不删锁**；A ready 后 B exit(2) | `[M0]` |
| AC-9.2 | **A 在 S2 recovery 耗时 120s（>GRACE）但心跳正常** | B 必须等待，**不得接管** | `[M0]` |
| AC-9.3 | A 卡死（心跳停止）超 GRACE、端口空闲 | B 接管；A 恢复后 listen EADDRINUSE → exit(2) 且不误删 B 的锁 | `[M0]` |
| AC-9.4 | kill -9 于 S0~S5 | 重启 stale 判定符合 07 §4 表 | `[M0]` |
| AC-9.5 | 锁文件损坏/legacy | 可恢复启动 | `[M0]` |
| AC-9.6 | 退出 drain 超时 | 在途 P3 op 标 recovery_blocked；未执行的排队 op 标 **aborted(shutdown)** | `[M0]` |
| AC-9.7 | 外部工具不取锁直写 store | bridge 侦测 EXTERNAL_WRITE + 重载 + 审计告警 | `[M0]` |

---

# 10. 备份与恢复

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-10.1 | 备份文件篡改 1 字节 | digest 校验失败，拒绝加载 | `[M0]` |
| AC-10.2 | 备份失败注入（磁盘满） | 告警 + Tier-2 审计；**删除前备份失败 → 阻断操作** | `[M0]` |
| AC-10.3 | dry-run 报告与执行一致 | willImport/willSkip/needReembed 计数对账 | `[M1]` |
| AC-10.4 | 恢复失败回滚 | pre_restore 还原，store 与恢复前逐条一致 | `[M1]` |
| AC-10.5 | 连续 2 次 remove 前备份 | 2 个不同时间戳文件 | `[M0]` |
| AC-10.6 | **M0 CLI 恢复校验** | 含已删记录的备份执行 `--apply` → 已删 id 不出现；报告列出跳过数 | `[M0]` |

---

# 11. 迁移

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-11.1 | 101 条含 1 坏行 + 2 重复 | 报告 `{input:101, badLines:1, duplicates:2→1, output:99}`；对账通过 | `[M0]` |
| AC-11.2 | 对账失败注入 | 整体回退，原 `memory-vectors.jsonl` 字节级未变 | `[M0]` |
| AC-11.3 | 迁移后 | 旧文件为 `.v1.bak`（只读）；tombstone 为空；V1 历史备份未被触碰 | `[M0]` |
| AC-11.4 | 含令牌的旧正文 | 迁移后 store 无令牌明文（重规范化生效） | `[M0]` |
| AC-11.5 | **P 层生成** | 迁移后 `memory-projection.json` 存在且与 store 按 memoryId 对齐 | `[M0]` |
| AC-11.6 | S 层 orphan | `social-v2-orphans.json` 含未匹配条目，A/P/S 均无静默变化 | `[M0]` |
| AC-11.7 | V1 approved 分享记录 | 导出 `v1-approved-shared.json`，不自动生成副本 | `[M0]` |

---

# 12. 端点与鉴权

| ID | 场景 | 期望 | 里程碑 |
|---|---|---|---|
| AC-12.1 | 记忆端点无 x-agent-token 头 | 必须持有效 consoleToken，否则 401 | `[M0]` |
| AC-12.2 | 带 x-agent-token 但 key 不匹配会话 | 403 | `[M0]` |
| AC-12.3 | `/api/memory/admin` 带 x-agent-token | 403（沿用 V1 6707） | `[M0]` |

---

# 13. 全局回归红线（任一失败即版本不通过）

1. **已删除记录不得出现在任何读取路径**（search/list/P/审批/恢复/缓存）。
2. **不得出现"业务成功 + 返回失败"**（Tier-1 审计阻断除外，且此时业务确实未发生）。
3. **不得出现"业务失败 + 实际已写盘"**（P3 单点生效）。
4. 任何崩溃点恢复后，**store/P/S/WAL/tombstone 五方对账一致**。
5. 任何写入路径**不包含未脱敏正文**（A/P/WAL/审计/日志）。
6. 进程内任意时刻**至多一个写操作在临界区**（INo-1）。
7. **`embedding_refresh` 不得覆盖更高 contentRev 的正文**（INo-7）。

---

# 14. 统计

| 里程碑 | 条目数 |
|---|---|
| `[M0]` | 73 |
| `[M1]` | 22 |
| 合计 | **95** |

（原 71 条 + 第二版新增 13 条 + 本次追加 11 条：三版本 3 + extra 2 + P 层版本 1 + interactionCount 1 + traits 裁剪 1 + 双写 effect 3）
