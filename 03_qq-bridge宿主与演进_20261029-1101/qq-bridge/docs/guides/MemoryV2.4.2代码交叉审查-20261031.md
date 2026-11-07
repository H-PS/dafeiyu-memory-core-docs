# Memory V2.4.2 第一批代码交叉审查报告

- 审查时间：2026-10-31 13:25
- 审查性质：**只读静态分析。** 未修改代码/配置/数据，未启动服务，未写入测试数据。
- 结论摘要：**V2.4.1 设计的 9 个模块在仓库中全部不存在。8 项审查中 5 项为 NOT_VERIFIABLE，3 项为 PARTIAL（仅有 V1 雏形）。**

---

# 一、基线信息

## 1.1 Git 基线

| 项 | 结果 |
|---|---|
| 仓库路径 | `D:\qq-bridge` |
| 是否 git 仓库 | ❌ **否**（`fatal: not a git repository`，无 `.git` 目录） |
| 当前分支 | **无法提供**（无 git） |
| HEAD commit hash | **无法提供**（无 git） |
| 工作区是否干净 | **无法判定**（无版本控制） |
| 未提交修改 | **无法判定** |
| 未跟踪文件 | **无法判定** |

**说明**：本项目无版本控制，因此无法区分"HEAD 已提交代码 / 未提交修改 / 未跟踪文件"。全部代码按"当前工作区唯一版本"对待。历史对比只能依赖 `D:\qq-bridge备份-20261030\*.zip`（37 个手工备份）。

## 1.2 Memory schema 版本

| 项 | 结果 |
|---|---|
| 设计文档中的 schemaVersion | V2.4.1 定为 **5** |
| 代码中 `schemaVersion` 字段 | **0 次出现**（不存在） |
| 实际数据字段 | `key, text, time, vec, category, subjectId, subjectName, subjectType, sourceType, visibility, needs_review, (share_review_status, share_requested_at, share_reviewed_at)` |

**实际 schema = V1（未版本化）。**

## 1.3 V2.4.1 模块存在性核查（关键基线）

| 设计的模块/文件 | 仓库中 | 证据 |
|---|---|---|
| `src/memory-wal.js` | ❌ 不存在 | `memory-wal` 在代码中 0 次 |
| `src/writer-lock.js` | ❌ 不存在 | `writer.lock` 0 次（注：另有 `bridge.lock`，见第 7 项） |
| `src/normalize.js` / `crc.js` | ❌ 不存在 | `_crc` 0 次，`crc32` 0 次 |
| `state/memory-tombstones.jsonl` | ❌ 不存在 | `tombstone` 0 次 |
| `state/share-requests.jsonl` | ❌ 不存在 | `ShareRequest` 0 次 |
| `state/fact-keys.json` | ❌ 不存在 | `factKey` 0 次 |
| `state/memory-wal/` | ❌ 不存在 | `state/` 下无 `wal` 目录 |
| `state/memory-audit.jsonl` | ❌ 不存在 | 只有 `memory-admin.log` |

## 1.4 关键符号计数（bridge.js + memory-vector.js）

| 符号 | 出现次数 | 结论 |
|---|---|---|
| `_crc` / `crc32` | 0 / 0 | 无行校验 |
| `embeddingStatus` | 0 | 无向量状态机 |
| `memory-wal` | 0 | 无 WAL |
| `tombstone` | 0 | 无删除墓碑 |
| `ShareRequest` / `share-requests` | 0 / 0 | 无独立分享对象 |
| `writer.lock` / `writerLock` | 0 / 0 | 无设计中的锁（有另一个 bridge.lock） |
| `effects` / `expectedRevision` | 0 / 0 | 无 effects、无版本约束 |
| `factKey` | 0 | 无事实键 |
| `revision` | 0 | 无乐观锁 |
| `lifecycleEvents` | 0 | 无生命周期历史 |
| `vecPurged` / `contentPurged` | 0 / 0 | 无清除标记 |
| `canonical` | 34（但指 `canonicalV2Key`，与 CRC 无关） | — |

## 1.5 Memory 相关代码主要入口

| 文件 | 行数 | 角色 |
|---|---|---|
| `src/memory-vector.js` | 381 | A 层全部实现（存储、embed、检索、CRUD） |
| `src/bridge.js` | ~7,180 | HTTP 端点（memory-append/remove/clear/vector/share-request/admin）+ `appendMemoryV2` + B 层状态 |
| `src/mcp-snowluma-safe.js` | ~1,050+ | MCP 工具层（`qq_memory_*`） |

## 1.6 数据文件当前状态

| 文件 | 大小 | 状态 |
|---|---|---|
| `state/memory-vectors.jsonl` | **0 字节** | 空（2026-10-31 05:18 清空） |
| `state/social-v2.json` | 101,136 字节 | 2 个会话，记忆字段全空 |
| `state/memory-admin.log` | 333 字节 | **2 条记录** |
| `state/bridge.lock` | 5 字节（内容 `12412`） | **僵尸锁**（PID 12412 已不存在，端口 3100 已停） |
| `state/memory-vectors.jsonl.pre-p0a.bak` | 2,162,534（101 条） | 手工备份 |
| `state/memory-vectors.jsonl.pre-vis.bak` | 2,076,154（101 条） | 手工备份 |

---

# 二、8 项审查结果汇总

| # | 项 | 状态 | 一句话结论 |
|---|---|---|---|
| 1 | CRC 递归规范化与完整性校验 | **NOT_VERIFIABLE** | 无 CRC 实现；现有坏行处理是静默跳过（P1 缺陷） |
| 2 | embeddingStatus 状态管理 | **NOT_VERIFIABLE** | 无状态机；embed 失败 = 抛错，B 已写 A 无（P1 缺陷） |
| 3 | WAL effects 与 revision fencing | **NOT_VERIFIABLE** | 无 WAL、无 revision；部分成功无法恢复（P0 缺口） |
| 4 | tombstone 与检索并发屏障 | **NOT_VERIFIABLE** | 无 tombstone；删除为物理删除 + `cache.mtime=0` |
| 5 | ShareRequest 与孤儿副本 | **NOT_VERIFIABLE** | 无 ShareRequest；现有 approveShare 只改字段不改 key |
| 6 | L2 删除与 B 层关联 | **PARTIAL** | 有 A/B 同步删除（P0 修复），但是物理删除 + text 匹配 |
| 7 | writer.lock 生命周期 | **PARTIAL** | 有 `bridge.lock`（实现较完善），缺 instanceId/心跳 |
| 8 | 特权操作审计 | **PARTIAL** | 有 `memory-admin.log`，但**审计失败会返回 500 而业务已执行**（P1 缺陷） |

