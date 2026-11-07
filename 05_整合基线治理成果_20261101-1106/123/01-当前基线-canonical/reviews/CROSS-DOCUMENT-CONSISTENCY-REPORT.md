# 跨文档一致性检查报告（CROSS-DOCUMENT-CONSISTENCY-REPORT）

> 文件：`reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md` ｜ 检查者：组 C 一致性检查 Agent ｜ 日期：2026-11-04
> 基准：`DD-AUTHORITY.md` v1.0（§4 R1–R10 裁决、§5 硬约束、§6 强制条款）+ `08-adjudication-ledger.md`（唯一权威）+ `contracts/**` 机器契约
> 范围：00–08 九份文档、`contracts/error-codes.md`、`contracts/api/api-contracts.md`、`contracts/state-machines/state-machines.md`、`contracts/schemas/*.json`（12）、`contracts/config/*.json`（7）、`diagrams/diagrams.md`、`test-specs/test-specs.md`、`DESIGN-DECISION-REGISTER.md`、`OPEN-ISSUES.md`、`README.md`、`DESIGN-STATUS.md`、`CHANGELOG.md`
> 权限：本次只修改 `Memory-Core-R4.1-Design\` 目录内文件；未触碰 `_sources/`、`08-adjudication-ledger.md` 及目录外任何文件。
> 结果用语：**PASS**（一致）/ **FIXED**（已按基准修复）/ **CONFLICT**（语义冲突，登记待裁决，未单方面改写）

---

## 0. 结论摘要

| 指标 | 值 |
|---|---|
| 检查项 | 10 项（sf §七 清单 1–10） |
| 逐项结论 | PASS 4 项、FIXED 19 项子检查、CONFLICT 9 项 |
| **修复处数** | **45 处**（按修改位置计，涉及 14 个文件） |
| **遗留 CONFLICT** | **9 项**（登记 OI-016…OI-024，均需用户裁决） |
| 一致性是否达成 | **数值/名称/字段级一致性：达成**；**语义级一致性：未完全达成**（9 项 CONFLICT 待裁决，其中 OI-016/OI-017/OI-018 为接口契约级，阻塞"契约冻结"；不阻塞 M01/M02 设计继续细化） |

结论口径：本次检查把"可由基准唯一判定"的差异全部就地修复；把"两种设计各自成立、无法由基准唯一判定"的差异保留为 CONFLICT 并登记，**未做任何单方面语义改写**（遵守修复规则第 2 条）。

---

## 1. 逐项检查结果

### C-01 Schema 一致性（Memory Object 字段 / Scope / 三状态 / 时间六字段 / ID / 版本 / Evidence / Provenance）

| 检查项 | 涉及文档 | 结果 | 说明 / 修复位置 |
|---|---|---|---|
| Memory Object 29 字段与 `memory-object.schema.json` | 01 §7.6、§8.1、§9.1；schema | **CONFLICT（OI-020）** | 01 定义并强制 `scope_id`（§7.5/§7.6.1/§9.1 ER 图），schema `additionalProperties:false` 且无该字段 → A01-01"diff=0"不成立。已登记 OI-020（并入既有 OI-01-01 建议） |
| 其余 29 字段名/类型/必填/枚举 | 01 §7.6.1–§7.6.5；schema | PASS | 逐列对照一致（含 `subtype/normalized_content/expires_at/supersedes/embedding_space_id`） |
| 三状态枚举与交叉约束 | 01 §7.6.3/§7.7；schema `allOf` | PASS | `active ⇒ confirmed|disputed`；`candidate|pending_review ⇒ unverified` 一致；`confirmed` 仅 M04 |
| 时间六字段（R6） | 00 §7.8、01 §7.14、05 §7.9；schema | PASS | 六字段语义、UTC RFC 3339、`observed_at ≠ recorded_at` 一致；05 日志时间引用 R6 |
| ID / 版本语义（R5） | 00 §7.8、01 §7.6.1、schema | PASS | `format: uuid` + 正则、版本不锁；`version` 整数 ≥1 单调递增，无模块私改 |
| Search Document 字段（抽查） | 01 §7.13.1、§8.1；`search-document.schema.json` | **FIXED（3 处）** | ①`document_id` 类型 uuid→string ≤256（schema 为 string，建议 `memory_id:version:space`）；②补 `cue`、`document_hash`、`index_versions` 三行（schema 有、正文无）；③A01-09 "18 字段"→"21 字段"；④§8.1 required 去除 schema 非必填的 `sensitivity`、补 `document_hash` |
| Outbox 事件字段（抽查） | 03 §7.6.4、§8；`outbox-event.schema.json` | **FIXED（2 处）** | ①`payload_version` int→string（schema 为 string ≤32）；②`next_retry_at` 由"兼容别名"改为正式字段，并补 `document_hash`/`embedding_profile_id` → 10 字段更正为 13 字段 |
| Embedding Space 八字段（抽查） | 01 §7.13.2、07 §7.5/§8.2；`embedding-space.schema.json` | **FIXED（3 处）** | `distance_metric` 的 `ip` → `dot_product`（保留 `ip` 为历史别名），01 §7.13.2 / 07 §7.5 / 07 §8.2 三处同步；八字段名/必填一致（**status 语义见 OI-024**） |
| Evidence 字段与来源类型 | 01 §7.8；`evidence.schema.json` | PASS | 21 字段一致；六规范来源类型 + 6 别名映射登记一致 |
| 契约文件存在性表述 | 00 §4/§8.2、01 §1/§4/§8.1、02 §8.8 | **FIXED（6 处）** | 三份 Schema 早已落盘但四处仍写"待补（已有 5 个）"：00 D7 5→12 并补全清单、00 §8.2 三行→已存在、01 §1 行、01 D7、01 §8.1 三行、02 §8.8 三处（retrieval-request/context-response/permission-context）→已存在 |

### C-02 接口一致性（M01/M02 字段、M02↔M05 权限上下文、Outbox 版本语义、Worker 事件、Context Adapter 主体标识、错误码统一）

| 检查项 | 涉及文档 | 结果 | 说明 / 修复位置 |
|---|---|---|---|
| M01 七函数与 03 §7.4/00 §8.1 | 00、03、api-contracts §2 | PASS | 名称/事务/乐观并发一致 |
| M02 检索请求字段 | 02 §7.3 ↔ 08 §8.2 ↔ `retrieval-request.schema.json` | **CONFLICT（OI-016）** | 三套命名并存，未单方面改写 |
| Context Adapter 响应结构 | 03 §7.3.4 / 08 §7.7 / api-contracts §7（八字段）↔ `context-response.schema.json` | **CONFLICT（OI-017）** | 八字段与 schema required 完全不同 |
| 检索/权限状态枚举 | 02 / 06 / 08 / 两份 response schema | **CONFLICT（OI-018）** | `complete/partial/failed`、`success/partial/failed`、`ok/ok_empty/unavailable/degraded`、`allow/deny/error/timeout/not_checked` 四套 |
| M02↔M05 权限上下文 | 02 §7.10、04 §7.7、`permission-context.schema.json` | **FIXED（1 处）/ 部分 PASS** | 主体枚举不一致：schema 缺 `bot` 而 04 §7.2 明确"六类 + bot"→ schema `actor.type` 补 `bot`；03 §7.3.2 补 `owner`（改为七类）。**字段名差异**（02 用 `auth_source/auth_expiry`，schema 用 `authorized_by/authorization_expires_at`）归入 OI-016 同类问题，未单方面改 |
| 权限判定三态命名 | 03 §7.5/§9.4 ↔ api-contracts §6 ↔ `context-response.schema` | **FIXED（2 处）** | `allow/deny/error` → `ALLOW/DENY/INDETERMINATE`（error/timeout 归 INDETERMINATE），03 正文与 §9.4 时序图同步 |
| Outbox 版本语义 | 03 §7.6.4/§7.6.7、SM-4、schema | PASS | `aggregate_version` + `v_event < v_current ⇒ SkipOrSupersede` 一致（状态值问题见 FIXED 项） |
| Worker 事件定义/幂等八项 | 03 §7.7、SM-4、05 §7.5、06 KT | PASS | 幂等、租约恢复、部分失败、dead_letter 一致 |
| 错误码统一（error-codes ↔ 03 §7.10 ↔ error.schema.json） | 03、06、08、README、CHANGELOG、契约 | **FIXED（12 处）** | 登记表/README/CHANGELOG/`error.schema.json` 为**16 规范码**，而 03/06/08 按"11 码"表述且把 5 个扩展码映射到其他码：03 §7.10 规范码集改为"11 核心 + 5 扩展 = 16"、别名表逐行改为独立保留、尾段删除 TBD 并写明 OI-03-02 已结、03 §8 与 §13 P03-5 计数改 16、03 OI-03-02 状态改"已结"；06 §2、08 §8.7、08 INT-07 同步为 16 |
| Context Adapter 主体标识 | 08 §7.3/§8.1、04 §7.6 | PASS | 五标识 + `subject_id` 派生规则 + actor_type 七类，与 04 一致 |
| 响应 `meta.algorithm_version` | 00 §2、03 §7.3.3、02 §2 ↔ api-contracts §1 | **FIXED（1 处）** | api-contracts 信封示例 `R4.1-V1` → `1.0`（与 00/02/03 一致） |
| M03/M04 逻辑接口名 | 00 §7.6 ↔ api-contracts §4/§5 | **FIXED（1 处）** | 00 的 `search/getRetrievalTrace`、`submitCandidate`、`consolidate/rebuildObservation` 改为契约名（`retrieve/rankCandidates/buildContext/invalidateCaches`、`submitSource/runQualityCheck/linkEvidence`、`reviewCandidate/detectDuplicates/classifyConflict/proposeMerge/recomputeObservation`），并把指针由"见 03"改为 `api-contracts.md` §3 |
| 候选状态取值 | 08 §7.6 ↔ 02 §7.8 ↔ `candidate.schema.json` | **FIXED（1 处）** | 08 "写候选池（`quality_status=unverified`）"→ `quality_status=draft` + `validation_state=unverified`（`unverified` 不是 quality_status 枚举值） |

### C-03 算法一致性（权威标识、Embedding 八字段、RRF、规则排序六特征、Token Budget、Observation 继承）

| 检查项 | 涉及文档 | 结果 | 说明 / 修复位置 |
|---|---|---|---|
| FTS5/LanceDB 共用 `memory_id + version` 权威标识 | 01 §7.13、02 §5.4.1/§7.3、`search-document.schema` | PASS | `memory_version` 防旧覆盖、`document_hash` 判定重建，三处一致 |
| Embedding Space 八字段三处一致 | 01 §7.13.2、07 §7.5、schema | **FIXED（3 处，见 C-01）** | 仅 `distance_metric` 取值不一致，已统一为 `cosine/l2/dot_product` |
| RRF：rank 从 1、缺失贡献 0、k=60 | 02 §7.4、00 §7.8、06 §7.10 ABL-C、`retrieval.schema.json` | PASS | 公式、示例手算值、`rank_start=1`、`missing_channel_contribution=0`、`k=60` 一致 |
| RRF 通道集合与权重 | 02 §7.4 ↔ `retrieval.schema.json` | **历史冲突；SRD-002 已裁决，待全量验证** | 02 计入 `structured`（w=1.0），config `fusion.weights` 仅 `fts`/`vector`；通道标识 `旧通道名与规范 `fts`/`vector` 不一致（已按 SRD-002 修订） |
| 规则排序六特征与权重 | 02 §7.5、06 §7.10、`retrieval.schema.json` | **FIXED（权重）/ CONFLICT（语义，OI-019）** | 权重 0.55/0.15/0.10/0.10/0.10 与 ζ=0.10 四处一致（PASS）；但 06 的符号表把 N/T 互换、缺失值用中性填充，与 02 的"available=false + 权重重分配"冲突 → OI-019 |
| Token Budget 1200 且 Token≠字符 | 00 §7.8、02 §7.6、06 §7.5、`retrieval.schema.json` | PASS | 默认 1200、真实 Tokenizer、五类计量、`estimated` 标记一致 |
| Context Builder `max_items` 默认值 | 02 §7.7 ↔ `retrieval.schema.json` | **FIXED（1 处）** | 02 默认 10 → 20（config `context.max_items=20`，且与 Top-20 一致） |
| Observation 继承来源权限 | 01 §7.11、02 §5.4/§5.5、`observation.schema.json`、04 §7.11 | PASS | 最严格交集、来源变化 ⇒ needs_recompute/invalidated/restricted 一致 |
| 部分结果与降级语义 | 02 §7.3、03 §7.10、05 §7.2 | PASS（命名归 OI-018） | "未批准降级不得启用"一致 |

