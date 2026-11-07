# 04 · 历史项目里的可用方案 → 新方向（A4 交付件）

> Archive-Review-2026 · 子 Agent A4 交付件
> 目标：从历史阶段挖出真正有价值、可复用的方案，作为 R4.1 后续演进方向（不是复述历史）。
> 判定纪律：不得把历史方案写成「已批准」；每条标 NEW PROPOSAL 或「可直接继承（有裁决依据）」并给出裁决出处；与 R4.1 冲突的明确写冲突点。

STATUS: COMPLETE

---

## 目录

- §1 结论（先行摘要）
- §2 十个方向卡（D1–D10：出处 / 核心机制 / 可复用点 / 与 R4.1 接口 / 风险 / 迁移成本 / 建议优先级）
- §3 与 R4.1 的冲突表
- §4 建议吸收顺序
- §5 不建议吸收的历史方案（及理由）
- §6 方法附录（检索命令、取证纪律、未验证项与验证命令）

---

## §1 结论（先行摘要）

本轮按目录定向读取了 R2 冻结基线（08/19）、V2.4.2 设计收敛（00–09）、work 目录 M 模块与裁决台账、R3 心理/人格（psy-model、PIPE-PERSONA）、R3 情绪与框架优化、B 独立审查（B-00/B-01/B-02/B-07）等历史资产。所有引用均给相对路径 + 行号（用 `grep`/`read` 定位，等价于 `Select-String`）。判定纪律贯穿全文：**历史方案一律不写成「已批准」**，每条标 `NEW PROPOSAL` 或「可直接继承（有裁决依据）」并给裁决出处；与 R4.1 冲突的写入 §3。

三条全局结论：

1. **【EXISTING / P0】R4.1 的正式机器合同集不在时间流内。** 背景事实 6/7 已实测：时间流内没有 `DD-AUTHORITY*.md`、没有 `01-记忆本体与数据模型.md`，唯一带机器合同的 121 文件 canonical 集（`C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design`）与 work 最新 mtime（2026-11-04 20:45）均不在时间流内；时间流最新仅 2026-11-04 14:48（索引本身）。因此本文所有「D1–D10 补 R4.1 的某某缺口」中的「R4.1」指的是**work 目录 canonical 基线**，而非时间流内的 V2.4.2/M 模块。任何「时间流已含完整 R4.1」的说法都会被此推翻。
2. **【EXISTING / P1】V2.4.2 是给 bridge.js 单体 JSONL 记忆做的执行模型，不是 R4.1 五模块/SQLite 架构。** V2.4.2 自述「第一版不引入新存储引擎：保持 JSONL + 文件锁」（`00-V2.4.2-DESIGN-SUMMARY.md:36`）、目标架构是 `src/memory-service.js` 单模块（`:45`）；而 R4.1/M 模块是 M01–M05 五模块 + SQLite 唯一权威（`03-m01.md:26` D1 双存储架构、`03-m05.md:18` M05 唯一权限权威）。两者存储引擎、写路径、模块边界都不同——V2.4.2 只能作为**机制/不变量**继承，不能当代码基线移植（详见 §3 冲突表 R1）。
3. **【COMPLETENESS / P1】裁决出处可复现，但「09-FUT-上下文压缩」类文档在指定目录不存在。** 裁决台账 `08-adjudication-ledger.md` 实测存在且 D-AP-05/K-1 等条目逐字可查（`:78–86`、`:21`）；B 独立审查 B-010/B-011/D-AP-05/K-1 的结论与行号可复现（`B-00-MATERIAL-AND-BASELINE-AUDIT.md:267–268`、`:173`、`:183`）。但任务点名的「09-FUT-上下文压缩 类文档」在 `2026-11-01_16_R4.1续作与终版交付` 下未找到：实测该目录的 FUT 档案是 `13-FUTURE-ROADMAP.md`（F-01~F-24）+ `A-04-V1-FUTURE-CLASSIFICATION.md`/`A7-V1-FUTURE-BOUNDARY.md`，压缩相关内容只有「报告精简规范 `limit v2.md`」与人格管线「抗压缩」（`03-报告5-人格分层与记忆库重建.md:99–100`），**没有独立的上下文压缩方案文档**。→ 见 §6 未验证项 V-1。

**十个方向的判定总览**（详见 §2）：D1/D2/D3/D5 为 `NEW PROPOSAL`（机制可复用、但与 R4.1 存储引擎冲突需重做接口）；D4 为「可直接继承（有裁决依据）」+ `NEW PROPOSAL` 混合（B-010/B-011 是已确认缺陷，补法是新提案）；D6/D7/D8 为 `NEW PROPOSAL`（Persona/心理/情绪整体未进入 R4.1 canonical，且部分与裁决台账 X-1/I-4/K-1 门禁冲突）；D9 逐条判『仍适用/已被吸收/冲突』；D10 为「可直接继承（有裁决依据）」（A1–A15 是 work 目录 `06-最小记忆闭环验收标准.md:30–44` 的现存验收集，不属历史方案，作为 V0.9 验收集是 NEW PROPOSAL 的用法建议）。

> 三分类口径：`EXISTING` = 实测/原文可复现；`COMPLETENESS` = 证据是否足以支撑结论；`RECOMMENDED`（本文一律标 `NEW PROPOSAL`）= 建议动作。优先级：P0（结论会被推翻/证据不可复现）、P1（严重失真或误导）、P2（可整理）。

---

## §2 十个方向卡

> 每张卡字段：出处（相对路径:行号）｜核心机制｜可复用点｜与 R4.1 接口｜风险｜迁移成本｜建议优先级｜判定标签。
> 路径缩写：`V242/` = `2026-10-31_04_V2.4.2设计收敛/2.42/`；`R2/` = `2026-10-31_03_R2架构基线冻结/R2架构-最新-20261101/设计文档/`；`PIPE/` = `2026-08-28_17_E-gptwork(agent3产物与研究报告)/E_gptwork/1739-extract/R3-MemoryCore-全套产物-20261102-1723/04-人格管线方案/`；`PSY/` = `2026-10-30_02_F-moren心理模型分支/F_moren/psy-model/`；`B/` = `2026-11-03_13_B独立审查交付/B/`；`WORK/` = `C:\Users\[USER]\Desktop\work/`（不在时间流内）。

### D1 · 写入并发与 WAL 纪律 → 补 R4.1 事务边界

- **出处**：`V242/02-WRITE-CONCURRENCY.md:17–46`（统一管线 P0–P4）、`:59–67`（INo-1 单写者 / INo-2 无旧快照 / INo-3 单点生效 / INo-4 rev 单调 / INo-5 opId 幂等 / INo-6 永不越权修改）、`:68–83`（原子提交 `atomicWriteLines`：随机 tmp + fsync + renameWithRetry + 0600）、`V242/03-AB-CONSISTENCY-WAL.md:70–80`（WAL 先行 + 崩溃点矩阵）、`V242/09-ACCEPTANCE-CRITERIA.md:10–21`（AC-1.1~1.11 并发验收场景）。
- **核心机制**：进程内 FIFO 写队列串行化 7 种写操作；`expectedRev` 乐观锁 + `opId` 幂等；WAL `pending→committed/aborted/superseded/recovery_blocked`；「写前重读、绝不复用请求进入时的数组/索引」；提交顺序固定 `WAL→tombstones→store→B 投影→SR→WAL(committed)`。
- **可复用点**：**六条不变量（INo-1~6）与 WAL effects 语义是引擎无关的设计资产**；`expectedRev` 冲突返回 409+t当前 rev、tombstone 命中返回 410、审计阻断业务不发生，这三条可直接搬成 R4.1 的写契约错误码。
- **与 R4.1 接口**：R4.1 是 M01 SQLite 事务（`03-m01.md:31` D6 事务和一致性、`03-m01.md:19–55` 决策索引 D1–D40）。建议把「单写者」落到 SQLite 事务 + `BEGIN IMMEDIATE`，把 `expectedRev` 落到 `03-系统运行与协作协议.md` 的写前置校验，把 WAL 降级为 Outbox/审计事件（R4.1 已有 Outbox 概念，见 `13-FUTURE-ROADMAP.md:43`、框架优化 `:124`）。**不搬 V2.4.2 的 `MemoryService` JSONL 重写实现**。
- **风险**：`【P1】` 存储引擎冲突（见 §3 R1）；V2.4.2 自认「JSONL 全量重写在 10 万条耗时 NOT_VERIFIABLE」（`V242/00:155`），R4.1 用 SQLite 后该风险已消除，但 `fsync 在 Windows/网络盘` 仍未验证（`V242/00:158`）。
- **迁移成本**：中。6 条不变量 = 契约改写（低）；验收场景 AC-1.x 需重写为 SQLite 事务用例（中）；无代码可直搬。
- **建议优先级**：**P1**（事务边界缺失会让 A11「进程中断重启 0 差异」验收无法通过，见 D10）。
- **判定**：`NEW PROPOSAL`（V2.4.2 是纯只读设计收敛稿，`V242/00:3` 自述「未改任何代码」，非已批准基线）。

