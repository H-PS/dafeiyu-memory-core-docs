# E 路过程报告 · 模块独立性与调整成本

## STATUS: COMPLETE

- 交付件：`E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\05-MODULE-ADJUSTABILITY-MATRIX.md`（STATUS 已改 COMPLETE）
- 本文件：`E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\SUBAGENT-REPORTS\E-REPORT.md`
- 角色：MC-CORE-CAPABILITY-REVIEW-01 子 Agent，E 路只读能力审查
- 遵循：开工先写骨架（`# 标题` + `STATUS: IN PROGRESS` + 章节目录），逐章 edit 追加，最后改 COMPLETE；分块 read（单次 ≤175 行，上限 350）；只读基线，未修改 Design 目录；写文件只用 write/edit。

---

## 1. 执行时间线

| 步 | 动作 | 结果 |
|---|---|---|
| 1 | `write` 建交付件骨架（标题/IN PROGRESS/9 章目录） | 完成，先写后读 |
| 2 | `todo_write` 建 8 步任务清单 | 完成 |
| 3 | `pwsh` 列基线根 / work 03-m0*.md / contracts/config / schemas 计数 | 发现 config=11、schemas=20、合计 31（口径纠正） |
| 4 | `pwsh` 批量 Select-String `^#{1,3} ` 列 10 个关键文件标题 | 得到 00 §7.3=306、DD-V2 §5=311/§10=403、07 §7.1=149 等精确 offset |
| 5 | `read` 00 §7.3–§7.9（306–480，175 行） | 取得依赖矩阵/职责边界/FUTURE/技术选型/12 不变量 |
| 6 | `read` DD-V2 §5/§10（311–425，115 行） | 取得 16 阶段 Pipeline、P0–P6 授权（P3/P4/P5 需阶段授权） |
| 7 | 并行 5×`read` 03-m01~m05 前 80 行 | 发现五份均标 R3 基线、M04「独立 Memory Core」、M05 未实现线索 |
| 8 | 并行 5×`read` 接口/依赖章节（§7.1/§9/§10/§11/§12/§10） | 发现大量「未冻结/示例/示意/PROPOSED」 |
| 9 | `read` 07 §7.1–§7.7（149–323，175）+ §7.8–§7.13（324–448，125） | 取得八维版本/迁移七要素/Embedding 九步/B-010/九必备四禁令/算法替换六前提/FUT-01…08 |
| 10 | `read` 03 §7.4（310–364，55）+ §7.2（118–137，20）+ §7.7–§7.11（497–586，90） | 取得 M01 七函数/六动作/事务边界/16 规范码/八类配置 |
| 11 | `read` 15 §7.2–§7.12（166–310，145）+ §7.16/§7.17（370–411，42） | 取得 42 参数 15 属性、三层次、四值状态、待登记 ID |
| 12 | `pwsh` 统计 31 个 json / 11 config 字段树 / 20 schema 字段数 / 关键词 grep | 取得破坏半径实证（anchor 219/28 文件、semantic_type 72/16 文件…） |
| 13 | `edit` 追加 §1 + §2 表1/表2 | 完成 |
| 14 | `edit` 追加 §2 表3/表4 | 完成 |
| 15 | `edit` 追加 §2 表5/表6 | 完成 |
| 16 | `edit` 追加 §3 十四场景 | 完成 |
| 17 | `edit` 追加 §4 十问 + §5 ASCII 图 | 完成 |
| 18 | `edit` 追加 §6 隐藏耦合 + §7 改善方向 + §8 结论 + §9 附录 | 完成 |
| 19 | `edit` STATUS → COMPLETE | 完成 |
| 20 | `write` 本 E-REPORT.md | 完成 |

**未发生**：未读 `02-核心算法规范.md`（含 NUL），未用清洗副本（本路不依赖 02 算法正文）；未通读 FORENSIC 49 条 FC，仅引用 A-C001–A-C004 / B-C001 编号；未进 node_modules/pydeps/evidence 无关目录；未用 PowerShell 写文件（无 BOM 风险）。

---

## 2. 关键取证与判断摘要

### 2.1 六张表均已建立
表1 模块职责矩阵（6 权 + 禁止行为）、表2 调用依赖矩阵（A/R/C/E/—，含 4 处不一致）、表3 数据所有权矩阵（14 类对象 × 唯一写入者/读取者/共享合同）、表4 模块调整影响矩阵（14 场景压缩索引）、表5 配置与参数归属表（42 参数 + 11 config + 15 号注册表 + 热改判定）、表6 共享合同依赖表（13 组合同 × 生产者/消费者/破坏半径）。

### 2.2 十四场景变更类别分布
| 类别 | 场景 |
|---|---|
| `CONFIG_ONLY` | 7 改 RRF、13 改预算、8（数值） |
| `MODULE_LOCAL_CHANGE` | 3 改分词器、10 改情绪、11 改 Persona Prompt、8（公式） |
| `CONTRACT_CHANGE` | 6 增 relation 类型、14（仅改字段） |
| `SCHEMA_MIGRATION` | 1 换 Embedding Provider、14（增删 section） |
| `MULTI_MODULE_CHANGE` | 5 改 Anchor 分类 |
| `ARCHITECTURE_CHANGE` | 2 换向量库、4 增 semantic_type、9 改 Tier、12 新增 Dream |

