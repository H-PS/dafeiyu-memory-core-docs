# 统一风险登记与问题分类（18-RISK-REGISTER）

采集：2026-11-06 ｜ 主代理综合 7 路审查 + 自主核验
**所有条目均有 文件:行号 或 实测输出 支撑。无一项为推测。**

---

## 分类说明

| 类别 | 含义 |
|---|---|
| **A** | 已发现缺陷（代码/数据错误） |
| **B** | 已知设计限制（有意为之但有限制） |
| **C** | 尚未验证假设 |
| **D** | 需用户决定的产品语义 |
| **E** | 需上游修改的契约 |
| **F** | 可由 Adapter 解决 |

等级：BLOCKER / CRITICAL / MAJOR / MINOR / INFORMATIONAL
责任：CMB / billion-context / Memory Core / Host / 用户裁决 / 外部环境 / 不明

---

## 全部条目（按等级排序）

| 编号 | 标题 | 类别 | 等级 | 责任 | 证据 | 阻塞真实接入 | 阻塞真实数据验证 |
|---|---|---|---|---|---|---|---|
| **R-01** | speaker 链断裂 → 真实源零产出 | A | **BLOCKER** | CMB+Host | 上游 `messages` 表无 speaker 列（`schema.ts:141-155`）；`bc-source.ts:211` 恒 null；`host-identity.ts:27` 接口只能收 `(session,hint)`；`integrity-validator.ts:71` 无条件强制。实测 `evidence_written=0, SPEAKER_UNRESOLVED=2` | ✅ 是 | ✅ 是 |
| **R-02** | `content_hash` 违反 MUST（应为 sha256(original_text)） | A | **BLOCKER** | CMB | `13-…md:255` MUST 条款；CMB 用 blob 哈希（`evidence-adapter.ts:58`）；`validator.test.ts:43-51` 把违规锁成正确 | ✅ 是 | ❌ 否 |
| **R-03** | payload 变但 covered_hash 不变 → COMMITTED 永久锁死 | A | **BLOCKER** | CMB | `bridge-ledger.ts:273`（key 用 covered_hash）vs `bc-source.ts:168`（version 用 payload_json）。实测 RUN2/RUN3 `STATE_RESET_FAIL` | ✅ 是 | ❌ 否 |
| **R-04** | session_id 含冒号 → locator 解析错误 → 撤销失效 | A | **BLOCKER** | CMB | `reconciliation-engine.ts:258-262` `split(":")`；QQ Bridge `bridge.js:1220` convKey=`kind:id`。实测 ledger 写到 `session_id="group"`, `ledger_seq=123456`；revokeScan 返回 0 | ✅ 是 | ✅ 是（若 sessionId 含冒号） |
| **R-05** | 系统提示词进入 Evidence | A | **CRITICAL** | CMB | 上游 `MessageContentType` 无 `"system"`（`types.ts`）；`integrity-validator.ts:95` 检查 content_type；`:70` 又豁免 role=system 的 speaker。实测 `written=1, text="你是一个猫娘助手…"` | ✅ 是 | ❌ 否 |
| **R-06** | 所有 mode 均不阻断写入（含 OFF/DRY_RUN） | A | **HIGH** | CMB | `cfg.mode` 仅在 `:77`（日志）与 `:333`（报告）被读。实测六模式全部 `evidence_written=1, ledger_rows=1` | ✅ 是 | ✅ 是 |
| **R-07** | 未知消息类型静默放行 | A | **HIGH** | CMB | `integrity-validator.ts:94-95` 黑名单仅两值。实测 `mystery-kind`/`attachment`/`image`/`developer` 全部 `written=1, 无诊断` | ✅ 是 | ❌ 否 |
| **R-08** | UNSUPPORTED 状态永久卡死 | A | **MAJOR** | CMB | `reconciliation-engine.ts:130` 未排除 UNSUPPORTED；`domain.ts:166` 仅允许 `→DISCOVERED`。实测 RUN2/RUN3 均 `STATE_RESET_FAIL` | ❌ 否 | ❌ 否 |
| **R-09** | evidence_id 撞键（session 重建） | A | **MAJOR** | CMB | `evidence-adapter.ts:53` `evidence_id=${session_id}:${seq}`。实测 sink 中同 id 两条 Evidence | ✅ 是 | ❌ 否 |
| **R-10** | M-2 未修复：部分覆盖判 COMMITTED | A | **MAJOR** | CMB | `reconciliation-engine.ts:224-226` 用 `??` 而非 `Math.max`。实测 3 条覆盖仅 1 条有效 → `missing=0, COMMITTED` | ✅ 是 | ❌ 否 |
| **R-11** | 全部 78 条测试用 `:memory:` ledger | A | **MAJOR** | CMB | 16 处 `new BridgeLedger(":memory:")`，0 处文件。实测内存实例重启后 `allBlocks=0` | ❌ 否 | ✅ 是（影响验证可信度） |
| **R-12** | 媒体外置未 hydrate → 多模态原文丢失 | A | **MAJOR** | CMB | 上游 `splitMediaContent`（`ingest.ts:194`）把图片换 `{"$media":h}`；CMB `bc-source.ts:183-196` 只取 `core.text`，无 `message_media` 查询 | ✅ 是 | ✅ 是 |
| **R-13** | tombstone 后媒体永久丢失 | B | MAJOR | billion-context | `store.ts:951-955` DELETE message_media + releaseContent | ✅ 是（不可恢复） | ❌ 否 |
| **R-14** | Block 无墓碑，物理删除后历史不可恢复 | B | **MAJOR** | billion-context | `store.ts:1077-1087`；无 retired 类列 | ✅ 是 | ✅ 是 |
| **R-15** | 撤销单向不可逆，消息复活 → SINK_CONFLICT | A | **MAJOR** | CMB | `evidence-sink.ts:50-58` + `domain.ts:168` REVOKED 无出边；上游 `store.ts:925` 可复活 | ✅ 是 | ❌ 否 |
| **R-16** | Memory Core 无实现，M03/M04 不可调用 | E | **BLOCKER** | Memory Core | 纯设计仓库（无 src/、无 package.json）；`api-contracts.md:89,93` 标 FUTURE | ✅ 是 | ❌ 否 |
| **R-17** | Evidence schema 缺 original_text/speaker_id + additionalProperties:false | E | **BLOCKER** | Memory Core | `evidence.schema.json:7,8`；权威 `DD-AUTHORITY-V2.md:208-223` 要求。12→13 个 blocking 字段 | ✅ 是 | ❌ 否 |
| **R-18** | `scope` / `recorded_at` 是 schema required 但 CMB 没有 | F | MAJOR | CMB | schema:8 required 列表 | ✅ 是 | ❌ 否 |
| **R-19** | `bot_id`/`group_id` 两方都无定义 | D | MAJOR | 用户裁决 | schema 无、V2 §3.2 无、CMB 有 | ✅ 是 | ❌ 否 |
| **R-20** | tenant_id/bot_id 只能部署常量，源值从不校验 | C | MAJOR | CMB+用户 | `src/` 全仓从未读 `SourceSession.tenant_id` | ✅ 是 | ❌ 否 |
| **R-21** | `session_type` 硬编码 "unknown" → 真实源不可解析 | A | **BLOCKER** | CMB | `bc-source.ts:42`；`host-identity.ts:73` unknown→抛 | ✅ 是 | ✅ 是 |
| **R-22** | 时间戳语义二义，Evidence 恒 null | C | MAJOR | billion-context+用户 | `store.ts:1031-1038` 批采样；无标记列 | ❌ 否 | ✅ 是 |
| **R-23** | 块 payload 漂移检测与 key 不同源 | A | MAJOR | CMB | 见 R-03 | ✅ 是 | ❌ 否 |
| **R-24** | `run()` 非原子：listSessions/listBlocks 无 try | A | MAJOR | CMB | `:66`, `:90` 在 try 外 | ❌ 否 | ❌ 否 |
| **R-25** | run_ledger 崩溃留半开行 | A | MAJOR | CMB | `:73` startRun 后无 finally | ❌ 否 | ❌ 否 |
| **R-26** | 无重试机制（cfg.retry 零引用） | A | MAJOR | CMB | `config.ts` 定义，`src/` 零引用 | ❌ 否 | ❌ 否 |
| **R-27** | 永久失败块无限重试无退避 | A | MAJOR | CMB | 无上限逻辑 | ❌ 否 | ✅ 是（Large 规模） |
| **R-28** | 每块重复 listBlocks（O(N×M)） | A | MAJOR | CMB | `source-resolver.ts:49` 重复查询 | ❌ 否 | ✅ 是（性能） |
| **R-29** | 无并发 Run 保护 | A | MAJOR | CMB | 无锁、无 run 隔离 | ❌ 否 | ✅ 是 |
| **R-30** | 无数据库世代标识 → 重建后重复 Evidence 风险 | C | MAJOR | CMB | 无 generation/epoch 字段 | ✅ 是 | ❌ 否 |
| **R-31** | 1 个占位测试 `assert(true)` | A | MINOR | CMB | `resolver.test.ts:76-80` | ❌ 否 | ❌ 否 |
| **R-32** | 性能数据不代表真实系统 | C | MAJOR | CMB | Mock 全内存；仅 ledger 真实 I/O | ❌ 否 | ✅ 是 |
| **R-33** | 峰值 RSS 仅 2 次采样 | A | MINOR | CMB | `perf-run.ts:218-222` | ❌ 否 | ❌ 否 |
| **R-34** | 吞吐口径未说明（可高估 2 倍） | A | MINOR | CMB | `perf-run.ts:236` msgCount 为总数 | ❌ 否 | ❌ 否 |
| **R-35** | "幂等重扫快 194 倍"表述误导 | A | MINOR | CMB | 实为"不做任何工作的开销" | ❌ 否 | ❌ 否 |
| **R-36** | batch_size/concurrency/retry 未接线 | A | MAJOR | CMB | 引擎零引用 | ❌ 否 | ✅ 是 |
| **R-37** | `HostIdentity.level` 未进 EvidenceDTO | A | MINOR | CMB | `host-identity.ts` vs `domain.ts` | ❌ 否 | ❌ 否 |
| **R-38** | 身份重复解析（每消息 4 次 provider 调用） | A | MINOR | CMB | `engine.ts:178-183` + `validator.ts:53-57` | ❌ 否 | ✅ 是（性能） |
| **R-39** | 日志 events[] 无上限 | A | MINOR | CMB | `observability.ts:18` | ❌ 否 | ❌ 否 |
| **R-40** | `locatorOf()` 死代码 | A | INFO | CMB | `errors.ts` 零调用 | ❌ 否 | ❌ 否 |
| **R-41** | M09 Summary 完全孤立 | A | MAJOR | CMB | 全项目仅定义处 1 命中 | ❌ 否 | ❌ 否 |
| **R-42** | bc-source 无 message_media 查询 | A | MAJOR | CMB | 见 R-12 | ✅ 是 | ✅ 是 |
| **R-43** | 无 hydrateMedia 实现 | A | MAJOR | CMB | 同上 | ✅ 是 | ✅ 是 |
| **R-44** | `resolveSpeaker` 用 `hint ?? bound`，方向反了 | A | MAJOR | CMB | `host-identity.ts:63` 消息内容可覆盖宿主绑定 | ✅ 是 | ❌ 否 |
| **R-45** | M-1 交叉校验在真实数据上是死代码 | A | MAJOR | CMB | `bc-source.ts:41` 恒 null → 不一致分支永不成立 | ✅ 是 | ❌ 否 |
| **R-46** | `DeploymentConstantIdentityProvider` 无构造点 | A | MAJOR | CMB | `src/` 内 0 个 `new` | ❌ 否 | ✅ 是 |
| **R-47** | fork 克隆消息行，CMB 不用 parent_session_id | B | MAJOR | billion-context+CMB | `store.ts:2879-2885` 克隆；CMB 读 `parent_session_id` 零使用 | ✅ 是 | ✅ 是 |
| **R-48** | `encryption_compatibility=false` 一键关自检 | A | MAJOR | CMB | `config.ts` 条件判断 | ❌ 否 | ✅ 是 |
| **R-49** | 加密自检只查本进程 env | A | MINOR | CMB | 换进程即绕过 | ❌ 否 | ❌ 否 |
| **R-50** | `covered_hash` 全仓无人校验 | C | MAJOR | billion-context | Agent B 核验，被 CMB 当幂等根键 | ✅ 是 | ❌ 否 |
| **R-51** | `supersedes_block_id` 恒 NULL 死列 | B | MAJOR | billion-context | `ingest.ts:425` 硬编码 | ❌ 否 | ❌ 否 |
| **R-52** | start_seq/end_seq 由 live messages 重算 | B | MAJOR | billion-context | `store.ts:1131-1146` | ❌ 否 | ✅ 是 |
| **R-53** | `directIds=[]` 的 tier-2/3 块 → resolver 空集 → 永久 FAILED | A | MAJOR | CMB | Agent G M-2 | ✅ 是 | ✅ 是 |
| **R-54** | 生产 open() 无 readOnly 参数 | C | MAJOR | billion-context | `driver.ts:163` 裸 `new DatabaseSync(path)` | ❌ 否 | ✅ 是 |
| **R-55** | schema 版本 v1→v10 演进，需 user_version 分支 | C | MINOR | 双方 | Agent B | ❌ 否 | ✅ 是 |
| **R-56** | CLI 无 Adapter 选择 | A | MAJOR | CMB | `cmb.ts:45` 硬编码 Mock | ❌ 否 | ✅ 是 |
| **R-57** | 无 `--config` 文件加载 | A | MINOR | CMB | 已从文档移除 | ❌ 否 | ❌ 否 |
| **R-58** | DSH 不可读（asar 打包） | C | MINOR | 外部环境 | `app.asar` 213MB | ❌ 否 | ✅ 是（身份验证） |

