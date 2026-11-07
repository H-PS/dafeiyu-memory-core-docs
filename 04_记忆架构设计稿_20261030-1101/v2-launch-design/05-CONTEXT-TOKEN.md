# 05 · 上下文管理与 Token 预算（修订版）

> 目标：**减少模型输入，不是截断上下文。**
> 本版相对上一版修复：P0-5（凭据不进 prompt、安全不靠历史注入、Ledger 能力边界、轮转原子性）与 P1-9（Digest 来源/权限/覆盖/恢复）。

---

## 1. 现状实测（`buildWakePromptV2`，`bridge.js:10669`）

| 块 | 来源 | 典型字符 | 每轮都发 | 内容变化 |
|---|---|---|---|---|
| 人格卡全文 | `currentRoleHintV2()` | ~1,380 | 是 | 几乎不变 |
| 你的身份 | `selfIdentityTextV2()` | ~180 | 是 | 不变 |
| **会话令牌** | `st.agentToken`（`:10674`） | ~50 | **是** | 罕见变 |
| B 层记忆 | `formatMemoryV2()` | 200~5,000 | 是 | 变化 |
| 参与度 | `formatParticipationV2()` | ~200 | 是 | 变化 |
| 表情策略+清单 | `buildStickerStrategyHint()` 等 | ~600 | 是 | 清单罕见变 |
| 首回复节奏规则 | 硬编码 | ~230 | 是 | 规则不变，数值变 |
| 沉睡前等待规则 | 硬编码 | ~430 | 是 | 规则不变，数值变 |
| 此刻状态 | `statusBits` | ~80 | 是 | 每轮变 |
| 黑话表 | `withSlangContext()` | ~500 | 是 | 罕见变 |
| 消息快照 | `buildWakeSnapshotV2()` | ≤6,000 | 是 | 变化 |
| **合计（典型）** | | **≈5,050** | | |

### 1.1 问题一：重复

`api.sessions.prompt({sessionId, content})`（`bridge.js:9543`）把每次唤醒的 prompt **追加为会话历史中的一条 user 消息**；会话是长生命周期的（`ensureSession` 仅在 DSH 侧丢失时重建，`:8461-8481`）。

20 轮唤醒 → 静态块累计约 `(1,380+180+50+600+230+430+500)×20 ≈ 67,400` 字符，其中 19 份是纯重复。

### 1.2 问题二：无预算

除消息快照（`wakeMaxChars: 6000`）外全部"有多少发多少"。`formatMemoryV2` 最坏 10 话题×200 + 10 想法×200 + 10 印象×70 ≈ 5,000 字符。

### 1.3 问题三：历史不可控

bridge 看不到 DSH 会话内部的 token 数，也没有轮转机制。

---

## 2. Block Ledger

### 2.1 数据结构（S 层，见 SCHEMA §5）

```jsonc
"promptLedger": {
  "group:123456": {
    "sessionId": "sess_abc", "epoch": 3,
    "blocks": {
      "persona":      { "hash":"a1b2", "sentTurn":1, "sentAt":1761900000000 },
      "identity":     { "hash":"d4e5", "sentTurn":1, "sentAt":1761900000000 },
      "credential":   { "hash":"",     "sentTurn":1, "sentAt":1761900000000 },
      "sleepRules":   { "hash":"1122", "sentTurn":1, "sentAt":1761900000000 },
      "stickerList":  { "hash":"4455", "sentTurn":1, "sentAt":1761900000000 },
      "slangTable":   { "hash":"7788", "sentTurn":1, "sentAt":1761900000000 },
      "replyRules":   { "hash":"aabb", "sentTurn":1, "sentAt":1761900000000 }
    },
    "turnsSincePersona": 1,
    "injectedMemo": [{ "fp":"m_9f2a", "turn":12, "at":1761900000000 }]
  }
}
```

### 2.2 判定