### C-04 权限一致性（所有读取经 M05、默认拒绝、派生继承、删除清理、群聊/私人隔离、生产禁 allow-all）

| 检查项 | 涉及文档 | 结果 | 说明 |
|---|---|---|---|
| 所有读取经 M05 终检 | 00 §7.6、02 §7.3(8)、03 §7.5、04 §7.4、08 §7.5 | PASS | "逐条终检、排序前判权、不可跳过"一致 |
| 默认拒绝（deny/error/timeout/未知） | 02 §7.10、03 §7.5、04 §7.8、05 §7.2、08 §7.7 | PASS | fail-closed 一致（命名归 OI-018） |
| 派生继承最严格交集 | 00 §11、01 §7.18、02 §5.3、04 §7.11 | PASS | 不可计算 ⇒ 拒绝 |
| 删除清理派生 | 03 §7.6.8、04 §7.9/§9.3、05 §7.11、SM-7、D-12 | PASS | 七步/六层一致，权威状态兜底 |
| 群聊/私人隔离 | 04 §7.5、08 §7.4.2、test-specs TC-PM | PASS | 双向禁令 + 隔离矩阵一致 |
| 生产禁 allow-all | 04 §7.12、`security.schema.json`、06 SEC-07、08 §11 | PASS | `default_policy` const deny、`test_allow_all.enabled` const false、`m05_mode=contract_deny` 一致 |
| 权限测试 11 场景 | 04 §12、06 §7.6、test-specs TS-3 | **CONFLICT（OI-021）** | 集合与编号不一致（04 无 `system_internal`；06 两管理角色合并为 P-11；test-specs 12 条） |

