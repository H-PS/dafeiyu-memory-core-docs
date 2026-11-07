# Memory Core R4.1 — 核心能力完整性与架构可行性最终审查报告

**任务编号**：MC-CORE-CAPABILITY-REVIEW-01
**日期**：2026-11-06 · Asia/Shanghai
**性质**：只读能力审查。未修改任何正式文档 / Schema / API / DDL / 裁决台账；未编写 V0.9 产品代码；未改动 `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design` 下任何文件。
**审查方式**：A–G 七条独立工作流并行取证 + 父 Agent 交叉复核（本文对七份报告做了合并同类项、权重下调/上调与独立判断，**不是拼接**）。
**配套证据**：`01`–`07` 七份能力附件 + `SUBAGENT-REPORTS\A–G` + `EVIDENCE\`（机器校验脚本与原始输出）。

> **阅读指引**：本报告可独立阅读。第 1–3 章给结论与事实基线；第 4–10 章按七项核心能力逐项判定；第 11–13 章是权责冲突与 P0/P1 登记；第 14–17 章是继承清单、最小补齐、推荐架构与 V0.9 边界；第 18 章是最终结论与十一个必答问题。每章末尾的 **判定** 行可直接引用。

---

## 1. 执行摘要

### 1.1 一句话结论

**这套 Memory Core R4.1 的"架构骨架"是成立的、且在设计密度上明显高于同类项目；但"从一句话到一次被正确回忆"的完整闭环，在 7 项核心能力里有 4 项只到概念或半成品——短期记忆没有存储与主人，长期记忆的层级轴没有一条转换，跨区块关系不是一等公民，Persona 根本没有进入契约，情绪链路被三道 `const:false` 与一张不存在的表封死。**

不是方向错了（所以不是 D），也不是只差打磨（所以不是 B），而是 **规范层与执行层成片缺失**（C）。

### 1.2 五个必须最先知道的事实

| # | 事实 | 证据 |
|---|---|---|
| 1 | **设计层几乎全绿，执行层大片红**。17 项能力都有落点文档（`DD-AUTHORITY-V2 §1`），但 5 个权威对象里 **3 个没有存储表**，全语料 **0 份可执行 DDL**（13 张表只是 Markdown 列清单） | `01 §7.16`、`01:3` 头注自相矛盾；A-路 §7；G-C003 |
| 2 | **`memory_tier` 是标签不是体系**：8 个状态机 57 条转换中 **0 条在 tier 轴**；`memory_tier` 只出现在 31 份机器合同的 **1 份**、13 张表的 **0 张**；晋升 0 条、降级 0 条、保留策略 0 条 | A-C001…A-C004；`contracts/state-machines/state-machines.md` |
| 3 | **读取侧有两套互斥算法，且被授权实施的那套绑着已废止的 v1 字段**：`02 ALG-04` 全局六特征线性排序 vs `11 §7.17` 分组装配"禁止单一全局权重"；ALG-04 的 4 个特征依赖 `memory-object`(v1) 独有字段，`memory-unit`(v2) 一个都没有 | F-C001；G-C001；`02 §7.5`、`11 §7.17`、`memory-unit.schema.json` |
| 4 | **Persona 在 canonical 里是 FUTURE**：`00 §7.7` 只给它一句"记忆消费者，经授权视图接收记忆"，无 schema、无参数、无状态机、无接口；能力全靠宿主侧手写人格卡（实测占注入量 **49.8%**，且放在 user 层会被 compaction 吃掉） | D-C001/D-C002；`00 L406`、`08 L338`；`07` 文档 §2.2 |
| 5 | **情绪是"写得进去、读不出来、影响不了"**：EmotionContext 的 14 字段与 ΔE 公式是全文最完整的设计，但 `emotion.query_enabled`/`influence_enabled`/`auto_infer` 三开关 `const:false`、`read-response.emotion_context.maxItems:0`、且无 `emotion_contexts` 表 | C-C001/C-C002；`contracts/config/emotion.schema.json`、`read-response.schema.json` |

### 1.3 七项能力评级（详见第 12 章）

| 能力 | 评级 | 一句话 |
|---|---|---|
| ① 短期与长期记忆 | **PARTIAL** | 长期存储有；短期无家、层级无路 |
| ② 长期记忆分区与关联 | **PARTIAL** | 分面池成立；跨对象关系与联合召回缺 |
| ③ 情绪附件 | **PARTIAL** | 对象与门禁完整；存储与运行时封死 |
| ④ Persona Prompt | **CONCEPT_ONLY** | 初始化与逐轮组装两端都没有规格 |
| ⑤ 信息读取与利用 | **PARTIAL** | 接口面丰富；契约三套互斥、末端注入不成立 |
| ⑥ 模块独立可调整 | **PARTIAL（LOW）** | 文件分了模块，依赖没分 |
| ⑦ 核心算法 | **MOSTLY_COMPLETE** | 检索/写入主干可实施；21 条重点中 10 条带缺口 |

### 1.4 判定框

```
CORE_ARCHITECTURE_STATUS      = PARTIAL
MAJOR_ARCHITECTURAL_CONFLICTS = PRESENT
整体系统结论                   = C（核心能力存在系统性缺失，但架构不需推倒）
进入 V0.9 的条件               = 不具备（需先完成 5 项裁决 + 1 次 schema 修订 + 1 份 DDL + 4 条能力链补齐）
```

---

## 2. 当前架构真实状态

### 2.1 有什么（EXISTING DESIGN，逐项可核）

| 资产 | 内容 | 位置 |
|---|---|---|
| 五个权威对象 | `MemoryUnit`(28 字段) · `Evidence`(13) · `Event`(15) · `EmotionContext`(14) · `PersonProfile`(10)，字段级定义齐备 | `DD-AUTHORITY-V2 §3.1–§3.5` |
| 31 份机器合同 | 20 个 schema + 11 个 config（另有 3 份 md：api/state-machines/error-codes），715 个字段，31/31 通过 draft-07 元校验 | `contracts/`（前轮实测） |
| 规范枚举 | `semantic_type` 6 · `semantic_tags` 12 · `anchor` 8 · `subject_type` 6 · `intent` 9 · `read_mode` 4 · `temporal_position` 4 · `emotion_label` 15 · `relation_type` 8 · `source_type` 5 · `status` 4 · `candidate.status` 6 · `visibility` 3 · `scope` 5 | `DD-AUTHORITY-V2 §2` |
| 五个模块 | M01 存储权威 · M02 检索编排 · M03 候选提取 · M04 整合裁决 · M05 权限 | `00 §7.3/§7.6` |
| 16 阶段摄取链 | `InputNormalizer` … `M03/M04 → M01 Commit → Outbox → FTS5/LanceDB` | `DD-AUTHORITY-V2 §5`、`09` |
| 9 个读写接口 | `search/getProfile/getByAnchor/getEvent/getEvidence/getRelated/getEmotionContext/getMemoryById/propose`，逐接口给了输入/输出/权限/幂等/超时/错误码 | `14 §7.2` |
| 五阶段检索 | 意图识别→锚点路由→锚点候选→排序→权限过滤→证据扩展与上下文构建 | `DD-AUTHORITY-V2 §8`、`11 §7.17` |
| 12 个算法 | `ALG-01` 写入 … `ALG-12` 评估，统一 17 要素（输入/输出/前置/步骤/公式/参数/边界/异常/幂等/测试/验收/禁止） | `02`（经清洗副本 `EVIDENCE/B/02-clean.md`） |
| 8 个状态机 | SM-1 生命周期 … SM-8 嵌入迁移，共 57 条转换 | `contracts/state-machines/state-machines.md` |
| 情绪公式 | `ΔE = I × C × R × D` + 8 条门禁（默认关闭、噪声不触发、单事件上限、多事件总量上限、只作用运行时） | `DD-AUTHORITY-V2 §9`、`13 §7.9/§7.10` |

### 2.2 没有什么（DESIGN COMPLETENESS: MISSING / CONFLICTED）

| 缺什么 | 后果 | 证据 |
|---|---|---|
| **0 份可执行 DDL** | M01 无法开工；13 张表只是列清单 | `01 §7.16`、FC-009 |
| **`events` / `emotion_contexts` / `person_profiles` 三张表不存在** | 事件、情绪、画像三个对象无处落库 | G-C003；`01:3` 自述需新增、`§7.16` 却没有 |
| **`memory_tier` 无列、无索引、无写入者、无迁移行** | 层级永不变化；直读 L3 无法表达 | A-C002/A-C006 |
| **tier 转换 0 条** | 短期记忆永远进不了长期 | A-C001；57 条转换中 0 条 |
| **`Relation` 两端只有 `memory_id`** | 人物↔事件、项目↔事件、事件↔情绪无法成为类型化关系 | B-C001；`relation.schema.json` |
| **宿主返回契约 3 套、字段零重合** | 任何 schema 合法响应都不能被宿主注入（实测） | G-C002；`context-response.schema.json` vs `08 §7.7` vs `03 §7.3.4` |
| **Persona 无 schema/参数/状态机/接口** | 稳定人格无法程序化初始化与装配 | D-C001；31 合同中 0 个 persona |
| **情绪三开关 `const:false` + `maxItems:0`** | 情绪回取与影响在契约层不可达 | C-C001；`config/emotion.schema.json`、`read-response.schema.json` |
| **`importance` 无赋值算法、`confidence` 无合成公式** | 过滤/并列/schema 三阈值分流输入不可信 | F-C002 |
| **FTS 分词器 TBD + 索引投影回退 TBD** | 中文检索验收无法执行 | F-C003 |

### 2.3 四层真实状态（左绿右红是本项目最重要的结构事实）

| 层 | 状态 | 说明 |
|---|---|---|
| 设计层（00–15 正文） | **基本完整** | 17 项能力全部有落点；对象/枚举/流程/禁令齐全 |
| 契约层（31 JSON） | **结构性合法、语义上互斥** | 两套对象模型、4 套 `source_type`、6 套 `status`、3 套返回体 |
| 存储层（13 张表） | **不存在** | 0 DDL；3 个对象无表；`memory_unit` 14 个 V2 字段无列 |
| 执行层（算法/状态机） | **开环** | tier 0 边；M03/M04/M05 全 FUTURE；候选池禁建；写入默认态非法 |

---

## 3. 核心产品目标还原（从产品反推，而不是从文件反推）

### 3.1 我们要的产品能力

一个能长期陪用户的聊天 AI，必须在一次对话里完成这七件事：

1. **接住当下**：本轮消息、当前话题、进行中的任务，必须在不落库的前提下被理解。
2. **记住值得记的**：把"我不吃香菜"这类陈述变成以后可用的记忆，并保留原句。
3. **分层管理**：琐事会过期，稳定事实不会；重要经历要留，临时状态要淡出。
4. **组织经历**：人是人、事是事、项目是项目、关系是关系；但"那次答辩"要能同时连到人、项目、情绪。
5. **理解用户**：问"我是谁"时先给自述与画像，问"去年干了啥"时先给事件与原句。
6. **带情绪但不被情绪绑架**：情绪是附件，能取回、能轻微影响语气，但永不改写事实。
7. **保持人格**：不管检索到什么，AI 还是那个 AI；人格不因一条记忆漂移。

### 3.2 反推出的 12 条必需数据流

```
F01 会话消息 → 会话上下文（不落库）
F02 会话消息 → 候选 → 确认 → MemoryUnit（权威）
F03 MemoryUnit → 原句 Evidence（可回溯）
F04 候选/记忆 → tier（L1/L2/L3）→ 晋升/降级/过期
F05 MemoryUnit → 语义区块（分类）+ 多锚点（获取）
F06 MemoryUnit/Event/Profile/Project → 类型化关系
F07 事件 ↔ 情绪（多条、按时间、按来源）
F08 画像 ← 记忆整合（11 分区，basis 分离）
F09 查询 → 意图 → 锚点路由 → FTS/向量 → 排序 → 权限 → 预算 → 上下文
F10 上下文 + 人格 + 画像 + 情绪 → Prompt → Chat LLM
F11 命中反馈 / 用户纠正 → 记忆修正（版本、SUPERSEDE）
F12 删除 / 恢复 → 派生索引与引用的一致性
```

**判定：F01、F02、F03、F09 有正式设计；F04、F06、F07、F10 缺失或封死；F05、F08、F11、F12 部分。** 后文按这 12 条流逐项核对，是本报告"从产品反推"的主线。

---

## 4. 短期与长期记忆完整性（目标一）

### 4.1 三层记忆完整性矩阵（必交表 6.1）

> 说明：`memory_tier` 是**生命周期层级**；`00 §7.5` 另有一套**时间尺度三层**（Session Context / Working Memory / Long-term）。两者是不同轴，下表第 1–3 行是 `memory_tier`，第 4–6 行是时间尺度层。**这正是当前设计被误读的根源，见 §4.3。**

| 层级 | 作用 | 主要数据 | 存储位置 | 写入者 | 读取者 | 进入条件 | 退出条件 | 晋升条件 | 降级条件 | 保留策略 | 检索策略 | 定义完整? |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **L1 临时** | 当前对话/短期任务/未确认状态 | 临时自述、一次性反馈、即时情绪 | **∅（无表；`session_state` 为 FUTURE 且禁止 P0–P3 创建）** | **∅** | M02（理论上） | M04 裁决或 bootstrap | ∅ | **禁止 L1→L3** | ∅ | ∅（无 TTL、无清理任务） | 可检索但无 store | **NO** |
| **L2 活跃** | 推进中计划/近期事件/阶段状态 | 阶段目标、关系变化观察、近期事件 | `memory_objects`（无 `memory_tier` 列） | 声明 M04；实际 ∅ | M02 | 同上 | ∅ | L2→L3 条件未定义（"稳定/明确/多次"不可判定） | ∅ | ∅ | 同表检索 | **NO** |
| **L3 长期** | 稳定人物/偏好/关系/经历/目标 | 自述、画像分区、长期偏好、MILESTONE | 同上 | 声明 M04；实际 ∅ | M02 / `getProfile` | 同上 | ∅ | —（终层） | ∅（无降级） | ∅（无复核） | 优先级由意图决定，非 tier | **NO** |
| **Session Context** | 本轮主题/任务/未完成问题 | 对话状态、临时约定 | 宿主（Memory Core 外） | 宿主 / Session Memory（FUTURE） | 宿主 | 会话开始 | **会话结束即丢弃** | **默认不自动晋升** | — | 会话级 | 不进入 Memory 检索 | **CONCEPT_ONLY** |
| **Working Memory** | 跨会话话题/未完成目标/阶段计划 | 待确认事实、临时关系状态 | 无独立存储 | 无 | 无 | 未定义 | 未定义 | "需 Evidence+Validation+Review"，无算法 | 未定义 | 无 | 无 | **CONCEPT_ONLY** |
| **Long-term** | 稳定事实/长期偏好/重要经历 | 同 L3 | 同 L3 | 同 L3 | 同 L3 | 同 L3 | 同 L3 | 终层 | — | 无 | 同 L3 | **PARTIAL** |

**矩阵读数**：3 个 tier × 4 个生命周期列 = 12 格，其中 **9 格为 ∅**（退出/降级/保留全空），3 格仅为意图（进入/晋升），第 13 列三层全 **NO**。

### 4.2 短期 → 长期：唯一被认可的路径，以及它断在哪

```
用户输入 ──► 候选（M03） ──► 审核（M04） ──► M01 写入 ──► tier 赋值 ──► 索引 ──► 检索
   ✅            ⚠️            ❌              ❌           ❌           ✅         ✅
                (候选池禁建)   (P4 未授权)    (默认态非法)  (无列/无转换)
