# 情绪管线设计

> 目标：把 `mind.mjs` 的状态机组织成一条**时序正确**的流水线。
> 记忆库后续接入，本文只定义接口位置。

---

## 0. 为什么需要管线这一层

`mind.mjs` 只回答了「状态怎么变」，没回答「**什么时候**变、**按什么顺序**变」。

真实对话里最常犯的三个错，全是时序问题：

| 错误 | 后果 |
|---|---|
| 先算回复再更新情绪 | 用上一轮的心情回答这一句 |
| 每句话都 `tick(1)` | 时间没走，情绪却衰减了 |
| 他的情绪和她的情绪混在一起 | 互相污染 |

管线用固定的七站解决它们。

---

## 1. 七站

```
他说的每一句
    │
    ▼
┌─────────────┐
│ 1 ingest    │  只记录，不改状态
└──────┬──────┘
       ▼
┌─────────────┐
│ 4 settle    │  ★ 先让时间流过
└──────┬──────┘
       ▼
┌─────────────┐
│ 2 perceive  │  抽线索、判意图、找重复
└──────┬──────┘
       ▼
┌─────────────┐
│ 3 appraise  │  算情绪增量
└──────┬──────┘
       ▼
┌─────────────┐
│ 5 compose   │  组装上下文
└──────┬──────┘
       ▼
   生成回复
       │
       ▼
┌─────────────┐
│ 6 emit      │  记录她说了什么
└──────┬──────┘
       ▼
┌─────────────┐
│ 7 absorb    │  她说的话对她自己也有影响
└─────────────┘
```

### 关键：settle 在 perceive 之前

这是整条管线最重要的一个设计决定。

```
错误顺序：perceive → appraise → settle
  → 评估用的是"上一轮结束时"的精力/负荷
  → 隔了一夜，还按昨天累的状态判断今天的话

正确顺序：settle → perceive → appraise
  → 评估用的是"此刻"的精力/负荷
```

**实测**：30 分钟后再说话，评估用的 P 从 −0.54 衰减到 −0.20，而不是拿 −0.54 去算。

---

## 2. 各站职责

### 站 1：ingest —— 只记录

```js
pipe.receive('你怎么这么蠢')
```

为什么单独一站？因为**记录失败**和**理解失败**的后果不同：

- 记录失败 → 丢上下文
- 理解失败 → 只是这一句算错

分开就能各自重试。

### 站 4：settle —— 让时间流过

**用真实经过的分钟数，不是固定值。**

```js
settle(nowMs = Date.now()) {
  const dtMin = (nowMs - lastSettleMs) / 60000
  ...
}
```

自动识别状态：

| 条件 | 处理 |
|---|---|
| ≥ 8 小时 | 当作睡了一觉（精力恢复、负荷下降） |
| ≥ 30 分钟 | 当作休息 |
| 其他 | 正常消耗 |

**实测**：隔夜 8 小时后，精力 0.69 → 0.99，情绪归零。

### 站 2：perceive —— 抽线索

只做**不改变状态**的解读：

```js
{
  event,          // R/C/G/K/N/O 六维
  repeat,         // 这件事被提第几次
  repeatedTopic,  // 重复的是哪件
  intent,         // attack/comfort/request/probe/chat
  replyVerdict,   // 是不是在回应上一轮
}
```

### 站 3：appraise —— 施加情绪

**顺序：先处理回应，再施加新事件。**

```js
// 3a. 先处理：这句话是不是在接住/否定
if (replyVerdict) handleReply(...)

// 3b. 再施加：这句话本身作为事件
//      但如果已经被当作"回应"处理过，就不要再当攻击
if (!skipAsEvent) mind.say(...)
```

反过来的话，一句「对不起」会被当成新事件先扣一遍分，再清零。

### 站 5：compose —— 组装

**人格和记忆由外部注入，管线只负责「此刻」这一块。**

```js
ctx.assemble({
  persona,      // 外部
  memory,       // 外部（后续接入）
  instructions, // 外部
})
```

