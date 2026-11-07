# C-REPORT — EmotionContext 与 Memory Core 联动（目标三）· 过程报告

## 0. Scope / 读了什么 / 没做什么

**轮次**：MC-CORE-CAPABILITY-REVIEW-01 · C 路（目标三）· 只读能力审查 · 2026-11-06。
**判断标准**：产品目标 7 条（事件只存事实 / 情绪独立附件 / 可关联事件·人物·互动 / 占比小不破坏事实 / 回忆可回取情绪线索 / 历史情绪有限影响 / 不凌驾事实·当前上下文·用户真实表达）。
**交付件**：`E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\03-EMOTION-MEMORY-INTEGRATION.md`（以 `# 标题` + `## STATUS: COMPLETE` 开头，独立可读）。
**原始证据**：`EVIDENCE/c/c-machine-checks.py`（只读校验脚本）、`EVIDENCE/c/c-machine-checks.json`（UTF-8 无 BOM 输出）。

**读了**（A 集合基线 `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design`）：
- `13-Evidence与EmotionContext.md` **全文 1034 行**（分 5 块 read），是本路主证据；
- `contracts/schemas/`：`emotion-context`（153 行，重点）、`evidence`、`event`、`memory-unit`、`person-profile`、`search-document`、`read-request`、`read-response`（后者用 `utf-8-sig` 读，未改基线）；
- `contracts/config/`：`emotion`（151 行，重点）、`retrieval`、`profile`；
- `00`（L40–69 / L398–411 + grep 24 命中）、`01`（grep 28 + §7.16 表 1–13 L664–687）、`02` 清洗副本 `EVIDENCE/B/02-clean.md`（L1–60 + grep 16 命中）、`09`（grep 43 + §7.15/§8.5）、`11`（grep 74）、`12`（grep 28）、`14`（grep 71 + §7.2.7 L439–483）、`15`（grep 28）、`V2`（grep 40 + §3.4 L240–269 / §8-§9 L368–401）、`08`（grep 42）、`03`（grep 2 命中）；
- `contracts/state-machines/state-machines.md` **全文 165 行**（SM-1~SM-8）+ grep（情绪 0 命中）；
- 导航材料：`E:\[PROJECT]\AUDIT-FORENSIC-2026\09-CONFLICT-REGISTER.md` **仅 grep**（Emotion/情绪/Psychology 共 3 命中，读 L20–39；FC-001 L30 的 "emotion scope" 属 A/B 差异登记，不构成本路新裁决）；`_shared/STANDARDS.md` 全文；`04-PERSONA-PROMPT-FLOW.md` 头部（确认 D 路仍 IN PROGRESS，不代其判断）。

**没做**（明确边界）：不重审 49 个 FC；不重做 A 路（目标一）/ B 路（目标二）结论；不逐条读 conflict register 全文；不争讼 A 集合 APPROVED 依据缺口、B 集合去留、CMB / R3 定位；不写 V0.9 产品代码；不改任何 schema / DDL / 裁决台账；不向用户提问。

**规则遵守**：写文件只用 `write` / `edit`（未用 PowerShell `Set-Content`/`Out-File`/`Add-Content`）；正文简体中文，字段名/枚举/模块名/章节号/算法 ID 保留原文（`ASSISTANT_INFERRED` 按语料原名，未自造 `LLM_INFERRED`；题面 `SYSTEM` 缺项显式对齐）；每条结论三分类；优先级只 P0/P1/P2；P0/P1 每条九问实答；未把 `NOT RUN` / 未标定当架构不可行，也未把「有 schema/功能名」当算法完整。

---

## 1. Headline（3 行）

1. **字段与防护最强、全链路最弱**：`EmotionContext` 14 必填字段 / 15 受控标签 / SEP-1~8 / ΔE=I×C×R×D + 8 门禁是 A 集合最完整的情绪设计；「情绪不得污染 Event 事实」防护最严密且成立。
2. **「回取情绪线索」与「历史情绪有限影响」在机器契约下 100% 不成立**：三开关 `const:false`（15 号注册表标「无人可写」）+ `read-request.include_emotion const:false` + `read-response.emotion_context.maxItems:0` + `getEmotionContext` P5 默认关闭 ⇒ 情绪写得进草案、读不出来、影响不了。
3. **成熟度：schema `MOSTLY_COMPLETE` / 全链路 `PARTIAL`（契约态 BLOCKED、设计态可救）**；缺口集中在写入链 W4–W6、关联模型、读取合并、Persona 落点、`R` 算法与参数一致性；多数可在 V0.9 实验原型验证，但「解锁三开关」必须走用户重新裁决。

