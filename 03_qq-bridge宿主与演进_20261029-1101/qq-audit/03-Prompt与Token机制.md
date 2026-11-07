# 03 · Prompt 与 Token 机制

> 只描述现状。分块尺寸为**估算**（中文按 3 字节/字折算字符数），未实测。

---

## 一、Prompt 组装：11 个块，逐块溯源

唯一自动注入点：`buildWakePromptV2(key, reason)`，`bridge.js:10669`。

| # | 块 | 源码 | 典型字符 | 每轮都发 | 有预算 | 内容变化 |
|---|---|---|---|---|---|---|
| 1 | 人格卡全文 | `:10671` `currentRoleHintV2()` | ~1,380 | ✅ | 受 `role.maxInjectChars=6000` 上限（`config.json:13`） | 几乎不变 |
| 2 | 你的身份 | `:10672` `selfIdentityTextV2()`（`:10657`） | ~180 | ✅ | 无 | 不变 |
| 3 | 【会话令牌】 | `:10674` `st.agentToken` | ~50 | ✅ | 无 | 退役时换 |
| 4 | B 层记忆 | `:10675` `formatMemoryV2(st)`（`:9173`） | 200–5,000 | ✅ | **无** | 变化 |
| 5 | 参与度 | `:10679` `formatParticipationV2(st)` | ~200 | ✅ | 无 | 变化 |
| 6 | 表情策略 + 清单 | `:10683-10685` `buildStickerStrategyHint()` + `buildStickerContext()`（`promptMaxStickers=8`） | ~600 | ✅ | 只限条数 | 清单罕见变 |
| 7 | 【首回复节奏】 | `:10689` 硬编码长句 + `replyTimingV2(st).quietMs` | ~230 | ✅ | 无 | 规则不变、数值变 |
| 8 | 【沉睡前强制等待】 | `:10690` 硬编码长句 + `preSleepMs` | ~430 | ✅ | 无 | 规则不变、数值变 |
| 9 | 【此刻状态】 | `:10693-10700` `statusBits` | ~80 | ✅ | 无 | 每轮变 |
| 10 | 【唤醒配置】 | `:10701-10710` `wc*` | ~120 | ✅ | 无 | 变化 |
| 11 | 黑话表 | `:9533` `withSlangContext()` | ~500 | ✅ | 无 | 罕见变 |
| 12 | 【本次消息快照】 | `:10590` `buildWakeSnapshotV2()` → `:9530` | ≤6,000 | ✅ | **是（唯一）** | 变化 |

合计典型 **≈5,050 字符/轮**。

### 1.1 已被删除的两个块（有注释留痕）

| 块 | 位置 | 删除理由（原文） |
|---|---|---|
| 反 AI 腔提示 | `:10681` `const antiAiLine = ''` | "已删：与人格卡「不是客服」重复" |
| 主动性提示 | `:10687` `const proactiveLine = ''` | "已删：与人格卡「懒猫」人设冲突（人格说她不太主动）" |

**这说明一件事**：去做"规则与人格去重"这件事在本项目**已经发生过**，且留下了代码级注释。Memory V2 的"Block Ledger 减重复"必须避开这条老路——**重复的是"同一份块被追加 20 次"，不是"不同块内容重复"**。

---

## 二、重复注入的规模（当前最大的 token 浪费）

### 2.1 机制

`api.sessions.prompt({sessionId, mode:'queue', content})`（`:9543`）把每次唤醒的 prompt **追加为会话历史里的 one 条 user 消息**。会话是长生命周期的（`ensureSession` 只在 DSH 侧丢失或策略变化时重建，`:8461-8481`）。

### 2.2 估算

一轮静态块（块 1+2+3+6+7+8+11）≈ `1,380+180+50+600+230+430+500 = 3,370` 字符。

| 会话轮次 | 静态块累计 | 其中"纯重复" |
|---|---|---|
| 5 轮 | 16,850 | 13,480 |
| 10 轮 | 33,700 | 30,330 |
| 20 轮 | 67,400 | 63,930 |
| 40 轮 | 134,800 | 131,430 |

按中文 ~1.3 字/token 粗算，**20 轮约 49k token 是纯重复**。

### 2.3 【无法确认】DSH 是否会压缩

