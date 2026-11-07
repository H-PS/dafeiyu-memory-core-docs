# 04 · Prompt Ledger 与凭据生命周期（Rev3.1 修订版）

> 修订依据：P0-6 / P0-7 / P0-8 / P0-9 / P0-10 / P1-2 / P1-3 + 五项一致性复核。
> 本文件替代 `D:\v3-memory-v2-design\04-Prompt-Ledger与凭据生命周期.md`；原文件保留不动。
> **Rev3.1-FD（最终决策版）**：已落实 **D2（`/reset` 触发 token 轮换）**、**D4（允许 structure dump，默认 meta）**、**D5（OS 密钥库暂缓）**。
> 对应改动见 §5.4、§7.1、§6.3；决策映射见 `17-决策落实对照表.md`。
> **Rev3.1-FD-R1（2026-10-31 纠错）**：§7.1 新增 structure 字段白名单硬约束；DUMP-02 扩展为"白名单外字段出现即失败"；见 `18-纠错收敛对照表.md`。
> **纯设计，不构成实施授权。**

---

## 〇、本版修了什么

| 编号 | 原问题 | 修法 | 位置 |
|---|---|---|---|
| **P0-6** | `shouldSendFull` 里 `turn - sentTurn >= ANCHOR`，`ANCHOR=0` 时恒真 → 每轮都重发，"0 = 不重发"的意图反了 | 显式加 `ANCHOR > 0` 前置；0 = 仅按 hash 失效 | §2.3 |
| **P0-7** | 一边说"凭据每轮发"，一边说"发成功后 `needsDelivery=false`"，语义冲突；且"构造出字符串"不等于"投递成功" | 拆成三个概念：**每轮注入** / **交付标记** / **DSH 接受**；只有 `accepted.result.ok` 才算交付 | §3 |
| **P0-8** | `credential.rotate()` 在 `retireSession()` 之前，而 `retireSession()` 内部**也**生成新 token（`:2795`）→ 双重轮换、状态分叉 | 唯一生成者 = `credential.js`；删除 `:2795` 的内联生成；`st.agentToken` 不再作为真相 | §4 |
| **P0-9** | `/reset` 现状是"删会话→token 随之消失"，Rev3 改成"轮换"，属行为变更但未标注 | 明确标注为**有意变更**，并定义立即失效/在途调用/失败恢复/与 journal 的关系；**已确认（D2）** | §5 |
| **P0-10** | `agent-token.json` 只写了 0600，未定义备份/导出/脱敏/损坏恢复/回滚/平台差异 | 六项逐一定义 + 明确"0600 不是完整保护" | §6 |
| **P1-2** | 全文 dump 的授权条件写成 `caller==='agent'` | 取消"全文落盘"；改为 `structure` 模式 + 一次性内存捕获 + 独立授权 | §7 |
| **P1-3** | 平台权限未定义 | Linux/Windows 分开定义 + **必须验证生效** | §6.6 |

---

## 一、块分级（保持 Rev3：人格不降频）

| 级别 | 块 | 策略 |
|---|---|---|
| **L-LOCKED** | 人格卡全文 | **每轮全文**（R3 强制，本版不改） |
| **L-SECRET** | 会话令牌 | 由 `credential.js` 提供，**每轮注入**；Ledger 不参与判定 |
| **L-LEDGER** | 表情清单 / 睡眠规则 / 黑话表 / 首回复规则 | hash + 锚点（§2） |
| **L-DYNAMIC** | 身份 / B 层 / 参与度 / 状态 / 唤醒配置 / 快照 | 每轮，受预算约束 |

**注入令牌的两个位置**（本版核实）：

| 位置 | 场景 |
|---|---|
| `bridge.js:4363` | `/api/socialV2/prompt` 端点返回的 prompt（MCP `qq_get_prompt` 走这条） |
| `bridge.js:10674` | `buildWakePromptV2`（自动唤醒投递） |