**capability_verdict：PARTIAL**（全链路）。理由：产品 7 条中 G1/G2/G7 成立，G4 过度满足（零占比），G3 部分成立，G5/G6/G8 不成立或严重失真。

---

## 2. Findings（交付件 §15 的 9 条，均含 EXISTING DESIGN / DESIGN COMPLETENESS / RECOMMENDED ADDITION + P0/P1 + cite）

| ID | P | 标题 | 核心 existing | 核心 missing | 阻断 V0.9 |
|---|---|---|---|---|---|
| **C-C001** | **P0** | 情绪回取与有限影响被 `const:false` + `maxItems:0` 硬封死 | `config/emotion L19–36` 三开关 const:false；`read-request L114–117`；`read-response L309–318`；`14 §7.2.7` P5 默认关闭；`00 §7.7 L405` Psychology DEFERRED | 可达读取路径、可启用影响路径、Psychology 状态载体、Context Builder emotion 分区 | **是（能力层）** |
| **C-C002** | **P0** | 写入链 W4–W6 全缺 | W1–W3/W8 完整（`09 §7.15` 阶段 11 `EmotionExtractor` 8 条 MUST；失败不阻塞事实 `L591`） | 草案→对象映射（`09 §14 OI-09-06 VERIFY`，缺 7 必填）、关联表、`emotion_context` 表 DDL（`01 §7.16` 表 1–13 无该表） | **是** |
| **C-C003** | P1 | 关联模型不可执行：无类型化指针/人物/互动/时间序列；SEP-2 与契约矛盾 | `event_id`/`related_memory_ids`/`emotion_refs` 三数组；`13 §7.7.3 L465` 双情绪示例 | `subject_ref_type`/`related_person_ids`/`interaction_id`/`version`/`status`/反查算法；`schemas/event` 无 `emotion_refs` ⇒ `CONFLICTED` | 否（最小链路可跑） |
| **C-C004** | P1 | ΔE 参数三处冲突、`R` `MISSING`、`Δt` 与多事件集合未定义 | ΔE + 8 门禁 COMPLETE；两组逐步算例 | `R`（`13 §7.9.1 L522` PROPOSED / `OI-13-03 TBD`）、`Δt` 时间源、集合选取；默认值 `13`=0.30/0.60/0.20/0.05 vs `config`/`15`=0.6/0.5/0.15/0.1 | 否（但污染实验结论） |
| **C-C005** | P1 | 情绪→Persona/Psychology 落点缺失，无 emotion 分区与合并算法 | `00 §0.4 L46`/`§7.7 L405–408`；`person-profile L55` 禁情绪入画像；`02-clean §7.7` 仅 `uncertain` 分区 | 字段/接口/组装步骤/Prompt 段名/预算占比；四模块无接口契约 | 否 |
| **C-C006** | P1 | 数据完整性边界矛盾与元数据缺失 | 14 必填字段、`allOf` 推断 ≤0.99、`not` 禁 public | `intensity`/`confidence` 必填无 null vs `13 §7.8.2 L495–496`「未知留空」`CONFLICTED`；`source_type` 缺 `SYSTEM`；来源元数据不可审计 | 否 |
| **C-C007** | P1 | `MemoryUnit.content` 可合法承载情绪并进 fts/vector RRF 共池（唯一事实污染路径） | `schemas/memory-unit L54/L102` 含 `EMOTION_CONTEXT` 但 L46–50 `content` **无情绪禁令**；`search-document L16` 无隔离 | 契约级禁令、`items[]` 过滤规则、隔离断言 | 否（但实验必须加断言） |
| **C-C008** | P1 | 附件无法独立关闭/调整/拔除 | `config/emotion` 三全局 const:false；`15 §7.8 L243–245`「无人可写」；J5 清除仅 `13 §5 L141` 一句原则 | 对象级 `enabled`/`detached`/`status`、清除 API、SM-9、K-2 边界清单 | 否 |
| **C-C009** | P1 | 读取侧编排 `CONCEPT_ONLY` | `14 §7.2.7 L439–471` 接口形态完整（输入/输出/权限/错误码） | 反查/排序/去重/合并算法、`emotion_tokens` 上限、意图判定规则 | 否（C-C001 解锁后成首要阻断） |

