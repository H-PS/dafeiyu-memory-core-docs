# 03 · 实施方案可开工性审查（Archive-Review-2026 · A3）

STATUS: COMPLETE

审查对象：`work/` 下 mtime 最新一版实施方案（canonical `Memory-Core-R4.1-Design/` 00–15 + 31 份 JSON 合同，与顶层 `14/15/00/06/13`）。
审查视角：一名不了解上下文的执行 AI，仅凭现有文档与合同、不追加任何决策，能否连续产出 P0–P3 代码并通过 `06` 验收。
标注法：**EXISTING**=原文/实测；**COMPLETENESS**=缺失；**RECOMMENDED**=本审查新增建议（不冒充已裁决）。优先级 P0/P1/P2 指**严重度**，不是阶段编号（阶段冲突见 §3.9 I-5）。
行号基准：各文件自身行号。`Design/` = `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\`。

## 目录

§1 结论 → §2 资产盘点 → §3 九问（3.1 八项判定 / 3.2 DDL / 3.3 Schema 冲突 / 3.4 写入闭环 / 3.5 检索口径 / 3.6 Prompt 归属 / 3.7 参数口径 / 3.8 A1–A15 映射 / 3.9 顺序倒置）→ §4 差异表 → §5 映射表 → §6 最小路径 → §7 方法附录

## §1 结论

### 1.1 一句话

**不能直接开工（M01/M02 产品实现）；只能开工 P0 隔离验证。** 三条独立理由，均非执行 AI 可自决：

1. **授权口径相反**（COMPLETENESS，P0）：`15:1751`、`00:79` 称 P0–P3 已授权；`Design/V1-IMPLEMENTATION-BASELINE.md:3,5`、`Design/SECOND-ROUND-ISSUE-REGISTER.md:7`、`Design/DESIGN-STATUS.md:70` 称"未授权产品实现，仅授权文档 + 隔离验证"。`Design/PENDING-USER-DECISIONS.md:18` D-3 明确这是**待用户决定**项。
2. **DDL 不存在**（COMPLETENESS，P0）：`Design/*.md` 中 `CREATE TABLE` 命中 0 条（复现 §7.2）；只有 `15:402–416` 的 13 个表名与 `Design/01:663–677` 的"关键列"，而 `Design/01:3` 自述还需 `event/emotion_context/person_profile/candidate/profile_version` 表。
3. **写入闭环死锁**（EXISTING 冲突，P0）：`06:20` 要求"人工确认后写入"，但 `Design/01:287`、`Design/14-ChatLLM读写API.md:530,672` 规定 `confirmed` 只能由 M04 写入，M04 在 P5（`15:1722`）。写 `confirmed`=违规；不写则 `active` 非法（`Design/01:284`）。

### 1.2 八项开工判定（含优先级）

| # | 维度 | 判定 | 主证据（文件:行） | 缺口 |
|---|---|---|---|---|
| 1 | 环境/依赖 | **需补**（P0） | `05:12–19`（驱动/模型/分词器三项"未定案"、Node 20/22 分叉）；`Design/SECOND-ROUND-ISSUE-REGISTER.md:10` FLAT BLOCKED | 三项选型待 P0 实测 |
| 2 | Schema | **需补**（P0） | 三套对象：`15:194–226` / `memory-object.schema.json`(1.0) / `memory-unit.schema.json`(2.0)；`Design/PENDING-USER-DECISIONS.md:46` OI-020 | 无权威对象裁定 |
| 3 | DDL | **不可开工**（P0） | `15:400–416`（13 表名）、`Design/01:658–679`（关键列）、`Design/PENDING-USER-DECISIONS.md:89` PD-002 仅"建议" | 缺列类型/PK/FK/CHECK/触发器 |
| 4 | 接口 | **需补**（P1） | `Design/PENDING-USER-DECISIONS.md:16,24–37`（D-1、OI-016/017/018）；`14:1532–1584` 第四套结构 | 契约冻结未达成；P2 前必须定 |
| 5 | 算法伪代码 | **可开工（带条件）**（P1） | `14:309–355,537–575,806–870,1192–1239,1472–1516`；冲突见 `06:22` vs `14:408` | 先裁通道集（§3.5） |
| 6 | 配置 | **需补**（P1） | `14:1592–1671` 完整；9 个权重"待登记"`Design/15:394–398`；`05:82` vs `Design/15:232` | 数值齐、ID/默认值不齐 |
| 7 | 验收 | **不可开工（按现状）**（P0） | `06:30–44`；A2/A3/A4/A10 无对象（§5）；`06:29` 要求全通过 | 验收集与范围不匹配 |
| 8 | 故障恢复 | **需补**（P1） | `15:1362–1396` 有流程；无 RPO/RTO、无对账口径；`06:40–41` | 判据缺失 |

计数：可开工 1（带条件）、需补 5、不可开工 2。

### 1.3 卡点（P0 四项 / P1 三项）

| 编号 | 卡点 | 位置 | 优先 |
|---|---|---|---|
| B-1 | 授权口径相反 | `15:1751`、`00:79` vs `Design/V1-IMPLEMENTATION-BASELINE.md:3,5`、`Design/SECOND-ROUND-ISSUE-REGISTER.md:7`、`Design/DESIGN-STATUS.md:70` | P0 |
| B-2 | 无 DDL；13 表名 vs 18 对象 | `15:400–416`、`Design/01:658–679`、`Design/01:3` | P0 |
| B-3 | 人工确认 ≠ confirmed 通道（死锁） | `06:20` vs `Design/01:284,287`、`Design/14:530,672` | P0 |
| B-4 | 契约冻结未达成（D-1 未裁） | `Design/PENDING-USER-DECISIONS.md:16,24–37` | P0 |
| B-5 | LanceDB FLAT 未实证 | `Design/SECOND-ROUND-ISSUE-REGISTER.md:10`、`Design/02:252–265` | P1 |
| B-6 | SQLite 驱动未裁决 | `05:13`、`Design/SECOND-ROUND-ISSUE-REGISTER.md:9` | P1 |
| B-7 | 排序因子集/权重未裁（4 vs 6 vs 8） | `13:69–73`、`Design/02:397–402`、`14:451–461`、`Design/PENDING-USER-DECISIONS.md:45` | P1 |

### 1.4 先补什么：M0 六件事（RECOMMENDED）

| 序 | 动作 | 产出 | 工作量 | 解锁 |
|---|---|---|---|---|
| M0-1 | 阶段与授权裁定（统一两套 P 编号） | `AUTH-PHASE-RULING.md` | S | B-1、§3.9 I-5 |
| M0-2 | 11 表可执行 DDL + 3 条负向测试（清单见 §3.2） | `db/schema-v1.sql` | M | B-2 |
| M0-3 | 对象裁定：`memory-unit`(v2) 唯一，`memory-object` 降 DEPRECATED-COMPAT | `OBJECT-AUTHORITY.md` | S | §3.3 |
| M0-4 | 勾选 D-1 选项 X + 八字段投影 + 状态枚举统一 | `CONTRACT-FREEZE.md` | M | B-4 |
| M0-5 | 登记 9 个 `recall.*` 参数；`budget_chars=2400`；`top_k=20` | `Design/15` 增补节 | S | §3.7 |
| M0-6 | 验收重写：A1–A15 拆"P0–P3 可判 11 / P4+ 4" | `06` v2 | M | B-3、§5 |

**不建议先做**：M01 CRUD、RRF、装 LanceDB。B-2/B-4 未解时，表名/字段/请求字段名三处都会返工。

## §2 资产盘点

### 2.1 顶层实施五件（`work/`，11-04 16:34–16:46，体积实测）

| 文件 | 字节 | 角色 | 可用度 | 关键缺口 |
|---|---:|---|---|---|
| `14-…算法实施规格.md` | 40,520 | 算法唯一基线（`00:32`） | 高 | 自造 `MemoryRecord`（1532–1584）；8 特征 vs 6 项公式（451–461 vs 467） |
| `15-…目标架构与实施基线.md` | 38,899 | 架构总纲（`00:33`） | 中 | 13 表名无列（400–416）；v1 形状对象（190–227）；Prompt 归属含糊（1028–1042） |
| `00-…交付总索引.md` | 10,174 | 导航+红线 | 高 | 只做 M01+M02 且"暂缓 M03"（57–63）→ 与 `06:19` 冲突 |
| `06-…验收标准.md` | 4,320 | P3 验收 | 低 | 步1 依赖 M03（19）；步2 confirmed（20）；步4 时间通道（22） |
| `13-…检索算法实现规约.md` | 8,158 | 实现约束（自称效力高，`13:4`） | 低 | 因子集（69–73）与 `02` 不兼容；L3 出最终 Prompt（27）与 `Design/08:331` 相反 |

### 2.2 canonical `Design/`（00–15 + contracts）

| 文件 | 字节 | 用途 | 自述状态/关键点 |
|---|---:|---|---|
| `00-总体架构与设计原则.md` | 65,261 | 五平面/不变量 | F3 达成 |
| `01-记忆本体与数据模型.md` | 80,930 | MU 28 字段 + 13 表列清单 | §7.6/§7.16 均 PROPOSED（30,40）；`:3` 需再加 5 表 |
| `02-核心算法规范.md` | 93,618 | ALG-01…12 | ALG-04 六特征（369–482）；`ALG-03` 含 structured 权重（343–345）与 `:3` 自相反 |
| `03`/`04`/`05` | 40,008/22,599/20,568 | 运行/权限/可靠性 | — |
| `06-测试与评估规范.md` | 75,557 | TS/TC 用例 | N/T 与 `02` 互换（OI-019） |
| `07`/`09`/`10`/`12`/`13` | 53,306/82,288/49,819/69,262/89,405 | 演进/摄取/LLM/画像/证据 | 属 P4+ 能力来源 |
| `08-DSH与QQBridge集成规范.md` | 56,100 | 注入边界 | `:331` 宿主唯一拼装者 |
| `11-…MemoryAnchor…md` | 69,462 | 锚点五阶段 | `:605,626,631` 禁单一固定权重全局排序 |
| `14-ChatLLM读写API.md` | 86,801 | 九接口 | `:210–218` 阶段编号与 `work/15` 错位一级 |
| `15-参数注册表.md` | 62,837 | 参数 ID 唯一来源 | `recall.*` 6 项已登记（228–237）；9 项待登记（390–398） |
| `contracts/`（31 JSON=20 Schema+11 Config） | — | 序列化形状 | meta 通过，业务样本未全量校验 |

### 2.3 决定"能不能开工"的治理件

| 文件 | 一句话 | 位置 |
|---|---|---|
| `Design/V1-IMPLEMENTATION-BASELINE.md` | NOT READY；仅授权文档/契约/隔离 P0 harness | `:3,5` |
| `Design/DESIGN-STATUS.md` | 同文既称"P0–P3 授权"又称 NOT_READY | `:63` vs `:70` |
| `Design/SECOND-ROUND-ISSUE-REGISTER.md` | MC-AUD-001 限隔离验证；004 FLAT BLOCKED；003 驱动未冻结；010 DDL 约束未证 | `:7,9,10,16` |
| `Design/PENDING-USER-DECISIONS.md` | D-1/D-2/D-3 待决；OI-016…024；PD-010 五分区 | `:16–18,24–53,97` |
| `Design/reviews/IMPLEMENTATION-READINESS-REPORT.md` | P0 仅验证；P1/P2 各 1 前置 | `:9–11,20` |

### 2.4 缺口统计（实测）

DDL 0 条；对象 3 套；表 13 vs 18；阶段编号 2 套；参数已登记 6 / 待登记 9。

## §3 九问逐条

### 3.1 问一：八项直接开工判定

| # | 维度 | EXISTING（有） | COMPLETENESS（缺） | 判定/优先 |
|---|---|---|---|---|
| 1 | 环境/依赖 | `05:88–101` 十条 P0 清单；`05:14–17` SQLite/FLAT/LanceDB/FTS5 方向 | `05:13` 驱动未定案；`05:18` 模型未定；`05:80` 分词器未选；Node 20/22 两路 | 需补/P0 |
| 2 | Schema | 20 份 Draft-07；`Design/01:242–309` 字段级表 | 无"权威对象"裁定；`Design/01:779–787` 只有要点映射 | 需补/P0 |
| 3 | DDL | `15:402–416` 13 表名；`Design/01:663–677` 关键列；`01:681–687` 五条建表硬规则 | 列类型/PK/FK/CHECK/索引/触发器/迁移脚本全无 | 不可开工/P0 |
| 4 | 接口 | `Design/contracts/api/api-contracts.md`；`Design/08:568–577`；`Design/14:191–218` 九接口 | `Design/PENDING-USER-DECISIONS.md:16,24–37` 三处二义；`14:1532–1584` 第四套 | 需补/P1 |
| 5 | 算法伪代码 | 五段伪代码齐（`14:309,537,806,1192,1472`） | 通道口径冲突（`06:22` vs `14:387–408`） | 可开工/P1 |
| 6 | 配置 | `14:1592–1671`；`Design/15:228–237` | 9 权重未登记（`Design/15:394–398`）；`05:82` vs `Design/15:232`；`14:1620` 硬编码 1024 vs `Design/15:259` | 需补/P1 |
| 7 | 验收 | `06:30–44`；`Design/test-specs/test-specs.md` | A2/A3/A4/A10 无对象；A7/A11 不可判；`06:20` 表名漂移 | 不可开工/P0 |
| 8 | 故障恢复 | `15:1364–1396`；`Design/15:278` 只读对账 APPROVED | 无 RPO/RTO、无对账口径、无 kill 判定 | 需补/P1 |

三条补充事实：`Design/PENDING-USER-DECISIONS.md:16` 自认"P0 仍可开工（不涉及三项），P2 API 前必须定"；`Design/SECOND-ROUND-ISSUE-REGISTER.md:16` 把 DDL+负向测试列为未闭 Gate；`05:96–98` 把 RRF/Outbox 列入 P0 清单（阶段倒置，见 §3.9）。

### 3.2 问二：DDL —— 13 表名能否支撑 P1 写入？

**不能。** P1 最少 9 表，P0–P3 共 11 表；13 表清单里 2 张 P1 用不上，V2 必需列却全部缺失。

**P1 最小 9 表（RECOMMENDED）**

| # | 表 | 必需列（据 `Design/01:663–677` + MU 28 字段） | 关键约束 |
|---|---|---|---|
| 1 | `schema_migrations` | `migration_id` PK, `from_version`, `to_version`, `script_hash`, `applied_at`, `result`, `backup_ref` | UNIQUE(from,to)；仅追加 |
| 2 | `subjects` | `subject_id` PK, `tenant_id`, `subject_type`, `mapping_ref`, `status`, `created_at`, `updated_at` | UNIQUE(tenant_id,subject_id) |
| 3 | `memory_units` | `memory_id` PK, `content`, `semantic_type`, `semantic_tags`, `subject_type`, `subject_id`, `anchor`, `memory_tier`, `source_type`, `confidence`, `validation_state`, `status`, `visibility`, `scope`, `valid_from`, `valid_until`, `event_refs`, `evidence_refs`, `relation_refs`, `emotion_refs`, `merged_from`, `profile_field`, `version`, `created_at`, `updated_at`, `tags_extra`, `schema_version`, `tenant_id` | PK memory_id；UNIQUE(memory_id,version)；CHECK 枚举；version≥1 |
| 4 | `memory_versions` | `memory_id`, `version`, `snapshot_json`, `changed_fields`, `operation`, `actor`, `created_at` | PK(memory_id,version)；禁 UPDATE/DELETE |
| 5 | `evidence` | `evidence_id` PK, `source_type`, `source_id`, `content_hash`, `observed_at`, `recorded_at`, `actor_id`, `tenant_id`, `scope`, `provenance_id`, `revoked` | CHECK length(hash)=64 |
| 6 | `memory_evidence_links` | `memory_id`, `evidence_id`, `support_kind`, `created_at` | PK(memory_id,evidence_id,support_kind)；双向 FK |
| 7 | `outbox_events` | `event_id` PK, `event_type`, `aggregate_id`, `aggregate_version`, `payload_version`, `payload_json`, `status`, `retry_count`, `next_retry_at`, `created_at`, `processed_at`, `last_error` | CHECK 五状态；INDEX(status,next_retry_at) |
| 8 | `event_ledger` | `ledger_id` PK, `event_id`, `aggregate_id`, `aggregate_version`, `from_state`, `to_state`, `operation`, `actor`, `model`, `prompt_version`, `algorithm_version`, `request_id`, `created_at` | 仅追加 |
| 9 | `audit_metadata` | `audit_id` PK, `actor`, `action`, `target_type`, `target_id`, `result`, `request_id`, `policy_version`, `decision_id`, `created_at` | 禁存正文 |
| 10 | `embedding_spaces`（P2） | `embedding_space_id` PK, `model_id`, `model_version`, `dimensions`, `distance_metric`, `normalized`, `chunker_version`, `text_projection_version`, `vector_schema_version`, `status`, `created_at` | UNIQUE 八字段组合；CHECK dimensions≥1 |
| 11 | `index_registry`（P2） | `index_name`, `index_version`, `memory_id`, `memory_version`, `embedding_space_id`, `last_synced_at`, `status` | PK(index_name,memory_id) |

**不建（P4+）**：`memory_relations`、`observations`（对象 FUTURE，`Design/01:671`）、`candidate`/`candidate_pool`（`Design/01:679` **MUST NOT** 在 P0–P3 建）、`event`/`emotion_context`/`person_profile`/`profile_version`、`dream_runs`/`runtime_kv`/`session_state`。

**三条硬理由**

1. **列不对齐**：`15:404` 的 `memory_objects` 指"当前权威状态"，但 MU 的 `semantic_type/semantic_tags/anchor/memory_tier/status/visibility/source_type/subject_type/event_refs/emotion_refs/profile_field/merged_from/tags_extra` 13 个字段**不在** `Design/01:666` 的"全部 Memory Object 字段"（那是 v1 的 29 字段）。
2. **FK 无宿主**：MU 要求 `event_refs/emotion_refs` 必填数组，但对应表在 P4+，且 `Design/01:661` 禁止"关联仅存 JSON"→ P1 只能存空数组并登记 FK 缺口。
3. **规则未落**：`Design/01:683–687` 的五条 MUST（WAL/FK=ON/RFC3339 TEXT/枚举 CHECK/只追加触发器）没有任何 SQL；`Design/SECOND-ROUND-ISSUE-REGISTER.md:16` 记"DDL + 负向测试"为未闭 Gate。

### 3.3 问三：`15 §2.3` vs `memory-object`(v1) vs `memory-unit`(v2)

**`15 §2.3` 是第三套形状，既非 v1 契约也非 v2 契约；16 项差异中 6 项是语义级（改名解不了）。**

| # | 字段 | `15:194–226` | v1（`Design/01:242–309`） | v2（MU 28 字段） | 性质 |
|---|---|---|---|---|---|
| 1 | 类型 | `type`+`subtype`（202–203） | `type` 七枚举+`subtype`（263–264） | `semantic_type`+`semantic_tags`+`anchor` | 语义（拆三） |
| 2 | 生命周期 | `lifecycle_state` 六值（207,255–261） | 六值（276） | `status` 四值+`memory_tier` L1–L3 | 语义（6→4+3 无映射） |
| 3 | 可用性 | `availability_state` 三值（208,265–269） | 同（277） | 无；只有 `visibility` 三值 | 语义（丢 restricted/suspended） |
| 4 | 敏感度 | `sensitivity`（212，无枚举） | 四值（280） | 无 | 语义+枚举缺 |
| 5 | 有效时间 | `valid_time{from,to}` 嵌套（214–217） | `valid_from`/`valid_to`（293–294） | `valid_from`/`valid_until` | 结构+命名 |
| 6 | 来源引用 | `source_refs[]`（219） | `source_refs[]`+`evidence_refs[]`（305–306） | `event_refs`+`evidence_refs`+`emotion_refs` | 语义（拆三类） |
| 7 | 重要度 | `importance`（211） | `importance` 0–1（279） | 无 | 语义（且 `Design/02:482` 禁入公式） |
| 8 | 过期 | `expires_at`（225） | `expires_at`（299） | 无（`valid_until` 是事实失效） | 语义 |
| 9 | 版本号 | `"1"`（197） | const `"1.0"`（252） | const `"2.0"` | 值 |
| 10 | Scope 标识 | 仅 `scope`（200） | `scope`+`scope_id`（255–256） | 仅 `scope` | 已登记 OI-020 |
| 11 | 记录时间 | 无（222–224） | `observed_at`+`recorded_at`（295–296） | 无 | 语义（丢事件时间） |
| 12 | 内容性质 | 无 | `content_kind` 八枚举（265） | 无（V2 废弃，`01:3`④） | 版本 |
| 13 | 检索文本 | 无 | `normalized_content`（267） | 无 | 功能（SearchDoc 需要，`13:42–52`） |
| 14 | 主体类型 | 无 | 无 | `subject_type` 必填 | v2 新增 |
| 15 | 置信度 | `null`（210） | 可选（278） | 必填 | 必填性 |
| 16 | 租户 | 有（199） | 必填（253） | 必填 | 一致 |

**影响链**：`00:33` 指定的架构总纲是 `15`，执行 AI 先读它 → 按 v1 建表 → 被 MU 的 `additionalProperties:false` 拒写 → 再回看 `Design/01:3` 才发现 V2 为准。属**阅读顺序导致的返工**。

**RECOMMENDED（S）**：`Design/01` 顶部加"对象权威位阶表"（MU v2 唯一写入/交换；MO v1 标 DEPRECATED-COMPAT 仅迁移读；`15 §2.3` 改"概念示意，字段以 MU 为准"）。

### 3.4 问四：写入闭环的断点

`06:19–20` 链路：对话 →（步1）候选 →（步2）人工确认 → M01 事务 →（步3）Outbox → 索引 →（步4）检索。

| # | 断点 | 位置 | 性质 | 后果 |
|---|---|---|---|---|
| D-1 | 候选生成器无授权：`06:19` 要"规则版 M03"，`00:63` 暂缓 M03，`15:1718` 排 P4；"规则版"无规格（只有 `origin{messageSeq,sender,at,channel,trigger}` 一句） | `06:19` vs `00:63`、`15:1718` | 授权+规格双缺 | 违规做 M03 或跳过步1（闭环不成立） |
| D-2 | 表名漂移：`06:20` 写"`memory` 表"，13 表无此表 | `06:20` vs `15:404`、`Design/01:666` | 命名 | A1 无法定位表；验收断言不可执行 |
| D-3 | **`confirmed` 死锁**：人工确认写 `confirmed` → 违 `Design/01:287`（仅 M04）与 `Design/14:530,672`；不写 → `active` 非法（`Design/01:284`） | `06:20` vs `Design/01:284,287` | **死锁** | P0–P3 无法产生合法 `active` 记忆 → 步3/4 无输入。唯一合法组合 `pending_review+unverified`（`Design/01:285`）按 A2 不得进检索（`06:31`） |
| D-4 | 候选池不存在：A2 要 TTL 7 天候选池，`Design/01:679` **MUST NOT** 在 P0–P3 建表；`Design/01:538` Candidate FUTURE | `06:31` vs `Design/01:538,679` | 对象缺席 | A2 无验收对象 |
| D-5 | 呈现与追溯互斥：`06:23`"不含内部 ID"vs `Design/02:582`"不得丢 `memory_id`/版本" | `06:23` vs `Design/02:582` | 语义 | 二者只能过一条；`Design/PENDING-USER-DECISIONS.md:31` 的分层并存方案可解 |

非阻塞补充：步3 的向量命中在 `05:18`（模型未定）+ `05:78`（mock）下只能算 MOCKED，`14:1693`、`00:72` 要求显式标注；"人工确认"无 API/CLI/表结构载体（`Design/14:203` 的 `propose` 只产候选且属 P5）。

### 3.5 问五：检索口径 —— `06 步4` vs `13/14`

**不能同时满足。** `06:22` 把"时间"当第三路 RRF 通道；五处一致规定 RRF 只有 fts+vector，时间/结构化只作过滤（4:1）。

| 文档 | 通道定义 | 原文 |
|---|---|---|
| `06` | 三路：关键词/向量/**时间**"各自返回，RRF 融合" | `06:22`（A5 又写"时间**过滤**"，`06:34`） |
| `13` | L1a 结构化过滤 / L1b FTS5 / L1c 向量；`Σ 1/(k+rank)` | `13:19–21,60–64` |
| `14` | 召回三路含结构化，但"**结构化过滤默认只承担范围约束，不当作普通排名通道重复加权**" | `14:287–293,408`；伪代码 weights 只 fts/vector `14:546–553` |
| `02` | 通道 `fts5/vector/structured`；"RRF 仍不计入 structured（OI-022）" | `02:3,311,322` |
| `11` | "结构化通道 MUST 只用于过滤……**MUST NOT** 计入 RRF（与 02 §7.4 一致）" | `11:617` |
| `15` | "通道标识统一 `fts`/`vector`（structured 只过滤不计分）" | `15:235` |

判定：①"关键词/向量进 RRF"可满足；②"时间进 RRF"不可满足——无 `rank_time` 定义，`Design/02:397–402` 只把时间当 ALG-04 的 `T` 特征，强行实现会与 `Design/02:352,367`（缺失通道贡献=0、禁补最差排名）冲突；③A5"时间过滤独立可用"可满足（`15:524–531` 列出 time range）。
另：`14:451–461` 列 8 个特征，`14:467` 公式只有 6 项（entity 是否并入 E、lifecycle_validity 是否丢弃未说明）；`Design/02:397–402` 六特征明确。
**修复（S）**：`06:22` 改"三路召回（结构化过滤+FTS5+向量）→ 两路进 RRF → 规则排序（含时间特征）"。改前 A5/A6 不可机械验收。

### 3.6 问六：Prompt 归属 + persona 段

**`15 §11.2` 与 `08 §7.9` 可对齐（需指名组装器）；`13:27` 与两者直接冲突。**

| 文档 | 原文 | 含义 |
|---|---|---|
| `15:1028` | "最终 Prompt 由上下文组装器构建" | 组装器归属未定义 |
| `15:1040–1042` | "Memory Core 只提供授权记忆内容及来源，不负责决定所有 Prompt 的最终顺序" | 与 `08` 兼容 |
| `08:326–331` | 人格/角色设定：MC **MUST NOT 提供、MUST NOT 覆盖**／宿主负责；最终 Prompt 拼装：MC **MUST NOT 参与**／宿主**唯一拼装者** | 明确 |
| `08:337` | MC MUST NOT 返回"完整 System Prompt"或"角色指令" | 硬规则 |
| `02:607` | ALG-06 禁止"把 Context 当完整 System Prompt" | 与 `08` 一致 |
| `13:27` | L3 = 权限终检 → Token Budget → **最终 Prompt** | 与 `08:331` 相反 |

**persona 段谁产**：文本由宿主拼（`08:327`）；Memory Core 只产画像**数据**（`Design/12` 的 `PersonProfile`、`person-profile.schema.json`、`memory.getProfile`@`Design/14:196,211`、`profile.*`@`Design/15:284–286`）。`15:1032–1038` 把 Persona 列为组装器输入，与"MC 不产 persona"不矛盾——它描述宿主侧输入，但 `15` 未写"组装器=宿主"，这是歧义源。
**第三处冲突（分区）**：`15:652–662` 七段 vs `Design/02:565–572` 四区 vs `PD-010` 五段（`Design/PENDING-USER-DECISIONS.md:97`）vs `context-response.schema` 的 `sections_present`（无枚举）。**RECOMMENDED（S）**：以 PD-010 五段为唯一枚举并写入 schema，`15`/`02` 引用之。

### 3.7 问七：参数口径 —— `14` 五权重 vs `02` 六特征 vs `15 recall.*`

**数值可 1:1 对齐；口径不能。**

| `14`（`14:1624–1640`） | `02`（`02:380–382`） | `15` ID | 状态 |
|---|---|---|---|
| `fusion.k 60` | `k=60` | `recall.rrf_k`（`15:235`） | EXP-DEFAULT |
| `weights.fts 1.0` / `vector 1.0` | `w_fts5=1.0`, `w_vector=1.0`（`02:345`） | `recall.channel_weight_fts/_vector`（`15:144–145`） | 待登记 |
| `rrf 0.55` / `exact 0.15` / `type 0.10` / `time 0.10` / `source 0.10` | `α .55 / β .15 / γ .10 / δ .10 / ε .10` | `recall.rerank_weight_rrf/_exact/_type/_time/_source`（`15:146–150`） | 待登记 |
| `duplicate_penalty 0.10` | `ζ=.10` | `recall.rerank_penalty_duplicate`（`15:151`） | 待登记 |
| `reranking.top_k 20` | `max_items 20`（`02:601`） | `recall.rerank_top_k`（`15:397`） | 待登记 |
| `context.token_budget 1200` | `budget` | `recall.budget_tokens`（`15:233`） | EXP-DEFAULT |
| （无） | — | `recall.budget_chars=2400`（`15:237`） | `14` 缺项 |

**三处口径冲突**

| # | 冲突 | 位置 | 影响 |
|---|---|---|---|
| C-1 | 因子集：`13:71` 只许 RRF+时间衰减+`importance`+时效/有效性（含 importance、无 exact/type/source）；`Design/02:482` 禁止 importance 入公式，六特征含 E/N/S | `13:69–73` vs `Design/02:397–402,482` | 按 13 缺 3 特征、多 1 被禁特征；按 02 违 13"禁自创加权公式"。且 `13:4` 自称效力高、`00:31` 称 13 已被 14 取代 → **位阶未定** |
| C-2 | N/T 互换：`Design/06` 用例里 N=时间、T=类型，与 `02` 相反（OI-019 未裁） | `Design/PENDING-USER-DECISIONS.md:45` | 实现按 02、测试按 06 → 必失败 |
| C-3 | 通道权重：`13:63` 公式无 `w_r`，`14:388`/`02:349` 有；1.0/1.0 时等价，标定后分叉 | `13:63` vs `14:388`、`02:349` | 首版等价，改权重即分叉 |

**默认值不一致**：`05:82` `topK=5` vs `Design/15:232` `top_k=20` vs `14:1633` `top_k=20`；`05:84` `budgetChars=1200` vs `Design/15:237` `2400`；`14:1620` `dimension:1024` vs `Design/15:259` MUST 读元数据。
**判定**：可对齐（数值 100%）、不可直接实现（3 处口径 + 9 个 ID 未登记）。动作：裁 C-1/C-2/C-3 + M0-5（S+S）。

### 3.8 问八：P0–P3 × A1–A15（摘要）

15 项中：有对象 9、有对象但不可判 2（A7/A11）、**无对象 4（A2/A3/A4/A10）**、过程项 2（A14/A15）。**4 项无对象**即：A2 候选池（`Design/01:679` 禁建表）、A3 黑名单（`Design/09` 属 P4）、A4 B-010 生成溯源（需 M03 记录模型/提示词/文本处理规则；`13:111–114` 的 B-010 只是向量空间八字段）、A10 tombstone（`Design/04:82,245` 只有词条，13 表无存储对象/列，K-2 删除语义未裁）。`06:29` 要求全部通过 → **P3 按现状不可能完成**。详见 §5。

### 3.9 问九：顺序倒置（5 处）

| # | 倒置 | 位置 | 后果 |
|---|---|---|---|
| I-1 | P1 建库依赖 P0 的 DDL，但 P0 交付清单只写"Schema"未写"可执行 DDL + 负向测试" | `15:1650–1657`、`Design/PENDING-USER-DECISIONS.md:89` | P1 开工才发现无 `CREATE TABLE` |
| I-2 | P3 验收（A2/A3/A4）依赖 P4 的 M03/摄取 | `06:31–33` vs `15:1718` | P3 无法"全部通过" |
| I-3 | P2 调 `canRead`，M05 在 P6；虽允许 `contract_deny`（`Design/15:269` APPROVED），但权限枚举四套未裁（OI-018） | `15:616,1726`、`Design/PENDING-USER-DECISIONS.md:34` | `permissionCheck` 签名 P6 重写 |
| I-4 | `06` 步1 用被 `00` 明令暂缓的 M03 | `00:57–63` vs `06:19` | 违规做 M03 或跳过步1 |
| I-5 | 两套 P 编号：`15:1718–1742` P4=M03/P5=M04/P6=M05；`Design/14:210–218` P3=M03/P4=M04/P5=API | 见 §7.5 对照表 | 读 canonical 14 者会把 M03 当 P3 提前做 |

附加（不计入 5 处）：`05:96–98` 把 RRF/Outbox 列入 P0 验证清单，二者实为 P2/P1 能力（`15:1667,1679`）。

## §4 与 canonical 差异表

| # | 主题 | 顶层（`work/`） | canonical | 判定 | 处置 |
|---|---|---|---|---|---|
| 1 | 表数量 | 13 表名（`15:402–416`） | 13 表关键列（`Design/01:663–677`）+ 需再加 5 表（`01:3`） | 冲突 | 见 §3.2 |
| 2 | 权威对象 | `15:194–226` 第三套 | MU 28 / MO 29 | 冲突 | M0-3 |
| 3 | RRF 通道 | `06:22` 三路含时间 | `02:3`、`11:617`、`15:235` 两路 | 冲突 | 改 `06:22` |
| 4 | RRF 权重 | `14:388` 有 `w_r` | `13:63` 无 `w_r`；`02:349` 有 | 冲突（首版等价） | 以 02 为准 |
| 5 | 排序因子 | `14:451–461` 8 项；`13:71` 4 因子 | `02:397–402` 六特征 | 冲突 | 裁 C-1 |
| 6 | 排序权重值 | `14:1635–1640` | `02:382` α..ζ | 一致 | 仅登记 ID |
| 7 | Token 预算 | `14:1643` 仅 token | `15:233,237` 双预算；`13:89` 取严 | 缺口+冲突 | `14` 补 chars |
| 8 | 上下文分区 | `15:652–662` 七段 | `02:565` 四区；PD-010 五段 | 冲突（三套） | 统一 PD-010 |
| 9 | Prompt 归属 | `15:1028` 含糊；`13:27` L3 出 Prompt | `08:331` 宿主唯一；`02:607` | 冲突 | `15` 指名；删 `13:27` |
| 10 | persona | `15:1090–1099` 输入清单 | `08:327` MC 不提供人格 | 可对齐 | `15` 加"组装器=宿主" |
| 11 | 阶段编号 | `15:1718–1742` | `14:210–218` | 冲突 | M0-1 |
| 12 | 授权 | `00:79`、`15:1751` 已授权 | `V1-BASELINE:3,5`、`MC-AUD-001`、`DESIGN-STATUS:70` 未授权 | 冲突（治理） | 用户裁决 D-3 |
| 13 | Embedding 维度 | `14:1620` 1024 | `15:259` 禁写死 | 冲突（软） | `14` 改 TBD |
| 14 | 参数 ID | `14` `fusion.*`/`reranking.*` | `15:140–151` 映射到 `recall.*` | 一致 | M0-5 |
| 15 | `13` 效力 | `13:4` 自称最高 | `00:31` 已被 14 取代 | 冲突（位阶） | `13` 头部加注 |
| 16 | 表名 | `06:20` "`memory` 表" | 13 表无 `memory` | 缺口 | 改 `memory_units` |

**canonical 内部不一致（6 条）**：①`02:343–345` 含 structured 权重 vs `02:3` 排除（同 `11:617`）；②`02` 与 `Design/06` N/T 互换（OI-019）；③`Design/01:256` `scope_id` vs schema 无此字段且 `additionalProperties:false`（OI-020）；④`Design/02:548` `ContextPackage` vs `context-response.schema` 结构不同（OI-017）；⑤`Design/08:568–577` 八字段 vs 同 schema；⑥`Design/14:218` `propose`=P5 vs `:224` 候选生成=P3。
**一致项（可直接用）**：五算法伪代码（`14:309,537,806,1192,1472`）；RRF 公式与 k=60（`14:388`、`02:349`、`13:63`、`15:235`）；α..ζ 权重值（`14:1635–1640`=`02:382`）；fail-closed（`15:621–623`、`13:85`、`14:1695`、`Design/15:268–269`）；Outbox 同事务（`15:435–447`）；备份铁律（`15:1400–1411`、`05:110`）。

## §5 P0–P3 × A1–A15 映射表

判定四值：**有对象** / **有对象不可判** / **无对象** / **过程项**。

| A# | 断言 | 原文 | 验收对象 | 阶段 | 判定 | 说明 |
|---|---|---|---|---|---|---|
| A1 | 写入路径唯一 | `06:30` | M01 事务五步 + 9 表 | P1 | 有对象 | 表名待 M0-2 |
| A2 | 候选池 TTL 7 天、不入检索 | `06:31` | `candidate_pool` + M03 生命周期 | P4 | **无对象** | `Design/01:679` 禁建；`:538` FUTURE；TTL 未登记 |
| A3 | 黑名单生效 | `06:32` | 摄取 Pipeline 黑名单/脱敏 | P4 | **无对象** | `Design/09` 属 P4；P0–P3 无候选生成即无作用点 |
| A4 | 生成溯源（模型/参数/文本规则） | `06:33` | ①`embedding_spaces`(P2) ②`event_ledger`(P1) ③抽取溯源(M03) | 部分/P4 | **无对象** | `13:111–114` 的 B-010 是空间八字段，不是抽取溯源 |
| A5 | 三路各自独立可用 | `06:34` | FTS5 + FLAT + 时间过滤 | P2 | 有对象 | 若按 `06:22` 做时间 RRF 则验不过 |
| A6 | RRF 稳定、k 可配 | `06:35` | ALG-03 + `recall.rrf_k` | P2 | 有对象 | `15:235` 已登记 |
| A7 | 呈现合规无 ID/分数 | `06:36` | `context-response` / 八字段 | P2 | **不可判** | vs `Design/02:582` 互斥，需 D-5 |
| A8 | 预算不超 `recall.budgetTokens` | `06:37` | `recall.budget_tokens`+Tokenizer+双预算 | P2 | 有对象 | ID 名不符；chars 缺；Tokenizer 未定 |
| A9 | edit 后向量重算一致 | `06:38` | Outbox + SearchDoc 版本 + 重嵌入 | P2 | 有对象 | B-011 未解但重算可做 |
| A10 | 删除同步 + tombstone | `06:39` | 逻辑删除 + tombstone 存储 | P2/P6 | **无对象** | 13 表无 tombstones/is_tombstone；K-2 未裁 |
| A11 | kill 后 0 差异 | `06:40` | WAL + `index_registry` 对账 | P3 | **不可判** | 无"0 差异"口径 |
| A12 | 备份可恢复 | `06:41` | `15:1366–1396` 十步 | P1/P3 | 有对象 | 无 RPO/RTO（OI-012） |
| A13 | 权限挂载点 + 默认 DENY | `06:42` | `permissionCheck` + `m05_mode` | P2 | 有对象 | `Design/15:268–269` APPROVED/LOCKED；OI-018 签名待定 |
| A14 | 无红线违规 | `06:43` | 文件系统/代码扫描 | 全程 | 过程项 | `00:69–79` 11 条 |
| A15 | 偏差清单 | `06:44` | `PROTOTYPE-PROGRESS.md` | 全程 | 过程项 | 基线冲突：m01/m02 已被 14/15 取代（`00:31–33`）vs 红线 9（`00:77`） |

统计：有对象 9、不可判 2、无对象 4、过程项 2（含在前 9 内）。**无对象 4 项 = A2/A3/A4/A10**。

## §6 最小可开工路径（NEW PROPOSAL）

工作量：S ≤0.5 人日；M 1–2；L 3–5。

| 序 | 动作 | 产出 | 量 | 出口判据 |
|---|---|---|---|---|
| M0-1 | 阶段授权裁定 + 两套 P 编号映射 | `AUTH-PHASE-RULING.md` | S | 唯一映射表；"P0 已授权/产品未授权"可引用 |
| M0-2 | 11 表 DDL + 3 条负向测试（FK/CHECK/版本唯一） | `db/schema-v1.sql`、`negative-tests.sql` | M | 空库执行成功；3 条全被拒；`schema_migrations` 有 v1 行 |
| M0-3 | 对象裁定 + 28 字段↔列全量映射 | `OBJECT-AUTHORITY.md` | S | 覆盖 §3.3 十六项，无"未映射" |
| M0-4 | D-1 选项 X + 八字段投影 + 状态枚举统一 | `CONTRACT-FREEZE.md` | M | OI-016/017/018 标 DECIDED |
| M0-5 | 登记 9 个参数；chars=2400；top_k=20 | `Design/15` 增补 | S | 9 项变 EXP-DEFAULT；`05`/`14`/`15` 三处一致 |
| M0-6 | A1–A15 拆 11 可判 + 4 移 P4；A7/A11 补口径；A5 改时间过滤 | `06` v2 | M | 每条有对象/判据/证据形式 |
| G1 | P0 出口 | 环境报告+驱动裁决+FLAT 证据+DDL+冻结 | — | M0-1…3 完成 + VF-002 通过 |

**P0 三条验证（可与 M0 并行）**：VF-002 SQLite 驱动六项矩阵（`Design/SECOND-ROUND-ISSUE-REGISTER.md:9`）；VF-001 LanceDB FLAT 八项含暴力余弦 Top-K 全等（`Design/02:252–265`，失败即 BLOCKED 不降级）；VF-003 中文分词对比（OI-003）。

**之后三段的最小定义**：P1（L，5–8 日）9 表+事务+版本+Outbox（仅 FTS5）+备份+完整性；P2（L，8–12 日）FTS5+Embedding(mock)+FLAT+两路 RRF+六特征+回源+`permissionCheck`(DENY)+双预算+ContextPackage；P3（M，3–5 日）合成数据+受控测试通道跑 7 步+kill 恢复+对账。**P3 需先裁 D-3 的测试通道**（标 `TEST_ONLY`、写 `unverified+pending_review`），否则无法产生合法 `active`。

**若只有 3 天**：M0-1 → M0-3 → 5 张核心表（`schema_migrations/subjects/memory_units/memory_versions/outbox_events`）→ VF-002 开跑。只回答"库能不能建、驱动选哪个"。

**不要做**：①按 `15:194` 建表；②实现时间 RRF 通道；③用 `13:71` 四因子；④先改 `14` 权重；⑤从响应删 `memory_id`；⑥一次接 15 项验收。

**工期**：M0 4–7 人日；VF 3–5；P1 5–8；P2 8–12；P3 3–5 → **23–37 人日**（单人 5–8 周）；两人并行关键路径 ≈ 4–6 周。

## §7 方法附录

### 7.1 边界

只读 `C:\Users\[USER]\Desktop\work\`（含 `Design/`、`contracts/`），未改任何源文件；只写本文件；未跑哈希、未建库、未装包、未执行 LanceDB/SQLite 实测（文档级审查）。

### 7.2 复现命令

```powershell
cd 'C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design'
Select-String -Path '*.md' -Pattern 'CREATE TABLE|CREATE VIRTUAL'   # 期望 0 条
$p='C:\Users\[USER]\Desktop\work\15-MemoryCore-R4.1目标架构与实施基线.md'
(Get-Content -LiteralPath $p -Encoding UTF8)[401..415]              # 13 表名
$j=Get-Content -LiteralPath '.\contracts\schemas\memory-unit.schema.json' -Raw -Encoding UTF8 | ConvertFrom-Json
($j.properties.PSObject.Properties | ForEach-Object { $_.Name }) -join ','   # 28 字段
$j.properties.schema_version.const                                   # 2.0
(Get-ChildItem '.\contracts' -Recurse -Filter *.json).Count          # 31
(Get-Content -LiteralPath '.\15-参数注册表.md' -Encoding UTF8)[389..404]  # 9 个待登记
```

### 7.3 未验证（明确标"未验证"）

| # | 项 | 复现方法 |
|---|---|---|
| U-1 | LanceDB FLAT 是否真精确 | 按 `Design/02:252–265` V1–V8，10k 合成向量比对暴力余弦 Top-20 |
| U-2 | 三驱动 WAL/并发/迁移 | `Design/SECOND-ROUND-ISSUE-REGISTER.md:9` 六项矩阵同脚本 |
| U-3 | FTS5 中文分词质量 | `06:11–13` 数据集统计专名/型号/数字召回 |
| U-4 | `05:41` 零依赖路径 | `node -v`；`node -e "require('node:sqlite')"` |
| U-5 | 31 份 Schema 业务样本 | 每份 1 正 1 负跑 ajv |
| U-6 | `event/emotion_context/person_profile/profile_version` 列 | 由三份 schema 反推 + 人工复核 |
| U-7 | 13 表是否覆盖 V2 字段 | 用 §3.3 十六项逐项落列 |

### 7.4 工具异常（影响后续审查）

`Design/02-核心算法规范.md` 被 read 工具判为二进制：93,618 字节，含 UTF-8 BOM 与 **1 个 NUL 字节**，位于**第 3 行**偏移 673（上下文"…见 ␀9-记忆摄取Pipeline.md…"，疑为 `09-记忆摄取Pipeline.md` 前的空字符）。UTF-8 解码合法，NUL 触发二进制判定。影响：用 read 类工具者读不到这份**唯一算法规范**（ALG-01…12 全在此文）。建议（P2，RECOMMENDED）删 BOM 与 NUL、修好引用；位置 = `Memory-Core-R4.1-Design/02-核心算法规范.md:3`。本审查改用 `Get-Content -Encoding UTF8` 按行读取，已用 `### 7.5 ALG-04` 等标题行交叉核对行号。

### 7.5 两套 P 编号对照

| 内容 | `work/15` | `Design/14` |
|---|---|---|
| 环境+Schema+接口 | P0 | P0 |
| M01 | P1 | P1 |
| M02 | P2 | P2（"原 P0–P3 范围"） |
| 最小闭环验收 | **P3** | 未单列 |
| M03 | P4 | **P3** |
| M04 | P5 | P4 |
| M05 + Chat LLM 接入 | P6 | P5 |
| Dream | P7 | — |
| Session/Persona | P8 | — |
| 宿主集成 | P9 | — |

### 7.6 方法

①先 `Select-String '^#{1,3} '` 列标题再选读，单次 read ≤250 行；②每条冲突给 ≥2 个文档位置，每条缺口给"应有而未有的位置"；③不采信自述（`DESIGN-STATUS:8–23` 全 F3 但 `:70` NOT_READY）；④三类标注，§6 全为 RECOMMENDED；⑤不做裁决（D-1/D-2/D-3、OI-019/020/022、A2/A3/A4/A10 归属只给选项）。

### 7.7 未展开

`Design/03/04/05/07/09/10/12/13` 仅标题级+关键词级检索（P4+ 会需要）；`test-specs` 与 A1–A15 的逐条映射未建表；31 份 Schema 只 diff 了 3 份；`E:\[PROJECT]` 母本与 `C:\Users\[USER]\Desktop\work` 副本是否同步未验证（`00:3` 称母本在 `E:\[PROJECT]\work-交付包`）。

---

审查完成：2026-11-06｜审查者：Archive-Review-2026 · A3｜状态：COMPLETE