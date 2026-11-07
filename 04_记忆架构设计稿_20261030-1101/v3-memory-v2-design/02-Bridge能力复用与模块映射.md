# 02 · Bridge 能力复用与模块映射

> 对应 R1 与 §4「Scope and Complexity Review」。
> 原则：**先证明现有实现不够用，才允许新建。** 每一项复用都给出行号。

---

## 一、九类现有能力的复用决定（R1 逐条）

### 1.1 Token 实测账本（复用，不改一行）

| 项 | 内容 |
|---|---|
| 现有实现 | `src/token-ledger.js`（959 行） |
| 权威来源 | DSH `session/follow` 的 `projections.values.tokenUsage`（`token-ledger.js:6-10` 注释 + `ingestBaseline :452`） |
| 幂等键 | `sampleKey = sessionId\|turn\|step\|retry`（`:41-43`）；`seq` **严格更新**才覆盖（`:199-202`） |
| 容量策略 | 超 `maxLines=20000` 淘汰最旧，**桶累加进 `evicted` 保总量**（`:321-355`）；基线覆盖后不重复计（`:365-371`） |
| 复用方式 | **MemRev3 不修改、不包装、不重算**。只读它的 `summary()/conversation()/series()` 做验收取数 |
| 禁止 | ❌ 自建"请求级 token 统计" ❌ 用 `estimateTokens` 替代它 ❌ 在 MemRev3 里再存一份 token 记录 |

**技术理由（为什么不该重造）**：它已经解决了一个 Memory V2 完全没意识到的难题——**重连时 snapshot 重发导致的重复计费**（历史 bug 实测 +33%）。重写等于重踩一遍。

### 1.2 语义检索底座（复用）

| 组件 | 位置 | 复用方式 |
|---|---|---|
| `embed(text)` | `memory-vector.js:86-108` | 直接调用。Ollama `/api/embeddings`，30s 超时，失败 throw |
| `cosSim(a,b)` | `:110-120` | 直接调用（新索引里另写一个 `cosSimF32` 用于扁平数组，**因为入参形态不同**，属于"新增变体"而非"重复实现"） |
| `searchVectorMemory(key, topic, topK)` | `:171-199` | **保留为兼容入口**，内部改为调 `memory-recall`；旧调用点（`bridge.js:6872`）零改动 |
| 降级约定 | `:10-11` 注释：`embed` 失败 throw，调用方退回关键词路径 | 沿用同一约定；三通道各自独立降级 |

### 1.3 原子写（复用；仅修一处遗留缺陷）

| 项 | 内容 |
|---|---|
| 现有实现 | `atomicWriteJson()` / `atomicWriteText()`（`SECURITY_BASELINE.md` §2.5）——唯一随机 tmp 名 + `rename` + Windows EPERM 退避 |
| 已知缺陷 | `memory-vector.js:376-379` `const tmp = VEC_FILE + '.tmp'`——**固定 tmp 名**，并发写互相覆盖 |
| 复用方式 | 所有新模块统一调 `atomicWriteJson/Text`；**不新写原子写函数** |
| 需修的一处 | `memory-vector.js:376` 的固定名改为委托 `atomicWriteText`（+2 行） |

**判断**：这是"复用 + 打一个 3 行补丁"，不是"重写 atomic 子系统"。

### 1.4 每日备份（复用）

`dailyBackup()`，`memory-vector.js:131-147`：当天首次写入前快照到 `backups/`，保留 7 份。

| 复用方式 | 说明 |
|---|---|
| 直接调用 | MemRev3 的 `commit()` 在写 A 之前调用它（与现状一致） |
| 扩展 | 需要覆盖新文件（`impression-stats` / `review-queue` / `agent-token`）时，**在同一函数里加 file 列表参数**，不新写备份器 |
| 不覆盖 | prompt dump 排除在备份外（见 `04` §7） |

### 1.5 准入与会话校验（复用，完全不改）

| 组件 | 位置 | 用途 |
|---|---|---|
| `allowed(kind,id,cfg)` | `bridge.js:1358` | 谁能对话 |
| `modeAllowed()` | `:2666` | 模式门（closed-agent 仅 owner 私聊） |
| `isSessionAllowedInCurrentMode()` | `:2713` | key 语法 + 模式 |
| `isCurrentSession()` / `sessionPolicy()` | `:2723 / :2719` | 三重归属校验 |
| `canonicalV2Key()` | `:1370` | key 规范化（必须复用，见 `03` §6） |

**明确禁止**：把 `allowed()` 塞进记忆 `canRead()`。两者正交（R5 第 9 条 + `06-C-4`）。

### 1.6 工具面前缀与执行守卫（复用，不改）

