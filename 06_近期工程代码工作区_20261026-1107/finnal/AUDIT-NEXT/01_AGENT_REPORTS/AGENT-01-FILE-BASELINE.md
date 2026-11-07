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

| 文件 | 字节 | 行数A(splitlines/ReadAllLines) | LF字符数 | split('\n')长度 | 编码 | BOM | 换行符 | SHA-256(完整) |
|---|---|---|---|---|---|---|---|---|
| B-00-MATERIAL-AND-BASELINE-AUDIT.md | 31282 | 299 | 299 | 300 | UTF-8 no BOM | NONE | LF | bdf76458a26730e7524a3f52c6dc0d8164540adba9ffa815eda9811f9c4759ad |
| B-01-M01-STORAGE-REVIEW.md | 39253 | 409 | 409 | 410 | UTF-8 no BOM | NONE | LF | 640b7ecd316b09143481acad5790c0fd0f3479cc95e44fa3e72b25a7fad89150 |
| B-02-M02-RETRIEVAL-REVIEW.md | 99247 | 1000 | 999 | 1000 | UTF-8 no BOM | NONE | LF | a66058c5847d15d19faba5fb68113e1a301630fffee351cbd8f3adaaca121955 |
| B-03-M03-EXTRACTION-REVIEW.md | 49772 | 460 | 460 | 461 | UTF-8 no BOM | NONE | LF | 8a3b347cd14ed49f2eef1b519971cc89dd450be5169aeceed1498e13ea4cc02f |
| B-04-M04-CONSOLIDATION-REVIEW.md | 44899 | 460 | 460 | 461 | UTF-8 no BOM | NONE | LF | 8eb5a0694171c63c3cec650f89a01721330fa41a4f0279df8cb2bbbf3fe1fb23 |
| B-05-M05-SECURITY-REVIEW.md | 39162 | 400 | 400 | 401 | UTF-8 no BOM | NONE | LF | a10198bbb32032dee30357c2dbdc98f99def8198888b18afb8844e37e21c5e14 |
| B-06-CROSS-MODULE-ARCHITECTURE-REVIEW.md | 60124 | 801 | 801 | 802 | UTF-8 no BOM | NONE | LF | 1a77f4f12db489cfcbc4a540c790fe68c45546a92cfdddf3f34a0eb9348ad8bb |
| B-06-EVIDENCE-MATRIX.md | 27528 | 705 | 705 | 706 | UTF-8 no BOM | NONE | LF | e16b8e9138008ec2e1b3be729b71e5bb70be7302b8a9567a9bcfa6d9155517e4 |
| B-06-INTERFACE-TRACE.md | 14348 | 249 | 249 | 250 | UTF-8 no BOM | NONE | LF | cad41d238b74ccde598b84a60b8d29552ca81aa0066e4937c02ebe6701558f2b |
| B-06-ISSUE-HANDOFF.md | 15870 | 200 | 200 | 201 | UTF-8 no BOM | NONE | LF | 2f6e705a8c734843a915ac585352496a446cf0c356e6489eaaa9c34f14369d68 |
| B-06-PHASE1-READONLY-AUDIT-v3.md | 71082 | 984 | 984 | 985 | UTF-8 no BOM | NONE | LF | cdeb4dd807608ff130b26f7eaf8a1c1318719ca13ed26c2445a1718f78db9b1a |
| B-06-PHASE1-READONLY-AUDIT.md | 49205 | 759 | 759 | 760 | UTF-8 no BOM | NONE | LF | 02329c264cc31f62852cfc5918c4c9e9627e0e34e7c56951e0870015b549cd52 |
| B-06-PHASE1-V2-SELF-CONSISTENCY-REVIEW.md | 29441 | 494 | 494 | 495 | UTF-8 no BOM | NONE | LF | 8dd7cfbf9defe4fe6877a352b28ed426871ef82c21b768ad125755d3f76af4d8 |
| B-06-PHASE1-V3-CHANGELOG.md | 10511 | 120 | 120 | 121 | UTF-8 no BOM | NONE | LF | e70ac85c220623d1e6b5647b11af07c8d8eda2ad12a590fe547cdff55ed54157 |
| B-06-PHASE1-V3-VERIFICATION.md | 6770 | 158 | 158 | 159 | UTF-8 no BOM | NONE | LF | a33799c11836b710cf3f4feb8f8712c5912eaf857fdfdeb39ce9a88b86dc92fd |
| B-06-STAGE7-INPUT-MANIFEST.md | 8862 | 195 | 195 | 196 | UTF-8 no BOM | NONE | LF | 8264cff1e9582f5c2cff460c3845aaa40de66346717d6feb313600a808309c18 |
| B-REVIEW-PROGRESS.md | 49993 | 611 | 611 | 612 | UTF-8 no BOM | NONE | LF | 6fcad56e17994d7c3e8af916151a11c976b5d2c5ccb1868dedd65a0342780308 |

### T2 备份目录 `_B-REVIEW-BACKUPS\`（50 份）

