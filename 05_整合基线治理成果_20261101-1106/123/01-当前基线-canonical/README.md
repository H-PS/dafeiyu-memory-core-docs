# README — Memory Core R4.1 V1 全量详细设计（V2 架构升级版）

> 交付：MC-R41-DESIGN-FREEZE-01 + **统一架构升级（2026-11-04）** ｜ 文档体系：**00–15 共 16 份** + 契约 + 图集 + 测试规格 + 治理文件
> 交付性质：**设计文书（不含代码实施授权）**。设计范围覆盖 P0–P6；**实施授权不变**（当前 P0–P3 = M01+M02，P3–P6 需阶段授权）。

## 1. 权威顺序（读我之前必读）

1. **`08-adjudication-ledger.md`**（`C:\Users\[USER]\Desktop\work\`）—— 正式裁决唯一权威。
2. **`DD-AUTHORITY-V2.md`** —— 本设计的**最高行为依据**：§2 全部枚举（semantic_type/anchor/tier/status/source_type/visibility/intent/read_mode/emotion_label）、§3 五对象字段表、§5 Pipeline 16 阶段、§7 九个 API、§8 检索五阶段、§9 ΔE 公式、§11 文档映射、§12 台账冲突 CR-01~07。
3. **`DD-AUTHORITY.md`**（v1）—— 历史基准与 R1–R10 冲突裁决；与 V2 冲突处以 V2 为准。
4. **`DOCUMENT-CONFLICT-REGISTER-V2.md`** —— 结构性冲突 SC-01~12、台账冲突 CR-01~07、文档级待统一项 DC-01~10、改造执行清单。

## 2. 文档体系（16 份）

| 编号 | 文件 | 内容 |
|---|---|---|
| 00 | 总体架构与设计原则 | V2 五原则/17 项能力/模块定位/16 份索引/P0–P6/旧→新映射 |
| 01 | 记忆本体与数据模型 | MemoryUnit(28 字段)/Evidence/Event/EmotionContext/PersonProfile + SQLite Schema + ER |
| 02 | 核心算法规范 | 五阶段检索/Anchor Router/RRF/规则排序/画像整合/ΔE/去重冲突/事务五步 |
| 03 | 系统运行与协作协议 | Pipeline 编排/9 API/Outbox/事务/幂等/错误 16 码/追踪链 |
| 04 | 权限与治理规范 | 十原则/权限矩阵/visibility 三值/证据继承/删除六层/审计/防注入 |
| 05 | 可靠性与运维规范 | 故障矩阵（含 LLM/情感/画像/证据新故障）/备份六步/恢复十步/对账/监控 |
| 06 | 测试与评估规范 | 9 层级/五态/三大关键测试/12 权限场景/污染 6 条/消融 A–F/8 组新测试 |
| 07 | 扩展与演进规范 | P0–P6/八+三维版本/迁移六要素/B-010/B-011/回滚/Future 登记 |
| 08 | DSH 与 QQ Bridge 集成 | 三组件边界/身份映射/会话隔离/读写落地/Psychology 边界/L3 事件 |
| **09** | **记忆摄取 Pipeline** | 16 阶段逐一定义 + 候选 6 态 + 降级 |
| **10** | **LLM 调用体系** | L1/L2/L3 分级/动态调用/11 项能力测试/结构化输出/成本限流 |
| **11** | **Memory Anchor 与语义分类** | 8 锚点/6 类语义/多标签/主体/人物优先/意图→锚点映射 |
| **12** | **人物画像整合** | 11 分区/basis 三值/自述优先/更新 7 步/版本 |
| **13** | **Evidence 与 EmotionContext** | 4 读取模式/原句追溯/事实情绪分离/ΔE 公式+8 门禁 |
| **14** | **Chat LLM 读写 API** | 9 接口详述/主动读写链路/时序图 |
| **15** | **参数注册表** | 15 属性 × 全参数/默认关闭项/变更流程 |

## 3. 契约层

```
contracts/
├── schemas/   13 份既有（memory-object / evidence / candidate / observation / relation /
│              search-document / embedding-space / outbox-event / permission-context /
│              retrieval-request / retrieval-response / context-response / error）
│              + 新增 7 份（memory-unit / event / emotion-context / person-profile /
│              anchor-set / read-request / read-response）
├── api/       api-contracts.md（REST v1 + M01–M05 + Context Adapter + Database Adapter）
├── state-machines/  state-machines.md（SM-1~SM-8）
├── config/    core/storage/retrieval/embedding/security/worker/test
│              + 新增 memory/emotion/anchor/profile
└── error-codes.md（16 规范码 + 12 兼容别名）
```

## 4. 其他目录

- `diagrams/diagrams.md` —— D-01~D-15
- `test-specs/test-specs.md` —— TS-1~TS-8
- `reviews/` —— 一致性报告 / 设计质量审查 / 裁决遵循 / 实施准备度
- `_sources/` —— m.md、sf.md 备份副本（只读）

## 5. 治理文件

| 文件 | 用途 |
|---|---|
| `DESIGN-DECISION-REGISTER.md` | 决策登记（状态纪律：PROPOSED 不得写成 APPROVED） |
| `OPEN-ISSUES.md` | 未决事项（OI-001~031） |
| `PENDING-USER-DECISIONS.md` | 第一轮历史待决清单；当前轮普通工程裁决以第二轮裁决登记表为准 |
| `DOCUMENT-CONFLICT-REGISTER-V2.md` | 冲突清单与统一裁决记录 |
| `DESIGN-STATUS.md` | 完成度看板（F1–F4） |
| `CHANGELOG.md` | 变更记录 |

## 6. 备份与 SHA256

改造前完整备份：`E:\[PROJECT]\备份全集\Memory-Core-R4.1-Design-pre-V1upgrade-20261104-1810\`（45 文件 + `_SHA256-BEFORE.csv`，逐字节校验通过）。

## 7. 给三类后续 Agent

- **代码 Agent（P0–P3）**：DD-AUTHORITY-V2 §2/§3 → 00 → 01 → 02 → 03 → contracts/ → 05 → 06 → test-specs。M05 契约默认拒绝；LanceDB FLAT 先过 VF-001；**`emotion.*` 三项默认关闭不得启用**。
- **测试 Agent**：06 + test-specs + schemas 可直接执行；Mock 标 MOCKED；真实模型与真实数据测试单列。
- **审查 Agent**：reviews/ 四份 + 冲突清单 + 台账对照。


## 第二轮审查材料（2026-11-04）

- `SECOND-ROUND-FULL-AUDIT-REPORT.md`：独立审查结论（当前未达到全包冻结）。
- `SECOND-ROUND-DECISION-REGISTER.md`：本轮裁决及验证状态。
- `SECOND-ROUND-ISSUE-REGISTER.md`：问题台账与关闭条件。
- `CONTRACT-FREEZE-REPORT.md`：契约冻结状态（当前未全包冻结）。
- `TECHNICAL-VERIFICATION-REPORT.md`：本机隔离验证结果。
- `V1-IMPLEMENTATION-BASELINE.md`：带门槛的实施基线候选。
- `SECOND-ROUND-CHANGELOG.md`、`evidence/second-round/`：变更与可复现证据。