### D2 · 生命周期与 tombstone → 补删除传播与恢复

- **出处**：`V242/04-LIFECYCLE-TOMBSTONE.md:5–39`（六态状态机 active/expired/evicted/suppressed/pending_delete/已删 + 可检索/可编辑/可分享矩阵）、`:54–68`（tombstone 结构、永久保留、只存身份不存正文）、`:70–88`（删除事务：SR superseded、级联副本、Tier-1 审计、提交顺序）、`V242/00:98–113`（保留决策 4/5/11/12）、`V242/09-ACCEPTANCE-CRITERIA.md:47–56`（AC-4.1~4.7 防复活验收）。
- **核心机制**：删除 = 物理移出 store + 永久 tombstone（身份+contentHash+级联标记）；恢复默认跳过已删记录（`V242/00:32` 原则 5）；`ignoreTombstone` 白名单逐条 RESURRECT 审计（`V242/08-DECISIONS-REQUIRED.md:17` D-09）；内容屏蔽与身份删除分离（`content_suppression`）。
- **可复用点**：**「防复活」不变量 + 备份恢复跳过 tombstone + 级联删除默认开启**，这三条是 B 独立审查已确认的 R4.1 残留风险的解法：`B/B-01-M01-STORAGE-REVIEW.md:267` 实测「恢复旧备份重放 Outbox／乱序消费／重建期间，已 Deleted 记忆可能重新写入向量与 FTS 索引」，并推荐「须在 M01 冻结前写入 D23」（`:270`）。
- **与 R4.1 接口**：落到 M01 生命周期（`03-m01.md:32` D7 删除、归档与生命周期）与 `06-最小记忆闭环验收标准.md:40` A10（remove 后两层消失）、`:41` A12（从 backup 恢复后 A11 通过）；FTS5 真删语义 B-022 仍是未确认项（`B/B-01:283`），tombstone 只能保证逻辑不复活。
- **风险**：`【P1】` tombstone 与 R4.1 的「已删记忆物理清理」语义需对齐；`B-022` 指出 FTS5 删除靠屏蔽+重建、词条残留（`13-FUTURE-ROADMAP.md:46–48`），tombstone 无法解决物理残留。
- **迁移成本**：中低。状态机 + 墓碑表 = schema 增补（M01 D9 迁移版本登记），无引擎冲突。
- **建议优先级**：**P1**（直接对应 A10/A11/A12 与 B-01 已确认缺陷）。
- **判定**：`NEW PROPOSAL`；其中「防复活」子项与 B 独立审查结论互证，但 V2.4.2 本身仍非已批准基线。

### D3 · 共享与权限安全 → 补 visibility 与 scope 判定

- **出处**：`V242/05-SHARE-SECURITY.md:5–22`（ShareRequest 对象 + `(memoryId, targetKey)` 申请键实现群隔离）、`:24–44`（无条件三重鉴权、源 key 非 private 校验、必须用 memoryId）、`:46–73`（审批 9 步复查：原件存在/tombstone/rev/lifecycle/visibility/suppression/Tier-1 审计/生成副本带 origin）、`WORK/03-m05.md:18–27`（M05 唯一权限权威 + private/group/shared 三范围）、`:72`（身份缺失默认拒绝）、`:486–489`（授权/权限状态不明确默认拒绝）、`:646–647`（D25 Scope 定义、D26 群记忆归属）。
- **核心机制**：申请/审批/撤销三段式；「申请必须用 memoryId」；审批时逐项复查 rev 与 lifecycle；`origin:{id,rev}` 使副本可追溯；private 源拒绝共享到群。M05 侧：`(actor_id, target_id, action, scope)` 四元组（`03-m05.md:384`）+ 默认 DENY。
- **可复用点**：**「逐项复查」清单与「默认拒绝」不变量**；`visibility`/`scope` 的判定输入可补齐 B 独立审查指出的缺口：`B/B-05-M05-SECURITY-REVIEW.md:144` 实测「PSYCH memory.mjs 无任何权限字段……可见性保证条款在 M05 侧缺失」、`B/B-07:139` 把「M01 权限表缺失」列为 B-058 P0。
- **与 R4.1 接口**：M05 `permissionCheck()` 挂载点（`06-最小记忆闭环验收标准.md:42` A13 要求接口契约默认 DENY）；`03-系统运行与协作协议.md` 的分享审批流；canonical 包 `04-权限与治理规范.md`。
- **风险**：`【P0】` B-058 是 P0（权限表缺失），D3 只是设计蓝图，不能替代「在 M01 建权限表」这个已确认缺陷；跨群分享 V2.4.2 明确「第一版仅同群」（`V242/05:36–37`、`V242/08:23` D-10），与 M05 的 `shared` 范围需对齐。
- **迁移成本**：中。ShareRequest 三段式与 M05 四元组是两套模型，需统一为一套；权限表是新增 schema。
- **建议优先级**：**P0**（权限表缺失会推翻「合规」结论，且 A13/A14 依赖它）。
- **判定**：`NEW PROPOSAL`；M05 的「默认拒绝」是 R3 冻结候选稿内的既有设计（`03-m05.md:4` Design Frozen），可作继承依据，但 V2.4.2 分享流不是。

### D4 · Embedding 审计 / 备份 / 迁移 → 补空间定义与迁移不变量

- **出处**：`V242/06-EMBEDDING-AUDIT-BACKUP.md:5–52`（六态状态机 pending/processing/ready/retry_wait/failed/cleared + 十项设计要点）、`:41–52`（vecModel/vecDim 校验、模型切换批量转 pending、维度/数值校验）、`:66–74`（查询降级 degraded 标注）；`B/B-00-MATERIAL-AND-BASELINE-AUDIT.md:267–268`（B-010/B-011 原始定级）、`B/B-01-M01-STORAGE-REVIEW.md:13–14`（B-010/B-011 成立 P1）、`:59–86`（§3/§4 核查结论）、`:311–312`（冻结前必补：向量溯源字段 + 索引空间/窗口写入规则）、`B/B-02-M02-RETRIEVAL-REVIEW.md:23,276,898–906`（M02 侧确认 + `VECTOR.md` 已明确三条约束但 M01/M02 未写）、`B/B-07-FINAL-ALGORITHM-REVIEW.md:392–393`（B-010/B-011「未解决/是」）、`WORK/06-最小记忆闭环验收标准.md:33`（A4 来源可追溯）。
- **核心机制**：V2.4.2 侧 = `vecModel`/`vecDim` + 检索谓词 `record.vecModel === currentModel` + 失败降级 + 蓝绿重建；B 审查侧 = **B-010**（D19 只有 `embedding_model`/`embedding_version`，缺维度/度量/归一化/投影版本/分块/`chunk_id`；`embedding_version` 一名两义）与 **B-011**（缺查询侧模型路由、迁移窗口写入归属、切换原子性、追赶判据、「一空间一模型」不变量，「索引空间」未定义）。
- **可复用点**：**B-010/B-011 是「已确认缺陷」而非「待讨论方案」**（`B/B-01:256`「全部确认为已确认问题（P1）」），可直接继承为 R4.1 的验收条目；V2.4.2 的「正文持久化与向量可用性解耦」（`V242/00:34` 原则 7）是补法的核心不变量。
- **与 R4.1 接口**：M01 D16/D17/D19/D23（`03-m01.md:41–43,48`）+ A4（每条正式记忆可回答「哪个模型、哪套参数、哪种文本处理规则生成」）+ `13-FUTURE-ROADMAP.md:41–44` F-06 ANN 触发时的双库形态。
- **风险**：`【P1】` 冻结后补需改 Schema、已生成向量无法回溯（`B/B-01:360`）；`B-011` 在 M02 侧同样缺失（`B/B-02:23,489`），单补 M01 不够；`VECTOR.md` 的知识「已存在但没进正式设计」（`B/B-02:906`）。
- **迁移成本**：高。需同时改 M01 字段、M02 查询路由、manifest/索引空间定义、迁移窗口双写/追赶方案；且必须在 M01 冻结前做。
- **建议优先级**：**P0**（B-010/B-011 是 P1 缺陷且「冻结前」窗口已过 → 属结论级风险；A4 验收无法通过）。
- **判定**：「可直接继承（有裁决依据）」= B-010/B-011 缺陷结论（B 独立审查 Stage 1/Stage 7 实测，见 `B/B-01:256`、`B/B-07:392`）；补法本身 = `NEW PROPOSAL`。关联裁决：`D-AP-04`（摘要+cue 索引主体，`08-adjudication-ledger.md:71–76`）、`D-AP-05`（`08-adjudication-ledger.md:78–86`）。

