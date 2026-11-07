# 证据索引（20-EVIDENCE-INDEX）

采集：2026-11-06 ｜ 主代理汇总全部关键结论的证据位置
**等级说明**：VERIFIED_SOURCE（源码直接确认）/ VERIFIED_RUNTIME（实测）/ VERIFIED_TEST（测试验证）/ MOCK_ONLY / STATIC_INFERENCE / DOCUMENT_ONLY / UNVERIFIED

---

## 1. CMB 本体

| 结论 ID | 结论 | 等级 | 仓库 | 文件:行号 | 函数/类 | 对应测试 |
|---|---|---|---|---|---|---|
| C-001 | 16 个源文件，2,014 行 | VERIFIED_SOURCE | CMB | `src/**/*.ts` | — | — |
| C-002 | 零第三方依赖 | VERIFIED_SOURCE | CMB | `package.json` | — | — |
| C-003 | CLI 硬编码装配 MockSourceStore | VERIFIED_SOURCE | CMB | `src/cli/cmb.ts:45` | `main()` | — |
| C-004 | BillionContextSourceAdapter 无生产调用者 | VERIFIED_SOURCE | CMB | 全仓 grep | — | — |
| C-005 | SummaryArtifactStore 零调用者 | VERIFIED_SOURCE | CMB | 全仓 grep（仅定义处） | — | — |
| C-006 | MemoryCoreEvidenceAdapter 全方法抛 SINK_UNSUPPORTED | VERIFIED_SOURCE | CMB | `src/evidence/evidence-sink.ts:85-105` | `MemoryCoreEvidenceAdapter` | `evidence.test.ts` |
| C-007 | 无任何上游写路径 | VERIFIED_SOURCE | CMB | 全仓 grep `writeFile\|INSERT\|UPDATE` | — | — |
| C-008 | bc-source 使用 readOnly 连接 | VERIFIED_SOURCE | CMB | `src/adapters/bc-source.ts:30` | 构造 | — |

---

## 2. 六个 BLOCKER

| 结论 ID | 结论 | 等级 | 文件:行号 | 实测输出 |
|---|---|---|---|---|
| **D-01** | speaker 链断裂 → 真实源零产出 | VERIFIED_RUNTIME | 上游 `schema.ts:141-155`（无 speaker 列）；`bc-source.ts:211`（恒 null）；`host-identity.ts:27`（接口无法按消息映射）；`integrity-validator.ts:71`（无条件强制） | `evidence_written=0, missing=1, SPEAKER_UNRESOLVED=2` |
| **D-02** | content_hash 违反 MUST | VERIFIED_SOURCE | `13-Evidence与EmotionContext.md:255`（MUST）；`evidence-adapter.ts:58`；`validator.test.ts:43-51` | — |
| **D-03** | payload 变但 key 不变 → COMMITTED 锁死 | VERIFIED_RUNTIME | `bridge-ledger.ts:273` vs `bc-source.ts:168` | RUN1 COMMITTED → RUN2/RUN3 `STATE_RESET_FAIL`, `failed=1` |
| **D-04** | session_id 含冒号 → 撤销失效 | VERIFIED_RUNTIME | `reconciliation-engine.ts:258-262`；QQ Bridge `bridge.js:1220` | ledger 写入 `session_id="group"`, `ledger_seq=123456`；`revokeScan` 返回 0 |
| **D-16** | Memory Core 无实现 | VERIFIED_SOURCE | `Memory-Core-R4.1-Design` 无 `src/`；`api-contracts.md:89,93` 标 FUTURE | — |
| **D-17** | Evidence schema 缺 original_text/speaker_id | VERIFIED_SOURCE | `evidence.schema.json:7,8`；`DD-AUTHORITY-V2.md:208-223` | — |
| **D-21** | session_type 硬编码 "unknown" | VERIFIED_SOURCE | `bc-source.ts:42`；`host-identity.ts:73` | — |

---

## 3. CRITICAL / HIGH

| 结论 ID | 结论 | 等级 | 文件:行号 | 实测 |
|---|---|---|---|---|
| **D-05** | 系统提示词进 Evidence | VERIFIED_RUNTIME | 上游 `types.ts`（无 "system" contentType）；`integrity-validator.ts:95`；`:70` | `written=1, text="你是一个猫娘助手…", speaker=null` |
| **D-06** | 所有 mode 都写入（含 OFF/DRY_RUN） | VERIFIED_RUNTIME | `reconciliation-engine.ts:77`（日志）、`:333`（报告） | 6 模式全部 `evidence_written=1, ledger_rows=1` |
| **D-07** | 未知类型静默放行 | VERIFIED_RUNTIME | `integrity-validator.ts:94-95` | `mystery-kind/attachment/image/developer` 全部 `written=1 无诊断` |

