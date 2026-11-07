# 03 — EmotionContext 与 Memory Core 联动（目标三）

## STATUS: COMPLETE

> MC-CORE-CAPABILITY-REVIEW-01 · C 路 · 只读能力审查 · 2026-11-06。
> **判断标准（产品目标，非现状描述）**：事件只保存事实不夹带情绪判断；情绪作为独立附件存在；情绪可关联到具体事件/人物/互动经历；情绪在原始事件中占比小、不得破坏事实记忆；回忆时能重新取回该事件的情绪线索；历史情绪对当前互动只有有限影响；情绪不得凌驾于事实、当前上下文和用户真实表达之上。
> 三分类贯穿全文：**EXISTING DESIGN**（已有正式设计，给 文件+章节）/ **DESIGN COMPLETENESS**（输入·输出·步骤·条件·模块责任·边界是否足以执行）/ **RECOMMENDED ADDITION**（一律标 `NEW PROPOSAL`，未获批准）。
> 基线 A 集合 APPROVED 依据存在证据缺口（前序已定，不重讼）；不争讼 B 集合 / CMB / R3；不重做 A 路、B 路结论。文中 `13` = `13-Evidence与EmotionContext.md`，`09` = `09-记忆摄取Pipeline.md`，`14` = `14-ChatLLM读写API.md`，`11` = `11-MemoryAnchor与语义分类.md`，`15` = `15-参数注册表.md`，`00/01/02/08/12` 同名编号文档；`config/*` = `contracts/config/`，`schemas/*` = `contracts/schemas/`，`V2` = `DD-AUTHORITY-V2.md`。

---

## 0. 范围与判断依据

**读了**：`13`（1034 行全文）；`schemas/emotion-context`（153 行）、`evidence`、`event`、`memory-unit`、`person-profile`、`search-document`、`read-request`、`read-response`；`config/emotion`、`retrieval`、`profile`；`00`（L40–69/398–411）、`01`（§7.16 表 1–13）、`02` 清洗副本 `EVIDENCE/B/02-clean.md`、`09`（§7.15/§8.5/§7.22/OI-09-06）、`11`（§7.7/§7.15/§7.19）、`12`、`14`（§7.2.7）、`15`（§7.8）、`V2`（§3.4/§8/§9/§12）、`08`（§7.1/§7.11）、`03` 头部、`contracts/state-machines/state-machines.md`（SM-1~8 全文）；`E:\[PROJECT]\AUDIT-FORENSIC-2026\09-CONFLICT-REGISTER.md` 仅 grep 情绪（3 命中，FC-001 L30 "emotion scope" 属 A/B 差异登记，不新裁决）；`_shared/STANDARDS.md`、`04-PERSONA-PROMPT-FLOW.md` 头部。机器校验见 `EVIDENCE/c/c-machine-checks.py` → `c-machine-checks.json`。

**没做**：不重审 49 个 FC；不重做 A/B 路；不写 V0.9 产品代码；不改任何 schema/DDL/裁决台账；不把「尚未实测/未调参」当「架构不可行」，也不把「有一个 schema/功能名」当「算法完整」。

**判断依据（自记录，不提问）**：①设计文档说「完整定义能力」而 `contracts/` 用 `const:false` / `maxItems:0` 封死路径 ⇒ 判「当前不可执行」（按 `STANDARDS.md §2` 反用）；②「全局永久关闭」≠「可独立开关」，「零影响」≠「有限影响」；③区分 schema 成熟度与全链路成熟度；④`EXP-DEFAULT` 未标定不判不可行，但三处默认值互矛盾属 `CONFLICTED`（设计缺陷，非调参）。

---

## 1. 三行结论

1. **字段与防护最强、全链路最弱**：`EmotionContext` 14 必填字段、15 受控标签、SEP-1~8 分离硬规则、ΔE=I×C×R×D + 8 门禁是 A 集合最完整的情绪设计；「情绪不得污染 Event 事实」被反复禁止且防护最严密。
2. **「回取情绪线索」与「历史情绪有限影响」在机器契约下 100% 不成立**：`emotion.query_enabled` / `influence_enabled` / `auto_infer` 三开关 `const:false` 且 `15 §7.8` 标「无人可写」，`read-request.include_emotion const:false`，`read-response.emotion_context.maxItems:0`，`getEmotionContext` 属 P5 默认关闭 —— 情绪写得进草案、读不出来、影响不了。
3. **成熟度：schema `MOSTLY_COMPLETE` / 全链路 `PARTIAL`（契约态 BLOCKED、设计态可救）**；缺口集中在关联模型、时间序列、写入者/存储表、读取合并、Persona 落点、`R` 算法与参数一致性，多数可在 V0.9 实验补齐，但「解锁三开关」必须走用户重新裁决，实验不能自行解决。

---

## 2. 现状成熟度评级（特别判断）

| 维度 | 评级 | 依据 |
|---|---|---|
| **schema 成熟度**（`schemas/emotion-context`） | **MOSTLY_COMPLETE** | Draft7 自检合法（jsonschema 4.26.0）；14 必填字段；15 枚举；`intensity`/`confidence` 0–1；`temporal_position` 4 值；`source_type` 2 值；`allOf` 推断 `confidence≤0.99`；`not` 禁 `public`。仅局部边界缺口（§6） |
| 写入侧 | **PARTIAL** | 阶段 11 `EmotionExtractor` 8 条 MUST + `EmotionDraft`；但草案缺 7 必填字段、映射自登记 `OI-09-06 VERIFY`；`01 §7.16` 表 1–13 无 `emotion_context` 表 |
| 关联模型 | **PARTIAL / CONFLICTED** | 三处数组；SEP-2 声称的 Event 侧 `emotion_refs` 契约中不存在；无类型化指针、无人物/互动、无时间序列 |
| 读取侧 | **BLOCKED（契约态）** | `read-response.emotion_context.maxItems:0`、`include_emotion const:false`、`query_enabled const:false`、`getEmotionContext` P5 默认关闭（CR-02/OI-026） |
| 影响侧 | **PARTIAL（公式）/ BLOCKED（落点）** | ΔE + 8 门禁 COMPLETE；但 `influence_enabled const:false`、Psychology `BLOCKED/DEFERRED`（CR-05/OI-029）、`R` `OI-13-03 TBD`、参数三处冲突 |
| **全链路综合（本路结论）** | **PARTIAL** | 输入→提取→对象→关联有定义；保存→检索→回取→当前交互断链。**是全链路评级** |

**为何不全链路 `BLOCKED`**：规则 7 + `STANDARDS.md §5`，公式、字段、门禁、接口形态、47 项测试规格、14 条验收均已定义，`NOT RUN` ≠ 不可实现，缺的是契约解锁与增量补位。但**必须同时写明：在当前已冻结机器契约下全链路不可执行**，V0.9 不先改 `contracts/` 就观测不到情绪回取或 ΔE。**为何不 `MOSTLY_COMPLETE`**：产品 7 条中「回取」「有限影响」「关联人物/互动」「独立开关/拔除」四条不成立或严重失真（§4）。

---

## 3. 特别判断：`schemas/emotion-context` 到底有没有（缺哪个写缺哪个，引用行号/字段名）

> 机器校验见 `EVIDENCE/c/c-machine-checks.json → emotion_context_schema.field_presence_checks`；`properties` 共 14 项。

| # | 判断项 | 有/缺 | 证据（文件+行号/字段名） | 后果 |
|---|---|---|---|---|
| 1 | **subject 指针**（事件/记忆/人物/互动） | **部分有、类型化指针缺** | 有 `subject_id`（L37–42，语义是「情绪**所属主体**稳定 ID，须经 EntityResolver」，非关联目标）、`event_id`（L32–36，`uuid\|null`，MUST=`Event.event_id`）、`related_memory_ids`（L96–99，`memory_id` 数组）。**缺** `subject_ref_type`/`target_type`/`related_person_ids`/`interaction_id`/`session_id`（`subject_pointer_typed=false`） | 无法表达「对某人的情绪」「某次互动的经历情绪」；`related_memory_ids` 无类型 ID 数组，读侧不知指向人、事件还是偏好 |
| 2 | **source_type**（USER_EXPLICIT / LLM_INFERRED / SYSTEM） | **有 2 值，缺 `SYSTEM`；命名与题面不同** | L86–90 `enum ["USER_EXPLICIT","ASSISTANT_INFERRED"]`；L6 `$comment` 错误示范(2) 把 `EXTERNAL_SOURCE`/`SYSTEM_GENERATED` 列 MUST 被拒；`13 §7.7.3 SEP-4`/`§7.8.2 L498`「仅 2 值」。题面 `LLM_INFERRED` 的语料原名是 **`ASSISTANT_INFERRED`**（术语纪律，不自造同义词）。**缺 `SYSTEM`** | 这是**有意的正式裁决**（`V2 §3.4`/SEP-4），非疏忽；但 `13 §7.7.2 L94` 提「system 侧汇总视图」、J8 预留 `affect_source='external'`，二者在 `source_type` 无合法落点 ⇒ 来源枚举与预留能力自相矛盾 |
| 3 | **confidence** | **有** | L75–80 必填 number 0–1（L16）；L118–128 `allOf`：`ASSISTANT_INFERRED ⇒ maximum 0.99` | 边界矛盾：必填无 `null` vs `13 §7.8.2 L496`「未知须留空/标 uncertain」⇒ `CONFLICTED` |
| 4 | **intensity** | **有** | L69–74 必填 number 0–1（L15），「ΔE 的 I」，禁百分制/1–10 量表 | 同上；无标定口径（谁给值、按什么量表） |
| 5 | **decay / 半衰期** | **schema 内缺** | 无 `decay_rate`/`half_life`/`decay`/`D`（`decay_field=false`）。衰减只在 `config/emotion L44–49` 全局 `emotion.decay_rate`（默认 0.1，1/天）与 `13 §7.9.1 L523` 的 `D=exp(−rate·Δt)` | 衰减是全局实验参数而非对象属性；`Δt` 取哪个时间未定义（对象只有 `created_at`；`event_id=null` ⇒ `D=0`） |
| 6 | **时间戳序列** | **缺** | 只有 `created_at`（L112–116）。无 `observed_at`/`recorded_at`/`valid_from`/`version`/`status`/`supersedes`/`history`/`timeline`（`timestamp_series=false`） | 「多次记录」「同一事件不同时间的情绪变化」无法表示（§7、13 问 4/5） |
| 7 | **可开关的 runtime 标记** | **缺** | 无 `enabled`/`runtime_enabled`/`active`/`detached`/`status`（`runtime_flag=false`）；开关是 `config/emotion` 的**全局**三项 `const:false` | 「独立关闭/调整/拔除」无对象级抓手；J5 清除仅原则句（13 问 12） |

**补充**：`evidence_refs`（L91–95「无证据 MUST NOT 进入正式库，除 system 侧汇总视图」）、`visibility`（L101–106 默认 `owner_private`，`public` 由 L130–134 `not` 禁）、`scope`（L107–111 v1 五值）、`temporal_position`（L81–84 4 值）、`raw_label`（L64–68 越界保留原词）**均有且设计良好**；`emotion_label` 15 值与 `V2 §2.14` 逐字一致（`13 §7.8.1 L471–482`）。

---

## 4. 产品目标逐条核对

| # | 产品目标 | 现状 | 判定 | 依据 |
|---|---|---|---|---|
| G1 | 事件只存事实、不夹情绪判断 | `Event.content` 仅事实；SEP-1/SEP-7；`schemas/event L33–38`+`$comment` 错误示范(1)(6)；`schemas/emotion-context L5`「MUST NOT 反写 Event.content」；E-13-08 写情绪入 content ⇒ `VALIDATION_ERROR` | **成立（最强项）** | `13 §7.7.3` |
| G2 | 情绪作为独立附件存在 | 独立对象；SEP-2「MUST NOT 作为 Event 内嵌字段」；`MemoryUnit.emotion_refs` 独立数组 | **成立（对象层）**，但「附件」要求可插拔 ⇒ G7 | `13 §7.7.3 SEP-2`；`schemas/memory-unit L173–177` |
| G3 | 情绪可关联具体事件/人物/互动经历 | 事件 `event_id` ✓；记忆 `related_memory_ids` ✓（无类型）；**人物/互动无字段**；Event 侧 `emotion_refs` 契约中不存在 | **部分成立 / 严重失真** | §3 第 1 项、§7 |
| G4 | 情绪在原始事件中占比小、不得破坏事实 | 设计上是「占比 0」：门禁 8 + `gates.writes_persistent_state=false` + X-1/X-3/I-5 | **过度满足**，但「占比小」缺可控比例，且存在二次污染路径 C-C007 | `13 §7.10 门禁 8`；`config/emotion L130–134` |
| G5 | 回忆时能重新取回情绪线索 | `emotion_context.maxItems:0`、`include_emotion const:false`、`query_enabled const:false`、`getEmotionContext` P5 默认关闭 | **不成立（P0）** | §9、C-C001 |
| G6 | 历史情绪对当前交互有限影响 | ΔE+8 门禁完整，但 `influence_enabled const:false`、Psychology BLOCKED、`R` TBD | **不成立（零影响且不可启用）** | §10、C-C001/C-C004 |
| G7 | 情绪不得凌驾事实/当前上下文/用户真实表达 | 完全隔离（不进 RRF/排序/`items[]`、`emotion_tokens=0`）；J2 来源分离；禁检索期 LLM 推断 | **成立**（但以「通道不存在」实现，非「有约束的有限接入」） | §12、C-C005/C-C007 |
| G8 | 附件可独立关闭/调整/拔除 | 全局三开关 `const:false` 永久关闭；对象无 runtime 标记；J5 清除 CONCEPT_ONLY | **不成立** | §14 问 12、C-C008 |

---

## 5. 完整链路还原（逐步判定十项）

> 十项 = 数据对象 / 关系字段 / 明确来源 / 历史记录 / 时间衰减 / 强度 / 影响比例 / 独立开关 / 污染事实 / 人格与关系影响。✓=有，—=无，△=部分。

