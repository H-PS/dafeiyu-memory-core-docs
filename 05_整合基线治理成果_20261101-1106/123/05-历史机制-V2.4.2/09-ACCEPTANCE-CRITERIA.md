# 09 · 验收标准与测试场景（ACCEPTANCE CRITERIA）

> 本文件是**未来实施阶段的验收清单**，本阶段未执行任何测试。
> 每个场景标注：验证的缺陷/不变量、前置、步骤、期望。运行环境需专用测试夹具（`state/` 重定向到临时目录），**禁止对生产 state/ 执行**。

## 1. 写并发（对应 02，验 F-01/F-06/F-10/F-15/F-16）

| ID | 场景 | 前置 | 步骤 | 期望 |
|---|---|---|---|---|
| AC-1.1 | edit ∥ remove（r 先提交） | 记录 M rev=2 | 并发提交 edit(M,eRev=2) 与 remove(M,eRev=2) | 后到者得 410/409；磁盘无 M；WAL 两条均有终态；**M 未被错误改写**（对比 R2） |
| AC-1.2 | edit ∥ clear | key 含 3 条 | clear 提交后 edit 到达 | edit 得 410；**0 条复活**（对比 R1） |
| AC-1.3 | append ∥ edit | 记录 M + 新文本 W | 并发 append(W) 与 edit(M) | 两者都成功；store 含 M(改)+W(新)（对比 R3） |
| AC-1.4 | edit ∥ edit | M rev=2 | 两个 edit(eRev=2) 并发 | 先者成功；后者 409 + currentRev=3（对比 R4） |
| AC-1.5 | append ∥ clear | — | 两种排列 | 两种排列语义明确（02 §5.5 表），无「悄悄残留/复活」 |
| AC-1.6 | share ∥ edit | SR pending(eRev=2) | edit 提交后审批 | SR=superseded，无副本生成（对比 05 §3 ④） |
| AC-1.7 | opId 幂等 | — | 同 opId 提交 3 次 | A 层条数不变，返回首次结果 |
| AC-1.8 | append 去重 | 已存在同文本 | 再次 append | 1 条，rev=2，响应含 `deduplicated:true` |
| AC-1.9 | 原子写注入失败 | — | mock renameSync EPERM×3 | 第 4 次重试成功；无 `.tmp` 残留 |
| AC-1.10 | 崩溃点 ①②③④ | — | 各点注入终止后重启 | 终态与正常提交一致（对账脚本：store/B/WAL/T 四方一致） |
| AC-1.11 | 旧 rev 重放 | store rev=3 committed | 重放 rev=2 pending effect | store 保持 rev=3（INo-4） |

## 2. A/B 一致性（对应 03，验 F-02/F-03/F-04/F-23）

| ID | 场景 | 期望 |
|---|---|---|
| AC-2.1 | 写入含会话令牌的 content | A/B/WAL/审计四方 grep 不到令牌明文；`contentHash` 一致 |
| AC-2.2 | content >200 字 | B.text === A.text 前 200 字；A.text ≤1000；remove 用同一 content 可删 A（对比 V1 失配） |
| AC-2.3 | remove 构造 A 少删 1 条 | 响应 `partial:true` + 残留明细（对比 F-03） |
| AC-2.4 | pendingThought 到期 | ≤ 扫描周期 + 读时判定双保险：到期后 search/list/B 三处均不返回（对比 F-04） |
| AC-2.5 | 21 条 activeTopic | 第 1 条 `lifecycle='evicted'`；B 同步移除；再提及时可复活 |
| AC-2.6 | `group:0123` 写入 | store 与 B 均 `group:123`（对比 F-23） |
| AC-2.7 | B 投影损坏 | `POST /api/socialV2/rebuild-b` 后 B 与 A 按 memoryId+rev 对齐 |
| AC-2.8 | memberImpression 按 target 删除 | A 层精准命中（对比 V1 `bridge.js:6932` 只删 B） |

## 3. WAL 与恢复（对应 03 §5）

