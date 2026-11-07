# 01 · Memory V2 整体架构设计（修订版）

> 修订：2026-10-31（第二轮定向修订）
> 权威数据模型见 `SCHEMA.md`。本文档与 SCHEMA 冲突时以 SCHEMA 为准。
> 基线：`D:\v2.4.2-revised\`（冻结）。本方案不构成实施授权。

---

## 1. 设计原则

| # | 原则 | 含义 | 违反示例 |
|---|---|---|---|
| P1 | **事实源唯一** | A 层是唯一事实源；索引/投影/摘要全部可重建 | 把 BM25 统计写进 A 记录 |
| P2 | **注入即成本** | 进入上下文的内容必须可计量、可裁剪、可关闭 | 每轮重发人格卡全文 |
| P3 | **先规则后模型** | 能确定性算出的分数不调 LLM | 用 LLM 给候选初排 |
| P4 | **归纳不覆盖** | 归纳产物与原始事实并列，不替换 | 合并 5 条为 1 条并删原 5 条 |
| P5 | **检索面收敛** | 并行只在内部，出边界只给少量高价值 | 四路 top20 全塞 prompt |
| P6 | **越权在通道内拦** | 每路检索打分前过滤，不做事后补丁 | 先 top50 再过滤 |
| P7 | **扩展靠注册表** | 新增通道/归纳器不改融合与预算 | 在 `recall()` 里加 if/else |
| **P8** | **权限与来源分离** | 「这条记忆从哪来」与「谁能读」是两个字段，不是一个枚举 | 用 `group_shared` 混表"群内可见"和"审核通过" |
| **P9** | **安全不靠提示词** | 越权拦截由代码执行，prompt 里只有提醒 | 把"不要泄露私聊"写进历史消息里指望模型记住 |
| **P10** | **可滞后不可领先** | 派生数据允许落后于事实源，绝不允许超前或残留 | 索引里有 A 层已删的 id |

---

## 2. 三层平面

```
┌────────────────────────────────────────────────────────────────────────┐
│ 平面 1 · 存储（继承 V2.4.2 + 修订）                                     │
│                                                                        │
│  A  state/memory-store.jsonl      事实源（不可重建）                    │
│  P  state/memory-projection.json  投影（可重建，依赖 A+S）              │
│  S  state/social-v2.json          运行态（不可重建）                    │
│  T  state/memory-tombstones.jsonl 墓碑（永久）                          │
│  ────                                                                  │
│  I  state/memory-index/           派生索引（可重建；**允许滞后**）      │
│     ├ meta.json    generation / coveredRev                             │
│     ├ vec.f32      Float32Array 扁平向量                               │
│     ├ ids.json     docId ↔ memId                                       │
│     ├ bm25.json    倒排索引                                            │
│     └ temporal.json 时间桶（按本地日）                                  │
└────────────────────────────────────────────────────────────────────────┘
        ▲ 派生（单向，只读 A）
        │
┌────────────────────────────────────────────────────────────────────────┐
│ 平面 2 · 召回（M1）                                                     │
│                                                                        │
│  semantic ─┐                                                          │
│  keyword  ─┼─► RRF 融合 ─► 规则重排 ─► 预算装载(两遍) ─► 去重 ─► 断言  │
│  temporal ─┘                                                          │
│  (graph M3)                                                            │
│                                                                        │
│  横切模块：permission（每通道打分前）/ tokenize / tokens（统一计量）     │
└────────────────────────────────────────────────────────────────────────┘
        ▲ 消费
        │
┌────────────────────────────────────────────────────────────────────────┐
│ 平面 3 · 认知（M2）                                                     │
│                                                                        │
│  Retain  信号 → 四闸 → 三池分流（A / candidates / quarantine）           │
│  Reflect 完全重复/过时/偏好/观察/断言分类 → 只改 lifecycle，无删除权      │
└────────────────────────────────────────────────────────────────────────┘
        │
        ▼
  消费端：buildWakePromptV2（唯一自动注入点）/ qq_memory_recall / 管理端
