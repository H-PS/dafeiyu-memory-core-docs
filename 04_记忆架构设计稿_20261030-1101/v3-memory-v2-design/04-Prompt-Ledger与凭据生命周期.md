# 04 · Prompt Ledger 与凭据生命周期

> 对应 R3（人格频率）、R4（凭据）、R6（dump 安全）。
> 三者的共同点：**都属于"往 prompt 里放东西"，但失效条件完全不同**。

---

## 一、现状 12 个块（`buildWakePromptV2`，`bridge.js:10669`）

| # | 块 | 位置 | 典型字符 | 每轮 | 有预算 | 失效条件 |
|---|---|---|---|---|---|---|
| 1 | **人格卡全文** | `:10671` | ~1,380 | ✅ | `role.maxInjectChars=6000` | 用户改卡（**即时生效**） |
| 2 | 你的身份 | `:10672` | ~180 | ✅ | — | 昵称/QQ 变化（罕见） |
| 3 | **会话令牌** | `:10674` | ~50 | ✅ | — | `retireSession` / `/reset` |
| 4 | B 层记忆 | `:10675` | 200–5,000 | ✅ | ❌ | 每轮可变 |
| 5 | 参与度 | `:10679` | ~200 | ✅ | — | 每轮可变 |
| 6 | 表情策略+清单 | `:10683-85` | ~600 | ✅ | 只限条数 | 收藏库变化 |
| 7 | 首回复节奏 | `:10689` | ~230 | ✅ | — | quietMs 每轮变 |
| 8 | 沉睡前等待 | `:10690` | ~430 | ✅ | — | preSleepMs 配置变 |
| 9 | 此刻状态 | `:10693` | ~80 | ✅ | — | 每轮变 |
| 10 | 唤醒配置 | `:10701` | ~120 | ✅ | — | 每轮变 |
| 11 | 黑话表 | `:9533` | ~500 | ✅ | — | 学到新词 |
| 12 | 消息快照 | `:10590` | ≤6,000 | ✅ | ✅ 唯一 | 每轮变 |

**结论**：只有块 6/7/8/11 是"规则类、变化慢"的候选；块 1/2/3 有硬性的"必须每轮"或"必须独立"理由。

---

## 二、块分级（R3 落点）

### 2.1 四级分类

| 级别 | 块 | 策略 | 依据 |
|---|---|---|---|
| **L-LOCKED（永不优化）** | 1 人格卡 | **每轮全文注入** | R3 强制 |
| **L-SECRET（独立管线）** | 3 会话令牌 | 由 `credential.js` 管理，**不经 Ledger 判定** | R4 强制 |
| **L-LEDGER（可优化）** | 6 表情清单 / 8 睡眠规则 / 11 黑话表 / 7 首回复规则 | 按 hash + 锚点重发 | 本设计 |
| **L-DYNAMIC（每轮必发）** | 2 身份 / 4 B 层 / 5 参与度 / 9 状态 / 10 唤醒配置 / 12 快照 | 每轮，受预算约束 | 内容本身在变 |

### 2.2 为什么人格不能进 Ledger（证据，不是偏好）

| 证据 | 内容 |
|---|---|
| `【源码】bridge.js:10671` | 每轮 `roleLine` 含完整 `currentRoleHintV2()` |
| `【审计】项目全景` 阶段 2 | 「人格**全文注入**（原来只给一句"调 qq_get_prompt 查看"，模型不调就看不到）」= 迁移成功的五个关键突破之一 |
| `【审计】项目全景` §7.1-3 | 「身份锚点是人格的头号杀手」——相反方向的证据：任何"少给一点"都会被别的文本压过 |
| `config.json:13` | `role.maxInjectChars: 6000`，为长卡预留空间而非限制 |

**Rev3 明确**：
- `personaAnchorEveryTurns` 参数**不引入**（Rev2 的 12 是错的）。
- 任何人要降频，必须先完成 `09` §5 的 A/B 协议，且结论要能证明"40 轮盲评人格一致性不下降"。
- **在拿到那个结论之前，默认行为就是每轮全文。**

### 2.3 可优化块的收益（诚实计算）