| 文件 | 字节 | 行数A | LF数 | splitLF | 换行符 | SHA-256(完整) |
|---|---|---|---|---|---|---|
| B-00-v1-20261103.md | 31282 | 299 | 299 | 300 | LF | bdf76458a26730e7524a3f52c6dc0d8164540adba9ffa815eda9811f9c4759ad |
| B-01-FULL-v1-20261103.md | 96898 | 904 | 904 | 905 | LF | c20a8bd17b6e919b9581c63109f50d096902bc8ae63a73a900ecbe43f5d8f20d |
| B-01-LITE-v1-20261103.md | 39253 | 409 | 409 | 410 | LF | 640b7ecd316b09143481acad5790c0fd0f3479cc95e44fa3e72b25a7fad89150 |
| B-02-v1-20261103.md | 89985 | 995 | 995 | 996 | LF | 11ae8357eeac5ed64d91346f488bb35cd03f0a61908949f4e751248935cef7f3 |
| B-02-v2-20261103.md | 96925 | 1050 | 1050 | 1051 | LF | d394ad68a41537b1a34fc96803476cc3996f56f4dd4eef3994f6dccf2892942f |
| B-02-v3-final-20261103.md | 115923 | 1267 | 1267 | 1268 | LF | b12495c3bae91ec175ba31e7e7993041607c6b1ea9dbe8381893066911b31027 |
| B-02-v4-DELIVERED-20261103.md | 121273 | 1307 | 1307 | 1308 | CRLF | 1bfcd8d35b1a68a75ac349d38ae2adba85c2f067decd21213994389d15bee511 |
| B-02-v5-pre-eol-fix-20261103.md | 121273 | 1307 | 1307 | 1308 | CRLF | 1bfcd8d35b1a68a75ac349d38ae2adba85c2f067decd21213994389d15bee511 |
| B-02-v6-pre-condense-20261103.md | 119966 | 1307 | 1307 | 1308 | LF | c524dc85c608944977280b4aa4f72f244b8062edec3d0dc8e8ee715984df7233 |
| B-02-v7-condensed-20261103.md | 81249 | 952 | 952 | 953 | LF | 9f2398101cf1c8d57a1d0dd4a043b299ed3207949b5bd4f34014363b0cad5dd8 |
| B-02-v8-pre-final-revision-20261103.md | 81249 | 952 | 952 | 953 | LF | 9f2398101cf1c8d57a1d0dd4a043b299ed3207949b5bd4f34014363b0cad5dd8 |
| B-02-v9-final-20261103.md | 99247 | 1000 | 999 | 1000 | LF | a66058c5847d15d19faba5fb68113e1a301630fffee351cbd8f3adaaca121955 |
| B-03-v1-20261103.md | 49772 | 460 | 460 | 461 | LF | 8a3b347cd14ed49f2eef1b519971cc89dd450be5169aeceed1498e13ea4cc02f |
| B-04-v1-20261103.md | 44899 | 460 | 460 | 461 | LF | 8eb5a0694171c63c3cec650f89a01721330fa41a4f0279df8cb2bbbf3fe1fb23 |
| B-04-v2-20261103.md | 44899 | 460 | 460 | 461 | LF | 8eb5a0694171c63c3cec650f89a01721330fa41a4f0279df8cb2bbbf3fe1fb23 |
| B-05-v1-20261103.md | 39162 | 400 | 400 | 401 | LF | a10198bbb32032dee30357c2dbdc98f99def8198888b18afb8844e37e21c5e14 |
| B-06-PHASE1-READONLY-AUDIT-v1-20261103.md | 32331 | 483 | 483 | 484 | LF | 2b2cc00671682e24e9c24ca23c6c2edc380ecf7f4b59ba280aa287c5b76abae4 |
| B-06-PHASE1-READONLY-AUDIT-v3-pre-1000line-fix-20261103.md | 70462 | 1020 | 1020 | 1021 | LF | 4c913583567e754c3a5a5e4fe798dc48e65862aca2a6b25fa98e0f3a18079021 |
| B-06-v1-CROSS-MODULE-ARCHITECTURE-REVIEW-20261103.md | 44188 | 636 | 636 | 637 | LF | 5f4f88c999294da7130975fad7c5e9301f1364da24a8470c2460b9fb90d392b9 |
| B-06-v1-EVIDENCE-MATRIX-20261103.md | 27528 | 705 | 705 | 706 | LF | e16b8e9138008ec2e1b3be729b71e5bb70be7302b8a9567a9bcfa6d9155517e4 |
| B-06-v1-INTERFACE-TRACE-20261103.md | 14348 | 249 | 249 | 250 | LF | cad41d238b74ccde598b84a60b8d29552ca81aa0066e4937c02ebe6701558f2b |
| B-06-v1-ISSUE-HANDOFF-20261103.md | 12931 | 187 | 187 | 188 | LF | f1df0563d53374e33fc377b45d2b6f4af7e2cc1f65f601162630de97b83c3102 |
| B-06-v1-STAGE7-INPUT-MANIFEST-20261103.md | 8804 | 195 | 195 | 196 | LF | 2c7ad8f124ee735fc2c0c09f163c675c18206eeedc49d5634b521478f4c01472 |
| B-06-v2-CROSS-MODULE-ARCHITECTURE-REVIEW-20261103.md | 50270 | 693 | 693 | 694 | LF | 869723b76c932554f03ce9c0131b2b748aac1027d0b1c5e0563620d4344da802 |
| B-06-v2-EVIDENCE-MATRIX-20261103.md | 27528 | 705 | 705 | 706 | LF | e16b8e9138008ec2e1b3be729b71e5bb70be7302b8a9567a9bcfa6d9155517e4 |
| B-06-v2-INTERFACE-TRACE-20261103.md | 14348 | 249 | 249 | 250 | LF | cad41d238b74ccde598b84a60b8d29552ca81aa0066e4937c02ebe6701558f2b |
| B-06-v2-ISSUE-HANDOFF-20261103.md | 13911 | 192 | 192 | 193 | LF | 9c8e6f66aacb3707840211b551537e62d36ed3237c6b864425ea33258c7d93c1 |
| B-06-v2-STAGE7-INPUT-MANIFEST-20261103.md | 8861 | 195 | 195 | 196 | LF | 50126e3c02dcdd7729a6d5d75fb162f7f4533456a1a6157b730b484ddc392f54 |
| B-06-v3-CROSS-MODULE-ARCHITECTURE-REVIEW-20261103.md | 53972 | 725 | 725 | 726 | LF | 65a755e98d46f8085315f9d419548b137aac8e7fc2a9ba886c8cfdf30392b25f |
| B-06-v3-EVIDENCE-MATRIX-20261103.md | 27528 | 705 | 705 | 706 | LF | e16b8e9138008ec2e1b3be729b71e5bb70be7302b8a9567a9bcfa6d9155517e4 |
| B-06-v3-INTERFACE-TRACE-20261103.md | 14348 | 249 | 249 | 250 | LF | cad41d238b74ccde598b84a60b8d29552ca81aa0066e4937c02ebe6701558f2b |
| B-06-v3-ISSUE-HANDOFF-20261103.md | 14339 | 194 | 194 | 195 | LF | ea5ed53d3220800cf97e8263faa65f075518f49b38679df8188404e3887bb4e5 |
| B-06-v3-STAGE7-INPUT-MANIFEST-20261103.md | 8862 | 195 | 195 | 196 | LF | 8264cff1e9582f5c2cff460c3845aaa40de66346717d6feb313600a808309c18 |
| B-06-v4-CROSS-MODULE-ARCHITECTURE-REVIEW-20261103.md | 60124 | 801 | 801 | 802 | LF | 1a77f4f12db489cfcbc4a540c790fe68c45546a92cfdddf3f34a0eb9348ad8bb |
| B-06-v4-EVIDENCE-MATRIX-20261103.md | 27528 | 705 | 705 | 706 | LF | e16b8e9138008ec2e1b3be729b71e5bb70be7302b8a9567a9bcfa6d9155517e4 |
| B-06-v4-INTERFACE-TRACE-20261103.md | 14348 | 249 | 249 | 250 | LF | cad41d238b74ccde598b84a60b8d29552ca81aa0066e4937c02ebe6701558f2b |
| B-06-v4-ISSUE-HANDOFF-20261103.md | 15870 | 200 | 200 | 201 | LF | 2f6e705a8c734843a915ac585352496a446cf0c356e6489eaaa9c34f14369d68 |
| B-06-v4-STAGE7-INPUT-MANIFEST-20261103.md | 8862 | 195 | 195 | 196 | LF | 8264cff1e9582f5c2cff460c3845aaa40de66346717d6feb313600a808309c18 |
| B-PROGRESS-v1-stage1-20261103.md | 15136 | 146 | 146 | 147 | LF | 34029e2520c1fd6d8d32c527eecf16a2940a3705d96a8b639b4fb60f191a3f3a |
| B-PROGRESS-v10-pre-stage6-final2-20261103.md | 48633 | 602 | 602 | 603 | LF | ffb7faf0281239321520609bd7cede0c574c55e47e6ce8df46ac515aaa77ccb0 |
| B-PROGRESS-v11-pre-stage6-redteam-20261103.md | 48931 | 604 | 604 | 605 | LF | 26af24c07d06f7d9c9869dc8b5b2ca9c977c4678e2a49ab982ec3912f4487b63 |
| B-PROGRESS-v2-20261103.md | 17004 | 169 | 169 | 170 | LF | f56ad28788c7d7d13c40fd225c139082ebcc980170fb1b3043857d6e9fbcebfc |
| B-PROGRESS-v3-pre-stage2-20261103.md | 17004 | 169 | 169 | 170 | LF | f56ad28788c7d7d13c40fd225c139082ebcc980170fb1b3043857d6e9fbcebfc |
| B-PROGRESS-v4-pre-stage2-final.md | 17004 | 169 | 169 | 170 | LF | f56ad28788c7d7d13c40fd225c139082ebcc980170fb1b3043857d6e9fbcebfc |
| B-PROGRESS-v5-stage2-done-20261103.md | 16655 | 200 | 200 | 201 | LF | bfb766177ef8f1ff92a3d3fb2aaee6c89d6b5305664c61a3f11271f63ed6d845 |
| B-PROGRESS-v6-limitv2-20261103.md | 21392 | 250 | 250 | 251 | LF | fb72740b534b1ed15ad7b43bb9b6544fc41c1c618aa383d2d310c4a0aaa63669 |
| B-PROGRESS-v7-pre-stage5-20261103.md | 39277 | 470 | 470 | 471 | LF | 9f1312afa5ebff96b6ee0a36be4d1920dd4c637bc9230d0415ca470b869b5355 |
| B-PROGRESS-v8-pre-stage6-20261103.md | 43057 | 512 | 512 | 513 | LF | fb2923afaa1737380f54040d803ee7ae6664bf7d99f1ccab84352d79490f07e3 |
| B-PROGRESS-v9-pre-stage6-final-20261103.md | 48156 | 599 | 599 | 600 | LF | ca2835c1db78da0ecc6971802a32c0cce90f10188609d227cd24630a0a35bbbb |
| RRF-PAPER-EXCERPT-20261103.txt | 1034 | 24 | 24 | 25 | MIXED(CRLF=1,LF=23,CR=0) | 56f8f9fed464b8f937271a8f5d601ca012d57eca4e6e08749d0a8b9f52ab5fe8 |

### T3 A-review 根目录（78 份）