如果 DSH 侧会自动压缩历史，实际影响会小很多。**这一点没有证据**（`项目全景` 也没有记载），列为 06 文档 F-3。

**但有一条间接证据**：`buildWakePromptV2` 每轮都重发人格全文，而项目全景 §二 阶段 2 记载"人格全文注入（原来只给一句'调 qq_get_prompt 查看'，模型不调就看不到）"是**关键突破**。这暗示：**开发者在实践中认为"每轮注入"是人格立住的必要条件**，不是冗余。

⇒ 这条直接冲击 Memory V2 的 `personaAnchorEveryTurns: 12`（见 06 文档 C-1）。

---

## 三、已有的预算机制（可复用的唯一范式）

`buildWakeSnapshotV2`，`bridge.js:10590-10641`，是全项目**唯一**带预算的 prompt 块：

```js
const maxChars = bounded('wakeMaxChars', 6000, 500, 20000);   // :10598
const tryAdd = (field, msg) => {                              // :10617-10629
  const previousThrough = packet.readThroughSeq;
  packet[field].push(compactModelMessage(msg, { now }));
  packet.includedUnreadCount = packet.messages.length;
  packet.readThroughSeq = readThroughSeqV2(st, [...packet.messages, ...packet.recent]);
  packet.partial = packet.messages.length < st.unread.length;
  if (JSON.stringify(packet).length <= maxChars) return true;
  packet[field].pop();                       // ← 回滚
  packet.includedUnreadCount = packet.messages.length;
  packet.readThroughSeq = previousThrough;   // ← 连游标一起回滚
  packet.partial = packet.messages.length < st.unread.length;
  return false;
};
```

**这个模式的三个设计要点值得复用**：

1. **装了再量、超了就回滚**（不是"先算再决定装不装"）。
2. **回滚必须连带回滚副作用**（`readThroughSeq` 是"已读位置"，回滚错会导致漏消息）。
3. **`partial` 标志显式告诉模型"不完整"**，模型可用工具补读。

Memory V2 的召回预算应直接照抄这个模式，而不是自创"先估算再装"。

---

## 四、Token 计量：已有一套实测账本（强于 Memory V2 原设计）

### 4.1 数据来源（关键）

`src/token-ledger.js`（959 行）的**权威总量不来自桥接自己累加**，而来自 DSH 的 `session/follow` 快照里的 `projections.values.tokenUsage`（`token-ledger.js:6-10`）。桥接只补"基线之后"的增量。

⇒ **这意味着"实测 token"是拿得到的**，不需要直连模型 API 读 `usage`。

### 4.2 存储

`state/token-usage.jsonl`，三种行（`:22-25`）：

```jsonc
{"k":"base", t, sid, key, seq, model, provider, b:{cacheMiss,cacheHit,cacheWrite,output,reasoning}}
{"k":"turn", t, sid, key, seq, turn, step, retry, model, provider, b:{...}}
{"k":"evict", t, sid, key, b:{...}, cost, peak, off, dropped, maxSeq}
```

### 4.3 幂等与幂等键

```
sampleKey = `${sessionId}|${turn}|${step}|${retry}`          // :41-43
覆盖规则：只有 seq **严格更新**才允许覆盖（prev.seq >= seq 则拒绝）  // :199-202
```

注释解释了为什么不能用"相等也覆盖"：重连时 snapshot 会重发同一批事件，放行相等覆盖会让旧快照把新数据改小。

### 4.4 淘汰策略（不丢总量）

超过 `maxLines=20000` 时淘汰最旧逐轮采样，但**先把桶累加进 `evicted` 汇总**（`:321-355`），并记 `maxSeq`；若后续投影基线覆盖到该 `maxSeq`，则不再叠加（`:365-371`，防重复计费 +33% 的历史 bug）。

**这是"丢明细不丢总量"的完整实现**——比 Memory V2 里"队列满丢最旧"要成熟得多。

### 4.5 可查询接口

| 方法 | 用途 |
|---|---|
| `summary({limit})` | 全局 + 每会话汇总（含 `cacheHitRate`、`avgCostPerTurn`） |
| `conversation(key,{turnLimit})` | 单会话 + 逐轮明细 |
| `series({hours,maxBuckets})` | 时间桶走势（空桶补零，桶宽阶梯 `BUCKET_LADDER_MS :52-57`） |
| `recent(limit)` | 最近采样流水 |
| `stats()` / `reset()` / `compact()` | 运维 |