```

---

## 3. 权限模型（P0-1 核心修订）

### 3.1 问题

上一版把三件事塞进两个字段：**来源**（哪个群/私聊产生）、**读取范围**（谁能读）、**审核状态**（是否人工批准）。

结果：`visibility: group_shared` + `share_review_status: approved` 联合表达"群内可见且已审"，而 `not_requested` 同时意味着"没申请"和"不可见"。语义耦合导致 S-11（群聊记忆恒不可检索）无法靠调参修复。

### 3.2 修订后的三字段模型

```
origin.groupId     来源群（事实，不可改）
scope.read         owner_only | origin_group          ← 读取范围
scope.contentClass public | sensitive | uncertain     ← 内容分级（决定能否群内可见）
review.status      not_required | pending | approved | rejected  ← 仅人工审核语义
```

**判定规则（唯一入口 `canRead`，见 SCHEMA §9）**：

| 场景 | 条件 |
|---|---|
| 群 G 内检索到本条 | `record.key === 'group:G'` ∧ `scope.read === 'origin_group'` ∧ `scope.originGroupId === G` ∧ `scope.contentClass === 'public'` ∧ `lifecycle ∈ {active, stale}` |
| 群 H 内检索到本条（H≠G） | **恒 false**（无 copies 时不存在跨群路径） |
| 私聊检索到群里产生的本条 | **恒 false**（key 严格相等） |
| 私聊检索到私聊产生的本条 | `key === 'private:<ownerQq>'` |
| 管理端查看 | 仅生命周期过滤，不受 scope 限制 |
| Reflect 处理 | 只能处理 `key` 相同且 `lifecycle ∈ {active, proposed, stale}` |

**`auto_shared` 状态值正式废弃**。表达"群内自产、无需人工审"用 `scope.read='origin_group'` + `review.status='not_required'` 两个字段，不再新增枚举。

### 3.3 内容分级（决定"能不能自动群共享"）

| contentClass | 触发条件（规则） | scope.read | 后果 |
|---|---|---|---|
| `public` | 无敏感命中 | `origin_group`（来源是群时） | 群内正常召回 |
| `sensitive` | 命中：手机号/证件/住址/真实姓名/金额/账号密码/医疗/情感隐私/第三人隐私 | `owner_only` | 该群也召回不到；进 quarantine 等主人审 |
| `uncertain` | 命中模糊信号：代词指代不明（"他""那个"）、含第三方评价、"别告诉别人"类措辞、分类器置信度不足 | `owner_only` | 同上 |

**分类器是规则版，不是 LLM**：命中即降级，宁可隔离不可放过。规则表是封闭集合，写在 `memory-permission.js` 顶部常量，可测试、可复查。

**只降不升**：自动流程只能把 `public` 降为 `sensitive/uncertain`，**永远不能**把 `sensitive` 提回 `public`。提升只能由主人通过管理端操作（INo-15）。

### 3.4 为什么这样能同时满足"群内可用"与"不越权"

- 群内自产、无敏感的普通记忆 → 立刻可召回（修复 S-11）。
- 涉及隐私的 → 降为 `owner_only`，**即使同一群也召回不到**（比"群内可见"更严格）。
- 跨群 → 无任何自动路径（`copies` 恒空，且当前无写入端点）。
- 私聊 → 任何群都读不到（key 严格相等）。

**没有放宽任何跨边界可见性**：放宽的只有"同群内、非敏感、自产"这一类，而这类信息本来就已在该群公开过。

### 3.5 权限失败的处置

| 失败原因 | 去处 | 能否自动晋升 |
|---|---|---|
| `contentClass ∈ {sensitive, uncertain}` | `state/memory-quarantine.jsonl`（`lifecycle='quarantined'`） | **否**，只能主人审 |
| `scope` 推导异常（groupId 缺失等） | quarantine，`classReason='ambiguous'` | 否 |
| 其他闸失败（新颖度/价值） | `state/memory-candidates.jsonl` | 否（只能重新过闸，见 06） |

**quarantine 与 candidates 是物理分离的两个文件**。理由：candidates 会被自动再评估，quarantine 绝对不能。

---

## 4. 数据一致性模型（P0-4 核心修订）

### 4.1 上一版的矛盾

上一版同时写了两条互斥的话：

- "索引更新必须与 A 层写在同一个 WriteQueue 临界区"
- "embed 在临界区外"

索引更新依赖向量，于是必然：要么把 30s 的 embed 塞进锁里（不可接受），要么索引更新在锁外（与第一条矛盾）。

### 4.2 修订：区分"必须原子"与"允许滞后"

| 操作 | 原子性要求 | 实现 |
|---|---|---|
| **删除 / clear** | 必须与 A 层同时生效 | 同一临界区：A 记录 lifecycle→deleted + 索引立即移除 docId + 墓碑写入 |
| **追加 / edit** | 只需 A 层先落地 | A 提交后异步更新索引（`indexQueue`） |
| **向量写入** | 允许滞后 | `embedding.status='pending'` → 后台补算 → 更新 `vecContentRev` |

**依据**：滞后在两个方向上的危害不对称。

- 追加滞后 → "刚存的记忆暂时搜不到"。可接受，有尾部扫描兜底。
- 删除滞后 → "删掉的记忆还能被搜到"。**不可接受**（幽灵记忆、隐私残留）。

因此采用**非对称策略**：写异步、删同步。

### 4.3 `coveredRev` 与尾部扫描

```
索引 meta.coveredRev = 已进入索引的 A 层最大 rev