| 文件 | 字节 | 行数A | LF数 | splitLF | 换行符 | SHA-256(完整) |
|---|---|---|---|---|---|---|
| A-00-MATERIAL-AUDIT.md | 13431 | 178 | 178 | 179 | LF | 77d98ee93a5546f0caf9109808d5833420e7a2b1a4ac17d8c855eb08a8381fb7 |
| A-01-EVENT-L3-REVIEW.md | 43823 | 341 | 341 | 342 | LF | c4d8803fa5bddbc903fd7f929f63bc701e3c724d6a7ee16b1b4bd8d8d2a8d1fe |
| A-01-ISSUE-REGISTER.md | 20047 | 185 | 185 | 186 | LF | c99bf38f341bab3a5650ff69df9858071e3f80b336c7a71862e28275f61fcd93 |
| A-02-EVIDENCE-CHAIN-REVIEW.md | 29244 | 229 | 229 | 230 | LF | f1a5da17f7252a8e0ba78af8fb34ddfc6375f31e51e551c08048311aa4111f4c |
| A-03-TWO-STAGE-K1-REVIEW.md | 35656 | 289 | 289 | 290 | LF | 4538056c3cd03d3408c125533f2aa7d71933cf1d38f8f87c78152f9e21703040 |
| A-04-A1-A4-CLASSIFICATION-REVIEW.md | 7589 | 75 | 75 | 76 | LF | a4c1f6d027327c2c934b04a34921dfbd8096912412b8822c71787f5875423228 |
| A-04-A5-A8-CLASSIFICATION-REVIEW.md | 14978 | 99 | 99 | 100 | LF | b906f4de2285bb10d04dbec6d07356ff0c1ecca964f2aa9e0c2ec66f0eb0f0bd |
| A-04-A8-FOCUSED-REVIEW.md | 9394 | 111 | 111 | 112 | LF | e897c45a9f79e3bd5cb824953bb72dde14ec9b4f2ec3bb21ef5f93cf28983dee |
| A-04-ACCEPTANCE-REPORT.md | 6152 | 50 | 50 | 51 | LF | 2dee439f1b3b1259cfe1669e4a88622f6b41ae6952c636801d149cfc5b43de04 |
| A-04-B1-B2-SCOPE-REVIEW.md | 6781 | 67 | 67 | 68 | LF | 4c55e5ee35ecee75289552275623e5e7ac110092dd37dc4d8e020d52b12463db |
| A-04-B1-B4-CLASSIFICATION-REVIEW.md | 11551 | 86 | 86 | 87 | LF | 9c7a6df79fcef681ce5f12984487b359ef54a12ef9bb5915f43534b43d502774 |
| A-04-BOUNDARY-CASES.md | 7257 | 80 | 80 | 81 | LF | 3011c277ec6c300df584c4c884e00789e56c50feba46c794d130e667ee2c32a3 |
| A-04-CLASSIFICATION-REVIEW.md | 16211 | 190 | 190 | 191 | LF | 6639ecaf0688277bb2d92e1b95e058d6b863dc53ccd5cb93aa5ac7bc481a529f |
| A-04-CLOSURE-STATUS.md | 18764 | 299 | 299 | 300 | LF | b6b9ffee026d1669af9142d13c3bfcc49385fec542711d42d63b08263ad2290a |
| A-04-CLOSURE-V1.2-CHANGELOG.md | 8105 | 91 | 91 | 92 | LF | 98814ccdfb4bcbcdf256cfce5d99738b41e0b7c82709ff743df2cb36a0a629e3 |
| A-04-CLOSURE-V1.2-VERIFICATION.md | 7995 | 140 | 140 | 141 | LF | 4bde91b4855c855ee0508c886425b74bf3008504d733e22ae9007b724c658f9c |
| A-04-CONSISTENCY-REVIEW.md | 40072 | 495 | 495 | 496 | LF | 95b82c4246d616e3cfda96535fe296b40a5f344e11e12be6fa6fd2c41afa0170 |
| A-04-CROSS-MATRIX.md | 7405 | 111 | 111 | 112 | LF | 8c97ce15c437dd23ea16a13f972551a71ee66c92d795e26bb62ebd830aa2a9c2 |
| A-04-HANDOFF.md | 5182 | 59 | 59 | 60 | LF | 2e5709178826111675f0b20d2ecaa45ef5052664b40ae8326d6b56d70e90acd8 |
| A-04-INPUT-MATERIAL-INDEX.md | 4930 | 64 | 64 | 65 | LF | 0fd3ac8567c002487eabb76fc946ed85446ab6052379e90ce3783828cb56c377 |
| A-04-ISSUE-DELTA.md | 8449 | 98 | 98 | 99 | LF | e0686431aeb2818b358a369e775923c3c294fad33c3bdbbeae1eb5086c4a174d |
| A-04-NOT-EXECUTABLE-REVIEW.md | 6842 | 81 | 81 | 82 | LF | 05ec342a684f80f28032ac511371c989456a3e13dfd05f66aa7f0029033d91ff |
| A-04-PREVIOUS-CONCLUSIONS-INHERITANCE.md | 5363 | 42 | 42 | 43 | LF | 805f4c6f8a2e4ea7459b277b2badb71509d6918cb3cb1a94a20f7270cbeb624b |
| A-04-STATE-CONSISTENCY-REVIEW.md | 6129 | 71 | 71 | 72 | LF | 517ca1731ba9f8382470d38832da3e6cf5bd6e0ad06a7a360d31dcc36a10be30 |
| A-04-STATIC-EXECUTABILITY-REVIEW.md | 5519 | 75 | 75 | 76 | LF | b5a587d3dcaeee5fd40af1e3bbe39a0e07f4ab8464394724e0cf4b89870b8057 |
| A-04-V1-FUTURE-CLASSIFICATION.md | 7374 | 79 | 79 | 80 | LF | 4866eaf9aad772a5f2b1e8e410114c15acbc3170862433aa2904af2b789f8453 |
| A-REVIEW-PROGRESS-CHANGELOG.md | 6608 | 69 | 69 | 70 | LF | a7928d2f363317dda6ad0631d6402f87b14f852b0523c2c5e59525f91dc58e1e |
| A-REVIEW-PROGRESS-v1.10-VERIFICATION.md | 7904 | 110 | 110 | 111 | LF | 8ab3da4f4ee4bb608a3e59f3a496c04a299edfcb387f215359ee3e8dff7fc960 |
| A-REVIEW-PROGRESS-v1.11-GOVERNANCE-CLOSURE.md | 4426 | 63 | 63 | 64 | LF | 4b7e224de5450b291403cb42907f53e5b7c60ca8e7573a5197fce039d0c027cc |
| A-REVIEW-PROGRESS.md | 34301 | 288 | 288 | 289 | LF | a10f6442b24f05ecea37c674ca0454ef740a9543943d6f8e285fbbf17281c1af |
| _consistency_evidence.txt | 9528 | 116 | 115 | 116 | CRLF | 8f689dbcfd176e732f8da17dcaf1d44b82988b1f3b38500bee2bc448ed6808d7 |
| _consistency_evidence2.txt | 4304 | 49 | 48 | 49 | CRLF | 82a679a711db3e697e92aaa24ff7f9406d2bda647c8c7cfc6d3085f09ad99752 |
| _secondpass_evidence.txt | 9310 | 94 | 93 | 94 | CRLF | 257954d4f3488b3eece91c2e6d97bc4bbfe32b81f4ba03eefa93c9e4f6154d92 |
| _secondpass_result.txt | 1045 | 25 | 25 | 26 | LF | 1067e5d57ff2eea1e7b419a52482bbdaa183d18302dd9e44dc851faf2bffda1b |
| _v12_facts.txt | 10333 | 131 | 130 | 131 | CRLF | 69f89919d2d4aed2ef4a65704fb83715793911b5a7714388055b436ac3b10700 |
| _v12_final_sha.txt | 1447 | 41 | 41 | 42 | LF | 0c4d03c07a63d51d97728cb74f99b20b061b22a816bd3390dcb1039033564b4a |
| _v12_verify.txt | 15689 | 185 | 184 | 185 | CRLF | 217b6f0c56c6c174a9d3a60550a0e1ec80a4f83be52748d2258e3eccfd49777a |
| check_structure.py | 1166 | 32 | 32 | 33 | LF | 093891c6305307ad024e4a9863a033429545b6e0fdd31e6bd926d2e7b48b9bbd |
| closure_count_check.py | 834 | 21 | 21 | 22 | LF | d1a015b7125b7802ca04262dde6c63816a0e86f0894ad479ffa3024c48d8385b |
| closure_finalize.py | 1391 | 37 | 37 | 38 | LF | ee7e9d38dfb8dbc6482c857a5858201d3015db4992cfd04787a0d649df6038d6 |
| consistency_backup.py | 952 | 25 | 25 | 26 | LF | 8ab166e92c445dd91fa45e8570afad1053c5bb9bda2284ce699629eb4a79a7bf |
| consistency_files.py | 1504 | 36 | 36 | 37 | LF | 94107c4437053254886da16066cdcc3463695e633b8ea45f59a18330ee9d59eb |
| consistency_probe.py | 1841 | 42 | 42 | 43 | LF | 73de6c6a05dbe24017f96320ab2edac6e12083db10b340a6d61448ed16aada65 |
| consistency_probe2.py | 4685 | 129 | 129 | 130 | LF | 3a9f560a6a7d793ffd72ccea457bd9330d4f7a2789eb3ba0ef9258ae065970e4 |
| consistency_probe3.py | 2091 | 47 | 47 | 48 | LF | 0969cfce4256ca8107bf4cd24362465d34c0060a9af5439b124bc6ee9f12772c |
| finalize_phase1.py | 2699 | 67 | 67 | 68 | LF | 3cd8d39073b30f82e6d1aceb3baf26fd450738383e5faa614e36f7d1b5e661c3 |
| fix_31.py | 5982 | 83 | 83 | 84 | LF | 455b2193f8fcae799ca49facd33e5c7ea720f9c95ef40f0611299e83e2c35dc9 |
| fix_52.py | 2646 | 57 | 57 | 58 | LF | 561018d71da75d8176e66fb38806f1533f3407bb07bb2385c8065287a1b946c5 |
| phase2_facts.py | 2040 | 60 | 60 | 61 | LF | c30c7fc9cea8d736037d868a78847d661b3337c7b2be4fc16d26a9125c28f822 |
| phase4_accept_check.py | 3532 | 73 | 73 | 74 | LF | c0f278bc89e24bb0fa501304fae8212e0b530be7863e50b8248a076e8c3a9011 |
| phase4_backup.py | 1174 | 35 | 35 | 36 | LF | 33d49457cd367c1abd478266394b75a0743aa564a05e0260b908e302ce2672d9 |
| phase4_main_verify.py | 1959 | 53 | 53 | 54 | LF | 8189750101ac85e74fc2d0388322833ad6dd6a925d7a9fb1379a5e52afeb1daf |
| phase4_materials.py | 3042 | 59 | 59 | 60 | LF | c67abe183e962bcff6cef96a37dd6c2658d9af6609cff15100c6394701e20af5 |
| second_pass_evidence.py | 1713 | 65 | 65 | 66 | LF | b19f2e9fab0100a043e8c99eaa2746bbb97ed014f02628ff5ce94442d26fbd78 |
| second_pass_final_check.py | 3075 | 65 | 65 | 66 | LF | de9b72f99f2bfcdb61fb34d44a9c2976c21455af13de8fa79aa77fd354c56f50 |
| second_pass_finalize.py | 2264 | 61 | 61 | 62 | LF | 650fdc816e8588f0eee1d0a174501009afacc731657a8da686c8342ae30213a9 |
| second_pass_fix_sha.py | 1944 | 44 | 44 | 45 | LF | 0a2195248a996adb308897b1a51945a7431ad042ed265f275cca707d63756352 |
| second_pass_freeze.py | 3132 | 76 | 76 | 77 | LF | 8c44bceb614678d42e0212307edeeb5022d58d34ec6f73d6e895faa668675ca9 |
| second_pass_freeze2.py | 2630 | 62 | 62 | 63 | LF | 3ad84d1c7ee5281bcc2d48ea347a97059caf4cef0515d7f249021a122709c4c9 |
| second_pass_pre.py | 2609 | 75 | 75 | 76 | LF | 4ab06b55308945203d9dd8835928c18db30d7e900b24b2a490fad4433ac364a4 |
| second_pass_verify.py | 2946 | 78 | 78 | 79 | LF | 0a88278cbf753eec057e67fbd09fc98b6c3c1e50b7f5ca8893ff38505d33811b |
| v12_baseline.py | 2272 | 61 | 61 | 62 | LF | 8cf9fb1acb3a22cc19e2e164709fe19aa12daa44141bda261d987c4ae7b0e879 |
| v12_facts.py | 5633 | 140 | 140 | 141 | LF | 08444ab5d65c0272e57b2a00312757e952b83f4073cc789acef8d2e8e905ee18 |
| v12_finalize.py | 3287 | 94 | 94 | 95 | LF | 5034b9e5af9d6a96b1997c4acdba1b1bce4adb3c030ee9a50cec79f85dabcee9 |
| v12_fix_closure.py | 7657 | 126 | 126 | 127 | LF | a64e1a205671e297007e7d2a4bb934b27a6496c95fcebe5d76b2ec71654a0e5f |
| v12_fix_closure_s1.py | 4280 | 80 | 80 | 81 | LF | 7e6155aaa01304513c67032a5f88ac958ac224225d9b3a6d78d7c2aea066e6da |
| v12_fix_closure_s7.py | 5148 | 100 | 100 | 101 | LF | 2fdfc8a95746b9fc42c5646ac870ba2074deefbca22ee5a4ed684f6ab830a19d |
| v12_fix_counts.py | 2471 | 43 | 43 | 44 | LF | 5cfab056abd2e630971c74ef8128a172798ab63280da16338d25e6b5d03753e3 |
| v12_last_check.py | 2534 | 58 | 58 | 59 | LF | 2fb9c224e612fce0848131cc0f1e07a99cdcff7a3517adbf1ad0f828a3b79472 |
| v12_recount.py | 865 | 23 | 23 | 24 | LF | 05fcc3975a85a8f064078cc592fc8ad37a145c74597d043296be8084d584790b |
| v12_verify.py | 6394 | 161 | 161 | 162 | LF | 9ff2146fea76c992a3e0e3a895382e2f5fa3ac16efd534dce7e9bc7c43e80789 |
| verify_timeline.py | 2017 | 51 | 51 | 52 | LF | e012f523b08d00291ad8a78a614cd1be6346e71f4896d836093e0829d2ebb80a |
| verify_v110.py | 4368 | 85 | 85 | 86 | LF | f1761c8efb8809e167fedeacd07c56c4982adfa70ec5a580a07e8ea3275dfd08 |
| verify_v110_b.py | 1261 | 26 | 26 | 27 | LF | b16a18f15fb35c868ba81f89f56c65d548cfb36ac5824d642b82b3ff1017d153 |
| verify_v110_c.py | 1383 | 34 | 34 | 35 | LF | 43ba2d5264bd4bfbc56d663c3bb8d7703a9f2e870a1f9d135d4be8ea6cf047e9 |
| verify_v111_post.py | 3398 | 74 | 74 | 75 | LF | 693c71550550071260af3ed21e840d585b606f5215a48d744daab3cb1fd3c1f5 |
| verify_v111_post2.py | 1431 | 29 | 29 | 30 | LF | 7fc454e0a872d671202e6c458d32053f8e395ce9b64ef2591cdce66af5025c05 |
| verify_v111_pre.py | 2285 | 51 | 51 | 52 | LF | bd82709a13c08b8e49c4007178d5b7fbdcde3c6c3bd09d629c8267eefaf9eda6 |

