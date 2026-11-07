# 06 · Recall 与索引

> 承接 `01` 的 L1 可重建性：索引与 `memoryView` 都只读 A，删了可重建。

---

## 一、为什么需要三路（现状单路的三个盲区）

现状只有 `searchVectorMemory`（`memory-vector.js:171`）。定向分析出三类它处理不了的查询：

| 查询 | 单路表现 | 原因 |
|---|---|---|
| "那个 STM32 板子多少钱" | 容易命中泛泛的"开发板选型" | 专有名词向量区分度弱 |
| "我说过的 [OWNER_QQ]" | 几乎必错 | 长数字串向量近似 |
| "上个月聊过什么" | 无法处理 | 纯时间约束，无实义主题 |

**M1 不做图检索**：本项目实体关系只有 `subjectId` 与 `participants`，单群几十人，图遍历收益低于复杂度。留注册表空位给将来。

---

## 二、索引（`memory-index.js`，L1 可重建）

### 2.1 文件结构

```
state/index/
├ meta.json     { generation, dim:1024, model:'bge-m3', docCount, coveredRev, tombstones, builtAt, crc }
├ ids.json      ["mem_a","mem_b",…]                docId = 数组下标
├ vec.f32       Float32Array(docCount × 1024)       与 ids 行对齐
├ bm25.json     { postings:{token:[[docId,tf],…]}, docLen:[…], avgdl, idfCache }
└ temporal.json { buckets: { "2026-10-31": ["mem_a",…] } }   ← 按 Asia/Shanghai 本地日
```

**体积（10k 条）**：vec 40MB + bm25 5MB + temporal 0.5MB ≈ **45MB**。全部可删可重建。

### 2.2 docId / generation / coveredRev

| 概念 | 定义 |
|---|---|
| `docId` | 构建期分配的连续整数（数组下标），**不持久、不跨重建保持** |
| `generation` | 每次全量重建 +1，单调；用于检测"读到了旧索引" |
| `coveredRev` | 已进入索引的 A 层最大 `rev`；**滞后标记** |

写入规则：
- 追加：`docId = docCount`，`vec.f32` 尾部追加，`ids.json` 尾部 append。**不重排。**
- 删除：`ids[docId] = null`，`meta.tombstones++`。**不压缩**（避免重排）。
- 压缩：`tombstones / docCount > 0.3` → 触发全量重建（retire 旧 generation）。
- 崩溃安全：先写 `*.tmp`（随机名）→ `rename`；`meta.json` **最后写**。

### 2.3 索引一致性（R2 的 L1 断言 + INo-4）

```
允许：索引滞后（写入异步）
禁止：索引领先（有 A 里不存在的 id）
强制：删除必须即时反映（同步移除 docId）
```

**为什么非对称**（`【审计】01 §4.2` 的结论）：

| 方向 | 表现 | 可接受？ |
|---|---|---|
| 追加滞后 | "刚存的暂时搜不到" | ✅ 可接受，有尾部补偿 |
| 删除滞后 | "删掉的还能搜到" | ❌ 幽灵记忆 + 隐私残留 |

**尾部补偿**：

```js
const tail = A.filter(r => r.rev > meta.coveredRev && canRead(r, ctx));
if (tail.length > 500) { 触发同步重建; return { degraded: ['index_lag'] }; }
// 否则 tail 现场打分，与 hits 按 id 合并（取较高分）
```

### 2.4 分词（并入本模块）

```js
function tokenize(text) {
  const t = String(text).toLowerCase(); const out = [];
  for (const m of t.matchAll(/[a-z][a-z0-9_+.#-]{1,30}|\d{4,}/g)) out.push('w:' + m[0]);
  const han = t.replace(/[^\u4e00-\u9fff]+/g, ' ');
  for (const seg of han.split(/\s+/)) {
    if (!seg) continue;
    if (seg.length === 1) { out.push('c:' + seg); continue; }
    for (let i = 0; i + 2 <= seg.length; i++) out.push('c:' + seg.slice(i, i + 2));
    if (seg.length <= 4) out.push('x:' + seg);
  }
  return out;
}
```

不引入 `nodejieba`（需编译）。bigram + ASCII 词 + 短整词三件套。

**已知缺口**：**错别字无能力**（"啊丢no"=Arduino 只能靠语义向量）。这是本项目真实会遇到的问题（用户打字常带同音错别字），Rev3 **不解决**，列为已知限制。

### 2.5 `estimateTokens`（放在本模块）

