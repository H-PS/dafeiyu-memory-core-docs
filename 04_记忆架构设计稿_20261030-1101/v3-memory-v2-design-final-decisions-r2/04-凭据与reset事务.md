# 04 · R2 凭据与 reset 事务架构

> 合并 19–23 号关于凭据生命周期、epoch fencing、resetTx、幂等、批次、恢复、双损的全部结论。本文件是 reset/credential 的**权威设计**。

---

## 一、凭据生命周期（token 全景）

### 1.1 事实基线（`CONFIRMED`，19 号 §二）

| 类别 | 位置 | 说明 |
|---|---|---|
| 写点（5 类） | init `:8782`、rotate `:2795`、载入修复 `:8834-8851`、快照 `:8945`、脱敏索引维护 `:540/:561` | token 出 S 后全部收敛到 `credential.js` |
| 读取（8 处） | 认证 `agentTokenOk :3146-3150`（约 41 端点）/ `isLiveAgentToken :3157-3165`+中间件 `:3303-3306` / 注入 `:4363`/`:10674`/`:10751` / 唤醒守卫 `:9526`/`:9556-9557` / 快照 `:8945` | 函数内改造即覆盖全部端点 |
| 外部读者 | 无文件级读者；仅 7 个测试脚本经内存 harness | 迁移检查表 = 19 号 §2.3 的 13 组 |

### 1.2 改造规则

- `credential.js` 为**唯一生成者**；全仓 `st.agentToken` 赋值目标 0 处。
- 存量迁移（维护窗口、bridge 停止）：读出 S 层 token → 写 credential → 从快照器删除 `agentToken` 字段（`:8945` 行移除 + 载入修复分支删除）。
- 验收三断言：`st.agentToken` 赋值 0、快照文件无 token、`agent-token.json` 不在备份/导出产物。

## 二、epoch fencing（Issue 1 的完整异步变更协议）

`DESIGN RECOMMENDATION`（C-2，**Correctness**）：

```
1 捕获    在第一个相关 await 之前，捕获当前 sessionEpoch
2 异步    执行异步操作（await DSH / QQ / 媒体解析）
3 复验    用捕获的 epoch 复验（getSocialV2State(key, { expectEpoch })）
4 取对象  仅复验通过后才取当前会话对象
5 变更    只变更步骤 4 取到的、已验证的对象
6 禁止    绝不通过异步边界之前获取的旧对象引用写入
```

- **同步段**：mark-read 型处理器（auth 后无 await）依赖事件循环原子性，零改动。
- **异步段**：send-message 型处理器（auth 后含 await）按上述协议改造；清单由 C-9 的实施期枚举给出，**不假设全部已 fence**。

## 三、reset 事务（resetTx）

### 3.1 四条路径与清理差异（`CONFIRMED`，19 号 §四 + Issue F）

| 路径 | 端点/位置 | 清理范围 | 事务形态 |
|---|---|---|---|
| A | POST `/api/socialV2/reset` `:4587` | 全部现存 key + sessionPolicies + S 文件整键移除 + `:4624` 直写空 | **批次协调 + per-key session resetTx** |
| B | POST `/api/session/reset` `:7389` | 单 key + DSH stop/archive | **单 key session resetTx** |
| C | QQ 命令 `/reset` `:11308` | 单 key + DSH stop/archive + QQ 回执 | **单 key session resetTx + QQ 回执** |
| W | POST `/api/workspace/reset` `:7445` | 全部 key + 内存 token 表 clear + DSH 全量 | **独立 workspace 事务，尚未设计，不可激活** |

三条会话级路径（A/B/C）共享 `credential.rotate`，phase 集不同（B/C 有 stop/archive，A 无 per-key stop 但有 batch 级 drain）；**W 不套 session resetTx phase**（B-2），其事务语义未设计（发布阻断，见 §3.7）。

### 3.2 事务模型（采纳 21 号模型 A）

