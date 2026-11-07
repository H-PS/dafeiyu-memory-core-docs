# DESIGN-DECISION-REGISTER — Memory Core R4.1 设计决策登记表

> 版本 1.0 ｜ 2026-11-04 ｜ 维护者：总控设计 Agent
> 规则（sf 后文 §六）：已批准决策必须引用正式来源；Agent 推荐方案标 PROPOSED；未确认技术能力标 VERIFY；缺信息标 TBD；有阻塞标 BLOCKED；否决标 REJECTED；延后标 DEFERRED。**禁止把 PROPOSED/TBD/DEFERRED 升级为 APPROVED。**

状态统计：APPROVED 38 ｜ PROPOSED 14 ｜ TBD 6 ｜ VERIFY 5 ｜ BLOCKED 3 ｜ DEFERRED 10 ｜ REJECTED 2

## A. APPROVED（来源=用户裁决台账 / 用户指令 / sf 后文规范条款）

| decision_id | title | module | description | source | rationale | impact | verification |
|---|---|---|---|---|---|---|---|
| AD-001 | 五模块边界维持 | 全局 | M01 存储粒度/M02 检索/M03 提取/M04 整合/M05 权限；M03 候选与 M04 裁决不可合并 | 台账 D-AP-01 | 用户裁决 | 全部文档 | 一致性审查 |
| AD-002 | m01/m02 冻结稿保持冻结 | M01/M02 | 冻结稿不因本设计修改 | 台账 D-AP-10 | 用户裁决 | 00/02 | — |
| AD-003 | M04 §6.2 唯一 LLM Gateway；Embedding Provider 归 M01 | M04/M01 | 受控 LLM 唯一入口 | 台账 D-AP-02 | 用户裁决 | 02/07 | PC-1 前置核验 |
| AD-004 | confirmed 仅 M04 通道写入 | M04 | active 记忆确认唯一路径 | 台账 D-AP-05 | 用户裁决 | SM-1/SM-2/02 | 单元测试 TC-SM1-02 |
| AD-005 | reinforce 只预留 schema 禁止写入 | 全局 | E14/B04 未落地前禁写；仅 DSH 回传最终 prompt 中实际出现记忆 ID 可触发 | 台账 D-AP-05/J1/K-4 | 用户裁决 | 02/08/API | B04 链路验证（BLOCKED 前置） |
| AD-006 | 摘要+cue 索引主体；无摘要回退=正文全文 | M01/M02 | SearchDocument content 语义 | 台账 D-AP-04 | 用户裁决 | search-document.schema | Benchmark |
| AD-007 | 生活化测试集为主验收；研究报告收益≠实测 | 06 | 验收依据 | 台账 D-AP-06 | 用户裁决 | test-specs | Benchmark 执行 |
| AD-008 | C40 权限引擎不可插拔；trusted_context_source: entry-only | M05 | 可插拔边界 | 台账 D-AP-07 | 用户裁决 | 04/permission-api | PC-2 前置核验 |
| AD-009 | 拒绝第三方托管平台；不得声称"记忆文本不出本机" | 全局 | 本地持久化与云端推理传输分开评估 | 台账 D-AP-09 | 用户裁决 | 00/04 | — |
| AD-010 | 心理层 L0+L2 起步、五层 schema 预留 | Psychology | 作用域起步集 | 台账 I-3 | 用户裁决 | 00（FUTURE 节） | PC-6 |
| AD-011 | 心理状态机只改运行时状态不产生记忆候选 | Psychology | 与记忆通道边界 | 台账 I-5 | 用户裁决 | 00/12 框架 | — |
| AD-012 | psy-model 暂不引入；五方法/十二阶段 DEFERRED | Psychology | 接口形态未定 | 台账 I-7/X-6 | 用户裁决 | 00/02 | 独立代码审查 |
| AD-013 | J2 写时情绪标注（区分事实/推断）；J4 15 标签配置管理；J6 查询侧关闭；J7 只预留接口；J8 external 字段预留 | M03/M02 | 情绪标注边界 | 台账 §3e J2/J4/J6/J7/J8 | 用户裁决 | candidate.schema | 实施期 |
| AD-014 | openLoop 永不由沉默触发；禁催促、禁与信任度联动 | Psychology | CR-09 伦理门禁（未验证前不闭环） | 台账 K-6/CR-09 | 用户裁决 | 00 FUTURE 节 | CR-09 实施期验证 |
| AD-015 | K-1 运行时 KV 有条件接受，K-2 清单+D2/D10 落位前不生效 | M01 | 非权威 KV 不参与判定/Outbox/判权 | 台账 K-1/K-2 | 用户裁决 | 01（预留注记） | K-2 清单 |
| AD-016 | cache.sqlite 表结构与校验实现不在设计阶段冻结 | M01 | U-1/U-2 组合决策 | 台账 §3e | 用户裁决 | 01/05 | 实施阶段 |
| AD-017 | X-1 delta-only 影子验证不改 M02 最终排序；X-3 Recall 参与推理/禁改 PAD/持久变更独立审核；X-2 Session 丢弃+L1 预留 | Psychology/M02 | 心理加成边界 | 台账 §3g | 用户裁决 | 00/02 | ME-6 diff |
| AD-018 | 参数 ID 规范：稳定/唯一/可追踪；展示名可改身份不丢 | 全局 | 优化-5 | 台账 §3e | 用户裁决 | config/core.schema | 参数注册表运行 |
| AD-019 | Token 计量五分类（系统提示词/聊天历史/记忆检索/实际注入/模块调用） | M02 | 优化-1 | 台账 §3e | 用户裁决 | 02/config | 计量测试 |
| AD-020 | Graphiti 旁路索引仅登记不启动 | 07 | 优化-3 | 台账 §3e | 用户裁决 | 07 Future | 准入条件评估 |
| AD-021 | 拔除测试从 7 条外部依赖起步可扩展 | Psychology | 优化-4 | 台账 §3e | 用户裁决 | 00 Future | ME-5 |
| AD-022 | SQLite 唯一权威；FTS5/LanceDB 派生可重建 | 全局 | 架构不变量 1–3 | sf 后文 §1.4；用户指令 | 规范 | 全部 | 一致性审查 |
| AD-023 | LanceDB FLAT 精确检索；禁止 ANN/Cross-Encoder 默认 | M02 | 首版检索路线 | sf 后文 §3.2.5；用户指令 | 规范 | 02/05/07 | P0 SDK 验证 |
| AD-024 | M02 权限终检契约默认 DENY；生产禁 allow-all | M02/M05 | fail-closed | sf 后文 §3.2.11/§13.3；用户指令 | 规范 | 04/api | TC-PM-008/009 |
| AD-025 | 派生数据继承来源权限；多来源最严格交集；无法确认=拒绝 | 全局 | 权限继承 | sf 后文 §3.5.5；用户指令 | 规范 | 04 | TC-PM-012 |
| AD-026 | 所有正式记忆保留 Evidence+Provenance；缺失来源不得编造 | 全局 | 可追溯性 | m.md §6；用户指令 | 规范 | 01 | TC-M01 链路抽查 |
| AD-027 | DSH 历史是来源不是第三个权威存储层 | 全局 | A/B/C 定位 | m.md §2.2/2.3；用户指令 | 规范 | 00/08 | — |
| AD-028 | Memory Core/Memory API/Host Adapter 三组件边界不变 | 全局 | 组件视图保留 | m.md §1.2；用户指令 | 规范 | 00/08 | — |
| AD-029 | A/B 层语义保留：A→语义检索视图；B 轻量状态；memberImpression 保留推断 | 01 | 迁移语义 | m.md §2.1；用户指令 | 规范 | 01 迁移映射表 | 迁移演练 |
| AD-030 | 时间六字段语义（事实时间≠记录时间；UTC RFC3339） | 01 | sf 后文 §2.4 | 规范 | 01 | Schema 校验 | |
| AD-031 | 三类状态独立禁止合并 status | 01 | sf 后文 §2.3.3 | 规范 | SM-1/2/3 | — | |
| AD-032 | 事务五步+乐观并发 expected_version | M01 | sf 后文 §3.1.4/3.1.5 | 规范 | 03/api | TC-M01-003/004 | |
| AD-033 | Outbox 事件规范名 memory_upsert/memory_delete/memory_reindex；旧名为兼容别名 | 03 | DD-AUTHORITY R2 | 规范 | outbox-event.schema | — | |
| AD-034 | 错误码规范集=sf 后文 §4.5；m.md 名称为别名 | 全局 | DD-AUTHORITY R1 | 规范 | error-codes.md | — | |
| AD-035 | Outbox 五态与通用任务五态是两个状态机 | 03 | DD-AUTHORITY R7 | 规范 | SM-4/SM-5 | — | |
| AD-036 | 参数全部实验初值 EXP-DEFAULT（k=60/Top50/Top20/1200/0.55…） | 02 | sf 后文 §15 | 规范 | config | Benchmark 后裁决 | |
| AD-037 | 完成等级 F1–F4；本设计目标 F2+（尽量 F3）；F2/F3≠F4 | 全局 | sf 后文 §17 | 规范 | 全部 | 审查 | |
| AD-038 | P0–P3=M01+M02；P4+ 另行授权；设计≠实施 | 全局 | sf 后文 §13；用户指令 | 规范 | 00 路线图 | — | |