### C-05 运行一致性（SQLite 事务覆盖 Outbox、Worker 幂等、删除传播、索引可重建、迁移可回滚、故障矩阵无静默降级）

| 检查项 | 涉及文档 | 结果 | 说明 |
|---|---|---|---|
| 事务五步覆盖 Outbox | 02 §7.2、03 §7.4/§7.8、D-02/D-03、06 KT-M01-01 | PASS | 同一事务、双向禁止半写一致 |
| Worker 幂等/租约/重试 | 03 §7.6.9/§7.7、SM-4、05 §7.5 | PASS | 幂等七要求 + 八项能力一致 |
| 删除传播与墓碑 | 03 §7.6.8、SM-7、04 §7.9、05 §7.11 | PASS | 七步、不回退权威删除态、备份重放一致 |
| 索引可重建 | 05 §7.4、07 §7.12、06 RC-03/RC-04 | PASS | 九步/六模式一致 |
| 迁移可回滚 | 07 §7.9、SM-6/SM-8、06 RC | PASS | 六明确 + 状态矩阵一致 |
| 故障矩阵无静默降级 | 05 §7.2/§9.3、00 §10、06 FI-03/FI-04 | PASS | LanceDB"仅 FTS5 降级"仍标 TBD 未批准 |
| Outbox 状态机矩阵（03 ↔ SM-4） | 03 §7.6.6、SM-4 | **FIXED（3 处）** | ①`retry_wait→pending` 改为 `→processing`（SM-4）；②阈值 `≥上限` 改为 `>retry_max`；③补齐 `cancelled`、`SkipOrSupersede` 两行；SM-4 第 5 行 dead_letter 触发扩展为 `retry_wait / processing`，第 6 行 `superseded` 终态标记改为 `succeeded` + `last_error`（不新增第六个 status 值，与 `outbox-event.schema` 五枚举一致） |
| SM-8 与 07 §7.4.3 迁移矩阵 | SM-8、07 §7.4.3 | **FIXED（1 处）** | SM-8 缺 `rolled_back → not_started`（重新发起）→ 补第 9 行 |
| 迁移状态枚举表述 | 07 §2/§4、05 §8 | **FIXED（3 处）** | 引用了不存在的 `embedding-migration-state.md`/`migration-state.md`/`outbox-state.md`/`worker-state.md` → 统一指向实际文件 `contracts/state-machines/state-machines.md`（SM-1…SM-8 同文件） |