| 步 | 环节 · 执行模块 | 对象/关系/来源 | 历史 | 衰减 | 强度 | 比例 | 开关 | 污染 | 人格 | 判定 |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 输入 | `EvidenceBinder`（Pipeline 阶段 10，`13 §7.1.1 L201`） | `Evidence` ✓`source_id/speaker_id` ✓来源 | ✓`content_hash`/`revoked` | — | — | — | — | ✓`original_text` 禁规范化 | — | **COMPLETE** |
| 2 提取 | **`EmotionExtractor`（阶段 11，M03，`09 §7.15 L597`）** | `EmotionDraft`（7 字段）✓`evidence_refs` ✓`source_type` 2 值；`auto_infer=false` 仅 `USER_EXPLICIT` | △幂等键防重复 | — | ✓`intensity` | — | △全局 `auto_infer` | ✓规则 7 禁写 content | ✓规则 8 禁写 PAD | **MOSTLY_COMPLETE**（草案缺 7 必填，`OI-09-06 VERIFY`） |
| 3 草案→对象 | **无明确模块** | 14 必填 `EmotionContext` ✓`event_id`/`related_memory_ids` | — | — | ✓ | — | — | — | — | **MISSING**（W4 映射算法无） |
| 4 关联 | **无** | `MemoryUnit.emotion_refs`/`event_refs` ✓数组；SEP-2 称经 `emotion_refs` | — | — | — | — | — | ✓`content` 禁情绪（L176） | — | **CONFLICTED**：`schemas/event` 无 `emotion_refs`；无反查算法/索引/关联表 |
| 5 保存 | M01（推定，`13 §11 L884`） | `emotion_context` 表？ | — | — | — | — | — | — | — | **MISSING**：`01 §7.16` 表 1–13 无该表，头部 L3 仅写「需新增」；无 DDL/列/索引/FK/事务/tombstone |
| 6 检索 | M02 / Intent Resolver / Anchor Router | `anchor` 含 `EMOTION_CONTEXT`、`intent` 含 `EMOTION_QUERY` | — | — | — | — | △`include_emotion const:false`；`group_quota.emotion=0`（`11 §7.19 L759`） | ✓不进 RRF | — | **设计定义、契约封死** |
| 7 回取 | `memory.getEmotionContext(event_id\|memory_id)`（`14 §7.2.7`，P5 默认关闭） | `read-response.emotion_context[]` 独立字段 | — | — | — | — | ✗`maxItems:0` | ✓不与 `items[]` 同池 | — | **BLOCKED（P0）**：关闭返回 `FEATURE_DISABLED`/`EMOTION_DISABLED`（`14 E-14-09 L714`） |
| 8 当前交互 | **Psychology（BLOCKED/DEFERRED，CR-05/OI-029）**；Persona FUTURE | ΔE→运行时状态→Persona Context | — | ✓全局 `decay_rate` | ✓`I` | ✓`max_influence` | ✗`influence_enabled const:false` | ✓门禁 8 | ✓`writes_persistent_state=false`、I-5、X-3 | **BLOCKED**：无落点、无 emotion 分区、无合并算法 |

**结论**：1–2 有正式设计；3–5（草案→对象→关联→存储）在契约/DDL 层断裂；6–8（检索→回取→使用）被 `const:false` + `maxItems:0` 硬封死。**全链路当前不可执行**，但各步设计意图可读，故 `PARTIAL` 而非 `BLOCKED`。

---

## 6. ① EmotionContext 数据完整性矩阵

> 来源=谁产生；可空按 `type`；「够用」按产品目标。

| 字段 | 类型/枚举 | 默认 | 必填 | 来源 | 可空 | 够用 | 缺口 |
|---|---|---|---|---|---|---|---|
| `emotion_id` | uuid（正则锁定） | 无 | ✓ | M03/M01 | 否 | 够用 | 「稳定不可复用」(L30)，但无 `version` 配对 |
| `event_id` | uuid\|**null** | 无 | ✓ | `Event.event_id`；L35「MUST NOT 填 memory_id」 | **是** | 部分 | `null` 允许，SEP-8 规定 `null` MUST NOT 触发 ΔE；但无 `subject_ref_type` 说明 null 时挂什么 |
| `subject_id` | string 1–128 | 无 | ✓ | `EntityResolver`(L41) | 否 | 部分 | 是**情绪所属主体**非关联目标；群体情绪「MUST 明确归属」但无 `group_id`/`member_ids` |
| `emotion_label` | enum **15** | 无 | ✓ | 受控词表(L62，J4 配置管理) | 否 | 够用 | 越界映射 `neutral`+保留 `raw_label`；`neutral` 丢原语义(P2) |
| `raw_label` | string\|**null** ≤128 | 无 | ✓ | 用户/模型原词 | **是** | 够用 | L67「MUST NOT 承载私人正文长文本」；>128 须拆分/截断并记录，但无拆分算法 |
| `intensity` | number **0–1** | 无 | ✓ | ΔE 的 `I`(L73) | 否 | 部分 | 无 `null`/未知态，与 `13 §7.8.2 L495`「未知须留空」`CONFLICTED`；无标定口径、无 `model_version` |
| `confidence` | number **0–1** | 无 | ✓ | ΔE 的 `C`(L79) | 否 | 部分 | `allOf` 推断 `≤0.99`；语义「≠事实真实性」；无 `null`、无校准来源 |
| `temporal_position` | `BEFORE`/`DURING`/`AFTER`/`UNKNOWN` | 无 | ✓ | `V2 §2.13` | 否 | 部分 | 是**相对事件的阶段**，不是时间戳；缺 `observed_at` |
| `source_type` | **`USER_EXPLICIT`/`ASSISTANT_INFERRED`** | 无 | ✓ | 用户明示/模型推断 | 否 | 部分 | **缺 `SYSTEM`**（§3 第 2 项）；推断侧默认不产出（CR-04） |
| `evidence_refs` | array\<uuid\> | 无 | ✓ | `Evidence.evidence_id` | 否(数组可空) | 够用 | L94 例外「system 侧汇总视图」但 `source_type` 无 `SYSTEM` 值可表达 |
| `related_memory_ids` | array\<uuid\> | 无 | ✓ | `memory_id`(L99「MUST NOT 填 event_id」) | 否(数组可空) | 部分 | **无类型 ID 数组**，无 `maxItems`/去重/冲突裁决/写入者 |
| `visibility` | `owner_private`/`group_shared`/`public` | `owner_private` | ✓ | 权限继承(L105) | 否 | 够用 | `public` 由 L130–134 `not` 在 V1 禁用(CR-07) |
| `scope` | enum **5 值** | 无 | ✓ | v1 五值，L110「派生情绪 MUST 继承来源最严格交集(INV-08)」 | 否 | 够用 | 映射见 `V2 §4.3` |
| `created_at` | date-time | 无 | ✓ | UTC RFC3339 | 否 | 部分 | **唯一时间字段**；无法区分「情绪发生时间」与「记录时间」（`01 §7.14` 事实时间≠记录时间纪律未延伸到情绪） |

**缺失字段清单（RECOMMENDED ADDITION / `NEW PROPOSAL`，见 §16.1）**：`subject_ref_type`+`target_refs`（类型化关联）·`related_person_ids`·`interaction_id`/`session_id`·`observed_at`/`recorded_at`·`status`(ACTIVE/SUPERSEDED/ARCHIVED/DELETED)·`version`·`supersedes`/`emotion_thread_id`·对象级 `decay_rate`/`half_life`·`runtime_enabled`/`detached`·`model_version`/`extraction_method`/`annotator`/`calibration_version`·`value_state`·`origin_channel`。

---

## 7. ② 事件与情绪关联模型（多重情绪 / 时间序列 / 来源分离）

```
Event(event_id) ──────────────┐
                              │  EmotionContext.event_id (uuid|null, MUST=Event.event_id)
                              ▼
                      EmotionContext(emotion_id)  ◄── 多条并列，无顺序/版本/状态
                              │  related_memory_ids[] (memory_id, 无类型)
                              ▼
MemoryUnit(memory_id) ── emotion_refs[] (emotion_id) ──► EmotionContext
        └── event_refs[] (event_id) ──► Event   （L161「情绪不写本字段，写 emotion_refs」）
```
引用：`schemas/emotion-context L32–42/L96–99`；`schemas/memory-unit L158–177`；`schemas/event L65–73`；`13 §7.7.3 SEP-2 L453/SEP-3 L454/示例 L465`；`11 §7.7 L377–379`。

| 维度 | EXISTING DESIGN | DESIGN COMPLETENESS |
|---|---|---|
| 多重情绪 | 三条数组 ⇒ 理论可挂多条；`13 §7.7.3 L465` 同事件 `anxious`(DURING)+`relieved`(AFTER) 两条 | **不可执行**：无 `maxItems`/基数/去重/冲突判定（对比 `13 §7.1.3` 为 `memory_evidence_links` 定义主键/索引/6 条 MUST，情绪关联**一条都没有**） |
| 时间序列 | 独立 `emotion_id` + `created_at` + `temporal_position` 4 值 | **不能表示**：4 值是「相对事件阶段」非时间轴；无 `version`/`status`/`supersedes`/`observed_at`；谁覆盖谁、何时变化、幅度均无表达 |
| 来源分离 | `source_type` 2 值；`09 §7.15 规则 2`「二者 MUST NOT 合并」；`13 §7.6.2 P-10`；`allOf` 推断≤0.99；门禁 1 `inferred_weight<1.0` | **部分**：纪律完整，但 `auto_infer=false` const 锁定 ⇒ 推断侧无产出路径；无 `model_version`/`annotator`，`confidence` 不可审计 |
| 人物/互动 | 无 | **否**（§3 第 1 项） |
| 反查 | `getEmotionContext(event_id)` 入参存在（`14 §7.2.7 L457`） | **否**：无「给定 event_id 取全部情绪」的算法/索引/关联表；实现未定义且默认关闭 |
| Event 侧关联 | SEP-2 称经 `emotion_refs` 关联 | **CONFLICTED**：`schemas/event` 的 15 个 `properties` **无 `emotion_refs`**（机器校验）；该字段只在 `schemas/memory-unit` |

---

## 8. ③ 情绪写入流程（逐步，标执行模块）

| 步 | 动作 | 输入→输出 | 执行模块 | 条件/边界 | 状态 |
|---|---|---|---|---|---|
| W1 | 绑定证据 | 消息→`Evidence` | `EvidenceBinder`（阶段 10） | `content_hash` 去重；权限最严格交集(`13 §7.4`) | COMPLETE |
| W2 | 拆 Event 事实 | 原话→`Event.content` | M03 / 阶段 7–9 | 禁写情绪判断(SEP-1、`schemas/event L37`) | COMPLETE |
| W3 | 提取情绪线索 | `draft`→`EmotionDraft[]`(可空) | **`EmotionExtractor`（阶段 11，M03）** | 8 条 MUST(`09 §7.15 L580–587`)；`auto_infer=false` 且无 `USER_EXPLICIT` ⇒ 跳过并记 `skipped_reason=emotion_disabled` | MOSTLY_COMPLETE |
| W4 | 草案→正式对象 | `EmotionDraft`(7)→`EmotionContext`(14) | **无明确模块** | 缺 `emotion_id`/`event_id`/`subject_id`/`related_memory_ids`/`visibility`/`scope`/`created_at` 共 7 字段生成规则；`09 §14` 自登记 `OI-09-06 字段映射 VERIFY` | **MISSING** |
| W5 | 建立三类关联 | ID→关联行 | **无明确模块** | 无关联表/FK/索引/`support_kind`/同一事务纪律 | **MISSING** |
| W6 | 落库 | →`emotion_context` 表行 | M01（推定，`13 §11 L884`） | `01 §7.16` 表 1–13 **无该表**；头部 L3 仅「需新增」；无 DDL/列/索引/版本/tombstone | **MISSING** |
| W7 | 校验与审计 | →`ValidationReport`（阶段 14） | M03/M04 | `09 §7.15` 幂等键；情绪专属校验仅 `13 §12.2 T-13-24~31`（**NOT RUN**） | PARTIAL |
| W8 | 失败处置 | →`emotion_extraction_failed` + `emotion_refs=[]` | `EmotionExtractor` | `09 §7.15 L591`「**MUST NOT 阻塞事实单元提交**（情绪为附属产物）」——设计正确且重要 | COMPLETE |

**结论**：W1–W3、W8 可执行；**W4–W6 三步全缺** —— 情绪线索提取后，没有任何正式设计把它变成可存储、可关联、可反查的记录。这是 C-C002 的直接依据。

---

## 9. ④ 情绪读取流程（逐步，标执行模块与接口）

