# 07 · M2-b · Memory Reflect（记忆整理）· 修订版

> 硬约束（任务书原话）：**禁止未经验证就让自动归纳结果覆盖原始事实。**
> 本版相对上一版修复 P0-3：取消单链聚类批量 supersede、`proposed` 默认不进召回、自动晋升检查独立证据、观察不得引入无支撑信息、偏好加维度/极性/反向/矛盾、统一断言模型。
> 本版另加一条权限约束：**Reflect 没有删除权**（SCHEMA §10）。

---

## 1. 五个整理动作

| # | 动作 | 风险 | 改原始 text？ | 改 lifecycle？ | M2 必做 |
|---|---|---|---|---|---|
| A1 | 完全重复识别 | 低 | 否 | 是（→ superseded） | 是 |
| A2 | 过时识别 | 低 | 否 | 是（→ stale） | 是 |
| A3 | 稳定偏好提取 | 中 | 否 | 否（新增记录） | 是 |
| A4 | 观察归纳 | 中高 | 否 | 否（新增记录） | 是 |
| A5 | 断言类型判定与校正 | 中 | 否 | 否（只改 `assertion`，`metaRev++`） | 是 |

**没有 A6"删除"**。Reflect 的生命周期写入白名单：`{stale, superseded, proposed}`。写 `deleted` 一律拒绝（代码层校验），这是"归纳不覆盖事实"的权限级实现。

---

## 2. 运行形态

### 2.1 进程内空闲调度（理由不变）

`writer.lock` 由 bridge 全程持有 → 外部脚本运行期无法写 A 层 → Reflect 必须在 bridge 进程内。

```
idleScheduler（60s 检查）
  条件：无在途投递 ∧ 队列空 ∧ 距上次 > minIntervalMs ∧ 该 key 新增 ≥ minNewMemories
  执行：reflectBatch(key)，单次 ≤200ms / ≤50 条，游标持久化
```

外部 `scripts/memory-maintenance.mjs reflect` 仅在 bridge 停止时用。

### 2.2 单次预算

| 项 | 值 |
|---|---|
| 单次时长 | ≤200ms |
| 单次记录数 | ≤50 |
| 并发 key | 1（串行） |
| LLM 调用 | 默认 0 |

`setInterval` 必须 `unref()`。

---

## 3. A1 · 完全重复识别（P0-3 核心修订）

### 3.1 上一版的问题

上一版用"贪心单链聚类（cosSim ≥ 0.92）"把整簇标 `superseded`。

单链聚类的已知缺陷是**链式漂移**：

```
A ≈ B (0.93)   B ≈ C (0.93)   C ≈ D (0.93)   A ↔ D (0.71)
→ 单链把 A/B/C/D 连成一簇，一次操作把 4 条互不相同的记录一起 supersede
```

对话数据里，链式漂移非常常见（围绕同一主题的多条不同事实，两两相似度都在 0.9 上下）。上一版的设计会**成片误伤**，且 `superseded` 不进召回 → 表现为"整理之后什么都想不起来了"。

### 3.2 修订：三档判定，取消批量

| 档 | 条件（**两者同时满足**） | 动作 |
|---|---|---|
| **完全重复** | ① `normalizeForDup(text)` 完全相同 ② cosSim ≥ 0.99 | 保留 `rev` 最高者为主记录，其余 → `superseded` + `duplicateOf` |
| **可能重复** | `0.92 ≤ cosSim < 0.99` | **不动 lifecycle**，写 `extra.possibleDuplicate[]` + 进审核队列 |
| **仅主题相似** | `0.65 ≤ cosSim < 0.92` | 不处理（留给 A4 归纳） |

`normalizeForDup`：去标点、去空白、去数字（时间数字视为噪声）、小写、去分类前缀。

**关键约束**：

1. **一次只处理一对**（不是一簇）。每轮 `reflectBatch` 最多执行 20 对待确认的完全重复。
2. 比较对象**只允许是"主记录"**（`lifecycle='active'` 且 `duplicateOf===null`），不允许与已被 supersede 的记录再比（否则会形成传递链）。
3. 每次 supersede 写审计：`{op:'dedup', primary, superseded, similarity, normalizedEqual:true}`。

