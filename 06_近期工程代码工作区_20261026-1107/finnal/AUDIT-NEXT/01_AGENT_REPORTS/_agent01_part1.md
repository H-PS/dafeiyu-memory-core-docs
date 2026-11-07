# AGENT-01 · 文件级基线核验报告（B-06 × A-04 交叉基线审查）

| 项 | 内容 |
|---|---|
| **审查编号** | AGENT-01（文件基线与完整性审查专员） |
| **审查性质** | **只读**。未修改任何被审查文件；仅在 `E:\[PROJECT]\AUDIT-NEXT\01_AGENT_REPORTS\` 新增本报告与 4 份实测证据文件（CSV×3、脚本×3） |
| **审查日期** | 2026-11-03 20:25–21:05（Asia/Shanghai） |
| **审查对象** | B-06 系列 17 份（`C:\Users\[USER]\Downloads\`）+ `_B-REVIEW-BACKUPS\` 50 份 + A-04 系列（`E:\[PROJECT]\A-review\`，含 `_backups\`，共 137 份）+ 旁证材料（m01~m05、治理文件、PSYCH 实现目录、纲领文件） |
| **测量工具** | Python 3（bundled runtime，`hashlib.sha256` + 字节级解析）；全部数字为本次实测，**未抄写任何报告中的数字** |
| **总体结论** | 两套成果的**文件身份体系总体可信**：B-06 v3 §2.2 的 7 份历史报告 SHA、VERIFICATION 的 18 个受保护文件 SHA、A-04 的 17 份交付物 SHA **全部逐一复算吻合**；发现 **1 项 CONFIRMED 过时基线**（B-06 §1.2/§18/STAGE7-MANIFEST 的进度文件行）、**3 份"零变化代际"交付物**、**行数口径混用**、**B 侧脚本未留存**等 11 项编号发现（FB-01～FB-11） |

---

## 1. 审查范围与方法

### 1.1 实际执行的命令与脚本

以下为本报告全部结论所依据的实测命令/脚本（均保存于本目录）：

1. **清单与测量脚本**：`_agent01_measure.py`（本目录）。对每个文件读取原始字节，实测：字节数、SHA-256（完整 64 位）、BOM（EF BB BF / FF FE / FE FF）、严格 UTF-8 可解码性、CRLF/LF/CR 计数、四种行数口径：
   - **口径 A** = `text.splitlines()`（等价 .NET `ReadAllLines().Count`；文件以换行结尾时不含末尾空行）；
   - **口径 B** = LF 字符数；
   - **口径 C** = `text.split("\n")` 长度（文件以 LF 结尾时 = 口径 A + 1）;
   - **口径 D** = 正则 `\r\n|\r|\n` 切分长度。
   执行：`python _agent01_measure.py`，输出 `_agent01_baseline_main.csv`（17 份）、`_agent01_baseline_backups.csv`（50 份）、`_agent01_baseline_areview.csv`（137 份，含 `_backups\` 递归）。
2. **设计与治理文件复算**：PowerShell 内嵌 Python，对 `E:\[PROJECT]\review-staging\...01_BASELINE\` 与 `...\Targeted-Claude-Round2-20261103\` 下的 m01~m05、adjudication-ledger.md、05-conflict-register.md、06-interface-map.md、R3-Architecture-Decision-Register.md、adjudication-supplement.md 实测字节/行数/换行/SHA-256（输出见 §6）。
3. **引用存在性检查**：`_agent01_refcheck.py`——从 22 份关键报告逐行提取 `*.md|*.py|*.txt|*.json` 文件名引用，对 `C:\Users\[USER]\Downloads\`（含 `_B-REVIEW-BACKUPS\`）与 `E:\[PROJECT]\`（递归）建立的文件名索引核对；并单独实测 `F:\[PROJECT]\psy-model\` 目录。
4. **A-04 计数**：PowerShell `Get-ChildItem` 分类计数（`.md`/`.py`/`.txt`/`_backups`）。
5. **报告原文核对**：read/grep 逐条定位声明行号（详见各发现标注）。

### 1.2 测量时刻的目录快照

- `C:\Users\[USER]\Downloads\`（B-* 文件 17 份）与 `_B-REVIEW-BACKUPS\`（50 份）；
- `E:\[PROJECT]\A-review\` 根目录 78 份（21 `.md` 的 A-04-* + 4 份 A-0x + 3 份 A-REVIEW-PROGRESS 系 + 7 份 `.txt` + 41 份 `.py` + 2 份纲领外文件见 T3）+ `_backups\` 59 份；
- 旁证：`E:\[PROJECT]\review-staging\`（m01~m05 等多副本）、`F:\[PROJECT]\psy-model\`（44 个文件）。

> **测量时点声明**：全部实测完成于 2026-11-03 20:25 之后；若此后文件被改动，以本报告 CSV 为准。

---

## 2. 文件基线总表

> 以下 T1～T4 覆盖本次审查的**全部 204 个被测文件**（B 侧 67 + A 侧 137）。行数列同时给出三种口径（A/ReadAllLines、B/LF 字符数、C/split("\n")），供第 3 节口径分析直接比对。SHA-256 为完整 64 位实测值。

### T1 主表：B-06 系列（Downloads 根目录，17 份）

{TABLE_T1}

### T2 备份目录 `_B-REVIEW-BACKUPS\`（50 份）

{TABLE_T2}

### T3 A-review 根目录（78 份）

{TABLE_T3}

### T4 A-review `_backups\`（59 份）

{TABLE_T4}

### T5 旁证材料（本次实测，B 侧报告引用的设计/治理文件）

| 文件 | 实测路径（本报告核验所用副本） | 字节 | 行数(可见行) | LF | CRLF | SHA-256(前16) |
|---|---|---:|---:|---:|---:|---|
| m01.md | `E:\[PROJECT]\review-staging\R4.1-CR08A-Claude-Review-PREP-20261103-034025\01_BASELINE\Memory-Core\m01.md` | 41,376 | 1198 | 1197 | 1197 | `777a27e68635044a` |
| m02.md | 同上 `\01_BASELINE\Memory-Core\m02.md` | 33,819 | 868 | 867 | 0 | `d8e80b39d407fa48` |
| m03.md | 同上 `\01_BASELINE\Memory-Core\m03.md` | 15,582 | 464 | 463 | 463 | `fcd9f53d07c8ffe8` |
| m04.md | 同上 `\01_BASELINE\Memory-Core\m04.md` | 33,253 | 1195 | 1194 | 1194 | `da02ffe8782d8424` |
| m05.md | 同上 `\01_BASELINE\Memory-Core\m05.md` | 24,819 | 708 | 707 | 707 | `fe5af2623681670e` |
| adjudication-ledger.md | `E:\[PROJECT]\review-staging\R4.1-CR08A-Targeted-Claude-Round2-20261103\01_EVENT_L3_REVIEW\adjudication-ledger.md` | 22,758 | 392 | 391 | 0 | `d1de733eecc74062` |
| 05-conflict-register.md | `...\01_BASELINE\R4-Integration\05-conflict-register.md` | 10,404 | 77 | 76 | 0 | `87c3d6eb935dfc60` |
| 06-interface-map.md | `...\01_BASELINE\R4-Integration\06-interface-map.md` | 6,787 | 66 | 65 | 0 | `fda34e7cb51c8477` |
| R3-Architecture-Decision-Register.md | `...\01_BASELINE\Architecture\R3-Architecture-Decision-Register.md` | 12,407 | 120 | 119 | 0 | `1aa363b75c6a7e1c` |
| adjudication-supplement.md | `...\01_BASELINE\R4-Integration\adjudication-supplement.md` | 15,206 | 143 | 142 | 0 | `ed9292f6926be6cc` |

全部 10 份与 B-06 主报告 §1.1（L95–L103、L119）及 v3 §2.3（L70–L79）声称的「字节数/行数/CRLF/SHA-256(前16)」**逐项一致**（行数口径见 §3）。