⇒ 改造时必须**两处都改**，否则会出现"一处新 token、一处旧 token"的分叉。

---

## 二、Ledger（P0-6 修正）

### 2.1 结构

```jsonc
// state/prompt-ledger.json（独立文件）
{
  "version": 1,
  "keys": {
    "group:123456": {
      "sessionId": "sess_abc",
      "sessionEpoch": 7,
      "turn": 23,
      "blocks": {
        "stickerList": { "hash":"a1b2c3", "sentTurn":20, "sentAt":1761900000000 },
        "slangTable":  { "hash":"d4e5f6", "sentTurn":18, "sentAt":1761900000000 },
        "sleepRules":  { "hash":"112233", "sentTurn":1,  "sentAt":1761900000000 },
        "replyRules":  { "hash":"445566", "sentTurn":1,  "sentAt":1761900000000 }
      },
      "injectedMemo": [ { "fp":"m_9f2a", "turn":12 } ]
    }
  }
}
```

### 2.2 各块的锚点值

| 块 | `ANCHOR` | 语义 |
|---|---|---|
| `stickerList` | **0** | 仅按 hash 失效（收藏库不变就永不重发） |
| `slangTable` | 15 | 每 15 轮补发一次（防被历史压缩挤掉） |
| `sleepRules` | 20 | 同上 |
| `replyRules` | **0** | 仅按 hash（其动态数值部分每轮都发，规则段丢了影响有限） |

### 2.3 `shouldSendFull()` —— P0-6 的修正

```js
// ❌ 旧（错误）：ANCHOR=0 时 turn - sentTurn >= 0 恒真 → 每轮重发
function shouldSendFull(rec, hash, turn, epoch, curEpoch) {
  if (curEpoch !== epoch) return true;
  if (!rec) return true;
  if (rec.hash !== hash) return true;
  if (turn - rec.sentTurn >= ANCHOR) return true;   // ← BUG
  return false;
}

// ✅ 新：0 明确表示"无周期重发"
function shouldSendFull({ rec, hash, turn, anchor, epoch, curEpoch }) {
  if (curEpoch !== epoch) return true;               // ① 会话代际变了 → 全量重发
  if (!rec) return true;                             // ② 从未发过
  if (rec.hash !== hash) return true;                // ③ 内容变了
  if (Number(anchor) > 0 && (turn - rec.sentTurn) >= Number(anchor)) return true;  // ④ 锚点到了
  return false;
}
```

**判定优先级**：① > ② > ③ > ④。任一为真即重发，全假才省。

### 2.4 三组必测用例（Review 要求）

| 用例 | 输入 | 期望 |
|---|---|---|
| ANCHOR-0 | `anchor=0, rec.hash===hash, epoch 相同, turn=100, sentTurn=1` | **false**（不重发） |
| ANCHOR-POS | `anchor=20, turn=20, sentTurn=0` | true（第 20 轮重发） |
| ANCHOR-POS-未到 | `anchor=20, turn=19, sentTurn=0` | false |
| HASH-CHANGED | `anchor=0, hash 变` | **true**（立即重发） |
| EPOCH-CHANGED | `epoch 不一致` | true |
| NO-REC | `rec=null` | true |

### 2.5 原子性 / 并发 / 崩溃 / 会话丢失（一致性复核项 1）

| 项 | 规定 |
|---|---|
| 写者 | 仅 `prompt-ledger.js`；`atomicWriteJson()`（随机 tmp + rename） |
| 并发 | 每 key 串行（`withKeyLock('ledger:'+key)`）；不与其他锁混用（锁序：token → memory → ledger） |
| 崩溃 | 文件损坏/解析失败 → **视为空**（L3 可丢弃，重发一次即可），记审计 `ledger_reset_corrupt` |
| 会话丢失 | `ensureSession` 的 `applied === 'not-found'` 分支**必须**调 `ledger.forget(key)`（`bridge.js:8471-8476`）。否则新会话以为"已发过"，规则块永不出现 |
| 会话轮转 | `epoch++` 天然触发全部重发；另显式 `forget(key)` 双保险 |
| 大小 | 每 key ≈2KB；50 会话 ≈100KB，无需淘汰 |

