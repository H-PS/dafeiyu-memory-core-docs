# 06 · M2-a · Memory Retain（记忆形成）· 修订版

> 目标：**不是把所有聊天存下来，而是判断哪些值得记。**
> 本版相对上一版修复 P0-2：候选池按失败原因物理分离、取消一切自动晋升、embedding 失败可重试、权限失败不进普通池。

---

## 1. 三层写入来源

| 来源 | 触发 | 特点 | 保留 |
|---|---|---|---|
| L1 模型主动 | `qq_memory_append` | 有语义判断但不稳定 | 是 |
| L2 规则兜底 | Retain（本节新增） | 稳定、可预测、无成本 | 是 |
| L3 人工 | 管理端 | 最可靠 | 是 |

L2 不替代 L1，只补 L1 的空白。同一条被两者命中时按 `fingerprint` 去重。

---

## 2. 候选信号（5 类，规则）

| # | 信号 | 匹配（示意） | 强度 | category |
|---|---|---|---|---|
| 1 | 自我披露 | `我(很|超|最|有点|不太)?(喜欢|讨厌|爱吃|怕|想要|打算|准备|在学|刚开始)` | 0.9 | memberImpression |
| 2 | 稳定属性 | `我(是|在|住|来自|读)(.{2,20})(的|上学|工作|住)` | 0.8 | memberImpression |
| 3 | 承诺约定 | `(说好|约好|答应|明天|下次|周末|到时候)(.{2,30})` | 0.75 | activeTopic + pendingQuestion |
| 4 | 未完成话题 | `looksLikeUnfinished()`（bridge 已有）或句末悬置 | 0.7 | activeTopic |
| 5 | 专名首现 | 连续 ASCII ≥3 字符（STM32/CAN/PID 类）或含书名号专名，且本 key 从未出现 | 0.65 | activeTopic |

### 黑名单（命中则不产生候选）

| 类型 | 例子 | 原因 |
|---|---|---|
| 纯情绪短句 | 哈哈哈 / 草 / 6 | 无长期价值 |
| 一次性事实 | 今天下雨了 | 无跨会话价值 |
| 公共信息转述 | 新闻说… | 可搜索 |
| **凭证类** | 手机号、身份证、密码、银行卡、验证码 | 脱敏后仍不记 |
| **第三人隐私** | 非本群成员的私人信息、他人病历/住址 | 越界 |

黑名单在**进入候选之前**执行，因此这类内容不会落到任何池子里（连 quarantine 都不进，因为没有记录价值且本身是风险源）。

---

## 3. 准入四闸

```
闸1 信息量     infoOK(text)：≥6 字且 ≥3 汉字（memory-vector.js:125）
闸2 新颖度     maxCosSim(候选, 同 key 已有 vec) < 0.92
闸3 长期价值   signalStrength × 类别系数 ≥ 0.55
闸4 权限       classify(origin, text) → contentClass + scope.read
               且 scope.read 必须可推导（groupId 存在、key 合法）
```

### 3.1 闸 4 的严格性（P0-1 要求）

`classify()` 返回三态：

| 返回 | 含义 | 后续 |
|---|---|---|
| `public` + `origin_group` | 群内自产、无敏感 | 可进 A 层 |
| `sensitive` / `uncertain` + `owner_only` | 命中敏感或判不准 | **进 quarantine，不进 candidates** |
| `null`（无法推导：groupId 缺失、key 格式非法） | 权限不确定 | **进 quarantine，`classReason='ambiguous'`** |

"权限不确定"的定义很宽：只要不能 100% 确定"这条能安全地让同群的人看到"，就进 quarantine。

---

## 4. 三池分离（P0-2 核心修订）

上一版只有一个 `memory-candidates.jsonl`，且"重复 3 次自动晋升"。这等于给权限失败开了一条后门：一条敏感候选只要在群里被反复提及，就能自动进 A 层并可能群内可见。

本版改为**三个物理分离的容器**：

| 容器 | 路径 | 收什么 | TTL | 能否自动进 A |
|---|---|---|---|---|
| **A 层** | `state/memory-store.jsonl` | 四闸全过 | 永久 | — |
| **候选池** | `state/memory-candidates.jsonl` | 闸2（新颖度）失败、闸3（价值）失败 | 7 天 | **否** |
| **隔离区** | `state/memory-quarantine.jsonl` | 闸4（权限）失败、scope 无法推导 | 无（等审） | **否** |
| **补算队列** | S 层 `pendingEmbed` | embedding 失败（记录已进 A，只是无向量） | 退避 5 次 | 不涉及 |

### 4.1 候选池记录

```jsonc
{ "id": "cand_...", "key": "group:123456",
  "text": "...", "category": "activeTopic",
  "signal": "commitment", "strength": 0.75,
  "failedGate": "novelty" | "value",
  "nearestId": "mem_...", "nearestScore": 0.94,
  "seenCount": 2, "lastSeenAt": 1761900000000,
  "createdAt": 0, "expiresAt": 0,
  "origin": { "messageSeq": 88123, "sender": "...", "at": 0 } }
```

