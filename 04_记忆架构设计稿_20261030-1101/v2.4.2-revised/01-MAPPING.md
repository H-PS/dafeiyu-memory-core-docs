# 01 · V1 缺陷 → V2.4.2 修订版映射表

覆盖原设计 F-01~F-26，并新增本版发现的 N-01~N-05。
缩写：MS=MemoryService，WQ=WriteQueue，WAL=memory-wal.jsonl，T=tombstones，SR=ShareRequest，**P=memory-projection.json**。

---

# 1. P0 缺陷

| 缺陷 | V1 现状（源码） | 修订版方案 | 新增字段 | M0? |
|---|---|---|---|---|
| **F-01** `editVectorMemory` 用 await 前快照全量覆盖（`m-v.js:362/367/372`） | R1 复活 / R2 改错记录 / R3 新记忆丢失 / R4 丢更新 | ①embedding 队列外；②入队后重读；③按 memoryId 定位；④**`expectedContentRev`** 校验；⑤单次原子重写 | `id`、`rev`、**`contentRev`**、`opId` | ✅ |
| **F-02** A 层存未脱敏、未截断原文（`bridge.js:6658`），B 层脱敏+截 200 | 隐私泄漏、A/B 分叉、删除失配 | MS 入口唯一规范化：`redactKnownTokensOnly` + `clampText(1000)`；A/P 用同一份文本，P 只做 200 字展示截断 | `text`（规范化后） | **✅ 上移** |
| **F-03** `aRemoved` 计算后未上报（`:6962/6964/7011`） | A 层残留被吞 | 响应 `{aRemoved, pRemoved, partial, details[]}`；`aRemoved<pRemoved` 强制 `partial:true` | — | ✅ |
| **F-04** B 淘汰、A 不淘汰（`:9210-9215/9234/9250/9264`） | 过期记忆仍被召回 | A 层 `lifecycle` 迁移 + 读时过滤；**M1 做写时淘汰联动** | `lifecycle`、`expiresAt` | M0 部分（读时过滤）/ M1 完整 |
| **F-05** 审计在业务后、无 try/catch；clear/remove 零审计 | 业务成功返 500；无审计 | 两级：Tier-1 阻断（删除/授权/恢复）、Tier-2 补偿 | `opId`、`actor`、`result`、`revBefore/After` | ✅ |
| **F-06** `atomicRewrite` 固定 tmp（`m-v.js:377`） | 并发冲突 | 统一 `atomicWriteLines`：`${file}.${pid}.${rand6}.tmp` + fsync + renameWithRetry + try/finally | — | ✅ |
| **F-07** bridge.lock 窗口（`:1390` acquire → `:7531` listen） | 运行中无锁 | 锁升级为状态文件（pid+instanceId+state+heartbeat）；**心跳 S1 启动**；stale 加 GRACE 等待；release 按 instanceId+pid 双匹配 | `instanceId`、`state`、`heartbeatAt` | ✅ |
| **F-08** embed 失败即丢正文（`m-v.js:153`） | 正文永久丢失 | 正文先落盘（`embedding.status=pending`），嵌入异步；**M0 三态**，M1 六态 + 重试 | `embedding`、`vecContentRev` | **✅ 上移（三态）** |
| **F-09** 审批依赖四级模糊匹配（`m-v.js:235-258`） | 可能批错记忆 | 申请/审批一律 memoryId；**审批校验 `expectedContentRevAtRequest`** | `requestId` | **M0 关闭分享** / M1 |
| **F-10** append 无去重、无 opId | 重复累积 | opId 幂等 + contentHash 去重（同 key+category+text） | `opId` | ✅ |
| **F-11** 无 tombstone / 无恢复函数 / 备份无 schema | 恢复必复活 | 删除写 tombstone（永久）；**M0 提供最小恢复校验函数**（读备份→比对墓碑→过滤）；M1 加 UI | `id`、`schemaVersion` | **✅ 上移（校验函数）** |

---

# 2. P1/P2/P3 缺陷