```

- **路径文本存在**：`00 §7.5` 写明"需 Evidence + Validation + Review 后才可晋升"；`11 §7.12` 规则 1 写明"最终 tier 由 M04 整合裁决，M03 只给 `lifecycle_candidate`"。
- **但每一步都缺执行体**：`candidate.schema.json` 的 `additionalProperties:false` 里**没有 `lifecycle_candidate`**（A-C008）；`candidate_pool` 被 `01 §7.16` 明令"P0–P3 不得创建"（G-C004）；M04 属 P4"待阶段授权"（`DD-AUTHORITY-V2 §10`）；`memory_tier` 无列（A-C002）。
- **结论**：这条路径是**文本闭环、执行开环**。用户说"记住我不吃香菜"，最远能走到"候选生成"，走不到"一条带 tier 的长期记忆"。

### 4.3 三层 ≠ 三种语义类别：文本上分得清，可执行处全混

| 轴 | 取值 | 谁在用 | 是否同一回事 |
|---|---|---|---|
| `memory_tier` | L1/L2/L3 | `memory-unit` 必填字段 | 生命周期层级 |
| 时间尺度 | Session Context / Working / Long-term | `00 §7.5` 组织维度 | 组织位置 |
| LLM 层 | L1 轻量 / L2 强语义 / L3 聊天 | `10`、`DD-AUTHORITY-V2 §6` | 模型能力分级 |

三者共用 `L1/L2/L3` 记号，语料内 `memory_tier` 出现 11 次，裸 `Lx` 记号 400+ 次（A-C005）。更实质的混淆在**赋值逻辑**：`11 §7.12` 的 3×6 矩阵是 `semantic_type → tier` 先验；`12 L390` 把画像行硬钉 `(L3, ACTIVE)`；`09 L755` 把 `AUTO_ACCEPT` 限定在 L1/L2——**层级被当成了语义类别的函数**。而规范正文又明确要求正交。

**判定（目标一）：PARTIAL。** 长期记忆的"存储 + 检索"存在；短期记忆没有存储与主人；层级转换 0 条；原句追溯在存储层断裂（A-C004：`read-response` 要求 `original_text`，DDL 与 `evidence.schema.json` 都没有）。**"我们有一套完整短长期体系"这一说法不成立——拥有的是一组字段与一套时间尺度词汇。**

### 4.4 必须回答的四个问题

1. **短期信息真的能被长期保留吗？** 不能。没有任何一条 tier 转换；唯一路径缺 3 个执行体（候选池/审核/列）。
2. **长期信息有独立管理机制吗？** 没有。与短期同一张表、同一索引、同一写入者、同一权限、同一检索路径；唯一"长期形状"的 `PersonProfile` 被禁止独立（A-路 Q2）。
3. **转换中会丢信息吗？** 已证一处：`lifecycle_candidate` 被 `candidate.schema.json` 拒收；另有会话结束处置未定义、无降级导致的"永不退出"。
4. **三层是否被错误等同于三种语义类别？** 规范文本清楚、**所有可执行痕迹都混**（唯一赋值指引是 `semantic_type→tier` 矩阵 + 两处硬编码）。

---

## 5. 长期记忆区块与关系结构（目标二）

### 5.1 长期记忆不是列表，但也不是树

`00 §0.1` 明文要求"多维组织，**不采用单一分类树**"。当前实现是**正交分面池**：

```
            ┌──────────── 一条 MemoryUnit（28 字段，同一张表）────────────┐
分类分面 →  │ semantic_type(1/6)  semantic_tags(0..12)  anchor(1..8)      │
主体分面 →  │ subject_type(1/6)   subject_id                                │
生命周期 →  │ memory_tier(1/3)    status(1/4)   validation_state(1/4)      │
权限分面 →  │ visibility(1/3)     scope(1/5)                                │
时间分面 →  │ valid_from/until    created_at/updated_at   observed_at       │
关联分面 →  │ event_refs[]  evidence_refs[]  relation_refs[]  emotion_refs[] │
            └───────────────────────────────────────────────────────────────┘
```

**关键判断：这是"分面池 + 浅层带类型的关系图"，不是"互连区块"。** 区块（人物/偏好/关系/事件…）在数据层**不是实体**，只是同一条记忆上的标签组合。

### 5.2 长期记忆区块矩阵（必交表 6.2）

> 区块集沿用 B 路 9 区块模型（6 个语料已有 + 3 个 NEW PROPOSAL），便于与附件 02 对齐。

| ID | 区块 | 内容范围 | 主要 Anchor | 关联区块 | 数据对象 | 读取优先级 | 典型查询 | 不应包含 | 完整性 |
|---|---|---|---|---|---|---|---|---|---|
| BLK-01 | 人物画像 | 11 分区整合结论 | `PERSON_PROFILE` | 02/03/04/08 | `PersonProfile` + 源记忆 | 人物理解第 2 | "介绍一下我" | 事件流水、情绪、原句 | MOSTLY（全 11 分区默认返回与 `14 §7.4` 冲突） |
| BLK-02 | 自我描述 | 主人明确自述原文性内容 | `SELF_EXPLICIT` | 01/03 | `MemoryUnit(PERSON/PREFERENCE)` | 人物理解第 1 | "我怎么形容自己" | 系统推断 | MOSTLY |
| BLK-03 | 偏好 | 偏好/互动习惯/长期约定 | `PREFERENCE` | 01/02/04 | `MemoryUnit(PREFERENCE)` | 人物理解第 3 | "我喜欢什么回应" | 一次性口味（应 L1） | MOSTLY |
| BLK-04 | 关系 | 关系与关系变化 | `RELATIONSHIP` | 01/06/09 | `MemoryUnit(RELATIONSHIP)` | 人物理解第 5 | "我和小 A 什么关系" | 单次互动当关系 | PARTIAL（无关系实体/无变化序列） |
| BLK-05 | **项目（NEW）** | 项目实体、参与人、阶段、事件集 | 无（8 锚点无 PROJECT） | 06/04/09 | **NEW `Project`** | 事件/状态查询内按项目召回 | "这个项目有哪些里程碑" | `scope=project_scoped` 权限值当对象 | **MISSING** |
| BLK-06 | 事件 | 客观事件事实 | `EVENT` | 01(experience)/02/04/05/08/11 | `Event` + `MemoryUnit(EVENT)` | 事件查询第 1 | "上周三我做了什么" | 情绪判断/评价/未来计划 | MOSTLY（反向"某人的所有事件"未定义） |
| BLK-07 | **稳定事实（NEW）** | 非人物的稳定事实（领域知识/流程） | 无专门锚点 | 05/06/01 | NEW（`semantic_tags` 子类或第 7 类） | 通用查询稳定层 | "这个项目的部署流程" | 被默认塞进 `PERSON` | **MISSING/CONFLICTED** |
| BLK-08 | 经历 | 重要经历（MILESTONE） | `EVENT` + `PERSON_PROFILE(experience)` | 01/06/02 | `MemoryUnit(EVENT)` + 画像 experience | 人物理解第 6（条数受限） | "我拿到学位那次" | 把 Event 拆成多条画像结论 | MOSTLY |
| BLK-09 | **情境记忆（NEW）** | 某次互动的场景/氛围/上下文 | 无（`EMOTION_CONTEXT` 最近） | 06/07/03/04 | NEW（`interaction/session` 对象） | 情绪/关系查询按需 | "那次聊到深夜" | 与 Session Context 混为一体 | **MISSING** |
| BLK-10 | 支撑层：证据 | 原句与上下文 | 无锚点（`Evidence`） | 全部（追溯） | `Evidence` | 按 `read_mode` 按需 | "我原话怎么说的" | 被规范化覆盖 | PARTIAL（无 `original_text` 列） |
| BLK-11 | 支撑层：情绪 | 情绪附件 | `EMOTION_CONTEXT` | 06/01/04 | `EmotionContext` | 默认不注入 | "我当时什么感受" | 写进 Event 正文 | PARTIAL（无表、开关关闭） |

**过大的区块 / 职责混杂**：`PersonProfile` 一个视图混装稳定（identity/value）与易变（current_state/derived）；`MemoryUnit` 一个对象承载分类、层级、权限与 6 个跨对象 ID 数组——**数组是唯一的跨区块链接，却没有完整性传播规则**（B-C007）。

### 5.3 树 / 图 / 混合：三种方案对比与裁定

| 维度 | 方案一 树状分类 | 方案二 关系图 | 方案三 混合（推荐） |
|---|---|---|---|
| 归属 | 每个节点唯一父 | 无归属概念 | 分面/区块归属（准树，可多标签） |
| 跨区块 | 只能靠祖先路径 | 类型化边 | 类型化边（浅层、有界） |
| 同一条记忆属多个区块 | 需复制节点（破坏唯一性） | 天然支持 | 一条记忆 + 多 anchor/tag（不复制） |
| 检索 | 路径下钻，易漏兄弟 | 图遍历，易爆炸 | 意图→分组→组内排序→按配额装配 |
| 代价 | 语义失真 | 深度/环/权限/预算不可控 | 需两套规则（归属 vs 关联）分明 |
| 语料是否支持 | `00 §0.1` 明文否定 | `02 §3` 图检索 REJECTED | 与 `11 §7.17` 五阶段、`DD-V2 §8` 一致 |

**裁定：HYBRID（混合式）。** 理由五条：

1. `00 §0.1` 明确否定单一分类树，且现实中一条记忆天然多父（"我不吃香菜"同时是偏好、自述、饮食习惯）。
2. 分面分类已经存在且可用（6 类型 + 12 标签 + 8 锚点），不需要推翻。
3. 跨区块关联需要类型化边（人物↔事件、项目↔事件、事件↔情绪），而 `Relation` 目前只能连 `memory_id`（B-C001）。
4. 图遍历被 `02 §3` 明确 REJECTED，且 1200 token 预算下深度遍历不可行；只能做**有界 1-hop**。
5. 检索优先级必须是**第四条独立轴**（意图驱动分组），不能由 tier/importance/时间任一字段代行（`11 §7.16/§7.17`）。

### 5.4 四个"不能合并"的维度（本报告最容易被误读的一点）

| 维度 | 回答什么 | 载体 | 禁止被谁替代 |
|---|---|---|---|
| 分类结构 | 这条记忆"是什么" | `semantic_type` + `semantic_tags` + `anchor` | 不得用 tier 代替 |
| 关系结构 | 它"和谁有关" | `Relation` / `*_refs[]` | 不得用 tag 代替 |
| 存储层级 | 它"留多久" | `memory_tier` | 不得用 importance 代替 |
| 检索优先级 | 本轮"先给谁" | `intent` + `anchor_plan` + 分组配额 | 不得用 tier/全局权重代替 |

现有契约的违反点：`anchor-set.schema.json` 的全局 `weights`（`PERSON_PROFILE 0.35 > EVENT 0.30 > SELF_EXPLICIT 0.20`）把"关系/分类"压成"全局排序"，正是 `11 §7.17` 禁令 1 要禁止的形态（B-C004、G-C007）。

### 5.5 九个硬案例（逐个给结论）

| 案例 | 现状能否表达 | 结论 |
|---|---|---|
| 一个人物 ↔ 多个事件 | `Event.participants[]` + `MemoryUnit.event_refs[]` | **可表达但无类型化边**，反查"某人所有事件"未定义 |
| 一个事件涉及多个人物 | `participants[]`（`subject_id` 列表） | **可表达**，但无角色（参与者/见证者）区分 |
| 一个项目包含多个事件 | 仅 `scope=project_scoped`（且无 `scope_id`） | **不可表达**（无 Project 实体） |
| 一个人物多个身份/关系 | `subject_id` 单一 + `RELATIONSHIP` 记忆 | **部分**，无别名表（`subject_alias` 缺失） |
| 一条记忆属多个语义区块 | `semantic_tags` 多选 + `anchor` 多选 | **可表达**（`11 §7.1` 总则 1） |
| 关系变化如何更新 | `SUPERSEDES` 关系 + 新记忆 | **部分**，无变化序列/时间线 |
| 删除一条记忆时关联怎么办 | `02 §7.9` 有传播清单，覆盖 `Observation`/`PersonProfile`/`Relation` | **不覆盖无类型 ID 数组**（悬空引用已实测，B-C007） |
| 如何避免循环引用 | `01 §7.10` 有"递归检查"约束 | **约束不是算法**，无 visited/深度（B-C008） |
| 检索时递归扩散过多 | 无深度/预算/visited 参数 | **不可控**（`getRelated` 只有 `memory_id`） |

### 5.6 跨区块联合召回：当前最大的"漏信息"风险

`memory.search` 返回**扁平排序集**；`getRelated` 是**手动 1-hop**，无深度/预算/visited 规范；`02 §3` 图检索 REJECTED。结果是：用户问"那次答辩"，系统召回了 `EVENT`，但**不会自动带出**该事件的参与人、所属项目、当时的情绪、以及支撑它的原句。`14 §7.4` 的 5 个场景是**人工编排的调用链**，不是系统的默认行为。

**判定（目标二）：PARTIAL。** 区块作为分面存在，关系只有"记忆↔记忆"且无类型端点；项目、稳定事实、情境记忆三个区块缺失；跨区块联合召回不存在。**"多个有组织、有关系、有优先级、可联合检索的区块"目前只成立前两项的一半。**

---

## 6. 情绪附件及联动（目标三）

### 6.1 数据对象：这是全语料最完整的设计

`EmotionContext` 14 个必填字段（`emotion-context.schema.json`）：

| 字段 | 值域 | 说明 |
|---|---|---|
| `emotion_id` | uuid | 稳定不可复用 |
| `event_id` | uuid 或 **null** | 有事件时必须等于 `Event.event_id`；`null` 表示无具体事件 |
| `subject_id` | string | 情绪**所属主体**（须 EntityResolver 解析） |
| `emotion_label` | 15 受控值 | 越界映射 `neutral` 并保留 `raw_label` |
| `raw_label` | string/null | 原词，不丢失原始表达 |
| `intensity` | 0–1 | ΔE 中的 `I`，禁百分制 |
| `confidence` | 0–1 | ΔE 中的 `C`；`ASSISTANT_INFERRED` 时 ≤0.99 |
| `temporal_position` | BEFORE/DURING/AFTER/UNKNOWN | 相对事件的阶段 |
| `source_type` | USER_EXPLICIT / ASSISTANT_INFERRED | **用户明示与模型推断强制分离** |
| `evidence_refs` | uuid[] | 无证据的情绪不得进正式库 |
| `related_memory_ids` | uuid[] | 关联记忆（无类型） |
| `visibility` | 3 值 | 默认 `owner_private`，`public` 被 `not` 禁止 |
| `scope` | 5 值 | v1 内部细分 |
| `created_at` | RFC3339 | 唯一时间字段 |

配合 `13` 的 SEP-1…SEP-8 硬规则（事实与情绪分离、不内嵌、不写 Event.content、不写 Psychology/PAD）与 `ΔE` 八门禁——**"情绪不污染事实"这一条，设计上是满分。**

### 6.2 情绪联动矩阵（必交表 6.4）

| 来源 | 绑定对象 | 存储方式 | 时间信息 | 强度 | 置信度 | 历史保留 | 检索条件 | 对当前状态影响 | 与 Persona 关系 | 成熟度 |
|---|---|---|---|---|---|---|---|---|---|---|
| 用户明示 | Event（可空）/ Memory | 独立行，`event_id` 外键 | `temporal_position` + `created_at` | `intensity` 0–1 | `confidence` 0–1 | 多条并列，**无版本/无状态** | `EMOTION_QUERY`（默认关） | ΔE（默认关） | 无接口 | PARTIAL |
| 模型推断 | 同上 | 同上 | 同上 | 同上 | ≤0.99 | 同上 | 同上 | 同上（权重折扣） | 无接口 | PARTIAL（`auto_infer=false`，无产出路径） |
| 系统汇总 | 画像/汇总视图 | **无合法 `source_type` 值** | — | — | — | — | — | — | — | **MISSING**（枚举缺 SYSTEM） |

### 6.3 完整链路还原（10 步）

| 步 | 环节 | 判定 | 关键证据 |
|---|---|---|---|
| 1 | 输入 → Evidence | **COMPLETE** | `13 §7.2` 13 字段；`evidence_refs` 要求 |
| 2 | 线索提取（EmotionExtractor） | **PARTIAL** | `09 §7.15` 8 条 MUST；但 `EmotionDraft` 缺 7 个必填字段，映射自登记 `OI-09-06` |
| 3 | Draft → EmotionContext | **MISSING** | 无字段映射算法、无执行模块 |
| 4 | 与 Event/MemoryUnit 关联 | **CONFLICTED** | `13 §7.7.3 SEP-2` 称经 `event_refs` 关联，但 `event.schema.json` **无此字段** |
| 5 | 保存 | **MISSING** | 无 `emotion_contexts` 表；无索引/外键/写入事务 |
| 6 | 检索 | **设计有、契约封死** | `include_emotion const:false`、`anchor.group_quota.emotion=0` |
| 7 | 回取 | **BLOCKED** | `read-response.emotion_context.maxItems:0` |
| 8 | 当前交互使用 | **BLOCKED** | `influence_enabled const:false`；Psychology BLOCKED |
| 9 | 衰减 | **PARTIAL** | `D=exp(−decay_rate·Δt)` 全局参数；Δt 取哪个时间未定义 |
| 10 | 独立关闭/拔除 | **不成立** | 只有全局三开关 `const:false`；对象无 runtime 标记；J5 清除仅原则句 |

### 6.4 十三问（编号作答，详见附件 03）

1. **数据结构是否充分？** schema 层面 MOSTLY_COMPLETE；全链路不足。
2. **通过什么关联？** `event_id`（→Event）+ `related_memory_ids[]`（→MemoryUnit）+ `MemoryUnit.emotion_refs`（反向）。
3. **一条记忆可关联几条情绪？** 契约无上限、无基数、无唯一约束——**技术上无限，工程上不可控**。
4. **多次情绪记录如何保存？** 多条并列快照；无版本、无 `supersedes`、无时间线。
5. **同一事件不同时间的情绪变化？** 只能靠多条 `temporal_position` 粗略表达；无法表达"从焦虑转为释然"这一变化本身。
6. **来源是明示还是推断？** `source_type` 2 值强制分离，且推断置信度 ≤0.99；但推断默认关闭。
7. **置信度？** `confidence` 0–1，必填、无 `null`，与 `13 §7.8.2`"未知须留空"**冲突**。
8. **强度？** `intensity` 0–1，必填、无标定口径（谁给值、按什么量表）。
9. **如何控制对长期记忆的影响？** 门禁 8"ΔE 只作用运行时、不写回 Event、不改 `validation_state`"——**完全隔离**（设计正确）。
10. **如何控制历史情绪对当前状态的影响？** ΔE 的 `R`（话题关联）与 `max_influence` 上限；但 `R` 的计算口径 TBD、参数三处默认值冲突。
11. **如何避免一次负面情绪影响长期人格？** 默认关闭 + 单事件上限 + 不写持久态；**但一旦开启，无"人格层免疫"规则**。
12. **能否独立关闭/调整/拔除？** 只能全局关；无对象级开关、无按事件/按人物拔除。
13. **Runtime 未启用时 Memory Core 是否正常？** **是**——这是设计得最好的一点：三开关均为配置位，Memory Core 主链不依赖情绪。

### 6.5 唯一的合法"事实污染"路径（父 Agent 补记，C-C007）

设计上把 Event 侧封得很死（`event.schema.json` 明文禁情绪判断、SEP-1/SEP-7），但 `memory-unit.schema.json` 的 `content` 定义只说"规范化后的独立记忆内容"，**没有禁止** `semantic_type=EMOTION_CONTEXT` 的单元把情绪判断写进正文。这样一条记忆会走 FTS5 + LanceDB → RRF → 精排 → `items[]`，与事实共池竞争预算，并被 LLM 当事实读走。门禁 8 与 `writes_persistent_state=false` 只管 ΔE，**管不到这条写入路径**。

**结论**：事实隔离在"事件对象"层面成立，在"记忆单元"层面有一个合法缺口。补法很小：`content.description` 增情绪禁令 + `search-document` 对情绪类设 `indexable=false` + `read-response.items[]` 强制情绪走 `emotion_context[]` + 一条"情绪正文进入 items[] 次数 = 0"的测试。

### 6.6 判定

**PARTIAL（schema 侧 MOSTLY_COMPLETE，全链路 PARTIAL，契约态 BLOCKED）。**
情绪是"设计完成度最高、运行时可达性最低"的一项。它证明了团队有能力把复杂语义对象写清楚；也暴露了一个系统性问题：**能力以 `const:false` 与 `maxItems:0` 的方式被"设计性地关闭"，而不是"以开关方式保留可启用性"**——要开功能就得改 schema，这本身就是架构缺陷。

---

## 7. Persona Prompt 初始化与运行（目标四）

### 7.1 两段流程的真实状态

```
A. 首次初始化
   人格卡(自然语言) ──► ??? ──► 结构化配置 ──► 持久化 ──► 版本
                        ▲
                   全语料无此环节（0 schema / 0 参数 / 0 状态机）