---

# 三、逐项详细报告

## 第 1 项：CRC 递归规范化与完整性校验

**状态：NOT_VERIFIABLE（设计未实施）**

### 涉及文件和函数

| 文件 | 函数 | 行号 |
|---|---|---|
| `src/memory-vector.js` | `loadRows` | 44-58 |
| `src/memory-vector.js` | `loadRowsStrict` | 68-83 |
| `src/memory-vector.js` | `atomicRewrite` | 376-380 |

### 代码证据

```js
// memory-vector.js:44-58  宽松读
function loadRows() {
  try {
    const mtime = fs.existsSync(VEC_FILE) ? fs.statSync(VEC_FILE).mtimeMs : 0;
    if (mtime === cache.mtime) return cache.rows;
    if (!mtime) { cache = { mtime: 0, rows: [] }; return []; }
    const rows = [];
    const lines = fs.readFileSync(VEC_FILE, 'utf8').split('\n');
    for (const line of lines) {
      if (!line.trim()) continue;
      try { rows.push(JSON.parse(line)); } catch {}   // ← 坏行静默跳过，无日志、无计数
    }
    cache = { mtime, rows };
    return rows;
  } catch { return []; }                              // ← 读取失败也吞成 []
}
```

```js
// memory-vector.js:376-380  写入
function atomicRewrite(rows) {
  const tmp = VEC_FILE + '.tmp';                      // ← 固定名，无 pid/随机
  fs.writeFileSync(tmp, rows.map((r) => JSON.stringify(r)).join('\n') + '\n', 'utf8');
  fs.renameSync(tmp, VEC_FILE);                       // ← 无 fsync
}
```

### 实际执行流程

```
写入：appendVectorMemory → infoOK → embed → dailyBackup → fs.appendFileSync
      （append 路径不用 atomicRewrite；remove/edit 用 atomicRewrite）
      
读取：searchVectorMemory → loadRows（宽松）→ 坏行静默丢弃
     removeNAll/removeAll → loadRowsStrict（严格）→ 坏行抛错
```

### 发现的问题

| # | 问题 | 严重度 |
|---|---|---|
| 1-1 | **无任何行校验**（无 `_crc`），无法检测"内容被改但结构合法" | P1 |
| 1-2 | `loadRows` 坏行静默跳过（`:53 catch {}`）→ 检索少结果**无感知、无日志** | P1 |
| 1-3 | `loadRows` 读取失败也返回 `[]`（`:57`）→ 与"文件为空"无法区分 | P2 |
| 1-4 | `atomicRewrite` **固定 tmp 名**（`:377`）→ 并发写会互相覆盖 | **P0** |
| 1-5 | `atomicRewrite` 无 `fsync` → 断电可能丢数据 | P2 |
| 1-6 | `appendFileSync` 前不检查尾行是否以 `\n` 结束 → 可能多行粘连 | P2 |

### 触发条件

- 1-2：JSONL 任一行损坏（磁盘错、手编、写入中断）
- 1-4：两个写操作（remove/clear/edit/approve）在毫秒级并发

### 影响范围

- 1-2：检索结果悄悄变少，用户/模型都不知道
- 1-4：可能丢记录或写错内容（最严重）

### 是否需要运行时验证

- 1-4 需要并发测试（本任务禁止）。但**静态即可判定**：`VEC_FILE + '.tmp'` 是固定字符串，两个进程/协程同时写必然冲突。

### 建议修复方向

1. `_crc` 用**递归 canonical 序列化**（每层 key 排序，数组保序），排除最外层 `_crc`
2. `atomicRewrite` 改 `${VEC_FILE}.${pid}.${random6}.tmp` + `fsync` + `renameWithRetry`（可直接复用 bridge.js:182 的 `atomicWriteJson`）
3. `loadRows` 坏行改为"跳过 + 写 `memory-corrupt.log` + 计数告警"
4. append 前检查尾字节是否为 `\n`

---

## 第 2 项：embeddingStatus 状态管理

**状态：NOT_VERIFIABLE（设计未实施）**

### 涉及文件和函数

| 文件 | 函数 | 行号 |
|---|---|---|
| `src/memory-vector.js` | `embed` | 86-108 |
| `src/memory-vector.js` | `appendVectorMemory` | 149-163 |
| `src/memory-vector.js` | `searchVectorMemory` | 171-199 |
| `src/bridge.js` | memory-append 端点 | 6628-6694 |

### 代码证据

```js
// memory-vector.js:149-163
export async function appendVectorMemory(key, text, time) {
  const clean = String(text ?? '').trim().slice(0, 1000);
  if (!clean) throw new Error('记忆内容为空');
  if (!infoOK(clean)) throw new Error('信息量不足（太短或无实义），不入库');
  const vec = await embed(clean);          // ← 失败直接 throw，无状态标记
  dailyBackup();
  const row = {
    key, text: clean, time: time ?? Date.now(), vec,
    share_review_status: 'not_requested',
  };
  fs.mkdirSync(path.dirname(VEC_FILE), { recursive: true });
  fs.appendFileSync(VEC_FILE, JSON.stringify(row) + '\n', 'utf8');
  cache.mtime = 0;
  return { text: clean, time: row.time };
}
```

```js
// bridge.js:6657-6662  append 端点的 A 层调用
try {
  await appendVectorMemory(key, `${category}: ${content}`);
} catch (error) {
  aSynced = false;
  aError = error?.message ?? String(error);   // ← 只记在变量里，返回 aSynced:false
}
```

### 实际执行流程

