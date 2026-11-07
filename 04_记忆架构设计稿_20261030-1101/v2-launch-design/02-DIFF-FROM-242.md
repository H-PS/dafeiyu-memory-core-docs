# 02 · 与 V2.4.2 的差异与修订说明（修订版）

> 本文档逐条处理 V2.4.2 遗留的 S-1 ~ S-10、上一轮方案新发现的 S-11，以及本轮修订新增的 S-12 ~ S-16。
> 每条给：问题 → 证据 → 裁决 → 影响面 → 阻塞里程碑。裁决若与其他文档冲突，以本文档 + `SCHEMA.md` 为准。

---

## 0. 修订总览

| 编号 | 一句话 | 严重度 | 状态 | 阻塞 |
|---|---|---|---|---|
| S-1 | `interactionCount` 在会被整文件重写的 `social-v2.json` | P1 | 已裁决（拆文件） | M0 |
| S-2 | `e3 S_stats` / `e4 S_compat` 双 effect 写同一文件 | P1 | 已裁决（合并单 effect） | M0 |
| S-3 | `clear` 后 P 层语义未定义 | P1 | 已裁决（删条目 + 墓碑） | M0 |
| S-4 | 删除不存在记录的幂等语义 | P2 | 已裁决（no_op 成功） | M0 |
| S-5 | WAL 提交前不复查 tombstone | P2 | 已裁决（持锁后二次查） | M0 |
| S-6 | P 重建容差未定义 | P2 | 已裁决（0 差异） | M0 |
| S-7 | 缺崩溃点覆盖验收 | P3 | 已补入 11 文档 | — |
| S-8 | 缺索引一致性验收 | P3 | 已补入 11 文档 | M1 |
| S-9 | 缺降级路径验收 | P3 | 已补入 11 文档 | M1 |
| S-10 | "5 条红线"实为 8 条 | P3 | 已更正 | — |
| **S-11** | 群聊自产记忆恒不可检索 | P1 | **重新裁决**（三字段权限模型） | M0 |
| **S-12** | 索引更新既要求原子又要求锁外（自相矛盾） | **P0** | 已裁决（写异步/删同步） | M0 |
| **S-13** | `agentToken` 作为 prompt 内容并作为工具参数传递 | **P0** | 已裁决（带外绑定，分两阶段） | M1 |
| **S-14** | `proposed` 状态"降权注入"违反归纳安全 | **P0** | 已裁决（默认不注入） | M2 |
| **S-15** | 单链聚类批量 supersede 会成片误伤 | **P0** | 已裁决（取消批量，改三档） | M2 |
| **S-16** | 候选池混装不同失败原因，等于给权限失败开了后门 | **P0** | 已裁决（三池分离） | M2 |

---

## S-1 · `interactionCount` 存储位置（P1，阻塞 M0）

### 问题与证据

`bridge.js:9199` 在 `formatMemoryV2` 读 `im.interactionCount`；`bridge.js:9258-9262` 在 `appendMemoryV2` 做 `+1` 后整体替换 `st.memberImpressions[target]`。`st` 来自 `social-v2.json`，该文件被 43 处整文件重写路径覆盖。

后果：并发写时 `+1` 静默丢失，表现为"互动次数偶尔回退"。

### 裁决

拆出 `state/memory-impression-stats.json`，复合键 `key + \u0000 + 群友名`，唯一写者 `memory-stats.js`。

**关键约束（修订）**：上一版写"每次写前重读文件"。本版改为**持 per-key 锁内的键级 read-modify-write**，重读只是恢复手段，不是并发控制：

```js
await withKeyLock(`stats:${key}`, async () => {
  const s = loadStats();               // 内存态，mtime 变化时才重读
  s.stats[ck].interactionCount += 1;
  s.rev += 1;
  atomicWrite(s);                      // tmp(pid+rand) + rename
});
```

`social-v2.json` 的 `memberImpressions[name]` 移除 `interactionCount`；`formatMemoryV2` 改读 stats 文件，读不到显示"互动次数暂无"。

---

## S-2 · 双 effect 写同一文件（P1，阻塞 M0）

### 裁决

合并为单一 effect `S_state`，一次原子写覆盖事务对 S 层的全部变更。

```
effects: [
  { type: 'A_append',    payload: {...} },
  { type: 'S_state',     payload: { target: {...} } },   // 目标值，非 delta
  { type: 'INDEX_update',payload: {...} }
]
```

