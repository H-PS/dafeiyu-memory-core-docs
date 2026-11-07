# 11 · 源码证据索引

> 所有对 V1 行为的判断，按本文件索引标注。
> `SOURCE_CONFIRMED`：已在源码逐行确认（附文件:行号）
> `DESIGN_INFERENCE`：依据设计规则推演，源码未实现故无法直接验证
> `NOT_VERIFIABLE`：证据不足

---

# 1. A 层（`src/memory-vector.js`，380 行）

| 编号 | 结论 | 行号 | 关键代码 | 标注 |
|---|---|---|---|---|
| E-01 | `editVectorMemory` 用 await 前快照全量覆盖 | 361-374 | `:362 const rows = loadRows();`<br>`:367 const vec = await embed(clean);`<br>`:372 atomicRewrite(rows);` | `SOURCE_CONFIRMED` |
| E-02 | `loadRows` 返回 `cache.rows` **引用**（非副本） | 44-58 | `:47 if (mtime === cache.mtime) return cache.rows;`<br>`:56 return rows;`（rows 即 cache.rows） | `SOURCE_CONFIRMED` |
| E-03 | `loadRowsStrict` 每次**新建**数组 | 68-83 | `:75 const rows = [];` | `SOURCE_CONFIRMED` |
| E-04 | `atomicRewrite` 固定 tmp 名 | 376-380 | `:377 const tmp = VEC_FILE + '.tmp';` | `SOURCE_CONFIRMED` |
| E-05 | 无 fsync / 无 renameWithRetry / 失败留 tmp | 378-379 | `:378 writeFileSync(tmp, …)`<br>`:379 renameSync(tmp, VEC_FILE)`（无 try/finally） | `SOURCE_CONFIRMED` |
| E-06 | 坏行静默跳过 | 51-54 | `:53 try { rows.push(JSON.parse(line)); } catch {}` | `SOURCE_CONFIRMED` |
| E-07 | 读异常返回 `[]` | 57 | `} catch { return []; }` | `SOURCE_CONFIRMED` |
| E-08 | append 无去重 | 149-163 | 仅 `:152 infoOK(clean)` 检查，无查重 | `SOURCE_CONFIRMED` |
| E-09 | embed 失败即不落盘 | 153 → 160 | `:153 const vec = await embed(clean);` 在 `:160 fs.appendFileSync` 之前 | `SOURCE_CONFIRMED` |
| E-10 | embed 超时 30s、无重试 | 86-108 | `:92 timeout: 30000` | `SOURCE_CONFIRMED` |
| E-11 | 向量维度无校验 | 110-120 | `cosSim` 仅判 `a.length !== b.length` → 返回 0 | `SOURCE_CONFIRMED` |
| E-12 | 缓存仅靠 mtime | 42 / 46-47 | `let cache = { mtime: 0, rows: [] }` | `SOURCE_CONFIRMED` |
| E-13 | `findMemoryRow` **四级**匹配（注释写"三级"） | 235-258 | `:238` 全等<br>`:240` 双向包含<br>`:242-246` 去前缀包含<br>`:249-256` 归一化包含 | `SOURCE_CONFIRMED` |
| E-14 | `approveShare` 不改 key | 277-291 | `:284-286` 只改 status/visibility/reviewed_at | `SOURCE_CONFIRMED` |
| E-15 | 分享状态仅单字段 | 157 | `share_review_status: 'not_requested'` | `SOURCE_CONFIRMED` |
| E-16 | `dailyBackup` 仅 append 触发、同日一次、`catch{}` 静默 | 131-147 / 154 | `:138 if (!fs.existsSync(dest)) copyFileSync`<br>`:146 } catch {}`<br>`:154 dailyBackup();` | `SOURCE_CONFIRMED` |
| E-17 | 无 tombstone / 无恢复函数 / 无 schemaVersion | 全文 | 0 命中 | `SOURCE_CONFIRMED` |
| E-18 | append 前不检查尾行 `\n` | 160 | `fs.appendFileSync(VEC_FILE, JSON.stringify(row) + '\n')` | `SOURCE_CONFIRMED` |
| E-19 | `splice` 就地修改（影响 R2 分支） | 311-319 | `:315 const [removed] = rows.splice(idx, 1);` | `SOURCE_CONFIRMED` |
| E-20 | `removeNAll` / `removeAll` 用 `loadRowsStrict` | 325 / 350 | `const rows = loadRowsStrict();` | `SOURCE_CONFIRMED` |