B. 每轮组装
   System 安全+工具协议 ──┐
   人格（手写卡，user 层）──┤
   记忆上下文（1200 token）─┼──► 宿主拼装 ──► Chat LLM
   会话历史 ──────────────┤        ▲
   任务指令 ──────────────┘   08 §7.9 只说"宿主唯一拼装者"，
                              无宿主侧规格；现网是三处复制粘贴
```

- **EXISTING DESIGN（边界）**：`00 §7.7` 把 Persona 定义为"记忆**消费者**；经授权视图接收记忆，形成角色上下文"，标 **FUTURE**；`00 L271` 反面清单禁止 Memory Core 当"人格提示词管理器"；`08 L338` 规定"宿主人格与安全规则 MUST NOT 被记忆内容覆盖，冲突时宿主规则优先"；`08 §7.9` 规定"宿主是最终 Prompt 唯一拼装者"；`V2 §13.7` 禁止 P0–P3 动 qq-bridge 的 Persona。
- **DESIGN COMPLETENESS（缺口）**：所谓"授权视图"**全语料 0 处定义**（无 schema、无 API、无字段表）；Persona 在 31 份合同中 0 个 schema、`15` 中 0 个参数、状态机中 0 个状态。
- **实践侧（不在 canonical）**：`07` 文档记载现网人格卡在 **user prompt**、占注入量 **49.8%**、每轮全文注入；R3 人格管线 SPEC（`PROPOSED_NOT_APPLIED`，authority=none）提出 L0/L1/L2 分层与"每 N 轮锚定"，与 `07` 记录的"不降频"裁决**直接冲突**且从未关闭（D-C005）。

### 7.2 Persona Prompt 完整性矩阵（必交表 6.5）

| Prompt 层 | 数据来源 | 结构 | 存储位置 | 读取模块 | 每轮注入 | 动态变化 | Token 预算 | 版本管理 | 更新方式 | 当前状态 |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 基础系统指令/安全 | DSH preset | 散文 + 5 条规则 | preset | 宿主 | 是 | 否 | 全局（不占记忆预算） | preset 版本 | 部署 | **EXISTING**（实践） |
| 2 工具协议 | DSH preset | 16 条 | preset | 宿主 | 是 | 否 | 全局 | 同上 | 部署 | **EXISTING** |
| 3 稳定 Persona | 人格卡 | 手写 Markdown（10 节） | `roles/大肥鱼.md` | qq-bridge | **是（49.8%）** | 否 | **无预算** | **无** | 整卡重写 | **CONCEPT_ONLY** |
| 4 用户/主人画像 | Memory `getProfile` | 11 分区 + basis | 无表 | M02（P5） | 否（默认不全读） | 是 | `profile_tokens` | `profile_version` | M04（P4） | **EXISTING 契约 / 未实施** |
| 5 相关长期记忆 | `memory.search`→M02→ALG-06 | 9 字段 + 分区 | SQLite | M02 | 是（有意图时） | 是 | **1200 token** | version | M01 | **EXISTING 契约 / 未实施** |
| 6 当前会话上下文 | 宿主 | 对话历史 | 宿主 | 宿主 | 是 | 是 | 宿主全局 | 无 | — | **EXISTING**（实践） |
| 7 相关事件 | `getEvent` | Event 事实 | 无表 | M02（P5） | 按需 | 是 | `event_tokens` | version | M01 | **契约有 / 未实施** |
| 8 EmotionContext | `getEmotionContext` | 附件 | 无表 | M02（P5） | **默认否** | 是 | `emotion_tokens` | 无 | M04 | **BLOCKED** |
| 9 任务指令/输出约束 | 宿主 + preset | 散文 | 宿主 | 宿主 | 是 | 是 | `task_tokens` | 无 | 每轮 | **PARTIAL**（R3 提议下沉） |

**唯一组装者**：`08 §7.9` 已裁定为宿主；工程上应落在 qq-bridge 内的单点 `PromptAssembler`。Memory Core **永远只交付"授权记忆上下文块"**，不参与最终拼装；DSH preset 只提供静态层。

### 7.3 十问（编号作答，详见附件 04）

1. **有真正的初始化方案吗？** 没有。canonical 只有定位与边界；实践只有手写卡。
2. **能一次人格 Prompt 转结构化配置吗？** 能。对 v1.2 十节卡逐节映射：7 节可机械抽取，2 处需 owner 确认，1 处（元层级指令）应原文整段保留（`04 §3` 流程 A）。
3. **配置存哪里？** 现有文档未给。**NEW PROPOSAL**：`profiles/persona/<persona_id>/{config.json, card.md, history/<n>.json}`，**不进 Memory Core 的 contracts**。
4. **每轮谁读？** 宿主 `PersonaLoader` 单点（qq-bridge 内），不是 M02、不是 DSH。
5. **如何与记忆合并？** 宿主按九层模板拼装；记忆块标 `[数据]`，永不置于人格层之前。
6. **如何避免每轮重生成人格？** 硬规则：逐轮链路对 `persona_config` **只读**；任何逐轮写入 = 红线。
7. **如何防止记忆覆盖人格？** 现有只有原则（`08 L338`、注入块"标为数据"），无顺序锁、无优先级字段、无回归测试。**NEW PROPOSAL**：注入顺序即指令强度序 + T-D-01 测试（prompt-dump 断言）。
8. **版本管理？** 无。**NEW PROPOSAL**：`persona_version` 单调递增 + `status(draft/active/retired)` + 历史不删 + `change_reason` 必填（可复用 `10 §7.7` 的 prompt 模板机制）。
9. **能单独切换 Persona 吗？** 现状不能（单文件单角色）。**NEW PROPOSAL**：按 `persona_id` 目录化，切换 = 切 active 指针。
10. **不改 Memory Core 能调 Persona 吗？** **能，且这是唯一正确路径**——依据 `00 L271` 的反面清单与 `00 §7.7` 的 FUTURE 定位。

### 7.4 Token 预算与"哪层先被挤掉"

- **记忆侧（已裁决）**：`budget_tokens=1200`（token）、`budget_chars=2400`（字符），**先到者裁剪**；五类计量（system/chat_history/memory_retrieval/injected_content/module_calls）+ 四类单位（token/字符/汉字/字节）必须分标（`14 §7.12`）。
- **现实算术**：把一张 1558 字符的人格卡放错到记忆预算里 ≈ 1250 token，**一张卡就能吃光 1200 预算**，记忆检索归零。这正是"人格必须留在宿主层并瘦身/锚定"的量化理由（D-路 E7/R-02）。
- **确定性降级序列（NEW PROPOSAL，可判定）**：`must` 永不主动弃 → `relevant` 按边际价值降序丢 → 先丢 `background` → 再丢 `uncertain` → 全丢则 `retrieval_status=failed`（降级为"无记忆对话"，禁止"假记忆对话"）。Persona 层自身：关情绪 → 关事件 → 画像降为关键偏好一行 → 记忆按上序裁 → Persona 从 full 降 anchor（31 字符锚）→ **永不降 System 安全/协议层，不静默删 `boundaries[]`**。

### 7.5 判定

**CONCEPT_ONLY（初始化） + PARTIAL（逐轮组装）。** 综合能力评级 **CONCEPT_ONLY**：产品要求的"稳定人格 + 长期互动"在 canonical 里是**边界齐全、管线为零**。

> **父 Agent 判断说明**：D 路把综合评为 `PARTIAL`（因其记忆侧组装 MOSTLY_COMPLETE）。本报告下调为 `CONCEPT_ONLY`，理由是：本项能力的判定对象是"Persona 能否被输入、保存、每轮装配"，而**初始化端为零**（无抽取、无配置、无版本），逐轮端又缺 Persona 层契约；记忆侧组装再完整也不能替代人格侧的零。D 路的分项评级（初始化 CONCEPT_ONLY / 逐轮 PARTIAL）与本判定并不矛盾，只是合并口径不同。

可直接继承的是边界裁定（谁不做什么）与相邻成熟部件（画像 11 分区、prompt 版本机制、Token 五分类、1200 预算）；必须新建的是一个宿主侧 Persona 模块（初始化 + 版本 + 装载 + 渲染策略）。

---

## 8. 信息读取与聊天 LLM 利用（目标五）

### 8.1 读取链的真实形态

```
查询 ──► 意图识别(9类) ──► 锚点路由(8锚点) ──► 候选(FTS50 + Vector50)
   ✅          ⚠️判定算法缺          ✅计划/⚠️权重反了        ✅
   ──► RRF(k=60) ──► 重排(6特征/分组?) ──► M05判权 ──► 预算(1200) ──► 组装(9字段)
        ✅              ❌两套互斥           ⚠️输入字段缺      ✅        ❌三套分区
   ──► 宿主注入
        ❌响应契约三套零重合