召回时：
  hits = 索引检索(...)
  tail = A 层中 rev > coveredRev 且 canRead(...) 的记录
  if (tail.length > 500) { 触发同步重建；本次返回 degraded=true }
  else { tail 现场打分（同算法）→ 与 hits 按 id 合并，取较高分 }
```

尾部通常 < 100 条（异步队列秒级追平），现场打分 < 5ms。这条机制让"索引可滞后"成为可实现的设计。

### 4.4 提交协议（统一 WriteQueue / WAL / Tombstone / clearEpoch）

```
阶段 0 归一化          脱敏、截断、fingerprint、内容分级（无 IO）
阶段 1 预检            内存 Map：tombstone 命中？状态允许？
                       记 observedRev = 当前 maxRev, observedEpoch = clearEpoch
阶段 2 embed（锁外）   失败则 embedding.status='pending'，不阻断写入
阶段 3 入队（持 per-key 锁）
  3.1 再预检           tombstone 二次查 + observedEpoch 校验（内存比较）
  3.2 写 WAL pending  携带 opId / effects[] / observedRev / observedEpoch
  3.3 应用 effects    A 记录写（append 或原子重写）/ 删除时同步移除索引
  3.4 写 WAL committed
  3.5 释放锁
阶段 4 出队后          异步：索引追加、向量补算、未读游标、审计
```

**五个崩溃点恢复语义**（统一为"set-to-state"，非 delta）：

| 崩溃点 | WAL 状态 | 恢复动作 |
|---|---|---|
| embed 中 | 无 WAL | 丢弃，无副作用 |
| 3.2 后 3.3 前 | pending | 重放 effects（幂等） |
| 3.3 中（A 写完、索引未删） | pending | 重放：A 已是目标态→跳过；索引删除重放 |
| 3.4 中 | pending | 重放 effects（全部已是目标态）→ 补写 committed |
| 3.4 后 | committed | 只补做阶段 4 的异步项 |

**关键**：effects 只携带**目标值**（`lifecycle='deleted'`），不携带 delta（`rev += 1` 由 store 内部按"是否真发生变更"决定），因此重复应用安全。

---

## 5. 并发串行化（回应「不允许用写前重读替代串行化」）

三层机制，缺一不可，且**没有一层是"重读文件"**：

| 层 | 机制 | 防什么 |
|---|---|---|
| L1 | `bridge.lock`（已有） | 多实例同时写 |
| L2 | **per-key 异步互斥锁**（`Map<key, PromiseChain>`，进程内） | 同 key 并发写互相覆盖；持锁范围 = 阶段 3 全程 |
| L3 | **WAL opId 唯一约束**（committed 日志内去重） | 重试导致重复提交 |

文件重读（`loadRowsStrict`）只用于：① 进程启动时恢复内存状态 ② 检测外部篡改。**不作为并发控制手段**。

**明确否定的写法**（上一版遗留，本版删除）：

> "每次写前重读文件，避免旧快照" —— 这不是串行化，而是让两个写者都可能读到同一个旧值后各自写回。正确做法是互斥锁 + fencing token。

### 5.1 检查-提交窗口

`observedEpoch` 是 fencing token：

```
持锁 → 校验 st.clearEpoch === observedEpoch → 提交
```

因为 `clear` 必须获取同一把 per-key 锁，"检查通过后、提交前被 clear"的窗口已被锁本身消除。`observedEpoch` 只是防"锁实现出错"的第二道保险。

---

## 6. P 层重建规则（与依赖一致）

```
rebuild(key) := f(
  A 层: key 相同 且 lifecycle ∈ {active, stale} 且 kind=fact,
  S 层: 该 key 的 recentMessages（仅 digest 使用）,
  S 层: 该 key 的 clearedKeys.convVersion
)
```

- 不读其他 key 的数据。
- 不读索引（索引是 A 的派生物，读索引等于读旧 A）。
- 产物写临时文件 + 原子 rename；失败时旧文件保持可用。
- 容差：除 `builtAt` / `digest.dirty` / 键序外，**0 差异**。

---

## 7. 不变量（INo-1 ~ INo-16）

| 编号 | 不变量 | 检验 |
|---|---|---|
| INo-1 | 单写者（A 层只由 `memory-store.js` 写） | grep 静态检查 |
| INo-2 | 无旧快照回写（写路径必须持锁） | 并发写入测试 |
| INo-3 | 单点生效（同事务同 type effect 唯一） | WAL 构造期校验 |
| INo-4 | 三版本单调（`rev ≥ contentRev ≥ vecContentRev`） | 随机操作 100 次断言 |
| INo-5 | opId 幂等 | 重复提交测试 |
| INo-6 | 永不越权修改（群聊不得删私聊记录） | 权限矩阵测试 |
| INo-7 | 向量不追新（`vecContentRev ≤ contentRev`） | 断言 |
| INo-8 | 元数据不阻断内容（仅 metaRev++ 不重算向量） | 断言 |
| INo-9 | A/P 不互相触发（重建不写 A） | 代码审查 |
| **INo-10** | 索引可滞后、不可领先、删除必须即时 | 删除后立即检索必须为空 |
| **INo-11** | 同一条记忆同会话 3 轮内不重复注入 | ledger fingerprint 测试 |
| **INo-12** | 删除全部 observation 后，事实问答仍正确 | 删 observation 回归 |
| **INo-13** | 召回结果 ⊆ `canRead` 通过集合 | 越权注入日志断言 |
| **INo-14** | 预算装载两次同 query 同预算结果一致 | 复现性测试 |
| **INo-15** | 自动流程只能降级 contentClass，不能升级 | 代码审查 + 用例 |
| **INo-16** | `proposed` / `quarantined` 不进召回与注入 | 状态机测试 |

---

## 8. 模块清单（统一编号，全文档引用此表）

| # | 文件 | 类型 | 行数 | 里程碑 |
|---|---|---|---|---|
| M01 | `src/memory-store.js` | 新增 | 750 | M0 |
| M02 | `src/memory-projection.js` | 新增 | 380 | M0 |
| M03 | `src/memory-stats.js` | 新增 | 150 | M0 |
| M04 | `src/memory-permission.js` | 新增 | 220 | M0 |
| M05 | `src/memory-index.js` | 新增 | 520 | M0(写) / M1(查) |
| M06 | `src/tokenize.js` | 新增 | 90 | M1 |
| M07 | `src/memory-recall.js` | 新增 | 560 | M1 |
| M08 | `src/memory-channels/semantic.js` | 新增 | 120 | M1 |
| M09 | `src/memory-channels/keyword.js` | 新增 | 130 | M1 |
| M10 | `src/memory-channels/temporal.js` | 新增 | 140 | M1 |
| M11 | `src/prompt-ledger.js` | 新增 | 170 | M1 |
| M12 | `src/memory-retain.js` | 新增 | 380 | M2 |
| M13 | `src/memory-reflect.js` | 新增 | 540 | M2 |
| M14 | `scripts/memory-maintenance.mjs` | 新增 | 320 | M0/M1 |

**合计：新增 14 个文件，约 4,470 行。**

| 文件 | 改动 |
|---|---|
| `src/memory-vector.js` | +70 / -15 |
| `src/bridge.js` | +600 |
| `src/mcp-snowluma-safe.js` | +120 / -60 |
| **合计** | **+790 / -75** |

---

## 9. 明确不引入的组件

| 组件 | 不引入理由 | 何时才考虑 |
|---|---|---|
| 向量数据库 | 万级数据 `Float32Array` 全扫 <15ms；引入=新进程+新故障点 | 单 key >20 万条 |
| 图数据库 | M3 前无图查询；实体规模几十 | 永不（内存邻接表足够） |
| Cross-encoder reranker | 本地无服务；200-400ms 拖慢回复 | 有 GPU 常驻服务时 |
| 消息队列 | 进程内 FIFO 足够 | 多机部署时 |
| 外部记忆框架 | 存储引擎与项目强绑定，迁移成本 > 收益 | 永不整体迁移 |

详见 `08-EXTERNAL-COMPARISON.md`。