| 项 | 位置 | 决定 |
|---|---|---|
| `SAFE_PREFIXES` | `qq-tool-restrict.mjs:215-220` | **不改**。记忆工具挂 `mcp__snowluma__` 即可（R12） |
| `tools.guard` | `:322-328` | 不改 |
| `tools.restrict` 名单 | `:120-208` | 不改 |

**回归影响**：新工具由于走同一前缀，**不需要动安全边界**，也就要不了额外的安全评审——这正是 R12 想要的。

### 1.7 出站审计与脱敏（复用，且免费获益）

| 组件 | 位置 | 对记忆的额外价值 |
|---|---|---|
| `SENSITIVE_RE` / `assertOutboundAuditOk` | `SECURITY_BASELINE` §2.3 | **记忆内容出站时自动过审**——即使召回出了敏感内容，也无法发到群里 |
| `redactKnownTokensOnly` | 同上 | 记忆文本里若含 token 会被脱敏 |
| CQ 转义 / @all 禁 | `onebotSend` | 记忆内容不会造成注入 |

**结论**：召回注入**不需要**再造一层出站过滤，现有链路已覆盖。这是本项目比 Rev2 设计假设更安全的地方。

### 1.8 单条消息压缩（复用）

`src/qq-model-view.js` `compactModelMessage()`（`bridge.js:24` 导入，`:10619` 用于快照）。

| 复用方式 | 召回结果注入前，若单条超过阈值，用同一函数压缩，**不自写裁剪逻辑** |
|---|---|
| 注意 | 该函数为"消息"设计（含 sender/时间），用于记忆文本时需要确认字段兼容【推断】；若不兼容，退化为 `truncateText` |

### 1.9 诊断与维护脚本约定（复用范式）

`scripts/` 87 个文件的既有约定（`FOLDER_MAP.md` §二）：

| 约定 | 沿用方式 |
|---|---|
| `test-*.mjs` 进 `test-audit.mjs` 清单 | MemRev3 的回归测试同样进清单 |
| `verify-*.mjs` 只读一致性校验 | `memory-maintenance.mjs` 的 `diff-projection` 属此类 |
| `probe-*.mjs` 只读探针，鉴权走 `probe-auth.mjs` | 不新增探针 |
| 破坏性脚本必须 `--dry-run` 默认 | `rebuild-index` 默认 dry-run |

### 1.10 原型污染防护（复用纪律）

现状：记忆写入/加载双侧拒绝 `__proto__` / `constructor` / `prototype`（`SECURITY_BASELINE` §2.5；`bridge.js:9253`）。

**要求**：新写的 `memory-store.js` / `permission.js` 必须在**读取与写入两侧**都做同样校验，否则是安全倒退。

---

## 二、九个新模块逐个论证（R4 要求：责任 / 为何现有不够 / 依赖 / 行数 / 改动文件 / 安全与回归）

### M01 `src/memory-store.js` — A 层权威写服务

| 项 | 内容 |
|---|---|
| **责任** | A 层唯一写入口：`commit(op)`；三版本；journal（WAL 简化版）；tombstone；脱敏；CRC；per-key 串行 |
| **现有为何不够** | `appendVectorMemory`（`:149`）只有"append"；修正走 `atomicRewrite`（`:376`）整文件重写；**无版本、无 journal、无墓碑、无 CRC**；5 条写入路径各写各的 |
| **依赖** | `atomicWriteJson/Text`、`dailyBackup`、`redactKnownTokensOnly`、`canonicalV2Key` |
| **行数** | ≈700 |
| **改动现有文件** | `memory-vector.js`（旧函数转发，+40/-25） |
| **安全影响** | 中：承载脱敏与 proto 防护；写错会影响所有记忆 |
| **回归影响** | 高：5 条旧路径必须继续工作（兼容适配器） |

### M02 `src/memory-permission.js` — 权限单一入口

| 项 | 内容 |
|---|---|
| **责任** | `classify(origin,text)` 产生 `contentClass`+`scope.read`；`canRead(record,ctx)` 五维判定；`assertReadable()` 断言 + 审计 |
| **现有为何不够** | 判定散在 `memory-vector.js:174-177` 两处 if；无内容分级、无来源维度、无 caller 维度（`03` §1） |
| **依赖** | 无（纯函数，最干净的一块） |
| **行数** | ≈240 |
| **改动现有文件** | `memory-vector.js` 两处过滤改调它；`bridge.js` 端点加断言 |
| **安全影响** | **极高**：这是隐私的唯一闸门 |
| **回归影响** | 高：改动检索过滤会直接影响群/私聊可见性 |

### M03 `src/memory-index.js` — 索引（含分词）