| ID | 场景 | 期望 |
|---|---|---|
| AC-3.1 | 崩溃于 WAL(pending) 后 | op 标 aborted(reason=crashed_before_apply)，无副作用 |
| AC-3.2 | 崩溃于 tombstone 后 store 前 | 重放收敛：记录删除 + 墓碑 + B 同步 |
| AC-3.3 | 崩溃于 store 后 B 前 | 重放收敛：B 补齐 |
| AC-3.4 | 恢复中 IO 持续失败 | `recovery_blocked` + 冻结集；`/api/status` 可见；resubmit 后收敛或 superseded |
| AC-3.5 | 双 op 同 target 高低 rev | 低 rev pending 被高 rev committed 遮蔽为 superseded |

## 4. 删除与防复活（对应 04，验 F-11）

| ID | 场景 | 期望 |
|---|---|---|
| AC-4.1 | 删除后 6 类读路径 | search/list/B/审批/恢复导入/缓存均不返回该 id |
| AC-4.2 | 删除级联（默认） | 2 个已批准副本均 tombstone（originCascade=true）；SR=revoked |
| AC-4.3 | 不级联路径 | 必须二次确认 + 警告文案完整（04 §5.2 逐字核对） |
| AC-4.4 | 备份恢复（含已删记录） | willSkip(tombstoned) 正确；导入后无复活 |
| AC-4.5 | ignoreTombstone 白名单 | 逐条 RESURRECT 审计；默认路径无此能力 |
| AC-4.6 | suppression(text, 保留正文) | 屏蔽后不可检索/分享；解除后恢复（D-14b=保留时） |
| AC-4.7 | 删除遇冻结 id | 503 recovery_pending，不叠加删除 |

## 5. 分享安全（对应 05，验 F-09）

| ID | 场景 | 期望 |
|---|---|---|
| AC-5.1 | 错误 memoryId 申请 | 404 |
| AC-5.2 | 审批时原件已编辑 | SR=superseded + 提示当前 rev |
| AC-5.3 | 审批时原件已删 | SR=superseded + 410 |
| AC-5.4 | 群 A 批准 / 群 B 拒绝 | 两 SR 独立；A 可见 B 不可见 |
| AC-5.5 | 副本追溯 | copy.origin.id 可反查原件或其墓碑 |
| AC-5.6 | private 源申请 | 400（与 V1 `bridge.js:6892` 一致） |
| AC-5.7 | 副本向量（原件 ready 同模型） | 直接复用，无嵌入请求（计数验证） |
| AC-5.8 | 撤销后 | 目标群 search 不返回该副本；SR=revoked |

## 6. Embedding（对应 06 §1，验 F-08/F-14/F-53/F-25）

| ID | 场景 | 期望 |
|---|---|---|
| AC-6.1 | Ollama 停机时 append | 返回 ok + `embedding:'pending'`；**正文已持久化**（重启后仍在）（对比 F-08） |
| AC-6.2 | Ollama 恢复 | ≤ 重试周期内转 ready |
| AC-6.3 | 超过 5 次重试 | `failed` 终态；正文仍在；降级检索可召回（D-04 开时） |
| AC-6.4 | 坏向量注入（NaN/零向量/长度≠dim） | 嵌入判 failed，不入库 |
| AC-6.5 | 模型切换 | 旧 vecModel 记录不再被语义召回，降级命中 + `degraded:true` |
| AC-6.6 | qq_memory_query（Ollama 停机） | B 层结果返回 + `vector.status='degraded'`（对比 F-22） |
| AC-6.7 | 运行期改 topK/threshold | 下次检索生效（对比 F-25） |

## 7. 审计（对应 06 §2，验 F-05/F-17）

