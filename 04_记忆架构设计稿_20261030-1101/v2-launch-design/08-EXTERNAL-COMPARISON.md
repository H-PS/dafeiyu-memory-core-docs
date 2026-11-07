# 08 · 外部方案对比与技术取舍

> 研究对象：Hindsight、Mem0、Zep/Graphiti、Letta/MemGPT。
> 说明：文中所有性能数字均为**厂商自published**，未经本项目独立复现，不作为选型依据，只作为"方向可行"的旁证。
> 结论先行：**没有任何一个可以整体引入**；四个里面各抄一部分，用现有 A/P/S 架构承载。

---

## 1. 四家的核心结构（一句话各一条）

| 系统 | 记忆的"单位"是什么 | 核心机制 | 最擅长 |
|---|---|---|---|
| **Hindsight** | 结构化的 fact（自足叙事单元，不是 512 token 切片） | Retain / Recall / Reflect 三操作；四路并行检索（语义 + BM25 + 图 + 时间）+ cross-encoder 重排；观察归纳带 evidence 与 proof count | 检索广度与归纳的可追溯性 |
| **Mem0** | 提取出的离散事实 | 提取 → 与已有记忆比对 → ADD/UPDATE/DELETE/NOOP；新版改为单遍 ADD-only（只增不覆盖） | 写入侧的选择性（少记但记准） |
| **Zep / Graphiti** | 实体（节点）+ 带有效期的事实边 | 双时态图：每条事实带 `valid_at / invalid_at`，旧事实失效不删除 | 时间维度查询（"搬家前的地址"） |
| **Letta / MemGPT** | 分层内存块 | Core（常驻上下文，agent 可自编辑）/ Recall（可检索历史）/ Archival（向量档案）；检索是 agent 的工具调用 | 上下文预算的自我管理 |

---

## 2. 逐项：值得直接借鉴的机制

### 2.1 从 Hindsight 抄三样

| 机制 | 抄法 | 落到哪个模块 |
|---|---|---|
| **四路并行检索 + 融合** | M1 抄三路（语义/BM25/时间），图留 M3 | `memory-recall.js` + `memory-channels/*` |
| **观察带 evidence + proof count** | observation 必须带 `evidence[{id, quote, at}]`，`proofCount` 随证据增长 | `memory-reflect.js` |
| **观察不覆盖、只追加历史** | 新证据支持→`metaRev++`；矛盾→新增而非改写 | A 层 `kind=observation` |

**不抄的**：cross-encoder 重排。理由见 §5.4——本地没有 reranker 服务，为一个重排环节引入 GPU 或外部 API，性价比不成立。

**本轮的加强**：Hindsight 的 observation 有 evidence 与 proof count，但没有"观察不得引入原始事实没有的信息"的强校验。本项目在生成后加了一道 `verifyObservation`（07 §5.3）：观察文本中的每个内容 bigram、数字、专名都必须能在 evidence 原文里找到，否则拒绝生成。这是比 Hindsight 更保守的做法，代价是规则版观察只能是模板句。

### 2.2 从 Mem0 抄两样

| 机制 | 抄法 | 落到哪个模块 |
|---|---|---|
| **写入侧的"选择性"** | Retain 四闸，宁可不记也不乱记 | `memory-retain.js` |
| **单遍 ADD-only（新版思路）** | 写入路径**永不 UPDATE/DELETE 已有记忆**；冲突只新增 + 标记 | 这条与我们"归纳不覆盖事实"天然一致，几乎零成本 |

**特别注意 Mem0 新版的设计转向**：它从"LLM 决定 ADD/UPDATE/DELETE/NOOP"改成了"单遍 ADD-only 提取，永远不覆盖不删除"。这个转向的理由和我们 V2.4.2 的结论完全一致——**自动 UPDATE/DELETE 太容易错，且错了无法回滚**。这是一个外部独立佐证：我们选 tombstone + lifecycle 而不是物理删除，是正确的。

**不抄的**：用 LLM 做 ADD/UPDATE/DELETE/NOOP 判定。每条消息一次模型调用，成本与延迟都不接受，且判定错误会静默改掉事实。

### 2.3 从 Zep 抄一样（只抄字段，不抄图谱）

| 机制 | 抄法 | 落到哪个模块 |
|---|---|---|
| **双时态（valid / invalid）** | 不建图，只在 A 层记录上加两个时间字段：`extra.validFrom` / `extra.validUntil` | A 层 Schema |

现有 `lifecycle` 只有 `active/stale/superseded` 三态，表达不了"某事在某段时间内为真，之后不再为真，但现在也没被否定"。加两个可空时间字段就能表达：

```json
"extra": { "validFrom": 1761000000000, "validUntil": 1761800000000 }
```

检索时：`validUntil` 存在且 < 查询时间窗起点 → 降权但不排除。这让"上个月他在准备考试"这类查询能正确命中。

