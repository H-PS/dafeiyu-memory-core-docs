# CHANGELOG — Memory Core R4.1 全量详细设计

## 1.0.0 — 2026-11-04（首版交付）

**新增**
- DD-AUTHORITY.md：设计基准；时间序冲突裁决 R1–R10（错误码/Outbox 事件名/五平面/ID/时间/状态机拆分等）；17 条硬约束；sf 后文强制条款摘要。
- 00–08 九份详细设计（15 节结构；合计 7,526 行、38 张 Mermaid）。
- contracts/schemas：12 份 JSON Schema（Draft-07，含合法样例；error.schema 为被引用的公共结构）。
- contracts/config：7 份配置 Schema（additionalProperties:false；测试/生产隔离；EXP-DEFAULT 标注）。
- contracts/api/api-contracts.md：REST v1 八端点 + M01–M05 + Context Adapter + Database Adapter。
- contracts/state-machines/state-machines.md：SM-1~SM-8（生命周期/验证/可用性/Outbox/Worker/迁移/删除/Embedding 迁移）。
- contracts/error-codes.md：16 规范码 + 12 兼容别名映射。
- diagrams：D-01~D-15。
- test-specs：TS-1~TS-8（含三大关键测试、权限 11 场景、消融 A–F、验收 19 项）。
- DESIGN-DECISION-REGISTER.md（78 项登记）；OPEN-ISSUES.md（OI-001~014 + 子文档 OI 汇总）。
- reviews/：四份审查报告。

**决策（PROPOSED，待批）**
- PD-001~PD-014（ID 格式/13 表结构/WAL/敏感度枚举/权限矩阵/revoke 语义/审计保留期等）。

**明确不做**
- 无代码实施；无服务启动；未触碰 D:\qq-bridge、台账、冻结稿；未扩大 P0–P3 授权（AD-038）。

## 修订规则（sf 后文 §18.2）

任何变更必须：引用原设计→新设计→原因→影响模块→数据兼容→安全影响→测试影响→迁移影响→回滚→是否需用户裁决，并在此登记。

## 2.0.0 — 2026-11-04（V2 架构升级）

**依据**：用户《Memory Core R4.1 V1 统一架构升级与设计文书迭代总任务书》（最高优先级架构任务）

**新增文档（7 份）**
- 09-记忆摄取Pipeline.md（16 阶段，1131 行）
- 10-LLM调用体系.md（L1/L2/L3，683 行）
- 11-MemoryAnchor与语义分类.md（8 锚点 + 6 语义类，972 行）
- 12-人物画像整合.md（11 分区 + basis 三值，759 行）
- 13-Evidence与EmotionContext.md（4 读取模式 + ΔE，1035 行）
- 14-ChatLLM读写API.md（9 接口，1107 行）
- 15-参数注册表.md（15 属性，537 行）

**新增治理文件**
- DD-AUTHORITY-V2.md（架构升级基准：枚举/字段/API/映射/CR-01~07）
- DOCUMENT-CONFLICT-REGISTER-V2.md（SC-01~12 / CR-01~07 / DC-01~10）

**改造文档（9 份）**
- 00：插入 §0 V2 升级摘要（五原则/17 能力/模块定位/16 份索引/旧→新映射/路线图）
- 01/02/03/04/05/06/07/08：插入 V2 对齐头（结构冲突声明 + 新增能力指向 + 保留项声明）

**契约**
- 新增 7 Schema：memory-unit / event / emotion-context / person-profile / anchor-set / read-request / read-response
- 新增 4 Config：memory / emotion / anchor / profile；retrieval 增量（evidence_expansion_limit / budget_chars / anchor_routing_enabled）
- 合计 20 Schema + 11 Config，JSON 全部校验通过

**决策与问题**
- DESIGN-DECISION-REGISTER 增 AD-039~056（V2 架构决策）+ VD-001~008（默认关闭项）
- OPEN-ISSUES 增 OI-025~031（7 项台账冲突）
- 备份：E:\[PROJECT]\备份全集\Memory-Core-R4.1-Design-pre-V1upgrade-20261104-1810\（45 文件 + SHA256 清单）

**明确不做**
- 无代码实施；无服务启动；未触碰 D:\qq-bridge；未改台账；未扩大实施授权

**修订规则**（沿用 v1 §20.4）：任何变更须引用原设计→新设计→原因→影响模块→兼容性→安全/测试/迁移影响→回滚→是否需裁决，并在此登记。