```js
function estimateTokens(s) {
  let cjk = 0, other = 0;
  for (const ch of String(s)) {
    const c = ch.codePointAt(0);
    if (c >= 0x4e00 && c <= 0x9fff) cjk++; else other++;
  }
  return Math.ceil((cjk * 0.75 + other / 4) * 1.15);
}
```

**唯一用途：注入前预算判断**。不得用于报告实际 token（`09` §2）。

---

## 三、三通道

### 3.1 通道注册表（扩展点）

```js
const channels = new Map();
registerChannel('semantic', fn, { weight: 1.0, timeoutMs: 300 });
registerChannel('keyword',  fn, { weight: 0.9, timeoutMs: 300 });
registerChannel('temporal', fn, { weight: 0.6, timeoutMs: 300 });
// M3: registerChannel('graph', fn, { weight: 0.8 });
```

融合层必须**遍历注册表**，不得硬编码三路。

### 3.2 语义通道

```js
const qv = await embed(ctx.query);                     // 复用 memory-vector.js:86
for (let i = 0; i < ids.length; i++) {
  if (!ids[i]) continue;
  const rec = ctx.byId.get(ids[i]);
  if (!canRead(rec, ctx.perm)) continue;               // 通道内过滤（P6）
  score = cosSimF32(qv, vec, i * dim, dim);
}
取 top perChannelLimit（默认 30）
```

- 用扁平 `Float32Array`（非 `number[]`），10k×1024 遍历 ≈8-15ms。
- 保留 `threshold=0.3` 仅作兜底下限，不作主裁量。
- `ids.length < 200` 时跳索引直接扫 A。

### 3.3 关键词通道（BM25）

```
k1=1.2, b=0.75
score = Σ IDF(t)·tf(k1+1)/(tf + k1(1-b+b|d|/avgdl))
IDF = ln(1 + (N-df+0.5)/(df+0.5))
查询侧权重：w:/x: ×1.8，c: ×1.0
```

### 3.4 时间通道（固定时区）

| 项 | 规定 |
|---|---|
| 时区 | **固定 `Asia/Shanghai`**，不跟随系统 |
| 存储 | UTC 毫秒时间戳 |
| 日界 | 本地 00:00（含）～ 次日 00:00（**不含**） |
| 周起点 | 周一 |
| 记录时间取值 | ① `origin.at` ② 记录级 `time`（legacy） ③ `extra.lastMentionAt`（仅 activeTopic） ④ 全无 → 不进本通道 |
| 表达式优先级 | 绝对日期 > 今天/昨天/前天 > 本周/上周 > 本月/上月 > 今年/去年 > 最近/前几天（7 天） > 很久以前（90 天前） |

不支持未来时间（"下周三"）与纯数字歧义（"3 月"必须带"月"字）。

---

## 四、融合（RRF）与重排

### 4.1 RRF

```
RRF(d) = Σ_c w_c / (k + rank_c(d))            k = 60
w: semantic 1.0 / keyword 0.9 / temporal 0.6
```

### 4.2 缺失通道（不补零、重归一）

```
W = Σ_{c ∈ 成功通道} w_c
RRF_norm(d) = [Σ 成功通道 w_c/(k+rank)] / (W / 2.5)
```

避免"某通道挂了导致所有分数整体缩水"。

### 4.3 归一化

```
norm(x) = (x - min)/(max - min)   over 本次候选集
若 max == min → 1.0
```

### 4.4 重排（四项，**无 channelAgreement**）

```
final = 0.55*norm(rrf) + 0.20*recency + 0.15*lifecycleWeight + 0.10*contentClassWeight
recency = exp(-ageDays/30)
lifecycleWeight: active 1.0 | stale 0.6
contentClassWeight: public 1.0 | sensitive 0.9 | uncertain 0.8
```

**为什么删掉 channelAgreement**（Rev2 的错误）：RRF 的求和本身已经体现"多路命中"，再加一项等于重复加权 15%。

**不做的**：cross-encoder reranker（本地无服务，200-400ms 拖慢回复）；LLM 重排默认关（多一次调用 + 不可复现）。

---

## 五、预算装载（两遍法）

```js
// 第一遍：顺序贪心，超预算 skip（不是 break）
const picked = []; let used = 0;
for (const r of ranked) {
  const t = estimateTokens(r.text);
  if (used + t > budget.tokens) continue;      // ← skip
  if (seen.has(fp(r))) continue;
  if (picked.length >= budget.maxItems) break;
  seen.add(fp(r)); picked.push(r); used += t;
}
// 第二遍：picked 已保持 ranked 相对顺序，直接返回
```

**为什么不按长度升序装填**：那会让长记忆永远排后面，破坏相关性顺序。顺序遍历 + skip 同时满足"不浪费预算"与"顺序稳定"。