## B. PROPOSED（总控设计默认，待用户批准后升级）

| decision_id | title | module | description | rationale | alternatives | approval_required |
|---|---|---|---|---|---|---|
| PD-001 | ID 采用 UUID 格式（不锁版本；m.md 建议 UUIDv7） | 01 | 兼容 rowid 禁令与稳定 ID | sf 后文未定；UUIDv7 时间有序利于索引 | ULID/雪花 | 是 |
| PD-002 | SQLite 13 表结构（§01 建表建议） | M01 | memory_objects/memory_versions/evidence/... | 覆盖母本 4.4 全概念 | 合并精简 | 是（P1 前） |
| PD-003 | journal_mode=WAL | M01 | 并发读 | 默认 DELETE | 是 |
| PD-004 | document_id=`memory_id:version:space` | 01 | 文档身份可读可追溯 | 纯 UUID | 是 |
| PD-005 | 敏感度枚举 public/normal/private/restricted，默认 private | 04 | fail-safe 默认 | 二值 | 是 |
| PD-006 | 权限矩阵全表（04 §权限矩阵） | M05 | 全 DENY 默认+显式 ALLOW 条件 | RBAC 简化版 | 是 |
| PD-007 | revoke 语义默认=撤销可用性（可逆），撤销授权需显式 mode | API | 避免误删 | 默认=授权撤销 | 是（OI-03-01） |
| PD-008 | Outbox retry_max=8、backoff 1s 基数、dead_letter 后人工修复 | 03 | EXP-DEFAULT | 指数/线性 | 否（参数级） |
| PD-009 | 冗余惩罚 λ=0.5 | 02 | EXP-DEFAULT | 0.3/1.0 | 否（参数级） |
| PD-010 | Context 分区五段命名 | 02 | 与 sf 前文一致 | 七段 sf 后文变体 | 是 |
| PD-011 | FTS5 external content table 挂接 memory_objects | M01 | 避免双写正文 | 独立 content 表 | 是 |
| PD-012 | health 端点暴露 outbox_backlog/oldest_age | API | 运维可见 | 不暴露 | 否 |
| PD-013 | 审计保留 365 天 | 05 | 合规基线 | 30/90 | 是 |
| PD-014 | Benchmark 阈值留白待基线 | 06 | 不编造数值 | 预设阈值 | 是（P3 后） |