```
qq_memory_append
  → bridge append 端点
    → ① 快照 B 三字段
    → ② appendMemoryV2(st, ...)           B 层内存已改
    → ③ await appendVectorMemory(...)      A 层
         └─ embed() 失败 → throw
            → catch: aSynced=false
            （记录未写入 A 层，但 B 层已改）
    → ④ saveSocialV2StateStrict()          B 层落盘
    → ⑤ partial=true，返回 ok:false + aSynced:false
```

### 发现的问题

| # | 问题 | 严重度 |
|---|---|---|
| 2-1 | **无 `embeddingStatus` 字段** → 无法表达"正文已存但向量待补" | P1 |
| 2-2 | embed 失败时**记录完全不写入 A 层**（连正文都丢）→ 只有 B 层有 | P1 |
| 2-3 | 无重试机制 → 那次 append 的 A 层内容**永久丢失**（除非模型重提） | P1 |
| 2-4 | 无 `vecModel` 字段 → 换模型后无法识别旧向量需重算 | P2 |
| 2-5 | 检索只看 `Array.isArray(r.vec)`（`:172`）→ 不检查维度是否 1024 | P2 |

### 触发条件

- Ollama 未启动 / 超时（30s）/ `bge-m3` 未 pull

### 影响范围

- 记忆"写了一半"：B 层有（会话活跃态），A 层无（跨会话检索不到）
- 用户感知：当前会话能看到，换个会话就没了

### 是否需要运行时验证

不需要。`appendVectorMemory` 的 `await embed()` 在 `fs.appendFileSync` 之前，静态可确认"embed 失败 = 记录不落盘"。

### 建议修复方向

1. 拆成两阶段：**先写正文 + `embeddingStatus='pending'`**，再异步补向量 → 正文不丢
2. `embeddingStatus: pending | ready | failed | purged`，检索只取 `ready`
3. `ready` 要求 `vec.length===1024` 且 `vecModel` 匹配
4. 补 `vecModel` 字段

---

## 第 3 项：WAL effects 与 revision fencing

**状态：NOT_VERIFIABLE（设计未实施）**

### 涉及文件和函数

| 文件 | 函数 | 行号 |
|---|---|---|
| `src/bridge.js` | memory-append 端点 | 6628-6694 |
| `src/bridge.js` | memory-remove 端点 | 6903-6986 |
| `src/bridge.js` | memory-clear 端点 | 7014-7123 |

### 代码证据

```
grep 'effects' → 0 次
grep 'expectedRevision' → 0 次
grep 'memory-wal' → 0 次
grep 'revision' → 0 次
```

现有的"部分成功"表达仅靠内存变量：

```js
// bridge.js:6655-6682（append 端点）
let aSynced = true, aError = '';
try { await appendVectorMemory(...); } catch { aSynced = false; aError = ...; }
let bSynced = true, bError = '', rollbackFailed = false;
try { saveSocialV2StateStrict(); } catch { ... 回滚 B 快照 ... }
const partial = !aSynced || !bSynced;
if (partial) { sendJson({ ok:false, aSynced, bSynced, partial, ... }, 500); return; }
```

### 实际执行流程（一条完整写入，静态追踪）

```
① MCP qq_memory_append
② POST /api/socialV2/memory-append（带 x-agent-token）
③ bridge:6637-6639 三重鉴权
④ bridge:6640 getSocialV2State(key)
⑤ bridge:6643-6651 快照 B 三字段（内存）
⑥ bridge:6653 appendMemoryV2（改 B 内存，含 24h 清理/超长截断/去重）
⑦ bridge:6657 await appendVectorMemory（A 层，含 embed）
⑧ bridge:6668 saveSocialV2StateStrict（B 落盘，失败回滚）
⑨ bridge:6683-6693 汇总返回
   ★ 全程无 WAL 记录、无 revision、无 expectedStatus
```

### 发现的问题

| # | 问题 | 严重度 |
|---|---|---|
| 3-1 | **无 WAL** → 进程在 ⑦ 与 ⑧ 之间崩溃，A 已写 B 未写，**重启后无人知道** | **P0** |
| 3-2 | 无 `revision` → 无法检测"旧操作覆盖新状态" | P0 |
| 3-3 | 无 `effects` → 无法表达"3 个资源中 2 个成功" | P1 |
| 3-4 | 部分成功只返回给调用方一次，**不持久化、不重试** | P0 |
| 3-5 | `saveSocialV2StateStrict` 失败时回滚是"内存回滚"，磁盘可能已写 | P1 |

### 触发条件

- 进程崩溃（kill、断电）在任何 await 点
- 磁盘满导致 ⑧ 失败

### 影响范围

- A/B 永久不一致，无自动发现机制
- 只有人手工对比两个文件才能发现

### 是否需要运行时验证

不需要。静态可确认**代码里没有任何持久化的操作记录**，崩溃后无恢复依据。

### 建议修复方向

按 V2.4.1 §7 设计：`effects[]` + `expectedRevision` + generation/MANIFEST。**这是 P0 级缺口，应最先实施（设计中的 P2 批次）。**

---

## 第 4 项：tombstone 与检索并发屏障

**状态：NOT_VERIFIABLE（设计未实施）**

### 涉及文件和函数

| 文件 | 函数 | 行号 |
|---|---|---|
| `src/memory-vector.js` | `removeVectorMemoryExact` | 311-319 |
| `src/memory-vector.js` | `removeNAllVectorMemory` | 324-344 |
| `src/memory-vector.js` | `removeAllVectorMemory` | 349-358 |
| `src/memory-vector.js` | `searchVectorMemory` | 171-199 |
| `src/memory-vector.js` | `loadRows` 缓存 | 44-58（`cache.mtime`） |

### 代码证据

```js
// memory-vector.js:311-319  物理删除（splice + 重写整个文件）
export function removeVectorMemoryExact(key, text) {
  const rows = loadRows();
  const idx = rows.findIndex((r) => r.key === key && r.text === String(text));
  if (idx < 0) return null;
  const [removed] = rows.splice(idx, 1);   // ← 物理删，无 tombstone
  cache.mtime = 0;
  atomicRewrite(rows);
  return removed;
}
```

