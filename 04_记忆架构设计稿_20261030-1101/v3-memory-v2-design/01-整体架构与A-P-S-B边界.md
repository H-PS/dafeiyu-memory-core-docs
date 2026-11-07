# 01 · 整体架构与 A / P / S / B 边界

> 本文回答 R2 的三个问题：B 与 P 是不是一回事、每个字段归谁、什么能重建什么不能。

---

## 一、四个概念的精确定义

| 代号 | 定义 | 存在形态 | 可重建性 | 谁写 |
|---|---|---|---|---|
| **A** | **断言级**长期记忆。一条 = 一个可独立引用的陈述（"她在学 STM32"） | `state/memory-store.jsonl`（Rev3 新增；现为 `memory-vectors.jsonl`） | **不可重建**（原始事实的唯一副本） | `memory-store.commit()` 单写者 |
| **P** | **派生视图**。为"注入"与"查询"服务，可丢弃可重算 | `state/projection/*.json` | **半可重建**（见 §3） | 投影器（只读 A + S） |
| **S** | **运行时状态**。会话、游标、唤醒配置、B 层业务记忆 | `state/social-v2.json` + 若干专用文件 | **不可重建**（进程状态与业务记忆的唯一副本） | 各领域单一写者（见 `08`） |
| **B** | **业务级记忆概念**。进行中的话题 / 想说没说 / 对某人的印象 | **当前住在 S 里**（`social-v2.json`） | **不可重建** | `appendMemoryV2()` 等 B 层函数 |

### 1.1 B 与 P 的关系（Rev2 在这里错了）

```
Rev2 的假设（错）：
   A ──投影──► P（含 B）──► 注入
   ⇒ "P 可从 A 重建"     ❌ B 的字段在 A 里没有对应粒度

Rev3 的事实：
   A ──投影──► P.memoryView      （可从 A 完整重建）
   S.B ──投影──► P.runtimeView   （只能从 S 重建；S 不可重建 ⇒ 这一段不可从 A 恢复）
```

**证据**：

- A（`memory-vector.js:155-158`）一条记录 = `{key, text, time, vec, …}`，**没有** `lastMentionAt`、`participants`、`pendingQuestion`、`expiresAt`、`motivation`、`interactionCount` 这些字段。
- B（`bridge.js:9227-9232` / `9243-9248` / `9258-9262`）这些字段只存在于 S。
- 两者之间**没有同步机制**（`项目全景` §8.3 坑 3 原文：「B 层 ≠ A 层，没有桥」）。

⇒ **B 不是 A 的投影，B 是 S 里的独立业务真相。** Rev3 不再声称"P 整体可重建"。

### 1.2 一条硬规矩

> 任何文档/代码/验收，凡写"P 可从 A 重建"，必须限定为 **`P.memoryView`**。
> `P.runtimeView` 的重建源是 S，而 S 不可重建 ⇒ 丢了就是丢了。

---

## 二、`social-v2.json` 字段逐个定性（R2 要求）

> 说明：**完整字段清单未核实**（U-8）。下表只覆盖**已在源码中被读/写到的字段**，其余标 `【UNCONFIRMED】`。
> 「去向」列是 Rev3 的设计决定，理由见 `08`。

