# Memory V2.4.2 · 独立设计与源码交叉核验

- 核验时间：2026-10-31 17:05 ~ 17:40
- 设计输入：`00`~`09` 共 10 份（2090 行）+ 第二轮审查报告（612 行）
- 源码输入：`D://qq-bridge`（`src/memory-vector.js` 380 行、`src/bridge.js` 12,482 行、`src/mcp-snowluma-safe.js` 1,465 行等）
- 核验方式：**只读**。未修改任何代码/配置/数据，未启动服务，未执行业务写入。
- 标注约定：
  - `SOURCE_CONFIRMED` — 已在源码中逐行确认
  - `DESIGN_INFERENCE` — 依据设计文档规则推演，源码未实现故无法直接验证
  - `NOT_VERIFIABLE` — 证据不足

---

# 〇、核验结论摘要

| 问题 | 结论 | 关键依据 |
|---|---|---|
| Q1 设计是否正确理解 V1 | **大部分正确，3 处关键误判** | B 层 43 处写入被当成"投影"；memberImpressions 是 1:N 被当成 1:1；A Schema 缺 B 独有字段 |
| Q2 MemoryService 是否覆盖所有写入入口 | **未覆盖** | A 层 8 个入口可覆盖；B 层 43 处运行状态写入未纳入 |
| Q3 三锁协同 | **基本自洽，但价值主张需修正** | bridge 全程持 writer.lock ⇒ 脚本必须等 bridge 停 ⇒ writer.lock 实际只做"多脚本互斥" |
| Q4 embedding 队列外顺序 | **存在顺序反转，设计未处理** | 无 `requestedAt`/`requestSequence`/`commitSequence` 字段 |
| Q5 memoryId/rev/WAL/tombstone 冲突 | **存在真实冲突（D-17）** | rev 混合内容与元数据 ⇒ pending edit 被误 superseded |
| Q6 A/B 投影兼容 | **不兼容** | memberImpressions 1:N；B 层非投影字段占比高 |
| Q7 分享审批是否避免错误授权 | **安全但过度失效** | 不会授错，但正常审批被元数据变更打断 |
| Q8 迁移/恢复风险 | **有边界风险（已诚实标注）** | tombstone 初始为空 ⇒ V1 期删除记录可经历史备份复活 |
| Q9 M0 是否具备独立安全运行条件 | **不具备** | 恢复流程/embedding 六态/B 投影重建均在 M1 |
| Q10 验收标准是否足够 | **覆盖广但缺 5 类用例** | 见 §9 |

**必须修改的设计问题：6 项**（§10）。**可以保留：7 项**（§11）。

---

# 一、设计文档完整性检查

## 1.1 结构完整性

| 文档 | 行数 | 覆盖内容 | 评价 |
|---|---|---|---|
| `00-SUMMARY` | 163 | 10 项新增、10 条原则、目标架构、Schema、12 决策、M0/M1/M2、Q1~Q10 | 完整，可作总纲 |
| `01-MAPPING` | 57 | F-01~F-26 逐条映射 + 里程碑归并 + 3 条自查修正 | 完整，表格密度高 |
| `02-WRITE-CONCURRENCY` | 201 | 7 类 op、P0~P4 管线、6 条不变量、原子写、WAL 条目、6 组并发对 | 核心，质量最高 |
| `03-AB-CONSISTENCY-WAL` | 221 | A/B 关系、B 结构、规范化位置、WAL 崩溃矩阵、superseded、3 个恢复时序 | 完整 |
| `04-LIFECYCLE-TOMBSTONE` | 175 | 生命周期六态、factKey、tombstone、级联、suppression、force_restore | 完整 |
| `05-SHARE-SECURITY` | 145 | SR 对象、申请/审批/撤销、副本向量、群隔离矩阵 | 完整 |
| `06-EMBEDDING-AUDIT-BACKUP` | 190 | 六态机、10 项要点、两级审计、备份 envelope、恢复 R0~R2 | 完整 |
| `07-LOCK-AND-STARTUP` | 130 | 两锁职责、六态启动、stale 算法、退出 drain、外部工具协议 | 完整 |
| `08-DECISIONS` | 56 | D-01~D-20 + 4 项 NOT_VERIFIABLE | 完整 |
| `09-ACCEPTANCE` | 140 | 71 个 AC + 5 条红线 | 完整 |

## 1.2 缺失项（设计文档层面）

| # | 缺失 | 影响 |
|---|---|---|
| M-1 | **无 `requestedAt` / `requestSequence` / `commitSequence` 字段定义** | 无法审计"请求顺序 vs 提交顺序"（用户特别要求核验项） |
| M-2 | **无 B 层非投影字段（recentMessages/unread/wakeConfig 等）的归属说明** | 43 处写入点在设计中无位置 |
| M-3 | **无 memberImpressions 1:N 投影规则** | A 层 N 条 → B 层 1 个 key 的映射未定义 |
| M-4 | **M0 没有独立的验收清单** | 09 的 AC 混编 M0/M1，无法判断 M0 可交付性 |
| M-5 | **无 `contentRev` vs `metaRev` 的取舍分析**（只有 D-17 的二选一） | 影响 §10.1 的判断 |
| M-6 | 无 `truncateText` 未导出模块的替代方案细节（只说"改用 clampText"） | 影响 B/A 截断一致性验收 |

---

# 二、真实源码写入入口清单

## 2.1 A 层（`memory-vectors.jsonl`）

| # | 入口 | 位置 | 触发方 | 设计是否覆盖 |
|---|---|---|---|---|
| A1 | `appendFileSync(VEC_FILE, ...)` | `memory-vector.js:160` | `appendVectorMemory` | ✅ MS/WQ |
| A2 | `writeFileSync(tmp)+renameSync` | `:378-379` | `atomicRewrite`（7 个函数） | ✅ MS/WQ |
| A3 | `copyFileSync(VEC_FILE_PATH, ...)` | `bridge.js:6716` | admin remove/edit 前备份 | ⚠️ 只读出，但设计未明确归属（Backup 模块？端点层？） |

`atomicRewrite` 的 7 个调用点：`requestShare:272` / `approveShare:289` / `rejectShare:306` / `removeVectorMemoryExact:317` / `removeNAllVectorMemory:341` / `removeAllVectorMemory:355` / `editVectorMemory:372`。**`SOURCE_CONFIRMED`**