### C-06 图文一致性（15 张图与正文接口/状态名）

| 检查项 | 涉及文档 | 结果 | 说明 |
|---|---|---|---|
| D-01 系统总体架构 | diagrams vs 00 §7.2/§7.3 | PASS | 模块名、Worker/Outbox/五平面与正文一致；`contract_deny` 标注一致 |
| D-04 Worker 索引同步 | diagrams vs 03 §7.7/05 §7.4 | PASS | `FTS5 delete + LanceDB delete + verify`、`retry_wait`、`dead_letter` 与正文一致 |
| D-06 M02 权限终检 | diagrams vs 02 §5.6/§7.6 | PASS | 排除原因枚举 NOT_FOUND/DELETED/INELIGIBLE/PERMISSION_DENIED/BUDGET_EXCEEDED/REDUNDANT 与 `context-response.schema.exclusion_reason` 一致 |
| D-14 Embedding 迁移 | diagrams vs 07 §7.4.2/§9.1、SM-8 | PASS | 九态与禁止项一致（补 SM-8 第 9 行后仍一致） |
| D-05/D-07/D-10/D-15 抽查 | diagrams vs 02/04/08 | **CONFLICT（OI-017/OI-018 导图）** | D-07 分区用五 section 名，02 §7.7 用 must/relevant/background/uncertain；D-15 `canRead` 返回 `allow/deny` 与 03 §7.5 新三态不一致（正文已改，图未改；因 OI-018 未裁决，图暂留） |
| 其余 8 张图 | diagrams vs 00–08 | PASS（抽查） | 未发现"图有正文无"的接口/状态；逐张全量比对未做（**NOT RUN**：D-02/D-03/D-08/D-09/D-11/D-12/D-13 仅结构核对） |
| D-15 返回八字段 | diagrams vs 08 §7.7 | **CONFLICT（OI-017）** | 图与 08 一致，但与 `context-response.schema` 不一致，随 OI-017 裁决 |