---

## 4. MAJOR 缺陷

| 结论 ID | 结论 | 等级 | 文件:行号 | 实测 |
|---|---|---|---|---|
| D-08 | UNSUPPORTED 永久卡死 | VERIFIED_RUNTIME | `reconciliation-engine.ts:130`；`domain.ts:166` | RUN2/RUN3 `STATE_RESET_FAIL` |
| D-09 | evidence_id 撞键 | VERIFIED_RUNTIME | `evidence-adapter.ts:53` | sink 中同 id 两条 |
| D-10 | M-2 未修：部分覆盖判 COMMITTED | VERIFIED_RUNTIME | `reconciliation-engine.ts:224-226` | 3 覆盖 1 有效 → `missing=0, COMMITTED` |
| D-11 | 全部 78 测试用 :memory: | VERIFIED_RUNTIME | 16 处 `new BridgeLedger(":memory:")` | 内存实例重启后 `allBlocks=0` |
| D-12 | 媒体未 hydrate → 多模态原文丢失 | VERIFIED_SOURCE | `ingest.ts:194`；`bc-source.ts:183-196` | — |
| D-13 | tombstone 后媒体永久丢失 | VERIFIED_SOURCE | `store.ts:951-955` | — |
| D-14 | Block 无墓碑，物理删除 | VERIFIED_SOURCE | `store.ts:1077-1087` | — |
| D-15 | 撤销不可逆，复活 → SINK_CONFLICT | VERIFIED_SOURCE | `evidence-sink.ts:50-58`；`domain.ts:168`；`store.ts:925` | — |
| D-44 | resolveSpeaker 方向反（hint ?? bound） | VERIFIED_SOURCE | `host-identity.ts:63` | — |
| D-45 | M-1 交叉校验是死代码 | VERIFIED_SOURCE | `bc-source.ts:41`（恒 null → 分支不触发） | — |
| D-53 | 空 directIds → 永久 FAILED | STATIC_INFERENCE | Agent G M-2 | — |
| D-59 | 隐含 user_version >= 10 门槛 | VERIFIED_SOURCE | `bc-source.ts:77`（选 `m.ts`）；`schema.ts:391-398`（v10 才加）；`:71`（VERSION=10） | — |
| D-60 | "加密的 index.db" 分支不存在 | VERIFIED_SOURCE | `persist.ts:562-569`（加密时不建库） | — |
| D-61 | CAS algo/byte_len 从未校验 | VERIFIED_SOURCE | Agent F | — |

---

## 5. 上游事实

| 结论 ID | 结论 | 等级 | 文件:行号 |
|---|---|---|---|
| U-01 | 正文在 `content.bytes`，messages 用外键 | VERIFIED_SOURCE | `schema.ts:133,149` |
| U-02 | blob 是 `splitMediaContent` 后的投影，**非原始字节** | VERIFIED_SOURCE | `ingest.ts:194-195` |
| U-03 | `payload_json` 是逐字 CompressionBlock | VERIFIED_SOURCE | `ingest.ts:429` |
| U-04 | `covered_hash` 由 id 数组哈希 | VERIFIED_SOURCE | `ingest.ts:406,428` |
| U-05 | `covered_hash` 全仓无人校验 | VERIFIED_SOURCE | Agent B `05:470` |
| U-06 | `supersedes_block_id` 恒 NULL 死列 | VERIFIED_SOURCE | `ingest.ts:425` |
| U-07 | start/end_seq 由 store 层从 live messages 重算 | VERIFIED_SOURCE | `store.ts:1131-1146` |
| U-08 | `identity` 剥 id/wireIndex/ledgerSeq | VERIFIED_SOURCE | `identity.ts:86-98` |
| U-09 | `retired_at` 可复活 | VERIFIED_SOURCE | `store.ts:925,949` |
| U-10 | 退役时媒体立即释放 | VERIFIED_SOURCE | `store.ts:951-955` |
| U-11 | `ts` 无首见戳时填批采样时刻 | VERIFIED_SOURCE | `store.ts:1031-1038` |
| U-12 | 加密时 unified sidecar 不打开 | VERIFIED_SOURCE | `persist.ts:562-569` |
| U-13 | 物理删除 blocks（不在当轮 state） | VERIFIED_SOURCE | `store.ts:1077-1087` |
| U-14 | MessageContentType 无 "system" | VERIFIED_SOURCE | `kernel/src/types.ts` |
| U-15 | schema 版本 10，v10 加 ts 列 | VERIFIED_SOURCE | `schema.ts:71,391-398` |
| U-16 | 生产 open() 无 readOnly 参数 | VERIFIED_SOURCE | `driver.ts:163`（Agent B） |