```js
const BLOCKS = {
  persona:     { anchorEveryTurns: 12, maxAgeMs: 6*3600e3 },
  identity:    { anchorEveryTurns: 0,  maxAgeMs: 0 },
  credential:  { anchorEveryTurns: 0,  maxAgeMs: 0 },   // 见 §3，目标态将移除
  sleepRules:  { anchorEveryTurns: 20, maxAgeMs: 24*3600e3 },
  stickerList: { anchorEveryTurns: 0,  maxAgeMs: 0 },
  slangTable:  { anchorEveryTurns: 15, maxAgeMs: 12*3600e3 },
  replyRules:  { anchorEveryTurns: 0,  maxAgeMs: 0 },
};

function shouldSendFull(ledger, name, hash, turn) {
  const rec = ledger.blocks[name], cfg = BLOCKS[name];
  if (!rec) return true;                                  // 会话首次
  if (rec.hash !== hash) return true;                      // 内容变了
  if (cfg.anchorEveryTurns && turn - rec.sentTurn >= cfg.anchorEveryTurns) return true;
  if (cfg.maxAgeMs && Date.now() - rec.sentAt >= cfg.maxAgeMs) return true;
  return false;
}
```

三种重发理由缺一不可：内容变（用户改了卡）、锚点到（防 DSH 压缩挤掉）、超时（跨天语义断）。

### 2.3 **Ledger 的能力边界（P0-5 明确要求）**

> **Block Ledger 只能减少"未来的重复追加"，不能从已经存在的会话历史里移除任何内容。**

这是硬限制，必须写进文档与验收，不能被当成"清缓存"：

| Ledger 能做 | Ledger 不能做 |
|---|---|
| 后续轮次不再重复发同一块 | 删除前 20 轮已经写进历史的 20 份人格卡 |
| 减少新增长度 | 压缩历史（DSH 侧行为，bridge 无权干预） |
| 降低追加速率 | 修改已追加消息的内容 |

**唯一能"清掉历史"的手段是会话轮转**（§4），而且轮转的代价是摘要带来的信息损失。这一点在 11 #11 的测试设计里必须体现：Ledger 的收益曲线是"从第 2 轮开始每轮省 2,600 字符"，不是"历史瞬间变短"。

### 2.4 一行锚点

不重发人格卡时仍发一行：

```
【你】你是「<人格卡名字>」，按人格卡说话，别变成客服。
```

约 25 字符。成本可忽略，但显著降低"长会话人格逐渐回到通用助手"的概率。

---

## 3. 凭据不进 prompt（P0-5 核心修订）

### 3.1 源码事实（已核验）

- `bridge.js:10674`：`tokenLine` 每轮写入 prompt。
- `mcp-snowluma-safe.js`：40+ 工具的 zod schema 把 `token` 声明为参数，描述统一为"见唤醒提示中的【会话令牌】"（`:587 :603 :926 :958 :985 :1064` 等）。
- 鉴权侧：`bridge.js:6867-6869`（`agentTokenOk` / `v2SessionAllowed` / `v2ToolEnabled`）。

**结论**：token 目前是"模型可见 + 模型传递"的显式凭据。把它从 prompt 删掉，工具调用会全部 403 —— 所以必须同时改绑定方式。

### 3.2 目标态（M1 完成）

```
模型永不接触 token；工具 schema 移除 token 参数；
MCP 侧按 key 解析凭据，来源优先级：
  ① 环境变量（若 MCP 由 bridge 以子进程启动，注入 per-session env）
  ② POST /api/agent/handshake（loopback + 进程级 secret）换取短期票据
  ③ state/agent-tokens/<hash(key)>.tok（权限 0600，仅同用户可读）
票据 TTL = min(会话 epoch 生命周期, 30 分钟)，轮转即失效
```

模型调用工具时只需 `key`，无需任何凭据。服务端把 `(key, ticket)` 绑定校验。

### 3.3 过渡态（M0 内可先落地，风险更低）

1. `token` 从每轮注入改为**会话首轮一次性注入**（纳入 `credential` 块，`anchorEveryTurns: 0`）。
2. token 与 `sessionEpoch` 绑定，轮转即换新 token（旧 token 立即失效）。
3. 服务端增加"同 token 短时间跨 key 调用"检测 → 拒绝 + 审计（越权试探特征）。

### 3.4 为什么分两阶段（不假装一步到位）

