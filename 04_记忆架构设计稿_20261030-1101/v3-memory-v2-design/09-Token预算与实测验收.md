# 09 · Token 预算与实测验收

> 对应 R8 与 R3 的"降频须实测"要求。
> 一句话：**估算负责"装不装得下"，实测负责"到底省没省"。两者不许混用。**

---

## 一、两个概念，两套数据，两条报告

| | 估算（`estimateTokens`） | 实测（`token-ledger`） |
|---|---|---|
| 时机 | 拼 prompt 时（请求前） | 请求返回后 |
| 数据源 | 字符数换算（cjk×0.75 + other/4，×1.15） | DSH `projections.values.tokenUsage`（`token-ledger.js:6-10`）+ 逐轮 `usage` |
| 精度 | 粗（±15-30%） | 精确（provider 口径） |
| 用途 | 预算裁剪、是否装得下、日志里记个量级 | 成本核算、验收对比、控制台 |
| **禁止** | ❌ 用它报告"token 下降 X%" | ❌ 用它决定本轮装不下时是否硬塞 |

```
预算员：estimateTokens(promptText) ≤ budget  → 决定装几条
会计：  tokenLedger.conversation(key).turns[i].buckets → 报实际值
```

### 1.1 为什么这条要写死

`token-ledger` 已经解决了本项目一次真实事故（重复计费 +33%，`:357-371` 注释）。用估算值去覆盖这个结论，等于把已验证的会计换成估算，**并且会让"省了 45%"变成无法审计的口号**。

---

## 二、实测可取哪些字段（R8 要求）

`token-ledger` 的桶结构（`normalizeUsage` / `ZERO_BUCKETS`）：

| 字段 | 含义 | 报告里必须出现 |
|---|---|---|
| `cacheMiss` | 未命中缓存的 prompt token | ✅ |
| `cacheHit` | 命中缓存的 prompt token | ✅ |
| `cacheWrite` | 写入缓存的 token | ✅ |
| `output` | 输出（completion）token | ✅ |
| `reasoning` | 推理 token（若模型支持） | ✅（有则报） |
| **prompt 合计** | `cacheMiss + cacheHit + cacheWrite` | ✅ |
| **total** | `totalTokensOf(buckets)` | ✅ |
| `cost` | 折算金额（峰谷档位） | ✅（辅助） |

派生指标：
- `cacheHitRate = cacheHit / (cacheMiss + cacheHit + cacheWrite)`（`token-ledger.js:709`）
- `avgCostPerTurn` / `avgCostPerStep`（`:710-711`）

---

## 三、必须先讲清的风险：**prompt 前缀缓存**（R8 隐含，但不许忽略）

这是本设计里最容易被忽略、也最可能让"省 token"变成"更贵"的一点：

```
provider 的 prompt cache 通常按"前缀完全一致"命中。
Block Ledger 让第 N 轮的 prompt 从"含 X 块"变成"不含 X 块"，
⇒ 第 N 轮与第 N-1 轮的公共前缀变短
⇒ cacheHit 下降、cacheMiss 上升
```

**后果**：`cacheHit` 的单价通常只有 `cacheMiss` 的 1/5-1/10（`model-prices.js` 的 `off.cacheHit` vs `off.cacheMiss`）。所以：

> **省下的 prompt token × cacheMiss 单价，可能小于"缓存命中率下降"带来的额外成本。**

### 3.1 因此验收必须同时看三个数

| 指标 | 只降不升是好事？ |
|---|---|
| prompt tokens 总量 | ✅ 越少越好 |
| **cacheHitRate** | ❌ **下降是坏事**，必须一起看 |
| 折算 cost | ✅ 综合指标，最终看它 |

**明确要求**：任何"省 token"的结论，必须同时给出 `cacheHitRate` 变化。只报 prompt token 下降 = 报告不合格。

### 3.2 本设计对此的缓解

- 人格块**保持每轮**（位置固定，前缀稳定）—— 反而保护了缓存。
- 可变块尽量集中在 prompt **尾部**（快照、召回结果），让前部稳定。
- 规则块 ledger 化后，若导致前缀抖动，**优先调整块顺序**，而不是放弃 ledger。

---

## 四、基线协议（R8 的可复现要求）

### 4.1 对照条件（必须完全一致）

| 项 | 要求 |
|---|---|
| 剧本 | 固定 40 轮对话脚本（含：话题漂移、未完成话题、重复提及、人格试探、技术名词、时间指代） |
| 模型 | 同一 provider + 同一 model（记录在报告里） |
| preset | 同一 preset + 同一版本（记录 patch 文件 hash） |
| 人格卡 | 同一版本（记录 `roles/*.md` 的 sha256） |
| 起点 | 空记忆库（或同一份预置 200 条语料） |
| 时区/时钟 | 固定 `--fake-now`，避免峰谷档位漂移影响 cost 对比 |
| 轮次间隔 | 固定（避免触发不同的社交阶段） |