---

## 三、P0-7 · 凭据交付的三个独立概念

### 3.1 三个概念（不再混为一谈）

| 概念 | 含义 | 判据 |
|---|---|---|
| **A. 每轮注入** | prompt 里**永远**有当前有效 token | `credential.get(key).token` 非空即注入。**与 `needsDelivery` 无关** |
| **B. 交付标记** | 这个 epoch 的 token 是否**已成功告诉模型** | `deliveryState === 'accepted'` 才可清 `needsDelivery` |
| **C. DSH 接受** | DSH 是否真的收下了这条 prompt | `accepted.result.ok === true`（`bridge.js:9548`） |

**关键**：A 恒为真（每轮都发），B 记录"这一代是否成功送达过"。`needsDelivery=true` 不代表"不注入"，只代表"要重点确认送达"，并在未送达时告警。

### 3.2 交付状态机

```
pending ──构造出 prompt──► constructed ──DSH 接受──► accepted（清 needsDelivery）
   ▲                            │
   │                            └──入离线队列──► queued（不算交付）
   │                            └──DSH 未接受──► failed（保持 needsDelivery）
   └───────────────────────── 下一轮重试 ─────────────────────┘
```

| 状态 | 何时 | `needsDelivery` |
|---|---|---|
| `pending` | 新建会话 / 轮换后尚未投递 | true |
| `constructed` | prompt 字符串已拼出（**不算交付**） | true |
| `queued` | DSH 未就绪，进了 `queued`（`:9498-9507`） | true |
| `accepted` | `api.sessions.prompt` 返回 `result.ok === true`（`:9548` 的反面） | **false** |
| `failed` | 返回 `result.ok === false`（`:9548`）或抛错 | true |

### 3.3 代码落点

```js
// deliverPromptNow 内，拼接后
credential.markConstructed(key, epoch);

const accepted = await api.sessions.prompt({ sessionId, mode:'queue', content });

if (!accepted.result.ok) {
  credential.markDeliveryFailed(key, epoch, accepted.result.error?.code);   // 保持 needsDelivery
  return { ok:false, error: … };
}
credential.markAccepted(key, epoch);   // 只有这里清 needsDelivery
```

**离线队列分支**（`:9498`）必须显式标记 `queued`，不能沿用 `constructed`（否则会被误判为"已交付但失败"）。

### 3.4 告警

`needsDelivery` 连续 3 轮仍未被 `accepted` → 控制台告警 `credential_delivery_stalled`，审计记 `{key, epoch, attempts, lastState}`。

---

## 四、P0-8 · 凭据唯一真相源

### 4.1 现状：token 有 3 个赋值点 + 1 个持久化拷贝（本轮核实）

| # | 位置 | 行为 | 字节数 |
|---|---|---|---|
| 1 | `bridge.js:8782` | 会话初始化：`agentToken: crypto.randomBytes(16).toString('hex')` | 16 |
| 2 | `bridge.js:2795` | `retireSession` 轮换：`st.agentToken = crypto.randomBytes(24).toString('hex')` | **24** |
| 3 | `bridge.js:8945` | `buildSocialV2Snapshot()` **把 token 拷进快照** → 随每次 S 层保存落盘 | — |
| 4 | `bridge.js:11308` / `:4613` | `forgetAgentToken(key)`（只清脱敏集合，不改 `st.agentToken`；随后 `socialV2.conversations.delete(key)` 使 token 消失） | — |

**附带发现（不一致）**：初始化 16 字节、轮换 24 字节。轮换后长度变化，虽不影响 `timingSafeEqual`（比较前先比长度，`tokenEq :3140-3145`），但属实现不一致，本版统一为 **24 字节**。

**读取点**：`agentTokenOk :3149`、`isLiveAgentToken :3162`、`prompt 注入 :4363` 与 `:10674`、快照 `:8945`。

