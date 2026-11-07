# 05 · Memory V2 能力映射

> 对每个 Memory V2 模块判定：**直接复用** ／ **适配后复用** ／ **需要新增** ／ **不应该重复建设**
> 判定依据全部来自 00–04 文档的已确认证据。凡"不该重造"的，我会指出已有实现的位置。

---

## 一、总表（模块编号沿用 Memory V2 的 M01–M14）

| # | Memory V2 模块 | 现有对应 | 判定 | 理由（一句话） |
|---|---|---|---|---|
| M01 | `memory-store.js`（A 层 + WAL + 三版本 + tombstone） | `memory-vector.js`（380 行，append + 整文件重写） | **需要新增** | 现有只有"追加 + 全量重写"，无 WAL/tombstone/版本；这是真缺口 |
| M02 | `memory-projection.js` | 无（B 层直接存在 `social-v2.json`） | **需要新增** | 没有投影层概念，B 层与 A 层各存一份、无同步（项目全景"坑3：B 层 ≠ A 层，没有桥"） |
| M03 | `memory-stats.js`（拆 `interactionCount`） | `social-v2.json` 内 `memberImpressions[*].interactionCount` | **需要新增**（但可极小） | 真实存在整文件重写丢更新风险（S-1），拆文件是对的 |
| M04 | `memory-permission.js`（`canRead`） | 散在 `memory-vector.js:174-177` 的两处 if | **需要新增**（适配现有字段） | 现有无权函数、无内容分级、无来源维度；但必须复用现有 `key`/`visibility` 语义，不得另立门户 |
| M05 | `memory-index.js` | 无（全文件 `JSON.parse` + 线性扫） | **需要新增** | 无索引，`loadRows` 每次整文件解析（`memory-vector.js:44-58`） |
| M06 | `tokenize.js`（分词 + `estimateTokens`） | 分词：无；token：`token-ledger`（实测） | **拆分判定** | 分词需新增；**`estimateTokens` 不可替代 `token-ledger`**，两者职责不同（见 §4） |
| M07 | `memory-recall.js`（多路融合） | `searchVectorMemory`（单路，模型主动调） | **需要新增** | 单路、无融合、无重排、无预算；但通道 1 可直接复用 |
| M08 | semantic 通道 | `embed()` + `cosSim()` + `searchVectorMemory` | **直接复用** | 现成：bge-m3 / Ollama / 1024 维 / cosSim 全都有 |
| M09 | keyword 通道（BM25） | 无 | **需要新增** | 只有 4 级字符串匹配（`findMemoryRow`），那是"定位某条"，不是"检索" |
| M10 | temporal 通道 | 无（只有 `time` 字段 + 排序） | **需要新增** | 无时间表达式解析、无时间桶 |
| M11 | `prompt-ledger.js` | preset 静态块 + 每轮注入（无账本） | **需要新增，但必须降级目标** | 见 §5：现有"每轮全文注入人格"是已验证的必要条件，Ledger 不能照原设计砍 |
| M12 | `memory-retain.js` | 模型主动 `qq_memory_append` + `infoOK()` 守卫 + `dailyBackup` | **适配后复用 + 新增** | 写入管道、守卫、备份已有；缺的是"候选抽取 + 准入 + 三池" |
| M13 | `memory-reflect.js` | 无 | **需要新增** | 完全空白。但**没有删除权**这条要写进代码约束 |
| M14 | `memory-maintenance.mjs` | `scripts/` 87 个脚本里有 diagnose/verify/probe 体系 | **适配后复用** | 脚本范式成熟（幂等、只读探针、退出码约定），照这个风格写即可 |

---

## 二、六个重点能力逐项分析

### 2.1 A/P/S 三层

| 层 | 现有形态 | 与 Memory V2 的差距 | 判定 |
|---|---|---|---|
| **A**（事实源） | `state/memory-vectors.jsonl`，一行一条 `{key,text,time,vec,share_review_status,visibility,…}` | ① 无 `id`（靠 key+text 定位）② 无版本 ③ 无 `origin`（谁/哪个群/哪条消息）④ 无 lifecycle ⑤ 无 CRC ⑥ `time` 是写入时刻不是发生时刻 | **重构，不是重写**：Schema 是加法，现有 12 个字段都需要保留兼容 |
| **P**（投影） | **不存在**。B 层（`activeTopics`/`pendingThoughts`/`memberImpressions`）直接住在 `social-v2.json` 里 | 无 convVersion、无重建、无 digest、无 recallCache | **新增**。这是 Memory V2 最干净的一块（无历史包袱） |
| **S**（运行态） | `state/social-v2.json`（含 B 层 + 会话状态 + agentToken + wakeConfig） | 一个文件干三件事：运行状态、社交记忆、凭据。且被 43 处整文件重写 | **必须拆**：至少把 `agentToken`、`interactionCount`、B 层分开考虑 |

**关键判断**：**P 层是新增，A 层是加固，S 层是拆分**。三者工作量差异很大，不能笼统说"落地 A/P/S"。

### 2.2 Recall（召回）