### D5 · 锁与启动序列 → 补 FC-023（无启停序列）

- **出处**：`V242/07-LOCK-AND-STARTUP.md:5–16`（bridge.lock vs writer.lock 职责边界、获取顺序固定）、`:18–28`（锁升级为状态文件 pid/instanceId/state/heartbeat）、`:30–51`（S0 initializing→S1 lock_acquired→S2 recovering→S3 listening→S4 ready→S5 shutting_down→S6 stopped 六态启动状态机）、`:53–77`（stale 判定 GRACE=90s、B2a 等待而非接管、releaseLock 按 instanceId+pid 双匹配）、`:95–108`（退出状态机 drain 5s + recovery_blocked）；`R2/08-实施计划与估算.md:35`（M17 启动顺序依赖：恢复先于 flush）、`:55–61`（阶段门禁）、`R2/19-R2最终实施门禁闭合修订记录.md:37–39`（Issue C：admission barrier 未落地则 M06 reset 路由 fail-closed）。
- **核心机制**：两锁职责分离（进程唯一性 vs 跨进程写互斥）；**所有重型初始化放在 S2（拿锁之后、监听之前）**；退出时 `WriteQueue.drain(timeout 5s)` 在途写必须落定，超时标 `recovery_blocked` 不允许静默丢弃（`V242/07:110`）。R2 侧 M17/PHASE 把「恢复先于 flushQueue」列为激活门。
- **可复用点**：**六态状态机 + 双锁顺序 + drain/recovery_blocked 退出纪律**，可直接回答「FC-023 无启停序列」。
- **与 R4.1 接口**：canonical 包 `03-系统运行与协作协议.md` / `05-可靠性与运维规范.md`；`06-最小记忆闭环验收标准.md:40` A11（进程中断重启 0 差异）、`:41` A12（备份可恢复）都隐含启停序列。
- **风险**：`【P0】` **FC-023 编号在本轮可见材料中未检出**：在 `WORK/Memory-Core-R4.1-Design/`、`WORK/*.md`、`fin/*.md` 内 grep `FC-023|FC-0[0-9][0-9]` 均无命中（见 §6 V-2）。即 FC-023 可能是父 Agent/兄弟 Agent 的索引编号，本文按任务书给定口径使用，但**出处不可复现**，标 P0。`【P1】` 另：V2.4.2 的 `consolePortInUse` 失败开放、GRACE=90s（`V242/08:14` D-06）是 bridge.js 单体假设，R4.1 无 console 端口概念。
- **迁移成本**：中。状态机与锁顺序是契约级，可写成 R4.1 运行协议；端口探测/心跳实现需重做。
- **建议优先级**：**P0**（证据不可复现 + A11/A12 依赖启停序列）。
- **判定**：`NEW PROPOSAL`；R2 的 M17/PHASE-07/08 是「已提设计/门禁」，不是已批准决策（`R2/08:5` 状态四区分原文明确「计划的测试 ≠ 已通过的测试」）。

### D6 · 人格管线（PIPE-PERSONA CU-1 到 CU-8 + 锚定）→ 补 Persona 逐轮组装

- **出处**：`PIPE/04-SPEC-人格管线-AI可读.md:12–31`（MACHINE HEADER：`pipeline_id: PIPE-PERSONA`、`status: PROPOSED_NOT_APPLIED`、`authority: none — 本管线只有建议权，无实施权`、human_blocking_decisions D1/D2/D3）、`:40–45`（四件事：位置→system、频率→锚定式、体量 1558→~350 字符、载体 preset）；CU-1 `:145–178`（新会话 `roleAnchorAt`/`roleAnchorCount` 初值）、CU-2 `:182–206`（加载时归一化）、CU-3 `:210–242`（快照白名单补字段 ★CRITICAL，漏改则静默退化）、CU-4 `:246`（`roleLineForV2`）、CU-5 `:300`（三处调用替换）、CU-6 `:369`（补一次落盘）、CU-7 `:408`（人格卡瘦身，hard_dependency CU-8）、CU-8 `:482–544`（preset 追加「说话框架」进 system 层、`deploy_chain` 六步、risk HIGH）；批次与顺序约束 `:548–566`（batch_3 MUST precede batch_4，batch_1 MUST precede batch_2）；`PIPE/03-报告5-人格分层与记忆库重建.md:99–100`（压缩会吃掉 user 层人格卡、compaction-basic）、`:367`（框架抗压缩：user 否 / system 是）。
- **核心机制**：把「每轮注入完整人格卡」改为**锚定式**（bootstrap / 跨天 / 每 N 次）注入，靠 `roleAnchorAt`+`roleAnchorCount` 两个计数字段 + 快照白名单落盘；把「说话框架」（形式纪律）下沉到 system 层 preset，只写一次、免疫 compaction、不占 user 层；8 个 CU 是原子可撤销单位，按 4 个批次上线并带 B1–B10 门禁断言。
- **可复用点**：**「锚定式注入」的频率模型与「system 层免疫压缩」的分层原则**，与具体 bridge.js 行号解耦后可用于 R4.1 的 Persona 逐轮组装；`CU-3` 的「白名单静默失效」教训（`:222`「漏本 CU → 计数永远归零 → 退化为每轮完整卡 → 表观无收益且无任何报错」）本身就是极有价值的验收用例设计范式。
- **与 R4.1 接口**：R4.1 canonical 包无独立 Persona 模块（背景事实 6：canonical 集是 00–08 + 合同；`08-...` 为 DSH/QQBridge 集成规范）。建议挂到 `08-DSH与QQBridge集成规范.md` 与 M02 上下文构建（`03-m02.md:16` 把「相关记忆交给 DSH」）之间，作为「Persona 注入前置层」；与 D7/D8 的 `mind.hint()`「此刻 50 字」段对齐（见 `R3情绪体系整合方案.md:45–47` 组装比例 人格 300 字 : 此刻 50 字 : 记忆 N 条）。
- **风险**：`【P1】` PIPE 自述 `PROPOSED_NOT_APPLIED`、`authority: none`（`:17–18`），**绝不可写成已批准**；且 `scope_out` 明确排除「记忆库结构/记忆检索/记忆权限」（`:20`）——它不能补 R4.1 的检索/权限，只能补注入层。`【P1】` 与 `13-FUTURE-ROADMAP.md` 的 F-09 心理层门禁（I-4 双签 + ME-1 语料）不是同一条链，别混用。`【P2】` CU 里 `bridge.js` 行号（8799/8866/8958/9961…）依赖 `D:\qq-bridge` 12482 行快照（`:6`），R4.1 无此文件 → 行号全部失效，只有 YAML 语义可复用。
- **迁移成本**：中高。8 个 CU 需重新映射到 R4.1 的 preset/上下文组装点；`roleAnchor*` 状态字段需新载体；`deploy_chain` 的 patch/test 脚本是 DSH 专用。
- **建议优先级**：**P2**（Persona 逐轮组装是体验优化，不推翻 R4.1 结论；但「静默失效」验收范式值得提前吸收）。
- **判定**：`NEW PROPOSAL`（原文自述 PROPOSED_NOT_APPLIED，无裁决出处）。

### D7 · 心理状态分层（L0+L2 起步、五层预留、K-1 落点）→ 补情绪与运行时态