```js
// memory-vector.js:44-48  缓存只依赖 mtime
const mtime = fs.existsSync(VEC_FILE) ? fs.statSync(VEC_FILE).mtimeMs : 0;
if (mtime === cache.mtime) return cache.rows;    // ← 只比 mtime
```

### 实际执行流程

```
删除：removeNAll/removeAll → loadRowsStrict → 生成 kept → atomicRewrite（rename）
      → cache.mtime = 0（下次读强制重载）

检索：searchVectorMemory → loadRows → 检查 mtime → 变了就重读，没变用缓存
```

### 发现的问题

| # | 问题 | 严重度 |
|---|---|---|
| 4-1 | 无 tombstone → 删除后无"永久拦截"依据（备份恢复/迁移重跑会复活） | P1 |
| 4-2 | 物理删除 → 无法审计、无法恢复（只能靠 `dailyBackup`，且只保 7 天） | P1 |
| 4-3 | 缓存失效只靠 `mtime`（`:47`）→ 同毫秒两次写入可能 mtime 相同 → 读到旧缓存 | P2 |
| 4-4 | `atomicRewrite` 后设 `cache.mtime=0`，但**并发请求可能已持有旧 rows 引用** | P2 |
| 4-5 | 删除后无"内存索引与文件"的发布顺序保证 | P1 |

### 触发条件

- 4-3：两次写入间隔 < 文件系统 mtime 精度（Windows NTFS 约 100ns，实际罕见但存在）
- 4-1：用户从 `backups/` 恢复快照

### 影响范围

- 已删除记忆可能通过备份恢复"复活"
- 无审计，无法追溯"谁删了什么"

### 是否需要运行时验证

4-3 需要构造同 mtime，本任务禁止。**其余静态可确认。**

### 建议修复方向

1. 引入独立 `memory-tombstones.jsonl`，删除时**先写 tombstone 再改 A 层**
2. 检索加 tombstone 检查（在权限之后、生命周期之前）
3. 缓存失效改 generation 计数（内存递增），不只靠 mtime

---

## 第 5 项：ShareRequest 与孤儿副本

**状态：NOT_VERIFIABLE（设计未实施）**

### 涉及文件和函数

| 文件 | 函数 | 行号 |
|---|---|---|
| `src/memory-vector.js` | `requestShare` | 261-274 |
| `src/memory-vector.js` | `approveShare` | 277-291 |
| `src/memory-vector.js` | `rejectShare` | 294-308 |
| `src/bridge.js` | memory-share-request 端点 | 6832-6870 |
| `src/bridge.js` | `/api/memory/admin` op=approve_share/reject_share | 6745-6758 |

### 代码证据

```js
// memory-vector.js:277-291  审批只改两个字段，不改 key、不建副本
export function approveShare(key, text, note) {
  const rows = loadRows();
  const idx = findMemoryRow(rows, key, text);
  if (idx < 0) return null;
  const r = rows[idx];
  const status = r.share_review_status || 'not_requested';
  if (status === 'approved') return { status: 'already_approved', text: r.text };
  r.share_review_status = 'approved';
  r.visibility = 'group_shared';       // ← 只改 visibility
  r.share_reviewed_at = Date.now();
  if (note) r.share_review_note = String(note).slice(0, 200);
  cache.mtime = 0;
  atomicRewrite(rows);
  return { status: 'approved', text: r.text, visibility: r.visibility };
}
```

### 实际执行流程（完整调用链追踪）

```
【创建申请】
① 群聊 MCP qq_memory_share_request(key='group:X', token, text)
② bridge:6832 POST /api/socialV2/memory-share-request
③ bridge:6837-6839 无条件三重鉴权（!token || !agentTokenOk）
④ bridge:6841 private key → 拒绝（400）
⑤ requestShare(key, text) → findMemoryRow 4 级匹配 → 标记 pending
   ★ 不保存 sourceRevision、contentHash
   ★ 不校验原件 status（因为 V1 无 status 字段）

【审批】
⑥ 控制台 POST /api/memory/admin {op:'approve_share', key, text}
⑦ approveShare → 只改 share_review_status + visibility
   ★ 不重建核原件（可能已改）
   ★ 不创建副本
   ★ 不校验 revision（无此概念）

【检索】
⑧ memory-vector 端点 → searchVectorMemory
   - key 全等过滤（:172）
   - private → 全部可见；group → visibility=group_shared && status=approved

【删除】
⑨ 删除时**不扫描关联分享**（V1 无副本概念，所以"关联"不存在）
```

### 发现的问题

| # | 问题 | 严重度 |
|---|---|---|
| 5-1 | **无 ShareRequest 对象** → 无法表达"群 A 批准、群 B 拒绝" | P1 |
| 5-2 | 审批时不重建核原件（可能已被编辑/删除） | P1 |
| 5-3 | **无 revision/contentHash 校验** → 旧授权可用于分享新版本 | **P0** |
| 5-4 | approveShare 只改 `visibility` 不改 `key` → **群检索永远搜不到**（key 是全等过滤） | **P0**（P0 审计已确认的设计死锁） |
| 5-5 | 无跨群隔离：`group_shared` 记录理论上可被任意群检索到（如果 key 恰好匹配） | P1 |
| 5-6 | 无目标群验证（模型传的 key 就是目标群，无白名单校验） | P1 |

### 触发条件

- 5-4：任何人审批任何共享 → 群聊永远搜不到（当前 0 条共享记忆，所以未暴露）
- 5-3：原件编辑后，旧 pending 申请仍有效

### 影响范围

- 分享功能**实质不可用**（5-4）
- 分享内容与原件版本脱节（5-3）

### 是否需要运行时验证

5-4 **静态可证**：`searchVectorMemory:172` 是 `r.key === key` 全等；`approveShare` 不改 `key`；群请求的 key 是 `group:X`，原件的 key 是 `private:A` → 永不匹配。

### 建议修复方向

按 V2.4.1 §12：独立 `ShareRequest` 对象 + 副本机制（新 memoryId，ownerId=group）+ 主人授权链 + revision 校验。

---

## 第 6 项：L2 删除与 B 层关联

**状态：PARTIAL**

**说明**：P0 修复时实现了 A/B 同步删除，但**不是 L2（逻辑删除）**，而是"物理删除 A + 过滤 B"。B 层关联靠 **text 匹配**，不是 slotId。