**P2（归档，未作主结论）**：C-C010（越界映射 `neutral` 丢原语义，`raw_label` 已缓解）、C-C011（词表 `OI-009 TBD`、`raw_label>128` 拆分算法缺）。

---

## 3. 关键矩阵（交付件 §3/§5/§6/§10/§13 的浓缩）

### 3.1 特别判断 7 项（题面要求逐项表态）

| 特别判断项 | 结论 | 证据 |
|---|---|---|
| subject 指针 | **部分有、类型化指针缺** | `subject_id`（所属主体）、`event_id`、`related_memory_ids` 有；无 `subject_ref_type`/`related_person_ids`/`interaction_id`（机器校验 `subject_pointer_typed=false`） |
| `source_type` | **有 2 值 `USER_EXPLICIT`/`ASSISTANT_INFERRED`，缺 `SYSTEM`**；题面 `LLM_INFERRED` 的语料原名是 `ASSISTANT_INFERRED` | `schemas/emotion-context L86–90`；L6 `$comment` 把 `EXTERNAL_SOURCE`/`SYSTEM_GENERATED` 列 MUST 被拒；`13 §7.7.3 SEP-4` 仅 2 值 |
| `confidence` | **有**（0–1 必填，`allOf` 推断 ≤0.99），但与「未知留空」`CONFLICTED` | L75–80 / L118–128；`13 §7.8.2 L496` |
| `intensity` | **有**（0–1 必填，禁百分制），同上有必填矛盾、无标定口径 | L69–74 |
| decay/半衰期 | **schema 内缺**（仅 `config/emotion L44–49` 全局 `emotion.decay_rate` + `13 §7.9.1 L523` 的 `D`） | `decay_field=false` |
| 时间戳序列 | **缺**（只有 `created_at`，无 `observed_at`/`version`/`status`/`history`） | L112–116；`timestamp_series=false` |
| 可开关 runtime 标记 | **缺**（无 `enabled`/`detached`/`status`；开关是全局三项 const:false） | `runtime_flag=false` |

### 3.2 情绪是否可能写回 Event 正文 / memory 内容（题面要求给证据）

- **Event 侧：不会，且防护最严密**。`schemas/event L33–38 content.description`「MUST NOT 写入情绪判断、感受评价、意图猜测或模型推断」；`$comment` 错误示范 (1)「主人今天很难过，因为项目延期」、(6) 用 `related_memory_ids` 指向 `event_id`；`schemas/emotion-context L5`「MUST NOT 反写 `Event.content`」；`13 §7.7.3 SEP-1/SEP-7`、`E-13-08`（写情绪入 content ⇒ `VALIDATION_ERROR`，L869）；`config/emotion L130–134 gates.writes_persistent_state=false`（const 红线）；门禁 8 / X-1 / X-3 / I-5。
- **MemoryUnit 侧：会，存在唯一合法污染路径（C-C007）**。`schemas/memory-unit L54` `semantic_type` 6 值含 `EMOTION_CONTEXT`、L102 `anchor` 含 `EMOTION_CONTEXT`，但 L46–50 `content` **无情绪禁令**（对比 `event.schema.json L37` 有）；`schemas/search-document L16` 检索投影亦无隔离；SEP-2 只禁「EmotionContext 作为 **Event** 内嵌字段」。若把情绪写成 `semantic_type=EMOTION_CONTEXT` 的 `MemoryUnit.content`，会走 FTS5+LanceDB→RRF→`items[]` 与事实共池。门禁 8 只管 ΔE，管不到这条。