- **出处**：`PSY/SPEC.md:378–382`（L1 词典查表 VAD 种子 / L2 规则（否定/程度/施事/情态/命令/压抑）/ `extract.py` 实现了 L1+L2，零依赖，70 词内嵌词表）、`PSY/别人的方案-评估.md:57–71`（三层情绪 + 按情绪分别指定半衰期；「我的模型用单一 tau_emotion=25min 把所有情绪压成同一个速度」是粗糙的）、`:72–80`（顶点 peak/unmet 机制）、`:17–31`（19 个激素/递质生物层 + HPA 负反馈）、`:44–51`（从激素派生 energy/valence/irritability/clinginess）；`PSY/MOUNT.md`、`PSY/PIPELINE.md`（管线分层，实测存在，见 §6 检索命令）；裁决依据 `WORK/08-adjudication-ledger.md:21`（K-1 **有条件接受 O1——待 K-2 完成后正式生效**）、`B/B-00-MATERIAL-AND-BASELINE-AUDIT.md:183,197`（K-1「有条件接受」；m01 无对应段落，标「已裁决、待补设计」）、`B/B-07:608`（K-1 不参与权威判定/判权，待 K-2 完成后生效）。
- **核心机制**：`L1（词典）+L2（规则）` 零依赖起步，L3+ 留给 LLM/更复杂层；心理状态是**会话级、高频写、可丢失可重置的运行时态**，独立文件 `psy-state.json`，不进权威库（`R3情绪体系整合方案.md:60`）；K-1 规定「M01 命名空间内运行时态非权威 KV」，且**不参与判权**。
- **可复用点**：**「运行时态与权威数据分离」的定位** + L0/L2 起步的渐进分层，正好补 R4.1 缺失的「情绪与运行时态」；`psy-state.json` 与 R4.1 已有先例一致（Recall Cache `cache.sqlite`、SocialV2 会话态留 Bridge，见 `R3情绪体系整合方案.md:60`）。
- **与 R4.1 接口**：M01 命名空间新增 KV 对象（K-1）+ M02 检索的注意力容量输入 + `03-系统运行与协作协议.md`；`13-FUTURE-ROADMAP.md:54–71` 已把心理层全部登记为 Future（F-09~F-12）并受 X-1/I-4 门禁约束。
- **风险**：`【P0】` **K-1 尚未生效**：裁决台账 `:21` 与 `B/B-00:183` 一致写明「待 K-2 完成后正式生效」，台账纪律 `:4` 更写明「K-2 先于 K-1」。把 D7 写成「可直接继承」会越过裁决。`【P1】` B 独立审查实测「m01 全文无 K-1 对应内容」（`B/B-01:348`、`B/B-00:197`），即 R4.1 落点仍缺。`【P1】` `PSY/` 顶层目录含 385 份 md（大量 `_backup/` 快照），本轮只取顶层 `SPEC.md`/`别人的方案-评估.md`/`MOUNT.md`/`PIPELINE.md`，其余快照未逐份核验（§6 V-3）。
- **迁移成本**：高。需先完成 K-2 → K-1 才生效；再补 M01 D2/D10 落位设计；心理层代码（psy-model ~5600 行，`R3情绪体系整合方案.md:213`）是否引入仍是裁决项 I-7（`:227`）。
- **建议优先级**：**P1**（有裁决前置条件，顺序错了会违规）。
- **判定**：「可直接继承（有裁决依据）」= **仅 K-1 的定位原则**（出处 `08-adjudication-ledger.md:21`，附条件）；具体分层方案与 psy-model 实现 = `NEW PROPOSAL`。

### D8 · 情绪体系整合（R3 情绪方案）→ 补情绪附件与检索隔离

- **出处**：`2026-11-02_0013_08_R3定点修复-Work工作区/R3-Fix-Work-20261102-001345/R3情绪体系整合方案.md:3`（**待审批 Pending Approval**）、`:8–9`（B 管「现在」A 管「过去」；状态机为大脑、记忆字段为沉淀）、`:15–19`（结论摘要 5 条，含「B 自认所有测试都是构造的，没有真实对话语料」）、`:52–60`（三层分工表：现在/过去/反馈 + 状态载体）、`:64–94`（重叠裁决：用 B 七步链替代 A 的 EmotionBoost，±10% 降为护栏参数；七步链与 M02 D5/D7/D13/D22/D23 逐条对接；「M02 主权不动摇」）、`:148–180`（槽位 #17 PsyStateEngine L2、三条硬护栏 G1/G2/G3、影子模式强制 ≥2 周且 ≥75% 判向一致率）、`:184–197`（批准后 m01–m05 + 框架优化 + 映射表落点清单）、`:201–213`（S0–S6 阶段，合计新增集成代码 ≈800–1400 行）、`:217–229`（裁决点 I-1~I-8）、`:245–250`（声明：未获批前不实施、PSYCH 组件未运行未验证）。另：`2026-11-02_17xx_12_R3整合交付包-Agent1+Agent2/123/02-审查与修订报告/06-情绪体系整合方案.md`（同族交付副本，实测存在）、`:05-情绪系统设计提案.md`（A 方案，实测存在）。
- **核心机制**：情绪信号分两处沉淀——运行时态在 `psy-state.json`（会话级、非权威、不参与 M01 备份，`:128`），记忆侧 `affect_*` 字段在 `memory.sqlite`（权威，`:57`）；检索排序用 B 七步链（可插拔实现，**不是 M02 替代**，`:94`）；**检索隔离** = G1 hint 只进「此刻 50 字」段禁止进入 system 人格段（`:167`）、G2 状态机不产生记忆、G3 不参与删除/权限（`:168–169`）；上线前强制影子模式。
- **可复用点**：**「情绪附件与检索隔离」的三条硬护栏 + 影子模式退出条件**，可直接作为 R4.1 引入任何情绪/心理组件的前置约束；`affect_labels` 结构化过滤与七步链「不重叠（过滤=进不进候选池，七步=池内怎么排）」的判定（`:79`）是可复用的职责切分。
- **与 R4.1 接口**：`06-最小记忆闭环验收标准.md:43` A14（无 reinforce 写入路径）与 D-AP-05（`WORK/08-adjudication-ledger.md:78–86`：reinforce 仅由 DSH 回传最终 prompt 中实际出现的记忆 ID 触发，E14/B04 未确认前禁止写入）；`B/B-02:562` 实测 PSYCH 的 `reinforce()` 是召回侧自触发、无门控、未接入 focus()，**与 D-AP-05 触发语义冲突**；`B/B-02:601–607` 实测 PSYCH 字段（valence/repair/importance/recallCount/lastRecall…）与 M01 D19/D4 零对应、命名不一致（B-029）。
- **风险**：`【P0】` 与 **D-AP-05 直接冲突**（reinforce 触发语义 + 字段命名；见 §3 R5），且 `B/B-00:107` 已把「PSYCH 的再巩固依赖调用方写回，与 D-AP-05 的 E14 未落地前禁写 reinforce 要一并核对」列为待核对项。`【P1】` 方案自述待审批、组件自报测试未复核（`:249`）；`X-1/I-4` 双签门禁（`13-FUTURE-ROADMAP.md:57`）未过。
- **迁移成本**：高。≈800–1400 行集成 + 影子期 ≥2 周 + 真实语料抽样复核；且必须先解决与 D-AP-05 的字段/触发语义对齐。
- **建议优先级**：**P1**（冲突可解但需先裁决，不宜先吸收）。
- **判定**：`NEW PROPOSAL`（原文 `:3` 待审批；无裁决出处；且 §八 I-1~I-8 仍是审批入口）。

### D9 · 框架结构优化 14 条 → 逐条判『仍适用 / 已被 R4.1 吸收 / 冲突』

- **出处**：`2026-11-02_17xx_12_R3整合交付包-Agent1+Agent2/123/02-审查与修订报告/04-框架结构优化方案.md:4`（**设计建议文档，不是已批准决策**）、`:188–207`（§7.1 可插拔决策点清单 **#1–#14**，即「14 条」的实证落点）、`:209–219`（§7.2 不可插拔区 5 条）、`:41–64`（现状盘点 17 项接缝）、`:130–143`（§6.1 升级单元四分类）、`:227–263`（改动估算 ≈1.5k–2.2k 行）、`:296–304`（实施顺序 A/B/C）、`:308–316`（§十一 5 个需用户裁决点）、`:320–331`（与既有文档关系「不推翻任何已冻结决策」）。
- **说明**：任务称「14 条」，实测该文档的「14」= **§7.1 规则/大模型决策点接口清单 14 行**（#1 预检索触发 ~ #14 Dream 联想）；另 §8.1 有 D-1~D-10 十条文档改动、§十一有 5 个裁决点。本文按 **§7.1 的 14 条**逐条判定，并在 §6 注明口径（V-4）。

