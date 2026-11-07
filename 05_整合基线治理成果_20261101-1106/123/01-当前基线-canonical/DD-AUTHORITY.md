# DD-AUTHORITY — 设计基准与冲突裁决（单一事实来源）

> 文件代号：DD-AUTHORITY ｜ 版本：1.0 ｜ 状态：APPROVED（用户 2026-11-04 指令）
> 本文件是本次"Memory Core R4.1 全量详细设计"任务的**单一事实来源**。所有设计文档（00–08）、契约、状态机、登记表必须与本文件一致。本文件由总控 Agent 依据用户指令与原始资料生成，其他 Agent 不得修改本文件。

---

## 1. 任务定义（用户 2026-11-04 指令原文要点）

将《MEMORY CORE R4.1 完整统一架构母本》（m.md，总则）与《sf.md》（最新规）展开为可直接指导工程实现的完整详细设计：

1. 生成 00–08 九份详细设计。
2. 生成完整 JSON Schema。
3. 生成 API 请求与响应契约。
4. 生成事件协议与错误码。
5. 生成全部核心状态机。
6. 生成配置 Schema。
7. 生成模块依赖关系图。
8. 生成完整数据流图。
9. 生成算法伪代码和边界处理规则。
10. 生成测试用例和验收标准。
11. 生成 DESIGN-DECISION-REGISTER.md。
12. 生成 OPEN-ISSUES.md。
13. 完成跨文档一致性检查。
14. 完成独立审查。

**交付性质：设计 + 契约 + 审查报告。本任务不授予代码实施授权（sf 后文 §10/§13.4：P0–P3 实施范围仍为 M01+M02；设计完成 ≠ 实施授权）。**

---

## 2. 输入资料与权威顺序

### 2.1 资料位置

