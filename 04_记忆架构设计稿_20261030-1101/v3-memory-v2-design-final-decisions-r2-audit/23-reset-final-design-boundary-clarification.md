# 23 · Reset 最终设计边界澄清（基于 19–22 号报告）

> 基线：`D:\v3-memory-v2-design-final-decisions-r1\`（R1，12 份）+ 19/20/21/22 号报告。
> 性质：**只读澄清审计**。本轮唯一新增文件即本报告；未修改 R1 目录、19–22 号报告、源码、配置、state、Persona、插件；未调用 reset 端点、未模拟清理、未迁移、未业务写入、未启停服务。
> 标签：`CONFIRMED` / `INFERRED` / `UNCONFIRMED` / `DESIGN RECOMMENDATION`。每条建议注明：**[Correctness]**（正确性必需）/ **[Defense]**（纵深）/ **[Ops]**（运维便利）/ **[Product]**（需主人批准的产品行为）。

---

## 一、执行摘要

五项边界全部给出显式决策，其中两项修正了 22 号的表述：

1. **A（锁策略）**：22 号 §3.2 与 §9 C-7 的表面矛盾属实——解法是**三级分层**：同步处理器不需要锁（事件循环原子）；跨 await 处理器用 **fencing**（不强制加锁）；只有多 phase 的 resetTx/credential/commit 强制 per-key 锁。
2. **B（双损恢复）**：认证恢复是确定性的；**reset 事务恢复不是**——pending 意图随 journal 损坏而不可辨识。修正 22 号 C-3：双损 = "认证面确定性自愈 + **事务面受控恢复**（控制台人工逐 key 决定）"。
3. **C（请求幂等）**：现状**不存在**任何请求级幂等标识（`CONFIRMED`，grep 零命中）。给出 requestId 设计。
4. **D（批次意图）**：intent 先于首个 effect、keys[] 在请求接受时冻结、恢复禁止重新枚举——三条硬规则成文。
5. **E（注入边界）**：**"deliverPromptNow 是唯一边界"不成立**——`api.sessions.prompt` 实有 **4 个调用点**（`:2170`/:`:2233` 内部学习会话、`:9543` deliverPromptNow、`:11527` 一代仿真直投）。修正为："**reserved2 QQ 会话的唯一出口 + 两类显式豁免路径**"；令牌注入后移到投递层（22 号 C-6 方向保持、边界定义精确化）。

**总评：PASS WITH CONDITIONS**（条件 = R2 修订纳入本报告 D-1～D-8 决策 + 既有 V/D 门禁关闭）。无 BLOCKED。

---

## 二、审计范围与证据

本轮新读（其余复用 19–22 号）：

| 证据 | 位置 | 用途 |
|---|---|---|
| `api.sessions.prompt` 全部调用点 | `bridge.js:2170` / `:2233` / `:9543` / `:11527` | E：投递出口枚举 |
| `runSlangExtraction` / `runSlangResearch` 函数体 | `bridge.js:2153-2213` / `:2215-2245` | 内部学习会话定性（`ensureSlangLearnerSession`，无令牌） |
| v1 直投路径 | `bridge.js:11495-11539`（`ensureSession`、`isCurrentSession :11526`、steer/queue `:11531`） | v1 豁免定性 |
| send-message 处理器前段 | `bridge.js:5200-5262` | auth→await→变更 的跨 await 样本 |
| mark-read 处理器 | `bridge.js:4783-4823`（22 号已读，引用） | 同步段样本 |
| `initial-prompt` 端点 | `bridge.js:5023-5076` | 确认只存/取，不投递 |
| 幂等机制存在性 | 全文件 grep `requestId|idempoten` **零命中** | C 项现状 |

---

## 三、A · 普通 SocialV2 写入的 per-key 锁策略

### 3.1 证据

| 事实 | 位置 | 级别 |
|---|---|---|
| mark-read：auth（`:4787-4789`）→ 取状态（`:4790`）→ **纯同步变更段**（`:4809-4819`）→ save（`:4820`），auth 后无 await | `bridge.js:4783-4823` | CONFIRMED |
| send-message：auth（`:5243-5247`）→ **跨 await**（deliverPrompt 投递）→ 变更/落账（`:5310-5344` 区段） | `bridge.js:5200-5349` | CONFIRMED |
| `appendSocialV2Message` 内部调用 `getSocialV2State`（可触发复活） | `:10248`（22 号 F-2） | CONFIRMED |
| 快照构建+落盘为同一同步段（无 await 间隔） | `:8935-8978`（22 号 F-8） | CONFIRMED |
| `withKeyLock` 为设计定义、现源码无此互斥 | `08:91-103` | DESIGN RECOMMENDATION |

### 3.2 三层区分（解决 22 号的表面矛盾）

| 层 | 机制 | 覆盖 |
|---|---|---|
| **L1 逻辑变更串行化**（同 key 操作顺序） | 同步处理器：单线程事件循环天然原子，**无需锁**；跨 await 处理器：**写前 fencing**（`getSocialV2State` 的 `expectEpoch` 参数，22 号 C-2）或 per-key 锁 | A 项主战场 |
| **L2 文件快照提交** | 同步重建式快照天然互斥（22 号 F-8），**不加锁** | C-1 保持 |
| **L3 跨进程文件访问** | `acquireLock`（防双实例）+ `stateRev` 校验（防脚本并发写） | C-1/C-7 保持 |

### 3.3 六问回答

1. **哪些操作变更特定会话的内存状态？** 全部 44 个写点的宿主（19 号 §一）：工具端点（mark-read/wake-config/send-*/wait/collect 等）、QQ 事件入库（appendSocialV2Message/Poke/bootstrap）、唤醒簿记、memory 四端点、三条 reset 路径。
2. **哪些会跨 await 重叠？** auth 后需 await 再变更的：send-message/send-burst/send-poke/send-sticker/collect-sticker/wait/forward/摘要投喂/唤醒投递等（await 的是 DSH/QQ/媒体解析）；mark-read 型同步处理器**不**跨 await。
3. **同 key 两个操作会互相观察并覆盖对方变更吗？** **会**——但不是文件级（L2 天然互斥），而是逻辑级：操作 X await 期间 reset 清理并置换对象，X 恢复后 (a) 写旧引用=孤儿（无害）或 (b) 重取状态=复活/写新对象（有害，F-2）。防覆盖靠 fencing，不靠"写前重读文件"。
4. **哪些必须取 per-key 锁？** **仅 L3 类**：resetTx 临界段、`credential.rotate/retire/ensure`（token 锁）、A 层 `commit()` 阶段 3、`prompt-ledger` 写。**同步 SocialV2 变更不取锁**（事件循环已保证原子）；**跨 await 的 SocialV2 变更用 fencing 而非锁**（见 3.4）。
5. **不同 key 能否继续独立处理？** 能：锁按 key、文件层无共享可变状态（重建式快照）。
6. **同步快照重建对文件级串行化是否仍然充分？** **充分**（22 号 F-8 结论维持），前提是 C-1 不变式（禁止持久化捕获的旧快照）与 C-8（消灭 `:4624`）成立。

### 3.4 一致锁策略（DESIGN RECOMMENDATION，C-7 修订）

> **C-7（修订）**：per-key 锁**只强制用于** resetTx / credential / A 层 commit / ledger 四类多 phase 操作；**同步** SocialV2 变更依赖单线程原子性（零改动）；**跨 await** 的 SocialV2 变更依赖 fencing（`expectEpoch`）而非锁——38 处同步写不动、逐个加锁不可实施且易漏。
> 由此 22 号 §3.2（"普通写可在锁外继续"）与 §9 C-7 的矛盾消解：§3.2 说的是 L1 同步类与 L2 文件层；C-7 修订后只管 L3 类。

---

## 四、B · journal 与 credential 双损恢复

### 4.1 恢复目标拆分与可达性

| 目标 | 双损后可达性 | 依据 |
|---|---|---|
| 1. 认证安全恢复 | **确定性可达**：`corruptRecovery` 全量重生成（从仍可读的 `social-v2.json` conversations 枚举 key——`social-v2.json` 是第三个文件，**未损**） | `04` §6.2-4；19 号 §一（文件清单） |
| 2. reset 事务恢复 | **不可自动**：pending 意图存储于 journal（已损）→ 无法辨识哪些 key 的 reset 未完成 | journal 是唯一进度权威（22 号 C-3/C-4） |
| 3. social-v2 会话状态一致性 | **天然完好**（该文件不在双损集合内）；已被 reset 清掉的 key 不在文件中，未清的仍在——**但无法区分"reset 未跑到"与"从未 reset"** | 19 号 §一 |
| 4. DSH 会话清理 | **不可自动**：不知道哪些旧会话属于待清理范围 → 人工在 DSH 侧核对归档 | 19 号 §四 |

### 4.2 逐问回答

- **能可靠重建什么？** 仅认证面：全部 key 的新 token（epoch+1、`needsDelivery=true`、审计+告警）。
- **能判定哪些 reset 在 pending 吗？** **不能**（journal 已损）。`CONFIRMED`（结构性结论）。
- **能识别全部受影响 key 吗？** 能枚举"现存会话"，但**不能**区分其中哪些是"reset 半途"——影响范围判定不可自动。
- **能区分完成与半完成吗？** **不能**（进度权威已失）。
- **旧 conversations / DSH 会话怎么办？** conversations 保留原状（不自动清）；DSH 旧会话全部 403（token 已全量重生成）但**仍在 DSH 侧运行/排队**，需人工归档。
- **corruptRecovery 自身失败？** **fail-closed**：credential 不可写/不可信 ⇒ 所有 reserved2 工具端点因认证失败 403（`agentTokenOk` 无可信基准），**v1 路径与控制台不受影响**（v1 无令牌机制，`:4361-4364` 的 isV2 门；控制台走 console-token）→ 桥接保持在线但二代工具面不可用，告警升级人工。

### 4.4 决策（修正 22 号 C-3）

| 失败类 | 策略 | 级别 |
|---|---|---|
| 仅 journal 损坏，credential 完好 | 以 credential 对账重建剩余 phase（22 号原规则，保持） | [Correctness] |
| 仅 credential 损坏，journal 完好 | corruptRecovery 全量重生成 + 按 journal 前滚（22 号原规则，保持） | [Correctness] |
| **双损** | **受控恢复**：corruptRecovery（认证面，确定性）+ **控制台人工逐 key 决定**是否补清 conversations + 人工在 DSH 侧归档；服务保持在线（v1/控制台可用），二代工具面在重生成成功前 403 | [Correctness]（**修正 22 号"不停机自愈"的表述**：自愈仅覆盖认证面，事务面是受控恢复） |
| corruptRecovery 连续失败 | fail-closed：二代工具面保持 403，告警升级，人工介入；不自动重试超过 3 次 | [Correctness] |

> **UNCONFIRMED**：corruptRecovery 连续失败的具体告警升级链路（控制台已有告警通道，但"3 次"阈值与人工 SOP 属实施细节）。

---

## 五、C · 重复 reset 提交的请求级幂等

### 5.1 现状

| 事实 | 证据 | 级别 |
|---|---|---|
| 全仓无 `requestId` / 任何请求级幂等标识 | grep `requestId|idempoten` 零命中 | CONFIRMED |
| reset 端点以 body.key 驱动，重复 POST = 重复执行 | `:4587-4627`（每次都整批重跑） | CONFIRMED |
| credential.rotate 幂等按 opId——但 opId 当前无请求级来源 | `04` §4.4 | CONFIRMED（机制在，请求侧缺位） |

### 5.2 设计（DESIGN RECOMMENDATION，D-2）[Correctness]

1. reset 请求新增**可选** `requestId` 字段（客户端生成，如 UUID）；**缺省 = 不幂等**（行为等同新操作）。
2. 服务端将 `requestId → batchId → 终态结果（ok/failed + per-key 结果）` 记入 reset journal（追加行）。
3. **同一 requestId 重放**：batch 已完成 → 直接返回记录结果，**不再次轮换**；batch 进行中 → 返回进行中状态；batch 失败 → 返回失败（客户端可换新 requestId 重试）。
4. **新 requestId** = 新操作 → 新 batchId → 新一轮轮换（用户有意重置的语义保持，D2）。
5. per-key `credential.rotate` 的 opId 由 `batchId+key` 派生 → rotate 幂等按 opId 的既有设计（`04` §4.4）无缝衔接。
6. **保留期**：幂等记录随 reset journal 存续，≥ 7 天（与 `08` 阶段 2 观察窗一致）；journal 压缩时**必须保留**：未完成 batch 的全部记录 + 已完成 batch 最近 N 条（建议 N=50）+ 全部 requestId 索引行——**压缩不得丢弃幂等索引**。
7. 边界：requestId 只保证"同请求不重复执行"，**不提供**跨请求去重（两次不同 requestId 的 reset 就是两次重置——D2 语义）。

> 现状无此机制，不得宣称重复提交已被防护——现状重复 POST 就是重复整批重跑（含再次写空文件）。

---

## 六、D · 批次意图的持久性与不可变 key 清单

### 6.1 现状

`/api/socialV2/reset` 在**请求处理时**实时枚举 `[...socialV2.conversations.keys()]`（`:4592`），无任何持久化意图记录。`CONFIRMED`。

### 6.2 七问设计（DESIGN RECOMMENDATION，D-3/D-4）[Correctness]

| # | 问题 | 规则 |
|---|---|---|
| 1 | intent 持久化与首个 per-key effect 的顺序 | **intent 追加写入 journal → 成功后才允许任何 rotate**（write-ahead 硬序） |
| 2 | intent 写失败 | **整批 fail-closed**：零副作用、请求以错误返回（与 21 号恢复矩阵 #1 同型） |
| 3 | 如何防重复 intent | 由 C 的 requestId 幂等承担：同 requestId 不会产生第二个 batchId |
| 4 | `keys[]` 是否冻结 | **是**：请求接受时刻枚举快照，写入 intent 后不可变 |
| 5 | 批次进行中新建的会话能否入批 | **不能**：新会话创建于 intent 冻结之后，且被 sessionEpoch 屏障（`:4588` bump）排除——它属于"reset 之后的世界" |
| 6 | 恢复如何判定完成/未完成 | **差集判定**：`intent.keys` − {journal 中有 per-key done 记录的 key} = 待续跑集合；**禁止重新枚举当前会话来扩大/缩小批次**（重建的会话不属于原批次） |
| 7 | 同一批次再次提交 | 同 requestId → 返回记录结果（C）；不同 requestId → 新批次新轮换（D2） |

**权威重建规则一句话（D-4）**：**批次的目标集合与进度只存在于 reset journal；恢复 = intent.keys 减去 done 差集的前滚，绝不重新枚举。**

---

## 七、E · 凭据注入边界的全覆盖核查

### 7.1 `api.sessions.prompt` 的全部调用点（本轮枚举，`CONFIRMED`）

| 调用点 | 归属 | 会话类型 | 是否涉及令牌 | 定性 |
|---|---|---|---|---|
| `:9543` | `deliverPromptNow`（`:9483`） | **reserved2 QQ 用户会话**（notify-ai `:4379`、唤醒 `:10847`、防永眠提醒 `:11932`、摘要投喂 `:9773`、离线重投 `:2845` 全部经 `deliverPrompt → :9543`） | **是**（当前：构造期嵌入，`:4363`/`:10674`/`:10751`） | **注入边界所在** |
| `:11527` | v1（一代仿真）消息直投 | QQ 用户会话，但 **v1 模式无令牌机制**（tokenLine 仅 `isV2` 注入，`:4361-4364`；v1 prompt 从不含令牌） | 否 | **豁免路径①**（显式声明） |
| `:2170` | `runSlangExtraction`（内部黑话学习会话，`ensureSlangLearnerSession`） | 内部辅助会话，非 QQ 会话 | 否 | **豁免路径②**（显式声明） |
| `:2233` | `runSlangResearch`（同上） | 同上 | 否 | **豁免路径②** |

另核：`/api/socialV2/initial-prompt`（`:5023-5076`）**只存/取文本，不投递**——不是 prompt 路径。`CONFIRMED`。

### 7.2 队列载荷语义（逐问）

| 问题 | 现状 | 级别 | 目标态（C-6） |
|---|---|---|---|
| 队列存裸 prompt 还是已注入 token？ | **已注入 token 的完整 promptText**（`:9504` 存的字符串含 `:10674` 嵌入的令牌行） | CONFIRMED | 存**无凭据裸 prompt** |
| 重试/重连会发旧凭据吗？ | **会**：离线队列重投（`:2845` flushQueue）原样发送存储文本 → 旧 token | CONFIRMED | 不会（投递时注入当前值） |
| 重复投递会累积重复令牌行吗？ | 现状不会（令牌在构造期嵌入一次；重投发送同一字符串） | CONFIRMED | 目标态同样不会（每次发送注入恰好一次） |
| stored payload 是否会进 ledger/dump/日志 | prompt-dump 的 meta/structure 不含正文（`04` §7）；ledger 只记用量 | CONFIRMED | 保持 |

### 7.3 结论与边界定义（修正 22 号 C-6 的表述）

> **C-6（精确化）**：凭据注入边界 = **"deliverPromptNow（reserved2 QQ 会话的唯一出口）"**，但必须同时**显式声明两条豁免路径**（v1 直投 `:11527`、内部学习会话 `:2170/:2233`）——它们无令牌机制，不受边界约束，若未来 v1/内部会话引入凭据则必须收敛进边界。
> 注入规则五条：①基础 prompt **裸存**（队列/journal/dump 均无凭据材料）；②注入发生在 `api.sessions.prompt` 调用前一刻；③发送时刻读当前 epoch；④**所有 reserved2 QQ 投递路径共用同一注入实现**（现状已天然满足：五条上游全部经 `deliverPrompt → deliverPromptNow`，`CONFIRMED`）；⑤每次发送注入恰好一次（不存在重复令牌行）。
> 级别：①②③⑤ [Correctness]；"单一实现"陈述为 [Ops]。

---

## 八、22 号 C-1～C-10 一致性复审

| 约束 | 判定 | 理由 | 必要措辞 |
|---|---|---|---|
| C-1 内存唯一真相+同步重建 | **Keep** | F-8 证据链完整；`:4624` 消灭路径不变 | 无变化 |
| C-2 getSocialV2State 单点 fencing | **Revise** | 从"逐处理器 helper"改为"`getSocialV2State` 内置 `expectEpoch` 可选参数"（22 号已建议单点，23 号确认其可行且覆盖全部调用点） | "fencing 经由 getSocialV2State 唯一入口生效；ensureSession 的既有检查仅覆盖会话创建路径" |
| C-3 前滚+credential 权威 | **Revise** | 双损时事务面**不可自动恢复**——"确定性自愈、服务不停机"只对认证面成立 | "认证面=corruptRecovery 确定性自愈；事务面=受控恢复（人工逐 key）；corruptRecovery 连续失败→二代工具面 fail-closed 403" |
| C-4 批次可从 journal 重建 | **Keep + Extend** | 补 requestId 幂等（D-2）与 intent 硬序（D-3/D-4） | 增"intent 先于首 effect；keys 冻结；恢复禁止重枚举" |
| C-5 清理绑定捕获 sid | **Keep** | `:2734-2741` 证据完好 | 无变化 |
| C-6 投递层注入 | **Revise（精确化）** | "deliverPromptNow 唯一边界"不成立（4 个调用点）；正确表述=reserved2 QQ 唯一出口 + 两类显式豁免 | 见 §7.3 |
| C-7 锁序/锁内禁网络 | **Revise** | "SocialV2 变更必须取锁"过强：同步变更零锁、跨 await 变更 fencing | 见 §3.4 三级分层 |
| C-8 删除 `:4624` 直写 | **Keep** | F-8 唯一违规者 | 无变化 |
| C-9 await 型处理器枚举 | **Keep** | send-message 实证该类存在（`:5243→:5310`），mark-read 实证同步类存在 | 配方照旧 |
| C-10 队列裸存 | **Keep** | 现状 `:9504` 存已注入文本（含 token）——正是要修的 | "队列/journal/dump 三处都不得出现凭据材料" |

**无静默变更**：C-3/C-6/C-7 的修正均在 22 号文本方向内精确化，且每条标注了修正原因。

---

## 九、合并后的最终设计约束（R2 修订输入）

沿用（修订版）：C-1（不变式）、C-2（单点 fencing）、C-3（受控双损恢复）、C-4（journal 重建+幂等扩展）、C-5（sid 绑定）、C-6（投递层注入+豁免声明）、C-7（三级锁策略）、C-8（消灭直写）、C-9（枚举清单）、C-10（三处无凭据）。

新增（本轮）：

| # | 约束 | 级别 |
|---|---|---|
| **D-1** | v1 直投路径（`:11527`）与内部学习会话（`:2170/:2233`）作为**显式豁免路径**写入边界定义；引入凭据时必须收敛 | [Correctness] |
| **D-2** | reset 请求级幂等：requestId → batchId → 记录结果重放；缺省不幂等；幂等索引随 journal 存续 ≥7 天，压缩保留 | [Correctness] |
| **D-3** | resetTx intent 硬序：intent 持久化成功先于任何 effect；写失败=整批零副作用 | [Correctness] |
| **D-4** | `keys[]` 请求接受时冻结；恢复差集前滚、禁止重枚举 | [Correctness] |
| **D-5** | 三级锁策略成文（同步零锁 / 跨 await fencing / 多 phase 锁） | [Correctness] |
| **D-6** | 双损=受控恢复；corruptRecovery 连续失败 → 二代工具面 fail-closed | [Correctness] |
| **D-7** | 初始提示词端点（`:5023`）永久排除出投递边界（只存不投） | [Defense] |
| **D-8** | 压缩器的幂等索引保留规则（D-2 第 6 条） | [Ops] |

---

## 十、仍未确认事项

| # | 事项 | 缺什么 |
|---|---|---|
| 1 | DSH `steer` 模式（`:11531` 私聊）对轮换期消息的处理细节 | DSH 侧源码；不影响边界设计（v1 无令牌） |
| 2 | corruptRecovery 连续失败的告警升级 SOP 细节 | 实施细节（控制台告警通道已存在） |
| 3 | 客户端（MCP 工具侧）如何传递 requestId | R2 接口定义时定 |
| 4 | V-7 / V-8 | 维持既有门禁 |

## 十一、R2 修订指令（汇总）

1. `04` §3：注入边界按 §7.3 重写（单出口+双豁免+五条注入规则）；§5.5 补 D-2/D-3/D-4 幂等与 intent 硬序；§6.2-4 补"双损=受控恢复"与 fail-closed 分级。
2. `05`/`08`：锁策略按 §3.4 三级分层改写（C-7 修订版）。
3. `10`：M07 行补 requestId 幂等与 intent 持久化的估算。
4. `14`：新增用例——`RESET-06`（同 requestId 重放不轮换）、`RESET-07`（intent 写失败零副作用）、`CRED-12`（双损→受控恢复演练）。
5. `16`：§四 门禁维持；本轮无新增事实门禁（D-1～D-8 为设计约束非事实缺口）。
6. `17`：D2 落点引用更新（`04` §5.5 幂等段）。

## 十二、最终就绪度评估

**PASS WITH CONDITIONS。**

理由：五项边界均得到显式、内部一致的设计决策；对 22 号的三处修正（C-3 受控恢复、C-6 边界精确化、C-7 三级锁）都是**收紧或精确化**，无方向翻转；新增事实（4 调用点、无请求幂等）均已被设计决策吸收。

**R2 起草前的剩余条件**：
1. 本报告 D-1～D-8 与 C 系修订版并入 R2 文档集（§十一 的 6 条指令）；
2. V-7（40 轮实测，实施后）/ V-8（DSH usage 缓存字段，实施前只读核查）关闭；
3. D-30 拍板（阻塞 M2，不阻塞 M0/M1）；
4. §十 的 4 项 UNCONFIRMED 中，第 3 项（requestId 客户端传递）需在 R2 接口定义时敲定。

**R2 起草完成、评审通过 ≠ 源码实施授权。**

---

> 本轮仅创建本报告于 `D:\v3-memory-v2-design-final-decisions-r2-audit\`；未修改 R1 目录、19–22 号报告、任何源码/配置/state/记忆数据/Persona/插件；未调用 reset 端点、未模拟会话清理、未迁移、未业务写入、未启停服务。报告不含真实凭据、令牌、私有 prompt 正文或个人会话内容。
