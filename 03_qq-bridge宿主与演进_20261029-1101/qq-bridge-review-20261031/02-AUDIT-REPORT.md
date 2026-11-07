# 02 · 审查报告（Memory V2.4.2 第二阶段）

- 审查对象：`D:\qq-bridge`（无 Git 仓库）
- 审查方式：纯静态代码分析 + 只读元数据核查
- 审查时间：2026-10-31 13:20 ~ 13:55
- 基线详见 `01-BASELINE.md`；哈希清单详见 `04-MANIFEST.md`

## 状态标准

| 状态 | 含义 |
|---|---|
| `NOT_IMPLEMENTED` | 已确认设计功能在代码中不存在 |
| `PASS` | 代码路径完整，未发现目标问题 |
| `FAIL` | 发现明确实现缺陷 |
| `PARTIAL` | 部分实现，存在缺口 |
| `NOT_VERIFIABLE` | 证据不足，无法确认 |

---

# 第一部分：对第一批报告 4 个疑点的补充核查

## 疑点 1：atomicRewrite 的真实并发风险

**结论：PARTIAL（存在真实并发风险，但与第一批报告的表述不同）**

### 全仓库写入 `memory-vectors.jsonl` 的路径（穷举）

| # | 位置 | 写入方式 | 触发方 |
|---|---|---|---|
| 1 | `memory-vector.js:160` `appendFileSync(VEC_FILE, ...)` | 追加 | `appendVectorMemory` |
| 2 | `memory-vector.js:378-379` `writeFileSync(tmp)` + `renameSync` | 原子重写 | `atomicRewrite`（7 个调用点） |
| 3 | `bridge.js:6716` `copyFileSync(VEC_FILE_PATH, ...)` | **只读源**（拷出去） | admin remove/edit 前备份 |

`atomicRewrite` 的 7 个调用点：

| 行号 | 所在函数 | 是否 async |
|---|---|---|
| 272 | `requestShare` | ❌ 同步 |
| 289 | `approveShare` | ❌ 同步 |
| 306 | `rejectShare` | ❌ 同步 |
| 317 | `removeVectorMemoryExact` | ❌ 同步 |
| 341 | `removeNAllVectorMemory` | ❌ 同步 |
| 355 | `removeAllVectorMemory` | ❌ 同步 |
| 372 | `editVectorMemory` | ✅ **async（含 `await embed`）** |

### 其他进程/脚本是否也可能写（全仓库搜索）

```
搜 appendFileSync(VEC_FILE / writeFileSync(tmp / renameSync(tmp / atomicRewrite(
→ 除 memory-vector.js 自身外，无第二个直接写 VEC_FILE 的模块
→ seed-group-owner.mjs（曾硬编码写 A 层）已于阶段 8 归档为 .disabled，不在运行路径
→ 无 maintenance / 迁移脚本调用 atomicRewrite
```

### 三个层次的风险分类

#### A. 已确认存在的多写者冲突：**同进程内的 async 丢失更新**

**这是真实缺陷，且与 `tmp` 固定名无关。**

```js
// memory-vector.js:361-374  editVectorMemory（唯一 async 的写函数）
export async function editVectorMemory(key, oldText, newText) {
  const rows = loadRows();                                  // ① 读快照
  const idx = rows.findIndex((r) => r.key === key && r.text === String(oldText));
  if (idx < 0) throw new Error('未找到...');
  const clean = String(newText ?? '').trim().slice(0, 1000);
  if (!clean) throw new Error('新文本为空');
  const vec = await embed(clean);                           // ② ★ await 让出控制权
  rows[idx].text = clean;                                   // ③ 改内存副本
  rows[idx].vec = vec;
  rows[idx].edited = true;
  cache.mtime = 0;
  atomicRewrite(rows);                                      // ④ 用旧 rows 覆盖整个文件
  return { text: clean };
}
```

**执行顺序推演（两个并发请求）**：

| 时刻 | 请求 A（edit 记录 X） | 请求 B（remove 记录 Y） | 磁盘状态 |
|---|---|---|---|
| t0 | `loadRows()` → rows=[X,Y,Z] | — | [X,Y,Z] |
| t1 | `await embed()` ← 让出 | `loadRows()` → rows=[X,Y,Z] | [X,Y,Z] |
| t2 | — | 删除 Y → `atomicRewrite([X,Z])` | [X,Z] |
| t3 | embed 返回 → 改 rows[0].text | — | [X,Z] |
| t4 | `atomicRewrite([X,Y,Z])` ← **把 Y 写回来了** | — | **[X,Y,Z]** ❌ |

**结果：B 已删除的 Y 复活。**

同理，两个 edit 并发会丢更新（后写的覆盖先写的）。

#### B. 具备冲突条件但尚未找到第二个写入者：**跨进程**

