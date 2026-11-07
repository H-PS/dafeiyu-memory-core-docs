# 文档冲突清单与统一裁决记录（架构升级 V2）

> 2026-11-04 ｜ 执行者：总控设计 Agent ｜ 依据：用户《统一架构升级与设计文书迭代总任务书》+ `DD-AUTHORITY-V2.md`
> 前置备份：`E:\[PROJECT]\备份全集\Memory-Core-R4.1-Design-pre-V1upgrade-20261104-1810\`（45 文件 + `_SHA256-BEFORE.csv`，逐字节校验通过）
> 本文件对应任务书 §20.4（矛盾处理）与 §21 D-3（冲突登记表）。

---

## 一、结构性冲突（架构级：影响全部文档）

| ID | 冲突 | 旧表述（v1 现状） | 新表述（V2 目标） | 受影响文档 | 裁决 |
|---|---|---|---|---|---|
| SC-01 | **生命周期模型** | `lifecycle_state` 六值单一枚举（candidate/pending_review/active/superseded/archived/deleted） | **双层拆分**：`memory_tier`(L1/L2/L3) + `status`(ACTIVE/SUPERSEDED/ARCHIVED/DELETED) + `candidate.status`(6 值) | 00,01,02,03,06,SM-1/SM-2 | 采用 V2（任务书 §4.1）。旧六值按 DD-AUTHORITY-V2 §4.2 映射，旧枚举保留为"历史别名→新值"映射表 |
| SC-02 | **内容性质字段** | `content_kind` 八值（fact/opinion/preference/intent/hypothesis/inference/event/procedure） | **废弃**，语义由 `semantic_type`(6) + `semantic_tags`(多) + `source_type`(5) 三轴共同表达 | 00,01,02,03,06,08 | 采用 V2。§4.2 映射：inference/hypothesis→source_type=ASSISTANT_INFERRED；procedure→PERSON+PROCEDURAL 标签；fact/opinion/event 由 semantic_type 承担 |
| SC-03 | **语义分类维度** | `type` 七值（semantic/episodic/procedural/preference/intent/relationship/observation） | `semantic_type` 六值 + `semantic_tags` 多标签（含 SELF_EXPLICIT/SELF_DERIVED 强制分离） | 00,01,02,06 | 采用 V2。§4.1 提供七值→六值逐条映射（procedural→PERSON+PROCEDURAL；observation→PERSON+OBSERVATION+ASSISTANT_INFERRED） |
| SC-04 | **检索路由** | 三路并行（FTS5+向量+结构化过滤）+ RRF + 规则排序 | **五阶段**：意图识别(9 类)→锚点路由(8 锚点)→候选检索→排序→权限过滤→证据扩展与预算 | 02,00 | 采用 V2（任务书 §5.6/§14.2）。原三路召回保留为"阶段3 候选检索"的实现方式之一，RRF 保留（不冲突） |
| SC-05 | **写入路径** | 候选→M04→M01→Outbox | 候选→**Ingestion Pipeline 16 阶段**→M03/M04→M01→Outbox（Pipeline 为协调层，不取代 M03、不直写库、不绕 M05） | 03,00 | 采用 V2（任务书 §7）。新增 09 号文档承载，03 号补编排与接口 |
| SC-06 | **情绪** | v1 仅有 Psychology 侧一句话提及 | **Event 与 EmotionContext 彻底分离** + ΔE 公式 + 8 条门禁 | 00,01,02,08 | 采用 V2（任务书 §10）。新增 13 号文档承载；01 补对象；02 补算法 |
| SC-07 | **证据/原句追溯** | Evidence 存在但未定义"原句反向追溯"与四种读取模式 | Evidence 完整字段 + 4 种 read_mode + related_evidence 反向 + 证据权限继承/失效 | 01,14 | 采用 V2（任务书 §8）。13 号承载；14 号 API 承载 |
| SC-08 | **聊天 LLM 读写** | 无正式读写 API（仅 Host Adapter 被动调用） | 9 个 Memory API（主动读取 + propose 写入） | 08,14 | 采用 V2（任务书 §11）。14 号承载；08 补宿主落地 |
| SC-09 | **人物画像** | 无（仅有"画像"零散提及） | PersonProfile 对象 + 11 分区 + basis 三值 + 明确自述优先 | 00,01,02 | 采用 V2（任务书 §9）。12 号承载 |
| SC-10 | **LLM 分层** | 无（仅"D-AP-02 M04 §6.2 唯一 LLM Gateway"） | L1/L2/L3 三层能力分级 + 动态调用 + 11 项能力测试 | 02,08 | 采用 V2（任务书 §6）。10 号承载；保持"C40 权限引擎不可插拔"与 LLM Gateway 唯一入口不冲突 |
| SC-11 | **文档体系** | 9 份（00–08） | **16 份**（00–15，新增 09 Pipeline / 10 LLM / 11 Anchor+语义 / 12 画像 / 13 Evidence+Emotion / 14 读写 API / 15 参数注册表） | 总索引 | 采用 V2（任务书 §21） |
| SC-12 | **路线图** | P0–P3（M01+M02）+ P4+ Future | **P0–P6**：P0 环境 / P1 M01 / P2 M02 / P3 M03+Pipeline / P4 M04 / P5 M05+读写 / P6 Psychology 集成 | 00,07 | 采用 V2（任务书 §19）。**实施授权不变**：历史裁决"当前 P0–P3=M01+M02"保持，P3–P6 新增部分标注"需阶段授权" |

---

## 二、台账级冲突（不得静默改写，已登记待重新裁决）

| ID | 冲突 | 双方 | 本设计处理 | 登记位置 |
|---|---|---|---|---|
| CR-01 | `AUTO_ACCEPT` 候选态 vs confirmed 仅 M04 通道 | 任务书 §7.16 ↔ 台账 D-AP-05 | AUTO_ACCEPT ≠ confirmed；仍需 M04 整合通道写 confirmed；默认仅 L1/L2 且 owner_private | OI-025 |
| CR-02 | 情绪检索（EMOTION_QUERY/getEmotionContext）vs 首版关闭查询侧情绪感知 | 任务书 §10.6/§14 ↔ 台账 J6 | 完整定义能力；`emotion.query_enabled=false` 默认；启用须重新裁决 | OI-026 |
| CR-03 | ΔE 影响当前状态 vs 禁止改 PAD/持久变更走审核/默认关闭 | 任务书 §10.5 ↔ 台账 X-3、CR-09、K-6 | 定义公式与上限；`emotion.influence_enabled=false`；仅运行时、不落库、不由沉默触发 | OI-027 |
| CR-04 | L1/L2 自动情绪标注 vs 只预留接口不实现 LLM 标注 | 任务书 §6.2/§7.12 ↔ 台账 J7 | 定义 EmotionExtractor；`emotion.auto_infer=false`；默认只接受 USER_EXPLICIT | OI-028 |
| CR-05 | Psychology 纳入正式架构图 vs psy-model 暂缓 | 任务书 §3.1/§19 P6 ↔ 台账 I-7/X-6 | 设计允许、实施仍 BLOCKED；接口标 DEFERRED | OI-029 |
| CR-06 | 原句长期保存 vs 数据最小化/导出边界 | 任务书 §8.3 ↔ 台账 K-2、D-AP-09 | Evidence 设 retention_policy；导出边界仍 BLOCKED；不得声称"不出本机" | OI-030 |
| CR-07 | `public` 可见性 vs 最小权限 | 任务书 §15.2 ↔ 台账原则 | 定义但 V1 禁用（`permission.public_enabled=false`） | OI-031 |

---

## 三、文档级待统一项（由各文档"变更记录"落账）

| ID | 事项 | 处理 |
|---|---|---|
| DC-01 | v1 文档中 `scope` 五值 vs V2 `visibility` 三值 | 两字段并存：`visibility` 对外三值、`scope` 内部细分；映射表见 DD-AUTHORITY-V2 §4.3 |
| DC-02 | v1 `source_type` 六值 vs V2 五值 | 逐条映射（§4.2），旧值保留为历史别名 |
| DC-03 | v1 `importance` 是否入排序公式 | 维持 v1 决定：只用于过滤与平局，不入公式（不变） |
| DC-04 | v1 `confidence` 语义 | 保留：系统评估值，不得当事实证明，不得与 LLM 自报等同 |
| DC-05 | v1 三态（validation/lifecycle/availability） | validation 保留；lifecycle→tier+status 双层；availability 保留为运行时可用性 |
| DC-06 | OI-019（六特征 N/T 互换、缺失值策略） | 以 02 号为算法权威，06 对齐（V2 未改变该结论） |
| DC-07 | OI-020（scope_id 取舍） | V2 采纳"删 scope_id"（scope 已承担），Schema 为准 |
| DC-08 | OI-022（RRF 是否计 structured） | 维持"不计入"（用户算法规格原文），通道标识统一 fts/vector |
| DC-09 | OI-023（参数 ID 三套） | 统一为 `15-参数注册表.md` 的 config 点号式（唯一 ID） |
| DC-10 | OI-024（embedding status 双义） | 拆 `space_lifecycle` + `migration_state` |

---

## 四、改造执行清单（本次写入）

| 文档 | 改造动作 |
|---|---|
| 00-总体架构与设计原则 | 补 V2 五原则、17 项能力表、五平面与模块定位表（Ingestion/Psychology 单列）、文档体系 16 份索引、路线图 P0–P6、旧→新术语映射指针；SC-01/02/03/11/12 |
| 01-记忆本体与数据模型 | 五对象字段表（MemoryUnit/Evidence/Event/EmotionContext/PersonProfile）替换旧 MemoryObject 为主模型；tier/status/candidate 三态分层；content_kind 废弃映射；ER 图更新；SQLite 表增 event/emotion/profile/candidate |
| 02-核心算法规范 | 补五阶段检索与 Anchor Router、意图 9 类、人物优先、ΔE、画像整合、去重冲突（16 阶段对应）、语义分类算法；保留 RRF/排序/预算/权限终检 |
| 03-系统运行与协作协议 | 补 Pipeline 编排、9 个 Memory API 契约、候选 6 态、Outbox 不变 |
| 04-权限与治理 | 补 visibility 三值、证据权限继承（related_evidence 不得绕过）、全链路权限覆盖 10 环节 |
| 05-可靠性与运维 | 补四类新故障（LLM 不可用/LanceDB/情感模块/画像整合失败/证据丢失） |
| 06-测试与评估 | 补 8 组新测试（自动分类/画像/锚点/证据/情感/读写/检索/真实运行） |
| 07-扩展与演进 | 更新 P0–P6 路线图与各阶段交付标准、新对象迁移 |
| 08-DSH与QQBridge集成 | 补聊天 LLM 主动读写落地、Psychology 接口边界、L3 事件不变 |
| 新增 09–15 | 见 DD-AUTHORITY-V2 §11 |
| contracts | 新增 7 schema + 4 config，更新 retrieval |
| DESIGN-DECISION-REGISTER | 增 AD-039~ 与 VD 系列 |
| OPEN-ISSUES / PENDING-USER-DECISIONS | 增 OI-025~031 |
| README / DESIGN-STATUS / CHANGELOG | 更新为 16 文档体系与 V2 状态 |