| ID | 场景 | 期望 |
|---|---|---|
| AC-7.1 | Tier-1 写失败注入（remove/clear/approve/suppress/restore） | 操作**被拒绝**；WAL=aborted；业务未发生 |
| AC-7.2 | Tier-2 写失败注入（append/edit） | 业务成功；补偿队列最终补齐事件 |
| AC-7.3 | clear / memory-remove 审计 | 均产生完整审计事件（对比 V1 零审计） |
| AC-7.4 | 审计文件 grep 正文 | 0 命中；contentHash 存在（对比 F-17） |
| AC-7.5 | 审计文件损坏 | 坏行入 quarantine + 告警，不静默丢 |
| AC-7.6 | 轮转 | 单文件 >10MB 触发；保留 10 份 |

## 8. 锁与启动（对应 07，验 F-07）

| ID | 场景 | 期望 |
|---|---|---|
| AC-8.1 | B 在 A 的 GRACE 窗口内启动 | B 等待重试，**不删锁**；A ready 后 B exit(2)（对比 V1 终态「无锁运行」） |
| AC-8.2 | A 卡死于 recovering 超 GRACE、端口空闲 | B 接管；A 恢复后 listen EADDRINUSE → exit(2) 且不误删 B 的锁 |
| AC-8.3 | kill -9 于 S0~S5 各态 | 重启 stale 判定符合 07 §4 表 |
| AC-8.4 | 锁文件损坏/legacy（纯 PID） | 可恢复启动 |
| AC-8.5 | 退出 drain 超时 | 在途 op 标 recovery_blocked，不丢失 |
| AC-8.6 | 外部工具不取锁直写 store | bridge 侦测 EXTERNAL_WRITE + 重载 + 审计告警 |

## 9. 备份与恢复（对应 06 §3~4，验 F-24/F-26）

| ID | 场景 | 期望 |
|---|---|---|
| AC-9.1 | 备份文件篡改 1 字节 | digest 校验失败，拒绝加载 |
| AC-9.2 | 备份失败注入（磁盘满） | 告警 + Tier-2 审计；**删除/恢复前备份失败 → 阻断操作** |
| AC-9.3 | dry-run 报告与执行一致 | willImport/willSkip/needReembed 计数对账 |
| AC-9.4 | 恢复失败回滚 | pre_restore 还原，store 与恢复前逐条一致 |
| AC-9.5 | 连续 2 次 remove 前备份 | 2 个不同时间戳文件（对比 F-26） |

## 10. 迁移（对应 03 §7）

| ID | 场景 | 期望 |
|---|---|---|
| AC-10.1 | V1 101 条含 1 坏行 + 2 重复 | 报告 {input:101, badLines:1, duplicates:2→1, output:99}；对账通过 |
| AC-10.2 | 对账失败注入 | 整体回退，原 `memory-vectors.jsonl` 未被修改（字节级比对） |
| AC-10.3 | 迁移后 | 旧文件存在为 `.v1.bak`（只读）；tombstone 为空；V1 历史备份未被触碰 |
| AC-10.4 | 含令牌的旧正文 | 迁移后 store 无令牌明文（重规范化生效） |
| AC-10.5 | B orphan | `social-v2-orphans.json` 含未匹配条目，A/B 均无静默变化 |

## 11. 端点与鉴权（验 F-20）

| ID | 场景 | 期望 |
|---|---|---|
| AC-11.1 | 记忆端点无 x-agent-token 头 | 必须持有效 consoleToken，否则 401 |
| AC-11.2 | 带 x-agent-token 但 key 不匹配会话 | 403 |
| AC-11.3 | `/api/memory/admin` 带 x-agent-token | 403（沿用 V1 6707） |

---

## 12. 全局回归红线（任一失败即版本不通过）

1. 任何场景下，**已删除记录不得出现在任何读取路径**。
2. 任何场景下，**不得出现「业务成功 + 返回失败」**（Tier-1 审计阻断除外，且此时业务确实未发生）。
3. 任何崩溃点恢复后，store/B/WAL/tombstone **四方对账一致**。
4. 任何写入路径**不包含未脱敏正文**（A/B/WAL/审计/日志）。
5. 进程内任意时刻**至多一个写操作在临界区**（INo-1）。