### 4.2 双重轮换问题（Review 指出的具体冲突）

Rev3 的顺序是：`credential.rotate()` → `retireSession()`。但 `retireSession()` 内部（`:2795`）**又**生成一个 token。结果：

```
credential 里: T2（epoch=8）
S 层 st.agentToken: T3（另一个随机值）
⇒ 认证读 S（T3），prompt 注入读 credential（T2）
⇒ 模型拿 T2 调工具 → 403
```

**必须消除两个真相源**，不能靠"顺序调整"绕开。

### 4.3 决定：`credential.js` 是唯一生成者，`st.agentToken` 移除

| 改动 | 位置 | 内容 |
|---|---|---|
| 删除内联生成 | `:2795` | 改为 `credential.rotate(key, 'retireSession', opId)`；不再写 `st.agentToken` |
| 删除初始化生成 | `:8782` | 改为 `credential.ensure(key)`（幂等：已存在则复用） |
| 移除快照拷贝 | `:8945` | 快照**不再包含 token**（这一步同时解决"每次保存 S 都写一遍凭据明文"） |
| 改认证读取 | `:3149` | `agentTokenOk` → `tokenEq(token, credential.get(key)?.token)` |
| 改全局判活 | `:3162` | `isLiveAgentToken` → 遍历 `credential.list()` 的 token（或复用 `KNOWN_AGENT_TOKENS` 做脱敏，但认证只走 credential） |
| 改注入读取 | `:4363` `:10674` | 取 `credential.get(key).token` |
| 改重置清理 | `:11308` `:4613` | `credential.retire(key, opId)`（同时清脱敏集合） |
| 保留脱敏 | `:540` `:561` | `rememberAgentToken` / `forgetAgentToken` 保留，改为由 credential 事件驱动（它们的职责只是"进脱敏集合"，不是"认证"） |

**硬规矩**：全仓 `agentToken` 的**赋值**只允许出现在 `credential.js`。验收方式：

```bash
grep -rn "st\.agentToken\s*=" src/        # 必须为 0
grep -rn "agentToken\s*:" src/            # 只允许 credential.js 内部与快照（快照已移除 → 0）
```

### 4.4 `rotate()` 的幂等（Review 要求保留 opId 幂等）

```js
rotate(key, by, opId) {
  const cur = state.get(key);
  const targetEpoch = (cur?.epoch ?? 0) + 1;

  // journal 已记录过同一个 opId → 只保证 epoch 推进到 targetEpoch
  const applied = journal.findApplied(opId);
  if (applied) {
    if ((cur?.epoch ?? 0) < applied.epochTo) {
      state.set(key, { token: newToken(), epoch: applied.epochTo, needsDelivery: true, by, opId });
    }
    return { epoch: applied.epochTo, idempotent: true };
  }

  journal.recordPending({ opId, op:'credential_rotate', key, epochTo: targetEpoch });
  state.set(key, { token: newToken(), epoch: targetEpoch, needsDelivery: true, by, opId });
  journal.recordCommitted(opId);
  return { epoch: targetEpoch, idempotent: false };
}
```

**必须写清的一点**：幂等的是 **epoch 推进**，不是 token 值。崩溃重放会生成不同的 token，这是可接受的——旧 token 已失效，只需要保证"epoch 只增一次、`needsDelivery` 为 true"。

**journal 里不存 token**（否则又造出一份明文副本），只存 `opId / key / epochTo`。

### 4.5 与 `rotation-journal` 的协作

```
rotationTx（state/rotation-journal.json）的 effects 里包含：
  { type:'credential_rotate', key, epochTo }

重放逻辑：epoch >= epochTo ⇒ 跳过；否则调 credential.rotate(key, by, 同一 opId)
```

⇒ journal 与 credential 各自维护自己的幂等键，但通过 `opId` 对齐，不会出现"journal 说做了、credential 没做"。

---