| 缺陷 | V1 现状 | 修订版方案 | M0? |
|---|---|---|---|
| **F-12** 坏行静默跳过（`m-v.js:53`） | 静默丢 | 坏行移入 `state/memory-store.badlines` + 告警 + 审计 | ✅ |
| **F-13** 读异常返回 `[]`（`:57`） | 静默空 | 抛错 → 端点 500 | ✅ |
| **F-14** vec 无维度/模型校验 | 换模型后静默空 | 写入校验 `length===vecDim` + finite + 范数>0；记录 `vecModel` | M0 维度校验 / M1 模型隔离 |
| **F-15** 无 fsync / 无 renameWithRetry | 断电丢 | 与 F-06 同一方案 | ✅ |
| **F-16** 缓存仅靠 mtime（`:46-47`） | 同毫秒读旧 | `storeVersion` 计数器 | ✅ |
| **F-17** 审计含正文 80 字、无轮转 | 隐私/膨胀 | 只存 contentHash + textLen；10MB×10 轮转 | ✅ |
| **F-18** 未知 op 文案过时（`:6759`） | 误导 | op 表驱动 | ✅ |
| **F-19** `countVectorMemory` 死代码 | 混淆 | V2 不导出 | ✅ |
| **F-20** 条件式鉴权 96 处 | 依赖兜底 | 记忆端点全部无条件三重校验 | ✅ |
| **F-21** 文档与产物不符 | 流程 | 本修订版如实记录 | — |
| **F-22** `qq_memory_query` A 失败丢 B 结果 | 可用性 | 分层返回 + `degraded:true` | M1（M0 带标记） |
| **F-23** A 用原始 key、B 用 canonical（`:6658` vs `:8765`） | 前导零分裂 | 入口统一 `canonicalV2Key` | ✅ |
| **F-24** dailyBackup 非原子 + catch{} + 仅 append 触发 | 备份失败不可知 | envelope + 原子写 + 失败告警；触发点扩展 | M1 完整 / M0 加时间戳 |
| **F-25** MEM_CONFIG 加载期只读一次 | 配置不生效 | 每次嵌入取当前 cfg | M1 |
| **F-26** admin-backup 同名覆盖（`:6716`） | 仅留最后状态 | 文件名带时间戳 | ✅ |

---

# 3. 本版新增发现（N-01~N-05）

> 均来自对源码的补充核验，不在原设计与原独立报告的缺陷清单中。

## N-01 ★ `memory-update` 端点只改 B 层、完全不碰 A 层（`SOURCE_CONFIRMED`）

```
位置：bridge.js:6765-6831
行为：
  :6789-6794  activeTopics[idx].text / pendingQuestion / participants 就地改
  :6795-6800  pendingThoughts[idx].text / motivation / expiresAt 就地改
  :6801-6827  memberImpressions[target].traits / interactionCount / 改名
  :6828       saveSocialV2State()          ← 非 strict，吞错
  ★ 全程无 appendVectorMemory / editVectorMemory 调用
后果：
  改 B 层 → A 层完全不知 → 检索仍返回旧文本（A/B 语义分叉）
  且用 findIndex(t => t.text === oldContent) 按文本定位，无 id
影响：
  ① 这是 V1 第 4 个"A/B 分叉源"（原报告只列了 append/remove/clear 三处）
  ② M1 必须把它改为转发 MS.edit(memoryId, expectedContentRev)
  ③ M0 期间若保留此端点，必须在响应中标注"A 层未同步"
```

**对设计的影响**：`03 §4` 说"B 单独修改入口取消"是对的，但**必须明确这是 4 条分叉路径之一**，且 M0 要给出临时处置（建议 M0 直接关闭此端点，返回 501，避免继续制造分叉）。

## N-02 ★ memberImpression 的 traits 上限两条路径不一致（`SOURCE_CONFIRMED`）

```
append 路径（bridge.js:9257）：  traits.push(...);  且 traits.slice(0,10) 读时截取
update 路径（bridge.js:6815）：  im.traits = newContent.split(...).slice(0, 20)
⇒ 同一字段，两个上限（10 vs 20）
```

**对设计的影响**：P 层 traits 上限统一为 **10**（取小值），并写进验收（AC-5.x）。

## N-03 ★ `memory-update` 支持 memberImpression 改名（`SOURCE_CONFIRMED`）

