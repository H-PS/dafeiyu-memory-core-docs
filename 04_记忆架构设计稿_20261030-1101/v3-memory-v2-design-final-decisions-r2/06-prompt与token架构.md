# 06 · R2 prompt 与 token 架构

> 合并人格注入、Ledger、token 账本、缓存指标、prompt-dump。本文件含 **Issue 4 的安全关键一致性结论**。

---

## 一、人格注入（B-1，保持）

- 人格卡**每轮全文注入**，不降频、不引入 `personaAnchorEveryTurns`。
- Ledger 只优化 4 个规则块（表情/黑话/睡眠/首回复），人格不在 Ledger 内。

## 二、Prompt Ledger（保持 R1 `04` §二）

| 块 | 锚点 | 语义 |
|---|---|---|
| stickerList | 0 | 仅 hash 失效 |
| slangTable | 15 | 每 15 轮补发 |
| sleepRules | 20 | 每 20 轮补发 |
| replyRules | 0 | 仅 hash 失效 |

- `shouldSendFull` 已修 `ANCHOR > 0` 前置（P0-6）。
- 会话丢失：`ensureSession` 的 `not-found` 分支必须 `ledger.forget(key)`。
- **F-3 未确认（压缩比）→ 锚点为初值，实施后按 V-7 实测校准。**

## 三、Token 账本与缓存指标（V-8 关键）

- 实际用量只来自 `token-ledger.js`：`cacheMiss/cacheHit/cacheWrite` 桶（`:23` 格式、`:464-466` 幂等折叠、`:686`）+ `cacheHitRate = cacheHit/inputTok`（`:709`）。
- bridge 将 DSH usage **原样透传**给账本（`:2434-2496`），自身不解析缓存字段。
- **风险**：`:709` 在字段缺失时输出 0，无法区分"真 0"与"数据源无字段"。
- **硬约束（C-10 相邻）**：V-8 确认字段存在前，`cacheHitRate` 标 `N/A`；**禁止引用 `:709` 的 0 值、禁止估算冒充实测**。
- `estimateTokens` 只做预算，不报告实际节省。

## 四、Prompt-dump 政策（Issue 4 结论，安全关键）

### 4.1 当前源码行为（`CONFIRMED`，本轮核验）

`deliverPromptNow` 开头**无条件、无开关**地：

```js
fs.appendFileSync(path.join(STATE_DIR, 'prompt-dump.log'),
  '\n\n========== ' + new Date().toISOString() + '  ' + key + '  ==========\n' + promptText + '\n');
```

- 位置：`bridge.js:9485-9486`。
- **写入的是完整 `promptText`，其中 reserved2 唤醒/提醒/notify-ai 的 prompt 含【会话令牌】行（`:4363`/`:10674`/`:10751`）→ 当前凭据明文被落盘。**
- 无 mode 开关、无配置项、无条件限制。`CONFIRMED`。

### 4.2 结论：19 号对、23 号错

- 19 号 §2.1"deliverPromptNow 追加完整 promptText（含凭据）到 prompt-dump.log"——**属实**。
- 23 号"dump 只含 meta/structure、无正文"——**描述的是 R1 目标态，不是当前行为**；R2 以本节区分"当前 vs 目标"。

### 4.3 R2 目标行为（B-3 + C-10，**Correctness**）

| 项 | 目标态 |
|---|---|
| 模式 | `off / meta / structure`（默认 meta；full 取消） |
| meta | 块名/字符数/hash/estTokens，无正文无凭据 |
| structure | meta + 4 具名规则块 + 3 类 id 列表（**字段白名单**，白名单外即失败，DUMP-02） |
| 凭据 | 任何模式永不落明文；dump 迁移时先清存量含凭据的 prompt-dump.log |
| 正文排障 | 一次性内存捕获（console token + 15min + 单 key + 不落盘） |

> **安全顺序**：`prompt-dump.log` 的凭据明文暴露是**当前已存在的事实风险**（不只是设计缺陷），实施 M0 即应处理（见 `08` 阶段 0）。

## 五、DSH 历史与压缩边界（F-3 + DSH 凭据副本）

- compaction 插件组**启用中**（`qq-chat.patch.yml:37-53`，工具结果修剪 50k/45k/2k）；会话存储 zstd。
- **DSH 凭据副本三级口径**（20 号 T3-1）：
  - `CONFIRMED`：bridge 把 token 放进 prompt 发 DSH；
  - `INFERRED`：prompt 可能随 DSH 会话历史持久化；
  - `UNCONFIRMED`：DSH 是否实际保存/压缩/保留/删除副本。
- 缓解 = 轮换 + 会话归档 + 脱敏索引；**不宣称消除**。