```

### 8.2 现有设计确实回答了的

| 问题 | 现有答案 | 出处 |
|---|---|---|
| 信息如何被找到 | 意图→锚点→FTS/向量→RRF→重排→权限→预算→组装 | `DD-V2 §8`、`11 §7.17` |
| 如何判断需要哪些记忆 | 9 类意图 → 锚点默认映射（可被调用方覆盖） | `11 §7.19`、`V2 §2.11` |
| 如何选人物画像/事件 | 人物理解 6 步顺序；事件查询另一条路由；互不干扰 | `11 §7.14/§7.15/§7.18` |
| 如何取回原句 | `getEvidence(memory_id\|evidence_id, mode)`，`related_evidence` 反向 | `14 §7.2.5` |
| 如何取情绪 | `getEmotionContext(event_id\|memory_id)`，默认不注入 | `14 §7.2.7` |
| 如何避免无关历史 | 意图路由 + 1200 token 预算 + GreedySelect 四分区 | `02 ALG-05/06` |
| 如何按预算裁剪 | 五类计量 + 双预算（先到者裁） | `14 §7.12` |
| 如何整理成 LLM 能懂 | 统一 9 字段 + 分区 + 来源标注 | `14 §7.3`、`11 §7.17 阶段5` |

### 8.3 真实断点（只列会造成核心信息错误/无法读取的）

| # | 断点 | 后果 | 证据 |
|---|---|---|---|
| 1 | **两套排序算法互斥**：`02 ALG-04` 全局六特征 vs `11 §7.17` 分组装配禁全局权重 | 实现者按任一套都被另一套判违规 | F-C001、G-C001 |
| 2 | **被授权的 P2 那套绑 v1 字段**：ALG-04 的 `N/T/P/importance` 依赖 `memory-object` 独有字段，`memory-unit` 一个都没有 | 用 V2 字段实现则 4 个特征不可计算，排序退化为单特征 | G-C001 |
| 3 | **返回契约三套、零重合**：`08 §7.7` 八字段 / `03 §7.3.4` 八字段 / `context-response.schema.json` 11 属性；实测满足 08 的响应被 schema 全拒，schema 合法响应又不满足 08 | **任何合法响应都不能被宿主注入** | G-C002（E3 实测） |
| 4 | **`tier` 无法作为过滤条件**：`read-request` 17 属性无 `tier_filter`，而 `11 §7.16` 第 4 条"直读长期记忆"要求按 tier 取 | 能力 4 无法表达 | A-C006、G-C012 |
| 5 | **无相关性下限**：ALG-04 只有 `α·R` 是相关性，`E/N/T/S` 与语义无关，`R=0` 时仍可达 **0.45**；`GreedySelect` 只以 `U≤0` 停 | 零相关候选可进 Prompt（"优先级≠强制注入"失效） | G-C008 |
| 6 | **无"已注入集合"**：`read-request`/`retrieval-request` 均无 `exclude_ids`，而 `14 §7.4` 硬规则 2 要求去重 | 同一记忆每轮重复注入 | G-C008 |
| 7 | **无分组配额**：`anchor-set` 无 `quotas`；`AnchorSet` 设计里的 `primary/ordered/quotas` 未实现 | 画像组无预算下限，事件可挤占 | B-C004、G-C007 |
| 8 | **三套上下文分区词表**：`must/relevant/background/uncertain`(4) vs 画像/事件/证据/情绪(4) vs `DirectlyRelevant/StableFacts/…`(5) | 组装无统一口径 | G-路 §7 #10 |

### 8.4 "人物锚点优先"的真实含义——文本正确、契约反向

- **文本**：`11 §7.14` 规定人物理解顺序 `SELF_EXPLICIT → PERSON_PROFILE → PREFERENCE → SELF_DERIVED → RELATIONSHIP → EVENT`；原则 1"自述优先于推断"；原则 2"画像优先于大量事件"；`11 §7.18` 明确事件查询不受此限、两者互不干扰。
- **契约**：`anchor-set.schema.json` 的全局 `weights` 是 `PERSON_PROFILE 0.35 > EVENT 0.30 > SELF_EXPLICIT 0.20 > PREFERENCE 0.15 > CURRENT_STATE 0.15 > SELF_DERIVED 0.10 > RELATIONSHIP 0.10 > EMOTION_CONTEXT 0`。**在所有查询里事件都压过自述**——正是 `11 §7.14` 反模式与 `§7.17` 禁令 1 所禁止的形态。
- **结论**：口号是对的，机器契约把它编码反了。这是"优先级不等于强制注入"这一类设计**在契约层失守**的典型样本。

### 8.5 判定

**PARTIAL。** 读取链的**设计意图完整、接口面丰富、错误码与降级齐全**；但排序形态未裁决、返回体三套互斥、关键过滤与闸门缺失，导致"聊天 LLM 能拿到正确、相关、结构化信息"这一条**在当前契约下不可执行**。这不是"没做安全审计"，是**交付端根本没有一个能同时通过 schema 与宿主校验的对象**。

---

## 9. 模块独立性和可调整性（目标六）

### 9.1 模块权责矩阵（必交表 6.3，压缩版）

| 模块 | 数据所有权 | 业务决策权 | 写入权 | 读取权 | 修改权 | 调用方 | 被调用方 | 禁止行为 | 可独立调整部分 |
|---|---|---|---|---|---|---|---|---|---|
| **M01** | 唯一权威存储（Memory/Evidence/Event/Emotion/Profile/Relation/Outbox/Audit） | 无（不做业务判断） | **唯一** | 全部 | 全部（经接口） | M02/M03/M04/M05/Pipeline | FTS/LanceDB/Embedding | 决定 policy、LLM 推理、检索排序 | 存储引擎、DDL、Outbox 实现 |
| **M02** | 检索计划、候选排序（不拥有正文） | 检索策略/排序/预算/组装 | **无**（声明可写 `last_retrieved_at`，但矩阵只给只读） | 经 M01 读函数 | 无 | 宿主/Context Adapter | M01/M05/FTS/LanceDB | 改事实、自动确认、绕过权限、直连 SQLite | 分词器、向量库、RRF 参数、预算 |
| **M03** | 候选（`Candidate[]`） | 提取/分类/锚点/证据绑定 | 候选池（**表被禁建**） | 授权来源 | 候选 | Pipeline | LLM Gateway/M05 | 持久化权威记忆、绕过 M01 | 抽取模型、原子化、分类规则 |
| **M04** | `confirmed` 决策、画像视图、Relation/Observation | 整合/去重/冲突/画像/版本 | 经 M01 | 候选 + 既有记忆 | 画像/关系/版本 | M03/M01 | M01/M05/L2 LLM | 无依据确认、删证据、越权改 Scope | 整合算法、画像分区、冲突规则 |
| **M05** | 权限决策（`PermissionDecision`） | canRead/Write/Update/Delete/Share/Export | 无（审计写由 M01 落） | 权限元数据 | 策略配置 | M02/M03/M04 | M01 | 发明策略、默认允许、被绕过 | 策略表、映射矩阵、fail-closed 规则 |
| Pipeline | 编排状态（**无模块行**） | 阶段编排 | **无** | 候选 | 无 | 宿主 | M03/M04/M05/M01 | 直写 SQLite、取代 M03、绕过 M05 | 阶段顺序、幂等重试 |
| Persona（宿主） | 人格配置 | 人格呈现 | 人格配置（Memory Core 外） | 经授权视图 | 人格配置 | 宿主 | —（**无接口**） | 直连库、绕过 M05、改记忆 | 全部（不进 Memory Core） |
| Psychology | 运行时状态 | 状态衰减 | 运行时（不落库） | 情绪 | 运行时 | — | — | 认定事实、改权限、影响排序 | 全部（FUTURE） |
| Dream | 维护任务 | 调度 | 无独立 DB 权 | 全库（只读） | 报告 | — | M04/M05 | 绕过 M04/M05、自动扩容 | 全部（FUTURE） |

### 9.2 调用依赖与数据所有权（要点）

- **依赖矩阵的核心缺陷**：M02→SQLite 一边写"禁止直连"，一边在 `02 ALG-02 §5.6`、`03 §7.2` 要求"SQLite 回源"（G-C006，**父 Agent 复核：严重性由 P0 下调为 P1**——`03 §7.4` 提供 `getMemory/queryMemories` 三个读函数，经 M01 回源存在合法实现，问题在措辞而非能力）。
- **数据所有权空洞**（表 6.3 第 3 列）：`candidate_pool` 声明 M03、合法 ∅；`validation_state=confirmed` 声明 M04、实际 ∅；`last_retrieved_at` 声明 M02、实际 ∅；`PersonProfile/Event/EmotionContext` 声明 M04/M01、实际 ∅（无表）。**12 处中有 8 处"声明写入者 ≠ 合法写入者"**（G §6）。
- **共享核心模型**：5 个对象 + 8 个枚举 + `Relation`/`Evidence` 是跨模块共享面；`memory-unit` 28 个全必填字段 + 6 个跨对象数组是**共享合同的最大破坏半径**（改动穿透 8–28 个文件，E-C009）。

### 9.3 14 个变更场景（必交表 6.6 摘要）

> 级别：C=CONFIG_ONLY · L=MODULE_LOCAL · T=CONTRACT · M=SCHEMA_MIGRATION · X=MULTI_MODULE · A=ARCHITECTURE

| # | 变更 | 级别 | 主要影响 | 需迁移 | 影响其他模块核心行为 |
|---|---|---|---|---|---|
| 1 | 换 Embedding Provider | **C + T** | M01(Embedding)/M02/配置/空间注册 | 是（维度/空间） | 是（索引全量重嵌） |
| 2 | 换向量库 | **L** | M01 派生存储 | 是（重建） | 否（若接口不变） |
| 3 | 改 FTS 分词器 | **C + L** | M02/FTS/参数 | 是（重建索引） | 是（召回质量） |
| 4 | 增加长期记忆区块 | **M + X** | schema/DDL/分类/检索/画像/预算 | 是 | 是 |
| 5 | 改 Anchor 分类 | **M + X** | `anchor` 8 值/路由/权重/测试 | 是 | 是 |
| 6 | 增加 relation 类型 | **T** | `relation` 枚举/M04 整合/检索 | 否 | 否（但结构缺口未解） |
| 7 | 改 RRF 参数 | **C** | 配置 | 否 | 否 |
| 8 | 改时间衰减 | **C + L** | ALG-04 参数 | 否 | 否（排序变化） |
| 9 | 改 Tier 晋升规则 | **A（当前无规则可改）** | SM/ALG/DDL/config | 是 | 是 |
| 10 | 改情绪算法/权重 | **C + L**（开关已在 config） | 13/ΔE/参数 | 否 | 是（若启用） |
| 11 | 改 Persona Prompt | **L（宿主侧）** | 宿主配置 | 否 | **否（这正是正确边界）** |
| 12 | 新增 Dream | **A（被 D-AP-03 禁止第六模块）** | 调度/权限/所有权 | 是 | 是 |
| 13 | 改检索预算 | **C** | `api.budget.tokens` | 否 | 否 |
| 14 | 改 PersonProfile 结构 | **M + X** | schema/DDL/画像/视图/预算 | 是 | 是 |

**统计**：CONFIG_ONLY **4**（1 部分/7/8 部分/13）、MODULE_LOCAL **2**、CONTRACT **2**、SCHEMA_MIGRATION **3**、MULTI_MODULE **4**、ARCHITECTURE **2**（9/12）。**8/14 场景外溢多模块。**

### 9.4 隐藏耦合（父 Agent 认可 E 路的 H-1…H-8）

| ID | 耦合 | 为什么是"隐藏"的 |
|---|---|---|
| H-1 | 两套 memory 主 schema 并存（`memory-object` v1 vs `memory-unit` v2） | 文档说"v1 是存储视图"，schema 没标 deprecated，两者都 required-valid |
| H-2 | 配置双位点/镜像/跨 config locator | `anchor.routing_enabled` 唯一权威位点在 config/anchor，retrieval 有只读镜像；不等即 `CONFIGURATION_ERROR` |
| H-3 | 调度 owner 三重（M04 任务队列 / M01 Worker / Dream） | 三份文档各自声明，违反 `00 §7.3` 规则 5"横切能力唯一 owner" |
| H-4 | LLM 通道疑似两条（M03 抽象 Provider vs M04 唯一 Gateway） | `00 §7.3` 规则 3 说唯一入口，`03-m03 §12` 说 M03 自己调 Provider |
| H-5 | M02→M04 标注 C(FUTURE) 但关系扩展必经 | `getRelated` 的邻居查询归 M04，而 M04 未授权 |
| H-6 | 模块稿 R3 基线 vs 规范集 R4.1/V2 | 五份 `03-m0x` 自标 R3，与 00–15 版本错位（E-C007） |
| H-7 | "禁止第六模块" vs 可插拔扩展目标 | `D-AP-03` 禁新模块，`00 §7.7` 要求扩展模块可插拔可拔除 |
| H-8 | 全必填字段 + 枚举在响应合同复制 | 改一个枚举穿透 8–28 文件；无受控词表注册 |

### 9.5 判定

**PARTIAL（独立性 LOW）。** 文件被拆成 M01–M05 **不等于**模块独立：共享 DDL/共享 schema/共享枚举把五模块绑在一起；唯一真正解耦的是 **Outbox / FTS5 / LanceDB 的单一所有者 + 派生可重建**（这是架构里最扎实的一处）。**"新增一个模块而不用重写核心"当前不成立**（被 `D-AP-03` 禁止 + 无接入契约）。改善方向见第 15 章 N-09。

---

## 10. 核心算法完整性（目标七）

### 10.1 全景统计

按 8 大算法族建立 **76 条**算法条目（附件 06）：

| 完整度 | 条数 | 占比 |
|---|---:|---:|
| COMPLETE | 13 | 17% |
| MOSTLY_COMPLETE | 43 | 57% |
| PARTIAL | 12 | 16% |
| CONCEPT_ONLY | 4 | 5% |
| CONFLICTED | 1 | 1% |
| MISSING | 3 | 4% |

**最重要的既有资产**：`02` 的 `ALG-01…ALG-12` 统一 **17 要素**（目的/输入/输出/前置/步骤/公式/参数/边界/异常/幂等/复杂度/配置/默认值/测试/验收/禁止/依赖），这是"工程 Agent 可以不猜测实现"的形态本身。**检索与写入主干（ALG-01/02/03/05/06）、Evidence 回源、Outbox 索引更新**已达 COMPLETE/MOSTLY_COMPLETE。

### 10.2 核心算法完整度矩阵（必交表 6.7，21 条重点）

| 算法 | 所属 | 输入 | 输出 | 公式 | 步骤 | 参数 | 边界 | 异常 | 测试 | 完整度 | 可直接实现? |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 摄取与原子化 | Pipeline/M03 | 消息/文件 | `Candidate[]` | 无 | 16 阶段逐步 | `ingest.*`（无 config 载体） | 有 | 有 | 有 | MOSTLY | 否（模型选型 TBD） |
| 分类与 Anchor | M03 | 候选单元 | `semantic_type/tags/anchor` | 无 | 有 | 词表 | 互斥规则 | 有 | 有 | MOSTLY | 否（依赖 LLM 选型） |
| Evidence 绑定 | M03/M01 | 单元+原文 | `evidence_refs`+span | hash | 有 | 有 | 有 | 有 | 有 | MOSTLY | **是**（偏移校验待补） |
| 重要性与置信度 | M03/M04 | 特征 | `importance/confidence` | **无公式** | 仅一句 | 三阈值 | 无 | 无 | 部分 | **PARTIAL** | **否** |
| 记忆去重 | M03 | 候选+既有 | 重复判定 | 3-gram Jaccard | 有 | 阈值 | 有 | 有 | 有 | MOSTLY | 是 |
| 冲突检测 | M03/M04 | 新旧记忆 | `conflict_flags` | 无 | 5 类/8 类**两套** | 有 | 有 | 有 | 有 | CONFLICTED | 否（需映射表） |
| 记忆版本 | M01 | 修订 | 新版本行 | 无 | 有 | 有 | 有 | 有 | 有 | COMPLETE | 是 |
| Tier 晋升/降级 | M04 | 记忆+依据 | tier 变更 | **无** | **无** | **无** | **无** | **无** | **无** | **MISSING** | **否** |
| 过期与复核 | M01/M04 | 记忆+策略 | 状态变更 | `T=exp(−λΔt)`（仅排序） | 有（无执行体） | `lifecycle.*`（未注册） | 有 | 有 | 有 | PARTIAL | 否 |
| FTS 检索 | M02 | 查询 | 候选 | BM25 | 有 | Top-K 50 | 有 | 有 | 有 | MOSTLY | 否（分词器 TBD） |
| Vector 检索 | M02 | 查询/向量 | 候选 | 余弦/FLAT | 有 | Top-K 50 | 有 | 有 | 有 | MOSTLY | 是（scan-only） |
| RRF | M02 | 通道排名 | `rrf_score` | `Σw/(k+rank)`，k=60 | 有 | 2 通道（`fts`+`vector`） | 有 | 有 | 有 | COMPLETE | 是 |
| Reranking | M02 | 候选 | `final_score` | 六特征 **或** 分组 | 有 | 权重（两套） | 有 | 有 | 有 | CONFLICTED | **否** |
| Temporal Decay | M02 | 时间 | 衰减因子 | `exp(−λΔt)` | 有 | `decay_lambda.{type}` | 有 | 有 | 有 | MOSTLY | 是（λ 待标定） |
| 跨区块检索 | M02 | 记忆 | 关联记忆 | 无 | **无** | 无 | 无 | 无 | 无 | **MISSING** | **否** |
| Evidence 回源 | M01/M02 | id | 原句 | 无 | 九步 | 有 | 有 | 有 | 有 | COMPLETE | **是** |
| Token Budget | M02 | 排序候选 | selected/skipped | 贪心 | 八步 | 1200/2400/2000 | 有 | 有 | 有 | MOSTLY | 是（分词器待定） |
| Context Assembly | M02 | 已判权候选 | `ContextPackage` | 无 | 八步 | 四分区 | 有 | 有 | 有 | PARTIAL | 否（分区三套） |
| 情绪关联与衰减 | M03/M02 | 事件/记忆 | `EmotionContext` | `ΔE=I·C·R·D` | 有 | 3 开关+上限 | 有 | 有 | 有 | PARTIAL | 否（无表/默认关） |
| Persona Prompt 组装 | 宿主 | 人格+记忆 | Prompt | 无 | **无** | 无 | 无 | 无 | 无 | **MISSING** | **否** |
| Outbox 索引更新 | M01/Worker | 事件 | 派生索引 | 无 | 有 | 有 | 有 | 有 | 有 | MOSTLY | 是（投影回退 TBD） |

### 10.3 十个最关键缺口（父 Agent 排序）

1. **importance 无赋值算法**（权威字段永远空值 → 过滤/并列失效）。
2. **confidence 无合成公式**（三阈值分流的输入不可信）。
3. **排序形态冲突**（全局 vs 分组，见第 8 章）。
4. **FTS 中文分词器 TBD**（中文召回验收无法执行；前轮实测 `unicode61` 4/19 查询零结果）。
5. **索引文本投影无摘要回退策略 TBD**（Worker 无法实现）。
6. **Tier 晋升/降级缺失**（与 A-C001 同一问题）。
7. **Intent Resolver 判定算法缺失**（9 类意图怎么判出来只有枚举，没有算法）。
8. **跨区块联合召回缺失**（与 B-C002 同一问题）。
9. **冲突枚举两套（5 vs 8）无映射**。
10. **关键参数未入 15 号注册表**（权重类 11 项、`consolidation.*` 4 项、`lifecycle.*`、`context.*` 部分）。

### 10.4 判定

**MOSTLY_COMPLETE。** 但必须加一句限定：**"完整"集中在检索/写入主干；"缺失"集中在质量、层级、组织、人格四类**。按"能否交给工程 Agent 不猜测地实现"这一条衡量：21 条重点算法中 **11 条能、10 条不能**（其中 3 条是 MISSING/CONFLICTED）。

---
## 11. M01～M05 权责冲突

### 11.1 五处边界缺陷（前轮已证，本轮复核）

| ID | 缺陷 | 本轮结论 |
|---|---|---|
| FC-018 | 边界矩阵禁 M02→SQLite，而算法要求"SQLite 回源" | **仍有效，但严重性下调为 P1**：`03 §7.4` 有 `getMemory/queryMemories` 读函数，经 M01 回源存在合法实现；问题是措辞自相矛盾，不是能力缺失 |
| FC-019 | 候选池无合法写者（M03 声明 / 全库持久化仅 M01） | **仍有效 P0**（写入闭环断点之一） |
| FC-020 | M02 须写权威态 `last_retrieved_at`，但被定义为读者 | **仍有效 P1**，且与 `ALG-01 §17`"禁止 M02 触发写入"三方互斥 → `last_retrieved_at` 实际无合法写入者 |
| FC-021 | 16 阶段 Pipeline 无模块行、无责任矩阵行 | **仍有效 P1**：编排无 owner，失败重入无载体 |
| FC-022 | "统一任务体系"与 `GET /tasks/{id}` 无 owner | **仍有效 P1**（运维级） |

### 11.2 声明写入者 vs 实际合法写入者（12 行，合并 G-路）

| # | 数据 | 声明写入者 | 实际合法写入者 | 后果 |
|---:|---|---|---|---|
| 1 | `MemoryUnit` 全行 | M01 | **∅**（默认态违反自身 schema） | 第一条记忆写不进 |
| 2 | `memory_tier` | M04（M03 只给候选） | **∅**（M04 FUTURE + 无列 + `candidate` 无字段） | 层级永不变化 |
| 3 | `candidate_pool` | M03 | **∅**（表禁建） | propose 无处可去 |
| 4 | `validation_state=confirmed` | M04 通道 | **∅**（P4 未授权） | 记忆永远 `unverified` |
| 5 | `last_retrieved_at` | M02 | **∅** | 使用反馈不可记录 |
| 6 | `PersonProfile` 行 | M04 | **∅**（P4 + 无表） | 人格/画像无载体 |
| 7 | `Event` 行 | M01（经 M04） | **∅**（无表） | 事件退化为 unit 标签 |
| 8 | `EmotionContext` 行 | M03→M04 | **∅**（无表 + 三开关关） | 情绪不可存取 |
| 9 | 删除掩码 | M01 | M01 写，但写在**已废止**的 `lifecycle_state` | 删除后可能仍被检索 |
| 10 | 审计 `audit_metadata` | M05（`ALG-09` 步骤 10） | M01（矩阵给 M05 只读） | 权限决策无法自证 |
| 11 | Pipeline 编排状态 | Pipeline | **∅**（无模块行） | 重入无载体 |
| 12 | `sensitivity`（权限输入） | 调用方传入 | **∅**（V2 对象无此字段） | M05 判权退化 |

**父 Agent 判断**：这张表比"49 个 FC"更能说明问题性质——**不是文档互相矛盾，而是大量关键数据根本没有合法写入者**。这是 P0 的共性根因。

### 11.3 责任分配建议（最小改动，不新增第六模块）

| 数据 | 建议唯一写入者 | 机制 |
|---|---|---|
| MemoryUnit/Event/Emotion/Profile/Relation | **M01** | M04 只产"决策对象"，M01 事务写入 |
| 候选池 | **M01**（表） + M03（产候选） | 允许 P1 建空表，propose 落 `PENDING` |
| `confirmed` | **M04**（经 M01） | 保留 D-AP-05；V0.9 用人工 CLI 通道替代（显式 `human_owner_review`，不写 confirmed） |
| `memory_tier` | **M04 裁决 → M01 写** | 见 A 路 `SM-9` + `ALG-13` + bootstrap |
| `last_retrieved_at` | **M01 异步记录**（M02 只上报 `decision_id`） | 或直接删除该字段（诚实：V1 不做使用反馈） |
| 审计 | **M01 落盘**（M05 返回 `decision_id`） | 与矩阵一致 |
| Pipeline | **无状态编排器**（不持有权威数据） | 登记为 M02/M01 组合职责 |

---

## 12. 七项能力完整度总表

| # | 核心能力 | 判定 | 主要原因 | 关键证据 |
|---|---|---|---|---|
| 1 | 短期与长期记忆 | **PARTIAL** | 长期存储与检索有；短期（L1）无存储无 owner；0 条 tier 转换；原句追溯在存储层断 | A-C001…A-C004 |
| 2 | 长期记忆分区与关联 | **PARTIAL** | 分面分类成立；关系端点只到 `memory_id`；项目/稳定事实/情境三区块缺；无自动联合召回 | B-C001/B-C002/B-C003 |
| 3 | 情绪附件 | **PARTIAL** | 对象 14 字段与 ΔE 完整；无表、三开关 `const:false`、`maxItems:0`；无人物/互动绑定；无 Persona 桥 | C-C001/C-C002/C-C003 |
| 4 | Persona Prompt | **CONCEPT_ONLY** | 无 schema/参数/状态机/接口；无初始化管线；逐轮组装只有"宿主唯一拼装者"一句原则（初始化端为零，故不因记忆侧组装完整而上调） | D-C001/D-C002 |
| 5 | 信息读取与利用 | **PARTIAL** | 9 接口/5 阶段齐；两套排序互斥、返回体三套零重合、无 tier 过滤/相关性下限/已注入集合 | G-C001/G-C002/G-C008 |
| 6 | 模块独立可调整 | **PARTIAL（LOW）** | 5 模块共享 DDL/schema/枚举；8 个场景外溢多模块；新增模块被禁；唯一解耦是 Outbox/FTS/向量 | E-C001…E-C012 |
| 7 | 核心算法 | **MOSTLY_COMPLETE** | 检索/写入主干可实施；质量、层级、组织、人格四类缺；21 条重点中 10 条不能直接实现 | F-C001…F-C008 |

**整体判定**：七项中 **0 项 COMPLETE、1 项 MOSTLY_COMPLETE、5 项 PARTIAL、1 项 CONCEPT_ONLY**。

---

## 13. P0 / P1 严重问题（合并去重后的最终清单）

> 合并原则：同一根因跨路出现时只留一条，标注来源路。**P0 = 7 条，P1 = 11 条。** P2 归档见 §13.4。

### 13.1 P0（核心能力无法成立）

#### P0-1 · 不存在合法的新建记忆（默认态违反自身 schema）
- **现状**：`memory-unit.schema.json` 默认 `status=ACTIVE` + `validation_state=unverified`，而 `allOf[1]` 要求 `ACTIVE ⇒ validation_state ∈ {confirmed, disputed}`；`D-AP-05` 又规定 `confirmed` 只能由 M04 写，M04 未授权。实测：字段齐备实例报 1 错（G-C005 E3 复现）。
- **根因**：把"流程规则"写成了"静态不变量"。同一错误也出现在 `read-response` 要求 `original_text` 而 `evidence.schema.json` 没有该字段。
- **影响**：M01 第一条 insert 即失败；所有写入路径阻断。
- **推荐**：把 `ACTIVE ⇒ confirmed/disputed` 从 `allOf` 移出，改成 `SM-2` 转换守卫（`unverified→confirmed` 仅 M04）；`allOf` 只保留静态可判约束（`DELETED` 终态、`SUPERSEDED` 必有 supersedes 关系）。
- **责任/影响面**：M01 + M04 + schema；**不改架构**；工作量 S（<1 天）；**阻断 V0.9：是**；**V0.9 实验可解：是**（最便宜的一条，应第一个做）。

#### P0-2 · 存储契约不存在：0 DDL，3 个对象无表
- **现状**：全语料 0 份可执行 DDL；`01 §7.16` 13 张表只是列清单，且 `01:3` 自述"需新增 event/emotion_context/person_profile/candidate/profile_version"而 §7.16 一张都没有；`memory_unit` 14 个 V2 字段无列、`evidence` 缺 7 列（含 `original_text`）。
- **影响**：M01 无法开工；事件/情绪/画像三类记忆无落库；跨对象引用无 FK/CHECK/索引。
- **推荐**：一次 DDL v2 修订产出 **17 张表** = 13 表按 V2 字段补齐 + `events` + `emotion_contexts` + `person_profiles` + `profile_versions`；`candidate_pool` 允许建空表（见 P0-6）；补 `INDEX(tenant_id, memory_tier, status)`、`INDEX(event_id)`、`INDEX(subject_id, created_at)`；同步迁移脚本与 `schema_version` 映射。
- **责任/影响面**：M01 + 全部读写；**改 schema + 迁移**；工作量 L；**阻断 V0.9：是**；**实验可解：不能**（DDL 只能写出来）。

#### P0-3 · 层级生命周期不存在（短期永远进不了长期）
- **现状**：`memory_tier` 无列、无索引、无写入者、无迁移行；8 个状态机 57 条转换中 0 条在 tier 轴；无晋升/降级/保留策略；L1 无存储（`session_state` FUTURE 且禁建）。
- **影响**：`memory_tier` 只是标签；"短期信息可被长期保留"不成立；过期/复核失去分层依据。
- **推荐**：① 决定 L1 是否在 Memory Core 内（**建议：L1 保留 3 值但由宿主提供会话存储，Memory Core 只存 L2/L3**）；② 新增 `SM-9 TierTransition`：初态 + 4 条边（L1→L2、L2→L3、L3→L2、L2→L1），禁 L1→L3 直跳；③ 新增 `ALG-13`：晋升判据 = 证据数 ≥N / 会话数 ≥M / 用户明示 / M04 确认；降级 = 矛盾 / 保留到期 / 合并；④ `RetentionPolicy(tier × semantic_type) → ttl_days/review_after_days/on_expiry`；⑤ `tier_source=PROVISIONAL` bootstrap 规则，使字段在 P1/P2 有合法写入者。
- **责任/影响面**：M04 决策 + M01 写入；**改 schema + 状态机 + 新算法**；工作量 M；**阻断 V0.9：是**（若 V0.9 声称"记忆分层"）；**实验可解：部分**（阈值可标定，是否设 L1 是裁决）。

#### P0-4 · 长期记忆关系不是一等公民
- **现状**：`Relation` 两端只有 `source_memory_id/target_memory_id`；跨对象关系是无类型 ID 数组（`event_refs/emotion_refs/relation_refs`）；无 `Project` 实体、无情境记忆对象、无 `subject_alias`。
- **影响**：人物↔事件、项目↔事件、事件↔情绪无法成为类型化关系；"那次答辩涉及谁/属于哪个项目/当时什么情绪"只能多次手工调用；删除时数组悬空（已实测）。
- **推荐**：`RelationEndpoint{type, id}`，`type ∈ {memory, event, profile, evidence, emotion, project, subject}`；保留 8 种关系 + 审计通过的 `PARTICIPATED_IN`/`PART_OF`；新增 `Project` 实体与 `subject_alias`；给 6 个 ID 数组定义传播算法（删除/合并/权限变更时同步）。
- **责任/影响面**：M01/M04 + schema + DDL；**改 schema + 迁移**；工作量 M–L；**阻断 V0.9：是**（若 V0.9 要做"组织经历"）；**实验可解：部分**（1-hop 扩展可实验，模型变更须裁决）。

#### P0-5 · 读取链两套互斥算法，且授权的那套绑 v1 字段
- **现状**：`02 ALG-04` 全局六特征 vs `11 §7.17` 分组装配禁全局权重；ALG-04 的 `N/T/P/importance` 依赖 `memory-object` 独有字段；`structured` 通道是否进 RRF 三处口径不一。
- **影响**：实现者按任一套都被另一套判违规；用 V2 字段则 4 个特征不可计算，排序退化为单特征；"命中过滤条件"可能被抬高排序。
- **推荐**：以 `11 §7.17` 为骨架重写 `02 §7.3–§7.7`：ALG-02 输入加 `intent`+`anchor_plan[]`+`anchor.max_candidates`；ALG-04 降为"组内打分函数"（`Score = w_R·R + w_E·E + w_T·T`，按锚点组建档）；RRF 固定 `{fts, vector}`；`ALG-06` 停止条件加"组配额耗尽"与"R < 相关性下限"。
- **责任/影响面**：M02 + 02/11/06 + `retrieval.schema.json`；**改算法 + config**；工作量 M；**阻断 V0.9：是**（M02 属 P0–P3）；**实验可解：部分**（"哪套权威"须裁决；权重可实验）。

#### P0-6 · 写入闭环在授权范围内不存在
- **现状**：`candidate_pool` 被禁建；`08 §7.6` 硬规则 5 规定 P3 前 `retain` 必须返回未实现；硬规则 4 规定 `confirm` 只能由未授权的 M04 通道触发；M03/M04 全 FUTURE。
- **影响**：没有任何合法路径把一句话变成一条长期记忆；V0.9 只能做 M01 的 CRUD 测试台。
- **推荐（二选一，需裁决）**：**A（推荐）** 允许 P1/P2 建空候选表，`propose` 落 `PENDING`，`confirm` 走人工 CLI/测试通道（显式 `human_owner_review`，不写 `confirmed`），tier 用 bootstrap；**B** V0.9 明确声明"无写入能力"，只验收 M01+M02 并从最小闭环验收标准删除写入步骤。
- **责任/影响面**：M01/M03/M04 + `01/08/09`；**改 schema + 流程**；工作量 M；**阻断 V0.9：是**；**实验可解：部分**（规则版候选生成可实验；"confirm 归谁"是权限裁决）。

#### P0-7 · Persona 无初始化、无逐轮组装规格
- **现状**：canonical 只有 FUTURE 定位与三条禁令；0 schema/0 参数/0 状态机/0 接口；现网手写人格卡在 user 层、占 49.8%、无版本、三处复制拼装。
- **影响**：稳定人格无法程序化创建、修订、装配；长会话人格会丢（compaction）；记忆预算被挤。
- **推荐**：宿主侧新建 **Persona 模块**（不进 Memory Core）：`initPersona/getPersonaConfig/assembleContext/updatePersona` 四接口 + `persona-config` 草案 schema + 版本归档 + 九层 Prompt 模板 + 分层预算（350/150/25 token）+ 单点 Loader/Assembler。
- **责任/影响面**：宿主（qq-bridge）+ 新文档；**不改 Memory Core**；工作量 M；**阻断 V0.9：是**（Persona 能力验收）；**实验可解：是**（用 v1.2 卡做一次真实初始化 + full/anchor A/B）。

### 13.2 P1（能跑但重要能力缺失/严重失真）

| ID | 问题 | 影响 | 推荐 | 阻断 V0.9 |
|---|---|---|---|---|
| P1-1 | 宿主返回契约三套、字段零重合 | 任何合法响应都不能注入 | 以 `context-response.schema.json` 为唯一契约，**加字段不改名**；同步 03/08 | 否（P5），但阻断集成验收 |
| P1-2 | 无相关性下限 + 无分组配额 + 无已注入集合 | 无关历史可进 Prompt；重复注入 | 加 `recall.relevance_floor` 硬闸；`exclude_memory_ids[]`；组配额 | 否 |
| P1-3 | `anchor-set` 全局权重反编码 | 事件默认压过自述 | 删/禁用 `weights`；`anchor_sequence` 升级为必填带配额；`profile_first` 绑下限 | 否 |
| P1-4 | `read-request` 无 `tier_filter/status_filter/time_range`，`operation` 只 3/9 | "直读 L3"不可表达 | 扩字段 + 扩枚举 + 6 个专用请求 schema | **是**（M02 读原语） |
| P1-5 | `retrieval-request.type_filter` 仍是 v1 七值，无 `intent/anchor` | 锚点路由在 M02 边界无输入 | 改 `semantic_type` 6 值 + 加 `intent/anchor_plan[]` | **是** |
| P1-6 | 情绪无表 + 三开关 `const:false` + `maxItems:0` | 情绪不可存取 | 先建表加索引，再改 `maxItems=N`（配置驱动 0 为默认），开关保持默认关 | 否（V1 默认关是已裁决） |
| P1-7 | 画像 11 分区默认全返回，与 `14 §7.4` 冲突 | Token 浪费 + 隐私面 | `sections` 默认取意图最小集；`current_state` 拆出稳定视图 | 否 |
| P1-8 | `last_retrieved_at` 三方互斥无写入者 | 使用反馈不可记 | M01 异步记录，或删字段 | 否 |
| P1-9 | 参数注册表滞后（11 权重 + consolidation 4 + lifecycle 2 + context 2 未登记） | 生产配置即违规 | 按 `15 §7.14` 八步补登记（先 TBD 值） | 否 |
| P1-10 | 模块独立性不足（8 个场景外溢、新增模块被禁） | 每次演进成本高 | 补接口冻结表 + 扩展点接入契约 + 受控词表 + 合并双 memory schema | 否 |
| P1-11 | `MemoryUnit.content` 可承载 `EMOTION_CONTEXT` 且进入 FTS/向量 RRF 共池 | 情绪可合法压过事实（唯一污染路径） | `content` 增情绪禁令；情绪类 `indexable=false`；`items[]` 强制改走 `emotion_context[]`；补"零进入"测试 | 否 |

### 13.3 前置裁决（不是能力缺陷，但不解决则无法确定"什么是权威"）

| 裁决 | 内容 | 现状 |
|---|---|---|
| DD-001 | 哪套 R4.1 治理（A 121 文件/31 合同 vs B 70 文件/0 合同） | 未定（FC-001） |
| DD-002 | V2 "APPROVED" 的依据是否存在 | 未证（FC-002；任务书在语料中不存在） |
| DD-003 | `P0–P3` 到底授权什么（V2 §10 表 = M03+Pipeline；开篇 = M01+M02 不变） | 未定（FC-003） |
| DD-004/005 | `memory-object` 还是 `memory-unit` 是对外契约；`content_kind` 废止还是必填 | 未定（FC-005/006） |

> 上一轮 FORENSIC 的结论"不具备 V0.9 条件"由这四条 + P0-1/P0-2 支撑，**本轮完全继承，不推翻**。本轮新增的是：即使这四条全部解决，七项核心能力里仍有 4 项需要补设计（本报告第 4–7 章）。

### 13.4 P2 归档（简要，不进入主结论）

参数未标定（λ、k、Top-K、阈值）· 中文 FTS 词表精度 · 向量索引（ANN 已在设计上 REJECTED，只需清 config 残留）· 监控/管理界面 · 日志与审计格式 · 80% 重复文件与 238 个 BOM（工程卫生）· 错误码别名陈述错误 · 变更日志缺项。**这些不改变能力判定，也不阻断 V0.9。**

---

## 14. 可直接继承的新旧设计

### 14.1 基线关系（已确认，不重新争讼）

| 集合 | 文件 | 机器合同 | 定位 |
|---|---:|---|---|
| **A（canonical）** | 121 | **20 schemas + 11 config** | 唯一有机器合同的主要基线；APPROVED 依据有缺口 |
| B（UNIFIED） | 70 | 0 | 竞争性设计候选，不可丢弃；无 L1/L2/L3 |
| CMB | 58 | 0 | 相邻项目（压缩桥），**不是第三套基线** |
| R3 | 45+ | 0 | 历史设计来源（M01–M05 模块稿、人格管线） |

### 14.2 新旧设计对照（必须记录的六项）

| # | 旧版设计 | 新版设计 | 变化 | 用户批准证据 | 影响核心能力 | 处理建议 |
|---|---|---|---|---|---|---|
| 1 | `memory-object` + 7 类 `type` + `content_kind` + `lifecycle_state` | `memory-unit` + 6 类 `semantic_type` + 12 标签 + 4 值 `status` | **对象模型替换**：`content_kind` 废止，`opinion` 语义丢失、`hypothesis/inference` 折叠 | **无**（`00:7/00:76/V2:291` 只有断言；迁移映射不完整） | 是（读取/存储/排序） | 需 DD-004/005 裁决；若保留 v2，必须先补 `opinion` 承载 |
| 2 | 层级 = 树状分类（R3 `lifecycle_state` 派生） | `memory_tier` 与 `semantic_type` **正交** | 从"层级即类型"改为"正交" | **无直接证据**（`V2 §2.5` 铁律无批准记录） | 是（目标一） | 继承正交原则；但必须补转换算法（P0-3） |
| 3 | Persona = 记忆的一部分/角色设定 | Persona = **记忆消费者**（FUTURE） | 从"Core 管人格"改为"宿主管人格" | 有（`08 L327/331/338`、`00 L271`） | 是（目标四） | **继承边界**，补宿主侧模块 |
| 4 | 检索 = 关键词 + 向量 + 时间三路，`structured` 参与 RRF | RRF **仅 `fts`+`vector`**，`structured` 只做过滤 | 通道收窄 | 有（round-2 SRD-002） | 是（排序正确性） | 继承；但 `02 §7.4` 仍写三通道，需回改 |
| 5 | 情绪 = 可选字段/写入正文 | 独立对象 + ΔE + 8 门禁 + 默认关 | 从"混写"改为"分离" | 部分（`13`、`V2 §9`、台账 J6/J7/X-3） | 是（目标三） | 继承分离原则与门禁；补表与对象级开关 |
| 6 | Anchor = 检索关键词 | 8 个**获取锚点** + 意图路由 + 人物优先 | 从"词"升级为"获取策略" | 部分（`11` 全文，无单独批准记录） | 是（目标五） | 继承；修正 `anchor-set.weights` 反编码 |

**父 Agent 判断**：新版在**对象语义、情绪分离、锚点路由、权限模型**上明确优于旧版，应当作为统一基线的骨架；旧版在**模块接口的工程细节**（`03-m01…m05`）与**人格实践**（`07` 文档）上仍有不可替代的资产。两者是"新骨架 + 旧血肉"，不是互斥关系。

### 14.3 可直接继承清单（无需改动，直接进 V0.9）

| 资产 | 出处 | 为什么可直接用 |
|---|---|---|
| 五对象字段定义 | `V2 §3.1–3.5` | 字段级完整，已过 schema 校验（除 Evidence 需合并 v2 修订版） |
| 16 阶段 Pipeline 的阶段划分与禁令 | `V2 §5`、`09` | 阶段责任、LLM 只提议、Pipeline 不写库 |
| 9 个读写接口签名与错误码 | `14 §7.1–§7.3` | 逐接口输入/输出/权限/幂等/超时齐备 |
| 五阶段检索骨架 + 9 类意图 + 8 锚点映射 | `V2 §8`、`11 §7.19` | 意图→锚点是可执行的路由表 |
| `ALG-01/02/03/05/06` 与 Evidence 回源九步 | `02` | 17 要素齐备，可直接实现 |
| Outbox/Worker/幂等/重建 | `03`、`05`、`ALG-01` | 单一所有者 + 派生可重建，是全项目最扎实的解耦点 |
| ΔE 公式 + 8 门禁 + 默认关闭 | `V2 §9`、`13` | 参数化、可审计、不污染事实 |
| 权限十原则 + 十链路覆盖 + fail-closed | `04`、`07` | 设计完整，模拟层已验证注入 |
| 画像 11 分区 + `basis` 三值 + 版本 | `12` | 自述/推断分离的正确形态 |
| R3 模块接口细节 | `03-m01…m05` | M01 七函数、M05 六动作冻结度较高 |
| 人格卡 v1.2（10 节）与注入实证 | `07` 文档 §四/§2.2 | 真实可运行的人格文本，可直接作为初始化输入 |

### 14.4 必须替换或重写的

`memory-object`（降为存储视图并标 deprecated）· `anchor-set.weights`（删或禁用）· `02 ALG-04` 全局公式（降为组内）· `candidate.schema.json`（补 `lifecycle_candidate` 与 6 态）· `evidence.schema.json`（补 `original_text` 与 5 值 `source_type`）· `context-response.schema.json`（加字段合并）· `read-response.emotion_context.maxItems`（改配置驱动）· `01 §7.16`（升级为可执行 DDL）。

---

## 15. 需要补充的最小设计（按"最小可行"排序）

| # | 补什么 | 落在哪 | 是否改 schema | 是否迁移 | 工作量 | 阻断 V0.9 |
|---|---|---|---|---|---|---|
| N-01 | 写入守卫：`ACTIVE` 不再要求 `confirmed`，改为 SM-2 转换守卫 | `memory-unit.schema.json` + `SM-2` | 是 | 否 | S | **是** |
| N-02 | 17 张表 DDL + 迁移脚本 + `schema_version` 映射 | 新 `contracts/ddl/` + `01 §7.16` | 是 | 是 | L | **是** |
| N-03 | `SM-9 TierTransition` + `ALG-13`（晋升/降级/过期/复核） | `state-machines.md` + `02` + `11 §7.12` | 是（列/索引） | 是 | M | **是** |
| N-04 | `RelationEndpoint` 类型化 + `Project` + `subject_alias` | `relation.schema.json` + 新 schema + DDL | 是 | 是 | M | 是（目标二） |
| N-05 | 排序形态裁决：`11 §7.17` 为外壳，`ALG-04` 降为组内 | `02 §7.3–7.7` + `11 §7.17` + `retrieval.schema.json` | 是 | 否 | M | **是** |
| N-06 | 候选池空表 + 人工 confirm 通道（V0.9 降级形态） | `01 §7.16` + `08 §7.6` + `09` | 是 | 是 | M | **是** |
| N-07 | 统一 `context-response` 契约（加字段）+ 删两处散文 | `context-response.schema.json` + `03/08` | 是 | 否 | S | 否（阻断集成验收） |
| N-08 | 相关性下限 + 分组配额 + `exclude_memory_ids` | `02 ALG-04/06` + `read-request` + `15` | 是 | 否 | S | 否 |
| N-09 | 宿主侧 Persona 模块（初始化/版本/装载/组装/预算） | 新《Persona 初始化与版本规范》《逐轮 Prompt 组装规范》 | 否（宿主侧） | 否 | M | **是**（Persona） |
| N-10 | 情绪表 + 对象级开关 + `maxItems=N` + Persona 桥 | `01 §7.16` + `emotion-context.schema.json` + `read-response` | 是 | 是 | M | 否（默认关合法） |
| N-11 | `importance`/`confidence` 计算规范 | `02` 新小节（或 `12`） | 否 | 否 | S | 否 |
| N-12 | 参数补登记（11+4+2+2 项）+ FTS 分词器/投影回退选型 | `15 §7.17` + `retrieval/storage.schema.json` | 是 | 否 | S–M | 否（但阻断验收） |
| N-13 | 接口冻结表 + 扩展点接入契约（新模块可插拔） | 新《接口冻结与扩展点契约》 | 否 | 否 | M | 否 |

**"最小可运行"= N-01 + N-02 + N-06 + N-05**（能写入、能存、能确认、能排对序）。**"最小可用"再加 N-03 + N-04 + N-09**（能分层、能组织、有人格）。

---

## 16. 推荐的统一架构与五张逻辑图

### 16.1 目标架构（一句话）

**扁平权威池（`MemoryUnit`）+ 正交分面分类 + 独立层级与状态轴 + 意图驱动的分组检索 + 有界 1-hop 类型化关系 + 宿主侧 Persona 装配 + 唯一上下文交付契约。**

### 16.2 图一 · 完整记忆生命周期（标 M01–M05 责任）

```
用户输入（QQ/DSH）
   │
   ▼  ①接收        ┌─ 宿主：归集消息、可信身份、会话态（不落库）
   ├──────────────►│
   ▼  ②理解/原子化  │  M03（P3，未授权）: 16 阶段 Pipeline 阶段 1–13
   │               └─ 产出 Candidate（PENDING / NEEDS_REVIEW）
   ▼  ③分类/锚点/证据        M03: semantic_type/tags/anchor/evidence_refs
   │
   ▼  ④候选校验              CandidateValidator + M05 canWrite
   │
   ▼  ⑤确认（M04，P4）       confirmed 只经 M04 通道；V0.9 用人工 CLI 替代
   │
   ▼  ⑥持久化（M01 唯一）     SQLite 事务五步：memory + version + ledger + outbox
   │                          ├─► Outbox ─► FTS5（派生）
   │                          └─► Outbox ─► LanceDB（派生）
   ▼  ⑦层级管理（M04 决/M01 写）  SM-9：L1→L2→L3 / 降级 / 过期 / 复核
   │
   ▼  ⑧检索（M02）            意图→锚点→FTS/向量→RRF→组内排序→M05→预算→组装
   │
   ▼  ⑨使用（宿主 + Chat LLM）  Prompt 九层装配；记忆块标 [数据]
   │
   ▼  ⑩更新（M04/M01）         SUPERSEDES / 版本 / 合并 / 冲突
   │
   ▼  ⑪过期/复核（M01 + Dream） RetentionPolicy 触发
   │
   ▼  ⑫删除（M04 判 + M05 准 + M01 执行）
                               tombstone → 派生索引 → 引用数组传播