| 步 | 动作 · 接口/模块 | 契约依据 | 当前实际结果 | 状态 |
|---|---|---|---|---|
| R1 | 意图识别 · `Intent Resolver`（9 类含 `EMOTION_QUERY`） | `schemas/read-request L68–81`；`11 §7.15` | L81「`query_enabled=false` 时 MUST 返回空结果」 | 定义存在、被封 |
| R2 | 锚点路由 · `Anchor Router`（8 锚点含 `EMOTION_CONTEXT`） | `schemas/read-request L49–67` | L66「关闭时**必须返回空而非降级**」；`11 §7.19 L759` `group_quota.emotion` 默认 **0** | 配额 0 |
| R3 | 请求构造 · `Context Adapter`（FUTURE，`00 §7.7 L408`） | `schemas/read-request` | `include_emotion` **`const:false`**（L114–117）「MUST 恒为 false…启用须重新裁决并同步修改本 schema」 | 恒 false |
| R4 | 候选检索 · M02 / FTS5+LanceDB | `config/retrieval L105–158` `fusion.weights` 仅 `fts`/`vector` | 情绪**不进 RRF**（§12） | 独立、不共池 |
| R5 | 事实返回 · `memory.search`/`getEvent`→`items[]` | `13 §7.12.1 L648–650`「默认 MUST 只返回 Event 事实；MUST NOT 附带 EmotionContext」 | 事实链完整 | COMPLETE |
| R6 | 情绪回取 · **`memory.getEmotionContext(event_id\|memory_id)`** | `14 §7.2.7 L439–471`；`13 §8.5 L765`；`V2 §7 L357` | **P5 且默认关闭**(CR-02/OI-026)；关闭 ⇒ `FEATURE_DISABLED`/`EMOTION_DISABLED`，`14 L714`「MUST NOT 返回空数组冒充无情绪」 | **BLOCKED** |
| R7 | 响应承载 · `read-response.emotion_context[]` | `schemas/read-response L309–318` | **`maxItems:0`** V1 恒空；「关闭态返回非空即违规」 | **BLOCKED** |
| R8 | 权限 · M05 `canRead` 单独判权 | `13 §7.7.3 SEP-6`；`14 §7.2.7 L447` | 设计完备（事件可读≠情绪可读），但无可达请求 | COMPLETE(不可达) |
| R9 | Token · `emotion_tokens` | `schemas/read-response L518`；`13 §7.13.1 L688` | V1 恒 0 | COMPLETE(恒 0) |
| R10 | 排除可观测 · `trace.excluded[].EMOTION_DISABLED` | `schemas/read-response L484–486` | 有专用枚举值，设计细致 | COMPLETE |
| R11 | 组装进 Prompt · `Context Builder`(ALG-06，`02 §7.7`) | `13 §7.12.3` 三种主动请求(PROPOSED 均关闭)；`14 §7.2.7` 输出 `emotion_contexts[]`+`items[]` | **无 emotion 分区、无合并顺序、无 prompt 段规范、无预算默认值** | **MISSING** |

**结论**：R1–R5、R8–R10 有正式设计且防护严谨；**R6/R7 被机器契约硬封死，R11 无合并算法**。「回忆时重新取回情绪线索」在 R6/R7/R11 三处同时断裂 ⇒ C-C001（P0）。

---

## 10. ⑤ 情绪衰减与影响机制

**EXISTING DESIGN（`13 §7.9/§7.10`、`V2 §9`）**：
```
ΔE_raw   = I × C × R × D
C_eff    = C × (source_type=ASSISTANT_INFERRED ? emotion.inferred_weight : 1.0)
R < emotion.relevance_threshold ⇒ ΔE=0                       (门禁2)
D        = exp(−emotion.decay_rate × Δt_days)                (门禁5)
ΔE_i     = min(ΔE_raw_i, emotion.max_influence)              (门禁3)
ΔE_total = min(Σ_i ΔE_i, emotion.max_influence)              (门禁4)
inferred_weight < 1.0 (门禁1) · influence_enabled=false (门禁6)
无明确事件/event_id=null ⇒ ΔE=0 (门禁7, K-6/CR-09)
ΔE 只作用 Psychology 运行时状态；MUST NOT 写 event.content / 改 validation_state /
   产生候选 / 进 M02 排序 / 改 PAD / 落库 (门禁8)
任一变量不可计算 ⇒ 取 0（MUST NOT 填 0.5 / 产生 NaN）        (13 §7.9.1 L541)
```
`I`←`intensity`、`C`←`confidence`（`13 §7.9.1 L520–523`）。两组逐步算例（`13 §7.9.2/§7.9.3`）标「设计演算、非实测」(D-AP-06)。公式形态与门禁条款判 **COMPLETE**（`STANDARDS.md §5`：参数未标定 ≠ 不可实现）。

**DESIGN COMPLETENESS：明确缺失**

| 缺失项 | 说明 | 证据 | 评级 |
|---|---|---|---|
| `R` 无计算算法 | 13 §7.9.1 L522 自标「PROPOSED，参数化待标定」；`13 §14 OI-13-03` TBD | L522/L1015 | **MISSING** |
| 多事件集合无选取规则 | 门禁 4 说「求和」，未定义取哪些事件：`top_k`/时间窗/`subject_id` 过滤/去重/`event_id=null` 处置 | `13 §7.10 门禁 4` | **MISSING** |
| `Δt` 时间源未定义 | L523「事件距今时长」，但对象只有 `created_at`（记录时间）无 `observed_at`；`event_id=null` ⇒ `D=0`；`01 §7.14` 事实时间≠记录时间纪律未延伸到情绪 | L523；`schemas/emotion-context L112–116` | **MISSING** |
| 衰减无持久化落点 | `config/emotion L130–134` `gates.writes_persistent_state=false`（const 红线）；X-2「Session 结束丢弃短期情绪」；X-3「不落库」⇒ ΔE 每轮重算，但触发点/输入快照/状态载体全无 | `config/emotion L130–134`；`13 §5 L148–149` | **MISSING** |
| 影响比例只覆盖 Psychology | `max_influence` 只裁剪 ΔE；产品要「情绪在原始事件中占比小」需可控比例，当前是硬隔离（占比 0） | `13 §7.10 门禁 3/4/8` | **CONCEPT_ONLY** |
| 参数默认值三处冲突 | 下表 | `13 §7.13.2` / `config/emotion` / `15 §7.8` | **CONFLICTED** |
| 无情绪状态机 | `state-machines.md` SM-1~SM-8 全表无 `emotion`/`情绪`/`EmotionContext`（机器校验 `emotion_related_sm=false`） | 全文 165 行 | **MISSING**(P2) |

**参数默认值冲突表（CONFLICTED，P1）**

| 参数 ID | `13 §7.13.2`（含 §7.9 算例） | `config/emotion` | `15 §7.8` |
|---|---|---|---|
| `emotion.relevance_threshold` | **0.30** | 0.6 | 0.6 |
| `emotion.inferred_weight` | **0.60** | 0.5 | 0.5 |
| `emotion.max_influence` | **0.20** | 0.15 | 0.15 |
| `emotion.decay_rate` | **0.05** | 0.1 | 0.1 |

后果：`13 §7.9.2` 用 0.20 裁剪得 `ΔE=0.20`、`§7.9.3` 对照演算 `0.020082` 与总上限 `0.20`，`§13 AC-13-10` 要求「算例与 §7.9 一致」；实现若按 `config`/`15` 的 0.15 则全部对不上、验收 FAIL。`15 §7.8 L527` 又声明「CR-03 未通过前不得标定」⇒ 这是**未裁决的契约冲突**而非待调参。

---

## 11. ⑥ 与 Persona 的关系（谁负责）

**明确表态：架构中不存在 `EmotionContext → Persona 上下文` 的路径 —— 无字段、无接口、无执行模块、无组装步骤。**

| 候选路径 | EXISTING DESIGN | DESIGN COMPLETENESS |
|---|---|---|
| `EmotionContext`→`PersonProfile` | `schemas/person-profile L5/L55`「MUST NOT 写事件流水或情绪判断」；11 个 `section` 无 emotion 分区；`basis` 三值不含情绪；`12 §7 L401` 同禁令 | 设计**主动禁止**情绪进画像（正确防污染，但「理解用户」不含情绪维度） |
| `EmotionContext`→ΔE→**Psychology** | 门禁 8 / `V2 §9 门禁 8`「只作用于 Psychology 运行时状态」；`00 §0.4 L46`「独立模块，不得并入 M01–M05」 | **Psychology `BLOCKED/DEFERRED`**（`13 §3.2 L91`、`§14 OI-029`、`00 §7.7 L405`、`12 §14 L729`、`14 §14 L1072`）；`psy-model 暂不引入`(I-7/X-6)；接口 `DEFERRED` |
| Psychology→**Persona Context** | `00 §7.7 L406` Persona「记忆消费者；经授权视图接收记忆」**FUTURE**；`08 §7.2 L78`「D1–D3 待 CR-08B 单独呈报」 | 无字段/接口/组装步骤；`08 §7.1 L229` 链路 `Host Agent→Context Adapter→…→Memory API` **无 Psychology 节点**；`Context Adapter` FUTURE(`00 §7.7 L408`) ⇒ **MISSING** |
| `Context Builder`(ALG-06) | `02-clean §7.7` 有 `uncertain` 分区（标 `inferred`/`uncertain`/`disputed`） | **无 `emotion` 分区**；`13 §7.12.3` 三种主动请求 PROPOSED 且均关闭；`14 §7.2.7` 输出 `emotion_contexts[]` 与 `items[]` 并列但无拼装规则 ⇒ **MISSING** |

**谁负责**（`NEW PROPOSAL`）：由 `Context Builder`（ALG-06，M02 侧）把「事实段/证据段/画像段/情绪段」分区组装，`Context Adapter` 注入宿主；`Psychology` 只提供**运行时**情绪状态快照，不写 Persona / `PersonProfile` / `MemoryUnit`（§16.2/§17）。

**对「稳定人格」**：当前「情绪完全不进入人格」保证了「不凌驾」，但产品要「有限影响+稳定人格」而非「零影响」。零影响来自通道不存在，不是有隔离算法的有限接入；一旦绕过裁决直接把情绪塞进 Prompt，反而**没有**「不得凌驾」的可验证边界（§12）。

---

## 12. ⑦ 与 Retrieval 的关系（排序信号 / 权重 / 会不会压过事实）

**明确表态：当前情绪不与事实检索共用 `RRF`、不进同一排序池、不会压过事实；情绪走独立字段/独立路由。代价是「回取」被完全切断，且「启用后如何合并」完全没有定义。**

| 判定点 | EXISTING DESIGN |
|---|---|
| RRF 通道 | `RRF(d)=Σ w_r/(k+rank_r(d))`，通道仅 **`fts`/`vector`** 两路；`structured 只过滤不参与 RRF 计分`(OI-022)（`config/retrieval L105–158`，L113/L126–147；`02-clean L3`） |
| 精排特征 | 五特征 `rrf`/`exact_match`/`type_relevance`/`time_relevance`/`source_quality`，**无 emotion 权重**（机器校验 `emotion_weight_in_reranking=false`）（`config/retrieval L186–223`；`02-clean §7.5 ALG-04`） |
| 明确禁止加成 | X-1「delta-only 影子验证，不改变 M02 最终排序」；`02-clean L481/L1104/L1463` 把「心理加成写入公式」列禁止行为（`00 §0 L213`；`13 §5 L147`） |
| 独立路由 | `EMOTION_QUERY → EMOTION_CONTEXT → 事件 → Psychology 当前状态(BLOCKED)`，独立路由不加到事实 score（`V2 §8 L376`；`11 §7.15 L573/L676`） |
| 独立承载 | `emotion_context[]` 与 `items[]` **并列不同字段**；`emotion_tokens` 独立；`trace.excluded` 有 `EMOTION_DISABLED` 专用原因（`schemas/read-response L51/L309/L518/L484`） |
| 关闭时 | `EMOTION_CONTEXT` 锚点「必须返回空而非降级」；`group_quota.emotion=0`；`include_emotion const:false`；`emotion_context.maxItems:0` |

**「如果独立，说明如何合并」——当前没有合并机制（MISSING）**：①`13 §7.12.3` 三种主动请求均 `PROPOSED` 且「均受默认关闭约束」(L657–669)，不是可执行算法；L669 还禁止「把当前对话看起来相关当作用户明确请求」。②`14 §7.2.7` 定义输出但未定义 `emotion_contexts[]` 与 `items[]` 在 Prompt 中的先后、段落名、占比上限、是否争 `budget_tokens`。③`02 §7.7 ALG-06` 分区表无 emotion 分区；`13 §7.13.1` 只要求分项计量，未给启用后 `emotion_tokens` 上限。④**潜在压过事实的二次路径 C-C007**：`schemas/memory-unit L54` `semantic_type` 6 值含 `EMOTION_CONTEXT`、L102 `anchor` 8 值含 `EMOTION_CONTEXT`，而 L46–50 的 `content` **无情绪禁令**（对比 `schemas/event L37` 有）；`schemas/search-document L16` 的 `content` 是「检索文本投影」亦无隔离。若把情绪写成一条 `semantic_type=EMOTION_CONTEXT` 的 `MemoryUnit.content`，它会走 **fts/vector RRF 正常通道**进 `items[]` 与事实**共池竞争**；`13 §7.7.3 SEP-2` 只禁「EmotionContext 作为 **Event** 内嵌字段」，门禁 8 只管 ΔE，**管不到这条写入路径**。这是唯一可能让情绪与事实共池并压过事实的合法路径，必须显式封堵。

---

## 13. ⑧ 缺失算法清单

> 口径 `STANDARDS.md §5`；`CONFLICTED`=文档互矛盾；`MISSING`=无有效定义；「已定义」与「参数已标定」分离。