per-key 独立事务 + 批次协调（journal 内一行 intent）。理由：rotate 不可逆 → 整批回滚是伪原子；per-key 与既有 fencing/锁基础设施同构。

### 3.3 phase（`DESIGN RECOMMENDATION`，Correctness）

```
resetTx(key, opId, batchId):
  prepare           journal pending（先于任何 effect；写失败=零副作用）
  tokenRotated      credential.rotate(key, 'reset', opId)   ← 旧 token 失效时点
  conversationsCleared  对象置换（非原地清空）+ 脱敏索引 forget
  stateFileRewritten    全量重建快照（天然排除已清 key；消灭 :4624 直写）
  done              journal committed
```

**A-path drain 的位置（Issue C，证据 `bridge.js:4591`）**：
- `drainAllPromptQueues`（A-path）/ `drainPromptQueue`（B/C-path）在当前代码中**先于 conversations 清空**（`:4591` 在 `:4592` 循环前；`/api/session/reset :7415` 在清理前）。它是**预清理正确性动作**——防止"清空后队列里残留 prompt 被重投"。
- 事务化后归入 **prepare phase 之后、tokenRotated 之前的同步段**（与 prepare 同锁内、无 await）：`drain` 排空在途 prompt 队列，属于**正确性必需**（不是轮换后的 best-effort）。
- **drain 作用域与锁**（Issue C 细化）：
  - A-path（批量）：`drainAllPromptQueues` 在**批次级**执行一次（先于逐个 key 的 resetTx），属批次意图的**全局预清理**，不进 per-key 锁；
  - B/C-path（单 key）：`drainPromptQueue(key)` 在**该 key 的 resetTx** prepare 之后、tokenRotated 之前，进 per-key 锁。
  - **drain 必须先于任何 token 轮换成功**——drain 失败则中止该 key/批次的轮换（fail-closed，零副作用），因为"残留 prompt 用新 token 重投"会造成旧会话语义污染。
  - drain 是内存操作，幂等、崩溃后无待恢复状态（队列随进程消失）。
- **stop/archive 仍为 done 之后的锁外 best-effort**（rotate 才是认证切断的正确性屏障）；崩溃后可跳过，不设持久化重试（见恢复矩阵第 5 行）。

**drain 隔离不完整（Issue C，`CONFIRMED` 源码缺口）**：
- 现状 `drainPromptQueue`/`drainAllPromptQueues`（`:11183-11196`）**只清 `promptQueues`（per-key 投递队列），不清 `queued`（离线队列 `:9504`）**。
- `flushQueue`（`:2824`）在 DSH 恢复后重投 `queued` 里的旧 prompt，重投时 `modeAllowed` 只查白名单/模式（`:2841`），**无会话身份/epoch 校验** → drain 之后旧 prompt 仍可能经离线队列重投。
- **安全不变式**："drain 屏障跨过后，旧会话 prompt 不得新入队或派发到能用新轮换凭据的投递路径"——**现有机制无法保证**，标 `UNCONFIRMED`。
- **必需 admission barrier（设计缺口，激活阻断）**：需新增"会话身份/epoch 校验"屏障，使 drain 屏障后（A：批次 keys 冻结 + 全局拒新入队；B/C：per-key 拒新入队直到 rotate 完成）旧 prompt 无法进入投递。**该 barrier 未设计、未验证 → 受影响 reset 路径为激活阻断项**（见 PHASE-07/08）。
- 注意：drain 只清已入队消息，**不能取消已派发/在途消息**——在途消息须靠"投递时 isCurrentSession 校验"（现有 `:9539`）拒绝，此点须单独验证（PHASE-07）。

- 恢复挂启动序列、**先于 HTTP 监听**（F-4）。

