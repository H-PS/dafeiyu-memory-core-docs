# 08 · social-v2 拆分与并发一致性

> 对应 R7。核心提醒（R7 原文）：**"把字段挪进独立文件"本身不解决并发。**
> 本文先给归属，再给六要素，最后说清"分文件解决了什么、没解决什么"。

---

## 一、现状：一个文件，三种性质，43 处整文件重写

| 事实 | 来源 |
|---|---|
| `social-v2.json` 同时承载：运行态（phase/游标/唤醒配置）、B 层业务记忆（话题/想法/印象）、凭据（token） | `【源码】bridge.js:9173-9262 / 2793-2799 / 10590-10710` |
| 整文件重写路径约 43 处 | `【审计】06-C-11`（Rev2 前序审计结论） |
| **完整清单未产出** | `【UNCONFIRMED】U-8` |
| 已知后果 | `interactionCount` 每轮 +1 却写在整文件里（`:9260`）→ 并发丢更新（S-1） |

**43 处的行为不可完全枚举**，但它们的行为模式可归类：

| 模式 | 典型 | 风险 |
|---|---|---|
| 读 → 改自己的字段 → 整文件写回 | `appendMemoryV2`、wakeConfig 更新、游标推进 | **旧快照覆盖**（两个写者各读到同一旧值） |
| 读 → 加一个新 key 的段 → 写回 | 新会话初始化 | 覆盖别人的段 |
| 整体重置（reset/clear 相关） | `/reset`、`reset` 端点 | 清掉其他字段 |
| 只改一层嵌套字段 | `memberImpressions[name]` | 整对象替换（`:9258-9262`） |

---

## 二、字段归属矩阵（R7 要求，含"为什么"）

| 字段 | 性质 | 写频 | 归属决定 | 理由 |
|---|---|---|---|---|
| `agentToken` | 凭据 | 极低（退役时） | **→ `state/agent-token.json`** | 安全敏感；不能被"改话题"这种高频写顺带覆盖；需要独立审计 |
| `bootstrapSent` / `modelSeenSeqs` | 会话语义 | 低 | **留 S** | 与会话强绑定，退役时清；无独立价值 |
| `unread[]` / `recentMessages[]` | 消息缓冲 | **极高**（每条消息） | **留 S，但写入收敛到单一消息写者** | 高频但只在消息线程内写；拆文件收益低、迁移成本高 |
| `activeTopics` / `pendingThoughts` / `memberImpressions` | **B 层业务记忆** | 中（每条记忆写入） | **留 S** | B 层是唯一副本，不能变成"派生"；且 `formatMemoryV2` 每轮读它 |
| `memberImpressions[*].interactionCount` | 计数 | **极高**（每轮 +1） | **→ `state/impression-stats.json`** | 唯一"高频 + 整对象替换"的字段，是 S-1 的直接原因 |
| `wakeConfig` | 配置 | 低 | **留 S** | 与会话绑定 |
| `phase` / `activeDeadlineAt` / `lastIncomingAt` 等 | 状态机 | 高（秒级） | **留 S，但由 `socialLoopTick` 单写** | 只有状态机自己改它 |
| **`promptLedger`**（新） | 账本 | 每轮 | **→ `state/prompt-ledger.json`** | 可丢弃（L3）；与 B 层无关；高频写，不该进大文件 |
| **`rotationTx`**（新） | 事务日志 | 极低 | **→ `state/rotation-journal.json`** | 必须原子、必须能在崩溃后读到 |
| **`clearEpoch`**（新） | fencing 计数 | 低（只在 clear 时） | **→ `state/memory-epochs.json`** | 与 A 层写入强相关，必须与 commit 同步读 |
| **`reviewQueue`**（新） | 队列 | 低 | **→ `state/review-queue.jsonl`** | append-only，天然适合 JSONL |
| **`pendingEmbed`**（新） | 重试队列 | 中 | **→ `state/pending-embed.jsonl`** | 同上；可全量重建 |

### 2.1 拆分后的文件清单

