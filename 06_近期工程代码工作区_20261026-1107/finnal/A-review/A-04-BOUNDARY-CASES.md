# A-04-BOUNDARY-CASES · 边界案例清单（阶段 4.3）

> 时间：2026-11-03 20:02 | 执行：主代理（整合 A/B/C 三代理 + A8 专项）
> 声明：以下案例除注明「原文级」外，均为**检验定义的假设测试输入**（非真实业务数据），用于验证分类判定的稳定性与唯一性。

## 一、唯一命中（应稳定归类）

| # | 假设输入 | 应归 | 依据 | 稳定性 |
|---|---|---|---|---|
| BC-01 | 证据缺 evidenceId | A1 → invalid | L51〔G-2〕 | 稳定 |
| BC-02 | 证据缺 correlationId | A1 → invalid | L51 | 稳定 |
| BC-03 | 快照含 (m1,v1)，versions 含 (m1,v1)+(m2,v2) | A2（多项）→ invalid | L52 | 稳定 |
| BC-04 | versions 含同一二元组两次 | A2（重复项）→ invalid | L52 | 稳定（**集合内**重复） |
| BC-05 | 引用指向快照中不存在的 (m9,v9) | A3（错配）→ invalid | L53 | 稳定 |
| BC-06 | componentMappings 内 component X 两条声明 | A5 → invalid | L55 | 稳定 |
| BC-07 | fragmentRefs 指向 fragments[] 中不存在的 fragmentId | A6 → invalid | L56 | 稳定（见 BC-15 判别） |
| BC-08 | fragmentDigest 与片段实际内容不符 | A7 → invalid | L57 | 稳定（判据未定义，见 A4-B-008） |
| BC-09 | mappingEvidenceRef 字段缺失 | B1 → unknown（13.5a，有效性可 passed） | L68/L100 | 稳定 |
| BC-10 | removalBasis 整体缺失 | B2 → unknown（不得判 removed） | L69/8 行表第 5 行 | 稳定 |
| BC-11 | 证据对象部分内容暂时不可读 | B3 → 版本 unknown | L70 | 较稳定（阈值未定义） |

## 二、多重命中（需判别）

| # | 假设输入 | 候选分类 | 判别结果 | 依据 |
|---|---|---|---|---|
| BC-12 | fragmentRefs 指向非快照版本的 memoryId | A3（错配）vs A4（绑定错位） | **A3 优先**（顺序句：先结构成立性含 A3，再绑定一致性 A4）〔G-2〕；v4.6 未明文 → A4-C-007 | L53/L54/L87 |
| BC-13 | fragmentRefs 指向**另一 assemblyId** 下存在的片段 | A4（跨 assemblyId 引用）vs A6 | **A4**（对象可取得但错位） | L54/L30 |
| BC-14 | 同一 fragmentId 在所有 assemblyId 下均不存在 | A6 | **A6**（对象不可取得 → 不满足 A4 前置） | L56/L54 |
| BC-15 | 片段不存在 ∧ 同时摘要不可比对 | A6 + A7 | **A6=failed（invalid，规则 1）+ A7=not_executable（不隐藏）** | 13.3/13.4a、T-04 |
| BC-16 | 证据外层损坏 ∧ mappingEvidenceRef 缺失 | A1 vs B1 | **A1 优先**（A1 明文排除清单只排 B1/B2 的**字段**，不排除整体结构损坏；结构成立性在顺序首位）〔G-2〕 | L51/L87 |
| BC-17 | A5 命中 ∧ DSH 同时主张「存在其他完整性错误」 | A5 vs A8 | **A5 应优先**（12.4f）；但**无程序门** → 实操可被记为 A8（降级通道）→ A4-B-013 | 12.4f/L55 |
| BC-18 | 证据整体不可读 | A1（证据对象不成立→invalid）vs B3（括注：证据缺失、不进入 completeness 判定） | **无优先级** → A4-C-003/P1 | L51/L70 |

## 三、无分类命中（缺口）