### 3.3 为什么不干脆不做 A1

完全重复在真实数据里确实会发生（用户重复说同一句话、模型重复记录），不管会造成 A 层虚增与检索并列。A1 保留，但把风险收敛到"两个条件同时满足 + 一次一对"，误伤概率极低。

---

## 4. A2 · 过时识别

| 类型 | 过时条件 | 动作 |
|---|---|---|
| `activeTopic` | 距 `origin.at` > 30 天且无后续 `mentions` | `lifecycle='stale'` |
| `memberImpression` | 距 `lastSeenAt` > 90 天 | **不标 stale**（印象是长期属性） |
| `kind=observation` | 其 `evidence` 全部处于 `superseded/deleted` | `stale`（结论悬空） |
| 任何 | 与更新记录矛盾（cosSim > 0.9 ∧ 一边含否定词） | 标 `extra.conflictWith`，进审核；**不改 lifecycle** |

`stale` 不是删除：

- 仍参与召回（lifecycle 权重 0.6），可被时间检索显式命中；
- 再次被提及时自动回 `active`；
- 管理端可见。

**判断**：过时识别最坏结果是"该注入的没注入"，而不是"数据丢了"。这是低风险操作的定义。

---

## 5. A4 · 观察归纳（最高风险，约束最多）

### 5.1 归纳门槛（全部满足）

```
1. 同 subjectId 或同 category 下，有 ≥3 条 kind=fact 记录
2. 两两 cosSim ≥ 0.65（同主题）
3. 时间跨度 ≥ 3 天（跨 ≥2 个自然日）
4. 无 conflictWith 关系
5. 全部记录 canRead（同 key，且不是 sensitive/uncertain —— 群聊敏感内容不参与归纳）
```

第 3 条是核心：**"模式"必须跨时间**。一天内说三次"我饿了"不叫饮食偏好。

第 5 条是本版新增：敏感内容连归纳都不参与，避免"由 3 条敏感事实归纳出一条敏感观察"扩大暴露面。

### 5.2 两种归纳方式

**档 1（默认，规则版，零成本）**

不生成自由文本，用固定模板 + 槽位填充：

```
模板：{主体} 多次提到「{共同关键词}」相关（{N} 次，最近 {日期}）
槽位来源：主体 = assertion.assertedBy 或 subjectId
         共同关键词 = 所有 evidence 文本中出现的公共 bigram（取 IDF 最高者）
         N = evidence.length
         日期 = max(evidence.at) 的本地日期
```

产出：

```jsonc
{ "kind": "observation", "category": "topic",
  "text": "群友A 多次提到「开发板」相关（3 次，最近 2026-10-28）",
  "assertion": { "epistemic": "inference", "sourceType": "ai_inference",
                 "assertedBy": "群友A", "confidence": 0.6,
                 "evidence": [
                   { "memId":"mem_a", "quote":"我在看那块最小系统板", "at":... },
                   { "memId":"mem_b", "quote":"STM32 比 Arduino 难一点", "at":... },
                   { "memId":"mem_c", "quote":"板子还没买", "at":... } ] },
  "lifecycle": "proposed" }
```

**档 2（可选，LLM 版，默认关）**

把 3-8 条 evidence 原话交给廉价模型，要求输出 ≤60 字观察 + 必须引用的原话。开启条件 `reflect.synthesis.llmSynthesis = true`。

默认关的理由：成本、不可复现（两次归纳结果不同，无法测试）、倾向过度概括。

### 5.3 生成后校验：观察不得引入无支撑信息（P0-3 核心）

**这是本版新增的强制关卡**，在写入 A 之前执行：