**不抄的**：整个 Graphiti 时序图。理由见 §5.3。

### 2.4 从 Letta 抄一样

| 机制 | 抄法 | 落到哪个模块 |
|---|---|---|
| **分层内存 + 上下文预算自我管理** | 我们的 A/P/S 其实已经是分层了；缺的是"预算"这一层 | Block Ledger + 预算总闸（05 文档） |

Letta 的 Core Memory 思想对我们最有价值的一点：**常驻上下文的应该是"被压缩过的、稳定的"东西，不是原始记录**。这直接支持本文档 05 的两个决定：
1. 人格卡只发一次（稳定），不每轮重发；
2. 会话摘要（digest）替代原始历史。

**不抄的**：让 agent 自己决定"把什么提升到 core memory"。我们的模型是 QQ 聊天 agent，让它自己管理内存块会导致不可预测的注入行为，且难以测试。改成**固定结构 + 规则判定**，稳定性优先。

---

## 3. 逐项：哪些能用现有架构直接实现（不需要新模块）

| 能力 | 现有可复用的东西 | 改造量 |
|---|---|---|
| 语义检索 | `memory-vector.js:86 embed()` + `:110 cosSim()` + `:171 searchVectorMemory` | 直接复用，只加扁平化索引 |
| 权限过滤 | `memory-vector.js:174-177` 的 key/visibility 过滤逻辑 | 迁移到通道内，语义不变 |
| 脱敏 | `bridge.js` 的 `redactKnownTokensOnly()` | 直接复用 |
| 信息量门槛 | `memory-vector.js:125 infoOK()` | 直接复用为 Retain 闸 1 |
| 预算裁剪 | `bridge.js:10617 buildWakeSnapshotV2` 的 tryAdd + 回滚 | 复制这个模式到 recall 预算 |
| 消息快照 | `buildWakeSnapshotV2`（已有 `wakeMaxChars` 预算） | 不动 |
| 状态记忆（B 层） | `formatMemoryV2`（`bridge.js:9173`） | 加字符预算即可 |
| 审计 | 现有 audit 机制 | 加事件类型 |
| 三级鉴权 | `bridge.js:6865-6870` 四项校验 | 新端点照抄 |
| 会话轮转 | `retireSession()`（`bridge.js:8469`）+ `enqueueForRetry` | 直接复用 |
| 黑话表去重注入 | `withSlangContext()` 的"统一注入不重复"思路 | 作为 Block Ledger 的先例 |

**结论**：大约 60% 的能力是"已有代码 + 重新编排"，不是从零写。

---

## 4. 逐项：哪些必须增加模块（无法复用）

模块编号与行数以 `01-ARCHITECTURE.md` §8 为唯一口径，此处只说明"为什么必须新增"：

| 模块 | 为什么现有实现不了 |
|---|---|
| M01 `memory-store.js` | A 层目前由 `memory-vector.js` 兼职读写，没有 WAL/tombstone/三版本 |
| M04 `memory-permission.js` | 可见性判断散落在 `memory-vector.js` 的两处 if 里，无内容分级、无单一入口（S-11 的根因） |
| M05 `memory-index.js` | 现有检索是全文件 parse + 线性扫，无索引，无 generation/coveredRev |
| M07 `memory-recall.js` + M08/M09/M10 | 无多路/融合/重排/预算/断言 |
| M06 `tokenize.js` | 无中文切分，也无统一 token 估算 |
| M12 `memory-retain.js` | 无候选抽取与准入，写入完全依赖模型自觉 |
| M13 `memory-reflect.js` | 无任何整理能力 |
| M11 `prompt-ledger.js` | 无注入账本，静态块每轮重发 |

**合计 14 个新文件、约 4,470 行；改动 3 个既有文件，+790 / -75。没有一行需要新依赖**（node 内置 `fs/http/crypto` 足够）。

---

## 5. 逐项：哪些暂时没必要引入（及理由）

这一节最重要——技术方案的成熟度体现在"敢不做什么"。

### 5.1 向量数据库（Qdrant / Milvus / pgvector）

**不引入**。理由：
- 数据规模：单用户、几十个会话，A 层预估 5,000-50,000 条。
- 性能：`Float32Array` 全扫 10,000 条 × 1024 维 ≈ 8-15ms，已在预算内。
- 成本：引入外部服务 = 新进程 + 新端口 + 新故障点 + 新备份对象，与"可靠、可维护"目标相反。
- 索引本地的 `vec.f32` 已经是"扁平数组"，本质就是一个极简的向量库。

**什么时候才考虑**：单 key 超过 20 万条，或需要多用户并发检索。以本项目的形态，不会到。

### 5.2 图数据库（Neo4j）

**不引入**。M3 之前没有图查询需求。即使 M3，实体规模（单群几十人、几十个话题）用内存 Map 做邻接表足够。