## 2.2 B 层（`social-v2.json`）— 关键发现

**`saveSocialV2State()` 在 bridge.js 中调用 43 处，`saveSocialV2StateStrict()` 4 处。`SOURCE_CONFIRMED`（全文件正则穷举）**

| 类别 | 数量 | 代表行号 | 是否记忆投影 |
|---|---|---|---|
| 记忆端点（strict） | 4 | 6668 append / 6941 remove / 6990 clear-mI / 7047·7100 clear | ✅ 设计覆盖 |
| 唤醒配置 | 多 | 5011（`/api/socialV2/wake`） | ❌ 运行状态 |
| 未读/已读 | 多 | 4583（`/api/socialV2/activity`）、6173 | ❌ 运行状态 |
| 消息/发送 | 多 | 5310/5323/5344/5440/5820/5830/5866/5874 | ❌ 运行状态 |
| 角色/模式/重启 | 若干 | 7365/7383/7435/7496/7519 | ❌ 运行状态 |
| 社交循环内部 | 多 | 8685/8758/8925/10243/10285/10312/10836~10874/11311/11384/11690/11888/11946 | ❌ 运行状态 |

**结论**：`social-v2.json` **不是纯投影**，它同时是"运行状态容器"（recentMessages/unread/wakeConfig/lastReadThroughSeq/agentToken 等）。

**设计 00 §2 原则 1 称「B 层永不由外部直接修改（B 是投影）」——与源码不符。`SOURCE_CONFIRMED`**

## 2.3 其他存储

| 文件 | 写入者 | 设计覆盖 |
|---|---|---|
| `backups/memory-vectors-*.jsonl` | `dailyBackup():138` `copyFileSync` | ⚠️ 设计说改 envelope，但未说是否归 Backup 模块 |
| `state/memory-admin.log` | `bridge.js:6733/6740/6754` | ✅ Auditor 接管，旧文件归档 |
| `state/bridge.lock` | `acquireLock:1085` / `releaseLock:1146` | ✅ 07 状态机 |

---

# 三、V1 实现与 V2.4.2 设计差异

## 3.1 设计准确理解的部分（`SOURCE_CONFIRMED`）

| 设计主张 | 源码依据 | 判定 |
|---|---|---|
| `editVectorMemory` 用 await 前快照全量覆盖（F-01） | `:362` loadRows → `:367` await embed → `:372` atomicRewrite | ✅ 准确 |
| `atomicRewrite` 固定 tmp 名（F-06） | `:377` `const tmp = VEC_FILE + '.tmp'` | ✅ 准确 |
| `loadRows` 坏行静默跳过（F-12） | `:53` `try{...}catch{}` | ✅ 准确 |
| 读异常返回 `[]`（F-13） | `:57` `catch { return []; }` | ✅ 准确 |
| 审计在业务后、无 try/catch（F-05） | `:6731→6733` / `:6739→6740` / `:6751→6754` | ✅ 准确 |
| `clear` / `memory-remove` 零审计 | 全文件仅 3 处审计写入（6733/6740/6754） | ✅ 准确 |
| 无 tombstone / 无恢复函数（F-11） | 全仓 0 命中 | ✅ 准确 |
| A 层未脱敏、B 层脱敏+截断 200（F-02） | `bridge.js:6658` vs `:9207/9228/9244` | ✅ 准确 |
| `aRemoved` 未上报（F-03） | `:7011` 只返回 `removed: bRemoved` | ✅ 准确 |
| `findMemoryRow` 四级匹配（F-09） | `:238/240/242/254` | ✅ 准确（注释写"三级"，实为四级） |
| `bridge.lock` 窗口（F-07） | `:1390` acquire → `:7531` listen | ✅ 准确，且窗口内含 `loadState:1391`、`loadSocialV2State:9275`、`bot.connect:12425` 等真实初始化 |
| `consolePortInUse` 失败开放（F-42） | `:1074-1077` `catch { return true; }` | ✅ 准确 |
| admin 拒绝 agent token | `:6707` `if (req.headers['x-agent-token'] !== undefined)` | ✅ 准确 |
| private 拒绝分享 | `:6892-6894` | ✅ 准确 |

## 3.2 设计误判的部分（`SOURCE_CONFIRMED`）

### 误判 ①：B 层 = 投影

```
设计（00 §2 原则 1、03 §1）：
  "B 层永不由外部直接修改（B 是投影 / projection / 摘要缓存）"
  "可写者：仅 Reconciler / WriteQueue 的 B-effect"
  "损坏后果：可从 A 重建"

源码实际：
  saveSocialV2State() 43 处调用，其中 39 处与记忆三字段无关
  social-v2.json 同时含 recentMessages(100)/unread/wakeConfig/lastReadThroughSeq/
                     lastWakeReason/sendTimes/wakeTimes/agentToken/seenForwardIds
  这些字段【无法从 A 层重建】
```

**影响**：
1. 若 Reconciler 按"从 A 重建 B"实现，会**丢失 39 类运行状态**
2. 若 Reconciler 只重建三字段（正确做法），则设计文档的"B 是投影"表述必须限定为"三字段是投影"
3. V2 的 `atomicWriteJson(social-v2.json)` 仍是**整文件重写**，MemoryService 写 B 与 39 处运行状态写 B 会**互相覆盖**（后者是最后写者赢）——这是 V1 已有问题，**V2 设计未解决**

### 误判 ②：memberImpressions 1:1

```
设计（03 §1.1）：
  memberImpressions["名字"] = { "memoryId":"mem_…", "rev":4, "traits":[…] }

源码实际（bridge.js:9251-9269）：
  st.memberImpressions[target] = {
    traits: [最多10条],        ← 数组，每次 append 追加
    interactionCount: N,
    lastSeenAt: ts
  }
  且每次 append memberImpression 会在 A 层新增 1 条记录
```

**后果**：
- A 层 N 条 `memberImpression`（同一 subjectId）↔ B 层 1 个对象
- 设计用单个 `memoryId` 表达，只能指向其中一条 → **其余 A 记录在 B 无投影**
- 删除/编辑时按 `memoryId` 定位，落到 B 层只有一个 key，无法精确移除某一条 traits

**这是设计必须补的映射规则**（见 §10.2）。