产出：

```markdown
【人格】
你是阿岚。说话直接，不爱绕弯。

【记忆】
· 上周他答应陪你看电影，后来加班没去

【此刻】
[情绪] 此刻：心情偏沉，有点绷着。倾向：容易过度解释，注意别讨好。

【要求】
按你的人格说话。上面只是此刻的状态，不是你的人格。
```

### 站 6：emit —— 记录她说了什么

不改情绪状态。她说什么是她的事。

### 站 7：absorb —— 她的话对她自己也有影响

| 机制 | 条件 | 效果 |
|---|---|---|
| 倾诉减压 | 长句 + 情绪强 | L −0.02，压抑 −0.15 |
| 说重话消耗 | 重话 + **正在气头上** | E −0.03，D −0.02 |
| 新增压抑 | 说「我没事」 | suppressed +0.12，L +0.015 |
| 主动求和 | 说「对不起」 | relatedness +0.03 |
| 做出承诺 | 「我明天…」 | 记入 promises |

**注意「说重话消耗」的条件**：不只看词表，还看**说的时候是不是已经在气头上**。

> 一个平静的人说「随便」不累；一个愤怒的人说「随便」很累。

---

## 3. 接口

```js
import { createPipeline } from './pipeline.mjs'

const pipe = createPipeline({ trust: 0.6 })

// 一步走完
const ctx = pipe.receive('你怎么这么蠢')
ctx.assemble({ persona, memory })   // → prompt
ctx.temperature                      // → 0.76

// 完整一轮
const r = await pipe.turn(text, async ({ prompt, temperature }) => {
  return await callLLM(prompt, { temperature })
}, { persona, memory })
```

### 全部方法

| 方法 | 作用 |
|---|---|
| `receive(text, opts)` | 七站走一遍，返回上下文 |
| `turn(text, generate, opts)` | receive + 生成 + emit + absorb |
| `assemble(opts)` | 组装 prompt |
| `idle(minutes)` | 手动推进时间（定时器用） |
| `snapshot()` | 完整状态快照 |

### receive 的选项

```js
pipe.receive(text, {
  nowMs: Date.now(),    // 用于测试，模拟时间流逝
  isReply: undefined,   // 自动判断；显式传 true/false 可覆盖
  verdict: 'catch',     // 强制指定回应类型
  trace: true,          // 开启调试轨迹
  ctx: { prevAgency },  // 跨句上下文
})
```

---

## 4. 自动判断「这是不是回应」

```js
isReply = peakAt > 0 && peak > 0.3 && isComforting(text)
```

**三个条件缺一不可。** 特别是第三个：

> 如果只看「顶点挂着」，一句新的攻击会被当成道歉——
> `classifyReply` 的兜底规则是「长度≥6 算 catch」，会把攻击吞掉。

这是实际踩过的 bug：第三次攻击后顶点被误清零。

### isComforting 的词表

```js
/(对不起|抱歉|我错了|是我不好|别难过|别生气|抱抱|乖|心疼|辛苦|
  我懂|我明白|理解|换我也会|怎么了|还好吗|没事吧|慢慢说|
  话说重了|太冲了|我太急)/
```

---

## 5. 实测数据

### 时序：settle 必须先于 appraise

```
第一次后 P=-0.54
30 分钟后收到新消息：
  结算了 30.0 分钟，resting=true
  评估用的 P=-0.20（已衰减）    ← 关键
  结果 P=-0.16
```

### 时序：道歉先于新事件

```
三次攻击后：peak=0.88
收到道歉后：peakAt=0 peak=0.00
判定：catch
效果：接住
```

### 时间推进只做一次

```
重复推进（手动 tick + receive 自动 settle）：
  P: -0.54 → -0.24
正确做法（只靠 receive 的真实时间）：
  P: -0.54 → -0.36
差别：0.119
```

**教训**：settle 只能在管线里做一次，外面不要再 tick。

### 完整弧线

