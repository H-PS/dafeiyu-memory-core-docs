# 情绪模块 · 独立挂载指南

> 面向：朋友式开放域聊天角色。人格先行，情绪只修饰不覆盖。
> 技术栈：Node / ESM，零依赖。

---

## 0. 三行接入

```js
import { createMind } from './mind.mjs'

const mind = createMind({ trust: 0.6 })

mind.say('你怎么这么蠢')   // 他说了一句
mind.tick(30)              // 推进 30 分钟（没对话时也要调）
console.log(mind.hint())   // → 给 LLM 的状态提示
```

**没有第 4 行。** 不用改你的人格卡、记忆库、模型调用。

---

## 1. 人格先行怎么保证

这是设计里最容易被做坏的一点。分三层保证：

### 1.1 输出的是修饰语，不是身份语

```js
// 好
"[情绪] 此刻：心情偏沉。倾向：话会变短，别主动展开。"

// 坏 —— 这会覆盖人格卡
"你是一个冷漠的人。"
```

`hint()` 里每个字都只描述**此刻**，不描述**你是什么人**。而且结尾固定带一句：

> （这是此刻的状态，不是你的人格；人格照旧。）

### 1.2 情绪只决定「长度、语气、是否展开」

不决定「说什么」。同一个状态，两个人格会说出完全不同的东西：

```
状态：[情绪] 心情偏沉。倾向：话会变短。

人格 A（直接、嘴硬）：
  → 「行，我蠢。你自己来吧。」

人格 B（温和、怕冲突）：
  → 「……对不起，我是不是又说错了。」
```

**这就是"人格先行"的实测含义。**

### 1.3 温度系数独立于人格

```js
const temp = mind.temperature()   // [0.3, 1.4]
finalTemperature = personaBaseTemperature * temp
```

疲惫直接降采样温度，让输出自然收敛，**不需要模型"表演"累**。这是移植自《没有卡的鱼》的手法：绕过模型的自我描述，直接改物理参数。

---

## 2. 接进 prompt 的正确顺序

```js
const prompt = [
  '【人格】',
  persona,              // ← 永远在最前，且最详细
  '',
  '【此刻状态】',
  mind.hint(),          // ← 只有一两句
  '',
  '【要求】',
  '按你的人格说话。上面只是你此刻的状态，不是你的人格；',
  '累了就话短一点，但别改变你说话的方式。',
].join('\n')
```

**比例很重要**：人格 300 字，状态 50 字。如果状态比人格长，模型就会开始演状态。

---

## 3. 生命周期

### 3.1 何时调用

| 时机 | 调用 | 说明 |
|---|---|---|
| 收到他的消息 | `mind.say(text)` | 抽取 + 更新状态 |
| 他回了一句 | `mind.reply(text)` | 四级判定，见 3.2 |
| 每 5-10 分钟 | `mind.tick(5)` | 情绪衰减、负荷恢复、承诺检查 |
| 生成回复前 | `mind.hint()` | 取状态提示 |
| 生成回复时 | `mind.temperature()` | 取温度系数 |
| 跨天 / 睡觉 | `mind.tick(480, { sleeping: true })` | 精力恢复 |

### 3.2 "被接住"怎么判定

**这是唯一需要语义判断的地方。** 我把它做成纯函数 + 可覆盖。

#### 四级，不是两级

大部分人把"他回了一句"当成"接住了"。这是错的。真实有四档：

| 判定 | 例子 | 效果 |
|---|---|---|
| **catch 接住** | "对不起，刚才话说重了"<br>"怎么了？跟我说说"<br>"我懂，换我也会" | 顶点清零、P 回升、trust↑ |
| **deflect 转移** | "对了，你吃饭了吗"<br>"话说那个文件我看了" | 顶点继续挂着，轻微缓和 |
| **ignore 忽略** | "嗯" "哦" "在忙" "" | **什么都不发生**，顶点继续挂着 |
| **invalidate 否定** | "你想多了" "至于吗"<br>"我又没说什么" "你太敏感了" | **比沉默更糟** |