目标态需要改 40+ 个工具 schema，并依赖一个**尚未核实的事实**：MCP 进程是否由 bridge 启动、是否一进程一会话。若 MCP 由 DSH 独立拉起且一个进程服务多会话，则方案 ① 不可用，只能走 ② 或 ③，实现量差一倍。

该事实列入 `12-RISKS-PERMISSIONS.md` §6「实施前必须重新核查的源码事实」，**核查前不得按目标态排期**。

---

## 4. 安全规则不得依赖会话历史（P0-5）

### 4.1 分层原则（P9）

| 层 | 承载什么 | 是否可靠 |
|---|---|---|
| **代码层**（bridge / MCP） | 越权拦截、key 隔离、visibility 过滤、工具白名单、per-session 权限 | **是**（唯一可信层） |
| **preset 层**（system 指令） | 行为契约：不装人、不越界、不说教、工具调用纪律 | 较可靠（每会话常驻，不依赖历史） |
| **prompt 注入层** | 本轮状态、记忆、节奏提示 | 不可靠（会被压缩、会漂移） |

**规定**：

- 任何"必须永远成立"的约束（私聊内容不得出现在群聊、不得调用未授权工具、不得泄露 token），**必须在代码层实现**，prompt 里最多写一句提醒。
- 一次性注入（本轮 prompt）**不得**承载安全规则。因为它可能在第 30 轮被 DSH 压缩掉，而模型仍会继续按"最近的语气"行事。
- 规则块（`sleepRules` / `replyRules`）在 Ledger 里保留锚点重发（12/20 轮），正是为了对抗压缩——但这是**行为风格**规则，不是安全规则。

### 4.2 具体到本项目

| 约束 | 代码层实现 | prompt 里的角色 |
|---|---|---|
| 群聊不得读到私聊记忆 | `canRead` + key 严格相等 + 注入前断言 | 无需提及 |
| 不得跨群 | `originGroupId` 严格相等 | 无需提及 |
| 敏感内容群内不可见 | `contentClass` 降级 | 无需提及 |
| 不得调用未授权工具 | `v2ToolEnabled` + MCP 工具白名单（已有） | 无需提及 |
| 不得泄露 token | 目标态：模型根本拿不到 | 过渡态：一行"令牌只用于工具参数，不要复述" |
| 不得声称自己是真人 | 无代码约束（无法判定） | preset 行为契约 |

---

## 5. 会话轮转与摘要（P0-5 原子性 + P1-9 Digest）

### 5.1 为什么必须轮转

Ledger 解决"重复注入"，但每轮的真实消息、状态块、快照仍会累积。连续跑一周可达数十万 token，导致成本线性上升、早期内容稀释注意力、模型开始"记混"。

### 5.2 触发条件（与 11 #10 统一）

| 条件 | 默认阈值 | 配置 |
|---|---|---|
| 累计轮次 | 60 | `context.rotateAfterTurns` |
| 估算历史 token | 40,000 | `context.rotateAfterTokens` |
| 空闲后重新激活 | 空闲 ≥ 12h 且累计轮次 ≥ 20 | `context.rotateAfterIdleMs` |
| 手动 | 管理端 | — |

**统一口径（11 #10 要求）**：这里的"轮次"指 `sessionMeta.turns`，每次 `deliverPromptNow` 成功投递 +1；"空闲"指 `Date.now() - sessionMeta.lastDeliveryAt`；"估算历史 token"用 `estimateTokens` 累加每次投递的 prompt 长度（下界估计，不含模型输出）。

### 5.3 轮转事务（原子 + 可恢复）

上一版只写了 4 步流程，没定义失败恢复。本版引入 `rotationTx`：

```
rotationTx = { phase, startedAt, fromSessionId, attempt, digest }

phase: prepare → digestSaved → drained → retired → done

步骤（每步幂等）：
1. phase='prepare'        写 S 层
2. 生成 digest（同 key）  写入 tx.digest
3. 写 P.digest            幂等（同内容覆盖）→ phase='digestSaved'
4. 等在途投递排空         最长 5s；超时 → phase 回退 'digestSaved'，30s 后重试（attempt++）
                          连续 3 次失败 → 放弃本次轮转，记审计，不影响服务
5. retireSession(key)     → phase='retired'
6. 清 promptLedger，epoch++，下次投递注入 digest → phase='done'，删除 tx

启动恢复：若 tx 存在且 phase != 'done' → 从该 phase 继续；
          digest 是确定性函数，重算结果一致；若 tx.digest 已存在则直接复用（不重算）。
```

