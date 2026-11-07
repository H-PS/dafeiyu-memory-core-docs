# ROUND2_EVIDENCE_INDEX · 本轮证据索引

| 项 | 内容 |
|---|---|
| 轮次 | ROUND2 |
| 原则 | **只登记实际读取的文件**。未读的不得写入「已核查」 |
| 证据分级 | 原始证据（P）/ 二手转述（S）/ 仅元数据（M）/ 未读取（N） |

---

## 一、本轮实际读取的文件

### 1.1 交付集文件（二手证据，但**本轮复核对象**）

| # | 文件 | SHA-256（前16，本轮实测） | 字节 | 读取范围 | 核实的问题编号 | 证据位置 | 证据类型 |
|---:|---|---|---:|---|---|---|---|
| 1 | `05_REVISION_PLAN.md` | 见下 | 22,457 | **全文 L1–421** | A08-09、A08-05 | §A 类 L39–275；C 类 L289–309；D 类 L313–319 | **S（本轮主要对象）** |
| 2 | `06_CHANGELOG.md` | 见下 | 6,144 | **全文 L1–120** | A08-09、A08-02 | §三 L59–88；L35 | **S** |
| 3 | `04_FINDINGS_REGISTER.md` | 见下 | 48,692 | **全文 L1–739**（ROUND1 已读，本轮定点复用） | A08-05、A08-09、A08-08 | §0 L13–29；F-21 L581–595；D-01~D-05 L704–710 | **S** |
| 4 | `FINAL_REPORT.md` | 见下 | 19,238 | **全文 L1–439**（ROUND1 已读，本轮定点复用） | A08-03、A08-05 | A-3 L63–75；B-2 L98–107；E-1 L273–284 | **S** |
| 5 | `08_AGENT08_ROUND1_DELIVERABLE_REVIEW.md` | 见下 | 37,052 | **全文**（本人 ROUND1 产物） | 全部 | A08-01~11 | S（自产） |

### 1.2 被复核对象（**原始证据**，B-06 / A-04 本体）

| # | 文件 | 路径 | 字节 | 读取范围 | 核实的问题编号 | 证据位置 | 证据类型 |
|---:|---|---|---:|---|---|---|---|
| 6 | `A-04-CONSISTENCY-REVIEW.md` | `E:\[PROJECT]\A-review\` | 40,072 | 定点（ROUND1 已读 L1–183 等） | A08-10、A08-09(A-19) | L5（时间）、L285、L467 | **P** |
| 7 | `A-04-CLOSURE-STATUS.md` | 同上 | 18,764 | 定点 | A08-10、A08-09(A-19) | L3（时间）、L49、L266 | **P** |
| 8 | `A-04-CLOSURE-V1.2-VERIFICATION.md` | 同上 | 7,995 | 定点 | A08-10、A08-08 | L3（时间）、L26、L68 | **P** |
| 9 | `_v12_verify.txt` | 同上 | 15,689 | **全文 L1–185** | A08-08 | L4–7、L11、L153–185 | **P（取证快照）** |
| 10 | `_v12_final_sha.txt` | 同上 | 1,447 | **全文 L1–41** | A08-10 | L2（时间）、L7–27 | **P（取证快照）** |
| 11 | `v12_verify.py` | 同上 | — | **全文 L1–165** | A08-08 | L12–13、L143–161 | **P（源码）** |
| 12 | `v12_finalize.py` | 同上 | — | 定点 | A08-10 | 含 `# 时间：2026-11-03 22:40` | **P（源码）** |
| 13 | `v12_fix_closure_s7.py` | 同上 | — | 定点 | A08-10 | 含 `## 七、v1.2 受控修订记录（2026-11-03 22:15` | **P（源码）** |
| 14 | `_backups\A-04-CLOSURE-STATUS.v1.1.bak-20261103-2200.md` | 同上 | 18,961 | Get-FileHash + 行数 | A08-09(A-17) | SHA-256 = `C640E96B…F49BFB`（64 位） | **P** |

### 1.3 实测命令证据（本轮新执行）