| # | 决策点（原文 `:192–207`） | 接口示例名 | 判定 | 依据 / 冲突点 |
|---|---|---|---|---|
| 1 | 预检索触发判断 | `QueryPlanner.shouldRetrieve` | **仍适用** | M02 D1/D2 规则触发；`03-m02.md:44` 两阶段混合检索仍在；接口未冻结（`:316` 裁决 5 选 b） |
| 2 | 查询改写/增强 | `QueryPlanner.rewrite` | **冲突** | `13-FUTURE-ROADMAP.md:104` F-20 明文「Query Rewriting 明确**不纳入**（m02 D1 L58/D20 L658）」；`06-...` A 类无此接口 |
| 3 | 关系扩展决策 | `ExpansionPolicy.plan` | **仍适用（受限）** | M02 D5 L0/L1/L2 预算内贪心；但焦点冲突 `focus()` 接管与 X-1 冲突（`13-FUTURE-ROADMAP.md:60` B-028） |
| 4 | 多路融合 | `FusionStrategy.merge` | **仍适用但参数未裁** | M04 D15「RRF 默认保留分数融合接口」；但 `B/B-07:25` 实测 k=60 唯一出处是 D-OK-19 且**正式台账零命中**，`13-FUTURE-ROADMAP.md:27–29` F-03 加权分数融合与 D-OK-19 取向相反、启用须新裁决 |
| 5 | 重排 | `Reranker.rerank` | **已被 R4.1 吸收（为 Future）** | `13-FUTURE-ROADMAP.md:10–19` F-01 Cross-Encoder 重排登记为 Future（重评触发：自有测试集就绪） |
| 6 | 时间衰减 | `DecayPolicy.factor` | **冲突** | 衰减存在多处定义、权威归属原文未说明（`B/B-00:270` B-013）；D-AP-05 禁止 `last_retrieved` 作衰减 v2 正向权重（`08-adjudication-ledger.md:85`） |
| 7 | 候选提取 | `ExtractionStrategy.extract` | **仍适用（有裁决）** | D-AP-01「M03 候选提取 / M04 整合裁决不可合并」（`08-adjudication-ledger.md:43–45`）；D-AP-02 唯一 Gateway（`63–68`） |
| 8 | 价值评分 | `ValueScorer.score` | **仍适用** | M03 §6/§11 权重版本化；`13-FUTURE-ROADMAP.md:109–110` F-22 参数面解冻需 ≥200 标注查询 |
| 9 | 更新关系判定 | Decision Provider（已有） | **已被 R4.1 吸收** | M04 D1 三态已定义（`:202` 原文自认「已有」） |
| 10 | 纠正语义 | 纠正策略（D5） | **已被 R4.1 吸收** | M04 D5 已声明可插拔（`:203`） |
| 11 | Observation 综合 | Decision Provider 同通道 | **已被 R4.1 吸收** | M04 4.3/6.1/6.4（`:204`） |
| 12 | 质量评估抽样 | 评估器接口 | **仍适用** | M04 §16 已声明（`:205`）；无实测 |
| 13 | 调优建议 | 建议器接口 | **仍适用** | M04 §15 已声明「规则或大模型可以提出」（`:206`） |
| 14 | Dream 联想 | 探索策略（未来） | **已被 R4.1 吸收（为 Future）** | `13-FUTURE-ROADMAP.md:74–76` F-13 Dream 记忆漫游、`:112–115` F-23 Dream MCP 接入，均登记 Future 且未启动 |

- **汇总结论**：14 条中 **仍适用 6**（#1/#3/#7/#8/#12/#13）、**已被吸收为 Future/既有机制 5**（#5/#9/#10/#11/#14）、**冲突 3**（#2 查询改写不纳入、#4 融合参数未裁、#6 衰减权威未定 + D-AP-05 禁止项）。§7.2 的「不可插拔区」（权限判定引擎禁 LLM 判权、Fail-Closed，`04-框架结构优化方案.md:211–217`）**与 M05 完全一致**（`03-m05.md:39,486–489`），建议整体继承。
- **风险**：`【P1】` 该文档 `:4` 自述「不是已批准决策」、`:337` 结尾「所有接口名、行数估算均为设计级内容，不代表已实现、已测试或已通过安全验证」，**不得写成已批准**。`【P2】` 「14 条」口径需固定（§6 V-4）。
- **迁移成本**：低（判定工作）+ 中（#2/#4/#6 三条冲突需各自取裁决）。
- **建议优先级**：**P2**（可整理；但 #4/#6 牵动 D-AP-05，建议随 D4/D8 一并处理）。
- **判定**：`NEW PROPOSAL`（逐条判定是本文新增工作）；其中「仍适用」项依托的是 R3/M0X 既有设计，非 R4.1 已批准项。

### D10 · 最小记忆闭环验收方法（A1 到 A15）→ 可作为 V0.9 验收集

- **出处**：`WORK/06-最小记忆闭环验收标准.md:30–44`（**A1–A15 逐条原文**）、`:51`（A1–A15 逐项结论：通过/不通过/模拟验证）、`:58`（A1–A15 全部有结论 → 输出报告 → 更新 PROTOTYPE-PROGRESS.md）、`:62`（**任何一项涉及红线的失败（A14）→ 立即停止并报告，不得继续**）。逐条摘要：A1 写入路径唯一 / A2 候选不入库（TTL 7 天）/ A3 黑名单生效 / A4 来源可追溯（含 B-010 字段齐备）/ A5 三路检索（FTS5、向量 FLAT、时间过滤）/ A6 RRF 融合稳定且 k 为配置参数 / A7 呈现合规（无内部 ID/分数）/ A8 预算生效（recall.budgetTokens）/ A9 修改同步（正文变了必须重算向量）/ A10 删除同步（两层消失）/ A11 恢复一致（重启后 SQLite/FTS5/向量 0 差异）/ A12 备份可恢复 / A13 权限挂载点存在（默认 DENY）/ A14 无红线违规（无 reinforce 写入路径、未触碰 `D:\qq-bridge`、正式设计文件未被修改）/ A15 偏差清单待裁决。
- **核心机制**：15 条可勾选清单 + 「A1–A15 全部有结论」才输出报告 + A14 红线一票停止 + 偏差必须登记（`PROTOTYPE-PROGRESS.md`）。
- **可复用点**：**整套清单可直接作为 V0.9 验收集**，无需重写；A4/A11/A12/A13 分别对应 D4/D1+D5/D2/D3 的验收口径，A6/A14 直接约束 k=60 与 D-AP-05。
- **与 R4.1 接口**：canonical 包 `06-测试与评估规范.md`、`test-specs/test-specs.md`；B 独立审查 `B/B-07:139–140` 的「部分可实施」结论可用 A1–A15 复测。
- **风险**：`【P0】` **A1–A15 不是历史方案，而是 work 目录现存验收标准**（背景事实 7：work 是 canonical 基线，不在时间流内）。把它当「从历史阶段挖出的方案」会失真 → 本文明确标注来源性质。`【P1】` A14 含「未触碰 `D:\qq-bridge`」，而 PIPE-PERSONA（D6）全部改动都在 `D:\qq-bridge`（`PIPE/04-SPEC:6,21`）——若 V0.9 同时吸纳 D6，A14 会被触发（见 §3 R6）。`【P2】` A6 要求「k 为配置参数」，与 `B/B-07:25`「k=60 是待验证参数不是已批准参数」一致但需声明。
- **迁移成本**：极低（清单可直接沿用）；需补的是把 A1–A15 与 §2 各方向的验收项做映射表（本文 §4 已给）。
- **建议优先级**：**P1**（V0.9 缺验收集会让 D1–D9 全部无法判定完成）。
- **判定**：「可直接继承（有裁决依据）」= A1–A15 本身是 work 目录现存正式文档（`06-最小记忆闭环验收标准.md`，非历史阶段产物）；**「作为 V0.9 验收集」这一用法 = `NEW PROPOSAL`**。关联裁决：`D-AP-06`（自有生活化测试集为主要验收依据，`08-adjudication-ledger.md:88–92`）、`CR-09`（门禁框架已接受但「未实施验证」，`08-adjudication-ledger.md:24`）。

---

## §3 与 R4.1 的冲突表

> 「R4.1」= `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design`（canonical 包，背景事实 6/7）+ work 目录 m0X/裁决台账。冲突按「若不处理会怎样」排序。

