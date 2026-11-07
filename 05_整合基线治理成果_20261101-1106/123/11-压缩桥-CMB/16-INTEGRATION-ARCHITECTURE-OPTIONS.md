# 16 · 接入架构方案比较（CMB v0.1 技术尽调 · Agent F）

> 文件编号：CMB-ARCH-0.1
> 状态：设计比较，**无实现、无执行、未连接任何数据库**
> 输入：`15-REAL-DATA-EXPERIMENT-PLAN.md`（下称"实验计划"）、原型 `docs/ARCHITECTURE.md`、`docs/adr/ADR-001…008`、`docs/reports/INTEGRATION-READINESS.md`、上游 `src/storage/{schema,store,persist,driver,integrity-check}.ts` @ `0aace2f`
> 结论位置：§5（推荐路线）、§6（推荐依据）、§7（不推荐的做法）

---

## 0. 一句话结论

**推荐路线 D（影子副本对账）：CMB 作为一个完全独立的只读进程，数据入口是上游产出的"一致性快照副本"，而不是活库。**
它把"能否在活库上安全地开只读连接"（R-18，至今 UNVERIFIED）从**架构前提**降级为**优化选项**；把权限边界从"能读全部用户明文"收缩到"能读一份备份产物"；把故障隔离做到物理级。
路线 A（活库只读直连）作为 **v0.3 的升级路径**，是否升级完全由 `E-3` 与 `E-10` 的实测数据决定，而不是由设计偏好决定。

---

## 1. 四条候选路线

### 1.1 路线 A —— 独立只读对账进程（定时跑，活库只读直连）

```
 bili (writer)                     CMB (reader, 独立进程)
 ┌──────────┐   WAL    ┌──────────────────────────────┐
 │ ingest   │─────────▶│ DatabaseSync(SRC,{ro:true})  │
 │ 单事务写  │          │   → Resolver → Validator     │
 └──────────┘          │   → EvidenceSink             │
      │                │   → BridgeLedger(独立库)      │
      │ 写 -wal/-shm    └──────────────┬───────────────┘
      ▼                               ▼
  index.db                      Evidence → Memory Core
```
定时器（如每 15 min）唤起进程，一次运行 = 一次全量或增量对账，跑完退出（无守护进程）。

| 维度 | 评估 |
|---|---|
| **优点** | ① 实现成本最低——原型已具备 `BillionContextSourceAdapter`（`bc-source.ts:25-218`）与七步引擎（`reconciliation-engine.ts:70-346`），只缺 `--config` 与增量游标；② 拿到的是**最新**数据（无备份延迟）；③ 进程级隔离，CMB 崩溃不影响 bot；④ WAL 的读不阻塞写，理论上零锁竞争（`schema.ts:88-101` 明确 WAL 模式；上游自己也用只读连接读活库，`integrity-check.ts:161-167`） |
| **缺点** | ① **押注在 R-18 上**：跨进程只读打开 WAL 库需要 `-shm` 可共享/可写，这一点从未验证（实验计划 U-2）；② 与上游争 page cache 与磁盘 IO；③ 长读事务会阻塞 WAL checkpoint（上游 `journal_size_limit=8 MiB`，`schema.ts:73-83`，被阻塞后 `-wal` 会增长）；④ **权限边界最宽**：进程必须能读 `index.db` = 能读全部用户明文消息；⑤ 需要给 CMB 一个"指向活库的路径"的部署配置，运维误配即事故 |
| **部署复杂度** | 中：一个定时任务 + 一个只读路径 + 一个 ledger 目录。但**安全评审成本高**（要论证"Cmb 只读"这件事，只能靠代码审计 + ACL，无法物理证明） |
| **权限边界** | **最宽**。可读范围 = `index.db` 全库（含所有会话的明文正文、`sessions.metadata_json`、`kstate_json` 等）。需要额外的行级/列级过滤才能在应用层"看起来"最小化，但 OS 层权限已经是全库 |
| **数据一致性** | 事务级强一致（单次读事务内），跨事务弱一致（两次读之间上游可能提交）。需显式 `BEGIN DEFERRED` 包读（上游范式：`store.ts:606-621`） |
| **故障隔离** | 中。进程崩溃/被杀 → bot 无感；但两者共享文件句柄、文件锁与磁盘，CMB 的 IO 风暴会传导到 bot |
| **实现成本** | **低**（估计最小）：补 `--config`（R-17）、增量游标、限流、`journal_mode=WAL` on ledger。**但验证成本高**（E-3 T3 需活库授权，E-10 需生产时间窗） |

### 1.2 路线 B —— 宿主内部插件或服务

在 `bili` 进程内以插件/服务形式运行 CMB，与写路径同进程。

| 维度 | 评估 |
|---|---|
| **优点** | ① 一致性最强（可与 ingest 同一事务/同一时刻读，甚至拿到事件）；② 无跨进程打开问题（R-18 消失）；③ 延迟最低（可做到秒级） |
| **缺点** | ① **直接违反项目前提**："不修改上游一行代码"（`README.md:11`、`ADR-001`）；② 上游对 `kernel/**` 与 `src/persist.ts` 有强制门禁：kernel 改动需 `kernel-change:` 声明 + owner 批准 + 双层测试（上游 `AGENTS.md` §Kernel Boundary）；`src/persist.ts` 被列为 hot-file（同文 §7.2）；③ CMB 的异常会传染 bot（同进程 = 同 GC、同事件循环、同崩溃域）；④ 上游的 plugin 机制是为 agent lane 服务的（`src/plugin.ts`），不是对外消费入口，语义不承诺稳定；⑤ 一旦上游升级，CMB 必然要跟改 → 维护成本是永久性的 |
| **部署复杂度** | 高：需要上游发版通道 + 版本对齐 + 灰度 |
| **权限边界** | **最宽**：继承宿主进程的全部权限（含网络、配置、所有会话） |
| **数据一致性** | 最强（可同事务读） |
| **故障隔离** | **最差**：同进程，无隔离 |
| **实现成本** | 高（跨仓库协作 + 门禁流程 + 长期维护） |

**判定：不采用。** 不是因为技术上不可能，而是它用"永久的上游耦合"换取"我们并不需要的实时性"（见 §3-Q1）。仅当上游**主动**提供稳定的"只读提交钩子/事件出口"时，才值得重新评估（那时它其实变成路线 C，而且权限边界也由上游定义）。