## C. TBD（信息不足，不得填正式值）

| decision_id | 事项 | 缺失信息 | 影响文档 |
|---|---|---|---|
| TD-001 | 中文分词器选型（unicode61 未解决中文） | 候选分词器实测对比 | 02/05 |
| TD-002 | 降级策略白名单（LanceDB 故障时是否允许仅 FTS5） | 用户未批准任何降级 | 05 |
| TD-003 | 备份周期/保留数量 | 用户运维偏好 | 05 |
| TD-004 | revoke 语义最终裁决 | PD-007 待批 | API/04 |
| TD-005 | J4 情绪标签 15 个具体词表 | 实施设计阶段确定（台账 J4） | candidate.schema |
| TD-006 | L0 跨用户作用域（PC-6） | 台账前置核验未完成 | 00 FUTURE |

## D. VERIFY（必须真实验证后才能写"已验证"）

| decision_id | 事项 | 验证方法 | 状态 |
|---|---|---|---|
| VF-001 | LanceDB Node SDK FLAT 实际执行路径（8 项：安装/目录/表/写入/精确查询/TopK/删除/重建） | 暴力余弦基准比对全量一致 | P0 待验 |
| VF-002 | SQLite 驱动（node:sqlite vs better-sqlite3）事务/并发/迁移/恢复 | P0 验证清单 | P0 待验 |
| VF-003 | QQ Bridge 身份解析真实行为 | 宿主接口实测（P9 前） | VERIFY |
| VF-004 | DSH 注入点与 L3 事件回传链路 | E14/B04 链路 | BLOCKED（前置） |
| VF-005 | 宿主 Tokenizer 精确计量可用性 | 宿主接口探测 | P2 待验 |

