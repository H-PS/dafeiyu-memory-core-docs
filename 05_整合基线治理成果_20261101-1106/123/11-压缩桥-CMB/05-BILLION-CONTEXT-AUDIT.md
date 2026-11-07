# 05 — billion-context 上游数据源与接口审查（CMB v0.1 技术尽调）

| 项 | 值 |
|---|---|
| 审查对象 | `E:\[PROJECT]\refs\billion-context`（npm `billion-context` / CLI `bili`） |
| HEAD（本 Agent 复核） | `0aace2f203b2da9065fd53c63f4be0c0ab03c90f` |
| HEAD 提交时间 | 2026-11-05 10:36:54 +0800（`Merge pull request #5696 …`） |
| 分支 | `master` |
| 工作区状态 | **干净**（`git status --porcelain` 输出为空） |
| 存储 schema 版本 | `STORAGE_SCHEMA_VERSION = 10`（`src/storage/schema.ts:71`） |
| 审查性质 | **只读**。未修改、未构建、未运行上游测试，未写入上游仓库任何文件 |
| 审查者 | Agent B（CMB 尽调，billion-context 上游数据源侧） |

## 证据等级约定

| 等级 | 含义 |
|---|---|
| **VERIFIED_SOURCE** | 直接来自源码/DDL/SQL 字面，附 `文件:行号`，可按行复核 |
| **STATIC_INFERENCE** | 由源码结构推断（未运行、未建库验证），标注推断依据 |
| **DOCUMENT_ONLY** | 仅来自注释/文档/CHANGELOG，代码未直接印证 |
| **UNVERIFIED** | 本次无法验证（无网络、无实库、无 issue 列表） |

## 证据文件指纹（`Get-FileHash -Algorithm SHA256`，HEAD 时刻）

```
348cd085a1416a060586a060af32a83bb336e75edd4c9ff124bfcc5d333f8557  src\storage\schema.ts
8debce0abb3d91e163acd2add7acf3e7c583572d3012c8fa91f2965d29236d5b  src\storage\ingest.ts
58a8162318df44365b0b0846094229b00f713964c3f05465899beb4f3de1d4ac  src\storage\store.ts
c4090622fa7bee4d5c7971af3aac417c779383a3aff208ca020a85f01d042648  src\storage\identity.ts
976db3fe453d272208d84af931df4027c9ebc5db77740d523a6310eb0e702a3e  kernel\src\types.ts
53788d715371a475e8d56f5501a5a8bdc230dc2644e650c708556a1074b0c5cf  kernel\src\sync.ts
5ed5400e29d24b3c081b1239b9082ed3d96d59e6ae80f806e1eafcb858707612  kernel\src\state.ts
1a8ed2d46b904a0f8907ee49801db448eb2f39d5388d0b167d8707d481af58f4  kernel\src\decompress.ts
65155a3d574226675ad870eca7450a3994a4f61c6412664b9865c47a26484cbe  src\session.ts
6059a5e54d397331f558dd4e2b110a522e8fe507c12eed1b064e3347af541f5e  src\knobs.ts
1a9c74e06854abcb299e32e292aa5049337ed2a07c50ef8a036a61c6c7a0e538  src\persist.ts
2c7cafcf6a0a8d0cb2005c2e60bb9bfd082d328e11cfbc91a562bd81f5796047  src\storage\driver.ts
58ecaa8aca9e6f83931fe4b2f473d7aeb71a203707df2c694f608be55228f56e  src\storage\integrity-check.ts
54177053ec1b16e0434fbe0a9257f00deebab47dc627d87e196cc4d906f711da  src\export.ts
```

---

# 0. 执行摘要（给决策层）

## 0.1 最影响 CMB 接入的 3 个事实

**事实 1 — `index.db` 默认存在，但它存的是"投影后的正文"，不是原始消息字节。**
自 #3818 起 `storage.unified` 缺省即为 `full`（`src/knobs.ts:880`，文档同步：`CONFIGURATION.md:200`、`:2082`），也就是说新装用户默认就在写 `index.db`，它已是官方读路径。但 `messages.content_hash → content.bytes` 里的字节**不是** `JSON.stringify(CoreMessage)`：写入前经过 `splitMediaContent()`，`imageBase64` 被替换成 `{"$media":"<sha256>"}`，大的 `data:…;base64,` 载荷被替换成 `data:<mime>;base64,{$media:<hash>}`（`src/storage/identity.ts:40-77`，调用点 `src/storage/ingest.ts:194`）。图片字节落在**另外的 CAS 行**，靠 `message_media(session_id, seq, media_hash)` 二级 JOIN 还原（`src/storage/schema.ts:267-272`）。
→ **CMB 若只 `SELECT content.bytes` 就当拿到了原文，图片消息会拿到占位符而不是图。必须做 `messages → content` + `messages/message_media → content` 的两次 hydrate。**

**事实 2 — 消息有历史，块没有历史。**
消息行是 append-only 日志，视图缺席时只打墓碑 `retired_at`，且可被同一 identity 复活（`src/storage/store.ts:923-927`），所以"客户端曾发过什么"是可追溯的。**但 block 没有墓碑机制**：`blocks` 表没有 `retired_at` 列（`src/storage/schema.ts:157-174`），不在本轮 `state.blocks` 里的块被**物理 `DELETE`**（`src/storage/store.ts:1077-1087`），连同它的 summary/one CAS 引用一并释放（delete-on-zero，`store.ts:790-809`）。
→ **"压缩历史 / 块的演化史"无法从 `index.db` 重建。** 残留痕迹只有三处，且都不完整：`sessions.orphans_json`（被 reap 的块文本，`store.ts:756-788`）、`sessions.metadata_json.preCompactionArchive`（块 id + 时间 + 原因，无正文，`src/session.ts:1729-1734`）、`blockContents` 在磁盘的遗留。

**事实 3 — 只读接入官方从没设计过，但物理上可行且有仓库内先例。**
生产打开路径**没有** `readOnly` 选项：`UnifiedStore.open()` 只接受 `(path, engine, {walJournalSizeLimitBytes})`（`store.ts:443-447`），底层 `new DatabaseSync(path)` 不传选项（`driver.ts:163`）。`readOnly` 一词在仓库里只作为**事务**参数出现（`BEGIN DEFERRED` vs `BEGIN IMMEDIATE`，`driver.ts:166`）。真实只读打开的唯一先例是后台完整性检查 worker：`new …DatabaseSync(p, { readOnly: true })`（`integrity-check.ts:163`），以及若干测试直接以 `{readOnly:true}` 打开 `index.db`（如 `tests/msgid-lane-flow.test.ts:95`、`tests/seq-interval-lane.test.ts:105`）。
→ **CMB 以只读方式打开 `index.db` 有可行性证据，但必须自己承担 WAL 语义（`-wal`/`-shm` 存在性）与"读到的是一致性快照"的责任；上游不会为第三方读者做任何兼容承诺。**

## 0.2 一句话结论

`index.db` **足以支撑 CMB 的"消息级"接入**（谁在什么时候说了什么、工具调用与结果、角色与类型、时间戳、引用编号、会话血缘），**不足以支撑"压缩决策级"接入**（块的生命周期、块被谁取代、块为何消失）。若 CMB 的产品价值依赖"压缩史"或"上下文折叠轨迹"，必须同时接入 `index.db` 之外的旁路（`orphans_json` / `preCompactionArchive` / 上游 export JSONL），或者接受该维度不可得。

---

# A. 存储结构真相

## A.1 `index.db` 逐表真实 DDL

来源：`src/storage/schema.ts:108-280`（`STORAGE_DDL_V1` 常量），版本演进在 `migrateSchema()`（`schema.ts:283-405`）。共 **11 张表 + 6 个索引**。

### A.1.1 `sessions`（`schema.ts:108-131`）

```sql
CREATE TABLE IF NOT EXISTS sessions (
    session_id          TEXT PRIMARY KEY,
    parent_session_id   TEXT,
    fork_point_seq      INTEGER,
    protocol            TEXT,
    label               TEXT,
    title               TEXT,
    host_title          TEXT,
    namespace           TEXT NOT NULL DEFAULT 'proxy',
    origin              TEXT,
    status              TEXT NOT NULL DEFAULT 'active',
    created_at          INTEGER NOT NULL,
    updated_at          INTEGER NOT NULL,
    next_block_id       INTEGER NOT NULL DEFAULT 1,
    next_run_id         INTEGER NOT NULL DEFAULT 1,
    meta_json           TEXT NOT NULL DEFAULT '{}',
    stats_json          TEXT NOT NULL DEFAULT '{}',
    metadata_json       TEXT NOT NULL DEFAULT '{}',
    kstate_json         TEXT NOT NULL DEFAULT '{}',
    tail_json           TEXT NOT NULL DEFAULT '{}',
    ccr_json            TEXT,
    last_ingested_seq   INTEGER,
    orphans_json        TEXT
);
```

关键点：
- `tail_json` 是**视图指针**，不是内容：`{pluginSnapshotSeqs, tailSeqs, tailFolded}`（写入点 `ingest.ts:498`，解析点 `store.ts:70-74`）。这是"模型这一轮实际看到的消息序列"的唯一权威表述。
- `kstate_json` = 内核状态里没有专用表的剩余字段（`ingest.ts:461-466` 用 `COMPRESSION_STATE_FIELDS` 逐字段筛选，`TABLE_OWNED_FIELDS` 排除 `blocks/messageRefs/deadRefs/nextBlockId/nextRunId`，见 `ingest.ts:42`）。
- `orphans_json` = 被 reap 掉的块的 `{one, full}` 文本日志（v6，#3841；`schema.ts:338-350`）。
- `last_ingested_seq` = 增量摄入水位（v5，#3133）。

### A.1.2 `content`（`schema.ts:133-139`）—— 唯一的 blob 表

```sql
CREATE TABLE IF NOT EXISTS content (
    hash        TEXT PRIMARY KEY,
    byte_len    INTEGER NOT NULL,
    refcount    INTEGER NOT NULL DEFAULT 0,
    algo        TEXT NOT NULL DEFAULT 'sha256',
    bytes       BLOB NOT NULL
);
```

- **这就是 CAS**。`bytes` 是 UTF-8 文本字节（不是二进制），`algo` 恒为 `'sha256'`（`putContent` 硬编码，`store.ts:749`）。
- 内容类型名不副实：注释里 `content.bytes BLOB` 被称作"chunks/ 文件抽取的替代方案"（`schema.ts:32-34`），说明它当前**就是**正文存放处，不是索引。
- 索引：`idx_content_refcount ON content(refcount)`（v9，#4796；`schema.ts:389`）。

### A.1.3 `messages`（`schema.ts:141-155`）—— 只存指针，不存正文

```sql
CREATE TABLE IF NOT EXISTS messages (
    session_id      TEXT NOT NULL,
    seq             INTEGER NOT NULL,
    msg_id          TEXT NOT NULL,
    ref             TEXT,
    ref_num         INTEGER,
    role            TEXT NOT NULL,
    kind            TEXT NOT NULL DEFAULT 'message',
    content_hash    TEXT NOT NULL REFERENCES content(hash),
    identity        TEXT NOT NULL DEFAULT '',
    retired_at      INTEGER,
    ts              INTEGER,
    PRIMARY KEY (session_id, seq),
    UNIQUE (session_id, msg_id, content_hash)
);
```

- **没有 `content` 列**。正文 100% 在 `content.bytes`，`messages` 只有 `content_hash` 外键（`schema.ts:149`）。
- 外键是**强制**的：`PRAGMA foreign_keys = ON`（`schema.ts:93`）。所以 `content` 行不能在仍被引用时被删除——这是 CMB 可以依赖的一条硬约束。
- `seq` 是"首见到达顺序"，0 基、会话内单调（`ingest.ts:11-13` 注释；`seq = log.length`，`ingest.ts:203`）。

### A.1.4 `blocks`（`schema.ts:157-174`）—— 压缩账本快照

```sql
CREATE TABLE IF NOT EXISTS blocks (
    block_id            TEXT NOT NULL,
    session_id          TEXT NOT NULL,
    ord                 INTEGER NOT NULL,
    start_seq           INTEGER NOT NULL,
    end_seq             INTEGER NOT NULL,
    start_ref           INTEGER,
    end_ref             INTEGER,
    tier                INTEGER NOT NULL,
    summary_hash        TEXT NOT NULL REFERENCES content(hash),
    one_hash            TEXT REFERENCES content(hash),
    supersedes_block_id TEXT,
    active              INTEGER NOT NULL DEFAULT 1,
    created_at          INTEGER NOT NULL,
    covered_hash        TEXT NOT NULL,
    payload_json        TEXT NOT NULL,
    PRIMARY KEY (session_id, block_id)
);
```

- `summary_hash` / `one_hash` 都是 CAS 外键 → 块摘要正文也在 `content`。
- `payload_json` = 逐字 `CompressionBlock`（`ingest.ts:429`），是"事实"；所有标量列是它的**投影**（`schema.ts:14-22` 的表示纪律）。
- **无 `retired_at`** → 块没有墓碑。

### A.1.5 其余 7 张表

| 表 | 行号 | 作用（VERIFIED_SOURCE） |
|---|---|---|
| `dead_refs` | `schema.ts:176-183` | 内核引用墓碑，`mNNNNN` 死号永不回收（`schema.ts:185-188` 注释；Kernel Contract） |
| `refs` | `schema.ts:189-197` | 内核 `messageRefs.byRaw` 的逐行投影，`UNIQUE(session_id, raw_id)`；`ref` 可为字面量 `"BLOCKED"` |
| `checkpoints` | `schema.ts:199-212` | 冻结的会话快照（replica），`state_hash → content`；`pinned` 的 checkpoint 会占 refcount 从而在 `deleteSession` 后存活 |
| `ccr_entries` | `schema.ts:214-220` | 内核 CCR 内容存储（"dying subsystem"，`store.ts:110`），`content_hash → content` |
| `conversations` | `schema.ts:229-235` | 身份索引 `(namespace, conversation_id) → session_id`，`WITHOUT ROWID` |
| `session_ext` | `schema.ts:243-249` | 宿主/插件/UI 的每会话任意状态（`WITHOUT ROWID`） |
| `kv_store` | `schema.ts:255-261` | 跨会话命名空间 KV（`WITHOUT ROWID`） |
| `message_media` | `schema.ts:267-272` | **图片/媒体 CAS 映射**，`(session_id, seq, media_hash)`，`WITHOUT ROWID` |