| ID | 冲突双方 | 冲突点（原文/实测） | 分类 | 优先级 | 处理建议 |
|---|---|---|---|---|---|
| **R1** | V2.4.2（D1/D2/D3/D5） vs R4.1 M01 | V2.4.2「第一版不引入新存储引擎：保持 JSONL + 文件锁」（`V242/00:36`）、唯一新增模块 `src/memory-service.js`（`:45`）、bridge.lock/writer.lock 双锁（`V242/07:5–16`）；R4.1 是 M01–M05 五模块 + SQLite 唯一权威（`03-m01.md:26` D1、`03-m05.md:18`）、M02「不得直接访问底层数据库」（`03-m02.md:38`） | **EXISTING（架构层）** | **P0** | 只继承不变量/验收场景（INo-1~6、AC-1.x/4.x），**禁止把 V2.4.2 的 JSONL/锁实现当 R4.1 代码基线**；否则结论「R4.1 事务边界已定义」会被推翻 |
| **R2** | V2.4.2 visibility（`owner_private`/`group_shared`） vs M05 scope（`private`/`group`/`shared`） | `V242/00:80` 与 `V242/05:65` 用两值；`03-m05.md:23,646` 用三值 + `(actor_id,target_id,action,scope)` 四元组（`:384`） | **COMPLETENESS** | **P1** | 统一为 M05 三值 + 四元组；V2.4.2 的「private 源拒绝共享到群」（`V242/05:29–30`）作为规则保留 |
| **R3** | V2.4.2 跨群分享「第一版仅同群」 vs M05 `shared` 范围 | `V242/05:36–37`、`V242/08:23` D-10 明确「跨群属未来扩展」；M05 有 `shared` 与 `target_group_id`（`03-m05.md:219`） | **COMPLETENESS** | **P2** | 若 R4.1 需跨群，须另取裁决（`V242/08` D-10 本身仍是「待用户决策」） |
| **R4** | 框架优化 #4 融合 / #6 衰减 vs R4.1 参数与裁决 | `B/B-07:25` k=60 唯一出处 D-OK-19 且**正式台账零命中**；`13-FUTURE-ROADMAP.md:27–29` F-03 与 D-OK-19 取向相反「启用须先取新裁决」；`B/B-00:270` B-013 衰减三处定义、权威归属未说明；`08-adjudication-ledger.md:85` D-AP-05 禁止 `last_retrieved` 作衰减 v2 正向权重 | **EXISTING** | **P1** | k 当「待验证参数」进参数 ID 规范（`B/B-00:261`）；衰减先画调用链再定权威；不得把 k=60 写成已批准 |
| **R5** | R3 情绪方案（D8） vs D-AP-05 | 情绪方案 `mind.reinforce(memoryId)` 由 Recall 命中/用户确认触发（`R3情绪体系整合方案.md:114–115`）；D-AP-05 要求 reinforce **仅由 DSH 回传最终 prompt 中实际出现的记忆 ID 触发**、E14/B04 未确认前禁写（`08-adjudication-ledger.md:83–84`）。`B/B-02:562` 实测 PSYCH `reinforce()` 召回侧自触发、无门控、未接入 focus()；`B/B-02:601–607` 实测 PSYCH 字段（`recallCount`/`lastRecall`）与 M01 D19/D4 零对应、命名不一致（B-029） | **EXISTING（实测）** | **P0** | 先对齐触发语义与字段命名，再谈情绪接入；不得先吸收再补 |
| **R6** | PIPE-PERSONA（D6） vs A14 红线 | `PIPE/04-SPEC:6,21` 全部改动在 `D:\qq-bridge`（`bridge.js` 12482 行 + preset）；`WORK/06-最小记忆闭环验收标准.md:43` A14 要求「未触碰 `D:\qq-bridge`」且 `:62` 规定 A14 失败立即停止 | **EXISTING** | **P1** | V0.9 验收集与 D6 不能同批上线；要么 D6 移出 V0.9 范围，要么显式修订 A14（需裁决） |
| **R7** | psy-model / 情绪「状态不进权威库」 vs K-1 未生效 | `R3情绪体系整合方案.md:60,128` 把 `psy-state.json` 定位非权威、不参与 M01 备份；但 K-1「M01 命名空间内运行时态非权威 KV」仍「待 K-2 完成后正式生效」（`08-adjudication-ledger.md:21`、`B/B-00:183`），且 B 实测 m01 无落点（`B/B-01:348`） | **EXISTING** | **P1** | 顺序：K-2 → K-1 生效 → M01 D2/D10 落位 → 再谈 psy-state；台账 `:4` 明写「K-2 先于 K-1」 |
| **R8** | 框架优化 #2 查询改写 vs R4.1 F-20 | `04-框架结构优化方案.md:195` 把 `QueryPlanner.rewrite` 列为可插拔决策点 #2；`13-FUTURE-ROADMAP.md:104` F-20「Query Rewriting 明确**不纳入**（m02 D1 L58/D20 L658）」 | **EXISTING** | **P2** | 不吸收 #2；如未来需要，走 F-20 重评而非直接启用 |
| **R9** | 「09-FUT-上下文压缩」类文档 vs 实测目录 | 任务点名在 `2026-11-01_16_R4.1续作与终版交付` 下找 09-FUT 类文档；实测该目录 FUT 档案为 `13-FUTURE-ROADMAP.md` + `A-04-V1-FUTURE-CLASSIFICATION.md`/`A7-V1-FUTURE-BOUNDARY.md`，无独立上下文压缩方案 | **COMPLETENESS** | **P1** | 标「未验证」+ 给验证命令（§6 V-1）；不得据此编造 D「上下文压缩方向」 |

---

## §4 建议吸收顺序

> 原则：**先解冲突，再补契约，最后谈体验层**；每一步都以 A1–A15 可勾选为完成判据。所有条目均为 `NEW PROPOSAL`，除标注「可直接继承」者。

| 阶段 | 动作 | 对应方向 | 前置 / 门禁 | 完成判据（A1–A15） | 优先级 |
|---|---|---|---|---|---|
| **S0 证据固化** | 把 canonical 包（`work\Memory-Core-R4.1-Design`）纳入可引用基线；补验 FC-023 出处；补验 09-FUT 文档是否存在；固定 D9「14 条」口径 | §1、§6 V-1/V-2/V-4 | 无 | 引用行号可 `Select-String` 复现 | **P0** |
| **S1 验收集就位** | 直接沿用 A1–A15 作为 V0.9 验收集；建立「方向 → 验收项」映射 | D10 | S0 | A1–A15 全部有「待验证/通过/不通过」结论；A14 红线一票停止 | **P1** |
| **S2 契约层（P0 缺陷）** | 补 M01 权限表（B-058）与 B-010/B-011 字段 + 索引空间/迁移规则；对齐 D-AP-05 触发语义与字段命名 | D3、D4、R5、R7 | S1；**须在 M01 冻结前**（`B/B-01:311–312,360`） | A4、A13；B-010/B-011 由「未解决」转「已解决」 | **P0** |
| **S3 事务与生命周期** | 把 INo-1~6 / WAL 语义 / tombstone 防复活 / 启停六态写成 R4.1 契约（SQLite 版，不搬 JSONL 实现） | D1、D2、D5、R1 | S2；启停序列需先补 FC-023 出处 | A1、A9、A10、A11、A12 | **P1** |
| **S4 裁决前置** | 完成 K-2 → 使 K-1 生效；补 M01 D2/D10 运行时态 KV 落位；X-1/I-4 双签门禁评估 | D7、R7 | S3；台账 `:4`「K-2 先于 K-1」 | A13（默认 DENY 不回退）；K-1 状态由「待生效」转「生效」 | **P1** |
| **S5 框架整理** | 按 D9 判定吸收 §7.2 不可插拔区 + 6 条仍适用项；#2/#4/#6 三条冲突逐条取裁决 | D9、R4、R8 | S1；#4/#6 与 D4/D8 联动 | A6（k 为配置参数，非 k=60 已批准） | **P2** |
| **S6 情绪/心理（带护栏）** | 若 S4 通过：以 G1/G2/G3 + 影子模式（≥2 周、≥75% 判向一致率）为前置，接入 PsyStateEngine；`affect_*` 走 M03 候选 + M04 审核 | D8、D7 | S4 + I-1~I-8 裁决 + 与 D-AP-05 对齐 | A14（无 reinforce 写入路径） | **P1** |
| **S7 体验层（最后）** | 锚定式 Persona 注入 + system 层免疫压缩；**移出 V0.9 或显式修订 A14** 后再上 | D6、R6 | S1；需裁决 A14 红线范围 | 不触发 A14 | **P2** |
| **S8 不吸收项归档** | 见 §5，登记「保留但不启动」清单 | §5 | 无 | 无新增 Future 混入 V1 | **P2** |

**吸收顺序一句话**：S0 固化证据 → S1 定验收集 → S2 补 P0 契约缺陷（权限表/embedding 溯源/迁移不变量/D-AP-05 对齐）→ S3 补事务/删除/启停 → S4 解 K-1 前置 → S5 整理框架 14 条 → S6 带护栏接情绪 → S7 才谈 Persona。

---

## §5 不建议吸收的历史方案（及理由）