---

## 6. Memory Core 事实

| 结论 ID | 结论 | 等级 | 文件:行号 |
|---|---|---|---|
| M-01 | evidence.schema.json 36 行，additionalProperties:false | VERIFIED_SOURCE | `contracts/schemas/evidence.schema.json:7` |
| M-02 | required 含 scope/recorded_at，缺 original_text/speaker_id | VERIFIED_SOURCE | 同上 `:8` |
| M-03 | source_type 12 值枚举（schema） | VERIFIED_SOURCE | `:13` |
| M-04 | 权威文档 §3.2 source_type 5 值（chat/file/import/tool/system） | VERIFIED_SOURCE | `DD-AUTHORITY-V2.md:212` |
| M-05 | content_hash MUST 为 original_text sha256 | VERIFIED_SOURCE | `13-…md:255`；`V2:221` |
| M-06 | 无实现代码（纯设计仓库） | VERIFIED_SOURCE | 目录结构 |
| M-07 | M03/M04 标 FUTURE | VERIFIED_SOURCE | `api-contracts.md:89,93` |
| M-08 | 内部审查"假绿"：比对的是被降级的一侧 | VERIFIED_SOURCE | `reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md:39`（Agent C） |
| M-09 | schema 与备份逐字节一致（升级遗漏） | VERIFIED_SOURCE | Agent C 物证 |
| M-10 | 无迁移机制（无 version 字段、无 migrations 目录） | VERIFIED_SOURCE | 目录结构 |

---

## 7. 测试事实

| 结论 ID | 结论 | 等级 | 证据 |
|---|---|---|---|
| T-01 | 78 passed, 0 failed, exit=0 | VERIFIED_RUNTIME | 主代理实跑 |
| T-02 | 74 静态 test() + 4 循环实例 = 78 | VERIFIED_RUNTIME | `phase6:33` for 循环 |
| T-03 | 0 条用真实 SQLite 文件 | VERIFIED_SOURCE | 16 处 `:memory:` |
| T-04 | 1 个占位测试 `assert(true)` | VERIFIED_SOURCE | `resolver.test.ts:76-80` |
| T-05 | validator.test.ts 锁定错误的 content_hash 语义 | VERIFIED_SOURCE | `:43-51` |
| T-06 | SEC-7 用上游不存在的 contentType 构造 | VERIFIED_SOURCE | `security.test.ts` |
| T-07 | 6 个已确认缺陷全部逃过测试 | VERIFIED_RUNTIME | D-01/03/04/08/09/10 |

---

## 8. 性能事实

| 结论 ID | 结论 | 等级 | 证据 |
|---|---|---|---|
| P-01 | Small 1,000 消息 1,180ms 847msg/s | VERIFIED_RUNTIME | 主代理实跑 |
| P-02 | Medium 100,000 消息 128,258ms 780msg/s | VERIFIED_RUNTIME | 报告（未复跑） |
| P-03 | Large 1,000,000 消息 2,677,395ms 373msg/s | VERIFIED_RUNTIME | 报告（未复跑） |
| P-04 | 吞吐口径为总消息数，可高估 2 倍 | VERIFIED_SOURCE | `perf-run.ts:236` |
| P-05 | 峰值 RSS 仅 2 次采样 | VERIFIED_SOURCE | `perf-run.ts:218-222` |
| P-06 | Mock 全内存，仅 ledger 真实 I/O | VERIFIED_SOURCE | `mock-source.ts` |
| P-07 | batch_size/concurrency/retry 未接线 | VERIFIED_SOURCE | 引擎零引用 |
| P-08 | CPU/RAM 无法采集 | VERIFIED_RUNTIME | `Get-CimInstance` PermissionDenied |

---

## 9. 文档差异

| 结论 ID | 结论 | 等级 | 证据 |
|---|---|---|---|
| DOC-01 | README 声称 M01-M12 全模块 | VERIFIED_SOURCE | 实际 M09 零调用 |
| DOC-02 | ARCHITECTURE 声称"三实两缺" | VERIFIED_SOURCE | speaker 实为"零实" |
| DOC-03 | TEST-REPORT 声称崩溃可恢复 | VERIFIED_SOURCE | 全部 `:memory:` |
| DOC-04 | RISK-REGISTER 缺 6 个 BLOCKER | VERIFIED_SOURCE | 对比本轮 18 号 |

---

## 10. 文件哈希索引

见 `repository-baselines.json` 的 `key_file_hashes_sha256_prefix16`。

CMB 16 个源文件哈希、billion-context 7 个关键文件哈希、Memory Core 5 个关键文件哈希均已记录。