**关键设计**：

- 第 4 步是唯一可能阻塞的步骤，因此设超时 + 放弃机制。**宁可这次不轮转，也不能让轮转卡住在途消息。**
- 第 5 步 `retireSession` 必须在"无在途投递"窗口执行，否则会丢掉正在处理的回复。
- 第 6 步之前，digest 已持久化；即使在第 5 步后崩溃，重启后仍能拿到 digest 注入新会话。

### 5.4 Digest：来源、权限、覆盖、恢复（P1-9）

```js
function buildDigest(key, A, S) {
  // 输入严格限定：同 key 的 A 层 active 记录 + 同 key 的 S.recentMessages
  // 不读其他 key、不读索引、不读 P（P 是本函数的输出目标，读了会自引用）

  const topics = A.filter(r => r.key===key && r.lifecycle==='active' && r.category==='activeTopic')
                  .sort((a,b)=> (b.extra.lastMentionAt||0)-(a.extra.lastMentionAt||0)).slice(0,3);
  const parts = [];
  if (topics.length) parts.push('之前聊到的：' + topics.map(t=>t.text).join('；'));

  const pending = topics.map(t=>t.extra.pendingQuestion).filter(Boolean);
  if (pending.length) parts.push('还没问完的：' + pending.join('；'));

  const recent = S.recentMessages.slice(-6)
                  .map(m => `${m.sender}：${String(m.text||'').slice(0,24)}`);
  if (recent.length) parts.push('最近的对话：\n' + recent.join('\n'));

  return {
    text: truncateText(parts.join('\n'), 600),
    coveredFrom: S.recentMessages[Math.max(0, S.recentMessages.length-6)]?.time ?? 0,
    coveredTo: S.recentMessages[S.recentMessages.length-1]?.seq ?? 0,
    generatedAt: Date.now()
  };
}
```

| 维度 | 规定 |
|---|---|
| **来源** | 仅同 key 的 A 层（`lifecycle='active'`）+ 同 key 的 `S.recentMessages`（最近 6 条） |
| **权限** | digest 在生成时逐条调用 `canRead`；群聊 key 的 digest **不得包含** `contentClass != 'public'` 或 `scope.read='owner_only'` 的记录 |
| **覆盖范围** | `coveredFrom` / `coveredTo` 明确记录覆盖的消息区间；**未覆盖区间不得假定已摘要**（模型可能被问到 6 条之前的细节，此时应显式说"记不清了"而不是编） |
| **长度** | ≤600 字符（`context.digestMaxChars`） |
| **恢复语义** | digest 存在 `P.digest` 与 `rotationTx.digest` 两处；轮转失败时 P 里的 digest 仍有效（下次可用）；`dirty=true` 表示 A 层已变但未重算 |
| **模式** | `deterministic`（默认，规则版，0 成本、可复现）/ `llm`（M2 可选，默认关） |

**注入格式**（新会话首轮，位于人格卡之后）：

```
【上次聊到这儿（会话已续接）】
之前聊到的：STM32 选型；漫展筹备
还没问完的：你到底买了哪块板子
最近的对话：
  主人：我看了下那个最小系统板
  你：多少钱啊
以上是上一段会话的收尾摘要，不是新消息，不要当作对方刚说的话。
```

最后一句必须有——否则模型会把摘要当新消息回复。

---

## 6. 预算分配（统一口径，P1-8）

单轮注入目标 ≤ 2,600 字符（≈1,950 tokens，按 `estimateTokens` 估算，实测以 API `usage` 校准）。

| 块 | 预算（字符） | 超预算策略 |
|---|---|---|
| 人格卡（重发轮） | 3,000 硬上限 | 截断 + 告警（用户卡片过长） |
| 一行锚点 | 40 | — |
| 身份 | 200 | 截断 |
| 凭据（过渡态） | 80 | 不截断 |
| B 层记忆 | 800 | 按 topics > thoughts > impressions 裁剪 |
| 召回结果 | 按 token：600 | 两遍装载（04 §9） |
| 参与度 + 表情 | 700 | 表情最多 8 个 |
| 规则块（重发轮） | 700 | — |
| 此刻状态 | 120 | — |
| 消息快照 | 6,000 | 已有 tryAdd + 回滚 |