#### 关键：否定 > 沉默

这是整个判定里最重要的一条。实测（同一句冒犯后）：

| 他的回应 | 判定 | ΔP | Δtrust | Δrel | ΔL | Δtrait |
|---|---|---:|---:|---:|---:|---:|
| （沉默） | ignore | 0 | 0 | 0 | 0 | 0 |
| "嗯" | ignore | 0 | 0 | 0 | 0 | 0 |
| "对了你吃饭了吗" | deflect | +0.02 | 0 | 0 | 0 | 0 |
| "对不起，话说重了" | catch | **+0.09** | +0.036 | +0.044 | 0 | 0 |
| "你想多了" | invalidate | 0 | **−0.060** | **−0.100** | **+0.060** | **+0.020** |

**"你想多了"比什么都不说更伤人。** 心理学上叫 invalidation，是关系修复的头号杀手。大多数情绪系统根本没这一档。

#### 感知折扣：防御强的人，接不住好意

```js
quality = max(catchFloor, 1 - perceptionDiscount * (1 - relatedness))
```

| relatedness | quality | 效果 |
|---:|---:|---|
| 0.2（很防御） | 0.60 | 只收到 60% |
| 0.5 | 0.72 | |
| 0.9（很安全） | 0.93 | 几乎全收 |

**这就是"她已经听不进去了"的可计算版本。**

但 `catchFloor=0.6` 是必须的：否则会死循环——越防御越收不到好意，越收不到越防御，最后永远修不好。这个地板保证**再糟也能收到一部分**。

#### 你该怎么用

```js
// 方式一：让它自己判（快速接入）
mind.reply('对不起，刚才话说重了')     // → { verdict: 'catch', quality: 0.72 }

// 方式二：你判，它执行（推荐）
const verdict = await llmClassify(reply)   // 四选一
mind.reply(text, verdict)

// 方式三：只判不改
const v = mind.classify(text)   // → 'catch' | 'deflect' | 'ignore' | 'invalidate'
```

**推荐方式二。** LLM 判语义比正则准，尤其是反讽。但让它输出**枚举值（四选一）**，不是数值——这样它没法"美化"。

---

### 3.3 跨句：三种情况，都不需要 LLM

我上一轮说"跨句需要 LLM"，说得太笼统。拆开看是三种不同的问题：

| 类型 | 例子 | 单句抽取的问题 | 需要 LLM 吗 |
|---|---|---|---|
| **A 指代** | "他上次说的那件事" | 不知道"那件事"是什么 | **不需要**，话题追踪 |
| **B 重复** | 同一件事被提第 5 次 | 每次都是独立事件 | **不需要**，相似度匹配 |
| **C 承诺** | "我明天给你答复" → 没兑现 | 当时是正性，之后没事件 | **不需要**，时间检查 |

**三种都不需要。** 我上一轮高估了。

#### A/B：话题累积

```js
matchTopic(st, text, p)   // → { topic, sim, repeat }  字符二元组 + Jaccard
trackTopic(st, text, p)   // → 重复次数（0 = 新话题）
```

两条通道，方向相反：

```js
// 1. 负荷涨得更快（"又是这件事"比"新的一件坏事"更累）
overload *= 1 + topicLoadMult * repeat

// 2. 感受变得更钝（骂多了会麻）
eff *= max(0.3, 1 - topicNumbMult * repeat)
```

**实测（同一句重复 6 次，间隔 30 分钟）：**

| 次数 | P | L |
|---:|---:|---:|
| 1 | −0.52 | 0.315 |
| 2 | −0.55 | 0.419 |
| 3 | −0.49 | 0.514 |
| 4 | −0.41 | 0.593 |
| 5 | −0.34 | 0.660 |
| 6 | **−0.32** | **0.732** |

**P 在回升，L 在飙升。** 这是"骂多了不疼了，但人越来越垮"。

对照：四次**不同**的坏事，P 一路下探到 −0.84，L 到 0.753。

> **重复让人麻木，多样性让人崩溃。** 这个结论是从数据里跑出来的，不是设计的。

