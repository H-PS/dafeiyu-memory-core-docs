# 07 — 重大架构冲突与整体产品能力复核（G 路）

## STATUS: COMPLETE

**轮次**：MC-CORE-CAPABILITY-REVIEW-01 · 只读能力审查 · 2026-11-06 · Asia/Shanghai
**本路范围**：G 路 = 严重架构冲突 + 整体产品能力复核（目标五「信息读取与利用」独立成章 + 全局冲突判定）
**基线**：`C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design`（A 集合，00–15 + `contracts/`）；B 集合 `E:\[PROJECT]\R4.1-UNIFIED-MEMORY-CORE` 为竞争候选；CMB 为相邻项目；R3 为历史来源。**本路未修改基线任何字节，未写产品代码，未改 schema/DDL/裁决台账。**
**证据**：`E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\EVIDENCE\g\`（`g-probe.py` / `g-probe2.py` / `schema-census.txt` / `read-chain-probe.txt` / `anchor-set.txt` / `retrieval-config.txt` / `validate-checks.json` / `validate-checks2.json`）

---

## 0. 阅读指引

### 0.1 本路回答什么、不回答什么

| 回答 | 不回答（已在前序路完成，禁重做） |
|---|---|
| 49 个 FC 中哪些**仍然**影响七项核心能力，其余如何归类 | 短期/长期分层的 16 条转换规则（A 路 `01-MEMORY-TIER-AND-LIFECYCLE-MATRIX.md`） |
| M01–M05 的**权责冲突**与关键数据的**合法写入者** | 长期记忆分区结构是否 HYBRID（B 路 `02-LONG-TERM-MEMORY-STRUCTURE.md`） |
| **目标五**：读取链的每一步是否可执行、契约是否违反「锚点优先」 | 13 个 FC 的逐条取证细节（`AUDIT-FORENSIC-2026\09-CONFLICT-REGISTER.md`） |
| 模块接口阻断、数据模型承载能力、算法闭环、整体评级 | 安全审计（按 brief 只保留会造成核心失效/数据丢失/跨主体混淆的条目） |

**证据等级**（沿用取证轮定义）：`E1` 文档文本 · `E2` schema/字节静态 · `E3` 隔离执行 · `E4` 真实依赖 · `E5` 真实宿主。本路**最高为 E3**（jsonschema 4.26.0 实际执行），**E5 = 0**。

### 0.2 与其他路的对齐状态（写作时的事实）

- A 路（目标一）、B 路（目标二）交付件与报告**已存在**并已读，结论按其原文引用，不重新取证。
- C 路（Evidence/Emotion）、D 路（Persona/提示流）、E 路（状态机）、F 路（算法闭环）交付件**在本路写作时不存在**（`04-PERSONA-PROMPT-FLOW.md` 仅有 55 行 `IN_PROGRESS` 骨架，正文全为 `_(pending)_`）。因此第 5/6/8 项判定由本路自行取证，并**在文中标注「本路独立取证」**，避免读者误以为是转述。

### 0.3 标签与优先级

每条结论标 `EXISTING DESIGN`（已有正式设计，给 文件+章节）/ `DESIGN COMPLETENESS`（该设计是否足以执行：输入·输出·步骤·条件·模块责任·边界）/ `RECOMMENDED ADDITION`（建议，显式标 **NEW PROPOSAL**，不代表已批准）。优先级仅用 `P0`/`P1`/`P2`。

### 0.4 一个必须澄清的计数（brief 与实测不一致处）

brief 写「contracts 下 31 个 schema」。实测：`contracts/schemas/` = **20** 个 `.json`，`contracts/config/` = **11** 个 `.json`，合计 **31 个机器契约**（E2，`schema-census.txt` 末行 `TOTAL FILES: 20 (parsed 20)`）。本文一律写「20 schema + 11 config = 31 契约」，不写「31 schema」。

---

## 1. 结论摘要

1. **架构骨架成立，读写闭环不可执行。** 五模块 + SQLite 权威 + 正交分面 + 锚点路由的目标形态与产品诉求一致（`00 §7.2` 五平面、`DD-AUTHORITY-V2 §2/§3`），但**读取侧存在两套互斥算法**（`02 §7.3–§7.7` 的全局六特征链 vs `11 §7.17` 的分组配额链），且**唯一被授权实施的 P2 那一套绑定已废止的 v1 字段**（`memory.type`/`content_kind`/`importance`/`normalized_content`/`observed_at` 均为 `memory-object` 独有，`memory-unit` 无此 14 个字段之一）。
2. **写入侧在授权范围内没有闭环。** M03/M04 为 FUTURE，`candidate_pool` 明确「MUST NOT 在 P0–P3 创建」（`01 §7.16` 表后注），而 `memory.propose` 要求 `queued=true` 写入候选池（`14 §7.2.9`）；`confirm` 通道未授权（`08 §7.6` 硬规则 4）。可写路径只剩人工直调 M01，且被 `FC-007` 的全量写入拒绝再堵一层（本路 E3 复现：1 条错误，`validation_state`）。
3. **5 个权威对象里 3 个没有存储表。** `01:3` 自述「SQLite 需新增 event / emotion_context / person_profile / candidate / profile_version 表」，但 `01 §7.16` 的 13 张表**一张都没有**；`memory_unit` 的 14 个 V2 字段、`evidence` 的 7 个 V2 字段（含 `original_text`）在 13 表中无列。目标三（情绪附件）、目标四（Persona）在存储层**不可落库**。
4. **目标五的末端断了。** 同时存在 **3 份宿主返回契约**：`03 §7.3.4` 八字段、`08 §7.7` 八字段（两者字段名一致）、`context-response.schema.json` 11 属性。实测：满足 `08 §7.7` 八字段的响应被 schema 以 `additionalProperties` **全部拒绝**；而 schema 合法的响应则**一个都不满足** `08 §7.7`，按 08 的硬规则「缺 `context_id` → 宿主 MUST 拒绝注入」⇒ **任何合法响应都不能注入**（E3，`validate-checks2.json` `V8`/`V8b`）。
5. **「人物锚点优先」在文本上被精确规定，在契约上被反向编码。** `11 §7.14` 规定 `SELF_EXPLICIT` 第一；`anchor-set.schema.json` 的全局 `weights` 却是 `PERSON_PROFILE 0.35 > EVENT 0.30 > SELF_EXPLICIT 0.20`——在**所有**查询里事件权重高于自述，正是 `11 §7.14` 原则 1/2 与 `§7.17` 禁令 1 明令禁止的形态。
6. **「优先级≠强制注入」没有闸门。** `02 §7.5` 的六特征中只有 α·R 是相关性，`E/N/T/S` 与语义无关（β+γ+δ+ε = **0.45** ceiling）；`02 §7.7` 的 GreedySelect 只以 `U ≤ 0` 停止 ⇒ **零相关候选可得 0.45 分并被选中**（E2+算术，`alg04_zero_relevance_floor`）。同时无分组配额（`anchor-set` 无 `quotas`）、无「已注入集合」字段（`read-request`/`retrieval-request` 均无）⇒ 每轮可重复注入（违反 `14 §7.4` 硬规则 2）。
7. **评级**：**C — 核心能力系统性缺失**。不是方向错误（骨架可用），是**规范层缺失**：算法两套、契约三套、表缺三张、写者缺一半。

### 1.1 两个必填字段值

| 字段 | 值 | 含义 |
|---|---|---|
| `CORE_ARCHITECTURE_STATUS` | `SOUND_SKELETON__NON_EXECUTABLE_LOOP` | 骨架（五模块/三轴/五对象/锚点路由）成立且自洽；读写闭环在执行层断裂（读取=2 套互斥算法且 v1 绑定；写入=无授权闭环；存储=3 对象无表） |
| `MAJOR_ARCHITECTURAL_CONFLICTS` | `5 P0 + 8 P1 = 13`（其中 3 条与 A/B 路同源，1 条修正 FC-018 的严重性） | P0：`G-C001`…`G-C005`；P1：`G-C006`…`G-C013` |

---

## 2. 七项能力 × 当前状态（一页速览）

| # | 核心能力 | 设计层（文档） | 契约层（schema/config） | 存储层（13 表） | 执行层（算法） | 判定 | 归口 |
|---|---|---|---|---|---|---|---|
| 1 | 短期 / 长期记忆 | `V2 §2.5`、`11 §7.12` 3×6 矩阵 | `memory-unit.memory_tier` 必填 | **无列、无索引** | **0 条 tier 转换** | 不可执行 | A 路 P0 + `G-C004`/`G-C005` |
| 2 | 长期记忆分区与关联 | 正交分面 9 维 + 8 关系类型 | `relation` 两端仅 `memory_id` | `memory_relations` 存在，但 Event/Profile 无表 | 无自动跨区块联合召回 | 部分可执行（同表内） | B 路 P0 + `G-C003` |
| 3 | 情绪附件 | `V2 §3.4`、`13` 全章、`§9` ΔE | `emotion-context` 完整；但 `read-response.emotion_context.maxItems=0`、`read-request.include_emotion.const=false` | **无 emotion_context 表** | ΔE 默认关闭（CR-03） | 可表达、**不可存取** | `G-C009` |
| 4 | Persona / 稳定人格 | `12` 画像 11 分区、`V2 §3.5` | `person-profile` 完整；**无任何对外接口 schema** | **无 person_profile 表** | M04（P4）未授权 ⇒ 无写者 | 无运行时接口 | `G-C010` |
| 5 | 信息读取与利用 | V2 §8 五阶段、`11 §7.17`、`14 §7.2` | 3 份返回契约、3 份请求形态 | 读取靠 M01 R | 2 套互斥算法 | **阻断** | `G-C001`/`G-C002`/`G-C007`/`G-C008` |
| 6 | 模块独立性 | 五模块 + 依赖矩阵（含 8 条禁令） | M01 七函数 R | — | M02 内串 5 个子系统 | 伪分离（依赖表结构） | `G-C006`/`G-C011` |
| 7 | 核心算法完整 | 12 个 ALG（17 要素齐） | 参数注册表滞后 | — | 4 个 FUTURE（ALG-07/08/09/11） | 开环 | §10 |

> 读法：**「设计层」几乎全绿**是本架构的真实优点（17 项能力都有落点，`V2 §1`），**「契约/存储/执行层」大面积红**是本路的核心结论。任何把「有文档章节」当成「有能力」的读法都会得出错误结论。

---

## 3. FC-001…049 三分类（合并同类项，全覆盖）

分类口径：`仍有效·进主清单` = 该冲突直接使七项核心能力之一失效/严重失真；`仍有效·不进` = 冲突真实存在但不改变能力结构（前置裁决、治理、运维）；`已被替代` = 后续证据或决策已使原判断不成立；`仅形式差异` = 命名/格式/记录形式，不产生能力后果。

| # | 含 FC id | 冲突（合并后） | 三分类 | 能力影响 | 主清单落点 |
|---:|---|---|---|---|---|
| 1 | 001, 002 | 两套 R4.1 集零交叉引用；`APPROVED` 无内部依据 | 仍有效·不进 | 全部（不可执行性的根因，非能力缺陷本身） | 前置裁决 DD-001/002，不重复取证 |
| 2 | 003 | `P0–P3` 二义（V2 §10 表 = M03+Pipeline；开篇 = M01+M02 不变） | 仍有效·不进 | 决定「V0.9」是什么，本文 `blocks_v09` 列按 M01+M02 口径 | DD-003 |
| 3 | 004 | 「交付包」钉在 pre-round-2 | 仍有效·不进 | 取证可信度，非能力 | §13 |
| 4 | **005, 006** | `memory-object`(v1) 与 `memory-unit`(v2) 并存；`content_kind` 三废五留 | **仍有效·进** | 能力 2/5：ALG-04 特征 N 依赖 `content_kind`；14 个 V2 字段无列 | `G-C001`（读取侧）、`G-C003`（存储侧） |
| 5 | **007** | 默认态 `ACTIVE`+`unverified` 违反自身 `allOf` | **仍有效·进** | 能力 1：**每条写入被拒** | `G-C005` |
| 6 | **008, 009** | Evidence 无法同时 schema 合法与 V2 合法；全域无 DDL | **仍有效·进** | 能力 2/3：证据链与原句追溯 | `G-C003` |
| 7 | 010, 011 | `source_type` 4 套枚举；`status` 6 套枚举 | 仍有效·不进 | 迁移期摩擦，不阻断能力 | §13 |
| 8 | 012 | `scope` 请求侧数组 / 实体侧单值 | 仅形式差异 | 重命名即可 | §13 |
| 9 | **013, 014, 015** | tier 轴 0 边；tier 被用作门禁；17 个转换未定义 | **仍有效·进** | 能力 1（A 路已取证，本路只判严重性） | `G-C004` 引用 |
| 10 | 016 | SM-1 混入 validation 轴两条边 | 仅形式差异 | 删两行 | §13 |
| 11 | **017** | 删除掩码写在已废止的 `lifecycle_state` | **仍有效·进** | 能力 2/5：删除后仍可被检索 | `G-C003` |
| 12 | **018** | 边界矩阵禁止 M02 直连 SQLite，而 ALG-02 §5.6 要求回源 | **仍有效·进（严重性下调）** | 能力 6 | `G-C006`（本路认为「永不通过」过强，见 §6 注） |
| 13 | **019, 021** | 候选池无合法写者；Pipeline 无模块行 | **仍有效·进** | 能力 1/7 | `G-C004` |
| 14 | **020** | M02 被要求写权威态 `last_retrieved_at` | **仍有效·进** | 能力 6/7 | `G-C006`/`G-C013` |
| 15 | 022, 023 | `GET /tasks/{id}` 无 owner；无启停序列 | 仍有效·不进 | 运维 | §13 |
| 16 | 024 | 「SQLite 唯一权威」只对内容成立 | 已被替代 | 上一轮的澄清结论已吸收 | §13 |
| 17 | 025 | `DD-AUTHORITY.md` 声明了不存在的目录树 | 仅形式差异 | 文档 | §13 |
| 18 | **026** | 参数注册表滞后（`recall.*` vs `retrieval.*`；分组配额无登记） | **仍有效·进** | 能力 5：**被禁用的全局权重有登记、被要求的分组配额没有** | `G-C001`/`G-C012` |
| 19 | 027 | `permission.public_enabled` locator 不存在 | 仍有效·不进 | 该开关默认 false，风险低 | §13 |
| 20 | **028** | 响应信封只有散文，无 schema | **仍有效·进** | 能力 5/6 | `G-C002` |
| 21 | 029, 030 | 无 HTTP 状态映射；`revoke` 默认 TBD | 仍有效·不进 | 实现期语义（`revoke` 属删除语义，见 §8 注） | §13 |
| 22 | 031 | `error-codes §2` 对自身枚举的错误陈述 | 仅形式差异 | 文档 | §13 |
| 23 | **032** | `subject_id` 文档为所有者、实现为请求者 | **仍有效·进** | 跨主体混淆风险 | `G-C011` |
| 24 | 033, 034 | 审计保留期被当备份保留期；`retention_policy` 无词表 | 仍有效·不进 | 运维 | §13 |
| 25 | 035 | 恢复会复活已删除记忆并复原已撤销授权 | 仍有效·不进 | 数据丢失/隐私，但属恢复路径，不改变能力结构；真实数据前必须解（DD-011） | §13 |
| 26 | 036, 037 | 明文落盘；无缓存失效设计 | 仍有效·不进 | 按 brief 不做独立安全审计 | §13 |
| 27 | **040** | config 暴露被否决的 ANN 旋钮 | **仍有效·进** | 能力 5：配置可让检索退化到 30% 召回 | `G-C001` 子项 |
| 28 | 038, 039, 041 | 索引选项被忽略 / ANN recall 0.30 / 精确性结论不成立 | 已被替代 | ANN 已在设计上 `REJECTED`（`02 §3`），只剩配置同步；错误数值应撤回 | §13 |
| 29 | 042, 043 | BOM / NUL 字节 | 仅形式差异 | 工具链 | §13 |
| 30 | 044–049 | 变更日志缺项 / 清单哈希 / 统计 / 继承矩阵错行 / 80% 重复 / 同步损坏 | 仅形式差异 | 治理 | §13 |

**计数**：仍有效·进主清单 **17 个 id**（005,006,007,008,009,013,014,015,017,018,019,020,021,026,028,032,040）；仍有效·不进 **16**；已被替代 **4**；仅形式差异 **12**。合计 **49** ✓

---

## 4. 核心功能阻断问题清单 — P0

> 九问顺序固定：①现在有什么 ②缺什么 ③为何伤害核心能力 ④模块权责冲突 ⑤算法缺失 ⑥推荐方案 ⑦需补/改文档 ⑧是否阻断 V0.9 ⑨能否在 V0.9 实验中解决。

### G-C001（P0）读取路径存在两套互斥算法，且唯一被授权实施的一套绑定已废止的 v1 字段

| 九问 | 内容 |
|---|---|
| ① 现在有什么 | `EXISTING DESIGN`：`02 §7.3 ALG-02`（多路召回，17 要素齐）、`§7.4 ALG-03`（RRF）、`§7.5 ALG-04`（六特征精排）、`§7.6 ALG-05`（Token 预算）、`§7.7 ALG-06`（Context Builder）；`11 §7.17`（Anchor 排序五阶段）+ `§7.19`（9 意图→锚点映射）；`V2 §8`（检索五阶段）；实施范围标 `P2`（`14 §7.1` 说明 1） |
| ② 缺什么 | `DESIGN COMPLETENESS: MISSING`（4 项）：(a) **两套链没有合并口径**——`02` 的链是「召回→RRF→六特征全局分→贪心→分区」，`11` 的链是「意图→锚点组→组内特征→跨组按配额装配→预算裁剪从低优先组开始」，两者在**排序单元**（候选 vs 锚点组）与**裁剪顺序**上不可调和；(b) `02` 全链**没有锚点/意图输入**（ALG-02 输入表只有 `query/身份/scope_filter/type_filter/time_range/top_k/budget`，E2）；(c) ALG-04 的 6 个特征中 `N` 依赖 `memory.type`+`content_kind`、`T` 依赖 `observed_at`、`P` 依赖 `normalized_content`、过滤用 `importance`——**这 5 个字段只存在于 `memory-object`（v1），`memory-unit` 一个都没有**（E2：`object-only fields` 列表）；(d) RRF 通道集三套口径：`02 §7.4` 三通道含 `structured` 且 `w_structured=1.0`，`retrieval.schema.json` 的 `fusion.weights` **只允许 `fts`+`vector`**（required 两项），`11 §7.17` 阶段 2 写「structured MUST NOT 计入 RRF（**与 02 §7.4 一致**）」——该括注**不成立** |
| ③ 为何伤害核心能力 | 能力 5 的每一步都落在这条链上。用 v1 字段实现则 `read-response` 要求的 9 字段（`semantic_type`/`anchor`/`lifecycle`）无处可取；用 V2 字段实现则 ALG-04 的 4 个特征不可计算（`available=false` → 权重重分配 → 排序退化为 α·R 单特征）。更严重的是 `structured` 参与 RRF：它本是被否决的「把过滤器当召回通道」，会让「命中过滤条件」直接抬高排序，与 `11 §7.16` 的「优先级 ≠ 强制注入」同源失效 |
| ④ 模块权责冲突 | Context 组装规则同时写在 `02 §7.7`（ALG-06）、`11 §7.17` 阶段 5、`08 §7.5` 步骤 7 三处，**无唯一 owner**，违反 `00 §7.3` 硬规则 5（横切能力 MUST 唯一 owner）与 D-AP-03。M02 在 R3 中拥有「检索 11 项编排权威」（`11-ALGORITHM-REVIEW-SCOPE §二.4`），但算法文档没有把编排权写成一套流程 |
| ⑤ 算法缺失 | 是。缺 **`ALG-02b AnchorPlan`**（意图→有序锚点组+每组配额，输入/输出/失败语义）与 **`ALG-04b GroupRank`**（组内特征、权重、并列打破、跨组装配），需明确「组内分不与组间比」这一条，否则实现者必然回到全局加权 |
| ⑥ 推荐方案 | **NEW PROPOSAL**：以 `11 §7.17` 为骨架**重写** `02 §7.3–§7.7` 的输入与排序段：① ALG-02 输入增加 `intent`、`anchor_plan[]`、`anchor.max_candidates`；② 删除 ALG-04 的全局 α…ζ，改为 `Score(group) = w_R·R + w_E·E + w_T·T`（组内），权重表按锚点组建档；③ RRF 通道固定 `{fts, vector}`，`structured` 只出现在过滤器；④ `retrieval.schema.json` 的 `fusion.weights` 与 `reranking.weights` 同步（后者当前是 5 项：`rrf/exact_match/type_relevance/time_relevance/source_quality`，缺 ζ 重复惩罚，与 ALG-04 的 6 项不一致）；⑤ ALG-06 停止条件加「组配额耗尽」与「R < 相关性下限」两道 |
| ⑦ 需补/改文档 | `02 §7.3–§7.7`（重写输入与 §3.1 特征表）、`02 §8.1 ScoredCandidate`（加 `anchor_group`）、`11 §7.17`（把「与 02 §7.4 一致」改为实际约束）、`contracts/config/retrieval.schema.json`（`fusion`/`reranking`/新增 `anchor` 段）、`15 §7.6`（登记分组配额 4 个参数：`anchor.group_quota.{profile,event,evidence,emotion}`） |
| ⑧ 是否阻断 V0.9 | **是**（M02 属原 P0–P3；`06-最小记忆闭环验收标准` 的步骤 4 与 A5/A6 直接验收这条链） |
| ⑨ 能否在 V0.9 实验中解决 | **部分**。可以实验证明「双通道 RRF + 分组配额」可行并给出基线指标；但**「哪一套算法是权威」必须先裁决**（属设计决定，不是实验能产出的事实）。至少要 1 个决定 + S/M 级改写 |

### G-C002（P0）宿主返回契约三套并存，且 schema 与 08 的强制字段零重合 ⇒ 任何合法响应都不能注入

| 九问 | 内容 |
|---|---|
| ① 现在有什么 | `EXISTING DESIGN` 三份：`03 §7.3.4`（recall 响应八字段：`request_id/context_id/memory_refs/versions/token_usage/retrieval_status/permission_status/adapter_version`）；`08 §7.7`（同名八字段 + 「缺失 `context_id` → 宿主 MUST 拒绝注入」硬规则）；`contracts/schemas/context-response.schema.json`（11 属性，required 7 项：`query_id/status/selected_memories/total_tokens/permission_trace/selection_trace/algorithm_version`） |
| ② 缺什么 | `DESIGN COMPLETENESS: CONFLICTED`。E3 实测（`validate-checks2.json`）：把 `08 §7.7` 八字段全给齐的响应 → **3 个错误**，`additionalProperties` 报「`adapter_version, context_id, memory_refs, permission_status, request_id, retrieval_status, token_usage, versions` 全部 unexpected」；反向构造 schema 最小合法实例 → **合法**，但 `field_map_08_vs_context_response` 显示 8 个字段**全部 ABSENT**。另：`retrieval_status` 在 08 是 5 值（`ok/ok_empty/partial/unavailable/degraded`），schema 的 `status` 是 3 值（`success/empty/failed`）；`permission_status` 5 值 vs `permission_trace[]` 数组 |
| ③ 为何伤害核心能力 | 能力 5 的**交付端**不存在可满足的实现：M02 无论按 02 还是按 08 实现，都无法产出一个「既 schema 合法、又能被宿主接受」的对象。这是**闭环断点**，不是字段美化问题：宿主按 08 判「缺 context_id」拒注入，M02 按 schema 不产生该字段 |
| ④ 模块权责冲突 | 返回体归属在 `03`（Memory API）、`08`（集成规范）、`contracts`（schema）三方各写一次；`00 §7.3` 的模块表没有「Context Adapter / Memory API 响应体 owner」一行。`08 §7.9` 又规定「最终 Prompt 拼装 MUST NOT 参与」⇒ 边界正确，但**交接对象**未定 |
| ⑤ 算法缺失 | 不是算法问题，是契约不一致（属 `DESIGN COMPLETENESS`） |
| ⑥ 推荐方案 | **NEW PROPOSAL**：以 `context-response.schema.json` 为唯一机器契约，**加字段而非改名**：增 `context_id`、`request_id`、`memory_refs[]`、`versions{}`、`token_usage{}`、`retrieval_status`（5 值）、`permission_status`（5 值）、`adapter_version`，保留 `selected_memories[]`/`excluded_memories[]`/`selection_trace[]`（这三个字段有价值，08/03 缺，不要删）；`total_tokens` 降为 `token_usage` 的子字段；`status` 改名 `retrieval_status` 或声明映射。同步改 `03 §7.3.4` 与 `08 §7.7` 引用该 schema，删除两处散文式字段表 |
| ⑦ 需补/改文档 | `contracts/schemas/context-response.schema.json`（**改 schema**）、`03 §7.3.3/§7.3.4`、`08 §7.7`、新增 `response-envelope.schema.json`（FC-028） |
| ⑧ 是否阻断 V0.9 | **否**（宿主注入属 P5/P9）；但**阻断「信息读取与利用」作为产品能力的验收**，也阻断 `08 §12` 的集成测试 |
| ⑨ 能否在 V0.9 实验中解决 | **能，且成本最低**：一次 schema 增字段 + 两处文档引用，S 级。不需要任何实验，只需一个人拍字段名 |

### G-C003（P0）5 个权威对象中 3 个无存储表；另 2 个各有 14 / 7 个 V2 字段无列

| 九问 | 内容 |
|---|---|
| ① 现在有什么 | `EXISTING DESIGN`：`01 §7.16` 13 张表（PROPOSED，Markdown 列清单，非可执行 DDL）：`subjects / memory_objects / memory_versions / evidence / memory_evidence_links / memory_relations / observations / outbox_events / event_ledger / index_registry / embedding_spaces / schema_migrations / audit_metadata`；5 个对象各有 schema（`memory-unit`/`evidence`/`event`/`emotion-context`/`person-profile`） |
| ② 缺什么 | `DESIGN COMPLETENESS: MISSING`，且是**文档内部自相矛盾**：`01:3` 头注 ⑥ 明写「SQLite 需新增 **event / emotion_context / person_profile / candidate / memory_evidence_link(已有) / profile_version** 表」，而 `§7.16` 的 13 表**没有任何一张**。逐项：`events` 表 ∅、`emotion_contexts` 表 ∅、`person_profiles`(+`profile_versions`) 表 ∅、`candidates` 表 ∅（`candidate_pool` 被列为 FUTURE 禁止建表）；`memory_objects` 表只写「全部 Memory Object 字段（§7.6）」= v1 字段集，`memory-unit` 的 **14 个 V2 独有字段**（`anchor/semantic_type/semantic_tags/source_type/status/subject_type/memory_tier/visibility/valid_until/event_refs/emotion_refs/relation_refs/merged_from/profile_field/tags_extra`）无列；`evidence` 表列清单缺 `original_text/speaker_id/context_before/context_after/visibility/revoked/tenant_id` 中的 7 项（E2，`read-chain-probe.txt` §7 + A 路 §7.16 #4） |
| ③ 为何伤害核心能力 | 能力 2/3/4 全部落库失败：事件记忆无表 ⇒ 事件与人物画像只能塞进 `memory_objects` 的 JSON 列（`01 §7.16` 通用约定又说「权限、版本、生命周期 MUST NOT 进 JSON」）；情绪无表 ⇒ 「取事件对应情绪附件」无对象可查；画像无表 ⇒ PersonProfile 是**唯一**的长期人格载体却无存储。且**无表 = 无 FK/CHECK/INDEX** ⇒ 跨对象引用（`event_refs`/`emotion_refs`）没有完整性约束，`FC-035` 的悬空引用问题无解 |
| ④ 模块权责冲突 | M01 是唯一写者（`00 §7.3` 矩阵），但**表不存在 ⇒ 无合法写入者**。M04 被指为画像维护者（`V2 §3.5`）却无表可写 |
| ⑤ 算法缺失 | 是（DDL 层的算法缺失）：无迁移脚本、无 `schema_version`↔`user_version` 映射、无「13 表之外新增 4 表」的迁移行 |
| ⑥ 推荐方案 | **NEW PROPOSAL**：一次 DDL 修订（v2）产出 17 张表 = 13 表按 V2 字段补齐 + `events` + `emotion_contexts` + `person_profiles` + `profile_versions`；`candidate_pool` 保持 FUTURE（但需回答「propose 的落点」，见 G-C004）；`emotion_contexts` 必须有 `INDEX(event_id)` 与 `INDEX(subject_id, created_at)`；`events` 必须有 `INDEX(time_start)` 与 `participants` 关联表或 JSON+校验列；`memory_objects` 补 14 列并把 `memory_tier`/`status` 进 `CHECK`。**同时**把 `01 §7.16` 从「建表建议」升级为「可执行 DDL + 迁移脚本」，否则 FC-009 不会被真正关闭 |
| ⑦ 需补/改文档 | `01 §7.16`（重写为 17 表 + 列级字段）、`01:3` 头注（与其一致）、`02 §7.3/§7.5/§7.6/§7.7`（数据依赖表名改 `events`/`memory_objects`；`memory_objects` 列集变更）、新 `contracts/ddl/`（**新增目录**）、`13 §7.x`（补 `emotion_contexts` 的持久化与索引约定）、`12 §7.x`（补 `person_profiles` 写入时机） |
| ⑧ 是否阻断 V0.9 | **是**（M01 = P1，无表即无 M01） |
| ⑨ 能否在 V0.9 实验中解决 | **不能**。DDL 只能被「写出来」，不能被实验出来；但**可在实验中验证** DDL 是否满足 5 类场景（写入/回源/删除传播/情绪反查/画像重建）。工作量 L |

### G-C004（P0）授权范围内不存在写入闭环：候选池被禁止建表，confirm 通道未授权，M03/M04 为 FUTURE

| 九问 | 内容 |
|---|---|
| ① 现在有什么 | `EXISTING DESIGN`：`V2 §5` 16 阶段 Pipeline（协调者，不写库）；`09` 全 16 阶段详规；`14 §7.5.2`（propose 只产出 Candidate，`queued` MUST 为 `true`）；`08 §7.6`（retain → M03 候选池，`quality_status=draft`） |
| ② 缺什么 | `DESIGN COMPLETENESS: MISSING`。(a) `candidate_pool` 被 `01 §7.16` 明令「MUST NOT 在 P0–P3 创建」⇒ propose 的落点不存在；(b) `08 §7.6` 硬规则 5：P3 阶段 M03 未实施 ⇒ `retain` MUST 返回未实现；(c) `08 §7.6` 硬规则 4：confirm 只能由 M04 通道触发，而该通道「未授权」；(d) `FC-007` 使任何 M01 写入被 schema 拒（见 G-C005）；(e) Pipeline 无模块行、无 owner（`FC-021`） |
| ③ 为何伤害核心能力 | 能力 1/7：**没有任何合法路径把一句话变成一条长期记忆**。V0.9 的最小闭环（`06-最小记忆闭环验收标准` 步骤 1–2）要求「先从对话生成候选（可规则版）→ 人工确认后写入 SQLite」，但 ①候选池禁建 ②confirm 未授权 ③写入被 schema 拒 —— 三步全断。结果：V0.9 只能是「M01 的 CRUD 测试台」，不是记忆系统 |
| ④ 模块权责冲突 | 候选池「声明写入者 M03 / 合法写入者 ∅」；Pipeline「声明协调者 / 模块表无行」；confirm「声明 M04 / M04 未授权」。§6 给全表 |
| ⑤ 算法缺失 | 是。缺 `ALG-07` 的 P0–P3 降级形态（规则版候选生成）与 `ALG-08` 的降级形态（人工确认算子），两者在 `02` 中标 FUTURE 且无降级算法 |
| ⑥ 推荐方案 | **NEW PROPOSAL（二选一，需裁决）**：**方案 A（推荐，最小 V0.9）**——允许 P1/P2 建**空表** `candidate_pool`（列定义 + 状态机，不含 M03 逻辑），`propose` 落库为 `PENDING`，`confirm` 走**人工 CLI/测试通道**（显式标 `human_owner_review`，不写 `confirmed`，写 `validation_state=unverified`），tier 用 `lifecycle_candidate`+`tier_source=PROVISIONAL`（与 A 路 A-C001 的 bootstrap 规则一致）；**方案 B**——V0.9 明确声明「无写入能力」，只验收 M01+M02，并在 `06` 的验收清单里删掉步骤 1/2（诚实但削弱产品价值）。无论哪条，都必须先把 `FC-007` 的 `allOf` 移到转换守卫 |
| ⑦ 需补/改文档 | `01 §7.16`（`candidate_pool` 建表或明确替代）、`08 §7.6` 硬规则 4/5、`02 §7.8 ALG-07`（加降级形态）、`09 §7.x`（P0–P3 可执行子集）、`06-最小记忆闭环验收标准` §二（与所选方案对齐） |
| ⑧ 是否阻断 V0.9 | **是** |
| ⑨ 能否在 V0.9 实验中解决 | **部分**。规则版候选生成可以在实验中跑（不依赖 LLM）；但「confirm 归谁」是权限决定，必须裁决。工作量 M |

### G-C005（P0）`MemoryUnit` 的文档默认态违反自身 schema ⇒ 不存在合法的新建记忆

| 九问 | 内容 |
|---|---|
| ① 现在有什么 | `EXISTING DESIGN`：`memory-unit.schema.json` — `status.default="ACTIVE"`、`validation_state.default="unverified"`，同时 `allOf[1]` 要求 `ACTIVE ⇒ validation_state ∈ {confirmed, disputed}`；`DD-AUTHORITY-V2 §3.1` 又规定 `validation_state` 默认 `unverified`、`status` 默认 `ACTIVE`；`D-AP-05` 规定 `confirmed` 只能由 M04 写入，而 M04 未授权 |
| ② 缺什么 | `DESIGN COMPLETENESS: CONTRADICTED`。本路 E3 复现（`validate-checks2.json`）：字段齐备的实例（`ACTIVE`+`unverified`）→ **1 个错误**：`validation_state: 'unverified' is not one of ['confirmed','disputed']`；对照组 `ARCHIVED`+`unverified` **合法**、`ACTIVE`+`confirmed` **合法** ⇒ 缺陷被精确隔离在「ACTIVE 且未确认」这一状态，而它正是**每一条新记忆的出生状态** |
| ③ 为何伤害核心能力 | 能力 1 的入口：**没有任何写入路径能写出第一条记忆**。这不是「未调参」，是逻辑矛盾（静态不变量表达了流程规则）。上一轮已捕获（FC-007），本路独立复现确认它**未被修复** |
| ④ 模块权责冲突 | M01 是唯一写者，但 schema 不给它合法状态；M04 拥有 `confirmed` 权却未授权 ⇒ 责任真空 |
| ⑤ 算法缺失 | 不是算法，是 schema 不变量错位 |
| ⑥ 推荐方案 | **NEW PROPOSAL（技术性，无需用户裁决——与 DD-006 选项 a 一致）**：把 `ACTIVE ⇒ validation_state ∈ {confirmed,disputed}` 从 `allOf` **移出**，改为 `SM-2` 的**转换守卫**（`unverified→confirmed` 仅 M04）；`allOf` 改为 `DELETED ⇒ status 终态`、`status=SUPERSEDED ⇒ 有 supersedes 关系` 这类真正静态可判的约束 |
| ⑦ 需补/改文档 | `memory-unit.schema.json`（**改 schema**）、`DD-AUTHORITY-V2 §3.1` 注、`contracts/state-machines/state-machines.md` SM-2 |
| ⑧ 是否阻断 V0.9 | **是**（M01 第一条 insert 即失败） |
| ⑨ 能否在 V0.9 实验中解决 | **能**：一次 schema 修订 + 一条守卫用例，S 级（< 1 天）。这是全部 P0 里**最便宜**的一条，应第一个做 |

---

## 5. 核心功能阻断问题清单 — P1

> 为控制篇幅，本节用统一 9 列紧凑表；每条仍回答全部九问。

| id | ② 缺什么 / ① 现状 | ③ 伤害哪项能力 | ④ 权责 | ⑤ 算法 | ⑥ 推荐（NEW PROPOSAL 除注明） | ⑦ 文档 | ⑧ 阻断 V0.9 | ⑨ 实验可解 |
|---|---|---|---|---|---|---|---|---|
| **G-C006** | `00 §7.3` 依赖矩阵给 M02→M01 = `R`（只读）、M02→SQLite = `—（MUST NOT 直连）`；但 `02 §7.3 §5.6` 让 M02 执行「查询 SQLite 权威行」九步，`03 §7.2` 又写「M02 内依次调用 FTS5、LanceDB、**SQLite 回源**、M05、Context Builder」，与同节禁令 6「业务模块直连 SQLite/LanceDB」**同页冲突**（E1） | 能力 6：M02 无法被单独实现或替换 | M02/M01 边界二义；Pipeline 无模块行 | 否（缺接口约束，不缺算法） | 声明「M02 只经 `M01.getMemory/getMemoryVersion` 回源」，把 ALG-02 §5.6 第 2 步改为调用 M01 七函数之一；`T00-01`（无 M02 直连 SQLite）即可通过；补 `retrieval-request → M01 读接口` 的映射 | `02 §7.3 §5.6/§6`、`03 §7.2`（删「SQLite 回源」改「M01 回源」）、`00 §7.3`（补注）、`contracts/api/api-contracts.md §3` | 否 | 能（一句话级，S） |
| **G-C007** | `11 §7.14` 规定人物理解顺序 `SELF_EXPLICIT → PERSON_PROFILE → PREFERENCE → SELF_DERIVED → RELATIONSHIP → EVENT`，原则 1「自述优先于推断」、原则 2「画像优先于大量事件」；而 `anchor-set.schema.json` 的全局 `weights` 默认值 = `PERSON_PROFILE 0.35 / EVENT 0.30 / SELF_EXPLICIT 0.20 / PREFERENCE 0.15 / CURRENT_STATE 0.15 / SELF_DERIVED 0.10 / RELATIONSHIP 0.10 / EMOTION_CONTEXT 0`（E2）。同一文件另有 `default_routes[].anchor_sequence`（**顺序正确**）与 `profile_first:boolean`（**无任何执行点**） | 能力 5/6：**事件默认压过自述**，正是 `§7.14` 反模式「把 20 条琐事事件排在画像前面」；且 `§7.17` 禁令 1 明令「MUST NOT 存在一套跨所有锚点的固定权重公式」——该文件就是那套公式 | Anchor Router 归 M02，但权重登记在 config，`15` 只有 4 个 `anchor.*` 参数，`anchor.profile_priority` 与 `weights.PERSON_PROFILE` 互相「MUST 一致」却无一致性检查 | 否（是契约形态错误） | ①删 `weights`（或降级为只读镜像并设 `const:0` 之外不可用）；②把 `anchor_sequence` 升级为**必填且带配额**：`[{anchor, quota_tokens}]`；③`profile_first=true` 时绑定 `anchor.group_quota.profile` 下限；④`retrieval.rank` 加「组内分不跨组比较」的断言 | `anchor-set.schema.json`（**改 schema**）、`11 §8.1`（`AnchorSet` 本就要求 `primary/ordered/quotas`，schema 未实现）、`15 §7.6` | 否（但直接决定读取质量） | 能：实验可量化「有/无分组配额下画像命中率」 |
| **G-C008** | 三道闸门全缺：(a) **相关性下限**——ALG-04 中仅 α·R 为相关性，`E/N/T/S` 与语义无关，`R=0` 时上限 **0.45**（E2+算术）；(b) **分组配额**——`anchor-set` 无 `quotas`，ALG-05 按单一序列累积 Token，ALG-06 只以 `U≤0` 停；(c) **已注入集合**——`read-request` 17 属性、`retrieval-request` 12 属性**均无** `already_injected`/`exclude_ids`，而 `14 §7.4` 硬规则 2 要求「Context Builder MUST 去已注入重复项」、`02 §7.7` 第 1 步依赖它 | 能力 5：无关历史可进 Prompt；同一记忆每轮重复注入（Token 浪费 + 人格漂移） | Context Builder 无 owner（同 G-C001 ④） | 是：缺 `RelevanceFloor` 与 `GroupQuota` 两条判定规则 | ①ALG-04 加 `R ≥ recall.relevance_floor`（默认 0.25，EXP-DEFAULT）硬闸；②`E/N/T/S` 只在 `R>0` 时参与；③请求加 `exclude_memory_ids[]`（客户端传入已注入 ID/版本对）；④ALG-06 停止条件加「组配额耗尽」 | `02 §7.5 §5`、`02 §7.7 §5`、`read-request`（**改 schema**）、`retrieval-request`（**改 schema**）、`14 §7.4`、`15`（登记 `recall.relevance_floor`） | 否 | 能：给 20 条无关记忆 + 1 条相关，测注入纯度（可直接做成 A/B 指标） |
| **G-C009** | 情绪链路「可表达、被封禁、无表」：`emotion-context.schema.json` 14 字段齐（`event_id` 可 `null`，`temporal_position` 4 值，`source_type` 仅 2 值）；但 `read-response.emotion_context` 为 `maxItems: 0`（「关闭态返回非空即违规」）、`read-request.include_emotion` 为 `const:false`、`14 §7.2.7` 返回 `FEATURE_DISABLED`；且 `emotion_contexts` **表不存在**（G-C003） | 能力 3：情绪附件作为能力**结构性存在、运行时不可达**；一旦开关打开，**无表、无 `event_id` 反向索引**，「按事件取情绪附件」无从查询 | M05 需对 Event 与 EmotionContext **分别判权**（`14 §7.2.4` SEP-6），但无表 ⇒ 无独立 `visibility` 列可判 | 是：ΔE（`V2 §9`）有公式，但「情绪→Prompt」的注入算法（除 ALG-06 的 `uncertain` 分区外）无落点 | ①先做表与索引（`emotion_contexts(event_id)`），再谈开关；②`maxItems:0` 改为 `maxItems: N`（N 默认 0，配置驱动），避免「改 schema 才能开功能」；③明确「情绪永不与事件正文混排」（`13 §7.12`）在 schema 层如何体现（当前靠散文） | `13 §7.x`、`emotion-context.schema.json`、`read-response.schema.json`（**改 schema**）、`01 §7.16` | 否（V1 默认关闭是**已裁决**的 CR-02，不是缺陷） | 能：情绪反查与注入可作为隔离实验（需先有表） |
| **G-C010** | Persona 与 Memory Core **没有运行时接口**，且这是**有意的边界**：`00 §7.7` 把 Persona 列为「记忆**消费者**；经授权视图接收记忆」FUTURE；`00 §7.3` 依赖矩阵 Persona 行 = `M02:C`（仅契约预留）、`M05:C`、`M01:—`；`08 §7.9` 规定 Memory Core「MUST NOT 提供人格/角色设定」；`V2 §13.7` + `CR-08B` 禁止 P0–P3 动 qq-bridge Persona。所谓「授权视图」**无 schema、无 API、无字段表**（全语料 0 处定义） | 能力 4：稳定人格只能由 `PersonProfile`（P4/P5）间接支撑，而它无表（G-C003）、无写者（M04 未授权）、无读者（getProfile 属 P5） | Persona 侧 owner 在宿主；Memory Core 侧无人拥有「授权视图」 | 是：缺「画像 → 授权视图」的投影算法（取哪些 section、如何保留 `basis`、如何裁剪 Token） | ①把「授权视图」定义为一个**只读投影 schema**（`ProfileView`：`section/content/basis/source_type/confidence/as_of`，禁带 `source_evidence_ids` 正文）；②唯一入口 = `memory.getProfile`；③明确「Persona 不得读 `SELF_DERIVED` 除非 `basis` 标注」（与 `11 §7.9` 强制分离一致）；④**不要**在 V0.9 实现 Persona | 新增 `contracts/schemas/profile-view.schema.json`、`12 §7.7`、`00 §7.7`（把「授权视图」指向该 schema）、`08 §7.9` | 否（Persona 明确 FUTURE） | 不能（无实现方）；但可用 mock 验证投影契约 |
| **G-C011** | M05 判定算法依赖 V2 已删除的字段：`02 §7.10 ALG-09` 输入 `PermissionContext.resource{scope, scope_id?, sensitivity}`，步骤 4「`sensitivity=restricted` 需显式授权」、§5.3「`sensitivity(derived)=max(...)`」、步骤 6 判 `lifecycle_state≠active`；而 `memory-unit` **无 `sensitivity`、无 `scope_id`、无 `lifecycle_state`**（只有 `scope`+`visibility`+`status`）（E2）。另 `FC-032`：`permission-context.schema.json` 的 `subject_id` 文档为「记忆所属主体」却处处当请求者用，且 `additionalProperties:false` 使上下文**不携带任何资源 ID** | 能力 6 + 跨主体风险：要么「一律 DENY」（能力 5 死），要么实现者为通过而漏判 `sensitivity`（跨主体泄露） | M05 是唯一判定入口但拿不到 V2 对象的权限输入；`visibility(3)`↔`scope(5)` 只有映射表（`V2 §4.3`）无算法 | 是：缺 `visibility×scope×sensitivity(V2 无)` 的判定矩阵与「最严格序」的形式化（`ALG-09 §5.3` 给了序 `system_internal > owner_private > project_scoped > session_only > group_shared`，但 V2 对象无 `sensitivity` 参与） | ①在 `memory-unit` 增 `sensitivity`（4 值）或用 `semantic_tags`+`visibility` 派生并写入 DDL 生成列；②`ALG-09` 的资源输入改为「读 M01 的权限元数据」而非调用方传参；③`permission-context` 增 `resource_id/resource_type`；④明确 `subject_id` = 所有者、`actor.id` = 请求者，并在 schema description 与 `04 §7.5` 同步 | `memory-unit`（**改 schema**）、`permission-context`（**改 schema**）、`02 §7.10`、`04 §7.5`、`01 §7.16` | 否（P5） | 部分：判定矩阵可做成表驱动单测；跨主体混淆必须在真实身份前解决 |
| **G-C012** | 契约表面碎片化（E2 计数）：①`memory.*` 九接口在 `contracts/api/api-contracts.md` 出现 **0 次**（该文件只有 `retrieve/rankCandidates/buildContext` 等模块面）；②`read-request.operation` 只枚举 **3/9** 个接口（`memory.search/getByAnchor/getProfile`），其余 6 个「另有专用请求」但**无 schema**；③`ReadRequest` 有 **3 份**形态（`11 §8.3` 含 `tier_filter`/`AnchorSet`；`read-request.schema.json` 是 `anchor[]` 数组、**无 `tier_filter`/`status` 过滤**；`14 §7.2.1` 是 `opts.*` 嵌套且含 `channel_top_k`/`time_range` 两个 schema 里没有的字段）；④`retrieval-request` 的 `type_filter` 仍是 **v1 七值**（`semantic/episodic/...`），且**无 `anchor`/`intent`** ⇒ M02 边界上表达不了锚点路由；⑤`retrieval-response.candidates[]` 只有 `memory_id + source_channels + source_ranks`，**无 `content`/`semantic_type`/`anchor`** ⇒ 阶段 1–2 的产物无法承载锚点；⑥RRF 通道 2 vs 3（G-C001 ②d）；⑦`06-最小记忆闭环验收标准` 步骤 4 仍要求「三路（关键词/向量/时间）各自返回，RRF 融合」——**验收标准本身与 SRD-002 冲突** | 能力 5：同一查询有 3 种请求形态、2 种通道口径、2 套验收口径；实现者无法判断「按哪个写」 | Memory API（`03`）与模块 API（`api-contracts`）两个 owner 各自维护接口面 | 否（契约层） | ①`api-contracts.md` 增 §10「九接口面」并逐条指向 `read-request`（扩展 `operation` 到 9 值 + 6 个专用请求 schema）；②`read-request` 增 `tier_filter`/`status_filter`/`time_range`/`channel_top_k`；③`retrieval-request` 的 `type_filter` 改 `semantic_type` 6 值 + 增 `intent`/`anchor_plan`；④`retrieval-response.candidates[]` 增 `anchor`/`content_ref`；⑤统一为「fts+vector 双通道」，同步改 `06` 验收标准 | `read-request`/`retrieval-request`/`retrieval-response`（**改 schema**）、`api-contracts.md`、`14 §7.2.1`、`11 §8.3`、`06-最小记忆闭环验收标准 §二` | 部分（`api-contracts` 缺九接口面属 P2，但 `retrieval-request` 的 v1 `type_filter` 阻断 M02） | 能（多数是枚举/字段级） |
| **G-C013** | 检索反馈闭环缺失：`02 §7.3 §7)` 允许 M02「在候选通过权限终检后写 `last_retrieved_at`」，`00 §7.6` 红线 2 同样允许；但 `02 §7.2 ALG-01 §17` 把「由 M02 触发写入」列为**禁止行为**，`00 §7.3` 矩阵又给 M02→M01 = `R`（只读）⇒ **`last_retrieved_at` 无合法写入者**。同时 `reinforce` 全域禁写（`02 §7.14`），`last_retrieved` **MUST NOT** 作正向权重（`11 §7.17` 禁令 3） | 能力 7（持续互动）：系统**没有任何基于使用的自适应通道**——不学「哪条记忆常被用到」，这也是「持续互动」与「稳定人格」缺少反馈信号的结构性原因 | M02 声明写 / M01 唯一写者 / ALG-01 禁 M02 触发 → 三方互斥 | 是：缺「命中反馈」的写入算子（谁、何时、写哪一列、如何幂等） | ①明确 `last_retrieved_at` 由 **M01** 在 `getMemory` 的读事务后异步记录（M02 只上报 `decision_id` 列表）；②或**删除该字段**（诚实：V1 不做使用反馈），避免「声明了但不可写」；③若要做反馈，单列 `reinforce_*` 的启用条件必须重新裁决（D-AP-05 禁写是已裁决项，不得绕过） | `02 §7.2 §17`/`§7.3 §7`、`00 §7.6` 红线 2、`01 §7.16`（列归属） | 否 | 能（二选一，S） |

---

## 6. 严重权责冲突表（数据 × 声明写入者 × 实际合法写入者 × 冲突方 × 处置）

| # | 数据 / 字段 | 声明写入者（出处） | 实际合法写入者 | 冲突方 | 后果 | 处置建议 |
|---:|---|---|---|---|---|---|
| 1 | `MemoryUnit` 行（28 字段） | M01（`00 §7.3`、`03 §7.4`） | **∅**（schema 无合法出生态，G-C005） | `memory-unit.allOf[1]` vs `D-AP-05`（confirmed 仅 M04，M04 未授权） | 第一条记忆写不进 | 不变量移入 SM-2 守卫（G-C005 ⑥） |
| 2 | `memory_tier` | M04 裁决，M03 给 `lifecycle_candidate`（`11 §7.12` 规则 1；`09` 阶段 6） | **∅**（M04=FUTURE；`candidate.schema.json` 无 `lifecycle_candidate` 字段，`additionalProperties:false`） | `11 §7.12` vs `candidate.schema.json` vs `01 §7.16`（无列） | 层级永不变化（A 路 A-C001/A-C002） | 见 A 路 `SM-9 TierTransition` + bootstrap 规则 |
| 3 | 候选池 `candidate_pool` | M03（`02 §7.8`、`08 §7.6`） | **∅**（表被禁建；`01 §7.16` FUTURE 表注） | `14 §7.2.9`（`queued=true` 必须落池） vs `01 §7.16` | propose 无处可去；写入闭环断 | 允许建空表（G-C004 方案 A） |
| 4 | `validation_state=confirmed` | M04 审核通道（D-AP-05） | **∅**（M04=P4 未授权；`03 §7.3.4 confirm` 标 FUTURE） | `candidate.status=AUTO_ACCEPT`（CR-01） vs D-AP-05 | 记忆永远 `unverified`，而 `ACTIVE+unverified` 被 schema 拒 | 二选一：授权 M04 最小审核，或允许 `unverified` 长期存在 |
| 5 | `last_retrieved_at` | M02（`02 §7.3 §7`、`00 §7.6` 红线 2） | **∅**（M02→M01 只读；ALG-01 §17 禁「由 M02 触发写入」） | 三处规则互斥 | 使用反馈无法记录（G-C013） | M01 异步记录，或删字段 |
| 6 | `PersonProfile` 行 | M04（`V2 §3.5`） | **∅**（M04=FUTURE + 无 `person_profiles` 表） | `12 §7.x`（画像维护） vs `01 §7.16`（无表） | 人格无载体 | 建表 + 明确 P4 前由人工/离线批写 |
| 7 | `Event` 行 | M01（经 M04 整合，`V2 §3.3`） | **∅**（无 `events` 表；`candidate` 无法表达 event 对象） | `V2 §3.3` vs `01 §7.16` | 事件记忆只能退化为 `semantic_type=EVENT` 的 unit | 建 `events` 表或明确「Event = unit 视图」并同步 schema |
| 8 | `EmotionContext` 行 | M03 抽取 → M04 维护（`V2 §3.4`、`13`） | **∅**（无表 + 三个开关全关） | `13`（能力定义）vs `read-response.maxItems:0` / `include_emotion.const:false` | 情绪附件不可存取 | 建表（含 `event_id` 索引）后再谈开关 |
| 9 | 删除掩码 | `02 §7.2`（写 `lifecycle_state`） | M01 写，但**写在已废止字段**（FC-017） | `V2 §2.6`（`status`）vs `02` | 删除的记忆仍可能被检索 | 掩码改写到 `status` + `deleted_at` |
| 10 | 审计 `audit_metadata` | M05（`02 §7.10` 步骤 10） | M01（`01 §7.16` #13 表；M05→M01 矩阵为 `R`） | ALG-09 让 M05 写审计 vs 矩阵只给 M05 读 | 权限决策无法自证 | 改为「M05 返回 `decision_id`，M01 落审计」 |
| 11 | Pipeline 编排状态 | Pipeline（`V2 §5`、`09`） | **∅**（无模块行、无持久化对象） | `00 §7.3`（五模块固定，禁第六模块）vs `09`（16 阶段协调者） | 编排无 owner，失败重入无载体 | Pipeline 登记为 M02/M01 的组合职责，或明确「无状态编排器」 |
| 12 | `sensitivity`（权限输入） | 调用方传入（`02 §7.10` 输入） | **∅**（V2 对象无此字段） | `memory-object` 有 / `memory-unit` 无 | M05 判权退化（G-C011） | 补 `sensitivity` 或用生成列派生 |

**§6 注（对 FC-018 的修正）**：`09-CONFLICT-REGISTER` 称 `T00-01`（`00:744`「无 M02 直连 SQLite」）「**永远无法通过**」。本路核查：`00 §7.3` 依赖矩阵给 M02→M01 = `R`（只读接口），`03 §7.4` 提供 `getMemory/getMemoryVersion/queryMemories` 三个读函数 ⇒ **存在合法实现**（M02 经 M01 读）。因此 FC-018 应判为「仍有效，但严重性从 P0 下调为 P1」，其「永不通过」的表述过强。**真正的缺陷**是 `02 §7.3 §5.6` 与 `03 §7.2` 的措辞把回源写成「查 SQLite」，与矩阵冲突（同页自相矛盾）。这条修正是本路对前序结论唯一的实质性下调，理由全部在文档内可核。

---

## 7. 重大数据模型冲突表（对象 × 冲突内容 × 影响能力 × 最小修复）

| # | 对象 | 冲突内容（证据） | 影响能力 | 最小修复 |
|---:|---|---|---|---|
| 1 | **同一记忆** | `memory-object`(1.0, 15 必填) 与 `memory-unit`(2.0, 28 必填) **互斥**：无实例能同时满足；`unit` 独有 14 字段、`object` 独有 15 字段（E2 `read-chain-probe.txt §6`） | 2/5 | 选 `memory-unit` 为对外契约；`memory-object` 标 `deprecated`（需先建 `deprecated` 约定，GAP-20），并把它降为**存储视图**（`01:3` 已这样声明，但 schema 未标） |
| 2 | **内容分类** | `content_kind` 三处废止（`00:76`、`00:7`、`V2:291`）五处必填（`01 §7.2/§7.3`、`02:629`、`memory-object.required`、`candidate.required`）；ALG-04 特征 `N` 依赖它 | 5 | 二选一裁决（DD-005）；若废止，ALG-04 的 `N` 改由 `semantic_type`+`semantic_tags` 计算，并接受 `opinion` 语义丢失 |
| 3 | **Event** | `V2 §3.3` + `event.schema.json`（15 必填）齐备；**无 `events` 表**；`event` 无 emotion 引用（E2：`event has emotion ref? []`） | 2/3 | 建表 + 用 `emotion_contexts(event_id)` 做反向关联（不改 event schema 即可） |
| 4 | **EmotionContext** | schema 14 必填齐备；`event_id` 可为 `null`（合法）；**无表**；`read-response` 置 `maxItems:0` | 3 | 建表（`event_id` 索引）+ 把 `maxItems` 改为配置驱动的 N |
| 5 | **PersonProfile** | `V2 §3.5` 10 字段 + `person-profile.schema.json` 齐备；**无表**；11 分区 vs schema 的 `section` 枚举（E2：schema 无 `sections` 枚举约束，靠 11 号文档） | 4 | 建 `person_profiles` + `profile_versions`；把 11 分区枚举写进 schema |
| 6 | **Relation** | 两端仅 `source_memory_id/target_memory_id`（`relation.schema.json`）；跨对象关系只能是**无类型 ID 数组**（`event_refs/emotion_refs/relation_refs`） | 2 | 新增 `RelationEndpoint{type, id}` 并允许 `kind ∈ {memory,event,emotion,profile}`（B 路 B-C001 的同一结论，本节只给模型侧最小改动） |
| 7 | **Evidence** | `V2 §3.2` 13 字段（含 `original_text/speaker_id/context_before/context_after`）vs `evidence.schema.json`（21 属性、**无 `original_text`**、`source_type` 为 v1 12 值、`required` 含 `tenant_id/recorded_at`）。E3：V2 实例被 **4 类错误**拒（`additionalProperties` 列 6 个 unexpected、缺 2 必填、`enum` 不匹配） | 2/5 | 合并已存在的 v2 修正版（`FC-008` 记录该文件在 `r41-baseline/04-contracts/` 下且已被证明可用），并同步 `01 §7.16 #4` 列 |
| 8 | **13 表 vs V2** | `01:3` 要求新增 4–5 表；`§7.16` 13 表为零（G-C003） | 2/3/4 | 17 表 DDL v2 |
| 9 | **读取产物** | `read-response`（11 必填/条，含 `semantic_tags/version`）vs `14 §7.3` 的「9 字段 MUST 一字不差，MUST NOT 增删」⇒ 按 14 构造的响应 **缺 2 字段被拒**（E3 `V2`：`items/0` 报 `semantic_tags`、`version`） | 5 | 把 14 §7.3 改成「9 必备 + 允许 schema 扩展」（一句话），或删 schema 的 2 个必填 |
| 10 | **上下文分区** | 三套分区词表：`02 §7.7 §5.1` = `must/relevant/background/uncertain`（4）；`11 §8.5` = 画像/事件/证据/情绪（4，Token 分区）；`context-response.selected_memories[].section` = `DirectlyRelevant/StableFacts/HistoricalEvents/ActivePlans/RelevantBackground`（5）。**三套无映射** | 5 | 选一套为主（建议 `11 §8.5` 的 Token 分区，因为它是预算单位），另两套作为 label 映射表 |
| 11 | **`memory_tier` 列** | `memory-unit` 必填，13 表无列，`search-document` 无该字段（A 路） | 1 | 补列 + `CHECK` + `INDEX(tenant_id, memory_tier, status)` |