```
[+  0min] 「早上好呀」          P=+0.33 L=0.28 E=0.69 兴奋
[+ 65min] 「你怎么这么蠢」       P=-0.48 L=0.44 敌意/愤怒
[+ 68min] 「你这个废物」         P=-0.98 L=0.63 敌意/愤怒   ← 顶点
[+ 71min] 「算了，我没事」       P=-0.88 L=0.64 敌意/愤怒
[+120min] 「在忙」              P=-0.19 L=0.63 敌意          ← 顶点超时
[+180min] 「你想多了」          P=-0.08 L=0.61 平静
[+300min] 「对不起，昨天话说重了」 P=-0.03 L=0.58 平静
最终：unmet=1  caught=0  peakTrait=0.01  peakSens=1.25
```

**注意 `caught=0`**：道歉发生在顶点超时之后。这是正确的——**太晚的修复不算修复**。

---

## 6. 顶点的触发条件（实测边界）

顶点不是随便就能形成的。需要**密集 + 高强度**同时满足：

| 事件间隔 | 强度要求 | 到顶点需要几次 |
|---|---|---|
| 3 分钟 | 高（>0.85） | 2 次 |
| 3 分钟 | 中（0.4-0.6） | 到不了 |
| 10 分钟 | 高 | 2 次 |
| 20 分钟 | 高 | 8 次内到不了 |

**这是刻意设计的。** 真实心理也是如此：

> 崩溃从来不是一天挨一句，是十分钟里挨三句。

如果想让顶点更容易触发，调低 `peakThreshold`（默认 0.6）——但要接受它会变敏感。

---

## 7. 调试轨迹

```js
const ctx = pipe.receive('你上次说的事又没做到', { trace: true })

for (const s of ctx.trace.stages) {
  console.log(s.stage, s)
}
```

输出：

```
[ingest    ] { text: '你上次说的事又没做到' }
[settle:pre] { dtMin: 0, sleeping: false }
[perceive  ] { label: '...', intensity: 0.768, repeat: 0 }
[appraise  ] { delta: { P: -0.523, A: 0.141, D: ... } }
[compose   ] { label: '焦虑', readiness: '僵住' }
```

**任何一步出了问题，都能看到它在哪一站偏离。**

---

## 8. 记忆库接入点（预留）

管线已经留好了位置，后续接入只需三步：

### 8.1 站 2 之后，站 5 之前

记忆检索需要「当前心情」，所以要放在 `appraise` 之后：

```
perceive → appraise → [recall] → compose
```

### 8.2 recall 需要的东西

```js
const recalled = recall({
  text,                    // 当前这句话（抽线索）
  affect: ctx.affect,      // ← 当前心情 P/A/D
  energy: ctx.energy,      // ← 当前精力（决定召回深度）
  topic: ctx.perceived,    // ← 已抽好的线索，不用重复抽
})
```

**注意 `affect` 和 `energy` 来自同一个 state**，与 `compose()` 用的完全一致。这保证不会出现「心情说低落，但召回的是开心记忆」。

### 8.3 compose 增加一个字段

```js
ctx.assemble({
  persona,
  memory: formatMemories(recalled),   // ← 新加
  instructions,
})
```

管线本身不需要改——`assemble` 已经支持任意数量的块。

---

## 9. 已知限制

| 限制 | 影响 | 补救 |
|---|---|---|
| `isComforting` 是词表 | 反讽的道歉判不出来 | 传 `verdict` 覆盖 |
| `classifyReply` 兜底偏宽 | 长句默认算 catch | 用 LLM 判四选一 |
| 话题匹配是字符二元组 | 同义词匹配不上 | 接受，或加同义词表 |
| 顶点需要密集攻击 | 慢节奏对话不会有顶点 | 调低 `peakThreshold` |

---

## 10. 文件

| 文件 | 内容 |
|---|---|
| `pipeline.mjs` | 管线本体，零依赖 |
| `test-pipeline.mjs` | 九组时序测试 |
| `mind.mjs` | 状态机（管线的核心） |
| `PIPELINE.md` | 本文档 |