### 3.3 情绪检索是否与事实检索共用 RRF / 同一排序池

**独立，不共用，不会压过事实；代价是回取被切断、启用后合并机制完全缺失。**
- `config/retrieval L105–158` RRF 通道仅 `fts`/`vector`；`structured 只过滤不参与计分`(OI-022)；精排五特征无 emotion 权重（机器校验 `emotion_weight_in_reranking=false`）。
- X-1「不改变 M02 最终排序」；`02-clean L481/L1104/L1463` 禁心理加成进排序。
- `read-response` 中 `items[]`（事实）与 `emotion_context[]`（情绪）是**并列不同字段**，`emotion_tokens` 独立，`trace.excluded` 有 `EMOTION_DISABLED` 专用原因。
- **合并机制 MISSING**：`13 §7.12.3` 三种主动请求 PROPOSED 且均关闭；`14 §7.2.7` 未定义 `emotion_contexts[]` 与 `items[]` 的先后/段名/占比/预算；`02 §7.7 ALG-06` 无 emotion 分区。详见交付件 §12 / C-C009。

### 3.4 完整链路状态（交付件 §5）

| 步 | 环节 · 模块 | 判定 |
|---|---|---|
| 1 输入 | `EvidenceBinder`（阶段 10） | COMPLETE |
| 2 提取 | **`EmotionExtractor`（阶段 11，M03，`09 §7.15 L597`）** | MOSTLY_COMPLETE（`EmotionDraft` 仅 7 字段） |
| 3 草案→对象 | **无明确模块** | **MISSING**（W4 缺 7 必填字段映射，`OI-09-06 VERIFY`） |
| 4 关联 | **无** | **CONFLICTED**（`schemas/event` 无 `emotion_refs`；无关联表/索引/反查） |
| 5 保存 | M01（推定） | **MISSING**（`01 §7.16` 表 1–13 无 `emotion_context` 表） |
| 6 检索 | M02 / Intent Resolver / Anchor Router | 设计定义、契约封死（`include_emotion const:false`、配额 0） |
| 7 回取 | `getEmotionContext`（P5 默认关闭） | **BLOCKED**（`maxItems:0`） |
| 8 当前交互 | Psychology（BLOCKED）/ Persona（FUTURE） | **BLOCKED**（无落点/无 emotion 分区/无合并算法） |

### 3.5 参数默认值冲突（`CONFLICTED`，交付件 §10.2）

| 参数 ID | `13 §7.13.2` | `config/emotion` | `15 §7.8` |
|---|---|---|---|
| `emotion.relevance_threshold` | 0.30 | 0.6 | 0.6 |
| `emotion.inferred_weight` | 0.60 | 0.5 | 0.5 |
| `emotion.max_influence` | 0.20 | 0.15 | 0.15 |
| `emotion.decay_rate` | 0.05 | 0.1 | 0.1 |

后果：`13 §7.9.2` 用 0.20 得 `ΔE=0.20`、`§7.9.3` 对照 `0.020082`，`§13 AC-13-10` 要求算例一致；按 config/15 的 0.15 则验收 FAIL。`15 §7.8 L527` 又声明「CR-03 未通过前不得标定」⇒ 未裁决的契约冲突，非待调参。

### 3.6 13 问判定一览（交付件 §14 逐条实答）

| 问 | 结论 | 问 | 结论 |
|---|---|---|---|
| 1 数据结构充分？ | 对象契约 MOSTLY_COMPLETE，作为「可关联/可演化/可治理附件」不充分 | 8 强度 | 有 0–1 必填，无标定口径/对象级衰减，`CONFLICTED` |
| 2 通过什么 ID 关联 | `event_id`+`related_memory_ids`+`emotion_refs`；单向无类型，SEP-2 与契约矛盾 | 9 控制对长期记忆影响 | 硬隔离占比 0（防护严密），但缺可控比例 + C-C007 未封堵 |
| 3 可关联几条 | 数组无上界 ⇒ 可多条但无基数/排序/去重/写入者 | 10 控制对当前状态影响 | 公式 COMPLETE，落地 BLOCKED（开关/Psychology/`R`/`Δt`/参数） |
| 4 多次如何保存 | 独立 `emotion_id` 并列快照，无 thread/version/历史版本表 | 11 避免一次负面情绪影响人格/关系 | 多层防护成立，但靠通道不存在；解锁后无可验证判据 |
| 5 同事件不同时间变化 | **不能表示**（`temporal_position` 是阶段分类非时间轴） | 12 独立关闭/调整/拔除 | **不成立**（全局 const:false ≠ 可治理；J5 清除 CONCEPT_ONLY） |
| 6 来源用户明示 vs 推断 | 分离纪律完整（最强项之一），但推断侧默认不产出；缺 `SYSTEM` | 13 Runtime 未启用是否正常 | **是，COMPLETE**（`09 §7.15 L591` 失败不阻塞事实；关闭态可独立降级） |
| 7 置信度 | 有 0–1 + 推断折扣，但必填无 null 矛盾、不可审计 | | |

