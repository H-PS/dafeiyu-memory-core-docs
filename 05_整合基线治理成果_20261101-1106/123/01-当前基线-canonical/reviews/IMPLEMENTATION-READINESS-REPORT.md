# IMPLEMENTATION-READINESS-REPORT — 实施准备度报告

> 2026-11-04 ｜ 目标读者：P0–P3 代码 Agent ｜ 结论先行：**P0 验证工作可在任务授权范围内开展；正式产品实现未获授权；P1/P2 各有 1 个前置验证项；P3 依赖前两者通过。**

## 1. 准备度总览

| 阶段 | 范围 | 设计完备度 | 前置条件（阻塞项） | 就绪判定 |
|---|---|---|---|---|
| P0 隔离验证 | 驱动/Schema/FLAT 证据 | 文书定义完成，实测部分未完成 | VF-001/VF-002 | ⚠️ 仅授权验证，不是产品实现授权 |
| P1 M01 | SQLite CRUD/版本/事务/Outbox/FTS5+LanceDB 同步基础/备份 | **95%**（01 表结构 PROPOSED 待批 PD-002；03 事务协议完备） | OI-002 驱动裁决（P0 产出）；PD-002 批准 | ⚠️ 待 P0 结果 |
| P2 M02 | FTS5/Embedding Adapter/LanceDB FLAT/RRF/规则排序/回源/权限契约/预算/Context Builder | **95%**（02 算法 17 要素；契约 4 份；14 号算法规格背书） | OI-001 FLAT 路径证明（否则 BLOCKED）；OI-003 分词器 | ⚠️ 待 VF-001 |
| P3 闭环验收 | M01+M02 联调、一致性、权限边界、备份恢复、E2E | **100%**（06+test-specs 全部用例与判定；19 项验收清单） | P1/P2 通过 | ⚠️ 串联 |

## 2. 代码 Agent 开工清单（P0 第一天）

1. 读 DD-AUTHORITY §5/§6（硬约束 17 条 + sf 强制条款）→ 00 → 01 → 02 → 03 → contracts/ → 05 → 06 → test-specs。
2. 建 `E:\prototype\memory-core`（禁止放 D:\qq-bridge 内）。
3. 执行 05 验证清单 10 项；执行 VF-001（LanceDB FLAT 八项基准比对：暴力余弦 vs SDK 全量一致）。
4. 执行 VF-002（SQLite 驱动三候选事务/并发/迁移/恢复实测）→ 把选型写入 P0 交付记录（这是 OI-002 的解锁动作）。
5. Schema v1 落库（按 01 §7.16 十三表建议，PD-002 待批项以 PROPOSED 注记实现）。
6. 交付：环境报告 + 驱动裁决记录 + FLAT 验证证据 + Schema + 目录骨架。**停在 P0，等待「继续 P1」。**

## 3. 必须内化的红线（违反=停止）

- reinforce 只预留 schema 禁止写入（BL-001）；confirmed 仅 M04 通道（当前 M04 未实现 ⇒ 无任何 confirm 路径）。
- M05 契约默认 DENY（m05_mode=contract_deny；生产禁 allow-all；测试替身仅 test 环境）。
- LanceDB 不可用 ⇒ 不降级只报错（OI-004 未批准）；禁止 SQLite Vector Adapter 替代。
- Token≠字符；15 参数 EXP-DEFAULT 可配置不得硬编码。
- 每次修改先备份（六步协议）；测试结果五态；Mock 标 MOCKED。
- D:\qq-bridge 只读禁区；不启动正式服务；P3 后停（P4+ 另行授权）。

## 4. 已知风险与缓解

| 风险 | 等级 | 缓解 |
|---|---|---|
| LanceDB SDK FLAT 实为 ANN | 高（阻断 P2 验收） | VF-001 暴力基准比对；失败⇒BLOCKED 报告，不换引擎 |
| 中文分词质量（unicode61） | 中 | OI-003：P2 前对比候选分词器；Benchmark 分通道报告 |
| A 层迁移语义冲突（OI-015） | 中 | 迁移属 P4+；P0–P3 用合成数据，不受影响；裁决后再做迁移设计 |
| 权重未调优 | 低 | EXP-DEFAULT 可跑通；P3 基线后 Benchmark 校准 |
| 宿主接口未验证 | 低（P9 前置） | 08 VERIFY-08-01…12 清单；不编造宿主 API |

## 5. 交付物映射（代码 Agent 每阶段要交什么）

| 阶段 | 必交 | 模板/依据 |
|---|---|---|
| P0 | 环境报告、驱动裁决记录、FLAT 验证证据、Schema、目录 | 05 验证清单；§16 五项验证 |
| P1 | M01 代码+测试（TS-1 全 PASS，三大关键测试必过） | test-specs TS-1 |
| P2 | M02 代码+测试（TS-2/3/4 PASS；TC-M02-002 FLAT 必过） | TS-2~4 |
| P3 | Benchmark 基线报告、故障/恢复/E2E 报告、对账 0 差异、PROTOTYPE-PROGRESS.md、偏差清单、已知问题 → **停止** | TS-5~8；sf 后文 §13.4 九项 |