| 项 | 内容 |
|---|---|
| **责任** | 建/查 `vec.f32` + `ids.json` + `bm25.json` + `temporal.json`；`generation` / `coveredRev` / docId；中文 bigram 分词；`estimateTokens` |
| **现有为何不够** | 无索引：`loadRows()`（`:44-58`）每次整文件 `JSON.parse`；无分词；无 token 估算 |
| **依赖** | A 层、`embed`/`cosSim`（建索引时批量调用） |
| **行数** | ≈560（合并了 Rev2 的 M05+M06） |
| **改动现有文件** | 无 |
| **安全影响** | 低：纯派生数据，删了可重建 |
| **回归影响** | 中：索引滞后会导致"刚存的搜不到"，须有尾部补偿 |

### M04 `src/memory-recall.js` — 召回编排

| 项 | 内容 |
|---|---|
| **责任** | 通道注册表（3 内置 + graph 空位）、RRF、min-max、两遍预算、三层去重、五维断言、降级标注 |
| **现有为何不够** | 只有单路 `searchVectorMemory`；无融合/重排/预算/断言 |
| **依赖** | `memory-index`、`memory-permission`、`embed`、`cosSimF32` |
| **行数** | ≈420（三个通道内联，不拆文件——通道都很短，拆开反而增加导航成本） |
| **改动现有文件** | `bridge.js` 新端点 + `buildWakePromptV2` 注入块；`mcp-snowluma-safe.js` 新工具 |
| **安全影响** | 高：注入前的最后一道可见性断言 |
| **回归影响** | 中：新增注入块会影响 prompt 长度（见 `09` §4 的 cache 风险） |

### M05 `src/prompt-ledger.js` — 块账本（**不含人格、不含凭据**）

| 项 | 内容 |
|---|---|
| **责任** | 记录 4 个可变静态块（stickerList / slangTable / sleepRules / replyRules）的 hash、发送轮次；判定是否重发 |
| **现有为何不够** | 无账本，4 块每轮重发 |
| **依赖** | `state/prompt-ledger.json`（独立文件） |
| **行数** | ≈180 |
| **改动现有文件** | `bridge.js:10669` 注入块改造（+120） |
| **安全影响** | 低 |
| **回归影响** | **中高**：若 hash 判定错，会导致规则块不重发 → 行为漂移（见 `04` §5 的失效矩阵） |

### M06 `src/credential.js` — 凭据生命周期（R4 新增）

| 项 | 内容 |
|---|---|
| **责任** | 持有 per-key token + `epoch` + `issuedAt` + `needsDelivery`；轮转/重置时递增 epoch 并置 `needsDelivery=true`；提供 `consumeDeliveryFlag()` |
| **现有为何不够** | token 混在 `social-v2.json` 里，被 43 处整文件重写；`retireSession` 直接改字段（`:2795`）；Ledger 若管它会被轮转打穿 |
| **依赖** | `crypto`、`atomicWriteJson` |
| **行数** | ≈120 |
| **改动现有文件** | `bridge.js` `retireSession` / `/reset` 改调它（+40）；`buildWakePromptV2` 凭据块改由它提供（+15） |
| **安全影响** | **极高**：凭据 |
| **回归影响** | 中：两条退役路径对 token 的处理不一致（换新 vs 清除），Rev3 保留差异但统一记账 |

### M07 `src/memory-retain.js` — 形成

| 项 | 内容 |
|---|---|
| **责任** | 5 信号、黑名单、四闸、三池分流（A / candidates / quarantine）、补算队列 |
| **现有为何不够** | 只有 `infoOK()` 一道闸；写入靠模型自觉；无候选/隔离概念 |
| **依赖** | `memory-store.commit`、`memory-permission.classify`、`embed` |
| **行数** | ≈340 |
| **改动现有文件** | `bridge.js` 消息链路挂 `retainScan`（异步，+20） |
| **安全影响** | 高：第 4 闸决定什么能进 A |
| **回归影响** | 中：异步旁路不得影响回复延迟 |

### M08 `src/memory-reflect.js` — 整理

| 项 | 内容 |
|---|---|
| **责任** | 完全重复识别（一次一对）、过时标记、偏好提取、观察归纳 + 验证器、审核入队；**无删除权** |
| **现有为何不够** | 完全空白；`findMemoryRow`（`:235`）是"定位某条"，不是"两两比对" |
| **依赖** | A 层、`memory-permission`、`memory-store.setLifecycle` |
| **行数** | ≈480 |
| **改动现有文件** | `bridge.js` 空闲调度挂载（+60） |
| **安全影响** | 中高：可能误改 lifecycle |
| **回归影响** | 中：必须让出 socialLoopTick 的 CPU |