### 3.7 缺失算法清单评级（交付件 §13，A1–A20）

`MOSTLY_COMPLETE`：A1 对象契约、A2 越界映射、A3 草案提取、A20 测试规格（NOT RUN）· `COMPLETE`：A8 ΔE 公式、A18 Event 侧污染防护、A19 关闭态降级 · `PARTIAL`：A7 来源校准、A12 衰减 · `CONFLICTED`：A9 参数一致性 · `CONCEPT_ONLY`：A15 Persona/Psychology 落点、A16 清除/拔除 · `MISSING`：A4 草案落库、A5 关联、A6 时间序列、A10 `R`、A11 集合选取、A13 回取、A14 合并、A17 情绪状态机 · `BLOCKED→MISSING`：A13 回取（契约态）。

---

## 4. 要使全链路可执行还缺什么（交付件 §16 的浓缩）

1. **契约解锁（用户裁决，实验不可自行解决）**：U1 CR-02/OI-026、U2 CR-03/OI-027（+X-3 审核链 + I-4 影子验证）、U3 CR-04/OI-028（+κ≥0.6）、U4 CR-05/OI-029 Psychology 阶段授权、U5 CR-08B Persona D1–D3、U6 K-2/CR-06/OI-030 删除导出边界、U7 参数真源裁决、U8 `CONTRACT_CHANGE`+`SCHEMA_MIGRATION` 变更流程。
2. **写入链补位**：阶段 11.5 `EmotionBinder`（M03 产出→M01 事务）、7 个缺失必填字段生成规则、`emotion_context` 表 DDL、`emotion_links` 关联表（带 `target_type`/`INDEX`/FK/同事务/tombstone）。
3. **对象字段补位**（`NEW PROPOSAL`）：`subject_ref_type`+`target_refs`、`related_person_ids`、`interaction_id`、`observed_at`/`recorded_at`、`emotion_thread_id`/`version`/`status`/`supersedes`、对象级 `decay_rate`、`enabled`/`detached`、`value_state`、`origin_channel`、`model_version`/`extraction_method`/`annotator`/`calibration_version`。
4. **算法补位**：A10 `R`（token 重叠基线 / cosine 升级）、A11 集合选取、A12 `Δt=now−observed_at`、A13 六步 `EmotionRetrieval`、A14 四分区合并（事实→证据→画像→情绪最后，情绪不挤占事实预算）、A16 清除/拔除、SM-9 情绪状态机、A9 参数真源统一并重算 `13 §7.9` 算例/AC-13-10。
5. **封堵 C-C007**：`schemas/memory-unit content.description` 增情绪禁令、`search-document indexable=false`、`read-response.items[]` 排除 `EMOTION_CONTEXT` 条目 + 断言测试。
6. **Persona 落点**：`Context Builder` emotion 分区 + `emotion.max_context_ratio`（EXP-DEFAULT 0.10）；`Psychology` 只读 `RuntimeAffectSnapshot`；`Context Adapter` 注入并回传 L3（不触发 reinforce）。

---

## 5. Recommended design（`NEW PROPOSAL` 摘要，详见交付件 §16）