**启动顺序缺口（Issue E，`CONFIRMED` 源码）**：
- 现状 `server.listen` 在 `main()` 内 `:7531`，而 `loadSocialV2State()` 在 `:9275`——**服务器先监听、后加载 social-v2 状态**；且 `flushQueue` 是异步 `setTimeout`（`:2596`/`:2878`/`:2887`）触发，**无显式机制保证 reset 恢复 + admission barrier 恢复先于离线队列 flush**。
- **必需启动依赖（未保证，UNCONFIRMED）**：未完成的 reset 事务及其 admission barrier 必须在任何 `flushQueue` 执行前恢复。现状无法证明此顺序 → 受影响 reset 路由须 fail-closed 直到该顺序被保证（PHASE-12）。
- 不得仅凭现有 `isCurrentSession`（`:9539`）宣称充分——它针对"存储的原始会话身份/epoch"的确切行为未证明。

### 3.4 批次意图（D-3/D-4）

- intent 追加写 journal（`{k:'batch', batchId, requestId, keys[]}`）**先于**首个 effect。
- `keys[]` 请求接受时冻结；批次中新建的会话被 sessionEpoch 屏障排除。
- 恢复 = `intent.keys` − done 差集前滚；**禁止重枚举**。

### 3.5 请求级幂等（D-2，Issue E 确定性重放）

- `requestId`（客户端可选 UUID）→ `batchId` → 结果。
- **同 requestId**：
  - 原 batch **未完成** → **续跑该 batch**（沿用原 batchId/opId，不建新 batch、不轮换）；
  - 原 batch **已终态** → 返回存储的终态结果（不轮换）；
  - **绝不**建新 batch、绝不再次轮换。
- **新 requestId**：新 batch + 有意新轮换（D2 语义）。
- **缺 requestId**：走 `01` D-2 非幂等默认（每次执行 = 新操作、新轮换）。
- opId = `batchId + key` 派生 → credential.rotate 幂等按 opId 无缝衔接。
- **保留规则（对齐 `01` D-8，Issue E）**：幂等索引与其关联结果作为**一个可重放单元**存续——**索引不得比其重放所需结果更长寿**；压缩保留：①未完成 batch（直到终态）②已完成结果的**时间下限（≥7 天）**③最近 50 完成 batch 为附加历史，**不替代时间下限**。

### 3.6 aborted 语义消除 + journal 损坏事实分离（Issue F / E）

**aborted 不是终态**（消除"标 aborted 或重放"的二义）：
- 本设计**不引入 aborted 终态**。原因：rotate 不可逆 ⇒ 一旦 `prepare` 持久化，唯一自洽路径是前滚到 done；"放弃"只在 prepare 之前的预检阶段成立（此时零副作用）。
- 语义归类（与 `01` D-2/D-8 对齐）：
  1. **自动恢复**：已持久化的未完成 batch，保留原 batchId/opId，续跑到 done；
  2. **同 requestId 重放**：原 batch **未完成** → 续跑原 batch（原 batchId/opId）；**已终态** → 返回存储终态结果；**两种情况都不建新 batch、不轮换**；
  3. **新 requestId**：新 batch、有意再轮换；
  4. **缺 requestId**：非幂等默认（每次=新操作）。

**journal 损坏时的可重建性（Issue E）**：

| 事实 | 可重建来源 |
|---|---|
| credential 状态（epoch/是否轮换过） | credential 文件（**可**） |
| batchId / requestId / keys[] / per-key phase 进度 / 终态结果 | **仅 journal**（缺 journal 即不可重建、不可伪造） |

- credential 文件**不构成** journal 的完整替代：不得从 credential 证据反推 requestId/batchId/keys[]/终态。
- 与既有规则一致：pending 意图随 journal 损坏不可辨识 → 不自动恢复，受控（人工逐 key）。

### 3.7 workspace/reset 事务边界（Issue D）

**事实（`CONFIRMED`）**：workspace/reset 当前是**内联同步代码**（`:7445-7509`），DSH 清理是 **await 同步串行**（`:7452-7469`），凭证清空用内存结构 `CONVERSATION_TOKENS.clear()`/`KNOWN_AGENT_TOKENS.clear()`（`:7494-7495`）——**无 intent/journal/requestId/幂等/恢复**。`credential.retireAll` 是设计概念，尚未实现。