## 1.1 R1/R2 分支的判定依据（重要）

```
edit ∥ removeVectorMemoryExact（E-19 splice 就地改）
  ⇒ edit 的 rows 与 remove 的是同一数组 ⇒ 不复活，而是"改错记录 / TypeError"

edit ∥ removeAll / removeNAll（E-20 新建数组）
  ⇒ edit 的旧数组仍含被删记录 ⇒ atomicRewrite 后复活

`SOURCE_CONFIRMED`（依据 E-02 / E-03 / E-19 / E-20）
```

---

# 2. B 层与端点（`src/bridge.js`）

## 2.1 S 层写入点（43 处 `saveSocialV2State` + 4 处 strict + 1 处 reset）

```
saveSocialV2State() 调用行（43 处）：
  1829, 2801, 4583, 4820, 4957, 5011, 5174, 5195, 5310, 5323, 5344, 5440,
  5820, 5830, 5866, 5874, 6173, 6828, 7365, 7383, 7435, 7496, 7519,
  8685, 8758, 8925, 10243, 10285, 10312, 10836, 10842, 10852, 10874,
  11311, 11384, 11690, 11888, 11946
  （另 7516/7519 在 /api/restart 路径）

saveSocialV2StateStrict() 调用行（4 处，记忆端点）：
  6668（append）, 6941（remove）, 6990（remove 的 mI 分支）, 7047/7100（clear 两分支）

整文件重置（1 处，不属于 save）：
  4624  atomicWriteJson(SOCIAL_V2_FILE, { conversations: {} })
```
`SOURCE_CONFIRMED`（全文件正则穷举）

## 2.2 S 层字段（实测 `state/social-v2.json`）

```
顶层：paused, conversations
会话内 21 字段：
  wakeConfig(10键), recentMessages(100项), unread, lastWakeReason, lastAiReplyAt,
  lastActionAt, agentToken, bootstrapSent, wakeTimes, sendTimes, stickerCollectTimes,
  lastIncomingAt, preSleepWaitSatisfiedAt, preSleepWaitObservedAt, preSleepWaitAccumMs,
  lastUnreadSeq, lastReadThroughSeq, activeTopics, pendingThoughts,
  memberImpressions, seenForwardIds
记忆三字段：activeTopics / pendingThoughts / memberImpressions（占 3/21）
运行状态：其余 18 个（占 18/21）
```
`SOURCE_CONFIRMED`

## 2.3 端点