### 误判 ③：A 层 Schema 缺 B 层独有字段

```
设计（00 §4 A Schema）字段：id/rev/key/category/subjectId/factKey/text/owner/
  visibility/lifecycle/vec/vecModel/vecDim/embedding/origin/expiresAt/shareCopies/
  suppression/createdAt/updatedAt/migratedFrom/_crc

V1 B 层实际（bridge.js:9227-9247、9251-9262）：
  activeTopics[i]    含 participants[] / pendingQuestion
  pendingThoughts[i] 含 motivation
  memberImpressions  含 interactionCount / lastSeenAt
```

**A Schema 无上述字段的对应位置** ⇒ "B 完全由 A 投影"不可行。
设计 `03 §4` 说 `memory-update` 改为转发 `MS.edit`，但 `newExtra`（participants/pendingQuestion/motivation/target）**在 A 层无处存放**。

见 §10.3。

---

# 四、关键并发与锁机制核验

## 4.1 进程内 FIFO 写队列（`DESIGN_INFERENCE`）

设计 `02 §7` 的实现：

```js
#chain = Promise.resolve();
enqueue(op) { const p = this.#chain.then(() => this.#execute(op)); this.#chain = p.catch(()=>{}); return p; }
```

**评价**：
- ✅ 服务 P3 串行，INo-1 成立
- ✅ 失败不断链（`catch(()=>{})`）
- ⚠️ **无界队列**：设计 §7 说"无界但可取消"。若 embed 慢导致积压，内存可涨。设计未设上限或背压。
- ⚠️ **`drain(timeoutMs=5000)` 与 P1（≤30s embed）冲突**：退出时 drain 只等 5s，但队列里可能有正在 P3 的 op（P3 是同步 IO，通常很快）——真正的问题是在**队列等待**中的 op（P1 已完成，等待 P3）。5s 内可能排不完。设计 §07 §5 说"超时未落定的 op → recovery_blocked"，这处理了，但会让正常退出频繁产生 recovery_blocked。

## 4.2 三锁职责（`SOURCE_CONFIRMED` + `DESIGN_INFERENCE`）

| 锁             | 设计职责                  | 与源码 V1 对比                             | 判定           |
| ------------- | --------------------- | ------------------------------------- | ------------ |
| `bridge.lock` | 进程实例唯一性（升级为状态文件 + 心跳） | V1 只有 pid（`bridge.js:1085` 写 PID 字符串） | 设计合理，修复 F-07 |
| `writer.lock` | A 层跨进程写互斥             | V1 **不存在**                            | 新增           |
| 进程内 FIFO      | 同进程时序                 | V1 **不存在**                            | 新增           |

### 关键语义问题（`DESIGN_INFERENCE`）

设计 `07 §1`：bridge 主进程**从启动到退出全程持有 writer.lock**。  
设计 `07 §6`：外部脚本必须 `acquireWriterLock` **且** 校验 `bridge.lock.state==='stopped'` 或文件不存在。

**推论**：

```
bridge 运行中 ⇒ writer.lock 被 bridge 持有 ⇒ 脚本永远拿不到 ⇒ 脚本必须等 bridge 停
⇒ writer.lock 的【实际作用】= "bridge 停止期间，多个脚本之间互斥"
⇒ "bridge vs 脚本"的互斥【不是靠 writer.lock，而是靠 D-08「强制停止」】
```

**这不是缺陷，但设计文档的表述会误导**。`07 §1` 说 writer.lock 的作用是"bridge vs 迁移/恢复/维护脚本互斥"，实际上该互斥由"要求 bridge 停止"实现，writer.lock 只覆盖"脚本 vs 脚本"。

**建议修正表述**，避免实施时误以为 writer.lock 能阻止并发。

## 4.3 bridge.lock 状态机（`DESIGN_INFERENCE`，有 2 处规格缺口）

设计 `07 §4` 的 stale 判定逻辑本身正确（GRACE 等待 + state 区分），但：

### 缺口 ①：心跳启动时机与 GRACE 判定不自洽

```
07 §2： "心跳：主进程每 10s 更新 heartbeatAt"（未说何时启动）
07 §3： S4（ready）才 "启动后台任务：… 心跳 …"
07 §4： B2a "age = now - max(startedAt, heartbeatAt)；age ≤ GRACE(90s) → 等待重试"
```

**推论**：S1~S3（lock_acquired → recovering → listening）期间**没有心跳**，`heartbeatAt` 为 null 或等于 startedAt，`age = now - startedAt`。

若 S2 recovery 耗时（WAL 有大量 pending op 时可能数十秒到数分钟）> GRACE 90s：

- B 判定 B2b → **接管** → 此时 A 仍在 recovery，未 listen → 双写风险

**这是设计内部不一致**：GRACE 假设有心跳维持，但心跳在 S4 才启动。

**修法**：心跳必须在 S1（lock_acquired）后立即启动，且 `07 §3` 的 S4 列表移除"心跳"。

### 缺口 ②：等待次数与 GRACE 不匹配

```
07 §4 B2a："等待重试（sleep 5s，最多 6 次）"  = 最多 30s
D-06 GRACE 建议 90s
```

**推论**：B 最多等 30s，但 A 可能仍在 90s GRACE 内启动中。30s 后 B 的行为**未定义**（退出？继续？接管？）。

**修法**：等待次数应 `ceil(GRACE / 5s)` 次（90s → 18 次），或改为"循环直到 age > GRACE 或 state=ready"。

---

# 五、WAL、tombstone、revision 核验

## 5.1 WAL 设计评价（`DESIGN_INFERENCE`，总体良好）

| 项                | 设计                                         | 评价                |
| ---------------- | ------------------------------------------ | ----------------- |
| WAL 先行           | P3.1 落盘 pending，先于业务写                      | ✅ 正确（可知道"曾打算做什么"） |
| 提交顺序             | T → A → B → SR → WAL(committed)            | ✅ 合理              |
| effect 语义        | set-to-state + targetRev                   | ✅ 幂等性正确           |
| superseded 判定    | 存在更高 targetRev 的 committed/更晚 pending → 跳过 | ⚠️ 见 5.3 冲突       |
| 崩溃点矩阵            | ①②③④ 四档                                    | ✅ 完整              |
| recovery_blocked | 冻结集 + resubmit/discard                     | ✅ 完整              |