- 唯一可能的第二写者是维护/迁移脚本，当前**不存在**（seed 已归档）
- 但 `atomicRewrite` 的固定 `tmp` 名意味着：**一旦将来有人写脚本并绕过 bridge 直接调用，冲突概率几乎 100%**
- 当前无 `writer.lock` 与脚本互斥（`bridge.lock` 只保护 bridge 自身，脚本不参与）

#### C. 当前证据不支持多写者竞争：**跨进程并发写**

- 无证据表明当前有任何并发写发生在两个进程之间
- 因此**不能**断言"当前必然发生数据损坏"

### 关于 Windows 下 renameSync 的约束

- `fs.renameSync(tmp, VEC_FILE)` 在目标文件被其他句柄打开时可能 `EBUSY`/`EPERM`
- A 层**没有** `renameWithRetry`（bridge.js 有，见 :165-180）
- 因此 A 层 rename 失败 = 抛错 → 调用方收到 500 → 但内存已改（`cache.mtime=0`）→ 下次读会读磁盘旧值 → 数据"回滚"但调用方以为失败
- 这本身是"失败时行为不一致"，不是并发问题

### 结论与建议

| 项 | 判定 |
|---|---|
| 同进程 async 丢失更新 | **FAIL（P0）** — `editVectorMemory` 的 `await embed` 是根因 |
| 删除后被 edit 的旧快照复活 | **FAIL（P0）** — 同上，且是可复现的"删除复活"路径 |
| 固定 tmp 名 | **FAIL（P1）** — 当前无跨进程写者，但一旦有则必然冲突 |
| 跨进程当前是否损坏 | **证据不足** — 未发现第二写入者 |

**修复方向**：
1. `editVectorMemory` 改为"先写临时结果到独立文件路径"或"await 前加锁、await 后重新读校验 revision"
2. `atomicRewrite` 用 `${VEC_FILE}.${pid}.${rand6}.tmp` + `renameWithRetry`
3. 引入进程内写队列（Promise 链），所有写操作串行

---

## 疑点 2：approveShare 完整调用链

**结论：PASS（功能正常）；第一批报告的"永不匹配"结论不成立，予以撤回**

### 完整调用链（带行号）

```
【第一步：谁发起，传什么 key】
MCP src/mcp-snowluma-safe.js:957-980
  qq_memory_share_request({key, token, text})
  → key 由模型传入，等于当前会话的 key
     私聊场景：key = 'private:<ownerQQ>'
     群聊场景：key = 'group:<群号>'

【第二步：Bridge 的验证与转换】
src/bridge.js:6832  POST /api/socialV2/memory-share-request
src/bridge.js:6834  const key = String(body.key ?? '').trim();     ← 只 trim，不做前缀转换
src/bridge.js:6837-6839  无条件三重校验（!token || !agentTokenOk / v2SessionAllowed / v2ToolEnabled）
src/bridge.js:6841-6844  if (key.startsWith('private:')) → 400「私聊记忆不支持共享到群」
src/bridge.js:6845  const r = requestShare(key, text);             ← key 原样传入

【第三步：requestShare 如何定位】
src/memory-vector.js:261-274  requestShare(key, text)
  → findMemoryRow(rows, key, text)
src/memory-vector.js:235-258  findMemoryRow
  :238  rows.findIndex(r => r.key === key && r.text === t)          ← ★ key 必须完全相等
  :240  rows.findIndex(r => r.key === key && (r.text.includes(t) || t.includes(r.text)))
  :242-246  去掉分类前缀后包含（仍要求 r.key === key）
  :249-256  归一化后包含（仍要求 r.key === key）
  → 四级匹配全部以 r.key === key 为前提

【第四步：原始记录的 key】
appendVectorMemory 写入时 key 直接来自调用方（bridge memory-append 端点）
群内产生的记忆 → key = 'group:<群号>'（不变）
私聊产生的记忆 → key = 'private:<QQ>'（不变）

【第五步：控制台审批传什么 key】
src/bridge.js:6695-6708  POST /api/memory/admin {op:'approve_share', key, text, note}
  :6696-6699  key 和 text 必须非空
  → key 必须是当初申请时用的同一个 key（group:<群号>）

【第六步：approveShare 的查找与修改】
src/memory-vector.js:277-291  approveShare(key, text, note)
  :279  const idx = findMemoryRow(rows, key, text);                 ← 与第三步同一套匹配
  :284-287  r.share_review_status = 'approved';
            r.visibility = 'group_shared';
            r.share_reviewed_at = Date.now();
  ★ 不改 r.key

【第七步：群聊检索用什么 key】
src/bridge.js:6862  GET /api/socialV2/memory-vector?key=...&topic=...
src/bridge.js:6865-6869  三重校验
src/bridge.js:6872  searchVectorMemory(key, topic, 5)
src/memory-vector.js:171-199  searchVectorMemory
  :172  const rows = loadRows().filter((r) => r.key === key && Array.isArray(r.vec));
        ★ key = 'group:<群号>'，能匹配第四步写入的群记录
  :174  const isOwnerPrivate = key.startsWith('private:');
  :175-177  群聊分支 → rows.filter(r => r.visibility === 'group_shared' && r.share_review_status === 'approved')
        ★ 第六步已满足这两个条件

【第八步：结论】
群内产生 → key=group:X → 申请(pending) → 审批(visibility=group_shared, approved) → 群查(key=group:X)
  → 第 172 行 key 全等匹配 ✅
  → 第 177 行 visibility/status 过滤 ✅
  → 【本群能检索到】
```

