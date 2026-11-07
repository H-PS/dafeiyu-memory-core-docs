# Memory Core R4.1 — 核心状态机契约（State Machines）

> 文件：`contracts/state-machines/state-machines.md` ｜ 版本 1.0 ｜ 状态：APPROVED（结构依据 sf 后文 §4.3；DD-AUTHORITY R7 裁决：Outbox 事件与通用异步任务是两个独立状态机）
> 每个 状态机必须明确：初始状态 / 可进入状态 / 禁止转换 / 触发条件 / 执行主体 / 事务要求 / 失败处理 / 是否可逆 / 是否审计 / 测试用例。状态转换 MUST 有事件记录、操作者、原因、版本信息。

---

## SM-1 Memory Lifecycle（lifecycle_state）

初始状态：**candidate**

| # | From | To | 触发条件 | 执行主体 | 事务 | 可逆 | 审计 |
|---|---|---|---|---|---|---|---|
| 1 | candidate | pending_review | 候选通过质量检查（有来源/主体/Scope） | M03（产生）→ M04 审核 | M01 事务 | 否（记录 provenance） | 是 |
| 2 | pending_review | active | M04 审核通道确认（validation=confirmed） | **仅 M04**（台账 D-AP-05：confirmed 仅允许 M04 审核通道写入） | M01 事务 + provenance | 否 | 是 |
| 3 | pending_review | deleted | 审核拒绝或用户撤销 | M04 / 用户 | M01 事务 + tombstone | 否 | 是 |
| 4 | active | active | 实质内容更新（同 memory_id，version+1；或 supersedes 新对象） | M01（授权写入） | M01 事务，旧版本保留 | 否 | 是 |
| 5 | active | superseded | 被新版本取代 | M04 建议经 M01 执行 | M01 事务 + SUPERSEDES 关系 | 否 | 是 |
| 6 | active | disputed | 检测到冲突（attribute/temporal/subject/scope/source_conflict） | M04 | M01 事务 | 是（冲突解除回 active） | 是 |
| 7 | disputed | active | 冲突依 M04 规则解决 | M04 | M01 事务 | — | 是 |
| 8 | active | archived | 人工可控归档策略（自动归档未授权） | 用户 / 授权维护 | M01 事务 | 是（可恢复 active） | 是 |
| 9 | archived | active | 恢复 | 用户 | M01 事务 | — | 是 |
| 10 | active/archived | deleted | 删除请求（含 revoke） | M01 | M01 事务 + tombstone + Outbox delete | **否（删除传播后不可逆）** | 是 |

**禁止转换（MUST NOT）**：
- candidate/pending_review → active（跳过 M04 审核）；任何模块直接写 active 的 validation=confirmed（台账 D-AP-05）。
- deleted → 任何状态（只能经备份恢复流程 + 重放 tombstone；禁止旧 UPDATE 重试恢复已删除记忆）。
- 任何状态 → candidate（版本链不可回滚为候选）。
- 状态字段直改：MUST 通过受控操作 + provenance，禁止任意模块 UPDATE 状态字段（sf 前文 §2.4）。

**失败处理**：转换事务失败 ⇒ 回滚，状态不变；删除传播失败 ⇒ SQLite 权威状态已删除生效，索引残留由 Outbox 重试 + 检索侧权威状态排除兜底。

测试用例：TC-SM1-01 candidate→active 直跳必须被拒；TC-SM1-02 非 M04 通道写 confirmed 必须被拒；TC-SM1-03 删除后旧 UPDATE 重试不得复活；TC-SM1-04 转换均产生 provenance + 审计。

---

## SM-2 Validation State（validation_state）

初始状态：**unverified**

| From | To | 触发 | 主体 | 说明 |
|---|---|---|---|---|
| unverified | confirmed | M04 审核确认（证据充分、来源可信） | 仅 M04 | 台账 D-AP-05 |
| unverified | disputed | 冲突检测命中 | M04 | |
| unverified | rejected | 质量检查不过/用户否决 | M03/M04/用户 | |
| confirmed | disputed | 新反例证据 | M04 | |
| disputed | confirmed | 冲突解决（用户明确纠正或证据裁决） | M04/用户 | |
| disputed | rejected | 判定为错误 | M04 | |
| rejected | unverified | 用户重新提供证据（重新入审） | 用户 | 唯一"复活"路径，需 provenance |

禁止：rejected → confirmed 直跳；跳过 M04 的任何 confirmed 写入。**置信度不得替代本状态**（sf 后文 §2.3.4）。

---

## SM-3 Availability State（availability_state）

初始状态：**available**

| From | To | 触发 | 主体 |
|---|---|---|---|
| available | restricted | 权限收紧/敏感度上调（M05 判定） | M05 策略 → M01 执行 |
| available | suspended | 用户暂停/系统维护封禁 | 用户/系统 |
| restricted | available | 明确解除限制 | M05 策略 |
| restricted | suspended | 限制升级 | M05 |
| suspended | available/restricted | 恢复 | 用户/系统 |

**权限变更传播七步**（sf 前文 §8.5）必须联动：更新权威状态 → 写事件 → 缓存失效 → 更新/删除派生索引 → 更新 Observation/关系可用状态 → 审计 → 检查未完成任务。禁止只改下一次查询。

---

## SM-4 Outbox Event（status）

初始状态：**pending**（与权威写入同事务创建）