| # | 历史方案 / 条目 | 出处 | 不建议吸收的理由 | 分类 | 优先级 |
|---|---|---|---|---|---|
| 1 | **V2.4.2 的 JSONL 存储引擎 + `MemoryService` 单模块实现** | `V242/00:36,45`、`V242/02:68–83` | 与 R4.1 SQLite 唯一权威 / 五模块边界根本冲突（R1）；且自认「10 万条全量重写耗时 NOT_VERIFIABLE」（`V242/00:155`）。只继承不变量，不继承实现 | **EXISTING** | **P0** |
| 2 | **bridge.lock / writer.lock 双锁 + `consolePortInUse` 失败开放 + GRACE=90s** | `V242/07:5–16,53–77`、`V242/08:14` | 依赖 bridge.js 单体、console 端口 3100、V1 行号；R4.1 无此运行时。继承「六态状态机 + drain/recovery_blocked」即可 | **COMPLETENESS** | **P1** |
| 3 | **PIPE-PERSONA 的 `bridge.js` 行号级改动（CU-1~CU-8 的 anchor/pre/post）** | `PIPE/04-SPEC:145–544` | 全部锚定 `D:\qq-bridge\src\bridge.js` 12482 行快照（`:6`），R4.1 无此文件；且触发 A14 红线（R6）。只继承「锚定式频率模型 + system 层免疫压缩 + 静默失效验收范式」 | **EXISTING** | **P1** |
| 4 | **查询改写 / Query Rewriting** | `04-框架结构优化方案.md:195`（#2）、`13-FUTURE-ROADMAP.md:104`（F-20） | R4.1 F-20 明文「明确不纳入」；如需要走重评，不直接吸收 | **EXISTING** | **P2** |
| 5 | **加权分数融合（F-03）** | `13-FUTURE-ROADMAP.md:27–29`、`B/B-07:574` | 与寄存器 D-OK-19「多路分数只经名次融合不直接相加」取向相反，启用须先取新裁决；当前 k=60 本身也未裁（`B/B-07:25`） | **EXISTING** | **P1** |
| 6 | **A 方案 EmotionBoost 独立决策点（±10% 乘性）** | `R3情绪体系整合方案.md:16,77,141` | 已被该整合方案自身「撤销独立决策点资格」，降为 B 七步链护栏参数；且与 B 七步链双重计权 | **EXISTING** | **P2** |
| 7 | **psy-model 的 19 激素生物层 / 单一 tau_emotion / 顶点 trait 演化直接进记忆** | `PSY/别人的方案-评估.md:17–31,57–80` | G2 硬护栏「状态机不产生记忆」（`R3情绪体系整合方案.md:168`）；且自认「所有测试都是构造的，没有真实对话语料」（`:19`）。可作研究参照，不作 R4.1 组件 | **COMPLETENESS** | **P1** |
| 8 | **Graphiti 旁路索引 / Hindsight 整机 / Mem0 向量库即权威** | `04-框架结构优化方案.md:109–118`、`:79`（反面教材） | 原文自判「保留但不启动」（报告2 H.1.2）；`13-FUTURE-ROADMAP.md:100–101` F-19 仅登记不启动；「权威数据永远不在可插拔件里」是硬约束 | **EXISTING** | **P2** |
| 9 | **R2 的 19–21 号门禁修订记录里的「已实施/已验收」表述** | `R2/19-...:3`（本轮仅改 04/08/09 三份 + 本记录）、`R2/08:5` | 08 号文状态四区分原文：「当前均未开工……计划的测试 ≠ 已通过的测试；文档记载的提议 ≠ 已批准的决策」。任何把 R2 门禁写成「已通过」的吸收都会严重误导 | **EXISTING** | **P1** |
| 10 | **B 独立审查的 50 份 `_B-REVIEW-BACKUPS` 快照** | `2026-11-03_13_B独立审查交付/_B-REVIEW-BACKUPS/`（实测 50 份） | 任务要求「取结论与方法，不取重复文件」；快照是过程副本，`B/B-REVIEW-PROGRESS.md:320–339` 才是结论登记表。吸收快照会引入版本漂移（同编号 B-02 有 v1~v9） | **COMPLETENESS** | **P2** |

---

## §6 方法附录

> 交付件状态：顶部 `STATUS: COMPLETE`（§6.4 记录骨架→填充→改 COMPLETE 的过程）。

### 6.1 检索命令（可复现）

> 本轮用 `glob`/`grep`/`read` 工具定位；下列为等价的 `Select-String` 命令，供父 Agent 复核行号（只读，不写源文件）。

```powershell
# 背景资产存在性
Select-String -Path 'C:\Users\[USER]\Desktop\时间流整理-全集\2026-10-31_04_V2.4.2设计收敛\2.42\00-V2.4.2-DESIGN-SUMMARY.md' -Pattern '第一版不引入新存储引擎','memory-service','M0（最小可交付）'
Select-String -Path 'C:\Users\[USER]\Desktop\work\08-adjudication-ledger.md' -Pattern 'D-AP-05','K-1','K-2 先于 K-1','CR-09'
Select-String -Path 'C:\Users\[USER]\Desktop\work\06-最小记忆闭环验收标准.md' -Pattern 'A1 ','A14','A15'

# B 独立审查结论（B-010/B-011/D-AP-05/K-1/B-029）
Select-String -Path 'C:\Users\[USER]\Desktop\时间流整理-全集\2026-11-03_13_B独立审查交付\B\B-00-MATERIAL-AND-BASELINE-AUDIT.md' -Pattern 'B-010','B-011','D-AP-05','K-1'
Select-String -Path 'C:\Users\[USER]\Desktop\时间流整理-全集\2026-11-03_13_B独立审查交付\B\B-01-M01-STORAGE-REVIEW.md' -Pattern 'B-010 成立','B-011 成立','冻结前','恢复旧备份'
Select-String -Path 'C:\Users\[USER]\Desktop\时间流整理-全集\2026-11-03_13_B独立审查交付\B\B-07-FINAL-ALGORITHM-REVIEW.md' -Pattern 'k=60','B-010','B-011','D-AP-05'

# CU-1~CU-8 / 锚定 / 批次顺序
Select-String -Path 'C:\Users\[USER]\Desktop\时间流整理-全集\2026-08-28_17_E-gptwork(agent3产物与研究报告)\E_gptwork\1739-extract\R3-MemoryCore-全套产物-20261102-1723\04-人格管线方案\04-SPEC-人格管线-AI可读.md' -Pattern 'CU-[1-8]','PROPOSED_NOT_APPLIED','锚定式','batch_3 MUST'

# 框架优化 14 条 / 不可插拔区
Select-String -Path 'C:\Users\[USER]\Desktop\时间流整理-全集\2026-11-02_17xx_12_R3整合交付包-Agent1+Agent2\123\02-审查与修订报告\04-框架结构优化方案.md' -Pattern '不是已批准决策','QueryPlanner','不可插拔区','设计建议文档'

# FC-023 / 09-FUT 未验证项复核
Select-String -Path 'C:\Users\[USER]\Desktop\work\*.md','C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\*.md','C:\Users\[USER]\Desktop\fin\*.md' -Pattern 'FC-023','启停序列'
Get-ChildItem -Path 'C:\Users\[USER]\Desktop\时间流整理-全集\2026-11-01_16_R4.1续作与终版交付' -Recurse -Filter '*FUT*' | Select-Object FullName
Get-ChildItem -Path 'C:\Users\[USER]\Desktop\时间流整理-全集\2026-11-01_16_R4.1续作与终版交付' -Recurse -Filter '*压缩*' | Select-Object FullName
```

### 6.2 取证纪律

1. **只读源目录**：本轮全部 `read`/`grep`/`glob` 均在 `C:\Users\[USER]\Desktop\时间流整理-全集`、`C:\Users\[USER]\Desktop\work`、`C:\Users\[USER]\Desktop\fin` 上只读；唯一写入是 `C:\Users\[USER]\Desktop\fin\04-HISTORICAL-PLANS-AS-DIRECTIONS.md`（本文件，用 `write`/`edit` 工具，未用 PowerShell `Set-Content/Out-File/Add-Content`，无 BOM）。**未修改任何源文件，本轮无需 `.bak-时间戳` 备份**。
2. **分块读**：单次 `read` 均 ≤200 行（上限 300），长文档按 offset 续读。
3. **行号口径**：`文件相对路径:行号`，行号来自 `grep -n` / `read` 的行号输出；文档级问题给相对路径 + 行号，文件级问题给相对路径。
4. **不写成已批准**：V2.4.2 自述「纯只读，未改任何代码」（`V242/00:3`）；框架优化自述「设计建议文档，不是已批准决策」（`:4`）；情绪整合自述「待审批 Pending Approval」（`:3`）；PIPE-PERSONA 自述 `PROPOSED_NOT_APPLIED`、`authority: none`（`:17–18`）；R2 08 号文状态四区分（`R2/08:5`）。以上均按 `NEW PROPOSAL` 处理。
5. **裁决出处**：`WORK/08-adjudication-ledger.md` 是「用户裁决的唯一登记处」（`:3`），其中 D-AP-01/02/04/05/06、K-1/K-2、X-1/I-4、CR-09 等条目均「已决定（用户）」但**「选择接受设计方案 ≠ 授权实施」**（`:4`）。本文标「可直接继承（有裁决依据）」时均给该台账行号 + 附条件。

### 6.3 未验证项（含验证命令）

