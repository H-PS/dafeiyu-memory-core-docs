# OPEN-ISSUES — Memory Core R4.1 全量详细设计 · 未决事项总表

> 2026-11-04 ｜ 汇总总控级 OI-001~014 与子文档 OI-03-xx/OI-04-xx/OI-05-xx/OI-06-xx/OI-07-xx/OI-08-xx。
> 状态：TBD / VERIFY / BLOCKED / DEFERRED（全部 NOT RUN，无编造结果）。判定来源：sf 后文 §十四。

## 1. 总控级（OI-001~014）

| id | 问题 | 模块 | 缺失信息 | 风险 | 推荐 | 阻塞 | 需用户决定 |
|---|---|---|---|---|---|---|---|
| OI-001 | LanceDB FLAT 执行路径未验证（VF-001 八项） | M02/P0 | 实际执行证据 | 检索质量声明失效 | P0 基准比对 | **P2 验收前是** | 否（验证后自动） |
| OI-002 | SQLite 驱动未裁决 | M01/P0 | 实测数据 | 事务/并发差异 | P0 验证后登记 | **P1 开工前是** | 批准选型记录 |
| OI-003 | 中文分词未解决 | M02 | 分词器对比 | 中文召回质量 | P2 前对比（TD-001） | 否 | 选型批准 |
| OI-004 | LanceDB 故障"仅 FTS5 降级"未批准 | 05 | 白名单 | 可用性 vs 正确性 | 未批准=不降级只报错 | 否 | 是否批准该降级 |
| OI-005 | revoke 语义二义（撤销授权 vs 撤销可用性） | API/04 | 用户裁决（PD-007 默认可用性撤销） | 误删授权 | 采纳 PD-007 | 否 | 确认语义 |
| OI-006 | K-1 运行时 KV 落位设计 | M01 | K-2 导出边界清单 + D2/D10 落位 | 缓存一致性 | 前置完成再冻结 | **BLOCKED** | 随 K-2 |
| OI-007 | reinforce 回传链路（E14/B04） | 08 | 链路验证 | 强化功能缺失 | 保持 schema 禁写 | **BLOCKED**（功能级） | B04 验证授权 |
| OI-008 | L0 跨用户作用域（PC-6） | 00/04 | 台账前置核验 | 隔离渗透风险 | FUTURE 前完成 PC-6 | 否（FUTURE） | PC-6 核验 |
| OI-009 | J4 情绪标签 15 词表内容 | M03 | 实施设计 | 标注粒度 | 实施期定 | 否 | 词表确认 |
| OI-010 | 宿主真实接口未验证（QQ Bridge/DSH） | 08 | 真实 API 行为 | 集成返工 | P9 前实测（VERIFY-08-01…12） | 否（P9 前置） | — |
| OI-011 | Benchmark 质量阈值数值 | 06 | 基线数据 | 无法宣布达标 | P3 基线后呈报（PD-014） | 否 | 阈值裁决 |
| OI-012 | 备份周期/保留数量 | 05 | 数值 | 运维风险 | 参考审计 365 天 | 否 | 数值批准 |
| OI-013 | ID 格式（UUIDv7?） | 01 | 用户偏好（PD-001） | 迁移成本 | UUIDv7 | 否 | 格式批准 |
| OI-014 | U-3~U-7 长期悬置 | 全局 | 触发条件 | 低 | 触发后呈报 | 否 | — |
| **OI-015** | **A 层迁移"立即可召回"与 D-AP-05 冲突（组 A OI-01-03，唯一设计级阻塞）** | 01/M01/M04 | A 层历史向量记忆迁移后无 Evidence，而 active 需 validation=confirmed，confirmed 仅 M04 通道写入（台账 D-AP-05）；M04 未实现前 A 层迁移记忆无法合法成为 active | 迁移语义与权限裁决冲突：若放宽=M04 旁路；若严格=A 层迁移后不可召回（违背"保留 A 层语义"母本要求） | 方案 A：迁移记忆以 lifecycle=active + validation=unverified 入库，检索可用但标记"未验证来源"；方案 B：M04 预授权批量确认通道（迁移专用、一次性、全审计） | **是（P1 迁移设计前）** | 在 A/B 两方案间裁决（或给出第三方案） |

## 2. 子文档登记（按文档索引）