```

**责任边界**：M01 只写不判；M02 只读不算业务；M03 只提不存；M04 只判不写；M05 只判不存；Pipeline 只编排不落库；宿主只装配不记忆。

### 16.3 图二 · 短期与长期记忆关系（两条轴分开画）

```
【轴 A：时间尺度（00 §7.5）】          【轴 B：生命周期 tier（memory-unit）】
Session Context  ── 会话结束丢弃 ──►    L1 临时   ── 快速过期 ──►
   │ 默认不自动晋升                      │  仅预留（无存储/无 owner）
   ▼                                    ▼
Working Memory   ── 需 Evidence+Review ─► L2 活跃  ── 定期维护 ──►
   │ 跨会话                               │  推进中计划/近期事件
   ▼                                    ▼
Long-term Memory ── 已是终层 ─────────► L3 长期  ── 稳定保留 ──►
                                        │  稳定人物/偏好/经历/目标
   保留期限：无（全部 ∅）                 │
   可检索性：宿主态 / 不检索              └─ 可检索（同表同索引）
```

**必须分开的原因**：轴 A 回答"这条信息属于哪段时间范围"，轴 B 回答"这条记忆该留多久"。把 Session Context 当成 L1、把 Working Memory 当成 L2，就会导致"会话结束丢 L1"被误解为"临时记忆自动过期"，而实际 L1 根本没有存储。

### 16.4 图三 · 长期记忆内部结构（包含 vs 关联）

```
                       ┌──────────────── 包含关系（分面归属）────────────────┐
                       │  MemoryUnit（唯一条目，不复制）                       │
                       │   semantic_type = 1/6（主区块）                       │
                       │   semantic_tags = 0..12（子类/横切）                  │
                       │   anchor = 1..8（获取入口，多选）                     │
                       └───────────┬──────────────────────────────────────────┘
                                   │ 属于
    ┌──────────┬──────────┬────────┴────────┬──────────┬──────────┬──────────┐
    ▼          ▼          ▼                 ▼          ▼          ▼          ▼
 BLK-01     BLK-02     BLK-03           BLK-04     BLK-06     BLK-08     BLK-05/07/09
 人物画像   自我描述    偏好             关系       事件       经历       项目/稳定事实/情境
 (Person    (SELF_     (PREFER-         (RELATION- (EVENT)    (MILESTONE) （NEW PROPOSAL）
  Profile)   EXPLICIT)  ENCE)            SHIP)
    │          │          │                 │          │          │
    └──────────┴──────────┴────────┬────────┴──────────┴──────────┘
                                    │ 关联关系（类型化，1-hop，有界）
                    ┌───────────────┴────────────────┐
                    ▼                                ▼
              Relation(type, source_ref, target_ref)   *_refs[]（数组，最终收敛为边）
              RELATED_TO / DERIVED_FROM / CONFIRMS /
              CONTRADICTS / SUPERSEDES / BUILDS_ON /
              PART_OF / ABOUT /（新）PARTICIPATED_IN
                    │
                    ▼
        Evidence（原句，BLK-10）    EmotionContext（附件，BLK-11）