## 五、P0-9 · `/reset` 的行为变更（显式声明）

### 5.1 现状 vs 本版

| 项 | 现状（`bridge.js:11268-11319`） | Rev3.1 |
|---|---|---|
| token | `forgetAgentToken(key)`（清脱敏集合）+ `socialV2.conversations.delete(key)`（token 随之消失） | `credential.rotate(key,'reset',opId)`（换新 + `needsDelivery=true`） |
| 旧 token | 立即随对象删除而失效 | **立即失效**（epoch 变了，认证只认新值） |
| 会话映射 | 删除 + `stopSessionWork` + `archiveSession` | 不变 |
| A 层 | **不动** | **不动**（R11 保持） |
| B 层 | `socialV2.conversations.delete` | 不变 |

**这是有意的行为变更**：token 从"消失"改为"轮换"。外部可观察行为一致（旧 token 立刻不能用），内部差异是"总是存在一个有效 token"。

**为什么要改**：
- 现状的"消失"会造成一个**无 token 窗口**：会话对象被删后、下次创建前的任何工具调用都 403 且无法自愈（模型不知道拿什么）。
- 轮换 + `needsDelivery` 让"下一次投递必然带上新 token"，窗口被消除。

### 5.2 必须定义的四个细节

| # | 细节 | 规定 |
|---|---|---|
| 1 | **立即失效** | 旧 token 在 `rotate()` 返回后立刻不可认证（`agentTokenOk` 只比当前值）。不设宽限期 |
| 2 | **在途工具调用** | 可能失败（403）。必须返回 `reason:'stale_token'` + 提示模型"重新调用 `qq_get_prompt` 取新令牌"；不得静默 |
| 3 | **失败恢复** | `rotate()` 与 reset 的其余步骤在同一 journal 事务内；任一步失败 → 事务停在某 phase，启动时续跑；token 已换但 reset 未完成时，下一轮仍可用新 token（可用但上下文未清） |
| 4 | **与 rotation-journal 的关系** | **共享 effect 与 journal，不共享 phase 列表**：<br>· `rotationTx`：prepare → digestSaved → drained → retired → done（含 digest）<br>· `resetTx`：prepare → tokenRotated → sessionRetired → done（无 digest） |

### 5.3 是否与 `retireSession` 共用同一协议

**共用 `credential.rotate()` 与 journal 的 effect 应用逻辑；不共用整个事务**。理由：轮转需要 digest 与"排空在途投递"，重置不需要（用户要求立刻清）。

### 5.4 决策结论（已确认：D2）

> **D2（2026-10-31 已确认）**：`/reset` 的 token 语义由"删除"改为"轮换"。
> 实施口径：`/reset` 调用 `credential.rotate(key,'reset',opId)`，旧 token 立即不可认证，新 token 由下一轮 prompt 携带。
> 备选方案（保留"删除"再补生成）**不再考虑**——它与轮换等价但多一次写盘，无收益。
> 该决策同时关闭 Rev3 的 D-31 与 R31-D1。

---

## 六、P0-10 · 凭据文件安全

### 6.1 文件定义

```jsonc
// state/agent-token.json
{
  "version": 1,
  "keys": {
    "group:123456": {
      "token": "…48 hex…",
      "epoch": 8,
      "issuedAt": 1761900000000,
      "rotatedBy": "init" | "retireSession" | "reset" | "corruptRecovery" | "backupRestore",
      "opId": "op_…",
      "needsDelivery": true,
      "deliveryState": "pending" | "constructed" | "queued" | "accepted" | "failed",
      "lastAcceptedAt": 0
    }
  },
  "restoredAt": 0
}
```

### 6.2 六项逐一定义