| # | 算法/能力 | 输入→输出 | 执行模块 | 评级 | 缺口 |
|---|---|---|---|---|---|
| A1 | `EmotionContext` 对象契约 | →14 必填 | M01 | **MOSTLY_COMPLETE** | `intensity`/`confidence` 必填无 null vs `13 §7.8.2`「未知留空」；缺 7 类字段(§6) |
| A2 | 15 标签越界映射 | 自由词→`neutral`+`raw_label` | M03 | **MOSTLY_COMPLETE** | `13 §7.8.1`/`09 §7.15 规则 3–4`；缺 >128 字符拆分算法；词表 `OI-009 TBD` |
| A3 | `EmotionDraft` 提取(阶段 11) | `draft`→`EmotionDraft[]` | M03 | **MOSTLY_COMPLETE** | 8 条 MUST+幂等键+失败不阻塞事实；草案仅 7 字段 |
| A4 | 草案→对象映射与落库 | `EmotionDraft`→正式 `EmotionContext` | **无** | **MISSING** | `09 §14 OI-09-06` 自登记 VERIFY；7 必填无生成规则、无 DDL/事务/写入者 |
| A5 | Event↔Emotion↔MemoryUnit 关联 | ID→关联行/反查 | **无** | **MISSING/CONFLICTED** | 无关联表/FK/索引/基数/去重/冲突裁决；SEP-2 与 `schemas/event` 矛盾 |
| A6 | 情绪时间序列/变化 | 多条→轨迹/当前值 | **无** | **MISSING** | 无 `version`/`status`/`supersedes`/`observed_at`；`temporal_position` 非时间轴 |
| A7 | 来源分离与置信度校准 | 原句/模型输出→`source_type`+`confidence` | M03 | **PARTIAL** | 分离纪律完整；缺 `model_version`/`annotator`/`extraction_method`/`calibration_version`；推断侧默认不产出 |
| A8 | ΔE=I×C×R×D | `I/C/R/D`→`ΔE_total` | Psychology(BLOCKED) | **COMPLETE(公式)** | 公式/门禁/裁剪/缺失归零/算例齐备；参数冲突(A9) |
| A9 | ΔE 参数一致性与标定 | 配置→4 数值 | 配置管理员 | **CONFLICTED** | 13/config/15 三处不同；15 声明「CR-03 前不得标定」 |
| A10 | `R` 话题—事件关联度 | 话题+事件→`R∈[0,1]` | **无** | **MISSING** | `13 §7.9.1 L522` PROPOSED；`OI-13-03 TBD`；无公式/标注集/失败处理 |
| A11 | 多事件集合选取与求和 | 事件集合→`ΣΔE_i` | **无** | **MISSING** | 门禁 4 只给求和裁剪，未给 `top_k`/时间窗/主体过滤/去重/`null` 处置 |
| A12 | 时间衰减 `D` | `Δt_days`→`D` | Psychology(BLOCKED) | **PARTIAL** | 公式有；`Δt` 时间源未定义；无 `observed_at`/`half_life`；`writes_persistent_state=false` 无状态载体 |
| A13 | 情绪回取 `getEmotionContext` | `event_id\|memory_id`→`emotion_contexts[]` | P5(默认关闭) | **BLOCKED→MISSING** | 接口形态有(`14 §7.2.7`)；但 `maxItems:0`+`query_enabled const:false`；反查/排序/去重/权限后过滤全无 |
| A14 | 情绪—事实上下文合并 | `items[]`+`emotion_contexts[]`→Prompt 段 | `Context Builder`(ALG-06) | **MISSING** | 无 emotion 分区/顺序/占比/预算上限/「不压过事实」判据 |
| A15 | 情绪→Persona/Psychology 落点 | ΔE→运行时状态 | **Psychology(BLOCKED)** | **CONCEPT_ONLY** | CR-05/OI-029；无接口契约/状态载体/与 `Context Adapter` 交互步骤 |
| A16 | 独立开关/拔除(J5) | 用户请求→清除回执 | M04 修订通道(仅原则) | **CONCEPT_ONLY** | `13 §5 L141` 一句原则；无 API/批量/缓存/索引/备份清理/对象级 runtime 标记；范围受 K-2 BLOCKED |
| A17 | 情绪状态机 | 附件状态→转换 | — | **MISSING** | `state-machines.md` SM-1~8 无情绪状态机（机器校验） |
| A18 | 事实污染防护 | 任意写入→拒绝/隔离 | M01/M03/M05 | **COMPLETE(Event 侧)** | SEP-1/7/8、门禁 8、`schemas/event L37`、`gates.writes_persistent_state=false`、X-1/X-3；**MemoryUnit 侧未封堵**(C-C007) |
| A19 | 关闭态降级(Runtime 未启用) | 读请求→事实结果 | M02/M05 | **COMPLETE** | `13 §7.12.1`、`14 E-14-09`、`09 §7.15 失败不阻塞`、`emotion_context=[]`、`emotion_tokens=0` |
| A20 | 测试与验收规格 | →47 项测试/14 条验收 | — | **MOSTLY_COMPLETE(规格)/NOT RUN(执行)** | `13 §12.1 T-13-01~23`、`§12.2 T-13-24~47`、`§13 AC-13-01~14`；按规则 7 不判架构不可行 |

---

## 14. 13 问逐条回答（编号，不合并省略）

> 每条三分类；`NEW PROPOSAL` = 未获批准的建议。

**1 `EmotionContext` 数据结构是否充分？** `EXISTING DESIGN`：`schemas/emotion-context L9–24` **14 必填字段**，`13 §7.7.2`/`V2 §3.4` 逐字一致，Draft7 自检通过。`DESIGN COMPLETENESS`：对象契约 `MOSTLY_COMPLETE`，作为「可关联/可演化/可治理的附件」**不充分**——缺类型化指针、人物/互动关联、`observed_at`/`version`/`status`/对象级 `decay_rate`/`runtime_enabled`/来源元数据；且 `intensity`/`confidence` 必填无 `null` 与 `13 §7.8.2 L495–496`「未知留空/标 uncertain」`CONFLICTED`。`RECOMMENDED ADDITION / NEW PROPOSAL`：按 §6 缺失清单做最小增补(§16.1)。

**2 情绪与事件通过什么 ID 或关系关联？** `EXISTING DESIGN`：`EmotionContext.event_id`（L32–36，`uuid|null`，MUST=`Event.event_id`，MUST NOT 填 `memory_id`）、`related_memory_ids`（L96–99，`memory_id` 数组）、`MemoryUnit.emotion_refs`（L173–177）与 `event_refs`（L158–162）；`13 §7.7.3 SEP-2 L453` 称「经 `event_id`/`emotion_refs` 关联」。`DESIGN COMPLETENESS`：**单向、无类型、无反查算法**——`schemas/event` 15 个 `properties` **无 `emotion_refs`**，SEP-2 与契约 `CONFLICTED`；无 `(event_id,emotion_id)` 关联表/FK/索引/基数/去重/冲突裁决（对比 `13 §7.1.3` 为 `memory_evidence_links` 定义主键+索引+6 条 MUST，情绪关联一条都没有）。`NEW PROPOSAL`：增 `emotion_links` + `INDEX(target_id)`(§16.1/§16.2)。

**3 一条记忆可关联几条情绪记录？** `EXISTING DESIGN`：`emotion_refs` 与 `related_memory_ids` 均 `array<uuid>` 无 `maxItems`；`13 §7.7.3 L465` 示例同事件两条（`anxious` DURING + `relieved` AFTER），多重情绪设计上被允许。`DESIGN COMPLETENESS`：**「可多条」但无上界/排序/去重/冲突判定/写入者**；多条并列时谁优先、`temporal_position` 相同如何去重、推断与明示能否共存全无规则。`NEW PROPOSAL`：设 `emotion.max_per_event`(EXP-DEFAULT 如 8) 与 `(subject_id,emotion_label,temporal_position,source_type)` 唯一约束(§16.1)。

**4 多次情绪记录如何保存？** `EXISTING DESIGN`：每条独立 `emotion_id`（「稳定不可复用」L30）+ `created_at`；`13 §7.4.2` 的 `content_hash` 去重**只针对 `Evidence`**；`09 §7.15 L595` 幂等键 `(candidate_draft_id,emotion_version,prompt_version)` 只防同次抽取重复。`DESIGN COMPLETENESS`：**PARTIAL**——可并列存互不相干快照，但无「同属一条情绪轨迹」表达，无 `prior_emotion_id`/`thread_id`/`sequence_id`，无历史版本表，无对象级 `retention_policy`。`NEW PROPOSAL`：`emotion_thread_id`+`version`+`supersedes`+`status`(§16.1)。

**5 同一事件不同时间的情绪变化如何表示？** `EXISTING DESIGN`：`temporal_position` 4 值（L81–84/`V2 §2.13`）表达「相对事件的阶段」；`created_at` 表达记录时间。`DESIGN COMPLETENESS`：**不能表示**——4 值是分类不是时间轴；无 `observed_at`/`version`/`status`/`supersedes`/变化幅度；`13 §7.9.1 L523` 的 `Δt_days` 在对象上无对应时间源（§10.2）；`01 §7.14`「事实时间≠记录时间」纪律未延伸到情绪。`NEW PROPOSAL`：`observed_at`+`recorded_at`+`version`+`supersedes`+`status`，定义「同一 `(subject_id,event_id,emotion_thread_id)` 最新 ACTIVE 为当前值」(§16.1/§16.2)。

**6 来源是用户明确表达还是 LLM 推断？** `EXISTING DESIGN`：`source_type` 2 值 `USER_EXPLICIT`/`ASSISTANT_INFERRED`（L86–90）；语料原名是 **`ASSISTANT_INFERRED`**（不是 `LLM_INFERRED`，术语纪律不自造同义词）；`09 §7.15 规则 2`「二者 MUST NOT 合并」；`13 §7.6.2 P-10`；`allOf` 推断 `confidence≤0.99`；门禁 1 `inferred_weight<1.0`。`DESIGN COMPLETENESS`：**分离纪律完整（本路最强项之一），但推断侧默认不产出**（`config/emotion L31–36 auto_infer const:false`、`15 §7.8 L245`「无人可写」、CR-04/OI-028）；运行态只可能 `USER_EXPLICIT`。另**缺 `SYSTEM` 值**，`13 §7.7.2 L94` 的「system 侧汇总视图」与 J8 `affect_source='external'` 无合法落点(§3 第 2 项)。`NEW PROPOSAL`：保留 2 值裁决，另设 `origin_channel` 记录通道而非改 `source_type` 枚举(§16.1)。

**7 置信度如何表达？** `EXISTING DESIGN`：`confidence` number 0–1 必填（L75–80），语义「情绪判断可信度 ≠ 事实真实性」；`allOf L118–128` 推断 `maximum 0.99`；作为 ΔE 的 `C`，推断时乘 `inferred_weight`(`13 §7.9.1 L521`)。`DESIGN COMPLETENESS`：**PARTIAL**——有字段/范围/折扣，但必填不可 `null` 与 `13 §7.8.2 L496`「未知留空」`CONFLICTED`；无 `calibration_version`/`model_version`/`annotator`，无法回答「0.8 是谁按什么标准给的」；T-13-28 `NOT RUN`。`NEW PROPOSAL`：`confidence` 允许 `null`+`value_state=UNCERTAIN`，增来源元数据(§16.1)。

**8 情绪强度如何表达？** `EXISTING DESIGN`：`intensity` number 0–1 必填（L69–74），「ΔE 的 I」，禁百分制/1–10 量表(L73/L6 错误示范 3)。`DESIGN COMPLETENESS`：**PARTIAL**——同 `confidence` 的必填/未知矛盾；**无标定口径**（谁把「有点破防」映射成 0.62、按什么量表、是否随主体归一化全无定义）；对象无 `decay_rate`/`half_life`，强度本身不随时间衰减（衰减只作用于 ΔE 的 `D`）。`NEW PROPOSAL`：增 `intensity_scale_version`/`extraction_method` 与对象级 `decay_rate`(§16.1)。

**9 如何控制情绪对长期记忆的影响？** `EXISTING DESIGN`：**硬隔离（占比 0）**——门禁 8 + `config/emotion L130–134 gates.writes_persistent_state=false`(const 红线) + X-1「不得进 M02 排序公式」 + X-3/I-5「不落库/不产生候选/不改 PAD」 + `schemas/person-profile L55` 与 `12 §7 L401` 禁情绪进画像；`02-clean L481/L1104/L1463` 禁心理加成进排序。`DESIGN COMPLETENESS`：**「不破坏事实记忆」成立且防护最严密**；但产品要「占比小、可控比例」，当前是零占比且无影响比例参数（`max_influence` 只裁剪 ΔE）；另有 C-C007 二次路径未封堵。`NEW PROPOSAL`：保留全部红线，补 `emotion.long_term_write_forbidden=true` 显式常量 + 契约级禁令(§16.1/§16.2)。

**10 如何控制历史情绪对当前状态的影响？** `EXISTING DESIGN`：ΔE=I×C×R×D + 8 门禁（`13 §7.9/§7.10`/`V2 §9`）：推断折扣、`R` 阈值归零、单事件上限、多事件总上限、时间衰减、默认关闭、禁沉默触发、不写回事实；`emotion.max_influence` 是总闸门。`DESIGN COMPLETENESS`：**公式 COMPLETE，落地 BLOCKED/PARTIAL**——`influence_enabled const:false`；Psychology BLOCKED(CR-05/OI-029)；`R` `OI-13-03 TBD`；`Δt` 时间源未定义；多事件集合未定义；参数三处冲突(§10.2)；`gates.max_delta_per_turn` 默认 `null` 且标「TBD 标定」。`NEW PROPOSAL`：先统一参数、定义 `R`/`Δt`/集合，再谈启用(§16.2/§16.4)。

**11 如何避免一次负面情绪影响长期人格与关系判断？** `EXISTING DESIGN`：多层防护——(a) 不写 `Event.content`/不改 `validation_state`(门禁 8)；(b) 不进 `PersonProfile`(`schemas/person-profile L55`、`12 §7 L401`)；(c) 不进 M02 排序(X-1)；(d) 不改 PAD/不落库(X-3)；(e) `intensity`/`confidence`/`decay_rate`+`max_influence` 裁剪；(f) 门禁 3「单事件不得持续主导」；(g) 禁沉默触发(K-6/CR-09)；(h) `11 §7.7 L277/L384`「不得由单次互动推断关系或情绪」。`DESIGN COMPLETENESS`：**「不影响长期人格」成立**，但靠「通道完全不存在」实现，不是「有隔离算法的有限接入」；`RELATIONSHIP` 语义类无「情绪不得作为关系判断输入」的显式禁令；一旦解锁 `influence_enabled`，当前没有「负面情绪不得跨轮累积成关系结论」的可验证判据（`13 §12/§13` 无此类测试）。`NEW PROPOSAL`：增 `emotion.negative_accumulation_guard`（跨轮累计上限+关系写入禁令）+验收项(§16.2)。

**12 情绪附件能否独立关闭、调整、拔除？** `EXISTING DESIGN`：三个**全局**开关 `influence_enabled`/`query_enabled`/`auto_infer` 在 `config/emotion L19–36` 均为 `const:false`+`default:false`，描述「启用等价于**重新裁决**，不是配置调优，任何覆盖 MUST 报 `CONFIGURATION_ERROR`」；`15 §7.8 L243–245` 标「**无人可写**（`const:false`）；启用须改 schema，等价重新裁决」；J5(`13 §5 L141`)「提供情绪标注清除能力；走 M04 修订通道和审计」。`DESIGN COMPLETENESS`：**不成立**——①「独立」是假的：三开关全局联动、无对象级 `runtime_enabled`/`detached`，不能对单条/单主体/单事件关闭；②「调整」是假的：`influence_enabled=false` 时数值参数「不参与计算」(L5) 且 15 声明未标定；③「拔除」`CONCEPT_ONLY`：J5 只有一句原则，无 API/字段/批量清除/缓存/索引/备份/tombstone 清理，删除范围还受 K-2/CR-06 `BLOCKED`；④无情绪状态机(机器校验 `emotion_related_sm=false`)。`NEW PROPOSAL`：对象级 `enabled`/`detached`/`status` + `memory.unlinkEmotion`/`purgeEmotion` + SM-9 + K-2 边界清单(§16.1/§16.2)。