| 块 | 字符 | 变化频率 |
|---|---|---|
| 表情清单 | ~600 | 收藏库变化时（天级） |
| 黑话表 | ~500 | 学到新词时（天级） |
| 睡眠规则 | ~430 | 配置变化（很少） |
| 首回复规则 | ~230 | 配置变化（很少） |
| **合计可优化** | **≈1,760** | — |
| 不可动（人格+身份+令牌） | ≈1,610 | — |

⇒ 单轮最多省 **≈1,760 字符（≈1,350 token 估算值）**，**不是** Rev2 声称的"3,370 字符 / 4,700 token"。

**但首回复规则块里含动态值 `quietMs`**：它是每轮变的，所以第 7 块只能"规则段 ledger 化 + 数值段每轮"，实际可省约 180 字符（230 → 50）。修正后合计可省 **≈1,710 字符**。

---

## 三、Ledger 结构

```jsonc
// state/prompt-ledger.json（独立文件，非 social-v2.json）
{
  "version": 1,
  "keys": {
    "group:123456": {
      "sessionId": "sess_abc",
      "sessionEpoch": 7,              // 与 S 层 sessionEpoch 对齐；不一致即视为失效
      "turn": 23,
      "blocks": {
        "stickerList": { "hash": "a1b2c3", "sentTurn": 20, "sentAt": 1761900000000 },
        "slangTable":  { "hash": "d4e5f6", "sentTurn": 18, "sentAt": 1761900000000 },
        "sleepRules":  { "hash": "112233", "sentTurn": 1,  "sentAt": 1761900000000 },
        "replyRules":  { "hash": "445566", "sentTurn": 1,  "sentAt": 1761900000000 }
      },
      "injectedMemo": [ { "fp": "m_9f2a", "turn": 12 } ]   // 召回去重用（07 文档）
    }
  }
}
```

### 3.1 重发条件（三条，缺一不可）

```js
function shouldSendFull(rec, hash, turn, epoch, curEpoch) {
  if (curEpoch !== epoch) return true;              // ① 会话代际变了
  if (!rec) return true;                            // ② 从没发过
  if (rec.hash !== hash) return true;               // ③ 内容变了
  if (turn - rec.sentTurn >= ANCHOR) return true;   // ④ 锚点到了（防被压缩挤掉）
  return false;
}
```

| 参数 | 默认 | 依据 |
|---|---|---|
| stickerList 锚点 | 0（不重发，只按 hash） | 表情清单丢了影响小，且每轮有"策略提示"兜底 |
| slangTable 锚点 | 15 轮 | 黑话表消失会让模型听不懂梗 |
| sleepRules 锚点 | 20 轮 | 规则消失会导致"该睡不睡" |
| replyRules 锚点 | 0（只按 hash） | 数值部分每轮都发，规则段丢了影响有限 |

### 3.2 失效矩阵（必须逐条测试）

| 场景 | 期望 | 若不做会怎样 |
|---|---|---|
| 用户改了人格卡 | 下一轮立即生效 | **人格块不在 Ledger 里，天然满足** |
| 用户在控制台加了表情 | 下一轮重发清单 | 模型看不到新表情 |
| 黑话表学到新词 | 下一轮重发 | 新词听不懂 |
| preset 改了睡眠规则 | **需要重启 DSH 才生效** | 若只改 patch 不重启，Hash 变了但浏览器里没生效 → 提前重发一次（无害） |
| 会话轮转（`retireSession`） | `sessionEpoch` 变化 → 全部重发 | 新会话没有规则，行为崩 |
| bridge 重启 | ledger 从磁盘读，`sessionEpoch` 连续 → 不重发 | 若 DSH 侧会话丢了呢？→ 见下 |
| **DSH 侧会话丢失（`not-found` 分支）** | **必须强制清该 key 的 ledger** | 新会话里模型看不到规则，且不会重发（因为它以为发过了） |

> 最后一行是 `【源码】bridge.js:8471-8476` 的既有分支，Rev3 必须在其中插入 `promptLedger.forget(key)`。这是本项目"清了映射但忘了清账本"的典型陷阱（`forgetConversationCaches :2752` 的注释正是在讲这类慢性泄漏）。

### 3.3 Ledger 的能力边界（必须写进文档，防误解）

