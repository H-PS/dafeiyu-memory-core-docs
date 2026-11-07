# DD-AUTHORITY V2 — 架构升级基准（统一事实来源 · 第二版）

> 版本 2.0 ｜ 2026-11-04 ｜ 状态：**APPROVED**（依据：用户《Memory Core R4.1 V1 统一架构升级与设计文书迭代总任务书》，2026-11-04 指令）
> 效力：**与 DD-AUTHORITY v1 冲突时以本文为准**（用户明确要求把该任务书定义的能力纳入 V1 正式架构）；**本文不推翻正式裁决台账**——台账冲突见 §12，一律"保留原裁决 + 标记重新裁决"，不得静默改写。
> 作用：所有文档（00–15）、契约、状态机、测试规格的唯一枚举/字段/接口定义来源。
> 备份：改造前 45 文件已备份至 `E:\[PROJECT]\备份全集\Memory-Core-R4.1-Design-pre-V1upgrade-20261104-1810\`（含 `_SHA256-BEFORE.csv`，逐字节校验通过）。

---

## 1. 架构升级总览（17 项能力 → V1 正式范围）

按任务书 §1.1，以下能力**属于 V1 必须定义的系统能力**，MUST NOT 标记为可选或"无限期 Future"：

| # | 能力 | 落点文档 |
|---|---|---|
| 1 | 自动记忆摄取 | 09 + 03 |
| 2 | LLM 协作式语义理解 | 10 + 02 |
| 3 | 信息自动碎片化 | 09 + 01 |
| 4 | 多维度语义分类（6 类 + 多标签） | 11 + 01 |
| 5 | 独立记忆获取锚点（8 锚点） | 11 + 02 |
| 6 | 人物核心信息优先获取机制 | 11 + 12 + 02 |
| 7 | 生命周期与语义类型解耦 | 01（tier 与 semantic_type 正交） |
| 8 | 原始证据与原句追溯 | 13 + 01 |
| 9 | 聊天 LLM 主动读取记忆 | 14 + 08 |
| 10 | 聊天 LLM 主动提出记忆写入 | 14 + 08 |
| 11 | 人物画像持续整合 | 12 + 02(M04) |
| 12 | 事件事实与情绪关联分离 | 13 + 01 |
| 13 | 事件情绪对当前情绪的有限影响 | 13 + ΔE 公式（§9） |
| 14 | 多模型能力分级和动态调用 | 10 |
| 15 | 自动去重、冲突检测和版本维护 | 02 + 09 |
| 16 | 统一权限控制 | 04 |
| 17 | 可观察、可配置、可验证的运行体系 | 15 + 05 + 06 |

**实施授权边界（不得混淆）**：设计范围 = P0–P6 全部；**实施授权仍按阶段授予**，历史裁决"当前 P0–P3 = M01+M02"保持不变，新增 P3–P6 内容属"V1 范围内、待阶段授权"。禁止把"设计已定义"写成"已授权实施"。

---

## 2. 规范枚举（MUST 逐字使用，禁止同义变体）

### 2.1 semantic_type（语义主类，单选，6 值）
```
PERSON           人物信息（身份/背景/自我描述/特征/能力/价值观/长期目标/自我认知）
PREFERENCE       偏好与互动习惯
RELATIONSHIP     关系信息
EVENT            事件信息（事实）
CURRENT_STATE    当前状态（计划/进行中/阶段性目标/困难/进度/临时状态）
EMOTION_CONTEXT  情感关联
```

### 2.2 semantic_tags（多标签，可多选；含以下受控标签 + 扩展必须登记）
```
SELF_EXPLICIT    主人明确自我描述（与 PERSON 组合）
SELF_DERIVED     系统推断的人物特征（与 PERSON 组合）
PROCEDURAL       流程/方法类（原 procedural 语义的承接标签）
OBSERVATION      派生观察（原 observation 语义的承接标签）
IDENTITY         身份背景
VALUE            价值观
GOAL             目标
ABILITY          能力
HABIT            习惯
INTERACTION      互动方式
CONVENTION       长期约定
MILESTONE        重要经历
```

### 2.3 anchor（获取锚点，多选，8 值）
```
SELF_EXPLICIT
SELF_DERIVED
PERSON_PROFILE
PREFERENCE
RELATIONSHIP
EVENT
CURRENT_STATE
EMOTION_CONTEXT
```

### 2.4 subject_type（主体，6 值）
```
OWNER          主人
ASSISTANT      聊天机器人
MEMBER         其他群成员
THIRD_PARTY    其他人物
GROUP          群体
SYSTEM         系统信息
```
配套：`subject_id`（稳定 ID 字符串，MUST 由 EntityResolver 解析，禁止因出现"我/你/他"直接认定主体）。

### 2.5 memory_tier（生命周期层级，单选，3 值；与 semantic_type 正交）
```
L1   临时记忆（当前对话/短期任务/未确认临时状态/待处理候选；可快速过期；不自动视为长期事实）
L2   活跃记忆（推进中计划/阶段性任务/近期重要事件/阶段性人物状态/关系变化；需定期维护与过期处理）
L3   长期记忆（稳定人物描述/明确自我描述/长期偏好/长期关系/重要经历/长期目标/已验证长期特征；保留证据与版本；不因新信息无条件覆盖）
```
**铁律**：tier ≠ 重要程度；事件即使 L3 也不得在所有查询中优先于人物描述（见 §8 锚点优先级）。

### 2.6 memory.status（记忆状态，单选，4 值；原 lifecycle_state 的成熟态部分）
```
ACTIVE
SUPERSEDED
ARCHIVED
DELETED
```

### 2.7 candidate.status（候选状态，单选，6 值；任务书 §7.16）
```
PENDING
AUTO_ACCEPT
NEEDS_REVIEW
REJECTED
DEFERRED
COMMITTED
```
> ⚠️ `AUTO_ACCEPT` 与台账 D-AP-05（confirmed 仅 M04 通道写入）存在张力 → 见 §12 CR-01，**实施前必须重新裁决**；设计上规定：AUTO_ACCEPT 只表示"无需人工复核即可进入 COMMITTED 流程"，**不写 validation_state=confirmed**，仍须经 M04 的整合通道写入。

### 2.8 validation_state（保留 v1，4 值，与台账绑定）
```
unverified / confirmed / disputed / rejected
```

### 2.9 source_type（来源类型，单选，5 值；任务书 §8.6 取代 v1 的 source_type 一套）
```
USER_EXPLICIT       用户明确表达
USER_INSTRUCTION    用户明确记忆指令（要求保存）
ASSISTANT_INFERRED  模型推断
EXTERNAL_SOURCE     外部来源（文件投喂/导入/工具结果）
SYSTEM_GENERATED    系统生成（派生视图/整合结果）
```

### 2.10 visibility（可见性，单选，3 值）
```
owner_private    仅授权主体可读（默认）
group_shared     明确共享给群组
public           仅在未来明确启用时（V1 默认禁用）
```
> 与 v1 的 `scope` 五值（owner_private/group_shared/session_only/project_scoped/system_internal）**必须映射**：`scope` 保留为内部细分字段，`visibility` 为对外三值；映射表见 §4.3。

### 2.11 intent（查询意图，9 类；任务书 §14.1）
```
PERSON_QUERY
PROFILE_QUERY
PREFERENCE_QUERY
RELATIONSHIP_QUERY
EVENT_QUERY
CURRENT_STATE_QUERY
EMOTION_QUERY
ORIGINAL_QUOTE_QUERY
GENERAL_MEMORY_QUERY
```

### 2.12 read_mode（读取模式，4 值；任务书 §8.5）
```
summary            整合后的摘要
memory             独立记忆单元
evidence           原始句子 + 必要上下文
related_evidence   指定记忆 → 相关原句
```

### 2.13 temporal_position（情绪相对事件的时间位置，4 值）
```
BEFORE / DURING / AFTER / UNKNOWN
```

### 2.14 emotion_label（受控情绪字典；V1 起步 15 标签，配置管理）
```
neutral, happy, excited, calm, relieved, grateful, affectionate,
sad, anxious, frustrated, angry, tired, lonely, guilty, confused
```
> 标签词表 MUST 由配置管理（台账 J4）；禁止模型自创标签；越界标签映射为 `neutral` + 记录 raw_label（不丢弃原始表达）。

### 2.15 关系类型（保留 v1 八种）
```
RELATED_TO, DERIVED_FROM, CONFIRMS, CONTRADICTS, SUPERSEDES, BUILDS_ON, PART_OF, ABOUT
```

---

## 3. 五个核心对象的权威字段表

### 3.1 MemoryUnit（独立记忆单元）
| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| memory_id | string | ✓ | 稳定唯一 ID（uuid 格式） |
| content | string | ✓ | **规范化后的独立记忆内容**（可独立理解；MUST NOT 当作原句） |
| semantic_type | enum §2.1 | ✓ | 语义主类 |
| semantic_tags | array<enum> | ✓（可为空数组） | 多标签 |
| subject_type | enum §2.4 | ✓ | 主体类型 |
| subject_id | string | ✓ | 主体稳定 ID |
| anchor | array<enum> | ✓ | 获取锚点（≥1） |
| memory_tier | enum §2.5 | ✓ | L1/L2/L3 |
| source_type | enum §2.9 | ✓ | 来源性质 |
| confidence | number 0–1 | ✓ | 提取/推断置信度（≠事实真实性） |
| validation_state | enum §2.8 | ✓ | 默认 unverified |
| status | enum §2.6 | ✓ | 默认 ACTIVE |
| visibility | enum §2.10 | ✓ | 默认 owner_private |
| scope | enum(v1 五值) | ✓ | 内部细分（映射见 §4.3） |
| valid_from / valid_until | datetime\|null | ✓ | 事实有效期（未知为 null，不猜） |
| event_refs | array<id> | ✓ | 关联事件 |
| evidence_refs | array<id> | ✓ | 证据引用（正式记忆 MUST 非空，除非 source_type=SYSTEM_GENERATED 且显式标注） |
| relation_refs | array<id> | ✓ | 关系 |
| emotion_refs | array<id> | ✓ | 情绪关联 |
| merged_from | array<id> | ✓ | 整合来源（M04 合并时） |
| profile_field | string\|null | ✓ | 归属画像字段（如 identity/goal/value） |
| version | integer ≥1 | ✓ | 乐观并发 |
| created_at / updated_at | datetime | ✓ | UTC RFC3339 |
| tags_extra | object | ✓ | 扩展标签（必须登记） |

### 3.2 Evidence（原始证据）
| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| evidence_id | string | ✓ | uuid |
| source_type | enum(chat/file/import/tool/system) | ✓ | 证据来源形态 |
| source_id | string | ✓ | 会话/文件/工具调用 ID |
| source_locator | string\|null | | 定位（session:msg:seq） |
| speaker_id | string | ✓ | 发言者主体 ID |
| timestamp | datetime\|null | | 原始时间（未知 null） |
| original_text | string | ✓ | **原始表达（原句）**；MUST NOT 被规范化覆盖 |
| context_before / context_after | string\|null | | 必要上下文（受限长度） |
| visibility | enum §2.10 | ✓ | 证据自身可见性 |
| scope | enum(v1 五值) | ✓ | 内部细分 |
| content_hash | sha256 | ✓ | original_text 哈希（去重/校验） |
| retention_policy | string\|null | | 保留策略 |
| revoked | boolean | ✓ | 失效标记（失效后关联记忆 MUST 标记证据缺失） |

### 3.3 Event（事件事实）
| 字段 | 类型 | 必填 |
|---|---|---|
| event_id | string uuid | ✓ |
| content | string（**仅事实**，禁止写入情绪判断） | ✓ |
| time_start / time_end | datetime\|null | ✓ |
| participants | array<subject_id> | ✓ |
| location | string\|null | ✓ |
| result | string\|null | ✓ |
| related_memory_ids | array<memory_id> | ✓ |
| evidence_refs | array<evidence_id> | ✓ |
| status | enum(ACTIVE/SUPERSEDED/ARCHIVED/DELETED) | ✓ |
| visibility / scope | enum | ✓ |
| version / created_at / updated_at | | ✓ |

### 3.4 EmotionContext（情感关联，与事件分离）
| 字段 | 类型 | 必填 |
|---|---|---|
| emotion_id | string uuid | ✓ |
| event_id | string uuid｜null（可无具体事件） | ✓ |
| subject_id | string | ✓ |
| emotion_label | enum §2.14 | ✓ |
| raw_label | string\|null | ✓ |
| intensity | number 0–1 | ✓ |
| confidence | number 0–1 | ✓ |
| temporal_position | enum §2.13 | ✓ |
| source_type | enum(USER_EXPLICIT/ASSISTANT_INFERRED) | ✓ |
| evidence_refs | array | ✓ |
| related_memory_ids | array | ✓ |
| visibility / scope | enum | ✓ |
| created_at | datetime | ✓ |

### 3.5 PersonProfile（人物画像，M04 维护的派生视图）
| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| profile_id | string | ✓ | |
| subject_id | string | ✓ | 通常 owner |
| section | enum(identity/self_explicit/preference/goal/value/ability/experience/habit/interaction/derived/current_state) | ✓ | 画像分区 |
| content | string | ✓ | 整合后描述 |
| basis | enum(SELF_EXPLICIT / SELF_DERIVED / MIXED) | ✓ | **明确自述与推断不得合并成无来源结论** |
| source_memory_ids | array | ✓ | 追溯 |
| source_evidence_ids | array | ✓ | 追溯 |
| confidence | number | ✓ | |
| profile_version | integer | ✓ | |
| updated_at | datetime | ✓ |

---

## 4. 旧模型 → 新模型映射（MUST 提供迁移路径，不得丢语义）

### 4.1 semantic_type 映射（v1 `type` 七值 → v2）
| v1 type | v2 semantic_type | v2 semantic_tags |
|---|---|---|
| semantic | 按内容路由：关于人/特征→PERSON；偏好→PREFERENCE；关系→RELATIONSHIP；计划/状态→CURRENT_STATE；默认 PERSON | 视内容加 IDENTITY/VALUE/GOAL/ABILITY |
| episodic | EVENT | MILESTONE（重要经历时） |
| procedural | PERSON | **PROCEDURAL**（能力/流程） |
| preference | PREFERENCE | HABIT/INTERACTION/CONVENTION 视内容 |
| intent | CURRENT_STATE | GOAL |
| relationship | RELATIONSHIP | — |
| observation | PERSON | **OBSERVATION**（+ source_type=ASSISTANT_INFERRED，basis=SELF_DERIVED） |

### 4.2 其他字段映射
| v1 | v2 | 规则 |
|---|---|---|
| lifecycle_state(candidate/pending_review) | **candidate.status** | candidate→PENDING；pending_review→NEEDS_REVIEW |
| lifecycle_state(active/superseded/archived/deleted) | **status** | 同名大写 |
| content_kind(fact/opinion/…) | **废弃** → source_type + semantic_tags | inference/hypothesis→ASSISTANT_INFERRED；其余→USER_EXPLICIT；procedure→EXTERNAL_SOURCE |
| source_type(v1 六值：direct_user_statement 等) | **§2.9 五值** | direct_user_statement→USER_EXPLICIT；explicit_user→USER_EXPLICIT；conversation→USER_EXPLICIT；repeated_statement→USER_EXPLICIT；imported/imported_document→EXTERNAL_SOURCE；tool_result→EXTERNAL_SOURCE；system_event/system_generated→SYSTEM_GENERATED；human_reviewed→USER_EXPLICIT（+审计标记）；assistant_observation/approved_inference→ASSISTANT_INFERRED |
| importance(v1) | **保留但不入排序公式**（沿用 v1 决策：过滤与平局） | |
| confidence(v1) | 保留 | 不得与事实真实性混同 |
| observed_at / recorded_at | 保留 | 事实时间≠记录时间 |

### 4.3 visibility ↔ scope 映射
| visibility | scope（内部） |
|---|---|
| owner_private | owner_private / session_only / project_scoped（细分） |
| group_shared | group_shared |
| public（禁用） | — |
| —（不外露） | system_internal（仅内部元数据，永不对聊天 LLM 输出正文） |

### 4.4 A/B 层兼容（保留历史裁决语义）
- A 层向量语义记忆 → MemoryUnit（semantic_type 按 §4.1 路由；evidence 缺失时标"无原始证据"，MUST NOT 编造原句）。
- B 层 SocialV2：activeTopic→CURRENT_STATE；pendingThought→CURRENT_STATE（GOAL 标签）；memberImpression→PERSON（SELF_DERIVED/OBSERVATION，source_type=ASSISTANT_INFERRED，保留推断性质，禁止跨用户泄露）；稳定偏好→PREFERENCE；稳定关系→RELATIONSHIP。

---

## 5. Ingestion Pipeline（16 阶段，名称 MUST 逐字一致）

```
1  InputNormalizer        统一输入格式（聊天/多轮/文件投喂/明确记忆指令/LLM 提议），保留来源与时间
2  RulePreprocessor       合法性、长度、重复、来源、权限预判、明确记忆指令识别（不得用关键词决定复杂语义）
3  MemoryValueEvaluator   长期保存价值判定（明确保存/稳定人物/稳定偏好/重要事件/阶段状态/闲聊/重复/不宜保存）
4  LLMExtractor           L1/L2 结构化抽取
5  SemanticSplitter       语义碎片化（一条一核心语义；主体/对象/时间明确；指代可独立理解；原句可追溯；禁机械切句）
6  MemoryClassifier       多标签分类 → semantic_type/subject_id/anchor/lifecycle_candidate/source_type/confidence
7  TemporalResolver       过去/当前/未来计划/假设/已取消/已结束/持续（禁止把未来计划写成已发生）
8  EntityResolver         人物与对象识别、统一 ID、处理别名/代词/昵称/关系指代；不确定保留待解析，禁止强行合并
9  AnchorAssigner         锚点分配（明确自述→SELF_EXPLICIT；推断→SELF_DERIVED；事件→EVENT；画像与事件不得混为一个单元）
10 EvidenceBinder         记忆单元 ↔ 原始证据绑定（支持反向取原句）
11 EmotionExtractor       有依据的事件情绪（允许为空；明确表达与推断分离）
12 DuplicateDetector      完全/语义重复、相近不同事件、同一事实不同表达（不同时间的相似事件禁止合并）
13 ConflictDetector       新旧矛盾、状态变化、自述与推断不一致、关系变化、时间冲突（禁止自动覆盖重要旧记忆）
14 CandidateValidator     Schema/主体/时间/来源/证据/锚点/权限/置信度/冲突状态校验
15 PermissionGate         M05 权限门禁（读/写/可见性）
16 M03/M04 → M01 Commit → Outbox → FTS5/LanceDB
```
Pipeline 职责：协调，**不直接写 SQLite、不取代 M03、不绕过 M05**。

---

## 6. LLM 三层（双档 + 聊天模型）

| 层 | 职责 | 说明 |
|---|---|---|
| L1 轻量理解 | 价值判断、候选提取、语义拆分、多标签分类、主体识别、时间识别、证据绑定、初步情绪识别 | 低成本高频；可批量异步 |
| L2 强语义 | 多条记忆整合、人物画像更新、复杂指代、跨事件关联、冲突分析、关系变化、复杂时间线、复杂情感关联 | 冲突处理与整合必须 L2 |
| L3 聊天模型 | 正常聊天、主动读记忆、判断是否需查询、选择锚点、请求原句、主动提议写入、保持连续性 | 复杂任务中可兼任 L2 |

调用纪律：普通聊天不触发完整抽取；明确记忆指令优先；普通候选批量异步；**后台抽取 MUST NOT 阻塞聊天**；单次失败不得丢弃候选（进入 DEFERRED/重试）。
模型选型 MUST 过统一测试集（中文理解/口语/结构化输出/指代/时间推理/事实与推断区分/多标签/事件拆分/画像整合/冲突检测/情绪识别），**不得以模型名和参数量代替能力测试**。

---

## 7. Memory Read/Write API（聊天 LLM 唯一入口）

```
memory.search(query, opts)                 混合检索（Anchor 路由 + FTS5/LanceDB + RRF）
memory.getProfile(subject_id, sections?)   人物画像快速读取（人物理解任务优先）
memory.getByAnchor(anchor[], subject_id?)  指定锚点读取
memory.getEvent(event_id)                  事件（事实）
memory.getEvidence(memory_id | evidence_id, mode)   原句证据（related_evidence 反向）
memory.getRelated(memory_id)               关联记忆/关系/事件
memory.getEmotionContext(event_id | memory_id)      情绪关联（按需，默认不注入）
memory.getMemoryById(memory_id, version?)  指定记忆直读
memory.propose(candidate)                  提议写入（候选，不直接写库）
```
统一返回 MUST 含：`memory_id, content, semantic_type, anchor, source_type, confidence, evidence_refs, lifecycle(tier+status), visibility`；按 read_mode 决定是否附证据与情绪。
**每次读取 MUST 经 M05**；`memory.propose` 只产生候选，M01 为唯一权威写入执行者，任何模块不得绕过链路。

---

## 8. 检索五阶段 + 意图路由

```
阶段1 意图识别与锚点路由     Intent Resolver(9 类) → Anchor Router
阶段2 锚点候选检索           按锚点取候选（画像/偏好/事件/状态/情绪/证据）
阶段3 排序                   语义相关性 + 证据质量 + 时间有效性（分特征归一化，禁单一固定权重全局排序）
阶段4 权限过滤               M05（DENY/INDETERMINATE 一律排除）
阶段5 证据扩展与上下文构建    按需 evidence expansion + Token 预算 + 分区组装
```
**人物理解默认获取顺序**（非所有查询的固定顺序）：SELF_EXPLICIT → PERSON_PROFILE → PREFERENCE → SELF_DERIVED → 相关 RELATIONSHIP → 相关 EVENT。
其他路由：事件查询 EVENT→证据→关联人物→（按需）情绪；关系查询 RELATIONSHIP→变化事件→画像→证据；状态查询 CURRENT_STATE→近期事件→目标偏好；情绪查询 EMOTION_CONTEXT→事件→Psychology 当前状态。
**人物画像优先**：人物理解任务不得把大量事件碎片排在画像前面；事件回忆任务不受此限。
**情绪默认不注入**：除非查询涉及情绪或当前对话与事件情绪关联足够，否则默认不注入 EMOTION_CONTEXT。

---

## 9. 情绪影响公式与门禁（任务书 §10.5）

```
ΔE = I × C × R × D
I = 事件情绪强度 (0–1)
C = 情绪判断可信度 (0–1)
R = 当前话题与该事件的关联程度 (0–1)
D = 时间衰减系数 (0–1，随时间单调递减)
```
门禁（保守默认，参数化待标定）：
1. `inferred` 权重 < `explicit` 权重（`emotion.inferred_weight` < 1.0）。
2. 无关联话题不触发（R 低于 `emotion.relevance_threshold` ⇒ ΔE=0）。
3. 单事件不得持续主导（单事件 ΔE 受 `emotion.max_influence` 上限）。
4. 多事件影响有总上限（求和后裁剪到 `emotion.max_influence`）。
5. 时间衰减（`emotion.decay_rate`）。
6. **默认关闭**（`emotion.influence_enabled=false`），启用须经台账重新裁决（见 §12 CR-03）。
7. 禁止由"沉默"触发（台账 K-6/CR-09）：无明确事件即无 ΔE。
8. ΔE 只作用于 Psychology 运行时状态，**不得写回 Event 事实、不得改记忆 validation_state**。

---

## 10. 路线图 P0–P6（V1 范围；实施按阶段授权）

| 阶段 | 内容 | 授权状态 |
|---|---|---|
| P0 | 环境、目录、Schema、数据模型、接口、配置体系、测试框架 | 待 D-3（原 P0–P3 已含，未变） |
| P1 | M01：SQLite、MemoryUnit、Evidence、Event、EmotionContext、关联表、版本、Outbox、备份恢复 | 原 P0–P3 范围 |
| P2 | M02：FTS5、LanceDB、Anchor Router、混合检索、RRF、证据读取、Token 预算、权限挂载 | 原 P0–P3 范围 |
| P3 | **M03 + Ingestion Pipeline**：LLM 抽取、碎片化、多标签、时间主体识别、锚点分配、证据绑定、情绪候选、候选校验 | **新增，需阶段授权** |
| P4 | M04：人物画像、整合、冲突、状态更新、关系维护、情绪关联维护 | 新增，需阶段授权 |
| P5 | M05 + 聊天 LLM 接入：权限、Read/Write API、主动读写、原句、情绪关联 | 新增，需阶段授权 |
| P6 | Psychology 集成与完整验收：事件情绪读取、有限影响、衰减、上限、全链路/性能/真实场景 | 新增，需阶段授权 |

---

## 11. 文档编号体系（交付映射，任务书 §21）

| 编号 | 文件 | 覆盖任务书条目 |
|---|---|---|
| 00 | 总体架构与设计原则 | A-1 架构总纲、A-2 模块职责、五原则、文档索引 |
| 01 | 记忆本体与数据模型 | B-1 数据模型、B-2 SQLite Schema、A-3…A-9 对象 |
| 02 | 核心算法规范 | B-6 混合检索、B-7 Anchor Router、A-11 画像整合算法、§9 ΔE |
| 03 | 系统运行与协作协议 | B-8 Read API、B-9 Write API、B-10 Outbox、A-3 Pipeline 编排 |
| 04 | 权限与治理规范 | B-11 权限设计、15 权限链路 |
| 05 | 可靠性与运维规范 | C-7 异常降级、B-10 Outbox 可靠性 |
| 06 | 测试与评估规范 | C-3~C-6、C-8、18 测试体系 |
| 07 | 扩展与演进规范 | 版本迁移（新增对象）、C-1/C-2 路线图与交付标准 |
| 08 | DSH 与 QQ Bridge 集成规范 | A-10 Psychology 接口、聊天 LLM 接入、14 读写落地 |
| **09** | **记忆摄取 Pipeline** | A-3 Pipeline 全 16 阶段 |
| **10** | **LLM 调用体系** | A-4、14 多模型分级与动态调用、测试集 |
| **11** | **Memory Anchor 与语义分类** | A-5 Anchor、A-6 人物优先、语义 6 类多标签 |
| **12** | **人物画像整合** | A-11 画像、9 画像系统、明确自述优先 |
| **13** | **Evidence 与 EmotionContext** | A-8 原句追溯、A-12 事实情绪分离、A-13 有限影响、10 章 |
| **14** | **Chat LLM 读写 API** | A-9/A-10、11 章、B-8/B-9 |
| **15** | **参数注册表** | 17 章（15 属性 × 全部参数） |
| contracts/ | schemas（新增 memory-unit / event / emotion-context / anchor-set / read-request / read-response / propose-request）、api、state-machines（新增 candidate 6 态、emotion 影响）、config、error-codes | B-3~B-5、B-12 |

---

## 12. 台账冲突清单（MUST 保留原裁决 + 标记重新裁决，禁止静默改写）

| 编号 | 冲突 | 冲突双方 | 影响 | 处理（本设计采取） |
|---|---|---|---|---|
| **CR-01** | `AUTO_ACCEPT` 候选状态 vs "confirmed 仅 M04 通道写入" | 任务书 §7.16 ↔ 台账 D-AP-05 | 自动接受可能绕过 M04 确认通道 | 设计规定 AUTO_ACCEPT **不等于** confirmed，只表示免人工复核进入 M04 整合；仍需 M04 通道写 confirmed。**是否允许任何形式的自动确认，须重新裁决**（当前默认：AUTO_ACCEPT 仅可用于 L1/L2 且 visibility=owner_private，且不写 confirmed） |
| **CR-02** | 情绪检索要求 vs "首版关闭查询侧情绪感知" | 任务书 §10.6/§14 全章 ↔ 台账 J6 | EMOTION_QUERY 与 getEmotionContext 与 J6 直接冲突 | 设计**完整定义**能力；实施默认 `emotion.query_enabled=false`（遵 J6），启用须重新裁决 |
| **CR-03** | ΔE 影响当前状态 vs "禁止直接修改 PAD、持久变更走独立审核（默认关闭）" | 任务书 §10.5 ↔ 台账 X-3、CR-09、K-6 | 情绪影响可能绕过审核 | 设计定义公式与上限；`emotion.influence_enabled=false` 默认关闭，启用须重新裁决；ΔE 仅作用运行时状态、不落库、不由沉默触发 |
| **CR-04** | 情绪标注由 L1/L2 LLM 生成 vs "只预留接口不实现 LLM 标注" | 任务书 §6.2/§7.12 ↔ 台账 J7 | 摄入期情绪抽取与 J7 冲突 | 设计定义 EmotionExtractor；实施默认只接受 USER_EXPLICIT 情绪，`emotion.auto_infer=false`；自动推断须重新裁决 |
| **CR-05** | Psychology 纳入正式架构 vs psy-model 暂缓 | 任务书 §3.1/§19 P6 ↔ 台账 I-7/X-6 | 架构图含 Psychology 可能被误读为解锁 | 设计允许（设计≠实施）；Psychology 实施仍 BLOCKED，接口按 DEFERRED 定义 |
| **CR-06** | 证据原句长期保存 vs 数据最小化/导出边界 | 任务书 §8.3 ↔ 台账 K-2、D-AP-09 | 原句保存范围与导出边界未定 | Evidence 设 retention_policy；导出边界仍 BLOCKED（K-2）；**不得声称"记忆文本不出本机"**（D-AP-09） |
| **CR-07** | `public` 可见性 vs 最小权限 | 任务书 §15.2 ↔ 台账原则 | public 可能被误启用 | 定义但 V1 禁用（`visibility.public_enabled=false`），启用须裁决 |

> 上述 7 项一律写入 `OPEN-ISSUES.md` 与 `PENDING-USER-DECISIONS.md`（编号 OI-025~OI-031），并在各文档"正式裁决引用/未决事项"节交叉标注。

---

## 13. 写作硬规则（补充 v1）

1. **术语统一（任务书 §20.3）**：统一使用 Memory Unit / Evidence / Event / EmotionContext / Memory Anchor / Lifecycle(tier) / Semantic Type / Subject / Memory Ingestion Pipeline / Memory Read API / Memory Write API / Psychology。**同一概念禁止两种定义**。
2. **禁止 Future 化**：§1 的 17 项能力 MUST NOT 标为可选或无限期 Future；只能标"分阶段实施（P3–P6，需阶段授权）"。
3. **参数**：全部进 15 号参数注册表（15 属性），业务代码禁止硬编码（台账 优化-5）。
4. **Token 单位**（任务书 §13.7）：字符/汉字/字数/Token MUST 分别标明；画像、事件、证据、情绪分别计入预算；禁止"1200 字=1200 Token"。
5. **Mock 纪律**：Mock 结果 MUST 标 MOCKED；测试桩全通过逻辑仅限隔离测试，禁用于真实私人数据的正式群聊注入（任务书 §15.4）。
6. **不删旧依据**（任务书 §20.5）：修订保留 v1 语义映射（§4），在"变更记录"写明。
7. **禁止行为**：不改台账、不改现网 qq-bridge/配置/state/Persona、不连真实机器人、不启正式服务、不把建议写成历史批准、不把 Mock 写成真实验收。