索引（`schema.ts:274-279`）：
```sql
CREATE INDEX IF NOT EXISTS idx_messages_session   ON messages(session_id, seq);
CREATE INDEX IF NOT EXISTS idx_messages_msg_id    ON messages(session_id, msg_id);
CREATE INDEX IF NOT EXISTS idx_blocks_session     ON blocks(session_id, ord);
CREATE INDEX IF NOT EXISTS idx_dead_refs_session  ON dead_refs(session_id, ord);
CREATE INDEX IF NOT EXISTS idx_refs_session       ON refs(session_id, ord);
CREATE INDEX IF NOT EXISTS idx_checkpoints_session ON checkpoints(session_id, created_at);
```
外加 v8 的 `idx_messages_identity ON messages(session_id, msg_id, identity)`（`schema.ts:368`）与 v9 的 `idx_content_refcount`（`schema.ts:389`）。

## A.2 原始消息正文到底存在哪里？完整链路

**结论：不在 `messages` 表内联，在 `content.bytes`，经 `content_hash` 单跳外键。图片再经 `message_media` 二跳。**

### A.2.1 写入链路（7 步，全部 VERIFIED_SOURCE）

```
① 客户端请求 → session.lastMessages / pluginSnapshot / state（CoreMessage[]）
   └ src/session.ts:307…（Session 类型），lastMessages 语义见 session.ts:647-652

② planIngest() / planIngestDelta() 逐条 intern
   └ ingest.ts:190-207（intern）、ingest.ts:323-346（delta 循环）

③ serializeForStore(m)：JSON.stringify(m)，仅删除 ts
   └ ingest.ts:155-160

④ splitMediaContent(raw)：媒体外置（→ 关键步骤，改写了字节！）
   └ ingest.ts:194 → identity.ts:40-77
   └ imageBase64 → {"$media": sha256(bytes)}   (identity.ts:51-57)
   └ data:<mime>;base64,<≥2048B> → data:<mime>;base64,{$media:<hash>}  (identity.ts:65-75)

⑤ 计算三个摘要
   └ hash     = sha256(媒体外置后的字符串)      identity.ts:76
   └ identity = sha256(剥 id/wireIndex/ledgerSeq 后的字符串)  identity.ts:86-107
   └ kind     = parsed.contentType ?? "message"  ingest.ts:195

⑥ 落库：putContent() 先写 CAS，再 INSERT OR IGNORE messages
   └ store.ts:1033  const hash = this.putContent(m.content, m.contentHash);
   └ store.ts:735-753（putContent：SELECT refcount → +1 或 INSERT bytes BLOB, algo='sha256'）
   └ store.ts:1036-1038（INSERT OR IGNORE INTO messages … content_hash=hash, identity, ts）

⑦ 媒体落库：putContent(图片字节) + INSERT message_media
   └ store.ts:1048-1068
```

**`④` 是本审查发现的最重要细节**：`messages.content_hash` 指向的字节**不等于** `JSON.stringify(CoreMessage minus ts)`。对一个带 `imageBase64` 的消息：

| 环节 | 字节形态 |
|---|---|
| 内存中的 CoreMessage | `{"id":"…","imageBase64":"iVBORw0KGgo…(可能数 MB)"}` |
| `serializeForStore` 输出 | 同上（原文） |
| **写入 `content.bytes` 的实际内容** | `{"id":"…","imageBase64":{"$media":"9f2c…"}}` |
| 图片字节去哪了 | 另一行 `content(hash=sha256(base64串), bytes=base64串)` + 一行 `message_media` |

### A.2.2 读取链路（6 步，全部 VERIFIED_SOURCE）

```
① 取会话行
   └ store.ts:2765-2779 getSessionRow()（SELECT … FROM sessions WHERE session_id=?）

② 解析 tail_json 得视图指针（不是内容）
   └ store.ts:70-74  viewPointerSeqs(tailJson) → [...tailSeqs, ...pluginSnapshotSeqs]

③ 按 seq 点查消息行（只取视图指针覆盖的行，不是全表）
   └ store.ts:3024-3054 getMessagesAt(sessionId, seqs)
   └ SQL: SELECT seq,msg_id,ref,ref_num,role,kind,content_hash,ts
          FROM messages WHERE session_id=? AND retired_at IS NULL AND seq IN (…)
   └ 每 500 个 seq 一批（CHUNK=500），最后按 seq 排序

④ 每行一次 CAS 点查拿正文
   └ store.ts:3048  this.hydrateMedia(this.getContent(r.content_hash as string) ?? "")
   └ getContent: store.ts:2677-2683（SELECT bytes FROM content WHERE hash=? → utf8 字符串）

⑤ hydrateMedia：把 {"$media":hash} 换回真实字节
   └ store.ts:3100-3128
   └ ① 顶层 imageBase64.$media → getContent(hash)   (3102-3116)
   └ ② 原始 wire sidecar 里的 data:…;base64,{$media:hash} → 正则替换回 data URL (3118-3126)
   └ 缺行时"回退到拆分形态而不是抛错"（3100-3099 注释；实测行为=保留 {$media:…} 字面量）

⑥ 组装 legacy 视图（可选，如果 CMB 想复现"上游 loadSync 看到的"）
   └ ingest.ts:590-696 LegacyViewBuilder（addMessages 里 JSON.parse(m.content)）
   └ ingest.ts:659-695 finish()（按 tailSeqs 顺序 resolve，缺行计入 tailGaps）
```

**全表扫描的读法**（CMB 若要"整段历史"而不是"当前视图"）：
`store.ts:2983-3016 getMessages(sessionId, {includeRetired?})` → `SELECT … FROM messages WHERE session_id=? AND retired_at IS NULL ORDER BY seq`，每行再 `getContent`。这是 **N+1 次 CAS 点查**（每行一次 `SELECT bytes FROM content WHERE hash=?`），但 hash 是 PRIMARY KEY，单次 O(log n)。

## A.3 序列化格式：完整 CoreMessage 还是投影？

**答案：完整 `CoreMessage`（非投影），但经过两道改写（去 ts、媒体外置）。**

`serializeForStore`（`ingest.ts:149-160`）原文语义：

```ts
function serializeForStore(m: CoreMessage): string {
    if (m.ts === undefined) return stringifyForStore(m);
    const copy = { ...m };
    delete copy.ts;
    return stringifyForStore(copy);
}
```

- 不做字段白名单、不做截断、不投影 → **除 `ts` 外所有字段原样进 JSON**。
- `stringifyForStore` 额外包一层 RangeError 语义化（`ingest.ts:166-175`，#5627：嵌套过深时错误里点名 `message ${m.id}`）。
- `ts` 被剥离的理由（#4313，`ingest.ts:149-154`）：保证 stored content 与 pre-feature 保存的字节逐字一致，这样 content hash / identity / boundary 探针不会因为"加了时间戳"而抖动；`ts` 走独立列。

因此 blob 里的字段 = `CoreMessage` 全集 − `ts`（+ 媒体槽位被替换）。

## A.4 字段归属矩阵（blob / 表列 / 两边都有）

| 字段 | 在 CAS blob 里 | 在表列里 | 说明 |
|---|---|---|---|
| `id` | ✅ | ✅ `messages.msg_id`（投影） | `verifyProjections` 断言两者相等（`store.ts:2496`） |
| `role` | ✅ | ✅ `messages.role` | 断言相等（`store.ts:2497`） |
| `contentType` | ✅ | ✅ `messages.kind`（`?? "message"`） | 断言相等（`store.ts:2498-2500`） |
| `text` | ✅ | ❌ | 只在 blob |
| `toolName` | ✅ | ❌ | 只在 blob |
| `toolCallId` | ✅ | ❌ | 只在 blob |
| `thinkingTokens` | ✅ | ❌ | 只在 blob（计量用，不渲染） |
| `summaryOfBlockId` | ✅ | ❌ | 只在 blob（#335 宿主 checkpoint 载体） |
| `ledgerSeq` | ✅ | ⚠️ 语义重复但值不同 | blob 里的 `ledgerSeq` 是内核发号；表列 `seq` 是存储发号。`identity` 计算时**剥掉** `ledgerSeq`（`identity.ts:96`） |
| `ts` | ❌ **被显式删除** | ✅ `messages.ts` | 唯一"只在表列"的 CoreMessage 字段 |
| `content_hash` | ❌ | ✅ | 派生量 = sha256(blob)，不冗余存 |
| `identity` | ❌ | ✅ | 派生量 = sha256(剥位后的 blob)，不冗余存 |
| `ref` / `ref_num` | ❌ | ✅ | 来自 `state.messageRefs.byRaw`，是**账本**事实而非消息事实（`ingest.ts:205`、`finishPlan` `ingest.ts:372-377`） |
| `retired_at` | ❌ | ✅ | 纯存储层墓碑 |
| `seq` | ❌ | ✅ | 纯存储层到达序 |

**CompressionBlock 侧**（`blocks` 表，`kernel/src/types.ts:68-110`）：

| 字段 | blob(`payload_json`) | 表列 | 说明 |
|---|---|---|---|
| 全部字段（`blockId/runId/tier/topic/summary/directMessageIds/effectiveMessageIds/directBlockIds/compressedTokens/createdAt/survivedCount/generation/active/expanded/restoredInline/durationMs/compressCallId/startRef/endRef/startSeq/endSeq`） | ✅ | 部分 | `payload_json = JSON.stringify(b)`（`ingest.ts:429`） |
| `blockId` / `tier` / `active` | ✅ | ✅ `block_id`/`tier`/`active` | 断言相等（`store.ts:2507-2511`） |
| `summary` | ✅ | ⚠️ `summary_hash` 指向 `{text,count}` BlockView，**语义不同** | 见 `store.ts:2512-2514` 注释：两者是"不同的 fact"，故意不比较 |
| `startSeq`/`endSeq` | ✅ | ✅ `start_seq`/`end_seq` | **值可能不同**：表列由 store 从"live messages 表"重算（见 B6） |

## A.5 `kind` 列存的是什么？

**答案：`kind` = 消息的 `contentType` 投影，缺省字面量 `"message"`。它不是"消息类别"的独立概念。**

证据链（三处独立印证）：

1. **写入**：`ingest.ts:195` — `kind: parsed.contentType ?? "message"`（`parsed` 是 `JSON.parse(serializeForStore(m))`）。
2. **自检**：`store.ts:2498-2500` — `verifyProjections` 明确断言 `(parsed.contentType ?? "message") !== m.kind` 即失败。
3. **导出**：`export.ts:277` — JSONL 里 `kind: m.kind ?? undefined`，与 `role` 并列输出。

`contentType` 的真实取值域见 §E.2。**注意**：`?? "message"` 这条兜底在正常情况下不会触发，因为 `contentType` 在 `CoreMessage` 里是**必填**字段（`kernel/src/types.ts:11`，非 optional）；它只为非 CoreMessage 形状的历史/导入载荷兜底。

## A.6 CAS blob 的寻址、refcount、删除时机

### 寻址（VERIFIED_SOURCE）

| 项 | 值 | 证据 |
|---|---|---|
| 算法 | SHA-256 | `store.ts:76-78` `createHash("sha256").update(bytes).digest("hex")`；`identity.ts:25-27` 同构 |
| 编码 | 小写 hex，64 字符 | `.digest("hex")` |
| 输入 | UTF-8 字节（`Buffer.from(str,"utf8")`） | `store.ts:736` |
| 列 | `content.hash TEXT PRIMARY KEY` | `schema.ts:134` |
| `algo` 列 | 恒 `'sha256'`（硬编码字面量） | `store.ts:749` |

**可外部验证的不变量（对 CMB 极有价值）**：对**每一行** `content`，都有 `hash == sha256(utf8(bytes))`。因为 `putContent` 只有两条路径——`precomputedHash ?? sha256Hex(bytes)`（`store.ts:741`）——而所有 planner 传入的 `precomputedHash` 也都是 `sha256Hex` 同一份字节（`identity.ts:76`、`store.ts:1033`）。CMB 可以据此做**端到端完整性自检**，不需要信任上游。

### refcount（VERIFIED_SOURCE）

- 表列 `refcount INTEGER NOT NULL DEFAULT 0`（`schema.ts:136`），但**插入时显式写 1**（`store.ts:749`：`INSERT INTO content (hash, byte_len, refcount, algo, bytes) VALUES (?, ?, 1, 'sha256', ?)`），`DEFAULT 0` 实际不会被用到。
- `putContent` 命中已有行 → `UPDATE content SET refcount = refcount + 1`（`store.ts:746`）。
- 谁持有引用：`messages.content_hash`（每行 1）、`blocks.summary_hash`、`blocks.one_hash`、`checkpoints.state_hash`、`ccr_entries.content_hash`、`message_media.media_hash`（每映射行 1）。

### 删除时机：delete-on-zero（`store.ts:790-809`）

```sql
UPDATE content SET refcount = refcount - 1 WHERE hash = ? AND refcount > 0 RETURNING refcount
-- 若返回 refcount <= 0：
DELETE FROM content WHERE hash = ? AND refcount <= 0
```

- **不是"延迟 GC"，是同语句删除**（#3987/#4242）。注释给的现场数据：旧实现留下 **11.32 GB** 零引用孤儿（`store.ts:793`）。
- `AND refcount > 0` 是防止账本已损坏时把计数压成负数（`schema.ts:489-490` 迁移里同样有此 floor）。
- **外键顺序纪律**（#4242）：必须先删引用方行、再释放，否则 `foreign_keys=ON` 会阻止删除（见 `store.ts:1163-1168` 与 `1083-1086` 的注释与顺序）。

**对 CMB 的含义**：`content` 表里出现零引用行是**短暂异常**（下一个 release 会清），而不出现零引用行是**常态**。若 CMB 看到 `refcount=0` 的存活行，说明进程在释放中途崩溃过。

---

# B. 对主代理此前结论的独立复核