### 5.3 完整时序知识图谱（Zep 路线）

**不引入**，只借两个时间字段。理由：
- 建图需要实体消歧（"老板"是哪个老板），这需要 LLM 或强规则，错误率高。
- 图查询的典型价值是"多跳关系"（朋友的朋友），QQ 群聊场景几乎没有这种需求。
- 维护成本：图一旦脏了（错误实体合并），修复比重建还难。

### 5.4 Cross-encoder 重排 / 外部 reranker 服务

**不引入**，M2 用可选 LLM 重排替代（默认关）。理由：
- 本地 Ollama 跑 bge-m3 embedding 没问题，但 reranker 模型（如 `bge-reranker-v2-m3`）在 Ollama 上支持不佳，需另起服务。
- 一次 rerank 对 15 个候选约 200-400ms，与"召回不得成为延迟瓶颈"冲突。
- 规则重排（相关性 + 时近性 + 通道一致性 + 类别权重）在本数据规模下差距有限。

### 5.5 让模型自主管理记忆（Letta 的 self-editing core memory）

**不引入**。理由：模型自己决定"把什么放进常驻上下文"在开放域 agent 上有效，在"人格固定、场景受限的聊天机器人"上会带来不可预测的注入与难以测试的行为漂移。

### 5.6 消息队列 / Redis

**不引入**。进程内 FIFO + 文件 WAL 已满足单机单写者场景。

---

## 6. 逐项：是否有现成组件可复用

| 需求 | 候选 | 结论 |
|---|---|---|
| 中文分词 | `nodejieba` / `segmentit` | **不用**。需编译或体积大；bigram + ASCII 切分对本场景足够（专有名词靠 ASCII 词，通用语义靠向量） |
| BM25 实现 | `wink-bm25-text-search` 等 npm 包 | **不用**。BM25 本身 40 行代码，引入包反而增加维护面；且我们需要中文 bigram 定制 |
| 向量检索 | `hnswlib-node` / `faiss-node` | **不用**。见 §5.1 |
| 时间解析 | `chrono-node` | **谨慎**。`chrono-node` 支持中文有限；我们的表达式集合是封闭的（今天/昨天/上周/上个月/N月N日），自己 80 行正则更可控。若后续要支持"下周三下午三点"再评估 |
| 摘要 | 现有 `webSearch` 的模型配置 | **可选复用**。M2 的 LLM 摘要在开启时复用同一模型配置，不新增密钥 |
| 嵌入 | 现有 Ollama + bge-m3 | **直接复用**。这是全项目最不该动的部分 |

**唯一"可以但不必"复用的是 `chrono-node`**——如果实现时发现正则维护成本高，可以降级引入；但不作为初始方案。

---

## 7. 四条路线的横向对照与我们的取位

| 维度 | Hindsight | Mem0 | Zep | Letta | **本项目（M1+M2）** |
|---|---|---|---|---|---|
| 记忆单位 | 结构化 fact | 提取的事实 | 实体+时序边 | 分层内存块 | **A 层记录（fact/observation/preference）** |
| 写入策略 | 抽取 + 实体解析 | ADD-only（新版） | 后台抽取 | agent 自决 | **模型主动 + 规则兜底 + 四闸** |
| 检索 | 4 路 + reranker | 语义+关键词+实体 | 语义+关键词+图 | agent 工具调用 | **3 路 + RRF + 规则重排** |
| 融合 | 有（有 reranker） | 有 | 有 | 无（agent 决定） | **RRF（无 reranker）** |
| 时间建模 | 时间戳索引 | 元数据过滤 | **双时态图** | 无 | **时间桶 + 两个可空字段** |
| 归纳 | observation + evidence | 无（只存事实） | 图更新 | 无 | **observation + evidence + proofCount** |
| 上下文管理 | token 预算（默认 4096） | 检索返回上限 | 检索返回上限 | **分层 + 自管理** | **Block Ledger + 预算总闸 + 轮转** |
| 存储 | Postgres | 向量库 + 图 | 图库 | Postgres | **JSONL + JSON（零服务）** |
| 部署 | Docker | 云/自托管 | 图库 + 服务 | 服务 | **单进程内** |

**取位说明**：我们把"检索广度"抄到 Hindsight 的 3/4，"写入选择性"抄到 Mem0，"时间建模"抄到 Zep 的 1/5（只要字段），"上下文预算"抄到 Letta 的思路但不给它自决权。存储与部署一律保持最简。

---

## 8. 一句话取舍总结

> 借 Hindsight 的**检索结构**，借 Mem0 的**写入克制**，借 Zep 的**时间字段**，借 Letta 的**预算意识**；
> 不借任何一个的**存储引擎**、**重排服务**、**图谱**、**自管理权**。

这个取舍完全可以在现有 A/P/S 上落地，不需要推翻架构。第 12 节（见 09 文档）给出最小必要改造清单。