### 1.3 路线 C —— 事件驱动 + 周期性对账结合

上游每次落库提交后发出事件（文件/inotify/队列），CMB 消费事件做增量；同时保留一个低频全量对账兜底。

| 维度 | 评估 |
|---|---|
| **优点** | ① 延迟最低（秒级）；② 增量成本低（只处理变化）；③ 周期对账提供"最终一致性兜底"（丢事件也能收敛） |
| **缺点** | ① **上游当前没有可用的提交事件出口**。可用的替代物只有：`PRAGMA data_version`（连接级、只能回答"变了没有"，不含"变了什么"）、日志文件（格式非契约、随时改）、`fs.watch`（Windows 上 WAL 频繁写会造成事件风暴）；② 事件一旦丢失，增量游标错位，必须靠全量兜底，于是复杂度 ≈ A + 事件层；③ 事件内容必然包含"哪个会话变了"，这本身就是敏感信息；④ 需要守护进程（与原型"无常驻进程"的定位冲突，`README.md:217`） |
| **部署复杂度** | 高（守护进程 + 事件源 + 兜底调度 + 两条路径的一致性证明） |
| **权限边界** | 同 A（仍要读库）或同 B（若要事件内容） |
| **数据一致性** | 事件层强、全局最终一致；但"事件 vs 全量"两条路径必须证明等价，否则会出现"补了又删"的抖动 |
| **故障隔离** | 中（守护进程崩溃需自愈；但仍是独立进程） |
| **实现成本** | **最高**（两条路径 + 等价性测试） |

**判定：v1.0 之后再考虑，且前提是上游先给出稳定事件面。** 现在做等于用自研的"轮询 + 推断差量"伪装成事件驱动。

### 1.4 路线 D —— 影子副本对账（独立进程 + 上游原生一致性快照）

CMB 永不接触活库。上游（或运维）周期性产出一份**一致性快照**，CMB 只读这份副本。

```
 bili (writer)
 ┌──────────┐
 │ ingest   │──写──▶ index.db (+ -wal/-shm)
 └──────────┘              │
                           │ ①上游原生 VACUUM INTO（store.ts:694-709，writer 连接，安全且不阻塞写者）
                           │   或 ②node:sqlite backup() from 只读连接（待 U-1 确认）
                           │   或 ③停写窗口冷拷贝（B-2 兜底）
                           ▼
                    snapshots/index-<ts>.db      ← 权限边界在这里切断
                           │ 只读
                           ▼
                    CMB (独立进程, 只读打开副本)
                           │
                           ▼
                    BridgeLedger (独立库) → Evidence → Memory Core
```

| 维度 | 评估 |
|---|---|
| **优点** | ① **权限边界最小**：CMB 及其运行用户**不需要 `index.db` 的读权限**，只需要"备份目录的读权限"。快照产出由拥有库权限的运维/宿主完成，CMB 是纯消费者；② **R-18 从前提降级为优化**：能否跨进程只读打开 WAL 活库，变得**无所谓**（副本可以没有 `-wal`，或用 `immutable` 之外的正确方式产出）；③ **故障隔离最强**：CMB 的 IO/CPU/崩溃完全碰不到活库；④ 长读事务不可能阻塞上游 checkpoint（不共享 WAL）；⑤ 可重放/可回溯：保留 N 份快照即可"穿越"历史做对账（调试与审计都极有价值）；⑥ 快照产出机制是**上游自己的**（`store.ts:694-709` 的注释明确写："online consistent snapshot via SQLite's native VACUUM INTO — safe while WAL writers are active (it reads a consistent snapshot and never blocks them)"） |
| **缺点** | ① 有延迟（快照周期，通常 15 min – 24 h），因此证据滞后；② 需要磁盘（每份副本 ≈ 库大小；10 GiB 库 × 保留 3 份 = 30 GiB）；③ 需要调度与保留策略（谁产出、多久一次、留几份、怎么删）；④ 快照产出本身需要一个"能写"的角色执行（但那是上游/运维，不是 CMB）；⑤ 增量对账需要"上一份快照"做差分（否则每次都全量扫） |
| **部署复杂度** | 中：定时备份任务 + 副本保留策略 + CMB 定时任务。比 A 多一个组件，但**安全评审成本最低** |
| **权限边界** | **最窄**。可读范围 = 备份目录。且可进一步收缩为"快照产出后由另一个低权限账户读取"，实现真正的职责分离 |
| **数据一致性** | **强**（快照由 SQLite 保证帧级一致）。跨快照之间是"离散时间点"，语义清晰（不会出现"读到半个事务"） |
| **故障隔离** | **最强**。唯一共享资源是磁盘空间 |
| **实现成本** | 中：增量游标 + 副本生命周期管理 + 差分算法。但省掉了 E-3 T3/E-10 的活库风险验证（仍建议做，作为"未来升级"的输入） |

### 1.5 四路线总览评分（1=差，5=好；"权重"按项目约束给定）

| 维度 | 权重 | A 活库直连 | B 宿主内嵌 | C 事件驱动 | **D 影子副本** |
|---|---|---|---|---|---|
| 实现成本（5=最省） | 0.20 | 5 | 1 | 2 | 3 |
| 权限边界（5=最窄） | 0.20 | 2 | 1 | 2 | **5** |
| 故障隔离（5=最强） | 0.20 | 3 | 1 | 3 | **5** |
| 数据一致性（5=最强） | 0.15 | 3 | 5 | 4 | 4 |
| 部署复杂度（5=最简单） | 0.15 | 4 | 1 | 1 | 3 |
| 实时性（5=最强） | 0.10 | 4 | 5 | 5 | 2 |
| **加权** | 1.00 | **3.55** | **2.00** | **2.75** | **4.05** |

> 权重说明：本项目的第一约束是"**不改上游 + 不冒险进热路径 + 缺失即拒绝**"（`README.md:11`、`ADR-001`、`ADR-005`），因此权限边界与故障隔离权重最高；实时性是**最不重要**的一项（§3-Q1 给出量化理由）。若把实时性权重提到 0.4，A 会反超 D —— 这也是为什么"是否需要实时性"必须先被回答，而不能被设计者默认。

---

## 2. 四条路线与原型现状的差距（实现工作量）

