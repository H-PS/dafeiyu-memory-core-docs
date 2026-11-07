# CMB v0.1 全量技术尽调 — 执行摘要（00-EXECUTIVE-SUMMARY）

**审阅材料包**：`E:\[PROJECT]\CMB-v0.1-review-pack\`
**生成时间**：2026-11-06
**性质**：只读技术尽调（未修改任何被审对象）

---

## 一、项目状态

| 项 | 值 |
|---|---|
| CMB v0.1 | 独立原型完成，**不可接入真实系统** |
| billion-context | `0aace2f`，与 2026-11-04 基线一致 |
| Memory Core | **路径已变更**：`R4.1-memory-bank-input` → `Memory-Core-R4.1-Design`；纯设计仓库，**无实现代码** |
| DSH | **不可读**（asar 打包，213MB） |
| QQ Bridge | 可读，非 git |

---

## 二、核心结论（三句话）

1. **CMB v0.1 是一个诚实的骨架，但有 6 个未发现的 BLOCKER** —— 78 条测试全绿，然而每一个 BLOCKER 都逃过了测试。
2. **真实数据源接入在架构上不可能** —— 上游 `messages` 表没有 speaker 列，CMB 又无条件强制 speaker，导致真实库上产出 **0 条 Evidence**。
3. **Memory Core 无法接入** —— 它不是"接口没对齐"，是**根本没有实现代码**；且其 `evidence.schema.json` 连"存原句"这个核心能力都不支持。

---

## 三、六个 BLOCKER

| 编号 | 问题 | 证据 | 谁能修 |
|---|---|---|---|
| **D-01** | speaker 链断裂 → 真实源零产出 | 上游无 speaker 列 + `bc-source.ts:211` 恒 null + `validator.ts:71` 无条件强制。**实测 `evidence_written=0`** | 用户裁决 + CMB/Host |
| **D-02** | `content_hash` 应为 `sha256(original_text)`，实际用 blob 哈希 | `13-…md:255` MUST；`validator.test.ts:43-51` 把违规锁成正确 | CMB |
| **D-03** | payload 变但 key 不变 → COMMITTED 永久锁死 | `bridge-ledger.ts:273` vs `bc-source.ts:168`。**实测 3 轮 `STATE_RESET_FAIL`** | CMB |
| **D-04** | session_id 含冒号 → locator 解析错 → 撤销失效 | `reconciliation-engine.ts:258-262`；QQ Bridge `convKey=kind:id`。**实测 revoke=0** | CMB |
| **D-16** | Memory Core 无实现 | 无 `src/`、无 `package.json`；M03/M04 标 FUTURE | Memory Core |
| **D-17** | Evidence schema 缺 `original_text`/`speaker_id` + `additionalProperties:false` | `evidence.schema.json:7,8` | Memory Core |

---

## 四、其他关键发现

### CRITICAL
- **D-05 系统提示词进入 Evidence**：`validator.ts:95` 查 `content_type==="system"`，但上游 system 提示词是 `role="system"` + `content_type="text"`。**实测系统提示词直接成为 Evidence**。

### HIGH
- **D-06 所有 mode 都写入**：`cfg.mode` 仅在日志和报告中读取。**实测 OFF/DRY_RUN 全部 `evidence_written=1`**。
- **D-07 未知类型静默放行**：黑名单只拦两个字符串。**实测 `mystery-kind`/`attachment`/`image` 全部无诊断通过**。

### MAJOR（择要）
- 全部 78 条测试用 `:memory:` ledger，**跨进程崩溃恢复从未验证**
- M-2 未修复：部分覆盖仍判 COMMITTED（**实测 3 覆盖 1 有效 → missing=0**）
- 媒体未 hydrate：多模态消息原文丢失
- B-1 修复漏了 UNSUPPORTED 状态
- 适配器隐含 `user_version >= 10` 门槛无检查
- "加密的 index.db" 分支**不存在**（加密时根本不建库）

---

## 五、主要风险

| 等级 | 数量 |
|---|---|
| BLOCKER | **6** |
| CRITICAL | **1** |
| HIGH | **2** |
| MAJOR | 31 |
| MINOR | 10 |
| INFORMATIONAL | 1 |
| **合计** | **58** |

- **阻塞真实接入：18 项**
- **阻塞真实数据验证：17 项**
- **可由 CMB 单方修正：11 项**
- **必须改上游：3 项**
- **需用户裁决：6 项**

---

## 六、四项遗留裁决摘要

| 编号 | 议题 | 推荐 | 关键理由 |
|---|---|---|---|
| **D-1** | Evidence Schema 权威冲突 | 方案 B：Memory Core 修订 schema 到 V2 标准 | V2 是文档声明的权威；schema 缺 `original_text` 等于禁止存原句，与 13 号文档核心要求矛盾；内部审查"假绿"证明是升级遗漏 |
| **D-2** | tenant / bot 定义 | `tenant` 用固定常量 `"default"`；`bot` 用 `self_id` | 单人部署无多租户概念；`self_id` 是真实可得的 |
| **D-3** | 时间戳可空 | 接受 `timestamp=null`，补 `recorded_at` = 桥接时间 | 上游 `ts` 二义无法区分；13 号文档明确"未知 MUST 为 null" |
| **D-4** | Block 删除窗口 | 接受限制，但必须定 SLA | 上游无墓碑机制；不可通过 CMB 解决 |

**新增两项待裁（本轮发现）**：
- **D-5**：speaker 来源五方案（上游补列 / 宿主注入 / 降级用 role / 放宽强制 / 放弃真实接入）
- **D-6**：`content_hash` 改语义后，blob 哈希存哪

---

## 七、真实数据实验准备情况

**12 项实验卡已完整**（`15-REAL-DATA-EXPERIMENT-PLAN.md`，767 行）：
- P0 = 5 项：E-1 结构恢复 / E-2 CAS 六维 / E-3 WAL 只读并发 / E-6 身份映射 / E-8 契约兼容
- 需真实数据 10 项、需用户授权 4 项、可纯合成 2 项
- **零项需要写入**

**但当前不建议立即执行** —— 需先修 4 项 CMB 单方可修缺陷（D-59/D-04/D-06），否则实验会得到"零结果"或"直接报错"。

---

## 八、是否建议进入受限真实数据验证

### **不建议现在进入。**

**必须先完成（CMB 单方，无需授权）**：
1. 修 D-59（`user_version` 检查）—— 否则旧库直接报错
2. 修 D-04（locator 分隔符）—— 否则撤销类实验失效
3. 修 D-06（mode 生效）—— 否则"只读验证"会真写入
4. 修 D-02/D-05/D-07/D-08/D-10 —— 数据正确性前置

**必须先由用户裁决**：
5. **D-5（speaker 方案）** —— 否则真实库上产出 0 条 Evidence，实验无意义

**修完后**：建议进入 v0.2 受限真实数据验证。

---

## 九、交付物索引

> ⚠️ **本文已有修订（2026-11-06）**：Agent D 交叉复核发现主代理 12/13 号报告有 15 项差异（3 高危），
> 涉及"测试数量口径""SEC-7 假安全""perf 复跑不可信""DB 复用导致虚高"等。
> 修订记录见 `evidence/LEAD-SELF-CORRECTIONS.md`，独立复核报告见 `12a-TEST-CREDIBILITY-CROSSCHECK.md`。

**关键数据更正**：

| 项 | 初版 | 更正后 | 依据 |
|---|---|---|---|
| Small 性能 | 500 evidence / 1180ms / 847 msgs/s | **340 / 875ms / 1143 msgs/s** | Agent D + 主代理双重实测（清 DB 后） |
| 性能可复现性 | "可复现 ✅" | **FAIL**（DB 复用 + 随机夹具） | `perf-run.ts:161` 固定路径不清理；`:105` 用 `Math.random()` |
| SEC-7 system 排除 | "PASS" | **RED（暴露真缺陷）** | 实测 `role=system+contentType=text` → `evidence_written=1` |
| 弱测试数 | 1 | **3** | 另有 RG-B1 析取式断言、SEC-5 无前置断言 |
| 回归测试条数 | "12 条" | **11 条** | `PHASE6-CHANGES.md:44` |
| 静态调用点 | 74 | **75** | Agent D 三法互证，总数 78 不变 |
| B-1 遗漏状态 | UNSUPPORTED | **UNSUPPORTED + COMMITTED 漂移** | Agent D D-7 |
| CPU | "无法采集" | **i9-12900H / 20 逻辑处理器** | Agent D 经注册表取得；仅 RAM 不可得 |

| 类别 | 文件 |
|---|---|
| **本摘要** | `00-EXECUTIVE-SUMMARY.md` |
| CMB 本体 | `01-CMB-SOURCE-AUDIT.md` / `02-CMB-DATAFLOW.md` / `03-CMB-STATE-MACHINES.md` / `04-CMB-DATA-DICTIONARY.md` / `01a-CMB-SOURCE-AUDIT-CROSSCHECK.md`（Agent A 复核） |
| 上游契约 | `05-BILLION-CONTEXT-AUDIT.md`（98KB，最详尽）/ `06-MEMORY-CORE-CONTRACT-AUDIT.md` / `07-HOST-IDENTITY-AUDIT.md` / `08-EVIDENCE-SCHEMA-DIFF.md` |
| 可靠性与安全 | `09-IDEMPOTENCY-TRANSACTION-RECOVERY.md` / `10-LIFECYCLE-REVOCATION-DELETION.md` / `11-SECURITY-PRIVACY-AUDIT.md` / `12-TEST-CREDIBILITY-AUDIT.md` / `12a-TEST-CREDIBILITY-CROSSCHECK.md`（Agent D 复核）/ `13-PERFORMANCE-REVIEW.md` |
| 下一阶段准备 | `14-USER-DECISIONS.md` / `15-REAL-DATA-EXPERIMENT-PLAN.md`（767 行）/ `16-INTEGRATION-ARCHITECTURE-OPTIONS.md` / `17-VERSION-ROADMAP.md` / `18-RISK-REGISTER.md` |
| 独立审查与证据 | `19-ADVERSARIAL-REVIEW.md`（59KB，22 反例）/ `20-EVIDENCE-INDEX.md` / `21-REPOSITORY-BASELINES.md` / `22-DOCUMENT-CODE-DIFF.md` / `23-OPEN-QUESTIONS.md` / `24-RECOMMENDATION-AND-READINESS.md` |
| 主代理核验与自纠 | `evidence/LEAD-CROSS-VERIFICATION-LOG{,-2,-3,-4,-5}.md` / `evidence/LEAD-SELF-CORRECTIONS.md` |
| 结构化数据 | `schema-diff.json`（13 blocking）/ `test-inventory.json` / `risk-register.json` / `experiment-matrix.json` / `repository-baselines.json` / `perf-results-annotated.json` |
| 补充图表 | `diagrams/SYSTEM-DIAGRAMS.md`（6 图） |

---

## 十、无法读取的材料

| 项 | 原因 |
|---|---|
| DSH 宿主代码 | `app.asar` 打包不可读 |
| CPU/RAM 配置 | `Get-CimInstance` PermissionDenied |
| 真实 index.db | 本机无副本 |
| Memory Core 实现 | **不存在** |
| `R4.1-memory-bank-input` | 路径不存在，已改用 `Memory-Core-R4.1-Design` |

---

## 十一、一句话

**CMB v0.1 的价值不在于它做了什么，而在于它把"接入前必须解决的问题"从模糊变成了 58 条具体条目——其中 6 条是致命的，且每一条都有实测或源码证据。这份材料包的目的，是让下一步不需要再猜。**
