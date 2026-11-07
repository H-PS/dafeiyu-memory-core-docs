# 06 — 修改历史与备份记录

**规则（本轮确立，后续沿用）**

1. **只读优先**：审查与索引不修改任何源文件（work / 时间流 / 备份全集 / canonical）。
2. **先备份再修改**：任何被修改的文件，必须先在同目录生成 `<原名>.bak-<yyyyMMdd-HHmmss>`，并在本文件登记（路径 / 原 SHA256 / 新 SHA256 / 原因 / 时间）。
3. **修改历史可查**：每次修改追加一行；不覆盖历史行。
4. **交付件自身也登记**：`Desktop\fin` 下所有产物的 SHA256 清单见 §3，便于分发后校验。

---

## 1. 本轮（Archive-Review-2026）修改记录

| 时间 | 文件 | 动作 | 备份 | 原 SHA256 | 新 SHA256 | 原因 |
|---|---|---|---|---|---|---|
| 2026-11-06 14:20 | `Desktop\fin\01-ARCHIVE-PLAN-FEASIBILITY.md` | 子 Agent 骨架（382 B）被父 Agent 全文替换 | 同路径覆盖，无源文件受影响 | `—`（骨架，无保留价值） | 见 `MANIFEST-SHA256.txt`（最终版） | A1 未产出，父 Agent 代拟 |
| 2026-11-06 | 其余 | — | — | — | — | 本轮其余全部为新建，未修改任何源文件 |

**说明**：`work\Memory-Core-R4.1-Design.backup-20261104-192558` 是**用户在 11-04 自建的备份**，不是本轮产物；本轮未触碰。

## 2. 历史备份线索（供追溯，非本轮创建）

| 位置 | 份数 | 时间 | 用途 |
|---|---:|---|---|
| `work\Memory-Core-R4.1-Design.backup-20261104-192558` | 66 | 11-04 19:25 | canonical 升级前快照 |
| `E:\[PROJECT]\备份全集\Memory-Core-R4.1-Design-pre-V1upgrade-20261104-1810` | 46 | 11-04 18:10 | V1 升级前，含 `_SHA256-BEFORE.csv` |
| `E:\[PROJECT]\备份全集\Memory-Core-R4.1-Design-20261104` | 50 | 11-04 | 设计前身 |
| `E:\[PROJECT]\备份全集\Memory-Core-R4.1-Design-V2-20261104` | 69 | 11-04 | V2 快照 |
| `时间流\…\_B-REVIEW-BACKUPS` | 50 | 11-03 | B 审查版本快照 |
| `时间流\…\psy-model\_backup\*` | 40+ 目录 | 11-02 | 心理模型迭代快照 |

## 3. 本轮产物清单（分发校验用）

> 生成后由 `Get-FileHash` 计算；若与你收到的不一致，说明传输被改。

| 文件 | 说明 |
|---|---|
| `00-NEW-INDEX-REPORT.md` | 新索引报告（主件） |
| `01-ARCHIVE-PLAN-FEASIBILITY.md` | 档案实施方案可行性 |
| `02-EVENT-FLOW-PROBLEM-INDEX.md` | 事件流问题索引（A2） |
| `02-EVENT-FLOW-PROBLEM-INDEX-APPENDIX.md` | 父 Agent 复核注（编号/时间/口径） |
| `03-IMPLEMENTATION-PLAN-FEASIBILITY.md` | 新版实施方案可行性 |
| `04-HISTORICAL-PLANS-AS-DIRECTIONS.md` | 历史方案 → 新方向 |
| `05-WORK-TIMELINE-BACKUP-CROSSWALK.md` | 三库对照 |
| `06-CHANGE-HISTORY.md` | 本文件 |
| `07-PRIOR-CAPABILITY-REVIEW-POINTER.md` | 上一轮能力审查指针（不复制） |
| `08-TOPIC-OLD-NEW-MATRIX.md` | 14 个知识主题 × 新/旧语料对照 |
| `MANIFEST-SHA256.txt` | 全部产物哈希 |

## 4. 本轮判定对"修改"的约束

- 本报告提出的整改（如重命名、补索引、移动目录）**一律标 NEW PROPOSAL，未执行**；执行前必须走 §1 流程。
- 不建议对 canonical 或时间流做任何原地改写；若需修订，应在**新目录**生成 vNext，并保留旧版。