#### C：承诺失约

```js
mind.say('我明天给你答复')      // 自动记录
mind.tick(25 * 60)             // 自动检测失约
```

到期未兑现：L +0.05，trust −0.05，relatedness −0.05，gOther +0.2，normViol +0.15。

**这解决了"他说了会做但没做"——最常见的伤害，单句抽取完全看不见。**

---

### 3.4 什么才真需要 LLM

| 场景 | 为什么 |
|---|---|
| **反讽** | "你可真行" → 正性词，负性意 |
| **阴阳怪气** | "行吧，你说了算" → 字面服从，实际怨气 |
| **潜台词** | "没事，你忙你的" → 字面体谅，实际委屈 |

**共同点：字面和实际相反。** 规则做不到，因为它需要世界知识。

但注意：这三种恰好 `SUPPRESS` 词表能兜一部分（"算了""我没事""随便你"）。规则能兜底，只是不如 LLM 准。

**建议**：先用规则跑，观察误判率。如果误判集中在反讽，再上 LLM——而且只做**二分类**（"真心还是反讽"），不做数值。

### 3.5 多用户

```js
const minds = new Map()

function getMind(userId, opts) {
  if (!minds.has(userId)) {
    minds.set(userId, createMind({ trust: opts?.trust ?? 0.5 }))
  }
  return minds.get(userId)
}
```

**每人一个实例。** 信任度是关系属性，不能共享。

---

## 4. 持久化

状态是纯 JSON，690 字节。

```js
// 存
fs.writeFileSync(`mind/${userId}.json`, JSON.stringify(mind.state))

// 取
const mind = createMind({ state: JSON.parse(fs.readFileSync(path, 'utf8')) })
```

**建议存盘时机**：每次 `say()` 之后、每 10 次 `tick()` 之后、进程退出前。

**注意**：`state` 里没有函数，也没有不可序列化的东西，直接 `JSON.stringify` 就行。

---

## 5. 接口清单

```js
const mind = createMind({ trust, state, params })

mind.say(textOrEvent, { replied })         // → { appraisal, delta, event, promise }
mind.reply(text, verdict?)                 // → { verdict, quality, effect }
mind.classify(text)                        // → 'catch'|'deflect'|'ignore'|'invalidate'
mind.tick(minutes, { resting, sleeping })  // → this
mind.responded()                           // → this（旧接口，等于强制 catch）
mind.hint()                                // → string（给 LLM）
mind.temperature()                         // → number（乘到 base temp）
mind.snapshot()                            // → object（调试/面板）
mind.repeatedTopics()                      // → [{ text, count }]
mind.missedPromises()                      // → [{ text }]
mind.state                                 // 原始状态，可读写
mind.params                                // 参数，可运行时改
```

**`say()` 接受字符串或事件对象**：

```js
// 自动抽取
mind.say('你怎么这么蠢')

// 手动构造（当你有更好的判断时）
mind.say({
  label: '被否定',
  relevance: 0.8,
  congruence: -0.7,
  agency: 'other',
  coping: 0.5,
  norm: -0.3,
  certainty: 0.9,
  novelty: 0.2,
  intensity: 0.7,
})
```

**手动构造是逃生舱**：如果你的系统已经知道发生了什么（比如用户点了"骂人"按钮），就别让抽取器猜。

**`say()` 里带 `replied` 可以一步完成**：

```js
// 他说了一句，然后自己接了回来
mind.say('你怎么这么蠢', { replied: '对不起，我刚才太冲了' })
// → 先施加冒犯，再按 catch 处理回应
```

---

## 6. 调参指南

### 6.1 先调什么

**顺序很重要**，否则分不清是参数错还是结构错。

| 顺序 | 参数 | 怎么判断调对了 |
|---|---|---|
| 1 | `tauEmotion` | 吵架后 30 分钟还带余温，1 小时后淡了 |
| 2 | `tauMood` | 一整天不顺，晚上说话还是平的 |
| 3 | `dPMax` | 单次事件不能让人格变样 |
| 4 | `kP / kA / kD` | 三类事件的相对强度合理 |
| 5 | `relWarm / relCold` | 朋友说和陌生人说，疼得不一样 |
| 6 | `lGain / lRecover` | 连续糟糕几天会垮，好几天会缓 |