### C-07 状态机一致性（SM-1~SM-8 ↔ 01 三状态表 / 03 Outbox 与任务）

| 检查项 | 涉及文档 | 结果 | 说明 |
|---|---|---|---|
| SM-1 lifecycle | SM-1、01 §7.7.2、01 图01-2 | PASS | T-L-01…T-L-09、禁止转换一致；SM-1 第 10 行合并 `active/archived→deleted` 与 01 的 T-L-07 等价 |
| SM-2 validation | SM-2、01 §7.7.1、图01-3 | PASS | 七转换一致；`rejected→unverified` 路径 01 T-V 表未列（SM-2 允许"重新入审"），属细化不矛盾，仅记录 |
| SM-3 availability | SM-3、01 §7.7.3、图01-4 | PASS | `suspended→available` 禁止一致 |
| SM-4 outbox | SM-4、03 §7.6.6、schema | **FIXED（2 处）** | 见 C-05；修复后五态、阈值、取消/过期标记三处一致 |
| SM-4 vs `outbox-event.schema.status` | SM-4、schema | **FIXED** | 删除 `superseded` 第六值 |
| SM-5 worker/task | SM-5、03 §7.9 | PASS | 五态、字段九项、cancelled≠succeeded 一致 |
| SM-6 migration | SM-6、07 §7.2.3/§9.2 | PASS | not_started→…→completed/rolled_back 一致 |
| SM-7 deletion | SM-7、03 §7.6.8、04 §9.3、05 §7.11 | PASS | 六步与七步传播映射一致（顺序等价） |
| SM-8 embedding migration | SM-8、07 §7.4 | **FIXED（1 处）** | 补 `rolled_back→not_started` |

### C-08 参数一致性（15 个 EXP-DEFAULT 参数）

| 检查项 | 涉及文档 | 结果 | 说明 |
|---|---|---|---|
| 数值一致性（FTS/Vector Top-K 50、RRF k 60、w 1.0/1.0、α0.55、β0.15、γ0.10、δ0.10、ε0.10、ζ0.10、Rerank Top-K 20、Token 1200、Outbox poll 1000、衰减 λ 表、S 表） | 00 §7.8、02 §7.4/§7.5/§7.6、06 §7.10、test-specs TS-2/TS-5、`retrieval.schema.json` | PASS | 数值全部一致；均标"实验初值（EXP-DEFAULT）/未裁决" |
| `max_items` 默认值 | 02 §7.7、`retrieval.schema.json` | **FIXED（1 处）** | 10 → 20 |
| 参数 ID 命名 | 02 §7.3/§7.5/§7.6、`retrieval.schema.json`、06 §7.5、08 §8.2 | **CONFLICT（OI-023）** | 三套 ID 方案（`retrieval.rrf.k` vs `fusion.k` vs `memory_context_budget`） |
| 参数标注合规 | 00–08、config | PASS | 无"已裁决"误标 |