| # | From | To | 触发 | 主体 | 失败处理 |
|---|---|---|---|---|---|
| 1 | pending | processing | Worker 认领 | Outbox Worker | 崩溃 ⇒ 认领租约超时后回到 pending |
| 2 | processing | succeeded | FTS5+LanceDB 全部更新且一致性校验通过 | Worker | — |
| 3 | processing | retry_wait | 部分失败（partial_failure_policy=retry_failed_only）/ 全失败 | Worker | next_retry_at=now+backoff(retry_count) |
| 4 | retry_wait | processing | 到达重试时间 | Worker | — |
| 5 | retry_wait / processing | dead_letter | retry_count > retry_max | Worker | 告警 + 人工修复入口 |
| 6 | processing | succeeded（终态标记） | v_event < v_current ⇒ SkipOrSupersede | Worker | 不新增 status 枚举值；记录 last_error=superseded_by_version |
| 7 | dead_letter | processing | 人工修复后重置 | 人工 | 必须审计 |
| 8 | pending/processing | cancelled | 聚合对象已删除且无需传播（幂等短路） | Worker | — |

规则：幂等 Process(event_n)=Process(event_n)；同事件重复投递 ⇒ 幂等跳过；删除事件优先处理；FTS5 成功 LanceDB 失败 ⇒ 事件整体不得标 succeeded；索引同步失败不得回滚已提交权威数据。

---

## SM-5 Worker（通用异步任务，task）

初始状态：**pending**（DD-AUTHORITY R7：与 SM-4 分离）

| From | To | 触发 |
|---|---|---|
| pending | running | 调度执行 |
| running | succeeded | 完成 |
| running | failed | 失败（retry_count+1，可回 pending） |
| running/pending | cancelled | 显式取消 |
| failed | pending | 重试（≤上限） |

规则：**cancelled ≠ succeeded**（sf 后文 §4.6）；字段：task_id/task_type/status/created_at/started_at/completed_at/retry_count/error/idempotency_key。Worker MUST 支持重复投递、进程重启、处理中断、版本过期、部分索引失败、长时间未处理、失败重试、人工修复（八项）。

---

## SM-6 Migration（Schema/数据迁移，migration）

初始状态：**not_started**

| From | To | 触发 | 可逆 | 审计 |
|---|---|---|---|---|
| not_started | backing_up | 迁移申请批准 | 是 | 是 |
| backing_up | migrating | 备份 SHA256 校验通过 | 是（可放弃） | 是 |
| migrating | validating | 脚本执行完成 | 条件性（看迁移类型） | 是 |
| validating | completed | 验收检查全过 | 否（已生效） | 是 |
| validating | rolled_back | 验收失败 | — | 是 |
| migrating | failed | 执行错误 | ⇒ rolled_back | 是 |
| completed | rolled_back | 仅在迁移声明"可逆"且窗口内 | 条件性 | 是 |

规则：迁移六要素齐备才可执行；禁止直接改生产库结构不记迁移；completed 后旧结构按保留策略归档。

---

## SM-7 Deletion（删除传播，deletion）

初始状态：**requested**

| # | From | To | 触发 | 说明 |
|---|---|---|---|---|
| 1 | requested | tombstoned | SQLite 权威标记删除（事务） | 立即生效：检索侧权威排除 |
| 2 | tombstoned | index_deleting | Outbox DELETE 事件 | 删除事件优先处理 |
| 3 | index_deleting | fts_deleted | FTS5 删除完成 | 部分完成态 |
| 4 | fts_deleted | lancedb_deleted | LanceDB 删除完成 | |
| 5 | lancedb_deleted | derived_cleaned | 缓存失效 + Observation/Relation 处理 | 按 04 规范 |
| 6 | derived_cleaned | completed | 一致性校验（索引无残留）+ 审计 | 完成判据 |

规则：每层删除 MUST 验证"索引不存在"；任何一层失败 ⇒ 事件回 SM-4 retry_wait，但**权威删除状态不回退**；备份恢复后 MUST 重放 tombstone；备份保留策略不得违反已确认的数据清除要求（sf 前文 §17.2）。

---

## SM-8 Embedding Migration（B-011，embedding_migration）

初始状态：**not_started**

| # | From | To | 触发 | 可逆 | 审计 |
|---|---|---|---|---|---|
| 1 | not_started | backfilling | 新空间创建（新 embedding_space_id） | 是 | 是 |
| 2 | backfilling | validating | 回填覆盖率校验通过 | 是 | 是 |
| 3 | validating | dual_running | 检索对比验证通过 | 是 | 是 |
| 4 | dual_running | switching | 追赶完成（Pending Outbox=0 且版本一致） | 是 | 是 |
| 5 | switching | monitoring | 路由已切新空间 | 条件性 | 是 |
| 6 | monitoring | retired | 观察期通过 | 否（旧空间退役） | 是 |
| 7 | 任意（非 retired） | rolled_back | 新空间失败 | 是（旧空间未删） | 是 |
| 8 | 任意 | failed | 不可恢复错误 | ⇒ rolled_back | 是 |
| 9 | rolled_back | not_started | 重新发起迁移（须重新授权；复用同一 space id 时先清空） | 是 | 是 |

**四禁令（MUST NOT）**：同维度直接混合；未完成回填就切换；未验证新空间就删除旧空间；新模型失败时静默返回旧模型结果并伪装为新模型结果。**不同空间向量禁止混合比较**；查询 MUST 校验 embedding_space_id。

---

## 通用要求

- 每次转换 MUST 记录：操作者、原因、版本、时间（provenance/审计）。
- 状态机文档中的状态名与契约、09 图 MUST 一字不差。
- 非法转换的实现层 MUST 返回 VALIDATION_ERROR（或对应错误码），禁止静默纠正。
