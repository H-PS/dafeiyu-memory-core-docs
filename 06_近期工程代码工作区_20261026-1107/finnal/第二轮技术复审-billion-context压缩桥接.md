# 第二轮技术复审报告：billion-context 压缩桥接（Compression Memory Bridge）

> 性质：**技术方案复审**。非实施授权、非设计批准、非 FUT-09 批准。
> 复审对象：`E:\[PROJECT]\压缩内容分流进记忆库-可行性方案.md`（第一轮）
> 代码基线：`E:\[PROJECT]\refs\billion-context`，HEAD `0aace2f`（2026-11-04），2061 文件，静态阅读
> Memory Core 基线：`C:\Users\[USER]\Desktop\work\R4.1-memory-bank-input`（只读）
> 本轮**未修改任何文件**（源文件、配置、记忆数据、原始文档）；仅新增本报告。
> 日期：2026-11-05

---

## 0. 复审结论摘要

第一轮方案有 **1 处根本性概念错误、3 处事实错误、1 处凭空发明的 API、1 项核心设计需要推翻**。

| 类别 | 结论 |
|---|---|
| 插入位置本身 | **可用**，但性质必须重新定义：它是「内存提交后、持久化提交前」的第三种状态，不是"提交前"也不是"提交后" |
| `ctx.session.state = res.state` | **不是持久化提交**。压缩成功路径不触发 `markDirty`，落盘靠 500 ms 防抖，另有完全独立的 SQLite 事务路径 |
| `protectedMessageIds` 闭环 | **必须推翻**。它与压缩目标直接冲突，且身份空间是 REF 不是 id，且不是跨调用持久属性 |
| 伪代码里 `ctx.session.tenantId` | **凭空发明**。`Session` 无此字段，违反"不得凭空创造 API" |
| Evidence 契约可用性 | **硬阻塞**。`evidence.schema.json` 缺 `original_text`/`speaker_id`，`additionalProperties: false`，现有 schema **无法承载**任何合规 Evidence |

---

## 1. 对审查问题一的回答：提交时序与可靠交付

### 1.1 该位置究竟属于提交前还是提交后？

**两者都不是。** 第一轮用"提交前/提交后"二分法描述是错误的，实际存在三个状态：

| 状态 | 位置 | 语义 | 崩溃后果 |
|---|---|---|---|
| ① 判定完成 | `stream.ts:1183` `applyCompression` 返回 `res` | 纯函数算出新 state，**无副作用** | 无 |
| ② **内存提交** | **`stream.ts:1287` `ctx.session.state = res.state`** | 仅替换进程内对象字段 | **折叠完全丢失**，内存视图回滚到未折叠 |
| ③ **持久化提交** | `scheduleSave` → debounce 500 ms → `writeNow` → `ingestSession`（SQLite 单事务） | 真正落盘 | 无（已落盘） |

第一轮推荐的 `stream.ts:1309–1321` 落在 **② 之后、③ 之前**。

### 1.2 `session.state` 更新是否意味着持久化提交？

**否。这一点有明确代码证据，且与第一轮的表述冲突。**

证据链：

1. `stream.ts:6` 从 `session.ts` 导入 `markDirty`；全文 `markDirty` 仅出现于 **3 处**：`L197`、`L205`、`L242`。
2. 这 3 处全部位于 **压缩失败计数 / 循环守卫的簿记**（`FAIL_STREAK_KEY`、`COMPRESS_LOOP_KEY`）。
3. **压缩成功路径（`L1287`–`L1360`）内没有任何 `markDirty`**（已逐行扫描确认）。
4. `session.ts:1610–1615` 定义：
   ```ts
   /** Mark a session's state as changed so it is persisted on the next debounce.
    *  Call this after any mutation (processTurn, compress, decompress, orphan GC). */
   export function markDirty(session: Session): void {
       session.revisionEpoch = (session.revisionEpoch ?? 0) + 1;
       getStore().scheduleSave(session);
   }
   ```
5. 落盘节流：`knobs.ts` `persistDebounceMs()` 默认 **500 ms**（`BILI_PERSIST_DEBOUNCE_MS > persist.debounceMs > 500`）。
6. 真正的持久化事务在 `storage/store.ts:832 ingestSession(...)`，`L838` 为 `this.db.transaction(...)`，**messages 行与 blocks 行在同一事务内**（`L1178` 块 INSERT 就位于该事务中）。

**推论**：`L1287` 只是把新 state 挂到内存对象上。若进程在 ②→③ 窗口内被 SIGKILL，**折叠从未落盘**，重启后该会话回到未折叠形态。

值得注意：这套代码自己把这个失败模式记录成了事故签名。`persist.ts:1623` 附近 `logColdLoad` 注释原文：