| 路线 | 原型已有 | 必须新增 | 必须删除/关闭 |
|---|---|---|---|
| A | 只读适配器（`bc-source.ts:30` `readOnly:true`）、七步引擎、三账本 | `--config`（R-17）、增量游标、限流、ledger 的 `journal_mode=WAL`+`busy_timeout`、单实例锁 | `MockSourceStore` 作为默认、内存 sink 作为唯一 sink |
| B | 无（不可复用，需在上游仓库内重写） | 上游插件骨架、门禁流程、双层测试 | 原型的 CLI 形态 |
| C | 七步引擎 | 事件源（**上游没有**，需自研或等上游）、守护进程、事件与全量的等价性测试 | 同上 |
| **D** | 只读适配器、七步引擎、三账本 | 快照目录消费（路径参数化）、副本生命周期（`source_instance_id`）、差分游标、ledger 的 WAL/busy_timeout、单实例锁、`run_ledger` 保留策略 | `MockSourceStore` 作为默认、内存 sink |

**关键观察**：A 与 D 在**原型侧的差距几乎相同**（都要补游标、限流、ledger 加固、单实例锁）。差别只在"数据入口是一个活库路径还是一个快照目录"——而这一个差别，同时买到了权限边界与故障隔离两项最高权重。

---

## 3. 十个工程问题逐条分析

> 编号沿用任务书 §B 的 1–10。每条给"结论 + 依据 + 若结论错误的代价"。

### Q1 是否需要实时性？（Evidence 延迟多久可接受？）

**结论：不需要秒级；建议分级 SLA，且"撤销"比"新增"要求更紧。**

| 事件类型 | 建议 SLA | 依据 | 违反后的后果 |
|---|---|---|---|
| 新增 Evidence（正常消息） | **T+15 min（P95）**，硬上限 T+2 h | 下游是 Memory Core 的 `candidate → pending_review → active` 流程（`contracts/state-machines/state-machines.md:14-16`），其中 `pending_review → active` **只能由 M04 审核通道**写入（同文 `:15`，台账 D-AP-05）。也就是说，Evidence 到达后**本来就要过人工/审核环节**，秒级对账不会让记忆更快被"确认" | 记忆晚一点可用，无数据错误 |
| **撤销/删除传播** | **T+1 h**，且必须文档化"上界 = 对账周期 + 1 个周期" | 用户删除/撤销是对外承诺（`04 权限与治理规范` §7），"删了还在"是治理事故。但注意：**CMB 无法比它的扫描周期更快地知道源变了**（见 Q4） | 治理事故（内容已删但记忆库仍有） |
| 历史回填（首次/重建） | T+24 h | 全量首扫成本高（Q3） | 只影响"历史记忆"可用时间 |

**推论**：15 min–1 h 的快照周期/对账周期完全够用；路线 D 的延迟劣势（2 分）在本项目里**不构成实质劣势**。这也是推荐 D 的核心逻辑之一：**为不需要的实时性付出权限与隔离代价，是错的交换。**

**若结论错误的代价**：如果将来产品要求"用户发完消息 5 秒内就能检索到记忆"，则 D 直接出局、A 也需要改造（因为 15 min 周期不够）→ 那时正确答案是 C/B。**因此这条必须由用户书面确认，不能由架构自证。**

### Q2 是否必须读侧对账？（能不能在压缩热路径同步写？）

**结论：必须读侧对账。同步写不是"更难"，而是"方向错误"。**

| 依据 | 位置 |
|---|---|
| 上游把 `messages/content/blocks` 定义为 **append-only facts**，"UPDATE is a bug；唯一合法变更是 blocks 的 active/supersedes 与 content 的 refcount"，且"compression ledger 的每一次状态变更是一个事务" | `schema.ts:6-9` |
| ingest 是**单事务**写入（`ingestSession`），事务越长，写锁持有时间越长 | `store.ts:811-819` |
| 上游自己因为"写放大/写锁饥饿"踩过重大事故：`#4677`（迁移把整库 OOM/永久不可打开）、`#4796`（一条未提交的 `DELETE ... WHERE refcount<=0` 跑 8 小时、WAL 无法 checkpoint、**"the held write lock starved live proxies"**）、`#4612`（full VACUUM 持独占写锁，**主动拒绝**在另一进程活跃写时执行） | `schema.ts:369-389`、`store.ts:2139-2155` |
| 上游为多进程共享拓扑专门做了 `BEGIN IMMEDIATE`（避免 `SQLITE_BUSY_SNAPSHOT`） | `driver.ts:112-117`、`:344-345` |
| ADR-001 已把"读侧对账、不进热路径"定为决策 | 原型 `docs/adr/ADR-001-bridge-architecture.md` |

**所以**：任何"在压缩时顺手写 Evidence"的方案，都要在**上游的写事务里加一次外部写入**（网络/磁盘/另一个库）。这在结构上等于把 CMB 的可用性并入 bot 的可用性——上游刚刚为此类耦合付出过 8 小时写锁的代价。

**若结论错误的代价**：如果坚持同步写，则必须改上游（路线 B），并接受"CMB 的任何延迟/故障都会拖慢或阻塞 bot 的每一次 save"。

### Q3 是否需要全量扫描？增量游标怎么维护（能用 `sessions.last_ingested_seq` 吗？）

**结论：不能把 `last_ingested_seq` 当桥接游标。增量游标必须由 CMB 自己的台账提供；全量只在"首次 / 重建 / 补偿"时做。**

**为什么 `last_ingested_seq` 不可用（三条硬理由，全部有行号）**：

1. **它每次进程启动都会被清空**：`resetAllIngestWatermarks()` 执行 `UPDATE sessions SET last_ingested_seq = NULL`，理由是"watermark 不携带 writer 身份，不能跨进程 handoff 继承信任"（`store.ts:1497-1512`）。也就是说它是一个**易失的写者内部状态**，不是持久游标。
2. **它的语义是"tail 的 seq + 该次 save 的 tailSeqs 必须自洽"**：`ingestWatermark()` 除了读列，还要校验 `tail_json.tailSeqs` 的最后一个元素等于该 seq，否则返回 null（`store.ts:1456-1467`）。任何"外部改写/部分升级"都会让它变 null。
3. **它可能被中途清除**：`clearIngestWatermark()` 在读回 digest 证明前缀失配时被调用（`store.ts:1487-1495`）。
   → 用它的结果是：**对账游标会随机归零**，导致要么漏扫（如果 CMB 把它当"已处理到此处"），要么全量重扫（如果它变 null）。前者是数据错误，后者是性能事故。