### 涉及文件和函数

| 文件 | 函数 | 行号 |
|---|---|---|
| `src/bridge.js` | memory-remove 端点 | 6903-6986 |
| `src/bridge.js` | memory-clear 端点 | 7014-7123 |
| `src/bridge.js` | `appendMemoryV2` | 9205-9273 |

### 代码证据

```js
// bridge.js:6918-6930  按 text 全等匹配（不是 slotId）
let bRemoved = 0;
if (category === 'activeTopic' && Array.isArray(st.activeTopics)) {
  bRemoved = st.activeTopics.filter((t) => String(t?.text ?? '') === content).length;
} else if (category === 'pendingThought' && Array.isArray(st.pendingThoughts)) {
  bRemoved = st.pendingThoughts.filter((t) => String(t?.text ?? '') === content).length;
} else if (category === 'memberImpression' && st.memberImpressions && typeof st.memberImpressions === 'object') {
  bRemoved = Object.prototype.hasOwnProperty.call(st.memberImpressions, target) ? 1 : 0;
}
```

```js
// bridge.js:6963-6986  先删 A 再删 B
try { aRemoved = removeNAllVectorMemory(key, `${category}: ${content}`, bRemoved); }
catch { aSynced = false; }
if (!aSynced) { sendJson({...aSynced:false, bSynced:'n/a'...}, 500); return; }   // A 失败则不动 B
// ... 快照 B ...
st.activeTopics = st.activeTopics.filter((t) => String(t?.text ?? '') !== content);   // 按 text 删
try { saveSocialV2StateStrict(); } catch { 回滚 B 快照 }
```

### 实际执行流程

```
remove(activeTopic, content='X')
① 计算 bRemoved = B 层 text==='X' 的条数
② bRemoved===0 → 404，不动任何层
③ memberImpression → 只删 B（A 层 text 对不上），返回 aSynced:'n/a'
④ removeNAllVectorMemory(key, `activeTopic: X`, bRemoved)   ← A 层，删 min(bRemoved, 匹配数) 条
⑤ A 失败 → 停止，不动 B
⑥ A 成功 → 快照 B → filter 删 B → saveSocialV2StateStrict → 失败回滚
```

### 发现的问题

| # | 问题 | 严重度 |
|---|---|---|
| 6-1 | **物理删除**（A 层 splice/rewrite）→ 无逻辑删除、无恢复、无审计 | P1 |
| 6-2 | B 层关联靠 **text 全等**（`:6919`）→ text 改过就失联 | P1 |
| 6-3 | 无 slotId → 同 text 多条时无法指定删哪条 | P1 |
| 6-4 | **memberImpression 的 A 层残留**（A 层 text=`memberImpression: <content>`，删除只有 target） | P1（已知，阶段 7B 明确保留） |
| 6-5 | 无 L2 内容清除（`contentPurged`）→ 删除就是物理移除 | P1 |
| 6-6 | 删除不碰 `share_review_status` 相关字段（无副本概念） | P2 |

### 触发条件

- 6-2：先用 `memory-update` 改 B 层 text，再 remove 旧 text → A 层残留
- 6-4：任何 memberImpression 删除

### 影响范围

- 删不干净的残留（A 层有、B 层无）会被检索到
- 无法审计"删了什么"

### 是否需要运行时验证

不需要。静态可确认 `filter((t) => String(t?.text ?? '') === content)` 是文本匹配。

### 建议修复方向

1. 引入 `status=deleted` + tombstone（不是物理删）
2. B 层加 `slotId`，A 层 `bRefs` 按 slotId 关联
3. L2 内容清除（`content='[deleted]'` + `vec=[]` + `vecPurged=true`）

### 已正确实现的部分（值得肯定）

```
✅ A 先于 B 的顺序（A 失败则不动 B）—— 避免"A 没删成但 B 删了"
✅ bRemoved 数量对齐（A 删 ≤ B 删）—— 避免 A 多删
✅ B 快照 + strict 落盘失败回滚
✅ memberImpression 诚实跳过 A 层（返回 aSynced:'n/a'，不伪装成功）
```

---

## 第 7 项：writer.lock 生命周期

**状态：PARTIAL**

**说明**：仓库中**已有** `bridge.lock` 单实例锁，实现比 V2.4.1 设计更细致（已考虑 PID 复用 + EPERM）。但缺 `instanceId`、缺心跳、无 maintenance 模式。

### 涉及文件和函数

| 文件 | 函数 | 行号 |
|---|---|---|
| `src/bridge.js` | `acquireLock` | 1080-1142 |
| `src/bridge.js` | `releaseLock` | 1143-1149 |
| `src/bridge.js` | `consolePortInUse` | 1064-1078 |
| `src/bridge.js` | `LOCK_FILE` 定义 | 1054 |

### 代码证据

```js
// bridge.js:1083-1091  原子创建 + 同时写 PID
const tryCreate = () => {
  try {
    fs.writeFileSync(LOCK_FILE, String(process.pid), { flag: 'wx', mode: 0o600 });
    return true;
  } catch (error) {
    if (error?.code === 'EEXIST') return false;
    throw error;
  }
};
```

```js
// bridge.js:1104-1128  PID 存活 + 端口双重探测（解决 Windows PID 复用）
try {
  process.kill(pid, 0);
  // 没抛异常只说明「存在某个持有该 PID 的进程」，不能证明那就是桥接
  // （PID 被回收给别的程序时会误判），所以再加一道端口探测。
  if (!consolePortInUse()) stale = true;
} catch (error) {
  if (error.code === 'ESRCH') { stale = true; }
  else if (error.code === 'EPERM') {
    if (consolePortInUse()) { console.error('...'); process.exit(2); }
    console.error('锁文件 PID ... EPERM 且端口空闲，按过期锁处理。');
    stale = true;
  } else { process.exit(2); }
}
```

```js
// bridge.js:1143-1149  释放时校验所有权
function releaseLock() {
  try {
    if (fs.existsSync(LOCK_FILE) && Number(fs.readFileSync(LOCK_FILE, 'utf8').trim()) === process.pid) {
      fs.unlinkSync(LOCK_FILE);
    }
  } catch {}
}
```