> **未获批准，不得写成已有设计。** 四层：①最小 schema 增补（§16.1）；②算法补位 A4/A5/A9–A14/A16（§16.2）；③写入 `bindEmotion` 与读取 `getEmotionContext` 两段伪代码（§16.3，含同一 M01 事务、M05 单独判权、来源/阶段/强度标记、`emotion.max_attached` 与 `max_context_ratio` 裁剪、不调用 LLM 现编情绪、不由沉默触发、不写 reinforce）；④解锁前置 U1–U8（§16.4）。ASCII 数据流图见交付件 §17（Event / EmotionContext / Evidence / PersonProfile / Psychology / Current Interaction / Emotion Retrieval / Persona Context 全覆盖，`[NEW]` 与契约封死路径分别标注）。

**哪些能在 V0.9 实验内解决**：§16.1 字段原型、§16.2 全部算法原型、§16.3 两段伪代码、C-C007 隔离断言、T-13-48~52 / AC-13-15 补测、关闭态回归测试、参数一致性对照报告（**只产出证据，不改基线契约**）。
**哪些不能**：三开关启用、Psychology 阶段授权、J6 独立验证、Persona D1–D3、K-2 删除边界、参数真源正式裁决、`contracts/` 正式修改 —— 全部交父 Agent / 用户。

---

## 6. Cross-scope handoffs（交付件 §19，最多 6 条）

| # | 对象 | 交接 |
|---|---|---|
| H1 | 父 Agent / Lead | C-C001 能力层 P0；三开关 `const:false`+`maxItems:0` 使情绪回取/有限影响归零；启用须 U1–U8 用户重新裁决，**V0.9 实验不得自行解锁** |
| H2 | 父 Agent / 变更流程 | C-C004 参数三处冲突需裁决唯一真源并重算 `13 §7.9` 算例 / AC-13-10，否则 V0.9 算例验收不可采信 |
| H3 | D 路（Persona Prompt Flow） | 确证 `schemas/person-profile L55` 禁情绪入画像、`Context Builder`(ALG-06) 无 emotion 分区、`08 §7.1 L229` 链路无 Psychology 节点、Persona/Context Adapter FUTURE、Psychology BLOCKED ⇒ 可作「情绪层未进入 per-turn 组装」既定事实 |
| H4 | B 路 | `related_memory_ids` 无类型 ID 数组、`getEmotionContext` 手动 1-hop 无深度/预算/visited 规范，独立印证 B 路「跨对象关系无类型、无联合召回」；建议 `emotion_links` 带 `target_type` |
| H5 | A 路 / M01 | `01 §7.16` 表 1–13 无 `emotion_context` 表；情绪对象无 `version`/`status`/保留策略，与 A 路「无分层体系」同源；本路 DDL 属 `SCHEMA_MIGRATION`，需与 A 路事务/Outbox/tombstone 纪律对齐 |
| H6 | V0.9 实验 | 可自行原型验证的清单（§5 上半），产出证据但不改基线契约 |

---

## 7. Method appendix

### 7.1 命令与证据

| 命令 | 结果 |
|---|---|
| `pwsh`：`$env:PYTHONPATH='…\evidence\second-round\pydeps'; python EVIDENCE\c\c-machine-checks.py` | 生成 `EVIDENCE/c/c-machine-checks.json`（6560 bytes）；`jsonschema_self_check.valid=true`（4.26.0） |
| `pwsh`：`[System.IO.File]::ReadAllBytes(...)` + `python -c "json.load(open(...))"` | 首字节 `123,10,32`（`{`/LF/空格）⇒ **UTF-8 无 BOM**；JSON 可解析；`parameter_default_conflict.verdict=CONFLICTED` |

脚本只读扫描基线（不修改任何基线文件），输出 10 个键：`emotion_context_schema`（14 必填 / 14 properties / `field_presence_checks` 逐项 false-true）、`emotion_config`（三开关 `const:false`、四数值默认、gates）、`read_contracts`（`include_emotion const:false`、`emotion_context.maxItems=0`、`EMOTION_DISABED` 枚举、`emotion_tokens`）、`linkage_contracts`（`event_has_emotion_refs=false`）、`retrieval_config`（RRF 仅 fts/vector、无 emotion 权重）、`parameter_default_conflict`（三处值）、`state_machines`（SM-1~8、`emotion_related_sm=false`）、`emotion_draft_gap`（草案缺 7 必填、`OI-09-06`）、`ddl`（`has_emotion_context_table=false`）、`jsonschema_self_check`（Draft7 自检）。

