# E 路 · 模块独立性与调整成本审查（Memory Core R4.1）

## STATUS: COMPLETE

## 章节目录（计划）
- §1 Headline（3 行）
- §2 六张矩阵表
  - 表1 模块职责矩阵
  - 表2 模块调用依赖矩阵
  - 表3 数据所有权矩阵
  - 表4 模块调整影响矩阵
  - 表5 配置与参数归属表
  - 表6 共享合同依赖表
- §3 十四个变更场景逐一判定
- §4 十问必答
- §5 模块依赖 ASCII 图
- §6 隐藏耦合清单
- §7 改善方向（NEW PROPOSAL，逐条对应 LOW 项）
- §8 结论
- §9 方法附录

---

## §1 Headline

- **结论先行**：R4.1 的"调整成本框架"（07 号迁移/版本/回滚/算法替换、15 号参数注册表、contracts 八维版本）设计完整度在文档层是 E 路中最高的一块，但这是**围绕 M01+M02 主链**建的；M03/M04/M05 仍是 FUTURE/未授权（00 §7.7、DD-V2 §10 P3–P5），其 03-m0*.md 为 R3 基线且接口多为"示例/未冻结"，故"五模块可独立调整"目前是 DESIGN COMPLETENESS 而非已落地能力。
- **真正的耦合点不在模块数**，而在共享合同（memory-unit 28 全必填字段、relation、anchor-set、person-profile）+ 单一权威（SQLite/Outbox/FTS5/LanceDB 全归 M01）+ 横切 owner（LLM Gateway 归 M04、Embedding 归 M01、权限归 M05）：改语义类型/关系/画像结构 = 跨 M01–M05 + 多 schema + 迁移 + 全链路重测（P1/P0 级）。
- **可调整性分级**：纯数值/权重/预算类（RRF k、衰减、Top-K、1200 token）≈ CONFIG_ONLY（MEDIUM–HIGH）；换 Provider/向量库/分词器/改枚举/改画像结构/新增 Dream = MULTI_MODULE_CHANGE 或 ARCHITECTURE_CHANGE（LOW）；当前无"新增模块而不重写核心"的插槽设计（D-AP-03 明令不新增第六模块）。

> 三分类标注：**EXISTING DESIGN**（给文件+章节）/ **DESIGN COMPLETENESS**（输入/输出/步骤/条件/模块责任/边界是否足够）/ **RECOMMENDED ADDITION**（一律标 **NEW PROPOSAL**，非已批准）。

---

## §2 六张矩阵表

### 表1 模块职责矩阵（M01–M05 × 数据所有权/业务决策权/写入权/读取权/修改权/禁止行为）

> 证据链：00 §7.3 依赖矩阵、§7.6 职责边界（含"不负责"清单）、§7.9 不变量 INV-01/06/07/12；03 §7.2 通信禁令、§7.4 M01 七函数、§7.5 六动作；03-m01 §1.1/§7.1/§9；03-m02 §1.1；03-m03 §1/§9；03-m04 §1.1/§1.2/§12；03-m05 §1/§2.1/§10.3。**DESIGN COMPLETENESS：职责文字层面完整（每个模块都有"负责/不负责"），但"写入权/修改权"仅对 M01 落到具体 API，M02–M05 多为逻辑名 + FUTURE 标注，边界足够判定、不足以实施。**

| 模块 | 数据所有权（唯一） | 业务决策权 | 写入权 | 读取权 | 修改权 | 禁止行为（硬约束） |
|---|---|---|---|---|---|---|
| **M01** Storage | SQLite 权威表、Memory/Version/Evidence/Relation/Observation/Outbox、FTS5+LanceDB 索引、备份、schema_migration（00 §7.3/§7.6；03-m01 §2.1/§7.1） | 无（"事实真实性判断、自动合并、保留决策"明确不归 M01，00 §7.6） | **唯一权威写者**：createMemory/updateMemory/deleteMemory + 同事务 outbox（03 §7.4 七函数） | 自身全部表 | 版本/删除/迁移/重建（D7/D9/D23） | 业务模块直拼 SQL、双写索引、索引覆盖权威、跨职责改他模块数据（00 §7.3 规则1–5、INV-10/12） |
| **M02** Retrieval | 无权威数据；只持有检索请求/响应、缓存、过程记录（00 §7.6） | 排序、去重、RRF 融合、Token Budget、Context 组装、扩展一跳的预算/方向（00 §7.6；03-m04 §12.4 明确"预算/是否扩展/去重"归 M02） | **禁止写入正式记忆**；仅允许按命中规则写 `last_retrieved`，reinforce 禁止（00 §7.6 跨模块红线 2/3） | 经 M01 Storage API 回源、经 Index Adapter 读 FTS5/LanceDB（**MUST NOT 直连**，00 §7.3） | 无（不改事实/不新建正式关系） | 绕过 M05 终检、直连 SQLite/LanceDB/FTS5、因命中提升置信度、创建正式关系（00 §7.6、03-m02 §1.1） |
| **M03** Extraction | 候选（Candidate）、候选评分/分流结果（03-m03 §10 候选数据结构） | 候选价值初评、候选分流、候选生成（03-m03 §9） | **不写正式记忆**；产出 Candidate → 交 M04（03-m03 §1/§9；00 §7.6） | 当前批次消息；不主动加载全历史（03-m03 §2.3） | 无 | 直接写 active、覆盖正式记忆、自动提升 Scope、来源不明构造证据、绕过 M04/M01（00 §7.6；03-m03 §9） |
| **M04** Consolidation | Observation、当前视图/视图版本、Relation 语义、PersonProfile 派生视图、任务/检查点（03-m04 §1.1/§12；DD-V2 §3.5） | 去重/冲突/合并/修订、Observation 生成、稳定性判断、版本发布（00 §7.6） | **不拥有 DB 写权**：经 M01 事务写入（00 §7.6 跨模块红线 5；03-m04 §1.1"底层删除执行由 M01"）；LLM Gateway 唯一入口在 M04 §6.2（00 §7.3 规则3） | 经 M01 读正式数据、接收 M03 候选 | 视图/Observation/Relation 语义修订（经 M01 落库） | 无依据确认事实、删除证据、越权改 Scope/权限、无限制批量重写、多跳遍历（归 M02）、另建 LLM 通道（00 §7.3/§7.6；03-m04 §12.4） |
| **M05** Permission | 权限记录持久化归 M01，但**权限决策权**唯一归 M05（03-m05 §10.3；00 §7.6） | 唯一权限判定 canRead/Write/Update/Delete/Share/Export、授权生命周期、撤销（00 §7.6；03 §7.5 六动作） | 判定结果/审计；权限数据由 M01 持久化（03-m05 §10.3） | 读权限元数据（M01，00 §7.3 M05→M01=R） | 授权/撤销/策略版本（00 §7.6） | 输出"默认允许"、被任何模块绕过、发明权限外的业务判断（00 §7.6；03-m05 §10.3） |

**判定：LOW（"职责边界"文字完备 / "写入权与修改权"未落到可冻结接口）**。00 §7.6 的核心接口列仍标 FUTURE；03-m01 §7.1 明写"M02、M03、M04 的具体接口尚未冻结"；03-m04 §12.4 明写"名称仅作设计示例，不代表最终 API 已冻结"；03-m04 §12.1–12.3 输入输出为方向示意。**RECOMMENDED ADDITION（NEW PROPOSAL）**：为 M02–M05 各补一份"接口冻结表"（输入字段/输出字段/错误码/幂等键/版本契约/超时），并把 00 §7.3 的 A/R/C 依赖矩阵升级为可机读契约清单（见 §7 改善方向 E-1）。

---

### 表2 模块调用依赖矩阵（行=调用方，列=被调用方；A=同步允许/R=只读/C=契约预留/E=事件/—=禁止）

> **EXISTING DESIGN**：00 §7.3 给出原矩阵（含 SQLite/FTS5-LanceDB/Outbox/Worker 列），03 §7.2 给出线性调用链，03-m04 §1.2/§12.4 与 03-m05 §10.3 给出 M04/M05 侧关系。下表把二者合并为纯 M01–M05 视图，并标出被文档违反/未闭合处。

| 调用方＼被调用方 | M01 | M02 | M03 | M04 | M05 | 外部/基础设施（所有者） |
|---|---|---|---|---|---|---|
| **M01** | — | — | — | — | A 判权（00 §7.3） | 唯一写 SQLite / 经 Index Adapter 写 FTS5·LanceDB / 写 Outbox（00 §7.3） |
| **M02** | R 权威回源（00 §7.3、03 §7.2） | — | — | C 关系一跳（FUTURE，00 §7.3；03-m04 §12.4 补"实际由 M04 提供一跳"） | A 权限终检（00 §7.3/§7.6 红线1） | R FTS5/LanceDB（**禁止直连** SQLite，00 §7.3、03-m02 §1.1） |
| **M03** | —（经 M04→M01 落库，00 §7.3） | — | — | A 提交候选（00 §7.3；03-m03 §9） | A 判权（00 §7.3） | 抽象 LLM Provider（03-m03 §12："通过抽象 LLM Provider 调用"） |
| **M04** | A 事务写入（00 §7.3；03-m04 §1.2） | —（M02 请求 M04，反向 C） | 接收候选（A/C） | — | A 判权（00 §7.3；03-m04 §1.2） | **唯一 LLM Gateway 入口 = M04 §6.2**（00 §7.3 规则3）；Model Gateway/Bridge（03-m04 §1.2，外部） |
| **M05** | R 读权限元数据（00 §7.3） | — | — | — | — | —（不依赖他模块决策） |
| **Worker**（非 M 模块） | A 回调同步结果（00 §7.3） | — | — | — | — | E 消费 Outbox → 写 FTS5/LanceDB（03 §7.2/§7.7；单一所有者 M01 的 Index Adapter） |
| **Dream / Persona / Context Adapter**（FUTURE） | C / — / — | C / C 授权视图 / — | C | C | C | 00 §7.7：Dream 无独立 DB 写权限；Persona 直连 SQLite 被禁 |

**单向依赖冲突（隐藏强耦合证据，详见 §6）**：
1. 00 §7.3 把 M02→M04 标为 **C（FUTURE）**，但 03-m04 §12.4 与 03-m02 §1.1 已把"关系读取经 M04 一跳查询"写成 M02 的必经路径 → 契约状态与调用事实不一致。
2. 00 §7.3 规则3「LLM Gateway 唯一入口 = M04 §6.2」，但 **M03 §12 写"通过抽象 LLM Provider 调用模型，不直接依赖 DSH"**，且 DD-V2 §5 的 16 阶段含 LLMExtractor/EmotionExtractor（属 M03/P3）→ 抽取侧 LLM 通道与"唯一入口"存在第二通道歧义。
3. 00 §7.3 规则5「横切能力（配置、日志、审计、任务调度）MUST 有唯一 owner，禁止新增进程/第六模块」，但 03-m04 §1.1 第 10 项「协调后台任务和模型调用」+ §7 统一任务队列/§7.6 任务 DAG 把**任务调度**放在 M04；03 §7.7 又把 Worker/Outbox 消费放 M01 → 调度 owner 疑似双重。
4. Outbox/FTS5/LanceDB 单一所有者明确（M01 + Worker 消费），但 00 §7.3 允许 Worker 对 Outbox「R/W」、对 FTS5/LanceDB「A 写索引」，而 03 §7.8 事务边界表把索引写入明确为"事务提交后异步"——**E（事件）而非 A（同步）**，两种语义需统一。

**判定：MEDIUM**（矩阵本身存在且细节丰富，上述 4 处不一致属 DESIGN COMPLETENESS 缺口，非"无设计"）。

---

### 表3 数据所有权矩阵（对象 × 唯一写入者 × 读取者 × 是否共享合同）

> 证据：00 §7.6/§7.9（INV-01/06/09）、03 §7.4 七函数/§7.6 事件/§7.8 事务边界、DD-V2 §3.1–3.5 五核心对象、contracts/schemas（20 个，字段数见 §9 附录）、03-m01 §2.2/§9、03-m04 §2.1 三层。

