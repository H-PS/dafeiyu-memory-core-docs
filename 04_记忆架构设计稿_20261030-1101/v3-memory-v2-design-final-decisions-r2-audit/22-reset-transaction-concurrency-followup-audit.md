# 22 · Reset 事务并发跟进审计（F-8 / F-2 / F-7 / 批次 / 停止延迟 / F-6 / 锁）

> 基线：`D:\v3-memory-v2-design-final-decisions-r1\`（R1，12 份）+ 19/20/21 号报告。
> 性质：**只读审计 + 设计建议**。本轮唯一新增文件即本报告（独立目录 `r2-audit`）；未修改 R1 目录、19/20/21 号报告、源码、配置、state、Persona、插件；未运行迁移/业务写入/reset 端点；未启停服务。
> 结论标签：`CONFIRMED`（直接源码证据）／`INFERRED`（有依据的结构性推断）／`UNCONFIRMED`／`DESIGN RECOMMENDATION`（设计建议，**不是**源码事实）。

---

## 一、执行结论

**PASS WITH CONDITIONS。**

- 七个审计项中，六项在"最小设计改动"下可达安全：共享快照提交**无需**全局锁（同步重建不变式即可，见 §三）；恢复规则可以做到**每个崩溃点一条确定性动作**（前滚唯一，见 §五）；停止/归档天然绑定旧 DSH sessionId，换绑不可能（§七）。
- 一项存在**方案级修正机会**：F-6 的兜底投递——21 号"挂 reconcile tick"的前提有误（`reconcileSessionPolicies` 非周期 tick，`:2806-2820`），且"无 prompt ⇒ 无 turn ⇒ 无工具调用 ⇒ 无需 token"的推理使**合成 prompt 默认不必要**。更优解是把令牌注入**后移到投递层单点**（§八），同时消灭 F-1 与 queued 旧 token 两个问题。
- 条件：§十 的 10 条 P0 设计约束并入下一轮（R2）修订；V-7/V-8 与 D-30 按既定节奏关闭。

## 二、证据与审计范围

本轮新读源码（其余复用 19–21 号）：

| 证据 | 位置 | 用途 |
|---|---|---|
| `atomicWriteJson` / `atomicWriteText` 函数体 | `bridge.js:182-193` / `:196-206` | 全同步（writeFileSync + renameWithRetry，无 await） |
| `getSocialV2State` 创建体 | `bridge.js:8764-8809` | 复活向量（`:8782` 新 token、`:8803` remember、`:8805-8806` 定时器） |
| mark-read 处理器全程 | `bridge.js:4783-4823` | "auth 后无 await"型处理器的代表样本 |
| `reconcileSessionPolicies` 全体 | `bridge.js:2806-2820` | 非 tick、事件驱动（空 preset 守卫 `:2816`） |
| `stopRetiredSessionWork` 函数体 | `bridge.js:2734-2741` | 清理以捕获的 DSH sessionId 绑定 |

复用：19 号 §一/§二（44 写点、token 读写全景）、20 号 §一（T2 修订建议）、21 号（F-1～F-7、模型 A/B、恢复矩阵）。未做全仓扫描。

## 三、F-8 · 共享 social-v2 快照提交串行化

### 3.1 证据

| 事实 | 位置 | 级别 |
|---|---|---|
| `atomicWriteJson` 为**全同步**：随机 tmp + `writeFileSync` + `renameWithRetry`，无任何 await | `bridge.js:182-193` | CONFIRMED |
| `buildSocialV2Snapshot()` 每次调用**从当前内存重建**完整快照（遍历 `socialV2.conversations`） | `bridge.js:8935-8963` | CONFIRMED |
| 普通保存 = `buildSocialV2Snapshot()` + `atomicWriteJson`，两者在**同一个同步段**内完成（无 await 间隔） | `bridge.js:8966-8972` | CONFIRMED |
| Strict 版同构 | `bridge.js:8976-8978` | CONFIRMED |
| 唯一非重建式写者：`/api/socialV2/reset` 直写**固定**空对象 `{conversations:{}}` | `bridge.js:4624` | CONFIRMED |
| 现源码**不存在**任何跨 key 的提交锁/串行化机制（只有单线程事件循环的天然串行） | 全文件无此类构造 | CONFIRMED |
| `withKeyLock` 是 08 号的**设计定义**（进程内 Promise 链） | `08-social-v2拆分与并发一致性.md:91-103` | DESIGN RECOMMENDATION（非现状） |

### 3.2 五个问题的显式回答

1. **两个不同 key 能否"并发"生成并提交快照？** 调用时机上可交错（各自 await 后恢复执行），但"构建快照 + 落盘"是**一个不可分割的同步段**——单线程事件循环使两次保存严格串行，不可能交错写。`CONFIRMED`（同步性 + 单线程）。
2. **后一次快照会覆盖前一次事务已完成的变更吗？** **不会**。每次保存都从活内存重建：只要变更先于保存进入内存，后一次快照必然包含它。唯一例外是 `:4624` 固定空对象（它不重建内存，会把批次进行期间新建会话的状态从文件里丢掉——内存仍是真相，下一次保存自愈，但该写法必须消灭，见 C-1/C-8）。`CONFIRMED`。
3. **所有现存写者参与同一提交串行化机制吗？** 现状**没有显式机制**；文件级互斥由"同步段 + 单线程"天然成立；逻辑操作顺序（同 key 的操作序列）目前无保护——这正是 `withKeyLock` 提议要补的。`CONFIRMED`（现状）+ `DESIGN RECOMMENDATION`（补锁）。
4. **per-key 锁保护的是共享文件还是单个会话状态？** **只保护操作顺序**（同一 key 的 reset 与普通变更互斥），不保护文件——文件一致性由"重建式快照 + 同步提交"独立保证。两者正交，不可混为一谈。`DESIGN RECOMMENDATION`（澄清性结论）。
5. **reset 事务提交全量快照时，普通写能否继续？** **能**：普通写在锁外同步执行，每次保存重建的是"已包含 reset 置换结果"的内存；唯一冲突窗口是 F-2 的在途复活（由 epoch 复验处理，不靠快照锁）。`CONFIRMED` + `DESIGN RECOMMENDATION`。

### 3.3 最小安全改动

`DESIGN RECOMMENDATION`（C-1）：确立**"内存唯一真相 + 同步重建快照"不变式**——任何写者禁止持久化"捕获的旧快照"；删除 `:4624` 的固定空对象（resetTx 的 `stateFileRewritten` effect 以全量重建取代）。**不引入全局快照锁**——证据不支持其必要性。

## 四、F-2 · epoch fencing 与过期请求防护

### 4.1 证据分布

| 符号 | 位置 | 行为 | 级别 |
|---|---|---|---|
| `sessionEpoch` 递增点 | `:4588`（socialV2/reset）、`:7393`（session/reset）、`:7446`（workspace/reset）、`:11271`（QQ /reset） | 四条重置路径全部先 bump | CONFIRMED |
| `ensureSession` 的 epoch fencing | `:8462`（入口捕获）、`:8478`/`:8530`/`:8540`（await 后复验） | **代码中已存在的同型防护** | CONFIRMED |
| `isCurrentSession` | `:2723` | 会话身份比对 | CONFIRMED |
| `getSocialV2State` 创建体 | `:8773-8807` | **不存在即创建**：新 token（`:8782`）+ remember（`:8803`）+ 入 map（`:8804`）+ 两个定时器（`:8805-8806`） | CONFIRMED |
| mark-read 处理器 | `:4783-4823` | auth（`:4787-4789`）→ `getSocialV2State`（`:4790`）→ **纯同步变更段**（`:4809-4819`）→ save（`:4820`）；auth 后**无 await** | CONFIRMED |
| wake 接受后守卫 | `:9555-9557` | `isCurrentSession` + token 比对后才清游标 | CONFIRMED |

### 4.2 四个问题的判定

| 问题 | 判定 |
|---|---|
| 1. reset 前已认证的请求能在 reset 后恢复执行吗？ | **能**（await 交错）。`CONFIRMED`（事件循环语义） |
| 2. 能重新获取到新建的会话对象吗？ | **能**：再次调用 `getSocialV2State` 命中"不存在即创建"分支（`:8774-8807`）→ 隐式复活（带新 token 与定时器）。`CONFIRMED` |
| 3. 能把过期结果写进新会话吗？ | 两条路径：**(a)** 直接写**捕获的旧 `st` 引用** → 写的是孤儿对象，快照器不可见（对象置换后无害）；**(b)** 写前**重新 `getSocialV2State`** → 拿到新对象，把过期结果写进新会话（有害）。`CONFIRMED`（路径 b） |
| 4. 会造成隐式会话重建吗？ | **会**（路径 b 的副作用，含新 token 与定时器），这是 F-2 的主发现。`CONFIRMED` |

### 4.3 防护机制分级

| 机制 | 分级 | 说明 |
|---|---|---|
| 入口屏障（tokenRotated 先行） | **正确性必需** | 轮换后旧 token 的新请求在中间件（`:3303-3306`）被 403，到不了状态层 |
| post-await epoch 复验 | **正确性必需**（仅对"auth 后含 await 再写状态"的处理器） | 推广 `ensureSession :8478` 的既有模式 |
| 对象置换（reset 置换而非原地清空） | **正确性必需** + 纵深 | 使路径 (a) 从"有害"降为"无害" |
| 捕获会话身份（`isCurrentSession`） | 纵深 | 已有；用于投递路径（`:9539`/`:9555`） |

### 4.4 `assertNotReset` 类助手的一致可应用性

`DESIGN RECOMMENDATION`（C-2）：**不要逐处理器包 helper**——`getSocialV2State` 是所有状态获取的**唯一单点**（canonicalV2Key 归一在 `:8766`）。最小改动是在其内部加可选 fencing 参数（如 `getSocialV2State(key, { expectEpoch })`：传入且不匹配 → 抛 `reset_during_request`，与 `:8478` 同型）。这样：
- auth 后无 await 的处理器（mark-read 型）：天然安全，零改动；
- auth 后含 await 的处理器：把"先 getSocialV2State 后 await 再写"的顺序改为"await 后带 expectEpoch 重取"即获得防护；
- **实施检查表**：以 grep 配方枚举（auth 行与 `saveSocialV2State` 行之间含 `await` 的端点）——属实施阶段清单，本轮不全列（见 §十一）。

**不得**以"ensureSession 已有 epoch 检查"宣称全局已防护——该检查只覆盖会话创建路径，不覆盖工具端点的状态写路径。

## 五、F-7 · 确定性恢复规则

### 5.1 证据

| 事实 | 位置 | 级别 |
|---|---|---|
| journal pending/committed 两段式（write-ahead）设计 | `05-记忆写入与Retain.md:40-55`、`:68-73` | DESIGN RECOMMENDATION（新建） |
| `credential.rotate` 的 opId/epoch 幂等 + journal 不存 token | `04-…凭据生命周期.md:230-253`（§4.4） | DESIGN RECOMMENDATION |
| credential 文件含 `rotatedBy/opId/epoch/needsDelivery` | `04` §6.1（`:310-330`） | DESIGN RECOMMENDATION |
| 双损兜底：corruptRecovery 全量重生成 | `04` §6.2-4（`:339`） | DESIGN RECOMMENDATION |

### 5.2 六个问题的确定性回答（每点一条规则，无"abort or replay"悬置）

| # | 问题 | 确定性规则 |
|---|---|---|
| 1 | prepare 是否在任何不可逆 effect 之前持久化？ | **是**（journal 先行是 resetTx 的硬序：pending 先于 tokenRotated）`DESIGN RECOMMENDATION` |
| 2 | journal 与 credential 写入会独立失败吗？ | **会**（两次独立落盘）→ 由 C-3 对账规则消解（§5.3） |
| 3 | credential 的 opId/epoch 能可靠判定轮换已完成吗？ | **能**：credential 文件是轮换的**唯一事实权威**（epoch 单调、opId 记录、rotate 幂等按 opId） |
| 4 | 恢复能重建缺失的 journal phase 吗？ | **能**：credential 记录 rotate 事实 → 补记 `tokenRotated`；会话/文件状态按幂等 effect 重放 |
| 5 | prepare 已持久化但 effect 未开始？ | **必须前滚**：`journal pending ⇒ 前滚至 done`，不存在"放弃"分支——rotate 不可逆使回滚不可能，前滚是唯一自洽选择。若业务上确需"未开始就放弃"，只能在 **prepare 之前**的预检阶段拒绝（此时零副作用） |
| 6 | journal 与 credential 同时损坏？ | **corruptRecovery 全量重生成**（所有 key epoch+1、`needsDelivery=true`、审计+告警）——这是**确定性自愈**：服务**不需要停机、不需要拒绝普通请求**；恢复完成后普通请求照常（旧 token 全部失效是预期，模型经下一轮自然 prompt 拿新 token）。仅当 corruptRecovery 自身连续失败才升级为人工介入 |

### 5.3 对账方向（固定）

`DESIGN RECOMMENDATION`（C-3）：**credential 为轮换事实权威；journal 为批次/phase 进度权威；恢复方向恒为前滚**。journal 落后 → 按 credential 补记后继续；journal 损坏 → 以 credential 重建剩余 phase 并告警；两者一致 → 常规续跑。

## 六、批次协调器的持久性（Model A）

### 6.1 证据与判定

| 事实 | 级别 |
|---|---|
| journal 追加行为已有格式先例（`05` :70-73：append-only JSONL、pending/committed） | DESIGN RECOMMENDATION |
| 现源码无任何批次协调器（reset 端点为内联循环，`:4592-4626`） | CONFIRMED |

**结论（DESIGN RECOMMENDATION，C-4）**：批次协调器**必须可从 per-key journal 完整重建**；持久化仅为一行 batch intent（追加进同一 journal：`{k:'batch', batchId, keys[], ts}`），**不建独立文件、不建内存单例**。

四个崩溃点的确定动作：

| 崩溃点 | 恢复动作 |
|---|---|
| 第一个 key 开始前 | intent 存在、无 per-key 记录 → 重放全部 key（各 key 幂等） |
| 部分 key 已完成 | 已 done 的跳过，剩余续跑 |
| 某 key 进行中 | 该 key 自身 resetTx 按 phase 续跑（与单 key 场景同） |
| 全部完成但响应未返回 | batch 判定完成；客户端重试同一请求 → 新 batch（再次轮换）——**符合 D2 语义**（每次 reset 都轮换），在 `04` §5.4 注明"重复 reset=多次轮换是预期" |

**权威恢复模型一句话**：批次状态 = `intent.keys` 与 `per-key done 记录` 的差集；不需要也不应该有第三个真相源。

## 七、延迟 stop/archive 的安全（E）

| 事实 | 位置 | 级别 |
|---|---|---|
| `stopRetiredSessionWork(sessionId, key)` 以**捕获的 DSH sessionId** 为参数，fire-and-forget + 双层 try/catch 告警 | `bridge.js:2734-2741` | CONFIRMED |
| 三条 reset 路径均先取 `sid = state.sessions[key]`（或 oldSessionId）再删除映射，清理调用全部携带 sid | `:4593-4599`、`:7438-7440`、`:11314-11316` | CONFIRMED |
| DSH 侧 API 以 sessionId 寻址（`api.stopSessionWork(sessionId)`、`api.workspace.archiveSession({sessionId})`），**不存在按 key 的停止 API** | 同上 | CONFIRMED |
| `reconcileSessionPolicies` 有空 preset 守卫，防"一次抖动全体退役" | `:2807-2816` | CONFIRMED |

**判定**：延迟清理**不可能**误停同 key 的新会话——只要遵守既有模式"先捕获 sid、再删映射、清理只认捕获的 sid"。因为 DSH API 无 key 寻址，"清理时重新解析 key→sid"在现状代码中根本不发生；风险只在未来实现引入（例如图省事传 key）。
`DESIGN RECOMMENDATION`（C-5）：成文两条——①清理操作**必须绑定捕获的 DSH sessionId**，禁止在清理时刻重新解析 key→sid；②stop/archive 失败=审计告警非致命（rotate 才是认证切断的正确性屏障，21 号 F-5 已定）。

## 八、F-6 · 凭据自动交付兜底

### 8.1 关键证据（推翻 21 号的一处前提）

| 事实 | 位置 | 级别 |
|---|---|---|
| `reconcileSessionPolicies` **不是周期 tick**：仅在模式切换/白名单变更等策略事件时被调用 | 定义 `:2806-2820`；调用点 `:2903`/`:3507`/`:4074` | CONFIRMED |
| `deliverPrompt` 每 key 串行队列；离线入队保留**构造时就嵌好的** promptText | `:9573`、`:9500-9507` | CONFIRMED |
| 构造器在构造期嵌入 tokenLine（`:4363`/`:10674`/`:10751`） | 19 号 §2.2-② | CONFIRMED |
| DSH `result.ok` 只代表收下消息，不代表校验内容 | `:9543-9553`（无 token 相关分支） | CONFIRMED |
| "无 prompt ⇒ 无 turn ⇒ 无工具调用 ⇒ 无需 token" | 结构推理 | INFERRED |

### 8.2 分析

1. **21 号 F-6 的"挂 reconcile tick"前提不成立**（不是周期任务）→ 该建议需修正。
2. **更深的修正**：`needsDelivery` 的真实意义是"下一轮 prompt 必须带新 token"。既然队列里的 prompt 是构造期嵌的旧 token（F-1），与其做合成 prompt，不如**把令牌注入后移到投递层**：

   `DESIGN RECOMMENDATION`（C-6，**修订 20 号 T1-1 与 21 号 F-1**）：三个构造器**不再内嵌** tokenLine；`deliverPromptNow` 在调用 `api.sessions.prompt` 之前，于**固定槽位**（保持现有"角色/身份之后"的前缀次序，兼顾 09 号的缓存稳定性）注入**当前** credential 的令牌段。效果：
   - 构造-轮换竞态（F-1）自然消失——发送时刻取的必然是当前 token；
   - 离线队列/重连补投的 prompt 在真正发送时自动带新 token（覆盖 DSH 恢复补投场景）；
   - 注入点从 3 个收敛为 **1 个**（`CRED-08` 断言随之简化）；
   - stale 检查（21 号 CRED-09）保留为纵深。
3. **兜底策略建议**（回答六小问）：
   1. 启动 reconciliation 处理 `needsDelivery=true`：**可以做且安全**——但注意其语义是"告警 + 依赖下一轮自然 prompt"，不是"发合成 prompt"；
   2. 周期性合成 prompt 的代价：每次=一次真实模型调用（token 成本 + 可能触发工具调用）→ **默认不做**；
   3. "保持沉默"指令**不能保证**模型沉默（它仍可调用发送工具）——合成 prompt 有真实的误发风险，不可作为静默兜底；
   4. 私聊/群聊在"自然投递"策略下**无需差异化**（策略相同：等自然 prompt）；
   5. 重试：仅对投递失败路径保留既有重试；兜底本身不需要 backoff（因为没有主动投递）；告警沿用 `credential_delivery_stalled`（3 turns 或超 N 小时）；
   6. **自然 future prompt 是更安全的兜底**——且在"无 prompt 就无需 token"的推理下，兜底缺口其实只覆盖"轮换发生在队列 prompt 在途"的场景，已由 C-6 的投递时注入完全解决。
4. **合成 prompt（主动推一条仅含令牌的 prompt）是产品行为**：若将来确需（例如要求"轮换后 X 分钟内必须完成一次投递确认"），需主人明确批准，并单独评估误发风险——本版**不排期**。

## 九、锁的范围与顺序（G）

| 事实 | 位置 | 级别 |
|---|---|---|
| `withKeyLock` 为设计定义（Promise 链、按 key、前错续跑、覆盖 commit 阶段 3 全程） | `08:91-103` | DESIGN RECOMMENDATION |
| 锁序：token → memory → ledger；credential 独立锁 `withKeyLock('token:'+key)` | `08:195-197` | DESIGN RECOMMENDATION |
| 现源码无该互斥（只有单线程同步段天然串行） | 全文件 | CONFIRMED |
| 锁内禁网络等待 | 21 号 F-5 | DESIGN RECOMMENDATION |

**取锁规则（DESIGN RECOMMENDATION，C-7）**：
1. **必须取 per-key 锁的操作**：A 层 `commit()`（阶段 3）、`credential.rotate/retire/ensure`（token 锁）、`prompt-ledger` 写（ledger 锁）、resetTx 的每 key 临界段、`socialStateWriter` 收敛后的 S 层变更。
2. **必须取共享快照提交锁的操作**：**没有**——F-8 已证明同步重建使文件级互斥天然成立，快照提交不加新锁。
3. **冲突顺序**：不存在。resetTx 按 `token → memory` 顺序取两把 per-key 锁，与全局锁序一致；普通写者每次只取一把；ledger 在两者之后——任何路径都不会形成环。
4. **网络等待**：锁内禁止（DSH stop/archive 在锁外）。
5. **死锁**：单序取锁 + 锁内无等待 ⇒ 无死锁路径。

**诚实声明**：以上全部是**提议机制**；现源码的"并发安全"来自单线程 + 同步保存，`withKeyLock` 落地前任何"已受锁保护"的表述都是不成立的。

## 十、P0 设计约束汇总（并入下一轮 R2 修订）

| # | 约束 | 来源 |
|---|---|---|
| C-1 | "内存唯一真相 + 同步重建快照"不变式；禁止持久化捕获的旧快照；不需要全局快照锁 | F-8 |
| C-2 | `getSocialV2State` 单点 fencing（expectEpoch 参数）；post-auth-await 处理器强制使用；禁止以 ensureSession 的检查宣称全局防护 | F-2 |
| C-3 | 前滚唯一；credential=轮换权威；journal=进度权威；双损=corruptRecovery 确定性自愈（服务不停机） | F-7 |
| C-4 | 批次协调器=journal 内一行 intent + per-key 收敛判定；不建独立文件/内存单例 | D |
| C-5 | 清理只认捕获的 DSH sessionId；禁止清理时重解析 key→sid；stop/archive 失败非致命 | E |
| C-6 | **令牌注入后移投递层单点**（固定槽位、发送时刻取当前值）；默认不做合成 prompt；合成推送如需=需主人批准的产品行为 | F-6/F-1 |
| C-7 | 锁序 token→memory→ledger；快照提交不加锁；锁内禁网络 | G |
| C-8 | 删除 `:4624` 固定空对象直写（并入 resetTx effect） | F-8 |
| C-9 | post-auth-await 处理器枚举清单进实施检查表（grep 配方：端点 auth 行与 save 行之间含 await） | F-2 |
| C-10 | 离线队列 prompt 的令牌在投递时重注（C-6 的直接推论，覆盖 DSH 恢复补投） | F-6 |

## 十一、尚未确认事项（UNCONFIRMED）

| # | 事项 | 缺什么 |
|---|---|---|
| 1 | DSH 对 prompt 内容中的 token 是否做任何校验/记录 | DSH 侧源码（按"不校验"设计，不影响 C-6） |
| 2 | post-auth-await 处理器的**完整**清单 | 实施阶段按 C-9 配方枚举（本轮给出方法与代表样本，不做全仓扫） |
| 3 | DSH 对离线队列 prompt 的保留时长（影响补投时 token 过期概率） | DSH 侧证据；C-6 使该问题失效（补投时重注） |
| 4 | V-7（40 轮压缩实测）/ V-8（DSH usage 缓存字段） | 维持既有门禁 |

## 十二、对 20/21 号报告与未来 R2 文档集的修订建议

| 对象 | 修订 |
|---|---|
| 20 号 T1-1 | 注入点方案升级：3 个构造器内嵌 → **投递层单点注入**（C-6）；CRED-08 断言改为"投递层注入后，三构造器产物在发送时均含当前 token" |
| 21 号 F-1 | 修复手段由"constructedEpoch 校验"升级为 C-6（stale 检查保留为纵深） |
| 21 号 F-6 | **撤销"挂 reconcile tick"前提**（非周期任务）；兜底=自然投递 + 告警；合成 prompt 不排期 |
| 20 号 T2-2 / 21 号 §五 | 恢复矩阵补"双损 → corruptRecovery，服务不停机"的确定性规则（C-3）；删除任何"abort or replay"悬置表述 |
| 21 号 §三 | 模型 A 补 batch intent 的 journal 落点（C-4） |
| R2 文档集 | `04` §3/§5.5 按本报告重写（注入后移、前滚规则、恢复矩阵 13→14 行含双损）；`10` 补 M07 行（resetTx/journal 模块）；`14` 用例同步（CRED-08 改、CRED-09 保留、新增"投递时注入"用例）；`16` §四 V-8 维持 |

## 十三、最终就绪度评估

**PASS WITH CONDITIONS。**

理由：七个审计项中不存在需要推翻方案的发现；F-8 证明文件级串行已由"同步重建"天然成立（省掉一把全局锁）；F-2 的防护有代码同型先例且可单点落地；F-7 可给出每崩溃点唯一确定性动作；批次与停止延迟均有确定解。唯一的方案级修正是 F-6/F-1 的"注入后移"——它使设计更简单而非更复杂。

**进入下一轮（R2）修订前的全部条件**：
1. §十 的 C-1～C-10 并入定点修订（含对 20/21 号两处建议的修正）；
2. `10` 补 M07 行（resetTx/journal 模块）；
3. V-7（40 轮实测，实施后）与 V-8（DSH usage 缓存字段，实施前只读核查）按既定节奏关闭；
4. D-30 拍板（阻塞 M2，不阻塞 M0/M1 评审）。

**事实核查、审计、设计通过均不构成源码实施授权。**

---

> 本轮仅创建本报告于独立目录 `D:\v3-memory-v2-design-final-decisions-r2-audit\`；未修改 R1 目录、19/20/21 号报告、任何源码/配置/state/Persona/插件；未运行迁移、业务写入、reset 端点或模拟清理；未启停服务。报告不含任何真实令牌、凭据、私有 prompt 正文或个人会话内容（所有 token 相关仅以行号与结构描述呈现）。