| 文档 | 未决项 |
|---|---|
| 03 | OI-03-01 revoke 语义（=OI-005）；OI-03-02 端点超时数值 EXP-DEFAULT；OI-03-03 幂等键 TTL/倍率；OI-03-04 Worker 租约/心跳阈值；OI-03-05 K-1 落位（BLOCKED，台账 K-1）；OI-03-06 reinforce 回传（BLOCKED，台账 D-AP-05）；OI-03-07 宿主真实接口（VERIFY） |
| 04 | OI-04-01 sensitivity 分级细则（关联 3.6）；OI-04-02 导出边界逐项清单（BLOCKED，台账 K-2）；OI-04-03 群 Session 归因、L0 跨用户（TBD/VERIFY，台账 CR-09、PC-6）；OI-04-04 宿主身份映射验证（VERIFY）；OI-04-05 revoke 精确语义（TBD） |
| 05 | OI-05-01 无摘要时索引主体回退策略（TBD，台账 D-AP-04）；OI-05-02 cache 一致性（暂缓，台账 U-1/U-2）；OI-05-03 FLAT 路径（VERIFY/BLOCKED）；OI-05-04 仅 FTS5 降级（TBD）；OI-05-05 备份周期/保留（TBD）；OI-05-06 Outbox 重试上限/退避/最老事件年龄（EXP-DEFAULT 可配置） |
| 06 | OI-06-01 延迟门槛（TBD）；02 指标容差；03 生活化测试集规模（实施期，D-AP-06）；04 M05 替身可信度（VERIFY，PC-2）；05 ME-5 拔除测试（BLOCKED）；06 CR-09 门禁（BLOCKED）；07 cache 一致性（DEFERRED，U-1/U-2）；08 逻辑时钟注入（VERIFY）；09 大集成本（TBD）；10 宿主 token 计量（VERIFY）；11 15 类覆盖度证明（TBD）。另：FLAT 无法证明 ⇒ V-04 BLOCKED；E2E-11（L3/reinforce）FUTURE 禁写 |
| 07 | OI-07-01 回填覆盖率阈值；02 双读容差；03 观察窗口；04 space-id 派生规则（PROPOSED）；05 cache 迁移（DEFERRED）；06 psy-model（DEFERRED，I-7/X-6）；07 B 层字段映射（TBD）；08 A 层历史向量空间归属（VERIFY）；09 迁移期写入归属（TBD）；10 API 弃用窗口（TBD）；11 K-2 导出边界（BLOCKED）。MV-01…14 迁移验收全部 NOT RUN；FUT-01 ANN／FUT-02 Cross-Encoder／FUT-03 关系图谱与 Graphiti（仅登记，优化-3）／FUT-04 自动反思／FUT-05 Dream／FUT-06 Context Compression／FUT-07 多宿主／FUT-08 分布式 |
| 08 | OI-08-01…12 + VERIFY-08-01…12（QQ Bridge 身份解析、DSH 注入点、L3 回传链路/端点、传输鉴权、宿主 tokenizer、宿主等待预算、群私聊切换刷新、多 Bot tenant、Context Adapter 进程边界、Prompt 快照可读性、DSH 是否已有直连 SQLite 路径、富媒体来源链）；BLOCKED：OI-08-04（PC-2）、OI-08-06（K-6/CR-09）；TBD：L3 端点/传输、云端传输隐私约束（D-AP-09：不得声称"记忆文本不出本机"）、记忆预算占比、P9 与母本 P4+ 编号对应；FUTURE：P9 宿主接入、M03/M04/M05、B 层统一管理、Context Compression、Persona（CR-08B 单独呈报）；DEFERRED：缓存（U-1/U-2）、psy-model（I-7/X-6）、L1 仅预留（X-2） |

## 3. 一致性处理记录（组 C 上报 3 项的裁决）

| # | 上报问题 | 处理 | 状态 |
|---|---|---|---|
| 1 | 03 recall 响应 7 字段缺 versions | 已修复为八字段（含 versions，注明与 sf 后文 §9.4、08 对齐） | ✅ 已修 |
| 2 | context_contract（07 八维）与 Context Format Version（sf §8.3）两称谓 | 维持 07/08 已写入的对齐条款：同一维度两称谓、不得各自独立取值、组合关系以 00 版本策略为准；登记为 PD 级命名统一待批 | ✅ 已对齐（登记） |
| 3 | 子文档 OI 编号合并 | 本文件 §2 表完成索引合并 | ✅ 已合并 |