| 对象 | 唯一写入者（权威） | 读取者 | 共享合同 / 字段数 | 判定 |
|---|---|---|---|---|
| **MemoryUnit / Memory 正式记忆** | M01（createMemory/updateMemory/deleteMemory，03 §7.4） | M01、M02（回源）、M04（整合）、M05（权限元数据）、Host/DSH（经 Memory API） | `memory-unit.schema.json` **28 字段全 required**；另有 `memory-object.schema.json` **29 字段/15 required**（两套并存，见 §6 H-1）；`read-response`/`context-response` 再镜像 semantic_type/anchor/confidence/… | **HIGH 耦合**：几乎全模块 + 至少 4 个 schema 依赖 |
| **MemoryVersion** | M01（同事务，03 §7.4 五步） | M01、M04（expected_version）、M02（getMemoryVersion） | 无独立 schema；随 memory 行；`schema_version`/`version` 在多个 schema 重复 | MEDIUM |
| **Evidence** | M01（DD-V2 §3.2；03-m01 §2.2） | M01、M03（引用）、M04（basis）、M02（证据扩展，逐条经 M05） | `evidence.schema.json` 21 字段/7 required；`memory-unit.evidence_refs`、`observation.source_evidence_ids`、`person-profile.source_evidence_ids`、`candidate.evidence_refs` | **HIGH**：4+ 合同引用 |
| **Event** | M01（DD-V2 §3.3） | M01、M02（事件查询）、M04（时间线）、EmotionContext 关联 | `event.schema.json` 15 字段全 required；`memory-unit.event_refs`、`read-response.items` | MEDIUM |
| **Relation** | M01 持久化，M04 负责语义（03-m01 §9.2"M04 负责语义关系，M01 保存"） | M04（listNeighbors 一跳，§12.4）、M02（扩展与去重）、Host（getRelated） | `relation.schema.json` 10 字段/7 required，端点仅 `source_memory_id`/`target_memory_id`（印证 B 路 B-C001…）；`memory-unit.relation_refs` | **HIGH**（改 relation_type = 枚举 + 3 模块） |
| **Observation** | M04 生成/修订，M01 持久化（03-m01 §9.2、03-m04 §1.1） | M04、M02（当前视图）、M01 | `observation.schema.json` 14 字段/11 required | MEDIUM（M04 FUTURE） |
| **PersonProfile** | M04 派生视图，`profile.write_channel=M04_only`（config/profile.schema.json） | M02（getProfile）、Host（memory.getProfile）、M04 | `person-profile.schema.json` 10 字段全 required（section/basis/source_memory_ids…）；config/profile.schema.json 11 section 枚举；read-response.profile；context-response.sections_present | **HIGH**：改结构 = 跨 4 处 |
| **Candidate** | M03 产出，非权威（03-m03 §10） | M03、M04（processCandidates，§12.1） | `candidate.schema.json` 21 字段/7 required | LOW（FUTURE，接口示例） |
| **EmotionContext** | M01 持久化（DD-V2 §3.4） | M02（默认不注入）、M04、Psychology（FUTURE） | `emotion-context.schema.json` 14 字段全 required；config/emotion.schema.json 15 标签词表 | MEDIUM（3 开关 DISABLED） |
| **SearchDocument** | M01 Index Adapter（派生可重建，07 §7.7） | M02 检索、Worker 重建 | `search-document.schema.json` 21 字段/14 required；含 `embedding_space_id` | MEDIUM（可重建 = 低迁移成本） |
| **Outbox Event** | M01（同事务，03 §7.6/§7.8） | Worker（唯一消费者，03 §7.7） | `outbox-event.schema.json` 13 字段/7 required；payload_version 消费者须支持 N 与 N-1（07 §7.1） | **HIGH 单点**：改 payload = Worker + 事件版本 + dead_letter |
| **FTS5 / LanceDB 索引** | M01 Index Adapter（**单一所有者**，00 §7.3/INV-02/03；Worker 消费 Outbox 触发） | M02（只读）、Worker（写） | `embedding-space.schema.json` 15 字段/12 required（B-010 八字段）；config/embedding、config/storage | MEDIUM（重建即可，07 §7.12） |
| **权限记录 / 授权 / 撤销** | M01 持久化，**M05 决策**（03-m05 §10.3） | 全部读写路径（00 §7.6 红线1）、M02 终检 | `permission-context.schema.json` 16 字段/4 required；config/security.schema.json；`context-response.permission_trace` | **HIGH**：C40 不可插拔（07 §7.10 明确排除） |
| **配置 / 参数注册表** | 配置管理员 + 15 号注册表（A10 写方） | 全部模块（A09 读方，15 §7.4–7.12） | 11 个 config schema + 42 参数（15 §7.16） | **HIGH 横切**（详见表5） |

**判定：LOW–MEDIUM**。所有权规则（M01 唯一权威、派生继承权限、索引不覆盖权威）是 **EXISTING DESIGN** 且写得极完整（INV-01…12）；但"唯一写入者"在 Relation/Observation/PersonProfile 上是 **M04 决策 + M01 落库** 的双段式，且 `memory-unit` 与 `memory-object` 两套字段集并存，**共享合同字段数过大（28/29 全必填）** 使任何结构微调都外溢。**RECOMMENDED ADDITION（NEW PROPOSAL）**：合并 memory-object / memory-unit 为单一权威 schema，并给"共享字段白名单"与"按模块投影视图"（见 §7 E-2/E-3）。

---

### 表4 模块调整影响矩阵（场景 × 影响模块 × 接口 × 是否改 Schema × 迁移 × 改算法 × 重测 × 改动范围 × 是否影响他模块核心行为 × 变更类别）

> 本表为 §3 十四场景的压缩索引；"改动文件数量级"的依据 = 该场景触及的 **config schema 数 + 业务 schema 数 + 00/03/07/09/11/12/13/14/15 文档数 + 03-m0*.md 数**（按 §9 实测文件清单估算）。变更类别取值封闭：`CONFIG_ONLY` / `MODULE_LOCAL_CHANGE` / `CONTRACT_CHANGE` / `SCHEMA_MIGRATION` / `MULTI_MODULE_CHANGE` / `ARCHITECTURE_CHANGE`。

| # | 场景 | 影响模块 | 影响接口 | 改 Schema? | 迁移? | 改算法? | 重测? | 改动范围（文件量级/依据） | 影响他模块核心行为? | 变更类别 |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 换 Embedding Provider（模型/维度） | M01 主 + M02/Worker 连带 | Embedding Adapter、Index Adapter、B-010 空间注册、检索路由 | 是（embedding-space 八字段 + config/embedding） | **是（07 §7.4 九步状态机 + 全量重嵌入）** | 否（算法不变，空间变） | 是（TC-M02-002/003 + 对账） | 中–大：2 config + 1 schema + 07/01/02/03/15 + 03-m01 ≈ **8–10 文件** | **是**（M02 排序漂移、Worker 回填） | `SCHEMA_MIGRATION` |
| 2 | 更换向量库 | M01 + M02 + Worker | Index Adapter、VectorRetriever、INV-03/04 | 是（config/storage 的 lancedb 组 + const 解锁） | 是（全量重建） | 否 | 是（FLAT 精确性、重建对账） | 大：2 config + 00 不变量 + 07/03/05/02 + 03-m01 ≈ **8–12 文件** | **是**（M02 检索、验收前提） | `ARCHITECTURE_CHANGE`（`vector.store` LOCKED，15 §7.9 明写"禁止变更；更换须重新裁决不变量"） |
| 3 | 改 FTS 中文分词器 | M01（Index Adapter）+ M02 | Search Document 重建、FTS5 tokenizer、中文回归集 | 否（config 字段已预留 `fts5.tokenizer`） | 否（07 §7.7 全量重建即可，权威不变） | 否（召回语义变，非排序公式） | 是（中文回归集无退化） | **小**：config/storage 1 + 07 §7.7 + 06 测试 ≈ **3–4 文件** | 轻微（M02 命中集变化） | `MODULE_LOCAL_CHANGE`（近似 CONFIG_ONLY 但需重建 + 回归，故定 MODULE_LOCAL_CHANGE） |
| 4 | 增加长期记忆区块 / semantic_type | **M01+M02+M03+M04+M05 全链** | Classifier、Anchor Router、Ranker、Memory API 统一返回、所有投影 schema | **是**（DD-V2 §2.1 六值枚举 + memory-unit.semantic_type + read-response + anchor-set + 09/11/12/14 文档） | **是**（memory_schema major + 07 §7.2 七要素） | 是（分类/路由/排序权重） | 是（全链路 + Benchmark 15 类） | **很大**：≥6 schema + 6–8 文档 + 5 模块稿 ≈ **14–20 文件** | **是**（全部） | `ARCHITECTURE_CHANGE`（枚举是 §2 "MUST 逐字使用"的规范枚举） |
| 5 | 改 Anchor 分类 | M02 主 + M01/M03/M04/Host 连带 | Anchor Router、MemoryClassifier、getByAnchor、read_mode 路由 | 是（DD-V2 §2.3 八值 + anchor-set.schema.json + config/anchor + memory-unit.anchor） | 部分（历史 anchor 值需回填/映射） | 是（意图路由 9 类 + 五阶段） | 是（锚点命中率基准 11 号） | 大：3–4 schema/config + 11/14/09/DD-V2/15 + 03-m02 ≈ **10–14 文件** | **是**（M02 人物优先、M03 分类） | `MULTI_MODULE_CHANGE` |
| 6 | 增加 relation 类型 | M04（语义）+ M01（持久化）+ M02（扩展） | listNeighbors、getRelated、Relation CRUD | 是（DD-V2 §2.15 八种 + relation.schema.json.relation_type 枚举） | 部分（既有关系值语义映射） | 是（冲突/依赖重算、一跳扩展） | 是（关系查询回归） | 中–大：1 schema + 01/11/14/DD-V2 + 03-m04/03-m01 ≈ **7–9 文件** | **是**（M02 扩展、M04 重算） | `CONTRACT_CHANGE` |
| 7 | 改 RRF 参数 | **仅 M02**（Fusion） | `recall.rrf_k` / config/retrieval.fusion.k | 否 | 否 | 否（**数值调优，07 §7.10 明确"权重数值变更 = minor 算法版本"**） | 是（TS-2/TC-M02-004 + k∈{10,60,120} 扫描） | **极小**：15 §7.7 一行 + config/retrieval 1 值 ≈ **2 文件** | 否 | `CONFIG_ONLY` |
| 8 | 改时间衰减 | M02（遗忘曲线）+ Psychology（ΔE 的 D，FUTURE） | 03-m02 D23、`emotion.decay_rate`、Rerank Time 特征 δ | 否 | 否 | 是（若改公式形态 → 算法版本 major） | 是（30/90/365 天曲线 + 排序回归） | 小：15 §7.8 + config/emotion + 02/03-m02 ≈ **4–5 文件** | 轻微（仅排序/情绪运行时） | `CONFIG_ONLY`（改数值）/ `MODULE_LOCAL_CHANGE`（改公式） |
| 9 | 改 Tier 晋升规则 | **M03+M04+M01+（M05 门禁）**，且 A 路已判 L1 无 owner/0 条转换 | CandidateValidator、reviewCandidate、晋升事务 | **是**（memory_tier/status 枚举 + 01 生命周期 + 保留策略） | **是**（历史 tier 值需映射） | 是（Evidence→Validation→Review 门禁） | 是（全链路） | 大：≥3 schema + 00/01/09/12/DD-V2/15 + 03-m03/03-m04 ≈ **10–14 文件** | **是**（全部） | `ARCHITECTURE_CHANGE`（叠加 A-C001…A-C004 P0：当前**无转换规则可改**） |
| 10 | 改情绪算法/权重 | M03（抽取）+ M02（查询）+ Psychology（FUTURE）+ M01（落库） | ΔE=I×C×R×D、8 门禁、15 标签词表 | 部分（config/emotion 数值否；改词表/公式需 emotion-context.schema + label_dict） | 否（默认 DISABLED，启用等价重新裁决 CR-03） | 是 | 是（门禁 8 条单测 + κ≥0.6） | 中：2 config/schema + DD-V2 §9/§12 + 13/15 + 03-m03 ≈ **6–9 文件** | 轻微（默认关闭，X-1 禁止影响 M02 最终排序） | `MODULE_LOCAL_CHANGE`（启用则升 `ARCHITECTURE_CHANGE` + 用户裁决） |
| 11 | 改 Persona Prompt | Persona（FUTURE）+ Host/Context Adapter（08） | 授权视图、context-response 注入 | 否（prompt 文本不属 schema） | 否 | 否 | 是（注入语义回归） | **小**：08 + 10-LLM 调用体系 + 测试 ≈ **2–3 文件** | 否（Persona 是消费者，被禁改权威） | `MODULE_LOCAL_CHANGE`（但 Persona 全部 FUTURE，00 §7.7，当前**无模块可改**） |
| 12 | 新增 Dream | **调度层 + M01/M04/M05**（FUT-05） | 维护任务八类、Outbox/Worker、视图重算 | 是（任务类型枚举 + 03-m04 §5.3 已"预留入口"） | 否 | 是（周期调度/幂等） | 是（拔除测试 7 条外部依赖，00 §7.7） | 大：00 §7.7 + 07 §7.13 FUT-05 + 03-m04 §5.3/§7 + 05 + 03-m05 ≈ **6–9 文件** | **是**（调度 owner 与 M04/Worker 冲突，见 §6 H-3） | `ARCHITECTURE_CHANGE`（D-AP-03 禁止第六模块；Dream 无独立 DB 写权） |
| 13 | 改检索预算（1200 token / Top-K） | **仅 M02**（ContextBuilder） | `recall.budget_tokens` / `recall.top_k` / config/retrieval.context | 否 | 否 | 否（若改预算策略公式 → 算法版本） | 是（TS-2/TC-M02-010 + token_usage 五类加总） | **极小**：15 §7.7 + config/retrieval 1 值 ≈ **2 文件** | 否 | `CONFIG_ONLY` |
| 14 | 改 PersonProfile 结构 | **M04+M01+M02+Host**（DD-V2 §3.5 11 分区） | getProfile、buildView、ProfileIntegrator、read-response.profile、context-response.sections_present | **是**（person-profile 10 全必填 + config/profile 11 section + read-request.sections + read-response.profile + memory-unit.profile_field） | **是**（画像版本 + 归档） | 是（整合规则/basis 三值） | 是（画像一致性 + 人物理解 NDCG 分组） | 大：4–5 schema/config + 12/14/DD-V2/15 + 03-m04 ≈ **9–12 文件** | **是**（M02 人物优先、Host 读取） | `SCHEMA_MIGRATION`（若增删 section 枚举）/ `CONTRACT_CHANGE`（仅改字段） |