| 编号 | 结论 | 行号 | 标注 |
|---|---|---|---|
| E-21 | A 层写入用**未脱敏、未截断** content | 6658 | `await appendVectorMemory(key, \`${category}: ${content}\`)` | `SOURCE_CONFIRMED` |
| E-22 | P/S 三字段写入用**脱敏 + 截断 200** | 9207 / 9228 / 9244 | `redactKnownTokensOnly(...)` / `truncateText(text,200)` / `text.slice(0,200)` | `SOURCE_CONFIRMED` |
| E-23 | `aRemoved` 计算后未上报 | 6962/6964/7011 | `:7011 removed: bRemoved` | `SOURCE_CONFIRMED` |
| E-24 | B 淘汰、A 不淘汰 | 9210-9215 / 9234 / 9250 / 9264 | 过期 filter、`length > 20` splice、`length > 50` delete | `SOURCE_CONFIRMED` |
| E-25 | A 用原始 key、B 用 canonical | 6658 vs 8765-8772 | `getSocialV2State` 内 `key = canonical`，但 6658 用未规范化的 key | `SOURCE_CONFIRMED` |
| E-26 | append：A 失败 B 已落盘 | 6655-6662 / 6667-6691 | A 失败仅 `aSynced=false`，B 仍 `saveSocialV2StateStrict()` | `SOURCE_CONFIRMED` |
| E-27 | append：B 失败回滚 B，A 已写 | 6667-6681 | 只回滚 B 快照，**不回滚 A** | `SOURCE_CONFIRMED` |
| E-28 | remove：A 已删、B 落盘失败 | 6964 / 6989-7010 | A 删除在前且不可逆 | `SOURCE_CONFIRMED` |
| E-29 | memberImpression 删 B 不删 A | 6932-6956 | `aSynced:'n/a'`，A 残留 | `SOURCE_CONFIRMED` |
| E-30 | 分类 clear 只清 B | 7024-7068 | `aSynced:'n/a'`，A 保留 | `SOURCE_CONFIRMED` |
| E-31 | 审计在业务之后、无 try/catch | 6731→6733 / 6739→6740 / 6751→6754 | `fs.appendFileSync` 裸调用；外层 catch 6760 → 500 | `SOURCE_CONFIRMED` |
| E-32 | clear / memory-remove **零审计** | 6903-7122 | 全文无 `memory-admin.log` 写入（仅 6733/6740/6754） | `SOURCE_CONFIRMED` |
| E-33 | 审计无操作者/结果/opId | 6733-6734 等 | `ISO时间 OP key \| text.slice(0,80)` | `SOURCE_CONFIRMED` |
| E-34 | 审计含正文前 80 字 | 6734/6741/6755 | `${removed.text.slice(0,80)}` | `SOURCE_CONFIRMED` |
| E-35 | admin 前备份同名覆盖 | 6716 | `copyFileSync(VEC_FILE_PATH, ...'memory-vectors-admin-backup.jsonl')` | `SOURCE_CONFIRMED` |
| E-36 | 未知 op 文案过时 | 6759 | `'未知 op，只支持 list/remove/edit'`（实际还支持 approve_share/reject_share） | `SOURCE_CONFIRMED` |
| E-37 | 条件式鉴权（96 处） | 6637-6639 等 | `if (req.headers['x-agent-token'] && !agentTokenOk(...))` | `SOURCE_CONFIRMED` |
| E-38 | 管理端拒绝 agent token | 6707 | `if (req.headers['x-agent-token'] !== undefined) → 403` | `SOURCE_CONFIRMED` |
| E-39 | private 拒绝分享 | 6892-6894 | `if (key.startsWith('private:')) → 400` | `SOURCE_CONFIRMED` |

## 2.4 ★ 本版新增证据