**CMB 自己的增量游标（建议定义）**：

```
游标 = (source_instance_id, session_id, cursor_kind, cursor_value, updated_at)

cursor_kind = "MESSAGES_MAX_SEQ"   →  value = 该 session 已处理到的最大 messages.seq
cursor_kind = "BLOCKS_SET_DIGEST"  →  value = sha256(sorted(block_id|block_version) 列表)
```

- **消息增量**：`SELECT ... FROM messages WHERE session_id=? AND seq > ?`。前提：上游 `messages` 的 seq 稳定且单调（`PRIMARY KEY (session_id, seq)`，`schema.ts:153`）。⚠ 但 seq 会被"编辑追加"改变（`store.ts:1552-1556` 注释："edits append；callers keep the latest per id"）→ 旧 seq 上可能出现新内容；靠 `BLOCKS_SET_DIGEST` 兜底触发重扫。
- **块的增量**：块会**物理消失**（`store.ts:1077-1087`），所以"增量"不能只靠"新块 id > 上次最大"——必须每轮对每个 session 拉一次全量 `(block_id, block_version)` 列表做集合差分。成本 = O(每会话块数)，这是**必须付的**成本，也是 R-06 的代价。
- **全量触发条件**：①首次；②`source_instance_id` 变化（库被重建，见 Q5）；③`BLOCKS_SET_DIGEST` 与台账不一致超过阈值；④用户显式重建台账（`ARCHITECTURE.md:401-411` 的"删 ledger 重建"路径）。

**是否需要全量扫 messages？** 只在首次与重建时。首次全量的成本未知，必须由 `E-9` / `E-11` 给出真实行数与行宽（实验计划 §9 U-6/U-8）。

**若结论错误的代价**：若误用 `last_ingested_seq`，最坏情况是**静默漏账**（游标归零后 CMB 以为"已处理完"), 只在"证据缺失"被下游发现时才暴露，且无法回溯——这是最坏的失败模式（无声错误 > 崩溃）。

### Q4 如何检测源数据变化？（轮询？mtime？WAL 变化？）

**结论：三层门控，权威判据是"台账差集"，唤醒信号用 `PRAGMA data_version`，mtime 只作粗粒度辅助。**

| 层 | 机制 | 成本 | 语义 | 用途 |
|---|---|---|---|---|
| L1 **快/粗** | `index.db` + `index.db-wal` 的 `(size, mtime)` | 极低（两次 `stat`） | 粗略（WAL 在写就变；但"变了"不等于"有新消息"；有快照隔离时 mtime 变化也可能是 checkpoint） | 快速跳过"什么都没发生"的周期 |
| L2 **准/轻** | **`PRAGMA data_version`**（同一连接上，别的连接提交后该值变化） | 一次 pragma | 精确（"自上次读以来有其他连接提交过"），但**不含"变了什么"** | 决定"这一轮要不要真的去查账" |
| L3 **权威** | 台账差集：`messages.seq > cursor` 的计数 + `blocks` 集合差分 | O(变化量) + O(每会话块数) | 权威（唯一能证明"确实有未桥接的数据"） | 真正的对账驱动 |
| ❌ 不用 | `fs.watch` / `ReadDirectoryChangesW` | — | Windows 上 WAL 每次提交都会触发事件 → 事件风暴 + 抖动 | — |
| ❌ 不用 | 只靠 mtime | — | 无法区分"checkpoint 搬页"与"新消息" | — |
| ❌ 不用 | `PRAGMA quick_check` / `integrity_check` 当"变化检测" | 极高（14 GiB ≈ 260 s，`store.ts:455`） | 完全错的工具（检测损坏，不是检测变化） | — |

**为什么 `data_version` 是最佳唤醒信号**：它是 SQLite 的连接级计数器，专门为"我是否需要重新读"设计；零 IO、零锁、不写库。在路线 D 下它不适用（我们读副本，副本不会变），因此路线 D 的 L2 退化为"副本目录 mtime"——**这是 D 相对 A 的一个真实劣势，必须承认**。

**若结论错误的代价**：只用 mtime 会导致"每轮都全扫"（性能事故）；只用 data_version 而不做台账 diff 会导致"上游提交了但与本会话无关的写入"引发无效扫描（噪声）。

### Q5 如何处理数据库重建（用户删了 `index.db` 重建）

**结论：必须显式建模"源实例"（`source_instance_id`），重建 = 换实例 = 强制全量重扫 + 旧台账行标 STALE（不删）。**

**上游的重建语义（必须先理解，否则设计错）**：
- 上游**没有自动重建**：`quick_check` 失败时，策略是"alarm + 启动时不用 index（文件原地不动、不自动搬、不自动重建）"，运维**手工**把文件移走再启动，新的 index 会**从 JSON envelope 重新导入**（`store.ts:327` 的提示文案、`store.ts:493-498`）。
- 因此重建后：`sessions` / `messages` / `blocks` 都会**重新编号**（`next_block_id=1`、`seq` 从 0 重排），但**内容可能高度相似**（同一批历史）。
- 重建后 `PRAGMA user_version` 会被 `migrateSchema` 设为当前版本（`schema.ts:403`）。
- 另有一个不易察觉的"重建"：`bili sessions rm` 会删除单个会话并把 index 物理收缩（`storage-maintenance.ts:119-150`）——这不是全库重建，但对 CMB 表现为"某些会话整体消失"。

**检测信号（建议全部采集，任一命中即 `SOURCE_RESET`）**：

| # | 信号 | 判据 |
|---|---|---|
| S1 | `PRAGMA user_version` 变化 | 版本回退或跳变 |
| S2 | 文件指纹变化 | `index.db` 首 64 KiB 的 sha256 变化（RO 审计 A2 已在采） |
| S3 | `<db>.lock` 的 pid 与 mtime 变化 | 上游重开（`store.ts:645-683`） |
| S4 | 台账里"全部已知 session 的 blocks 集合"同时全空 | 强信号 |
| S5 | `sessions.created_at` 的最小值上移 + `SELECT COUNT(*) FROM sessions` 骤降 | 强信号 |
| S6 | `source_instance_id` 推导值变化 | 见下 |

