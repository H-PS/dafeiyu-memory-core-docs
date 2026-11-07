# 04 · M1 · 智能召回引擎（Recall）· 修订版

> 参考 Hindsight 的 Recall：多路并行 + 融合 + 重排 + 预算收敛。
> 本版实现 **3 路**（语义 / 关键词 / 时间），第 4 路（图）留注册表空位给 M3。
> 本版相对上一版修复：P1-1 ~ P1-8（评分统一、归一化、预算装载、索引 docId、时间语义、查询构造、权限断言、token 口径）。

---

## 1. 为什么需要多路

单靠 `searchVectorMemory`（`memory-vector.js:171`）会漏三类查询：

| 查询 | 单路表现 | 原因 |
|---|---|---|
| "那个 STM32 板子多少钱" | 可能命中泛泛的"开发板选型" | 专有名词的向量区分度弱 |
| "我说过的 [OWNER_QQ]" | 几乎必错 | 长数字串向量近似 |
| "上个月聊过什么" | 无法处理 | 纯时间约束，无实义主题词 |

M1 不做图检索：本项目实体关系只有 `subjectId` 与 `participants`，单群几十人，图遍历边际收益低于复杂度。

---

## 2. 模块与注册表

```
src/memory-recall.js          编排：注册、并行、融合、重排、预算、断言
src/memory-channels/semantic.js
src/memory-channels/keyword.js
src/memory-channels/temporal.js
src/memory-permission.js      横切：canRead（每通道打分前调用）
src/tokenize.js               横切：分词 + estimateTokens（全项目唯一实现）
src/memory-index.js           索引：docId / generation / coveredRev
```

```js
const channels = new Map();
export function registerChannel(name, fn, opts = {}) {
  channels.set(name, { fn, weight: opts.weight ?? 1, timeoutMs: opts.timeoutMs ?? 300 });
}
registerChannel('semantic', semanticChannel, { weight: 1.0 });
registerChannel('keyword',  keywordChannel,  { weight: 0.9 });
registerChannel('temporal', temporalChannel, { weight: 0.6 });
// M3: registerChannel('graph', graphChannel, { weight: 0.8 });
```

融合层必须**遍历注册表**，不得硬编码三路（否则 M3 要改融合代码，违反 P7）。

---

## 3. 查询构造（P1-6 修订）

### 3.1 上一版的问题

上一版用"当前消息批次 + 最近 2 轮 AI 发言"拼 query。这会导致**自我强化**：AI 上一次回答里出现的词被当作检索信号，越聊越偏离用户真正说过的内容。

### 3.2 修订规则

```
query =
  ① 当前批次全部【用户】消息（sender ≠ 自己），按时间拼接，上限 400 字
  ② 若当前批次用户消息 < 6 字（如"嗯""6""在吗"），
     追加最近 3 条用户消息（不含 AI），按时间正序
  ③ 永不包含 AI 自己的发言
  ④ 永不包含人格卡、规则、B 层记忆文本（避免把"已有内容"再当查询）
```

补充：

- 群聊场景，`query` 里若出现 `@自己`，先剥离再检索（否则所有 query 都带同一个词，召回全部趋同）。
- 若 query 为空（只有图片/表情），召回直接返回空，不猜。
- 时间通道额外接收**原始 query 文本**（不能用剥离时间词后的版本），否则"上个月"会被剥掉。

### 3.3 为什么不用"整段会话历史"

成本高（embed 4000 字上限已经截断）、信号被稀释（历史里 90% 与当前问题无关）、且会把上一轮的错误答案固化。**query 只应表达"这一轮在问什么"。**

---

## 4. 通道 1 · 语义

```js
async function semanticChannel(ctx) {
  const qv = await embed(ctx.query);                    // 复用 memory-vector.js:86
  const { vec, ids } = ctx.index.vectors;               // Float32Array
  const out = [];
  for (let i = 0; i < ids.length; i++) {
    const rec = ctx.byId.get(ids[i]);
    if (!canRead(rec, ctx.perm)) continue;              // P6：通道内过滤
    out.push({ id: ids[i], score: cosSimF32(qv, vec, i * ctx.index.dim, ctx.index.dim) });
  }
  out.sort((a, b) => b.score - a.score);
  return out.slice(0, ctx.perChannelLimit);
}
```

要点：

- 扁平 `Float32Array`（非 `number[]`）：10,000×1024 遍历快 2-4 倍，GC 压力小。
- 保留 `SEARCH_THRESHOLD=0.3` 仅作**兜底下限**，不作主裁量（名次与重排决定取舍）。
- `ids.length < 200` 时跳过索引直接扫 A 层（索引开销大于收益）。