| 能力 | 现状 | 判定 |
|---|---|---|
| 语义检索 | ✅ `embed` + `cosSim` + `searchVectorMemory(key, topic, topK)` | **直接复用**（通道 1） |
| 关键词检索 | ❌ 无 | 新增（BM25 + 中文 bigram） |
| 时间检索 | ❌ 无 | 新增（表达式解析 + 时间桶） |
| 多路并行 | ❌ 无（只有一路） | 新增（注册表 + allSettled） |
| 融合去重 | ❌ 无 | 新增（RRF，k=60） |
| 重排 | ❌ 无 | 新增（规则版；**不要 channelAgreement**） |
| 有效性检查 | 🟡 只有 `Array.isArray(r.vec)` 与 group 双闸 | 新增 lifecycle 过滤 |
| 降级 | ✅ **已有范式**：`embed` 失败 throw，调用方退回关键词路径（`memory-vector.js:10-11` 注释明确） | **适配后复用** |
| 触发方式 | ⚠️ 仅"模型主动调工具"，无自动注入 | 新增自动注入（接在 `buildWakePromptV2`） |

**现状的一个重要事实**：召回**完全依赖模型自觉**。项目全景"坑1：她只翻到当天那一页 → 能力在，说明书没写"就是这个问题的现场记录。修复动作当时是"补 description"（v21-memory-discovery），**不是加自动召回**。

⇒ Memory V2 的自动召回是**真正的新增价值**，也是唯一能绕开"模型不调工具"这个根本问题的路径。

### 2.3 Retain（形成）

| 子能力 | 现状 | 判定 |
|---|---|---|
| 写入管道 | ✅ `appendVectorMemory`（embed → 每日备份 → append → 清缓存） | **直接复用** |
| 信息量守卫（闸1） | ✅ `infoOK()`：≥6 字且 ≥3 汉字（`memory-vector.js:125-130`） | **直接复用** |
| 去重（闸2） | 🟡 `appendVectorMemory` **无去重**；只有 `findMemoryRow` 4 级匹配用于"分享定位" | 新增（但可复用 `findMemoryRow` 的归一化思路） |
| 价值判断（闸3） | ❌ 无 | 新增（5 信号） |
| 权限闸（闸4） | 🟡 只有写死的 `share_review_status:'not_requested'` | 新增（内容分级 + 来源推导） |
| 候选池 / 隔离区 | ❌ 无 | 新增 |
| 触发方式 | ⚠️ 全靠模型调 `qq_memory_append` | 新增规则兜底 |
| 备份 | ✅ `dailyBackup`（保留 7 份） | **直接复用** |

**已知历史问题（项目全景 P1-3）**："写入无去重 → 三条路径各写一份（运行/手工/回填）"。Memory V2 的 Retain 会成为第 5 条路径，**必须先收敛写入入口**，否则问题放大。

### 2.4 Reflect（整理）

| 子能力 | 现状 | 判定 |
|---|---|---|
| 重复识别 | 🟡 `findMemoryRow` 的 4 级匹配（逐字/双向包含/去前缀/归一化） | **思路可复用，能力不可复用**：它是"给定一句话找那条记忆"，不是"两两比对找重复簇" |
| 过时识别 | ❌ 无 | 新增 |
| 偏好提取 | ❌ 无 | 新增 |
| 观察归纳 | ❌ 无 | 新增 |
| 断言分类 | 🟡 A 层有 `category`（activeTopic 等）与 `sourceType`（observed/manual），但项目全景 P1-6 指出"sourceType 只有两种，无法区分回填与模型归纳" | 改造（扩枚举 + 加 `assertedBy`） |
| 纠错 | 🟡 管理端 `edit`（改文本 + 重算向量） | **复用 edit 路径** |
| 审核 UI | ✅ 控制台已有"记忆分享审核"面板（v23） | **适配后复用**：同一面板可承载 quarantine/observation 审核，但**两套状态机不能并存**（见 06 C-9） |
| 删除权 | ⚠️ 管理端能物理删（`removeVectorMemoryExact` / `removeNAll` / `removeAll`） | Reflect **不得**调用这三个函数 |

### 2.5 Block Ledger

| 项 | 现状 | 判定 |
|---|---|---|
| 已有的"预算"范式 | ✅ `buildWakeSnapshotV2` 的 tryAdd + 回滚 + partial（唯一） | **直接复用这个模式** |
| 已有的"去重注入" | ✅ `withSlangContext` 的"统一注入不重复"（`deliverPromptNow` 里黑话表只注入一份） | **直接复用这个思路** |
| 每轮重发的静态块 | ⚠️ 实测 7 块 ≈3,370 字符/轮 | Ledger 的目标 |
| 人格是否可降频 | **风险点** | 见 06 C-1：项目已用实践证明"每轮全文注入"是人格立住的必要条件 |
| 规则改动生效时机 | preset 改动需重启 DSH，人格改动即时生效 | Ledger 的 hash 必须区分这两种时机 |

### 2.6 权限隔离