### T4 A-review `_backups\`（59 份）

| 文件 | 字节 | 行数A | 换行符 | SHA-256(完整) |
|---|---|---|---|---|
| A-00-MATERIAL-AUDIT.v1.0.bak-20261103-1548.md | 13431 | 178 | LF | 77d98ee93a5546f0caf9109808d5833420e7a2b1a4ac17d8c855eb08a8381fb7 |
| A-01-EVENT-L3-REVIEW.md.v1.0.bak-20261103-1615.md | 36440 | 319 | LF | 1cdff2dc74bad0d31f74e4d9679c5774bef8f67115030e2f3e1bc52af24ca0c1 |
| A-01-EVENT-L3-REVIEW.md.v1.1.bak-20261103-1650.md | 43823 | 341 | LF | c4d8803fa5bddbc903fd7f929f63bc701e3c724d6a7ee16b1b4bd8d8d2a8d1fe |
| A-01-ISSUE-REGISTER.md.v1.0.bak-20261103-1615.md | 13225 | 151 | LF | 07c35436ab7b1cc672ca964bd9ebfffeca441139794924049fac81ea453e01bf |
| A-01-ISSUE-REGISTER.md.v1.1.bak-20261103-1650.md | 20047 | 185 | LF | c99bf38f341bab3a5650ff69df9858071e3f80b336c7a71862e28275f61fcd93 |
| A-02-EVIDENCE-CHAIN-REVIEW.v1.0.bak-20261103-1745.md | 29244 | 229 | LF | f1a5da17f7252a8e0ba78af8fb34ddfc6375f31e51e551c08048311aa4111f4c |
| A-03-TWO-STAGE-K1-REVIEW.v1.0.bak-20261103-1840.md | 35656 | 289 | LF | 4538056c3cd03d3408c125533f2aa7d71933cf1d38f8f87c78152f9e21703040 |
| A-04-A1-A4-CLASSIFICATION-REVIEW.v1.0.bak-20261103-2035.md | 7589 | 75 | LF | a4c1f6d027327c2c934b04a34921dfbd8096912412b8822c71787f5875423228 |
| A-04-A5-A8-CLASSIFICATION-REVIEW.v1.0.bak-20261103-2035.md | 14978 | 99 | LF | b906f4de2285bb10d04dbec6d07356ff0c1ecca964f2aa9e0c2ec66f0eb0f0bd |
| A-04-A8-FOCUSED-REVIEW.v1.0.bak-20261103-2035.md | 9394 | 111 | LF | e897c45a9f79e3bd5cb824953bb72dde14ec9b4f2ec3bb21ef5f93cf28983dee |
| A-04-ACCEPTANCE-REPORT.v1.0.bak-20261103-2035.md | 6152 | 50 | LF | 2dee439f1b3b1259cfe1669e4a88622f6b41ae6952c636801d149cfc5b43de04 |
| A-04-B1-B2-SCOPE-REVIEW.v1.0.bak-20261103-2035.md | 6781 | 67 | LF | 4c55e5ee35ecee75289552275623e5e7ac110092dd37dc4d8e020d52b12463db |
| A-04-B1-B4-CLASSIFICATION-REVIEW.v1.0.bak-20261103-2035.md | 11551 | 86 | LF | 9c7a6df79fcef681ce5f12984487b359ef54a12ef9bb5915f43534b43d502774 |
| A-04-BOUNDARY-CASES.v1.0.bak-20261103-2035.md | 7257 | 80 | LF | 3011c277ec6c300df584c4c884e00789e56c50feba46c794d130e667ee2c32a3 |
| A-04-CLASSIFICATION-REVIEW.v1.0.bak-20261103-2035.md | 16211 | 190 | LF | 6639ecaf0688277bb2d92e1b95e058d6b863dc53ccd5cb93aa5ac7bc481a529f |
| A-04-CLOSURE-STATUS.v1.0-presecond.bak-20261103-2115.md | 8995 | 169 | LF | 3c4d9e09901753f0498988c13be1d721296f22d8c0f7f17bdcfc1f0a3fe2954a |
| A-04-CLOSURE-STATUS.v1.0.bak-20261103-2105.md | 8995 | 169 | LF | 3c4d9e09901753f0498988c13be1d721296f22d8c0f7f17bdcfc1f0a3fe2954a |
| A-04-CLOSURE-STATUS.v1.1.bak-20261103-2135.md | 18961 | 270 | LF | c640e96bb60a8388c3deb1219f3c9dde46928ad25d4d02437a822bb8a2f49bfb |
| A-04-CLOSURE-STATUS.v1.1.bak-20261103-2200.md | 18961 | 270 | LF | c640e96bb60a8388c3deb1219f3c9dde46928ad25d4d02437a822bb8a2f49bfb |
| A-04-CLOSURE-STATUS.v1.2.bak-20261103-2240.md | 18764 | 299 | LF | b6b9ffee026d1669af9142d13c3bfcc49385fec542711d42d63b08263ad2290a |
| A-04-CLOSURE-V1.2-CHANGELOG.v1.2.bak-20261103-2240.md | 8105 | 91 | LF | 98814ccdfb4bcbcdf256cfce5d99738b41e0b7c82709ff743df2cb36a0a629e3 |
| A-04-CLOSURE-V1.2-VERIFICATION.v1.2.bak-20261103-2240.md | 7995 | 140 | LF | 4bde91b4855c855ee0508c886425b74bf3008504d733e22ae9007b724c658f9c |
| A-04-CONSISTENCY-REVIEW.v1.0-presecond.bak-20261103-2115.md | 24727 | 343 | LF | dc8739f9c17244d6c6c3b5c97bcee77ec6800d4e7b3baba04147846610b3cfa3 |
| A-04-CONSISTENCY-REVIEW.v1.0.bak-20261103-2055.md | 24727 | 343 | LF | dc8739f9c17244d6c6c3b5c97bcee77ec6800d4e7b3baba04147846610b3cfa3 |
| A-04-CONSISTENCY-REVIEW.v1.1.bak-20261103-2135.md | 26642 | 353 | LF | 92deb694de8904612beaf5ee8d9194aead88e87bf1179283b7440df78a4f3558 |
| A-04-CONSISTENCY-REVIEW.v1.1.bak-20261103-2200.md | 26642 | 353 | LF | 92deb694de8904612beaf5ee8d9194aead88e87bf1179283b7440df78a4f3558 |
| A-04-CONSISTENCY-REVIEW.v1.2.bak-20261103-2240.md | 40072 | 495 | LF | 95b82c4246d616e3cfda96535fe296b40a5f344e11e12be6fa6fd2c41afa0170 |
| A-04-CROSS-MATRIX.v1.0.bak-20261103-2035.md | 7405 | 111 | LF | 8c97ce15c437dd23ea16a13f972551a71ee66c92d795e26bb62ebd830aa2a9c2 |
| A-04-HANDOFF.v1.0.bak-20261103-2035.md | 5182 | 59 | LF | 2e5709178826111675f0b20d2ecaa45ef5052664b40ae8326d6b56d70e90acd8 |
| A-04-INPUT-MATERIAL-INDEX.v1.0.bak-20261103-2035.md | 4930 | 64 | LF | 0fd3ac8567c002487eabb76fc946ed85446ab6052379e90ce3783828cb56c377 |
| A-04-ISSUE-DELTA.v1.0.bak-20261103-2035.md | 8449 | 98 | LF | e0686431aeb2818b358a369e775923c3c294fad33c3bdbbeae1eb5086c4a174d |
| A-04-NOT-EXECUTABLE-REVIEW.v1.0.bak-20261103-2035.md | 6842 | 81 | LF | 05ec342a684f80f28032ac511371c989456a3e13dfd05f66aa7f0029033d91ff |
| A-04-PREVIOUS-CONCLUSIONS-INHERITANCE.v1.0.bak-20261103-2035.md | 5363 | 42 | LF | 805f4c6f8a2e4ea7459b277b2badb71509d6918cb3cb1a94a20f7270cbeb624b |
| A-04-STATE-CONSISTENCY-REVIEW.v1.0.bak-20261103-2035.md | 6129 | 71 | LF | 517ca1731ba9f8382470d38832da3e6cf5bd6e0ad06a7a360d31dcc36a10be30 |
| A-04-STATIC-EXECUTABILITY-REVIEW.v1.0.bak-20261103-2035.md | 5519 | 75 | LF | b5a587d3dcaeee5fd40af1e3bbe39a0e07f4ab8464394724e0cf4b89870b8057 |
| A-04-V1-FUTURE-CLASSIFICATION.v1.0.bak-20261103-2035.md | 7374 | 79 | LF | 4866eaf9aad772a5f2b1e8e410114c15acbc3170862433aa2904af2b789f8453 |
| A-REVIEW-PROGRESS-CHANGELOG.v1.0.bak-20261103-1905.md | 6608 | 69 | LF | a7928d2f363317dda6ad0631d6402f87b14f852b0523c2c5e59525f91dc58e1e |
| A-REVIEW-PROGRESS-v1.10-VERIFICATION.v1.0.bak-20261103-1925.md | 7904 | 110 | LF | 8ab3da4f4ee4bb608a3e59f3a496c04a299edfcb387f215359ee3e8dff7fc960 |
| A-REVIEW-PROGRESS-v1.11-GOVERNANCE-CLOSURE.v1.0.bak-20261103-1942.md | 4426 | 63 | LF | 4b7e224de5450b291403cb42907f53e5b7c60ca8e7573a5197fce039d0c027cc |
| A-REVIEW-PROGRESS.md.v1.10.bak-20261103-1905.md | 30057 | 260 | LF | 7d190e1710b270a7052974c1e3a8f94826894f3373345eae5a921ada54299b0d |
| A-REVIEW-PROGRESS.md.v1.10.bak-20261103-1906.md | 30306 | 262 | LF | e6d6a303f6fb483f1ee3c6b8e12b14304febd3fe734e7bdbe490b48edbcd033b |
| A-REVIEW-PROGRESS.md.v1.11.bak-20261103-1940.md | 32138 | 276 | LF | ecfdc202bf3f22a4871f49ae9b97cd372995d90394c44e75c12eca0b3f00f88f |
| A-REVIEW-PROGRESS.md.v1.11.bak-20261103-1942.md | 32138 | 276 | LF | ecfdc202bf3f22a4871f49ae9b97cd372995d90394c44e75c12eca0b3f00f88f |
| A-REVIEW-PROGRESS.md.v1.12.bak-20261103-2035.md | 34301 | 288 | LF | a10f6442b24f05ecea37c674ca0454ef740a9543943d6f8e285fbbf17281c1af |
| A-REVIEW-PROGRESS.md.v1.2.bak-20261103-1615.md | 6891 | 91 | LF | 95bc5afd9149fd1070ae6112cb0d77d925b77bb3ae56dd99604a3bd7b39f55e2 |
| A-REVIEW-PROGRESS.md.v1.4.bak-20261103-1650.md | 9644 | 95 | LF | ec7c1530627cc7990cd7a1f406a378d43828519ac3dbdbba4cf2bc1bcfc1b9dd |
| A-REVIEW-PROGRESS.md.v1.5.bak-20261103-1652.md | 10035 | 98 | LF | fa25acf9e87e7338a8976faf67656b689fe17bf48b14e4ce5da878f8260689a9 |
| A-REVIEW-PROGRESS.md.v1.5.bak-20261103-1655.md | 10035 | 98 | LF | fa25acf9e87e7338a8976faf67656b689fe17bf48b14e4ce5da878f8260689a9 |
| A-REVIEW-PROGRESS.md.v1.6.bak-20261103-1658.md | 17315 | 160 | LF | f47d2bc5d63aaf2378755d167e3da34b48f8132cb1a8fbf1bd1325041f21a3d7 |
| A-REVIEW-PROGRESS.md.v1.6.bak-20261103-1730.md | 17315 | 160 | LF | f47d2bc5d63aaf2378755d167e3da34b48f8132cb1a8fbf1bd1325041f21a3d7 |
| A-REVIEW-PROGRESS.md.v1.7.bak-20261103-1750.md | 19369 | 172 | LF | 6e4b472fe7ecfd52e9e5a6d9d16a4de263970c725c2c2f56fb354749b68dd597 |
| A-REVIEW-PROGRESS.md.v1.7.bak-20261103-1815.md | 19369 | 172 | LF | 6e4b472fe7ecfd52e9e5a6d9d16a4de263970c725c2c2f56fb354749b68dd597 |
| A-REVIEW-PROGRESS.md.v1.8.bak-20261103-1830.md | 22155 | 189 | LF | 886a4c3f2bc111fa0230b974a61ec1840ff4fefd0e048fa3ea6b06af7235f9bf |
| A-REVIEW-PROGRESS.md.v1.8.bak-20261103-1850.md | 22155 | 189 | LF | 886a4c3f2bc111fa0230b974a61ec1840ff4fefd0e048fa3ea6b06af7235f9bf |
| A-REVIEW-PROGRESS.md.v1.9.bak-20261103-1835.md | 24710 | 202 | LF | 50be36cd2cda5390621d864e173d3d0b91f61da0036cd2f7483af6f8f0e3bb2b |
| A-REVIEW-PROGRESS.md.v1.9.bak-20261103-1900.md | 24710 | 202 | LF | 50be36cd2cda5390621d864e173d3d0b91f61da0036cd2f7483af6f8f0e3bb2b |
| A-REVIEW-PROGRESS.v1.0.bak-20261103-1548.md | 3912 | 60 | LF | 0a1b2b3fc8b276a09bb071d75400aa97f1dc7a36e3dcc703cb81fc36941f9ec3 |
| A-REVIEW-PROGRESS.v1.1.bak-20261103-1549.md | 4942 | 75 | LF | 81228a5c074a074bf709bcda6553377e5cc1a88d9b99d2fa644bb96fef0276e8 |
| A-REVIEW-PROGRESS.v1.3.bak-20261103-1640.md | 7361 | 93 | LF | 2e89e8d6cbdc807bd6f122c776296eef345d231949cdaedb9bced99414d719ee |

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