**边界（DESIGN RECOMMENDATION）**：
- workspace/reset 是**独立批次事务模型**，**不套用 session resetTx 的 phase**（保留既有决策 B-2）。
- 但**不得声称它具备未定义的事务保证**。其自身语义（是否需 intent/幂等/恢复）**尚未完整设计** → 标 `UNCONFIRMED`，作为该端点的**发布阻断项**。
- `credential.retireAll` 幂等、部分失败恢复、keys 冻结、双损行为——全部**待独立设计**，不沿用 session resetTx。
- 激活门（Issue B 修正）：**workspace/reset 不参与 A/B/C 的 session resetTx 激活事件**。A/B/C 经各自 M07 集成+恢复验收后激活；W 须**独立设计自身事务模型、凭据退役语义、恢复规则、双损行为 + 独立验收**，经**独立批准门**后才可启用。**不得套用 session resetTx phase 草拟 W 的 phase**；不得声称 `credential.retireAll` 存在/幂等/可恢复（未证实）。

## 四、恢复矩阵（13 行，源自 21 号 §五 + 23 号修正）

前置：journal 先行、credential 为轮换事实权威、effect 幂等、恢复先于监听、**前滚唯一**。

| # | 崩溃点 | 重启后动作 | 分类 |
|---|---|---|---|
| 1 | intent/pending 写失败 | 零副作用、请求报错 | 可重试 |
| 2 | prepare 后、rotate 前 | **前滚**（整体重放，幂等）——**无 aborted 分支**；"prepare 已持久化 ⇒ 必须前滚"（rotate 不可逆、回滚不可能） | 可重试 |
| 3 | tokenRotated 后 | 续跑 conversationsCleared→stateFileRewritten→done（正确性 phase）；done 后**尝试** stop/archive（post-done best-effort，见第 9 行，无自动重试） | 可重试（仅正确性 phase） |
| 4 | conversationsCleared 后 | 续跑 stateFileRewritten | 可重试 |
| 5 | done 后、stop/archive 前崩溃 | **done 已提交，事务终态已定**；stop/archive 是 done 后锁外 best-effort——若崩溃时未执行，重启后**允许跳过**（认证已由 rotate 切断，无正确性损失）；若要求最终收敛，须另设**持久化 cleanup-pending 记录**（本版**不引入**，记审计告警 + DSH 侧人工核对） | 幂等（stop/archive 可跳过） |
| 6 | done 后重复恢复 | 跳过 | 幂等 |
| 7 | **phase append 写失败（原 batch intent 完好）** | 以 credential（opId/epoch）+ 完好 intent 对账补记；**credential/opId 证据不足则不得标 phase 完成**，该 key 保持"进行中"待人工 | 可重试（证据充分时） |
| 8 | 文件写成功 phase 未提交 | 重放 stateFileRewritten（幂等） | 可重试 |
| 9 | DSH stop/archive 失败/超时 | **不 revert done、不再次轮换、不自动恢复/持久化重试**；仅独立运维告警 + DSH 侧人工核对；若需最终收敛须另设持久化 cleanup-pending 机制（本版不引入） | 运维告警（非"可重试"） |
| 10 | 重复恢复 | epoch 只增一次、phase 幂等 | 可重试 |
| 11 | 重复提交请求（**区分四种**） | 同 requestId（未完成）→ 续跑原 batch、原 batchId/opId、不轮换；同 requestId（已终态）→ 返回存储终态、不轮换；新 requestId → 新 batch、再轮换（有意重置）；缺 requestId → 走 `01` D-2 非幂等默认；**不存在 aborted 终态**（见 §3.6） | 可重试/幂等 |
| 12 | journal+credential 双损 | corruptRecovery（认证面）+ 人工逐 key（事务面） | 受控 |
| 13 | journal 损坏 credential 完好 | **仅能重建 credential 事实（轮换已发生/epoch）**；batchId/requestId/keys[]/phase 进度/终态**不可从 credential 重建**——缺 journal 即不可辨识，归入受控恢复（人工逐 key） | 受控（非"可重试"） |

