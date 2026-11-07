# 05-STATE-MACHINES · 状态机与状态转换

> 依据：m01 D7（存储四态）、m04 §九（四维状态）、m04 §7.4（任务态）、v4.6 候选（检查四态/请求 15 态/聚合规则）、A5 阶段五审查（S5 系列发现）、15-sm 摘录（10 态可确认）。
> 本文件是全部状态机的**统一登记**；A 侧确认的定义缺口在此如实标注（S5-02/S5-03 为 P0，修复方向已列，待裁决）。

---

## 一、记忆生命周期状态（四维分离，m04 §九，已冻结）

四维不合并为单一枚举；组合须过合法性校验：

```
审核维度:  candidate → pending → approved          （M03 分流 → M04 终审）
有效维度:  active → expired | superseded            （时间四字段驱动，M04 判断）
存储维度:  normal → archived → deleted              （M01 持久化执行；Deleted 立即屏蔽 Recall）
权限维度:  authorized → revoked                      （M05 判定；撤销立即阻断，异步清理）
```

## 二、任务状态机（M04 §7.4，已冻结）

```
queued → ready → running → succeeded
异常分支: → retry_wait → (指数退避) → running
         → paused / cancelled / expired / dead_letter
每任务: task_id/task_type/priority/idempotency_key/checkpoint/retry_count/max_retries/status
优先级: P0 权限撤销·删除·安全 > P1 当前对话必要更新 > P2 普通提取·视图 > P3 一致性维护·索引优化
```

## 三、Event-L3 检查状态机（v4.6 候选 §2.3.13，**候选+审查修订**）

### 3.1 检查级四态

| 状态 | 判据明确度（A5 审查） | 修订方向 |
|---|---|---|
| `not_started` | **极低——全材料无正面判据**〔S5-02 P0〕 | V1 必须补定义/判据/退出（V1 必须 #2） |
| `not_executable` | 高（引用字段存在但对象不可取得等） | 保持；无退出条件=吸收态——V1 简化为"保持不 passed 直至超时/重试" |
| `passed` | 中低（单项级无定义式判据） | V1 必须补单项级定义 |
| `failed` | 中低（A 类命中） | 保持 |

互斥链（13.3 五不等式）：not_started ≠ not_executable ≠ failed ≠ passed；单项 passed ≠ 整体 passed；已发现错误不得因另一项 not_executable 被隐藏。

### 3.2 整体有效性聚合（13.4a，线性全序）

```
任一必需检查 failed        → 整体 failed
否则 任一 not_executable   → 整体 not_executable（不得 passed）
否则 任一 not_started      → 整体 not_started
否则                        → 整体 passed
```

配套必要条件（13.4b）：所有必需检查已执行且通过；无 A 类命中（含 A8 依 12.4 确认）；检查 II 通过；必要的版本完整性检查已执行（结论可 unknown）；**所有结构和身份绑定校验所必需的引用已完成核验**（语义映射证据可核验性不在此列——13.5b 承重括注，A5 确认冗余度为零）；任何必需检查无法执行 → 整体不得 passed。

### 3.3 unknown 三分（13.5）

| 型 | 触发 | 检查状态 | 版本 result | 整体可否 passed |
|---|---|---|---|---|
| a 合法缺失 | mappingEvidenceRef/removalBasis 字段合法缺失（结构可选） | passed（隐含） | unknown | **可以** |
| b 语义不足 | 对象可取得、结构通过、语义不可证 | passed（隐含） | unknown | **可以** |
| c 必需检查无法执行 | 字段存在但对象不可取得 | **not_executable** | unknown（fail-closed） | **不得** |

边界：a 与 c 由"引用字段是否出现"唯一决定；A4 明确错位不降为 unknown（failed→invalid）；A1 核心缺失不套 B1；not_executable 不得伪装成普通 B1。

### 3.4 版本级 result 聚合（规则 1–6，固定顺序首个命中）