| 编号 | 结论 | 行号 | 关键代码 | 标注 |
|---|---|---|---|---|
| **E-40** | **`memory-update` 只改 B 层，完全不碰 A 层** | 6765-6831 | `:6789-6794` 改 activeTopics<br>`:6795-6800` 改 pendingThoughts<br>`:6801-6827` 改 memberImpressions<br>`:6828 saveSocialV2State()`（非 strict）<br>**全程无 appendVectorMemory / editVectorMemory** | `SOURCE_CONFIRMED` |
| **E-41** | **memberImpression traits 上限两条路径不一致** | 9257 vs 6815 | append：`traits.slice(0,10)`（读时）<br>update：`.split(...).slice(0,20)` | `SOURCE_CONFIRMED` |
| **E-42** | **`memory-update` 支持 memberImpression 改名** | 6812-6819 | `:6812 newTarget=...`<br>`:6818 delete st.memberImpressions[oldTarget]`<br>`:6819 st.memberImpressions[newTarget]=im` | `SOURCE_CONFIRMED` |
| **E-43** | **`reset` 整文件重写 social-v2.json** | 4624 | `atomicWriteJson(SOCIAL_V2_FILE, { conversations: {} })` | `SOURCE_CONFIRMED` |
| **E-44** | **`formatMemoryV2` 读 S 层三字段** | 8945-8975 | 读 `st.activeTopics` / `st.pendingThoughts` / `st.memberImpressions` | `SOURCE_CONFIRMED` |
| **E-46a** | **`participants` 两条路径上限不一致** | 9223 vs 6794 | append：`[...set].slice(0,10)`（:9223）<br>update：`= cleanNewExtra.participants` **直接赋值无上限**（:6794） | `SOURCE_CONFIRMED` |
| **E-46b** | **`pendingQuestion` 两条路径截断不一致** | 9225 vs 6793 | append：`truncateText(...,200)`（:9225）<br>update：`String(...)` **无截断**（:6793） | `SOURCE_CONFIRMED` |
| **E-46c** | **`motivation` 两条路径截断不一致** | 9241 vs 6799 | append：`.slice(0,50)`（:9241）<br>update：`String(...)` **无截断**（:6799） | `SOURCE_CONFIRMED` |
| **E-46d** | **A 层无 extra 字段**（V1 只有 B 层有） | 全仓 | `memory-vector.js:155-158` 写入的 row 只有 key/text/time/vec/share_review_status，无 participants/pendingQuestion/motivation | `SOURCE_CONFIRMED` |
| **E-47a** | **`interactionCount` 全部读写仅 3 处** | 9260 / 6817 / 9199 | `:9260 = (old\|\|0)+1`（append）<br>`:6817 = Math.max(0, Number(...))`（update 可改）<br>`:9199` 显示（formatMemoryV2） | `SOURCE_CONFIRMED` |
| **E-47b** | **`lastSeenAt` 全部读写仅 3 处** | 9261 / 6824 / 9265 | `:9261 = Date.now()`（append）<br>`:6824` / `:9265` 淘汰排序用<br>★ update 路径**不更新** lastSeenAt | `SOURCE_CONFIRMED` |
| **E-47c** | **`expiresAt` 的过滤点共 4 处** | 6842 / 9185 / 9211 / 11124 | memory GET / formatMemoryV2 / append 开头清理 / 唤醒时过滤 | `SOURCE_CONFIRMED` |
| **E-45** | **memberImpressions 结构是"名字 → 一个对象 + traits 数组"** | 9251-9269 | `:9258 st.memberImpressions[target] = { traits, interactionCount, lastSeenAt }`<br>`:9257 traits.push(...)`（追加） | `SOURCE_CONFIRMED` |

---

# 3. 锁与启动（`src/bridge.js`）

| 编号 | 结论 | 行号 | 标注 |
|---|---|---|---|
| E-46 | `LOCK_FILE` 定义 | 1054 | `path.join(STATE_DIR, 'bridge.lock')` | `SOURCE_CONFIRMED` |
| E-47 | `consolePortInUse` 失败开放 | 1064-1078 | `:1074-1077 catch { return true; }` | `SOURCE_CONFIRMED` |
| E-48 | `acquireLock` 原子创建 + PID | 1080-1142 | `:1085 writeFileSync(LOCK_FILE, String(process.pid), {flag:'wx'})` | `SOURCE_CONFIRMED` |
| E-49 | `releaseLock` 仅按 PID 匹配 | 1143-1149 | `Number(...) === process.pid` | `SOURCE_CONFIRMED` |
| E-50 | `acquireLock()` 在 main 早期调用 | 1390 | `acquireLock();` | `SOURCE_CONFIRMED` |
| E-51 | `loadState()` 紧随其后 | 1391 | `loadState();` | `SOURCE_CONFIRMED` |
| E-52 | `server.listen` 在 main near 末尾 | 7531 | `server.listen(port, '127.0.0.1', cb)` | `SOURCE_CONFIRMED` |
| E-53 | EADDRINUSE → exit(2) | 7535-7542 | `:7537-7539` | `SOURCE_CONFIRMED` |
| E-54 | 退出钩子四处 | 12447/12453/12471/12477 | SIGINT/SIGTERM/uncaughtException/exit → releaseLock | `SOURCE_CONFIRMED` |
| E-55 | 锁→监听之间含真实初始化 | 9275 / 9277 / 12425 | `loadSocialV2State()` / `loadInitialPrompts()` / `bot.connect()` | `SOURCE_CONFIRMED` |