### M09 `scripts/memory-maintenance.mjs` — 维护

| 项 | 内容 |
|---|---|
| **责任** | `rebuild-index` / `diff-projection` / `verify-crc` / `repair-stats` / `dump-write-paths` |
| **现有为何不够** | `scripts/` 有 87 个脚本但无记忆维护；`memory-vector.js` 的 `dailyBackup` 只做备份 |
| **依赖** | 上述模块 |
| **行数** | ≈280 |
| **改动现有文件** | 无 |
| **安全影响** | 低（默认 dry-run） |
| **回归影响** | 低 |

### 2.1 汇总

| 模块 | 行数 | 安全影响 | 回归影响 |
|---|---|---|---|
| M01 store | 700 | 中 | 高 |
| M02 permission | 240 | **极高** | 高 |
| M03 index | 560 | 低 | 中 |
| M04 recall | 420 | 高 | 中 |
| M05 prompt-ledger | 180 | 低 | 中高 |
| M06 credential | 120 | **极高** | 中 |
| M07 retain | 340 | 高 | 中 |
| M08 reflect | 480 | 中高 | 中 |
| M09 maintenance | 280 | 低 | 低 |
| **合计** | **≈3,320** | — | — |

改动现有文件 4 个：`memory-vector.js`（+40/-25）、`bridge.js`（+395）、`mcp-snowluma-safe.js`（+70/-10）、`config.json`（+45，仅配置）。

---

## 三、明确"不新建"的清单（附 Rev2 对照）

| Rev2 计划 | Rev3 决定 | 理由 |
|---|---|---|
| 独立 token 计量模块 | ❌ 不做 | `token-ledger.js` 已有且更强 |
| 独立原子写封装 | ❌ 不做 | `atomicWriteJson/Text` 已有 |
| 独立备份模块 | ❌ 不做 | `dailyBackup` 已有 |
| 独立脱敏模块 | ❌ 不做 | `redactKnownTokensOnly` + 出站审计已有 |
| 3 个通道文件（semantic/keyword/temporal） | ❌ 合并为 1 | 每个通道 80-120 行，拆开增加导航成本 |
| `tokenize.js` 独立文件 | ❌ 并入 index | 只有 2 个调用方 |
| `memory-stats.js` 独立模块 | ❌ 并入 store 的统计写入 | 只有 1 个函数 |
| `memory-projection.js` 独立模块 | ❌ 并入 recall 的视图构建 | 投影就是"读 A 生成插槽"，无独立状态 |
| 新 MCP server | ❌ 不做 | 会动 `SAFE_PREFIXES`（安全边界） |

**净效果**：14 → 9 文件，4,470 → 3,320 行。

---

## 四、轻量 Scheduler vs 通用任务队列（R4 明确要求对比）

| 维度 | 方案 A：轻量 Scheduler（选定） | 方案 B：通用任务队列 |
|---|---|---|
| 形态 | 1 个 `setInterval`（60s）+ 每 key 游标 + 单次预算 200ms | 优先级堆 + 持久化 + 重试策略 + 背压 + 并发控制 |
| 行数 | ≈60 | ≈350-450 |
| 需要的能力 | Retain 是"来一条判一条"（同步旁路）；Reflect 是"定期扫一批"（批处理） | 需要"任务可重排/可取消/可依赖" |
| 实际需求盘点 | **没有**需要重排、依赖、取消的任务 | — |
| 风险 | 长任务会占住事件循环 → 用 200ms 预算 + 游标续跑解决 | 引入一个没人维护的调度器；失败语义要重定义 |
| 结论 | **选 A** | 不选：属于"不必要的抽象"（R4 明确禁止） |

**唯一借用的东西**：`token-ledger` 的"append + 定期 compact + 淘汰保总量"范式（`:273-378`），用在 `pending-embed.jsonl` 与 `review-queue.jsonl` 上。这不算引入队列，算复用文件增长策略。

---

## 五、依赖图（决定排期顺序）

```
M06 credential ──┐
                 ├─► M01 store ──► M03 index ──► M04 recall ──► 注入
M02 permission ──┘        │                        ▲
                          ├─► M07 retain ──────────┘
                          └─► M08 reflect（依赖 M04 才能验证观察有没有用）

M05 prompt-ledger ── 独立，可最先做（不依赖任何记忆模块）
M09 maintenance ── 依赖 M01/M03，可随时插
```

**并行机会**：`M05` 与 `M01` 可同时开工；`M02` 可与 `M01` 并行（纯函数，独立可测）。
**串行硬约束**：`M08` 必须在 `M04` 之后（否则无法验证"观察是否被正确召回"）。