| 能力 | 现状 | 判定 | 说明 |
|---|---|---|---|
| 会话级准入 | ✅ `allowed()` + `modeAllowed()` + 三重 session 校验 | **直接复用，不要重造** | 这是本项目最成熟的一块 |
| 工具面 | ✅ `restrict` + `guard` 精确前缀 | **直接复用** | 新工具必须挂 `mcp__snowluma__` |
| 出站审计 | ✅ `SENSITIVE_RE` + `redactKnownTokensOnly` | **直接复用** | 记忆内容出站同样过审（免费获得） |
| 记忆读取隔离 | 🟡 **只有 key 等值 + group 双闸** | **需要新增 `canRead`** | 这是真缺口，也是 S-11 的根因 |
| 来源维度 | ❌ A 层无 `origin` | **需要新增字段** | 没有来源，任何"按来源判权限"都做不了 |
| 内容分级 | ❌ 无 | **需要新增** | 现在"敏感"只在出站时拦，不在入库时分级 |
| 跨群共享 | ❌ 无路径 | **保持没有** | `copies` 字段先不要落地，避免开了口子没人管 |

---

## 三、"不应该重复建设"清单（最重要的一节）

以下 8 项**已经在项目里做得比 Memory V2 设计文档假设的更好**，照原计划重写就是浪费：

| # | 能力 | 现有实现 | 为什么不该重造 |
|---|---|---|---|
| 1 | **Token 计量与成本** | `token-ledger.js`（959 行，权威来自 DSH 投影，逐轮幂等，淘汰保总量，峰谷计价） | Memory V2 原设计只写了 `estimateTokens`。**实测账本比估算强一个量级**，且已验证过重复计费的坑（+33% bug） |
| 2 | **脱敏** | `redactKnownTokensOnly` + `redactSensitiveText` + 出站 `SENSITIVE_RE` | Memory V2 只提"复用 `redactKnownTokensOnly()`"，但没看到还有出站审计这一层——记忆内容出站会自动过审 |
| 3 | **原子写** | `atomicWriteJson/Text`（随机 tmp + rename + EPERM 退避） | 只需把 `memory-vector.js:376` 的固定 tmp 名换掉，而不是新写一套 |
| 4 | **每日备份** | `dailyBackup`（7 份滚动） | Memory V2 的 `memory-maintenance.mjs` 不必再实现备份 |
| 5 | **会话级准入与失效** | `allowed` / `modeAllowed` / `sessionPolicy` 三重校验 | 比 Memory V2 设想的一层 key 校验更严；直接复用 |
| 6 | **工具面白名单** | `restrict`（~100 名）+ `guard`（4 前缀 + 2 精确） | Memory V2 完全没提这一层，但它是"agent 能做什么"的真边界 |
| 7 | **单条消息压缩** | `qq-model-view.js` `compactModelMessage` | 快照已在用；召回结果注入可直接复用同一压缩 |
| 8 | **原型污染防护** | 写入/加载双侧拒绝 `__proto__`/`constructor`/`prototype` | 新写的 `memory-store.js` 必须继承这条，否则是安全倒退 |

---

## 四、`estimateTokens` 与 `token-ledger` 的职责边界（必须写清）

| | `token-ledger.js`（已有） | `estimateTokens`（Memory V2 新增） |
|---|---|---|
| 时机 | **事后**（请求已发出，usage 已返回） | **事前**（拼 prompt 时，还不知道实际值） |
| 精度 | 精确（provider 实测） | 粗略（字符数换算） |
| 用途 | 成本核算、控制台展示、验收基线 | 预算裁剪、是否装得下 |
| 单位 | prompt/completion/cache 分桶 | 只估总 token |
| 关系 | **会计** | **预算员** |

**禁止的写法**：用 `estimateTokens` 的结果去报告"token 下降了 45%"。必须用 `token-ledger` 的 turn 级实测值。估算值只能用于"这次能不能装下"。

---

## 五、结论：Memory V2 的真实增量

把 14 个模块拆开看，**真正需要从零写的只有 6 块**：

```
① 索引（M05）            因为现在每次全文件 parse
② 关键词通道（M09）       因为现在只有字符串匹配
③ 时间通道（M10）         因为现在没有时间语义
④ 融合/重排/预算（M07）    因为现在只有单路 topK
⑤ Reflect（M13）         因为完全没有
⑥ 通用任务队列（新增项）    因为现有 5 个队列都是场景专用
```

其余 8 块是"适配现有"或"直接复用"，其中 **4 块（token/脱敏/原子写/备份）属于明确的不该重造**。

**这个结论对排期的影响**：Memory V2 原估"新增 14 文件 4,470 行"，在本项目现状下应能压到 **10–11 个文件、约 3,200–3,500 行**——前提是承认现有底座（尤其 token 账本与工具面守卫）已经解决了它以为要解决的问题。

**但反过来**：原估的 16 周**不应缩短**。因为省下来的行数会转移到"适配"上——适配现有 43 处整文件重写、43 处 token 字面量、4 级匹配、junction 式 preset 生效链，这些比从零写更难，因为它们有既有行为要保。