---

---

## 3. 行数口径分析

### 3.1 本报告使用的口径

本报告每个文件的行数实测了三种口径并全部列出（T1/T2/T3）：

| 口径 | 定义 | 文件以 LF 结尾时 | 文件不以换行结尾时 |
|---|---|---|---|
| **A** | `splitlines()`（≈.NET `ReadAllLines().Count`） | = LF 数 | = LF 数 + 1 |
| **B** | LF 字符数 | = LF 数 | = LF 数 |
| **C** | `split("\n")` 长度 | = LF 数 + 1（含末尾空串） | = LF 数 + 1 |

### 3.2 报告实际使用的口径（逐套判定，依据=声明值与实测口径的唯一匹配）

| 声明来源 | 声明示例 | 与实测匹配的口径 | 判定 |
|---|---|---|---|
| B-06 主报告 §1.1/§1.2（L95–L115）、§18 L776 | B-00「300 行」（实测 A=299, C=300）；B-02「1000」（A=1000, C=1000，该文件无末尾换行）；B-06 五份 §2.1「802/706/250/201/196」（A=801/705/249/200/195） | **口径 C**（`split("\n")`） | CONFIRMED |
| B-06-PHASE1 v3 §2.2（L58–L64） | 同上（300/410/1000/461/461/401/**612**） | **口径 C** | CONFIRMED |
| B-06 v3/VERIFICATION 对**自身**行数 | v2「759 行」（A=759, C=760）、v3「984 行」「余量 16」（A=984, C=985） | **口径 A**（ReadAllLines） | CONFIRMED |
| 设计/治理文件行数（B-06 §1.1、v3 §2.3） | m02「868」（A=867, C=868）；m03「464」（C=464）；ledger「392」（C=392）；m01「1198」（A=1198，无末尾换行） | **口径 C**（对无末尾换行的文件与 A 重合） | CONFIRMED |
| A-04-CONSISTENCY-REVIEW §2.2 全部 17 行 | A-04-A1-A4「75 行」（A=75, C=76）等 17/17 | **口径 A**（ReadAllLines） | CONFIRMED |
| A-04 §2.1 进度文件「288 行」 | A=288, C=289 | **口径 A** | CONFIRMED |

### 3.3 口径结论（FB-03）

- **B 侧内部混用两种口径**：输入文件表用口径 C，自身行数（759/984）用口径 A。对以 LF 结尾的文件两口径恒差 1（如 B-06 主报告：801（A）vs 802（C）；v3：984（A）vs 985（C））。后果：**跨表直接比较行数会产生 ±1 偏差**，但不影响任何 SHA/字节结论。→ **FB-03**。
- **A 侧口径统一**（全部口径 A），B、A 两套之间不可直接比较行数。
- 「报告声明的行数与实测行数是否一致」的最终判定：**在各自口径下，本报告核对过的全部行数声明（B 侧 26 项 + A 侧 18 项 + 设计文件 10 项）均与实测一致，无一偏差**；偏差只存在于口径混用层面（FB-03），不存在「数字写错」级别的行数错误。

---

## 4. SHA-256 对照表（报告声称值 vs 实测值）

### 4.1 B-06 v3 §2.2 历史报告 7 份（v3 L58–L64）

| 文件 | 报告声称(前16) | 实测(前16) | 完整实测 SHA-256 | 结果 |
|---|---|---|---|---|
| B-00-MATERIAL-AND-BASELINE-AUDIT.md | `bdf76458a26730e7` | `bdf76458a26730e7` | bdf76458a26730e7524a3f52c6dc0d8164540adba9ffa815eda9811f9c4759ad | ✅ |
| B-01-M01-STORAGE-REVIEW.md | `640b7ecd316b0914` | `640b7ecd316b0914` | 640b7ecd…a7fad89150 | ✅ |
| B-02-M02-RETRIEVAL-REVIEW.md | `a66058c5847d15d1` | `a66058c5847d15d1` | a66058c5…aca121955 | ✅ |
| B-03-M03-EXTRACTION-REVIEW.md | `8a3b347cd14ed49f` | `8a3b347cd14ed49f` | 8a3b347c…ea4cc02f | ✅ |
| B-04-M04-CONSOLIDATION-REVIEW.md | `8eb5a0694171c63c` | `8eb5a0694171c63c` | 8eb5a069…fe1fb23 | ✅ |
| B-05-M05-SECURITY-REVIEW.md | `a10198bbb32032de` | `a10198bbb32032de` | a10198bb…21c5e14 | ✅ |
| B-REVIEW-PROGRESS.md | `6fcad56e17994d7c` | `6fcad56e17994d7c` | 6fcad56e…42780308 | ✅ |

（完整值见 T1；此处缩写为排版省略，CSV 为准。）**7/7 全部吻合。**

### 4.2 B-06 v3 §2.1 本阶段五份交付物（v3 L48–L52）

| 文件 | 声称(前16) | 实测(前16) | 结果 |
|---|---|---|---|
| B-06-CROSS-MODULE-ARCHITECTURE-REVIEW.md | `1a77f4f12db489cf` | `1a77f4f12db489cf` | ✅ |
| B-06-EVIDENCE-MATRIX.md | `e16b8e9138008ec2` | `e16b8e9138008ec2` | ✅ |
| B-06-INTERFACE-TRACE.md | `cad41d238b74ccde` | `cad41d238b74ccde` | ✅ |
| B-06-ISSUE-HANDOFF.md | `2f6e705a8c734843` | `2f6e705a8c734843` | ✅ |
| B-06-STAGE7-INPUT-MANIFEST.md | `8264cff1e9582f5c` | `8264cff1e9582f5c` | ✅ |

### 4.3 B-06 版本身份声明

| 声明 | 声称 SHA(前16) | 实测对象 | 实测(前16) | 结果 |
|---|---|---|---|---|
| v3 头部 L7/CHANGELOG/VERIFICATION：v2 = 759 行/49,205 B | `02329c264cc31f62` | B-06-PHASE1-READONLY-AUDIT.md | `02329c264cc31f62` | ✅ |
| v3 = 984 行/71,082 B | `cdeb4dd807608ff1` | B-06-PHASE1-READONLY-AUDIT-v3.md | `cdeb4dd807608ff1` | ✅ |
| v3 头部 L13：v1 保留于备份 | `2b2cc00671682e24` | 备份 B-06-PHASE1-READONLY-AUDIT-v1-20261103.md | `2b2cc00671682e24` | ✅ |
| VERIFICATION §四.5：v3 修订前 1020 行 | —（未给 SHA） | 备份 B-06-PHASE1-READONLY-AUDIT-v3-pre-1000line-fix-20261103.md | `4c913583567e754c`，实测 1020 行 | ✅（行数吻合） |

### 4.4 A-04 §2.2 十七份交付物声称 SHA（L84–L100）——17/17 全部吻合

| # | 文件 | 声称(前16) | 实测(前16) | 字节（声称=实测） | 行数（声称/口径A实测） | 结果 |
|---|---|---|---|---|---|---|
| 1 | A-04-A1-A4-CLASSIFICATION-REVIEW.md | `a4c1f6d027327c2c` | `a4c1f6d027327c2c` | 7,589 | 75/75 | ✅ |
| 2 | A-04-A5-A8-CLASSIFICATION-REVIEW.md | `b906f4de2285bb10` | `b906f4de2285bb10` | 14,978 | 99/99 | ✅ |
| 3 | A-04-A8-FOCUSED-REVIEW.md | `e897c45a9f79e3bd` | `e897c45a9f79e3bd` | 9,394 | 111/111 | ✅ |
| 4 | A-04-ACCEPTANCE-REPORT.md | `2dee439f1b3b1259` | `2dee439f1b3b1259` | 6,152 | 50/50 | ✅ |
| 5 | A-04-B1-B2-SCOPE-REVIEW.md | `4c55e5ee35ecee75` | `4c55e5ee35ecee75` | 6,781 | 67/67 | ✅ |
| 6 | A-04-B1-B4-CLASSIFICATION-REVIEW.md | `9c7a6df79fcef681` | `9c7a6df79fcef681` | 11,551 | 86/86 | ✅ |
| 7 | A-04-BOUNDARY-CASES.md | `3011c277ec6c300d` | `3011c277ec6c300d` | 7,257 | 80/80 | ✅ |
| 8 | A-04-CLASSIFICATION-REVIEW.md | `6639ecaf0688277b` | `6639ecaf0688277b` | 16,211 | 190/190 | ✅ |
| 9 | A-04-CROSS-MATRIX.md | `8c97ce15c437dd23` | `8c97ce15c437dd23` | 7,405 | 111/111 | ✅ |
| 10 | A-04-HANDOFF.md | `2e57091788261116` | `2e57091788261116` | 5,182 | 59/59 | ✅ |
| 11 | A-04-INPUT-MATERIAL-INDEX.md | `0fd3ac8567c00248` | `0fd3ac8567c00248` | 4,930 | 64/64 | ✅ |
| 12 | A-04-ISSUE-DELTA.md | `e0686431aeb2818b` | `e0686431aeb2818b` | 8,449 | 98/98 | ✅ |
| 13 | A-04-NOT-EXECUTABLE-REVIEW.md | `05ec342a684f80f2` | `05ec342a684f80f2` | 6,842 | 81/81 | ✅ |
| 14 | A-04-PREVIOUS-CONCLUSIONS-INHERITANCE.md | `805f4c6f8a2e4ea7` | `805f4c6f8a2e4ea7` | 5,363 | 42/42 | ✅ |
| 15 | A-04-STATE-CONSISTENCY-REVIEW.md | `517ca1731ba9f838` | `517ca1731ba9f838` | 6,129 | 71/71 | ✅ |
| 16 | A-04-STATIC-EXECUTABILITY-REVIEW.md | `b5a587d3dcaeee5f` | `b5a587d3dcaeee5f` | 5,519 | 75/75 | ✅ |
| 17 | A-04-V1-FUTURE-CLASSIFICATION.md | `4866eaf9aad772a5` | `4866eaf9aad772a5` | 7,374 | 79/79 | ✅ |

### 4.5 A-04 §2.4 前序报告 5 份（L122–L127）

| 文件 | 声称(前16) | 实测(前16) | 结果 |
|---|---|---|---|
| A-00-MATERIAL-AUDIT.md | `77d98ee93a5546f0` | `77d98ee93a5546f0` | ✅ |
| A-01-EVENT-L3-REVIEW.md | `c4d8803fa5bddbc9` | `c4d8803fa5bddbc9` | ✅ |
| A-01-ISSUE-REGISTER.md | `c99bf38f341bab3a` | `c99bf38f341bab3a` | ✅ |
| A-02-EVIDENCE-CHAIN-REVIEW.md | `f1a5da17f7252a8e` | `f1a5da17f7252a8e` | ✅ |
| A-03-TWO-STAGE-K1-REVIEW.md | `4538056c3cd03d34` | `4538056c3cd03d34` | ✅ |
| A-REVIEW-PROGRESS.md（v1.12, 34,301 B, 288 行） | `a10f6442b24f05ec` | `a10f6442b24f05ec`（34,301 B, 288 行口径A） | ✅ |

### 4.6 B-06 主报告 §1.1/§1.2 阶段基线（L95–L115）

- 9 个设计文件（L95–L103）+ 增补层（L119）：**10/10 全部吻合**（见 T5）。
- B-00～B-05（L109–L114）：**6/6 全部吻合**。
- **B-REVIEW-PROGRESS（L115）：不吻合当前文件** → 专项见 §7（FB-01）。

---

## 5. B-06 历史代际差异表（v1/v2/v3/v4 备份逐代比对）

数据源：T2（备份目录实测字节+完整 SHA-256）。

### 5.1 五份交付物 × 四代比对

| 交付物 | v1（字节/SHA前16） | v2 | v3 | v4 | v4=当前文件？ | 代际内容变化判定 |
|---|---|---|---|---|---|---|
| CROSS-MODULE-ARCHITECTURE-REVIEW | 44,188 / `5f4f88c9` | 50,270 / `869723b7` | 53,972 / `65a755e9` | 60,124 / `1a77f4f1` | ✅（60,124/`1a77f4f1`） | **四代均不同**——每次升级都有内容变化 |
| EVIDENCE-MATRIX | 27,528 / `e16b8e91` | 27,528 / `e16b8e91` | 27,528 / `e16b8e91` | 27,528 / `e16b8e91` | ✅ | **四代完全相同**——「版本升级」未发生任何内容变化（FB-02） |
| INTERFACE-TRACE | 14,348 / `cad41d23` | 14,348 / `cad41d23` | 14,348 / `cad41d23` | 14,348 / `cad41d23` | ✅ | **四代完全相同**（FB-02） |
| ISSUE-HANDOFF | 12,931 / `f1df0563` | 13,911 / `9c8e6f66` | 14,339 / `ea5ed53d` | 15,870 / `2f6e705a` | ✅ | **四代均不同** |
| STAGE7-INPUT-MANIFEST | 8,804 / `2c7ad8f1` | 8,861 / `50126e3c` | 8,862 / `8264cff1` | 8,862 / `8264cff1` | ✅ | **v3=v4 完全相同**——最后一次"升级"无内容变化（FB-02） |

### 5.2 其他备份组内的相同内容对（完整 SHA 相等，实测判定）

| 相同内容组 | 实测字节/SHA(前16) | 说明 |
|---|---|---|
| B-02-v4-DELIVERED = B-02-v5-pre-eol-fix | 121,273 / `1bfcd8d3`（**CRLF**） | 两份"不同版本名"实为同一字节流 |
| B-02-v7-condensed = B-02-v8-pre-final-revision | 81,249 / `9f239810` | 同上 |
| B-04-v1 = B-04-v2 | 44,899 / `8eb5a069` | 且与当前 B-04 相同 |
| B-PROGRESS-v2 = v3-pre-stage2 = v4-pre-stage2-final | 17,004 / `f56ad287` | 三个版本名、一份内容 |
| B-00-v1 = 当前 B-00；B-01-LITE-v1 = 当前 B-01；B-03-v1 = 当前 B-03；B-05-v1 = 当前 B-05 | 见 T2 | 交付后未再改动（与各报告"冻结"声明一致） |
| B-02-v9-final = 当前 B-02 | 99,247 / `a66058c5` | 当前 B-02 即 v9 |

### 5.3 换行符历史验证（正面结论）

B-REVIEW-PROGRESS §0.1（L26）自述「曾用 Python 重排 B-02 时默认写入 CRLF，已用 `_fix_eol.py` 修复为 LF」。实测：备份 **B-02-v4/v5 为 CRLF（1307×CRLF）**，v6 起至当前 v9 均为 LF——**自述与备份实证完全吻合**（且 `_fix_eol.py` 本体未留存，见 FB-06）。

---

## 6. 受保护文件清单核验（18 个）

### 6.1 清单是否被明确列出

**是。** `B-06-PHASE1-V3-VERIFICATION.md` §三（L78–L99）以表格**逐一列出 18 个文件名 + 预期 SHA(前16) + 实测 SHA(前16)**；CHANGELOG L89 引用同一结论。18 个为：v2 报告、B-06 五份交付物、B-04、B-05、进度文件、m01～m05、台账（adjudication-ledger）、冲突册（05-conflict-register）、接口地图（06-interface-map）、寄存器（R3-Architecture-Decision-Register）。

> 注意口径：18 个**不含** B-00/B-01/B-02/B-03，也**不含**增补层 adjudication-supplement（后者在 v3 §2.3 材料表 L79 中出现但不在受保护清单内）。「18」与清单逐项相加（1+5+2+1+5+4）一致，无计数错误。

### 6.2 逐一复算结果（18/18）

| # | 文件 | VERIFICATION 预期(前16) | 本次实测副本 | 实测(前16) | 结果 |
|---|---|---|---|---|---|
| 1 | v2 报告 | `02329c264cc31f62` | Downloads\B-06-PHASE1-READONLY-AUDIT.md | `02329c264cc31f62` | ✅ |
| 2 | B-06 主报告 | `1a77f4f12db489cf` | Downloads（=v4 备份） | `1a77f4f12db489cf` | ✅ |
| 3 | B-06 证据矩阵 | `e16b8e9138008ec2` | Downloads（四代同值） | `e16b8e9138008ec2` | ✅ |
| 4 | B-06 接口追踪 | `cad41d238b74ccde` | Downloads（四代同值） | `cad41d238b74ccde` | ✅ |
| 5 | B-06 问题交接 | `2f6e705a8c734843` | Downloads（=v4 备份） | `2f6e705a8c734843` | ✅ |
| 6 | B-06 阶段7清单 | `8264cff1e9582f5c` | Downloads（=v3/v4 备份） | `8264cff1e9582f5c` | ✅ |
| 7 | B-05 | `a10198bbb32032de` | Downloads | `a10198bbb32032de` | ✅ |
| 8 | B-04 | `8eb5a0694171c63c` | Downloads | `8eb5a0694171c63c` | ✅ |
| 9 | 进度文件 | `6fcad56e17994d7c` | Downloads\B-REVIEW-PROGRESS.md | `6fcad56e17994d7c` | ✅ |
| 10 | m01 | `777a27e68635044a` | review-staging\...\01_BASELINE\Memory-Core\m01.md | `777a27e68635044a` | ✅ |
| 11 | m02 | `d8e80b39d407fa48` | 同上 m02.md | `d8e80b39d407fa48` | ✅ |
| 12 | m03 | `fcd9f53d07c8ffe8` | 同上 m03.md | `fcd9f53d07c8ffe8` | ✅ |
| 13 | m04 | `da02ffe8782d8424` | 同上 m04.md | `da02ffe8782d8424` | ✅ |
| 14 | m05 | `fe5af2623681670e` | 同上 m05.md | `fe5af2623681670e` | ✅ |
| 15 | 台账 | `d1de733eecc74062` | Targeted-Round2\01_EVENT_L3_REVIEW\adjudication-ledger.md（22,758 B） | `d1de733eecc74062` | ✅（见 FB-07 双副本警示） |
| 16 | 冲突册 | `87c3d6eb935dfc60` | 01_BASELINE\R4-Integration\05-conflict-register.md | `87c3d6eb935dfc60` | ✅ |
| 17 | 接口地图 | `fda34e7cb51c8477` | 01_BASELINE\R4-Integration\06-interface-map.md | `fda34e7cb51c8477` | ✅ |
| 18 | 寄存器 | `1aa363b75c6a7e1c` | 01_BASELINE\Architecture\R3-Architecture-Decision-Register.md | `1aa363b75c6a7e1c` | ✅ |

**结论：CONFIRMED——18 个受保护文件被明确列出、全部可复算、18/18 SHA 一致。**（VERIFICATION 自述其为「主代理自验，非第三方独立审计」（L9）；本报告为独立复算，结果支持其声明。）

---

## 7. 进度文件基线矛盾专项（任务第 6 项）

### 7.1 双方声称与实测

| 来源 | 位置 | 声称 | 实测判定 |
|---|---|---|---|
| B-06-PHASE1 v3 §2.2 | v3 L64（另 L704） | 612 行 / 49,993 B / SHA `6fcad56e17994d7c` | **与当前文件完全一致**：实测 49,993 B、`6fcad56e17994d7c…`、口径 A=611 行 / 口径 C=612 行 |
| B-06 主报告 §1.2 | B-06-CROSS-MODULE-ARCHITECTURE-REVIEW.md **L115**（另 §18 L776「进度 513」、L9） | 43,057 B / 513 行 / SHA `fb2923afaa173738` | **与当前文件不一致**；当前进度文件为 49,993 B |

### 7.2 `fb2923afaa173738` 的定位（备份目录穷举匹配）

在 `_B-REVIEW-BACKUPS\` 全部 50 份中做 SHA 匹配，唯一命中：

> **`B-PROGRESS-v8-pre-stage6-20261103.md`：43,057 字节 / 512 行（口径A）/ 513 行（口径C）/ SHA-256 = `fb2923afaa1737380f54040d803ee7ae6664bf7d99f1ccab84352d79490f07e3`**

即 B-06 §1.2 记录的是**进度文件第 8 版（stage6 前）的快照**。

### 7.3 时间线（文件系统 LastWriteTime 实测）

| 时刻 | 事件 |
|---|---|
| 18:40:51 | B-PROGRESS-v8 保存（43,057 B / `fb2923af`） |
| 19:07:13 | B-06 v1 主报告写入，§1.2 记录 43,057/513/`fb2923af`（**当时正确**） |
| 19:09:34 | B-PROGRESS-v9 保存（48,156 B / `ca2835c1`）——此后 v8 快照即过时 |
| 19:16:23 / 19:16:48 / 19:17:41 / 19:17:55 | B-06 v2、PROGRESS-v10、B-06 v3、PROGRESS-v11（v10/v11 = 48,633/48,931 B） |
| 19:19:27 | B-06 v4 写入，**§1.2 仍为 43,057/513/`fb2923af`，未刷新**（L95→L109→L113→L115 四代完全相同） |
| 19:19:42 | 当前进度文件保存（49,993 B / `6fcad56e`） |

### 7.4 判断

**「B-06 引用过时」，v3 未引用错误基线。** 依据：
1. v3 §2.2 的三个数值与当前文件实测**逐项一致**（含 SHA 完整 64 位匹配）；
2. B-06 §1.2 的三个数值精确等于备份 `B-PROGRESS-v8-pre-stage6-20261103.md`，且 v1→v4 四代从未更新该行——v1 时点该值尚属正确，进度文件在 B-06 修订期间继续演进（v9→v11→当前）后，B-06 未回填；
3. 同一过时值还存在于 B-06 §18 L776（「进度 513」）与 B-06-STAGE7-INPUT-MANIFEST.md L23（513/`fb2923af`，v1~v4 同）；
4. 同类问题：B-REVIEW-PROGRESS.md L97 仍写 B-06 主报告「637 行」（v1 时代数值，当前为 801/802），亦未回填。
→ **FB-01**（含 FB-10 关联项）。

---

## 8. A-04 文件数量核验

### 8.1 「阶段 4 原始交付 17 份」逐一实测（A-04-CONSISTENCY-REVIEW §2.2 L84–L100 清单）

17 个文件名逐一在 `E:\[PROJECT]\A-review\` 实测：**17/17 存在**，字节/行数/SHA 全部吻合（见 §4.4 表）。

### 8.2 目录实测计数（2026-11-03 20:29）

| 类别 | 实测数 | 说明 |
|---|---:|---|
| `A-04-*.md`（根目录） | **21** | = 17 原始 + CONSISTENCY-REVIEW + CLOSURE-STATUS + CLOSURE-V1.2-CHANGELOG + CLOSURE-V1.2-VERIFICATION |
| `*.py`（根目录） | **41** | 阶段1~3 链 + 阶段4 链 + 二次校核链 + v1.2 链 |
| `*.txt`（根目录） | **7** | `_consistency_evidence*.txt`、`_secondpass_*.txt`、`_v12_*.txt` |
| 其他扩展名 | 0 | — |
| 根目录合计 | 78 | — |
| `_backups\` | **59**（其中 `A-04-*` 前缀 29） | — |

### 8.3 与报告声明比对

- §2.3 L116「当前 A-04-* 目录总数 20；VERIFICATION 创建后 21」——与实测 **21** 一致（VERIFICATION 已存在）。✅
- **但**同一报告 §五.4 第 8 条（L285）与 R-3 行（L467）写「当前 `A-04-*` 总数 **19**（目录实测）」，未带「v1.1 时点」限定语，与 §2.3 的 20/21 并存——**同报告三个口径（19/20/21）并存**，实测当前值 21。→ **FB-11**。
- §3（L66）「实际目录共 33 个 `.py`」：当前实测 41 个。多出的 8 个为 v1.2 链脚本（`v12_*` 等，LastWriteTime 晚于该句所在版本的修订过程），该句在其写作时点是否成立**无法回溯验证** → UNVERIFIED（历史时点），当前值 41 为 CONFIRMED。

---

## 9. 缺失与不一致清单（FB 编号）

> 分级：CONFIRMED（实测证实）/ PROBABLE / UNVERIFIED / CONFLICTED / REJECTED。

| 编号 | 级别 | 结论 | 一句话描述 | 证据位置 |
|---|---|---|---|---|
| **FB-01** | P2 | CONFIRMED | B-06 主报告 §1.2（L115）、§18（L776）、STAGE7-MANIFEST（L23）记录的进度文件基线 43,057 B/513 行/`fb2923afaa173738` 已过时，实测当前为 49,993 B/`6fcad56e17994d7c`；该过时值精确对应备份 `B-PROGRESS-v8-pre-stage6-20261103.md`，v1→v4 四代未回填；v3 §2.2 基线正确 | B-06 L115/L776；MANIFEST L23；v3 L64/L704；T1/T2 |
| **FB-02** | P3 | CONFIRMED | B-06「五份交付物 v1→v4」中，EVIDENCE-MATRIX 与 INTERFACE-TRACE 四代 SHA 完全相同、STAGE7-MANIFEST v3=v4 相同——这三个文件的多代「版本升级」未发生任何内容变化；实际演变的只有主报告与 ISSUE-HANDOFF | T2（SHA 逐代比对） |
| **FB-03** | P3 | CONFIRMED | B 侧行数口径混用：输入文件表用 `split("\n")` 口径（对 LF 结尾文件=编辑器行数+1），自身行数（v2=759、v3=984、余量16）用 ReadAllLines 口径；同文件两口径差 1（B-06 主报告 801 vs 802）；A 侧统一用 ReadAllLines 口径 | §3.2/§3.3；T1 |
| **FB-04** | P3 | CONFIRMED | 备份目录存在多组「不同版本名、字节级相同内容」：B-02 v4=v5（且为 CRLF）、v7=v8；B-04 v1=v2；B-PROGRESS v2=v3=v4——版本命名粒度大于实际变更粒度 | §5.2；T2 |
| **FB-05** | P3 | CONFIRMED | B 侧声称实际执行的脚本 `_check_v2.py`（B-REVIEW-PROGRESS L42）、`_fix_eol.py`（L26）在 Downloads、`_B-REVIEW-BACKUPS\`、`E:\[PROJECT]\` 均不存在——B 侧工作脚本零留存（对照：A 侧留存 41 个 .py），B 侧统计与修复过程不可独立复现 | B-REVIEW-PROGRESS L26/L42；refcheck 输出 |
| **FB-06** | P3 | CONFIRMED | `adjudication-ledger.md` 存在两个不同副本并存：22,758 B/`d1de733e`（Targeted-Round2 两处 + A-review-workspace\source）与 22,763 B/`d152471b`（PREP\02_ADJUDICATION + release\R4-Integration-20261102\），无权威版本标注；B 侧基线取 22,758 副本 | §6.2 #15；目录实测 |
| **FB-07** | P3 | CONFIRMED | B-04-CONSISTENCY 类比项：A-04-CONSISTENCY-REVIEW 内部并存 19（L285/L467）、20（L30/L116）、21（L116）三个 `A-04-*` 目录计数，实测当前 21；「19」未标时间限定语，构成同报告口径冲突 | A-04-CONSISTENCY-REVIEW L30/L116/L285/L467；§8.2 |
| **FB-08** | INFO | CONFIRMED | `RRF-PAPER-EXCERPT-20261103.txt` 带 UTF-8 BOM 且换行混用（1×CRLF+23×LF），不符合 B 侧交付物「UTF-8 无 BOM/LF」规范（属归档附件，非正式交付物） | T2 末行 |
| **FB-09** | INFO | CONFIRMED | B-REVIEW-PROGRESS L97 记录 B-06 主报告「637 行」为 v1 时代数值（当前 801/802），与 FB-01 同类：进度文件对阶段 6 交付物的行数记载未随 v2~v4 更新 | B-REVIEW-PROGRESS L97 |
| **FB-10** | INFO | UNVERIFIED | B-06 v3/CHANGELOG/VERIFICATION 以「B-06 补充任务书（一）～（四）」「v3 受控修订任务书」为修订/验证依据，但全盘（Downloads、E:\[PROJECT] 递归）未找到任何任务书文件——推测为对话内指令，**其内容与授权边界无法复核** | v3 L4/L8；CHANGELOG L8；refcheck 输出 |
| **FB-11** | INFO | CONFIRMED | 「缺失文件」检查中，所有被 B/A 报告当作**实际输入**引用的文件均已找到（m01~m05、治理 5 件、纲领 2 件、`limit v2.md`、PSYCH 44 文件在 `F:\[PROJECT]\psy-model\`、RRF 摘录在备份目录）；仅 4 个引用名属「规划/假设性文件」且报告已如实标注为未创建/示例：`B-06-ISSUE-DECISION-MAP.md`（v3 L507：需用户裁决后才建立）、`B-07-FINAL-ALGORITHM-REVIEW.md`（PROGRESS L98：阶段7待授权）、`B-04-ERRATA-v1-YYYYMMDD.md`（PROGRESS L606：「如需…如」示例名）、`B-04-M04-INTEGRATION-REVIEW.md`/`B-05-M05-PERMISSION-REVIEW.md`（v3 §1.1 D-01 已声明的任务书命名偏差） | refcheck 输出；各标注行 |

**REJECTED（未发现支持的问题）**：任务书第 2 项担心的「报告行数与实测不一致」「主报告声称的历史 SHA 不符」在核对范围内**均不成立**——B 侧 33 项 SHA 声称、A 侧 23 项 SHA 声称、全部行数声称（在各自口径下）与实测一致。

---

## 10. 未检查范围与限制

### 10.1 未检查范围

1. **报告内容的技术正确性**：本审查仅做文件级基线（存在性/字节/行数/编码/换行/SHA），不复核 B-06 与 A-04 的技术结论、证据编号（E-xx/B-xxx/S4-xx）指称是否成立。
2. **m01~m05 与治理文件的全文内容**：仅实测元数据（字节/行数/换行/SHA），未读全文；且仅核验了 review-staging 中各一副本（FB-06 的双副本问题提示这些文件在盘上有多副本，未逐一穷举全部副本的 SHA）。
3. **F:\[PROJECT]\psy-model\ 44 个文件**：仅确认存在性与清单，未实测其 SHA/行数（B-02 引用了其中 7 个文件的行号级内容，未回源逐行核对）。
4. **历史时点状态**：无法验证任何文件在「报告写作时点」的行数/SHA（只能验证当前值与备份值）；所有「当时是否正确」的判断基于备份链 + 文件时间戳推断。
5. **任务书内容**（FB-10）：不存在文件，无法核对其与报告的对应关系。
6. **Downloads 根目录非 B-* 的其他文件**（如 ChatGPT 导出等）：不在审查范围，仅因 grep 命中而略过。

### 10.2 本次未能确认的事项

| 事项 | 状态 |
|---|---|
| A-04 §3「写作时点目录共 33 个 .py」是否当时为真 | UNVERIFIED（当前 41；无法回溯时点） |
| 任务书是否曾以文件形式存在过 | UNVERIFIED（现不存在） |
| B-02 备份 v4/v5 的 CRLF 是否仅此一处（其他历史文件是否曾有 EOL 事故被完全覆盖） | PROBABLE（现有 50 份备份中仅 v4/v5 为 CRLF，未见其他异常） |
| `adjudication-ledger.md` 两副本中何者为权威 | UNVERIFIED（无标注；B 侧基线取 22,758 B 版） |
| 18 个受保护文件在 v3 修订**之前**是否与更早基线一致 | UNVERIFIED（VERIFICATION 仅对照 v2 时点预期值；更早基线在 B-06 主报告 §1，两者一致，但无独立第三方时点快照） |

### 10.3 审查产物清单（本目录）

| 文件 | 用途 |
|---|---|
| `AGENT-01-FILE-BASELINE.md` | 本报告 |
| `_agent01_baseline_main.csv` / `_agent01_baseline_backups.csv` / `_agent01_baseline_areview.csv` | 全部 204 个文件的原始实测数据（字节/四口径行数/编码/BOM/换行/完整 SHA-256） |
| `_agent01_measure.py` / `_agent01_refcheck.py` / `_agent01_mktbl.py` | 可复跑的测量与检查脚本 |

---

*AGENT-01 审查完毕。未修改任何被审查文件。*