```js
function verifyObservation(text, evidence, templateSlots) {
  // 1) 提取观察文本中所有"内容性"汉字 bigram
  const contentBigrams = tokenize(text).filter(t => t.startsWith('c:'));
  const slotBigrams = new Set(tokenize(Object.values(templateSlots).join(' ')));
  const evidenceText = evidence.map(e => e.quote).join('');

  // 2) 每个内容 bigram 必须出现在：证据原文 ∪ 模板固定文本
  const unsupported = contentBigrams.filter(b =>
    !slotBigrams.has(b) && !evidenceText.includes(b.slice(2))
  );

  // 3) 任一 bigram 无支撑 → 拒绝生成
  if (unsupported.length > 0) {
    audit('observation_rejected', { text, unsupported: unsupported.slice(0,10) });
    return null;      // 不写 A 层
  }

  // 4) 数字与专名单独校验：观察里的每个数字/ASCII 词必须出现在证据里
  const nums = (text.match(/\d+|[a-zA-Z][a-zA-Z0-9]{2,}/g) || []);
  for (const n of nums) if (!evidenceText.toLowerCase().includes(n.toLowerCase())) {
    audit('observation_rejected_number', { n, text }); return null;
  }
  return text;
}
```

**判定原则**：宁可不生成观察，也不生成含幻觉的观察。这条校验对档 1（模板）几乎总能通过，对档 2（LLM）是必需的闸门。

### 5.4 `proposed` 不进召回与注入（P0-3 核心修订）

上一版：`proposed` 降权 0.6 注入 + 标注"（我的印象）"。

本版：**完全不进召回、不进注入**（INo-16）。

理由：未经审核的归纳一旦进入上下文，模型会把它当既有认知使用，事后无法区分"我知道"与"我猜的"。降权与标注只是降低概率，不解决问题。

升级路径（任一）：

| 路径 | 条件 |
|---|---|
| 自动 | `evidence.length >= 5` **且** 独立来源 ≥3（不同 `memId`）**且** 跨 ≥2 自然日 **且** 生成后 48 小时无 `conflictWith` |
| 人工 | 主人在管理端批准 |

升级时：`lifecycle: proposed → active`、`review.status: pending → approved`、`assertion.confidence` 提到 0.75（上限）。

**代价**：归纳的价值要等升级后才体现。这是有意的取舍。

### 5.5 观察与事实的关系（INo-12）

- 观察**引用**事实，不**替代**事实。
- 删除全部 observation 后，事实问答必须仍然正确（回归测试）。
- 观察到其 `evidence` 的引用永不被回收；若某个 evidence 被删，观察的 `evidence` 保留 `memId` 但 `quote` 可能失配 → 观察降为 `stale`（`evidence` 失效）。

---

## 6. A3 · 偏好提取（P0-3 修订：维度、极性、反向、矛盾）

### 6.1 上一版的缺失

上一版只有 `dimension` 与 `text`，无法表达"后来不喜欢了"，也无法处理"既说喜欢又说讨厌"。而偏好恰好是最容易被模型直接用于行为的记忆类型，错误代价最高。

### 6.2 修订后的偏好结构

```jsonc
{ "kind": "preference", "category": "preference",
  "text": "偏好：回复要短，别绕",
  "extra": {
    "dimension": "reply_style",        // 白名单，见下
    "polarity": "positive",            // positive | negative
    "subject": "回复风格",
    "negatedFrom": null,               // 若本条是对旧偏好的反向，填旧 id
    "conflictWith": [],                // 与之矛盾的偏好 id
    "hitCount": 3
  },
  "assertion": { "epistemic": "opinion", "sourceType": "self_statement",
                 "assertedBy": "[OWNER_QQ]", "confidence": 0.6, "evidence": [...] },
  "lifecycle": "proposed" }
```

### 6.3 维度白名单（封闭，不允许自由扩展）

| dimension | 含义 | 判定关键词示例 |
|---|---|---|
| `reply_style` | 回复长度/风格 | 短点/别啰嗦/详细点/说人话 |
| `topic_interest` | 话题兴趣 | 喜欢/爱/有意思/想聊 |
| `schedule` | 作息与活跃时间 | 几点/熬夜/早起/上班 |
| `address_style` | 称呼偏好 | 别叫/叫我 |
| `dislike` | 明确不喜欢 | 讨厌/烦/受不了/别 |

