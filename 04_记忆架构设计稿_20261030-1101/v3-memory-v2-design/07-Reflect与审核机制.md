# 07 · Reflect 与审核机制

> 对应 R10。核心不变量：**Reflect 没有删除权**；**`proposed` 不进召回**；**归纳不得引入无支撑信息**。

---

## 一、五个动作与权限白名单

| # | 动作 | 改 text？ | 改 lifecycle？ | 风险 |
|---|---|---|---|---|
| A1 | 完全重复识别 | ❌ | ✅ → `superseded` | 低 |
| A2 | 过时标记 | ❌ | ✅ → `stale` | 低 |
| A3 | 偏好提取 | ❌ | ❌（新增记录） | 中 |
| A4 | 观察归纳 | ❌ | ❌（新增记录） | 中高 |
| A5 | 断言分类校正 | ❌ | ❌（只改 `assertion`，`metaRev++`） | 中 |

**代码级约束**：

```js
const REFLECT_ALLOWED_LC = new Set(['stale', 'superseded', 'proposed']);
setLifecycle(id, lc) {
  if (!REFLECT_ALLOWED_LC.has(lc)) throw new Error('reflect 无权设置该状态');
}
```

物理删除（`removeVectorMemoryExact` / `removeNAll` / `removeAll`）**不暴露给 reflect 模块**。

---

## 二、A1 · 完全重复识别（不用单链聚类）

### 2.1 为什么不聚类

单链聚类的链式漂移：

```
A≈B(0.93)  B≈C(0.93)  C≈D(0.93)   A↔D(0.71)
→ 单链把 4 条连成一簇，一次 supersede 4 条互不相同的记录
```

对话数据里"围绕同一主题的多条不同事实"两两相似度都在 0.9 上下，**这是成片误伤**。且 `superseded` 不进召回 ⇒ 表现为"整理完什么都想不起来"。

### 2.2 三档判定

| 档 | 条件（**同时满足**） | 动作 |
|---|---|---|
| 完全重复 | ① `normalizeForDup(text)` 完全相同 ② `cosSim ≥ 0.99` | 保留 `rev` 最高者为主，其余 → `superseded` + `duplicateOf` |
| 可能重复 | `0.92 ≤ cosSim < 0.99` | **不改 lifecycle**，写 `extra.possibleDuplicate[]` + 进待办 |
| 主题相似 | `0.65 ≤ cosSim < 0.92` | 不处理（留给 A4） |

`normalizeForDup`：去标点、空白、数字（时间数字视为噪声）、小写、去分类前缀。

**三条硬约束**：
1. **一次只处理一对**（每轮 ≤20 对）。
2. 比较对象只能是 `lifecycle='active'` 且 `duplicateOf===null` 的**主记录**（禁止与已 supersede 的比较，防传递链）。
3. 每次写审计 `{op:'dedup', primary, superseded, similarity, normalizedEqual:true}`。

---

## 三、A2 · 过时标记

| 类型 | 条件 | 动作 |
|---|---|---|
| `activeTopic` | `origin.at` > 30 天且无后续 mentions | `stale` |
| `memberImpression` | `lastSeenAt` > 90 天 | **不标**（印象是长期属性） |
| `kind=observation` | 其 `evidence` 全部 superseded/deleted | `stale`（结论悬空） |
| 任何 | 与更新记录矛盾（cosSim>0.9 ∧ 一侧含否定词） | 标 `extra.conflictWith` + 进待办，**不改 lifecycle** |

`stale` 仍参与召回（权重 0.6），可被时间检索命中，再次提及自动回 `active`。最坏结果是"该注入的没注入"，不是"数据丢了"。

---

## 四、A4 · 观察归纳

### 4.1 门槛（全满足才做）

```
1. 同 subject 或同 category 有 ≥3 条 kind=fact
2. 两两 cosSim ≥ 0.65（同主题）
3. 时间跨度 ≥ 3 天（跨 ≥2 个自然日）
4. 无 conflictWith
5. 全部记录 canRead，且 contentClass='public'
      ← 敏感/不确定内容连归纳都不参与，避免"由 3 条敏感事实归纳出 1 条敏感观察"扩大暴露面
```

### 4.2 模板化生成（默认档）

```
模板：{主体} 多次提到「{共同关键词}」相关（{N} 次，最近 {日期}）
共同关键词 = evidence 文本中 IDF 最高的公共 bigram
```

```jsonc
{ "kind":"observation", "category":"topic",
  "text":"群友A 多次提到「开发板」相关（3 次，最近 2026-10-28）",
  "assertion": { "epistemic":"inference", "sourceType":"ai_inference",
                 "assertedBy":"群友A", "confidence":0.6,
                 "evidence":[{ "memId":"mem_a", "quote":"我在看那块最小系统板", "at":… }, …] },
  "lifecycle":"proposed" }
```