## 5.2 tombstone 设计评价（`DESIGN_INFERENCE`，良好）

| 项             | 设计                               | 评价      |
| ------------- | -------------------------------- | ------- |
| 永久保留          | 不参与 checkpoint 压缩                | ✅       |
| 不存正文          | 只有 contentHash                   | ✅       |
| 默认级联          | 原件删除 → 副本 tombstone              | ✅（决策 4） |
| 恢复拦截          | 恢复时比对 tombstone 集合跳过             | ✅       |
| force_restore | 默认仍跳过墓碑，需 `ignoreTombstone` 白名单  | ✅ 专业    |
| 内容屏蔽独立        | content_suppression vs tombstone | ✅ 语义清晰  |

## 5.3 ★ revision 冲突（真实缺陷，`DESIGN_INFERENCE`）


### 冲突链

```
设计 05 §3 ⑨：批准分享 → 原件 shareCopies 变更 → 原件 rev+1
设计 06 §1.3：embedding 成功回写 → embedding_refresh op → A upsert → rev+1
设计 08 D-17：承认该 rev+1 会使 pending SR superseded，建议"是（简单）"
设计 03 §5.3：superseded(op) = 同 targetId 且【targetRev 更高】的 committed/更晚 pending
设计 02 §1.1 P3.4：record.rev === expectedRev 校验，失配 → 409
```

**推演出的一条静默数据丢失路径**：

| 时刻 | 用户 edit(mem_M, expectedRev=2 → targetRev=3) | 后台 embedding_refresh(mem_M)       | 结果           |
| -- | ------------------------------------------- | --------------------------------- | ------------ |
| t0 | P1 embed 中（≤30s）                            | —                                 | —            |
| t1 | —                                           | 上次 append 的嵌入完成 → 入队 → 提交：rev 2→3 | —            |
| t2 | 入队，expectedRev=2                            | —                                 | —            |
| t3 | P3.4 校验：当前 rev=3 ≠ 2                        | —                                 | **409 版本冲突** |

**后果**：用户的编辑被拒绝，且原因是"系统后台刷新了向量"——**用户无法理解**。

更严重的变体（WAL 重放场景）：

| 时刻     | WAL 中 pending 的 edit（targetRev=3） | embedding_refresh 提交（targetRev=4）                                                   |
| ------ | --------------------------------- | ----------------------------------------------------------------------------------- |
| t0     | 写入 WAL pending                    | —                                                                                   |
| t1     | 服务崩溃                              | —                                                                                   |
| t2     | 重启恢复：扫到 edit pending              | —                                                                                   |
| t3     | —                                 | 先于 edit 重放：扫描按 seq 升序 → edit 的 seq 更小，先重放 → 如果 edit 先重放则正常                          |
| 变体 t3' | —                                 | 若 embedding_refresh 的 seq 更小（先入队）→ 它先提交 rev=4 → **edit 被判 superseded，effects 全部跳过** |

**结论**：`superseded` 规则 + `rev` 混用 ⇒ **用户的内容编辑可能被系统元数据刷新静默丢弃**。

### 为什么必须修

`rev` 承担了三种语义：

1. 内容版本（text 变化）
2. 元数据版本（shareCopies / embedding / suppression）
3. 乐观锁令牌

三种语义共用一个计数器，导致：

- 元数据变更破坏内容编辑的乐观锁
- 元数据变更破坏 pending SR
- 无法区分"内容真的变了"与"只是向量刷新了"

### 最小修法（成本远低于"完整双 revision 体系"）

```
新增字段：contentRev（只在 text 变化时 +1）
保留：rev（全量，仍用于内部幂等/WAL）
乐观锁：edit / remove / share_request / share_approve 的 expectedRev → 改为比对 contentRev
SR.expectedRevAtRequest → 改为 expectedContentRevAtRequest
superseded 判定 → 改为比较 contentRev（而非 rev）
embedding_refresh / shareCopies 变更 → 只 +rev，不动 contentRev ⇒ 不再影响审批与编辑
```

**成本**：一个字段 + 4 处比较改动。设计 `D-17` 说"否需引入双 revision，复杂度显著上升"——**该评估偏保守**，实际增量很小。

---

# 六、A/B 一致性核验

## 6.1 设计的一致性机制（`DESIGN_INFERENCE`）

| 机制                                                  | 设计位置    | 评        |
| --------------------------------------------------- | ------- | -------- |
| B 层带 memoryId + rev 镜像                              | 03 §1.1 | ✅        |
| `B.rev < A.rev` ⇒ 投影过期                              | 03 §1   | ✅        |
| 读时惰性刷新 / Reconciler 批量修复                            | 03 §1   | ⚠️ 见 6.2 |
| A 删除 ⇒ B 级联移除（同 op effect）                          | 04 §4   | ✅        |
| B 单独修改入口取消                                          | 03 §4   | ⚠️ 见 6.3 |
| 统一读谓词（lifecycle + expiresAt + tombstone + vecModel） | 03 §4   | ✅ 修 F-04 |

## 6.2 Reconciler 的覆盖风险（`DESIGN_INFERENCE`，真实风险）

设计 `03 §1` 说"读时惰性刷新或由 Reconciler 批量修复"，`06` 未定义 Reconciler 的写入粒度。

**风险**：

```
若 Reconciler 用"从 A 重建整个 social-v2.json"：
  → 43 处运行状态（recentMessages/unread/wakeConfig/agentToken…）全部丢失
  → 这是灾难性数据丢失

若 Reconciler 只改三字段：
  → 需要与 39 处运行状态写入并发安全
  → 但 atomicWriteJson 是整文件重写 ⇒ 谁后写谁赢
  → Reconciler 写三字段时，会把 39 处刚写的运行状态回退到它读到的旧快照
```

**设计未定义 Reconciler 的"读-改-写"边界**。这是必须补的（见 §10.4）。

## 6.3 B 单独修改入口"取消"的可行性

```
设计 03 §4："/api/socialV2/memory-update 改为转发到 MS.edit，需 memoryId+expectedRev"
设计 04 §4："B 层单独删除：入口取消（V1 memory-remove 的 B-only 路径全部改为走 A 层删除）"
```