| 字段 | 语义角色 | 权威来源 | 可重建 | 生命周期 | 写者（现状） | 去向 |
|---|---|---|---|---|---|---|
| `agentToken` | 会话凭据 | S（本字段） | ❌ | 会话级；`retireSession` 重置（`bridge.js:2795`）、`/reset` 清除（`:11308`） | retireSession / reset | **外移** `state/agent-token.json` |
| `bootstrapSent` | 首轮引导是否已发 | S | ❌ | 会话级；退役时置 false（`:2797`） | retireSession | **留 S** |
| `modelSeenSeqs` | 已见模型消息游标（Set） | S | ❌ | 会话级；退役时清空（`:2798`） | retireSession | **留 S** |
| `unread[]` | 未读消息队列 | S | ❌ | 消息级；mark-read 推进 | 消息入口 / mark-read | **留 S** |
| `recentMessages[]` | 最近消息环形缓冲 | S | ❌ | 滚动（`recentLimit`） | 消息入口 | **留 S** |
| `activeTopics[]` | **B**：进行中的话题 | S | ❌ | 24h 未提及自动移除（`:9213-9215`）；上限 20 | `appendMemoryV2` | **留 S（B 层）**，进 `P.runtimeView` |
| `pendingThoughts[]` | **B**：想说未说 | S | ❌ | `expiresAt` 到期移除（`:9210-9212`）；上限 20 | `appendMemoryV2` | **留 S（B 层）** |
| `memberImpressions{}` | **B**：对群友的印象 | S | ❌ | 长期；`lastSeenAt` 更新 | `appendMemoryV2` | **留 S（B 层）** |
| `memberImpressions[*].interactionCount` | 互动计数 | S（错位） | ❌ | 每轮 +1（`:9260`） | `appendMemoryV2` | **外移** `state/impression-stats.json` |
| `wakeConfig` | 下一次唤醒配置 | S | ❌ | 每次 set_wake_config | tool / 状态机 | **留 S** |
| `phase` / `activeDeadlineAt` / `activeExitAt` | 社交阶段机 | S | ❌ | 秒级变动（`enterActive :9365`） | `socialLoopTick` | **留 S** |
| `lastIncomingAt` / `lastAiReplyAt` | 时间戳 | S | ❌ | 每次消息/回复 | 消息入口 | **留 S** |
| `clearedKeys[]`（Rev2 提出） | 清空墓碑 + epoch | S | ❌ | 长期 | `clear` | **外移** `state/memory-epochs.json` |
| `promptLedger`（Rev2 提出） | 块账本 | S | ⚠️ 可丢（丢了只是重发一次） | 会话级 | `prompt-ledger.js` | **外移** `state/prompt-ledger.json` |
| `rotationTx`（Rev2 提出） | 轮转事务日志 | S | ❌ 不可丢 | 事务级 | 轮转流程 | **外移** `state/rotation-journal.json` |
| `reviewQueue`（Rev2 提出） | 待审队列 | S | ❌ 不可丢 | 长 | 审核流 | **外移** `state/review-queue.jsonl` |
| `pendingEmbed`（Rev2 提出） | 待补算向量 | S | ⚠️ 可丢（可全量重扫） | 短 | store | **外移** `state/pending-embed.jsonl` |
| 其余字段 | — | — | — | — | — | `【UNCONFIRMED】`，迁移前须产出完整清单 |

### 2.1 从这张表能读出三件事

1. **S 里有三件不同性质的东西混在一起**：纯运行态（phase/游标）、业务记忆（B 层）、凭据（token）。它们的写频、写者、安全性要求完全不同。
2. **B 层字段生命周期短且可变**（话题 24h 过期、上限 20、想法 2h 过期）——它们是"工作记忆"，不是"长期记忆"。这解释了为什么不能把它们当 A 的投影：A 的语义是"永久事实"，B 的语义是"当前还在进行的事"。
3. **`interactionCount` 是唯一"每轮都写、且写在整文件里"的字段**——这是它必须外移的直接理由（S-1）。

---

## 三、可重建性分级（R2 的硬要求）

| 级别 | 定义 | 成员 | 丢失后果 | 恢复手段 |
|---|---|---|---|---|
| **L0 不可重建** | 没有第二份副本，删了就是没了 | `memory-store.jsonl`（A）、`social-v2.json` 的 B 段与运行态、`agent-token.json`、`review-queue.jsonl` | 永久丢失 | **只能备份恢复** |
| **L1 可从 A 重建** | 输入只有 A | `P.memoryView`（记忆视图、向量索引、BM25、时间桶） | 检索能力暂时退化 | 全部重扫 A，10k 条约 40-60 秒 |
| **L2 只能从 S 重建** | 输入含 S | `P.runtimeView`（B 层快照、digest 的对话部分） | 注入内容暂时变少 | 从 S 重算；**S 若已丢，此层无法恢复** |
| **L3 可丢弃** | 缓存/账本，丢了只影响体验或观测 | `prompt-ledger.json`、`pending-embed.jsonl`、投影缓存 | 重发一次 / 重扫一次 | 重建 |

### 3.1 三处必须写进代码注释的判定

```
memory-index/（向量/BM25/时间桶）  → L1：只读 A
projection/memoryView             → L1：只读 A
projection/runtimeView            → L2：读 S 的 B 段（不是 A）
digest                            → L2：读 A（话题相关）+ S.recentMessages   ← 双源！
```

> **digest 是唯一的双源产物**：`activeTopics`（S）与"最近 6 条消息"（S）都是 S；A 只参与"话题文本"的措辞。实际上 digest ≈ 纯 S 派生。Rev2 把它写成"从 A + S 派生"是对的，但必须注明：**A 只贡献文本，不贡献结构**。

---

## 四、数据流（Rev3）