### 4.2 "重复出现"的正确语义（P0-2）

上一版：`seenCount >= 3 → 自动晋升为记忆`。**删除**。

本版：

```
每次命中同一 fingerprint 的候选：
  seenCount += 1
  若 seenCount >= 3 → 标记 reevaluate = true（在下一个空闲窗口重跑四闸）
  重跑仍不过 → 保持候选，记录本次失败原因
  重跑通过   → 正常写入 A 层（此时它已满足全部闸，包括权限闸）
```

区别在于：**重复只触发"重新评估"，不构成"通过"。** 重新评估时四闸全部重跑，闸 4 尤其不会因为"说过很多次"而放松——敏感内容说一百次仍然是敏感内容。

### 4.3 新颖度失败的正确归宿（P0-2）

若一条候选因"与已有记录太像"被拒，**不应该新增第二条**，而应该：

```
就近合并到 nearestId 指向的记录：
  target.metaRev += 1
  target.extra.mentions.push({ at: 候选.origin.at, seq: 候选.origin.messageSeq })
  // text 不变、vec 不重算（INo-8）
```

这是"重复出现"最合理的归宿：它不是新记忆，是同一件事被再次提及。上一版的"自动晋升"会把同一件事变成 N 条记录，正是要消除的重复堆积。

### 4.4 隔离区记录

```jsonc
{ "id": "quar_...", "key": "group:123456",
  "text": "...", "category": "activeTopic",
  "contentClass": "sensitive" | "uncertain",
  "classReason": "credential" | "third_party" | "address" | "medical" |
                 "finance" | "relationship" | "ambiguous",
  "signal": "...", "strength": 0.8,
  "origin": { ... },
  "status": "pending",               // pending | approved_as_private | approved_for_group | rejected
  "createdAt": 0 }
```

隔离区的行为：

- **不参与检索、不参与注入、不参与 Reflect**（`lifecycle='quarantined'` 的 `canRead` 恒 false）。
- 只能由主人通过管理端处理，三个出口：
  - `approved_as_private` → 写入 A 层，`scope.read='owner_only'`（默认推荐）
  - `approved_for_group` → 写入 A 层，`scope.read='origin_group'` + `review.status='approved'`（需主人明确提升，INo-15 唯一例外路径）
  - `rejected` → 删除隔离记录
- **没有 TTL**：敏感内容不会因为"过期"被自动清理掉而不留痕迹；也不会被自动删除（可能主人还没看过）。

### 4.5 补算队列（P0-2：embedding 失败允许重试）

embedding 失败**不阻断记忆写入**：

```
A 记录写入时 embedding.status='pending'（或 'failed'）
加入 S 层 pendingEmbed: { id, attempts, nextRetryAt }
后台每 60s 扫描，指数退避（1min, 2min, 4min, 8min, 16min），最多 5 次
成功 → vec 写入、vecContentRev=contentRev、status='ready'
5 次失败 → status='failed'，进管理端告警列表（不静默）
```

**理由**：记忆的文本价值与向量价值是两回事。Ollama 挂了 1 小时，这 1 小时的记忆不应该全部丢失——文本先落下，向量后补。

---

## 5. 归一化与去重

```
原文
 → 去 CQ 码（\[CQ:[^\]]*\]）
 → 脱敏（redactKnownTokensOnly）
 → 语气词/重复标点归一（哈+→哈，。+→。）
 → 去首尾空白与引号
 → 截断 ≤1000 字
 → fingerprint = sha256(category + ':' + normalize(text)).slice(0,16)
```

| 级别 | 依据 | 动作 |
|---|---|---|
| 精确 | `fingerprint` 相同 | 不新增，`metaRev++` + `extra.mentions.push` |
| 语义 | cosSim ≥ 0.92 | **不新增**，合并到最近记录（§4.3） |

**注意**：这里的 0.92 与 Reflect 的"完全重复"阈值 0.99 不同。Retain 阶段是"防新增"，允许宽一些；Reflect 阶段是"改 lifecycle"，必须严。

---

## 6. 来源与时间

每条 A 记录必须带 `origin`（SCHEMA §1）：

```jsonc
"origin": { "channel":"qq_group", "groupId":"123456", "messageSeq":88123,
            "sender":"...", "at":1761900000000, "trigger":"retain_rule:commitment" }
```

用途：① 时间检索 ② Reflect 的证据链（`evidence[].memId` + `quote` 需要回溯原文）③ 纠错定位 ④ 审计"这条是群内自产还是私聊外泄"。

`origin.at` 与记录级 `time` 的区别：前者是**事件发生时间**，后者是**写入时间**（兼容旧数据保留）。时间检索优先用 `origin.at`（04 §6.1）。

---

## 7. 权限推导（唯一入口）

