# 05 · 记忆写入与 Retain

> 对应 R9（写入收敛）、R11（reset 语义）、R12（工具边界）。

---

## 一、现状：五条写入路径，各写各的

| # | 路径 | 入口 | 写 A | 写 B | 去重 | 版本 |
|---|---|---|---|---|---|---|
| 1 | 模型工具 | `qq_memory_append`（`mcp:939`）→ `/api/socialV2/memory-append` | ✅ `appendVectorMemory` | ✅ `appendMemoryV2` | ❌ | ❌ |
| 2 | 管理端 | 控制台 → `/api/memory/admin` | ✅ `editVectorMemory` / `remove*` | ❌ | — | ❌ |
| 3 | 分享审核 | `requestShare` / `approveShare` / `rejectShare`（`:261/:277/:294`） | ✅ 改状态 | ❌ | — | ❌ |
| 4 | 回填脚本 | `seed-group-owner.mjs`（已归档，但历史污染过） | ✅ | ❌ | ❌ | ❌ |
| 5 | 历史第四条 | `#063「喵」`绕过 `infoOK`（代码已不在仓库） | ✅ | — | ❌ | ❌ |

**已知后果**（`【审计】项目全景` P1-3）："写入无去重 → 三条路径各写一份"。

**Rev3 的 Retain 会成为第 6 条**——所以必须先收敛，否则问题放大。

---

## 二、单一写服务：`memory-store.commit(op)`

### 2.1 op 结构

```js
commit({
  opId,                       // 幂等键（必填，全局唯一）
  type,                       // 'append' | 'edit' | 'remove' | 'clear' | 'setLifecycle' | 'setReview'
  key,                        // 会话键（必填，必须已规范化）
  expectEpoch,                // fencing：调用方读到的 clearEpoch
  payload,                    // 类型相关
  actor,                      // 'tool' | 'admin' | 'retain' | 'reflect' | 'script'
  originHint                  // 可选，Retain 提供
})
→ { ok, id, rev, contentRev, metaRev, deduped?: boolean, noop?: boolean, error? }
```

### 2.2 六阶段（与 `01` §4 的 INo 对齐）

```
0 归一化     去 CQ 码 → 脱敏 → 截断 1000 字 → fingerprint → classify(origin,text)
             （纯 CPU，无 IO）
1 预检       内存索引查 tombstone / 状态合法性
             记 observedEpoch = epochs.get(key)
2 embed      锁外；失败 → embedding.status='pending'，不阻断
3 持锁提交   withKeyLock(key):
   3.1 二次预检（tombstone 内存查）+ 校验 epochs.get(key) === expectEpoch
   3.2 写 journal pending（原子）
   3.3 应用 effects（A 记录 / 索引删除 / 统计）
   3.4 写 journal committed（原子）
4 释放锁
5 异步后续  索引追加、向量补算、审计
```

### 2.3 四个并发要素（R7 也要求，这里先定义）

| 要素 | 做法 | 不是什么 |
|---|---|---|
| **单写者** | 只有 `memory-store.js` 打开 A 文件；其他模块通过 `commit()` | ❌ 不是"大家都写，但小心点" |
| **per-key 串行** | `Map<key, Promise>` 链式锁（进程内），持锁覆盖阶段 3 全程 | ❌ 不是"写前重读文件"（那只是让两个写者都读到同一个旧值） |
| **原子替换** | 追加用 `appendFileSync`；修正用 `atomicWriteText`（随机 tmp + rename） | ❌ 不是固定 `.tmp`（现状 `memory-vector.js:376` 的缺陷） |
| **陈旧写防护** | `expectEpoch` fencing：commit 前比对 `epochs.get(key)`，不等则 abort | ❌ 不是靠时间戳或重试 |

> **R7 的提醒是对的**：把字段挪到独立文件**不自动解决并发**。分文件只降低"整文件重写互相覆盖"的概率；真正的串行化仍靠上面的 per-key 锁 + fencing。

### 2.4 journal 与崩溃恢复

```jsonc
// state/journal/memory.jsonl（append-only）
{"k":"pending",   "opId":"op_1", "type":"append", "key":"group:A", "expectEpoch":88, "effects":[…], "ts":…}
{"k":"committed", "opId":"op_1", "ts":…}
{"k":"aborted",   "opId":"op_1", "reason":"cleared_during_write", "ts":…}
```

| 崩溃点 | 恢复动作 |
|---|---|
| 阶段 2（embed 中） | 无 journal 记录，丢弃 |
| 3.2 后 3.3 前 | 重放 effects（幂等：set-to-target，非 delta） |
| 3.3 中 | 已写部分视为完成，索引删除重放 |
| 3.4 中 | 重放 effects（全部已是目标态），补 committed |
| 3.4 后 | 只补做阶段 5 异步项 |

**幂等关键**：effects 携带**目标值**（`lifecycle:'deleted'`），不携带 `rev+1`。`rev` 由 store 内部按"是否真发生变更"决定。

### 2.5 为什么不用完整 WAL（与 Rev2 的差异）