### 关于 `private:X` 与 `group:X` 的关系

```js
// bridge.js:1370-1376
function canonicalV2Key(key) {
  const m = /^(group|private):(\d+)$/.exec(String(key ?? '').trim());
  if (!m) return null;
  const id = Number(m[2]);
  if (!Number.isSafeInteger(id) || id <= 0) return null;
  return `${m[1]}:${id}`;      // 只去掉前导零，不改前缀
}
```

- 规范化**不改变 `group`/`private` 前缀**，只去前导零
- 因此 `private:123` 与 `group:123` 永远是两个不同的 key
- 但 `approveShare` **不需要**把 private 改成 group —— 因为群记忆本来就是 group key

### 修正声明（重要）

> **撤回**第一批报告中的结论：
> 「`approveShare` 只改 `visibility` 不改 `key` → 群检索永远搜不到 → 分享功能实质不可用」
>
> **正确结论**：
> 1. **群内产生的记忆**（key=`group:X`）审批后，**本群能正常检索到** ✅
> 2. **私聊记忆分享到群**不可用 —— 但这是**已声明的设计边界**：`bridge.js:6841-6844` 对 private key 直接返回 400，根本不会进入审批流程
> 3. 真实缺口只有一处：`share_review_status` 是**单字段**，无法表达"群A批准、群B拒绝"（但 V1 也没有"一条记忆分享到多个群"的设计，故不构成当前缺陷）

### 触发条件 / 影响范围

| 场景 | 结果 |
|---|---|
| 群内记忆申请 → 审批 → 本群检索 | ✅ 正常 |
| 私聊记忆申请 | ❌ 400 拒绝（设计如此） |
| 一条记忆分享给多个群 | ❌ 不支持（无 targetGroup 维度） |

**风险等级**：无（不是缺陷）。**建议**：V2 引入 ShareRequest 时需要新增 `targetOwnerId` 维度以支持多群。

---

## 疑点 3：bridge.lock 启动竞争窗口

**结论：FAIL（P1 — 存在真实窗口，但端口监听兜底，不会真双实例）**

### 执行顺序（带行号）

```
bridge.js:1390   acquireLock();          ← 第一道防线：拿锁
bridge.js:1391   loadState();            ← 加载状态（可能读大文件）
bridge.js:1392-7530  大量函数定义（不执行，仅编译）
bridge.js:7531   server.listen(port, '127.0.0.1', cb);   ← 第二道防线：绑端口
bridge.js:7535-7542   server.on('error') → EADDRINUSE → process.exit(2)
bridge.js:12447-12477  SIGINT/SIGTERM/exit → releaseLock()
```

**关键：`acquireLock()` 与 `server.listen()` 之间隔 6141 行**。虽然中间多为函数定义（执行几乎无耗时），但仍有 `loadState()` 等实际调用。

### 竞争窗口推演

| 时刻 | 进程 A | 进程 B | 锁文件 | 端口 3100 |
|---|---|---|---|---|
| t0 | `acquireLock()` 成功，写入 PID_A | — | PID_A | 空闲 |
| t1 | 执行 `loadState()`（进行中） | `acquireLock()` → `tryCreate` **EEXIST** | PID_A | 空闲 |
| t2 | — | 读锁 → PID_A；`process.kill(PID_A, 0)` **成功**（A 活着） | PID_A | 空闲 |
| t3 | — | `consolePortInUse()` → **A 还没 listen → 返回 false** | PID_A | 空闲 |
| t4 | — | 判定 `stale = true` → **`unlinkSync(LOCK_FILE)`** → `tryCreate` 成功 | **PID_B** | 空闲 |
| t5 | `server.listen` **成功** | — | PID_B | **A 占用** |
| t6 | 正常运行 | `server.listen` → **EADDRINUSE** → `process.exit(2)` | PID_B | A 占用 |
| t7 | 正常运行 | `process.on('exit')` → `releaseLock()` → 锁内容 === PID_B → **`unlinkSync`** | **不存在** | A 占用 |

**最终状态**：进程 A 正在运行，但**锁文件已不存在**。

### 后续影响链

```
t8  进程 C 启动 → tryCreate 成功（锁不存在）→ 认为自己是唯一实例
t9  C 继续初始化 → server.listen → EADDRINUSE → exit(2) → releaseLock 删除 C 写的锁
→ 回到"无锁运行"，如此循环

结果：
· 不会出现双实例（端口是硬防线）
· 但第一道防线（锁）被破坏，每次启动都要靠端口兜底
· 若 A 恰好在 t5 之前崩溃（未 listen），B 已删锁并接管 —— 这时是"正确接管"
· 若 A 在 t5 后崩溃，端口释放，C 能正常启动 —— 也无害
```