**判定**：14 场景中 **4 个 CONFIG_ONLY（7/8/13 及 3 的近似）**、**1 个 MODULE_LOCAL_CHANGE（10/11）**、**5 个 MULTI_MODULE_CHANGE/ARCHITECTURE_CHANGE（4/5/9/12/2）**、**3 个 SCHEMA_MIGRATION/CONTRACT_CHANGE（1/6/14）** → 可低调整的场景集中在 M02 数值层；一旦触及"记忆本体/枚举/画像/关系/模块拓扑"，成本骤升。**DESIGN COMPLETENESS**：07 §7.10 已给"算法替换六前提"、07 §7.2 给"迁移七要素"、15 §7.14 给"参数变更八步"，**这些步骤/条件/责任是充分的**；不足在于它们大多把执行主体写成"配置管理员/运维/批准人"，未绑定到具体模块接口签名，故"改动范围"只能按文档估算而非按代码契约计算。

---

### 表5 配置与参数归属表（参数/配置项 × 所属模块 × 定义位置 × 15 号注册表 × config schema × 可否热改）

> 证据：15 §7.4–7.12（A04 所属模块 / A09 读方 / A10 写方 / A13 运行时值 / A14 状态）、15 §7.16（合计 42 参数：4 DISABLED / 3 LOCKED / 3 APPROVED / 32 EXP-DEFAULT）、15 §7.17（待登记参数，MUST NOT 静默使用）、03 §7.11 配置八类、contracts/config 11 个 schema。
> **口径纠正（EXISTING DESIGN 实测）**：contracts 目录共 **31 个 .json**（= 11 config + 20 schemas），并非"config 12 / schemas 31"；`contracts/config` 无 `llm.schema.json`（15 §7.2 明示"P3 待建"），故 `ingest.*` 6 参数**无 config schema 落地**。

| 参数组 / 代表参数 | 所属模块（A04） | 定义位置 | 在 15 号注册表? | 在 config schema? | 状态 | 可否热改（判定） |
|---|---|---|---|---|---|---|
| `ingest.*`（6：model / model_tier / batch_size / timeout / retry / max_input_tokens） | Pipeline / LLM Gateway（**挂 M04 §6.2 唯一入口，但 15 归 Pipeline**） | 15 §7.4；`config/llm.schema.json` **未落地** | 是（6） | **否**（15 §7.2 自认"配置文件未落地 ⇒ 现阶段仅注册表默认值"） | EXP-DEFAULT | **否**（无载体；且模型选型须用户裁决 + 10 号能力测试） |
| `memory.*`（4：min_confidence / auto_accept_threshold / review_threshold / max_units_per_input） | M03 / CandidateValidator | 15 §7.5 → `config/memory.schema.json` | 是（4） | 是（memory.schema.json 5 顶层 / 4 required） | EXP-DEFAULT；AUTO_ACCEPT 通道未裁决（CR-01） | 有限可改（序关系 `min≤review≤auto` 违反 ⇒ CONFIGURATION_ERROR；**不得**据此开启自动确认） |
| `anchor.*`（4：routing_enabled / profile_priority / event_priority / max_candidates） | M02 / AnchorRouter | 15 §7.6 → **唯一权威位点 `config/anchor.schema.json`**，retrieval 同名字段为只读镜像 | 是（4） | 是（anchor.schema.json；retrieval 镜像） | EXP-DEFAULT | **受限**（改 `routing_enabled` 属故障降级，须记 degraded=true；镜像不等 ⇒ 启动失败 → 实际需重启校验，非纯热改） |
| `recall.*`（6：top_k / budget_tokens / rerank_enabled / rrf_k / evidence_expansion_limit / budget_chars） | M02 / ContextBuilder·Fusion·Ranker | 15 §7.7 → `config/retrieval.schema.json`（context / fusion / reranking） | 是（6，+§7.17 预留 9 个 ID 未给值） | 是（retrieval.schema.json 6 顶层 / 10 propBlocks） | 1 APPROVED（rerank_enabled，裁决 R10）+ 5 EXP-DEFAULT | **是（数值类）**：rrf_k / budget_tokens / top_k 为 CONFIG_ONLY；`evidence_expansion_limit`/`budget_chars` 标"P2 增量未落地" |
| `emotion.*`（7：3 开关 DISABLED + 4 数值） | Psychology 接口 / M02 / M03 / M05 | 15 §7.8 → `config/emotion.schema.json` | 是（7） | 是（3 个 `const:false` 锁定） | 3 DISABLED + 4 EXP-DEFAULT | **否**（3 开关 const 锁定，启用 = 改 schema + CR-02/03/04 重新裁决）；4 数值仅在开关启用后才参与计算 |
| `vector.*`（6：provider / model / dimension / store / metric / index_mode） | M01 / Embedding·LanceDB Adapter | 15 §7.9 → `config/embedding.schema.json` + `config/storage.json` + `config/retrieval.json` | 是（6） | 是（embedding 11 顶层；storage.lancedb；retrieval.vector） | 2 LOCKED（store / index_mode const）+ 4 EXP-DEFAULT | **否/重**：store/index_mode 禁改；provider/model/metric 改须新建 `embedding_space_id` + 全量重建（07 §7.4 九步） |
| `permission.*`（3：default / m05_mode / public_enabled） | M05 | 15 §7.10 → `config/security.schema.json` | 是（3） | 是（security.schema.json；default_policy const=deny） | 1 LOCKED + 1 APPROVED + 1 DISABLED | **否**：default 禁改；m05_mode 切 `implemented` 须 P5/P6 阶段授权 + 10 环节覆盖证据；public_enabled 须 CR-07 |
| `indexing.*`（3：outbox.poll_ms / retry_max / consistency_check） | M01 / Worker | 15 §7.11 → `config/storage.json` + `config/worker.json` | 是（3） | 是（storage.outbox、worker.maintenance_tasks） | 1 APPROVED + 2 EXP-DEFAULT | 是（数值/开关；但 `maintenance_tasks.duplicate_scan/auto_delete` 等写操作型 MUST 保持关） |
| `profile.*`（3：auto_update / max_sections / version_retention） | M04 / M01 | 15 §7.12 → `config/profile.schema.json` | 是（3） | 是（profile.schema.json；`write_channel=M04_only` const） | EXP-DEFAULT | **否（当前）**：M04 属 P4 需阶段授权；`auto_update` 默认 false；`max_sections` 受 11 section 枚举约束，改枚举须改 schema |
| 八类配置（Core/Storage/Retrieval/Embedding/Index/Security/Test/Host Adapter） | 跨模块；03 §7.11 仅规定"分类 + Schema 校验 + 未知项拒绝" | 03 §7.11 → 11 个 config schema（core/storage/retrieval/embedding/anchor/emotion/memory/profile/security/test/worker） | 部分（注册表只覆盖 42 参数，config schema 另有 `core.algorithm_version`、`test.mock_*`、`worker.*`、`storage.sqlite.*` 等**未在 15 号登记的字段**） | 是 | 混合 | **DESIGN COMPLETENESS 缺口**：15 §7.17 已预留部分 ID，但 `storage.busy_timeout_ms`、`worker.claim_timeout_ms`、`core.log_level`、`security.audit.*` 等仍无注册表 ID → 违反 00 §7.8「每个参数 MUST 具备稳定唯一参数 ID」与 15 §7.17「MUST NOT 静默使用」 |

**判定：MEDIUM–HIGH**。**EXISTING DESIGN** 是强项：42 参数逐条给出 15 属性（A01–A15）、三层次（可观察/可配置/可验证）、四值生效状态、八步变更流程、LOCKED/EXP-DEFAULT/DISABLED 纪律，且 config schema 用 `const` 把安全红线锁死——这使"哪些可配置"**可以明确回答**。缺口（DESIGN COMPLETENESS）有三：① `ingest.*` 无 config 载体；② 存在未登记的配置字段；③ "热改"未定义（多数配置描述为"启动时校验/启动快照/非法配置拒绝应用"，03-m01 §7.2、15 A13 运行时值），**实际是启动期加载而非运行时热更新**。**RECOMMENDED ADDITION（NEW PROPOSAL）**：补 `config/llm.schema.json`、为 §7.17 与未登记 config 字段补 ID，并明确定义"热改 vs 重启生效"两档 + 配置变更事件（见 §7 E-4）。

---

### 表6 共享合同依赖表（contract/schema × 生产者 × 消费者 × 破坏半径）

> 证据：contracts/schemas 20 个（字段数/required/enums 见 §9）；grep 实测 `semantic_type` 命中 72 次分布于 **16 个文件**、`anchor` 219 次分布于 **28 个文件**、`read_mode` 91 次、`section` 83 次、`memory_tier` 21 次、`relation_type` 8 次；07 §7.1 八维版本登记；07 §7.8 兼容性六版本。

| contract / schema | 字段数(required) | 生产者（唯一写者） | 消费者 | 破坏半径（改字段/枚举的连带范围） | 分级 |
|---|---|---|---|---|---|
| `memory-unit.schema.json` | **28 (28)** | M01（经 Memory API） | M02 回源、M03/M04 候选与整合、M05 权限元数据、read-response / context-response / search-document / candidate / 14 号 API | **最大**：semantic_type(6枚举)/anchor(8)/memory_tier(3)/status(4)/visibility(3)/subject_type(6)/source_type(5)/validation_state 全在此；grep `semantic_type` 跨 16 文件 → 改一处需同步 ≥8 schema + 6 文档 | **HIGH / P1** |
| `memory-object.schema.json` | 29 (15) | 与上行**并存**（DD-V2 §3.1 五核心对象用此套：type/subtype/lifecycle_state/availability_state…） | 01 号数据模型、07 §7.1 `memory_schema` 版本维度 | **语义二义**：与 memory-unit 用不同字段名表达同一对象（type vs semantic_type、lifecycle_state vs memory_tier+status）→ 任一改动需决定改哪套 | **HIGH / P1**（见 §6 H-1） |
| `relation.schema.json` | 10 (7) | M01 持久化 / M04 语义 | M02 listNeighbors 扩展、14 号 getRelated、01 号关系模型 | 中–大：`relation_type` 八种枚举（DD-V2 §2.15）+ 端点仅 memory_id（B 路已判）；改类型需 01/11/14/03-m04/03-m01 | **HIGH / P1** |
| `anchor-set.schema.json` | 5 (4) | M02 / AnchorRouter（config/anchor 为权威位点） | 09 摄取、11 分类、12 画像、14 API、read-request/read-response、anchor.schema/retrieval.schema | **大**：`anchor` 8 值 + `intents` 9 类 + `default_routes` + `weights`；grep `anchor` 跨 28 文件 | **HIGH / P1** |
| `person-profile.schema.json` | 10 (10) | M04（M04_only） | M02 getProfile、Host、read-response.profile、context-response.sections_present、config/profile(11 section)、12 号 | **大**：section 枚举 + basis 三值 + source_memory_ids 追溯；改分区 = 4 处 schema/config + 3 文档 | **HIGH / P1** |
| `search-document.schema.json` | 21 (14) | M01 Index Adapter（派生可重建） | M02 检索、Worker 重建、07 §7.7 迁移 | **小–中**：可全量重建（07 §7.7/§7.12），权威不受影响；但 `text_projection_version`/`chunker_version` 变更会**牵连向量空间** | MEDIUM |
| `outbox-event.schema.json` | 13 (7) | M01（同事务） | Worker（唯一消费者）、03 §7.6/§7.7 | **单点高**：`payload_version` 消费者须支持 N + N−1（07 §7.1），不支持则 dead_letter → 改 payload = 双版本窗口 + Worker 改造 | **HIGH / P1** |
| `embedding-space.schema.json` | 15 (12) | M01 / Embedding Adapter（B-010 八字段） | M02 检索、Worker 回填、config/embedding、07 §7.4/§7.5 | **中–大**：任一字段变化 ⇒ 新 space id（不可原地改），零兼容、禁混排（07 §7.5/§7.6） | **HIGH / P1** |
| `read-response.schema.json` | 10 (9) | M02（经 Memory API） | Host/DSH、14 号、context-response | 中：21 处 enum，items/evidence/emotion_context/profile/trace/meta 全镜像上游字段 | MEDIUM |
| `context-response.schema.json` | 11 (7) | M02 ContextBuilder | Host Adapter（08）、Persona（FUTURE） | 中：`context_contract_version`（八维之一）、permission_trace、sections_present；宿主须声明支持区间（07 §7.8） | MEDIUM |
| `permission-context.schema.json` | 16 (4) | M05（判定）/ M01（持久化） | **全部读写路径**（00 §7.6 红线1）、03 §7.5 六动作 | **大且不可插拔**：C40 引擎不属 07 §7.10 算法替换范围；decision_id/policy_version 进每次审计 | **HIGH / P0–P1** |
| `evidence` / `event` / `observation` / `emotion-context` / `candidate` | 21/15/14/14/21 | M01（前四）/ M03（candidate） | M02/M03/M04 + 4–5 个交叉 schema（见 表3） | 中：`evidence_refs`/`source_evidence_ids` 在 ≥4 合同重复出现 | MEDIUM |
| `error.schema.json` + `contracts/error-codes.md`（16 规范码） | 10 (2) | 全模块共用（03 §7.10） | 全模块 + API 响应 | 中：MUST NOT 各模块自行命名错误码；新增/归并须同步 error-codes.md / error.schema.json enum / 03 §7.10 **三处**（03 §7.10 明示） | MEDIUM（规则明确） |
| 11 个 `config/*.schema.json` | 见 表5 | 配置管理员 + 15 号注册表 | 全部模块（A09 读方） | 中–大：未知字段 MUST 拒绝；`anchor.routing_enabled` 双位点镜像；`retrieval` 与 `anchor`/`storage`/`embedding` 交叉 locator | MEDIUM–HIGH |