**13 Emotion Runtime 未启用时 Memory Core 是否仍正常运行？** `EXISTING DESIGN`：**是，且这是设计得最扎实的一条**——`09 §7.15 L591`「情绪抽取失败 **MUST NOT** 阻塞事实单元提交（情绪为附属产物）；失败记 `emotion_extraction_failed`，`emotion_refs=[]`」；L593 条件可跳过并记 `skipped_reason=emotion_disabled`；`13 §7.12.1 L648–650` 默认只读 Event 事实；`14 E-14-09 L714` 关闭时 `FEATURE_DISABLED` 且「MUST NOT 返回空数组冒充无情绪」；`read-response.emotion_context=[]`+`emotion_tokens=0`+`trace.excluded[EMOTION_DISABLED]`；`13 §11 L884` M01 存储/M02 检索/M03 提取/M04 整合/M05 权限边界不依赖情绪；`02-clean ALG-01…ALG-10` 主链无情绪依赖。`DESIGN COMPLETENESS`：**COMPLETE（可独立降级）**。风险仅在：①跳过时须确保不误删已存在的 `USER_EXPLICIT` 情绪（无此类测试，`13 §12.2` 未覆盖）；②解锁三开关需改 `contracts/`，改动面横跨 `read-request`/`read-response`/`config/emotion`/15 号注册表，属 `CONTRACT_CHANGE`+`SCHEMA_MIGRATION` 而非 `CONFIG_ONLY`(`STANDARDS.md §6`)。`NEW PROPOSAL`：补「关闭态回归测试」与「解锁影响面清单」(§16.4)。

---

## 15. 核心问题清单（P0/P1，每条必答九问）

> 九问：①现在有什么 ②缺什么 ③为什么伤害核心能力 ④有无模块权责冲突 ⑤有无算法缺失 ⑥推荐方案 ⑦需补/改哪些文档 ⑧是否阻断 V0.9 ⑨能否在 V0.9 实验中解决。**P0=核心能力根本无法成立；P1=能跑但重要能力缺失或严重失真。** P2 见 §15 末表。

### C-C001 · P0 · 情绪回取与有限影响在机器契约层被 `const:false` + `maxItems:0` 硬封死
①`EXISTING DESIGN`：`config/emotion L19–36` 三开关 `influence_enabled`/`query_enabled`/`auto_infer` 全 `const:false`（启用须重新裁决，覆盖报 `CONFIGURATION_ERROR`）；`schemas/read-request L114–117 include_emotion const:false`；`schemas/read-response L309–318 emotion_context.maxItems:0`（V1 恒空）；`14 §7.2.7 getEmotionContext` 属 **P5 且默认关闭**；`00 §7.7 L405` Psychology **FUTURE/DEFERRED**，`13 §14 OI-029` 功能级 BLOCKED；`13 §7.11/§7.14`、`11 §7.19`、`12 §14`、`14 §14` 一致口径「设计定义，默认关闭，启用须重新裁决」(CR-02/03/04→OI-026/027/028)。②缺「可达读取路径」与「可启用影响路径」：无 `query_enabled=true` 的合法配置、无 `emotion_context` 非空响应、无 R6→R7→R11 组装算法、无 Psychology 运行时状态载体、无 `Context Builder` 的 emotion 分区。③产品 G5「回取情绪线索」与 G6「历史情绪有限影响」**直接不成立**；当前是「零影响」而非「有限影响」，系统能保存情绪却永远读不出来，`EmotionContext` 退化为只写不读的死数据，「记得你当时的感受」能力归零。④**有（软冲突）**：`13 §11 L884` 把情绪候选归阶段 11（`09 §7.15 L597` 属 M03）、存储归 M01；查询侧启用归 M02/M05(`15 §7.8 L244`)；ΔE 落点归 Psychology（独立模块，`00 §0.4 L46` 不得并入 M01–M05）且实施 BLOCKED。四模块各持一段，无一模块对「情绪回取端到端」负责，启用决策还被上移到用户裁决，不在任何模块运行职权内。⑤`MISSING`：A13(回取)、A14(合并)、A12/A15(状态载体)、A10(`R`)。⑥`RECOMMENDED ADDITION / NEW PROPOSAL`：**(a) 实验可做**——沙箱内定义「读取侧最小可达路径」：`query_enabled` 改为实验态 `feature_flag`（**不写回基线契约**）、`emotion_context.maxItems` 设 `emotion.max_attached`(EXP-DEFAULT 3)、`include_emotion` 放开并带 `intent=EMOTION_QUERY` 前置校验、`Context Builder` 增 emotion 分区且**排在事实/证据之后**、`emotion_tokens` 设硬上限(§16.3/§16.4)；**(b) 必须交父 Agent 走用户重新裁决**——CR-02/03/04+OI-026/027/028 启用、Psychology 阶段授权(CR-05/OI-029)、J6 独立验证，实验**不得**自行解锁。⑦`V2 §8/§9/§12` · `13 §7.11/§7.12/§14` · `14 §7.2.7/E-14-09` · `11 §7.15/§7.19` · `schemas/read-request` · `schemas/read-response` · `config/emotion` · `15 §7.8/§7.13` · `00 §7.7` · `08 §7.1` · `PENDING-USER-DECISIONS.md`/`OPEN-ISSUES.md`(OI-026/027/028/029)。⑧**是（能力层）**：V0.9 若以情绪回取/有限影响为验收目标，任何未改契约的配置都观测不到结果。⑨**部分能**：(a) 读取侧最小路径/合并算法/预算上限/回归测试可在 V0.9 实验内原型验证并产出证据；(b) 启用裁决、Psychology 阶段授权、J6 独立验证**不能**在实验内解决，须父 Agent 上交用户。按 `STANDARDS.md §5`「未启用」≠「不可实现」，故本条判 P0 是「能力当前归零」，不是「架构物理不可行」。

### C-C002 · P0 · 情绪写入链 W4–W6 全缺：草案→对象→关联→存储无可执行设计
①`EXISTING DESIGN`：W1–W3/W8 完整——`EvidenceBinder`(阶段 10，`13 §7.1.1 L201`)、`Event` 事实拆分、`EmotionExtractor`(阶段 11，M03，`09 §7.15 L574–597`，8 条 MUST+`EmotionDraft` 7 字段+幂等键+失败不阻塞事实)；`EmotionContext` 14 字段契约；`schemas/memory-unit emotion_refs/event_refs` 数组；`13 §11 L884` 模块边界表。②缺 W4 `EmotionDraft`→`EmotionContext` 映射（7 个必填字段无生成规则，`09 §14 OI-09-06` 自登记 `VERIFY`）；W5 关联表/FK/索引/基数/去重/冲突裁决；W6 `emotion_context` 表的 **DDL/列/索引/FK/版本/tombstone/同一 M01 事务**：`01` 头部 L3 写「SQLite 需新增 event / emotion_context / person_profile … 表」，但 `§7.16` 表 1–13 清单**无 `emotion_context` 表**（机器校验 `has_emotion_context_table=false`），FUTURE 表清单(L679)也无它。③「情绪作为独立附件存在」在**对象层**成立、**存储层**不成立：提取出的情绪线索没有合法写入目标，`EmotionContext` 只是 schema 里的形状，不是可持久化/可反查/可审计的记录；没有表就没有回取，G2/G3/G5 连锁失效。④**有**：`09 §7.15 L597` 说草案「所属模块：M03」；`13 §11 L884` 说「情绪候选归阶段 11」「M01 存储」；`01 §7.16` 建表清单不含该表；`schemas/memory-unit L173` 又把 `emotion_refs` 写进 MemoryUnit（暗示 M01 写 MemoryUnit 时一并维护）。**无任何文档指定「谁把草案落成一个 `emotion_id` 并写进哪张表」**，且 `13 §7.1.3 L227`「关联写入 MUST 与 MemoryUnit/Evidence 写入处于同一 M01 事务」未延伸到情绪。⑤`MISSING`：A4(草案→对象映射与落库)、A5(关联写入/反查)、A6(时间序列)；对照 `13 §7.1.3` 为 `memory_evidence_links` 给出的 6 条 MUST+主键+`INDEX(evidence_id)`，情绪关联是**零规范**。⑥`NEW PROPOSAL`：按 §16.1 增补最小字段 + §16.2/§16.3 的 `emotion_context` 表 DDL 与 `emotion_links` 关联表（含 `INDEX(target_id)`、`INDEX(event_id,created_at)`、FK、`version`、tombstone）；在 `09 §7.15` 与 `13 §7.1.3` 之间增「阶段 11.5 `EmotionBinder`（M03 产出→M01 事务落库）」并复用 `13 §7.1.3` 六条 MUST 模板；定义 `(candidate_draft_id,emotion_version,prompt_version)`→`emotion_id` 确定性生成规则。⑦`09 §7.15/§8.5/§14(OI-09-06)` · `13 §7.1/§7.7/§11` · `01 §7.16(表清单)` · `V2 §3.4/§5(16 阶段)` · `03(事务五步)` · `schemas/emotion-context` · `state-machines.md`(新增 SM-9)。⑧**是**：V0.9 若要验证「情绪能存、能关联到事件」，必须先有可执行写入链；当前只能停在草案。⑨**能**（属 `CONTRACT_CHANGE`+`SCHEMA_MIGRATION`，但全是增量补位，不动既有裁决）：映射规则/DDL/关联表/事务/幂等均可在 V0.9 实验内原型实现并用 `13 §12.2 T-13-24~31` 验证；**关联合约与 schema 的正式修改须交父 Agent 走变更流程**，实验只产出证据与建议稿。

### C-C003 · P1 · 事件—情绪关联模型不可执行：无类型化指针、无人物/互动、无时间序列、SEP-2 与契约矛盾
①`EXISTING DESIGN`：`event_id`(事件)、`related_memory_ids`(无类型 `memory_id` 数组)、`subject_id`(**情绪所属主体**非关联目标)、`MemoryUnit.emotion_refs/event_refs`；`13 §7.7.3 SEP-2/SEP-6`、L465 同事件双情绪示例；`11 §7.7 L377–379` 无具体事件持续情绪 `event_id=null`。②缺 `subject_ref_type`/`target_refs`/`related_person_ids`/`interaction_id`/`session_id`；缺 `observed_at`/`version`/`status`/`supersedes`/`thread_id`；缺 `(event_id,emotion_id)` 关联表与反查算法；`schemas/event` **无 `emotion_refs`**(SEP-2 声称有)。③G3「情绪可关联具体事件/人物/互动经历」只满足「事件」一半；「对某人的情绪」「某次互动的经历情绪」「同一事件情绪如何变化」三种真实长期互动场景无法表达，`getEmotionContext(event_id)` 即便解锁也无反查依据。④**有（文档级）**：`13 §7.7.3 L453 SEP-2` 与 `schemas/event L26–107` 直接矛盾；`subject_id` 在 `schemas/emotion-context L41` 由 `EntityResolver` 解析，而 `schemas/memory-unit L88` 同名字段语义为「记忆所属主体」，两处同名不同职责且无辨析条款（对比 `13 §7.6.3` 专门辨析两个 `source_type`）。⑤`MISSING`：A5(关联写入/反查)、A6(时间序列/变化)、A7(来源元数据)。⑥`NEW PROPOSAL`：`subject_ref_type ∈ {EVENT,MEMORY,PERSON,INTERACTION,NONE}` + `target_refs[]` + `related_person_ids[]` + `interaction_id`；`emotion_links` 关联表；`emotion_thread_id`/`version`/`supersedes`/`status`；在 `13 §7.7.3` 与 `schemas/event` 之间二选一统一（建议**不**给 Event 加 `emotion_refs`，改由关联表反查，保持 Event 纯事实）。⑦`schemas/emotion-context` · `schemas/event`(澄清即可) · `13 §7.7.3/§8` · `11 §7.7` · `V2 §3.4` · `01 §7.16`。⑧**否（最小链路可跑）**：仅靠 `event_id`+`related_memory_ids` 可先做「事件→情绪」最小验证；人物/互动/时间序列可后置。⑨**能**：最小关联模型+反查算法+时间序列表达可在 V0.9 实验内原型验证；正式 schema 修改交父 Agent。