**推论（DESIGN_INFERENCE）**：E-50 → E-52 之间的窗口远大于"函数定义耗时"，因为含 E-55 的真实 IO 与网络操作。

---

# 4. 审计与鉴权

| 编号 | 结论 | 行号 | 标注 |
|---|---|---|---|
| E-56 | 全局 console token 定长比较 | 3263-3282 | `crypto.timingSafeEqual(a,b)`（:3272） | `SOURCE_CONFIRMED` |
| E-57 | 全局 x-agent-call 门禁 | 3302-3318 | `/api/socialV2/*` 无令牌智能体调用 403 | `SOURCE_CONFIRMED` |
| E-58 | Host 白名单（抗 DNS rebinding） | 3254-3261 | 仅 `127.0.0.1/localhost/[::1]` | `SOURCE_CONFIRMED` |
| E-59 | CSRF（JSON-only + Origin） | 3322-3337 | 非 GET 必须 `application/json` + 同源 | `SOURCE_CONFIRMED` |
| E-60 | 会话令牌定长比较 | 3140-3145 | `timingSafeEqual` | `SOURCE_CONFIRMED` |
| E-61 | 空令牌启动失败关闭 | 3132 | 抛错拒绝启动 | `SOURCE_CONFIRMED` |

---

# 5. MCP 层（`src/mcp-snowluma-safe.js`，1465 行）

| 编号 | 结论 | 行号 | 标注 |
|---|---|---|---|
| E-62 | `agentApi` 超时 15s（< A 层 embed 30s） | 155 | `init.timeoutMs \|\| 15000` | `SOURCE_CONFIRMED` |
| E-63 | MCP 不再直连 A 层 | 21 / 944-945 | 注释 + 转发 bridge | `SOURCE_CONFIRMED` |
| E-64 | `ok:false` 时抛错 | 165-168 | `if (body.ok === false) throw` | `SOURCE_CONFIRMED` |
| E-65 | 恒发 `x-agent-call: 1` | ~194 | `headers['x-agent-call'] = '1'` | `SOURCE_CONFIRMED` |
| E-66 | 无关键词降级兜底 | 全文 | `m-v.js:10-11` 声明的降级契约**未实现** | `SOURCE_CONFIRMED` |
| E-67 | `qq_memory_query` A 失败丢 B 结果 | 993 / 998 | B 层 `out` 被丢弃 | `SOURCE_CONFIRMED` |

---

# 6. 其他源文件（与 Memory 无耦合）

| 文件 | 结论 | 标注 |
|---|---|---|
| `src/mcp-diary.js`（113 行） | 只写 `diary/YYYY-MM-DD.md`，与 Memory **零耦合** | `SOURCE_CONFIRMED` |
| `src/mcp-host-server.js`（93 行） | 仅探活 SnowLuma，**零耦合** | `SOURCE_CONFIRMED` |
| `src/qq-model-view.js`（134 行） | 纯序列化 `serializeModelData`，不写状态 | `SOURCE_CONFIRMED` |
| `src/token-ledger.js`（958 行） | 写 `token-usage.jsonl`，与 Memory 无交集 | `SOURCE_CONFIRMED` |
| `src/slang-learner.js`（382 行） | 写 slang 独立文件，与 Memory 无交集 | `SOURCE_CONFIRMED` |
| `src/sticker-lib.js`（302 行） | 写 `stickers.json`，与 Memory 无交集 | `SOURCE_CONFIRMED` |
| `src/self-test.js`（118 行） | 0 处 memory 引用 | `SOURCE_CONFIRMED` |

---

# 7. 全仓写入入口穷举（最终结论）

## 7.1 A 层 `memory-vectors.jsonl`