> 任务书称"B 段 11 条"，实际列出的是 **12 条**。以下逐条复核 **12 条**。

## B.0 复核结果总表

| # | 主代理结论 | 复核判定 |
|---|---|---|
| B1 | `messages.content_hash REFERENCES content(hash)`，正文在 `content.bytes BLOB` | ✅ **通过**（附 1 处重要补充） |
| B2 | `serializeForStore` = `JSON.stringify(CoreMessage)` 去掉 `ts` | ❌ **主代理结论有误/不完整**（见 B2） |
| B3 | `blocks.payload_json` = 逐字 CompressionBlock，含 `directMessageIds`/`effectiveMessageIds` | ✅ **通过** |
| B4 | `covered_hash = sha256(JSON.stringify({effectiveMessageIds, directMessageIds}))` | ✅ **通过**（附键序与"无人校验"补充） |
| B5 | `supersedes_block_id` 唯一写入点硬编码 null（ingest.ts:425） | ✅ **通过**（比主代理想的更彻底：列恒为 NULL） |
| B6 | `start_seq`/`end_seq` 可为 -1 哨兵 | ✅ **通过结论 / 机制需替换** |
| B7 | blocks 权威集清扫：不在本轮 state 中的块被物理 DELETE | ✅ **通过**（附"仅 blocks，messages 不删"补充） |
| B8 | `retired_at` 是视图缺席墓碑，可复活（`UPDATE messages SET retired_at=NULL`） | ✅ **通过**（附媒体 ref 释放的补充） |
| B9 | `getContent` 缺失时调用侧 `?? ""` 静默空串 | ✅ **通过** |
| B10 | `ts` 无首见戳时填批采样时刻 | ✅ **通过**（附"导入路径写 NULL"补充） |
| B11 | 加密模式 `BILI_ENCRYPTION_KEY` 生效 ⇒ unified sidecar 不打开 | ✅ **通过** |
| B12 | `identity` 列 = 内容身份哈希（剥 id/wireIndex），`getMessages` SELECT 不含它 | ✅ **通过**（附"还剥 ledgerSeq"补充） |

**计分：12 条 = 7 条原样通过 + 4 条成立但需补充 + 1 条纠正。**

---

## B1 — ✅ 通过（附补充）

**主张**：`messages.content_hash REFERENCES content(hash)`，正文在 `content.bytes BLOB`

**证据**：
- `schema.ts:149`：`content_hash    TEXT NOT NULL REFERENCES content(hash),`
- `schema.ts:133-139`：`content` 表，`bytes BLOB NOT NULL`
- `schema.ts:93`：`PRAGMA foreign_keys = ON` —— 外键**真被强制**，不是装饰

**补充（重要）**：「正文」这个词要打引号。`content.bytes` 里的字节是**媒体外置后的 UTF-8 文本**（见 A.2/A.3）。对纯文本消息，它等于 `JSON.stringify(CoreMessage − ts)`；对含图消息，它含 `{"$media":…}` 占位符。所以更准确的表述是：**"消息的可读正文在 `content.bytes`；图片字节在另外的 CAS 行，经 `message_media` 关联"**。

---

## B2 — ❌ **主代理结论有误：实际是 `JSON.stringify(CoreMessage − ts)` 再经 `splitMediaContent()` 媒体外置，`content.bytes` 不是那个 stringify 的输出**

**主张**：`serializeForStore` = `JSON.stringify(CoreMessage)` 去掉 `ts`

**函数本体是对的**（`ingest.ts:155-160`，确实只 `delete copy.ts`）。**但把它当作"存入 blob 的格式"是错的**，中间还有一道改写：

```ts
// ingest.ts:190-207  intern()
const raw = serializeForStore(m);           // ① JSON.stringify minus ts
const parsed = JSON.parse(raw) as {…};      // ② 仅用于取 role/contentType
const split = splitMediaContent(raw);       // ③ ← 关键：媒体外置，返回 {content, hash, media}
const entry = { …, content: split.content,  // ④ 落库的是 split.content，不是 raw
                hash: split.hash, … };
```

`splitMediaContent`（`identity.ts:40-77`）做两件事：

1. **`imageBase64` 槽位**（`identity.ts:43-58`）：若 `bytes.length > 0`，
   ```ts
   const mediaHash = sha256Hex(bytes);
   parsed.imageBase64 = { $media: mediaHash };   // identity.ts:54
   split = JSON.stringify(parsed);                // identity.ts:55
   ```
2. **原始 wire sidecar 里的 data URL**（`identity.ts:65-75`，正则 `MEDIA_URL_MARKER_RE`，阈值 `MIN_MEDIA_URL_SPLIT_BYTES = 2048`，`identity.ts:33`）：
   `data:<mime>;base64,<payload>` → `data:<mime>;base64,{$media:<hash>}`（`identity.ts:73`）。

而 `hash = sha256Hex(split)`（`identity.ts:76`）——**注意：hash 覆盖的是拆分后的字节，不是拆分前的**。这解释了为什么 `getMessages` 必须先 `getContent` 再 `hydrateMedia`（`store.ts:3048`）。

**影响 CMB 的具体差异（务必写进接入方案）**：
- 只读 `content.bytes` → 图片消息拿到的是 `{"$media":"9f2c…"}`，**不是 base64**。必须二次 JOIN `message_media` 或按 `$media` 值再查 `content`。
- `content.byte_len` 是**拆分后**的长度，不是原始消息大小。
- 同一张图被两条消息引用时，`content` 只有一行、`refcount=2`、`message_media` 有两行——去重是按**媒体 blob** 粒度做的，不是按消息。
- `hydrateMedia` 缺行时**不抛错**：保留 `$media` 字面量返回（`store.ts:3100-3128` 注释明说 "falls back to the split form rather than throwing — visible, not silent"），所以 CMB 必须自己检测 `"$media"` 残留并判为数据缺失。

---

## B3 — ✅ 通过

**主张**：`blocks.payload_json` = 逐字 CompressionBlock，含 `directMessageIds`/`effectiveMessageIds`

**证据**：
- `ingest.ts:429`：`payloadJson: JSON.stringify(b)`，其中 `b` 直接来自 `session.state.blocks.map(...)`（`ingest.ts:402`），未做任何字段裁剪。
- 类型侧确认字段全集：`kernel/src/types.ts:68-110`，`directMessageIds: string[]`（:74）、`effectiveMessageIds: string[]`（:75）、`directBlockIds: string[]`（:76）。
- 消费侧确认可按此解析：`store.ts:1103-1104` 从 `payloadJson` 里读 `effectiveMessageIds`/`directMessageIds` 去算 seq 跨度；`store.ts:1135-1136` 同样。

**补充**：`payload_json` 是"事实"，`summary_hash` 指向的 `{text,count}` **不是** `payload.summary` 的投影（`store.ts:2512-2514` 明确声明两者是不同 fact，故意不比对）。`payload.summary` 是纯字符串；`blocks.summary_hash → content` 是 `BlockView`（`{text, count}`）。CMB 若要摘要文本，用 `summary_hash` 那条更好（带计数），但两者内容本应一致。

---

## B4 — ✅ 通过（附 2 处补充）

**主张**：`covered_hash = sha256(JSON.stringify({effectiveMessageIds, directMessageIds}))`

**证据**：
- `ingest.ts:406`：`const covered = { effectiveMessageIds: b.effectiveMessageIds, directMessageIds: b.directMessageIds };`
- `ingest.ts:428`：`coveredHash: sha256Hex(JSON.stringify(covered)),`

**补充 1（键序）**：键序是 **`effectiveMessageIds` 在前、`directMessageIds` 在后**。JSON 键序敏感，CMB 若要独立重算校验，必须用这个顺序，不能按字母序或按阅读直觉。

**补充 2（更重要的负面发现）**：**没有任何代码校验 `covered_hash`。** 全仓库 `coveredHash` 的写入点只有 `ingest.ts:428` 一处；其余全部是"搬运"（sqlite 的 upsert 绑定、MySQL 的读写、import/restore 的 dump 往返、`getBlocks` 的投影、`export` 的可选输出）。
- `verifyProjections()`（`store.ts:2461-2517`）对 blocks 只校验 `blockId`/`tier`/`active` 三项（`store.ts:2507-2511`），**不碰 `covered_hash`**。
- 测试里甚至手写 `coveredHash: "[]"`（`tests/issue4739-storage-read-snapshot.test.ts:27`），说明它不参与任何一致性判定。

→ **`covered_hash` 目前是一个"只写不校验"的列**。CMB 不要把它当完整性凭据用；要校验覆盖集，请直接解析 `payload_json` 并与 `messages` 对比。这同时是一条"上游自身未完成项"（见 F 段）。

---

## B5 — ✅ 通过（比主代理表述更强）

**主张**：`supersedes_block_id` 唯一写入点硬编码 null（`ingest.ts:425`）

**逐字确认**：`ingest.ts:425` 就是 `supersedesBlockId: null,` —— 行号精确无误。

**更强的结论**：这个列在整个 sqlite 生产路径上**恒为 NULL**：
- 唯一写入方 `ingest.ts:425` 硬编码 null；
- `store.ts:1202` / `store.ts:1242` 只是把 `b.supersedesBlockId` 绑进 SQL；
- 只有 import/restore 路径（`store.ts:1992`、`store.ts:2117`）会复制 dump 里的值，而 dump 里的值又来自这张表（恒 null）——**闭环**；
- 内核 `CompressionBlock` 类型根本没有 `supersedes` 字段（`kernel/src/types.ts:68-110`），代的语义由 `directBlockIds`（消费关系）+ `active` 表达。

→ `supersedes_block_id` 是**预留但未通电的死列**。CMB 应把它当"永远 NULL"处理，不要基于它建血缘图（真正的父子关系在 `payload_json.directBlockIds` 里）。

---

## B6 — ✅ 结论通过，⚠️ **但机制描述需要替换**

**主张**：`start_seq`/`end_seq` 可为 -1 哨兵

**"可为 -1" 成立**，但**不是**主代理隐含的那条路径（"ingest 计划算出 -1 就写进库"）。真实是两段：

**第一段（计划层，`ingest.ts:402-431`）** 确实算 -1：
```ts
let startSeq = -1;                       // ingest.ts:407
let endSeq = -1;                         // ingest.ts:408
for (const id of b.effectiveMessageIds) { … }   // 只在该 plan 的 log 内找
```
找不到任何覆盖 id → 保持 `-1/-1`（`ingest.ts:419-420`）。

**第二段（存储层，`store.ts:1088-1146`，实际落库值来自这里）**：
- `store.ts:1096-1129`：从 `payload_json` 取出全部覆盖 id，按 500/批去 **live messages 表**点查 `SELECT msg_id, seq … WHERE session_id=? AND msg_id IN (…) AND retired_at IS NULL`（`store.ts:1119`）。
- `store.ts:1131-1144` `boundsForCovered()`：取 min/max；**一个都解析不到 → 返回 `[-1,-1]`**（`store.ts:1143`）。
- `store.ts:1145-1147`：`const [bStartSeq, bEndSeq] = boundsForCovered(b.payloadJson);` —— **计划里的 `b.startSeq/b.endSeq` 在这里被完全丢弃**，SQL 绑定的是 `bStartSeq/bEndSeq`（`store.ts:1195-1196`、`store.ts:1235-1236`）。

**附带一条 #4608 保护**（`store.ts:1183-1184`）：
```sql
start_seq = CASE WHEN excluded.start_seq >= 0 THEN excluded.start_seq ELSE blocks.start_seq END,
end_seq   = CASE WHEN excluded.end_seq >= 0 THEN excluded.end_seq   ELSE blocks.end_seq   END,
```
即"入参未知时保留库里已有区间"。这条是**单向棘轮**：真实区间一旦写入就不会被后续 `-1` 覆盖（`store.ts:1158-1162` 注释）。

**语义（`sync.ts:6-8` 定义，权威）**：
> The -1/-1 "no claim" sentinel … is absence of evidence, not a positional claim (a `[0,0]` row IS a real claim over seq 0).

→ **CMB 必须把 `start_seq=-1 AND end_seq=-1` 读作"该块的覆盖区间不可知"**，绝不能当作"覆盖第 -1 行"或"覆盖空区间"而做区间运算（`[0,0]` 才是真主张，见 `store.ts:1092-1095` 的 PR #4725 N1 教训）。

---

## B7 — ✅ 通过（附关键区分）

**主张**：blocks 权威集清扫：不在本轮 state 中的块被物理 DELETE

**证据**：`store.ts:1072-1087`
```ts
// blocks: the incoming set is AUTHORITATIVE. … Drop absent ids first …
const incomingBlockIds = new Set(input.blocks.map((b) => b.blockId));
const staleBlocks = … "SELECT block_id, summary_hash, one_hash FROM blocks WHERE session_id = ?" …
for (const stale of staleBlocks) {
    if (incomingBlockIds.has(stale.block_id)) continue;
    this.db.prepare("DELETE FROM blocks WHERE session_id = ? AND block_id = ?").run(…);  // 1084 物理删除
    this.releaseContent(stale.summary_hash);                                              // 1085
    if (stale.one_hash !== null) this.releaseContent(stale.one_hash);                      // 1086
}
```

**补充（CMB 最容易踩的坑）**：这条"物理 DELETE"**只适用于 blocks**。同一函数里 messages 的处理是**相反的**——不是删，是打墓碑（`store.ts:930-950`）：
> `#4390 historian sweep: the row is absent from the authoritative set — RETIRE it, don't delete. … a rewind/undo that drops a message from the view must tombstone it, not destroy the bytes`

→ **同一份"权威集清扫"逻辑，对 messages 是"留历史"，对 blocks 是"毁历史"。** 这是 CMB 历史可查性的根本不对称（见 C.6）。

---

## B8 — ✅ 通过（附 1 处危险补充）

**主张**：`retired_at` 是视图缺席墓碑，可复活（`UPDATE messages SET retired_at=NULL`）

**证据（两半都对）**：
- 打墓碑：`store.ts:948-950`
  ```sql
  UPDATE messages SET retired_at = ? WHERE session_id = ? AND seq = ? AND retired_at IS NULL
  ```