硬规则：同一事务同 type 最多一次；构造期 `validateEffects()` 拒绝重复，fail-fast 不入 WAL。

`S_state` 的 application 是"set 到目标状态"，因此重复应用安全。

---

## S-3 · `clear` 后 P 层语义（P1，阻塞 M0）

### 裁决

**删除投影条目 + S 层 key 级墓碑**。

```jsonc
"clearedKeys": { "group:123456": { "clearedAt": 0, "clearEpoch": 88 } }
```

1. `clear` 事务：删 A 记录（lifecycle→deleted + 墓碑）→ 删 P 条目 → `clearEpoch += 1`。
2. 重建时若 key 不在 P 但存在于 `clearedKeys`，以墓碑 `clearEpoch` 为起点，不从 0 开始。
3. 墓碑不参与注入与检索，只在重建时读一次。
4. 清理：该 key 新增 ≥1 条记忆后由投影器顺手移除墓碑（保留 epoch 值，见 §7 语义）。

**与上一版的差异**：上一版把 `convVersion` 放在墓碑里；本版拆成 `clearEpoch`（fencing）与 `convVersion`（投影代数）两个独立计数器，语义更清晰（见 SCHEMA §7）。

---

## S-4 · 删除不存在的记录（P2）

**裁决**：成功且不增 `rev`。

```
{ ok: true, removed: 0, noop: true }     // 响应体不含任何记忆正文
```

安全约束：`noop` 响应用于避免"用删除接口探测存在性"；审计记 `result:'no_op'`。

---

## S-5 · 提交前复查 tombstone（P2）

**裁决**：持锁后、写 WAL 前二次查询（内存 Map，< 1μs）。命中则 abort，返回 `{ok:false,error:'目标已删除'}`。

---

## S-6 · P 重建容差（P2）

**裁决**：0 差异。白名单仅 `builtAt` / `digest.dirty` / 键序。由 `scripts/memory-maintenance.mjs diff-projection` 在 CI 断言，运行时不做逐字节校验（拖慢重建）。

---

## S-7 / S-8 / S-9 · 验收缺口（P3）

已并入 `11-ACCEPTANCE.md`：

- S-7：5 崩溃点 × 4 操作 = 20 用例，全部要求"恢复后 A/P/索引三者一致"。
- S-8：索引一致性 —— `索引 id 集合 == {A 层 active ∪ rev > coveredRev 的尾部}`。
- S-9：四种降级（Ollama 不可用 / 索引损坏 / 索引缺失 / 索引滞后超限），均要求**显式向模型标注**。

---

## S-10 · 红线计数（P3）

更正为 8 条，逐条给检验方式（见 12 文档 §4）。

---

## S-11 · 群聊自产记忆恒不可检索（P1，**重新裁决**）

### 问题与证据（不变）

1. `memory-vector.js:157` 写入默认 `share_review_status: 'not_requested'`
2. `memory-vector.js:176-177` 群 key 检索强制 `visibility==='group_shared' && share_review_status==='approved'`
3. `bridge.js:6880-6895` 共享申请端点拒绝 `private:` 前缀，只允许 group key 申请

结论：群聊写入的 A 层记忆，不逐条人工审批就永远检索不到 → 群聊长期记忆失效 → 验收 #10 在群聊场景必失败。

### 上一版裁决的问题（本轮修正）

上一版提出引入 `auto_shared` 状态值，让群 key 写入默认可见。这解决了"恒不可检索"，但引入了三个新问题：

1. **状态值语义混淆**：`share_review_status` 本是"审核状态"，塞进一个"自动通过"的值，等于把"没审"伪装成"审过了"。
2. **无内容分级**：群里说的私事会被自动群共享，比原设计更危险。
3. **没有跨群边界**：`group_shared` 字面意思不限定原群，后续若加跨群能力会直接踩坑。

### 本轮裁决（三字段权限模型）

改用 `origin.groupId` + `scope.read` + `scope.contentClass` + `review.status` 四字段（完整定义见 `SCHEMA.md` §1、`01-ARCHITECTURE.md` §3）：

| 场景 | 结果 |
|---|---|
| 群 G 内自产、内容 `public` | `scope.read='origin_group'`，`contentClass='public'`，**立即可在同群召回**（修复 S-11） |
| 群 G 内自产、内容 `sensitive/uncertain` | `scope.read='owner_only'`，进 quarantine，**同群也召回不到**（比原设计更严） |
| 群 G 的记忆出现在群 H | 恒 false（`originGroupId` 必须严格等于当前群） |
| 私聊记忆 | `scope.read='owner_only'`，任何群恒 false |