**参数**：`injectTokens=600`、`maxItems=8`、`toolReturnTokens=1200`（工具返回更宽松，因为是模型主动要看）。

---

## 六、查询构造（避免自我强化）

```
query =
  ① 当前批次全部【用户】消息（sender ≠ 自己），拼接，上限 400 字
  ② 若 <6 字（"嗯""6""在吗"）→ 追加最近 3 条用户消息（不含 AI）
  ③ 永不含 AI 自己的发言
  ④ 永不含人格/规则/B 层文本
群聊：先剥离 "@自己"
时间通道：额外接收**原始文本**（不能用剥掉时间词的版本）
```

**为什么不能含 AI 发言**：上一轮回答里的词会变成检索信号 → 越聊越偏，且会把上一轮的错误固化。

---

## 七、去重（三层）

| 层 | 依据 | 动作 |
|---|---|---|
| 本轮内 | `fingerprint` | 跳过 |
| 与 B 层已注入 | token 集合 ⊆ B 层某条 ≥80% | 跳过（它已在上下文里） |
| 与最近 N 轮 | `prompt-ledger.injectedMemo`（fp + turn） | 3 轮内不重复（INo-11） |

---

## 八、权限断言（注入前最后一道）

```js
function assertReadable(results, ctx) {
  const bad = results.filter(r =>
       r.key !== ctx.key
    || !canRead(r, ctx.perm)
    || (ctx.key.startsWith('group:') && String(r.origin.groupId) !== String(ctx.groupId))
    || ['proposed','quarantined','superseded','deleted'].includes(r.lifecycle)
  );
  if (bad.length) audit('recall_permission_violation', { key: ctx.key, ids: bad.map(r=>r.id) });
  return results.filter(r => !bad.includes(r));
}
```

**违规要记审计，不静默过滤**：越权是"不该发生"，留下证据才能发现过滤条件写错。

---

## 九、降级（与现有约定一致）

| 故障 | 行为 | 对模型的标注 |
|---|---|---|
| `embed` 失败（Ollama 挂） | 语义路空，其余继续 | `【召回：语义不可用，已降级】` |
| BM25 索引缺失/损坏 | A 全表扫（≤2000 条），否则跳过 | `【召回：关键词索引重建中】` |
| 索引滞后 >500 | 同步重建 + 尾部扫描 | `【召回：索引追平中，结果可能不完整】` |
| 三路全空 | 返回空 + `reason:'no_match'` | 明确"没找到"，**不得编造** |
| 单通道超时 300ms | 放弃该通道 | `【召回：{通道}超时】` |

**统一原则**：失败必须可见。静默空结果会让模型说"我们没聊过"——最伤体验的错误类型。

---

## 十、延迟预算

| 环节 | P95 | P99 |
|---|---|---|
| embed(query) | 150ms | 250ms |
| 语义路（10k） | 15ms | 25ms |
| 关键词路 | 3ms | 6ms |
| 时间路 | 5ms | 10ms |
| 融合+重排 | 2ms | 4ms |
| **单次 recall 总计** | **250ms** | **400ms** |
| 单通道超时 | 300ms | — |
| 总超时 | 450ms（超则返回已完成通道 + `degraded`） | — |
| 注入路径预算 | 500ms（超则本轮跳过召回，不阻塞回复） | — |

---

## 十一、接口

```js
recall(key, query, { budgetTokens, maxItems, caller, includeProposed:false })
→ { items: RecallItem[], meta: { channelsUsed, degraded[], elapsedMs, estTokens, indexLag } }
```

- HTTP：`GET /api/socialV2/memory-recall?key=&q=`，复用 `bridge.js:6865-6870` 四项校验。
- MCP：新增 `qq_memory_recall`（`mcp__snowluma__` 前缀，R12）。
- 注入：`buildWakePromptV2` 新增块：

```
【想起的相关往事】（参考，不必逐条复述）
- 3 天前｜……（事实）
- 上周｜……（他当时说的）
```

`epistemic` 标注是安全要求：让模型区分事实与推断。

---

## 十二、与现状的接口兼容

| 现状调用点 | Rev3 处理 |
|---|---|
| `bridge.js:6872` `searchVectorMemory(key, topic, 5)` | 保留函数签名，内部转 `recall()`，`topK` 映射为 `maxItems` |
| `mcp-snowluma-safe.js:998` `/api/socialV2/memory-vector` | 端点保留；新增 `memory-recall` 端点供新工具用 |
| `mcp:1004` 返回文本格式 `【语义相关记忆（向量检索）】` | 新工具用自己的格式；旧工具输出不变 |