**`source_instance_id` 建议定义**（CMB 侧新增，不影响上游）：
```
source_instance_id = sha256( user_version ‖ count(sessions) ‖ min(sessions.created_at)
                             ‖ sha256(index.db 首 64KiB) )   # 每轮采样，不稳定则视为重建
```
**重建后的处置（四步，缺一不可）**：
1. 旧 `bridge_key` **全部失效**（因为 `block_id` 从头编号，`covered_hash` 可能相同也可能不同）→ 不能靠 `bridge_key` 命中来跳过。
2. 旧 ledger 行**不删**，标 `stale=1` + `superseded_by_instance=<new>`。理由：它们是"曾经存在过"的唯一证据（D-8 的可见性），删了就永久丢失。
3. 强制该实例全量重扫；Evidence 幂等仍由 `(content_hash, source_locator)` 兜底（`evidence-adapter.ts:73-76`）。⚠ 但重建后 `seq` 与 `msg_id` 可能变化 → **会产生重复 Evidence**（R-10 的放大版）。这是**必须书面接受的代价**，不得用"文本级去重"掩盖（那会违反"同文本不同消息不合并"，`TC-I-04`）。
4. 报告里必须出现 `SOURCE_RESET` 事件与"可能重复的 Evidence 数量上界"。

**若结论错误的代价**：若不建模 `source_instance_id`，重建后 CMB 会用旧 `bridge_key` 误命中并**跳过整块**（因为它认为"这个块已 COMMITTED"）→ **静默漏写全部历史证据**，且台账显示"一切正常"。这是本设计里最危险的单一失败模式。

### Q6 如何处理历史数据缺口

**结论：三类缺口，全部**显式标注**，一律不猜、不补、不伪造。且**不改 12 态状态机**（避免迁移），用 `failure_reason` 承载。**

| 缺口类型 | 成因（行号） | 是否可恢复 | CMB 处置 |
|---|---|---|---|
| G-1 块已物理删除 | `store.ts:1077-1087` | **不可恢复**（块的 `payload_json` 已随行删除） | 台账保留 `first_seen_at/bridged_at`；`failure_reason="GAP_BLOCK_DELETED"`；报告单列"不可追溯区间" |
| G-2 CAS 行缺失（哈希在、原文没了） | `store.ts:790-808` delete-on-zero；`store.ts:3013` 的 `?? ""` 静默 | **不可恢复** | Validator 判 `CONTENT_MISSING`（`integrity-validator.ts:78-79`）→ 已正确处置；额外要求：报告里给出"孤儿率"（E-2 P4） |
| G-3 v10 之前无 `ts` | `schema.ts:391-400`（no backfill）+ `store.ts:1029-1038`（batchTs 兜底） | **不可恢复**（时间信息从未落盘） | Evidence `timestamp` 恒 null（`evidence-adapter.ts:60`）→ 已正确处置；D-7 接受 |

**"缺口"不能只是统计数字，必须是可查询的状态**：
- 现有 12 态里 `MISSING` / `CORRUPTED` 可承载 G-2/G-3，但**没有**"已知不可恢复"的语义（`MISSING` 可回 `DISCOVERED` 重试，`domain.ts:164`）。
- 建议（v0.2，不改状态机）：`message_ledger.processing_status` 仍用 `MISSING`，另加 `failure_reason="GAP_UNRECOVERABLE:BLOCK_DELETED"`，并在 `ReconciliationReport` 里单列 `unrecoverable_gaps` 计数。
- **禁止**：把 G-1 的块"重建"出来（不可能）、把 G-3 的时间"用 `created_at` 代替"（伪造，违反 M-3 与 13 号 §7.2-2）。

### Q7 如何保证不影响压缩热路径

**结论：靠"物理隔离 + 白名单 + 限流 + 禁区"四件套，而不是靠"我们小心一点"。**

| 措施 | 具体做法 | 依据 |
|---|---|---|
| 物理隔离 | 独立进程；独立 ledger 库；路线 D 下连文件都不共享 | 本文件 §1 |
| 只读白名单 | 脚本/代码里只允许 `SELECT` + 只读 `PRAGMA`（`query_only/data_version/table_info/user_version`）；CI 用 grep 黑名单断言（实验计划 §1.4） | 实验计划 §1.4 |
| 禁区 | 禁止 `UnifiedStore.open` / `migrateSchema` / `wal_checkpoint` / `quick_check` / `VACUUM` / 写 `.lock` | 实验计划 §1.4 的 N-1..N-5（每条附行号） |
| 限流 | 单连接串行；`batch_size=200`（`config.ts:70`）+ 每批 `sleep`（**注意：`batch_size`/`concurrency` 目前是空转配置**，见 R-14，`config.ts:37-39,70-72` 但引擎从不读取 → v0.2 必须实现或删除）；单次读事务 ≤2 s；禁在事务里读正文 | `RISK-REGISTER.md:266-273`（R-14） |
| 事务时长 | 长读事务会阻止 WAL checkpoint（上游 `journal_size_limit=8 MiB`，`schema.ts:73-83`）→ 只读侧必须短事务 | 同上 |
| IO 优先级 | Windows 进程优先级 `BelowNormal`；路线 D 下快照产出避开低峰 | — |
| 观测 | `E-10` 的基线/扫描对比是唯一能证明"没影响"的方法 | 实验计划 E-10 |

**注意一个反直觉点**：真正危险的不是"读得多"，而是**"读得久"**（长事务阻塞 checkpoint）+ **"打开方式错"**（writable 打开会写 `.lock`、跑迁移）。前者靠限流，后者靠只读连接 + 代码白名单。

### Q8 如何隔离生产故障（CMB 崩了会不会影响 bot？）

**结论：路线 D 下"结构上不可能"；路线 A 下"基本不会，但有四条传导路径"。**

