# 待用户裁决问题清单（PENDING USER DECISIONS）

> 生成：2026-11-04 ｜ 维护者：总控设计 Agent ｜ 性质：**单独汇总所有需要用户拍板的事项**，供逐项裁决。
> 来源：`OPEN-ISSUES.md`（OI-001~024）+ `DESIGN-DECISION-REGISTER.md`（PD-001~014）+ 交付报告。
> 规则：以下所有"总控推荐"均为 **PROPOSED**，不具有批准效力；你回复「编号 + 选项」后，总控才在 DESIGN-DECISION-REGISTER 中把对应项状态改为 APPROVED（记录来源="用户 2026-11-04 裁决"）并同步三处。
> 未裁决项的默认行为：一律按当前文档标注的保守路径执行（fail-closed / 不降级 / 不实现）。

---

## 〇、历史最小决策集（第二轮已按授权任务书处理）

状态说明：本轮用户任务授权全量文书审查、设计裁决、Schema 修改和隔离验证。D-1/D-2 按任务书与现有推荐方案处理；D-3 仅授权 P0 验证工作，不授权正式产品实现或真实数据/宿主接入。未验证技术行为仍不得标为已冻结。

| # | 决定 | 内容 | 不决定的后果 |
|---|---|---|---|
| **D-1** | 契约冲突基准（OI-016/017/018） | 选项 X：**以 contracts/*.schema.json 为唯一基准**，各文档向 schema 收敛（总控推荐）；选项 Y：以文档为准改 schema；选项 Z：三项逐个裁 | OI-016/017/018 保持 CONFLICT，"契约冻结"无法宣布；P0 仍可开工（P0 不涉及这三项），但 P2 API 实现前必须定 |
| **D-2** | 工程默认批准（PD-001~014） | 一键批准全部（推荐，全部是保守默认且可后改）；或逐项挑选（见 §6） | 相关项保持 PROPOSED，代码 Agent 按"PROPOSED 可实现但标注待批"处理 |
| **D-3** | P0 开工授权 | 授权 / 暂缓 | P0 不开工，其余全部挂起 |

---

## 一、契约级冲突（阻塞"契约冻结"；P2 API 实现前必须定 = D-1）

### OI-016 M02 检索请求字段三套命名
- 问题：02 用 `query`、08 用 `query_text`、retrieval-request.schema 用 `current_message`，宿主与 M02 无法按同一契约实现。
- 选项：X=schema 为准（`current_message`，另含 recent_context 等 12 字段）｜Y=文档为准。
- 推荐：**X**。

### OI-017 宿主响应结构二义
- 问题：03/08/api-contracts 的"宿主八字段"（context_id/request_id/memory_refs/versions/token_usage/retrieval_status/permission_status/adapter_version）与 context-response.schema 的 required 结构完全不同，同一响应无法两处解析。
- 选项：X=**分层并存**——schema 是 Core 内部结构化结果，八字段是 Context Adapter 对外信封（由 Adapter 从 schema 结构投影生成），两层各自成立（总控推荐，消解冲突不牺牲任一方）｜Y=合并为一份结构｜Z=八字段为准。
- 推荐：**X**（需在 03/08/api-contracts 写明投影关系）。

### OI-018 检索/权限状态枚举四套
- 问题：`complete/partial/failed`、`success/partial/failed`、`ok/ok_empty/unavailable/degraded`、`allow/deny/error/timeout/not_checked` 四套并存；`partial` 与 `INDETERMINATE` 语义不可对齐。
- 选项：X=schema 枚举为准（检索状态 `success/partial/failed`；权限三态 `ALLOW/DENY/INDETERMINATE`，error/timeout/未知一律归 INDETERMINATE）｜Y=另定第五套。
- 推荐：**X**。

---

## 二、设计语义级（P1/P2 期间定即可；每项给推荐）

| 编号 | 问题 | 选项 | 总控推荐 |
|---|---|---|---|
| OI-019 | 排序六特征 N（类型）/T（时间）在 02 与 06 互换；缺失值一处"available=false+权重重分配"、一处"中性填充" | A=以 02 为准（N=类型、T=时间；缺失=特征不可用置 false 并重分配权重）｜B=以 06 为准 | **A**（02 是算法规范权威文件；重分配可解释且不产生假 0 分） |
| OI-020 | Memory Object 是否含 `scope_id`：01 强制定义，schema 无此字段且 additionalProperties:false 会拒绝 | A=删 scope_id（scope 字段已承担边界语义，schema 为准）｜B=保留 scope_id 并改 schema | **A**（scope_id 与 scope 冗余） |
| OI-021 | 权限测试场景集合与编号（04 无 system_internal；06 把两管理角色合并为 P-11；test-specs 12 条） | A=统一 12 条（test-specs 为准，04/06 对齐编号）｜B=回到 11 条 | **A**（12 条覆盖更全） |
| OI-022 | RRF 是否计入 structured 通道及权重；通道标识 fts5/lancedb vs fts/vector | A=**不计入 structured**（sf 前文 2.2 原文"结构化过滤默认只承担范围约束，不把它当作普通排名通道重复加权"——可由基准唯一判定）；通道标识统一 fts/vector（schema 为准）｜B=计入并给权重 | **A**（02 删 structured w=1.0 行） |
| OI-023 | 15 个 EXP-DEFAULT 参数存在三套参数 ID（retrieval.rrf.k / fusion.k / memory_context_budget），违反 A16/优化-5 可追踪要求 | A=统一采用 config 路径式 ID（如 `retrieval.fusion.k`）并建参数注册表｜B=保留现状 | **A** |
| OI-024 | `embedding_spaces.status` 同时当"空间生命周期"与"迁移状态"，四套枚举互不相容 | A=**拆两个字段**：`space_lifecycle(active/retired)` + `migration_state`（引用 SM-8 九态）｜B=五态合一 | **A** |
| OI-015 | **A 层迁移"立即可召回" vs D-AP-05**（迁移无 Evidence→无法合法 confirmed） | A=迁移记忆 lifecycle=active + validation=unverified，可召回但标"未验证来源"｜B=M04 迁移专用一次性批量确认通道（全审计）｜C=暂缓（迁移属 P4+，P0–P3 用合成数据不受影响） | **C 暂缓**（P4 迁移设计前必须定，届时在 A/B 间裁决） |
| OI-005 | revoke 语义（撤销授权 vs 撤销可用性） | A=默认"撤销可用性"（可逆），撤销授权须显式 mode="authorization"（PD-007）｜B=反转 | **A** |
| OI-013 | ID 格式（m.md 建议 UUIDv7；sf 未定） | A=UUIDv7｜B=UUIDv4｜C=ULID | **A**（时间有序利于索引与去重） |

---

## 三、实施期验证与运维参数（P0–P3 途中自然解锁，无需现在决定）

| 编号 | 事项 | 时点 | 备注 |
|---|---|---|---|
| OI-001 | LanceDB FLAT 执行路径八项验证 | P0 执行 → 结果决定 P2 向量验收是否 BLOCKED | 不需要你决定，需要你看证据 |
| OI-002 | SQLite 驱动三候选实测 | P0 执行 → **你批准选型记录**（一次性） | |
| OI-003 | 中文分词器对比 | P2 前 | 需你批准选型 |
| OI-004 | LanceDB 故障时"仅 FTS5 降级"是否批准 | 任意时点 | 未批准=不降级只报错（当前保守路径） |
| OI-011 | Benchmark 质量阈值数值 | P3 基线出来后 | 届时呈报 |
| OI-012 | 备份周期/保留数量（PD-013 默认 365 天） | 随 D-2 | |

---

## 四、功能级 BLOCKED（随前置解锁，当前无法决定）

| 编号 | 事项 | 解锁条件 |
|---|---|---|
| OI-006 | K-1 运行时 KV 落位 | K-2 导出边界清单 + M01 D2/D10 落位设计（台账前置） |
| OI-007 | reinforce 回传链路（写路径） | E14 落地 + B04 链路验证 |
| OI-008 | L0 跨用户作用域 | PC-6 核验（台账 I-3） |
| OI-009 | J4 情绪标签 15 词表内容 | 实施设计阶段 |
| OI-010 | 宿主真实接口（QQ Bridge/DSH 12 项 VERIFY） | P9 前实测 |
| OI-014 | U-3~U-7 长期悬置项 | 触发条件到达 |
| BL-003 | psy-model 引入 | 独立代码质量/安全审查 |

---

## 五、PROPOSED 工程默认一览（PD-001~014，对应最小决策集 D-2）

| PD | 内容 | 默认 |
|---|---|---|
| PD-001 | ID 格式 | UUID（版本不锁；UUIDv7 为方向，=OI-013） |
| PD-002 | SQLite 13 表结构 | 按 01 §7.16 建表建议（PROPOSED，P0 定稿） |
| PD-003 | journal_mode=WAL | 是 |
| PD-004 | document_id=`memory_id:version:space` | 是 |
| PD-005 | sensitivity 枚举 public/normal/private/restricted，默认 private | 是 |
| PD-006 | 权限矩阵全表（默认 DENY+显式 ALLOW） | 是 |
| PD-007 | revoke 默认=撤销可用性 | 是（=OI-005） |
| PD-008 | Outbox retry_max=8、退避基数 1s、dead_letter 后人工修复 | 是 |
| PD-009 | 冗余惩罚 λ=0.5 | 是 |
| PD-010 | Context 分区五段命名（DirectlyRelevant/StableFacts/HistoricalEvents/ActivePlans/RelevantBackground） | 是 |
| PD-011 | FTS5 external content table 挂接 memory_objects | 是 |
| PD-012 | health 端点暴露 outbox_backlog/oldest_age | 是 |
| PD-013 | 审计保留 365 天 | 是 |
| PD-014 | Benchmark 阈值留白待 P3 基线 | 是 |

---

## 六、程序性问题

| # | 问题 | 选项 |
|---|---|---|
| PR-1 | 独立审查子代理曾中断，已由总控接管完成（DESIGN-QUALITY-REVIEW） | A=接受总控版｜B=P9 前另派一次真独立复审（推荐） |
| PR-2 | 本清单裁决后的落盘方式 | 你回复「编号+选项」→ 总控更新 REGISTER/OPEN-ISSUES → 三处同步核验 → 在 CHANGELOG 记录（引用"用户裁决"） |

---

## 七、速查：一句话状态

- **P0 现在就能开工**（不依赖任何未决项；D-3 授权即可）。
- **P2 API 实现前**必须定 D-1（OI-016/017/018）。
- **P1 开工前**需要 D-2 里的 PD-002（表结构）批准（否则按 PROPOSED 实现并标注）。
- 其余全部不阻塞 P0–P3。

---

# 八、V2 架构升级新增待裁决（2026-11-04，OI-025~031）

> 来源：`DOCUMENT-CONFLICT-REGISTER-V2.md` §二。这 7 项是**新架构与既有台账裁决的真实冲突**，处理方式一律「保留原裁决 + 标记重新裁决」，设计上已给保守默认（不启用/不生效），**不裁决也能开工 P0–P3**。

| 编号 | 冲突 | 保守默认（当前生效） | 你的选项 | 总控推荐 |
|---|---|---|---|---|
| **OI-025** (CR-01) | `AUTO_ACCEPT` 候选态 vs「confirmed 仅 M04 通道写入」(D-AP-05) | AUTO_ACCEPT ≠ confirmed；仅 L1/L2 + owner_private，仍走 M04 整合通道 | A=维持现状（推荐）｜B=允许 AUTO_ACCEPT 直接确认（需改台账） | **A** |
| **OI-026** (CR-02) | 情绪检索 vs「首版关闭查询侧情绪感知」(J6) | `emotion.query_enabled=false`；返回 `FEATURE_DISABLED`/`EMOTION_DISABLED` | A=维持关闭（推荐，P5 后再议）｜B=现在启用 | **A** |
| **OI-027** (CR-03) | ΔE 影响当前状态 vs「禁止改 PAD/持久变更走审核」(X-3/CR-09/K-6) | `emotion.influence_enabled=false`；仅运行时、不落库、不由沉默触发 | A=维持关闭（推荐）｜B=启用并给上限参数 | **A** |
| **OI-028** (CR-04) | L1/L2 自动情绪标注 vs「只预留接口不实现 LLM 标注」(J7) | `emotion.auto_infer=false`；仅 USER_EXPLICIT | A=维持关闭（推荐）｜B=启用自动推断 | **A** |
| **OI-029** (CR-05) | Psychology 纳入架构 vs psy-model 暂缓 (I-7/X-6) | 设计已纳入、**实施 BLOCKED**；接口 DEFERRED | A=继续暂缓（推荐）｜B=解锁（需独立代码质量/安全审查） | **A** |
| **OI-030** (CR-06) | 原句长期保存 vs 数据最小化/导出边界 (K-2/D-AP-09) | Evidence 设 retention_policy；导出仍 BLOCKED；不得声称"不出本机" | A=先定保留期（推荐：给个数值）｜B=继续 BLOCKED | **A（并请给保留期）** |
| **OI-031** (CR-07) | `public` 可见性 vs 最小权限 | `permission.public_enabled=false`，V1 禁用 | A=维持禁用（推荐）｜B=启用 | **A** |

## 八.1 如果你只想做最小的一个决定

**全部选 A（维持保守默认）** 即可 —— 这 7 项都不会阻塞设计交付，也不阻塞 P0 开工；等 P3–P6 进入实施前再逐项解锁即可。

## 八.2 组 A 提交的两项技术跟进（已由总控处理）

| # | 跟进项 | 处理结果 |
|---|---|---|
| 1 | OI-025~031 未进 OPEN-ISSUES / PENDING-USER-DECISIONS | ✅ 已合并（OPEN-ISSUES §7 + 本节） |
| 2 | 02/03 中"模型层/ALG-07·08"与 10 号重复定义 | ✅ 已在 02 号 V2 对齐头声明"LLM 三层见 10 号"，避免双定义 |
| 3 | 11↔12（画像 basis）、09↔15（参数 ID）互引 | ✅ 抽检一致（basis 三值、参数 ID 点号式风格统一） |
## 八.3 组 C 上报的两项技术判断（总控已处理，供你知悉）

| # | 判断 | 总控处理 | 结论 |
|---|---|---|---|
| 1 | `read-response.items[].visibility` 枚举只有两值（owner_private/group_shared），不含 public | 采纳组 C 写法（schema 层拦截违规更可靠），并补权威枚举交叉引用（DD-AUTHORITY-V2 §2.10） | ✅ 通过，无需裁决 |
| 2 | `top_k` 一参两义（14 号的"每通道 50" vs 契约的"最终注入 20"） | **已修 14 号**：改为 `opts.channel_top_k`(50，每通道) 与 `top_k`(20，最终注入)，并登记 OI-14-06 | ✅ 已修复（这正是 OI-023 同类问题，已防复发） |