| # | 命令 | 输出要点 | 用于 |
|---:|---|---|---|
| 15 | `Get-ChildItem A-review -Filter *.py` + 逐文件读文本匹配 `22:05\|22:15\|22:35\|22:40` | 仅 `v12_finalize.py`(22:40)、`v12_fix_closure_s7.py`(22:15) 命中；22:05/22:35 **0 命中** | A08-10 |
| 16 | `Get-Item LastWriteTime`（A-04 四份产物） | 20:07:01 / 20:07:01 / 20:07:17 / 20:07:30 | A08-10 |
| 17 | `Get-Date` 采样 | 2026-11-03 21:31:11 +08:00；TimeZone = China Standard Time | A08-10 |
| 18 | `v12_verify.py` 全文读取 | L12 `def w(s): out.append(s)`；L159 `write("\n".join(out))`；L161 `print("allok =", allok)` | A08-08 四层区分 |
| 19 | `Select-String` / 逐行扫描 `05_REVISION_PLAN.md` 的 A-01~A-19 字段 | 提取「是否改变原结论」7 项、「证据依据」19 项 | A08-09 |
| 20 | 逐项提取 `06_CHANGELOG.md` L63–L81 的「改变结论？」列 | 标「是」= A-11/13/14/15/16 共 **5 项** | A08-09 |
| 21 | 集合运算：19 项 REV 映射到 F 编号 | 覆盖 15 个 F；**未覆盖 11 个**；P1 中 F-03/F-08 无 REV | A08-09 |
| 22 | `grep F-08` on `05_REVISION_PLAN.md` | 仅 L18/L379，**无任何 REV 项** | A08-09 |
| 23 | `grep 33 份\|审查深度` on `05_REVISION_PLAN.md` | **0 命中** | A08-03 影响面判定 |
| 24 | 两副本 `adjudication-ledger.md` 逐行 diff | 差异行数 = **1**（L32 路径脱敏）；均 391 行 / `split` 392 | A08-07 |
| 25 | `Get-Item` `02_CONFLICT_REGISTER.md` | 11,695 B（vs 记录 7,901） | A08-02 |
| 26 | 逐代理编号提取（7 份代理报告） | FB11+EM3+SC15+LC25+RC9+AU10+HV6 = 79；+LF24+SUP5 = **108** | A08-04 |

---

## 二、本轮**未**读取的文件（不得视为已核查）

| # | 文件 | 为什么没读 | 影响哪些结论 | 状态 |
|---:|---|---|---|---|
| N-1 | `AGENT-01-FILE-BASELINE.md`（580 行） | 仅 ROUND1 做元数据核验；本轮范围限定在 P0–P4 | A08-11 的「双重标准」判定 | `NOT_CHECKED` |
| N-2 | `AGENT-04-CONSISTENCY.md`（720 行） | 同上 | A08-11 | `NOT_CHECKED` |
| N-3 | `AGENT-03-STATISTICS.md` 全文 | 仅 ROUND1 定点读过 L67/L101–107/L126 | A08-04 的 108 明细 | `NOT_CHECKED` |
| N-4 | `AGENT-02/05/06/07` 全文 | 仅做编号提取 | — | `NOT_CHECKED` |
| N-5 | `01_AGENT_REPORTS\_agent01_baseline_*.csv` | 未逐行读 | A08-06 的 A 侧覆盖 | `NOT_CHECKED` |
| N-6 | `00_BASELINE.md` | ROUND1 全文读过，本轮未重读 | — | ROUND1 已记 |
| N-7 | `02_CONFLICT_REGISTER.md` 全文 | 仅读了目录 + L1–210（ROUND1） | — | 部分 |
| N-8 | 原始材料 `m01~m05`、`B-00~B-05` | **本轮不需要**（P0–P4 均不涉及原文） | — | `NOT_CHECKED` |
| N-9 | `release\...\adjudication-ledger.md` 全文 | 仅做逐行 diff 比对，未读全文 | A08-07 | 部分 |
| N-10 | B-06 五份交付物原文 | 仅用 ROUND1 已复核的数据 | — | ROUND1 已记 |

**明确声明**：**N-1 ~ N-5 是本轮 D 类判定 A08-11 的直接原因**——判定「是否存在双重标准」需读这两份代理报告的任务分工，本轮按「不扩大范围」原则未读。

---

## 三、证据可靠性分级