## 4. 阻塞与完成判定（sf 后文 §十四）

- 影响"设计完成"宣布的关键 TBD：**无**（数据安全/权限隔离/数据一致性/核心接口四类均已定稿或以契约默认拒绝兜底）。
- 功能级 BLOCKED（不阻塞设计，阻塞对应功能实施）：BL-001 reinforce、BL-002 K-1、BL-003 psy-model、OI-005/OI-007 相关链路。
- 实施期验证（NOT RUN）：VF-001~005、MV-01~14、TS 全系列。
- 结论：**设计体系达到 F2（全）+F3（规格齐备）；F4 与实施授权不在本次范围。**

## 5. 跨文档一致性 CONFLICT 登记（OI-016~OI-024，2026-11-04 组 C 一致性检查新增）

> 以下 9 项为**语义冲突**（两处设计各自成立但互不相容），按修复规则不做单方面改写，仅登记待裁决。逐项证据见 `reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md`。

| id | 冲突 | 双方位置 | 影响 | 建议方案（PROPOSED） | 需用户裁决 |
|---|---|---|---|---|---|
| OI-016 | M02 检索**请求**字段三套命名 | 02 §7.3（`query`/`scope_filter`/`top_k_per_channel`/`budget_tokens`）↔ 08 §8.2（`query_text`/`scope_candidates`/`memory_context_budget`/嵌套 `identity`）↔ `retrieval-request.schema.json`（`current_message`/`scope`/`retrieval_options.*`/`requester_id`） | 宿主与 M02 间契约不可实现 | 以 schema 为唯一线上契约，02/08 改为"字段映射表"表述 | 是（契约冻结前） |
| OI-017 | `ContextResponse` **响应**结构两套 | 03 §7.3.4 / 08 §7.7 / api-contracts §7 的"八字段"（context_id/request_id/memory_refs/versions/token_usage/retrieval_status/permission_status/adapter_version）↔ `context-response.schema.json` 的 required（query_id/status/selected_memories/total_tokens/permission_trace/selection_trace/algorithm_version） | 宿主无法按同一契约解析响应 | 二选一：① schema 增补八字段（作 API 信封层）；② 八字段降为 schema 的超集说明 | 是 |
| OI-018 | 检索/权限**状态枚举**四套 | 02 §7.3/§8.2（`complete/partial/failed`、`all_allowed/partial_denied/denied`）↔ `retrieval-response.schema`（`success/partial/failed`）↔ `context-response.schema`（`success/empty/failed`、`ALLOW/DENY/INDETERMINATE`）↔ 08 §7.7（`ok/ok_empty/partial/unavailable/degraded`、`allow/deny/error/timeout/not_checked`） | 判定与降级语义不可对齐 | 统一为 `success|partial|failed|empty` + 权限 `ALLOW/DENY/INDETERMINATE`（timeout/error 归 INDETERMINATE） | 是 |
| OI-019 | 六特征 **N/T 定义互换**且缺失值策略冲突 | 02 §7.5 3.1（N=类型匹配、T=时间适配；缺失 ⇒ `available=false`、权重重分配）↔ 06 §7.10（N=recency、T=type_match；缺失 ⇒ 0 / 0.5 中性填充） | 同一 `Score` 公式两套语义，结果不可复现 | 以 02 为准（含权重重分配），06 符号表按 02 改写 | 是 |
| OI-020 | Memory Object 是否含 `scope_id` | 01 §7.5/§7.6.1/§9.1（定义 `scope_id` 并对群/会话/项目强制非空）↔ `memory-object.schema.json`（`additionalProperties:false` 且无 `scope_id`，A01-01 声称 diff=0 不成立） | `group_shared/session_only/project_scoped` 无法定位到具体群/会话/项目 | 采纳 OI-01-01 建议：schema 增补可空 `scope_id`（≤128） | 是（Schema 变更） |
| OI-021 | 权限测试"11 场景"集合与编号不一致 | 04 §12 表（11 条：含群管理员、Bot Owner 两条，无 `system_internal`）+ §12 编号对齐说明 ↔ 06 §7.6（P-01…P-11：含 P-05 `system_internal`，两管理角色合并为 P-11）↔ `test-specs` TS-3（TC-PM-001…012，12 条） | 验收口径不一，P-xx 编号引用错位（04 第 5 条实为 06 P-11） | 以 06 §7.6 为准重排 04 §12 与 test-specs（12 条全集，编号一一对应） | 是 |
| OI-022 | RRF channel inconsistency | Historical algorithm/schema mismatch | Ranking and response channel inconsistency | Task §3.5: only `fts` and `vector` participate in RRF; `structured` is filter/router only; response schema edited | DECIDED; global synchronization remains open |
| OI-023 | Parameter ID naming | Historical algorithm/config/alias naming overlap | Parameter tracing/reproducibility | Registry canonical ID is unique; Schema paths are locators; old names are compatibility aliases only | DECIDED; reference scan remains open |
| OI-024 | Embedding-space lifecycle/migration state conflated | 07/01/schema/SM-8 enum mismatch | Migration recovery ambiguity | Split `space_lifecycle` and `migration_state`; Schema edited | DECIDED/schema changed; consumer sync and validation remain open |