```

**读法**：实线向下 = "这条记忆属于哪个区块"（包含）；底部 = "它和谁有关"（关联）。**两者必须分开设计**：区块归属用单选 `semantic_type` + 多选 tag/anchor；跨区块关联用类型化边。当前系统把后者退化成无类型数组，所以"人—事—项目—情绪"连不起来。

### 16.5 图四 · EmotionContext 与 Memory Core 联动（事实与情绪分离）

```
                    ┌─────────────── 事实侧（可长期、可检索、可引用）───────────────┐
 用户原话 ──► Evidence(original_text) ──► Event.content（仅事实）
                    │                          │
                    │                          ├── participants[] ──► PersonProfile
                    │                          └── related_memory_ids ──► MemoryUnit
                    │                                                          │
                    └──────────────────────────┬───────────────────────────────┘
                                               │ 通过 event_id / related_memory_ids
                    ┌──────────────────────────▼───────────────────────────────┐
                    │ 情绪侧（附件、默认关、永不写回事实）                        │
                    │ EmotionContext{label, intensity, confidence,             │
                    │   temporal_position, source_type, evidence_refs}          │
                    │   ├─ 多条并列（历史）                                     │
                    │   └─ status/version/supersedes（NEW，表达变化）            │
                    └──────────────────────────┬───────────────────────────────┘
                                               │ 按需取回（EMOTION_QUERY）
                    ┌──────────────────────────▼───────────────────────────────┐
                    │ ΔE = I×C×R×D（默认关闭；上限；只作用运行时）                │
                    │        │                                                  │
                    │        ▼                                                  │
                    │ Psychology 运行时状态（BLOCKED/DEFERRED）                  │
                    │        │                                                  │
                    │        ▼                                                  │
                    │ Persona Context（情绪只影响语气/长度白名单，不改人格）      │
                    └───────────────────────────────────────────────────────────┘