调用方：控制台每 3 秒拉 `/api/tokens/summary`（`buildIndex` 已优化为单遍 O(采样数)，`:499-509`）。

### 4.6 计价

`src/model-prices.js`：峰谷档位（`isPeakAt` / `ratesAt` / `peakMultiplier` / `peakWindows` / `peakWeekdays` / `holidays`），缓存命中/未命中/写入分桶计价，模型价目表可由 `config.json:47-58` 覆盖。

---

## 五、缺口：没有"事前估算"

| 能力 | 现状 |
|---|---|
| 事后实测（本轮用了多少） | ✅ 有，且精确（DSH 投影） |
| 单轮明细 | ✅ 有（按 turn/step/retry） |
| 成本折算 | ✅ 有（含峰谷、缓存） |
| **发请求前估算"这次会花多少"** | ❌ **没有** |

原因很直接：桥接在 `deliverPromptNow` 里拼完 prompt 就直接投递，**拼之前不知道会有多长**，拼之后也不统计长度。

⇒ Memory V2 若要"按 token 预算裁剪"，需要在**拼接过程中**引入估算（因为拿不到未来的实测值）。这正是 `estimateTokens` 的合理用途：**它不是"计量"，是"预算"**。两者不能混为一谈，也不能用估算值去报告"降了 45%"。

### 5.1 校准口径（建议写入验收）

```
实际值 = token-ledger 的 turn 级 buckets（cacheMiss + cacheHit + output）
估算值 = estimateTokens(promptText)
calibrationFactor = median(实际 / 估算) 的滚动中位数
```

**已有实测源 + 缺事前估算** 这个组合，恰好是"估算器可被持续校准"的理想条件——比纯估算或纯实测都好。

---

## 六、模型请求路径

```
deliverPromptNow(key, promptText, opts)               bridge.js:9483
 ├─ fs.appendFileSync('state/prompt-dump.log', …)     :9485  ← **完整明文 prompt 落盘**
 ├─ isSessionAllowedInCurrentMode                     :9488
 ├─ dshReady? 否 → offline queued                     :9498
 ├─ ensureSession(key)                                :8510
 │    └─ ensureChatModel(existing)                    :8471  ← 会话级模型
 ├─ wakeReason → 重建 prompt + 快照                    :9521-9531
 ├─ content = [{type:'text', text: withSlangContext(promptText)}]  :9533
 ├─ media → image parts                               :9534-9537
 ├─ isCurrentSession 终校验                            :9539
 └─ api.sessions.prompt({sessionId, mode:'queue', content})  :9543
```

**串行化**：`promptQueues` 按 key 串行（`cleanupSocialForModeChange` 里可见 `promptQueues.clear()` 与逐项 `reject`，`:9411-9414`），保证同一会话不会并发投递。

**失败路径**：
- 未被接受 → `sendToQQ` 回"⚠️ 消息未被接受：{code}:{message}"（敏感则隐藏，`:9550-9552`）
- 会话创建期间被重置 → `enqueueForRetry`（`:9513`）
- DSH 未就绪 → 入 `queued`，上限 50，**进程退出即丢**（SECURITY R5）

---

## 七、结论：三件事可以直接复用，一件事必须新做

| 项 | 结论 |
|---|---|
| 预算搭载范式 | **直接复用** `buildWakeSnapshotV2` 的 tryAdd + 回滚 + partial 标志 |
| Token 实测与成本 | **直接复用** `token-ledger`，不要自建计量 |
| 落盘/压缩/幂等 | **直接复用**（`seq` 严格更新才覆盖、淘汰保总量） |
| 事前估算 | **必须新做**（`estimateTokens` + `calibrationFactor`），且必须与实测分离报告 |

**反模式警告**：Memory V2 原设计把 `estimateTokens` 写成"全项目唯一实现"，容易让人误以为它替代了 token-ledger。实际是：

```
token-ledger  = 会计（事后、精确、可审计）
estimateTokens = 预算员（事前、粗略、可校准）
```

两者不可互相替代，报告里必须分开写"预算值 / 实测值"。
