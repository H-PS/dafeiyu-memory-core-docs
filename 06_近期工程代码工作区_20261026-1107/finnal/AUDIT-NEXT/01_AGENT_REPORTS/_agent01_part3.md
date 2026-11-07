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