| # | 项 | 规定 |
|---|---|---|
| 1 | **备份** | **排除**。`dailyBackup()` 明确跳过 `agent-token.json`。理由：备份是第二份明文副本，且备份常被复制到别处 |
| 2 | **诊断导出** | **永不包含**。任何"打包排障"脚本必须显式排除该文件；导出脚本内置断言 `!files.includes('agent-token.json')` |
| 3 | **日志脱敏** | 不记录任何片段（含前缀）。只允许记录 `tokenHash8 = sha256(token).slice(0,8)` 与 `epoch`。审计事件形如 `{op:'credential_rotate', key, epoch, tokenHash8}` |
| 4 | **损坏恢复** | 解析失败 → **全量重新生成所有 key 的 token**（epoch+1、`needsDelivery=true`、`rotatedBy='corruptRecovery'`），审计 + 控制台告警。**不尝试部分恢复**（损坏说明不可信） |
| 5 | **从旧备份回滚** | 备份不含 token ⇒ 恢复数据后所有 token 必然"对不上"。规定：恢复动作触发 `credential.rotateAll('backupRestore')`，并在 `restoredAt` 记录时间。**禁止把旧 token 写回**（那等于让旧泄漏重新生效） |
| 6 | **平台权限** | 见 §6.6 |

### 6.3 为什么"0600 不是完整保护"（Review 明确要求）

| 残留风险 | 说明 | 是否本版能解决 |
|---|---|---|
| 同用户的其他进程可读 | 0600 只挡其他用户；同一登录用户下的任何进程（含被投毒的编辑器插件）都能读 | ❌ 不能（需 OS 级密钥库/DPAPI） |
| 管理员/root 可读 | 提权即可读 | ❌ 不能 |
| 备份/同步工具可读 | OneDrive/坚果云/Time Machine 会复制 | ⚠️ 部分（已排除备份，但外部工具不管） |
| 进程内存可被 dump | — | ❌ 不能 |
| 文件被快照（VSS/还原点） | — | ❌ 不能 |

**结论**：文件权限只是"降低同机其他用户读取的概率"，不是凭据保护的全部。真正的缓解是"缩短有效期 + 轮换 + 泄露可检测"：

- 轮换触发点：会话退役、重置、模式切换（`reconcileSessionPolicies` 会退役全部会话）、bridge 重启（可选）
- 泄露检测：出站文本出现已知 token 时**直接抛错不发**（现状 `onebotSend` 已有，保留）
- **已确认（D5）：密钥库暂缓，本版不引入。** 理由：跨平台实现差异大、部署门槛升高（`SECURITY_BASELINE` R2 已把这条列为"不修"的取舍）。缓解手段保留三项：轮换、泄露可检测（出站含已知 token 直接抛错不发）、缩短暴露窗口。
  该决策**关闭** Rev3 的 B-6 与 R31-D6；若将来引入 DPAPI/libsecret，只需替换 `credential.js` 的存储层，不影响本文其他结论。

### 6.4 目录级保护

| 项 | 规定 |
|---|---|
| 位置 | `state/agent-token.json`（与 `state/` 同级收紧） |
| 目录 | `state/` 已由 `hardenStateDirAcl()` 切断继承（`SECURITY_BASELINE` §2.5） |
| 文件 | Linux `0o600`；Windows 依赖目录 ACL 继承（无继承 ⇒ 需显式 ACE） |

### 6.5 与 `state/` ACL 的已知副作用

`SECURITY_BASELINE` §2.6 已记载：切继承后 DSH 沙箱写不进 `state/`。这是既有取舍，MemRev3 的所有新文件都在 `state/` 下，因此同样适用。**不改变这个取舍**。

### 6.6 P1-3 · 平台权限的具体定义与验证

| 平台 | 目标 | 实现 | **必须验证**（Review 要求"actually enforced"） |
|---|---|---|---|
| **Linux** | 目录 0700、文件 0600、属主为运行用户 | `fs.mkdirSync(..., {mode:0o700})` + `fs.writeFileSync(..., {mode:0o600})`；注意 **umask 会削弱 mode**，因此写完必须 `fs.chmodSync(path, 0o600)` | 启动自检：`fs.statSync` 检查 `mode & 0o077 === 0`，否则告警 `state_perm_loose` |
| **Windows** | 仅当前用户 + SYSTEM + Administrators | `hardenStateDirAcl()` 切继承 + 显式 ACE（`icacls`） | 启动自检：`icacls <file>` 解析输出，断言**不含** `Authenticated Users` / `BUILTIN\Users`；失败只告警不阻断（沿用现状取舍，`SECURITY_BASELINE` R9） |
| **macOS** | 0700/0600（与 Linux 同） | 同上 | 同上 |