**判定：LOW（独立性角度）**。所有核心对象都是**共享核心模型**，没有"模块私有 schema"；07 §7.1 的八维版本登记与 §7.8 兼容性六版本是 **EXISTING DESIGN** 且设计质量高（能阻断静默不兼容），但它同时也**证明了耦合的广度**：一次 `semantic_type` 或 `PersonProfile.section` 变更要穿过 8–28 个文件。**RECOMMENDED ADDITION（NEW PROPOSAL）**：为每个共享合同显式登记"生产者/消费者矩阵 + 破坏半径"，并作为 07 §7.1 `VersionRegistryEntry` 的必填列（见 §7 E-5）。

---

## §3 十四个变更场景逐一判定

> 统一口径：**"文件数量级"依据 = 实测 contracts 清单（11 config + 20 schema）+ 00–15 文档清单 + 03-m01~m05 五份模块稿 + DD-V2 + 15 号**。凡标 `const`（LOCKED/DISABLED）或"MUST 逐字使用"的枚举，改它即触发重新裁决，不属于普通变更。**变更类别只用六个封闭值**。

### 场景 1 更换 Embedding Provider（模型/维度）
- **EXISTING DESIGN**：00 §7.3 规则3「Embedding Provider 归 M01（M02 只经 Embedding Adapter 调用）」、00 §7.8「Adapter 可替换 / 维度不得写死（裁决 R9）」、07 §7.4 九步状态机 + §7.5 B-010 八字段 + §7.6 九必备四禁令、15 §7.9 `vector.*`、config/embedding.schema.json（11 顶层，含 `active_space`/`spaces`）。
- **影响模块 / 接口**：M01（Embedding Adapter、Index Adapter、B-010 空间注册）主；M02（VectorRetriever 路由）、Worker（回填）连带。接口 = `vector.provider`/`vector.model`/`vector.dimension`/`vector.metric`、检索路由配置、TC-M02-003 空间隔离。
- **改 Schema？是**（embedding-space 八字段任一变化 ⇒ 新 `embedding_space_id`，07 §7.5 派生规则；config/embedding 的 provider/model/metric）。**迁移？是且重**（九步：冻结旧空间→建新空间→全量 Backfill→Validate 覆盖率 100%→双读双写→对比→切换→监控→退役，07 §7.4）。**改算法？否**（算法版本不变，语义空间变）。**重测？是**（FLAT 精确性 TC-M02-002、自检索一致、重建对账 INV-04）。
- **文件数量级依据**：config/embedding + config/retrieval.vector + config/storage.lancedb（3）+ embedding-space.schema.json + search-document（2）+ 07 §7.4/7.5/7.6/7.12 + 01/02/03 + 15 §7.9 + 03-m01 §4.3/4.4 ≈ **8–10 文件**。
- **影响他模块核心行为？是**（M02 排序漂移、Worker 回填期间写入归属）。**变更类别：`SCHEMA_MIGRATION`**。
- **判定 MEDIUM**：这是全 14 场景中**设计最完整**的一个——输入（八字段）、输出（新 space id）、步骤（九步）、条件（覆盖率 100%、容差 TBD）、模块责任（migration runner/批准人/运维）、边界（旧空间只读、禁混排、禁静默降级）全部齐备。缺口仅"容差/观察窗口 TBD 至实测"，属 **DESIGN COMPLETENESS** 待补而非「不可行」。**不得**把"未实测"读作"不可行"。

### 场景 2 更换向量库
- **EXISTING DESIGN**：00 §7.8「LanceDB APPROVED（不变量 3）」、00 §7.9 INV-03/04/11「删除 LanceDB 后可从 SQLite 重建 / 不许静默降级」、15 §7.9 `vector.store` = **LOCKED（`enum const=lancedb`，无人可写）**、config/retrieval.vector.store = const、07 §7.12 索引重建六模式。
- **影响模块 / 接口**：M01（LanceDB Adapter、Index Adapter、INV-03/04）、M02（VectorRetriever）、Worker（重建）；接口 = `vector.store`、FLAT 执行路径、重建对账。
- **改 Schema？是**（解锁 config 的 const + storage.lancedb 组 + 00 不变量表）。**迁移？是**（全量重建，07 §7.12「Embedding 空间重建/全量重建」）。**改算法？否**（但 FLAT 精确性验收前提失效）。**重测？是**（TC-M02-002 FLAT vs 暴力余弦、INV-04 对账 0 差异）。
- **文件数量级依据**：00 §7.8/§7.9 + 07 §7.12 + 03 §7.8 + 05 + 02 + 15 §7.9 + config/retrieval + config/storage + 03-m01 ≈ **8–12 文件**。
- **影响他模块核心行为？是**。**变更类别：`ARCHITECTURE_CHANGE`**（15 §7.9 原文：「禁止变更；更换须重新裁决不变量」；INV-03 是 12 条系统不变量之一）。
- **判定 LOW**：不是"技术上做不到"（Index Adapter 已隔离、索引可重建），而是**制度上被锁死且需改不变量**。**RECOMMENDED ADDITION（NEW PROPOSAL）**：若未来确需可换，应把"向量库"提升为 `vector_store_interface` 契约 + 能力探测（FLAT/维度/度量），并把它从 INV-03 的具体产品名中解耦（见 §7 E-6）。

### 场景 3 改 FTS 中文分词器
- **EXISTING DESIGN**：07 §7.7「`tokenizer` 变更 ⇒ 重建 + 记录分词器版本 + 中文查询回归集无退化」、config/storage.schema.json `fts5.tokenizer` 已预留、00 §7.8「FTS5 APPROVED（不变量 2）」、03-m01 §4.8 D21 中文全文检索。
- **影响模块 / 接口**：M01（Index Adapter / FTS5 重建）主，M02（命中集）连带；接口 = `fts5.tokenizer`、Search Document 重建、中文回归集。
- **改 Schema？否**（config 已有该字段，权威数据不变）。**迁移？否**（07 §7.7 全量重建即可，**不改变权威数据**）。**改算法？否**（召回语义变，排序公式不变）。**重测？是**（中文回归集无退化）。
- **文件数量级依据**：config/storage（1）+ 07 §7.7 + 06 测试/Benchmark + 03-m01 §4.8 ≈ **3–4 文件**。**影响他模块核心行为？轻微**（M02 命中集变化，权限/排序/预算不受影响）。**变更类别：`MODULE_LOCAL_CHANGE`**（接近 CONFIG_ONLY，但需全量重建 + 回归，故不归 CONFIG_ONLY）。
- **判定 HIGH（最低成本场景）**：输入/输出/步骤/条件/责任/边界齐备。**EXISTING DESIGN** 完整。

### 场景 4 增加长期记忆区块 / semantic_type
- **EXISTING DESIGN**：DD-V2 §2.1「semantic_type 六值，**MUST 逐字使用，禁止同义变体**」、DD-V2 §4.1 旧→新映射、00 §3.2「明确不覆盖（FUTURE / 不实施）」、07 §7.1「memory_schema 字段/语义变更 = major」、11 号语义分类、09 号 Pipeline stage 6 MemoryClassifier。
- **影响模块 / 接口**：**M01–M05 全链 + Host/DSH**；接口 = MemoryClassifier、AnchorAssigner、Anchor Router、Ranker（Type 权重 γ）、Memory API 统一返回（DD-V2 §7「统一返回 MUST 含 semantic_type」）、candidate.proposed_type、read-response.items、search-document.type、context-response。
- **改 Schema？是且广**：`memory-unit.semantic_type`（6 值枚举）+ `candidate.proposed_type` + `read-response` + `search-document.type` + `anchor-set.default_routes` + DD-V2 §2.1 枚举 + 09/11/12/14 文档（grep `semantic_type` 跨 **16 文件 72 次**）。**迁移？是**（memory_schema major + 07 §7.2 七要素 + 历史值映射）。**改算法？是**（分类/路由/排序权重 γ）。**重测？是**（Benchmark 15 类 + 全链路）。
- **文件数量级依据**：≥6 schema + DD-V2/00/09/11/12/14/15（7）+ 5 份 03-m0*.md ≈ **14–20 文件**。**影响他模块核心行为？是（全部）**。**变更类别：`ARCHITECTURE_CHANGE`**。
- **判定 LOW / P1**：B 路已判"长期记忆是正交分面池"（02-LONG-TERM-MEMORY-STRUCTURE.md），**"增加区块"在概念上等价于新增分面/枚举**，但当前 `semantic_type` 是封闭 6 值且被 16 文件镜像 → 无"局部新增"路径。**RECOMMENDED ADDITION（NEW PROPOSAL）**：把 semantic_type 从"封闭枚举"改为"受控词表 + 注册登记表（新增值自带路由/排序/权限默认）"，并把镜像字段改为引用而非复制（见 §7 E-2/E-7）。

### 场景 5 改 Anchor 分类
- **EXISTING DESIGN**：DD-V2 §2.3「anchor 8 值」、§8 检索五阶段 + 意图路由（9 类 intent）、00 §7.8「RRF 权重 / Anchor Router」、config/anchor.schema.json（唯一权威位点，含 `anchors` 8 值 + `weights` + `intent_routing`）、anchor-set.schema.json（5 顶层）、15 §7.6 `anchor.*`。
- **影响模块 / 接口**：M02（AnchorRouter、IntentResolver、Ranker）主；M01（持久化 anchor 字段）、M03（MemoryClassifier 分配）、M04（Cue Anchor 生成，03-m01 §9.2）、Host（getByAnchor）连带。接口 = `anchor.*`、getByAnchor、read_mode 路由、profile_priority/event_priority。
- **改 Schema？是**（DD-V2 §2.3 + memory-unit.anchor + anchor-set + config/anchor + read-request.anchor + read-response）。**迁移？部分**（历史 anchor 值需映射/回填）。**改算法？是**（意图路由 9 类、人物优先、互斥路由断言）。**重测？是**（11 号锚点命中率基准 + 人物/事件查询分组 NDCG）。
- **文件数量级依据**：3–4 schema/config + 11/14/09/DD-V2/15（5）+ 03-m02/03-m03/03-m01 ≈ **10–14 文件**（grep `anchor` 跨 **28 文件 219 次**，为全部关键词之最）。**影响他模块核心行为？是**。**变更类别：`MULTI_MODULE_CHANGE`**。
- **判定 LOW**：anchor 是横切度最高的字段；15 §7.6 已用"唯一权威位点 + 只读镜像 + 镜像不等 ⇒ CONFIGURATION_ERROR"把**配置层**收敛得很好，但**枚举层**仍是 8 值硬编码在 ≥5 个合同里。

### 场景 6 增加 relation 类型
- **EXISTING DESIGN**：DD-V2 §2.15「关系类型保留 v1 八种」、01 号关系模型、03-m04 §5.1 关系类型 / §12.4 `listNeighbors(memory_refs, relation_types)`、14 号 getRelated、relation.schema.json（10 字段/7 required，`relation_type` 枚举）、07 §7.10 算法替换六前提（关系通道是否适用未明说）。
- **影响模块 / 接口**：M04（语义 + 依赖重算）、M01（Relation CRUD 持久化）、M02（一跳扩展 + 去重 + 预算）、Host（getRelated）。接口 = `relation_type` 枚举、listNeighbors、Relation CRUD、memory-unit.relation_refs。
- **改 Schema？是**（relation.schema.json.relation_type 枚举 + DD-V2 §2.15 + 01 号）。**迁移？部分**（既有关系值语义映射）。**改算法？是**（冲突检测/依赖重算/一跳扩展，03-m04 §5.2）。**重测？是**（关系查询回归）。
- **文件数量级依据**：1 schema + 01/11/14/DD-V2（4）+ 03-m04/03-m01/03-m02（3）≈ **7–9 文件**（grep `relation_type` 8 次 / 4 文件，横切度低于 anchor）。**影响他模块核心行为？是**。**变更类别：`CONTRACT_CHANGE`**。
- **判定 MEDIUM**：B 路已判 relation 两端只有 memory_id、无类型化跨对象关系（B-C001 P0）；"增加类型"只改枚举不解决该结构缺口。**RECOMMENDED ADDITION（NEW PROPOSAL）**：先按 B 路补 `endpoint_type` 与自动跨区块联合召回，再谈类型扩展（见 §7 E-8，与 02 号交付件衔接）。