白名单之外的内容不提炼成 preference，只留在 fact 层。这是"控制复杂度"的具体做法。

### 6.4 极性（polarity）

| 表达 | polarity |
|---|---|
| "我喜欢短回复" | `positive` on `reply_style` |
| "我讨厌长篇大论" | `negative` on `reply_style` |

同一 dimension 的 `positive` 与 `negative` 可能语义等价（"喜欢短" ≡ "讨厌长"），因此**不做自动合并**，只记录；注入时按 dimension 取"最近一条 + 置信度最高"的组合。

### 6.5 反向表达（negatedFrom）

检测："其实我不喜欢了" / "现在不喜欢了" / "改主意了" / "没那么喜欢了"。

```
若新偏好与旧偏好同 dimension、同 subject，且新证据时间晚于旧 ≥3 天：
  → 新偏好 negatedFrom = 旧偏好 id
  → 旧偏好 lifecycle = 'superseded'（supersededBy = 新 id）
```

**注意**：这是本设计里唯一允许"用新记录取代旧记录"的场景，且是**偏好**不是事实。理由：偏好的本质是"当前状态"，旧状态确实不再成立。即使如此，旧记录仍保留（superseded 可查），不删除。

### 6.6 矛盾（conflictWith）

若同一 dimension 出现两条极性相反的偏好，且都未过期：

- 两条都保留，`conflictWith` 互指；
- `confidence` 各降 0.1；
- 注入时**不选边**，用中性表述："他对回复长度的偏好不太一致（有时想短，有时想详细）"；
- 进审核队列等主人裁决。

**不自动裁决**是硬规则。模型没有能力判断"人是不是真的改主意了"。

---

## 7. A5 · 断言类型与统一数据模型（P0-3 修订）

### 7.1 统一到 `assertion`（SCHEMA §1）

上一版把 `epistemic` 平铺在记录上，与 `sourceType` 分离，导致"谁说的"和"可不可信"无法一致表达。本版统一：

```jsonc
"assertion": {
  "epistemic": "fact" | "inference" | "opinion",
  "sourceType": "self_statement" | "third_party" | "ai_inference" | "ai_opinion" | "manual",
  "assertedBy": "<QQ号或群友名>",
  "confidence": 1.0,
  "evidence": []
}
```

### 7.2 判定规则（确定性，无 LLM）

| 来源 | 文本特征 | epistemic | sourceType | confidence |
|---|---|---|---|---|
| 本人明确陈述 | 无推测词 | `fact` | `self_statement` | 1.0 |
| 本人陈述带主观词（我觉得/应该/可能/大概） | 有推测词 | `opinion` | `self_statement` | 0.6 |
| 他人转述 | "他说…" / 第三人称 | `fact`（转述是事实）但 `sourceType=third_party` | `third_party` | 0.8 |
| AI 归纳产品 | — | `inference` | `ai_inference` | ≤0.75 |
| AI 主观评价 | — | `opinion` | `ai_opinion` | ≤0.6 |
| 管理端录入 | — | `fact` | `manual` | 1.0 |

**"他人转述"的处理**：内容本身是事实（"他确实这么说了"），但信息的可靠性取决于被转述者 → 用 `sourceType` 而非降级 `epistemic` 表达。注入时标注"（据说）"。

### 7.3 校正

`assertion` 变更只 `metaRev++`，不重算向量（INo-8）。校正来源：主人否定、后续事实推翻、管理端修改。

---

## 8. 审核队列

`proposed` 与 `conflict` 与 `quarantine` 都进 `S.reviewQueue`，但**类型不同、处置不同**：

| type | 来源 | 默认是否注入 | 主人可做什么 |
|---|---|---|---|
| `observation` | A4/A3 | 否 | 批准（→active）/ 驳回（→superseded）/ 编辑 |
| `conflict` | A2/A3 | 两者都不注入该维度 | 选一个为准 / 都保留 |
| `quarantine` | Retain 闸4 | 否 | 批准为私有 / 批准为群可见 / 驳回 |