| 传导路径 | A 下是否存在 | 机制 | 缓解 |
|---|---|---|---|
| P1 进程崩溃 | 否（独立进程） | — | 无需 |
| P2 文件锁 / `.lock` 覆盖 | **是**（若误用可写打开） | 上游以 `<db>.lock` 判"另一进程正在写"（`store.ts:645-683`）；误写会让上游的 `gc full VACUUM` 被拒（`store.ts:2139-2155`） | 强制 `readOnly:true`（只读连接不会写 .lock）+ 代码白名单（禁 `open()`） |
| P3 WAL 无法 checkpoint | **是** | 长读事务持有 WAL 读标记 → checkpoint 无法推进 → `-wal` 增长到不受 `journal_size_limit` 约束（该 pragma 只限制 checkpoint **之后**的截断，`schema.ts:73-83`） | 单次事务 ≤2 s、总并发 1、只读计数不读正文 |
| P4 磁盘/IO/CPU 争用 | **是** | 与上游共享磁盘 | 限流 + IO 优先级 + `E-10` 量化 |
| P5 磁盘写满（我们自己） | 是（所有路线） | ledger 增长 / 副本堆积 | 保留策略（Q9）+ 配额告警 |

**关键设计原则（建议写成硬约束）**：**CMB 的故障必须 fail-open 到"停止对账"，绝不 fail-closed 到"阻止 bot 运行"。**
对应到实现：CMB 的定时任务失败只写自己的告警文件/日志，**不碰上游任何状态**；心跳超时由运维看板发现，而不是由上游发现。

### Q9 如何控制长期存储增长（ledger 会无限增长吗？）

**结论：会。当前实现有**三处**无界增长，且没有一条 `DELETE`、没有 WAL、没有保留策略。**

**现状证据**：
- `bridge-ledger.ts:59-63`：`new DatabaseSync(dbPath)` —— **没有设 `journal_mode`、没有 `busy_timeout`**（对比源侧 `schema.ts:88-101` 每连接都设）。
- `bridge-ledger.ts:65-108`：三张表 DDL，**无任何 `DELETE` / `VACUUM` / 保留列**（全仓库 grep 无 DELETE，仅 `summary-artifact.ts` 有失效标记）。
- 索引只有一条：`idx_msg_session`（`bridge-ledger.ts:106`）。

**增长量级估算**（待 E-9/E-11 的真实行数校准）：

| 表 | 行数来源 | 单行宽度（估） | 1 万会话 / 100 万消息 / 10 万块 | 是否有界 |
|---|---|---|---|---|
| `message_ledger` | 每条被桥接的消息 | ~200 B（key+locator+hash 均 64 hex） | ≈ 200 MB | 与"已折叠消息数"同阶，**单调增**（消息删除不回收） |
| `block_ledger` | 每个曾出现过的块 | ~400 B | ≈ 40 MB | **单调增**（块物理删除后行仍在——这正是 D-8 的价值所在） |
| `run_ledger` | 每次 run | ~200 B | 15 min 一次 → 35k 行/年 ≈ 7 MB/年；1 min 一次 → 525k 行/年 ≈ 100 MB/年 | **无界且无价值**（R-16 的孤儿行还会叠加） |
| `summary_artifact` | v0.1 不写 | — | 0 | — |

**控制策略（建议 v0.2 落地）**：

| # | 措施 | 说明 |
|---|---|---|
| S-1 | `journal_mode=WAL` + `busy_timeout=5000` + `synchronous=NORMAL` | 与源侧同款（`schema.ts:90-92`）；当前实现连并发都不支持（Q10） |
| S-2 | `run_ledger` 保留 90 天 + 每次 run 结束时批量删旧行 | **唯一可以安全裁剪的表**（审计价值随时间指数下降） |
| S-3 | `message_ledger` 分两层：`active`（块仍存在）与 `archived`（块已消失）；只对 `archived` 且已撤销的行做 365 天保留 | 保留"撤销能力"的最短必要期 |
| S-4 | `block_ledger` **不裁剪** | 它是"曾经存在过哪些块"的唯一记录；删掉就永久失去 D-8 的可见性 |
| S-5 | 定期 `VACUUM`（写自己的 ledger 是允许的）+ 监控文件大小 | 注意：`auto_vacuum=INCREMENTAL` 是源侧的设置（`schema.ts:99`），ledger 需要自己设 |
| S-6 | 硬配额 + 告警 | 例如 ledger > 2 GiB 时告警，> 5 GiB 时停止写入（fail-open 到"停止对账"） |

**上限结论**：只要 S-2/S-6 落地，ledger 的增长是 **O(已桥接消息数)** 且有硬上限；不落地则**无界**。

### Q10 多实例部署怎么办

**结论：v0.2 **必须**硬性单实例（启动即抢独占锁，抢不到就退出告警）；v1.0 才谈分片。**

**为什么现在不能多实例（三条具体障碍）**：

1. **ledger 没有并发保护**：`new DatabaseSync(dbPath)` 无 `busy_timeout`（`bridge-ledger.ts:59-63`）→ 两个进程写同一个 ledger 会**立刻**抛 `SQLITE_BUSY`（不是排队），表现为随机失败。
2. **没有实例身份**：台账里没有 `worker_id`，无法回答"这条证据是哪台机器写的"（`run_ledger` 只有 `run_id`，`bridge-ledger.ts:93-104`）。
3. **桥接键是全局的**：`bridge_key = sha256(session|block|covered_hash)`（`bridge-ledger.ts:272-274`）→ 两个实例对同一个块做同样的写入，会靠 `bridge_key` 主键冲突互相踩（谁先提交谁赢，另一个报错）。

**上游对多实例的态度可以参考**：上游**支持**多进程共享同一 `index.db`（`#2671`/`#394` 拓扑），并为此做了三件事：`BEGIN IMMEDIATE`（`driver.ts:112-117`）、`busy_timeout=5000`（`schema.ts:92`）、以及"检测到另一进程在写就拒绝 full VACUUM"（`store.ts:2139-2155`）。**CMB 一条都没做**。所以"上游能多实例"≠"CMB 能多实例"。

**分级方案**：

| 阶段 | 方案 | 细节 |
|---|---|---|
| v0.2 | **单实例硬锁** | 启动时 `open(ledger.lock, 'wx')` 写 pid+started_at；失败则退出码 3 + 告警。等价于上游的 `.lock` 纪律（`store.ts:645-683`），但用我们自己的文件 |
| v0.3 | **分片**（推荐方向） | 每个 `(tenant_id, bot_id)` 或每个 `host` 一个 ledger 文件 + 一个实例；分片键写进 `source_instance_id`。天然无冲突，且故障域更小 |
| v1.0 | 共享库 + 行级租约 | `BEGIN IMMEDIATE` + `busy_timeout` + `worker_id` 列 + 块级租约（`claimed_by/claimed_at`）。复杂度高，仅在分片不够时采用 |