| ID | 未验证内容 | 已做的核查 | 验证命令 / 建议 | 影响 |
|---|---|---|---|---|
| **V-1** | 「09-FUT-上下文压缩」类文档 | 在 `2026-11-01_16_R4.1续作与终版交付` 下 grep `compact\|压缩\|Compaction\|09-FUT\|上下文压缩` → 62 处命中，均为 `limit v2.md`（报告精简规范）、`inventory.csv` zip 名、人格管线「抗压缩」、m01 `cache.sqlite ... 压缩`、B-06 备份「v3 压缩说明」；`*FUT*` 实测 8 份（13-FUTURE-ROADMAP + A-04/A7 + 备份）、`*上下文*` 实测 0 份 | `Get-ChildItem -Path '...\2026-11-01_16_R4.1续作与终版交付' -Recurse -Filter '*压缩*' \| Select FullName`；并核对父 Agent 是否指 `13-FUTURE-ROADMAP.md` 的 F-01~F-24 | **P1**：若该文档真实存在而本文未取，D「压缩与上下文」方向会缺一段证据；本文未编造该方向卡 |
| **V-2** | **FC-023（无启停序列）编号出处** | 在 `WORK/*.md`、`WORK/Memory-Core-R4.1-Design/*.md`、`fin/*.md` grep `FC-023\|FC-0[0-9][0-9]\|启停序列\|无启停` → **无命中**；全时间流 grep 因 psy-model `_backup` 1402 份文件超 20MB 输出上限被截断 | `Select-String -Path 'C:\Users\[USER]\Desktop\work\*.md','C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\*.md' -Pattern 'FC-023'`；向父 Agent 索取 FC-023 的原始问题索引文件与行号 | **P0**：D5 的「补 FC-023」结论依赖该编号；出处不可复现时必须标 P0，不得伪称已核实 |
| **V-3** | `PSY/` 顶层 `_backup/` 快照（385 份 md） | 只读取顶层 `SPEC.md`/`别人的方案-评估.md`/`MOUNT.md`/`PIPELINE.md` 的存在性与 `SPEC.md:378–382`；未逐份核验 32 个 `_backup` 子目录内容是否含「三套方案总结 / v1.2 研究」正式版 | `Get-ChildItem '...\F_moren\psy-model' -File -Filter '*.md' \| Select Name`；对比任务点名的「三套方案总结」「v1.2 研究」是否在顶层（本轮实测顶层仅 4 份 md：别人的方案-评估/SPEC/MOUNT/PIPELINE） | **P2**：D7 引用的是顶层 SPEC/评估，若正式「三套方案总结」只在备份里，需补引 |
| **V-4** | D9「14 条」口径 | 实测 `04-框架结构优化方案.md` 中「14」= §7.1 决策点清单 #1–#14（`:192–207`）；另 §8.1 有 D-1~D-10（10 条）、§十一有 5 个裁决点 | `Select-String -Path '...\04-框架结构优化方案.md' -Pattern '^\| [0-9]+ \|'` 数表行 | **P2**：若父 Agent 指的是 D-1~D-10，本文 D9 判定表需重做；请确认口径 |
| **V-5** | D9 中「已被 R4.1 吸收」的判定 | 依据 `13-FUTURE-ROADMAP.md` F-01/F-13/F-23 与 M04 D1/D5 原文；未逐字比对 canonical 包 `02-核心算法规范.md`/`03-系统运行与协作协议.md` 全文（canonical 包不在时间流内，本轮仅按任务要求定向读取时间流资产 + work 目录 m0X/台账） | `Select-String -Path 'C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design\*.md' -Pattern 'Reranker','QueryPlanner','DecayPolicy','Dream'` | **P2**：判定为「Future/既有机制」，若 canonical 包已实装则判级需上调 |

### 6.4 交付件状态

- 骨架：先写 `STATUS: IN PROGRESS` + §1–§6 目录 → 分章填充（§1/§2 → §3–§6）→ 改 `STATUS: COMPLETE`。
- 本文件为 A4 唯一交付件；未写 `fin` 下其他文件名（同目录 `02-EVENT-FLOW-PROBLEM-INDEX.md` 为兄弟 Agent 交付件，本轮只读未改）。
- 三分类与优先级已按硬性规则 4 贯穿：每条方向卡/冲突行/不吸收项均带 `EXISTING`/`COMPLETENESS`/`NEW PROPOSAL` 语义与 P0/P1/P2。

---

## §7 父 Agent 增补（不在 A4 原始范围内，补两处遗漏）

> 说明：A4 的 D1–D10 与 V-1…V-5 原样保留；本节只补它**未覆盖**的两条方向，并关闭它的两个"未验证项"。均标 `NEW PROPOSAL`。

### 7.1 关闭 V-1：压缩类文档确实存在，但在 `E:\[PROJECT]`，不在时间流

A4 在时间流内找不到"上下文压缩"文档，判定为"未验证"。父 Agent 复核：**文档存在，位置不同**。

| 文件 | 大小 | mtime | 内容 |
|---|---:|---|---|
| `E:\[PROJECT]\FUT-09-上下文压缩代理-登记草案.md` | 6.3 KB | 11-05 18:44 | 拟并入 13 号 §7.13 的 FUT 行、六项接入门槛、待裁决项 |
| `E:\[PROJECT]\压缩内容分流进记忆库-可行性方案.md` | 12.6 KB | 11-05 18:51 | 三种分流位置对比、推荐方案 A、可行性分级、分阶段建议 |
| `E:\[PROJECT]\第二轮技术复审-billion-context压缩桥接.md` | 38.0 KB | 11-05 19:12 | §5 专章"Memory Core 兼容性" |
| `E:\[PROJECT]\第三轮专项技术审查报告.md` | 38.7 KB | 11-05 19:30 | 数据源与身份完整性 |
| `E:\[PROJECT]\CMB-v0.1-review-pack\` | — | 11-06 09:04 | 审查包（含 TEST-CREDIBILITY 审计） |

**因此 V-1 关闭**：不是"不存在"，而是"不在时间流"；这也证明时间流的"全集"连比自己新两天的材料都没收（见 `01` 的 AR-11）。

### 7.2 关闭 V-2：FC-023 的出处

`E:\[PROJECT]\AUDIT-FORENSIC-2026\09-CONFLICT-REGISTER.md:236` —— "FC-023 · No startup / shutdown sequence exists anywhere"。V-2 由 P0 降为 **P2（引用范围问题）**，D5 的"补 FC-023"结论成立。

### 7.3 新增 D11 · L1–L5 证据链（补"注入 ≠ 使用"）

- **出处**：`work\04-HANDOFF-20261103-R4.1-CR08A.md:152-192`（L1–L5、两条证据链、两阶段事件模型）；`work\12-A-REVIEW-PROGRESS.md §5.1`（F-CK-1…10 静态核查完成、运行验证未执行）。
- **机制**：L1 检索到 → L2 准备注入 → **L3 记忆版本确实进入最终 Prompt** → L4 实际使用（无可靠信号）→ L5 reinforce（禁写）。L3 由两条独立链共同证明：请求关联链（InjectionRecord→correlationId→turn）与内容映射链（memoryId+version→组装追踪）。两阶段事件（先记"已组装"，再判"有效 L3"）解决"有效性判定反过来依赖事件"的循环。
- **与 R4.1 的接口**：`08 §7.9` 的"宿主唯一拼装"、`14 §7.9` 的 `l3_event`（只证明 Injected）、`D-AP-05` 的 reinforce 禁写。
- **风险**：v4.6 本身是**未落盘、未批准的候选**（HANDOFF §3.1–3.2），只能当设计素材；DSH 侧 B04 回传未确认。
- **成本**：中（宿主侧埋点 + 事件表 + 两条链对账）。
- **优先级**：P1（它补的正是"记忆到底有没有被用上"这个空洞）。
- **判定**：`NEW PROPOSAL`。

### 7.4 新增 D12 · 压缩内容分流进记忆库（CMB）

- **出处**：见 §7.1 五份材料；核心是 `压缩内容分流进记忆库-可行性方案.md`（三种分流位置 + 推荐方案 A）与 `第二轮…§5`（Memory Core 兼容性）。
- **机制**：billion-context 的压缩块（summary/block）不直接写记忆，而是**分流**到 Memory Core 的候选通道（M03），保留原始消息 ID 以便回溯；压缩产物必须重新走 M03/M04/M05，不得自动成为长期记忆——这一点与 `15 §11.3`"压缩结果不能自动成为长期记忆"一致。
- **与 R4.1 的接口**：`15 §11.3`（Compression 独立）、`09 Pipeline`（候选入口）、`13 Evidence`（原始消息 → evidence_refs）、`14§7.9`（L3 事件）。
- **风险**：CMB 原型自述 mock；`CMB-v0.1-review-pack\12-TEST-CREDIBILITY-AUDIT.md` 对测试可信度有专项审计；"压缩摘要被当事实"是最大语义风险。
- **成本**：中高（需 CMB 侧稳定接口 + Memory Core 侧候选入口）。
- **优先级**：P1（它决定"长会话压缩后还能不能记住"）。
- **判定**：`NEW PROPOSAL`；CMB 是**相邻系统**，不是 Memory Core 基线（前轮取证已判）。

### 7.5 增补后的方向计数

D1–D10（A4 原始）+ D11/D12（父 Agent 增补）= **12 个方向**；其中 P0：D3/D4/D10；P1：D1/D2/D5/D7/D8/D11/D12；P2：D6/D9。