### 实际执行流程

```
启动：
① acquireLock()
② tryCreate() → wx 写 PID
   ├─ 成功 → 继续
   └─ EEXIST → 读锁
      ├─ 空内容 / 非正整数 → stale
      ├─ PID 存活 → 端口探测
      │              ├─ 端口占用 → exit(2)（判定双实例）
      │              └─ 端口空闲 → stale（PID 被复用）
      ├─ ESRCH → stale
      └─ EPERM → 端口探测：占用→exit(2)；空闲→stale
③ stale → unlink → tryCreate() 重试一次
④ 仍失败 → exit(2)

退出：
releaseLock() → 校验内容===自己 PID → unlink
```

### 发现的问题

| # | 问题 | 严重度 |
|---|---|---|
| 7-1 | **无 `instanceId`** → 无法区分"同 PID 不同进程"（虽然有端口探测兜底） | P2 |
| 7-2 | **无心跳** → 长时间 compact/写大文件时若被 kill -9，锁残留且 PID 可能存活 | P2 |
| 7-3 | `consolePortInUse` 探测失败时**保守返回 true**（`:1076`）→ 会导致"永远起不来"，需人工删锁 | P1 |
| 7-4 | **无 maintenance 模式** → 运维脚本与 bridge 无互斥机制 | P1 |
| 7-5 | 无"排空队列后释放"逻辑（`releaseLock` 直接 unlink） | P2 |
| 7-6 | 当前 `state/bridge.lock` 内容 `12412`，**PID 已不存在**（残留锁） | 观察项 |
| 7-7 | 依赖 `netstat`（`:1069`）→ 无 netstat 环境探测必然失败 → 保守 true | P2 |

### 触发条件

- 7-3：`netstat` 不可用 / 超时 → 每次启动都判定"已有实例"，需人工删锁
- 7-4：手工跑迁移脚本时 bridge 同时启动 → 无互斥

### 影响范围

- 7-3：启动失败，需人工介入（用户体验差但不丢数据）
- 7-4：脚本与 bridge 并发写 → `atomicRewrite` 固定 tmp → **数据损坏**（与第 1 项 1-4 叠加）

### 当前残留锁的状态（实测）

```
state/bridge.lock 内容 = '12412'
tasklist /FI "PID eq 12412" → 无匹配（进程不存在）
端口 3100 → 已停
→ 判定：僵尸锁。按代码逻辑，下次启动时会检测到 PID 不存在 → stale → 自动删除 → 重试成功。
→ 不阻碍启动，但说明上次退出未走 releaseLock（可能被 kill 或异常退出）
```

### 是否需要运行时验证

- 7-1/7-2 需要构造场景（禁止）
- **7-6 已完成只读验证**（tasklist + 端口探测）

### 建议修复方向

1. 加 `instanceId`（进程启动时随机生成，写入锁文件）
2. 加 `heartbeatAt`（每 10s 更新，用于判断"进程卡住但存活"）
3. `consolePortInUse` 探测失败时改为**失败关闭 + 明确提示**，而不是保守 true
4. 加 `mode: 'bridge' | 'maintenance'`，脚本也走同一把锁
5. `releaseLock` 前先排空写队列

### 值得肯定的部分

```
✅ wx 原子创建 + 同时写 PID（避免"先建空文件再写"的窗口）
✅ PID 存活 + 端口双重探测（已解决 V2.4.1 设计里提到的 Windows PID 复用问题）
✅ EPERM 特殊处理（避免受保护 PID 导致永久无法启动）
✅ 空内容视为过期
✅ releaseLock 校验所有权（不会误删别人的锁）
```

---

## 第 8 项：特权操作审计

**状态：PARTIAL（有明确缺陷）**

### 涉及文件和函数

| 文件 | 位置 | 行号 |
|---|---|---|
| `src/bridge.js` | REMOVE 审计 | 6733-6734 |
| `src/bridge.js` | EDIT 审计 | 6740-6741 |
| `src/bridge.js` | APPROVE/REJECT_SHARE 审计 | 6754-6755 |
| `state/memory-admin.log` | 审计文件 | 333 字节，2 条 |

### 代码证据

```js
// bridge.js:6730-6736  REMOVE：先执行，后写审计，无 try/catch
if (op === 'remove') {
  const removed = removeVectorMemoryExact(String(body.key ?? ''), String(body.text ?? ''));
  if (!removed) { sendJson({ ok: false, error: '未找到...' }, 404); return; }
  fs.appendFileSync(path.join(STATE_DIR, 'memory-admin.log'),
    `${new Date().toISOString()} REMOVE ${removed.key} | ${removed.text.slice(0, 80)}\n`);
  sendJson({ ok: true, removed: { text: removed.text, time: removed.time } });
  return;
}
```

```js
// bridge.js:6745-6758  APPROVE/REJECT
if (op === 'approve_share' || op === 'reject_share') {
  ...
  const r = op === 'approve_share' ? approveShare(key, text, note) : rejectShare(key, text, note);
  if (!r) { sendJson({ ok: false, error: '未找到...' }, 404); return; }
  fs.appendFileSync(path.join(STATE_DIR, 'memory-admin.log'),
    `${new Date().toISOString()} ${op.toUpperCase()} ${key} | ${r.text.slice(0, 80)} | note=${note || '-'}\n`);
  sendJson({ ok: true, ... });
}
```

### 实际审计记录（现存 2 条）

```
2026-10-30T18:30:51.762Z APPROVE_SHARE test:e2e-check | 10/31 02:30 大肥鱼说：端到端三级匹配验证用的临时记忆条目 | note=e2e
2026-10-30T19:07:02.535Z APPROVE_SHARE private:[OWNER_QQ] | [真实姓名]（[OWNER_QQ]），电脑的主人。嫌叫全名伤人，我定了叫"网管"。 | note=-
```

### 发现的问题