### 4.3 生成后校验：`verifyObservation`（不可跳过）

```js
function verifyObservation(text, evidence, slots) {
  const contentBigrams = tokenize(text).filter(t => t.startsWith('c:'));
  const slotSet = new Set(tokenize(Object.values(slots).join(' ')));
  const evidenceText = evidence.map(e => e.quote).join('');

  const unsupported = contentBigrams.filter(b =>
    !slotSet.has(b) && !evidenceText.includes(b.slice(2)));
  if (unsupported.length) { audit('observation_rejected', { unsupported }); return null; }

  const nums = text.match(/\d+|[a-zA-Z][a-zA-Z0-9]{2,}/g) || [];
  for (const n of nums) if (!evidenceText.toLowerCase().includes(n.toLowerCase())) {
    audit('observation_rejected_number', { n }); return null;
  }
  return text;
}
```

**原则**：宁可不生成，也不生成含幻觉的观察。

**必须承认的取舍**：这条校验让"观察"只能是**统计 + 套模板**，不是"理解"。任何自然概括（"他在认真准备比赛"）都会被拒。这是用归纳能力换零幻觉，**必须在验收时明确接受**（`11` D-30）。

### 4.4 `proposed` 不进召回、不进注入

```js
// canRead 里已硬编码
if (record.lifecycle === 'proposed') return false;   // 含 admin 也只在管理端列表可见
```

代价：归纳价值要等升级后才体现。**有意为之**——未审核的归纳一旦进入上下文，模型会把它当既有认知，事后无法区分"我知道"与"我猜的"。

### 4.5 升级条件（任一）

| 路径 | 条件 |
|---|---|
| 自动 | `evidence.length ≥ 5` ∧ 独立 `memId` ≥3 ∧ 跨 ≥2 自然日 ∧ 48h 内无 `conflictWith` |
| 人工 | 管理端批准 |

升级时：`lifecycle: proposed → active`，`review.status: pending → approved`，`confidence` 提到 0.75（上限）。

**"独立"的可执行定义**（`SCHEMA` 级）：
- `evidence[].memId` 互不相同；
- `new Set(evidence.map(e => new Date(e.at).toDateString())).size >= 2`；
- 所有 `memId` 的 `origin.messageSeq` **不是连续区间**（同一段对话连说三句不算三个来源）【推断：用 seq 跨度近似判断】。

---

## 五、A3 · 偏好提取

```jsonc
{ "kind":"preference", "category":"preference",
  "text":"偏好：回复要短，别绕",
  "extra": {
    "dimension":"reply_style",     // 封闭白名单 5 个
    "polarity":"positive",          // positive | negative
    "subject":"回复风格",
    "negatedFrom":null,             // 反向时指向旧偏好 id
    "conflictWith":[],              // 矛盾偏好 id
    "hitCount":3
  },
  "assertion":{ "epistemic":"opinion", "sourceType":"self_statement",
                "assertedBy":"[OWNER_QQ]", "confidence":0.6, "evidence":[…] },
  "lifecycle":"proposed" }
```

### 5.1 维度白名单（不可自由扩展）

`reply_style` / `topic_interest` / `schedule` / `address_style` / `dislike`

白名单之外不提炼成 preference，只留在 fact 层。

### 5.2 极性 / 反向 / 矛盾

| 情形 | 处理 |
|---|---|
| "我喜欢短回复" | `positive` on `reply_style` |
| "我讨厌长篇" | `negative` on `reply_style`（**不与上一条自动合并**，语义等价但不推断） |
| "现在不喜欢了"（≥3 天后） | 新偏好 `negatedFrom` 指向旧；旧 → `superseded` |
| 同维度极性相反且都未过期 | 两条都保留，`conflictWith` 互指，`confidence` 各降 0.1，**注入时不选边**，进待办 |

**这是唯一允许"新记录取代旧记录"的场景，且仅限偏好**（事实永不由此机制取代）。即使如此，旧记录仍保留（superseded 可查）。

---

## 六、A5 · 断言统一模型

```jsonc
"assertion": {
  "epistemic": "fact" | "inference" | "opinion",
  "sourceType": "self_statement" | "third_party" | "ai_inference" | "ai_opinion" | "manual",
  "assertedBy": "<QQ号或群友名>",
  "confidence": 1.0,
  "evidence": []
}
```