### 场景 7 改 RRF 参数
- **EXISTING DESIGN**：00 §7.8「RRF k=60 实验初值」、15 §7.7 `recall.rrf_k`（默认 60，范围 1–1000，A09 读方 Fusion/Ranker，A15 TS-2/TC-M02-004 + k∈{10,60,120} 扫描）、config/retrieval.fusion.k、07 §7.10 算法替换六前提第 3 条「**权重数值变更 = minor，特征集/公式/权重语义变更 = major**」、07 §7.1 `search_algorithm_version`。
- **影响模块 / 接口**：**仅 M02 / Fusion**；接口 = `recall.rrf_k` 单个参数 ID。**改 Schema？否。迁移？否。改算法？否**（数值调优，但按 07 §7.10 仍应升 minor 算法版本并让结果可追溯）。**重测？是**（TS-2 可复现 + 缺失通道用例）。
- **文件数量级依据**：15 §7.7 一行 + config/retrieval 一个值（+ 07 §7.10 版本登记）≈ **2 文件**。**影响他模块核心行为？否**。**变更类别：`CONFIG_ONLY`**。
- **判定 HIGH**：这是"算法可独立调整"的标杆——参数 ID 唯一、读方单一、有验证方法、有版本纪律。**EXISTING DESIGN** 完整。

### 场景 8 改时间衰减
- **EXISTING DESIGN**：03-m02 §7 D23「遗忘曲线与时间衰减」、00 §7.8「Time 权重 δ=0.10（EXP-DEFAULT）」、DD-V2 §9「D = 时间衰减系数，随时间单调递减」+ 门禁 5、15 §7.8 `emotion.decay_rate = exp(-decay_rate × Δdays)`、00 §7.6 红线 2「`last_retrieved` MUST NOT 作为衰减 v2 的正向权重」、X-1「Psychology 不得直接影响 M02 最终排序」。
- **影响模块 / 接口**：M02（遗忘曲线 / Rerank Time 特征 δ）主；Psychology（ΔE 的 D，FUTURE/DISABLED）次。接口 = `emotion.decay_rate`、reranking.weights.time、03-m02 D23。
- **改 Schema？否**（数值在 config）。**迁移？否**。**改算法？改数值=否 / 改公式形态=是**（后者触发 07 §7.10 major 版本）。**重测？是**（30/90/365 天曲线 + 单调性单测 + 排序回归）。
- **文件数量级依据**：15 §7.8 + config/emotion + 02 算法 + 03-m02 + 06 测试 ≈ **4–5 文件**。**影响他模块核心行为？轻微**（默认情绪开关 DISABLED；X-1 禁止污染 M02 排序）。**变更类别：`CONFIG_ONLY`（数值）/ `MODULE_LOCAL_CHANGE`（公式）**。
- **判定 HIGH–MEDIUM**：数值层完全可配；但"衰减"在 M02（检索排序）与 Psychology（情绪 ΔE）**两处各有一套 D**，且 00 §7.6 明令 `last_retrieved` 不得作为正向权重 → 口径需统一。**RECOMMENDED ADDITION（NEW PROPOSAL）**：登记单一 `decay` 参数族并显式声明两处 D 是否同源（见 §7 E-9）。

### 场景 9 改 Tier 晋升规则
- **EXISTING DESIGN**：00 §7.5 时间尺度三层 +「晋升 MUST 经过 Evidence → Validation → Review」+ 禁止清单、DD-V2 §2.5「memory_tier 3 值，与 semantic_type 正交」/§2.6 status 4 值/§2.7 candidate.status 6 值、01 号生命周期、07 §7.1 `memory_schema` major 规则。**叠加 A 路已知结论**：memory_tier 只是标签轴，L1 无存储无 owner，tier 转换 0 条、写入者 0、保留策略 0（A-C001…A-C004 = **P0**，见 01-MEMORY-TIER-AND-LIFECYCLE-MATRIX.md）。
- **影响模块 / 接口**：M03（CandidateValidator）、M04（reviewCandidate/稳定性判断）、M01（晋升事务 + 版本）、M05（门禁 stage 15）、M02（读取 tier 过滤）。接口 = 晋升状态机、CandidateValidator、reviewCandidate、createMemory。
- **改 Schema？是**（memory_tier/status/candidate.status 三套枚举 + 01 号 + 保留策略字段）。**迁移？是**（历史 tier 值映射 + 07 §7.2）。**改算法？是**（Evidence→Validation→Review 门禁）。**重测？是**（全链路）。
- **文件数量级依据**：≥3 schema + 00/01/09/12/DD-V2/15（6）+ 03-m03/03-m04（2）≈ **10–14 文件**。**影响他模块核心行为？是（全部）**。**变更类别：`ARCHITECTURE_CHANGE`**。
- **判定 LOW / **P0 阻断**：本场景在字面上是"改规则"，但 A 路已证明**当前没有可改的转换规则、没有 owner、没有 L1 存储、没有保留策略**——这是"核心能力根本无法成立"（P0），不是"调整成本高"。任何 Tier 调整都必须先补 A 路四项，再谈本表。**RECOMMENDED ADDITION（NEW PROPOSAL）**：按 01 号交付件 A-C001–A-C004 补 tier 状态机、L1 存储/owner、转换条件与保留策略（见 §7 E-10，与 A 路衔接）。

### 场景 10 改情绪算法/权重
- **EXISTING DESIGN**：DD-V2 §9「ΔE = I×C×R×D」+ 8 门禁、15 §7.8（7 参数，3 个 `const:false` DISABLED，启用须 CR-02/03/04 重新裁决）、config/emotion.schema.json（3 const + label_dict 15 标签 + gates）、13 号 Evidence/EmotionContext、00 §7.7 Psychology「直接影响 M02 最终排序」被禁（X-1）、07 §7.10 规则「MUST NOT 借算法替换引入心理层加成」。
- **影响模块 / 接口**：M03（EmotionExtractor）、M02（查询侧，默认关闭）、Psychology（FUTURE）、M01（落库）。接口 = ΔE 公式、8 门禁、15 标签词表、getEmotionContext。
- **改 Schema？部分**（改数值=否；改词表/公式需 emotion-context.schema + label_dict）。**迁移？否**（默认 DISABLED，无历史生效数据；启用 = 重新裁决）。**改算法？是**。**重测？是**（门禁 8 条单测 + 反向用例 + κ≥0.6 标注一致性）。
- **文件数量级依据**：2 config/schema + DD-V2 §9/§12 + 13/15（4）+ 03-m03 ≈ **6–9 文件**。**影响他模块核心行为？轻微**（X-1 硬禁止；ΔE 只作用运行时，不写回 Event）。**变更类别：`MODULE_LOCAL_CHANGE`**（若启用 3 开关则升为 `ARCHITECTURE_CHANGE` + 用户裁决）。
- **判定 MEDIUM**：设计把情绪影响**主动锁在默认关闭 + const**，这是很强的边界（EXISTING DESIGN），因此"改权重"在生产上当前**不可执行**；但算法/门禁/参数/验证方法齐备，属 DESIGN COMPLETENESS 良好，不是"不可行"。

### 场景 11 改 Persona Prompt
- **EXISTING DESIGN**：00 §7.7「Persona = 记忆**消费者**；经授权视图接收记忆；禁止直连 SQLite/LanceDB、绕过 M05、修改权威记忆」、07 §7.13 FUT 清单（Persona 未单列，属 00 §7.7 FUTURE）、08 号 DSH/QQBridge 集成（Context Adapter 构造请求/注入宿主）、10 号 LLM 调用体系、DD-V2 §6 L3 聊天模型。
- **影响模块 / 接口**：Persona（FUTURE）+ Host/Context Adapter；接口 = 授权视图、context-response 注入。
- **改 Schema？否**（prompt 文本不属 schema）。**迁移？否**。**改算法？否**。**重测？是**（注入语义回归）。
- **文件数量级依据**：08 + 10 + 06 测试 ≈ **2–3 文件**。**影响他模块核心行为？否**（消费者，被禁改权威）。**变更类别：`MODULE_LOCAL_CHANGE`**。
- **判定 MEDIUM–LOW（当前 LOW）**：作为"消费者"其隔离设计是干净的（**EXISTING DESIGN** 边界充分），但 **00 §7.7 明标 Persona 全部 FUTURE、D1–D3 待 CR-08B 单独呈报** → 当前**没有可改的模块实例**。**不得**把"边界写得好"当成"模块已存在"。

### 场景 12 新增 Dream
- **EXISTING DESIGN**：00 §7.7「Dream = 周期维护调度器，**不是超级 Agent**；无独立 DB 写权限；任务八类；禁止绕过 M04/M05」、00 §7.3 依赖矩阵 Dream 行全为 `C`、00 §7.7「扩展模块 MUST 具备独立拔除能力，拔除后 R3 主链（M01+M02）MUST 仍可运行」、07 §7.13 **FUT-05**（启用需"维护任务可幂等执行且不绕过 M01/M05 的证据齐备"）、03-m04 §5.3「Dream 关系候选入口（设计边界预留）」、00 §7.3 规则5「横切能力 MUST 有唯一 owner，MUST NOT 通过新增进程或第六模块实现（D-AP-03）」。
- **影响模块 / 接口**：调度层 + M01（Outbox/Worker）+ M04（任务队列/DAG/视图重算）+ M05（判权）。接口 = 维护任务八类、Outbox 事件、统一任务队列（03-m04 §7）、拔除测试 7 条外部依赖。
- **改 Schema？是**（任务类型枚举 + `integrity_reflection` 与 Dream 任务标识必须分离，03-m04 §1.1 术语说明）。**迁移？否**。**改算法？是**（周期调度/幂等/依赖 DAG）。**重测？是**（拔除后主链仍运行）。
- **文件数量级依据**：00 §7.7 + 07 §7.13 FUT-05 + 03-m04 §5.3/§7 + 05 + 03-m01 + 03-m05 ≈ **6–9 文件**。**影响他模块核心行为？是**（与 M04 任务队列 / M01 Worker 的调度 owner 冲突，见 §6 H-3）。**变更类别：`ARCHITECTURE_CHANGE`**。
- **判定 LOW**：D-AP-03 明令"不新增第六模块"、Dream 无独立 DB 写权、横切能力须唯一 owner → "新增模块"在当前架构下**不是可插拔动作而是架构变更**。**RECOMMENDED ADDITION（NEW PROPOSAL）**：先把"任务调度 owner"在 M01/Worker 与 M04 之间唯一化，再定义 Dream 为 M04 内的 `task_type` 插件而非第六模块（见 §7 E-3/E-11）。

### 场景 13 改检索预算（1200 token / Top-K）
- **EXISTING DESIGN**：00 §7.8「Memory Token Budget 1200 tokens（EXP-DEFAULT，必须真实 Tokenizer）」、15 §7.7 `recall.budget_tokens`（1200，范围 64–8192）+ `recall.top_k`（20）+ `recall.budget_chars`（2400，P2 增量）+ §7.15 Token 单位纪律、config/retrieval.context（token_budget / max_items / budget_chars / tokenizer）、07 §7.10 算法替换适用对象含「Token 预算策略」。
- **影响模块 / 接口**：**仅 M02 / ContextBuilder·TokenBudget·GreedySelect**；接口 = `recall.budget_tokens` / `recall.top_k` / `recall.budget_chars`。**改 Schema？否。迁移？否。改算法？否**（改预算**策略公式**才升 major）。**重测？是**（TS-2/TC-M02-010 + token_usage 五类加总核对）。
- **文件数量级依据**：15 §7.7 + config/retrieval 一个值 ≈ **2 文件**。**影响他模块核心行为？否**。**变更类别：`CONFIG_ONLY`**。
- **判定 HIGH**：与场景 7 同为低调整成本标杆；且 15 §7.7 明确区分 `recall.top_k`（最终注入）与 `memory.search.opts.top_k`（每通道候选，ID `recall.channel_top_k_*`，§7.17 待登记），避免 ID 串用——**DESIGN COMPLETENESS** 好。唯一缺口：`channel_top_k_*` 等 9 个 ID 仍"待登记、本轮不给值"（15 §7.17）。