| # | 问题 | 严重度 |
|---|---|---|
| 8-1 | **审计写入在业务操作之后** → 业务成功、审计失败 → 无痕操作 | **P0** |
| 8-2 | **`appendFileSync` 无 try/catch** → 写日志失败（磁盘满/权限）会**抛错 → catch 到 500，但业务已执行** | **P0** |
| 8-3 | 无操作者标识（只有 consoleToken，未记录 who） | P1 |
| 8-4 | 无操作结果（成功/失败） | P1 |
| 8-5 | 无 opId / 无可追溯 ID | P1 |
| 8-6 | 只有 4 类操作有审计（remove/edit/approve/reject）；**`force_restore`、模板管理、归档都不存在，也就无审计** | P1（对应功能未实现） |
| 8-7 | 审计写的是 `text.slice(0,80)` → **审计日志本身含记忆正文**（隐私风险） | P2 |
| 8-8 | 无审计轮转/上限（`appendFileSync` 无限增长） | P2 |
| 8-9 | `:6759` 错误信息"未知 op，只支持 list/remove/edit"**过时**（实际支持 approve_share/reject_share） | P2 |

### 触发条件

- 8-2：磁盘满、`state/` 权限异常（如 ACL 被改）、杀软锁定文件

### 影响范围

- **最严重**：管理员删了一条记忆，审计日志写失败 → 记忆没了，日志没记，且用户看到的是 500 错误（以为没删成，实际已删）
- 这是"业务成功但返回失败"的典型，比"两者都失败"更危险

### 是否需要运行时验证

不需要。静态可确认：
1. `removeVectorMemoryExact()` 先执行（:6731）
2. `fs.appendFileSync`（:6733）在无 try/catch 的位置
3. 外层 `try { ... } catch (error) { sendJson({ok:false, error:'记忆管理失败'}, 500) }`（:6760-6762）

→ 如果 appendFileSync 抛错，catch 返回 500，但删除已完成。

### 建议修复方向

1. **审计改为 `required` effect 的前置条件**（V2.4.1 §7.3）
2. 或：审计失败 → 不提交业务（先写审计再执行，或两阶段）
3. 补齐字段：`operator / op / targetId / result / opId / timestamp`
4. 审计正文改为 `contentHash` 或前 20 字（降低隐私面）
5. 加轮转（按大小/日期）

### 关于"是否允许审计 optional"

**V2.4.1 设计说 audit 是 optional（失败不阻止 committed + auditWarn）**。但本项审查发现：
- 普通模型写入（append/remove/clear）→ 可以不要求审计
- **特权操作（approve/reject/remove by admin/force_restore）→ 必须要求审计**

建议修订 V2.4.1：`memory_audit` 的 requirement 应**按操作类型区分**：
```
普通模型操作 → optional
特权操作（consoleToken） → required
```

---

# 四、问题严重程度排序

## P0（必须优先修，影响数据正确性或安全性）

| # | 问题 | 位置 | 后果 |
|---|---|---|---|
| **P0-1** | `atomicRewrite` 固定 tmp 名，并发写互相覆盖 | `memory-vector.js:377` | 数据损坏/丢失 |
| **P0-2** | 无 WAL → 崩溃后 A/B 不一致无法发现和恢复 | 全局（无 memory-wal） | 数据永久不一致 |
| **P0-3** | 无 revision → 旧操作可覆盖新状态 | 全局（无 revision 字段） | 删除后复活 |
| **P0-4** | 审计在业务后 + 无 try/catch → 业务成功但返回 500 | `bridge.js:6731-6734, 6740-6741, 6754-6755` | 无痕删除，用户误判 |
| **P0-5** | `approveShare` 只改 visibility 不改 key → 群检索永远搜不到 | `memory-vector.js:285` + `:172` | 分享功能实质不可用 |
| **P0-6** | 分享审批无 revision/contentHash 校验 → 旧授权分享新版本 | `memory-vector.js:277-291` | 分享内容与版本脱节 |

## P1（影响功能正确性）

| # | 问题 | 位置 |
|---|---|---|
| P1-1 | `loadRows` 坏行静默跳过，检索少结果无感知 | `memory-vector.js:53` |
| P1-2 | embed 失败 → 记录完全不写 A 层，正文丢失 | `memory-vector.js:153` |
| P1-3 | 无 tombstone → 备份恢复会复活已删记忆 | 无 tombstone |
| P1-4 | B 层关联靠 text 匹配，改 text 后失联 | `bridge.js:6919` |
| P1-5 | memberImpression 的 A 层残留 | `bridge.js:6934-6957`（已知，已诚实标注） |
| P1-6 | `consolePortInUse` 探测失败保守 true → 可能永久无法启动 | `bridge.js:1076` |
| P1-7 | 无 maintenance 模式 → 脚本与 bridge 无互斥 | 无 writer.lock 模式 |
| P1-8 | 审计无操作者/结果/opId | `bridge.js:6733, 6740, 6754` |

## P2（改进项）

| # | 问题 | 位置 |
|---|---|---|
| P2-1 | 缓存失效只靠 mtime（同毫秒可能失效） | `memory-vector.js:47` |
| P2-2 | `atomicRewrite` 无 fsync | `memory-vector.js:378` |
| P2-3 | append 前不检查尾行 `\n`（多行粘连） | `memory-vector.js:160` |
| P2-4 | 检索不检查 vec 维度是否 1024 | `memory-vector.js:172` |
| P2-5 | 无 vecModel 字段 | 字段缺失 |
| P2-6 | 审计日志含记忆正文前 80 字（隐私） | `bridge.js:6734` |
| P2-7 | 审计无轮转 | `bridge.js` 3 处 |
| P2-8 | `:6759` 错误信息过时 | `bridge.js:6759` |
| P2-9 | 无 instanceId / 心跳 | `bridge.js:1085` |

---

# 五、设计文档与实际代码的不一致