| Ledger 能做 | Ledger 不能做 |
|---|---|
| 减少未来重复追加 | 移除已经写进会话历史的内容 |
| 降低追加速率 | 压缩历史（DSH 侧行为） |
| 让新会话首轮"按需补齐" | 修改已追加消息 |

**唯一的"清历史"手段是会话轮转**，代价是信息损失。

---

## 四、凭据生命周期（R4）

### 4.1 现状的三条证据

| 事实 | 位置 |
|---|---|
| 令牌每轮写入 prompt | `bridge.js:10674` `const tokenLine = \`【会话令牌】${st.agentToken}…\`` |
| 令牌由模型作为工具参数传入 | `mcp-snowluma-safe.js` 40+ 处 `token: z.string().describe('会话令牌（见唤醒提示中的【会话令牌】）')` |
| `retireSession` 换新令牌 | `bridge.js:2793-2797`：`st.agentToken = crypto.randomBytes(24)…`；`rememberAgentToken`；`bootstrapSent=false`；`modelSeenSeqs.clear()` |
| `/reset` 用另一条路径 | `:11308` `forgetAgentToken(key)` —— **清除**而非换新 |

⇒ **两条退役路径的令牌处理不一致**（换新 vs 清除），这个不一致本身是风险。

### 4.2 Rev3 的决定：独立模块 + 强制交付

```
state/agent-token.json           ← 独立文件，0600
{
  "version": 1,
  "keys": {
    "group:123456": {
      "token": "…", "epoch": 7, "issuedAt": 0,
      "needsDelivery": true,       // 必须在本轮 prompt 里出现
      "rotatedBy": "retireSession" | "reset" | "init",
      "rotatedAt": 0
    }
  }
}
```

**规则**：

1. **Ledger 无权判断凭据有效性**。凭据块永远每轮发送（与现状一致），Ledger 不参与。
2. `retireSession` / `/reset` 必须调 `credential.rotate(key, by)`：
   - `retireSession` → 生成新 token，`epoch++`，`needsDelivery=true`
   - `/reset` → 同 `retireSession`（**Rev3 统一为"换新"而不是"清除"**，理由：清除会让下一次工具调用用旧 token 失败且无法自愈）
3. `buildWakePromptV2` 从 `credential.get(key)` 取 token，**不从 S 层读**。
4. `needsDelivery` 为 true 时，本轮 prompt 必须包含完整凭据块；成功后置 false 并记审计 `credential_delivered`。
5. 若 `needsDelivery` 连续 3 轮仍为 true（说明投递一直失败）→ 告警。

### 4.3 设计 A vs 设计 B（R4 要求评估两者）

| 维度 | **A：prompt 重发** | **B：带外绑定** |
|---|---|---|
| 机制 | 沿用现状：每轮 prompt 写 token | 模型永不接触 token；MCP 按 key 从文件/环境/握手取 |
| 依赖 | 无 | **F-1：MCP 进程模型**（`【UNCONFIRMED】`） |
| 改动量 | ≈+80 行（模块化 + 统一两条路径） | ≈+300 行（40+ schema 删参数 + 三选一取据 + TTL） |
| 安全属性 | 中（token 在 prompt 里，可能被注入诱导泄露；但有 `redactKnownTokensOnly` 兜底） | 高（模型拿不到） |
| 失败模式 | 模型忘传/传错 → 403（可自愈：下一轮重发） | 若绑定写错 → 全工具不可用（不可自愈） |
| **本版决定** | **✅ 采用** | **⛔ 不采用，标 `blocked-by-F-1`** |

**为什么不能选 B**（R4 明确要求）：MCP 是否由 bridge spawn、是否一进程一会话，**没有任何证据**（`【审计】06-F-1`）。如果按 B 实施而实际是"一进程多会话"，那么"用环境变量注入 per-session 凭据"根本不可行，方案要重做。**拿未验证的假设当实施方案，是把风险转嫁给未来。**

### 4.4 轮转序列（原子 + 可恢复）