**验证的边界（诚实说明）**：上述自检只能证明"权限位/ACE 配置正确"，**不能证明"其他进程实际读不到"**（那需要第二用户或提权测试，不适合在启动自检里做）。因此：

- 自动化：检查配置（能自动）
- 人工：发布前用另一普通用户账号尝试读取一次（写进发布检查单，`SECURITY_BASELINE` §四 已有类似条目）

---

## 七、P1-2 · prompt-dump 的授权与内容

### 7.1 取消"全文落盘"

| 模式 | 内容 | 默认 |
|---|---|---|
| `off` | 不写 | — |
| **`meta`** | 块名、字符数、hash、`estTokens`、总字符 | ✅ **默认** |
| **`structure`** | `meta` + **规则块文本**（表情策略/睡眠规则/黑话表/回复规则）+ 记忆 id 列表 + 召回 id 列表 + 快照的消息 id 列表 | **已确认允许（D4）**，但默认值仍是 `meta` |
| ~~`full`~~ | ~~完整 prompt 正文~~ | **取消** |

**永不落盘的内容**（任何模式）：
- 凭据（token 明文）
- B 层记忆正文
- 召回结果正文
- 消息快照正文
- 私聊/群聊消息原文

**理由**：这些正是"权限模型要保护的东西"。把它们写进一个没有权限模型的日志文件，等于给整套 `canRead` 开后门。

**structure 模式字段白名单（B-1，硬约束）**：

1. `structure` 模式**只允许**写入 §7.1 表格该行列出的字段（`meta` 字段 + 4 个具名规则块 + 3 类 id 列表）。
2. **白名单之外的字段一律不写**——dump 产物中出现任何白名单外字段即视为缺陷，写入路径必须拒绝并告警（测试 `DUMP-02`）。
3. **新增字段必须先修订本白名单并通过评审**，禁止实施时"顺手加一个字段"。
4. 凭据与敏感字段的排除**独立于白名单继续生效**（上方"永不落盘"清单 + §7.3 导出脱敏）：白名单管"允许写什么"，排除清单管"无论什么模式都不许写"，两者**叠加而非互替**。

### 7.2 一次性内存捕获（真需要正文时）

```
POST /api/debug/capture     需 console token
  body: { key, ttlMs: ≤900000 (15min), confirm: "CAPTURE" }
  行为：仅在内存里保留该 key 后续 N 条 prompt 的**完整文本**（含正文）
        不落盘；进程退出即消失；到 TTL 自动清除
  审计：{op:'debug_capture_start', key, ttlMs, actor, at}
        结束：{op:'debug_capture_end', key, prompts, at}
  限制：同时只允许 1 个 key 处于捕获态
```

**为什么这样设计**：排障确实需要看正文；但"落盘"是不可撤销的泄露面，而"内存 + 15 分钟 + 单 key"是可撤销的。

### 7.3 导出控制

| 动作 | 要求 |
|---|---|
| 导出 dump | console token + 审计 `dump_export{range, count}` |
| 导出前 | 必须过 `redactKnownTokensOnly()` + 移除 credential 块 |
| 导出后 | 文件名带时间戳；**不允许**自动上传/同步 |

### 7.4 `mode` 的配置约束

```json
"promptDump": { "mode": "meta", "retainDays": 7, "maxBytes": 52428800 }
```

- `mode` 只接受 `off | meta | structure`；写入非法值 → 启动即 fail-fast（不是静默降级）。
- 保留 7 天 + 总量 50MB，超限删最旧。
- 目录：`state/logs/prompt-dump/YYYY-MM-DD.jsonl`，0700/0600，**不纳入备份**。