### 为什么风险等级是 P1 而不是 P0

| 判定依据 |
|---|
| ✅ 存在真实窗口（静态可证） |
| ✅ 会导致锁文件被误删 |
| ❌ **不会双实例**：`server.on('error')` 的 EADDRINUSE 分支（:7535-7542）是硬防线 |
| ❌ 不会直接数据损坏（bridge 是唯一写入者，双实例被端口挡住） |

### 修复方向

1. **锁获取顺序调整**：先 `server.listen` 成功，再 `acquireLock()`（或拿到锁后立即 listen，不要在中间做长初始化）
2. 或：`consolePortInUse()` 返回 false 时**不立即判定 stale**，而是加"锁年龄"判断（如锁创建 < 10s 则等待重试，而不是接管）
3. 或：锁文件写入后加一个"启动完成"标记，未完成标记的锁不视为可接管

---

## 疑点 4：特权操作审计顺序

**结论：FAIL（P0 — 业务成功但返回失败；且 clear 完全无审计）**

### 逐个操作追踪

| 操作 | 业务执行行 | 审计写入行 | 顺序 | 审计失败后果 |
|---|---|---|---|---|
| `remove` | :6731 `removeVectorMemoryExact` | :6733-6734 | **业务→审计** | `appendFileSync` 抛错 → 外层 catch(:6760) → **返回 500，但已删** |
| `edit` | :6739 `editVectorMemory` | :6740-6741 | **业务→审计** | 同上，**已改但返回 500** |
| `approve_share` | :6750-6752 `approveShare` | :6754-6755 | **业务→审计** | 同上，**已批准但返回 500** |
| `reject_share` | :6750-6752 `rejectShare` | :6754-6755 | **业务→审计** | 同上 |
| `clear` | :7014-7123 | **无** | — | **完全无审计记录** |

### 代码证据（remove）

```js
// bridge.js:6730-6736
if (op === 'remove') {
  const removed = removeVectorMemoryExact(String(body.key ?? ''), String(body.text ?? ''));   // ← 业务已生效
  if (!removed) { sendJson({ ok: false, error: '未找到（key+text 需全匹配，不支持模糊）' }, 404); return; }
  fs.appendFileSync(path.join(STATE_DIR, 'memory-admin.log'),                                  // ← 无 try/catch
    `${new Date().toISOString()} REMOVE ${removed.key} | ${removed.text.slice(0, 80)}\n`);
  sendJson({ ok: true, removed: { text: removed.text, time: removed.time } });
  return;
}
```

```js
// bridge.js:6760-6762  外层 catch
} catch (error) {
  sendJson({ ok: false, error: `记忆管理失败：${error?.message ?? error}` }, 500);
}
```

**推演**：
1. `removeVectorMemoryExact` 成功（记录已从文件移除）
2. `fs.appendFileSync` 抛错（磁盘满 / ACL 变更 / 文件被占用）
3. 跳到 :6760 catch → 返回 `{ok:false, error:'记忆管理失败：...'}` **HTTP 500**
4. 调用方（控制台）看到"删除失败"
5. **实际：记录已删，日志未记**

### 是否可能"业务失败但审计显示成功"

- `approveShare` 返回 `null`（找不到）→ :6753 提前 return 404，**不写审计** → 一致 ✅
- `editVectorMemory` 抛错（找不到旧文本）→ 跳到 catch，**不写审计** → 一致 ✅
- **结论：不存在"业务失败但审计成功"的情况**（因为审计在业务之后）

### 是否可能"业务成功但调用方收到失败"

- **remove / edit / approve_share / reject_share 全部可能**（审计写失败）

### 操作者标识

```js
// bridge.js:6653-6660（/api/memory/admin 的鉴权）
const consoleTok = String(process.env.QQ_CONSOLE_TOKEN ?? '') || String(cfg.consoleToken ?? '');
const viaBearer = String(req.headers.authorization ?? '') === `Bearer ${consoleTok}`;
const viaConsoleHeader = String(req.headers['x-console-token'] ?? '') === consoleTok;
```

- **只有令牌校验，没有任何操作者身份标识**
- 审计日志里没有 who、没有 client IP、没有 session
- 无法回答"是谁删的"

### 审计字段完整性

| 应有字段 | 实际 |
|---|---|
| 时间 | ✅ ISO 时间戳 |
| 操作类型 | ✅ REMOVE / EDIT / APPROVE_SHARE |
| 目标 key | ✅ |
| 目标内容 | ⚠️ 前 80 字（**含记忆正文，隐私风险**） |
| 操作者 | ❌ 无 |
| 结果 | ❌ 无（只有成功才写，所以隐含成功） |
| opId / 可追溯 ID | ❌ 无 |