### C-09 引用一致性（台账编号真实存在、禁止 PROPOSED 写成 APPROVED）

| 检查项 | 涉及文档 | 结果 | 说明 |
|---|---|---|---|
| 台账编号存在性 | 00 §5、01 §5、02 §5、03 §5、04 §5、05 §5、06 §5、07 §5、08 §5 vs `08-adjudication-ledger.md` | PASS | 实测台账存在的引用码：D-AP-01…10、I-1…8、K-1/K-2/K-4/K-6、X-1…8、U-1/U-2/U-3…7、CR-08/09/10、PC-2/PC-6、ME-4/5、优化-1/3/4/5；抽查未发现引用不存在的编号 |
| PROPOSED/APPROVED 表述 | 全部文档 + 契约 | PASS | 未发现把 Agent 建议写成 APPROVED；`error-codes.md`/`api-contracts.md`/`state-machines.md` 的"结构 APPROVED、名称级 PROPOSED"表述合规；F2/F3 未被写成 F4 |
| 文档间文件级交叉引用 | 04 §8、05 §8、07 §2/§4、08 §4/§8 | **FIXED（6 处）** | 指向不存在的 `contracts/api/permission-api.md`、`context-adapter-api.md`、`contracts/state-machines/{outbox,worker,migration,embedding-migration}-state.md` → 改为实际存在的 `api-contracts.md`/`state-machines.md` |
| OI 编号索引 | OPEN-ISSUES §2 ↔ 各文档 §14 | **FIXED（1 处）** | OI-05-01 在 05 §14 为"无摘要时索引主体回退策略（D-AP-04）"，索引表却写"磁盘阈值"→ 已按 05 修正 |
| 自编号一致性 | test-specs TS-3 | **FIXED（1 处）** | 标题"11 场景"而表内 TC-PM-001…012 共 12 条 → 标题改 12（集合定义本身见 OI-021） |

### C-10 交叉引用可解析（含组 A 自查修复的 7 处）

| 检查项 | 结果 | 说明 |
|---|---|---|
| 00 内部 §引用（§7.x/§8/§9/§12/§13/§14） | PASS | 均指向存在小节 |
| 01 内部引用（§7.4/§7.6/§7.8/§7.16/§7.18/§8.3） | PASS | 存在，编号连续 |
| 02 内部引用（§7.5 3.1/3.2/3.3、§8.5、§9.1–9.12） | PASS | 存在；`§7.10 注` 指向 ALG-09 小节，可解析 |
| 03/04/05 互引（03 §7.5→04、04 §9.3→03、05 §7.4→01、05 §7.6→03-m01 D8） | PASS | 可解析 |
| 06/07/08 互引（06 §7.16.2→08 VERIFY、07 §7.8→00 版本策略、08 §7.13→07 §7.1） | PASS | 可解析 |
| 组 A 自查 7 处修复是否回归 | PASS | 未发现重新出现的失效引用（抽查 00/01/02 的 §引用与图号） |
| 契约↔文档引用 | **FIXED（6 处，见 C-09）** | 修复后全部可解析 |

---

## 2. 修复清单（45 处，按文件）

> 依据优先级：DD-AUTHORITY（R1–R10/§5/§6）> 裁决台账 > `contracts/schemas`/`config` > `error-codes.md`（唯一登记处）> 正文多数一致侧。全部为精确替换，未重写章节。