**性能**：10,000 条 × 1024 维约 8-15ms。

---

## 5. 通道 2 · 关键词（BM25）

### 5.1 分词（`tokenize.js`，全项目唯一实现）

```js
export function tokenize(text) {
  const t = String(text).toLowerCase();
  const out = [];
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

不引入 `nodejieba`（需编译、体积大）。bigram + ASCII 词 + 短整词三件套，对本场景够用。

### 5.2 索引结构（`bm25.json`）

```jsonc
{ "docCount": 8123, "avgdl": 42.7,
  "posting": { "c:漫展": [[docId, tf], ...] },
  "docLen": [42, 55, ...],        // 与 ids.json 下标对齐
  "idfCache": { "c:漫展": 4.12 } }
```

### 5.3 打分

```
k1=1.2, b=0.75
score(q,d) = Σ IDF(t) · tf·(k1+1) / (tf + k1·(1-b + b·|d|/avgdl))
IDF(t) = ln(1 + (N-df+0.5)/(df+0.5))
```

查询侧权重：`w:` 与 `x:` 前缀 token ×1.8，`c:` ×1.0。理由：问"那个 STM32"时 `w:stm32` 应是决定性的。

**性能**：15 token × 平均 300 posting ≈ 4,500 次累加 < 2ms。

---

## 6. 通道 3 · 时间（P1-5 修订：时区 / 边界 / 优先级）

### 6.1 时区与时基（新增，明确到不可歧义）

| 项 | 规定 |
|---|---|
| 时区 | **固定 `Asia/Shanghai`（UTC+8）**，不跟随系统时区（避免容器/CI 环境差异） |
| 存储 | 所有时间字段为 UTC 毫秒时间戳 |
| 「今天」的边界 | 本地 00:00:00.000（含）～ 次日 00:00:00.000（**不含**） |
| 「本周」起点 | 周一 00:00（中国习惯，非周日） |
| 「上个月」 | 上月 1 日 00:00（含）～ 本月 1 日 00:00（不含） |
| 闰/月末 | 用日历运算（`Date` 的 UTC 换算），不做 `-30天` 近似 |
| 记录时间取值优先级 | ① `origin.at` ② 记录级 `time`（旧数据） ③ `extra.lastMentionAt`（仅 activeTopic） ④ 三者皆无 → **不进时间通道**（不猜） |

### 6.2 表达式优先级（顺序敏感，命中即返回）

```
1. 绝对日期   2026年10月31日 / 10月31日
2. 相对日     今天 昨天 前天 大前天
3. 周         本周 这周 上周 上上周
4. 月         这个月 本月 上个月 上月
5. 年         今年 去年
6. 模糊区间   最近 这几天 前段时间（= 过去 7 天）｜很久以前（= 90 天以前）
7. 季度       Q3 / 第三季度（映射到自然季度）
```

不支持"下周三下午三点"这类未来时间（聊天记忆几乎不查未来），也不支持纯数字歧义（"3 月"里的 3 必须带"月"字）。

### 6.3 打分

```
窗口内：score = 1.0 - 0.5 * |dist(record, center) / halfWidth|     ∈ [0.5, 1.0]
窗口外：不返回
额外：若记录时间距当前 < 3 天，score += 0.1（上限 1.0）
```

时间通道的分数**只用于名次**（RRF），不参与跨通道加权——见 §8。

---

## 7. 索引的 docId / generation / 恢复（P1-4 修订）

| 概念 | 定义 |
|---|---|
| `memId` | 持久身份（`mem_01J8Z...`），A 层主键 |
| `docId` | 索引内连续整数（数组下标），**构建期分配，不持久** |
| `generation` | 索引构建代数，每次全量重建 +1，单调递增 |
| `coveredRev` | 已入索引的 A 层最大 `rev` |

写入规则：

- 追加：`docId = docCount`，`vec.f32` 尾部追加一行，`ids.json` 尾部 append。**不重排**。
- 删除：`ids.json` 对应位置置 `null`（不压缩），`meta.tombstoned += 1`。
- 压缩：`tombstoned / docCount > 0.3` 时触发全量重建（retire 旧 generation）。
- 崩溃安全：所有索引写都先写 `*.tmp` + `rename`；`meta.json` 最后写（记 `generation` 与 `coveredRev`）。
- 恢复：启动时若 `meta.json` 不存在 / `crc` 不符 / `dim` 与配置不一致 → 从 A 层全量重建（10k 条约 40-60 秒，后台线程做，期间召回只走 A 层尾巴扫描 + 关键词降级）。

**`docId` 为什么不做成持久 id**：持久 id 会随重建变化，反而制造两套身份。数组下标 + `ids.json` 映射是最简方案，重建后自动一致。

---

## 8. 融合：RRF（P1-1 修订：与重排统一）

### 8.1 RRF

```
RRF(d) = Σ_c  w_c / (k + rank_c(d))        k = 60
w: semantic 1.0 / keyword 0.9 / temporal 0.6
```

### 8.2 缺失通道处理（P1-2 修订）

某通道失败或返回空时，**不补 0 分**，而是把它的权重从分母里移除并重归一：

```
W = Σ_{c ∈ 成功通道} w_c
RRF_norm(d) = [ Σ_{c ∈ 成功通道} w_c / (k + rank_c(d)) ] / (W / W_all)
```

其中 `W_all = 1.0+0.9+0.6 = 2.5`。等价于"按实际可用通道的权重比例折算"，避免通道缺失导致所有分数整体缩水、进而把 `norm` 压到阈值以下。

### 8.3 归一化（P1-2）

```
norm(x) = (x - min) / (max - min)     over 本次候选集
若 max == min → norm(x) = 1.0（全部同分）
```

min-max 而非 z-score：候选集小（≤90）、分布偏斜，z-score 对长尾敏感。

### 8.4 重排去掉"通道一致性"（P1-1 核心修订）

**上一版错误**：RRF 里已经通过"多通道各自贡献一项"体现了通道一致性，重排里再加一个 `channelAgreement` 项等于**重复加权**，会让被两路命中的记录获得不成比例的加成（0.55+0.20+0.15+0.10 里 15% 完全冗余）。

**修订后的最终分**：

```
final = 0.55 * norm(rrf)
      + 0.20 * recency                    // exp(-ageDays / 30)
      + 0.15 * lifecycleWeight            // active 1.0 | stale 0.6
      + 0.10 * contentClassWeight         // public 1.0 | sensitive 0.9 | uncertain 0.8
