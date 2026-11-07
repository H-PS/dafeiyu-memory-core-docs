# 21 · reset 事务与并发设计压力审查（针对 20 号 T2-1～T2-8）

> 基线：`D:\v3-memory-v2-design-final-decisions-r1\`（R1，11 份）。
> 依据：19 号（源码事实）、20 号（修订建议）；本轮补充只读定位：`08` §三/§四/§五（锁与迁移）、`05` §二（commit 六阶段）、`10`（文件计划）、`bridge.js` 的 `sessionEpoch`/`isCurrentSession` 分布。
> 边界：只读 + 本报告；未修改任何文档/源码/配置/state/Persona/插件；未迁移、未业务写入、未清理、未启停服务。
> **审查通过 ≠ 实施授权。**

---

## 〇、总结论

| 对象 | 结论 |
|---|---|
| T2-1～T2-8 整体 | **PASS WITH CONDITIONS**（无 BLOCKED 级问题；5 项新约束并入下一轮修订后可实现） |
| 竞态"T2-A 五步曲" | **可能发生**（在途请求跨 phase 边界时）——需三层防护，代码中已有同型 fencing 先例（`ensureSession` 的 epoch 复验，`:8462`/`:8478`），设计需将其成文化并扩展 |
| 最大的三个缺口 | ① 构造-轮换竞态（prompt 带旧 token 被 DSH 接受）② `getSocialV2State` 复活向量 ③ 无后续唤醒时待交付 token 无兜底 |
| 意外的计划缺口 | `10` 的文件计划**没有** resetTx / rotation-journal 的行（M01 的 journal 只覆盖 A 层记忆）——事务机制是净新增，必须补 M07 行或并入 M06 估算 |

---

## 一、已确认的设计完整性（哪些不用改）

| 项 | 现状 | 判定 |
|---|---|---|
| 失效时点 | `04` §5.2-1 + 20 号 T2-3（绑定 `tokenRotated` phase） | ✅ 完整 |
| 旧 token 禁止复活 | `04` §6.2-5 + 迁移后 S 层物理无字段（20 号 T1-5） | ✅ 完整（双保险） |
| 五态交付状态机 | `04` §3.2（只有 `accepted.result.ok === true` 清位；queued 不算交付） | ✅ 规则保持，需补两个边界（见 F-1/F-3） |
| per-key 锁原语 | `08` §3.2（`:91-103`：进程内 Promise 链、按 key、非文件锁、前错续跑） | ✅ 原语可复用，覆盖范围需扩展（见 F-5） |
| fencing 先例 | `05` §2.3 `expectEpoch`（A 层）；`08` §3.4（S 层 `stateRev`）；**代码同型**：`ensureSession` 捕获 `sessionEpoch`（`:8462`）并在 await 后复验（`:8478`/`:8530`/`:8540`） | ✅ 设计与代码先例齐备，缺"推广到工具端点"的规定 |
| 锁序 | `08` :197（token → memory → ledger） | ✅ 与 credential 独立锁（`:195`）一致 |
| credential.rotate 幂等 | `04` §4.4（opId 幂等、epoch 只增一次、journal 不存 token） | ✅ 是恢复矩阵成立的基础 |
| DSH 旧会话 403 = 预期 | 代码注释 `:4596-4598` + 20 号 T2-5 | ✅（需按 20 号成文为验收断言） |
| 投递串行 | `deliverPrompt` 每 key 串行队列（`:9573` 注释"保证 turn 完成顺序"） | ✅ 同 key 并发投递问题不存在（见 §四 T2-C-2） |

## 二、发现的竞态与恢复漏洞（7 项，编号 F-1～F-7）

### F-1 构造-轮换竞态（T2-C 核心）
prompt 在 `deliverPromptNow` 内构造（token 已拼入字符串），之后才 `await api.sessions.prompt`（`:9543`）。若构造与接受之间发生 rotate：**DSH 会"接受"一条携带旧 token 的 prompt**（`api.sessions.prompt` 对内容不做校验）→ turn 白跑，模型拿旧 token 调工具全 403，且 `qq_get_prompt` 无法取新 token（19 号已确认该端点不携带 token）→ 本轮产出为纯浪费，`needsDelivery` 仍为 true。
**修复约束**：构造时记录 `constructedEpoch`；`markAccepted` 仅当 `constructedEpoch === credential.get(key).epoch`；不匹配 → 保持 `needsDelivery=true`、记 `stale_delivery` 审计、下一轮重投。现有五态机加一条边：`accepted(stale)` → 不清位。

### F-2 `getSocialV2State` 复活向量（T2-A 核心）
`getSocialV2State`（`:8764`）在会话不存在时会**创建新状态**（含新生成 token，`:8782`，并 `conversations.set`，`:8804`）。reset 清理后，任何在途请求再次调用它 → **复活一个全新会话**（新 token 无模型知晓，且污染刚清空的 S 层）。
**修复约束**（三层，缺一不可）：
1. **入口屏障**：`tokenRotated` 先于 `conversationsCleared`——轮换后旧 token 的新请求在中间件（`:3303-3306`）即被 403，永远到不了状态层；
2. **epoch 复验**：凡"await 之后才写状态"的处理器，写前复验 `sessionEpoch`/会话身份——代码已有同型（`ensureSession :8478`），需推广为工具端点统一助手（如 `assertNotReset(key, capturedEpoch)`）；
3. **对象置换**：reset 清理时**置换** `conversations` 中的对象（set 新对象）而非原地清空字段——在途请求持有的旧 `st` 引用变成孤儿，其对孤儿的一切写入对快照器不可见（快照器只遍历当前 map）。

### F-3 用户画出的五步竞态——逐步判定

```
reset prepare            → 仅写 journal pending，无任何状态变更
旧 token 请求进入        → prepare 窗口内：合法通过（此时无东西可破坏，其写入会被随后的清理覆盖）
                           tokenRotated 之后：在中间件被 403（入口屏障）
