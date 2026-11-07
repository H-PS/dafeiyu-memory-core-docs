# A-04-INPUT-MATERIAL-INDEX · 阶段四输入材料索引（4.0）

> 阶段：4.0 输入材料确认 | 时间：2026-11-03 19:10（Asia/Shanghai）
> 实测方式：Python `phase4_materials.py`（SHA-256 / size / mtime），非人工填写。
> 性质：本文件为阶段四输入基线，**只读登记**；不含技术判断。

## 一、正式审查依据（13 份，全部实测存在）

### 1.1 治理与前序报告

| 文件 | 大小 | mtime | SHA-256（前16） | 用途 |
|---|---:|---|---|---|
| A-REVIEW-PROGRESS.md（v1.11） | 32,138 | 11-03 18:52 | `ecfdc202bf3f22a4` | 治理基线 + §6.3 十一项验收标准 |
| A-00-MATERIAL-AUDIT.md | 13,431 | 11-03 15:46 | `77d98ee93a5546f0` | 材料范围与缺口 G-1~G-6 |
| A-01-EVENT-L3-REVIEW.md（v1.1） | 43,823 | 11-03 16:36 | `c4d8803fa5bddbc9` | Event/L3 结论、12.2 矩阵、A8 初判 |
| A-01-ISSUE-REGISTER.md（v1.1） | 20,047 | 11-03 16:35 | `c99bf38f341bab3a` | **既有问题权威定义** S1-01~S1-13 |
| A-02-EVIDENCE-CHAIN-REVIEW.md（v1.0） | 29,244 | 11-03 17:24 | `f1a5da17f7252a8e` | 证据链 S2-01~S2-07 |
| A-03-TWO-STAGE-K1-REVIEW.md（v1.0） | 35,656 | 11-03 18:15 | `4538056c3cd03d34` | 两阶段/K-1/S3-01~S3-06 |
| A-REVIEW-PROGRESS-v1.11-GOVERNANCE-CLOSURE.md | 4,426 | 11-03 18:53 | `4b7e224de5450b29` | 治理收尾 11 项验收通过 |

### 1.2 分类定义原始材料

| 文件 | 大小 | SHA-256（前16） | 阶段四用途 |
|---|---:|---|---|
| 01_EVENT_L3_REVIEW/v4.6-latest-design-candidate.md | 22,047 | `3928b8c94d7e5e31` | 12.2 矩阵、12.3、12.4 A8 六要素、13.2/13.4/13.5、第 10-12 条、T/U 系列 |
| 01_EVENT_L3_REVIEW/v4.6-latest-delta-spec.md | 6,495 | `44c0b707fa5db09a` | D-06 B1/B2 收窄定性（「实质性边界修订」） |
| 01_EVENT_L3_REVIEW/implementation-fact-checks.md | 4,076 | `630fc7b1b5efae2a` | F-CK-1~10 |
| 01_EVENT_L3_REVIEW/REVIEW_SCOPE.md | 6,920 | `9e4f1ce57eaa2bbc` | 十项问题索引（问题 3=A8、问题 4=状态语义） |
| 03_SOURCE_EXTRACTS/v4.5-extract-2.3.6.md | 11,802 | `d9f02d055ab0dd3e` | **A1-A8/B1-B4 原文定义、三分法、8 行对照表、聚合规则 1-6** |
| 03_SOURCE_EXTRACTS/v4.5-extract-2.3.10.md | 4,495 | `b5b76351d1a987b6` | 检查①②③ |
| 03_SOURCE_EXTRACTS/v4.5-extract-15-state-machine.md | 3,711 | `2b8d8c716537747d` | 10 态 + 7 迁移规则（含规则 5/6 落点） |
| 03_SOURCE_EXTRACTS/v4.5-extract-appendix-F.md | 4,737 | `65407169d353cb94` | F1~F8 事实 + F-CK 摘要 |
| 03_SOURCE_EXTRACTS/CORRECTION-DIFF-v4.5-extract.md | 8,355 | `6c8992a3347130c5` | O-1~O-6（A8 括注来源 O-2、O-5 缺失段） |
| 03_SOURCE_EXTRACTS/CORRECTION-DIFF-ROUND2-O5-O2.md | 8,364 | `258dca5ebb8b9690` | O-5 分类原则/8 行表补录、O-2 A8 来源归属 |
| 01_EVENT_L3_REVIEW/adjudication-ledger.md | 22,758 | `d1de733eecc74062` | D-AP-05/W-3 reinforce 禁写红线（权威裁决） |

## 二、证据等级说明（继承阶段 1/2 纪律）

- 〔原文级〕= 已与导出原文逐字核对；**但 G-2：纠错依据提取件不在包内，逐字声明不可独立复核**。本阶段所有引用 v4.5 摘录〔原文级〕处，均须同时标注 G-2。
- 〔归纳〕= 仅有概括、未逐字复核，不得当原文引用。
- 〔未能取得〕= 空缺不填补。
- 五级判断标注：〔事实〕/〔推导〕/〔设计缺口〕/〔依赖实现〕/〔无法核实〕。

## 三、材料缺失清单（不补造）

| 缺失项 | 影响 | 状态 |
|---|---|---|
| ORIGINAL-v4.5-fulltext-L125544-125966.md | 「逐字校正」不可独立复核（G-2） | 不存在 |
| ORIGINAL-v4.5-section-2.3.6.5-L125735-125791.md | 分类原则/8 行对照表无法对照原文（G-2/O-5） | 不存在 |
| CLAUDE-REVIEW-20261103.md | 上轮外部审查意见不可交叉核对（G-1） | 不存在 |
| 00_README/LOCAL_MATERIAL_INVENTORY.md | 材料清单自述不可核（G-6） | 不存在 |
| v4.5 附录 D~I | 相关条款未取得（G-3） | 未取得 |
| 15 态中其余 5 态 + 完整迁移图 | A 类命中后的状态落点核对受限（G-4） | 未取得 |
| X-01~X-14 反例定义 | 不得声称完成 X 系列验证（G-5） | 缺失 |
| M01/M02 schema | memoryId/memoryVersion 定义域不可核 | 域外 |

## 四、对话指令类依据（无磁盘文件，不得伪装为文件）

1. 《A-REVIEW-PROGRESS 修订增补》11 项验收标准 → 已转录 v1.11 §6.3（v1.11 sha `ecfdc202…`）；
2. 本阶段《阶段 4 启动补充要求：治理问题隔离与阶段门禁》→ 仅对话形态，记录于阶段四报告依据栏。

## 五、输入完备性结论

**阶段四可以开始。** 分类定义核心材料（v4.5 摘录 A/B 清单 + v4.6 12.2/12.4/13.x）齐备；缺失项均为**已知 G 系列缺口**，影响范围已在各分类审查中逐项标注，不阻塞分类边界审查（阻塞项集中在运行验证与原文逐字复核，均属「无法核实」，非「无法审查」）。