```
rotationTx（state/rotation-journal.json）
phase: prepare → digestSaved → drained → rotated → done

1. phase=prepare（原子写）
2. 生成 digest（只读 A 的 topic + S 的 recentMessages）
3. 写 projection.digest（幂等）
4. 排空在途投递（≤5s；超时则回退 phase=digestSaved，30s 后重试，最多 3 次 → 放弃本次轮转）
5. credential.rotate(key,'retireSession') + retireSession(key)
       ← 顺序重要：先 rotate 再 retire，避免 retire 内部直接改字段的旧逻辑覆盖
6. ledger.forget(key)（epoch 变化会天然触发全部重发，这里显式清更稳）
7. 下一轮投递：凭据块（needsDelivery）+ digest + 全部规则块
8. phase=done，删除 tx
```

**恢复语义**：启动时若 tx 未完成，从当前 phase 续跑（每步幂等）。`digest` 是确定性函数，重算结果一致；`credential.rotate` 若已执行，`epoch` 已变，重跑会再 `epoch++`——因此 `rotate` 必须接受 `opId` 幂等（同 opId 只递增一次）。

### 4.5 403 的处理（R4 要求"显式处理与审计"）

| 情形 | 现状 | Rev3 |
|---|---|---|
| 模型没传 token | 端点 403（`bridge.js:6867`） | 不变 |
| token 与 key 不匹配 | 403 | 不变 |
| token 已被轮转（旧值） | 403 | **新增**：错误码可区分 `reason: 'stale_token'`，并返回 `{retryAfterNextPrompt:true}` |
| MCP 收到 403 | 返回 `isError:true` + 通用文案 | **新增**：明确告知模型"凭据已失效，请先调用 `qq_get_prompt` 取最新令牌后重试"，同时写审计 `tool_403_stale` |
| 连续 403 ≥3 次 | 无感知 | 告警 + 控制台红点 |

**审计字段**：`{op:'tool_call', key, tool, code:403, reason, tokenEpochSeen, tokenEpochCurrent}`。
注意：审计里**不记 token 明文**，只记 epoch。

### 4.6 为什么"清了 Ledger"不等于"解决了凭据问题"

R4 原文要求不得声称这点。原因：

- Ledger 清空只保证"下一轮会重发凭据块"。
- 但**本轮**（轮转发生的那个瞬间）模型手里还是旧 token；若它在同一轮里调工具，仍会 403。
- 而且轮转是异步的，可能与一次在途工具调用并发。

⇒ 必须靠 `§4.5 的 403 显式处理` 兜底，而不是靠 Ledger。

---

## 五、Hash 与"生效时机"的对应（易错点）

| 内容源 | 生效方式 | Hash 计算依据 | 误判后果 |
|---|---|---|---|
| 人格卡 `roles/*.md` | 保存即生效 | （不在 Ledger 内） | — |
| 表情库 | 保存即生效 | `stickerEntries` 的 id 列表 hash | 少发一次清单 |
| 黑话表 | 保存即生效 | 词条数组 hash | 新词听不懂 |
| 睡眠/回复规则 | **改配置即生效** | 配置值 hash | 少发一次规则 |
| preset 规则 | **需重启 DSH** | patch 文件 hash | 若只改文件未重启，hash 变 → 提前重发（无害） |

**规则**：Hash 一律基于"**实际注入内容的字符串**"，不基于"配置来源"。这样无论上层怎么变，只要注入内容变了就重发——最不容易错。

---

## 六、预算与总闸（保持不变，只改块归属）

| 块 | 预算 | 超预算策略 |
|---|---|---|
| 人格 | 3,000 硬上限（`role.maxInjectChars` 另有 6,000） | 截断 + 告警 |
| 凭据 | 80 | 不截断 |
| B 层 | 800 | topics > thoughts > impressions |
| 召回 | 600 token | 两遍装载（`06` §5） |
| 表情+参与度 | 700 | 表情最多 8 个 |
| 规则块 | 700（重发轮） | — |
| 快照 | 6,000 | 已有 tryAdd + 回滚 |

总闸砍序（固定，规则与人**格不砍**）：

```
snapshot → recall → bMemory → sticker → 仍超则审计 warn（不阻断投递）
```

---

## 七、`prompt-dump.log` 安全（R6）

### 7.1 现状问题