```
① memory-vector.js:160  appendFileSync         — appendVectorMemory
② memory-vector.js:378  writeFileSync + :379 rename — atomicRewrite（7 个调用点）
③ bridge.js:6716        copyFileSync（拷出）    — 只读源，不写回

→ 无第三个写入者。seed-group-owner.mjs 已归档为 .disabled
```
`SOURCE_CONFIRMED`

## 7.2 S 层 `social-v2.json`

```
① saveSocialV2State()        × 43 处（运行状态 + 1 处 memory-update:6828）
② saveSocialV2StateStrict()  × 4 处（append/remove/clear）
③ atomicWriteJson(SOCIAL_V2_FILE, {conversations:{}})  × 1 处（reset:4624）
④ fs.statSync(SOCIAL_V2_FILE) — 只读（:8820）

→ 共 48 个写入点，其中仅 5 个与记忆相关（4 strict + 1 memory-update 的非 strict）
```
`SOURCE_CONFIRMED`

## 7.3 其他

```
backups/memory-vectors-*.jsonl    — dailyBackup:138 copyFileSync
state/memory-admin.log            — 6733/6740/6754
state/bridge.lock                 — acquireLock:1085 / releaseLock:1146
```
`SOURCE_CONFIRMED`

---

# 8. DESIGN_INFERENCE 清单（推演前提）

| # | 结论 | 推演前提 |
|---|---|---|
| D-1 | `rev` 混用会让 pending edit 被误 superseded | 前提：①设计 `03 §5.3` 的 superseded 规则按 targetRev 比较 ②`06 §1.3` 的 embedding_refresh 会 rev+1 ③两者都按设计实现 |
| D-2 | 多群审批会互相失效 | 前提：①`05 §3 ⑨` 原件 rev+1 ②`05 §3 ④` 审批校验 expectedRevAtRequest ③两者都按设计实现 |
| D-3 | Reconciler 若全量重建会丢 39 类运行状态 | 前提：①Reconciler 用 `atomicWriteJson` 写整个 social-v2.json ②设计未定义"只改三字段"的实现约束 |
| D-4 | embedding 提交顺序 ≠ 请求顺序 | 前提：①P1 队列外并行 embed ②P2 FIFO 按入队顺序 |
| D-5 | M0 缺恢复流程则人工 copy 绕过 tombstone | 前提：①tombstone 检查在恢复流程内（06 §4 R1）②M0 无该流程 |

**以上均为"按设计实现后会发生"，不是"当前 V1 已发生的故障"。**

---

# 9. NOT_VERIFIABLE 清单

| # | 项 | 缺什么 | 如何验证 |
|---|---|---|---|
| N-1 | 并发竞态的实际触发概率 | 运行时并发测试 | 禁用；仅静态推演 |
| N-2 | `truncateText`（`md-to-plain.js`）语义 | 该文件未在导出包内 | 本版改用自实现 `clampText`，不再依赖 |
| N-3 | JSONL 全量重写在 10 万条的耗时 | 性能测试（当前 0 条） | 实施期压测 |
| N-4 | Ollama 并发嵌入吞吐 | 环境测试 | 实施期观察 |
| N-5 | `fsync` 在 Windows NTFS 的实际性能 | 环境测试 | 实施期测量 |
| N-6 | `consolePortInUse` 无 netstat 时的行为 | 环境测试 | 实施期模拟 |
| N-7 | 是否有项目外进程写 store | 静态不可排除 | 运行时 EXTERNAL_WRITE 侦测 |
| N-8 | 39 处 S 层写入与 P 层的真实并发频率 | 运行时观察 | M0 上线后监控 |
| N-9 | 原始 SHA256 与项目原文件的一致性 | 脱敏不可逆 | 本版直接读源文件，**不依赖导出包** |

---

# 10. 本版对证据的使用声明

```
✅ 本版所有 V1 行为判断均直接读取 D:\qq-bridge 源码
✅ 未依赖上一轮的导出包（source/*.md），避免脱敏/分卷问题
✅ 未虚构任何行号或代码片段
✅ DESIGN_INFERENCE 均标注推演前提（§8）
✅ NOT_VERIFIABLE 均说明缺少什么证据（§9）
```