**源码对照**：`memory-update` 端点存在（`bridge.js:6765`），参数是 `{key, category, oldContent, newContent, newExtra, target}`，**没有 memoryId**。

**影响**：

- MCP 侧有对应工具（`qq_memory_update`？需确认）
- 若有，模型侧调用需同步改签名 → 属 D-20（兼容 1 版本）范围
- 但 `newExtra` 支持改 `participants/pendingQuestion/motivation/target`——这些是 **B 层独有字段**，A 层 Schema 里只有 text/vec/lifecycle，**没有 participants 的对应位置**

**这是设计的结构缺口**：A 层 Schema（00 §4）不含 `participants` / `pendingQuestion` / `motivation`，但 B 层有。若 B 完全由 A 投影，这些字段无处存放。

见 §10.3。

---

# 七、分享与权限安全核验

## 7.1 安全性（`DESIGN_INFERENCE`，正确）

| 检查                                        | 设计          | 评价                                    |
| ----------------------------------------- | ----------- | ------------------------------------- |
| 申请用 memoryId（非文本）                         | 05 §2       | ✅ 修 F-09 四级模糊匹配                       |
| 审批时复查 5 项（存在/墓碑/rev/lifecycle/visibility） | 05 §3 ①~⑦   | ✅ 严格                                  |
| private 源拒绝                               | 05 §2       | ✅ 与 V1 `:6892` 一致（`SOURCE_CONFIRMED`） |
| 群隔离（每群一条 SR）                              | 05 §1       | ✅ 修 V1 单字段缺陷                          |
| 副本可追溯 origin                              | 05 §3 ⑨     | ✅                                     |
| 控制台令牌 + 拒绝 agent token                    | 05 §3       | ✅ 与 V1 `:6707` 一致（`SOURCE_CONFIRMED`） |
| 撤销与级联                                     | 04 §5、05 §4 | ✅                                     |

**结论：不会发生"错误授权"**（用错 id / 原件已变 / 已删均被拦截）。

## 7.2 可用性（`DESIGN_INFERENCE`，过度失效）

**pending ShareRequest 会被无关元数据变更意外失效**：

| 触发                                  | 后果                                   | 设计是否承认                |
| ----------------------------------- | ------------------------------------ | --------------------- |
| 批准群 A 的分享（原件 shareCopies+1 → rev+1） | 群 B 的 pending SR 全部 superseded，需重新申请 | ✅ 05 §3 ⑨ 明确承认，列 D-17 |
| 后台 embedding 刷新完成（rev+1）            | 所有 pending SR 失效                     | ✅ 06 §1.3 明确承认，列 D-17 |
| 内容屏蔽/解除（rev+1，若实现为 rev+1）           | 同上                                   | ❌ 未提及                 |
| 任何 B 投影元数据回写                        | 若 rev+1 则同上                          | ❌ 未提及                 |

**实际业务影响**：

> 主人想分享一条记忆到 3 个群 → 申请 3 条 pending → 批准第 1 个 → 另外 2 个 pending 立即失效 → 必须重新申请 → 批准第 2 个 → 第 3 个又失效……

这是**用户可感知的功能缺陷**，不是理论风险。

**修法**：同 §5.3（contentRev）。

---

# 八、迁移和恢复风险

## 8.1 迁移（`DESIGN_INFERENCE`，整体稳健）

| 项                | 设计                                | 评价 |
| ---------------- | --------------------------------- | -- |
| 前置               | 服务停止 + writer.lock                | ✅  |
| 全量备份             | 步骤 1                              | ✅  |
| dry-run          | 步骤 2                              | ✅  |
| 对账               | 步骤 6（output == input - bad - dup） | ✅  |
| 临时文件 + 原子 rename | 步骤 4                              | ✅  |
| 失败整体回退           | 步骤 6                              | ✅  |
| 旧文件 `.v1.bak` 只读 | 步骤 4                              | ✅  |
| 不触碰历史备份          | 03 §7 末                           | ✅  |

**关键边界（已诚实标注）**：

```
V1 是物理删除，无墓碑
⇒ V1 时期被删的记录，在 V2 tombstone 中无记录
⇒ 若日后人工恢复 V1 历史备份，这些记录会"复活"且 V2 无法识别
⇒ 设计用 D-12 挂起（接受 + 建议人工比对 101 条 × 3 份备份后录入初始墓碑）
```

**我的评价**：D-12 的"接受"是合理的（否则迁移前要做人工考古），但**必须在迁移报告中显式列出这个边界**，并提示用户"历史备份不再具备防复活保护"。

## 8.2 恢复（`DESIGN_INFERENCE`，设计完整但 M0 缺失）

`06 §4` 的 R0/R1/R2 三阶段设计完整：

- 强制 dry-run（R1 必经）
- digest 校验
- tombstone 跳过
- 版本冲突跳过
- pre_restore 回滚
- confirmToken 二次确认

**但**：`01 §3` 把备份/恢复归入 **M1**（F-24/F-26 在 M1），而 `00 §6` 说 M0 解决 F-11（备份恢复复活）。

**矛盾**：M0 有 tombstone（F-11 的一半），但**没有恢复流程**（F-11 的另一半）⇒ **M0 期间人工 copy 备份仍会绕过 tombstone ⇒ F-11 未真正解决**。

见 §9。

---

# 九、M0/M1 边界评价

## 9.1 M0 声明（`00 §6`）

> M0 = MemoryService 写队列 + memoryId/revision + WAL + tombstone + 两级审计 + V1 迁移 + bridge.lock 状态机  
> 解决：F-01/F-03/F-05/F-06/F-11/F-15/F-17/F-23

## 9.2 四个缺口（`DESIGN_INFERENCE`）

### 缺口 ①：M0 无恢复流程 ⇒ F-11 只解决一半

```
M0 有：tombstone 写入（删除时）
M0 无：备份恢复流程（M1）
⇒ 人工 copy backups/*.jsonl 到 state/memory-store.jsonl
   → 不经过 MS，不比对 tombstone
   → 已删记录复活
   → M0 的 tombstone 对"人工恢复"完全无效
```

**M0 不能声称解决 F-11。** 建议：把"最小恢复校验函数"（读备份 → 比对 tombstone → 过滤）纳入 M0，即使没有 UI 和 dry-run。

### 缺口 ②：M0 无 embedding 六态 ⇒ F-08 未解决

