# 向量库接入

> SQLite 存结构化字段，LanceDB 存向量，本地 embedding 生成。

---

## 0. 一句话

**向量库只负责「找相似」，不负责「排序」。**

```
LanceDB.search(q).limit(50)  →  50 个候选
                                    ↓
                            情绪重排（attention.mjs）
                                    ↓
                              取 3-5 条进上下文
```

---

## 1. 为什么不能直接用向量分数

### 问题一：向量相似 ≠ 该被想起

```
「他说明天一定来」         v=+0.6
「他上次也这么说结果没来」   v=-0.8
```

向量相似度可能 **0.9**（字面极像），但情绪相反。

- 开心时：想起「明天一定来」是对的，想起那次失约是**反刍**
- 难过时：反过来

**向量管「是不是同一件事」，情绪管「该不该现在想起」。两者正交。**

### 问题二：向量不知道状态

向量不知道：
- 你现在累不累
- 压力大不大
- 这件事之前提过几次
- 你现在能不能装下这么多

这些都在 `attention.mjs` 里算。

---

## 2. 三个适配器

`vector-store.mjs` 不写死任何实现，给你三个可替换的：

| 适配器 | 适用 | 依赖 |
|---|---|---|
| `memoryAdapter()` | ≤1000 条，测试 | 零 |
| `sqliteAdapter(db)` | ≤1 万条，和你现有库同处 | better-sqlite3 |
| `lancedbAdapter(table)` | 大规模，ANN | LanceDB |

### 用你的 LanceDB

```js
import * as lancedb from '@lancedb/lancedb'
import { lancedbAdapter, createRecall } from './vector-store.mjs'

const db = await lancedb.connect('./data/lance')
const table = await db.openTable('memories')

const adapter = lancedbAdapter(table)
const recall = createRecall({
  adapter,
  embedder: myLocalEmbed,      // 你的本地 embedding
  getMeta: (id) => memoryMap.get(id),
})
```

### 用你的 SQLite

```js
import Database from 'better-sqlite3'
import { sqliteAdapter } from './vector-store.mjs'

const db = new Database('./her.db')
db.exec(`
  CREATE TABLE IF NOT EXISTS mem_vectors(
    id INTEGER PRIMARY KEY,
    vec BLOB NOT NULL
  )
`)

const adapter = sqliteAdapter(db)
```

**好处**：向量和结构化字段在同一个库，不用维护两套存储。

---

## 3. 接口

适配器必须实现四个方法：

```js
async upsert(item)          // 插入或更新
async search(vector, n)     // → [{ id, distance }]，按距离升序
async remove(id)
async count()
```

**注意返回的是 `distance`（越小越像），不是 similarity。** 适配器负责转换。

---

## 4. 两阶段检索

```js
const candidates = await recall.recall(query, att, {
  n: 5,                    // 想要几条
  minSimilarity: 0.3,      // 太不像的丢掉
  filter: (m) => m.kind !== 'reply',
})
// → [{ ...meta, similarity, score }]

const final = focus(candidates, st)
// → { picked, capacity, tunneled, attention }
```

### overFetch

```js
const overFetch = 10   // 默认召回 10 倍候选
```

**为什么要 10 倍？**

因为情绪重排会大改顺序。只召回 5 条，重排之后可能全是反向的。

实测（查询「他答应我的事又没做到」，召回 10 条）：

```
向量召回顺序（按相似度）：
  sim=0.182 v=-0.9 「他说我太敏感了」
  sim=0.174 v=+0.6 「他帮我修了电脑」
  sim=0.167 v=-0.7 「他答应陪我看电影，后来加班没去」
  sim=0.114 v=+0.5 「他记得我不吃香菜」
  ...

情绪重排（难过）：
  v=-0.9 score=0.497  ← 升到第一
  v=-0.7 score=0.427
  v=+0.6 score=0.141  ← 降到第三
  v=+0.5 score=0.095
```