总闸（固定砍序，规则与人格永不砍）：

```
snapshot → recall → bMemory → sticker → 仍超则审计 warn（不阻断投递）
```

**"不阻断投递"是刻意的**：超预算的代价是成本，不回复的代价是功能失效。宁可超，不可不回。

---

## 7. 三个边界（Persona / 规则 / 记忆）

| 维度 | Persona | 规则（preset） | 记忆 |
|---|---|---|---|
| 回答什么 | 我是谁、怎么说话 | 必须遵守什么 | 我记得什么 |
| 变更频率 | 月级 | 月级 | 分钟级 |
| 注入方式 | 首轮 + 每 12 轮锚点 | preset 常驻（不进 prompt） | 每轮（预算内） |
| 冲突谁赢 | **规则** | — | 谁都不赢：记忆是参考，不是指令 |
| 可否进记忆库 | 否 | 否 | 是（它就是记忆） |

三条硬规则：

1. 记忆永不覆盖规则（注入前缀必须写"以下是参考"）。
2. 规则不进 A 层（否则会被 Reflect 归纳、被检索注入，形成自我强化）。
3. Persona 不进记忆库（人格是配置，不是经历）。

---

## 8. 群聊/私聊隔离（与权限模型一致）

| 层 | 机制 |
|---|---|
| 存储 | `record.key === ctx.key` 严格相等（`memory-vector.js:172` 起沿用） |
| 可见性 | `canRead`：`scope.read` + `originGroupId` + `contentClass`（SCHEMA §9） |
| 摘要 | digest 生成时逐条 `canRead`；群 key 的 digest 不含敏感/owner_only 内容 |
| 注入 | `assertRecall` 五项断言（04 §10） |
| 脱敏 | 群聊场景 `origin.sender` 为主任 QQ 时显示"管理员" |

---

## 9. 配置

```json
"context": {
  "recentLimit": 100, "unreadLimit": 30, "contextWindow": 20,
  "inlineWakeMessages": true, "wakeMessageLimit": 12, "wakeRecentLimit": 4,
  "wakeMaxChars": 6000,

  "blockLedger": { "enabled": true, "personaAnchorEveryTurns": 12,
                   "sleepRulesAnchorEveryTurns": 20, "slangAnchorEveryTurns": 15,
                   "personaMaxAgeMs": 21600000 },
  "credential": { "mode": "session_once", "ttlMs": 1800000 },
  "personaMaxChars": 3000, "bMemoryMaxChars": 800, "maxPromptTokens": 6000,

  "rotateAfterTurns": 60, "rotateAfterTokens": 40000, "rotateAfterIdleMs": 43200000,
  "rotateDrainTimeoutMs": 5000, "rotateMaxAttempts": 3,
  "digestMaxChars": 600, "digestMode": "deterministic"
}
```

`credential.mode` 取值：`every_turn`（现状，不推荐）/ `session_once`（过渡态）/ `out_of_band`（目标态，依赖核查结论）。

---

## 10. 风险

| # | 风险 | 缓解 |
|---|---|---|
| C1 | 锚点不足 → 人格漂移 | 12 轮锚点 + 每轮一行锚点；11 #11 第 40 轮实测 |
| C2 | 轮转丢在途消息 | `rotationTx` + 排空超时 + 放弃机制 |
| C3 | digest 被当新消息 | 注入末句显式声明 |
| C4 | ledger 与 DSH 会话不一致 | `ensureSession` 的 `not-found` 分支强制清 ledger |
| C5 | 总闸砍掉不该砍的 | 固定砍序；规则与人格不进砍序 |
| C6 | **误以为 Ledger 能清理历史** | §2.3 明确边界；11 #11 的指标按"新增速率"而非"历史长度"设计 |
| C7 | 目标态凭据绑定依赖未核实事实 | 分两阶段；核查项进 12 文档 §6 |