### 场景 14 改 PersonProfile 结构
- **EXISTING DESIGN**：DD-V2 §3.5「PersonProfile = M04 维护的**派生视图**，11 分区」、12 号人物画像整合（grep `section` 36 次）、config/profile.schema.json（`sections` 11 值 + `basis_policy` 三值 + `write_channel=M04_only` const + `profile_first_only_for_person_intents` const）、person-profile.schema.json（10 字段全 required）、15 §7.12 `profile.*`、read-request.sections / read-response.profile / context-response.sections_present / memory-unit.profile_field。
- **影响模块 / 接口**：M04（ProfileIntegrator/buildView）、M01（持久化/版本）、M02（getProfile + 人物优先）、Host（memory.getProfile）。接口 = getProfile、buildView、getProfile 返回、section 枚举、basis 追溯。
- **改 Schema？是**（person-profile 10 全必填 + config/profile 11 section + read-request/read-response/context-response + memory-unit.profile_field）。**迁移？是**（profile_version + 版本归档，15 §7.12 `version_retention`）。**改算法？是**（整合规则 + basis 三值 + 人物优先）。**重测？是**（12 号画像一致性 + 人物/事件分组 NDCG）。
- **文件数量级依据**：4–5 schema/config + 12/14/DD-V2/15（4）+ 03-m04 ≈ **9–12 文件**。**影响他模块核心行为？是**。**变更类别：`SCHEMA_MIGRATION`（增删 section 枚举）/ `CONTRACT_CHANGE`（仅改字段）**。
- **判定 LOW–MEDIUM / P1**：B 路已判"无 Project 实体、无情境记忆对象"；PersonProfile 是唯一成形的派生视图，但 11 分区被 config + schema + 4 处响应合同同时硬编码，改结构外溢大。**RECOMMENDED ADDITION（NEW PROPOSAL）**：section 改为注册表驱动的受控词表 + 按 `sections?` 动态投影（DD-V2 §7 已支持 `getProfile(subject_id, sections?)`），减少响应合同里的枚举复制（见 §7 E-2）。

**§3 汇总（逐项 HIGH/MEDIUM/LOW）**：HIGH = 3、7、13（+1 设计完整度）；MEDIUM = 1、6、8、10、11；LOW = 2、4、5、9、12、14。其中 **场景 9 为 P0 阻断**（A 路结论）、**场景 2/4/12 为 ARCHITECTURE_CHANGE 且被 LOCKED/D-AP-03 约束**。

---

## §4 十问必答

> 每题给结论 + 证据 + 三分类 + HIGH/MEDIUM/LOW。**文件拆成 M01–M05 不等于模块独立**（本轮判定纪律）。

**Q1 M01–M05 是否真正独立？**
**答：否（LOW）。** EXISTING DESIGN 有 00 §7.3 依赖矩阵、§7.6 职责边界、§7.9 INV-12「不允许模块跨越职责边界直接修改其他模块数据」、03 §7.2 通信禁令 8 条——**职责与依赖规则是明确写出来的**。但：① M03/M04/M05 在 00 §7.6/§7.7 与 DD-V2 §10（P3/P4/P5「新增，需阶段授权」）标 FUTURE，03-m05 §10.3 的 `permission.m05_mode=contract_deny`（15 §7.10）明写 **M05 未实现**；② 03-m01 §7.1「M02、M03、M04 的具体接口尚未冻结」，03-m04 §12.4「名称仅作设计示例，不代表最终 API 已冻结」；③ 五份 03-m0*.md 全部自称 **R3 基线**，M04 写「适用架构：独立 Memory Core」并依赖 Bridge/DSH/Model Gateway，与 00–15 的 V2/R4.1 不同版本（§6 H-6）；④ 全部核心对象为共享合同（表6），无模块私有 schema。**DESIGN COMPLETENESS**：边界充分、接口不足、实施主体部分不存在。

**Q2 有无明确接口？**
**答：部分（MEDIUM）。** EXISTING DESIGN 冻结度最高的是：03 §7.4 **M01 七函数**（含乐观并发/事务边界/超时）、03 §7.5 **M05 六动作**、DD-V2 §7 **Memory Read/Write API 九个方法**（含统一返回字段）、03-m05 §10.1/10.2 权限请求/返回结构、03-m04 §12.4 一跳邻居职责表。未冻结/示例级的：03-m01 §7.1（M02/M03/M04 接口未冻结）、03-m04 §12.1–12.3（方向示意，不冻结字段）、00 §7.6 核心接口标 FUTURE、03-m01 §7.4「具体 API 字段和调用协议待接口设计阶段确定」、03 §7.3 REST API 标 **PROPOSED 基线**。**RECOMMENDED ADDITION（NEW PROPOSAL）**：E-1（接口冻结表）。

**Q3 哪些是共享核心模型？**
**答：几乎全部（LOW 独立性）。** 表6 已列：`memory-unit`（28 全必填）、`memory-object`（29）、`relation`（10，端点仅 memory_id）、`anchor-set`（5）、`person-profile`（10 全必填）、`search-document`（21）、`outbox-event`（13）、`embedding-space`（15）、`permission-context`（16）、`evidence`/`event`/`observation`/`emotion-context`/`candidate`（21/15/14/14/21）、`read-response`/`context-response`。grep 实证：`semantic_type` 16 文件、`anchor` 28 文件、`read_mode` 9 文件、`section` 10 文件。**两个 memory schema 并存**（H-1）是最大的共享模型二义。

**Q4 哪些算法可独立调整？**
**答：M02 数值层可独立调整（HIGH），其余大多不能（LOW）。** EXISTING DESIGN：07 §7.10「算法替换六前提」适用范围明列 **RRF、规则排序特征集、Token 预算策略、文本投影、分块算法、检索通道实现**，并明确排除 C40 权限引擎与 M05；07 §7.1 `search_algorithm_version`（数值变 = minor）、15 §7.7 `recall.rrf_k`/`budget_tokens`/`top_k`。可独立 = 场景 7/8(数值)/13 + 3（分词器重建）。不可独立 = 分类/路由（牵 anchor-set + 5 模块）、ΔE（3 开关 DISABLED + CR 裁决）、Tier 晋升（P0 无规则）、画像整合（M04 P4 需授权）。

**Q5 哪些可配置？**
**答：42 个已登记参数 + 11 个 config schema（MEDIUM–HIGH）。** EXISTING DESIGN：15 §7.16 合计 42（32 EXP-DEFAULT / 3 APPROVED / 3 LOCKED / 4 DISABLED）、§7.2 三层次、§7.14 八步变更流程、03 §7.11 八类配置、config schema 用 `const` 锁 `default_policy=deny`/`vector.store=lancedb`/`vector.index_mode=flat`/3 个 emotion 开关/`write_channel=M04_only`。缺口：`ingest.*` 6 参数无 `config/llm.schema.json`（15 §7.2 自认）、§7.17 有 20+ 预留 ID 未给值、`storage.busy_timeout_ms`/`worker.claim_timeout_ms`/`core.log_level` 等 config 字段**未在 15 号登记**（违反 00 §7.8「每个参数 MUST 具备稳定唯一参数 ID」）。另：配置多描述为「启动时校验 / 启动快照 / 非法配置拒绝应用」→ **"热改"未定义**，实际是启动期生效。

**Q6 哪些要改 Schema？**
**答：9 个场景（1/2/4/5/6/9/10部分/12/14）。** 依据：07 §7.1「memory_schema 字段删除/语义变更 = major」「embedding_space 任一字段变化即产生新 space id」「event_schema payload 结构变更 +1」「context_contract 字段增删 = major」、07 §7.3 API 破坏性变更规则、DD-V2 §2「规范枚举 MUST 逐字使用，禁止同义变体」。纯 CONFIG_ONLY 的 7/13 与近似 CONFIG_ONLY 的 3、8(数值) 不改 schema。

**Q7 哪些要迁移？**
**答：1（重迁移）、2、4、9、14（+5/6 部分）。** EXISTING DESIGN 极完整：07 §7.2 迁移七要素（当前版本/目标版本/前向+回滚脚本/数据影响/兼容策略/回滚策略/验证方法）、§7.4 Embedding 九步状态机 + 14 条转换矩阵 + 7 条禁止转换、§7.6 九必备四禁令、§7.9 回滚六明确、§7.12 重建六模式、§7.11 A/B 层迁移规则。判定：**这些是 DESIGN COMPLETENESS 充分项，不是"未实测⇒不可行"**；07 §13.1 自认「当前全部 NOT RUN」只表示未演练，不改变设计存在。缺口：容差/观察窗口/双版本窗口长度多处 **TBD 至实测**。

**Q8 哪些影响多模块？**
**答：2/4/5/6/9/12/14（+1 连带）= 8 个，占 14 场景的 57%。** 证据：表2/表4 的"影响模块"列；00 §7.3 依赖矩阵中 M02→M01/M04/M05、M04→M01/M05、M03→M04/M05 均为跨模块；DD-V2 §7「统一返回 MUST 含 semantic_type/anchor/confidence/…」使任何本体字段变更穿透到 Host。低影响的只有 3/7/8/11/13（M01 或 M02 局部）。

**Q9 有无隐藏强耦合？**
**答：有，共 8 项（LOW）。** 详见 §6：H-1 双 memory schema、H-2 参数镜像/双位点、H-3 调度 owner 双重、H-4 LLM Gateway 唯一入口 vs M03 抽象 Provider、H-5 00 §7.3 中 M02→M04 标 C 但 03-m02/m04 写成必经路径、H-6 R3/R4.1 版本错位、H-7 横切 owner 与「禁止第六模块」叠加、H-8 全必填字段 + 枚举复制放大破坏半径。全部给 **NEW PROPOSAL** 见 §7。

**Q10 能否新增模块而不重写核心？**
**答：不能（LOW / ARCHITECTURE_CHANGE）。** EXISTING DESIGN：00 §7.3 首行「MUST 固定为五个核心模块 + 扩展模块，**不新增第六模块**——台账 D-AP-03」、规则5「横切能力（配置、日志、审计、任务调度）MUST 有唯一 owner，MUST NOT 通过新增进程或第六模块实现」、00 §7.7「扩展模块 MUST 具备独立拔除能力，拔除后 R3 主链（M01+M02）MUST 仍可运行；**C40 权限引擎不可插拔**」、07 §7.13 FUT-01…08 全部 FUTURE 且「启用 MUST 附：影响模块、兼容性、回滚方案、测试方案、是否需用户裁决」。**DESIGN COMPLETENESS**："拔除能力"是很好的可插拔纪律，但它只覆盖**已有扩展模块的移除**，没有覆盖**新模块的接入契约**（输入事件/权限挂载点/版本协商/数据所有权申请）。**RECOMMENDED ADDITION（NEW PROPOSAL）**：E-11「扩展点契约 + 能力注册 + 拔除测试矩阵」。

---

## §5 模块依赖 ASCII 图

> 依据 00 §7.3 依赖矩阵 + 03 §7.2 调用链 + 03-m04 §12.4 + 03-m05 §10.3 + 07 §7.13 FUT 清单图例：`──>` 同步允许调用；`-R->` 只读；`-E->` 事件/异步；`··C··` FUTURE/契约预留；`✗` 明令禁止；`[LOCK]` 不可插拔。

```
                        ┌───────────────────────────────────────────┐
                        │  Host / DSH / QQ Bridge / Context Adapter  │
                        │  (FUTURE, 08号；提供 Trusted Context)       │
                        └──────────────┬────────────────────────────┘
                                       │ memory.search / getProfile / propose
                                       ▼
   ┌──────────────────────────────────────────────────────────────────────────┐
   │                     Memory Read/Write API  (DD-V2 §7)                     │
   └───────┬───────────────────────────────────────────┬──────────────────────┘
           │ 读                                         │ 写（propose→candidate）
           ▼                                           ▼
 ┌───────────────────────┐                    ┌────────────────────────┐
 │  M02  Retrieval &      │                    │  M03  Extraction       │ FUTURE/P3
 │  Context Builder       │  ··C··(FUTURE)···▶│  (Candidate)           │
 │  (P2 / 主链之一)       │◀···················│  抽象 LLM Provider ⚠H-4 │
 └───┬──────┬──────┬─────┘   M02→M04 关系一跳  └───────────┬────────────┘
     │      │      │          00 §7.3 标 C,                 │ A 提交候选
     │R     │R     │A        03-m02/03-m04 写成必经 ⚠H-5    ▼
     │      │      │                              ┌────────────────────────┐
     │      │      └─────────────────────────────▶│  M04  Consolidation    │ FUTURE/P4
     │      │          A 权限终检 (deny/error/     │  · Relation 语义       │
     │      │            timeout/未知 一律拒绝)    │  · PersonProfile 视图  │
     │      │                                     │  · 唯一 LLM Gateway §6.2 ⚠H-4
     │      │                                     │  · 统一任务队列/DAG ⚠H-3│
     │      ▼                                     └───────┬────────┬───────┘
     │  ┌────────────────────┐                            │A 事务写入 │A 判权
     │  │  M05  Permission   │◀───────────────────────────┘           │
     │  │  [LOCK] C40 不可插拔│◀────────── A 判权 ────────── M01/M03    │
     │  │  contract_deny 当前 │                                        │
     │  └─────────┬──────────┘                                        │
     │            │R 读权限元数据                                     │
     ▼            ▼                                                   ▼
 ┌──────────────────────────────────────────────────────────────────────────┐
 │  M01  Storage & Data Management   （唯一权威 / 主链之一 / P1）            │
 │  ├─ SQLite  [唯一写者]  memory + version + outbox 同事务 (03 §7.4)        │
 │  ├─ Outbox  [唯一写者] ──E──▶ Worker ──E──▶ Index Adapter                │
 │  ├─ FTS5 / LanceDB  [单一所有者, 派生可重建, INV-02/03]                  │
 │  ├─ Index Adapter / Embedding Adapter (Provider 归 M01, 00 §7.3 规则3)   │
 │  └─ Backup / schema_migration / ChangeLog                                │
 └──────────────────────────────────────────────────────────────────────────┘
            ▲                                    ▲
            │ ✗ M02 直连 SQLite/LanceDB/FTS5      │ ✗ 业务模块直拼 SQL
            │   (00 §7.3 / 03-m02 §1.1)           │   (00 §7.3 规则1, INV-12)
            │                                    │
 ┌──────────┴────────────────────────────────────┴───────────────────────────┐
 │  FUTURE 扩展层（00 §7.7 / 07 §7.13 FUT-01..08，全部需单独授权 + 可拔除）  │
 │  Dream ··C··(全 M01–M05，无独立 DB 写权)  Persona ··C··(M02 授权视图/M05) │
 │  Psychology(DEFERRED, X-1 禁影响 M02 排序)  ANN/Cross-Encoder/Graphiti     │
 │  Context Compression(独立于 Core)  Audit  Evaluation  多宿主  分布式       │
 └──────────────────────────────────────────────────────────────────────────┘

 横切约束：配置(15号+11 config) / 日志 / 审计 / 任务调度  MUST 唯一 owner，
           MUST NOT 通过新增进程或「第六模块」实现  (00 §7.3 规则5, D-AP-03) ⚠H-7
```