### 现有审计记录（2 条，脱敏后）

```
2026-10-30T18:30:51.762Z APPROVE_SHARE test:e2e-check | 10/31 02:30 [REDACTED_NAME]说：端到端三级匹配验证用的临时记忆条目 | note=e2e
2026-10-30T19:07:02.535Z APPROVE_SHARE private:[REDACTED_OWNER_QQ] | [REDACTED_NAME]（[REDACTED_OWNER_QQ]），电脑的主人。... | note=-
```

### 修复方向

1. **审计前置**：先写 `{result:'pending', op, key, by}`，业务完成后再写 `{result:'ok'}`（两次写入）
2. 或：审计失败**不抛错**（只报警），避免"业务成功返回 500"
3. `clear` 补审计（当前完全无记录）
4. 记录 `consoleToken` 的**指纹**（如 sha256 前 8 位）作为操作者标识，不记录明文
5. 审计正文改为 `contentHash` 或前 20 字

---

# 第二部分：新增审查项目

## 第 9 项：写入幂等性与并发操作

**状态：FAIL（P0 — 存在删除复活路径；无任何幂等机制）**

### 逐场景推演

#### 9.1 MCP 重复调用 append

| 步骤 | 结果 |
|---|---|
| 触发 | 模型连续两次 `qq_memory_append(key, token, 'activeTopic', '同一个内容')` |
| 执行顺序 | ① 第一次：B 层 `existing` 命中 → 只更新 `lastMentionAt`；A 层 `appendFileSync` **新增一条**<br>② 第二次：同样 |
| 最终 A 状态 | **2 条**（`activeTopic: 同一个内容` 完全重复） |
| 最终 B 状态 | **1 条**（文本去重生效） |
| 调用方结果 | 两次都 `ok:true` |

**根因**：`appendVectorMemory` 无查重（memory-vector.js:149-163，只做 `infoOK` 长度检查）。

#### 9.2 调用方超时重试

- MCP 的 `agentApi` 超时 15s（`mcp-snowluma-safe.js:155`）
- 若 bridge 在 15s 后才完成写入，MCP 已超时 → 模型可能重试 → **A 层重复写入 1 条**
- 无 opId / 无幂等键 → 无法识别"这是同一次操作的重试"

#### 9.3 A 成功、B 失败后重试

| 步骤 | 结果 |
|---|---|
| 首次 | A 写入成功（1 条）；B `saveSocialV2StateStrict` 失败 → 回滚 B 内存 → 返回 `aSynced:true, bSynced:false` |
| 重试（模型再调一次） | B 层重新 `appendMemoryV2` → `existing` 未命中（因为上次回滚了）→ 新增<br>A 层再 `appendFileSync` → **又 1 条** |
| 最终 A | **2 条重复** |
| 最终 B | 1 条 |

#### 9.4 B 成功、A 失败后重试

| 步骤 | 结果 |
|---|---|
| 首次 | A `embed` 失败 → 不写入（记录丢失）；B 落盘成功 → 返回 `aSynced:false, bSynced:true` |
| 重试 | B 层 `existing` 命中（同 text）→ 只更新时间戳（**不重复**）<br>A 层若 embed 成功 → 新增 1 条 |
| 最终 A | 1 条 |
| 最终 B | 1 条 |

**这是唯一"重试无害"的场景**（因为 B 层有文本去重）。

#### 9.5 ★ embed 未完成时删除 → **删除复活**（最严重）

```js
// memory-vector.js:361-374  editVectorMemory
const rows = loadRows();                      // ① 读全量快照
const idx = rows.findIndex(...);
const vec = await embed(clean);               // ② ★ await（可能 30s）
rows[idx].text = clean;                       // ③ 改内存副本
atomicRewrite(rows);                          // ④ 用旧快照覆盖文件
```

| 时刻 | edit 请求 | remove 请求 | 磁盘 A 层 |
|---|---|---|---|
| t0 | `loadRows()` → [X,Y,Z] | — | [X,Y,Z] |
| t1 | `await embed()` 挂起 | — | [X,Y,Z] |
| t2 | — | 删除 Y → `atomicRewrite([X,Z])` | **[X,Z]** |
| t3 | embed 返回 | — | [X,Z] |
| t4 | `rows[0].text = ...`（rows 仍是 [X,Y,Z]） | — | [X,Z] |
| t5 | `atomicRewrite([X,Y,Z])` | — | **[X,Y,Z] ← Y 复活** ❌ |

**触发条件**：`editVectorMemory` 的 embed 期间（最长 30s）有人执行 remove/clear。

**影响**：已删除的记忆重新出现，且检索可命中。

**这是本次审查发现的最严重缺陷**（比固定 tmp 名更实质，因为它不依赖并发，单线程即可复现）。

#### 9.6 两个 edit/remove/clear 并发