| 级别 | 含义 | 本轮使用于 |
|---|---|---|
| **P（原始）** | 被审查对象本身、取证快照、源码 | A08-08、A08-10、A08-09(A-17/A-19) |
| **S（二手）** | 交付集自述 | A08-01~07、A08-09 全部 |
| **M（仅元数据）** | 只取字节/行数/SHA，未读内容 | A08-06 的 csv/json 计数 |
| **N（未读）** | 见 §二 | A08-11（D 类） |

**重要**：A08-09 的结论**主要建立在 S 级证据上**（交付集自己的三份文件互相比对）。**但这类结论恰恰适合用二手证据判定**——「A 说 X，B 说非 X」可由文件本身直接证实，不需回源原始材料。

---

## 四、本轮未使用的证据（避免误导）

以下材料**存在且可读，但本轮未用于支撑任何结论**，如实列出以防被误读为已使用：

- `AUDIT-NEXT\01_AGENT_REPORTS\` 下全部 `_*.py`、`_*.csv`、`_*.md` 临时文件（18 个 `_agent03_scan*.py`、`_agent01_*.py` 等）——本轮**未执行**其中任何脚本；
- `AUDIT-NEXT\_lead_measure_b06.json` / `_lead_measure_a04.json`——ROUND1 读过条目数，本轮未用；
- `E:\[PROJECT]\A-review\_secondpass_result.txt`、`_consistency_evidence*.txt`、`_v12_facts.txt`——本轮未读（`05_REVISION_PLAN` A-17 引用 `_secondpass_result.txt` L20 作为 64 位 SHA 佐证，但**我已用 Get-FileHash 直接实测得到同一值**，无需依赖该转述）。

---

## 五、可复现性说明

本轮所有结论均可用下列方式复现：

```powershell
# A08-09：提取 A-01..A-19 的「是否改变原结论」字段
Select-String -Path 'E:\[PROJECT]\AUDIT-NEXT\05_REVISION_PLAN.md' -Pattern '是否改变原结论'
Select-String -Path 'E:\[PROJECT]\AUDIT-NEXT\06_CHANGELOG.md'   -Pattern '改变结论'

# A08-10：定位时间字符串来源
Select-String -Path 'E:\[PROJECT]\A-review\*.py' -Pattern '22:05|22:15|22:35|22:40'
Get-ChildItem 'E:\[PROJECT]\A-review\A-04-*.md' | Select Name,LastWriteTime

# A08-08：四层区分
Select-String -Path 'E:\[PROJECT]\A-review\v12_verify.py' -Pattern 'def w|allok|write\(OUT'
Select-String -Path 'E:\[PROJECT]\A-review\_v12_verify.txt' -Pattern 'allok'

# A08-04：107 vs 108
Get-ChildItem 'E:\[PROJECT]\AUDIT-NEXT\01_AGENT_REPORTS\AGENT-*.md' | ForEach-Object {
  $t = Get-Content $_.FullName -Raw
  [regex]::Matches($t,'(FB|EM|SC|LC|RC|AU|HV)-\d+') | ForEach-Object {$_.Value} | Select -Unique
}
```

---

## 六、SHA 记录（本轮实测，仅记录实际计算过的）

| 文件 | SHA-256 |
|---|---|
| `_backups\A-04-CLOSURE-STATUS.v1.1.bak-20261103-2200.md` | `C640E96BB60A8388C3DEB1219F3C9DDE46928AD25D4D02437A822BB8A2F49BFB` |
| `A-04-CONSISTENCY-REVIEW.md` | `95B82C4246D616E3…`（前16，ROUND1 实测） |
| `A-04-CLOSURE-STATUS.md` | `B6B9FFEE026D1669…`（前16，ROUND1 实测） |
| `adjudication-ledger.md`（release） | `d152471b81a396e8…`（前16） |
| `adjudication-ledger.md`（workspace/脱敏） | `d1de733eecc74062…`（前16） |

**未计算的不得补记。** 交付集自身 5 份文件（05/06/04/FINAL/08）的完整 SHA **本轮未逐一计算**——如需用于 A/B 修改前的基线冻结，须另行实测。

---

*本索引为新增交付物。未修改任何被审查文件。*
