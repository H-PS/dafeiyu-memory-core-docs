# D-REPORT — Persona Prompt 初始化与逐轮组装（过程报告）

> Agent D · MC-CORE-CAPABILITY-REVIEW-01 · 2026-11-06 · 只读审查，未修改任何基线文件
> 交付件：E:\[PROJECT]\CORE-CAPABILITY-REVIEW-2026\04-PERSONA-PROMPT-FLOW.md
> 证据：EVIDENCE\d\{persona-evidence-facts.md, persona-card-extraction-mapping.md, persona-config.schema.draft.json, token-budget-pressure-calc.md}

## 工作过程

1. **通读必读材料**：07-记忆库框架（383 行全文，人格卡 v1.2 十节逐字 + 三层注入实证）；R3 04-人格管线方案 4 份（SPEC 的 MACHINE HEADER/CU-1–CU-8/验收断言/阻塞项、实施方案 P0–P7、报告5 §一/§三/§四、审计版改动 1–4）。
2. **canonical 取证**：00（Persona FUTURE 五处 + 反面清单 + 依赖矩阵）、08（注入边界/宿主唯一拼装者/人格红线/OI-08-07）、14（九接口/预算/propose 纪律）、10（prompt 版本管理 §7.7 + Token 五分类）、12（画像 11 分区 + basis）、15（参数注册表 grep persona=0）、02（ALG-06，经 B 路清洗副本）、contracts（read-request/context-response/profile/memory schema、state-machines grep=0）。
3. **取证报告交叉**：09-CONFLICT-REGISTER grep Persona=0（49 FC 无专项）；03-ARCHITECTURE-FACTS 依赖矩阵 Persona 行与 15 号基线七协同模块佐证。
4. **方法验证**：人格卡 v1.2 十节逐节映射结构化槽位（7 节可机械抽取 / 2 处 owner 确认 / 1 处 core_text 兜底）；persona-config 草案 schema 通过 jsonschema 4.26.0 Draft7 check_schema + 实例校验。
5. **量化推演**：1200 token 预算挤占计算（完整卡 ≈1250 token > 1200；瘦身 350 字符 ≈280 token）。

## 结论摘要

- **初始化 = CONCEPT_ONLY，逐轮 = PARTIAL（记忆侧 MOSTLY_COMPLETE / Persona 侧 CONCEPT_ONLY）**。
- P0×2：D-C001 无初始化管线；D-C002 无 Persona 层逐轮契约（49.8% 占比 + compaction 吃卡）。
- P1×4：D-C003 记忆覆盖人格仅原则级防护；D-C004 无版本管理（hash×19 实证）；D-C005 注入频率裁决悬置（R3 锚定 vs 07 每轮全文，直接冲突且未关闭）；D-C006 情绪↔人格接口未定义。
- P2×1：D-C007 机器人自述偏好 vs 用户画像混用风险。
- 关键判定：Persona 调整**不需要也不应该**改 Memory Core（00 L271 反面清单 + Persona=记忆消费者边界）；全部 NEW PROPOSAL 落宿主侧，跨侧唯一接口是 Memory API（P5）。

## 遵守与偏差记录

- 只用 write/edit 写文件；未触碰 C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design 与 V0.9 代码；02 号含 NUL 文件改读 B 路清洗副本并按章节号引用。
- draft schema 明确标注 DRAFT-UNREVIEWED，禁止写入 canonical contracts/。
- 所有 NEW PROPOSAL 显式标注，未与 EXISTING DESIGN 混写。
- 向用户提问：无（按任务规则自行判断，判断依据均写入交付件 §0/§8/§12）。