| 资料 | 路径（备份副本，与桌面原件逐字节一致） |
|---|---|
| m.md（母本总则） | `E:\[PROJECT]\work-交付包\design\_sources\m.md`（2156 行） |
| sf.md（最新规，8081 行） | `E:\[PROJECT]\work-交付包\design\_sources\sf.md` |
| 正式裁决台账（唯一权威） | `C:\Users\[USER]\Desktop\work\08-adjudication-ledger.md` |
| M01/M02 冻结设计 | `C:\Users\[USER]\Desktop\work\03-m01.md`、`03-m02.md` |
| M03/M04/M05 设计（只读接口章节） | `C:\Users\[USER]\Desktop\work\03-m03.md`、`03-m04.md`、`03-m05.md` |
| 交接文档 | `C:\Users\[USER]\Desktop\work\04-HANDOFF-20261103-R4.1-CR08A.md` |
| 环境与验收 | `C:\Users\[USER]\Desktop\work\05-环境配置总索引.md`、`06-最小记忆闭环验收标准.md` |
| 数据样例（A/B 层真实格式） | `C:\Users\[USER]\Desktop\work\data-samples\` |
| 其他 | `03-m01-problem.md`、`09-05-conflict-register.md`、`10-06-interface-map.md`、`11-ALGORITHM-REVIEW-SCOPE.md` |

### 2.2 权威顺序（MUST 遵守）

1. **第一层：正式裁决** — `08 adjudication-ledger.md` 是用户裁决唯一权威。
2. **第二层：已批准设计文件** — V1 Delivery Charter、03-m01/m02 冻结稿、正式交付/环境索引、已批准验收规范。
3. **第三层：任务书时间序**（用户 2026-11-04 明确规则）——**sf.md 文本先后即时间先后，后文优先于前文；sf 整体新于 m.md；m.md 为总则**。即：`sf 后文 > sf 前文 > m.md`。
4. **第四层：Agent 技术建议** — 仅补未定设计，一律标 PROPOSED，无批准效力。

### 2.3 sf.md 内部结构（按时间序）

| sf 行号段 | 文件 | 效力 |
|---|---|---|
| L1–1533 | 五大核心算法实施规格 | 有效（≈ 交付包 14 号文件） |
| L1534–1984 | 12 框架地图与文档体系论证 | 有效（背景） |
| L1986–3875 | 完整目标架构与实施基线 | 有效（≈ 交付包 15 号文件，但被后文部分更新） |
| L3878–6731 | **《完整技术规格与 Agent 实施约束总规范》** | **最新约束规范，与前文冲突时以此为准** |
| L6732–8081 | **《MC-R41-DESIGN-FREEZE-01》** | **本任务直接任务书，最新** |

---

## 3. 已裁决关键约束（引用台账，设计必须服从）

以下均为台账"已决定（用户）"，引用格式 = 台账小节：

| # | 约束 | 台账来源 |
|---|---|---|
| A1 | 五模块边界维持；M03 候选提取与 M04 整合裁决不可合并 | D-AP-01（§3） |
| A2 | m01/m02 冻结稿保持冻结；M04 §6.2 为唯一 LLM Gateway 入口；Embedding Provider 归 M01 | D-AP-02/10（§3/§3b） |
| A3 | confirmed 仅允许 M04 审核通道写入；last_retrieved 按命中规则写入且不得作为衰减 v2 正向权重 | D-AP-05（§3b） |
| A4 | **reinforce 仅由 DSH 回传最终 prompt 中实际出现的记忆 ID 触发；E14 未落地、B04 回传链路未确认前只预留 schema，禁止写入** | D-AP-05/J1/K-4（§3b/§4.4） |
| A5 | 有规范摘要时索引主体="摘要+cue"，无摘要须定义回退策略 | D-AP-04（§3b） |
| A6 | 生活化测试集为主要验收依据；研究报告收益不得写成实测结果 | D-AP-06（§3b） |
| A7 | C40 权限引擎不可插拔；五级可插拔分类接受；trusted_context_source: entry-only | D-AP-07（§3b） |
| A8 | 拒绝第三方托管记忆平台；不得声称"记忆文本不出本机"；本地持久化与云端推理传输分开评估 | D-AP-09（§3b） |
| A9 | 心理层：L0+L2 起步、五层 schema 预留；影子模式最低门禁；心理状态机只改运行时状态、不直接产生记忆候选 | I-3/I-4/I-5（§3c） |
| A10 | **psy-model 暂不引入**（I-7）；五方法/十二阶段接口形态暂缓（X-6）——相关接口一律标 DEFERRED | I-7/X-6（§3c/§3g） |
| A11 | J2：M03 写时情绪标注，检索期不调用 LLM；J4：15 个受控标签配置管理；J6：首版关闭查询侧情绪感知；J7：只预留接口不实现 LLM 标注；J8：预留 affect_source='external' 字段不启用 | §3e |
| A12 | **K-6：openLoop 只能由明确事件触发，永不由沉默触发；禁止催促、禁止与信任度联动**（CR-09 门禁未验证前不闭环） | K-6/CR-09（§3f/§3g） |
| A13 | K-1 有条件接受（M01 内非权威运行时 KV），**待 K-2 导出边界清单 + M01 D2/D10 落位设计完成后才生效**——设计中只能标 PROPOSED | K-1/K-2（§3f/§4.3） |
| A14 | **U-1/U-2：不在设计阶段预先冻结 cache.sqlite 表结构与校验实现**——cache 相关只写"暂缓至实施阶段"，不得给出表结构 | U-1/U-2（§3e） |
| A15 | X-1：心理加成 delta-only 影子验证，不改变 M02 最终排序；X-3：Recall 结果可参与推理、禁止直接改 PAD、持久变更走独立审核（③默认关闭）；X-2：Session 结束丢弃短期情绪、L1 仅预留 | X-1/X-2/X-3（§3g） |
| A16 | 参数可观测规范（11 属性/三层次）+ 稳定唯一可追踪参数 ID 规范（优化-5） | §3d/§3e |
| A17 | Token 计量须能区分系统提示词/聊天历史/记忆检索/实际注入/模块调用消耗（优化-1） | §3e |
| A18 | Graphiti 旁路索引仅登记不启动（优化-3） | §3e |

---

## 4. 时间序冲突裁决（总控依"sf 后文 > sf 前文 > m.md"裁定）

以下冲突已由总控裁决，所有文档直接采用"裁决结果"，不得再自行取舍：

| # | 冲突点 | 裁决 | 依据 |
|---|---|---|---|
| R1 | **错误码集**：m.md §15.5（INVALID_ARGUMENT/UNAUTHORIZED/STORAGE_UNAVAILABLE…14 个）vs sf 后文 §4.5（VALIDATION_ERROR/PERMISSION_DENIED/STORAGE_ERROR…11 个） | **采用 sf 后文 §4.5 为规范名**；m.md 名称作为兼容别名登记于 error-codes.md 映射表；并按 sf 后文 §4.4 扩展属性（code/category/message/retryable/severity/handling/exposure_policy） | sf 后文更新 |
| R2 | **Outbox 事件类型名**：sf 前文/m（CREATE/UPDATE/DELETE/REINDEX）vs sf 后文 §3.1.6（memory_upsert/memory_delete/memory_reindex） | **规范名 = memory_upsert / memory_delete / memory_reindex**；CREATE/UPDATE/DELETE/REINDEX 登记为历史别名，契约中 mapping 字段兼容 | sf 后文更新 |
| R3 | **逻辑平面命名**：sf 前文 §0.3（记忆处理/持久化/检索与上下文/治理/集成与运行）vs sf 后文 §1.2（数据/检索/认知/治理/协同扩展） | **采用 sf 后文五平面**（数据/检索/认知/治理/协同扩展）；sf 前文版本在 00 文档中作为历史提法一笔带过 | sf 后文更新 |
| R4 | **三组件架构**：m.md §1.2 Memory Core + Memory API + Host Adapter | **保留，组件视图与 sf 后文五平面并存**（组件=部署边界，平面=逻辑职责）；用户约束明确"Memory Core、Memory API、Host Adapter 三部分边界不得改变" | 用户指令 + m 总则 |
| R5 | **ID 格式**：m.md 第二十二（UUIDv7）vs sf 后文 §2.3.1（"ID 生成算法由 Schema/运行协议正式裁决"） | **Schema 采用 UUIDv7 格式约束（format: uuid）作为 PROPOSED 默认**，登记 DESIGN-DECISION-REGISTER（status=PROPOSED，来源 m.md 第二十二），状态机/契约中 id 字段 format 用 "uuid" 不锁版本 | sf 后文更保守 |
| R6 | **时间语义**：m.md（UTC ISO 8601）vs sf 后文 §2.4（created/updated/valid_from/valid_to/observed_at/recorded_at 六字段、事实时间≠记录时间、UTC 存储） | **采用 sf 后文六字段语义**；格式 RFC 3339 / UTC ISO 8601（两者一致） | sf 后文更细 |
| R7 | **Outbox 状态机**：m.md §7.5（pending/processing/succeeded/retry_wait/dead_letter）vs sf 后文 §4.6（任务状态 pending/running/succeeded/failed/cancelled） | **两者是不同对象**：Outbox 事件状态机用 m.md 五态（+ dead_letter 细则）；通用异步任务状态机用 sf 后文五态。两份状态机文件并存，不合并 | 各自对象不同 |
| R8 | **REST API 面**：仅 m.md §15.2 定义了 `/api/v1/...` 端点 | **保留为 API 设计基线（PROPOSED）**，按 sf 后文 §4.2 要求补全请求/响应 Schema 与行为契约 | m 唯一定义，sf 不冲突 |
| R9 | **Embedding 维度 1024** | 现网参考值，**不得写死**；Schema 中 dimension 为必填整数 ≥1，具体值来自 EmbeddingProfile | 一致，重申 |
| R10 | **规则排序公式**：sf 前文六特征（αRRF+βE+γN+δT+εS−ζP）vs m.md 五特征（0.55R+0.15E+0.10T+0.10C+0.10Q） | **采用 sf 前文六特征公式（含 duplicate_penalty ζ）**；m.md 五特征登记为简化别名。权重全部=实验初值 | sf 前文更新于 m；sf 后文未改公式仅加归一化约束 |

---

## 5. 最高优先级硬约束（用户 2026-11-04 指令，逐条生效）

1. 必须读取正式 adjudication ledger；已有用户裁决优先于母本一般默认建议。
2. 保留原有 A 层向量语义记忆与 B 层 SocialV2 记忆（语义保留，迁移见 00/01/07）。
3. DSH 历史是记忆来源，不得认定为第三个权威记忆存储层。
4. Memory Core / Memory API / Host Adapter 三部分边界不得改变。
5. SQLite 是唯一权威业务存储；FTS5/LanceDB 是派生索引。
6. LanceDB 使用 FLAT 精确检索，不得静默替换（P0 须真实验证 SDK 执行路径，否则标 BLOCKED）。
7. M01–M05 职责不得混淆。
8. M02 必须执行 M05 权限终检契约（默认 DENY；deny/error/timeout/未知一律拒绝；生产禁止 allow-all）。
9. 所有派生数据必须继承来源权限（多来源取最严格交集；无法确认即拒绝）。
10. 所有正式记忆必须保留 Evidence 和 Provenance。
11. 不得把模拟数据、推测能力、未验证接口写成已实现（区分 PASS/FAIL/BLOCKED/NOT RUN/MOCKED；F2/F3 ≠ F4）。
12. 不得把建议写成用户批准（PROPOSED/TBD/VERIFY/BLOCKED/DEFERRED 不得写成 APPROVED）。
13. 不得擅自扩大 P0–P3 实施授权（设计可先行，FUTURE/DEFERRED 标注）。
14. 不得接触 D:\qq-bridge 的实际源码、配置、state 或 Persona。
15. 不得启动正式机器人服务。
16. 普通实现细节按母本采用明确默认方案，不反复询问用户；遇正式裁决冲突/不可逆数据风险/权限风险/客观技术阻塞才停止并报告。
17. **每次修改先备份**（本次为全新文件，不覆盖任何既有正式文件；_sources 已备份）。

---

## 6. sf 后文《总规范》强制条款摘要（各文档必须内化）

- **规范用语**：MUST / MUST NOT / SHOULD / SHOULD NOT / MAY / TBD / VERIFY / BLOCKED / FUTURE（§0.2），文档中使用这些用语。
- **设计状态**：APPROVED / PROPOSED / VERIFIED / REJECTED / DEFERRED / TBD / BLOCKED（§0.3）；禁止把 PROPOSED/TBD/DEFERRED 解释为 APPROVED；禁止把代码已实现解释为设计已批准。
- **架构不变量 12 条**（§1.4）：SQLite 唯一权威；FTS5/LanceDB 非权威；索引可重建；权威恢复不依赖索引；最终读取回权威源；最终读取过权限终检；派生继承权限；写入必有来源与事务；索引不覆盖权威；不静默降级；不跨职责边界改数据。
- **数据访问接口**（§3.1.4）：createMemory/getMemory/getMemoryVersion/queryMemories/updateMemory(memoryId,patch,expectedVersion)/deleteMemory/listMemoryVersions——所有输入 Schema 校验、更新带预期版本（乐观并发）、事务写入、禁止任意 SQL。
- **事务五步**（§3.1.5）：Validate→Check expected version→Write memory→Write memory version→Write outbox→Commit；禁止 Memory 已提交但 Outbox 未写入（及其反向）。
- **Outbox 处理规则 10 条**（§3.1.6）与**删除传播 7 步**。
- **一致性四分**（§3.1.7）：Authority/Index/Version/Permission Consistency。
- **M02 十四节**（§3.2.1–3.2.14）：含查询规范化禁令（不得删否定词/不得扩权限）、FLAT 八项验证、RRF 六要求、排序特征七要素（输入来源/数值范围/缺失值/方向/归一化/权重/计算版本）、权威回源九步、权限终检、Token 计量、Context Builder 禁令、Event-L3 五分（Retrieved/Authorized/Injected/Model Used/Answer Correct）。
- **M03/M04/M05 = FUTURE**（§3.3–3.5 标注），设计先行、P0–P3 不实施；M05 的 M02 契约必须预留。
- **运行协议**（§4.1–4.8）：模块通信禁令、事件八问、幂等、乐观并发、错误协议、异步任务字段、Worker 八项、配置八类。
- **权限十原则**（§5.1）与读写/共享/导出/审计细则。
- **故障矩阵**（§6.2）：SQLite/FTS5/LanceDB/Embedding 四类故障处置。
- **测试九层级**（§7.1）与 M01 三大关键测试（§7.2：事务回滚/旧版本事件不覆盖/删除幂等）、禁止虚假验收七条（§7.7）。
- **迁移**（§8）：Schema 六要素、Embedding 九步流程、四禁令、兼容性六版本、回滚六明确。
- **集成**（§9）：身份五标识、会话隔离四禁令、宿主调用链、返回八字段、注入边界、压缩协作。
- **P0 技术验证五项**（§16）：SQLite Driver、LanceDB FLAT、B-010 八字段、B-011 九步、SQLite/LanceDB 边界（禁止 SQLite Vector Adapter 静默替代）。
- **完成等级 F1–F4**（§17/§九）：本次设计目标 **F2 全部达到、尽可能 F3**；禁止把 F2/F3 写成 F4。
- **冻结与变更**（§18）：冻结清单十项；变更申请十要素。
- **参数登记**（§15）：15 个参数全部为"实验初值"，正式化前必须可配置。
- **29 项最终验收**（§十四）：关键 TBD（数据安全/权限隔离/数据一致性/核心接口）未解决不得宣布全部完成——如有则明确列阻塞项。

---

## 7. 交付目录与文件清单（写入根目录）

```
C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\
├── README.md                        （总控写）
├── DESIGN-STATUS.md                 （总控写）
├── DESIGN-DECISION-REGISTER.md      （总控写）
├── OPEN-ISSUES.md                   （总控写）
├── CHANGELOG.md                     （总控写）
├── 00-总体架构与设计原则.md
├── 01-记忆本体与数据模型.md
├── 02-核心算法规范.md
├── 03-系统运行与协作协议.md
├── 04-权限与治理规范.md
├── 05-可靠性与运维规范.md
├── 06-测试与评估规范.md
├── 07-扩展与演进规范.md
├── 08-DSH与QQBridge集成规范.md
├── contracts/
│   ├── schemas/  memory-object / evidence / candidate / observation / relation /
│   │             search-document / embedding-space / outbox-event /
│   │             permission-context / retrieval-request / retrieval-response /
│   │             context-response  （12 × .schema.json）
│   ├── api/      m01-api.md m02-api.md m03-api.md m04-api.md m05-api.md
│   │             context-adapter-api.md permission-api.md database-adapter-api.md
│   ├── state-machines/  memory-lifecycle.md validation-state.md availability-state.md
│   │             outbox-state.md worker-state.md migration-state.md
│   │             deletion-state.md embedding-migration-state.md
│   ├── config/   core / storage / retrieval / embedding / security / worker / test （7 × .schema.json）
│   └── error-codes.md
├── diagrams/   （15 张 Mermaid，与正文一致）
├── test-specs/ （06 配套测试规格）
├── reviews/    CROSS-DOCUMENT-CONSISTENCY-REPORT.md / DESIGN-QUALITY-REVIEW.md /
│               ADJUDICATION-COMPLIANCE-REPORT.md / IMPLEMENTATION-READINESS-REPORT.md
└── _sources/   m.md、sf.md（备份副本，只读）
```

母本同步：`E:\[PROJECT]\work-交付包\Memory-Core-R4.1-Design\`；备份：`E:\[PROJECT]\备份全集\Memory-Core-R4.1-Design-20261104\`（交付后总控统一同步三处并逐字节核验）。

---

## 8. 每份设计文档的强制结构（sf 后文 §19）

每份 00–08 文档必须包含以下小节（顺序可调整、不可缺失）：

1. 文档状态（含 F 等级自评）
2. 适用版本（R4.1-MASTER-01 / DD-AUTHORITY 1.0）
3. 适用范围（含"FUTURE/不实施"边界）
4. 依赖文档
5. 正式裁决引用（逐条引台账编号）
6. 术语表
7. 规范正文（MUST/SHOULD 用语）
8. 接口或数据结构
9. 流程图（Mermaid，与正文一致）
10. 异常情况
11. 安全边界
12. 测试要求
13. 验收标准
14. 未决事项（编号 OI-xx，与 OPEN-ISSUES.md 对齐）
15. 变更记录

**写作硬规则**：
- 中文；Mermaid 图必须可渲染且与正文一致（禁止图有正文无）。
- 所有评分/公式必须给出：数学表达式、变量定义、数值范围、缺失值处理、归一化方式（禁止"综合考虑"式模糊描述）。
- 所有接口必须写：名称/目的/输入/输出/必填/可选/类型/默认值/错误码/幂等/权限/超时/重试/版本/是否当前实施范围。
- 禁止编造 SDK 能力、API、测试结果、历史裁决；不确定 → TBD/VERIFY/BLOCKED。
- 参数一律标"实验初值（EXP-DEFAULT）"，不得写成已裁决。
- P0–P3 范围 = M01+M02；M03/M04/M05/Dream/Psychology/Persona/Session/Context Compression = FUTURE 设计。
- A/B 层迁移：缺失信息不得编造（m.md §2.1）；B 层 memberImpression 保留推断性质（m.md §2.1）。