### 7.2 关键人工判读（非机器可得）

- `13 §7.7.3 SEP-2 L453` 与 `schemas/event` 属性清单的 `emotion_refs` 矛盾（人工对照 15 个 properties 确认）；
- `schemas/memory-unit L54/L102` 含 `EMOTION_CONTEXT` 而 L46–50 `content` 无禁令 ⇒ C-C007 的唯一合法污染路径（由「Event 有禁令、MemoryUnit 没有」的对照得出）；
- `13 §7.9.2/§7.9.3` 算例数值与 `config`/`15` 默认值不一致 ⇒ 验收判据连锁失效（人工重算）；
- `09 §7.15 L591`「情绪抽取失败 MUST NOT 阻塞事实单元提交」⇒ 13 问 13「Runtime 未启用时 Memory Core 仍正常运行」为 **COMPLETE**，是本路最强项之一；
- `13 §7.12.3` 三种主动请求自标 PROPOSED + L669 禁止推测意图，与「关联触发」场景自相牵制。

### 7.3 自检与判断依据记录

- **三分类**：全文每条结论带 `EXISTING DESIGN`（文件+章节）/ `DESIGN COMPLETENESS` / `RECOMMENDED ADDITION`（`NEW PROPOSAL`）；未写「已批准设计」。
- **九问**：交付件 §15 的 9 个 P0/P1 问题逐条 ①~⑨ 实答（含「需补/改哪些文档」「是否阻断 V0.9」「能否在 V0.9 实验中解决」），无「只报问题不给方案」条目。
- **13 问**：交付件 §14 编号 1–13 逐条实答，未合并省略。
- **10 项必备内容**：①数据完整性矩阵(§6) ②关联模型(§7) ③写入流程(§8) ④读取流程(§9) ⑤衰减与影响(§10) ⑥与 Persona(§11) ⑦与 Retrieval(§12) ⑧缺失算法清单(§13) ⑨NEW PROPOSAL 含最小 schema 增补与写入/读取伪代码(§16) ⑩ASCII 数据流图(§17) —— 全部实答。
- **优先级**：P0 = C-C001 / C-C002（2 条）；P1 = C-C003…C-C009（7 条）；P2 = C-C010 / C-C011（仅末表归档，未作主结论）。
- **成熟度表态**：明确区分「schema `MOSTLY_COMPLETE`」与「全链路 `PARTIAL`（契约态 BLOCKED）」，并说明为何不判全链路 `BLOCKED`（规则 7：`NOT RUN`/未启用 ≠ 不可实现）也不判 `MOSTLY_COMPLETE`（产品 4 条目标不成立）。
- **篇幅判断依据（不向用户提问，自行决定）**：交付件约 **92.8 KB**，超出「目标 25–45 KB」建议值。原因是硬性规则 9 同时要求「13 问逐条不合并」「每个核心问题九问实答」「要求的每一张表都必须实答」「10 项必备内容」，且规则 6 规定「只报问题不给方案的条目一律不合格」。本路选择**优先保证必答项完整、不删任何表/问题/方案**，未做注水或重复论证；已压缩两轮（初稿 120.5 KB → 92.8 KB），并在 §0/§20 记录该判断。若父 Agent 要求严格 ≤45 KB，建议以「§15 九问改 9 行紧凑表格 + §14 引用串精简」再压一轮，但不建议删除任何必答项。
- **未改基线**：所有写入仅落在 `E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\` 下（交付件、本报告、`EVIDENCE/c/` 两个文件）；`read-response.schema.json` 的 BOM 仅在读取时用 `utf-8-sig` 兼容，未写回；`02` 的 NUL 通过既有清洗副本 `EVIDENCE/B/02-clean.md` 规避，未新建重复副本。

---

**结论**：C 路完成。交付件 `03-EMOTION-MEMORY-INTEGRATION.md` 与本过程报告、机器证据三者一致，摘要按给定 JSON schema 返回。