**图注（隐藏耦合已在图中标 ⚠）**：H-4（LLM 双通道）、H-5（M02→M04 契约状态不符）、H-3（调度 owner）、H-7（禁止第六模块）；H-1/H-2/H-6/H-8 属数据/版本层，见 §6。

---

## §6 隐藏耦合清单（逐项给 HIGH/MEDIUM/LOW + 理由 + 违反的单向依赖）

> 判定纪律落地：用**共享 schema 字段数、config 被几个模块读、Outbox/FTS/向量是否单一所有者、是否存在被违反的单向依赖**说话，不靠"文件拆成 5 份"下结论。

| ID | 隐藏耦合 | 证据（EXISTING DESIGN 中的冲突） | 被违反的单向依赖 / 规则 | 分级 | 影响 |
|---|---|---|---|---|---|
| **H-1** | **两套 memory 主 schema 并存** | `memory-unit.schema.json` 28 字段全 required（semantic_type/memory_tier/status/anchor…）vs `memory-object.schema.json` 29 字段/15 required（type/subtype/lifecycle_state/availability_state…）；07 §7.1 把 `memory_schema` 载体写为 `memory-object.schema.json`，而 15 号/DD-V2/14 号用的是 memory-unit 字段名 | 00 §7.9 INV-01「SQLite 是唯一权威业务数据源」+ 07 §7.1「禁止一个总版本号代替全部 / 禁止各模块各自定义同名不同义的版本」——**同一对象两套字段语义** | **LOW / P1** | 改任一本体字段需决定改哪套；场景 4/9/14 成本被放大；01 号数据模型与 contracts 可能各引用一套 |
| **H-2** | **配置双位点 / 镜像 / 跨 config locator** | 15 §7.6：`anchor.routing_enabled` 唯一权威位点在 `config/anchor.schema.json`，`config/retrieval.schema.json` 同名字段为**只读镜像**，镜像独立取值 ⇒ `CONFIGURATION_ERROR`；15 §7.9 `vector.store` 落 `config/retrieval.json`、`vector.index_mode` 落 `config/storage.json`、`vector.provider` 落 `config/embedding.json`；07 §7.8 又要求 `context_contract` 与 "Context Format Version" MUST NOT 各自独立取值 | 00 §7.8「每个参数 MUST 具备稳定唯一参数 ID」+ 15 §7.17「MUST NOT 静默使用」——物理位点分散到 3–4 个 config，靠"镜像相等"人工维持 | **MEDIUM / P1** | 任一配置变更需同步多点 + 启动校验；热改几乎不可能；易出现"改了 A 忘了 B" |
| **H-3** | **任务调度 owner 双重** | 00 §7.3 规则5「横切能力（配置、日志、审计、**任务调度**）MUST 有唯一 owner，禁止新增进程/第六模块」；但 03-m04 §1.1 第 10 项 + §7「统一任务队列」/§7.6「任务依赖 DAG」把调度放 **M04**；03 §7.7 Worker/Outbox 消费 + config/worker.schema.json 把消费调度放 **M01/Worker**；00 §7.7 Dream 又定义为「周期维护调度器」 | 唯一 owner 规则被三处同时主张 | **LOW / P1** | 场景 12「新增 Dream」直接撞车；拔除测试（00 §7.7 7 条外部依赖）无明确主责 |
| **H-4** | **LLM 调用通道疑似两条** | 00 §7.3 规则3「**LLM Gateway 唯一入口 = M04 §6.2**（台账 D-AP-02）；任何模块 MUST NOT 另建 LLM 调用通道」；但 DD-V2 §5 的 16 阶段含 `LLMExtractor`(stage4)/`EmotionExtractor`(stage11)（属 M03/P3），03-m03 §12「M03 通过**抽象 LLM Provider** 调用模型，不直接依赖 DSH」；15 §7.4 `ingest.*` 6 参数 A04 归「Pipeline / LLM Gateway」 | 「唯一入口」与「M03 自带 Provider」并存 | **LOW / P1** | 场景 10/4 的 LLM 侧改动落点不清；`config/llm.schema.json` 至今未建（15 §7.2）可能正是该歧义的后果 |
| **H-5** | **M02→M04 依赖状态标注与实际路径不符** | 00 §7.3 依赖矩阵 M02→M04 = **C（仅契约预留，FUTURE）**；但 03-m02 §1.1「关系数据的读取**通过 M04 的关系查询能力完成**」、03-m04 §12.4「M04 只提供关系查询能力…M02 逐条提交 M05 权限检查」、DD-V2 §7 `memory.getRelated` 已列为统一 API | 依赖矩阵的 C 语义被下游文档写成必经 A 路径 | **MEDIUM / P1** | 场景 6/12 的调用链判定分歧；M04 FUTURE 时 M02 关系扩展实际无实现可依赖 |
| **H-6** | **模块稿与规范集版本错位（R3 vs R4.1/V2）** | 03-m01~m05 五份开头均写「属于**新一代 Memory Core 设计基线（暂称 R3）**；R2 为历史基线」；03-m04 写「适用架构：**独立 Memory Core**」并依赖 Bridge/DSH/Model Gateway；而 00–15 / DD-V2 为 V2/R4.1 且 00 §7.3 把 Bridge/DSH 归入 FUTURE 的 Context Adapter；03-m01 §9.4 仍写「**M02 尚未设计**」，但 03-m02.md 与 00 §7.6 M02 已成形 | 00 §0.6「旧→新术语与模型映射 MUST，禁止两套并存」、07 §7.1「禁止各模块各自定义同名不同义版本」 | **LOW / P1** | 接口/职责证据来自不同基线；表1/表2 的"未冻结"结论部分可能只是版本滞后而非真实缺口 |
| **H-7** | **「禁止第六模块」与「可插拔/扩展演进」目标冲突** | 00 §7.3 首行「不新增第六模块（D-AP-03）」+ 规则5；00 §7.7 却列 8 类扩展模块 + 「扩展模块 MUST 具备独立拔除能力」；07 §7.13 登记 FUT-01…08 且「FUT 启用 MUST 附影响模块/兼容性/回滚/测试/是否需用户裁决」；07 §7.10 又把 C40 权限引擎排除在可插拔外 | 单向依赖：扩展机制依赖"接入契约"，但契约被"不新增模块"提前封死 | **LOW / P1** | Q10 答案为"不能"；场景 12 必为 ARCHITECTURE_CHANGE |
| **H-8** | **全必填字段 + 枚举在响应合同里复制** | `memory-unit` 28/28、`person-profile` 10/10、`event` 15/15、`emotion-context` 14/14 required；`semantic_type` 跨 16 文件 72 次、`anchor` 跨 28 文件 219 次、`read_mode` 91 次、`section` 83 次；DD-V2 §7「统一返回 MUST 含 semantic_type/anchor/source_type/confidence/evidence_refs/lifecycle/visibility」 | 07 §7.3「MUST NOT 修改旧版本字段语义」+ 07 §7.1 major 规则——任何新增字段都沿"复制链"扩散 | **LOW / P1** | 破坏半径无法局部化；表6 中 5 个合同判 HIGH |

**补充取证（单一所有者判定）**：Outbox / FTS5 / LanceDB **确实是单一所有者**（M01 写、Worker 消费 Outbox 后经 Index Adapter 写索引；00 §7.3 规则4「索引同步只能由 Outbox → Worker 驱动；MUST NOT 双写」、03 §7.8 事务边界表、07 §7.12 重建六模式）——这一项是 **EXISTING DESIGN 的强项**，不计为隐藏耦合。但 00 §7.3 把 Worker→Outbox 标 `R/W`、Worker→索引标 `A`，与 03 §7.8「索引写入 = 事务提交后异步（否）」存在 **A vs E 语义差**（计入表2 第 4 点，MEDIUM）。

**config 被几个模块读（实证）**：15 §7.2 三层次覆盖矩阵显示 `recall.*` 读方 = ContextBuilder/Ranker/AnchorRouter/IntentResolver/GreedySelect/EvidenceExpander（M02 内部 6+ 组件）；`anchor.*` 读方跨 IntentResolver/AnchorRouter/ContextBuilder/Ranker（M02）+ retrieval 镜像；`permission.*` 读方 = **所有读写路径 + ContextBuilder + Pipeline stage 15**；`vector.*` 读方 = Embedding Adapter + LanceDB Adapter + VectorRetriever（M01+M02）。→ **没有哪个 config 是真正"单模块私有"**，最低也是跨 M01/M02 两个模块。

**§6 总判定：LOW**。8 项中 6 项 LOW、2 项 MEDIUM；全部有文档内证据，不依赖"未实测"推断。

---

## §7 改善方向（全部为 NEW PROPOSAL，逐条对应 LOW 项；不得写成已批准）

> 以下每条对应 §3/§4/§6 判为 LOW 的项，标注优先级。**均为建议，未获批准，不改变现有裁决（D-AP-03、LOCKED、FUTURE、CR-*）**。