```
00 §6 M0 列表不含 F-08（F-08 在 M1）
但 02 §1.1 的 P1 阶段（embedding 队列外执行）是 M0 的 WriteQueue 前提
⇒ M0 必须实现"embed 失败怎么办"
⇒ 若沿用 V1 的"失败即抛"，则 F-08（正文丢失）在 M0 仍存在
⇒ 但 M0 的 WAL + 正文先持久化（03 §1）又要求 embedding 解耦
⇒ 设计内部不一致
```

|                                            |       |                                                                 |
| ------------------------------------------ | ----- | --------------------------------------------------------------- |
| \*\*M0 必须至少实现 \`embedding.status = pending | ready | failed`三态**（不需要完整六态 + 重试队列），否则 WAL 的`effects\` 无法表达"正文已写、向量待补"。 |

### 缺口 ③：M0 无 B 投影重建 ⇒ A/B 分叉不可修复

```
M0 无 Reconciler（M1）
但 M0 有 WAL 崩溃恢复 ⇒ 恢复会重放 B-effect
   ⇒ 若 B 写入本身有 bug 或 B 文件损坏，M0 无修复手段
```

**可接受**（B 损坏概率低），但应在 M0 文档中声明"B 损坏需人工处理"。

### 缺口 ④：M0 的 A/B 语义分叉未修（F-02/F-04 在 M1）

```
F-02（A 层不脱敏/不截断 vs B 层脱敏/截断）→ M1
F-04（B 淘汰 A 不淘汰）→ M1

⇒ M0 期间：
   A 层仍存未脱敏原文（隐私风险）
   A 层仍不淘汰（B 已淘汰的记录仍被向量召回）
⇒ M0 的"一致性"只覆盖"写入原子性"，不覆盖"语义一致性"
```

## 9.3 结论

**M0 不具备"独立安全运行"的最低条件**，理由：

1. F-11 只解决一半（无恢复流程）
2. F-08 未解决（embed 失败丢正文），且与 WAL 的 effects 表达冲突
3. F-02（A 层存储未脱敏原文）未解决 → **隐私风险仍在**
4. F-04（过期记忆仍被召回）未解决

**建议重新划分 M0**：

```
M0 必须包含（否则不能上线）：
  F-01（写队列）✅ 已在
  F-02（A 层脱敏 + 同口径截断）← 必须从 M1 上移
  F-05（审计）✅ 已在
  F-06/F-15（原子写）✅ 已在
  F-11（tombstone + 最小恢复校验）← 恢复校验必须上移
  F-08（至少三态 embedding，正文优先持久化）← 必须上移
  F-13（读失败抛错）✅ 已在
  F-23（canonical key）✅ 已在

M0 可以不含：
  F-04（B 淘汰联动）— 但有隐私残留风险，需明示
  F-09（ShareRequest）— 若 M0 不开放分享功能，可推迟
  F-24/F-26（备份 envelope）— 但需至少保证"备份失败不静默"
```

---

# 十、必须修改的设计问题

## 10.1 ★ P0-1：`rev` 混用导致内容编辑被静默丢弃（D-17）

| 项   | 内容                                                                                  |
| --- | ----------------------------------------------------------------------------------- |
| 位置  | `05 §3 ⑨`、`06 §1.3`、`08 D-17`、`03 §5.3`                                             |
| 性质  | **静默数据丢失 + 功能缺陷**                                                                   |
| 证据  | 设计自述 + 规则推演（`DESIGN_INFERENCE`）                                                     |
| 影响  | ① 用户 edit 因后台向量刷新而 409；② WAL 重放时 pending edit 被 superseded 丢弃；③ pending SR 被元数据变更打断 |
| 修法  | 引入 `contentRev`（仅 text 变化 +1），乐观锁与 superseded 判定改用 contentRev                       |
| 优先级 | **P0（必须在 M0 前定）**                                                                   |

## 10.2 ★ P0-2：memberImpressions 投影未定义（1:N）

| 项       | 内容                                                                                 |
| ------- | ---------------------------------------------------------------------------------- |
| 位置      | `03 §1.1`                                                                          |
| 性质      | **投影模型不完整**                                                                        |
| 证据      | `bridge.js:9251-9269`（`SOURCE_CONFIRMED`）：B 层一个名字一个对象，traits 是数组；A 层每次 append 新增一条 |
| 影响      | A 层 N 条无法映射到 B 层 1 个 key；按 memoryId 删除时无法精确定位 traits                               |
| 修法（二选一） | ① B 层 traits 改为 `[{memoryId, text, at}]` 数组；② B 层 key 改为 `${name}#${memoryId}`     |
| 优先级     | **P0（M0 前必须定，因为删除逻辑依赖）**                                                           |

## 10.3 ★ P1-1：A 层 Schema 缺 B 层独有字段

| 项   | 内容                                                                                                                              |
| --- | ------------------------------------------------------------------------------------------------------------------------------- |
| 位置  | `00 §4`（A Schema）vs `bridge.js:6765`（memory-update 的 newExtra）                                                                  |
| 性质  | **结构缺口**                                                                                                                        |
| 证据  | B 层有 `participants[]` / `pendingQuestion` / `motivation` / `interactionCount` / `lastSeenAt`；A Schema 无对应字段（`SOURCE_CONFIRMED`） |
| 影响  | "B 完全由 A 投影"不可行——这些字段无处存放                                                                                                       |
| 修法  | A Schema 增加 `extra: {participants?, pendingQuestion?, motivation?}` 字段；或明确这些字段是"B 独有、不参与投影"                                     |
| 优先级 | P1                                                                                                                              |

## 10.4 ★ P1-2：Reconciler 的读-改-写边界未定义

| 项   | 内容                                                                                                 |
| --- | -------------------------------------------------------------------------------------------------- |
| 位置  | `03 §1`                                                                                            |
| 性质  | **并发风险**                                                                                           |
| 证据  | `atomicWriteJson(social-v2.json)` 是整文件重写；39 处运行状态写入与 Reconciler 并发会互相覆盖（`SOURCE_CONFIRMED` 43 处调用） |
| 影响  | Reconciler 可能回退刚写入的 recentMessages/unread                                                          |
| 修法  | 明确 Reconciler 只在**写队列内**执行，且必须"读最新快照 → 只改三字段 → 整文件写回"；或把三字段拆到独立文件                                  |
| 优先级 | P1                                                                                                 |