```
:6812  const newTarget = String(cleanNewExtra.target || oldTarget).trim().slice(0, 64);
:6818  if (newTarget !== oldTarget) delete st.memberImpressions[oldTarget];
:6819  st.memberImpressions[newTarget] = im;
```

**对设计的影响**：P 层的 `memberImpressions` key 是**可变**的。若 A 层用 `subjectId` 作身份，改名意味着 `subjectId` 变化 → 需要一条 `edit` 操作同时改 `subjectId`。设计需明确：**`subjectId` 是否允许编辑**。

- 本版决策：**允许**，但走 `MS.editSubjectId`（独立 op，`expectedContentRev` 校验），并同步更新 P 层 key。列入 `08 D-22`（待批准）。

## N-04 ★ `reset` 会整文件重写 social-v2.json（`SOURCE_CONFIRMED`）

```
bridge.js:4624  try { atomicWriteJson(SOCIAL_V2_FILE, { conversations: {} }); } catch ...
```

**对设计的影响**：
- 这是第 44 个 S 层写入点（不在 43 处 `saveSocialV2State` 之列）
- 它把**整个 conversations 清空** ⇒ 若 P 层独立，reset 不清 P ⇒ **A 层记录仍在，P 层也仍在** ⇒ 语义为"重置会话但不删记忆"（这是合理的，但必须写清楚）
- 若 reset 意图是"彻底重置"，则应同时清 A 层（走 `MS.clear`）

**本版决策**：reset 只清 S 层，不动 A/P。若用户要清记忆，应显式调用 `qq_memory_clear`。写入 `08 D-23`（待确认）。

## N-05 ★ `formatMemoryV2` 读的是 `st`（S 层），本版需改为读 P 层（`SOURCE_CONFIRMED`）

```
bridge.js:8945-8975  formatMemoryV2(st) 读 st.activeTopics / st.pendingThoughts / st.memberImpressions
调用点：memory GET 端点、memory-update 端点、memory-append 端点
```

**对设计的影响**：P 层独立后，`formatMemoryV2` 必须改为 `formatMemoryProjection(projection[convKey], S 层可选的运行状态)`。
M0 双写期：优先读 P，P 缺失降级读 S（兼容）。

---

# 4. 缺陷 → 里程碑归并（修订）

| 里程碑 | 缺陷 | 数量 |
|---|---|---|
| **M0** | F-01、F-02、F-03、F-05、F-06、F-07、F-08（三态）、F-10、F-11（校验函数）、F-12、F-13、F-15、F-16、F-17、F-18、F-19、F-20、F-23、F-26、**N-01（关闭端点）** | 20 |
| **M1** | F-04（写时淘汰）、F-09、F-14（模型隔离）、F-22、F-24（完整）、F-25、**N-02/N-03/N-05** | 9 |
| **M2** | factKey 模板、关系七型、ShareSuggestion、SQLite、WAL 压缩、interactionCount 迁 S 层 | — |
| 降低概率/补偿 | F-07（锁窗口收窄）、F-10（语义重复）、F-12（磁盘级）、F-14（切换窗口）、F-16（同毫秒） | — |

**与原设计的差异**：
- M0 从 15 项增至 **20 项**（F-02/F-08/F-11 上移 + N-01）
- M1 从 9 项减为 **9 项**（但因 M0 上移，工作量重新分配）

---

# 5. 与原设计的映射差异（逐条）

| 原映射 | 修订 | 原因 |
|---|---|---|
| F-02 → M1 | **→ M0** | 隐私风险不能延后 |
| F-08 → M1 | **→ M0（三态）** | 正文丢失不能延后；六态仍 M1 |
| F-11 → M0（tombstone）+ M1（恢复） | **→ M0（tombstone + 最小恢复校验）** | 否则人工恢复绕过墓碑 |
| F-03 → M0 | 保持 | — |
| F-04 → M1 | 保持，但 M0 加"读时过滤"补偿 | 减少过期召回 |
| F-09 → M1 | 保持，M0 **关闭分享端点** | 避免用 V1 模糊匹配授权 |
| F-14 → M1 | 保持，M0 加维度校验 | 防坏向量入库 |
| F-22 → M1 | 保持，M0 响应带 `degraded` 标记 | 最小可用性 |
| — | **新增 N-01~N-05** | 源码补充核验发现 |