> `refs>0` with zero blocks is the incident signature (#3573): identity loaded but every fold missing, so the full history replays and preflight must rebuild from scratch.

### 1.3 旁路投递失败是否会造成已折叠消息没有 Evidence？

**会，且是双向的，这比第一轮描述的更严重。**

| 崩溃点 | 折叠（上下文） | Evidence（记忆库） | 后果 |
|---|---|---|---|
| 投递后、`markDirty` 前 | ❌ 丢失 | ✅ 存在 | **孤儿证据**：记忆库声称"这段被折叠过"，但上下文从未折叠 → 违反 `DESIGN-STATUS` 账实一致 |
| 折叠落盘后、投递前 | ✅ 存在 | ❌ 缺失 | **证据缺口**：原句被折叠但无 Evidence，`related_evidence` 反查断链 |
| 投递中崩溃（写一半） | ✅ | ⚠️ 部分 | 半条证据，`content_hash` 校验可能通过但 `evidence_refs` 不完整 |

**关键判断**：只要"投递"与"落盘"是两个独立写者、两个独立事务、且中间隔着 500 ms 防抖窗口，**就不存在把三者绑成一个原子的办法**（除非改 `ingestSession` 事务——本轮明令禁止）。

### 1.4 是否需要 Transactional Outbox / 持久化队列？

**需要 outbox 语义，但不需要新建队列。** `index.db` 本身已经是 outbox。

理由（全部为静态证据）：

- `storage/store.ts` 文件头：`One SQLite database (index.db, WAL) + content-addressed blobs`
- `blocks` 表列：`block_id, session_id, ord, start_seq, end_seq, start_ref, end_ref, tier, summary_hash, one_hash, supersedes_block_id, active, created_at, covered_hash, payload_json`（`L1178–1180`）
- `messages` 行带 `contentHash` 与 `retiredAt`（`L2983` `getMessages` 返回 `MessageRow & { contentHash: string; retiredAt: number | null }`）
- 写入是**单事务**（`L838`），所以 "blocks 行 + 它覆盖的 messages 行" 天然原子

因此：**消息与折叠的落盘已经原子；Evidence 只是这份原子数据的派生物。** 正确做法不是"在压缩路径里同步投递"，而是**读侧对账**：

> 以一个外部只读连接扫描 `index.db`，找出「已落盘、但尚未桥接」的 block，逐条生成 Evidence，写成功后记录 `(session_id, block_id, covered_hash)` 到记忆库侧的对账表。重复扫描幂等。

这样：
- 崩溃窗口不再产生孤儿证据（未落盘的折叠不会被扫到）
- 证据缺口变为**可观测的延迟**而非静默丢失
- 不需要改任何源码，不需要新队列

**但有两个前提必须验证**（见 §8）：默认存储模式是 `full`（`knobs.ts:869–881` 缺省返回 `"full"`，即 `index.db` 是加载路径），**但 `BILI_ENCRYPTION_KEY` 生效时 unified sidecar 会被禁用**（`persist.ts:562–568`：`unified sidecar DISABLED: BILI_ENCRYPTION_KEY is active and index.db would hold cleartext ingest payload`）。若用户开启加密，此方案失效。

### 1.5 如何保证压缩提交与 Evidence 投递之间的一致性？

**答：不能保证强一致，只能保证最终一致 + 缺口可观测。** 这是本轮的明确结论，不是妥协措辞。

可达成的一致性等级：

| 等级 | 是否可达 | 手段 |
|---|---|---|
| 强一致（同一事务） | ❌ 禁止（须改 `ingestSession`） | — |
| 顺序一致（先折叠后证据） | ✅ | 对账器严格按 `start_seq` 升序、且只扫已落盘 block |
| 最终一致 + 幂等 | ✅ | 对账表 + `(session_id, block_id)` 唯一键 |
| 缺口可观测 | ✅ | 对账器每次运行输出「已落盘未桥接」计数，>0 且超时即告警 |

**第一轮"必须发生在提交之前"的说法应废弃**，正确的表述是：

> Evidence 必须由**已持久化的折叠**派生，而不是由**内存中的折叠**派生。派生时刻可以在压缩之后任意晚；关键是数据源必须是 `index.db`，不是 `ctx.messages`。

---

## 2. 对审查问题二的回答：`protectedMessageIds` 设计复审

### 2.1 事实纠正（第一轮两处不准）

| 项 | 第一轮说法 | 实际 |
|---|---|---|
| 身份空间 | "需做 id → ref 转换" | **正确但不够**。`unit-projection.ts:261–263` 原文：`protectedMessageIds holds REF ids (mNNNNN) from computeProtectedRefs; filteredIds holds RAW message ids, so convert via state.messageRefs.byRaw` |
| 持久性 | 隐含为可跨轮使用 | **错**。它是**单次 `applyCompression` 调用的入参**（`compress.ts:150`），不进入 state，**不是消息的持久属性**。下一轮调用若不传就等于不存在 |

### 2.2 逐项分析用户提出的五个区分

| 维度 | 该不该用 `protectedMessageIds` | 理由 |
|---|---|---|
| **长期证据保护** | ❌ 不该 | 证据的长期性由 Evidence 表 + `retention_policy` 承担，与"上下文是否折叠"无关。绑定二者等于用上下文结构表达记忆语义 |
| **当前上下文保护** | ✅ 这是它唯一正当用途 | 它本来就是给"本轮压缩不得吞掉的窗口内容"用的（`#3953`：保护消息若在 range 内部，会导致摘要锚点落在它所回答的问题之前） |
| **原始消息压缩** | ❌ 语境错误 | 它只能让某些消息**不被压缩**，无法表达"压缩后仍可反查" |
| **历史恢复** | ❌ 无效 | 历史恢复走 `decompress` / `acp_retrieve`，与 protected 无关；且 `protectedMessageIds` 不进 state，重启即消失 |
| **重复投递** | ❌ 完全无关 | 重复投递是幂等键问题，用上下文保护机制去防重复是错配 |

### 2.3 是否与上下文压缩目标冲突？

**会，而且是结构性冲突。** 逻辑推演（基于代码事实，非猜测）：

1. 若"已进入 Memory Core"的消息被永久保护，则该消息**永远不可折叠**。
2. 记忆库随时间单调增长 ⇒ 不可折叠集合单调增长。
3. 于是上下文可压缩余量单调减少 ⇒ **压缩能力随时间衰减至零**。
4. 最终结果：上下文持续增长，而压缩层失效——**正好是第一轮想避免的那个问题**。

这就是第一轮 §3.4 建议"闭环"的致命缺陷：**它把"记忆摄取"变成了"上下文增长的许可证"。**

### 2.4 替代机制与正确的保护对象

**应先纠正一个认识**：内核里另有更贴切的机制，且它不是为记忆设计的。

`kernel/src/types.ts`（`#2419`）提供了宿主级谓词：

```ts
/** Host-declared message-level hard protection (billion-context#2419):
 *  when set, ANY message for which the predicate returns true is excluded
 *  from compression exactly like a protectedTools hit — BLOCKED ref in
 *  assignRefs (never advertised, never foldable-by-ref) AND dropped from
 *  effectiveMessageIds in applyCompression ... Use for host-injected
 *  DURABLE STATE carried as ordinary conversation messages — e.g. dsh's
 *  skill catalog ... */
isMessageProtected?: (msg: CoreMessage) => boolean;
```

**正确的保护对象**（按该注释的原始意图）：**"丢失即永久失去可见性"的宿主注入持久态**——人格锚点、技能目录、当前任务指令。这类内容折叠后无法从别处恢复，因此必须常驻可见。

**记忆内容不在其中**：记忆的长期副本在 Memory Core，上下文中丢失它**不构成信息损失**，只构成一次检索。这正是 13 号 §7.3 D-02「优先引用而非复制」的同一逻辑：**上下文的职责是"当前"，记忆的职责是"长期"。**

**修正结论**：

> 不得以"已入记忆库"为条件保护任何消息。`protectedMessageIds` 只用于单次调用的窗口保护；`isMessageProtected` 只用于宿主注入的持久态。两者都**不得**与记忆摄取状态耦合。

---

## 3. 对审查问题三的回答：幂等与身份体系

### 3.1 内核侧真实身份（无一凭空假设）

| 身份 | 是否存在 | 证据 | 稳定性 |
|---|---|---|---|
| `Session.id` | ✅ | `session.ts:347` | 会话级稳定 |
| `Session.meta.*` | ✅ | `session.ts:349`（protocol/upstreamOrigin/label/title） | **无 group / tenant / bot** |
| `CoreMessage.id` | ✅ | `kernel/src/types.ts` `CoreMessage` | raw id，`id-never-reused`（content-store 头注释） |
| `CoreMessage.ledgerSeq` | ✅ | 同上（`#4316`） | 会话内单调，**ingest 时分配一次** |
| `ledgerSeq` 与 `seq` 关系 | ✅ | `types.ts` 注释：`ref = store row seq + 1`（`#4316` layer 3） | 可换算 |
| ref `mNNNNN` | ✅ | `assignRefs` | **可重置**（generation boundary → `m00001`） |
| `block_id` `bN` | ✅ | `CompressionBlock.blockId`；`blocks.block_id` | 会话内唯一，**重算可重建** |
| `content_hash` | ✅ | `storage/store.ts` `MessageRow & { contentHash }`；`blocks.covered_hash` | 内容级稳定 |
| `start_seq` / `end_seq` | ✅ | `blocks` 表列 | **权威区间** |
| `tenant_id` / `bot_id` / `group_id` | ❌ **内核完全没有** | `session.ts:307` `Session` 类型无此字段 | — |

**第一轮错误**：伪代码写了 `tenantId: ctx.session.tenantId`、`sessionKey: ctx.session.key`。两者都不存在。这是凭空发明 API，予以撤回。

### 3.2 幂等键的推荐构造

```
bridge_key = sha256( session_id ‖ block_id ‖ covered_hash )
```

- **不用 ref**：generation boundary 会重置到 `m00001`（`content-store.ts` 头注释明确 generation boundary 语义），ref 复用会导致错配。
- **不用 `block_id` 单独**：`block_id` 在会话内唯一但可因重算重建；叠加 `covered_hash` 才能区分"同一编号不同内容"。
- **不用 `message_id` 单独**：同一消息可被多个 block 覆盖（嵌套），会重复。
- **必须带 `session_id`**：跨会话同 `block_id`（都是 `b1`）必然碰撞。

### 3.3 六类场景逐项处置

| 场景 | 风险 | 处置 |
|---|---|---|
| **ref 重置**（generation boundary） | ref 复用 → 错配 | 全程不落 ref，只用 `seq` / `content_hash` |
| **session 重建**（rebase / fork） | 新 session 重放历史 → 重复摄取 | `session_id` 变更即视为新主体；fork 场景须显式继承白名单，否则不桥接 |
| **block 重建**（refold） | 同区间新摘要 → 是否算新证据 | `block_id` 相同则视为**更新**（`supersedes_block_id` 列已存在），只追加新 summary 版本，不重复存原句 |
| **嵌套 block** | `effectiveMessageIds` 含子块 → 父子重叠 | **关键缺陷，见 §4.1**。必须改用 `directMessageIds` 或全局 message 级去重 |
| **重复覆盖**（同一消息被两次折叠） | 双份 Evidence | 以 `(session_id, message_id)` 为原句去重键，`content_hash` 二次校验（对齐 13 §7.4.2） |
| **同文本不同消息**（用户重复说同一句） | 哈希碰撞成一条 | 13 §7.4.2 要求"命中既有未撤销**且 `source_id`+`source_locator` 可对齐**的哈希才复用"。故去重键须 `(content_hash, source_locator)` |

---

## 4. 对审查问题四的回答：Evidence 数据完整性

### 4.1 `collectBlockContent(full=true)` 是否提供完整、无重复的原始消息？

**不完整、且会重复。这是第一轮的实质性遗漏。**

代码（`kernel/src/decompress.ts:176–221`）：

```ts
const targetIds = new Set(block.effectiveMessageIds);   // L183
if (full) {
    const msgs = messages.filter((m) => targetIds.has(m.id));   // L186
    return { text: msgs.map(formatMessage).join("\n\n"), count: msgs.length };
}
```

而 `CompressionBlock` 同时有 `directMessageIds` 与 `effectiveMessageIds`（`types.ts`），后者是**传递闭包**（含所有嵌套子块的覆盖 id）。因此：

- 父块 `full=true` → 返回**全部后代**消息
- 子块 `full=true` → 返回**自身**消息
- **两者交集非空 ⇒ 逐块投递必然重复**

另外，`full=false` 分支（`L212`）才用 `nestedCovered` 排除子块，`full=true` 分支**没有任何排除**。

**修正**：桥接不得用 `full=true` 做逐块投递。应：
1. 以 `directMessageIds` 为每块的自有集合；或
2. 建全局 `message_id → 最内层 block_id` 映射，每条消息只投一次。

### 4.2 reasoning 排除规则

**`collectBlockContent` 不排除 reasoning。** `formatMessage`（`L223–229`）对所有消息一视同仁。

排除逻辑在**另一个函数**里：`kernel/src/ccr.ts:606 storeCoveredOriginals`，其中

```ts
if (message.contentType === "reasoning") continue;   // L628
```

**第一轮把两者的行为混为一谈，此处纠正。** 桥接必须自己实现 reasoning 排除，不能指望 `collectBlockContent`。

### 4.3 tool 消息与 assistant 消息的来源归属

`formatMessage` 输出：

```ts
if (message.toolName && message.contentType !== "text")
    return `[${message.role} • ${message.toolName}]\n${text}`;
return `[${message.role}]\n${text}`;
```

**它不输出 `message.id`、不输出 `toolCallId`、不输出时间戳。** 所以：

- **不能**从拼接文本反解身份——必须直接用 `CoreMessage` 对象逐条构造 Evidence
- tool 调用与结果的配对需借 `toolCallId`（`storeCoveredOriginals` 里 `callArgsById` 就是这么做的，`ccr.ts:620–625`）
- 第一轮伪代码把 `collectBlockContent(...).text` 当作"原句"传下去，**会丢身份**，属设计错误

### 4.4 `sender` / `channel` / `group_id` 如何补充

**内核零支持，必须宿主提供**，且 04 §7.6 第 5 条要求：

> 身份字段 MUST 由可信 Host Adapter 构造，MUST NOT 信任 LLM 自造身份。

| 字段 | 来源 | 状态 |
|---|---|---|
| `role`（user/assistant/tool） | `CoreMessage.role` | ✅ 内核 |
| `toolName` / `toolCallId` | `CoreMessage` | ✅ 内核 |
| `message_id` | `CoreMessage.id` | ✅ 内核 |
| 时间 | `state.tsStamps` / `blocks.created_at` | ✅ 内核（`#4313` 不可变戳） |
| **`speaker_id`** | 宿主身份映射 | ❌ 须宿主 |
| **`group_id` / `bot_id` / `tenant_id`** | QQ Bridge / DSH | ❌ 须宿主，且**必须经 VERIFY** |

**注意**：`evidence.schema.json` 中**没有 `speaker_id` 字段**（见 §5.3），这使问题更复杂。

### 4.5 时间戳与消息顺序如何保持

- 顺序：`ledgerSeq`（= `seq`）单调递增，可靠；`messages` 数组顺序亦为原始顺序。**不得按 `block_id` 或 summary 重组顺序。**
- 时间：`#4313` 不可变戳，`render 路径只读不写 Date.now()`（`types.ts` 注释），故可信任；缺失时为 `null`，按 13 §7.2 第 2 条 **MUST NOT 填推测时间**。

### 4.6 空视图折叠

`stream.ts:1309` 已有处理：

```ts
const contentSource = ctx.messages.length > 0 ? ctx.messages : (ctx.compressMessages ?? ctx.messages);
```

`#3982` 场景（EMPTY-view fold 从统一存储恢复源）下 `ctx.messages` 为空，须回退 `ctx.compressMessages`。**第一轮说"照抄即可"是对的**，予以保留。但若走 §1.4 的读侧对账方案，此问题自动消失——因为数据源是 `index.db` 而非内存视图。

---

## 5. 对审查问题五的回答：Memory Core 兼容性

### 5.1 两个 `source_type` 字段（第一轮用错枚举）

13 号 §7.6.3 明确二者**不同字段、不同枚举、MUST NOT 混用**：

| 字段 | 枚举 | 语义 |
|---|---|---|
| `MemoryUnit.source_type` / `EmotionContext.source_type` | **5 值**：`USER_EXPLICIT` / `USER_INSTRUCTION` / `ASSISTANT_INFERRED` / `EXTERNAL_SOURCE` / `SYSTEM_GENERATED` | 来源**性质** |
| `Evidence.source_type` | **5 值**：`chat` / `file` / `import` / `tool` / `system` | 证据**形态** |

**第一轮错误**：我在报告里写"摘要的 `source_type` 必须是 `ASSISTANT_INFERRED`"，未区分是哪个字段。正确表述应为：

- `MemoryUnit.source_type = ASSISTANT_INFERRED`（或 `SYSTEM_GENERATED`）
- `Evidence.source_type = system`（摘要作为派生视图）或无 Evidence（摘要不应产生 Evidence，见 §5.2）

### 5.2 摘要能不能成为 Evidence？

**不能，且有三条独立禁令。**

| 禁令 | 出处 | 原文 |
|---|---|---|
| P-02 | 13 §7.3 | 把 Agent 总结 / 检索结果伪装成原始证据 |
| P-08 | 13 §7.6.2 | 把 `ASSISTANT_INFERRED` 的 MemoryUnit 作为 `original_text` 返回 |
| P-05 | 13 §7.3 | 因"节省 Token"删除原句中的否定词 / 时间条件 |

**P-05 是决定性的**：压缩摘要的生成目标本身就是"更短"，`compression-rules.ts` 的 `COMPRESS_PHILOSOPHY` 明确"compress by need, not by percentage"，但仍然是改写。**改写必然可能丢否定词/时间条件**——这正是 P-05 禁止的。因此摘要**结构上不可能**满足 `original_text` 的"不得规范化覆盖"要求。

**结论**：摘要只能作为 **M03 候选的输入素材**，`source_type` 标 `SYSTEM_GENERATED`，**永不进入 Evidence 表**。原句必须直接取自 `index.db` 的 messages 行。

### 5.3 硬阻塞：`evidence.schema.json` 与权威文档冲突

这是本轮最重要的独立发现，**与 billion-context 无关，是既有缺陷**。

| 项 | 权威（DD-AUTHORITY §3.2 / 13 §7.2） | 契约 `evidence.schema.json` | 判定 |
|---|---|---|---|
| `source_type` 枚举 | 5 值 `chat/file/import/tool/system` | **12 值** v1 枚举（`direct_user_statement` … `human_reviewed`） | ❌ 冲突 |
| `original_text` | **必填 ✓**，"MUST NOT 被规范化覆盖" | **字段不存在** | ❌ 缺失 |
| `speaker_id` | **必填 ✓** | **字段不存在** | ❌ 缺失 |
| `timestamp` | 有 | 为 `observed_at`（非必填） | ⚠️ 命名不一致 |
| `visibility` | 必填 ✓（enum §2.10） | 为 `visibility_scope`（string，非必填） | ⚠️ 类型/必填不一致 |
| `tenant_id` | 权威表未列 | **必填** | ⚠️ 契约更严 |
| `additionalProperties` | — | **`false`** | 🔴 致命 |

`additionalProperties: false` 意味着：**按权威文档构造、带 `original_text` 与 `speaker_id` 的 Evidence 对象，会被该 schema 直接判定非法。** 且 `source_type: "chat"` 会被 12 值枚举拒绝。

已用程序核验（非目测）：

```
additionalProperties = False
required = evidence_id, source_type, source_id, tenant_id, scope, content_hash, recorded_at
has original_text: False
has speaker_id: False
```

反观 `memory-unit.schema.json` 的 `source_type` 为 5 值 `USER_EXPLICIT | USER_INSTRUCTION | ASSISTANT_INFERRED | EXTERNAL_SOURCE | SYSTEM_GENERATED` —— **与权威一致**。故问题**局限在 evidence 契约**，且方向明确：schema 停留在 v1（`DD-AUTHORITY` `L292` 记录着 v1 六值 → §2.9 五值的迁移映射，说明 v1 早已被取代）。

**处置建议（不由本报告批准）**：
- 记为 `DOCUMENT-CONFLICT-REGISTER-V2` 的新条目，与既有的"文档数量不一致"并列
- **P0 不得在 schema 未裁决前实现**，否则要么写出非法对象，要么被迫自造字段
- 三选一需用户裁决：改 schema / 改文档 / 登记为例外并冻结

### 5.4 M03 / M04 权限边界（严格照既有规定）

| 规则 | 出处 | 对本桥接的约束 |
|---|---|---|
| 候选由 M03 产出，6 态 | 09 §7.21 | 桥接只**投递素材**，不产候选 |
| `confirmed` 仅 M04 唯一写入 | D-AP-05；09 §7.24 | 桥接 **MUST NOT** 写 `validation_state=confirmed` |
| `AUTO_ACCEPT ≠ confirmed` | 09；V2 §12 CR-01 | 不得把"自动通过"当确认 |
| Pipeline 是协调层，不持有重要性判断 | 09 §7.1 P-02 | 桥接 **MUST NOT** 自带"值得记住"启发式 |
| 情绪只在摄取期标注 | J2 / 09 §7.15 | 桥接不得做情绪推断 |
| LLM 调用须经 M04 Gateway | D-AP-02 | 桥接不得自建 LLM 通道（含"让模型挑哪些值得记"） |

**推论**：P0 阶段桥接的唯一合法动作是"把原句 + 定位写到 Evidence 表"，不含任何判断。任何"挑哪些进入记忆库"的筛选都必须由 M03 的既有组件做，且属 P1+。

### 5.5 `owner_private` 与 `group-shared` 隔离

04 §7.5 隔离禁令第 7、8 条：

> 7. 群聊记忆 MUST NOT 自动进入私人记忆；私人记忆 MUST NOT 自动进入群聊上下文。
> 8. 同一 Bot 的多个群聊 MUST NOT 因共享一个进程就共享全部上下文。

**这是本轮对第一轮 §3.3 的确认——第一轮判断正确，且比它说的更严重**：内核 `Session` 完全没有群概念（`session.ts:307` 逐字段核对），所以群身份**只能**来自宿主。若宿主拿不到可信群身份，桥接必须**整体关闭**，不能"降级为 session 粒度凑合"——那正好构成禁令 8 的违规。

---

## 6. 八项必需输出

### 6.1 上一轮方案的保留项

| # | 保留项 | 理由 |
|---|---|---|
| R-1 | **不改内核、不改压缩算法** | `content-store.ts` 头注释"zero I/O / core holds no state"是硬契约；`compress.ts` 2500 行 + 130 回归测试 |
| R-2 | **否决"包装 `applyCompression`"** | 四道回闸（scope / stub / coverage / no-net-shrink，`stream.ts:1154/1173/1204/1265`）可能拒绝，包装层写入会造成账实不符 |
| R-3 | **空视图 `contentSource` 三元处理** | `stream.ts:1309` 已正确，`#3982` |
| R-4 | **rebase 幂等键不得用 ref** | `content-store.ts` generation boundary 语义 |
| R-5 | **群隔离为硬门槛** | 04 §7.5-8 + `Session` 无群字段 |
| R-6 | **摘要不得标 `USER_EXPLICIT`** | 13 §7.6.1-3 / P-08 / P-09 |
| R-7 | **许可证与维护成本提示** | MIT + 附加署名条款；832 open issues |
| R-8 | **引用 `compression-rules.ts` 的伪造原话事故** | 该注释真实存在，可作 06 测试反面案例 |

### 6.2 上一轮的错误或不严谨项

| # | 错误 | 性质 | 纠正 |
|---|---|---|---|
| **E-1** | 声称"投递必须发生在压缩提交之前"，但推荐位置在 `state` 赋值之后 | **概念矛盾** | 三分法替代二分法：判定完成 / 内存提交 / 持久化提交（§1.1） |
| **E-2** | 把 `ctx.session.state = res.state` 当提交点 | **事实错误** | 非持久化；压缩路径无 `markDirty`；落盘走 500ms 防抖 + 独立 SQLite 事务（§1.2） |
| **E-3** | 伪代码 `ctx.session.tenantId` / `ctx.session.key` | **凭空发明 API** | `Session` 无此字段；身份须宿主提供并 VERIFY（§3.1） |
| **E-4** | §3.4 建议"已入记忆库 → 加 `protectedMessageIds`"闭环 | **设计错误，须推翻** | 与压缩目标结构性冲突；且身份是 ref、非持久属性（§2） |
| **E-5** | 称 `collectBlockContent(full=true)` 提供完整无重复原句 | **事实错误** | `effectiveMessageIds` 含子块 ⇒ 父子重复；`full=true` 无排除逻辑（§4.1） |
| **E-6** | 暗示 `collectBlockContent` 排除 reasoning | **事实错误** | 排除在 `ccr.ts:628`，不在 `collectBlockContent`（§4.2） |
| **E-7** | 用 `ASSISTANT_INFERRED` 指代 Evidence 的 `source_type` | **枚举混用** | 两个 `source_type` 字段不同枚举，13 §7.6.3 明令 MUST NOT 混用（§5.1） |
| **E-8** | 未发现 `evidence.schema.json` 不可用 | **重大遗漏** | 缺 `original_text`/`speaker_id`，`additionalProperties:false`（§5.3） |
| **E-9** | 把拼接文本 `text` 当原句传递 | **设计缺陷** | `formatMessage` 不含 id/toolCallId/时间，必须用 `CoreMessage` 逐条构造（§4.3） |
| **E-10** | 未识别"投递与落盘"跨事务窗口 | **遗漏** | 双向不一致风险（§1.3） |

### 6.3 必须修正的设计

| # | 修正 |
|---|---|
| M-1 | **数据源从 `ctx.messages` 改为 `index.db`**。桥接不再嵌入压缩热路径 |
| M-2 | **删除 §3.4 的 `protectedMessageIds` 闭环**；若需保护，改用 `isMessageProtected` 且只保护宿主注入持久态 |
| M-3 | **幂等键 `sha256(session_id ‖ block_id ‖ covered_hash)`**；原句去重键 `(content_hash, source_locator)` |
| M-4 | **逐块投递改用 `directMessageIds`**，或建全局 `message → 最内层 block` 映射 |
| M-5 | **自行实现 reasoning 排除**（`contentType === "reasoning"` 跳过） |
| M-6 | **身份字段全部改为宿主注入 + 显式 VERIFY**，不得从 `Session` 假设 |
| M-7 | **摘要与 Evidence 彻底分离**：摘要只进 M03 素材，Evidence 只取原句 |
| M-8 | **P0 前必须先裁决 `evidence.schema.json` 冲突** |

### 6.4 暂时不应实现的设计

| # | 不应实现 | 原因 |
|---|---|---|
| N-1 | 在压缩热路径内同步投递 | 跨事务、阻塞压缩、崩溃即孤儿证据 |
| N-2 | 任何"已入记忆库就不许折叠"的机制 | 使压缩能力单调衰减（§2.3） |
| N-3 | 桥接层自带"值得记住"筛选 | 违反 09 §7.1 P-02、D-AP-02 |
| N-4 | 桥接层调用 LLM 判别 | 违反 D-AP-02（须经 M04 Gateway） |
| N-5 | 桥接层写 `confirmed` | 违反 D-AP-05 |
| N-6 | 修改 `evidence.schema.json` | 本轮禁止改既有规范；须裁决 |
| N-7 | 在未验证群身份前启用 | 违反 04 §7.5-8 |
| N-8 | 依赖 `collectBlockContent(full=true)` 做逐块投递 | 重复覆盖（§4.1） |
| N-9 | 把压缩摘要写为 Evidence | 违反 P-02 / P-05 / P-08 |

### 6.5 推荐的最终数据流

```
① 模型调用 compress（内含 summary + ranges）
        ↓
② applyCompression 纯函数（无副作用）
        ↓
③ 四道闸门全部通过
        ↓
④ ctx.session.state = res.state        ← 仅内存
        ↓
⑤ 请求尾部 markDirty → scheduleSave → debounce 500ms
        ↓
⑥ ingestSession() 单 SQLite 事务
     ├─ messages 行（含 content_hash、retired_at）
     └─ blocks 行（start_seq / end_seq / covered_hash / summary_hash）   ← 原子落盘
        ↓
═══════════════ 以下为桥接侧，与压缩路径完全解耦 ═══════════════
        ↓
⑦ 对账器（只读打开 index.db）
     · 取 blocks WHERE active=1 AND 未在 bridge_ledger
     · 按 start_seq 升序
        ↓
⑧ 逐 block 解引用：
     · 用 directMessageIds（或全局最内层映射）取 messages 行
     · 跳过 reasoning
     · 逐条构造 Evidence：
         source_type        = chat          （证据形态 5 值）
         source_id          = session_id
         source_locator     = "<session_id>:<msg_id>:<seq>"
         original_text      = messages.content      ← 原句，零改写
         content_hash       = sha256(original_text)
         speaker_id         = 宿主身份映射（须 VERIFY）
         timestamp          = messages.ts（缺失为 null，不得推测）
         visibility / scope = 由 group 身份推导（owner_private | group_shared）
         tenant_id          = 宿主注入
         revoked            = false
        ↓
⑨ 幂等写入：以 (content_hash, source_locator) 去重；
   命中且可对齐 ⇒ 复用 evidence_id（13 §7.4.2）
        ↓
⑩ 写 bridge_ledger(session_id, block_id, covered_hash, evidence_ids[], bridged_at)
        ↓
⑪ 摘要单独处理：仅作为 candidate 素材投递，
   source_type = SYSTEM_GENERATED，永不进 Evidence，永不 confirmed
        ↓
⑫ M03（P3 授权后）→ M04（P4）→ M01 唯一权威写入
```

**关键点**：⑦–⑫ 是**纯读派生**，压缩层完全不知道记忆库存在。压缩照常丢信息，记忆库照常持有原句——职责分离，无耦合。

### 6.6 失败恢复与幂等方案

| 机制 | 设计 |
|---|---|
| **幂等键** | `bridge_key = sha256(session_id ‖ block_id ‖ covered_hash)`，`UNIQUE` 约束 |
| **原句去重** | `(content_hash, source_locator)`；对齐 13 §7.4.2 的"可对齐才复用" |
| **至少一次** | 对账器可重复运行；失败不推进 ledger |
| **崩溃恢复** | 对账器无状态，重启重扫；ledger 未写即可重做 |
| **顺序保证** | 按 `start_seq` 升序处理，保证父块先于子块（或反之，二者择一但须固定并在测试中断言） |
| **缺口可观测** | 每次运行输出 `待桥接数 / 已桥接数 / 跳过数(含原因)`；`待桥接数` 持续 >0 超阈值即告警 |
| **孤儿防护** | 只从 `index.db` 派生 ⇒ 逻辑上不可能出现"内存折叠但库中无块"的孤儿证据 |
| **删除传播** | `messages.retired_at` 非空 ⇒ 对应 Evidence 标 `revoked=true`（13 §7.4.5） |
| **加密模式** | `BILI_ENCRYPTION_KEY` 生效 → unified sidecar 禁用 → **桥接自动停用并告警**，不得静默降级 |

### 6.7 P0 只读投递的完整验收标准

P0 定义：**只投递原句 Evidence，不接 M03、不影响任何决策、不产生任何上下文副作用。**

| ID | 验收项 | 通过判据 | 类型 |
|---|---|---|---|
| P0-A1 | 契约前置 | `evidence.schema.json` 冲突已裁决，且存在一个**能表达 `original_text` + `speaker_id`** 的合法目标形态 | 阻塞门 |
| P0-A2 | 存储模式前提 | 运行环境 `storageMode=full`，且 `BILI_ENCRYPTION_KEY` 未生效 | 阻塞门 |
| P0-A3 | 群身份前提 | 宿主可提供可信 `(tenant_id, bot_id, group_id)`，且经 VERIFY 记录 | 阻塞门 |
| P0-A4 | 只读性 | 对 `index.db` 全程只读（`?mode=ro`），无任何写事务 | 静态+运行 |
| P0-A5 | 零侵入 | 未修改 `src/`、`kernel/` 任一文件（diff 为空） | 静态 |
| P0-A6 | 不影响压缩 | 开启桥接前后，同一会话的 `blocks` 数量、`summary` 字节、`state.stats` 一致 | 运行 |
| P0-A7 | 原句完整性 | 抽样 ≥30 条 Evidence，`sha256(original_text)` 与 `messages.content_hash` 逐条相等 | 运行 |
| P0-A8 | 零改写 | 抽样 Evidence 与源消息**逐字节相等**；不得出现摘要片段、不得去掉否定词/时间词 | 运行 |
| P0-A9 | reasoning 排除 | 源中含 reasoning 消息的 block，其 Evidence 数 = 非 reasoning 消息数 | 运行 |
| P0-A10 | 无重复 | 嵌套 block 场景下，同一 `(content_hash, source_locator)` 只出现一次 | 运行 |
| P0-A11 | 幂等 | 对账器连续运行 3 次，Evidence 总数不变 | 运行 |
| P0-A12 | 崩溃安全 | 在对账中途 kill，重启后重跑，结果与不中断一致 | 运行 |
| P0-A13 | 缺口可观测 | 人为制造未桥接 block，对账器输出中该计数 >0 且带原因 | 运行 |
| P0-A14 | 删除传播 | 源消息 `retired_at` 置位后，对应 Evidence `revoked=true` | 运行 |
| P0-A15 | 摘要隔离 | 全库检索确认**没有任何** Evidence 的文本来自 `block.summary` | 运行 |
| P0-A16 | 权限继承 | Evidence 的 `visibility`/`scope` 与来源群身份一致，且 `system_internal` 不出正文（13 §7.2-5） | 运行 |
| P0-A17 | 不写 confirmed | 全库确认无 `validation_state=confirmed` 写入 | 运行 |
| P0-A18 | 存档可回滚 | 关闭桥接后系统行为与开启前完全一致；记忆库侧数据可整批删除 | 运行 |

**P0 通过定义**：A1–A5 全绿 + A6–A18 全绿，且 A1/A2/A3 三项阻塞门任一未决则 P0 **不得开始**。

### 6.8 尚未验证的事实与必须实际运行的实验

**明确区分：以下均**不是**已验证事实。**

| # | 未验证项 | 性质 | 需要的实验 |
|---|---|---|---|
| U-1 | 压缩成功路径确实不触发任何持久化 | **静态阅读推断**（基于 `markDirty` 仅 3 处） | 在压缩后立即 `kill -9`，重启观察 block 是否存在；重复 ≥10 次 |
| U-2 | 防抖窗口实际时长与丢数据概率 | 未测 | 注入延迟/崩溃，统计折叠丢失率 |
| U-3 | 对账器只读打开 WAL 库的可行性 | 未测 | 并发只读 + 主写，验证无 `SQLITE_BUSY` 恶化、无锁升级 |
| U-4 | `storageMode` 实际默认值在生产环境的落点 | 仅读代码，未查运行配置 | 打印实际 `resolveStorageMode()` 结果 |
| U-5 | `isMessageProtected` 在 dsh 宿主是否已接线 | 未查插件装配 | 全仓检索调用点 + 运行时断点 |
| U-6 | 宿主能否提供可信 `group_id` | **完全未知** | 查 QQ Bridge / DSH 会话属性；不得凭空假设 |
| U-7 | `ledgerSeq` 与 `blocks.start_seq` 的实际换算是否恒等 | 注释推断 | 抽样比对 `ref = seq + 1` 在真实库中成立 |
| U-8 | 嵌套 block 在实际语料中的出现频率 | 未知 | 统计 `directBlockIds` 非空的 block 占比 |
| U-9 | 一次压缩的实际 Token 收益是否可达宣称的 5× | **厂商自述** | 本地 A/B 复现，不得引用其数字作验收 |
| U-10 | `evidence.schema.json` 冲突的裁决方向 | 未裁决 | 用户裁决，非实验 |
| U-11 | DSH 侧 `x-deepseek-harness-session-id` 与 `Session.id` 的对应 | 未验证 | 抓包比对 |
| U-12 | 加密模式下桥接停用的行为 | 未测 | 开启 `BILI_ENCRYPTION_KEY` 验证 sidecar 禁用与告警 |

---

## 7. 本轮的事实边界声明

1. 所有代码结论来自 **HEAD `0aace2f` 的静态阅读**，附文件路径与行号；**未运行任何测试、未启动任何服务、未构造任何运行时状态**。
2. §6.8 的 U-1、U-3、U-4、U-6、U-9 等属**待实验项**，本报告不将其表述为已验证。
3. 厂商自述指标（5× Token、95–97% 前缀缓存、ContextBench 0.3097 等）**一律不采信为实测**，仅作背景。
4. 本报告**不批准** FUT-09，**不修改**任何既有设计规范，**不提出**对 `evidence.schema.json` 的改动方案（仅登记冲突并请求裁决）。
5. 本轮**未修改** `C:\Users\[USER]\Desktop\work\` 下任何文件。

---

## 8. 需用户裁决的事项（本轮新增，均为阻塞级）

| 编号 | 裁决项 | 不裁决的后果 |
|---|---|---|
| **D-1** | `evidence.schema.json`（v1，无 `original_text`/`speaker_id`，`additionalProperties:false`）与 DD-AUTHORITY §3.2（v2，要求二者必填）冲突，如何处置？ | P0 **无法开工**；任何 Evidence 写入必非法 |
| **D-2** | 是否接受"读侧对账"替代"压缩路径内同步投递"（引入最长 500ms+ 的桥接延迟）？ | 若不接受，则须改 `ingestSession`，违反本轮禁令 |
| **D-3** | 宿主侧能否提供可信 `(tenant_id, bot_id, group_id)`？ | 不能则桥接**整体不可启用**（04 §7.5-8） |
| **D-4** | 是否确认"不保护已入记忆库的消息"（推翻第一轮 §3.4）？ | 若坚持保护，压缩能力将随时间衰减至零 |
| **D-5** | 压缩摘要是否允许作为独立 MemoryUnit（`SYSTEM_GENERATED`）存在？ | 影响 P1 是否纳入摘要路径 |