| # | 文件 | 位置 | 修复内容 |
|---|---|---|---|
| 1 | 00 | §4 D7 | Schema 数量 5→12，补全 12 份清单 |
| 2 | 00 | §8.2 | Search Document / Embedding Space / Outbox Event 三行"待补"→"已存在（PROPOSED）" |
| 3 | 00 | §7.6 | M02/M03/M04 核心接口名改为 api-contracts 契约名；指针"见 03"→`api-contracts.md` §3 |
| 4 | 01 | §1 | "待补契约"行改为"其余契约…均已存在" |
| 5 | 01 | §4 D7 | Schema 数量 5→12 |
| 6 | 01 | §8.1 | 三行"待补"→"已存在（Draft-07）"；search-document required 去 `sensitivity`、补 `document_hash` |
| 7 | 01 | §7.13.1 | `document_id` 类型 uuid→string ≤256 |
| 8 | 01 | §7.13.1 | 增 `cue` 行 |
| 9 | 01 | §7.13.1 | 增 `document_hash`、`index_versions` 行 |
| 10 | 01 | §13 A01-09 | "18 字段"→"21 字段" |
| 11 | 01 | §7.13.2 | `distance_metric` `ip`→`dot_product`（别名） |
| 12 | 02 | §8.8 | 三处"待补"→"已存在" |
| 13 | 02 | §7.7 14) | `max_items` 10→20 |
| 14 | 03 | §7.6.4 | `payload_version` int→string；`next_retry_at` 正式化；补 `document_hash`/`embedding_profile_id` |
| 15 | 03 | §8 | Outbox 事件字段数 10→13 |
| 16 | 03 | §7.6.6 | 状态矩阵对齐 SM-4（retry_wait→processing、`>retry_max`、补 cancelled/SkipOrSupersede 行） |
| 17 | 03 | §7.3.2 | `actor.type` 补 `owner`（七类） |
| 18 | 03 | §7.5 | 权限三态改 `ALLOW/DENY/INDETERMINATE` |
| 19 | 03 | §9.4 | 时序图 `allow/deny/error`→三态名 |
| 20 | 03 | §7.10 | 规范码集改为 11 核心 + 5 扩展 = 16 |
| 21 | 03 | §7.10 | 兼容别名表逐行对齐 error-codes（5 扩展码独立保留） |
| 22 | 03 | §7.10 | 删除 DUPLICATE/BUDGET TBD 段，写明 OI-03-02 已结 |
| 23 | 03 | §8 | "11 规范码"→"16 规范码（11+5）" |
| 24 | 03 | §13 P03-5 | 同上计数修正 |
| 25 | 03 | §14 OI-03-02 | 状态 TBD→已结 |
| 26 | contracts/schemas/permission-context | `actor.type.enum` | 补 `bot` |
| 27 | contracts/state-machines | SM-4 #5/#6 | dead_letter 触发扩为 `retry_wait / processing`；`superseded`→`succeeded` 终态标记（不新增枚举值） |
| 28 | contracts/state-machines | SM-8 | 补第 9 行 `rolled_back → not_started` |
| 29 | 04 | §8 | `contracts/api/permission-api.md`→`api-contracts.md`（§6） |
| 30 | 05 | §8 | 三个不存在的 state-machine 文件→`state-machines.md`（SM-4/5/6/7/8） |
| 31 | 07 | §2 | `embedding-migration-state.md`→`state-machines.md` SM-8 |
| 32 | 07 | §4 | 同上（两文件→同文件两 SM） |
| 33 | 07 | §7.5 | `distance_metric` 取值为 `cosine/l2/dot_product`（别名 ip） |
| 34 | 07 | §8.2 | 同上 |
| 35 | 07 | §7.7 | `search_document_version`→`schema_version`（两处：变更类型行、规则 1） |
| 36 | 08 | §4 | `context-adapter-api.md`/`permission-api.md`→`api-contracts.md` §6/§7 |
| 37 | 08 | §8 | 同上前言指向修正 |
| 38 | 08 | §7.6 | `quality_status=unverified`→`draft` + `validation_state=unverified` |
| 39 | 08 | §8.7 | 错误码"11 码"→"16 规范码之一" |
| 40 | 08 | §12 INT-07 | 同上（并注明 §7.12 列 11 核心码） |
| 41 | 06 | §2 | 错误码"十一个规范名"→16 规范码说明 |
| 42 | contracts/api/api-contracts.md | §首 | `algorithm_version` `R4.1-V1`→`1.0` |
| 43 | OPEN-ISSUES.md | §2 | OI-05-01"磁盘阈值"→"无摘要时索引主体回退策略（D-AP-04）" |
| 44 | test-specs | TS-3 标题 | "11 场景"→"12 场景"（与表内 12 条一致） |
| 45 | 01 | §1 | "待补契约"标签改为"其余契约"（措辞修正，避免与已存在契约矛盾） |

附加（不计入 45 处的登记动作）：`OPEN-ISSUES.md` 新增 §5（OI-016…OI-024 九项 CONFLICT 登记）。

---

## 3. CONFLICT 清单（9 项，均已在 OPEN-ISSUES §5 登记）