```js
// bridge.js:9485-9487
fs.appendFileSync(path.join(STATE_DIR, 'prompt-dump.log'),
  '\n\n========== ' + new Date().toISOString() + '  ' + key + ' ==========\n' + promptText + '\n');
```

| 风险 | 说明 |
|---|---|
| 含令牌 | `promptText` 含【会话令牌】块 ⇒ **明文凭据落盘** |
| 含私聊内容 | 私聊会话的 prompt 里有 B 层记忆、消息快照 |
| 含群内容 | 群里的话题、群友发言 |
| 无保留策略 | 只追加，不轮转，不清理 |
| 无访问控制 | 仅靠 `state/` ACL（R6 明确：**ACL 不等于解决**） |
| 无脱敏 | 出站有 `redactKnownTokensOnly`，但落盘没有 |

### 7.2 Rev3 方案：默认元数据 dump

**默认（`promptDump.mode = 'meta'`）**，写 `state/logs/prompt-dump/YYYY-MM-DD.jsonl`：

```jsonc
{
  "ts": 1761900000000,
  "key": "group:123456",
  "sessionId": "sess_abc",
  "reason": "wake:new_message",
  "blocks": [
    { "name": "persona",     "chars": 1380, "hash": "a1b2c3", "sent": true },
    { "name": "credential",  "chars": 50,   "hash": "—",      "sent": true, "redacted": true },
    { "name": "bMemory",     "chars": 620,  "ids": 7 },
    { "name": "recall",      "chars": 540,  "ids": ["mem_a","mem_b"], "tokens": 410 },
    { "name": "snapshot",    "chars": 2400, "msgs": 8 }
  ],
  "totalChars": 5050,
  "estTokens": 3800,
  "dropped": []
}
```

**可选（`promptDump.mode = 'full'`，默认关闭）**：写全文，但**强制删除 credential 块**（替换为 `【会话令牌】<REDACTED>（长度 N）`），且需显式配置 + 启动时告警。

| 维度 | 规定 |
|---|---|
| **凭据** | **任何模式都不得落明文**。实现上在 dump 前对 credential 块做硬替换（不是"匹配正则"，是按块名替换），并有单测断言 `dump 输出中不含当前 token` |
| 记忆正文 | 默认不落（只落 `ids` + `chars`）；`full` 模式落，但仅限 `agent` 会话 |
| 私聊内容 | `full` 模式下私聊 dump 单独目录 `prompt-dump/private/`，权限更严 |
| 群内容 | 同上，`prompt-dump/group/` |
| 保留 | 按天轮转，保留 **7 天**，总量上限 **50MB**（超限删最旧） |
| 访问 | 目录 0700 + 文件 0600；控制台不提供下载 |
| 备份 | **不纳入** `dailyBackup`；备份脚本显式排除 `logs/` |
| 诊断导出 | 任何导出（排障时打包）必须过 `redactKnownTokensOnly` + 移除 credential 块，并记审计 `dump_export` |

### 7.3 为什么这不是"调试偏好"

- 凭据明文落盘 = 一旦 `state/` 被同机其他用户读到，等于拿到该会话的工具调用权（`SECURITY_BASELINE` §三 R2 已把"state 里令牌是明文"列为残余风险）。
- 记忆正文落盘 = 一份没有权限模型的记忆副本，绕过了 `canRead` 的全部设计。
- 52 周不清理 = 磁盘与检索面无限增长。

**结论**：dump 是"旁路数据出口"，必须与主数据同等对待，而不是"反正只是日志"。

---

## 八、三个必须回答的"如果"

| 问题 | 答案 |
|---|---|
| 如果 DSH 侧压缩了历史，规则块被挤掉怎么办？ | 锚点重发（sticker 不设、slang 15、sleep 20）；人格不在 Ledger 内所以不受影响。**但压缩比未知（F-3）**，锚点值是初值，需实测调。 |
| 如果 bridge 重启，Ledger 会不会误判"已发过"？ | 不会：ledger 持久化在独立文件，`sessionEpoch` 连续；只有 DSH 会话丢失（`not-found`）才强制清。 |
| 如果同一 key 并发两次投递？ | `promptQueues` 已按 key 串行（`bridge.js:9411-9414`），Ledger 的 turn 递增在队列内，无竞态。 |