### 4.2 两组

| 组 | 配置 |
|---|---|
| A（基线） | 现状代码（Rev2 前）= 所有块每轮发 |
| B（Rev3） | Ledger 生效（4 块按需） |

### 4.3 取数

```bash
# 每轮记录（来自 token-ledger，不是估算）
node scripts/eval-token.mjs --dialogue fixtures/dialogue-40turns.json \
  --modes baseline,rev3 --fake-now "2026-10-31T18:00:00+08:00" \
  --out reports/token-YYYYMMDD.json
```

### 4.4 报告表（固定格式，缺列即不合格）

| 指标 | A 基线 | B Rev3 | Δ | Δ% |
|---|---|---|---|---|
| 第 1-20 轮 prompt tokens | — | — | — | — |
| 第 1-40 轮 prompt tokens | — | — | — | — |
| cacheHit tokens（1-40） | — | — | — | — |
| cacheMiss tokens（1-40） | — | — | — | — |
| **cacheHitRate** | — | — | — | — |
| completion tokens | — | — | — | — |
| 总 tokens | — | — | — | — |
| 折算 cost（同档位） | — | — | — | — |
| 单轮 P50 / P95 prompt tokens | — | — | — | — |

**"Δ%"列不得留空，也不得用估算值填。** 若某项数据不可得（如 provider 不返回 cache 分桶），写"不可得"，不得写 0。

### 4.5 不承诺固定百分比（R8 明确要求）

Rev2 写过"下降 ≥35% / ≥45%"。**Rev3 撤回这两个数字**，改为：

> 验收判据 = **B 组 cost 不高于 A 组，且 B 组 prompt tokens 低于 A 组**。
> 具体幅度由实测填入，不在设计阶段预设。

理由：
- 可节省上限由块构成决定（≈1,710 字符/轮，见 `04` §2.3），不是由愿望决定；
- prefix cache 的抵消效应无法先验估算；
- 一旦预设百分比，实施时就会去"凑数"（比如偷偷裁掉不该裁的块）。

---

## 五、人格频率的 A/B 协议（R3 要求，独立于上面）

只有在有人想**降低人格注入频率**时才需要跑。默认不跑，因为默认不改。

| 项 | 要求 |
|---|---|
| 剧本 | **40 轮**（不是 20 轮——前 20 轮人格通常还稳，漂移出现在后段） |
| 对照 | A：每轮全文；B：每 N 轮锚点（N=12 或 8） |
| 评价 | **盲评**：回复顺序打乱、隐去组别，由两名评分者独立按 1-5 分评"人格一致性" |
| 一致性 | 两人分差 > 1 分时第三人裁决 |
| 通过门槛 | B 组均值 ≥ A 组 − 0.3，**且** 最低分不低于 3 |
| 附加 | 第 40 轮单独检查：是否仍自称人格卡中的名字、是否符合语气 |

**未通过 ⇒ 参数不得启用。** 写进代码注释与配置默认值。

---

## 六、预算参数（`estimateTokens` 用）

| 块 | 预算 | 超预算策略 |
|---|---|---|
| 人格 | 3,000 字符硬上限 | 截断 + 告警 |
| 凭据 | 80 | 不截断 |
| B 层 | 800 字符 | topics > thoughts > impressions |
| 召回 | 600 token | 两遍装载（`06` §5） |
| 表情+参与度 | 700 字符 | 表情 ≤8 |
| 规则块（重发轮） | 700 字符 | — |
| 快照 | 6,000 字符 | tryAdd + 回滚（已有） |
| **总闸** | — | 砍序：snapshot → recall → bMemory → sticker；**规则与人格不砍** |

总闸超限时：记审计 `prompt_over_budget`，**仍投递**（宁可超预算，不可不回复）。

---

## 七、与现状的接口

| 现状 | Rev3 |
|---|---|
| 无 `estimateTokens` | 新增（`memory-index.js` 导出，全项目唯一实现） |
| `token-ledger` 照常运行 | **不改一行**；验收脚本只读它的 `summary/conversation` |
| 控制台 `/api/tokens/*` | 不变 |
| 无 prompt 长度统计 | `prompt-dump` 的元数据里记 `totalChars` / `estTokens`，供长期观测 |

---

## 八、验收判据（本文件负责的部分）

| # | 判据 |
|---|---|
| T-1 | 报告表 10 列全填（或明确写"不可得"） |
| T-2 | 必须同时报 `cacheHitRate` 变化 |
| T-3 | 结论口径 = "cost 不升 + prompt tokens 下降"，不得只报 token |
| T-4 | 估算值与实测值分别成列，不得混算 |
| T-5 | 两组跑的是同一剧本、同模型、同 preset hash、同人格卡 hash（报告里贴 hash） |
| T-6 | 人格频率未改（除 A/B 试验外），第 40 轮人格一致性人工检查通过 |