**若结论错误的代价**：若 v0.2 就允许多实例（例如用户在两台机器上同时装），最坏情况是**台账损坏/证据重复**，且因为 `bridge_key` 冲突，表现是"随机的一些块永远 COMMITTED 不了"——极难诊断。

---

## 4. 十个问题的结论汇总表

| Q | 结论一句话 | 对路线选择的影响 |
|---|---|---|
| Q1 实时性 | 不需要秒级；新增 T+15 min，撤销 T+1 h | 削弱 A，**不削弱 D** |
| Q2 读侧对账 | 必须；同步写方向错误 | 排除 B |
| Q3 游标 | 不能用 `last_ingested_seq`；用台账 cursor + 块集差分 | A/D 都要做，成本相同 |
| Q4 变化检测 | `data_version` > mtime；权威判据是台账 diff | A 略优（D 只能用副本 mtime） |
| Q5 重建 | 必须建 `source_instance_id`，重建=重扫+旧账标 STALE | A/D 都要做 |
| Q6 缺口 | 三类全显式，不猜不补 | 与路线无关 |
| Q7 不影响热路径 | 物理隔离 + 白名单 + 限流 + 禁区 | 强化 D |
| Q8 故障隔离 | CMB 必须 fail-open 到"停对账" | **强化 D**（P2/P3 传导路径在 D 下不存在） |
| Q9 存储增长 | 会无限增长；需 S-1..S-6 | 与路线无关（但 D 多了副本磁盘） |
| Q10 多实例 | v0.2 硬单实例；v1.0 分片 | 与路线无关 |

---

## 5. 推荐路线

### 5.1 推荐：**路线 D（影子副本对账）作为 v0.2 起点，路线 A 作为 v0.3 的条件升级**

**分三阶段落地**：

| 阶段 | 形态 | 数据入口 | 前置条件 |
|---|---|---|---|
| **v0.2**（真实数据只读验证期） | 路线 D 的**最小形态**：手动产一份副本 → 跑一次 CMB → 出报告 | `E:\[PROJECT]\_exp\snap\` 下的副本 | 用户提供副本（A-1）+ 副本版本号（A-6） |
| **v0.3**（沙盒端到端） | 路线 D 的**完整形态**：定时快照 + 增量对账 + 一次性记忆库 sink | 快照目录（保留 3 份） | E-8 结论 + D-9 裁决 + D-11 换键 |
| **v0.4+** | **条件升级到 A**：若 `E-3 T3` 与 `E-10` 同时通过（CANTOPEN=0、BUSY<1%、p95 增幅<10%），则允许"低峰期活库只读直连 + 常时副本"的混合形态 | 活库（低峰）/ 副本（常时） | E-3、E-10、D-10 授权、单实例锁、限流落地 |

**注意 v0.4 的升级是"可选优化"，不是"必然演进"**：如果副本周期的 15 min 延迟已被接受（Q1），那么升级 A 的唯一收益是"少一次备份"和"少 30 GiB 磁盘"。**收益不足以抵消权限边界从"备份目录"扩大到"全库明文"** —— 除非用户明确要求更短延迟。

### 5.2 数据入口（快照怎么来）的三个选项与推荐顺序

| 序 | 方案 | 执行者 | 是否写源库 | 结论 |
|---|---|---|---|---|
| 1 | **上游原生 `VACUUM INTO`**（`store.ts:694-709` 的 `backupInto`；注释明确"safe while WAL writers are active…never blocks them"、"refuses an existing target"） | 上游/运维（有写权限的角色） | **是**（在 writer 连接上执行，但这是**上游自己的**操作，不是 CMB） | **推荐**。语义由上游保证，CMB 零参与 |
| 2 | `node:sqlite` 的 `backup()`（从只读连接产出副本） | 运维脚本（只读权限即可） | 否（读锁） | **次选**，需先验证 `E-3 T2b`（U-1） |
| 3 | 停写窗口冷拷贝 | 运维 | 否（但要停服务） | **兜底**，用于应急/交叉验证 |

**明确排除**：裸拷 `db+wal+shm`（torn 快照，实验计划 §1.3 B-4）、`immutable=1`（假定不变，会读到撕裂页，B-5）。

### 5.3 副本生命周期（路线 D 必须回答的运维问题）

| 项 | 建议 |
|---|---|
| 产出频率 | 15 min（与 Q1 的 T+15 min 对齐）；低峰可加密，高峰期可降频 |
| 保留份数 | 3 份（当前 + 上一份用于差分 + 一份周度快照用于回溯） |
| 磁盘预算 | `3 × 库大小`。10 GiB 库 → 30 GiB。**必须在 v0.3 前确认磁盘余量** |
| 命名 | `index-<UTC ISO8601 紧凑>-<user_version>.db`（版本进文件名，防 F-01 类的"版本错配"） |
| 元数据 | 同名 `.meta.json`：`{produced_at, method(VACUUM_INTO|backup|cold_copy), user_version, size, sha256, sessions, messages, blocks}` |
| 删除 | 超过 3 份的最旧一份；或 30 天前的全部；删除前确认 CMB 游标已越过它 |
| 完整性 | 每次产出后跑一次**只读**的 `PRAGMA quick_check`（在副本上跑是安全的；**绝不**在活库上跑） |

---

## 6. 推荐依据（六条，按权重排序）

| # | 依据 | 证据 |
|---|---|---|
| 1 | **权限边界从"全库明文"收缩到"一份备份"** | 路线 A 要求 CMB 运行用户能读 `index.db`（明文：所有会话正文、`metadata_json`、`kstate_json`）；路线 D 只要求读快照目录。`index.db` 明确是 cleartext（`persist.ts:562-563` 的注释："index.db holds the ingest payload in cleartext"） |
| 2 | **R-18 从"架构前提"降级为"优化选项"** | `R-18`/`E-3` 至今 UNVERIFIED（`RISK-REGISTER.md:302-309`）。选 A 等于把整个项目押在一个未验证项上；选 D 则即使 R-18 永远为假，项目仍可运行 |
| 3 | **故障传导路径被物理切断** | §Q8 的 P2/P3（`.lock` 覆盖、WAL checkpoint 阻塞）在 D 下结构性不存在；上游刚为"写锁饥饿"付过 8 小时代价（`schema.ts:369-389` 的 `#4796`） |
| 4 | **实时性不是需求（Q1）** | 下游是 `candidate → pending_review → active` 的人工/审核流（`state-machines.md:14-16`），且 `pending_review → active` 只能由 M04 写入（同文 `:15`）。秒级对账不会让记忆更快被确认 |
| 5 | **与原型现状的差距相同** | A 与 D 都需要补：`--config`（R-17）、增量游标、限流、ledger 的 WAL/busy_timeout、单实例锁。**选 D 不额外付出实现成本，却买断两项最高权重** |
| 6 | **快照产出机制是上游自己的、且被文档承诺"不阻塞写者"** | `store.ts:694-709` 的注释原文 |