| 规则 | 条件 | result | 审查修订 |
|---|---|---|---|
| 1 | A 清单任一情形 | invalid | **S5-08/S5-10〔修订〕：按 A 项逐项状态判定，not_executable 分支不得吞成 invalid**（候选文本规则 1 未同步——修订方向待 V-10 系列裁决） |
| 2 | 任一版本 unknown | unknown | B4 后果未定（H5-04） |
| 3 | 任一版本 partial | partial | 仅审计汇总 |
| 4 | 全部 removed | removed | 空集真空双命中〔S5-03/A-STAGE5-ISSUE-003，待裁 V-9〕 |
| 5 | 全部 complete | complete | 同上 |
| 6 | complete 与 removed 并存 | partial | 仅审计汇总，不参与 L3 |

partial 双来源（规则 3/6）均不参与 L3 判定（L127 明文）；聚合 partial 不污染其他 complete 版本。

**〔交叉复核修正〕A 线内部版本漂移登记**：S5-01 定级存在同级文本不一致——`A-05-CHECKS-STATES-AGGREGATION-REVIEW.md`（24:20，P0×3/PARTIAL_PASS）与 `A_STAGE5_FINAL_AUDIT_REPORT.md`（24:00，**P0×1（仅 S5-01）/CONDITIONAL_HOLD**）并存，A7 采纳前者未说明取代关系。本方案处理：S5-01 的 P0 定级**两条路径双验证可辩护**（条文冲突性质）；S5-02/S5-03 在早期 STAGE5 报告中未定级为 P0、在 A-05/A7 中升级为 P0——按升级后口径登记（判据：not_started 无定义与请求级无落点均为"四态/请求状态机不完备"级缺口），同时保留此漂移记录供用户复核。

## 四、请求级状态机（v4.6 候选，15 态仅 10 态可确认——G-4 材料缺口）

可确认态（15-sm 摘录，红队 RT-18 修正口径）：**10 态总数（其中 9 个态名可枚举：submitted / accepted / turn_bound / evidence_ready / l3_confirmed / l3_rejected / evidence_timeout / l3_revoked（预留，机制未定义 S5-10）/ bridge_consistency_checked〔名称待裁 5b〕；第 10 态仅存在于台账计数、态名未见于可确认摘录）** + 7 条迁移可确认；其余 5 态原文缺失（v4.5 §2.3.6.2 未取得，**不补造**）。

**A5 S5-03〔P0〕**：not_executable / not_started 在请求级**无落点**——10 态终态清单（含 l3_rejected/evidence_timeout）不含"无法核验"类终态。**修订方向（V1 必须 #3）**：补 `l3_unevaluable`（或等价）终态承载"整体 not_executable/not_started"情形，确保任何请求都有终态归宿、悬空请求不残留。此为设计修订提案，待用户裁决。

**终态迁移约束**（可确认部分）：

```
accepted → turn_bound（turn 绑定）
turn_bound + 第一阶段观察记录 → evidence_ready
evidence_ready + 检查I/II/III → l3_confirmed（仅整体 passed 且版本 complete）| l3_rejected（failed）
evidence_timeout：不可恢复终态；迟到证据不进入任何检查；重试须新 correlationId（T-10）
```

## 五、健康与降级状态（三态，M01 D28 + M04 §14.3 统一枚举）

```
normal / degraded / abnormal
- 缓存库(cache.sqlite)异常 → 不降级整体健康，只降缓存命中（P3）
- 权限/删除/安全状态不确定 → 阻断受影响访问（P0，非降级而是拒绝）
- 探针失败只降级不阻塞（F4），F1 类能力除外
```

## 六、状态机层未决问题汇总

| # | 问题 | 级别 | 裁决去向 |
|---:|---|---|---|
| 1 | not_started 无定义/判据/退出 | **P0** | V1 必须；14 号 V 组 |
| 2 | not_executable/not_started 请求级落点 | **P0** | 修订提案 l3_unevaluable；14 号 |
| 3 | 13.2 缺整体有效性门（V-10） | **P0** | 08 号 §五；14 号 |
| 4 | 空集双命中（V-9） | P1 | 14 号 |
| 5 | l3_revoked 撤销机制未定义（S5-10） | V1 预留 | 13 号 |
| 6 | 15 态其余 5 态原文缺失 | 材料缺口 | 不补造；14 号 |
| 7 | 版本级四态/result 五值无原文级完整定义（S5-04/S5-07） | P1 | V1 必须 #12；14 号 |

---

*状态机登记完毕。Event-L3 判定链与证据结构见 08。*