- 复活：`store.ts:919-927`
  ```ts
  if (incomingIds.has(`${row.msg_id}\u0000${row.identity}`)) {
      if (row.retired_at !== null) {
          this.db.prepare("UPDATE messages SET retired_at = NULL WHERE session_id = ? AND seq = ?").run(…);
      }
      continue;
  }
  ```
  复活条件是 **identity 回归**（`msg_id + identity`，不是 content_hash），且**在原 seq 上原地复活**（不新插行）。

**补充（危险，必须写进 CMB 方案）**：打墓碑时 **TEXT 的 CAS 引用不释放，MEDIA 的 CAS 引用释放**（`store.ts:941-955`）：
```
// TEXT CAS refs are NOT released: retired rows still own their bytes.
// MEDIA refs ARE released (#4506): image bytes are bulky and low-value as history…
```
```ts
this.db.prepare("DELETE FROM message_media WHERE session_id = ? AND seq = ?").run(…);   // 954
for (const rm of retiredMedia) this.releaseContent(rm.media_hash);                       // 955
```
→ **一个退役消息的文本永存，但它的图片可能在退役那一刻就被删了。** 之后若复活（`store.ts:1048-1056` 会重建映射），只有当本次 ingest 载荷里**仍然带着**那份 `$media` 槽位和字节时才恢复；若客户端已不再发送该消息（典型的 rewound 历史），图片**永久丢失**，而消息行还在。
→ CMB 若把 retired 行也纳入导出，会遇到"有文本、无图、且 `hydrateMedia` 静默返回 `$media` 占位"的组合。

---

## B9 — ✅ 通过

**主张**：`getContent` 缺失时调用侧 `?? ""` 静默空串

**证据**：
- `getContent` 未命中返回 `null`：`store.ts:2677-2683`
  ```ts
  const row = … "SELECT bytes FROM content WHERE hash = ?" …;
  if (!row) return null;
  return Buffer.from(row.bytes).toString("utf8");
  ```
- 调用侧全部兜底（穷举）：

| 行号 | 调用 | 兜底 |
|---|---|---|
| `store.ts:2738` | block summary（分页读） | `?? ""` |
| `store.ts:2739` | block one（分页读） | **无兜底**（`getContent` 返回 null 时 `one` 为 null） |
| `store.ts:2979` | `uncoveredRows` 消息正文 | `?? ""` |
| `store.ts:3013` | `getMessages` 消息正文 | `?? ""` |
| `store.ts:3048` | `getMessagesAt` 消息正文 | `?? ""` |
| `store.ts:3154` | `getBlocks` summary | `?? ""` |
| `store.ts:3155` | `getBlocks` one | `?? null` |
| `store.ts:3213` | CCR 正文 | `?? ""` |

**下游后果（STATIC_INFERENCE）**：`""` 流到 `LegacyViewBuilder.addMessages` 会 `JSON.parse("")` 抛错（`ingest.ts:656`），进而落入 persist 的 corrupt-fallback 分支；而流到 `getMessages` 的调用方则可能拿到空 `content` 继续处理。**"静默"体现在没有日志、没有告警、没有计数**——CMB 若直接复用这个读法，将无法区分"这条消息本来是空"和"CAS 行丢了"。

**CMB 建议**：不要用 `?? ""`。自己判 `content` 行是否存在（`LEFT JOIN content` + `bytes IS NULL`），把缺失当**一等公民错误**上报。

---

## B10 — ✅ 通过（附 2 处限定）

**主张**：`ts` 无首见戳时填批采样时刻

**证据**：`store.ts:1029-1038`
```ts
// #4313: one wall-clock sample per save batch — rows without a
//  per-message first-sight stamp all carry the same second.
const batchTs = Date.now();
for (const m of messages) {
    …
    "INSERT OR IGNORE INTO messages (… ts) VALUES (…, ?)" …
    .run(…, typeof m.ts === "number" ? m.ts : batchTs);     // 1038
}
```
- `INSERT OR IGNORE` ⇒ 已存在行不覆盖 ⇒ **`ts` 一次写入永不更新**（schema 注释 `schema.ts:66-70`、`store.ts:142-146` 均声明 "stamped once at ingest, never updated"）。
- 分批一致：同一批里所有无戳消息共享同一个 `batchTs`。

**限定 1（措辞）**：它填的是**入库那一批的时刻**，不是"消息产生的时刻"。对首次入库的行两者近似（save 紧跟 request）；对补录/导入的行可能相差很远。所以叫"首见戳"要理解为"本库首次见到它的时刻"。

**限定 2（可为 NULL）**：导入/恢复路径不走 `batchTs`：`store.ts:1986` 是
```ts
… typeof m.ts === "number" ? m.ts : null
```
即 dump 里没有 ts 就写 **NULL**。加上 v10 迁移明确不回填（`schema.ts:391-400`："Nullable, no backfill — pre-feature rows read NULL"），所以 **`ts IS NULL` 是完全合法的常态**（v10 之前建的库、导入的库）。CMB 必须做 NULL 处理，不能 `new Date(ts)`。

---

## B11 — ✅ 通过（行号可精确到 562-569）

**主张**：加密模式 `BILI_ENCRYPTION_KEY` 生效 ⇒ unified sidecar 不打开

**证据**：`src/persist.ts:562-569`
```ts
// Encryption is a hard opt-out: index.db holds the ingest payload in
// cleartext, so the sidecar refuses to open while BILI_ENCRYPTION_KEY
// is active and the store stays legacy-only.
const storageMode = resolveStorageModeSafe();
const unifiedBlockedByEncryption = storageMode !== "legacy" && key !== null;
if (unifiedBlockedByEncryption) {
    baseLog("warn", "[storage] unified sidecar DISABLED: BILI_ENCRYPTION_KEY is active and index.db would hold cleartext ingest payloads — staying legacy-only");
}
```
- 密钥解析：`persist.ts:540-543`，`parseEncryptionKey` 要求恰好 32 字节的 hex/base64（`src/encrypt.ts:55-71`），非法值**启动即抛**（fail fast）。
- 旁证：`runtime-status.ts:21` 把该 opt-out 记为 "INTENTIONAL legacy-only"；`notice-store.ts:8` 也提到"no unified sidecar, e.g. BILI_ENCRYPTION_KEY opt-out"；`CONFIGURATION.md:2082` 同样表述。

→ **CMB 若目标环境开了加密，`index.db` 根本不存在**（严格说：不会被打开/写入，`sessions/*.json` 是 `BILIENC1` 密文）。这是接入前的**前置探测项**，不是边角情况。

---

## B12 — ✅ 通过（附 2 处补充）

**主张**：`identity` 列 = 内容身份哈希（剥 id/wireIndex），`getMessages` SELECT 不含它

**两半都成立**：

**剥位实现**（`identity.ts:86-98`）：
```ts
function identityInputOf(raw: string): string {
    … parsed = JSON.parse(raw) …
    if (parsed.id === undefined && parsed.wireIndex === undefined && parsed.ledgerSeq === undefined) return raw;
    delete parsed.id;
    delete parsed.wireIndex;
    delete parsed.ledgerSeq;
    return JSON.stringify(parsed);
}
```

**`getMessages` 不含它**（`store.ts:2998`）：
```sql
SELECT seq, msg_id, ref, ref_num, role, kind, content_hash, retired_at, ts
FROM messages WHERE session_id = ? … ORDER BY seq
```
—— 列清单里没有 `identity`。同样不含的还有 `getMessagesAt`（`store.ts:3033`）与 `uncoveredRows`（`store.ts:2962`）。只有 dump/backup 路径会 SELECT 它（`store.ts:1878`）。

**补充 1**：剥的**不只是** `id`/`wireIndex`，还有 **`ledgerSeq`**（`identity.ts:96`）。这一点主代理漏了。理由在 `identity.ts:79-85` 注释：`ledgerSeq` 是服务端晚加的**位置戳**，保留它会导致"每轮都重铸 identity、把整个视图当新行追加"。
→ CMB 若要独立重算 `identity` 做交叉校验，必须**同时剥三个字段**。

**补充 2**：`identity` **没有独立唯一索引**。唯一约束是 `UNIQUE (session_id, msg_id, content_hash)`（`schema.ts:154`），`idx_messages_identity` 是 `(session_id, msg_id, identity)` 的普通索引（`schema.ts:368`）。所以 identity 只保证"join 时能命中"，不保证"会话内唯一"——真正的主键仍是 `(session_id, seq)`。

---

# C. Block 生命周期真相

## C.1 active/inactive 如何产生与流转

**产生（永远 active=true 起步）**：`kernel/src/compress.ts:1813-1834`
```ts
const block: CompressionBlock = {
    blockId, runId, tier, topic, summary,
    directMessageIds: filteredIds,
    effectiveMessageIds: [...effectiveMessageIds],
    directBlockIds: [...consumedBlockIds],
    compressedTokens, createdAt: Date.now(),
    survivedCount: 0, generation: "young",
    active: true,                    // ← 出生即 active
    …
};
input.state.blocks.push(block);      // ← 唯一的新增点
for (const consumedId of consumedBlockIds) {
    const consumed = blockById(input.state, consumedId);
    if (consumed) consumed.active = false;   // ← 被父块消费 → 立刻失活
}
```
`blockId` 由 `allocateBlockId(state)` 发号（`kernel/src/state.ts:34-38`）：`b${nextBlockId}`，然后 `nextBlockId = max(1,id)+1`。**单调、永不复用**。

**流转（全部集中在 `syncBlocks`，每轮跑一次）**：`kernel/src/sync.ts:40-175`

| 状态转移 | 触发条件 | 证据 |
|---|---|---|
| active → **false** | 被别的块 `directBlockIds` 消费（成为父块的子块） | `sync.ts:141-145` |
| active → **false** | `expanded === true`（用户显式解压过，故意保持失活） | `sync.ts:146-153` |
| **true**（无条件先置真） | 每个非消费、非 expanded 的块都会先被置 `active = true` | `sync.ts:154` |
| true → **false** | `stillPresent === false`：覆盖 id 全不在、summary 载体不在、seq 区间内无可见消息 | `sync.ts:164-171` |
| 显式失活 | `deactivateBlock(state, blockIds, {deep?})` —— 纯函数，只改 `active: false` | `kernel/src/decompress.ts:62-100` |
| 显式失活（代理车道） | `reapOrphanBlocks(session, msgs, deactivateBlock)` 连续 N 轮无命中 | `src/orphan-gc.ts:43`、调用点 `src/server.ts:4592` |

`stillPresent` 的三个或条件（`sync.ts:164-167`）：
```ts
block.effectiveMessageIds.some((id) => presentBases.has(baseIdOf(id))) ||   // 覆盖 id 还在线上
presentIds.has(summaryMessageId(block.blockId)) ||                          // 摘要载体还在
blockSeqPresent(block, messages)                                            // seq 区间内有可见消息（#4316 layer 3）
```