---

## 8. 模块接口阻断表

| # | 接口 / 边界 | 一方声明 | 另一方声明 | 阻断形态 | 最小修复 | 是否阻断 V0.9 |
|---:|---|---|---|---|---|---|
| 1 | M02 → SQLite（权威回源） | `00 §7.3` 矩阵：`—（MUST NOT 直连）`；`03 §7.2` 禁令 6 | `02 §7.3 §5.6`：M02「查询 SQLite 权威行」；`03 §7.2` 正文：「M02 内依次调用 … SQLite 回源」 | 同文档自相矛盾；实现者按任一方都被另一方判违规 | 统一为「经 M01 读函数回源」（G-C006） | 否 |
| 2 | Memory API 九接口面 | `V2 §7` + `14 §7.1`（9 个 `memory.*`） | `contracts/api/api-contracts.md`：0 次出现；只有 REST 八端点 + 模块函数 | 聊天 LLM 的接口面**没有机器契约** | `api-contracts.md` 增 §10 指向 `read-request` | 否（P2 只需 3 个读原语） |
| 3 | 读请求形态 | `read-request.schema.json`（`anchor[]`、无 tier/status/time 过滤、`operation` 3/9） | `11 §8.3`（`AnchorSet`、`tier_filter`、`memory_id/event_id/evidence_id` 直读） | 能力 4「直读长期记忆（`tier=L3`）」**无法表达**（`11 §7.16` 第 4 条） | 加 `tier_filter/status_filter/time_range/channel_top_k`；`operation` 扩到 9 | **是**（M02 读原语需要 tier/status 过滤） |
| 4 | M02 输入 | `retrieval-request.schema.json`：`type_filter` = v1 七值，无 `anchor`/`intent` | `11 §7.17` 阶段 1–2：必须逐锚点取候选 | 锚点路由在 M02 边界**无输入通道** | 增 `intent`+`anchor_plan[]`，`type_filter`→`semantic_type` | **是** |
| 5 | M02 输出 | `retrieval-response.candidates[]`：`memory_id/source_channels/source_ranks/...`，无 `content`/`semantic_type`/`anchor` | `read-response` 要求每条 11 字段（含 `semantic_type`/`anchor`/`lifecycle`） | 阶段 2 产物无法直接进入阶段 5 组装 | 候选增 `anchor`/`content_ref`/`tier`，或明确「组装阶段回源单查」 | 否（可回源补齐） |
| 6 | 宿主返回体 | `08 §7.7` 八字段 + 「缺 `context_id` 拒绝注入」；`03 §7.3.4` 同名八字段 | `context-response.schema.json`（11 属性，8 个字段名全 ABSENT，E3） | **任何合法响应都被拒注入** | 按 G-C002 ⑥ 增字段 | 否（P5/P9） |
| 7 | 注入项字段 | `08 §7.8.1` 七字段：`type`（v1 七值）/`scope`/`source_refs` | `14 §7.3` 九字段：`semantic_type`/`anchor`/`lifecycle`/`visibility`；`context-response` 只有 `representation` 字符串 | 注入项**丢 provenance**：无法标 `ASSISTANT_INFERRED`、无法标 tier | 注入项统一为「9 字段 + `representation`」 | 否 |
| 8 | Context Builder 归属 | `02 §7.7`（ALG-06，M02 内） | `08 §7.5` 步骤 7（Context Builder 独立步骤） | 同一职责两种归属；`00 §7.3` 无该行 | 明确归 M02（`00 §7.6` 已把 `buildContext` 列为 M02 核心接口） | 否 |
| 9 | 候选池 / Pipeline | `08 §7.6`（retain → M03 候选池）、`14 §7.2.9`（`queued=true`） | `01 §7.16`（`candidate_pool` MUST NOT 在 P0–P3 创建） | 写入链无落点 | G-C004 方案 A | **是** |
| 10 | Persona 读取 | `00 §7.7`（经「授权视图」接收记忆） | 全语料无该视图 schema/API；`08 §7.9`（Core MUST NOT 提供人格） | 能力 4 无接口 | 定义 `ProfileView` 只读投影（G-C010） | 否 |

**接口层结论**：10 条阻断里 **3 条阻断 V0.9**（第 3、4、9），其余 7 条属 P5/P9 或可由实现期补齐。这与「V0.9 的瓶颈在 M01/M02，而不在集成层」一致。

<!-- PART2 -->