**只调权重，先固定 τ 和饱和。** 反过来的话，你永远不知道是时间常数错了还是增益错了。

### 6.2 怎么验证调对了

```js
// 跑一段剧本，打印轨迹
const mind = createMind()
for (const line of script) {
  mind.say(line)
  const a = mind.state.emo
  console.log(`${line} → P=${a.P.toFixed(2)} A=${a.A.toFixed(2)}`)
  mind.tick(5)
}
```

**看三件事**：
1. 有没有"跳变"（相邻两句差太多）
2. 有没有"卡死"（一直是同一个值）
3. 有没有"记不住"（上一句的影响完全消失）

### 6.3 参数速查

```js
// 情绪留得久一点
params.tauEmotion = 40

// 更不容易被激怒
params.kP = 0.7

// 朋友关系更重要
params.relCold = 0.8

// 更容易到顶点（更敏感）
params.peakThreshold = 0.5

// 压力恢复更快
params.lRecover = 0.0008
```

---

## 7. 已知边界

### 7.1 抽取器的极限

| 能做到 | 做不到 |
|---|---|
| 显式否定、程度、施事、命令 | 反讽（"真厉害啊"） |
| 常见冒犯词 | 暗示、阴阳怪气 |
| 20/20 归因正确 | 跨句指代 |
| 完全可审计 | 领域黑话 |

**反讽是最大的漏洞。** "你可真行"会被判成夸奖。

**补救**：如果检测到明显反讽（"真厉害" + 上下文负面），用 `say({ ... })` 手动构造覆盖。

### 7.2 顶点机制的触发条件

只在**密集连击**下有效：

| 事件间隔 | 到顶点需要几次 |
|---|---|
| 3-10 分钟 | 2 次 |
| 20 分钟 | 8 次内到不了 |

这不是缺陷——真实心理也是如此。但如果你想让"单次重击"也能形成顶点，把 `peakThreshold` 降到 0.35，并接受它更容易触发。

### 7.3 没有的东西

- **没有记忆**：它不知道"上次吵架是因为什么"
- **没有目标**：它不驱动行为，只调制措辞
- **没有道德**：它不知道什么该说什么不该说

**这三样都在人格层和记忆层，不在情绪层。** 这是刻意的分工。

---

## 8. 与 Python 版的关系

| 文件 | 用途 |
|---|---|
| `mind.mjs` | **生产用**，零依赖，接进你的 Node 系统 |
| `psy_model.py` | **参照实现**，用于验证、调参、画图 |
| `compare.mjs` | **对拍脚本**，保证两边行为一致 |

对拍结果：**133 项抽取 + 19 条读出 + 6 条关系放大，全部一致。**

改任何一边之后跑一遍：

```bash
python export_baseline.py
node compare.mjs
```

不一致会直接报出来，不会静默漂移。

---

## 9. 最小示例

```js
import { createMind } from './mind.mjs'

const minds = new Map()
const getMind = (uid) =>
  minds.get(uid) ?? (minds.set(uid, createMind({ trust: 0.5 })), minds.get(uid))

// 收到消息
function onMessage(userId, text, isResponded) {
  const mind = getMind(userId)
  mind.say(text, { responded: isResponded })
  mind.tick(3)

  // 组装 prompt
  const prompt = [
    '【人格】', persona,
    '', '【此刻状态】', mind.hint(),
  ].join('\n')

  // 调模型
  const reply = await callLLM(prompt, {
    temperature: baseTemp * mind.temperature(),
  })

  return reply
}

// 定时衰减
setInterval(() => {
  for (const mind of minds.values()) mind.tick(5)
}, 5 * 60 * 1000)
```

**这就是全部。** 没有框架、没有依赖、没有需要改动的现有代码。