## E. BLOCKED（前置条件不足）

| decision_id | 事项 | 阻塞原因 | 解锁条件 |
|---|---|---|---|
| BL-001 | reinforce 写入实现 | E14 未落地、B04 回传未确认（台账 D-AP-05） | B04 链路验证 |
| BL-002 | K-1 运行时 KV 落位 | K-2 导出边界清单 + M01 D2/D10 落位设计未完成（台账 K-1） | 两前置完成 |
| BL-003 | psy-model 引入/集成 | 独立代码质量与安全审查未做（台账 I-7/X-6） | 审查+架构映射 |

## F. DEFERRED（明确延后，FUTURE 设计）

DF-001 M03 完整自动提取（P4）｜DF-002 M04 自动反思（P5）｜DF-003 M05 完整引擎（P6）｜DF-004 Dream 自动维护（P7）｜DF-005 Psychology 接入（P8）｜DF-006 Persona 授权视图（P8）｜DF-007 Session Memory 自动迁移（P8）｜DF-008 Context Compression（独立方向）｜DF-009 关系图谱/多跳检索（触发条件评估，D-AP-08）｜DF-010 ANN/Cross-Encoder（Benchmark 触发）

## G. REJECTED（已否决，防止复活）

| decision_id | 事项 | 否决依据 |
|---|---|---|
| RJ-001 | 第三方托管记忆平台 | 台账 D-AP-09 |
| RJ-002 | EmotionBoost ±10% 直排 | 台账 I-1（撤销，归七步链护栏参数，且 X-1 delta-only） |

---

# OPEN-ISSUES — 未决事项清单

> 每项含：问题/所属模块/当前信息/缺失信息/风险/推荐方案/是否阻塞/需要用户作出的决定。