| 来源 | 特征 | epistemic | sourceType | confidence |
|---|---|---|---|---|
| 本人明确陈述 | 无推测词 | `fact` | `self_statement` | 1.0 |
| 本人带主观词 | 我觉得/应该/可能 | `opinion` | `self_statement` | 0.6 |
| 他人转述 | "他说…" | `fact`（转述是事实） | `third_party` | 0.8 |
| AI 归纳 | — | `inference` | `ai_inference` | ≤0.75 |
| AI 评价 | — | `opinion` | `ai_opinion` | ≤0.6 |
| 管理端 | — | `fact` | `manual` | 1.0 |

**他人转述的处理**：内容本身是事实（"他确实这么说了"），可靠性取决于被转述者 → 用 `sourceType` 表达，注入时标注"（据说）"。

`assertion` 变更只 `metaRev++`，不重算向量。

---

## 七、审核机制（R10：语义分离，不合并状态机）

### 7.1 三类待办，语义不同

| type | 来源 | 默认可见性 | 可做的处置 |
|---|---|---|---|
| `share` | 现有分享申请 | 记录仍 `owner_only` | 批准（→ `origin_group`）/ 驳回 |
| `quarantine` | Retain 闸4 失败 | 记录在隔离区，不可读 | 批准为私有 / 批准为群可见（默认关闭）/ 驳回 |
| `observation` | A4/A3 产出 | `proposed`，不进召回 | 批准（→active）/ 驳回 |
| `conflict` | A2/A3 检出矛盾 | 两条都保留，不注入该维度 | 选一个为准 / 都保留 / 驳回 |

**为什么不能合并成一个 `approved` 布尔**：

- `share` 批准的是**跨范围可见**（`scope.read` 变化）；
- `quarantine` 批准的是**是否允许留存**；
- `observation` 批准的是**是否可进上下文**；
- `conflict` 根本没有"批准"概念，只有"裁决"。

四者的状态机不同，合并后必然出现"点了同意但语义不明"的情况（`【审计】06-C-9`）。

### 7.2 但 UI 应统一

一个"待处理"列表，按 `type` 分组/筛选，每行显示：类型、`contentClass`、来源群、摘要（≤40 字）、创建时间、操作按钮。

**复用现有面板**：控制台已有"记忆分享审核"面板（v23），扩展其数据源为统一 `review-queue.jsonl` 即可，不需要新建页面。

### 7.3 队列文件

```jsonc
// state/review-queue.jsonl（append + 定期 compact）
{ "id":"rev_1", "type":"observation|quarantine|share|conflict",
  "memId":"mem_x", "relatedIds":[], "at":0, "status":"pending|done",
  "summary":"…", "reason":"…" }
```

增长策略复用 `token-ledger` 的范式：append → 超阈值 compact → **淘汰已完成的旧条目但保留计数**。

---

## 八、调度（避开 socialLoopTick）

```
idleScheduler: setInterval(60s)
条件：无在途投递 ∧ promptQueue 空 ∧ 距上次 > minIntervalMs ∧ 该 key 新增 ≥ minNewMemories
执行：reflectBatch(key)，单次 ≤200ms / ≤50 条，游标持久化
```

| 项 | 值 | 理由 |
|---|---|---|
| 检查频率 | 60s | 与 `socialLoopTick`（5s）错开，不抢帧 |
| 单次时长 | ≤200ms | 保证不阻塞消息处理 |
| 单次记录 | ≤50 | 可用游标续跑 |
| 并发 key | 1（串行） | 简单、可预测 |
| LLM 调用 | 0 | 规则版 |

`setInterval` 必须 `unref()`；必须在 `retireSession` 与进程退出路径清理（本项目铁律，`SECURITY_BASELINE` §五.5）。

---

## 九、配置

```json
"reflect": {
  "enabled": true, "idleScheduler": true,
  "minIntervalMs": 21600000, "minNewMemories": 10,
  "batchSize": 50, "budgetMsPerRun": 200,
  "dedup": { "mode": "exact_only", "exactCosMin": 0.99, "possibleRange": [0.92,0.99], "pairsPerRun": 20 },
  "stale": { "topicDays": 30, "impressionDays": 0 },
  "synthesis": { "enabled": true, "minEvidence": 3, "minTimeSpanDays": 3,
                 "maxConfidence": 0.75, "verifyBigrams": true,
                 "autoActivateAfterProofs": 5, "autoActivateQuietHours": 48,
                 "llmSynthesis": false },
  "preference": { "enabled": true, "minOccurrences": 3, "minSpanDays": 7, "negationMinGapDays": 3 }
}
```

**机制类配置不可改**：`dedup.mode` 只接受 `exact_only`；`llmSynthesis` 改 true 需先改本文档与验收。