| OI | 一句话冲突 | 双方位置 | 需用户裁决 |
|---|---|---|---|
| OI-016 | M02 检索请求字段有三套命名（`query` / `query_text` / `current_message`），主机与 M02 无法按同一契约实现 | 02 §7.3 ↔ 08 §8.2 ↔ `retrieval-request.schema.json` | 是 |
| OI-017 | 宿主消费的"响应八字段"与 `context-response.schema.json` 的 required 结构完全不同，同一响应无法两处解析 | 03 §7.3.4 / 08 §7.7 / api-contracts §7 ↔ schema | 是（阻塞契约冻结） |
| OI-018 | 检索与权限的状态枚举存在四套取值，`partial/degraded/INDETERMINATE` 语义不可对齐 | 02 / 06 / 08 / 两份 response schema | 是 |
| OI-019 | 六特征中 N（类型）与 T（时间）在 02 与 06 中互换，且缺失值处理一为权重重分配、一为中性填充，同一 Score 不可复现 | 02 §7.5 3.1 ↔ 06 §7.10 | 是 |
| OI-020 | Memory Object 是否含 `scope_id`：01 定义并强制，`memory-object.schema.json` 无该字段且 `additionalProperties:false` 会拒绝，A01-01"diff=0"不成立 | 01 §7.5/§7.6.1/§9.1 ↔ schema（关联 OI-01-01） | 是（Schema 变更） |
| OI-021 | 权限测试"11 场景"集合与编号不一致（04 无 `system_internal`、两管理角色分列；06 合并为 P-11；test-specs 12 条），04 §12 的对齐说明编号错位 | 04 §12 ↔ 06 §7.6 ↔ test-specs TS-3 | 是 |
| OI-022 | RRF 是否计入结构化通道及其权重（02 计 w=1.0，config 无 `structured` 权重），且通道标识 `fts5/lancedb` 不在 schema 枚举内 | 02 §7.4 ↔ `retrieval.schema.json`/`retrieval-response.schema`（关联 OI-02-09） | 是 |
| OI-023 | 15 个 EXP-DEFAULT 参数存在三套参数 ID（`retrieval.rrf.k` / `fusion.k` / `memory_context_budget`），不满足 A16/优化-5 的可追踪要求 | 02 §7 ↔ `contracts/config/retrieval.schema.json` ↔ 06/08 | 是 |
| OI-024 | `embedding_spaces.status` 被同时用作"空间生命周期"与"迁移状态"，三处枚举互不相容（07 三态 / schema 五态 / 01 两态 / SM-8 九态） | 07 §8.2 ↔ `embedding-space.schema.json` ↔ 01 §7.16 ↔ SM-8 | 是 |

---

## 4. 需用户裁决汇总

1. **契约冻结前必须裁决**：OI-016、OI-017、OI-018（接口字段与状态枚举），否则宿主适配与 M02 契约无法落码。
2. **Schema 变更需批准**：OI-020（`scope_id`）、OI-024（status 拆分）。
3. **算法口径需裁决**：OI-019（六特征语义与缺失策略）、OI-022（结构化通道是否计分）、OI-023（参数 ID 规范）。
4. **验收口径需裁决**：OI-021（权限场景集合与编号）。
5. 以上均不阻塞 M01/M02 的 P0 技术验证与 Schema/事务实现，但阻塞"契约冻结"与"P2 验收口径"。

---

## 5. 总结论

- **数值/名称/字段级一致性：达成**。45 处不一致已就地修复，覆盖 Schema 字段清单、Outbox 字段与状态机、错误码集（16 码）、权限三态、契约/状态机文件引用、参数默认值、OI 索引、图表状态名。
- **语义级一致性：未完全达成**。遗留 **9 项 CONFLICT**（OI-016…OI-024），全部为"两种设计各自成立、基准未唯一裁定"的事项，已按规则登记而**未单方面改写**；其中 3 项（OI-016/OI-017/OI-018）为接口契约级，建议在契约冻结前裁决。
- 本次共**修复 45 处**，**遗留 9 项 CONFLICT**，**0 项未检查**（图文逐张全量比对仅 D-01/D-04/D-06/D-14 完整、其余抽查，已在 C-06 注明）。
- 未触碰 `_sources/`、`08-adjudication-ledger.md` 与目录外任何文件；未修改 08 台账编号引用与任何 APPROVED/PROPOSED 状态语义。

（END）