---

## 6. 待用户裁决事项 → 单独问题文件（2026-11-04）

全部需要用户拍板的事项（OI-015~024 裁决建议、D-1/D-2/D-3 最小决策集、PD-001~014 一览、程序性问题）已单独归集至：

**PENDING-USER-DECISIONS.md**（同目录）。

裁决流程：用户在该文件的「编号+选项」上回复 → 总控更新本文件与 DESIGN-DECISION-REGISTER 状态（来源=用户裁决）→ 三处同步核验 → CHANGELOG 登记。未裁决项一律按保守路径执行（fail-closed / 不降级 / 不实现）。
---

## 7. V2 架构升级新增未决（OI-025~OI-031，2026-11-04）

来源：DOCUMENT-CONFLICT-REGISTER-V2.md §二（CR-01~CR-07）。全部为**台账冲突**，处理方式一律「保留原裁决 + 标记重新裁决」，未静默改写。

| 编号 | 冲突 | 双方 | 当前默认（保守） | 需用户裁决 |
|---|---|---|---|---|
| OI-025 | AUTO_ACCEPT 候选态 vs confirmed 仅 M04 通道 | 任务书 §7.16 ↔ 台账 D-AP-05 | AUTO_ACCEPT 不写 confirmed；仅 L1/L2 + owner_private，仍走 M04 整合通道 | 是否允许任何形式的自动确认 |
| OI-026 | 情绪检索（EMOTION_QUERY / getEmotionContext）vs 首版关闭查询侧情绪感知 | 任务书 §10.6/§14 ↔ 台账 J6 | emotion.query_enabled=false，情绪检索接口存在但返回 EMOTION_DISABLED | 是否启用查询侧情绪 |
| OI-027 | ΔE 影响当前状态 vs 禁止直接改 PAD / 持久变更走独立审核 | 任务书 §10.5 ↔ 台账 X-3、CR-09、K-6 | emotion.influence_enabled=false；ΔE 仅运行时、不落库、不由沉默触发 | 是否启用事件情绪影响及其上限 |
| OI-028 | L1/L2 自动情绪标注 vs 只预留接口不实现 LLM 标注 | 任务书 §6.2/§7.12 ↔ 台账 J7 | emotion.auto_infer=false；只接受 USER_EXPLICIT，推断字段仅预留 | 是否启用自动情绪推断 |
| OI-029 | Psychology 纳入正式架构 vs psy-model 暂缓 | 任务书 §3.1/§19 P6 ↔ 台账 I-7/X-6 | 设计允许、**实施仍 BLOCKED**；接口标 DEFERRED | 解锁 psy-model 的时点 |
| OI-030 | 原句长期保存 vs 数据最小化/导出边界 | 任务书 §8.3 ↔ 台账 K-2、D-AP-09 | Evidence 设 retention_policy；导出边界仍 BLOCKED；**不得声称记忆文本不出本机** | 原句保留期与导出范围 |
| OI-031 | public 可见性 vs 最小权限 | 任务书 §15.2 ↔ 台账原则 | permission.public_enabled=false，V1 禁用 | 是否启用 public |