**审核不是可选项**：`proposed` 没有别的路径变成 `active`（除 §5.4 的自动条件）。

---

## 9. 调度与成本

| 项 | 值 |
|---|---|
| 检查频率 | 60s |
| 单次时长 | ≤200ms |
| 单次条数 | ≤50 |
| 全量整理（bridge 停止） | 10k 条约 40-60 秒 |
| LLM 调用 | 默认 0 |
| 常态 CPU | <1% |

---

## 10. 配置

```json
"reflect": {
  "enabled": true, "idleScheduler": true,
  "minIntervalMs": 21600000, "minNewMemories": 10,
  "batchSize": 50, "budgetMsPerRun": 200,

  "dedup": { "mode": "exact_only", "exactCosMin": 0.99,
             "possibleRange": [0.92, 0.99], "pairsPerRun": 20 },
  "stale": { "topicDays": 30, "impressionDays": 0 },
  "conflict": { "enabled": true,
                "negationWords": ["不","没","别","停止","取消","不再","换","改","其实","反而"] },

  "synthesis": { "enabled": true, "minEvidence": 3, "minTimeSpanDays": 3,
                 "maxConfidence": 0.75, "llmSynthesis": false,
                 "verifyBigrams": true, "autoActivateAfterProofs": 5,
                 "autoActivateQuietHours": 48 },

  "preference": { "enabled": true, "minOccurrences": 3, "minSpanDays": 7,
                  "dimensions": ["reply_style","topic_interest","schedule","address_style","dislike"],
                  "negationMinGapDays": 3 }
}
```

`dedup.mode` 只接受 `exact_only`；`possible` 档只写标记不改状态。若未来要开启自动合并，必须先改本文档与 11 文档的验收项，**不允许只改配置**。

---

## 11. 验收（M2 部分）

| # | 验收项 | 判据 |
|---|---|---|
| M2-F1 | 完全重复合并 | 2 条归一化文本相同的记录 → 1 active + 1 superseded；原 text 均可读 |
| M2-F2 | 不批量误伤 | 造 6 条链式相似（两两 0.93）→ **0 条被 supersede**，只标 possibleDuplicate |
| M2-F3 | Reflect 无删除权 | 代码审查 + 运行：Reflect 期间 `lifecycle='deleted'` 数量不变 |
| M2-F4 | 不删事实 | 整理前后 A 层总条数只增不减 |
| M2-F5 | 过时不删 | 40 天前 activeTopic → `stale`，仍能被时间检索命中 |
| M2-F6 | 观察有证据 | 每个 `evidence[].quote` 是原文连续子串；`memId` 均存在 |
| M2-F7 | 观察无幻觉 | 注入 5 段文本，生成观察；观察中每个 bigram/数字/专名都能在 evidence 找到 |
| M2-F8 | 观察被拒即不写 | 构造会引入新词的 LLM 输出 → `verifyObservation` 返回 null，A 层无新增 |
| M2-F9 | proposed 不注入 | 100 条 proposed 存在时，正常对话注入中 0 条 |
| M2-F10 | 独立证据检查 | 3 条同一天、同一段对话的证据 → 不升级为 active |
| M2-F11 | 偏好极性 | "我讨厌长篇" → `polarity=negative`；与"我喜欢短回复"不自动合并 |
| M2-F12 | 反向表达 | 先"喜欢短"，3 天后"现在不喜欢了" → 新偏好 `negatedFrom` 指向旧，旧 `superseded` |
| M2-F13 | 矛盾不选边 | 造极性冲突 → 两条都保留、`conflictWith` 互指、注入用中性表述 |
| M2-F14 | 敏感内容不归纳 | 3 条 sensitive 记录 → 不产生 observation |
| M2-F15 | 幂等 | 同一批数据连续整理两次，第二次 0 新增 |
| M2-F16 | 不抢 CPU | 整理运行时发起 10 次对话，P99 增量 < 100ms |