## 10.5 ★ P1-3：embedding 顺序反转未处理

| 项   | 内容                                                                                                                                   |
| --- | ------------------------------------------------------------------------------------------------------------------------------------ |
| 位置  | `02 §1.1 P1~P2`                                                                                                                      |
| 性质  | **语义偏差**                                                                                                                             |
| 证据  | P1 在队列外，入队顺序 = embed 完成顺序；设计无 `requestedAt`/`requestSequence`/`commitSequence`（`NOT_VERIFIABLE` in 设计，但字段确实不存在于 00~09 任何文档）          |
| 影响  | ① 后发的 edit 先提交，先发的 409（用户感知反常）；② createdAt 可能倒序；③ 容量淘汰按 updatedAt 可能错序                                                               |
| 修法  | ① WAL 增加 `requestedAt` + `requestSeq`（进程内单调）+ `commitSeq`；② 或在 P2 入队时按 requestSeq 排序（但违背 FIFO）；③ 或明确接受"提交顺序 = embed 完成顺序"并在 API 文档说明 |
| 优先级 | P1（至少要在文档中明确语义，不能默认）                                                                                                                 |

## 10.6 ★ P1-4：M0 边界不足（见 §9.3）

| 项   | 内容                                 |
| --- | ---------------------------------- |
| 位置  | `00 §6`                            |
| 性质  | **里程碑划分错误**                        |
| 受影响 | F-02（隐私）/F-08（丢正文）/F-11（半解决）       |
| 修法  | 把 F-02、F-08（三态）、F-11（最小恢复校验）上移到 M0 |
| 优先级 | P1（影响能否上线）                         |

## 10.7 P2-1：bridge.lock 心跳启动时机（见 §4.3 缺口①）

| 项   | 内容                            |
| --- | ----------------------------- |
| 位置  | `07 §2` vs `07 §3`            |
| 修法  | 心跳在 S1 后立即启动，S4 列表移除心跳        |
| 优先级 | P2（会导致误接管，但概率取决于 recovery 耗时） |

## 10.8 P2-2：等待次数与 GRACE 不匹配（见 §4.3 缺口②）

| 项   | 内容                                       |
| --- | ---------------------------------------- |
| 位置  | `07 §4 B2a`                              |
| 修法  | 等待次数 = `ceil(GRACE/5s)`，或循环至 age > GRACE |
| 优先级 | P2                                       |

## 10.9 P2-3：writer.lock 价值主张表述

| 项   | 内容                                                   |
| --- | ---------------------------------------------------- |
| 位置  | `07 §1`                                              |
| 修法  | 明确"bridge vs 脚本互斥靠 D-08 强制停止，writer.lock 覆盖脚本 vs 脚本" |
| 优先级 | P2（文档修正）                                             |

---

# 十一、可以保留的设计（质量良好，不建议改）

| #  | 设计                                                           | 理由                      |
| -- | ------------------------------------------------------------ | ----------------------- |
| 1  | **P0~P4 五阶段管线 + 6 条不变量**（`02 §1.1`）                          | 边界清晰，INo-1~INo-6 可逐条验收  |
| 2  | **effect = set-to-state + targetRev 比较**（`03 §5.3`）          | 幂等性正确，是整套恢复的基础          |
| 3  | **崩溃点 ①②③④ 矩阵**（`03 §5.2`）                                   | 覆盖完整，恢复动作明确             |
| 4  | **tombstone 永久保留 + 不存正文**（`04 §3`）                           | 隐私与防复活的最优平衡             |
| 5  | **content_suppression 与 tombstone 分离**（`04 §6`）              | 语义清晰（身份消失 vs 正文不可用）     |
| 6  | **两级审计（Tier-1 阻断 / Tier-2 补偿）**（`06 §2.2`）                   | 直接修 V1「业务成功返回 500」，分级合理 |
| 7  | **备份 envelope + digest + dry-run + confirmToken**（`06 §3~4`） | 专业，覆盖了 F-11/F-24/F-26   |
| 8  | **副本向量复用策略**（`05 §5`）                                        | 省嵌入成本，条件明确              |
| 9  | **迁移的 dry-run + 对账 + 临时文件原子生效 + 失败回退**（`03 §7`）              | 防丢失设计完整                 |
| 10 | **09 的 5 条全局红线**（`09 §12`）                                   | 可直接作为 CI 门禁             |



---

# 十二、尚无法验证的问题

| # | 问题 | 原因 |
|---|---|---|
| N-1 | 设计代码尚未存在（`memory-service.js` 未实现） | 全部设计结论为 `DESIGN_INFERENCE` |
| N-2 | 并发竞态的实际触发概率 | 需运行时测试 |
| N-3 | `truncateText`（`md-to-plain.js`）的确切语义 | 该文件不在 12 个导出源文件中 |
| N-4 | JSONL 全量重写在 10 万条时的耗时 | 需性能测试（当前 0 条） |
| N-5 | Ollama 并发嵌入吞吐与 `maxConcurrent=2` 匹配度 | 需环境测试 |
| N-6 | `fsync` 在 Windows NTFS 上的实际行为与性能 | 需环境测试 |
| N-7 | `consolePortInUse` 在无 netstat 环境的行为 | 需环境测试 |
| N-8 | M0 的 A/B 语义分叉（F-02/F-04）在真实记忆中造成的实际影响 | 当前 A/B 均为空 |
| N-9 | 是否有项目外进程写 `memory-vectors.jsonl` | 静态不可排除 |
| N-10 | `qq_memory_update` MCP 工具是否存在（影响 D-20 的兼容范围） | 未在本次核验范围内逐一确认 |

---

# 十三、对 8 项"特别核验争议"的直接回答