| issue_id | 问题 | 模块 | 当前信息 | 缺失信息 | 风险 | 推荐 | 阻塞 | 需要用户决定 |
|---|---|---|---|---|---|---|---|---|
| OI-001 | LanceDB FLAT 执行路径未验证 | M02/P0 | SDK 候选已定 | 实际执行证据 | 若非精确检索，检索质量声明失效 | P0 执行 VF-001 八项基准比对 | **是（P2 验收）** | 否（验证后自动） |
| OI-002 | SQLite 驱动未裁决 | M01/P0 | 三候选 | 实测数据 | 事务/并发行为差异 | P0 验证后登记 | **是（P1 开工）** | 批准选型记录 |
| OI-003 | 中文分词未解决 | M02 | unicode61 仅基础 | 分词器对比 | 中文召回质量 | P2 前完成 TD-001 对比 | 否（质量风险） | 选型批准 |
| OI-004 | 降级策略未批准 | 05 | 故障矩阵标"未批准不降级" | 白名单 | 可用性 vs 正确性 | 默认不降级只报错 | 否 | 是否批准"仅 FTS5 降级" |
| OI-005 | revoke 语义二义 | API/04 | PD-007 默认可用性撤销 | 用户裁决 | 误删授权 | 采纳 PD-007 | 否 | 确认语义 |
| OI-006 | K-1 落位设计 | M01 | 台账要求 D2/D10 落位 | 落位文档 | 缓存一致性 | K-2 清单后补 | BLOCKED | 随 K-2 |
| OI-007 | reinforce 回传链路 | 08 | schema 预留完成 | E14/B04 | 强化机制缺失（非阻塞） | 保持禁写 | BLOCKED（功能级） | B04 验证授权 |
| OI-008 | L0 跨用户作用域 | 00/04 | PC-6 未完成 | 台账核验 | 隔离渗透风险 | FUTURE 前完成 PC-6 | 否（FUTURE） | PC-6 核验 |
| OI-009 | J4 标签词表 | M03 | 15 个占位 | 词表内容 | 情绪标注粒度 | 实施设计阶段定 | 否 | 词表确认 |
| OI-010 | 宿主真实接口未验证 | 08 | 契约按逻辑定义 | 真实 API | 集成返工 | P9 前实测 | 否（P9 前置） | — |
| OI-011 | Benchmark 质量阈值 | 06 | 指标与流程已定 | 数值阈值 | 无法宣布"达标" | P3 基线后呈报 | 否 | 阈值裁决 |
| OI-012 | 备份周期/保留 | 05 | 协议已定 | 数值 | 运维风险 | PD-013 参考 | 否 | 数值批准 |
| OI-013 | ID 格式（UUIDv7?） | 01 | PD-001 | 用户偏好 | 迁移成本 | UUIDv7 | 否 | 格式批准 |
| OI-014 | U-3~U-7 长期悬置项 | 全局 | 台账保留 | 触发条件 | 低 | 触发后呈报 | 否 | — |

> 关键阻塞（影响"设计完成"宣布）：OI-001/OI-002 属 P0–P1 实施期验证项，**不阻塞设计文档 F2/F3 状态**；BL-001/002/003 为功能级阻塞，均已在文中标注，不属于"关键接口/数据安全/权限隔离"级设计 TBD。因此本设计体系可宣布达到 F2（全）+F3（可验证规格齐备），但 **F4（实践验证）与实施授权不在本次范围**。

---

## H. V2 架构升级决策（2026-11-04，AD-039~AD-056）

来源：用户《统一架构升级与设计文书迭代总任务书》；基准 DD-AUTHORITY-V2.md。