---

## 八、一致性复核五项（Review "Additional consistency review"）

### 8.1 `prompt-ledger.json` 的原子性/并发/崩溃/会话丢失

见 §2.5（已逐项定义）。

### 8.2 prompt 被 DSH "接受"的确切点

```
deliverPromptNow (:9483)
  …
  const accepted = await api.sessions.prompt({ sessionId, mode:'queue', content });   // :9543
  if (!accepted.result.ok) { …失败分支… return }                                      // :9548
  // ← 这一行之后才算"接受"
```

| 状态 | 判定点 |
|---|---|
| 已构造 | `content` 数组拼好（`:9533-9537`） |
| 已入离线队列 | `dshReady === false` 分支（`:9498-9507`），返回 `{queued:true}` |
| **已接受** | `accepted.result.ok === true`（`:9548` 的反面） |
| 已失败 | `accepted.result.ok === false` 或抛错 |

**明确**：`api.sessions.prompt` 返回 `ok:true` 只代表"DSH 收下了这条消息"，**不代表模型已生成回复，也不代表模型已读取 token**。因此交付标记只到"接受"这一层，不追求"已理解"。

### 8.3 所有剩余的 `st.agentToken` 写入

见 §4.1（4 类：`:8782` 初始化、`:2795` 轮换、`:8945` 快照拷贝、`:11308`/`:4613` 会话删除）。

**改造后目标**：`st.agentToken` 完全消失；全仓赋值 0 处。

### 8.4 `credential.rotate()` 与 `rotation-journal` 的幂等

见 §4.4 / §4.5。

### 8.5 权限矩阵覆盖所有生命周期与调用者

见 `03` §5.5（6 状态 × 3 调用者 = 18 格，全部显式）。

---

## 九、本版验收用例编号

| 编号 | 内容 |
|---|---|
| LED-01~06 | §2.4 六组用例（ANCHOR-0 / POS / POS未到 / HASH / EPOCH / NO-REC） |
| LED-07 | ledger 文件损坏 → 视为空 + 审计 |
| LED-08 | DSH 会话丢失（`not-found`）→ ledger 被清 → 规则块重发 |
| CRED-01 | 轮换后旧 token 立即 403（无宽限） |
| CRED-02 | `constructed` 不算交付（`needsDelivery` 仍 true） |
| CRED-03 | 离线入队 → 状态 `queued`，不算交付 |
| CRED-04 | DSH 接受 → `accepted` + `needsDelivery=false` |
| CRED-05 | 同一 opId 重放 → epoch 只增一次 |
| CRED-06 | `st.agentToken` 赋值点为 0（grep 断言） |
| CRED-07 | 快照文件中不含 token（读 `social-v2.json` 断言） |
| CRED-08 | 两处 prompt 注入（`:4363`/`:10674`）取到同一 token |
| RESET-01 | `/reset` 后旧 token 403、新 token 可用 |
| RESET-02 | `/reset` 中途崩溃 → 续跑后状态一致 |
| FILE-01 | `agent-token.json` 不在备份产物中 |
| FILE-02 | 导出包不含该文件 |
| FILE-03 | 审计里只有 `tokenHash8`，无明文 |
| FILE-04 | 文件损坏 → 全量重新生成 + 告警 |
| FILE-05 | Linux 权限位自检通过；Windows ACL 不含 Authenticated Users |
| DUMP-01 | `meta` 模式 dump 不含任何正文与凭据 |
| DUMP-02 | `structure` 模式 dump **仅含 §7.1 白名单字段**；白名单外字段（含 B 层/召回/快照正文）出现即失败 |
| DUMP-03 | 任何模式 dump 不含当前 token（单测断言） |
| DUMP-04 | 非法 `mode` 值 → 启动失败 |
| DUMP-05 | debug capture 到 TTL 自动清除，且不落盘 |