### 6.1 什么情况下应该改选 A / C / B

| 触发条件 | 应改的路线 | 理由 |
|---|---|---|
| 用户要求"删除后 5 分钟内必须从记忆库消失"且快照周期无法缩短到 5 min | **A**（低峰直连）或 **C** | 只有直连/事件才能做到分钟级以下 |
| 用户要求"用户发完 5 秒内可检索" | **C 或 B** | 只有事件/内嵌能做到秒级 |
| 上游主动提供稳定的"提交后只读事件出口" | **C** | 事件层的成本被上游承担后，C 的复杂度大幅下降 |
| 磁盘余量 < 2× 库大小 | **A** | 副本方案在磁盘上不可行 |
| `E-3 T3` 证明活库只读直连零风险且 `E-10` 证明对上游零影响 | **A 可作 v0.4 升级** | 此时 A 的唯一缺点（R-18 风险）已被实证消除 |
| 用户要求 CMB 与 bot 部署在**同一台机器且无额外磁盘** | **A** | D 的磁盘成本不可接受 |

---

## 7. 明确不推荐的做法（反例清单）

| 做法 | 为什么不推荐 | 依据 |
|---|---|---|
| 让 CMB 用**可写连接**打开源库（哪怕"只 SELECT"） | 会写 `<db>.lock`、会跑 `migrateSchema`（`ALTER TABLE`）、会触发 `wal_checkpoint` | `schema.ts:283-405`、`store.ts:582-586`、`store.ts:645-683` |
| 用 `bili export` / `bili sessions ls` 作为数据入口 | 它们走 `UnifiedStore.open()`（含 quick_check + 迁移 + checkpoint），**不是只读路径** | `export.ts:437-446`、`storage-maintenance.ts:86-117` |
| 在活库上跑 `PRAGMA quick_check` 作为"体检" | 14 GiB ≈ 260 s 全页扫（`store.ts:455`），与上游争 IO，且上游已在自己的 cadence 上做 | `store.ts:474`、`integrity-check.ts:3,43` |
| 把 CMB 写成守护进程常驻（路线 C 的副产品） | 原型定位是"无常驻进程、无后台服务"（`README.md:217`）；常驻会引入自愈/守护/资源限制全套问题 | `README.md:217-226` |
| 用 `fs.watch` 监听 index.db 做事件驱动 | Windows 上 WAL 每次提交都触发（`journal_mode=WAL`），事件风暴 | `schema.ts:90` |
| 用 `sessions.last_ingested_seq` 做桥接游标 | 每次启动被清空（`store.ts:1497-1512`）、语义是 writer 信任前缀（`store.ts:1456-1467`） | 同左 |
| 多实例共享一个 ledger | ledger 无 WAL、无 `busy_timeout`、无 `worker_id` | `bridge-ledger.ts:59-63,93-104` |
| 在压缩热路径同步写 Evidence | 违反 append-only + 单事务原则；上游为写锁饥饿付过 8 小时代价 | `schema.ts:6-9`、`store.ts:811-819`、`schema.ts:369-389` |
| 用"文本级去重"抹掉重建后的重复 Evidence | 违反"同文本不同消息不合并"（TC-I-04） | `RISK-REGISTER.md:235-237`（R-10） |
| 把 `VACUUM INTO` 写进 CMB 的代码 | 它是写操作，必须由上游/运维角色执行 | `store.ts:706` |

---

## 8. 与实验计划的映射（哪张卡决定哪条结论）

| 架构结论 | 决定性实验 | 若实验失败则 |
|---|---|---|
| 路线 D 的副本可用 | E-1、E-2 | 项目命题崩塌（原话不可恢复），架构无意义 |
| 副本产出方法（选项 1/2/3） | E-3 T2b（`backup()` 可行性） | 降级到停写窗口冷拷贝 |
| 是否可升级到 A（v0.4） | E-3 T3 + E-10 | 永不升级，D 为终态 |
| 15 min 周期是否足够 | Q1 的用户书面确认（非实验） + E-10 的"长期开启"结论 | 缩短到 5 min 或改 A/C |
| `source_instance_id` 的必要性 | E-5（id 重铸）+ E-7（块消失） | 若 id 绝对稳定且块不消失，可简化（但不应假设） |
| ledger 容量估算 | E-9、E-11 的真实行数/行宽 | 调整 S-2/S-3 保留期 |
| 多实例方案 | 无实验（纯设计） | — |

---

## 9. 本文件的未验证项

| # | 未验证 | 出处 |
|---|---|---|
| V-1 | 跨进程只读打开 WAL 活库是否可行（`-shm` 依赖） | 实验计划 U-2 |
| V-2 | `sqlite.backup()` 是否接受只读源连接 | 实验计划 U-1 |
| V-3 | 副本产出的实际耗时与磁盘峰值（10 GiB 级库的 `VACUUM INTO` 时长） | 未测；v0.2 的运维前置 |
| V-4 | 真实 `blocks` 每会话行数（决定"每轮拉全量块列表"的成本） | 实验计划 E-9 |
| V-5 | 真实消息行宽与 total 行数（决定 ledger 容量） | 实验计划 E-9/E-11 |
| V-6 | 上游是否有可用的写延迟指标（决定 E-10 能否量化到"延迟"层） | 实验计划 U-7 |

---

> 本文件结束。**未实现任何路线，未连接任何数据库，未修改上游与原型。**