```
state/
├── social-v2.json             ← 只留：运行态 + B 层 + 会话元信息
├── agent-token.json           ← 凭据（0700/0600）
├── impression-stats.json      ← interactionCount
├── prompt-ledger.json         ← 块账本（L3 可丢）
├── rotation-journal.json      ← 轮转事务
├── memory-epochs.json         ← per-key clearEpoch
├── review-queue.jsonl         ← 待办（append + compact）
├── pending-embed.jsonl        ← 待补算
├── memory-store.jsonl         ← A 层
├── memory-impressions?        ← （不新建）
├── memory-candidates.jsonl    ← 候选池
├── memory-quarantine.jsonl    ← 隔离区
├── index/                     ← L1 可重建
├── journal/memory.jsonl       ← A 层写入 journal
└── logs/prompt-dump/          ← 元数据 dump（04 §7）
```

**文件数从 1 个（S）+1 个（A）变成 12 个左右**。这是有意的代价：**每个文件一个写者、一种原子性要求**。

---

## 三、六要素（R7 的核心要求）

> 分文件只降低"整文件重写互相覆盖"的概率。真正的正确性靠下面六条。

### 3.1 单写者 ownership

| 文件 | 唯一写者 | 其他模块如何改它 |
|---|---|---|
| `social-v2.json` | `socialStateWriter`（一个新函数出口） | **不允许**直接 `writeFileSync`；所有修改走它 |
| `agent-token.json` | `credential.js` | 通过 `rotate()` / `get()` |
| `impression-stats.json` | `memory-store.js` 的统计写入 | 通过 `commit({type:'bumpImpression'})` |
| `prompt-ledger.json` | `prompt-ledger.js` | 通过 `markSent()` / `forget()` |
| `memory-epochs.json` | `memory-store.js` | 通过 `getEpoch()` / `bumpEpoch()` |
| `review-queue.jsonl` | `memory-store.js` | 通过 `enqueueReview()` / `resolveReview()` |
| `pending-embed.jsonl` | `memory-store.js` | 通过 `enqueueEmbed()` / `popEmbed()` |
| `memory-store.jsonl` | `memory-store.js` | 通过 `commit()` |

**验证方式**：`grep -rn "writeFileSync\|appendFileSync" src/ | grep -v memory-store.js` 中不得出现上述文件名。

### 3.2 per-key 串行化

```js
const locks = new Map();   // key -> Promise
function withKeyLock(key, fn) {
  const prev = locks.get(key) ?? Promise.resolve();
  const next = prev.then(fn, fn);            // 前一个失败也要继续
  locks.set(key, next.catch(() => {}));
  return next;
}
```

- 粒度：**按 key**，不是全局（不同群可并行）。
- 范围：覆盖 `commit()` 的阶段 3 全程（含预检、journal、写、提交）。
- 不是文件锁，是进程内 Promise 链。

### 3.3 原子替换

| 写类型 | 方式 |
|---|---|
| JSON 全量 | `atomicWriteJson()`（已有：随机 tmp + rename + EPERM 退避） |
| JSONL 追加 | `appendFileSync`（单行 < 4KB，POSIX 下原子；Windows 下依赖单写者） |
| JSONL 压缩 | 写 tmp → rename（**不得原地截断**） |

**禁止**：`const tmp = file + '.tmp'`（现状 `memory-vector.js:376` 的缺陷）。

### 3.4 陈旧写防护（fencing）

```js
// 所有对"会被 clear 影响"的写，必须带 expectEpoch
commit({ ..., expectEpoch: epochs.get(key) })
// store 内部：
if (epochs.get(key) !== expectEpoch) → abort('cleared_during_write')
```

**为什么不靠时间戳/重试**：时间戳无法判定"我读到的是最新状态"；重试会放大错误。

**对 S 层的同类保护**：`social-v2.json` 写入前校验 `stateRev`（文件级单调计数），不等则重读重试一次，仍不等则放弃本次写并告警（说明有第三方在写）。

### 3.5 崩溃恢复