### C-C004 · P1 · ΔE 参数三处冲突、`R` `MISSING`、`Δt` 时间源与多事件集合未定义
①`EXISTING DESIGN`：ΔE=I×C×R×D + 8 门禁(`13 §7.9/§7.10`/`V2 §9`)；`config/emotion` 四数值参数 + `gates` 五项 const 红线；`13 §7.9.2/§7.9.3` 两组算例（标「设计演算、非实测」）；`13 §13 AC-13-10` 要求算例与 §7.9 一致。②缺：①`R` 算法(`13 §7.9.1 L522` 自标 PROPOSED、`§14 OI-13-03 TBD`)；②`Δt` 取哪个时间（对象只有 `created_at` 无 `observed_at`；`event_id=null`⇒`D=0`）；③「多事件求和」取哪些事件(`top_k`/时间窗/`subject_id` 过滤/去重/`null` 处置全无)；④四默认值 **`13 §7.13.2` = 0.30/0.60/0.20/0.05** ≠ `config/emotion` = 0.6/0.5/0.15/0.1 ≠ `15 §7.8` = 0.6/0.5/0.15/0.1；⑤`gates.max_delta_per_turn` 默认 `null` 且标「TBD 标定」；⑥无情绪状态机。③公式形态 `COMPLETE` 却**不可判定实现正确**：`13 §7.9.2` 用 0.20 裁剪得 `ΔE=0.20`、`§7.9.3` 对照 `0.020082` 与总上限 `0.20`，若实现按 config/15 的 0.15 则 AC-13-10「算例与 §7.9 一致」必然 FAIL；`R`/`Δt`/集合缺失使门禁 2/4/5 无法执行，G6「有限影响」无法验证。④**有（参数权威）**：`13 §7.13.2 L714`「上表所有 EXP-DEFAULT MUST 在 15 号注册表登记 15 属性」，`config/emotion L5` 声明「参数身份以 `emotion.*` 点号 ID 为准」，`15 §7.8` 又声明「唯一 ID 来源 = 15 号注册表」，但三处**值不同**且 `15 §7.8 L527`「`emotion.*` 四个数值参数的标定值：未标定（CR-03 未通过前不得标定）」⇒ `CONFLICTED` 而非待调参。⑤`MISSING`：A10(`R`)、A11(集合选取)、A12(`Δt`)；`CONFLICTED`：A9(参数一致性)。⑥`NEW PROPOSAL`：(a) 以 `config/emotion` 为唯一机器真源，把 `13 §7.13.2` 与 §7.9 算例同步改为 0.6/0.5/0.15/0.1，**并重算两组算例与 AC-13-10 判据**（或反之，但须由父 Agent 裁决一处）；(b) 定义 `R` 最小算法（如 `R=|tokens(query)∩tokens(event_fact)|/|tokens(query)∪tokens(event_fact)|` 基线，升级路径 `cosine`），登记 `OI-13-03`；(c) `Δt = now − observed_at`（新增字段），`event_id=null` 显式排除；(d) 集合 = 当前 `subject_id`+时间窗+`status=ACTIVE`+按 `observed_at` 取 top-N。⑦`13 §7.9/§7.10/§7.13.2/§12.2/§13/§14` · `config/emotion` · `15 §7.8/§7.13/§7.17` · `V2 §9` · `schemas/emotion-context`(`observed_at`)。⑧**否（但会污染实验结论）**：ΔE 本身可在实验内跑通；不同步参数则算例与验收判据对不上，结论不可采信。⑨**能**：`R` 基线/`Δt`/集合选取/参数一致性方案与标定证据可在 V0.9 实验内产出；**正式改哪一处为准须父 Agent 裁决**。

### C-C005 · P1 · 情绪→Persona/Psychology 落点缺失，Context Builder 无 emotion 分区与合并算法
①`EXISTING DESIGN`：`00 §0.4 L46` Psychology「当前情绪状态与事件情绪关联的有限影响管理（独立模块，不得并入 M01–M05）」；`00 §7.7 L405–408` Psychology **FUTURE/DEFERRED**、Persona **FUTURE**(D1–D3 待 CR-08B 单独呈报)、`Context Adapter` **FUTURE**；门禁 8「ΔE 只作用于 Psychology 运行时状态」；`02-clean §7.7` ALG-06 `Context Builder` 有 `uncertain` 分区；`08 §7.1 L229` 注入链路；`schemas/person-profile L5/L55` 与 `12 §7 L401` 禁情绪进画像。②缺 `EmotionContext`/ΔE→`Persona Context` 的字段/接口/组装步骤/Prompt 段名/预算占比；`Context Builder` 无 emotion 分区；`08 §7.1 L229` 链路无 Psychology 节点；`Context Adapter`/`Persona` 均 FUTURE，Psychology BLOCKED(CR-05/OI-029，`13 §3.2 L91`、`§14 L1002`、`12 §14 L729`、`14 §14 L1072`)；`13 §7.12.3` 三种主动请求 PROPOSED 且均关闭。③「持续互动/稳定人格」中「有限情绪影响」无落点：情绪既不进画像（正确）也不进 Persona（缺失），ΔE 只能作用于一个 BLOCKED 的模块；产品要「有限影响+稳定人格」，当前是「零影响+无约束」——一旦绕过裁决把情绪塞进 Prompt，反而**没有**「不得凌驾」的可验证边界(G7 靠通道不存在实现)。④**有**：ΔE 落点 Psychology(独立模块) vs 上下文组装 `Context Builder`(M02/ALG-06) vs 注入 `Context Adapter`(宿主侧 FUTURE) vs `Persona`(FUTURE 消费者) 四者无接口契约；`13 §7.12.3` 三种主动请求由谁判定未指定；L669 还禁止「把当前对话看起来相关当作用户明确请求」，与场景 3「关联触发」自相牵制。⑤`MISSING`：A14(合并)、A15(落点)；`CONCEPT_ONLY`：三种主动请求编排。⑥`NEW PROPOSAL`：`Context Builder` 增四分区（事实/证据/画像/情绪），顺序固定「事实→证据→画像→情绪（可选，最后）」；emotion 段带 `source_type`+`temporal_position`+`intensity`+「系统推断」标记；设 `emotion.max_context_ratio`(EXP-DEFAULT 0.10) 且**不得挤占事实段预算**；`Psychology` 只提供只读 `RuntimeAffectSnapshot`（不写 Persona/PersonProfile/MemoryUnit）；`Context Adapter` 注入并回传 L3 事件(`08 §7.11`，不触发 reinforce，D-AP-05)。⑦`00 §0.4/§7.7` · `02 §7.7(ALG-06)` · `13 §7.12.3/§7.13` · `14 §7.2.7/§7.5` · `08 §7.1/§7.11` · `12 §7` · `V2 §8/§9` · `schemas/read-response`(emotion_context 形状)。⑧**否（可增量验证）**：合并算法与分区可在实验内原型验证；Psychology 阶段授权与 CR-08B 呈报不能在实验内解决。⑨**部分能**：分区/顺序/预算/合并与「不压过事实」判据可在 V0.9 实验内验证并产出证据；Psychology 解锁(CR-05/OI-029)与 Persona D1–D3(CR-08B)须父 Agent 上交用户。

### C-C006 · P1 · `EmotionContext` 数据完整性边界矛盾与元数据缺失
①`EXISTING DESIGN`：14 必填字段(§6 矩阵)；`allOf` 推断 `confidence≤0.99`；`not` 禁 `public`；15 标签+`raw_label` 越界保留；`evidence_refs`「无证据 MUST NOT 进入正式库」。②缺：①`intensity`/`confidence` 必填 number 不可 `null` vs `13 §7.8.2 L495–496`「未知 MUST NOT 默认 0.5，须留空/标 uncertain」⇒`CONFLICTED`；②`source_type` 仅 2 值缺 `SYSTEM`，而 L94 允许「system 侧汇总视图」、J8 预留 `affect_source='external'`，例外无合法表达；③无 `observed_at`/`recorded_at`/`version`/`status`/`runtime_enabled`/`decay_rate`；④无 `model_version`/`extraction_method`/`annotator`/`calibration_version`，`intensity=0.62`/`confidence=0.8` 不可审计；⑤`raw_label` >128 字符「须拆分或截断并记录」但无拆分算法。③「来源分离/置信度/强度」是 G6/G7 可信基座：值不可空使系统被迫编造数值（违反 `13 §7.9.1 L541`「MUST NOT 中性填充」同源精神），来源不可审计使「用户真实表达」与「模型推断」在数值层面无法复核，错误情绪会被当作既定事实累积。④**有（正文 vs 契约）**：`13 §7.8.2` 是 APPROVED 口径(`§1` 分节状态矩阵 L36)，`schemas/emotion-context` 是机器契约，二者对「未知」处理矛盾；`13 §7.7.2 L94` system 例外与 SEP-4（仅 2 值）矛盾；无文档指定以谁为准（对比 `13 §7.6.3` 对两个 `source_type` 有专门辨析）。⑤`PARTIAL`：A1/A2/A7；`MISSING`：`intensity` 标定算法、`raw_label` 拆分算法。⑥`NEW PROPOSAL`：`intensity`/`confidence` 允许 `null` 并增 `value_state ∈ {KNOWN,UNCERTAIN}`；`source_type` 保持 2 值裁决，另增 `origin_channel` 表达 SYSTEM_AGGREGATE/EXTERNAL_RESERVED；增 §6 缺失字段清单中元数据与对象级 `decay_rate`；`raw_label` 超长按句切分并记 `truncated=true`（禁硬切）。⑦`schemas/emotion-context` · `13 §7.7.2/§7.8.2/§8.3/§12.2/§13` · `09 §8.5` · `V2 §3.4/§2.9` · `15 §7.8`。⑧**否**：现有 14 字段足以支撑最小写入/读取验证。⑨**能**：字段增补与一致性修订可在 V0.9 实验内验证；正式 schema 修改交父 Agent。

### C-C007 · P1 · `MemoryUnit.content` 可合法承载情绪并进入 fts/vector RRF 共池（唯一事实污染路径）
①`EXISTING DESIGN`：`schemas/memory-unit L54` `semantic_type` 6 值**含 `EMOTION_CONTEXT`**、L102 `anchor` 8 值**含 `EMOTION_CONTEXT`**、L46–50 `content`「规范化后的独立记忆内容」**无情绪禁令**；`schemas/search-document L16` `content`「检索文本投影」亦无隔离；`11 §7.7 L353` 定义 `EMOTION_CONTEXT` 语义类（含「持续情绪」）；`config/retrieval L105–158` RRF 通道 `fts`+`vector`。反观 `schemas/event L37` 明确「MUST NOT 写入情绪判断」，`13 §7.7.3 SEP-2` 只禁「EmotionContext 作为 **Event** 内嵌字段」。②缺「`semantic_type=EMOTION_CONTEXT` 的 `MemoryUnit.content` 不得承载情绪判断」显式禁令；缺「情绪类 MemoryUnit 不得进入 `items[]` 事实排序池」过滤规则；缺 `read-response.items[]` 对 `semantic_type=EMOTION_CONTEXT` 的排除/独立承载约束。③这是**唯一**可能让 G1/G4/G7 失守的合法路径：若把「主人当时很焦虑」写成一条 `semantic_type=EMOTION_CONTEXT` 的 MemoryUnit 正文，它会走 FTS5+LanceDB→RRF→精排→`items[]` 与事实**共池竞争**，挤占 `budget_tokens`、压过事实、被 LLM 当作事实读走；门禁 8 / `gates.writes_persistent_state=false` 只管 ΔE，**管不到这条写入路径**。④**有**：M03(`EmotionExtractor` 产草案) 与 M01(写 MemoryUnit) 间无「情绪不得进 content」跨模块校验；SEP-1 约束 Event 未约束 MemoryUnit；`search-document` 投影由 M02/Outbox 生成(`01 §7.16` 表 10 `index_registry`)，无情绪过滤位。⑤`MISSING`：情绪类文档在索引构建与 `items[]` 组装时的隔离过滤器；`CONCEPT_ONLY`：`emotion_context` 独立承载（有字段但 `maxItems:0`）。⑥`NEW PROPOSAL`：在 `schemas/memory-unit` 的 `content.description` 增「`semantic_type=EMOTION_CONTEXT` 时 MUST NOT 承载情绪判断，情绪 MUST 存于独立 `EmotionContext`」；`search-document` 对情绪类文档设 `indexable=false`；`read-response.items[]` 的 `allOf` 增「`semantic_type=EMOTION_CONTEXT` 的条目 MUST 走 `emotion_context[]` 而非 `items[]`」；补测试项「情绪正文进入 `items[]` 次数 = 0」。⑦`schemas/memory-unit` · `schemas/search-document` · `schemas/read-response` · `13 §7.7.3/§7.12/§12.2/§13` · `11 §7.7` · `02 §7.5` · `01 §7.16`。⑧**否**（但实验必须加该断言，否则「事实不被污染」验收结论不可信）。⑨**能**：契约补丁+隔离过滤+回归测试可在 V0.9 实验内完成并产出证据；正式 schema 修改交父 Agent。

### C-C008 · P1 · 情绪附件无法独立关闭/调整/拔除（全局 `const:false` ≠ 可治理能力）
①`EXISTING DESIGN`：`config/emotion L5/L19–36` 三个**全局** `const:false` 开关，说明「启用等价于重新裁决，不是配置调优」；`15 §7.8 L243–245`「无人可写；启用须改 schema」；J5(`13 §5 L141`)「提供情绪标注清除能力；走 M04 修订通道和审计；删除范围服从 K-2」；`gates.writes_persistent_state=false`/`silence_trigger_forbidden=true`/`explicit_weight=1.0` 等 const 红线。②缺对象级 `enabled`/`detached`/`status`；单条/单主体/单事件关闭能力；运行时可调参数(`influence_enabled=false` 时数值参数「不参与计算」L5)；`memory.unlinkEmotion`/`purgeEmotion` API；清除的缓存/索引/备份/tombstone 清理规程；情绪状态机(机器校验 `emotion_related_sm=false`，SM-1~8 无情绪态)；K-2/CR-06 删除边界仍 `BLOCKED`(`13 §14 L1003`)。③13 问 12 直接不成立。「零影响」看似满足「不得凌驾」，实为**不可配置**：用户无法只要「回取情绪线索」而不要「ΔE 影响」，也无法拔除某条错误情绪；一旦整体解锁，缺少细粒度熔断会使错误情绪全量进入交互，与「稳定人格/用户真实表达优先」冲突。J5 清除能力是台账已接受的产品级承诺，当前只有一句原则。④**有**：J5 指定「走 M04 修订通道」(`13 §5 L141`/`09 §7.22`)，但 D-AP-05 规定 `confirmed` 仅 M04 通道、删除传播按 `01 §7.16`「7 步」由 M01+Outbox 执行(`13 §7.4.6 DEL-1~7`)，K-2 又要求缓存/备份/导出边界单独裁决——清除动作横跨 M04/M01/M05/Outbox/缓存，无协调者与事务边界；`09 §7.22` 只说「清除回执走 M04，Pipeline 不直接删」。⑤`CONCEPT_ONLY`：A16(清除/拔除)；`MISSING`：A17(情绪状态机)、对象级开关语义、清除传播算法。⑥`NEW PROPOSAL`：`EmotionContext` 增 `enabled:boolean(default true)`/`detached:boolean`/`status:enum(ACTIVE/SUPERSEDED/ARCHIVED/DELETED)`；新增 SM-9 `Emotion Attachment`（初始 ACTIVE，ACTIVE→ARCHIVED/DELETED 需审计+tombstone，DELETED 不可逆）；API `memory.unlinkEmotion(emotion_id)`（软解除关联）与 `memory.purgeEmotion(emotion_id,scope)`（走 M04+M01 同事务+Outbox+缓存失效+K-2 边界清单）；三开关保留为「全局总闸」，另设对象级开关；标定值运行时可调（不必改 schema）。⑦`config/emotion` · `schemas/emotion-context` · `state-machines.md`(SM-9) · `13 §5/§7.4.6/§7.10/§14` · `09 §7.22` · `14 §7(9 接口外新增)` · `04-权限与治理规范.md` · `15 §7.8/§7.13` · `OPEN-ISSUES.md`(K-2/OI-030)。⑧**否**：关闭态本身可运行；拔除能力可作为实验项后置。⑨**部分能**：对象级开关、SM-9、清除 API 与传播算法可在 V0.9 实验内原型验证；K-2/CR-06 删除/导出边界裁决、三开关启用裁决**不能**在实验内解决，须父 Agent 上交用户。