reset 清理 conversations → per-key 锁内 + 对象置换；在途请求持有的旧引用被孤立
旧请求写回状态           → ①同步写孤儿对象 → 快照不可见（无害）
                           ②写前调 getSocialV2State → 若无 epoch 复验则复活（F-2，必须堵）
                           ③经锁 → 排在 reset 之后，写的是新对象 → 需 epoch 复验决定放行/放弃
reset 完成               → stateFileRewritten 输出当前 map（孤儿写入天然排除）
```
**结论：竞态可能发生（仅在"已过认证的在途请求 + 写前复活"组合下）；三层防护（入口屏障 / epoch 复验 / 对象置换）可完全阻断，其中 epoch 复验是唯一必需层，另两层是纵深。**

### F-4 reset 恢复期间旧会话继续修改状态
崩溃恢复（续跑）发生在 `main()` 启动序列内：`acquireLock → loadState → loadSocialV2State（:9275）→ 恢复扫描 → 服务器开始监听`。**约束**：恢复必须在监听之前完成——启动期不存在并发请求，窗口天然关闭。需在 `04` §5.5 明文规定"resetTx 恢复挂启动序列、先于 HTTP 监听"。

### F-5 锁内无界等待
`withKeyLock` 是 Promise 链，无超时。若 resetTx 在锁内 `await stopSessionWork`（DSH 控制超时可达秒级~超时上限），同 key 的普通请求被长时间压住。
**修复约束**：**锁内只做内存变更 + credential 原子操作（无网络）**；`stopSessionWork`/`archiveSession` 移到锁外（best-effort，失败仅告警——rotate 才是正确性屏障，stop 只是省 token 的优化）。这样锁临界区恒有界，无需引入超时机制。

### F-6 无后续唤醒的待交付 token
`needsDelivery=true` 只在"下一次该 key 的 prompt"才能清位。私聊/潜水会话可能长期无唤醒 → 令牌长期未送达（现仅有 `deliveryWarnAfterTurns:3` 告警，turns=0 时不触发）。
**修复约束**：兜底投递 = ①启动清扫（恢复完成后扫描 `needsDelivery=true` 的 key）②周期兜底（挂 `reconcileSessionPolicies` 的既有 tick，对超阈 key 投递一条最小 prompt——仅身份行+令牌行+一条"保持沉默"指令）。二者都不新增独立线程。

### F-7 journal 与 credential 双记账的对账规则
rotate 写 credential 文件（opId/epoch）与写 journal 是两次落盘，中间可崩。
**修复约束**：**以 credential 文件为"事实权威"**——恢复扫描发现 journal 落后于 credential 时，按 credential 补记 phase 后继续；journal 领先（不可能，journal 先行）则按 journal 重放。journal 文件本身损坏 → 告警 + 以 credential 为准重建剩余 phase；**credential 与 journal 同时损坏** → 走 `04` §6.2-4 的 corruptRecovery 全量重生成（人工确认级）。

---

## 三、两个事务模型比较（T2-B 要求）

### 模型 A：per-key 独立事务 + 批次协调记录（推荐）

```
POST /api/socialV2/reset
  → 建 batchCoordinator {batchId, keyList[]}（仅记录，不是锁）
  → 逐 key：独立 resetTx(opId=op_<batchId>_<key>)，per-key 锁内完成
      prepare(journal) → tokenRotated(credential) → conversationsCleared(置换)
      → stateFileRewritten(全量快照，天然排除已清 key)
  → 全部完成 → 响应 200；部分失败 → 响应 207+失败 key 列表，剩余由恢复续跑