```
                    ┌─────────────────────────────┐
   5 条写入路径 ───► │ memory-store.commit(op)      │ ← 唯一写入口（R9）
                    │  · 单写者 · per-key 串行     │
                    │  · journal + 幂等重放        │
                    └──────────┬──────────────────┘
                               │ 只写 A
                               ▼
                    ┌─────────────────────┐
                    │ A: memory-store.jsonl│  L0
                    └──────────┬──────────┘
                               │ 派生（异步，可滞后）
              ┌────────────────┼──────────────────┐
              ▼                ▼                  ▼
      ┌──────────────┐ ┌──────────────┐  ┌──────────────┐
      │ index（L1）   │ │ memoryView   │  │ recall 结果   │
      │ vec/bm25/时间 │ │ （L1）        │  │ （纯内存）    │
      └──────────────┘ └──────────────┘  └──────┬───────┘
                                                 │
    ┌─────────────────────────────┐              │
    │ S: social-v2.json（L0）      │              │
    │  · B 层（话题/想法/印象）      ├──► runtimeView（L2）
    │  · 运行态（phase/游标/唤醒）   │              │
    └─────────────────────────────┘              │
                                                 ▼
                                     ┌──────────────────────┐
                                     │ buildWakePromptV2    │ 唯一注入点
                                     │  · persona（每轮全文）│
                                     │  · credential（独立） │
                                     │  · runtimeView        │
                                     │  · recall 结果        │
                                     └──────────────────────┘
```

**关键约束（INo）**：

| 编号 | 不变量 |
|---|---|
| INo-1 | 只有 `memory-store.commit()` 能写 A |
| INo-2 | 索引/投影只能读，不能写 A |
| INo-3 | `runtimeView` 不得被当作"A 的视图"使用（它没有 A 血缘） |
| INo-4 | 索引可滞后、不可领先、删除必须即时 |
| INo-5 | 凭据状态只由凭据模块决定，Ledger 无权判断有效性 |

---

## 五、Persona / 规则 / 记忆 / 凭据 的四分（R3 落点）

| 维度 | Persona | 规则（preset） | 记忆（A + B） | 凭据 |
|---|---|---|---|---|
| 放在哪 | `roles/*.md` | `dsh/agent-presets/*` → patch | A 层 + S 的 B 段 | `agent-token.json` |
| 注入频率 | **每轮全文**（R3 固定） | preset 常驻，不进 prompt | 每轮（预算内） | **每轮**（见 `04` §4） |
| 谁决定失效 | 用户改卡 | 重启 DSH | 生命周期规则 | `retireSession` / `/reset` |
| 能否进记忆库 | ❌ | ❌ | — | ❌ |
| Ledger 能优化 | **不能**（R3） | 部分（睡眠规则等） | 结果条数 | **不能**（R4） |

### 5.1 为什么人格必须每轮（证据，不是偏好）

| 证据 | 内容 |
|---|---|
| `bridge.js:10671` | 每轮 `roleLine` 含 `currentRoleHintV2()` 全文 |
| `项目全景` 阶段 2 | 「人格**全文注入**（原来只给一句"调 qq_get_prompt 查看"，模型不调就看不到）」被列为迁移成功的**五个关键突破之一** |
| `config.json:13` | `role.maxInjectChars: 6000`，为长人格卡预留空间 |
| `SECURITY_BASELINE` §五.5 | 改动检查单要求"新增定时器/Map 在退役与退出路径清理"——说明本项目对"状态漂移"极敏感 |

⇒ **减弱人格注入是一次已验证失败路线的回退**。Rev3 明确禁止在本版改默认值；任何降频须走 `09` §5 的 A/B 协议。

---

## 六、与 Rev2 的差异汇总

| Rev2 | Rev3 | 依据 |
|---|---|---|
| "P 可从 A 重建" | `memoryView` 可，`runtimeView` 不可 | R2 |
| B 当作 P 的一部分 | B 是 S 的一部分 | R2 |
| Ledger 管 7 块含人格 | Ledger 管 4 块，**人格与凭据除外** | R3/R4 |
| `personaAnchorEveryTurns: 12` 默认 | **不启用** | R3 |
| 凭据进 `credential` 块、由 Ledger 判有效 | 凭据独立模块 + 强制重发 | R4 |
| `interactionCount` 拆 `memory-stats.js` | 并入 `state/impression-stats.json`（不单独成模块） | R4 精简 |
| 14 文件 / 4,470 行 | 9 文件 / ≈3,320 行 | R4 |
| `appendVectorMemory` 改为 throw | 改为**兼容适配器**（不 throw） | R9 |