| 组合 | 结果 |
|---|---|
| 两个 `edit` | 后完成者覆盖前者（丢失更新）—— 因为两者都基于各自的旧快照 |
| 两个 `remove` | 后完成者的 `atomicRewrite(kept)` 基于旧快照 → 前者删的记录**复活** |
| `remove` + `clear` | 同 9.5，可能互相覆盖 |

**注**：`remove` / `clear` 本身是同步的（无 await），但 `editVectorMemory` 是 async。所以：
- 纯 remove 之间：Node 单线程，同步执行不交错 → **安全**
- remove 与 edit：edit 的 await 期间 remove 可插队 → **不安全**（9.5）

#### 9.7 绕过 Bridge 的文件写入脚本

```
全仓库搜索：除 memory-vector.js 外，无第二个写 VEC_FILE 的模块
seed-group-owner.mjs → 已归档为 .disabled（scripts/archive/）
→ 当前无绕过 Bridge 的写入脚本
```

#### 9.8 opId / 幂等键

```
grep 'opId' → 0 次
grep 'idempot' → 0 次
→ 完全无幂等机制
```

### 汇总表

| 场景 | 触发条件 | 最终 A | 最终 B | 调用方结果 | 等级 |
|---|---|---|---|---|---|
| 9.1 重复 append | 模型重复调用 | 2 条重复 | 1 条 | ok×2 | P1 |
| 9.2 超时重试 | bridge 慢于 15s | 2 条重复 | 1 条 | 超时+ok | P1 |
| 9.3 A成B败重试 | B 落盘失败 | 2 条重复 | 1 条 | partial+ok | P1 |
| 9.4 B成A败重试 | embed 失败 | 1 条 | 1 条 | partial+ok | ✅ 无害 |
| **9.5 embed 期间删除** | edit + remove 并发 | **已删记录复活** | 不变 | ok+ok | **P0** |
| 9.6 remove+edit | 同上 | 复活 | 不变 | ok+ok | **P0** |
| 9.7 脚本写入 | 无 | — | — | — | ✅ 无 |
| 9.8 幂等键 | 无 | — | — | — | FAIL |

### 修复方向

1. **`editVectorMemory` 改为"await 后重新读 + 校验 revision"**（而不是用 await 前的旧快照）
2. 或：`await embed` 前先加进程内写锁，整个函数持锁（阻止其他写插队）
3. `appendVectorMemory` 加 contentHash 去重
4. 引入 `opId` + WAL（V2 设计）

---

## 第 10 项：备份恢复与数据防复活

**状态：PARTIAL（有备份，无恢复；防复活完全缺失）**

### 所有创建/恢复 `memory-vectors.jsonl` 的路径

| 类型 | 位置 | 说明 |
|---|---|---|
| **创建/写入** | `memory-vector.js:160` `appendFileSync` | 追加 |
| **创建/写入** | `memory-vector.js:378-379` `atomicRewrite` | 全量重写 |
| **备份（拷出）** | `memory-vector.js:138` `fs.copyFileSync(VEC_FILE, dest)` | `dailyBackup` |
| **备份（拷出）** | `bridge.js:6716` `copyFileSync(VEC_FILE_PATH, ...admin-backup.jsonl)` | admin remove/edit 前 |
| **恢复（拷入）** | **不存在** | 全仓库搜索无实现 |

**结论：代码中没有任何"从备份恢复到 memory-vectors.jsonl"的函数。** 恢复只能人工 `copy` 文件。

### dailyBackup 实现细节

```js
// memory-vector.js:131-147
function dailyBackup() {
  try {
    if (!fs.existsSync(VEC_FILE)) return;
    const today = new Date().toISOString().slice(0, 10);
    const dir = path.join(projectRoot(), 'backups');
    const dest = path.join(dir, `memory-vectors-${today}.jsonl`);
    fs.mkdirSync(dir, { recursive: true });
    if (!fs.existsSync(dest)) fs.copyFileSync(VEC_FILE, dest);   // ← 同一天只备份第一次
    for (const f of fs.readdirSync(dir)) {                        // ← 保留 7 天
      const m = f.match(/^memory-vectors-(\d{4}-\d{2}-\d{2})\.jsonl$/);
      if (!m) continue;
      const age = (Date.now() - new Date(m[1]).getTime()) / 86400000;
      if (age > 7) fs.unlinkSync(path.join(dir, f));
    }
  } catch {}                                                      // ← 失败静默
}
```

| 项 | 实际 |
|---|---|
| 触发时机 | 仅在 `appendVectorMemory:154`（append 前） |
| 每日粒度 | 同一天只拷第一次（`if (!fs.existsSync(dest))`） |
| 保留 | 7 天，超期 `unlinkSync` |
| 失败处理 | `catch {}` 静默 → 备份失败无任何提示 |
| 删除/编辑是否触发 | **否** |

### 关键风险

#### 10.1 备份不检查 schema 版本