| 文件 | 恢复方式 |
|---|---|
| `memory-store.jsonl` + `journal/memory.jsonl` | 重放 pending（幂等，见 `05` §2.4） |
| `memory-epochs.json` | 原子写，无需恢复；丢失则从 journal 末条推导 |
| `rotation-journal.json` | 启动时若 phase≠done，从该 phase 续跑（`04` §4.4） |
| `review-queue.jsonl` / `pending-embed.jsonl` | 坏行跳过 + 计数；压缩时重建 |
| `index/` | `meta.json` crc 不符 or generation 不连续 → 全量重建 |
| `social-v2.json` | 解析失败 → **拒绝启动**（现状 fail-fast，保持） |

### 3.6 顺序保证

| 需求 | 机制 |
|---|---|
| A 先于索引 | commit 提交后入 indexQueue（异步）。索引可滞后，不可领先 |
| 删除先于一切 | 删除时**同步**移索引（在同一临界区） |
| epoch 先于写入 | `bumpEpoch` 在 clear 事务的第 1 步，写入在后续校验它 |
| token 先于会话退役 | `credential.rotate()` 在 `retireSession()` 之前（`04` §4.4） |

---

## 四、拆分解决了什么、没解决什么（诚实结论）

| 问题 | 分文件能否解决 | 说明 |
|---|---|---|
| 高频字段被低频整文件重写覆盖（`interactionCount`） | ✅ 能 | 独立文件 + 独立写者，物理隔离 |
| 凭据被业务写顺带覆盖 | ✅ 能 | 同上 |
| 两个写者并发改同一文件丢更新 | ❌ **不能** | 需要 §3.2 的锁 + §3.4 的 fencing |
| 崩溃后状态半写 | ❌ **不能** | 需要 §3.5 的 journal/原子替换 |
| 索引与 A 不一致 | ❌ **不能** | 需要 §3.6 的顺序 + `coveredRev` 补偿 |
| 43 处重写点的"语义不明" | ⚠️ 部分 | 拆走后剩下的 B 层/运行态仍需收敛到单一写者，工作量大头在这里 |

### 4.1 因此必须做的额外工作

1. **把 `social-v2.json` 的所有 `writeFileSync` 收敂到 `socialStateWriter`**（这一步无法靠分文件绕过）。
2. **产出 43 处清单**（U-8）：迁移前必须做，方式是 `grep` + 人工归类，不是再扫一遍架构。
3. **给 `socialStateWriter` 加 `stateRev` 校验**，让"第三方写入"变成可检测错误而非静默丢数据。

---

## 五、迁移步骤（可回滚）

```
阶段 0  产出 43 处清单（U-8），标注每处的"读-改-写"边界
阶段 1  新建 7 个文件（空结构），双读单写：
         读：优先新文件，缺失则回退 S 层字段
         写：只写新文件（S 层字段保留但不再更新）
阶段 2  观察 7 天：新文件与 S 层字段的差异告警（应始终一致）
阶段 3  写路径切换完成：S 层字段标记 deprecated（不删）
阶段 4  移除 S 层旧字段（需一次 purge 级确认）
```

**回滚**：任一阶段可回退到"只读 S 层"（新文件都在，不影响旧逻辑）；阶段 4 之后不可回退（旧字段已删）。

**数据量**：当前记忆库 0 条、B 层只有 2 个 key（`private:[OWNER_QQ]` / `group:[GROUP_ID]`，`【审计】项目全景` §九），所以迁移应在一个维护窗口内完成，不需要长时间双写。

---

## 六、对 Recall/Retain 的影响

| 模块 | 依赖的文件 | 影响 |
|---|---|---|
| Recall | A + index + epochs（读） | epoch 只用于过滤，不阻塞读 |
| Retain | A + epochs + candidates + quarantine + pendingEmbed | 一次 commit 可能触及 3 个文件 → 必须在同一 per-key 锁内，且 journal 记录全部 effects |
| Reflect | A + reviewQueue | 只 append queue 与改 lifecycle |
| Ledger | prompt-ledger.json | 独立，不参与记忆锁 |
| Credential | agent-token.json | 独立锁（`withKeyLock('token:'+key)`），与记忆锁分开，避免死锁 |

**死锁预防**：锁顺序固定 —— `token → memory → ledger`。任何跨模块操作必须按此顺序取锁；不允许反向。