```

**四条铁律**：① 情绪永不写 `Event.content`；② 情绪默认不注入；③ ΔE 不落库、不改 `validation_state`；④ 情绪只能通过 `affects` 白名单影响语气/长度，不能改人格 traits/boundaries。

### 16.6 图五 · 聊天 LLM 的完整 Prompt 组装（标产生模块）

```
  ┌──────────────────────────── 宿主侧（唯一组装者）────────────────────────────┐
  │                                                                            │
DSH preset ──► [1] 基础系统指令/安全规则       （静态，全局预算）                 │
DSH preset ──► [2] 工具协议                    （静态）                           │
Persona Loader ─► [3] 稳定 Persona 配置         （350 tok / 锚 25 tok）           │
Memory.getProfile ─► [4] 用户/主人画像           （≤200 tok，意图最小集）          │
M02+ALG-06 ──► [5] 相关长期记忆（标 [数据]）    （1200 tok / 2400 char）          │
宿主会话 ──► [6] 当前会话上下文                 （宿主全局预算）                   │
memory.getEvent ─► [7] 相关事件                 （≤150 tok，事件意图时）          │
memory.getEmotionContext ─► [8] EmotionContext  （≤100 tok，默认整层不出现）      │
宿主/预设 ──► [9] 当前任务指令 + 输出约束       （≤150 tok）                      │
  │                                                                            │
  └───────────────┬────────────────────────────────────────────────────────────┘
                  ▼
        Prompt Assembler（单点；顺序即指令强度序）
                  │  裁剪序：background → uncertain → relevant(低边际)
                  │  must 永不主动弃；人格/安全/会话态不占记忆预算
                  ▼
            Chat LLM（L3）──► 回复 ──► QQ
                  │
                  └─ 候选记忆（若有）──► memory.propose ──► Pipeline ──► M01