```
- **恢复复杂度**：低。每个 key 独立幂等，崩溃点即续跑点；batchCoordinator 只用于"哪些 key 还没跑"的展示与重启续扫。
- **一致性**：单 key 内强（锁 + journal + credential 对账）；批次间最终一致（重启续跑到收敛）。
- **实现成本**：中。resetTx 原语一套，批次只是循环 + 汇总。
- **代价**：中途崩溃的中间态是"部分 key 已清"——与现状"要么全空要么全不变"不同，**属有意的行为变更，需写入文档**。

### 模型 B：整批单事务

```
一个 resetTx 持有全部 key 的锁（或全局锁），journal 记 keyList，逐 key 执行 effects，
最后一次性 stateFileRewritten；任一 key 失败 → 整批回滚（token 不轮换）
```
- **恢复复杂度**：高。需要"整批回滚"语义——但 credential rotate 不可回滚（旧 token 已失效，且 04 §6.2-5 禁止写回），**回滚点只能放在 rotate 之前**，意味着"任一 key 的预检失败就全批不做"；执行中途崩溃则整批挂起，恢复要重放 N 个 key。
- **一致性**：看起来"全或无"，但 rotate 不可逆使真正的原子性根本做不到——伪原子。
- **实现成本**：高。需要嵌套 phase、批量回滚逻辑、全局锁（压死所有 key 的并发）。

### 判定与理由

**采用模型 A**。三个理由：
1. **rotate 不可逆**（旧 token 立即失效是 D2 的安全属性）→ 模型 B 的"整批回滚"在 rotate 之后是谎言，在 rotate 之前则迫使预检承担全部正确性——收益为零复杂度高；
2. 单 key 粒度与既有 `withKeyLock`/`expectEpoch`/`sessionEpoch` 基础设施**同构**，实现是"循环 + 汇总"，不是新机制；
3. 批次收敛性由 journal 恢复保证（重启续跑），最终状态与一次成功等价——"部分完成后收敛"比"伪原子"更诚实、更可测（`RESET-05`：批量中途崩溃 → 重启续跑 → 终态=全清）。

**三条路径的边界**：
| 路径 | 事务形态 | 清理范围 | 备注 |
|---|---|---|---|
| `/api/socialV2/reset` | 批次协调 + N 个 per-key resetTx | state.sessions / sessionPolicies / conversations / 脱敏索引 / seenForwardIds / S 文件整键移除 | 批次内 key 之间相互独立 |
| `/api/session/reset` | 单 key resetTx | 同上（单 key）+ DSH stop/archive | 现有 `:7389-7443` 流程的事务化 |
| QQ 命令 `/reset` | 单 key resetTx | 同 B 路径 | `:11308-11317` 事务化；额外的 QQ 回执消息在 done 之后发送 |
| `workspace/reset` | 独立批次事务（同模型 A） | 上述全部 key + `CONVERSATION_TOKENS`/`KNOWN_AGENT_TOKENS` 清表 + DSH 全量 stop/archive | **需要独立批次事务**：它的 keyList 来自 `state.sessions` 枚举而非请求体，且必须先于清表完成 DSH 侧枚举（`:7452`）；不与 socialV2/reset 共用批次实例（语义不同：一个是"全部"，一个是"请求指定的全部现存"） |

**不假设三条路径语义相同**：A 是"批量清空 S 层"，B/C 是"单 key 重开上下文"，workspace 是"全量退役"——三者共享 resetTx 原语与 credential.rotate，但 phase 集合与副作用不同（B/C 有 stop/archive，A 无 per-key stop 但有 drainAllPromptQueues）。此差异表必须进 `04` §5.5。

## 四、T2-C：新 token 交付状态机（逐问）

| 问题 | 结论 |
|---|---|
| `needsDelivery` 创建时点 | `credential.rotate()` 内随新 epoch 写入（`04` §4.4），即 resetTx 的 `tokenRotated` phase 提交时——**不是** reset 请求进入时 |
| 三构造器竞争 | **不存在并发竞争**：同 key 投递被 `deliverPrompt` 串行队列（`:9573`）序列化；notify-ai 与唤醒分别入队，先后到达 DSH |
| 同一 token 重复投递 | 可能且无害：重试/重连/兜底都会再发同一 epoch 的 token；DSH 接受是幂等的，重复投递不改变状态机 |
| 接受成功但清位前崩溃 | `markAccepted` 未落盘 → 重启后 `needsDelivery` 仍 true → 下一轮重投同一 token → 再次 accepted → 自愈。**无需特殊恢复**（写入恢复矩阵） |
| 投递失败/排队 | `queued`/`failed` 均保持 `needsDelivery=true`；离线队列在内存，崩溃即失 → 靠 F-6 兜底重投 |
| 没有后续唤醒 | F-6 兜底：启动清扫 + reconcile tick 周期兜底；`credential_delivery_stalled` 告警条件从"连续 3 turns"扩为"3 turns **或** 超 N 小时无投递机会" |
| 三种 reset 的交付语义差异 | **reset/retireSession**：key 存活，新 epoch 需要投递（`needsDelivery=true`）；**workspace/reset**：key 整体移除（`retireAll`），**无投递语义**（没有收件人）——`04` §3 需补一行注明此区别 |
| 保持的既定规则 | 只有 `accepted.result.ok === true` 清位；queued 不算交付；`constructed` 不算交付——**全部保持，且新增 `accepted(stale)` 不清位**（F-1） |

## 五、事务恢复矩阵（T2-D）

前置约定：journal 先行（write-ahead）；credential 文件自身记录 opId/epoch（事实权威）；所有 effects 幂等；恢复挂启动序列、先于 HTTP 监听（F-4）。

| # | 崩溃位置 | 磁盘状态 | credential 状态 | DSH 旧会话状态 | 重启后动作 | 分类 |
|---|---|---|---|---|---|---|
| 1 | prepare：journal pending 写失败 | 全部不变 | 未轮换 | 运行中，旧 token 有效 | reset 请求以错误返回，零副作用 | **可重试**（重新发起） |
| 2 | prepare 完成后、rotate 前 | journal=pending | 未轮换 | 运行中 | 恢复扫描：pending 无已执行 effect → 标 aborted 或整体重放（幂等） | 可重试 |
| 3 | tokenRotated 后崩溃 | social-v2 未清；journal=pending+phase1 | **已轮换**（新 epoch，needsDelivery=true） | 运行中，**工具调用全 403** | 续跑 conversationsCleared → stateFileRewritten → done；stop/archive 补做（锁外 best-effort） | 可重试（自动续跑） |
| 4 | conversationsCleared 后崩溃 | 磁盘仍是旧内容（快照未写） | 已轮换 | 同上 | 续跑 stateFileRewritten | 可重试 |
| 5 | stateFileRewritten 后、done 前 | 磁盘已无该 key | 已轮换 | stop/archive 可能未完成 | 补 stop/archive；标 done；`needsDelivery` 由后续投递清位 | 可重试 |
| 6 | done 之后重复恢复 | 终态 | 终态 | 已停止/归档 | journal done → 跳过 | 幂等，无操作 |
| 7 | journal 写失败（rotate 已提交后） | — | **已轮换（权威）** | 403 | **对账规则（F-7）**：以 credential 为准补记 journal，继续剩余 phase；journal 持续不可写 → 控制台告警 | 可重试（对账后） |
| 8 | 文件写入成功但 phase 未提交 | 磁盘已清 | 已轮换 | 同 #5 | 重放 stateFileRewritten（全量快照幂等）→ 标 done | 可重试 |
| 9 | DSH stopSessionWork 失败/超时 | — | — | 仍在跑，工具调用 403 | 非致命：rotate 已保证认证切断；审计 warn；人工在 DSH 侧确认归档 | 可重试（stop/archive 幂等）；长期未归档 → 人工 |
| 10 | 重复恢复（同一 opId 重放 N 次） | — | epoch 只增一次（rotate 幂等，`04` §4.4） | — | 各 phase 幂等 → 结果不变 | 可重试 |
| 11 | 同一 reset 请求被重复提交 | — | 每次提交获得**新 opId** → 再轮换一代 | — | per-key 锁串行；第二次 reset 合法（D2 语义：每次 reset 都轮换）；文档注明"重复 reset=多次轮换"是预期 | 可重试 |
| 12 | journal 与 credential **同时**损坏 | — | 不可信 | — | 唯一出路：`04` §6.2-4 corruptRecovery 全量重生成（epoch+1、全 key）+ 控制台人工确认 | **人工介入** |
| 13 | journal 损坏但 credential 完好 | — | 权威 | — | 以 credential 重建 journal 剩余 phase + 告警 `journal_rebuilt` | 可重试（对账） |

**不可重试类**：无（唯一"人工"项是双真相同时损坏，已有 corruptRecovery 兜底）。**fail-closed 原则**：journal 不可写时 resetTx 不得开始任何 effect（#1 的推广）。

## 六、对 20 号 T2-1～T2-8 的逐项结论

| 项 | 结论 | 条件 / 本轮新增约束 |
|---|---|---|
| T2-1 三路径全覆盖 | **PASS WITH CONDITIONS** | 条件：批量维度（模型 A）与 workspace/reset 独立批次事务成文（§三边界表） |
| T2-2 直写点 + 半恢复 | **PASS WITH CONDITIONS** | 条件：`:4624` effect 化之外，补 F-2 三层防护与 F-4 恢复时序 |
| T2-3 失效时点 | **PASS** | 无 |
| T2-4 禁止复活 | **PASS** | 无（物理 + 行为双保险已足） |
| T2-5 旧会话认证处理 | **PASS WITH CONDITIONS** | 条件：明文规定"stopSessionWork 是优化、credential.rotate 才是正确性屏障"（支撑锁外移，F-5） |
| T2-6 新 token 交付 | **PASS WITH CONDITIONS** | 条件：F-1 构造-轮换竞态处理（epoch-at-construct + stale 不清位）+ F-6 兜底投递 |
| T2-7 新建模块标注 | **PASS WITH CONDITIONS** | 条件：`10` 文件计划补 resetTx/rotation-journal 行（净新增，估算行数与测试）——20 号 T2-7 只改了措辞，没补计划行 |
| T2-8 共享/不共享清单 | **PASS WITH CONDITIONS** | 条件：共享原语清单补 `credential.retireAll()`（workspace/reset 用），phase 不共享的第三实例（workspace） |

## 七、需要修订的文档和章节（在 20 号清单之上**增量**）

| 文档 | 章节 | 内容 |
|---|---|---|
| `04` | §3.2/§3.3 | F-1：`constructedEpoch` + `accepted(stale)` 不清位；§3.4 告警条件扩展 |
| `04` | §5.5（20 号新增的小节） | 模型 A 定稿、三路径/边界表、恢复矩阵、F-2 三层防护、F-4 恢复时序、F-5 锁内禁网络、F-6 兜底投递、F-7 对账规则 |
| `04` | §3 末尾 | workspace/reset 无交付语义的一行区别 |
| `05` | §2.2 | 注明 resetTx 复用同一 journal 基础设施（`state/journal/`），phase 集不同 |
| `08` | §3.2 | withKeyLock 使用边界："锁内禁止网络等待"成文 |
| `10` | 文件计划表 | **新增 M07 行**：resetTx/rotation-journal 模块（journal 读写、phase 重放、对账），估算 ≈200 行 + 恢复演练用例（RESET-03/05） |
| `13` | §三 T-05 | 评审要点补"模型 A 批次语义 + 恢复矩阵 13 行" |
| `14` | §五 | 新增 `RESET-03/04/05`（已在 20 号）+ `CRED-09`（stale 不清位，20 号已列）+ `CRED-11`（兜底投递触发） |

## 八、尚未解决的问题

| # | 问题 | 状态 |
|---|---|---|
| 1 | DSH 是否会"接受"构造于轮换前、投递于轮换后的 prompt 并照常执行 turn | 属 DSH 行为：从 bridge 侧证据（`:9543-9548`）看 `result.ok` 只表示收下，**无** token 校验——按"会执行"设计（F-1 的修复不依赖此答案，但答案影响浪费幅度）；列为 DSH 侧 UNCONFIRMED |
| 2 | 批量部分完成时控制台的呈现（207 响应体形态） | 产品细节，实施前定 |
| 3 | F-6 兜底投递的最小 prompt 文案（需不含任何记忆/正文） | 文案级，实施前定 |
| 4 | D-30（阻塞 M2）仍未决 | 维持 |

## 九、最终分级

> **PASS WITH CONDITIONS** —— reset 事务/并发/交付的骨架（rotate 幂等 + journal 先行 + per-key 锁 + epoch fencing + 五态机）在 20 号修订后已经完整且与代码先例同构；本轮发现的 7 项缺口（F-1～F-7）全部可用"补约束"解决，无一项需要推翻方案，也不存在 BLOCKED 级问题。**条件**：§七 的增量修订并入下一轮定点修订，且 V-7/V-8 门禁与 D-30 决策按既定节奏关闭。
>
> 事实核查、设计审查、设计通过均不构成源码实施授权。本轮仅新增本报告，未修改任何现有文档与源码；交付后停止，等待人工审核。