## 五、双损受控恢复（Issue 2 结论 + V-8b 可验证化 + 独立基线 + K_enum 来源）

**迁移前（`CONFIRMED`）**：凭据持久副本 = `social-v2.json` 内的 `agentToken` 字段（脱敏集合是内存 Map/Set）→ 枚举 `conversations` 覆盖全部凭据记录。**但这只对迁移前成立。**

**迁移后（Issue D 澄清）**：`agentToken` 字段已从 S 层移除（§1.2），`social-v2.json` **不再是凭据副本**。因此三 key 集来源须分阶段明确：

| key 集 | 迁移前来源 | 迁移后来源 |
|---|---|---|
| `K_pre`（恢复前全集） | 独立 manifest（M14，迁移前从 S 层 token 枚举建立）+ 完整性校验 | **独立 manifest（持久独立文件）**，非 S 层 |
| `K_enum`（枚举集） | — | **credential manifest**（独立可信基线），**不得与 `K_pre` 同源重复枚举**（否则是"同一不完整源的两次枚举"冒充独立证明） |
| `K_reg`（重生成集） | — | `credential.rotate` 的恢复重生成结果 |

- **K_enum 不得与 K_pre 同源**：若 manifest 损坏/不可信，`K_enum` 无独立来源 → **V-8b 保持 UNCONFIRMED，恢复保持受控/人工**。
- **独立性证据不足（Issue C）**：现状设计只写了"K_pre 来自 manifest、K_enum 来自独立 manifest/另一独立源"，但**未证明二者是真正独立的对象**（分离文件名/分离读取 ≠ 独立性证明）。若二者实为"同一不完整数据集的两次枚举"，则不构成独立覆盖。**独立性、全面性、上游失败模式是否共享——均未证实 → V-8b 标 UNCONFIRMED**，确定性恢复不得声称，恢复保持受控/人工（V8b-11）。
- **V-8b 身份比较**：`K_enum ⊇ K_pre`、`K_reg == K_enum`、归属/关联校验——全部保留。
- 认证面：corruptRecovery 全量重生成——**确定性仅以 V-8b PASS 为前提**。
- 事务面：pending 意图随 journal 损坏不可辨识 → **不声称自动完成**；人工逐 key + DSH 侧人工归档。
- corruptRecovery 连续失败 → 二代工具面 fail-closed 403（v1/控制台不受影响），告警升级人工。

## 六、锁与顺序（C-7，三级 + 共享账本）

| 层 | 机制 | 覆盖 |
|---|---|---|
| L1 同步变更 | 单线程事件循环原子 | 38 处同步写，零锁 |
| L2 跨 await 变更 | fencing（expectEpoch） | send 类处理器 |
| L3 多 phase 操作 | per-key 锁（token→memory→ledger） | resetTx/credential/commit/ledger |
| **L4 共享账本** | **单写者队列/账本锁（实施期择一）** | `token-usage.jsonl`（跨 key 共享，per-key 锁不保护） |

- 快照提交不加锁（同步重建天然互斥，C-1）。
- 锁内禁网络（stop/archive 锁外；账本临界区禁 await）。
- 无死锁：单序取锁 + 锁内无等待。
- **账本并发与持久性**：提交序 = 样本进入单写者队列/串行提交的顺序；append 与 compact 互斥；**同步 append 原子性 ≠ 崩溃持久性**（持久性需实施期测试）；崩溃点行为见 `01` §六·C。
- **账本与 reset 解耦**：账本记录由 DSH 会话事件驱动（`bridge.js:11763/11769`），非 reset 必要 effect；账本失败不逆转 reset、不触发再次轮换、不入 `overallResult`（D-8）。