```

**层级优先序（写死，不允许实现自由发挥）**：System 安全/协议（不可覆盖）> Persona（仅次于安全）> 任务/输出约束 > 用户画像 > 相关记忆（必须标 `[数据]`）> 会话上下文。**记忆永远晚于人格出现**——这是"普通记忆不能覆盖人格"的实现级答案。

### 16.7 推荐架构的八条设计决定

| # | 决定 | 理由 |
|---|---|---|
| D1 | 保留五模块，不新增第六模块；新能力以"扩展点"接入（订阅点/权限挂载/版本协商） | `D-AP-03` 已裁决；但需补接入契约（N-13） |
| D2 | 层级用 T1/T2/T3 记号（或 L1/L2/L3 但**只在 tier 语境**使用），与时间尺度、LLM 层彻底分名 | 消除三义混用（A-C005） |
| D3 | L1 是否在 Core 内一次裁决；建议 **L1 由宿主会话存储承担，Core 只存 L2/L3** | 避免为"临时记忆"造一套存储与清理 |
| D4 | 关系升级为类型化边（含 `Project` 实体），但**只做 1-hop 有界扩展**，不做图检索引擎 | `02 §3` REJECTED + 1200 token 预算 |
| D5 | 检索形态以"分组装配"为外壳、六特征降为组内打分 | `11 §7.17` 禁全局权重；`02` 的六特征仍有价值 |
| D6 | Persona 放宿主侧，Core 只提供只读 `ProfileView` 投影 | `00 L271` 禁止 Core 当人格管理器 |
| D7 | 情绪保持"附件 + 默认关 + 对象级开关"，先建表再谈开关 | CR-02 已裁决；但"改 schema 才能开"是缺陷 |
| D8 | 所有共享枚举收敛为受控词表注册（单一来源 + 投影引用） | H-8：改一个枚举穿透 8–28 文件 |

---

## 17. V0.9 可实施边界

### 17.1 进入 V0.9 的五道门（缺一不可）

| 门 | 内容 | 类型 | 现状 |
|---|---|---|---|
| G1 | DD-001/003 裁决：哪套治理、P0–P3 授权范围 | 用户 | 未决 |
| G2 | N-01 写入守卫修订（P0-1） | 技术 | 未做（S） |
| G3 | N-02 DDL（17 表）（P0-2） | 工程 | 未做（L） |
| G4 | N-05 排序形态裁决（P0-5） | 用户 + 工程 | 未决 |
| G5 | N-06 候选池与人工 confirm（P0-6） | 用户 + 工程 | 未决 |

### 17.2 建议的 V0.9 范围（诚实版）

| 在范围内 | 不在范围内 | 为什么这样切 |
|---|---|---|
| M01 存储（17 表、WAL、事务、Outbox、备份/恢复） | 真实数据的删除/恢复传播（FC-035） | 恢复会复活已删除数据，未解决前不碰真实数据 |
| M02 检索原语（双通道 + RRF + 组内排序 + 预算 + 组装） | ANN/索引路径 | 前轮实测 ANN recall 0.30、索引参数被忽略 |
| 规则版候选生成 + 人工 confirm 通道 | LLM 自动抽取（M03 正式形态） | M03 属 P3，需阶段授权 |
| `Persona` 初始化 + 九层组装（宿主侧，隔离） | 真实 QQ 宿主注入 | 需真实身份/权限；E5=0 |
| 人工标注的小规模评测集 | 生产规模延迟/召回宣称 | 无真实规模测量 |
| 情绪表 + 开关默认关（可观测不启用） | ΔE 实际影响 | CR-03 默认关，启用需重新裁决 |

### 17.3 V0.9 不得声称的事

- 不得声称"检索可用"（中文 FTS 产品判据未过、分词器未定、排序形态未裁决）。
- 不得声称"权限已验证"（E5=0，全部为模拟层）。
- 不得声称"删除会传播"、"恢复是安全的"（FC-035 未解）。
- 不得声称"技术选型已冻结"（全部 EXP-DEFAULT）。
- 不得把 Mock 通过写成真实验收；不得把"设计已定义"写成"已授权实施"。

### 17.4 V0.9 应做的六个实验（都能产出可判定结论）

| E# | 实验 | 判据 |
|---|---|---|
| E1 | 分组装配 vs 全局六特征：同一 200 条语料，人物查询/事件查询各 50 条 | 画像命中率、事件命中率、注入纯度 |
| E2 | 相关性下限与已注入集合：20 条无关 + 1 条相关 | 无关注入条数 = 0 |
| E3 | full vs anchor 人格注入（N=1/10/20） | 人设保持率、预算占用、旁白/汇报率 |
| E4 | 中文 FTS 两候选分词器 | 19 条中文查询的 hit@10 |
| E5 | 情绪反查（建表后、开关临时开） | 给定 event_id 取回全部情绪，recall=1 |
| E6 | 端到端 24 步链（附件 06 §7） | 17 步闭环全部通过 |

---

## 18. 最终结论

### 18.1 字段判定

```
CORE_ARCHITECTURE_STATUS      = PARTIAL
MAJOR_ARCHITECTURAL_CONFLICTS = PRESENT
整体系统结论                   = C（核心能力存在系统性缺失，但架构不需推倒重来）
```

**为什么不是 A/B**：七项能力 0 项 COMPLETE；P0 有 7 条，其中 3 条（tier、关系、Persona）是**整条能力链没有设计**，不是"少量阻断"。
**为什么不是 D**：骨架（五模块/五对象/三轴/锚点路由/Outbox）自洽且可扩展；所有缺口都能在**不改变架构形态**的前提下补齐（本报告第 15 章 N-01…N-13）；没有证据表明目标不可达。

### 18.2 十一个必答问题

**1. 是否真正拥有完整的短期和长期记忆体系，而不只是三层字段？**
**否。** 拥有的是"长期存储 + 三层枚举 + 一套时间尺度词汇"；短期无存储无主人，层级 0 转换，保留/降级/复核全空。

**2. 长期记忆是否已形成多个有组织、有关系、有优先级、可联合检索的区块？**
**部分。** 有组织（分面）、有优先级（意图路由，但契约反编码）、有关系（仅记忆↔记忆、无类型端点）、**没有联合检索**；项目/稳定事实/情境三个区块缺失。

**3. 区块之间是否有树状或图状关系结构？推荐哪一种？**
现有是**分面池 + 浅层关系图**，不是树也不是互连区块。**推荐 HYBRID**：分面分类负责归属，类型化边负责关联，1-hop 有界扩展负责联合召回。

**4. 情绪附件能否和事件、人物、长期记忆及当前互动真正联动？**
**不能（当前契约下）。** 对象与门禁完整、事实隔离正确；但无表、三开关 `const:false`、`maxItems:0`、无人物/互动绑定、无 Persona 桥。

**5. 首次人格 Prompt 如何输入、保存、结构化？每轮如何获取并组装？**
**现状：没有方案。** 只有 FUTURE 定位与三条禁令；实践是手写卡 + user 层全文注入。**推荐方案（NEW PROPOSAL）**：`initPersona` 七步初始化 → `profiles/persona/<id>/` 持久化 → 宿主 `PersonaLoader` 单点读取 → 九层模板 + 分层预算组装 → 版本归档与回滚。详见附件 04。

**6. 记忆模块是否可独立调整？调整 Anchor/情绪/Tokenizer/Persona 是否影响无关模块？**
**整体 LOW。** Anchor 变更外溢（schema/路由/权重/测试）；Tokenizer 影响 M02+FTS+索引重建；Persona 是唯一真正零外溢的（宿主侧）。8/14 场景多模块。

**7. 核心算法是否达到可实现程度？哪些只是概念？**
**MOSTLY_COMPLETE。** 检索/写入主干可达 COMPLETE/MOSTLY；**MISSING**：importance、tier 降级、跨区块联合召回、Persona 组装；**CONFLICTED**：排序形态、冲突枚举。

**8. 是否还存在会让 V0.9 无法建立的严重架构冲突？**
**存在 7 条 P0。** 最硬的三条：写入默认态非法（P0-1）、0 DDL（P0-2）、写入闭环无授权通道（P0-6）。

**9. 能否通过少量合同和模块边界修订解决，而不必推倒重来？**
**能。** 7 条 P0 中 6 条是"合同/DDL/状态机/宿主模块"级修订；只有 P0-4（关系模型）与 P0-7（Persona）需要新增对象/新模块，但都不触动 M01/M02 的骨架。

**10. 若没有根本性问题，现有方案能否形成一套真正可落地的 Memory Core V0.9？**
**能，但必须先过 17.1 的五道门，并接受"窄版 V0.9"的边界**（M01+M02+规则版候选+人工确认+宿主侧 Persona；不碰真实数据、不宣称检索可用）。

**11. 修复顺序建议？**
`P0-1（S）→ N-02 DDL（L）→ N-06 候选/确认（M）→ N-05 排序（M）→ N-03 tier（M）→ N-04 关系（M）→ N-09 Persona（M）→ N-07/N-08（S）→ N-10 情绪（M）→ N-11/N-12（S）`。**前四项完成即可开工 M01/M02。**

### 18.3 报告末尾必须明确告诉用户的六件事

**第一：目前是否存在严重架构冲突。**
**存在。** `MAJOR_ARCHITECTURAL_CONFLICTS = PRESENT`。

**第二：如果存在，究竟是哪几项。**
七条 P0：① 默认态非法导致无合法新建记忆；② 0 DDL 且 3 个对象无表；③ 层级生命周期不存在（0 转换/无 L1/无保留）；④ 关系非一等公民（无 Project/无类型端点）；⑤ 读取两套互斥算法且授权那套绑 v1 字段；⑥ 写入闭环无授权通道；⑦ Persona 无初始化与逐轮组装。另有 11 条 P1（返回契约三套、无相关性下限、anchor 权重反编码、tier 过滤缺失、情绪无表、画像默认全量、`last_retrieved` 无主、参数未登记、模块独立不足、情绪正文可进事实池等）。

**第三：现有设计能否在不推翻整体架构的前提下修复。**
**能。** 所有 P0 都是合同、DDL、状态机、宿主模块级修订；M01/M02 的骨架、五对象、Outbox/派生索引、权限模型、检索五阶段均可原样保留。**不需要推倒重来。**

**第四：短期记忆、长期记忆、记忆区块、情绪附件和 Persona 是否已经构成完整系统。**
**没有。** 五项里：长期记忆有存储无层级；短期记忆无存储；区块有分类无关联与联合召回；情绪有对象无存取；Persona 只有边界没有管线。它们目前是**五个半成品，不是一套系统**。

**第五：核心算法是否足以交给工程 Agent 实施。**
**部分足以。** 检索/写入主干（ALG-01/02/03/05/06、Evidence 回源、Outbox）可以；质量（importance/confidence）、层级（tier 升降）、组织（跨区块）、人格（组装）四类不行。21 条重点算法中 10 条不能直接实施。

**第六：进入 V0.9 之前，真正必须完成的工作还有多少。**
**5 项前置（2 个用户裁决 + 3 个技术修订）+ 1 次 schema 修订 + 1 份 DDL + 7 条 P0 修复 + 4 条能力链补齐（tier/关系/Persona/情绪）**。按工作量估：S 级 4 项、M 级 6 项、L 级 1 项；**最小可运行路径为 N-01→N-02→N-06→N-05**（约 2–3 周量级，不含实验与验收）。

---

## 附录 A · 证据地图

| 文件 | 内容 | 作者 |
|---|---|---|
| `01-MEMORY-TIER-AND-LIFECYCLE-MATRIX.md` | 三层矩阵、tier 状态机实测、16 条缺失转换规则 | A |
| `02-LONG-TERM-MEMORY-STRUCTURE.md` | 9 区块模型、树/图/混合对比、关系与传播、联合召回 | B |
| `03-EMOTION-MEMORY-INTEGRATION.md` | 情绪数据完整性、13 问、10 步链路、缺失算法 | C |
| `04-PERSONA-PROMPT-FLOW.md` | 初始化流程 A、逐轮组装 B、9 层模板、四接口、预算降级 | D |
| `05-MODULE-ADJUSTABILITY-MATRIX.md` | 6 张表、14 场景、H-1…H-8、12 条改善 | E |
| `06-CORE-ALGORITHM-COMPLETENESS.md` | 76 条算法、21 条重点、12 项补齐、24 步验收链 | F |
| `07-MAJOR-CONFLICTS-ONLY.md` | FC 三分类、5 条 P0、8 条 P1、权责/模型/接口表 | G |
| `SUBAGENT-REPORTS\A–G` | 各路上作过程与方法附录 | 各 Agent |
| `EVIDENCE\a…g\` | 机器校验脚本与原始 JSON/TXT 输出 | 各 Agent |

## 附录 B · 方法与局限

- **证据等级**：E1 文档 / E2 schema·config 静态 / E3 隔离校验 / E4 真实文件依赖 / **E5 = 0（未触真实宿主）**。本报告所有"不可执行"结论均为 E1–E3 级；所有"已验证"均为前轮 E3–E4 级。
- **未做**：未重做全量文件清点；未逐条重审 49 个 FC；未做安全专项；未测性能/规模；未触真实身份、真实记忆、QQ Bridge、DSH 会话。
- **口径更正**：任务书称"31 个 schema"，实测为 **20 schemas + 11 config = 31 个 JSON 合同**（另 3 份 md）。
- **父 Agent 独立判断的三处修正**：① FC-018 严重性由 P0 下调为 P1（存在经 M01 回源的合法实现）；② 任务书假设"长期记忆是树/图之争"，实测是**分面池 + 浅层关系图**，故结论表述改为 HYBRID 而非二选一；③ 全链路能力评级统一采用 `PARTIAL/CONCEPT_ONLY`，不采用 G 路自造的 `SOUND_SKELETON__NON_EXECUTABLE_LOOP`（该描述保留在附件 07 作为解释性短语）。
- **父 Agent 独立判断的第四处修正**：Persona 综合评级由 D 路的 `PARTIAL` 下调为 `CONCEPT_ONLY`（理由见 §7.5 说明框）；分项评级（初始化 CONCEPT_ONLY / 逐轮 PARTIAL）保留不变。
- **置信度**：能力评级 HIGH；P0 清单 HIGH；工作量估算 MEDIUM；参数/效果类结论 LOW（无实测）。
- **基线未改动**：`C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design` 本轮全程只读；聚合哈希与前轮记录一致（`32BF58…BEBBA`）。

---

**报告结束。**
*本轮为只读能力审查，未修改任何正式设计文档、Schema、API、DDL 或裁决台账，未编写 V0.9 产品代码。*