Rev2 设计了 `pending/committed/aborted/superseded/recovery_blocked` 五态 + 五个崩溃点。Rev3 简化为**三态 + 追加式 journal**：

| 理由 | 说明 |
|---|---|
| 单机单写者 | 没有分布式事务需求 |
| 效果幂等 | set-to-target 使重放天然安全；不需要 `superseded` 这种"部分覆盖"状态 |
| 减少状态 = 减少恢复分支 | 五态恢复逻辑的测试成本是实打实的（Rev2 列了 12 个场景） |

**代价**：无法表达"部分成功"。但本项目所有 op 都是单记录操作（append/edit/remove），不存在多记录事务。**若未来出现批量 op，再升级。**

---

## 三、五条路径的兼容适配器（R9 要求"保留现有行为"）

| 原路径 | 适配方式 | 签名变化 | 行为变化 |
|---|---|---|---|
| `appendVectorMemory(key,text,time)` | 内部转 `commit({type:'append',…})` | 无 | 新增去重（同 fingerprint 只 `metaRev++`） |
| `editVectorMemory(key,old,new)` | 转 `commit({type:'edit'})` | 无 | 旧版"await 前快照"竞态修正（await 后重读定位） |
| `removeVectorMemoryExact` / `removeNAll` / `removeAll` | 转 `commit({type:'remove'|'clear'})` | 无 | 返回结构一致；新增 `noop` 标记 |
| `requestShare` / `approveShare` / `rejectShare` | 转 `commit({type:'setReview'})` | 无 | `approved` 不再隐式改 `visibility`；改由 `canRead` 判定 |
| 管理端 list | 直接读，不经 commit | 无 | 新增 `contentClass`/`scope` 字段展示 |
| `seed-group-owner.mjs` | **保持归档，不复活**（R9 允许"文档化的兼容适配器"，但这条路径本身是污染源） | — | 若需回填，走 `memory-maintenance.mjs import` |

**关键决定**：旧函数**不 throw**（Rev2 曾计划 throw 以暴露漏网调用）。理由（R9 明确要求保留行为）：throw 会让任何未迁移的调用点直接崩，而本项目是单人在产线跑的系统，宁可"静默转发"也不要"上线即炸"。改用**审计告警**：旧函数被调用时记 `legacy_write_path` 事件，控制台可见，逐步清零。

---

## 四、Retain（形成）

### 4.1 五类信号（规则，无 LLM）

| # | 信号 | 匹配（示意） | 强度 | category |
|---|---|---|---|---|
| 1 | 自我披露 | `我(很\|超\|最\|有点\|不太)?(喜欢\|讨厌\|爱吃\|怕\|想要\|打算\|准备\|在学\|刚开始)` | 0.9 | memberImpression |
| 2 | 稳定属性 | `我(是\|在\|住\|来自\|读)(.{2,20})(的\|上学\|工作)` | 0.8 | memberImpression |
| 3 | 承诺约定 | `(说好\|约好\|答应\|明天\|下次\|周末\|到时候)(.{2,30})` | 0.75 | activeTopic |
| 4 | 未完成话题 | 复用 `looksLikeUnfinished()`（bridge 已有） | 0.7 | activeTopic |
| 5 | 专名首现 | ASCII ≥3 字符（STM32/CAN/PID）本 key 未见 | 0.65 | activeTopic |

### 4.2 黑名单（**在产生候选之前**执行）

| 类型 | 例 |
|---|---|
| 纯情绪短句 | 哈哈哈 / 6 / 草 |
| 一次性事实 | 今天下雨了 |
| 公共信息转述 | 新闻说… |
| **凭证类** | 手机号 / 身份证 / 密码 / 银行卡 / 验证码 |
| **第三人隐私** | 非本群成员的私人信息、他人病历住址 |

黑名单命中 ⇒ 三池都不进（连 quarantine 都不进），因为这类内容没有留存价值且本身是风险源。

### 4.3 四闸

```
闸1 信息量   infoOK(text)（复用 memory-vector.js:125：≥6 字且 ≥3 汉字）
闸2 新颖度   maxCosSim(候选, 同 key 已有 vec) < 0.92
闸3 长期价值 signalStrength × 类别系数 ≥ 0.55
闸4 权限     classify(origin,text) 必须返回非 null
               public     → scope.read=origin_group（群）/ owner_only（私聊），review=not_required
               sensitive  → scope.read=owner_only，**进 quarantine**
               uncertain  → scope.read=owner_only，**进 quarantine**
               null       → **进 quarantine，classReason='ambiguous'**
```

**闸 4 是硬闸**：不过就不进 A，且**没有"绕过"参数**。

### 4.4 三池物理分离（R5 第 5 条的落点）

| 池 | 文件 | 收什么 | TTL | 能否自动进 A |
|---|---|---|---|---|
| A 层 | `memory-store.jsonl` | 四闸全过 | 永久 | — |
| 候选池 | `state/memory-candidates.jsonl` | 闸2、闸3 失败 | 7 天 | **永不** |
| 隔离区 | `state/memory-quarantine.jsonl` | 闸4 失败 | 无（等人工） | **永不** |
| 补算队列 | `state/pending-embed.jsonl` | embedding 失败（记录已在 A） | 退避 5 次 | 不涉及 |