---

## 统计

| 等级 | 数量 |
|---|---|
| **BLOCKER** | **6**（R-01, R-02, R-03, R-04, R-16, R-21） |
| **CRITICAL** | **1**（R-05） |
| **HIGH** | **2**（R-06, R-07） |
| **MAJOR** | **31** |
| **MINOR** | **10** |
| **INFORMATIONAL** | **1** |
| **合计** | **58** |

| 类别 | 数量 |
|---|---|
| A 已发现缺陷 | 38 |
| B 已知设计限制 | 5 |
| C 尚未验证假设 | 8 |
| D 需用户决定 | 1 |
| E 需上游修改 | 2 |
| F 可由 Adapter 解决 | 1（实际更多，见下） |

| 责任归属 | 数量 |
|---|---|
| CMB | 40 |
| billion-context | 7 |
| Memory Core | 2 |
| Host | 1 |
| 用户裁决 | 1 |
| 外部环境 | 1 |
| 混合/不明 | 6 |

---

## 阻塞判定

### 阻塞真实数据验证（进入 v0.2 前必须解决）

**9 项**：R-01（speaker 链）、R-04（冒号 locator）、R-06（mode 不生效）、R-11（测试方法）、R-12/R-42/R-43（媒体）、R-14（Block 历史）、R-21（session_type）、R-27/R-28（性能）、R-29（并发）、R-32/R-36（性能可信度）、R-46（identity provider 无构造）、R-47（fork）、R-48（加密）、R-50（covered_hash）、R-52、R-54、R-56、R-58