### C-C009 · P1 · 情绪读取侧编排 `CONCEPT_ONLY`：`getEmotionContext` 无反查/排序/去重/合并算法
①`EXISTING DESIGN`：`14 §7.2.7 L439–471` 完整定义 `getEmotionContext(event_id|memory_id)` 输入（2 选 1+`request_id`）、输出（`emotion_contexts[]`/`emotion_enabled`/`items[]` 9 字段/`emotion_tokens`/`decision_id`）、`read_mode`(summary/memory)、M05 单独判权、幂等键、7 错误码含 `FEATURE_DISABLED`；`13 §7.12.3` 三种主动请求；`13 §8.5 L761–770`；`read-response` 有 `emotion_context[]`/`emotion_tokens`/`EMOTION_DISABLED`。②缺「给定 `event_id` 反查全部 `emotion_id`」的算法/索引/关联表；无多条情绪的排序（按 `created_at`？`intensity`？`temporal_position`？）、去重、冲突裁决、`maxItems`；无「`emotion_contexts[]` 与 `items[]` 如何拼进 Prompt」的段名/顺序/预算；无 `emotion_tokens` 启用后上限；无「用户明确问情绪」的意图判定规则(`13 §7.12.3 L669` 禁止推测意图，与场景 3「关联触发」牵制)。③即使 C-C001 开关解锁，R6/R7/R11 仍无算法：接口返回什么、返回几条、排第几位、与事实如何共存全靠实现者猜 → G5「重新取回情绪线索」与 G7「不得凌驾事实」不可验证。④**有**：`14 §7.2.7` 属 P5(M05+聊天 LLM 接入)，`13 §7.12` 把路由归 `V2 §8`「EVENT→证据→关联人物→（按需）情绪」，`11 §7.15` 把 `EMOTION_QUERY` 归 Intent Resolver/Anchor Router(M02)，`Context Builder`(ALG-06) 归 M02 上下文组装——三段无人合并；且 `Context Adapter` FUTURE(`00 §7.7 L408`)。⑤`MISSING`：A13(回取)、A14(合并)、A11(集合/排序)；`CONCEPT_ONLY`：三种主动请求编排(`13 §7.12.3` 自标 PROPOSED)。⑥`NEW PROPOSAL`：定义 `EmotionRetrieval` 六步（①判 `query_enabled` ②M05 单独判权 ③按 `emotion_links.target_id` 反查 ④按 `status=ACTIVE`+`source_type` 分层+`created_at DESC` 排序 ⑤按 `(subject_id,emotion_label,temporal_position)` 去重 ⑥受 `emotion.max_attached` 与 `emotion_tokens` 裁剪）；合并规则见 C-C005；补 T-13-48~52 测试与 AC-13-15。⑦`14 §7.2.7/§7.5/§7.11` · `13 §7.12/§8.5/§12.2/§13` · `11 §7.15/§7.19` · `02 §7.7` · `V2 §8` · `schemas/read-response` · `config/retrieval`(`emotion.max_attached`)。⑧**否**（但 C-C001 解锁后本条即成首要阻断）。⑨**能**：反查/排序/去重/合并/裁剪与测试可在 V0.9 实验内原型验证并产出证据；接口与契约正式修改交父 Agent。

### P2（简要归档，不作主结论）

| ID | 问题 | 归档 |
|---|---|---|
| **C-C010** | 越界情绪一律映射 `neutral` 会丢失原情绪语义 | `schemas/emotion-context L62`+`13 §7.8.1`：保留 `raw_label` 已缓解；后续可对 `raw_label` 二次归一，属调优 |
| **C-C011** | 词表内容 `OI-009 TBD`、`raw_label >128` 拆分算法缺、`emotion-15-v1` 配置版本未落 | `13 §7.13.2 L710`/`§14 L1010`/`config/emotion label_dict`；属实施设计阶段(J4)与调优，不升 P1 |

---

## 16. ⑨ 推荐联动方案（`NEW PROPOSAL` — 未获批准，不得写成已有设计）

> **以下全部为本路建议，不是 A 集合正式设计，也不是已裁决方案。** 所有新增字段/表/接口须交父 Agent 走变更流程；V0.9 实验只在沙箱内原型验证并产出证据。

### 16.1 最小 schema 增补（`schemas/emotion-context`）

| 新增字段 | 类型/枚举 | 默认 | 必填 | 解决 |
|---|---|---|---|---|
| `subject_ref_type` | `EVENT`/`MEMORY`/`PERSON`/`INTERACTION`/`NONE` | `NONE` | ✓ | 问 1/2；C-C003 |
| `target_refs` | array\<uuid\>（配合类型化） | `[]` | ✓ | 问 2；C-C003 |
| `related_person_ids` | array\<subject_id\> | `[]` | 否 | 「对某人的情绪」；C-C003 |
| `interaction_id` | uuid\|null | null | 否 | 「某次互动经历」；C-C003 |
| `observed_at` / `recorded_at` | date-time\|null / date-time | null / =created_at | ✓ | 问 5/8、`Δt` 时间源；C-C004 |
| `emotion_thread_id` / `version` / `status` / `supersedes` | uuid / int≥1 / 4 值 / uuid\|null | 新生成 / 1 / ACTIVE / null | ✓ | 问 4/5/12；C-C003/C-C008 |
| `decay_rate`（对象级） | number>0\|null | null(继承全局) | 否 | §3 第 5 项；C-C004 |
| `enabled` / `detached` | boolean | true / false | ✓ | 对象级独立关闭/拔除；问 12；C-C008 |
| `value_state` | `KNOWN`/`UNCERTAIN` | `KNOWN` | ✓ | 强度/置信度未知态；C-C006 |
| `origin_channel` | `USER_EXPLICIT`/`ASSISTANT_INFERRED`/`SYSTEM_AGGREGATE`/`EXTERNAL_RESERVED` | 由 `source_type` 映射 | ✓ | `SYSTEM` 例外合法落点（**不改 `source_type` 2 值裁决**）；§3 第 2 项 |
| `model_version`/`extraction_method`/`annotator`/`calibration_version` | string\|null | null | 否 | 置信度/强度可审计；C-C006 |

**同时修订（均 `NEW PROPOSAL`）**：`intensity`/`confidence` 的 `type` 改 `["number","null"]` 并与 `value_state=UNCERTAIN` 联动（C-C006）；`schemas/memory-unit` 的 `content.description` 增「`semantic_type=EMOTION_CONTEXT` 时 MUST NOT 承载情绪判断」（C-C007）；`schemas/event` **不新增** `emotion_refs`，改在 `13 §7.7.3 SEP-2` 统一为「经 `emotion_links` 反查」，保持 Event 纯事实（C-C003 的 `CONFLICTED`）；`state-machines.md` 新增 **SM-9 Emotion Attachment**（初始 ACTIVE；ACTIVE→ARCHIVED/DELETED 需审计+tombstone；DELETED 不可逆；与 M01 同事务）；`config/emotion` 增 `emotion.max_attached`(默认 3)、`emotion.max_context_ratio`(默认 0.10)、`emotion.long_term_write_forbidden=true`(const 红线)；**三开关 `const:false` 保留不动**（启用须用户重新裁决，§16.4）。

### 16.2 算法补位（`NEW PROPOSAL`）

| 算法 | 最小可执行定义 |
|---|---|
| A10 `R` | 基线 `R=|tokens(query)∩tokens(event_fact)|/|tokens(query)∪tokens(event_fact)|` 经 min-max 归一；升级 `cosine(query_vec,event_vec)`；登记 `OI-13-03`，`relevance_threshold` 统一取 `config` 值 |
| A11 集合 | 当前 `subject_id`+`status=ACTIVE`+`event_id!=null`+`observed_at` 在 `emotion.lookback_days` 内+按 `observed_at DESC` 取 `emotion.max_events`(默认 5)；同 `emotion_thread_id` 只取 `version` 最大者 |
| A12 `D` | `Δt_days=(now−observed_at)/86400`；`D=exp(−coalesce(object.decay_rate, emotion.decay_rate)×Δt_days)`；`observed_at=null`⇒`D=0` |
| A4 映射 | 见 §16.3 写入伪代码：`EmotionDraft` 7 字段+生成 7 个缺失必填+关联表落库 |
| A5 关联 | `emotion_links(emotion_id,target_type,target_id,link_kind,created_at)`，主键 `(emotion_id,target_type,target_id)`，`INDEX(target_id)`、`INDEX(target_type,target_id,created_at)`，FK 双向；写入与 `EmotionContext`/`MemoryUnit` 同 M01 事务（复用 `13 §7.1.3` 六条 MUST 模板） |
| A13 回取 | 见 §16.3 读取伪代码（六步） |
| A14 合并 | `Context Builder` 四分区，固定顺序「事实→证据→画像→情绪（可选，最后）」；emotion 段带来源/阶段/强度标记；`emotion_tokens ≤ min(emotion.max_context_ratio×budget_tokens, emotion.max_attached×per_item_cap)`；**不得挤占**事实段已分配预算 |
| A16 清除 | `memory.unlinkEmotion`（置 `detached=true`+`status=ARCHIVED`，软）与 `memory.purgeEmotion`（M04 修订通道→M01 同事务→Outbox delete→索引/缓存失效→tombstone→K-2 边界清单） |
| A9 参数 | **由父 Agent 裁决**一处为准；本路建议以 `config/emotion` 为机器真源，同步 `13 §7.13.2`/§7.9 算例/AC-13-10 |

### 16.3 写入 / 读取伪代码（`NEW PROPOSAL`）

```
# 写入：阶段 11.5 EmotionBinder（M03 产出 → M01 事务落库）
FUNCTION bindEmotion(draft: EmotionDraft, ctx) -> EmotionContext | SKIP
  IF emotion.auto_infer==false AND draft.source_type=="ASSISTANT_INFERRED": SKIP("emotion_disabled_inferred")
  IF draft.evidence_refs IS EMPTY: SKIP("no_evidence")                 # 13 §7.7.2 L94
  ec.emotion_id=uuid(); ec.emotion_label=mapToControlled15(draft.emotion_label)  # 越界->neutral
  ec.raw_label=draft.raw_label                                          # 保留原词, 不丢弃
  ec.intensity=draft.intensity; ec.confidence=draft.confidence          # 推断侧 <=0.99
  ec.temporal_position=draft.temporal_position; ec.source_type=draft.source_type  # 仅2值, 不合并
  ec.evidence_refs=draft.evidence_refs; ec.subject_id=ctx.subject_id
  ec.event_id=ctx.event_id_or_null; ec.related_memory_ids=ctx.memory_ids
  ec.subject_ref_type=pick(ctx); ec.target_refs=typedRefs(ctx, ec.subject_ref_type)
  ec.related_person_ids=ctx.resolved_person_ids; ec.interaction_id=ctx.interaction_id
  ec.observed_at=ctx.observed_at_or_null; ec.recorded_at=now(); ec.created_at=now()
  ec.emotion_thread_id=findOrCreateThread(ec.subject_id, ec.event_id, ec.emotion_label)
  ec.version=nextVersion(ec.emotion_thread_id); ec.supersedes=latestActive(ec.emotion_thread_id)
  ec.status=ACTIVE; ec.visibility=strictestIntersect(ctx); ec.scope=strictestIntersect(ctx)  # INV-08
  ec.enabled=true; ec.detached=false; ec.origin_channel=mapOrigin(ec.source_type)
  ec.model_version=ctx.model_version; ec.extraction_method="EmotionExtractor"; ec.decay_rate=null
  BEGIN TX                                                             # 复用 13 §7.1.3 事务纪律
    INSERT emotion_context(ec)                                         # 新增表, 01 §7.16 待补
    INSERT emotion_links(ec.emotion_id, ec.subject_ref_type, each target_ref, "attached")
    UPDATE memory_unit SET emotion_refs=array_append(emotion_refs, ec.emotion_id)
           WHERE memory_id IN ec.related_memory_ids                     # 不写 content
    # Event 表不写任何情绪字段；SEP-1/SEP-7 保持
    WRITE provenance(operation="emotion_bind", actor="M03/M01", request_id=ctx.request_id)
  COMMIT                                                               # 失败整体回滚
  RETURN ec                                                            # 失败不阻塞事实单元(09 §7.15 L591)

# 读取：memory.getEmotionContext 六步 + 合并
FUNCTION getEmotionContext(event_id | memory_id, request) -> EmotionReadResult
  IF emotion.query_enabled==false:                                      # 当前 const:false；实验态 feature_flag
      RETURN {status:"FEATURE_DISABLED", reason_code:"J6_FIRST_RELEASE_DISABLED",
              emotion_contexts:[], emotion_tokens:0}                    # 14 E-14-09, 不冒充"无情绪"
  decision=M05.canRead(requester, target, scope="emotion")              # SEP-6: 事件可读 != 情绪可读
  IF decision!=ALLOW: RETURN PERMISSION_DENIED                         # INDETERMINATE 一律排除
  ids=SELECT emotion_id FROM emotion_links
      WHERE (target_type,target_id)=(EVENT|MEMORY, event_id|memory_id) AND link_kind="attached"
  rows=SELECT * FROM emotion_context WHERE emotion_id IN ids
       AND status=ACTIVE AND enabled=true AND detached=false
  rows=dedupBy(rows, key=(subject_id,emotion_label,temporal_position,source_type))
  rows=SORT(rows, BY observed_at DESC, version DESC)                    # 最新情绪在前
  rows=TAKE(rows, emotion.max_attached)                                 # 默认 3
  cap=MIN(emotion.max_context_ratio*request.budget_tokens, emotion.max_attached*per_item_cap)
  rows=TRUNCATE_BY_TOKEN(rows, cap, truncated=true, no_hard_cut=true)
  result.emotion_contexts=PROJECT(rows, ["emotion_label","raw_label","intensity","confidence",
        "temporal_position","source_type","origin_channel","observed_at"])
  result.items=FACT_ITEMS_ONLY                                          # 与 items[] 不共池
  result.emotion_tokens=countTokens(result.emotion_contexts); result.decision_id=decision.id
  trace.excluded+=EMOTION_DISABLED / PERMISSION_DENIED reasons
  RETURN result
  # 不调用 LLM 现编情绪(J2)；不由沉默触发(K-6/CR-09)；不写 reinforce(D-AP-05)
```