- 备份内容就是当时的 JSONL 原文
- 无 `schemaVersion` 字段、无版本标记
- 恢复到未来 V2 系统时：字段缺失（无 id/status/revision），需靠 `normalizeRow` 补默认值

#### 10.2 防复活机制完全缺失

| 复活路径 | 是否被阻止 |
|---|---|
| 从 `backups/` 恢复 | ❌ 无任何检查 |
| 从 `state/memory-vectors-admin-backup.jsonl` 恢复 | ❌ |
| 从 `D:\qq-bridge备份-*\*.zip` 恢复 | ❌ |
| 从 `state/*.pre-*.bak` 恢复 | ❌ |

**这是本次审查确认的最大缺口**：删除一条记忆后，任何人用任意历史备份覆盖文件，被删记忆就会全部复活，且代码不会察觉。

#### 10.3 空库被错误恢复成历史非空

- 当前 `state/memory-vectors.jsonl` = 0 字节
- `backups/memory-vectors-2026-10-30.jsonl` = 101 条
- `state/memory-vectors.jsonl.pre-p0a.bak` = 101 条
- `state/memory-vectors.jsonl.pre-vis.bak` = 101 条
- **无任何机制**防止误操作把 101 条恢复到当前空库

#### 10.4 历史分享审批状态会被带入

- 备份记录含 `share_review_status` / `visibility` / `share_requested_at` / `share_reviewed_at`
- 恢复后这些状态原样生效
- 例如：某条曾被批准为 `group_shared`，恢复后群聊立即可检索

#### 10.5 恢复 A 层会影响 SocialV2 关联

- 备份**只含 A 层**
- 恢复 A 层后，B 层的 `activeTopics` / `memberImpressions` 不会同步
- 结果：A 层有记录，B 层无对应槽位；或反之
- 无一致性检查

### 修复方向（对应 V2 设计）

1. 引入 `tombstone`（按 memoryId 拦截备份恢复）
2. 备份文件加 `schemaVersion` + 导出时间 + 记录数
3. 恢复函数必须：检查 schema → 对比 tombstone → 跳过已删 → 校验行数 → 原子写入
4. 空库保护：恢复前若当前库为空且有历史备份，要求显式确认

---

# 第三部分：审查结果汇总

| # | 审查项 | 状态 | 风险等级 | 核心结论 |
|---|---|---|---|---|
| 1 | CRC 递归规范化 | `NOT_IMPLEMENTED` | — | 无 `_crc`；坏行静默跳过（P1 附带缺陷） |
| 2 | embeddingStatus | `NOT_IMPLEMENTED` | — | 无状态机；embed 失败即丢记录（P1 附带） |
| 3 | WAL effects + revision | `NOT_IMPLEMENTED` | — | 无 WAL/无 revision；部分成功不可恢复（P0 缺口） |
| 4 | tombstone 检索屏障 | `NOT_IMPLEMENTED` | — | 无 tombstone；物理删除 |
| 5 | ShareRequest 孤儿副本 | `NOT_IMPLEMENTED` | — | 无独立对象；**群内分享实际可用（撤回上一轮误判）** |
| 6 | L2 删除与 B 层 | `PARTIAL` | P1 | 有 A/B 同步删除，但物理删除 + text 匹配 |
| 7 | writer.lock 生命周期 | `PARTIAL` | **P1** | 有 bridge.lock（质量较好），但存在启动窗口 |
| 8 | 特权操作审计 | `PARTIAL` | **P0** | 业务后审计 + 无 try/catch → 已执行但返回 500；clear 无审计 |
| **9** | **写入幂等性** | **`FAIL`** | **P0** | **embed 期间删除 → 记录复活**；无任何幂等机制 |
| **10** | **备份恢复防复活** | **`PARTIAL`** | **P1** | 有备份无恢复；防复活完全缺失 |

---

# 第四部分：问题严重程度排序

## P0（数据正确性）

| # | 问题 | 位置 | 触发 | 后果 |
|---|---|---|---|---|
| **P0-1** | **embed 期间删除 → 记录复活** | `memory-vector.js:361-374` | `editVectorMemory` 的 `await embed`（最长 30s）期间执行 remove/clear | 已删记忆重新出现在检索结果中 |
| **P0-2** | **审计在业务后 + 无 try/catch** | `bridge.js:6731-6734 / 6739-6741 / 6750-6755` | 磁盘满 / ACL 变更 | 记忆已删/已改，但返回 HTTP 500，且无审计痕迹 |
| **P0-3** | **无 WAL → 崩溃后 A/B 不一致不可恢复** | 全局 | 进程崩溃 | 永久不一致，无自动发现 |

## P1（功能正确性）