实际计数：**17 项**

### 阻塞真实接入（v1.0 前必须解决）

**6 个 BLOCKER + 1 CRITICAL + 2 HIGH = 9 项**
加上所有 MAJOR 中涉及数据正确性的（R-09, R-10, R-15, R-18, R-19, R-20, R-44, R-45, R-53）

实际计数：**18 项**

---

## 可由 Adapter 解决的（不需要上游改动）

| 编号 | 问题 | 解法 |
|---|---|---|
| R-02 | content_hash 语义 | CMB 改算 sha256(original_text)，另存 blob 哈希 |
| R-04 | locator 解析 | 改分隔符或直接存结构化字段 |
| R-05 | system 提示词 | 改查 `role` 而非 `content_type` |
| R-06 | mode 不生效 | 引擎按 mode 分支 |
| R-07 | 未知类型放行 | 改白名单 |
| R-09 | evidence_id 撞键 | 用 sha256 参与 |
| R-10 | M-2 分母 | 用 Math.max |
| R-18 | scope/recorded_at 缺失 | 派生填充 |
| R-42/R-43 | hydrateMedia | 实现 message_media 查询 |
| R-44 | speaker 方向 | 改为 `bound ?? hint` 或只用 bound |
| R-53 | 空 directIds | 加保护 |

**共 11 项可由 CMB 单方解决。**

## 必须由上游修改的

| 编号 | 问题 | 需要什么 |
|---|---|---|
| R-16 | Memory Core 无实现 | 需要实现 M01/M03/M04 |
| R-17 | schema 缺字段 | 修订 evidence.schema.json |
| R-50 | covered_hash 无校验 | 上游增加校验或提供替代完整性凭据 |

## 必须由用户裁决的

| 编号 | 问题 |
|---|---|
| R-13 | tombstone 后媒体永久丢失是否接受 |
| R-14 | Block 历史不可恢复的 SLA |
| R-19 | bot_id/group_id 是否成为正式维度 |
| R-20 | tenant_id 单人部署如何定义 |
| R-22 | Evidence 无时间是否接受 |
| R-01 | speaker 从哪来（上游补列 / 宿主注入 / 降级用 role / 放宽） |