```

四项权重和为 1.0。**没有任何一项与 RRF 的通道数相关**。`proposed` / `quarantined` / `superseded` / `deleted` 不进入候选，因此不需要在权重里表达。

### 8.5 LLM 重排（M2 可选，默认关）

触发：候选 > 15 ∧ `recall.rerank.mode==='llm'` ∧ 当日配额未用尽（默认配额 0 = 关闭）。

把候选压成 `id + 30 字摘要` 列表问模型排序，约 800 in / 100 out token。默认关的理由：多一次调用 = 多 300-800ms 延迟 + 不可复现。

---

## 9. 预算装载（P1-3 修订）

### 9.1 上一版的问题

上一版用：

```js
if (used + t > budget.tokens) break;   // ← 错误：首条超预算就整个放弃
```

若第一条是很长的摘要（800 token）而预算 600，会直接返回 0 条，而后面明明有 5 条 50 token 的短记忆可用。

### 9.2 修订：两遍法

```js
// 第一遍：按序贪心，超预算则【跳过】，不中断
const picked = [];
let used = 0;
for (const r of ranked) {
  const t = estimateTokens(r.text);
  if (used + t > budget.tokens) continue;      // skip，不是 break
  if (seen.has(r.fingerprint)) continue;
  if (picked.length >= budget.maxItems) break;
  seen.add(r.fingerprint);
  picked.push(r);
  used += t;
}
// 第二遍：picked 已保持 ranked 的相对顺序（因为顺序遍历），无需再排
return picked;
```

**为什么不用"按长度升序装填"**：那会让长记忆永远排后面，破坏相关性顺序。顺序遍历 + skip 同时满足"不浪费预算"与"顺序稳定"（INo-14）。

### 9.3 去重（三层）

1. 本轮内：`fingerprint` 相同 → 跳过。
2. 与 B 层已注入内容：把 `formatMemoryV2` 输出 tokenize 后比对，若某召回结果的 token 集合 ⊆ B 层某条 ≥80%，跳过。
3. 与最近 3 轮注入历史：S 层 `promptLedger.injectedMemo`（fingerprint + turn），3 轮内不重复（INo-11）。

---

## 10. 权限断言（P1-7 修订：完整断言）

上一版只断言 `r.key === key`，不完整。本版断言**五项全过**：

```js
function assertRecall(results, ctx) {
  const bad = results.filter(r =>
       r.key !== ctx.key
    || !canRead(r, ctx.perm)                      // 含 scope/lifecycle/contentClass
    || (ctx.key.startsWith('group:') && r.scope?.originGroupId !== ctx.groupId)
    || ['proposed','quarantined','superseded','deleted'].includes(r.lifecycle)
    || (ctx.caller === 'agent' && r.lifecycle === 'stale' && !ctx.allowStale)
  );
  if (bad.length) {
    audit('recall_permission_violation', {
      key: ctx.key, caller: ctx.caller,
      ids: bad.map(r => r.id),
      reasons: bad.map(r => ({ id: r.id, key: r.key, lifecycle: r.lifecycle, cls: r.scope?.contentClass }))
    });
    return results.filter(r => !bad.includes(r));
  }
  return results;
}
```

**事故记录而不是静默过滤**：越权是"不该发生"的，一旦发生必须留下证据（否则过滤条件写错也永远发现不了）。这条同样适用于注入前的最后一关。

---

## 11. 失败与降级（P1 保持，补充口径）

| 故障 | 行为 | 对模型的标注 |
|---|---|---|
| Ollama 不可用 | 语义路空，其余继续 | `【召回：语义通道不可用，已降级】` |
| BM25 索引缺失/损坏 | A 层全表扫（≤2000 条），否则跳过该路 | `【召回：关键词索引重建中】` |
| 索引滞后 > 500 条 | 触发同步重建，本次用尾部扫描 | `【召回：索引追平中，结果可能不完整】` |
| 三路全空 | 返回空 + `reason:'no_match'` | 明确"没有找到相关记忆"，**不得编造** |
| 单通道超时 300ms | 放弃该通道 | `【召回：{通道}超时】` |

**统一原则**：失败必须可见。静默空结果会让模型说"我们没聊过这个"——这是最伤体验的错误类型。

---

## 12. 对外接口

```js
export async function recall(key, query, opts = {}) {
  // opts: { budgetTokens, maxItems, channels, caller, allowStale, includeProposed }
  // → { items: RecallItem[], meta: { channelsUsed, degraded[], elapsedMs, usedTokens, indexLag } }
}
```

`RecallItem`：

```jsonc
{ "id": "mem_...", "text": "...", "kind": "fact", "category": "activeTopic",
  "score": 0.72, "channels": ["semantic","keyword"],
  "originAt": 1761900000000, "ageText": "3 天前",
  "epistemic": "fact", "contentClass": "public" }
