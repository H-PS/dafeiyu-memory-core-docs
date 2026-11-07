# Memory Core R4.1 — 交付报告（关键信息版）

**日期**：2026-11-05 · **基线**：`E:\[PROJECT]\r41-baseline` · **完整报告**：`07-reports/MEMORY-CORE-R4.1-FINAL-TECHNICAL-AND-IMPLEMENTATION-REPORT.md`

---

## 一、总体状态（六个轴，各自独立）

| 轴 | 状态 |
|---|---|
| 设计完成度 | **PARTIAL** |
| 合同冻结 | **NOT_FROZEN**（0 / 18 族冻结） |
| 技术验证 | **PARTIALLY_VERIFIED** |
| 代码落地 | **VERIFICATION_CODE_READY** |
| 工程准入 | **PARTIALLY_READY**（仅 M01 部分） |
| **正式实现授权** | **NOT_AUTHORIZED**（未变） |

规模：**648 条证据记录 · 147 个可运行文件 · 9 份 TASK-RESULT · 13 份合同 · 0 结构缺陷**
（Stage G 8/8 通过，重复运行稳定）

---

## 二、四项关键技术结论

| 技术 | 状态 | 关键实测 |
|---|---|---|
| **SQLite** | `PARTIALLY_VERIFIED` | 55 项检查 **0 失败**；WAL、事务、锁竞争、迁移、备份恢复全通；**SIGKILL 中断事务回滚 25 行→0**；busy_timeout 实测 929ms |
| **LanceDB** | `PARTIALLY_VERIFIED` | 扫描路径精确性成立；但 **ANN recall@10 = 0.30 且 nprobes 无效**；`createIndex` **忽略 indexType/distanceType** |
| **中文 FTS** | `PARTIALLY_VERIFIED`，**产品准则 FAIL** | `unicode61` 把整段 CJK 当一个 token（run-equality）；**19 个中文词查询中 4 个零命中**；最佳替代 0.956 hit@10，代价 3× 索引 |
| **宿主身份** | `SYNTHETIC_IDENTITY_VERIFIED` | 222 项断言，加固配置 **0 失败**；6 条 FAIL 是**负对照**（故意破坏的弱配置），各配一条加固后 BLOCKED |

**四项全部不能冻结。** 原因：SQLite 无对比证据（本机无包管理器）；LanceDB 索引路径失效；FTS 中文不达标；身份未接触真实宿主。

---

## 三、阻塞级缺陷（P0）

| ID | 缺陷 | 影响 |
|---|---|---|
| **OI-R-15** | **权威自身默认值非法**：§3.1 默认 `ACTIVE`+`unverified`，但 schema `allOf[1]` 要求 `ACTIVE⇒confirmed\|disputed`。实测 `'unverified' is not one of ['confirmed','disputed']` | **阻塞全部写入路径** —— 新建记忆无合法状态 |
| **OI-R-10** | `outbox-event.schema.json` 用 `additionalProperties:false` 拒绝 7 个租约字段（`lease_owner`/`claim_token` 等） | M01 无法持久化它自己要求的 worker 协议 |
| **OI-R-12** | 恢复旧备份**复活已删记忆、重授已撤销权限** | 安全事故，非便利问题 |

已修复两项：**OI-R-04**（evidence schema 拒绝任何合规实例 → v2 已交付）、**OI-R-05**（BOM 致 JSON 解析失败）。

---

## 四、需要你裁定的架构决策

| ID | 问题 | 为什么必须你定 |
|---|---|---|
| **E-1** | `Memory-Core-R4.1-Design`（含 contracts/）与 `R4.1-UNIFIED-MEMORY-CORE` 谁权威？ | 两套**互不引用**，7 处架构冲突；选谁决定哪些文档是法 |
| **E-2** | `memory_tier` 转换规则？ | 规则**已存在但分散**（`11 §7.12` 有 3×6 矩阵+5 条规则），但**无任何状态机转录**，且缺 3 项（执行判据/L3 门槛/降级路径） |
| **E-3** | `content_kind` 废除还是必填？ | V2 说废除，但 01 号文档必填、schema 要求 8 值枚举 |
| **E-4** | `memory-object`（v1）与 `memory-unit`（V2）谁是权威？ | contracts/ 同时发两份，无弃用标记 |
| E-5 | 备份保留期 | 全库无数字，编造即虚构政策 |
| E-6 | 向量通道：scan-only 还是 indexed？ | indexed 意味着接受 0.30 召回 |

---

## 五、本轮我自己犯错，一并记录

| 错误 | 发现方式 |
|---|---|
| **越权写入**：从只读 provenance 树删除 BOM（20:11:26） | 三个独立工作流各自检出；影响面已量化：1 文件、3 字节、CJK 完好 |
| 误判 3 条 FAIL 为"假失败" | 重跑证明：1 假 2 真（tier 升迁确实被拒） |
| 误称权限 schema "无请求者字段" | 重读证明：`actor` 即请求者，真实缺陷是 `role` 可选 |
| 被代理反驳：称 error schema 无法强制规范码 | 执行证明相反；真实缺陷是 `error-codes.md §2` 撒谎 |

另有代理用未文档化的 `tier_review_approved`（全库 0 处出现）把红判绿，已驳回并要求改为 `PARTIAL_PASS`。

---

## 六、下一步

**M01 先行**，前置只需三件廉价动作（非工程工时）：
1. 合入 v2 Evidence schema
2. 扩展 outbox schema（或租约状态移到独立表）
3. 在声明范围内冻结 `node:sqlite`

之后 M02，受两个**决策**（非更多验证）门控：E-2（tier 规则）、中文 tokenizer 选择。

**正式产品实现仍需你单独授权。** 本轮所有代码均为隔离验证代码，不构成、也不得读作产品实现的开端。