| ID | 对应 LOW 项 | NEW PROPOSAL（建议内容） | 优先级 | 预期收益 |
|---|---|---|---|---|
| **E-1** | Q1/Q2、表1（接口未冻结）、H-6 | 为 M02–M05 各补一份《接口冻结表》：输入字段/输出字段/错误码（引用 03 §7.10 16 规范码）/幂等键/版本契约/超时/权限契约；同时给 00 §7.3 的 A/R/C 矩阵补"机读契约清单 + 每格对应的接口 ID"。先做版本对齐（把 03-m0*.md 标注为 R3 并明确其相对 R4.1 的取舍），再冻结接口。 | **P1** | 使"改动范围"可按契约计算而非按文档估算；消除 H-6 |
| **E-2** | 表3/表6、H-1、H-8、场景 4/5/14 | 合并 `memory-object.schema.json` 与 `memory-unit.schema.json` 为**单一权威 schema**（或明确一者为投影并给字段映射表）；把 `semantic_type`/`anchor`/`section` 从"封闭枚举 + 多合同复制"改为**受控词表 + 注册登记表**（新增值自带默认路由/排序权重/权限默认/迁移映射），响应合同只引用不复制。 | **P1** | 场景 4/5/14 从 ARCHITECTURE_CHANGE/MULTI_MODULE_CHANGE 降为 CONTRACT_CHANGE + 注册项 |
| **E-3** | H-3、H-7、场景 12、Q10 | 在 M01/Worker 与 M04 之间**唯一化任务调度 owner**（建议：Outbox/Worker 管"投递与幂等"，M04 管"业务任务 DAG"，Dream 定义为 M04 内的 `task_type` 插件而非第六模块）；补《扩展点接入契约》：事件订阅点、权限挂载点、版本协商、数据所有权申请、拔除测试矩阵（在 00 §7.7 现有 7 条上扩展）。 | **P1** | Q10 从"不能"变为"可经受控扩展点新增"；消除 H-3/H-7 |
| **E-4** | Q5、表5、场景 3/7/13 | 补 `config/llm.schema.json`（承载 `ingest.*` 6 参数）；为 15 §7.17 预留 ID 与 `storage.busy_timeout_ms`/`worker.claim_timeout_ms`/`core.log_level`/`security.audit.*` 等未登记 config 字段补 ID；明确定义 **"热改 vs 重启生效"两档** + 配置变更事件（进 03 §7.6 事件协议）。 | **P2** | 使 CONFIG_ONLY 场景名副其实；消除 H-2 的镜像维护负担 |
| **E-5** | 表6、Q3、H-8 | 在 07 §7.1 `VersionRegistryEntry` 增加必填列：`producer` / `consumers[]` / `blast_radius`（引用文件数）；对每个共享合同生成"生产者—消费者矩阵"，作为 major 变更审批的强制输入。 | **P2** | 破坏半径可机读；支撑场景 4/6/14 的变更评审 |
| **E-6** | 场景 2（LOW）、Q6 | 若未来允许更换向量库：把 INV-03 从"LanceDB 具体产品"解耦为"独立向量检索层 + 能力探测（FLAT/维度/度量/重建）"，并新增 `vector_store_interface` 契约；须与 `vector.store` LOCKED 的裁决一并重新提交，**本建议不主张现在解锁**。 | **P2** | 场景 2 从 ARCHITECTURE_CHANGE 降为 CONTRACT_CHANGE（需裁决） |
| **E-7** | 场景 4、B 路结论 | 按 02 号交付件（正交分面池）把"新增长期记忆区块"表达为**新增分面/标签注册项**而非新 semantic_type 枚举；配套 `semantic_tags` 扩展登记流程（DD-V2 §2.2 已允许"扩展必须登记"，可复用）。 | **P1** | 与 B 路结论对齐，避免把"分面"误做成"树/区块" |
| **E-8** | 场景 6、B 路 B-C001（P0） | 先按 B 路补 `Relation.endpoint_type` 与自动跨区块联合召回，再评估 `relation_type` 扩展；否则"增加类型"只在 8 值枚举里打转，不解决无类型化跨对象关系的结构缺口。 | **P0（衔接 B 路）** | 避免把 CONTRACT_CHANGE 误判为结构修复 |
| **E-9** | 场景 8、H-8 | 登记单一 `decay.*` 参数族，显式声明 M02 检索衰减与 Psychology ΔE 的 D 是否同源；把 00 §7.6 红线 2（`last_retrieved` 不得作正向权重）写成可执行断言。 | **P2** | 消除两处 D 口径漂移 |
| **E-10** | 场景 9（**P0 阻断**）、A 路 A-C001–A-C004 | 按 01 号交付件补：tier 状态机与转换条件、L1 存储与 owner、转换写入者、保留策略；完成后再重新评估"改 Tier 晋升规则"的变更类别。 | **P0（衔接 A 路）** | 解除场景 9 的 P0 阻断；本路不重复 A 路工作 |
| **E-11** | 场景 12、10、11、Q10、07 §7.13 | 为 FUT-01…08 补统一的《扩展点启用模板》：触发判定（已有）、**接入接口签名**、权限挂载点、数据所有权声明、拔除测试、回滚，全部引用 E-3 的扩展点契约；明确 Persona/Psychology/Dream 当前 FUTURE 状态不得因本模板视为已批准。 | **P1** | 场景 10/11/12 的"当前无模块可改"变成"有受控启用路径" |
| **E-12** | H-4、H-5、表2 | 出具一份《调用链单一事实源》裁决：明确 ① LLM 通道是否仅 M04 §6.2（若是，则 M03 抽象 Provider 须改为对 M04 Gateway 的客户端）；② M02→M04 关系一跳是 A 还是 C（若 M04 FUTURE，则 M02 关系扩展须给降级路径）；③ Worker→索引改标 `E` 而非 `A`。 | **P1** | 消除 3 处文档内单向依赖冲突 |

**纪律声明**：以上 12 条全部为 **NEW PROPOSAL / 建议**，未获用户或台账批准；其中 E-8/E-10 明确标注为**衔接 A/B 路已知结论**，本路不重复其论证，也不改写其 P0 判定。

---

## §8 结论

**整体结论：模块独立性 = LOW；调整成本框架的设计完整度 = MEDIUM–HIGH；"新增模块而不重写核心" = 当前不成立（LOW）。**

1. **EXISTING DESIGN（写得完整、可据此执行）**：00 §7.3 依赖矩阵与 5 条硬性依赖规则、§7.6 职责边界 + 6 条跨模块红线、§7.9 十二条不变量；03 §7.2 通信禁令 8 条、§7.4 M01 七函数、§7.5 六动作、§7.6/§7.7 Outbox/Worker 事件协议、§7.8 事务边界、§7.10 16 规范码、§7.11 八类配置；07 §7.1 八维版本、§7.2 迁移七要素、§7.4 Embedding 九步状态机、§7.5 B-010 八字段、§7.6 九必备四禁令、§7.9 回滚六明确、§7.10 算法替换六前提、§7.12 重建六模式、§7.13 FUT-01…08；15 号 42 参数 × 15 属性 + 三层次 + 四值状态 + 八步变更；contracts 31 个 .json（11 config + 20 schema）。**尤其 Outbox/FTS5/LanceDB 单一所有者、索引可重建、权限默认 deny 被 const 锁死，是真正确立的解耦点。**

2. **DESIGN COMPLETENESS（有名字、输入/输出/步骤/条件/责任/边界大体齐备，但不能读作"算法完整"或"已落地"）**：M03/M04/M05 的接口多为"示例/未冻结/PROPOSED"（03-m01 §7.1、03-m04 §12.4、03 §7.3）；`permission.m05_mode=contract_deny` 明写 M05 未实现；07 的迁移/回滚步骤执行主体是"运维/批准人/配置管理员"而非绑定模块接口；15 号多处 TBD 至实测与"未落地"；03-m0*.md 为 R3 基线且与 R4.1 版本错位。**按纪律：不得把「未实测」当「不可行」（07 §13.1「当前全部 NOT RUN」只表示未演练，九步状态机本身成立）；也不得把「有名字」当「算法完整」（Tier 有 memory_tier 字段但 A 路证明 0 条转换规则、0 写入者、0 保留策略 = P0）。**

3. **RECOMMENDED ADDITION（NEW PROPOSAL，未批准）**：12 条改善方向（§7），其中 **E-8/E-10 为衔接 A/B 路 P0**、**E-1/E-2/E-3/E-7/E-11/E-12 为 P1**、其余 P2。核心方向是：合并双 memory schema、受控词表替代封闭枚举、唯一化调度/LLM owner、补接口冻结表与扩展点接入契约、给每个共享合同登记破坏半径。

4. **对"是否具备构建长期记忆聊天 AI 的完整设计基础"的本路回答**：**部分具备，但模块独立性与可调整性不足以支撑"低调整成本地演进"**。R4.1 把 M01+M02 主链、迁移/版本/回滚/参数治理做得很扎实（场景 1/3/7/13 可低成本调整，HIGH）；但 M03/M04/M05 仍 FUTURE、A/B 路 P0 未补、8 项隐藏耦合（H-1…H-8）、57% 变更场景影响多模块、D-AP-03 禁止第六模块 → 一旦需要"新增记忆区块 / 改 Tier / 改画像 / 新增 Dream"（这正是长期记忆聊天 AI 演进的高频动作），成本即升到 MULTI_MODULE_CHANGE 或 ARCHITECTURE_CHANGE。**本路结论不推翻 A/B 路，而是补充：即使 A/B 路的 P0 全部补齐，若不做 E-1/E-2/E-3/E-12 的解耦，后续每次能力演进仍会外溢到全链。**

**逐项 HIGH/MEDIUM/LOW 汇总**：HIGH = 场景 3/7/13、Q4(数值层)、Q5(参数体系)、Outbox/FTS/向量单一所有者、07 迁移设计；MEDIUM = 场景 1/6/8/10/11、Q2、表2、表5、H-2/H-5；LOW = 场景 2/4/5/9/12/14、Q1/Q3/Q9/Q10、表1/表3/表6、§6 总体。

---

## §9 方法附录

**A. 阅读清单执行记录（按顺序，控制总量）**
| 序 | 文件 / 章节 | 读取方式 | 单次行数 |
|---|---|---|---|
| ① | `00-总体架构与设计原则.md` §7.3/§7.6/§7.7（含 §7.8/§7.9 连带） | 标题列举（Select-String `^#{1,3} `）后 `read` offset=306 limit=175 | 175 |
| ② | `DD-AUTHORITY-V2.md` §5（16 阶段）/§10（P0–P6） | `read` offset=311 limit=115 | 115 |
| ③ | `03-m01.md` / `03-m02.md` / `03-m03.md` / `03-m04.md` / `03-m05.md` | 各前 80 行（并行 5 个 read）+ 接口/依赖章节（§7.1/§9、§9/§10/§11、§1.2/§12、§10） | 80 + 60–150 |
| ④ | `07-扩展与演进规范.md` §7.1–§7.13 | `read` offset=149 limit=175 + offset=324 limit=125 | 175 + 125 |
| ⑤ | `03-系统运行与协作协议.md` §7.2/§7.4/§7.5/§7.6/§7.7/§7.8/§7.10/§7.11 | 3 次 read（20 / 55 / 90 行） | ≤90 |
| ⑥ | `contracts/config/*.schema.json`（11 个） | pwsh `ConvertFrom-Json` 提取顶层字段/子字段/const/default/required + 统计 `$ref` | 非全文通读 |
| ⑦ | `contracts/schemas/` 共享合同 | pwsh 统计 20 个 schema 的字段数/required/enums/refs；重点 memory-unit / relation / anchor-set / person-profile | 非全文通读 |
| ⑧ | `15-参数注册表.md` 目录 + §7.2/§7.4–7.12/§7.16/§7.17 | 标题列举 + `read` offset=166 limit=145 + offset=370 limit=42 | 145 + 42 |
| ⑨ | 前轮取证（A/B 路交付件、FORENSIC 结论） | **按任务书"已知结论不要重做"**，只引用 01/02 号交付件标题与编号；未通读 FORENSIC 49 个 FC 条目，仅引用 A-C001–A-C004 / B-C001–B-C009 编号 | — |

**B. 目录实测（口径纠正）**
- `contracts/config/` = **11** 个 .json：anchor / core / embedding / emotion / memory / profile / retrieval / security / storage / test / worker（**无** `llm.schema.json`，15 §7.2 明示"P3 待建"）。
- `contracts/schemas/` = **20** 个 .json；`contracts/` 合计 **31** 个（任务书写的"schemas 31 / config 12"应为总数与分布记串，本交付件按实测记录）。
- 基线根 `00–15` 共 16 份主文档 + DD-AUTHORITY-V2.md 等；`03-m0*.md` 位于 `C:\Users\[USER]\Desktop\work\` 而非 Design 目录（任务书清单已正确指出）。
- `02-核心算法规范.md` 含 NUL 字节，本路**未读取**原文；清洗副本 `EVIDENCE\B\02-clean.md` 亦未使用（本路不依赖 02 号算法正文，仅引用 00 §7.8 的 EXP-DEFAULT 数值表与 07 §7.10 的算法替换前提）。

**C. 关键词 grep 实证（判定破坏半径）**
| 关键词 | 命中次数 | 分布文件数 | 主要分布 |
|---|---|---|---|
| `anchor` | 219 | 28 | 11 号(54) / anchor-set.schema(24) / 14 号(23) / anchor.schema(14) / 09 号(13) / DD-V2(13) / 15 号(12) … |
| `read_mode` | 91 | 9 | 14 号(29) / anchor-set(14) / 13 号(14) / read-response(11) … |
| `section` | 83 | 10 | 12 号(36) / person-profile(12) / 14 号(11) / profile.schema(6) … |
| `semantic_type` | 72 | 16 | 11 号(11) / 09 号(10) / DD-V2(9) / 12 号(7) / memory-unit(6) / 14 号(6) … |
| `semantic_tags` | 40 | 12 | 11 号(10) / memory-unit(7) / DD-V2(4) … |
| `memory_tier` | 21 | 14 | memory-unit(6) / 00(2) / DD-V2(2) / 11 号(2) … |
| `relation_type` | 8 | 4 | 01 号(4) / relation.schema(2) / 14 号(2) |

**D. schema 字段数速查（20 个，供复核）**
`memory-object 29(15)` · `memory-unit 28(28)` · `search-document 21(14)` · `evidence 21(7)` · `candidate 21(7)` · `read-request 17(14)` · `permission-context 16(4)` · `embedding-space 15(12)` · `event 15(15)` · `emotion-context 14(14)` · `observation 14(11)` · `outbox-event 13(7)` · `retrieval-request 12(5)` · `context-response 11(7)` · `read-response 10(9)` · `error 10(2)` · `relation 10(7)` · `person-profile 10(10)` · `retrieval-response 8(4)` · `anchor-set 5(4)`。（config 11 个字段树见 表5 / pwsh 输出。）

**E. 写文件与工具纪律**
- 交付件：`E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\05-MODULE-ADJUSTABILITY-MATRIX.md`，全程只用 `write`（建骨架）+ `edit`（逐章追加），未用 PowerShell `Set-Content/Out-File/Add-Content`。
- 过程报告：`E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\SUBAGENT-REPORTS\E-REPORT.md`（同目录，另文件）。
- 只读基线：未修改 `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\` 下任何文件；未进入 `node_modules` / `pydeps` / `evidence` 无关目录；未通读 FORENSIC 49 条 FC。
- 分块：单次 `read` 最大 175 行（≤350 行上限）；先列标题再按需 offset/limit。
- 表述：正文简体中文；字段名/枚举/模块名/章节号保留原文；每条结论按 EXISTING DESIGN（带文件+章节）/ DESIGN COMPLETENESS / RECOMMENDED ADDITION（NEW PROPOSAL）三分类；优先级 P0/P1/P2；未把"未实测"当"不可行"、未把"有名字"当"算法完整"。