```js
// memory-permission.js
export function classify(origin, text) {
  const cls = detectContentClass(text);          // public | sensitive | uncertain
  const groupId = origin?.groupId ?? null;

  if (origin?.channel === 'qq_group' && groupId) {
    return { contentClass: cls,
             read: cls === 'public' ? 'origin_group' : 'owner_only',
             originGroupId: groupId,
             classReason: cls === 'public' ? 'none' : detectReason(text) };
  }
  if (origin?.channel === 'qq_private') {
    return { contentClass: cls, read: 'owner_only', originGroupId: null,
             classReason: cls === 'public' ? 'none' : detectReason(text) };
  }
  return null;        // 无法推导 → quarantine
}
```

未知来源一律返回 `null`（fail-closed），与 `ensureSession` 的 preset 校验（`bridge.js:8490-8498`）同一思路。

---

## 8. 与现有写入路径的衔接

```
消息入库后（不阻塞投递）
   │
   └─► retainScan(key, newMsgs)   异步，setImmediate 排队
         ├─ 黑名单过滤（纯 CPU）
         ├─ 信号匹配（纯 CPU，<1ms/条）
         ├─ 闸1/闸3（纯 CPU）
         ├─ 闸4 classify（纯 CPU）→ 失败即 quarantine，不再继续
         ├─ 闸2 需要 embedding → 失败则按 §4.5 处理（记录仍进 A，标记 pending）
         └─ 全过 → 投递 WriteQueue（与工具写入同队列、同锁）
```

铁律：

1. Retain **必须异步**，绝不能出现在回复主链路。
2. Retain 失败（异常）**不得影响回复**，只记审计。
3. 写入必须走 WriteQueue，不得直接 append 文件（INo-1）。

---

## 9. 反模式

| 反模式 | 为什么不行 |
|---|---|
| 每条消息都存一条 | A 层爆炸，检索被噪音淹没 |
| LLM 判断"值不值得记" | 每条一次模型调用，成本延迟不可接受 |
| 同步抽取 | embed 30s 超时拖死回复 |
| 把群友转述当事实 | 应 `epistemic='inference'` / `assertion.sourceType='third_party'` |
| 写入失败静默 | 必须审计 + 模型侧可感知（工具返回 `ok:false`） |
| **"说过 3 次"就当重要** | 重复 ≠ 重要；只触发重新评估 |
| **敏感内容在池子里等过期** | 敏感内容进 quarantine，无 TTL，只等人工 |

---

## 10. 配置

```json
"retain": {
  "enabled": true,
  "signals": { "selfDisclosure": true, "stableAttribute": true, "commitment": true,
               "unfinished": true, "properNoun": true },
  "blacklist": { "emotionOnly": true, "oneOff": true, "credentials": true, "thirdParty": true },
  "gates": { "minLength": 6, "minHan": 3, "noveltyThreshold": 0.92, "minValueScore": 0.55 },
  "pools": {
    "candidateTtlDays": 7,
    "candidateReevaluateAtSeen": 3,
    "quarantineTtlDays": 0,
    "autoPromote": false
  },
  "embedding": { "retryMax": 5, "retryBaseMs": 60000, "retryFactor": 2 },
  "async": true
}
```

`pools.autoPromote` 固定为 `false`，且**不接受配置改为 true**（不是参数，是机制约束）。同理 `async` 不允许设为 false。

---

## 11. 验收（M2 部分）

| # | 验收项 | 判据 |
|---|---|---|
| M2-R1 | 自我披露必被记 | "我特别讨厌香菜" → A 层出现记录，`origin.trigger` 含 `selfDisclosure` |
| M2-R2 | 情绪短句不入库 | "哈哈哈"/"6"/"草" → 三池均无新增 |
| M2-R3 | 不无差别保存 | 200 条无信号闲聊 → A 层新增 ≤10 条 |
| M2-R4 | 语义重复不新增 | 同一件事 3 种说法 → A 层只 1 条，第 2/3 次只 `metaRev++` + `mentions` |
| M2-R5 | 来源可回溯 | 任一记录用 `origin.messageSeq` 能查到原消息 |
| M2-R6 | 群聊默认可见 | group key 写入 public 内容 → 同群检索立即可命中 |
| M2-R7 | 私聊不外泄 | private key 写入 → 任何 group key 检索均不命中 |
| M2-R8 | 敏感进隔离区 | 含手机号的群消息 → 进 quarantine，A 层无新增，`classReason='credential'` |
| M2-R9 | 隔离区不参与检索 | quarantine 有 100 条时，召回结果不含任何隔离项 |
| M2-R10 | 重复不晋升 | 敏感候选被提及 5 次 → 仍在 quarantine，A 层无新增 |
| M2-R11 | 新颖度失败不是新增 | 重复内容 5 次 → A 层 1 条，`mentions.length==4` |
| M2-R12 | embedding 失败不丢记忆 | 关掉 Ollama 写入 3 条 → A 层有 3 条（`status='pending'`）；恢复后自动补算为 `ready` |
| M2-R13 | 黑名单前置 | 含 18 位身份证 → 三池均无记录（连 quarantine 都没有） |
| M2-R14 | Retain 不影响回复 | 让 embed 超时，回复延迟增量 < 50ms |