| # | 争议 | 回答 |
|---|---|---|
| 1 | **requestedAt / requestSequence / commitSequence 定义与实际执行顺序** | **设计中不存在这三个字段**（`NOT_VERIFIABLE` in 设计，全文档检索无命中）。WAL 只有 `seq`（提交序号）。因此**无法区分请求顺序与提交顺序**；实际执行顺序 = embed 完成顺序（P1 队列外）。这是 §10.5 的问题。 |
| 2 | **clear 与未完成 embedding 的 append/edit 如何交互** | 设计 `02 §5.5` 明确"clear 不是未来屏障"：clear 之后到达的 append 会写入。若 append 的 embed 在 clear 期间完成，入队后重读为空 → 新增。**语义明确但用户可能意外**（D-16 挂起，建议"不需要屏障"）。这是可接受决策，但需在 UI 文案中说明。 |
| 3 | **contentRev 与 metaRev 是否有必要分离** | **有必要**。当前 rev 混用导致：① 用户编辑被后台向量刷新打成 409；② WAL 重放时 pending edit 被 superseded 丢弃；③ 多群分享审批互相失效。修法成本低（一个字段 + 4 处比较）。见 §5.3、§10.1。 |
| 4 | **pending ShareRequest 是否会被无关元数据变更意外失效** | **会**，且设计自己承认（`05 §3 ⑨`、`06 §1.3`、`D-17`）。触发源：批准其他群的分享、embedding 刷新完成。实际业务影响：分享到 N 个群时，批准第 1 个会让其余 N-1 个失效。见 §7.2。 |
| 5 | **进程内 FIFO 是否与跨进程 writer.lock 配合完整** | **配合基本完整，但职责表述有误**。bridge 全程持有 writer.lock ⇒ 脚本必须等 bridge 停 ⇒ writer.lock 实际只做"脚本 vs 脚本"互斥；"bridge vs 脚本"靠 D-08 强制停止。另外：FIFO 无界（无背压）、`drain(5s)` 与 P1(30s) 时序不匹配。见 §4.1、§4.2。 |
| 6 | **WAL 写入、effect 执行、审计、原子提交之间的顺序** | 设计顺序：`WAL(pending) → Tier-1 审计 → 原子提交(T→A→B→SR) → WAL(committed) → Tier-2 审计`。**该顺序正确**：Tier-1 审计在提交前（失败则 aborted，业务不发生），Tier-2 在提交后（失败进补偿）。唯一问题：多文件"原子提交"实际是顺序写，中间崩溃靠 §5.2 矩阵恢复——设计已正确处理，但 `03 §5.5` 说"不再出现 A 成功 B 失败"表述过强（应说"不作为终态暴露"）。 |
| 7 | **tombstone 与备份恢复的优先级** | **设计正确**：恢复时 tombstone 优先（默认跳过 tombstoned），force 也不跳过墓碑，只有 `ignoreTombstone` 白名单可复活（逐条 + RESURRECT 审计）。唯一缺口：**M0 没有恢复流程** ⇒ tombstone 对人工 copy 无效（§9.2 缺口①）。 |
| 8 | **B 层投影失败后的修复是否可能覆盖更新状态** | **存在风险**（§6.2）：Reconciler 用 `atomicWriteJson` 整文件重写，会回退并发写入的 43 处运行状态。设计未定义 Reconciler 的读-改-写边界。 |
| 9 | **M0 是否缺少必要的 B 层一致性机制** | **缺少两处**：① 无 Reconciler（B 损坏需人工）；② F-04（B 淘汰 A 不淘汰）在 M1 ⇒ M0 期间 A 层仍会保留 B 已淘汰的记忆并被向量召回。见 §9.2。 |

---

# 十四、总体评价

## 14.1 设计质量

```
优点：
  · 执行模型（P0~P4 + 6 条不变量）定义清晰，是 V2.4.1 最大的补强
  · effect = set-to-state 的幂等设计正确，是整套恢复的地基
  · 崩溃点矩阵、tombstone、两级审计、备份 envelope 都很专业
  · 对 V1 缺陷的映射（F-01~F-26）覆盖完整，且自查出 3 处历史报告错误

不足：
  · 对 V1 的两处理解偏差（B 层=投影、memberImpressions 1:1）会直接导致实现走偏
  · rev 混用是架构级隐患，被 D-17 用"简单优先"挂起，但代价是静默数据丢失
  · M0 边界把隐私（F-02）和丢正文（F-08）留给 M1，使 M0 不具备上线条件
  · 缺少 requestedAt/requestSequence，无法审计顺序语义
```

## 14.2 建议

**不建议按当前 M0 直接实施。** 建议顺序：

```
第 0 步（设计定稿前，1~2 天）
  ① 定 contentRev（§10.1）—— 影响所有乐观锁与 SR
  ② 定 memberImpressions 投影（§10.2）—— 影响删除逻辑
  ③ 定 A Schema 是否含 B 独有字段（§10.3）
  ④ 修 M0 边界（§10.6）—— F-02/F-08/F-11 上移

第 1 步（M0 实施）
  按修正后的 M0 实施，但【不开放分享功能】（ShareRequest 在 M1）
  ⇒ 减少 M0 的依赖面（不需要 SR 的 contentRev 校验）

第 2 步（M1 实施）
  ShareRequest + B 投影重建 + 备份 envelope + Reconciler
```

## 14.3 是否具备进入实施评审的条件

```
具备：
  · 02（写并发）、03（WAL）、04（tombstone）、06（审计/备份）、07（锁）的设计已达可实施精度
  · 09 的 71 个 AC 可作为验收基线

不具备：
  · §10.1（rev 混用）、§10.2（memberImpressions）、§10.6（M0 边界）必须先定
  · 其余 5 项可在实施中迭代
```

---

# 十五、只读声明

```
✅ 本轮只读：
   · 读取 10 份设计文档（2090 行）+ 第二轮审查报告（612 行）
   · 读取 D:\qq-bridge 源码：memory-vector.js（全文 380 行）、bridge.js（定向：锁 1050-1165、
     canonicalV2Key 1370-1377、share-request 6882-6902、memberImpressions 9251-9273、
     saveSocialV2State 全调用点穷举、admin 6707 等）
   · 执行只读命令：正则穷举、行号定位、文件读取

✅ 未修改：项目代码 / 配置 / state 数据 / 记忆数据 / 原设计文档 / 原审查报告
✅ 未启动或停止任何服务
✅ 未执行迁移
✅ 未执行业务写入
✅ 未安装依赖
✅ 未自动修复任何问题
✅ 只新增本报告一个文件（D:\memory-v242-independent-review\MEMORY-V2.4.2-INDEPENDENT-REVIEW.md）
```

---

*报告完成，停止。等待人工审核。*