**`auto_shared` 废弃**，"群内自产免审"由 `review.status='not_required'` 表达。

### 影响面

- `memory-vector.js`：写入默认值（`:157`）、检索过滤（`:176`）、`listVectorMemory` 过滤器（`:217`）—— 三处，改为调用 `memory-permission.js` 的 `classify()` 与 `canRead()`。
- `bridge.js`：`approveShare`/`rejectShare` 语义不变（仍写 `review.status`）；管理端列表需展示 `contentClass` 与 `scope.read`。
- 迁移：历史记录 `review.status` 保持原映射（`approved→approved`、其余→`pending`），`contentClass` 统一置 `uncertain`（保守），**不自动升级**。

### 为什么这是修复而非放宽

放宽的只有"同群 + 非敏感 + 自产"一类，这类信息已在该群公开过。所有跨边界方向（跨群、私聊→群）全部维持或加强。且新增了原设计没有的**内容级降级**（sensitive → 连原群也不可见）。

---

## S-12 · 索引原子性自相矛盾（**P0**，本轮新增）

### 问题

上一版同时要求：

> "索引增量更新必须与 A 层写在同一个 WriteQueue 临界区"
> "embed 在临界区外（30s 超时）"

索引依赖向量 → 两条不可能同时成立。且 `memory-reflect` 的整理、Retain 的批量写入都会放大这个矛盾。

### 裁决

按危害不对称原则**拆开**（详见 `01-ARCHITECTURE.md` §4）：

| 操作 | 策略 |
|---|---|
| 删除 / clear | 同步：索引移除与 A 变更在同一临界区 |
| 追加 / edit | 异步：A 提交后进 `indexQueue`，允许滞后 |
| 向量 | 允许滞后：`embedding.status='pending'` + 后台补算 |

补偿机制：`meta.coveredRev` + A 层尾部扫描（`rev > coveredRev`），滞后 > 500 条触发同步重建并返回 `degraded`。

不变量 INo-10 相应改写为：**索引可滞后、不可领先、删除必须即时**。

---

## S-13 · `agentToken` 进入 prompt 并作为工具参数（**P0**，本轮新增）

### 源码事实（已核验，非推测）

- `bridge.js:10674`：`const tokenLine = '【会话令牌】' + st.agentToken + ...`，**每轮唤醒写入 prompt**。
- `mcp-snowluma-safe.js`：40+ 个工具的 zod schema 把 `token` 声明为必填或可选参数，描述文字统一为"见唤醒提示中的【会话令牌】"（如 `:587`、`:603`、`:926`、`:958`、`:985`、`:1064`）。
- 鉴权侧：`bridge.js:6867` `agentTokenOk(key, token)`、`:6868` `v2SessionAllowed(key)`、`:6869` `v2ToolEnabled('memory')`。

**因此这不是"删一行"**：token 目前是"模型可见 + 模型传递"的显式凭据。移除它意味着会话身份必须改为**带外绑定**。

### 裁决（分两阶段，不假装一步到位）

**目标态（M1 完成）**：

```
模型不接触 token；工具 schema 移除 token 参数；
MCP 侧按 key 解析凭据，凭据来源优先级：
  ① 进程环境变量（若 MCP 由 bridge 以子进程方式启动）
  ② 握手端点 POST /api/agent/handshake（loopback + 进程级 secret）换取的短期票据
  ③ 本地文件 state/agent-tokens/<hash(key)>.tok（权限 0600，仅同用户可读）
```

**过渡态（M0 内可先做，降低风险）**：

1. token 从**每轮注入的块**中移出，改为**会话首轮一次性注入**（纳入 Block Ledger 的 `credential` 块，`anchorEveryTurns: 0`）。
2. token 有效期与会话 epoch 绑定，轮转即失效。
3. 服务端增加"同 token 高频跨 key 调用"检测（越权试探告警）。

**为什么分两阶段**：目标态需要改 40+ 个工具 schema 与 MCP 的生命周期假设。**MCP 是否由 bridge 启动、是否一进程一会话，目前没有源码证据**（见 12 文档 §6 的"实施前必须核查项"）。在证据缺失时把过渡态写成目标态，是把未验证的假设伪装成设计。

### 影响面

`mcp-snowluma-safe.js` +120/-60；`bridge.js` 新增握手端点 +60；`05-CONTEXT-TOKEN.md` §3 相应改写（安全规则不依赖历史注入）。