### 4.5 "重复出现"的正确语义

```
候选再次命中同 fingerprint：
  seenCount++
  若 seenCount >= 3 → 标记 reevaluate=true，下一空闲窗口**重跑全部四闸**
  重跑仍不过 → 保持候选
  重跑通过   → 正常 commit（此时它满足全部闸，含权限闸）
```

**明确禁止**：`seenCount>=3 → 自动晋升`。重复只触发"重新评估"，不构成"通过"。敏感内容说一百次仍是敏感。

### 4.6 新颖度失败的正确归宿

不新增第二条，而是合并到最近记录：

```
target.metaRev += 1
target.extra.mentions.push({ at, seq })
// text 不变、vec 不重算
```

---

## 五、Retain 的触发点与代价

```
消息入库后（不阻塞投递）
  └─► retainScan(key, newMsgs)   异步，setImmediate 排队
        ├─ 黑名单（纯 CPU）
        ├─ 信号匹配（纯 CPU，<1ms/条）
        ├─ 闸1/闸3/闸4（纯 CPU）
        ├─ 闸2 需 embedding → 失败按补算队列处理
        └─ 通过 → commit()
```

**三条铁律**：
1. 必须异步，不得出现在回复主链路。
2. 异常不得影响回复，只记审计。
3. 写入必须走 `commit()`，不得直接 append 文件。

**CPU 代价**：信号匹配 + 四闸（除闸2）在纯 CPU 上 <1ms/条；闸2 一次 embedding（已有 30s 超时，实际 30-80ms），异步排队，不影响回复。

---

## 六、`/reset` 语义（R11）

### 6.1 保持现状（证据）

`【源码】bridge.js:11268-11319`：`/reset` 清 ~25 个内存结构 + `socialV2.conversations.delete(key)` + `forgetAgentToken` + `stopSessionWork` + `archiveSession`；**全段无 `memory-vector` 调用**。

⇒ **现状 = reset 不清 A 层**。Rev3 **保持**。

### 6.2 三个动作的边界

| 动作 | 触发 | 清 A？ | 清 B？ | 清会话 | 授权 |
|---|---|---|---|---|---|
| `/reset` | owner 在聊天里 | ❌ | ✅ | ✅ | owner（已有） |
| `memory clear`（工具） | 模型调 `qq_memory_clear` | ✅（该 key） | ✅ | ❌ | agent token（已有） |
| `purge`（新） | 管理端显式 | ✅（该 key 或全部） | ✅ | ✅ | **二次确认 + 记审计** |

**purge 的设计要求**：
- 必须有 `confirm` 参数（字面量 `PURGE`），不是布尔；
- 必须写审计 `{op:'purge', key, count, actor}`；
- 必须先用 `dailyBackup` 打快照；
- 不允许 agent token 调用（仅 console token）。

### 6.3 为什么不能"顺手让 reset 清 A"

- 用户已习惯"reset 只是重开上下文"（`【审计】项目全景` §六 用户待办里有"别 reset"这种表述，说明它被当作"忘掉刚才聊的"用）；
- 改语义会让一次误操作永久丢失长期记忆；
- B 层已清（进行中的话题没了），若 A 也清，等于"整段关系归零"，且不可恢复。

---

## 七、工具边界（R12）

| 决定 | 说明 |
|---|---|
| 新工具挂 `mcp__snowluma__` 前缀 | 即写入 `mcp-snowluma-safe.js`；`SAFE_PREFIXES` **不改**（`qq-tool-restrict.mjs:215-220`） |
| 不新建 MCP server | 新建需改 `SAFE_PREFIXES` = 安全边界变更 + 回归计划 |
| 新工具进 `socialV2.tools.*` 开关表 | 与现有工具一致，可单独停用 |
| **工具可用性 ≠ 记录级授权** | 能调 `qq_memory_recall` 不代表能读任何一条；`canRead` 才是闸门 |

### 7.1 新增工具

| 工具 | 说明 | 鉴权 |
|---|---|---|
| `qq_memory_recall` | 三路召回（RRF + 重排 + 预算） | 三重校验（同 `memory-vector` 端点） |
| `qq_memory_get`（可选） | 按 id 取单条（管理/排障用） | 三重校验 + `canRead` |

`qq_memory_query` 的 `topic` 分支内部改调 `recall()`，**返回格式保持**（模型无需改调用习惯）。

---

## 八、写入路径的"收敛验证"（可测）

| 断言 | 方法 |
|---|---|
| 只有一处写 A | `grep -rn "appendFileSync.*memory-store\|writeFileSync.*memory-store" src/` 结果 = 1 |
| 旧函数都被调用 | 运行 24h 后查审计 `legacy_write_path` 计数，目标 = 0 |
| 每个 op 有 opId | 抽样 100 条 journal 记录，`opId` 非空且唯一 |
| 重复提交幂等 | 同一 opId 提交两次，A 条数不变 |