HIGH=3/7/13（+1 设计完整度）；MEDIUM=1/6/8/10/11；LOW=2/4/5/9/12/14。场景 9 叠加 A 路 **P0 阻断**（tier 转换 0 条/写入者 0/保留策略 0/L1 无 owner），本路不重做 A 路论证。

### 2.3 十问结论速记
1 不真正独立（LOW，M03/M04/M05 FUTURE + 接口未冻结 + R3 版本错位）／2 部分明确（M01 七函数、M05 六动作、DD-V2 九方法冻结度高，其余示例级）／3 几乎全部共享（memory-unit 28 全必填等，无模块私有 schema）／4 仅 M02 数值层可独立（07 §7.10 明列范围并排除 C40/M05）／5 42 参数 + 11 config，但 `ingest.*` 无载体、有未登记字段、热改未定义／6 9 场景改 Schema／7 1/2/4/9/14 需迁移（07 设计充分，NOT RUN ≠ 不可行）／8 8 场景影响多模块（57%）／9 8 项隐藏耦合（H-1…H-8）／10 不能（D-AP-03 禁止第六模块）。

### 2.4 8 项隐藏耦合（均给证据）
H-1 双 memory schema（28 vs 29 字段）／H-2 配置双位点镜像 + 跨 config locator／H-3 任务调度 owner 双重（M04 vs M01/Worker vs Dream）／H-4 LLM Gateway 唯一入口 vs M03 抽象 Provider／H-5 00 §7.3 标 M02→M04=C 但 03-m02/m04 写成必经 A／H-6 03-m0*.md 为 R3、00–15 为 R4.1／H-7 禁止第六模块 vs 可插拔目标／H-8 全必填字段 + 枚举在响应合同复制。
**单一所有者成立项**：Outbox/FTS5/LanceDB 归 M01 + Worker 消费（00 §7.3 规则4、03 §7.8、07 §7.12）——不计为耦合，是强项；仅 A/E 语义标注差记为 MEDIUM。

### 2.5 三分类与优先级纪律
- 每条结论按 EXISTING DESIGN（带文件+章节）/ DESIGN COMPLETENESS / RECOMMENDED ADDITION（NEW PROPOSAL）标注。
- §7 共 12 条改善方向全部标 **NEW PROPOSAL、未获批准**；E-8/E-10 明确"衔接 A/B 路 P0，不重复论证"；E-6 明确"不主张现在解锁 vector.store LOCKED"。
- 未把「未实测」当「不可行」（07 §13.1 全部 NOT RUN 只表示未演练）；未把「有名字」当「算法完整」（memory_tier 有字段但 A 路证明无转换规则 = P0）。

---

## 3. 与前轮 / 他路的衔接

- **A 路**（01-MEMORY-TIER-AND-LIFECYCLE-MATRIX.md）：场景 9 直接引用 A-C001–A-C004 为 P0 阻断；§7 E-10 为衔接项，不重做。
- **B 路**（02-LONG-TERM-MEMORY-STRUCTURE.md）：场景 4/6 与 §6 表3 引用 B-C001（relation 端点仅 memory_id、无 Project/情境对象、无自动联合召回）；§7 E-7/E-8 为衔接项。
- **FORENSIC**：仅引用其结论编号，未逐条复读 49 个 FC 条目。
- 本路独立贡献：**调整成本 / 模块独立性**视角，结论为「即使 A/B 路 P0 补齐，不做 E-1/E-2/E-3/E-12 解耦，每次能力演进仍外溢全链」。

---

## 4. 口径纠正记录（供主 Agent 汇总时留意）

- 任务书基线入口写「contracts\schemas\*.json（31 个）、contracts\config\*.json（12 个）」；**实测**：`contracts/config` = 11 个（无 `llm.schema.json`，15 §7.2 自认 P3 待建），`contracts/schemas` = 20 个，`contracts/` 合计 = 31 个。交付件 §9.B 已按实测记录。
- `03-m01.md` §9.4 仍写「M02 尚未设计」，与 `03-m02.md` 及 00 §7.6 已成形冲突 → 计入 H-6（版本错位），未据此推翻 M02 设计。
- 00 §7.3 把 Worker→Outbox 标 `R/W`、Worker→索引标 `A`，与 03 §7.8「索引写入 = 事务提交后异步（否）」的 E 语义差 → 计入表2 第 4 点（MEDIUM），未判为 P0。

---

## 5. 交付自检

- [x] 开工第一件事用 write 建交付件（IN PROGRESS + 目录），写完再读文件
- [x] 每章用 edit 追加，未憋到最后一次性写
- [x] STATUS 已改 COMPLETE
- [x] 分块读：先列标题（Select-String `^#{1,3} `），再 read offset/limit；单次最大 175 行 ≤350
- [x] 未读 node_modules/pydeps/evidence 无关目录；未通读 FORENSIC 49 FC
- [x] 只写 `E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\` 下两个文件；未改 Design 目录
- [x] 写文件只用 write/edit（无 Set-Content/Out-File/Add-Content，无 BOM）
- [x] 正文简体中文；字段名/枚举/模块名/章节号保留原文
- [x] 三分类标注 + P0/P1/P2 + 每个核心问题给推荐方案（NEW PROPOSAL）
- [x] 六张表 / 十四场景（六封闭类别）/ 十问 / ASCII 图 / 隐藏耦合 / 改善方向 / 结论 / 方法附录 齐备
- [x] 未问用户问题

**返回**：按 schema 给紧凑 JSON 摘要（headline ≤3 行 + ≤12 条 P0/P1 findings + handoffs）。
