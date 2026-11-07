# 00_BASELINE · A-04 / B-06 联合审查治理任务 基线冻结

| 项 | 内容 |
|---|---|
| 建立时间 | 2026-11-03 20:18–20:35（Asia/Shanghai） |
| 工作目录 | `E:\[PROJECT]` |
| 交付根目录 | `E:\[PROJECT]\AUDIT-NEXT\` |
| 本轮性质 | **只读审查**。Phase 0–5 不修改任何被审查文件；如需修订，须经用户在 Phase 6 授权 |
| 基线冻结状态 | **已冻结**（本文件为唯一基线权威） |
| 测量工具 | PowerShell 5.1（`Get-FileHash` / `[System.IO.File]::ReadAllLines` / `[System.IO.File]::ReadAllBytes`） |
| 哈希算法 | SHA-256（**完整 64 位十六进制**） |

---

## 一、工作目录实际结构（实测）

```
E:\[PROJECT]\
├── A-review\                 # A-04 系列审查成果 + 脚本 + 取证件 + _backups
├── A-review-workspace\       # A 包原始材料（source/）
├── history-archive\          # R4.1 v4.5/v4.6 历史
├── release\                  # R4-Integration-20261102 发布包
├── review-staging\           # R4.1 CR08A 复审暂存包（含 m01~m05 基线）
├── _source\                  # 原始 zip 解包
├── AUDIT-NEXT\               # 本轮审查交付（本次创建）
├── HANDOFF-20261103-R4.1-CR08A.md
├── R4.1-V1-DELIVERY-CHARTER.md.md
├── top-记忆库项目设计宗旨与统一设计原则.md.md
├── limit.md / limit v2.md / limit v3.md
├── R4-Integration-20261102.zip
└── _s4_audit.py
```

**目录文件实测计数**：

| 范围 | 文件数 | 构成 |
|---|---:|---|
| `E:\[PROJECT]`（全递归） | 218 | 见 `_inventory_b06.csv` 与下文分项 |
| `E:\[PROJECT]\A-review`（仅顶层） | 78 | `.md` 30 / `.py` 41 / `.txt` 7 |
| `E:\[PROJECT]\A-review\_backups` | 59 | `.md` 备份 |
| `E:\[PROJECT]\A-review` 中 `A-04-*` | **21** | 全部 `.md` |

---

## 二、B-06 材料清单（基线，17 份，实测）

> 全部位于 `C:\Users\[USER]\Downloads\`。**完整 SHA-256 见 `_inventory_b06.csv`**。

| # | 文件 | 字节 | 行数 | 换行 | BOM | 编码 | 角色 |
|---:|---|---:|---:|---|---|---|---|
| 1 | B-00-MATERIAL-AND-BASELINE-AUDIT.md | 31,282 | 299 | LF | 无 | UTF-8 | 历史报告 |
| 2 | B-01-M01-STORAGE-REVIEW.md | 39,253 | 409 | LF | 无 | UTF-8 | 历史报告 |
| 3 | B-02-M02-RETRIEVAL-REVIEW.md | 99,247 | 1000 | LF | 无 | UTF-8 | 历史报告 |
| 4 | B-03-M03-EXTRACTION-REVIEW.md | 49,772 | 460 | LF | 无 | UTF-8 | 历史报告 |
| 5 | B-04-M04-CONSOLIDATION-REVIEW.md | 44,899 | 460 | LF | 无 | UTF-8 | 历史报告 |
| 6 | B-05-M05-SECURITY-REVIEW.md | 39,162 | 400 | LF | 无 | UTF-8 | 历史报告 |
| 7 | B-06-CROSS-MODULE-ARCHITECTURE-REVIEW.md | 60,124 | 801 | LF | 无 | UTF-8 | **被审查对象（主报告）** |
| 8 | B-06-EVIDENCE-MATRIX.md | 27,528 | 705 | LF | 无 | UTF-8 | 被审查对象（证据矩阵） |
| 9 | B-06-INTERFACE-TRACE.md | 14,348 | 249 | LF | 无 | UTF-8 | 被审查对象（接口追踪） |
| 10 | B-06-ISSUE-HANDOFF.md | 15,870 | 200 | LF | 无 | UTF-8 | 被审查对象（问题交接/台账） |
| 11 | B-06-STAGE7-INPUT-MANIFEST.md | 8,862 | 195 | LF | 无 | UTF-8 | 被审查对象（Stage7 Manifest） |
| 12 | B-06-PHASE1-READONLY-AUDIT.md | 49,205 | 759 | LF | 无 | UTF-8 | v2 历史版本 |
| 13 | **B-06-PHASE1-READONLY-AUDIT-v3.md** | **71,082** | **984** | LF | 无 | UTF-8 | **本轮核心审查报告 v3** |
| 14 | B-06-PHASE1-V2-SELF-CONSISTENCY-REVIEW.md | 29,441 | 494 | LF | 无 | UTF-8 | v2 自洽复核 |
| 15 | **B-06-PHASE1-V3-CHANGELOG.md** | **10,511** | **120** | LF | 无 | UTF-8 | **核心变更记录** |
| 16 | **B-06-PHASE1-V3-VERIFICATION.md** | **6,770** | **158** | LF | 无 | UTF-8 | **核心验证记录** |
| 17 | B-REVIEW-PROGRESS.md | 49,993 | 611 | LF | 无 | UTF-8 | 进度文件 |

**完整 SHA-256（前 16 位，辅助）**：

| 文件 | SHA-256（前16） |
|---|---|
| B-06-PHASE1-READONLY-AUDIT-v3.md | `cdeb4dd807608ff1` |
| B-06-PHASE1-V3-CHANGELOG.md | `e70ac85c220623d1` |
| B-06-PHASE1-V3-VERIFICATION.md | `a33799c11836b710` |
| B-06-PHASE1-READONLY-AUDIT.md（v2） | `02329c264cc31f62` |
| B-06-CROSS-MODULE-ARCHITECTURE-REVIEW.md | `1a77f4f12db489cf` |
| B-06-EVIDENCE-MATRIX.md | `e16b8e9138008ec2` |
| B-06-INTERFACE-TRACE.md | `cad41d238b74ccde` |
| B-06-ISSUE-HANDOFF.md | `2f6e705a8c734843` |
| B-06-STAGE7-INPUT-MANIFEST.md | `8264cff1e9582f5c` |
| B-REVIEW-PROGRESS.md | `6fcad56e17994d7c` |

---

## 三、B-06 历史备份（`C:\Users\[USER]\Downloads\_B-REVIEW-BACKUPS\`）

实测存在的备份族：

| 族 | 代际 | 说明 |
|---|---|---|
| B-06 交付物 | v1 / v2 / v3 / v4 × 5 份 = 20 | 逐代字节数见 Agent-01 报告 |
| B-06 AUDIT | v1 / v3-pre-1000line-fix | **v2 无独立备份**（即 `Downloads\B-06-PHASE1-READONLY-AUDIT.md` 本身） |
| B-00~B-05 | 各代 | — |
| B-PROGRESS | v1~v11 | **v8 的 SHA 前16 = `fb2923afaa173738`**（关键，见 §五） |

**已实测的关键事实**：`B-06-EVIDENCE-MATRIX` / `B-06-INTERFACE-TRACE` 的 v1=v2=v3=v4 字节数完全相同（27,528 / 14,348），即**四代之间内容未变**，属名义版本。

---

## 四、A-04 材料清单（基线，核心 16 份 + 扩展）

> 位于 `E:\[PROJECT]\A-review\`。**完整 SHA-256 见 `_baseline_a04.json`**。

| # | 文件 | 字节 | 行数 | SHA-256（前16） | 角色 |
|---:|---|---:|---:|---|---|
| 1 | A-04-CONSISTENCY-REVIEW.md | 40,072 | 495 | `95b82c4246d616e3` | **A-04 一致性复核主报告 v1.2** |
| 2 | A-04-CLOSURE-V1.2-VERIFICATION.md | 7,995 | 141 | `4bde91b4855c855e` | **A-04 验证记录** |
| 3 | A-04-CLOSURE-V1.2-CHANGELOG.md | 8,105 | 92 | `98814ccdfb4bcbcd` | **A-04 变更记录** |
| 4 | A-04-CLOSURE-STATUS.md | 18,764 | 299 | `b6b9ffee026d1669` | **A-04 收口状态报告 v1.2** |
| 5 | A-REVIEW-PROGRESS.md | 34,301 | 288 | `a10f6442b24f05ec` | 总进度 v1.12 |
| 6 | A-REVIEW-PROGRESS-CHANGELOG.md | 6,608 | 70 | `a7928d2f363317dd` | 总进度变更 |
| 7 | A-REVIEW-PROGRESS-v1.10-VERIFICATION.md | 7,904 | 111 | `8ab3da4f4ee4bb60` | v1.10 验证 |
| 8 | A-REVIEW-PROGRESS-v1.11-GOVERNANCE-CLOSURE.md | 4,426 | 64 | `4b7e224de5450b29` | v1.11 治理收口 |
| 9 | A-01-ISSUE-REGISTER.md | 20,047 | 186 | `c99bf38f341bab3a` | 问题登记册 |
| 10 | A-00-MATERIAL-AUDIT.md | 13,431 | 179 | `77d98ee93a5546f0` | 材料审计 |
| 11 | A-04-INPUT-MATERIAL-INDEX.md | 4,930 | 65 | `0fd3ac8567c00248` | 输入材料索引 |
| 12 | A-04-ACCEPTANCE-REPORT.md | 6,152 | 51 | `2dee439f1b3b1259` | 验收报告 |
| 13 | A-04-HANDOFF.md | 5,182 | 60 | `2e57091788261116` | A-04 交接 |
| 14 | A-04-CROSS-MATRIX.md | 7,405 | 112 | `8c97ce15c437dd23` | 跨文件矩阵 |
| 15 | A-04-ISSUE-DELTA.md | 8,449 | 99 | `e0686431aeb2818b` | 问题增量 |
| 16 | A-04-PREVIOUS-CONCLUSIONS-INHERITANCE.md | 5,363 | 43 | `805f4c6f8a2e4ea7` | 历史结论继承 |

**A-04 扩展材料**：`A-04-*` 共 **21** 个 `.md`；`_backups\` **59** 个备份；`.py` **41** 个；`_*.txt` 取证 **7** 个。

---

## 五、⚠ 基线层面的两处既存冲突（Phase 0 即发现，非本轮引入）

### 基线冲突 BL-01：B-REVIEW-PROGRESS 存在两个互斥的"基线"

| 声明处 | 声明值 | 实测/回溯结果 |
|---|---|---|
| **B-06 主报告 §1.2（L115）** | 43,057 字节 / 513 行 / SHA `fb2923afaa173738` | 对应 `_B-REVIEW-BACKUPS\B-PROGRESS-v8-pre-stage6-20261103.md`（43,057 B / 512 行）——**是历史快照，非当前文件** |
| **v3 §2.2（L64）** | 49,993 字节 / 612 行 / SHA `6fcad56e17994d7c` | 与**当前实际文件**一致（49,993 B / 611 行）✅ |

**判定**：v3 引用的是**当前值**（正确）；但 **B-06 主报告 §1.2 与 §18「阶段 0～5 报告读取」仍沿用 v8 的历史快照**，与 v3 引用值不一致。
**结论**：这不是 v3 的错误，而是 **v3 未发现的上游缺陷**——B-06 主报告的输入基线已过时，且 §18 声称"全部读取"所依据的字节数/行数并非当前文件。**新增发现**。

### 基线冲突 BL-02：全系统行数口径差 1（未声明口径）

实测：**除 B-02 外，所有 B 系列文件均以换行符结尾**。

| 口径 | 定义 | 示例（B-06 主报告） |
|---|---|---|
| 口径 A | `ReadAllLines().Count`（不含末尾空串） | **801** |
| 口径 B | `split("\n").Length`（含末尾空串） | **802** |
| 口径 C | LF 字符计数 | **801** |

**报告使用的实际是「口径 B」**（B-06 §1.1/§1.2、v3 §2.1/§2.2 的行数均 = 口径B）。
**B-02 是唯一例外**：文件不以换行结尾，故口径 A = 口径 B = 1000，报告值 1000 恰好两种口径都成立。

**判定**：行数本身**不是错误**（口径 B 是一种合法口径），但**报告从未声明使用哪种口径**，导致：
- v3 VERIFICATION 声称 "v2 行数 759"——口径 A/B 一致，✅
- v3 VERIFICATION 声称 "984 ≤ 1000 余量 16"——口径 B=985，**余量实为 15**；若按口径 A=984，余量 16。**口径未声明导致余量数字有 ±1 的不确定性**。
- **风险**：任何"行数一致"的 PASS 都建立在未声明的口径上，第三方复算会得到不同数字。

---

## 六、受保护文件清单（本轮冻结）

依据任务书 §2.1「只读优先」，以下文件**本轮不得修改**：

### P1 级保护（绝对禁止修改）
1. `C:\Users\[USER]\Downloads\B-06-*.md` 全部 17 份
2. `C:\Users\[USER]\Downloads\_B-REVIEW-BACKUPS\` 全部历史备份
3. `E:\[PROJECT]\A-review\A-0*.md`、`A-REVIEW-PROGRESS*.md` 全部
4. `E:\[PROJECT]\A-review\_backups\` 全部 59 份
5. `E:\[PROJECT]\review-staging\`、`E:\[PROJECT]\release\`、`E:\[PROJECT]\history-archive\`、`E:\[PROJECT]\_source\` 全部
6. `E:\[PROJECT]\A-review\*.py`（41 个脚本）与 `_*.txt`（7 个取证件）

### P2 级保护（未经本基线登记不得修改）
7. `E:\[PROJECT]\*.md`（根目录 8 份）

### 本轮允许创建的范围（仅此）
- `E:\[PROJECT]\AUDIT-NEXT\**`（全部为本轮新增交付）

---

## 七、审查深度定义（本轮统一口径）

| 级别 | 定义 | 本轮实际达到 |
|---|---|---|
| **L0** | 未审查 | — |
| **L1** | 元数据核验（字节/行数/SHA/编码） | 全部 17 B 文件 + 21 A 文件 |
| **L2** | 引用或特定内容核对 | 多数文件 |
| **L3** | 定点回源核验（回原文验证具体行号/文本） | m01~m05 相关段落、B-06 §11/§12/§14/§15/§18、证据矩阵全文 |
| **L4** | 全文审查 | **B-06 v3 主报告（984 行）、CHANGELOG、VERIFICATION、B-06 主报告（801 行）全文**；A-04 两主报告主要章节 |

**单文件主深度原则**：每文件只记一个主深度；章节级差异另列。

---

## 八、⚠ 本轮能力边界（必须如实披露）

| 项 | 实际状态 |
|---|---|
| 子代理并行 | ✅ **真实可用**（7 个专项代理已实际启动，非伪造） |
| 子代理独立性 | ⚠ **同一主代理编排下的子代理**，共享同一工作环境；**不构成外部第三方独立审计** |
| 原始设计文件 m01~m05 | ⚠ **仅定点回源**，未全文重审 |
| B-00~B-05 历史报告 | ⚠ **仅 SHA 核验 + 引用段落核对**，未全文重审 |
| 未修改任何受保护文件 | ✅ 已执行（Phase 0–5 全程只读） |
| 用户验收 | ❌ **未发生** |
| 下一阶段授权 | ❌ **未获得** |

---

## 九、基线结论

| 项 | 结论 |
|---|---|
| 基线是否冻结 | ✅ **已冻结**，可进入 Phase 1 |
| 是否存在缺失材料 | 否（17 份 B 文件、21 份 A-04 文件、59 份备份全部可读） |
| 是否存在读取失败 | 否（全部 UTF-8 可解码，无 BOM，无损坏） |
| 是否存在既存基线冲突 | **是，2 处**（BL-01 进度文件基线过时；BL-02 行数口径未声明） |
| 是否可支持多子代理并行 | ✅ 是，已启动 7 个 |

---

*基线冻结完成。下一阶段：Phase 1 子代理并行审查（结果见 `01_AGENT_REPORTS/`）。*