---

## S-14 · `proposed` 降权注入违反归纳安全（**P0**，本轮新增）

### 问题

上一版写："`proposed` 状态的观察仍然注入，但降权 0.6 并标注'（我的印象）'"。

这违反任务书"禁止未经验证就让自动归纳结果覆盖原始事实"的精神：未经审核的归纳一旦进入上下文，模型会把它当既有认知使用，事后无法区分"我知道"与"我猜的"。

### 裁决

**`proposed` 默认完全不进召回、不进注入**（INo-16）。

- 只有 `includeProposed: true`（管理端 / reflect 自检工具）时可见。
- 升级为 `active` 的路径见 `07-REFLECT.md` §5.4（独立证据 ≥ 3 条且跨 ≥ 2 天，或主人批准）。
- 代价：归纳的价值要等到升级后才体现。这是**有意的取舍**——宁可归纳不出东西，也不让猜测进入上下文。

---

## S-15 · 单链聚类批量 supersede（**P0**，本轮新增）

### 问题

上一版 A1 用"贪心单链聚类（相似度 ≥0.92）"把整簇标 `superseded`。单链聚类的已知缺陷是**链式漂移**：A≈B、B≈C、C≈D 会把语义上已经不相干的一串记录连成一簇，一次操作误伤几十条。

### 裁决

取消批量聚类，改为**三档判定**（详见 `07-REFLECT.md` §3）：

| 档 | 条件 | 动作 |
|---|---|---|
| 完全重复 | 归一化文本（去标点/去数字/小写）**完全相同** | `lifecycle='superseded'` + `duplicateOf`，单次只处理一对 |
| 可能重复 | `0.92 ≤ cosSim < 0.99` | **不改 lifecycle**，只写 `extra.possibleDuplicate[]` + 进审核队列 |
| 仅主题相似 | `0.65 ≤ cosSim < 0.92` | 不处理（留给归纳） |

阈值 0.99 而非 0.92，且加"归一化文本相同"这个硬条件——两个条件同时满足才允许 supersede。

---

## S-16 · 候选池混装不同失败原因（**P0**，本轮新增）

### 问题

上一版只有一个 `memory-candidates.jsonl`，且规则"同一条候选反复出现 3 次以上 → 自动提升为正式记忆"。

这带来两个漏洞：

1. **权限失败可被"重复"洗白**：一条被判为敏感的候选，只要它在群里被反复提及 3 次，就自动进 A 层并可能群内可见。
2. **新颖度失败被误当价值信号**："说了 3 次"不代表"重要"，只代表"重复"。

### 裁决

三池物理分离 + 取消一切自动晋升：

| 池 | 收什么 | TTL | 再评估 | 自动晋升 |
|---|---|---|---|---|
| `memory-candidates.jsonl` | 新颖度失败、价值分不足 | 7 天 | 重现时**重新过全部四闸** | **永不** |
| `memory-quarantine.jsonl` | 内容分级敏感/不确定、scope 推导异常 | 无（等主人审） | 不自动 | **永不** |
| 待补算队列（S 层 `pendingEmbed`） | embedding 失败 | 无（退避重试 5 次） | 是 | 不涉及（记录已在 A） |

"重现 3 次"的唯一作用是**触发重新评估**，不是晋升。重新评估仍要过全部四闸，包括权限闸。

---

## 附：与 V2.4.2 文档的覆盖关系

| V2.4.2 文档 | 覆盖情况 |
|---|---|
| `00-SUMMARY.md` | effect 改 3 类；新增 INo-10~16 |
| `02-WRITE-CONCURRENCY.md` | 重写并发模型（per-key 锁 + fencing，非重读） |
| `03-AB-CONSISTENCY-WAL.md` | 补提交协议 5 阶段与崩溃点语义 |
| `04-LIFECYCLE-TOMBSTONE.md` | 生命周期扩为 6 态；`proposed`/`quarantined` 语义 |
| `05-SHARE-SECURITY.md` | **整体替换**为三字段权限模型 |
| `06-EMBEDDING-AUDIT-BACKUP.md` | 不改（补 embedding 三态与待补算队列） |
| `07-LOCK-AND-STARTUP.md` | 补三层串行化与 fencing token |
| `09/10-ACCEPTANCE` | 并入 `11-ACCEPTANCE.md` |
| `11-SOURCE-EVIDENCE.md` | 行号仍有效；新增 S-13 的三处行号 |
| `12-CHANGELOG.md` | 本文档充当增量 changelog |