**如果只召回 3 条，重排后只剩 1 条负面的。**

---

## 5. 格式化

```js
formatMemories(picked, { maxChars: 800 })
// →
// · 他说我太敏感了，让我别想多
// · 他答应陪我看电影，后来加班没去
```

**不标分数、不标 valence、不标 importance。**

模型看到 `score=0.73` 会开始演「这条很重要」。我们只要它自然地说出来。

---

## 6. 完整链路

```
情绪状态（mind.mjs）
    P/A/D/E/L
        ↓
注意力分配（attention.mjs）
    capacity / breadth / inhibition / tunnel
        ↓
向量召回（vector-store.mjs）
    LanceDB.search(q).limit(capacity × 10)
        ↓
情绪重排（attention.mjs）
    情绪一致 → 扩散 → 侧抑制 → 按容量选
        ↓
格式化
    · 记忆文本（无分数）
        ↓
组装进 prompt（pipeline.mjs）
```

---

## 7. 规模建议

| 条数 | 全量算余弦 | 建议 |
|---:|---:|---|
| 100 | ~0.1ms | 全量，零依赖 |
| 1,000 | ~1.3ms | 全量，零依赖 |
| 10,000 | ~12.8ms | 全量可接受，或用 SQLite |
| 100,000 | ~128ms | **需要 ANN（LanceDB）** |

**注**：上表按 DIM=128 估算。真实 embedding 多为 512–1536 维，耗时 ×4–12。

### 分界点

- **< 1 万条**：`sqliteAdapter` 全量算，最简单
- **> 1 万条**：`lancedbAdapter`，用 ANN

你现在的规模「未知」，我建议：

1. 先用 `sqliteAdapter` 跑起来
2. 到 5000 条时测一次检索耗时
3. 超过 50ms 再换 LanceDB

**接口一样，换适配器不用改上层。**

---

## 8. 向量库不负责什么

| 任务 | 谁负责 | 为什么不是向量库 |
|---|---|---|
| 情绪一致性加权 | attention.mjs | 向量不懂情绪 |
| 激活扩散 | attention.mjs | 向量只返回 top-k，没有"顺带想起" |
| 侧抑制 | attention.mjs | 向量不知道"重复"是坏事 |
| 容量限制 | attention.mjs | 向量不懂"此刻能装几个" |
| 时间遗忘 | memory.mjs | 向量没有时间概念 |
| valence 计算 | extract.py | 向量不能判断好坏 |

**向量库只做一件事：给定查询向量，返回最近的 k 个。**

---

## 9. 你的 embedding

你说用本地 embedding。接口是：

```js
const embedder = async (text) => number[]  // 返回归一化向量
```

**注意**：
- 必须归一化（否则余弦计算不准）
- 维度固定（建库时定，不能中途改）
- 同一个模型（换了模型要重建索引）

### 建议的本地模型

| 模型 | 维度 | 中文 | 速度 |
|---|---:|---|---|
| bge-small-zh | 512 | 好 | 快 |
| bge-base-zh | 768 | 很好 | 中 |
| bge-m3 | 1024 | 最好（多语言） | 慢 |
| text2vec-base-chinese | 768 | 好 | 中 |

**512 维够用。** 记忆是短文本，不需要大模型。

---

## 10. 已知限制

| 限制 | 影响 | 补救 |
|---|---|---|
| 没装 LanceDB | 无法真跑集成测试 | 接口已抽象，你接上即可 |
| mock embedding 是字符哈希 | 相似度不准 | 换成真模型 |
| 归一化靠调用方 | 忘了归一化会出错 | 在 adapter 里再归一化一次 |
| 没有增量更新 | 改了记忆要重建向量 | 用 upsert |

---

## 11. 文件

| 文件 | 内容 |
|---|---|
| `vector-store.mjs` | 三个适配器 + 两阶段检索 + 格式化 |
| `test-vector.mjs` | 六组验证（用 mock embedding） |
| `VECTOR.md` | 本文档 |