| decision_id | title | module | description | status | source | approval_required |
|---|---|---|---|---|---|---|
| AD-039 | 生命周期三层取代单一枚举 | 01 | memory_tier(L1/L2/L3) + status(4) + candidate.status(6) 三层分离 | APPROVED（用户任务书 §4.1） | 任务书 §4.1 | 否（已入 V1 架构） |
| AD-040 | 语义类型 6 类 + 多标签 | 11 | semantic_type 六值；semantic_tags 多选；SELF_EXPLICIT/SELF_DERIVED 强制分离 | APPROVED | 任务书 §4.2/§4.3 | 否 |
| AD-041 | content_kind 废弃 | 01 | 由 semantic_type + semantic_tags + source_type 三轴替代；旧值映射见 V2 §4.2 | APPROVED | 任务书 §4.2 | 否 |
| AD-042 | Memory Anchor 8 锚点体系 | 11 | 锚点为**获取路由维度**，独立于生命周期与语义分类；可多选 | APPROVED | 任务书 §5 | 否 |
| AD-043 | 人物理解优先策略 | 11 | 默认顺序 SELF_EXPLICIT→PERSON_PROFILE→PREFERENCE→SELF_DERIVED→RELATIONSHIP→EVENT；非全局固定 | APPROVED | 任务书 §5.3 | 否 |
| AD-044 | 五阶段检索取代单一全局排序 | 02 | 意图→锚点→候选→排序→权限→证据扩展；禁止单一固定权重全局排序 | APPROVED | 任务书 §5.6 | 否 |
| AD-045 | Ingestion Pipeline 为正式组件 | 09 | 16 阶段协调层；不直写库、不取代 M03、不绕 M05 | APPROVED | 任务书 §7 | 否 |
| AD-046 | LLM 三层（L1/L2/L3） | 10 | 能力分级 + 动态调用；可为同一模型不同配置；L3 可兼任 L2 | APPROVED | 任务书 §6 | 否 |
| AD-047 | LLM 无数据库写权限 | 09/14 | LLM 只产候选与读取请求；规则校验、M01 落库 | APPROVED | 任务书 §2.2 原则五/§11.5 | 否 |
| AD-048 | Event 与 EmotionContext 分离 | 13 | 事实与情绪分别存储；情绪不得写入 event.content | APPROVED | 任务书 §10.1 | 否 |
| AD-049 | ΔE=I×C×R×D 情绪影响公式 | 13 | 含 8 条门禁；默认关闭；仅运行时 | APPROVED（公式）/ DISABLED（启用） | 任务书 §10.5 | **是（启用时）** |
| AD-050 | Evidence 原句追溯 + 4 读取模式 | 13/14 | summary/memory/evidence/related_evidence；禁止默认返回全部历史 | APPROVED | 任务书 §8.5 | 否 |
| AD-051 | 9 个 Memory Read/Write API | 14 | search/getProfile/getByAnchor/getEvent/getEvidence/getRelated/getEmotionContext/getMemoryById/propose | APPROVED | 任务书 §11.1 | 否 |
| AD-052 | 人物画像为可追溯派生视图 | 12 | 11 分区 + basis 三值；禁止无证据重生成 | APPROVED | 任务书 §9 | 否 |
| AD-053 | source_type 五值 | 01 | USER_EXPLICIT/USER_INSTRUCTION/ASSISTANT_INFERRED/EXTERNAL_SOURCE/SYSTEM_GENERATED | APPROVED | 任务书 §8.6 | 否 |
| AD-054 | visibility 三值 + public 禁用 | 04 | owner_private/group_shared/public(禁用)；scope 保留为内部细分 | APPROVED | 任务书 §15.2 | 否（public 启用需裁决） |
| AD-055 | 文档体系扩为 16 份 | 全局 | 新增 09 Pipeline/10 LLM/11 Anchor+语义/12 画像/13 Evidence+Emotion/14 读写 API/15 参数注册表 | APPROVED | 任务书 §21 | 否 |
| AD-056 | 路线图 P0–P6（实施授权不变） | 07 | P3–P6 新增内容需阶段授权；当前仍 P0–P3=M01+M02 | APPROVED（架构）/ 待授权（实施） | 任务书 §19 | **是（各阶段开工）** |

### V2 参数与开关决策（VD 系列，全部 EXP-DEFAULT 或 DISABLED）

| decision_id | 参数 | 默认 | 状态 |
|---|---|---|---|
| VD-001 | emotion.influence_enabled | false | DISABLED（CR-03） |
| VD-002 | emotion.query_enabled | false | DISABLED（CR-02） |
| VD-003 | emotion.auto_infer | false | DISABLED（CR-04） |
| VD-004 | permission.public_enabled | false | DISABLED（CR-07） |
| VD-005 | memory.auto_accept_threshold | 未定（EXP-DEFAULT） | EXP-DEFAULT + CR-01 待裁决 |
| VD-006 | nchor.profile_priority 等锚点权重 | 见 15 号注册表 | EXP-DEFAULT |
| VD-007 | ecall.evidence_expansion_limit | 见 15 号 | EXP-DEFAULT |
| VD-008 | emotion.max_influence / decay_rate / inferred_weight / elevance_threshold | 见 15 号 | EXP-DEFAULT（启用后才生效） |

> 上述 AD-039~056 均为**任务书直接授权纳入 V1 架构**（APPROVED 来源=用户任务书），与 v1 的 AD/PD 体系并存；VD 系列为参数级默认值，**一律不得写成已裁决**。