| # | 设计说 | 代码实际 | 性质 |
|---|---|---|---|
| 1 | V2.4.1 有 `schemaVersion: 5` | 无 schemaVersion 字段 | **未实施** |
| 2 | V2.4.1 有 `_crc` 行校验 | 无 `_crc` | **未实施** |
| 3 | V2.4.1 有 `embeddingStatus` 四态 | 无此字段，embed 失败即抛错 | **未实施** |
| 4 | V2.4.1 有 `effects[]` + `expectedRevision` | 无 | **未实施** |
| 5 | V2.4.1 有 tombstone 三态 | 无 | **未实施** |
| 6 | V2.4.1 有 ShareRequest 独立对象 | 无（只有 `share_review_status` 字段） | **未实施** |
| 7 | V2.4.1 有 `writer.lock`（含 instanceId/心跳/维护模式） | 有 `bridge.lock`（无 instanceId/心跳/维护模式） | **部分不一致** |
| 8 | V2.4.1 有 `factKey` + 模板 | 无 | **未实施** |
| 9 | V2.4.1 有 `revision` 乐观锁 | 无 | **未实施** |
| 10 | V2.4.1 有 `lifecycleEvents` | 无 | **未实施** |
| 11 | V2.4.1 有 `vecPurged`/`contentPurged` | 无 | **未实施** |
| 12 | V2.4.1 审计是 optional | 代码里审计写失败会导致 500（相当于 required 但位置错） | **实现比设计更糟** |
| 13 | V2.4.1 的删除是"先 tombstone 后 A 层" | 代码是"直接物理删除" | **未实施** |
| 14 | V2.4.1 的分享是"创建副本" | 代码是"改 visibility"（且不改 key，功能失效） | **未实施且现有实现有缺陷** |

---

# 六、尚未检查或证据不足

| # | 项 | 原因 |
|---|---|---|
| 1 | 并发写损坏的实际概率 | 需并发测试（本任务禁止） |
| 2 | `consolePortInUse` 在无 netstat 环境的行为 | 需环境测试（禁止） |
| 3 | mtime 同毫秒碰撞的实际概率 | 需构造测试（禁止） |
| 4 | `src/mcp-snowluma-safe.js` 的全部记忆工具路径 | 本报告聚焦 bridge + memory-vector；MCP 层只抽查了 append/query/remove/clear/share_request 的转发逻辑 |
| 5 | `dailyBackup` 的实际触发频率 | 需运行观察（禁止）；静态确认只在 `appendVectorMemory:154` 调用 |
| 6 | `social-v2.json` 的写入是否有并发问题 | `atomicWriteJson` 用 pid+随机 tmp（安全），但未见写锁 |
| 7 | 是否有其他进程写 memory-vectors.jsonl | 全仓库搜无 `appendFileSync(VEC_FILE` 之外的第二处，但未排除外部脚本 |

---

# 七、建议后续修复顺序

## 立即（P0，不依赖 V2.4.1 完整实施）

```
1. 修 atomicRewrite 的 tmp 名（5 分钟，零风险）
   → memory-vector.js:377 改为 `${VEC_FILE}.${pid}.${rand}.tmp`
   → 复用 bridge.js:182 的 atomicWriteJson 模式

2. 修审计写入（业务后 → 业务前，或加 try/catch 不回滚业务）
   → bridge.js:6733/6740/6754
   → 最小改法：审计失败只报警，不抛错（不返回 500）
   → 更好改法：先写审计（含 result:'pending'），业务后再写一条 result
```

## 随 V2.4.1 P2（WAL）一起

```
3. 引入 WAL + revision（P0-2/P0-3）
4. 引入 tombstone（P1-3）
```

## 随 V2.4.1 P3（事实管理）

```
5. 引入 factKey / 两阶段更新
```

## 随 V2.4.1 P4（生命周期与分享）

```
6. 修分享机制（P0-5/P0-6）：ShareRequest + 副本
```

## 可独立做

```
7. loadRows 坏行加日志（P1-1）
8. embed 失败时先存正文（P1-2）
9. B 层加 slotId（P1-4）
10. consolePortInUse 探测失败改为失败关闭（P1-6）
```

---

# 八、审查结论

## 核心结论

```
V2.4.1 的设计在代码中完全未实施。
8 项审查对象中：
  · 5 项（1/2/3/4/5）无任何实现 → NOT_VERIFIABLE
  · 2 项（6/8）有 V1 雏形，但与设计不符且有明确缺陷 → PARTIAL
  · 1 项（7）有独立于设计的 bridge.lock，实现质量较好 → PARTIAL

因此本次审查的实际产出不是"验证设计实现正确性"，
而是"发现现有 V1 代码中的真实缺陷"，共 23 项（P0×6 / P1×8 / P2×9）。
```

## 最重要的三个发现

```
① atomicRewrite 固定 tmp 名（P0）
   与第 7 项"无 maintenance 互斥"叠加 → 脚本与 bridge 并发写会直接损坏数据

② 审计在业务后 + 无 try/catch（P0）
   管理员删记忆 → 日志写失败 → 返回 500 → 但记忆已删
   这是"业务成功但用户以为失败"的最坏组合

③ approveShare 只改 visibility 不改 key（P0）
   分享功能实质不可用（群检索 key 全等过滤永不匹配）
   这个在 P0 审计时已确认，至今未修
```

## 是否建议进入下一阶段

```
建议：
  ① 先修 2 个零风险 P0（tmp 名 + 审计），不需要等 V2.4.1
  ② V2.4.1 的实施仍按 P0→P1→P2 串行推进
  ③ 不建议在未修 tmp 名之前并行跑任何迁移/维护脚本
```

---

# 九、只读确认

```
✅ 本轮只读：
   - 读取源码：memory-vector.js（全文 381 行）、bridge.js（相关段落、
     行号级定位）、deploy/config 文件存在性
   - 读取数据：state/ 目录列表、bridge.lock 内容、memory-admin.log 内容、
     memory-vectors.jsonl 大小、两个 .bak 的条数统计
   - 执行只读命令：git 状态检查（确认无仓库）、tasklist（查 PID 12412）、
     端口探测（3100）、文件存在性/大小/MD5

✅ 未修改：任何代码、配置、数据、Persona、rules、preset
✅ 未写入测试记忆/用户/群聊数据
✅ 未执行迁移、清理、恢复、重建索引
✅ 未启动/停止/重启 QQ Bridge、DSH、Ollama
✅ 未安装/升级/卸载依赖
✅ 未自动修复任何发现的问题
✅ 只新增本报告一个文件
```

---

*报告完成，停止。等待人工审核后再决定下一阶段。*