| # | 问题 | 位置 |
|---|---|---|
| P1-1 | `atomicRewrite` 固定 tmp 名（并发/脚本场景会损坏） | `memory-vector.js:377` |
| P1-2 | 锁启动窗口（`acquireLock:1390` → `listen:7531`）导致锁被误删 | `bridge.js:1390/7531` |
| P1-3 | `clear` 完全无审计 | `bridge.js:7014-7123` |
| P1-4 | append 无去重（A 层可无限重复） | `memory-vector.js:149-163` |
| P1-5 | 无幂等键，超时重试重复写入 | 全局 |
| P1-6 | 备份无 schema 版本、无恢复函数、防复活缺失 | `memory-vector.js:131-147` |
| P1-7 | `loadRows` 坏行静默跳过 | `memory-vector.js:53` |
| P1-8 | embed 失败 → 记录完全不写（正文丢失） | `memory-vector.js:153` |
| P1-9 | B 层关联靠 text 全等（改 text 即失联） | `bridge.js:6919` |
| P1-10 | `consolePortInUse` 探测失败保守返回 true → 可能永久无法启动 | `bridge.js:1076` |
| P1-11 | 审计无操作者/结果/opId | `bridge.js:6733/6740/6754` |

## P2（改进项）

| # | 问题 |
|---|---|
| P2-1 | 缓存只靠 `mtime` 失效（同毫秒可能读到旧值） |
| P2-2 | `atomicRewrite` 无 `fsync` |
| P2-3 | `appendFileSync` 前不检查尾行 `\n`（多行粘连） |
| P2-4 | 检索不校验 `vec.length===1024` |
| P2-5 | 无 `vecModel` 字段 |
| P2-6 | 审计含记忆正文前 80 字（隐私） |
| P2-7 | 审计无轮转/无上限 |
| P2-8 | `bridge.js:6759` 错误信息过时（"只支持 list/remove/edit"） |
| P2-9 | `bridge.lock` 无 `instanceId` / 无心跳 |
| P2-10 | A 层 rename 无重试（bridge 有 `renameWithRetry`，A 层没用） |

---

# 第五部分：对第一批报告的修正

| # | 第一批结论 | 修正 | 依据 |
|---|---|---|---|
| **1** | 「approveShare 只改 visibility 不改 key → 群检索永远搜不到 → 分享功能实质不可用」 | **撤回**。群内记忆审批后**本群可正常检索**；私聊→群不支持（已 400 拒绝，属设计边界） | 完整追踪 key 流转（见疑点 2） |
| **2** | 「atomicRewrite 固定 tmp 名 → 并发写必然数据损坏」 | **细化**。跨进程当前无第二写入者；真实风险是**同进程 async 丢失更新**（`editVectorMemory` 的 await） | 穷举写入口 + 7 个调用点的 async 分析 |
| **3** | 「P0：approveShare 导致分享功能不可用」 | 降级为**无缺陷**（见第 1 项修正） | 同上 |
| **4** | 未提及 embed 期间删除 | **新增 P0-1**：这是可单线程复现的删除复活路径 | 第 9.5 节推演 |

---

# 第六部分：尚未检查 / 证据不足

| # | 项 | 原因 |
|---|---|---|
| 1 | 并发写损坏的实际概率 | 需并发测试（禁止） |
| 2 | `consolePortInUse` 在无 netstat 环境的行为 | 需环境测试（禁止） |
| 3 | `mtime` 同毫秒碰撞概率 | 需构造测试（禁止） |
| 4 | `mcp-snowluma-safe.js` 全部 1465 行的逐函数审查 | 只抽查了记忆相关工具（append/query/remove/clear/share_request）；其余工具与 Memory 无关 |
| 5 | `token-ledger.js` / `slang-learner.js` / `sticker-lib.js` 与 Memory 的交互 | 它们各自写独立文件（token-usage.jsonl / slang / stickers.json），不写 memory-vectors |
| 6 | 是否有外部进程（非本项目脚本）写 `memory-vectors.jsonl` | 无法排除操作系统层面的其他程序 |
| 7 | 真实运行时的内存缓存行为 | 需运行时观察（禁止） |

---

# 第七部分：建议修复顺序

## 立即（不依赖 V2 完整实施）

```
1. P0-1：修 editVectorMemory 的 await 竞态
   最小改法：await 之后重新 loadRows + 按 key+text 重新定位，而不是用旧 rows 索引
   
2. P0-2：修审计顺序
   最小改法：appendFileSync 包 try/catch，失败只报警不抛错（避免业务成功返回 500）
   
3. P1-1：修 atomicRewrite tmp 名
   改 `${VEC_FILE}.${process.pid}.${crypto.randomBytes(6).toString('hex')}.tmp`
   + 复用 renameWithRetry
   
4. P1-2：调整锁与监听的顺序（先 listen 成功再拿锁，或锁年龄判断）
```

## 随 V2 P2（WAL）

```
5. P0-3：WAL + revision
6. P1-5：opId 幂等
```

## 随 V2 P4（生命周期）

```
7. P1-6：tombstone + 备份恢复校验
8. P1-3：clear 审计
9. P1-9：B 层 slotId
```

---

*本报告为只读分析结果。未修改任何代码/配置/数据，未启动服务。*