### 16.4 解锁前置条件（`NEW PROPOSAL` — **必须交父 Agent / 用户裁决，实验不得自行解决**）

| # | 前置 | 依据 | 归属 |
|---|---|---|---|
| U1 | CR-02/OI-026 重新裁决(`query_enabled`) | J6「首版关闭查询侧情绪感知，开启须独立验证和重新裁决」 | 用户 |
| U2 | CR-03/OI-027 重新裁决(`influence_enabled`)+X-3 审核链+I-4 影子验证 | `13 §7.11`；`15 §7.8` | 用户 |
| U3 | CR-04/OI-028 重新裁决(`auto_infer`)+标注一致性 κ≥0.6(I-8) | J7 | 用户 |
| U4 | CR-05/OI-029 Psychology 阶段授权（psy-model 暂缓 I-7/X-6） | `13 §3.2 L91`/`§14 L1002` | 用户 |
| U5 | Persona D1–D3 / CR-08B 单独呈报 | `08 §7.2 L78`；`00 §7.7 L406` | 用户 |
| U6 | K-2/CR-06/OI-030 删除与导出边界 | `13 §14 L1003` | 用户 |
| U7 | 参数真源裁决（13 vs config vs 15） | §10.2 / C-C004 | 父 Agent |
| U8 | `contracts/` 变更：`read-request`/`read-response`/`emotion`/`emotion-context`/15 号注册表 | `CONTRACT_CHANGE`+`SCHEMA_MIGRATION` | 父 Agent 变更流程 |

**V0.9 实验可自行完成**（只产出证据，不改基线）：§16.1 字段增补原型、§16.2 全部算法原型、§16.3 两段伪代码、C-C007 隔离断言、T-13-48~52 / AC-13-15 补测、关闭态回归测试、参数一致性对照报告。

---

## 17. ⑩ 数据流图（ASCII）

> 实线=当前 `EXISTING DESIGN` 可达；虚线=设计定义但契约封死 / FUTURE / BLOCKED；`[NEW]`=`NEW PROPOSAL`；`⚠`=唯一可能让情绪与事实共池的合法路径(C-C007)。

```
 用户输入（原话："答辩通过了，当时特别紧张，现在总算松了口气"）
        │
        ▼
 Evidence (original_text / content_hash / speaker) · EvidenceBinder · 阶段10 · 13 §7.1
        ├──────────── 事实 ────────────┬──────────── 情绪线索 ──────────────┐
        ▼                             │                                    ▼
 Event (content=仅事实)              │        EmotionExtractor (阶段11, M03) → EmotionDraft[](7字段)
 event_id/time_*/participants/        │        source_type: 仅 USER_EXPLICIT 可产; auto_infer=const:false
 result; 15字段, 无 emotion_refs      │                                    │ W4 [NEW] 缺映射
        │ related_memory_ids          │                                    ▼
        ▼                             │        [NEW] EmotionBinder(阶段11.5) 补7必填/同M01事务
 MemoryUnit (28字段) ◄───────────────┘        → emotion_context 表 + emotion_links 关联表
 emotion_refs[] / event_refs[]                                          │
 semantic_type 6值含 EMOTION_CONTEXT ⚠                                 ▼
 content 无情绪禁令 ⚠                          EmotionContext (14必填): event_id / related_memory_ids /
        │                                      subject_id / emotion_label(15) / intensity / confidence /
        │                                      temporal_position / source_type / evidence_refs / created_at
        │                                      [NEW] subject_ref_type / observed_at / version / status /
        │                                             enabled / detached / decay_rate
        │                                             │            │ 门禁8红线: 不写Event/不改
        ├── 事实检索池 ───────────┐                  │            │ validation_state/不进M02排序
        │  (fts + vector RRF)     │                  │            ▼
        │  config/retrieval L105-158                │      ΔE = I×C×R×D + 8门禁
        │  精排五特征, 无emotion权重│                  │      I←intensity C←confidence
        │  → read-response.items[] │                  │      R←TBD(OI-13-03) D←exp(-rate·Δt), Δt无源
        │                         │                  │      influence_enabled=false
        │  ⚠ C-C007: 若情绪写进     │                  │      [NEW] R/集合/Δt补位 · 参数三处冲突待裁决
        │    MemoryUnit.content,     │                  │            ┆ 仅运行时,不落库
        │    会走同一RRF池压过事实   │                  │            ▼
        │                         │                  │      Psychology (FUTURE/DEFERRED/BLOCKED,
        │                         │                  │        CR-05 / OI-029 / I-7/X-6)
        │                         │                  │            ┆ 无接口
        │                         │                  │            ▼
        │                         │                  │      Persona Context (FUTURE, 00 §7.7 L406/CR-08B)
        │                         │                  │      Context Builder 无 emotion 分区 [NEW] 四分区合并
        │                         │                  ┆ 回取路径(契约封死)
        │                         │        ┌─────────┴───────────────────────────────────┐
        │                         │        │ memory.getEmotionContext(event_id|memory_id)│
        │                         │        │ 14 §7.2.7 · P5 · query_enabled=const:false  │
        │                         │        │ include_emotion=const:false                  │
        │                         │        │ emotion_context.maxItems=0                   │
        │                         │        │ ⇒ FEATURE_DISABLED / EMOTION_DISABLED       │
        │                         │        └─────────┬───────────────────────────────────┘
        │                         │                  ┆ [NEW] 六步 EmotionRetrieval · 情绪段排事实之后
        ▼                         ▼                  ▼
 Emotion Retrieval → 当前交互 (Current Interaction)
 read-response: items[](事实) + emotion_context[](恒空) + evidence[] + profile + emotion_tokens=0
 Context Adapter (FUTURE) 注入宿主 → Chat LLM

 PersonProfile (M04派生视图, 11分区, basis三值): schemas/person-profile L55 + 12 §7 L401 主动排除情绪 ⇒ 隔离
 ─── 事实链(实线, 可达且防护严密) ───   ┄┄┄ 情绪链(虚线, 设计定义/契约封死/FUTURE/BLOCKED) ┄┄┄
```

**图例**：`EMOTION_CONTEXT` 作为 `semantic_type`/`anchor` 值是既存枚举，但 `items[]` 与 `emotion_context[]` 是**不同承载字段**；`Context Builder`(ALG-06) 当前只有 `uncertain` 分区，情绪分区为 `[NEW]`。

---

## 18. 与另两路的边界（不重做）

- **A 路**：不重审 `memory_tier`/`TierTransition`/L1 存储/`original_text` 回溯。仅指出情绪对象自身**无 `memory_tier`/`status`/保留策略**，`11 §7.7 L179` 说 `EMOTION_CONTEXT` 随事件 `L1/L2`、L506「情绪不作长期稳定属性」是设计立场但无 tier 转换/保留落地 —— 不重复立 P0，归 C-C003 的 `version/status` 增补一并解决。
- **B 路**：B 路「跨对象关系是无类型 ID 数组、无自动跨区块联合召回」在本路独立印证：`related_memory_ids` 正是无类型 ID 数组，`getEmotionContext` 是手动 1-hop、无深度/预算/visited 规范(C-C003/C-C009)。B-C001 不重做。
- **D 路（Persona Prompt Flow）**：§11 与 C-C005 给出「`EmotionContext` 无任何进入 Persona Context 的路径」确证，交接作「情绪层未进入 per-turn 组装」输入；不代 D 路判断首次初始化与逐轮组装。

---

## 19. 交接（Handoffs）

| # | 对象 | 交接 |
|---|---|---|
| H1 | **父 Agent / Lead** | C-C001 是能力层 P0：三开关 `const:false`+`emotion_context.maxItems:0` 使「情绪回取」与「有限影响」归零；启用须走 U1–U8 用户重新裁决（CR-02/03/04+OI-026/027/028、CR-05/OI-029、CR-08B、K-2），**V0.9 实验不得自行解锁** |
| H2 | **父 Agent / 变更流程** | C-C004 参数三处冲突（`13 §7.13.2` vs `config/emotion` vs `15 §7.8`）需裁决唯一真源并重算 `13 §7.9` 算例/AC-13-10，否则 V0.9 算例验收结论不可采信 |
| H3 | **D 路** | 确证：`schemas/person-profile L55` 禁情绪入画像；`Context Builder`(ALG-06) 无 emotion 分区；`08 §7.1 L229` 注入链路无 Psychology 节点；Persona/Context Adapter FUTURE、Psychology BLOCKED。可据此把「情绪层未进入 per-turn 组装」列为既定事实 |
| H4 | **B 路** | `related_memory_ids` 无类型 ID 数组、`getEmotionContext` 手动 1-hop 无深度/预算/visited 规范，独立印证 B 路结论；建议 `emotion_links` 采用带 `target_type` 关联表以避免重蹈 |
| H5 | **A 路 / M01** | `01 §7.16` 表 1–13 无 `emotion_context` 表（头部 L3 仅「需新增」）；情绪对象无 `version`/`status`/保留策略，与 A 路「无分层体系」同源；本路 DDL 属 `SCHEMA_MIGRATION`，需与 A 路事务/Outbox/tombstone 纪律对齐 |
| H6 | **V0.9 实验** | 可自行原型验证：§16.1 最小字段增补、§16.2 A4/A5/A10~A14/A16、§16.3 两段伪代码、C-C007 隔离断言、T-13-48~52 / AC-13-15 补测、关闭态回归测试；产出证据但不改基线契约 |

---

## 20. 方法附录（§7 强制）

**20.1 读取清单**：`13` 全文（1–1034，分 5 块）；`schemas/emotion-context`(153)、`evidence`、`event`、`memory-unit`、`person-profile`、`search-document`、`read-request`、`read-response`（`utf-8-sig` 兼容 BOM，未改基线）；`config/emotion`、`retrieval`、`profile`；`00`(grep+L40–69/398–411)、`01`(grep+L664–687)、`EVIDENCE/B/02-clean.md`(L1–60+grep 16 命中)、`09`(grep 43+§7.15/§8.5)、`11`(grep 74)、`12`(grep 28)、`14`(grep 71+L439–483)、`15`(grep 28)、`V2`(grep 40+L240–269/L368–401)、`08`(grep 42)、`03`(grep 2)、`state-machines.md` 全文 165 行+grep(0 命中情绪)、`AUDIT-FORENSIC-2026/09`(grep+FC-001 段)、`_shared/STANDARDS.md`、`04-PERSONA-PROMPT-FLOW.md` 头部。

**20.2 命令/脚本**：`pwsh` 运行 `$env:PYTHONPATH='…\pydeps'; python EVIDENCE\c\c-machine-checks.py` → 生成 `EVIDENCE/c/c-machine-checks.json`（6560 bytes，首字节 `123,10,32` = `{`/`LF`/空格，**UTF-8 无 BOM**），`jsonschema_self_check.valid=true`(4.26.0)；二次 `pwsh` 用 `[System.IO.File]::ReadAllBytes` + `python -c json.load` 确认无 BOM / 可解析 / `parameter_default_conflict.verdict=CONFLICTED`。脚本盘点：字段 `field_presence_checks`、三开关 const、`include_emotion`、`emotion_context.maxItems=0`、`EMOTION_DISABLED` 枚举、`event_has_emotion_refs=false`、参数三处冲突、`state_machines.emotion_related_sm=false`、`emotion_draft_gap.missing_vs_emotion_context_required`(7 项)、`ddl.has_emotion_context_table=false`、Draft7 自检。

**20.3 自检**：全文每条结论带三分类；§14 的 13 问、§15 的 9 个 P0/P1 问题逐条九问实答，无「只报问题不给方案」条目；P0=C-C001/C-C002(2 条)，P1=C-C003…C-C009(7 条)，P2=C-C010/C-C011(仅末表归档)。术语原样使用 `MemoryUnit`·`Evidence`·`Event`·`EmotionContext`·`PersonProfile`·`memory_tier`·`status`·`validation_state`·`semantic_type`·`semantic_tags`·`anchor`·`visibility`·`scope`·`RRF`·`M01…M05`·`Ingestion Pipeline`(16 阶段)·`Context Builder`/`Context Adapter`·`Psychology`·`Persona`·`Dream`；`source_type` 按语料原名 `ASSISTANT_INFERRED`（未自造 `LLM_INFERRED`），题面 `LLM_INFERRED`/`SYSTEM` 在 §3 显式对齐。未改基线任何文件；未写 V0.9 产品代码；未改 schema/DDL/裁决台账；未把 `NOT RUN`/未标定当架构不可行；未把「有 schema/功能名」当算法完整。