| # | 假设输入 | 应有归属 | 现状 | 编号 |
|---|---|---|---|---|
| BC-19 | versions[] 可解析，但条目内缺 memoryId | 应 invalid（版本不可识别） | A1 不命中（字段是子项）、A3 不管缺失 → 无归类 | **S2-03 / A4-A-001**（P2） |
| BC-20 | versions[] 可解析，但条目内缺 memoryVersion | 同上 | 同上 | 同上 |
| BC-21 | requiredComponents 数组内同一 component 出现两次 | 应 invalid（声明重复） | A5 只管 componentMappings → 无归类 | A4-B-011（P2） |
| BC-22 | fragmentDigest 字段整项缺失（非不符） | 应 fail-closed | A1 未列、A7 只管不符 → 无归类 | A4-B-008 邻域（P2） |
| BC-23 | mappingEvidenceRef 存在、对象存在、但内容格式非法/不可解析 | 应 fail-closed | A1（明文排除）、A4（非错位）、13.5c（对象可得）、B1（12.1e 只管语义充分性）均不命中 → **无归口** | **A4-C-001（P1）** |
| BC-24 | requiredComponents 为空数组 `[]` | 待裁决（invalid 还是 unknown） | 无归类，唯一路径 A8 兜底 | **P0-1a（待裁决）** |
| BC-25 | requiredComponents 字段缺失 / 无法解析 | 同上 | 同上 | **P0-1a** |
| BC-26 | 注入快照不可得（A2 无基准） | 应 not_executable 或 B4 | A2 无落点 | A4-B-012（P2） |

## 四、边界不稳定（微小变化导致分类跳变）

| # | 变化 | 跳变 | 合理性判定 |
|---|---|---|---|
| BC-27 | fragmentId 从「全不存在」→「在另一 assemblyId 存在」 | A6（invalid，无 not_executable 分支）→ A4（可能 not_executable） | **语义合理**，但后果跳变；判别式未写→实操风险 |
| BC-28 | mappingEvidenceRef 从「字段缺失」→「字段存在+对象不可得」 | B1 unknown（有效性可 passed）→ not_executable（不得 passed） | **符合第 12 条意图**（更严），但门槛跳变需业务方知情 |
| BC-29 | mappingEvidenceRef 从「对象可得+语义不足」→「默认缺失」 | B1 unknown（可 passed）→ B1 unknown（可 passed） | 稳定 |
| BC-30 | A7 从「摘要不符」→「摘要缺失」 | A7 invalid → 无归类（A1 未列） | **不合理**：同属摘要问题却一个有归类一个无 → A4-B-008 |
| BC-31 | requiredComponents 从 `[]` → 缺失 | 无分类 → 无分类（但语义不同） | 不稳定在语义层（P0-1a） |
| BC-32 | 同一错误归 A5 → 归 A8 | invalid → not_executable | **不合理**：编号选择改变整体结论 → A4-B-013 |

## 五、A8 四类边界（引自 A8 专项，均为假设测试）

| 类 | 案例 | 结论 |
|---|---|---|
| (a) 应归 A1–A7 却被 A8 接收 | BC-17（A5 被记 A8） | 违反 12.4f，且构成降级通道 |
| (b) 不应归 A1–A7 也不应归 A8 | 对象不可得（→not_executable）；部分不可读（→B3） | A8 须显式排除「不可取得/不可读」类 |
| (c) A8 合理接收 | 版本级 completeness 与 fragments 实际内容**可确认**不一致；fragmentDigest 整项缺失（若属必填） | A8 合理适用区=「已确认」错误 |
| (d) 无法判定归属 | BC-19/BC-20（memoryId/Version 缺失）；BC-23（不可解析） | 须走三分法第 8 句 fail-closed，不得由 A8 吸收 |

## 六、统计与结论

| 类别 | 数量 | 其中缺口/问题 |
|---|---:|---|
| 唯一命中（稳定） | 11 | 0（BC-08 判据待定义、BC-11 阈值待定义） |
| 多重命中（需判别） | 7 | BC-12/13/14/15 可判别（BC-15 已闭环）；BC-17/18 隐患 |
| 无分类命中 | 8 | **P1×1（BC-23）**、待裁决×1（BC-24/25）、P2×5 |
| 边界不稳定 | 6 | BC-27/30/32 为实质问题，BC-28 为设计意图 |

**核心结论**：
1. 分类体系在**明确命中区（BC-01~11）**是稳定的——A1–A8 的定义足以支撑「典型错误」的唯一归类；
2. **缺口集中在「定义边缘」**：子字段缺失（BC-19/20）、非映射类兜底（BC-21/22）、不可解析（BC-23）、快照不可得（BC-26）；
3. **最危险的不是分类错误，而是编号选择改变结论**（BC-17/32）——A5/A6 无 not_executable 分支、A8 有，构成系统性降级通道；
4. **P0-1a（BC-24/25）不裁决会被 A8 默认决定**。