```

HTTP：`GET /api/socialV2/memory-recall?key=&q=&purpose=`，鉴权复用 `bridge.js:6865-6870` 四项校验，**不得新增跳过分支**。

MCP：新增 `qq_memory_recall`；`qq_memory_query` 的 topic 分支内部改调 `recall()`（保持返回格式）。

注入（`buildWakePromptV2`）：

```
【想起的相关往事】（参考，不必逐条复述）
- 3 天前｜……（事实）
- 上周｜……（他当时说的）        ← epistemic=opinion
```

`kind` / `epistemic` 标注是安全要求：让模型区分"原始事实"与"推断/看法"。

---

## 13. 性能与 Token 口径（P1-8 修订）

### 13.1 唯一 token 计量实现

```js
// tokenize.js —— 全项目只允许这一个实现，禁止各处自写
export function estimateTokens(s) {
  let cjk = 0, other = 0;
  for (const ch of String(s)) {
    const c = ch.codePointAt(0);
    if (c >= 0x4e00 && c <= 0x9fff) cjk++; else other++;
  }
  return Math.ceil((cjk * 0.75 + other / 4) * 1.15);
}
```

**校准机制**：模型 API 返回 `usage.prompt_tokens` 时，记录 `estimated / actual` 比值，滚动 100 次取中位数作为 `calibrationFactor`（默认 1.0，仅在 0.8-1.4 区间内才采纳）。这样"估算"会随时间收敛到实际值，而不是永远拍脑袋。

所有文档中的 token 数字**统一标注为估算值**，实测值必须由 `eval-context.mjs` 或 API usage 给出。

### 13.2 延迟预算

| 环节 | 目标 | 实测口径 |
|---|---|---|
| embed(query) | P95 ≤ 150ms | 单次计时 |
| 语义路（10k 条） | ≤ 15ms | 单次 |
| 关键词路 | ≤ 3ms | 单次 |
| 时间路 | ≤ 5ms | 单次 |
| 融合+重排 | ≤ 2ms | 单次 |
| **单次 recall（串行部分）** | **P95 ≤ 250ms、P99 ≤ 400ms** | 三通道 `allSettled` 后到返回 |
| 通道超时 | 单通道 300ms；**总超时 450ms** | 超总时直接返回已完成的通道结果 + `degraded` |

上一版只写了"合计 ≤200ms"，没有区分 P95/P99、没有总超时。本版补齐。