**关键机制（#2695 之所以必须"删除"而非"失活"）**：`sync.ts:154` 会**先把所有块无条件置 active=true**，再判断 `stillPresent`。所以 `deactivateBlock` 的效果在下一轮 `syncBlocks` 就会被**复活**——除非块被从 `state.blocks` 里**移除**。这条写在 `src/session.ts:1850-1857` 的注释里：
> `syncBlocks` re-activates every non-consumed, non-expanded block whose id set still intersects the wire … the zombie carrier that renders (and flaps) every turn (#2695). **Removing the block from state.blocks is sticky**: sync/prune/anchors all iterate state.blocks.

## C.2 block 被删除的完整条件？物理 DELETE 还是墓碑？

**答案：物理（行 + CAS 引用一起没）。没有墓碑。** 四条删除路径：

| # | 触发 | 删除动作 | 证据 |
|---|---|---|---|
| **1** | **`reapDestroyedSubstrate`**（#2695 僵尸折叠回收）：块覆盖的消息被客户端无声重写，连续多轮"严格多数 id 不在重发历史里" | `session.state.blocks = session.state.blocks.filter(b => !blockIds.has(b.blockId))` → 下次 ingest 触发 store 权威集清扫 → **物理 DELETE** | 移除：`src/session.ts:1862-1883`（filter 在 **1872**）；落库删除：`store.ts:1077-1087`（DELETE 在 **1084**） |
| **2** | 权威集清扫（每次 `ingestSession` 都跑）：`blocks` 表里有、本轮 `input.blocks` 里没有 | `DELETE FROM blocks …` + `releaseContent(summary_hash)` + `releaseContent(one_hash)` | `store.ts:1077-1087` |
| **3** | `deleteSession` / 会话归档 gc | `DELETE FROM blocks WHERE session_id = ?` | `store.ts:1696`、`store.ts:1914` |
| **4** | 单块删除（管理/回滚接口） | `DELETE FROM blocks WHERE session_id = ? AND block_id = ?` + 释放 CAS | `store.ts:1643`、`store.ts:2066` |

**保留的痕迹（三条，都不完整）**：

1. **`sessions.orphans_json`**（#3841/#4216/#4242）：`reapDestroyedSubstrate` 保留 `blockContents`，随 ingest 计划进 `sessions.orphans_json`（`ingest.ts:439-444`：`knownBlockIds` 之外的 `blockContents` 条目 → `orphanEntries`）；store 侧 merge 语义写入（`store.ts:756-788`，注释 762-767："No orphans in this save — keep whatever the journal already holds"）。
   → **块的 `{one, full}` 摘要文本可从这里找回**，但 `payload_json`（覆盖 id 集、compressedTokens、时间）**不在**日志里。
2. **`sessions.metadata_json.preCompactionArchive`**（#395）：块 id → `{at, reason}`。
   - `reapDestroyedSubstrate` 写 `reason: "covered substrate destroyed by unannounced client history rewrite (#2695)"`（`src/session.ts:1876`）
   - `applyCompactionArchive` 写 `reason: "content replaced by client native compaction summary"`（`src/session.ts:1781`）
   → **只有 id/时间/原因，无正文、无覆盖集**。类型定义 `src/session.ts:1729`。**无容量上限**（`readPreCompactionArchive` 只做 JSON 解析，`session.ts:1731-1737`，未见裁剪代码）。
3. **`preCompactionArchive` 的镜像**：`bili export` 的 JSONL 若曾导出过，会带完整 block 记录（`export.ts:279-297`）。

## C.3 refold / 重建时 block_id 如何变化？

| 场景 | block_id | 证据 |
|---|---|---|
| **refold-in-place**（#398：同一段被解压回原位后重新折叠） | **同一个 id，原地更新**（摘要替换、`restoredInline` 清除） | 类型契约 `kernel/src/types.ts:89-95`（"compress() updates such a block IN PLACE (same id, replaced summary, flag cleared)"）；判定 `kernel/src/refold.ts:51-95`；执行点 `kernel/src/compress.ts:294` `input.state.blocks = input.state.blocks.map(…)` |
| **fork 恢复重建**（`rebuildCompressionState`） | **全新 id**（重放历史 compress 调用 → 每次 `allocateBlockId` 取新号） | `kernel/src/rebuild.ts:27-62`，循环内 `core.applyCompression` → `compress.ts:1805` `allocateBlockId` |
| **新一轮正常压缩** | 新 id（`nextBlockId` 单调递增，`state.ts:34-38`） | 同 C.1 |

→ **"块被重建后还是不是同一个块"没有稳定答案**：refold 保号，rebuild 换号。CMB 若用 `block_id` 做主键做跨版本 diff，refold 表现为"内容变了 id 没变"，rebuild 表现为"内容一样 id 全新"。

## C.4 fork / rebase 时 block 如何复制？（`cloneSessionRows`）

**答案：`cloneSessionRows` 完全不动 blocks——它只复制 messages 与 message_media。**

`store.ts:2839-2929`，事务内只有三条复制/创建语句：
```sql
-- ① 消息行（含 retired_at / ts / identity 原样继承）
INSERT INTO messages (session_id, seq, msg_id, ref, ref_num, role, kind, content_hash, identity, retired_at, ts)
SELECT ?, seq, msg_id, ref, ref_num, role, kind, content_hash, identity, retired_at, ts
FROM messages WHERE session_id = ? AND seq <= ?            -- store.ts:2881-2883

-- ② 媒体映射
INSERT INTO message_media (session_id, seq, media_hash)
SELECT ?, seq, media_hash FROM message_media
WHERE session_id = ? AND seq <= ?                          -- store.ts:2888-2890

-- ③ 子会话行（只带血缘）
INSERT INTO sessions (session_id, created_at, updated_at, parent_session_id, fork_point_seq)
VALUES (?, ?, ?, ?, ?) ON CONFLICT … SET parent_session_id=…, fork_point_seq=…   -- store.ts:2911-2918
```
- **没有** `INSERT … SELECT` 到 `blocks` / `refs` / `dead_refs` / `ccr_entries` / `session_ext`。
- refcount 纪律：**每复制一行消息，给它引用到的 hash +1**（`store.ts:2893-2910`，`putContentRef`，缺失即抛错 `store.ts:2941`——"refusing to fork with an uncounted ref"）。注释 2895-2902 记录了这一处曾经的 bug：早先用 `DISTINCT` 少算，导致"删子再删父 → 计数归零 → 零引用清扫删掉第三方还在用的媒体"。
- `throughSeq` 是**闭区间**（`store.ts:2844-2845`，#4730：端点传父的 last live seq）。
- 幂等：同一 `(parent, child, throughSeq)` 重放返回 true（`store.ts:2870-2872`）。

**那么 fork 后子会话的块从哪来？** 来自子会话自己的 `initial` ingest（`store.ts:2864`、`store.ts:2926`，`{completeLog:true}`）——即上层 `forkSnapshot` 构造的子 state（`src/plugin.ts:1549-1550` 有按 `effectiveMessageIds ⊆ rawIds` 挑选父块的逻辑）。**子会话的块是"重新生成的新块"，不是拷贝的**，因此 block_id 会重新发号。

**对 CMB 的直接后果**：`blocks` 表的行在 fork 之后**不能按 `(block_id)` 跨父子会话 join**，只能按内容/覆盖集相似度匹配。要建血缘，靠 `sessions.parent_session_id` + `sessions.fork_point_seq`（`store.ts:2911-2925`，写入语义见 C.5）。

## C.5 是否存在"块历史"可查？还是只保留当前快照？

**只保留当前快照。** 逐项确认：

| 想看什么 | 能不能看到 | 依据 |
|---|---|---|
| 当前所有块 | ✅ `blocks` 全表（active + inactive 都在，因为失活只改标志位） | `kernel/src/sync.ts:141-172`；`store.ts:3137-3160` |
| 被 refold 覆盖掉的旧摘要 | ⚠️ **只能看到最新一版**（原地更新，无版本列） | `kernel/src/types.ts:89-95`；旧 CAS 行会因 refcount 归零被删（`store.ts:1208-1213`） |
| 被 reap 删掉的块 | ⚠️ 只剩 id/时间/原因（metadata）+ 摘要文本（orphans_json），**无覆盖集/无 payload** | C.2 痕迹 ①② |
| 块之间的代际关系（谁吃了谁） | ✅ 当前快照内可查（`payload_json.directBlockIds`），历史代际不可查 | `kernel/src/types.ts:76` |
| 块"何时失活/复活" | ❌ 无时间戳、无事件表 | `blocks` 无对应列（`schema.ts:157-174`） |

**结论**：`blocks` 是一张**当前快照表**（`PRIMARY KEY (session_id, block_id)`，`ord` 是数组位置），不是事件流。唯一"近似历史"的东西是 `sessions.orphans_json` + `metadata_json.preCompactionArchive`，它们都是**旁路补丁**，不是账本。

## C.6 【关键】如果 CMB 只读 index.db，能发现多少历史？会漏掉什么？

### 能发现（VERIFIED_SOURCE）

| 维度 | 可得性 | 依据 |
|---|---|---|
| 客户端**曾发送过**的每一条消息（含已被视图淘汰的） | ✅ 完整，append-only | `store.ts:1036`（INSERT OR IGNORE，从不 UPDATE 内容）；`store.ts:930-950`（淘汰只打墓碑） |
| 消息的**到达顺序**（会话内绝对序） | ✅ `seq`（0 基，单调） | `schema.ts:143`；`ingest.ts:11-13` |
| 消息**曾经在哪个视图里** | ⚠️ 仅当前视图（`tail_json` 是覆盖式更新，`store.ts:986`） | `sessions.tail_json` 无历史版本 |
| 消息被**退役**过（曾被淘汰又回归） | ✅ `retired_at` 非空即为"当前不在视图" | `schema.ts:151`；但**退役过又复活的行 `retired_at` 会被清回 NULL**（`store.ts:925`）→ 复活后看不出它曾退役 |
| 消息的**引用编号** `mNNNNN` | ✅ `refs` 表（含死号） | `schema.ts:189-197` |
| **死号**（曾发号、消息已亡） | ✅ `dead_refs` | `schema.ts:176-183` |
| 会话**血缘**（fork 树） | ✅ `parent_session_id` + `fork_point_seq` | `schema.ts:110-111`；写入 `ingest.ts:468-476`、`store.ts:2911-2925` |
| 块**当前**状态（含被消费失活的子块） | ✅ | C.5 |
| 块**被删过**这一事实 | ⚠️ 只在 `metadata_json.preCompactionArchive` / `orphans_json` 里，且可能为空 | C.2 |
| 会话**有多少轮请求** | ✅ `stats_json.requests` | `store.ts:718-732` 的 `sessionCounters` 明确只读这两个字段 |

### 会漏掉（VERIFIED_SOURCE / STATIC_INFERENCE）

1. **块的生命周期事件流**：无"块 X 在 T 时刻失活/复活/被 reap"的记录。`blocks` 只有 `active` 当前值和 `created_at`，**没有 `updated_at`、没有 `deactivated_at`**（`schema.ts:157-174` 逐列核对）。
2. **被物理删除的块的完整 payload**：`payload_json`（覆盖 id 集、`compressedTokens`、`runId`、`startRef/endRef`）随行消失。orphans 只保 `{one,full}` 文本。
3. **refold 的历史版本**：同一 block_id 的旧摘要字节被 refcount 归零删除（`store.ts:1208-1213` 的 release 分支）。**只有最新版**。
4. **视图的历史版本**：`tail_json` 覆盖写（`store.ts:986` `tail_json = excluded.tail_json`）。**看不到"三天前这个会话的 visible tail 是什么"**。
5. **退役消息的图片**：`store.ts:951-955` 退役即释放媒体 ref → 若客户端不再重发，图片永久丢失而文本还在（见 B8）。
6. **被 gc 归档的会话正文**：`storage.archiveAfterDays` 默认 **365 天**（`knobs.ts:854-867`），归档会"release content/media refs"（`CONFIGURATION.md:206` 原文：`a tombstone row keeps the label and identity, content/media refs are released`）→ **超过一年的会话，行还在、正文没了**。这是一条**按策略的数据销毁**，不是 bug。
7. **压缩"为什么"**：块只有 `summary` 与覆盖集，没有决策依据（token 数、被拒范围、nudge 触发点）。这些在内核状态的其他字段里（`kstate_json`）部分存在，但**不是块粒度的账本**。
8. **`covered_hash` 不可信**（B4）：它从不被校验，无法用作"覆盖集未被篡改"的凭据。

### 一句话
**CMB 拿到的是"消息的完整生平 + 块的当前照片"，不是"压缩的编年史"。**

---

# D. 读取路径与并发

## D.1 `getMessages` 的真实 SQL 与过滤条件

`store.ts:2983-3016`。动态拼接，最终形态：

```sql
SELECT seq, msg_id, ref, ref_num, role, kind, content_hash, retired_at, ts
FROM messages
WHERE session_id = ?
  AND retired_at IS NULL              -- 默认；includeRetired:true 时整个子句消失
  AND seq IN (?, ?, …)                -- 仅当传 opts.seqs
  AND seq > ?                         -- 仅当传 fromSeqExclusive 且未传 seqs
ORDER BY seq
```

逐项确认：
- **`retired_at IS NULL` 是默认开**：`store.ts:2992` `const retiredFilter = opts?.includeRetired === true ? "" : " AND retired_at IS NULL";` → **默认过滤退役行**。要看全史必须显式 `{includeRetired: true}`（唯一使用者：`export.ts:265` 的 `#4664: historian read`）。
- `opts.seqs === []` 早退返回 `[]`（`store.ts:2989`）。
- 空 `seqs` 数组会被特判，避免生成 `seq IN ()` 语法错误。
- 每行 `results content` 的取法：`this.hydrateMedia(this.getContent(r.content_hash) ?? "")`（`store.ts:3013`）—— 见 B2/B9 的坑。
- 行读出计数：`messageRowReads.count++`（`store.ts:3002`，#4739 测试探针）。

**普通加载并不走 `getMessages`**：`sessionViewRows()`（`store.ts:3056-3081`）在**一个事务**里读 session + `getMessagesAt(视图指针)` + blocks + deadRefs + refs，注释明确 "Messages are fetched ONLY at the tail/pluginSnapshot pointer seqs — the historical log is not decoded on ordinary loads"（`store.ts:3059-3060`）。→ **上游自己的常规读路径只读视图窗口，不读全史**；CMB 想读全史是**非上游典型用法**。

## D.2 是否有 msg_id → seq 的索引？查询复杂度？

**有，两条独立索引**：

| 索引 | 定义 | 覆盖的查询 |
|---|---|---|
| `PRIMARY KEY (session_id, seq)` | `schema.ts:153` | `seq` 点查/区间（`WHERE session_id=? AND seq > ?`），**覆盖排序**，无需额外排序步骤 |
| `idx_messages_msg_id` | `ON messages(session_id, msg_id)`，`schema.ts:275` | msg_id 点查 |
| `UNIQUE (session_id, msg_id, content_hash)` | `schema.ts:154` | 三元组唯一 + 隐式索引（做 `INSERT OR IGNORE` 的去重判据） |
| `idx_messages_identity` | `ON messages(session_id, msg_id, identity)`，`schema.ts:368` | identity join（v8，#4390） |
| `idx_messages_session` | `ON messages(session_id, seq)`，`schema.ts:274` | 与 PK 前缀重复（冗余但无害） |

**复杂度**：
- 单会话全表：`PRIMARY KEY` 前缀扫描 → **O(N) 行读** + **每行一次 CAS 点查**（`content.hash` 是 PK，O(log M)）。所以端到端约 **O(N · log M)**。
- 视图窗口读（上游常规）：`getMessagesAt` 用 `seq IN (…)`，走 PK，**O(k · log N)**，k = 视图指针数。
- msg_id → seq：`idx_messages_msg_id` 点查 **O(log N)**。用途见 `store.ts:1557` `messagesForIds`。

**CMB 注意**：`messages` 表**没有 `session_id` 之外的分区/剪枝**，全库导出需按 `session_id` 逐个扫，或直接 `SELECT … ORDER BY session_id, seq` 一次扫（走 PK）。

## D.3 WAL 配置（`storagePragmas`）

`src/storage/schema.ts:88-101`，逐条：

```ts
export function storagePragmas(walJournalSizeLimitBytes: number): string[] {
    return [
        "PRAGMA journal_mode = WAL",                                        // :90
        "PRAGMA synchronous = NORMAL",                                      // :91
        "PRAGMA busy_timeout = 5000",                                       // :92
        "PRAGMA foreign_keys = ON",                                         // :93
        `PRAGMA journal_size_limit = ${walJournalSizeLimitBytes}`,          // :94
        "PRAGMA auto_vacuum = INCREMENTAL",                                 // :99
    ];
}
```

| PRAGMA | 值 | 含义/影响 |
|---|---|---|
| `journal_mode` | `WAL` | 读写不互斥；**第三方只读必须处理 `-wal`/`-shm`** |
| `synchronous` | `NORMAL` | WAL 下崩溃可能丢**最后若干个已提交事务**（不丢库一致性）。这是性能取舍，已是有意为之 |
| `busy_timeout` | **5000 ms** | 遇到写锁最多等 5 秒；**5 秒后抛 SQLITE_BUSY** |
| `foreign_keys` | `ON` | 强制 `messages.content_hash → content(hash)` 等外键。**CMB 可以依赖** |
| `journal_size_limit` | `8 * 1024 * 1024`（默认，`schema.ts:83`） | 每次 checkpoint 后 WAL 截断回 8 MiB（#3250；现场：曾恒定 55 MiB） |
| `auto_vacuum` | `INCREMENTAL` | 释放页可增量归还；**老库需一次全量 VACUUM 才切换**（`schema.ts:95-99`） |

另：`WAL_CHECKPOINT_EVERY_INGESTS = 20`（`store.ts:33-40`）——每 20 次成功 ingest 跑一次 PASSIVE checkpoint；打开时跑一次 `PRAGMA wal_checkpoint(TRUNCATE)`（`store.ts:583`，失败只 warn 不影响打开）。

## D.4 多进程写同一 index.db 是否被官方支持？

**支持。而且是"明确、书面、带 owner 决策日期"的支持。**

证据（`store.ts:394-415`，`StorageSecondWriterError` 的完整注释）：
```
/** @deprecated Public export surface only — no code path inside this repo
 *  throws it anymore (owner decision 2026-10-31: whatever lanes are involved,
 *  a second-writer condition NEVER blocks startup, it reports and coexists).
 *  …
 *  Owner decision 2026-10-31: multiple processes writing
 *  one index.db IS the supported topology (#5082/#5091) — SQLite WAL is
 *  multi-process safe on a local filesystem — so the condition went back to
 *  informational reporting plus the web/status surface; the genuine hazard,
 *  two instances writing the SAME session, alarms via the split-brain contest.
 *  Kept exported for external code that still imports/catches it. */
```
运行期行为（`store.ts:515-551`）：
```ts
// #4175: second-process-on-the-same-db detection. A live foreign pid
// is surfaced (web wire + banner) as INFORMATIONAL ONLY. #5082: sharing
// one index.db is the SUPPORTED topology …
// #5091/#5082 (owner decision 2026-10-31): reporting is
// INFORMATIONAL for every live foreign holder — whatever
// its lane identity (same lane, cross-lane, or unregistered
// inline/pre-#5046 marker). SQLite WAL multi-process open
// is safe on a local disk; the real hazard (two instances
// writing the SAME session) alarms separately (split-brain
// contest).
loggerLog("info", `[storage] process pid ${pid}${peerOf} is also using ${path} — multiple instances sharing one sessions dir is the supported topology (#5082/#3923); two instances walking the SAME session alarm separately (split-brain contest)`);
```
- 只是 `loggerLog("info", …)` + `recordSecondProcessOwner(pid, {…, benign: true})`（`store.ts:549`），**不阻塞、不降级**。
- 历史脉络：`#5046/B` 的 "refuse to start" 策略在 2026-10-30 被**撤回**（`store.ts:397-400`、`store.ts:519-523`）。
- 真正的危险被单独处理：**两个实例写同一个 session** → "split-brain contest"（#3923）。

**对 CMB 的含义**：CMB 若是**只读**第三方，与 bili 共存是被支持的拓扑；但 CMB 若**写**，它就是"第二个 writer"，官方立场是"允许，但同一 session 双写会触发 split-brain alarm"——等于**CMB 绝不能写**，写了没有一致性保证。

## D.5 只读连接打开方式？是否有 readOnly 选项证据？

**分两半回答，这是本审查最容易被误判的一条。**

### 生产打开路径：**没有 readOnly 选项**（VERIFIED_SOURCE）

```ts
// store.ts:443-447
static open(
    path: string,
    engine?: SqliteEngineName,
    opts?: { walJournalSizeLimitBytes?: number },
): UnifiedStore
```
第三参只有 WAL 上限，**无 readonly**。底层：
```ts
// driver.ts:152-164 openNodeSqlite
const { DatabaseSync } = require("node:sqlite") as { DatabaseSync: new (path: string) => NodeSqliteDatabase };
return new DatabaseSync(path);          // :163 ← 不传任何选项
```
```ts
// driver.ts:473-497 openSqliteDatabase
export function openSqliteDatabase(path: string, engine?: SqliteEngineName): OpenedDatabase
```
同样无 readonly 参数。better-sqlite3 分支也是 `new ctor(path)`（`driver.ts:488`），不传 options。

**`readOnly` 在 driver.ts 里的真实含义**是**事务类型**，不是连接模式：
```ts
// driver.ts:166
const begin = (readOnly: boolean) => db.exec(depth === 0 ? (readOnly ? "BEGIN DEFERRED" : "BEGIN IMMEDIATE") : `SAVEPOINT sp${depth}`);
// driver.ts:201
readTransaction: (fn) => transaction(fn, true),
```
→ `UnifiedStore.readSnapshot()`（`store.ts:605-608`）用的是 `db.readTransaction`，即 **DEFERRED 事务（拿读快照）**，连接本身仍是读写连接。**这是"快照隔离"，不是"只读打开"**。CMB 不要误读成只读支持。

### 真·只读打开：仓库内**有先例**，但只在非主路径（VERIFIED_SOURCE）

```ts
// integrity-check.ts:161-163（后台 quick_check 的 worker 源码字符串内）
const openReadOnly = isBunRuntime
    ? (p) => new (require(BUN_SQLITE_SPEC).Database)(p, { readonly: true })
    : (p) => new (require(SQLITE_SPEC).DatabaseSync)(p, { readOnly: true });
```
用途：`scheduleBackgroundIntegrityCheck`（`integrity-check.ts:192-204`）在 store **已打开并服务中**的情况下，起一个 Worker，用**第二条只读连接**跑 `PRAGMA quick_check`。注释（`store.ts:589-594`）描述为 "runs post-open on a second read-only connection"。

测试里的直接只读打开（≥6 处，证明该用法在本仓环境可用）：
- `tests/msgid-lane-flow.test.ts:95` — `new DatabaseSync(join(dir,"sessions","index.db"), { readOnly: true })`
- `tests/seq-interval-lane.test.ts:105`、`:114`
- `tests/issue4612-gc-online-guard.test.ts:187`
- `tests/d5-5-render-window.test.ts:229`（反例：`{ readOnly: false }`，用于写入断言）

**CMB 可用结论**：
1. 只读打开 `index.db` 有**仓库内同环境先例**，可行性 VERIFIED_SOURCE。
2. 但**上游不为第三方读者提供任何契约**（无 API、无文档、无版本承诺）。`STORAGE_SCHEMA_VERSION = 10` 会继续演进（v1→v10 已有 10 次迁移，`schema.ts:37-71`），**列会增删**。
3. WAL 模式意味着只读连接需要 `-wal`/`-shm` 可用；**在 bili 未运行时（干净 shutdown 后）最安全**；bili 运行中读取，需接受"读到某时刻快照"。
4. **不要用 `?immutable=1`**：会忽略 `-wal`，读到**缺少最近提交**的旧快照（STATIC_INFERENCE，非仓库证据）。

---

# E. 消息类型与身份

## E.1 CoreMessage 的真实字段全集

`kernel/src/types.ts:8-55`，逐字：

```ts
export interface CoreMessage {
  id: string;                          // :9   内容派生 id（见下）
  role: MessageRole;                   // :10  "user" | "assistant" | "system" | "tool"
  contentType: MessageContentType;     // :11  "text" | "tool-call" | "tool-result" | "reasoning"（必填）
  text?: string;                       // :12
  toolName?: string;                   // :13
  toolCallId?: string;                 // :14
  thinkingTokens?: number;             // :22  仅计量，不渲染/不截断/不索引
  summaryOfBlockId?: string;           // :32  宿主自带的块摘要载体（#335）
  ledgerSeq?: number;                  // :42  服务端账本号（#4316 layer 3），不序列化到 wire
  ts?: number;                         // :54  不可变本地时间戳（epoch ms，#4313）
}
```
共 **10 个字段**（4 必填 + 6 可选）。注意 `ledgerSeq` 的注释（`types.ts:33-42`）明确："Not serialized into wire payloads — it rides CoreMessage sidecars and the persisted store rows verbatim"，即**它会进 CAS blob**（这也解释了为什么 `identity` 必须剥它，`identity.ts:96`）。

`MessageRole`（`types.ts:3`）：`"user" | "assistant" | "system" | "tool"` —— **4 个值，无 `developer`、无 `function`**。

**`developer` 去哪了（重要，有精确证据）**：OpenAI/Responses 线的 `developer` 角色在进内核时被**归一成 `system`**，但原始角色存在一个 sidecar 字段里，回写时恢复：

```ts
// kernel/src/wire/bili-message.ts:88-90
/** OpenAI: original role was "developer" (reconstructed as "system" by
 *  openaiToCore for the kernel; coreToOpenai restores "developer"). */
originalRole?: "system" | "developer";
```
- 归一：`openaiToCore` 把 `developer → system`；回写：`openai.ts:488`、`openai.ts:496` 用 `m.originalRole === "developer" ? "developer" : "system"` 还原。
- 因此 **`index.db` 里 `messages.role` 只会是 4 个值之一**；一个原本是 `developer` 的消息，行里是 `role='system'`，而 CAS blob 里带着 `originalRole:"developer"` 这个额外字段（`CoreMessage` 类型未声明它，属于 wire sidecar，随 blob 原样保留）。
- → **CMB 若需要还原"这是 system 还是 developer"，必须读 blob 里的 `originalRole`，不能只看 `role` 列。** 这是本审查发现的又一处"列投影丢失信息"。

## E.2 `contentType` 有哪些真实取值？（证据，非猜测）

**取值域定义**：`kernel/src/types.ts:5-6`
```ts
export type MessageContentType =
  "text" | "tool-call" | "tool-result" | "reasoning";
```
**4 个值。**

独立印证（消费/序列化处）：

| 证据 | 位置 | 说明 |
|---|---|---|
| 类型定义 | `kernel/src/types.ts:5-6` | 唯一权威定义 |
| 序列化处 | `ingest.ts:195`、`ingest.ts:341` | `kind: parsed.contentType ?? "message"` |
| 持久化投影自检 | `store.ts:2498-2500` | `verifyProjections` 逐行断言 `(parsed.contentType ?? "message") === m.kind` |
| 消费处（工具调用） | `kernel/src/rebuild.ts:74` | `if (message.toolName !== "compress" \|\| message.contentType !== "tool-call") continue;` |
| 消费处（reasoning 判定） | `src/reasoning-guard.ts:302` | `st.kind.set(oi, "reasoning")` |
| 导出 | `export.ts:277` | `kind: m.kind ?? undefined` |

**不存在第 5 个值。** 特别地：**没有 `"image"`、没有 `"system"`、没有 `"tool"`** —— 图片走 `text` + `imageBase64` 槽位（见 A.2）；system/tool 的区分在 `role` 上，不在 `contentType` 上。

**兜底值 `"message"` 不属于取值域**：它只在 `parsed.contentType` 缺失时出现（历史载荷 / 非 CoreMessage 形状）。因为 `contentType` 是必填（`types.ts:11`），正常写入路径不会产生 `kind="message"`（STATIC_INFERENCE，可由 `store.ts:2498` 的断言反证：若真出现 `"message"`，那 `parsed.contentType` 必为 undefined）。

## E.3 reasoning / system / tool / toolResult 如何区分？

**全部靠 `role` + `contentType` 的二维组合，没有第三个区分维度。**

| 语义 | role | contentType | 附加字段 | 证据 |
|---|---|---|---|---|
| 普通用户/模型文本 | `user` / `assistant` | `text` | `text` | `types.ts:3,5` |
| **system 提示** | `system` | `text`（或 `tool-call`） | — | `types.ts:3`（`system` 是 role 值） |
| **工具调用** | `assistant`（通常） | `tool-call` | `toolName` + `toolCallId` + `text`（= 参数 JSON） | `types.ts:13-14`；消费 `rebuild.ts:74` |
| **工具结果** | `tool` | `tool-result` | `toolCallId`（配对键） | `types.ts:3,5,14` |
| **reasoning（思考）** | `assistant` | `reasoning` | `text` = 思考正文；`thinkingTokens` = 计量 | `types.ts:5-6,22`；`reasoning-guard.ts:302` |
| **压缩摘要** | `system` | `text` | `id` 以 `acp_summary_` 开头（保留前缀）；或 `summaryOfBlockId` | `types.ts:26-32` |

**关键**：
- **工具调用/结果靠 `toolCallId` 配对**，不是靠相邻位置（`types.ts:14` 注释；配对逻辑见 `kernel/src/tool-pairs.ts`、`src/tool-pair-order.ts`）。
- **reasoning 是一种独立 `contentType`，不是 `text` 的变体**。它是 `thinking`/`redacted_thinking`/`thoughtSignature` 家族，AGENTS.md KDD #10 明确"NEVER rewrite them"（签名校验）。→ CMB 以只读方式导出时**必须原样保留**，任何重排都可能导致上游 session 报废（这是上游的硬约束，也是 CMB 不能"重写后再喂回去"的原因）。
- **`system` 在 `contentType` 上没有独立取值**。`role === "system"` 是唯一判据。

## E.4 Session 类型有哪些字段？有没有 tenantId / groupId / botId？

**答案：没有。一个都没有。**

**负证据（穷举 grep）**：对 `src/*.ts`、`src/storage/*.ts`、`kernel/src/*.ts` 执行
`Select-String -Pattern "tenantId|tenant_id|groupId|group_id|botId|bot_id"`
→ **0 匹配**。

**Session 类型全文结构**（`src/session.ts:307` 起，>370 行）：

| 字段 | 行号 | 类型/说明 |
|---|---|---|
| `id` | `session.ts:347` | `string` —— 会话 id（客户端原生 id 或派生） |
| `meta` | `session.ts:349-382` | `{protocol?, upstreamOrigin?, label?, title?, hostTitle?, activePack?, summaryInstructions?}` |
| `stats` | `session.ts:385-617` | `{requests, costUsd?, tokensSaved, inputTokens, cachedTokens, outputTokens, cacheSamples, lastInputTokens, lastInputTokensSource?, emergencyShrinkAt?, …, storedBytes, storeBytesSaved, rangeRestores, wholeBlockRestores?, …}` |
| `metadata` | `session.ts:621` | `Record<string, unknown>` —— **自由逃生口**（`pluginAgent`、`derivedFromSessionId`、`preCompactionArchive`、`compactionBoundary` 等都塞这里） |
| `state` | `session.ts:622` | `CompressionState`（内核状态） |
| `restampSuperseded?` | `session.ts:630` | `Map<string,string>`，进程内 |
| `createdAt` / `lastSeen` | `session.ts:631-632` | `number` |
| `blockContents` | `session.ts:646` | `Map<string, BlockContent>` —— 块原文本 |
| `lastMessages?` | `session.ts:652` | `CoreMessage[]` —— **当前视图快照**（客户端每轮全量重发） |
| `lastMessagesFolded?` | `session.ts:659` | `boolean` —— 该快照是否已折叠渲染 |
| `pluginSnapshot?` | `session.ts:660` | `CoreMessage[]` —— 冻结的原始前缀 |
| `decompositionLog?` | `session.ts:665` | `CoreMessage[]` —— 外部转换器导入载荷（一次性） |
| `revisionEpoch?` | `session.ts:669` | 内存only |
| `pluginRevisionCache?` | `session.ts:673` | 内存only |
| `contentStore?` | `session.ts:679` | `MessageContentStore`（CCR），独立文件 |
| `transient?` | `session.ts:309-346` | 内存only（不持久化） |

**身份层缺失清单（对 CMB 是硬约束）**：

| CMB 通常需要的 | 上游有吗 | 替代 |
|---|---|---|
| `tenantId` | ❌ | 无。`namespace` 是**agent 车道名**（`proxy`/`dsh`/`pi`/…），不是租户 |
| `groupId` | ❌ | 无 |
| `botId` | ❌ | `metadata.pluginAgent` 近似（车道内的 agent 标识） |
| `userId` | ❌ | 无 |
| `orgId` | ❌ | 无 |
| 上游账号/密钥归属 | ❌ | 只在 `meta.upstreamOrigin`（URL） |
| 会话的人类可读名 | ✅ | `host_title` / `title` / `label`（`schema.ts:113-115`） |

→ **CMB 若需要多租户隔离，必须在 CMB 侧自己建映射（session_id → tenant），上游不提供任何租户语义。** 这是接入设计的一个硬前提，不是配置问题。

## E.5 namespace / conversations 表的作用

**`sessions.namespace`**（`schema.ts:116`）：
- 默认 `'proxy'`；写入时取 `session.metadata.pluginAgent`，为空则 `'proxy'`（`ingest.ts:484-486`）：
  ```ts
  namespace: typeof session.metadata.pluginAgent === "string" && session.metadata.pluginAgent
      ? session.metadata.pluginAgent
      : "proxy",
  ```
- 语义（`schema.ts:37-45` v2 注释）：**"the agent namespace that owns the conversation"** —— 即**哪条 agent 车道**产生的会话，**不是租户**。

**`conversations` 表**（`schema.ts:229-235`）：
```sql
CREATE TABLE IF NOT EXISTS conversations (
    namespace        TEXT NOT NULL,
    conversation_id  TEXT NOT NULL,
    session_id       TEXT NOT NULL,
    claimed_at       INTEGER NOT NULL,
    PRIMARY KEY (namespace, conversation_id)
) WITHOUT ROWID;
```
- 作用（`schema.ts:222-228` 注释原文）：**"(namespace, conversation_id) 对是会话的持久身份；sessions.session_id 是它当前解析到的行键。v1 阶段两者相同，表的存在是为了让唯一性契约成为承重结构，并为行键解耦（R1/Phase 3）提供免迁移的落点。"**
- 写入（`store.ts:1019-1027`）：`INSERT … ON CONFLICT(namespace, conversation_id) DO NOTHING` —— **先到先得，同 id 重摄入是 no-op**。
- 回填（`schema.ts:311-313`）：v3 迁移从 `sessions` 自身生成。

**对 CMB 的实际用途**：
- 若 CMB 只按 `sessions.session_id` 索引，**v1 阶段够用**（两值恒等）。
- 但**不要写死这个假设**：上游已明说 Phase 3 会把行键与身份解耦（`schema.ts:226-228`）。CMB 应统一通过 `conversations(namespace, conversation_id) → session_id` 解析，未来才不破。
- `deleteSession` **会**清理这条索引：`DELETE FROM conversations WHERE session_id = ?`（`store.ts:1700`，完整删除序列 `store.ts:1694-1702`）。所以这是仅有的"级联干净"的一张表。
- **但同时注意**：同一次 `deleteSession` 会**物理删除** `messages` / `message_media` / `blocks` / `dead_refs` / `refs` / `ccr_entries`（`store.ts:1694-1699`）——**会话一旦被删，全部历史（含消息）立即消失**，不是墓碑。这与"消息用墓碑保全史"（B8）**并不矛盾**：墓碑保护的是"会话内部视图淘汰"，`deleteSession` 是"整会话销毁"这一更高层动作。CMB 必须把 `deleteSession` 视为**不可恢复的硬删除**（除 pinned checkpoint 外，`store.ts:1686-1693`）。

---

# F. 上游自身的问题（对 CMB 有影响）

## F.1 关于"832 个 open issue"这一前提

**本次无法验证 832 这个数字，且它很可能口径不符。**

- 审查时（2026-11-06）对 `https://api.github.com/repos/ranxianglei/billion-context` 的实时读取：`"open_issues_count": 844`。
- GitHub 的 `open_issues_count` **同时包含 issue 与 PR**（官方语义），所以它既不是"issue 数"也不是稳定值。
- 仓库体积 65 MB，`pushed_at: 2026-11-05T21:05:29Z`，`stargazers: 4120`，`forks: 334`。

→ **"832 个 open issue"标记为 UNVERIFIED（数字层面），"该仓库存在数百个未关闭条目"标记为 VERIFIED（API 事实）。** 本报告不基于 issue 数量做任何论断，改用**代码内注释引用**（可逐行复核）作为证据源。

## F.2 代码注释中与数据完整性/持久化相关的 issue（摘录，可直接复核）

这是本段唯一可靠的证据源——每条都是上游作者写在代码里的自述。按主题分组：

### ① CAS 泄漏 / 孤儿 blob（已修，但暴露过规模）

| Issue | 内容 | 证据 |
|---|---|---|
| **#3987 / #4242** | 旧实现"只减计数不删行"，在生产语料上留下 **11.32 GB** 零引用孤儿；修复为 delete-on-zero | `store.ts:790-798`（原文："11.32 GB of orphans on a real corpus … the only sweeps lived in deleteSession()/gc(), which nobody runs on a schedule"） |
| **#4796** | 零引用清扫无索引 → 现场 **766K 行 / ~20 GB**，一条未提交的 `DELETE … WHERE refcount <= 0` **跑了 8 小时零提交**，其 WAL 帧永不 checkpoint，写锁**饿死在线代理**。修复：加 `idx_content_refcount` + 分批提交 | `schema.ts:378-390`（"Field report: 766K rows / ~20 GB, one uncommitted `DELETE … WHERE refcount <= 0` ran 8 hours with zero committed rows"）；分批常 `store.ts:46-51`（`GC_ZERO_REF_BATCH_SIZE = 2000`）、`store.ts:2196`（"ran 8 hours, zero rows changed"） |
| **#4506** | 退役消息的**媒体** ref 不释放 → 无界泄漏；修复为退役即释放媒体 ref（正文 ref 保留） | `store.ts:941-955` |
| **#3762 / #3901** | 媒体外置后 refcount 一度按 `DISTINCT` 算，导致"删子再删父 → 计数归零 → 零引用清扫删掉第三方还在用的媒体" | `store.ts:2893-2902`（修复说明）、`store.ts:2937-2941`（`putContentRef` 缺行即抛"refusing to fork with an uncounted ref"） |

### ② 库不可打开 / 迁移 OOM（已修）

| Issue | 内容 | 证据 |
|---|---|---|
| **#4677** | v8 identity 迁移在**一个事务内** `.all()` 载入每条消息的 content 字节 → 多 GB 库**首次打开即 OOM**；且 `user_version` 提升在同一事务里 → 每次打开重跑重 OOM → **永久不可打开的库**。修复：按会话分事务、可续跑、分页 | `schema.ts:369-376`（原文："v0.1.207 loaded EVERY message's content bytes into memory in a single .all() here inside one transaction, so multi-GB stores OOM'd on first open — and because the user_version bump sat in the same transaction, every subsequent open re-ran the whole migration and re-OOM'd: permanently unopenable stores (#4677)"）；实现 `schema.ts:425-514` |
| **#4175** | `PRAGMA quick_check` 在打开时跑；损坏时**不做任何自动修复**（自动恢复已撤回），只告警并让进程以"无索引"方式启动 | `store.ts:352-368`（`StorageIntegrityError` 文档注释）、`store.ts:449-514` |
| **#5108 / #4816** | 损坏形状的**抛错**（截断文件、垃圾头、坏溢出链）曾以原始 driver 错误逃出完整性守卫 → 损坏文件集永远留在原地。修复：归一为 `StorageIntegrityError` | `store.ts:483-500`、`store.ts:567-573`；分类器 `sqlite-errors.ts` |
| **#4609** | 14 GB 库的 `quick_check` 约 **260 秒**，超过 60 秒原生 spawn 预算 → 子进程被杀；改为"大库后台检查" | `store.ts:454-456`；实现 `integrity-check.ts:192+` |
| **#4939** | 大 `index.db` 打开是**整库前置**（完整性检查 + WAL 恢复），导致两车道共用一个 store 冷启会争写锁；现场观测 **28s+ 到首行日志、60s+ 无输出**（Windows，1.7 GiB store） | `CONFIGURATION.md:2076`（`BILI_SPAWN_WAIT_MS` 条目） |

### ③ 数据不一致 / 视图漂移（部分开放）

| Issue | 内容 | 证据 |
|---|---|---|
| **#4182** | **增量摄入（watermark fastpath）的设计性缺口**：同一 tail 内两条**字节相同**的消息共享一个内容派生 identity，双双通过"已存 identity"探针，但 `UNIQUE` 约束静默跳过第二条 → 其 tailSeqs 槽指向空 → **读视图丢一条，持续一个保存周期**。作者自述："Bounded and unusual-input-only; closing it … is a separate decision"（**即：至今未关**） | `ingest.ts:244-252`（原文完整保留）；下一保存自愈机制见 `ingest.ts:248-250` |
| **#4216** | `record()` 曾因 orphan 形状无法表示而**跳过 ingest** → 会话冻结在数小时前的行；修复：orphan journal 上路 | `ingest.ts:433-438`（"This is what makes the reap shape REPRESENTABLE, so record() no longer skips the ingest (the freeze in #4216: sessions stuck at hours-old rows)"） |
| **#3166** | 水位以下**同 id 字节漂移**：增量路径永远不会改写已存行，只能靠"每 N 次采样校验"兜底 | `ingest.ts:258-266`（"Interior edits BELOW the boundary are the watermark design's accepted trade-off — they never mis-attribute, only leave stale rows"）；采样周期 `shadow.ts:65` `VERIFY_EVERY_INGESTS = 20` |
| **#4323** | 连续 3 次采样漂移才升级告警（`DRIFT_ESCALATE_STREAK = 3`），因为单次漂移可能是外部churn | `shadow.ts:67-75` |
| **#4390** | 头截断会重铸整代哈希（"new hashes, new seqs, block interval bounds built on sand"）→ 引入 position-independent `identity` | `identity.ts:1-22`、`schema.ts:351-377` |
| **#4608** | 重存时用计划内相对 seq 会**覆盖掉**已durable的区间 → 加 `CASE WHEN excluded.start_seq >= 0` 棘轮 | `store.ts:1158-1195` |
| **#4801** | 块边界曾跨退役代际 → 边界声称覆盖了replay从不服务的行；修复：span 查询加 `retired_at IS NULL` | `store.ts:1107-1114`（SQL 在 `store.ts:1119`） |
| **#4730** | fork 端点传 `liveRowCount-1` 而非 last live seq → 克隆错位 | `store.ts:2844-2845`（"the endpoint passes the parent's last live seq — #4730"） |
| **#4739** | 普通加载曾解码**整条历史日志** → 改为只取视图指针覆盖的行 | `store.ts:56-62`、`store.ts:3018-3022` |
| **#5155** | 热路径指针维护曾走全表 | `store.ts:2987-2988`（`fromSeqExclusive`） |

### ④ 只写不校验 / 未完成项（对 CMB 直接可用性有影响）

| 发现 | 证据 | 影响 |
|---|---|---|
| `covered_hash` **从不被任何代码校验** | 唯一写入点 `ingest.ts:428`；`verifyProjections` 不碰它（`store.ts:2503-2515`）；测试手写 `"[]"`（`tests/issue4739-storage-read-snapshot.test.ts:27`） | CMB 不能把它当完整性凭据 |
| `supersedes_block_id` **恒 NULL** | `ingest.ts:425` 硬编码；内核类型无此字段（`types.ts:68-110`） | 块血缘不能靠它 |
| `blocks` **无墓碑机制** | `schema.ts:157-174` 无 `retired_at` | 块历史不可重建（C.6） |
| `getContent` 缺失 **静默 `""`** | `store.ts:2738/2979/3013/3048/3154/3213` | CMB 需自建缺失检测（B9） |
| `generate` 的 `blockContents` 在 **reap 后仍保留**，但 `blocks` 行已删 | `session.ts:1859-1860`、`ingest.ts:439-444` | 需从 `orphans_json` 取摘要文本 |
| 设计文档 `6-OVERALL-DESIGN.md` **在仓库中不存在** | `schema.ts:3` 引用；全仓 `Get-ChildItem -Filter "*OVERALL-DESIGN*"` → **0 结果** | 权威 schema 设计说明**不可得**，只能读代码（DOCUMENT_ONLY 缺口） |

### ⑤ Windows/环境类（影响 CMB 的部署面）

| Issue | 内容 | 证据 |
|---|---|---|
| **#3823** | `SQLITE_CANTOPEN`（"unable to open database file"）：杀软/Defender、OneDrive 占位文件、路径 >260、受限进程 | `shadow.ts:78-96`（给出可操作排查串） |
| **#4614** | `attempt to write a readonly database`（文件已存在但首写被拒）：Windows 完整性控制（MIC No-Write-Up）、AV 锁、独占写者 | 同上 |
| **#5154** | `node.exe` 自身带 **Low integrity label** → 所有对 Medium 文件的写失败，而 ACL 全对；`icacls` 可诊断/修复 | `shadow.ts:100-103`（完整修复步骤）；`CONFIGURATION.md:2568` |
| **#5096** | 由上述机制导致的"无可见原因的崩溃循环"启动类 | `CONFIGURATION.md:2568` |

**CMB 部署提示**：以上四条**全部是 win32**。CMB 若在 Linux 侧只读接入，这一整类风险不适用；若在客户 Windows 机器上部署，需把 `icacls` 完整性标签纳入前置检查。

## F.3 F 段的诚实边界

- 本段全部是**代码注释自述**，不是独立的 issue tracker 审查。上游作者的自述可能滞后于实际修复状态（尤其标"已修"的，我只验证了修复代码存在，**没有验证修复有效性**）。
- **未验证**：这些 issue 的当前状态（open/closed）、是否有回归、是否有更新版本再次破坏。需要 GitHub API 逐条查询（本次仅做了仓库元数据一次调用）。
- **未验证**：`#4182` 那条"Bounded and unusual-input-only"是否仍在线上存在（注释是 HEAD 时的，说明**代码里仍保留该缺口**，注释 `ingest.ts:244-252` 明确写"closing it … is a separate decision"）。

---

# G. 对 CMB 接入的建议（基于以上证据）

## G.1 三种接入路径的可行性评级

| 路径 | 可行性 | 依据 | 风险 |
|---|---|---|---|
| **A. 只读 `index.db`（SQLite 直连）** | ⚠️ 可行但脆弱 | D.5 有先例；schema v10 | schema 演进无契约；WAL 语义要自负；媒体需二次 JOIN；`?? ""` 静默缺失 |
| **B. 走上游 `export` JSONL**（`src/export.ts:240-300`） | ✅ 最稳 | 官方代码路径，`#2671 Phase 2c` 明确"generated artifact — never a source of truth"；含 `retiredAt`（`export.ts:265`）、block 完整记录（`export.ts:279-297`） | 需要跑 CLI/HTTP，不是实时；`full` 模式才deref CCR |
| **C. 只读 legacy JSON envelope（`sessions/*.json`）** | ✅ 可行 | 默认仍在写（`storage.jsonWriter` 默认 on，`knobs.ts:893-898`） | 加密模式唯一可行路径；但**不是**默认读路径，且 `jsonWriter=off` 时不存在 |

**推荐：B 为主（导出面），A 为补充（实时/增量），C 为兜底（加密或老版本）。**

理由：
- B 是唯一"上游自己承认的产物"，`export.ts:14-17` 注释自述"Deterministic JSONL dump … Generated artifact — never a source of truth"，语义稳定。
- A 能拿实时数据，但必须自己实现 hydrate（媒体）+ 缺失检测（`$media` 残留）+ retired 语义。
- C 覆盖加密场景（B11：加密时 `index.db` 根本不存在）。

## G.2 若走 A（只读 index.db），最小可用投影 SQL

```sql
-- 会话清单
SELECT session_id, parent_session_id, fork_point_seq, protocol, label,
       title, host_title, namespace, origin, status,
       created_at, updated_at, next_block_id, next_run_id,
       last_ingested_seq
FROM sessions;

-- 全史消息（含退役），正文 + 媒体占位
SELECT m.session_id, m.seq, m.msg_id, m.ref, m.ref_num, m.role, m.kind,
       m.identity, m.retired_at, m.ts,
       c.bytes AS body_bytes, c.byte_len, c.refcount
FROM messages m
LEFT JOIN content c ON c.hash = m.content_hash
WHERE m.session_id = ?
ORDER BY m.seq;

-- 该行的媒体（需再查 content 拿 base64）
SELECT mm.seq, mm.media_hash, c.byte_len
FROM message_media mm
LEFT JOIN content c ON c.hash = mm.media_hash
WHERE mm.session_id = ?;

-- 当前视图顺序（模型实际看到的序列）
SELECT tail_json FROM sessions WHERE session_id = ?;
-- 解析 {tailSeqs:[…], pluginSnapshotSeqs:[…], tailFolded:bool}

-- 块快照
SELECT block_id, ord, start_seq, end_seq, start_ref, end_ref, tier,
       summary_hash, one_hash, supersedes_block_id, active, created_at,
       covered_hash, payload_json
FROM blocks WHERE session_id = ? ORDER BY ord, block_id;
```

**必须自己实现的 5 个防御**：
1. `c.bytes IS NULL` → 判为"CAS 行缺失"，**不要**退化成 `""`（B9）。
2. `LENGTH(body_bytes) > 0 AND body_bytes LIKE '%"$media"%'` → 需 hydrate，否则是占位符（B2）。
3. `ts IS NULL` → 不要 `new Date(null)`（B10）。
4. `start_seq = -1 AND end_seq = -1` → 判为"无区间主张"，跳过区间运算（B6）。
5. `kind` 的 `"message"` 兜底值 → 视为 `contentType` 缺失，不要当第 5 种类型（A.5/E.2）。

**完整性自检（可做，成本低）**：对每个被引用的 hash 验证 `sha256(content.bytes) == content.hash`。这是上游 CAS 的硬不变量（A.6）。**唯一例外**：`precomputedHash` 由 planner 传入时可能与 bytes 不一致（理论上不会，因为同一份字节），但**值得抽样验证**。

## G.3 数据保留红线（CMB 必须与产品方确认）

| 机制 | 默认 | 后果 | 证据 |
|---|---|---|---|
| `storage.archiveAfterDays` | **365 天** | 超过一年的会话会被归档：**content/media refs 被释放**，只留 label+identity 的 tombstone 行 | `knobs.ts:854-867`；`CONFIGURATION.md:206` |
| 退役消息的**媒体** | 立即释放 | 消息文本永存，图片可能永久丢失 | `store.ts:941-955`（B8） |
| 块被 reap | 立即物理删除 | 块 payload 不可恢复（仅 orphans 文本残留） | `store.ts:1077-1087`（C.2） |
| 会话 `deleteSession` | 显式调用 | 行 + CAS 全删（pinned checkpoint 除外） | `schema.ts:208-211` 注释 |
| `synchronous = NORMAL` | 默认 | 崩溃可能丢最后若干已提交事务 | `schema.ts:91`（D.3） |

→ **CMB 若承诺"完整历史归档"，必须自建落盘（导出到 CMB 侧存储），不能依赖 `index.db` 的长期保留。**

## G.4 接入前必须探测的 4 件事

1. **加密是否开启**：`BILI_ENCRYPTION_KEY` 非空 → `index.db` 不存在，走 C 路径（B11）。
2. **`storage.unified` 实际值**：`BILI_STORAGE_UNIFIED` / 配置文件 `storage.unified`；`legacy` 时不写 `index.db`（`knobs.ts:869-881`）。注意：**默认 `full`（#3818 之后）**，但**老安装**可能是 `legacy`。
3. **`storage.jsonWriter`**：`off` 时没有 JSON 兜底（`knobs.ts:883-898`）；`off` 且 `index.db` 损坏 = 无备份。
4. **schema 版本**：`PRAGMA user_version`（`schema.ts:288`、`schema.ts:403`）。≠ 10 则列集可能不同，CMB 需按版本分支或拒绝（**不要**对未知版本做乐观解析）。

---

# 附录 A：证据索引（按文件）

| 文件 | SHA-256 前 12 位 | 本报告引用行 |
|---|---|---|
| `src/storage/schema.ts` | `348cd085a141` | 37-71, 71, 83, 88-101, 108-131, 133-139, 141-155, 157-174, 176-183, 189-197, 199-212, 214-220, 229-235, 243-249, 255-261, 267-272, 274-279, 283-405, 311-313, 338-350, 351-377, 368, 378-390, 389, 391-400, 425-514, 489-490 |
| `src/storage/ingest.ts` | `8debce0abb3d` | 11-13, 42, 149-160, 166-175, 190-207, 194, 195, 199, 203, 205, 244-252, 258-266, 323-346, 341, 372-377, 402-431, 406, 407-420, 425, 428, 429, 433-444, 461-466, 468-476, 484-486, 498, 590-696, 656, 659-695 |
| `src/storage/store.ts` | `58a8162318df` | 33-51, 56-62, 70-74, 76-78, 110, 142-146, 352-368, 394-415, 443-447, 449-514, 483-500, 515-551, 558-559, 567-573, 583, 589-594, 605-608, 718-732, 735-753, 741, 746, 749, 756-788, 790-809, 832-1297, 911-957, 923-927, 930-950, 941-955, 948-950, 1029-1038, 1036, 1038, 1048-1068, 1072-1087, 1084, 1088-1146, 1092-1095, 1096-1129, 1119, 1131-1144, 1143, 1158-1195, 1183-1184, 1208-1213, 1557, 1690-1710, 1878, 1986, 1990-2002, 2112-2118, 2196, 2461-2517, 2496-2500, 2503-2515, 2677-2683, 2738-2739, 2839-2929, 2844-2845, 2870-2872, 2881-2890, 2893-2910, 2911-2925, 2931-2941, 2957-2981, 2983-3016, 2992, 2998, 3018-3022, 3024-3054, 3056-3081, 3100-3128, 3137-3160, 3154-3155, 3213 |
| `src/storage/identity.ts` | `c4090622fa7b` | 1-22, 25-27, 33, 40-77, 43-58, 51-57, 65-75, 76, 79-85, 86-98, 96, 105-107 |
| `kernel/src/types.ts` | `976db3fe453d` | 3, 5-6, 8-55, 9-14, 22, 26-32, 32-42, 33-42, 54, 68-110, 74-76, 89-95 |
| `kernel/src/sync.ts` | `53788d715371` | 6-8, 40-175, 141-145, 146-153, 154, 164-171 |
| `kernel/src/state.ts` | `5ed5400e29d2` | 34-38, 53-55 |
| `kernel/src/decompress.ts` | `1a8ed2d46b90` | 40-56, 58-60, 62-100, 69-77, 102, 107 |
| `kernel/src/compress.ts` | — | 1805, 1813-1834, 1836-1839, 201-207, 294 |
| `kernel/src/refold.ts` | — | 16-18, 51-95 |
| `kernel/src/rebuild.ts` | — | 17-26, 27-62, 74 |
| `kernel/src/wire/bili-message.ts` | — | 88-90（`originalRole` sidecar） |
| `kernel/src/wire/openai.ts` | — | 43, 86, 488, 496, 553 |
| `src/session.ts` | `65155a3d5742` | 307, 309-346, 347, 349-382, 385-617, 621, 622, 630, 631-632, 646, 652, 659, 660, 665, 669, 673, 679, 1729, 1731-1737, 1765-1805, 1781, 1814-1846, 1848-1883, 1850-1857, 1872, 1876 |
| `src/knobs.ts` | `6059a5e54d39` | 841-848, 854-867, 869-881, 880, 883-898 |
| `src/persist.ts` | `1a9c74e06854` | 540-543, 562-569 |
| `src/storage/driver.ts` | `2c7cafcf6a0a` | 152-164, 163, 166, 201, 473-497 |
| `src/storage/integrity-check.ts` | `58ecaa8aca9e` | 161-163, 192-204 |
| `src/export.ts` | `54177053ec1b` | 10-18, 14-17, 240-300, 263-265, 277, 279-297 |
| `src/storage/shadow.ts` | — | 1-25, 49-75, 65, 67-75, 78-96, 100-103 |
| `CONFIGURATION.md` | — | 200, 204, 206, 2076, 2082, 2555, 2568 |

# 附录 B：本次未能验证的项（诚实清单）

| 项 | 等级 | 原因 | 若要验证需什么 |
|---|---|---|---|
| "832 个 open issue" 的数量 | **UNVERIFIED** | 离线无 issue 列表；实时 API 返回 844（含 PR） | GitHub API 分页拉取 + 去重 PR |
| 各 issue 的当前 open/closed 状态 | **UNVERIFIED** | 同上 | 逐条 API 查询 |
| schema 各表的**真实行数据分布**（是否有 `kind="message"`、`ts IS NULL` 占比） | **UNVERIFIED** | 本次未打开任何实际 `index.db` | 用户机器上的实库 |
| `content.bytes` 里是否真的全是 UTF-8 JSON | **STATIC_INFERENCE** | 从 `getContent` 的 `Buffer.toString("utf8")` 推断 | 实库抽样 |
| WAL 只读连接在本机环境的确切行为（`-shm` 可写性） | **STATIC_INFERENCE** | 未做实验 | 实机实验 |
| `#4182` 的双同字节消息缺口是否仍在线上触发 | **DOCUMENT_ONLY** | 只有代码注释自述 | 构造语料复现 |
| `6-OVERALL-DESIGN.md` 的原始 schema 设计 | **不可得** | 文件不在仓库（`schema.ts:3` 引用但不存在） | 上游历史分支或 issue #2671 |
| `preCompactionArchive` 是否有隐藏上限 | **STATIC_INFERENCE** | 未找到裁剪代码，但不能排除写入方限制 | 长跑会话实测 |
| MySQL backend 与本报告的差异 | **不适用** | CMB 只关心 sqlite；`mysql.ts` 未审 | — |
| 端到端"上游写入 → CMB 读取"的实机验证 | **未做** | 只读审查，未运行上游 | 沙箱内跑 `bili` + 造数据 |

---

## 审查声明

- **只读**：本 Agent 未对 `E:\[PROJECT]\refs\billion-context` 做任何写操作；`git status --porcelain` 在审查前后均为空。
- 未构建、未运行上游测试、未启动 `bili`、未生成任何上游产物。
- 所有行号基于 HEAD `0aace2f203b2da9065fd53c63f4be0c0ab03c90f`；若上游前进，行号会漂移，请以 SHA-256 指纹 + 符号名定位。
- 本报告区分了"代码事实"（VERIFIED_SOURCE）与"推断"（STATIC_INFERENCE），未把推断写成事实。B 段对主代理结论的纠正是基于逐行复核，**不是**基于记忆或印象。
