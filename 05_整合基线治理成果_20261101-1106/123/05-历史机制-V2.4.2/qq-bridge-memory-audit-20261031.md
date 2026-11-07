# QQ Bridge · Memory 系统源码审查报告（第二轮 · 独立复核）

- 审查日期：2026-10-31
- 审查对象：`qq-bridge-review-20261031.zip` 内 `source/` 的 12 个源文件
- 审查方式：**只读** · 定向深读 + 全局符号穷举（非逐行全量）
- 权限边界：未修改任何项目文件、未启动/停止服务、未执行业务写入、未调用任何业务接口
- 历史报告 `02-AUDIT-REPORT.md` 仅作**待验证线索**，其结论均已逐条复核

---

# 第 0 部分 · 文件完整性与脱敏核查

## 0.1 SHA256 核验

```
sha256sum -c SHA256SUMS.txt  →  11/11 OK，0 失败
```

| 项 | 结果 |
|---|---|
| 6 个报告 md | 全部 OK |
| 5 个 source 分卷 | 全部 OK，且与 `04-MANIFEST.md` 第 1 节「导出 SHA256」逐一吻合 |
| 12 条「原始 SHA256」 | **NOT_VERIFIABLE** —— 对应 `D:\qq-bridge` 项目原文件，不在交付包内，脱敏不可逆（`05` §2.5 已自认），无法验证「导出副本 ↔ 项目源码」的一致性 |

## 0.2 源码行数与分卷连续性

| 源文件 | MANIFEST 行数 | 实测 | 结论 |
|---|---|---|---|
| `src/bridge.js` | 12,482 | 4200 + 4200 + 4082 = **12,482** | ✅ 精确 |
| 其余 11 个 | — | 全部精确匹配 | ✅ 12/12 |

**bridge.js 分卷连续性四重验证**

| 检查 | 方法 | 结果 |
|---|---|---|
| 段边界 | 围栏内容计数 | 4200 / 4200 / 4082 ✅ |
| 跨段重叠 | 40 行滑窗 MD5 跨段比对 | 0 / 0 / 0 ✅ |
| 段内重复 | 全文自比对 | 0 ✅ |
| 边界语义 | 人工查看接缝 | seg1 末 `}` → seg2 首 `if (!/^[A-Za-z0-9_-]+$/.test(newToken)) {`；seg2 末 `const peak = (h >= 9 && h < 12) \|\| (h >= 14 && h < 18);` → seg3 首 `if (!peak) return 0;` —— 语法连续 ✅ |
| **最强证据** | `node --check` | 拼接后 12,482 行**语法零错误**；9 个 JS + 2 个 JSON 全部通过 ✅ |

`01-BASELINE.md` 末尾称「导出包中 `source/` 下 **8 个** Markdown 文件」——**实际为 5 个**，该处为旧稿残留。

## 0.3 脱敏与敏感信息核查（新增要求）

**结论：未发现任何真实敏感信息，审查可继续。** 未触发中止条件。

| 扫描项 | 方法 | 命中 | 结论 |
|---|---|---|---|
| QQ 号 / 群号 | `'"/[0-9]{5,12}/'"` 与 `ownerQQ|botQQ|groupId = <数字>` | **0** | 无硬编码账号 |
| 长数字串 | `[0-9]{8,}` | 3 处 | 均为时间戳/常量（`1792294529471`、`86400000`、`20971520`），非账号 |
| Token / 密码 / API Key | 关键词扫描 | 12 处 | **全部为空值或环境变量名**：`config.example.json` 的 `consoleToken:""`、`snowluma.accessToken:""`、`dsh.authToken:""`、`webSearch.apiKeyEnv:"DEEPSEEK_API_KEY"`；代码中均为 `cfg.xxx` 读取，无字面量 |
| 邮箱 | 正则 | **0** | — |
| 外网 IP / 域名 | 正则 | 2 类 | 仅 `127.0.0.1`（回环）与 `gxh.vip.qq.com`（QQ 公开业务域名） |
| 真实姓名 | 模式扫描 | **0** | — |

**文档与产物不一致（不影响审查结论，如实记录）**

| # | 文档声明 | 实际情况 |
|---|---|---|
| D-1 | `05` §2.2 定义 5 条替换规则（`[REDACTED_OWNER_QQ]` 等），§2.3 称 `bridge.js`「多处」命中 | 导出源码中 `[REDACTED_*]` 出现 **0 次**；占位符仅存在于 `05` 规则表与 `02` 第 376-377 行的日志摘录中 |
| D-2 | `00-README.md:131` 称「导出源码中的 `[REDACTED_OWNER_QQ]` / `[REDACTED_NAME]` 是占位符」 | 同上，源码中不存在 |
| D-3 | `01-BASELINE.md` 末称 `source/` 下 8 个分卷 | 实际 5 个 |

> 最合理解释：源码本身**不含**硬编码 QQ/姓名，故脱敏实际命中为 0，文档是「按规则处理」的模板表述。**判定为文档口径问题，非脱敏失效。** 原值是否真被移除无法从包内验证 → **NOT_VERIFIABLE**。

---

# 第 1 部分 · 审查方法与覆盖率声明

## 1.1 方法（严格区分「穷举」与「深读」）

1. **全局符号穷举**：对 `bridge.js` 12,482 行全文做正则/符号级检索（11 个 A 层导出函数的全部调用点、9 个记忆端点、130 个 `pathname ===` 路由、`atomicRewrite`/`appendFileSync`/`VEC_FILE` 全部写入口、V2.4.1 九模块关键词、锁与监听调用点）。
2. **定向深读**：对穷举命中的关键函数**全文阅读**（约 1,170 行，占 bridge.js 的 9.4%）。
3. **调用链双向追踪**：MCP 工具 → bridge 端点 → A 层函数 → 文件写入，逐段核对。
4. 未做逐行全量阅读的部分，一律在 §1.3 列出。

## 1.2 已检索符号（穷举结果）

**A 层 11 个导出函数的全部调用点（bridge.js）**

| 函数 | bridge.js 调用行 | 所在端点 |
|---|---|---|
| `appendVectorMemory` | 6658 | `/api/socialV2/memory-append` |
| `searchVectorMemory` | 6872 | `/api/socialV2/memory-vector` |
| `listVectorMemory` | 6720 | `/api/memory/admin` op=list |
| `removeVectorMemoryExact` | 6731 | `/api/memory/admin` op=remove |
| `removeNAllVectorMemory` | 6964 | `/api/socialV2/memory-remove` |
| `removeAllVectorMemory` | 7076 | `/api/socialV2/memory-clear`（无 category） |
| `editVectorMemory` | 6739 | `/api/memory/admin` op=edit |
| `approveShare` / `rejectShare` | 6751 / 6752 | `/api/memory/admin` op=approve_share / reject_share |
| `requestShare` | 6896 | `/api/socialV2/memory-share-request` |
| `countVectorMemory` | **无调用点** | 死代码 |

**记忆相关端点（9 个，全部检查）**

`memory-append`(6628) · `/api/memory/admin`(6696) · `memory-update`(6765) · `memory` GET(6832) · `memory-vector`(6862) · `memory-share-request`(6882) · `memory-remove`(6903) · `memory-clear`(7014) · MCP 侧 5 工具(922/953/980/1013/1037)

## 1.3 覆盖率表（可审计）

| 文件 | 总行数 | 全文阅读 | 定向深读 | 符号穷举 | 覆盖率 |
|---|---|---|---|---|---|
| `src/memory-vector.js` | 380 | **380 (100%)** | — | ✅ | 全量 |
| `src/bridge.js` | 12,482 | — | **≈1,170 (9.4%)** | **✅ 全部 12,482 行** | 定向 |
| `src/mcp-snowluma-safe.js` | 1,465 | — | 219 (15%)：100-175、915-1057 | ✅ | 定向 |
| `src/qq-model-view.js` | 134 | **134 (100%)** | — | ✅ | 全量 |
| `src/mcp-diary.js` | 113 | **113 (100%)** | — | ✅ | 全量 |
| `src/mcp-host-server.js` | 93 | **93 (100%)** | — | ✅ | 全量 |
| `src/token-ledger.js` | 958 | — | — | ✅（无 memory 耦合） | 符号级 |
| `src/slang-learner.js` | 382 | — | — | ✅（无 memory 耦合） | 符号级 |
| `src/sticker-lib.js` | 302 | — | — | ✅（仅注释提及 note 通道） | 符号级 |
| `src/self-test.js` | 118 | — | — | ✅（0 处 memory） | 符号级 |
| `package.json` | 72 | **72 (100%)** | — | ✅ | 全量 |
| `config.example.json` | 279 | **279 (100%)** | — | ✅ | 全量 |

**bridge.js 已深读区间**：155-205（`renameWithRetry`/`atomicWriteJson`）· 1050-1165（锁）· 1366-1382（`canonicalV2Key`）· 3100-3180（`v2ToolEnabled`/`agentTokenOk`/`isLiveAgentToken`）· 3255-3405（全局鉴权中间件）· **6620-7160（全部记忆端点）** · 7505-7545（`server.listen`/EADDRINUSE）· 7554-7575（`redactKnownTokensOnly`）· 8764-8800（`getSocialV2State`）· 8960-9020（`saveSocialV2State*Strict`）· 9170-9280（`formatMemoryV2`/`appendMemoryV2`）· 12440-12482（启动/退出）

**未逐行阅读的区域**（声明，非遗漏）：
- bridge.js 中约 11,300 行与 Memory 无关的逻辑（消息收发、媒体、表情、黑话、唤醒调度、token 账本、控制台 UI 端点等）
- `mcp-snowluma-safe.js` 除 `agentApi`/`agentHeaders`/5 个记忆工具外的约 1,246 行

**交叉引用核对（用户要求项）**：
- `mcp-diary.js`：仅写 `<root>/diary/YYYY-MM-DD.md`，与 Memory 系统**零耦合** ✅
- `mcp-host-server.js`：仅探活 SnowLuma，与 Memory **零耦合** ✅
- `qq-model-view.js`：纯序列化（`serializeModelData`），被 MCP 用于渲染记忆返回，不写任何状态 ✅
- 全仓确认：`memory-vectors.jsonl` 的写入者**只有** `memory-vector.js`（`appendFileSync:160`、`atomicRewrite:378`），bridge.js 仅 `copyFileSync` 拷出备份（6716）✅

---

# 第 2 部分 · 对历史报告的逐条复核

## 2.1 结论总表

| # | 历史结论 | 我的复核 | 处置 |
|---|---|---|---|
| 1 | P0-1「embed 期间删除 → 记录复活」（`m-v.js:361-374`） | **部分成立，机制描述错误** | **修正**（见 F-01） |
| 2 | P0-2「审计在业务后 + 无 try/catch」 | **成立**，但等级过高；且遗漏 `memory-remove` 无审计 | **降级 P1 + 补漏** |
| 3 | P0-3「无 WAL → 不可恢复」 | 成立（设计缺口），但端点已显式回报 partial | **降级 P1** |
| 4 | 「V2.4.1 九模块 0 次出现」 | **成立**（实测 0/0/0/0/0/0/0/0/0） | 确认 |
| 5 | 撤回「分享功能不可用」 | **成立**，撤回正确 | 确认 |
| 6 | 「atomicRewrite 固定 tmp 名」 | **成立**（`m-v.js:377`） | 确认 |
| 7 | 「bridge.lock 启动窗口 P1」 | **成立**，但窗口描述低估 | **修正加强** |
| 8 | 「clear 完全无审计」 | **成立**，且 `memory-remove` 同样无审计 | 补漏 |
| 9 | 「append 无去重 / 无 opId」 | **成立** | 确认 |
| 10 | 「无恢复函数」 | **成立**（全仓无 restore） | 确认 |
| 11 | 「`loadRows` 坏行静默跳过 P1」 | 成立，但等级偏高；且**写操作会放大**（用过滤后快照整体重写） | 修正 P2 + 补充 |
| 12 | 「embed 失败即丢记录」 | **成立** | 确认 |
| 13 | 「consolePortInUse 保守返回 true → 可能永久无法启动」 | **条件性成立**（仅 PID 存活/EPERM 分支） | 修正为条件性 |
| 14 | 未提及项 | 发现 **6 个新问题**（F-02/F-03/F-04/F-09/F-12b/F-20） | 新增 |

## 2.2 ★ 对 P0-1 的关键修正（重要）

历史报告 §9.5 的时序表断言：edit 的 `rows` 在 remove 之后「仍是 [X,Y,Z]」，从而得出「Y 复活」。

**该推断在 `removeVectorMemoryExact` 路径下不成立。** 依据：

```js
// src/memory-vector.js:44-58  loadRows —— 返回的是 cache.rows 的【同一个数组引用】
function loadRows() {
  const mtime = fs.existsSync(VEC_FILE) ? fs.statSync(VEC_FILE).mtimeMs : 0;
  if (mtime === cache.mtime) return cache.rows;      // ← 返回引用，非副本
  ...
  cache = { mtime, rows };
  return rows;                                       // ← rows 即 cache.rows
}
```

```js
// src/memory-vector.js:311-319  removeVectorMemoryExact —— 就地 splice，改动的是同一个数组
export function removeVectorMemoryExact(key, text) {
  const rows = loadRows();                 // ← 与 edit 持有的 rows 是同一个对象
  const idx = rows.findIndex(...);
  const [removed] = rows.splice(idx, 1);   // ★ 就地修改，edit 的 rows 同步变化
  ...
}
```

因此 `edit + removeVectorMemoryExact` 的实际后果**不是复活**，而是（详见 §4 时序表 R2）：

- `idx < j`：edit 正常，Y 保持删除 —— **无害**
- `idx == j`：edit 把**下一条记录**的 text 覆盖成新文本 —— **静默改错记录**
- `idx > j`：`rows[idx]` 已前移，edit 改到**另一条记录** —— **静默改错记录**
- `idx` 越界：`rows[idx]` 为 `undefined` → `TypeError` → HTTP 500

**但复活路径确实存在**，只是经由**严格读取**的两个函数：

```js
// src/memory-vector.js:68-83  loadRowsStrict —— 每次新建数组，不复用 cache.rows
function loadRowsStrict() {
  ...
  const rows = [];                         // ★ 全新数组
  ...
  cache = { mtime, rows };
  return rows;                             // ← 与 edit 持有的 A 不是同一个对象
}
```

`removeNAllVectorMemory`(324) 与 `removeAllVectorMemory`(349) 使用 `loadRowsStrict` → 它们构造 `kept` 新数组重写文件，edit 持有的旧数组 **A 仍含 Y** → edit 恢复执行后 `atomicRewrite(A)` → **Y 复活** ✅

> **修正结论**：P0-1 的「复活」结论**成立但路径收窄**（仅限 `memory-clear` 全清 / `memory-remove`）；原报告对该路径的时序推演有误，且**漏掉了更危险的「改错记录」分支**。两者同根因：`editVectorMemory` 用 `await` 之前的快照做全量覆盖。

## 2.3 其他需修正的细节

| 项 | 历史报告 | 实际 |
|---|---|---|
| `findMemoryRow` 匹配级数 | 称「三级匹配」 | 代码注释写三级，**实为四级**（238 全等 / 240 双向包含 / 242-246 去前缀包含 / 249-256 归一化包含） |
| `mcp-snowluma-safe.js` agentApi 超时行号 | 称 `:155` | 实际为 `:155`（`const timeoutMs = init.timeoutMs || 15000;`）—— 此项**准确** ✅ |
| lock→listen 之间「多为函数定义，执行几乎无耗时」 | 低估 | 实际 main 主体含 `loadState()`(1391)、`loadSocialV2State()`(9275)、`loadInitialPrompts()`(9277)、`new SnowLumaWebSocketClient`(12222)、`bot.connect()`(12425) 等真实初始化与网络操作 → **窗口远大于描述** |
| P1-10 `consolePortInUse` 永久无法启动 | 表述为普遍 | 仅在「PID 存活(1108)」或「EPERM(1118)」两个分支才调用；PID 不存在走 ESRCH(1110) 直接判过期 → **条件性风险** |
| `01-BASELINE` 分卷数 | 8 个 | 5 个 |

---

# 第 3 部分 · 源码证据索引

## 3.1 A 层（`src/memory-vector.js`）

| 编号 | 问题 | 行号 | 关键代码 |
|---|---|---|---|
| E-01 | `editVectorMemory` 用 await 前快照全量覆盖 | 361-374 | `const rows = loadRows();` (362) → `const vec = await embed(clean);` (367) → `atomicRewrite(rows)` (372) |
| E-02 | `loadRows` 返回 cache 数组**引用** | 44-58 | `if (mtime === cache.mtime) return cache.rows;` (47) |
| E-03 | `loadRowsStrict` 每次新建数组 | 68-83 | `const rows = [];` (75) |
| E-04 | `atomicRewrite` 固定 tmp 名 | 376-380 | `const tmp = VEC_FILE + '.tmp';` (377) |
| E-05 | 无 fsync / 无 renameWithRetry / 失败留 tmp | 376-380 | `fs.writeFileSync(tmp, ...)` (378) + `fs.renameSync(tmp, VEC_FILE)` (379)，无 try/finally |
| E-06 | 坏行静默跳过 | 51-54 | `try { rows.push(JSON.parse(line)); } catch {}` (53) |
| E-07 | 读异常一律返回 `[]` | 57 | `} catch { return []; }` |
| E-08 | append 无去重 | 149-163 | 仅 `infoOK`(152) 检查，无查重 |
| E-09 | embed 失败即不落盘 | 153 | `const vec = await embed(clean);` 在 `appendFileSync`(160) 之前 |
| E-10 | embed 无重试、超时 30s | 86-108 | `timeout: 30000` (92) |
| E-11 | 向量维度无校验 | 4 / 110-120 | 注释称 1024 维；`cosSim` 仅判 `a.length !== b.length` 返回 0 |
| E-12 | 缓存仅靠 mtime | 42 / 46-47 | `let cache = { mtime: 0, rows: [] }` |
| E-13 | `findMemoryRow` 四级模糊匹配 | 235-258 | 240 `r.text.includes(t) \|\| t.includes(r.text)`；254 归一化后 `includes` 双向 |
| E-14 | `approveShare` 不改 key | 277-291 | 284-286 只改 status/visibility/reviewed_at |
| E-15 | 分享状态仅单字段 | 157 | `share_review_status: 'not_requested'` |
| E-16 | `dailyBackup` 仅在 append 触发、同日只一次、`catch{}` 静默 | 131-147 / 154 | `if (!fs.existsSync(dest)) fs.copyFileSync(...)` (138)；`} catch {}` (146) |
| E-17 | 无 tombstone / 无恢复函数 / 备份无 schema 版本 | 全文 | 0 命中 |
| E-18 | append 前不检查尾行 `\n` | 160 | `fs.appendFileSync(VEC_FILE, JSON.stringify(row) + '\n')` |

## 3.2 B 层与端点（`src/bridge.js`）

| 编号 | 问题 | 行号 | 关键代码 |
|---|---|---|---|
| E-19 | A 层写入用**未脱敏、未截断**的 content | 6658 | `await appendVectorMemory(key, \`${category}: ${content}\`)` |
| E-20 | B 层写入用**脱敏 + 截断 200** 的 text | 9207 / 9228 / 9244 | `const text = redactKnownTokensOnly(String(content ?? '')).trim();` / `truncateText(text, 200)` / `text.slice(0, 200)` |
| E-21 | `aRemoved` 计算后**从未上报** | 6962 / 6964 / 7011 | `removed: bRemoved`（7011） |
| E-22 | B 层自动淘汰，A 层不淘汰 | 9210-9215 / 9234 / 9250 / 9264 | 过期与 24h 清理；`activeTopics.length > 20` splice；`memberImpressions > 50` delete |
| E-23 | A 层用原始 key，B 层用 canonical key | 6630 / 6658 vs 8765-8772 | `getSocialV2State` 内 `key = canonical`，但 `appendVectorMemory(key, ...)` 用的是未规范化的 `key` |
| E-24 | append：A 失败 B 已落盘 → 分叉 | 6655-6662 / 6667-6691 | A 失败仅置 `aSynced=false`，B 仍 `saveSocialV2StateStrict()` |
| E-25 | append：B 失败回滚 B，A 已写入 → 分叉 | 6667-6681 | 只回滚 B 快照，**不回滚 A** |
| E-26 | remove：A 已删，B 落盘失败 → B 回滚 → 分叉 | 6964 / 6989-7010 | A 删除在前且不可逆 |
| E-27 | memberImpression 删 B 不删 A（设计声明） | 6932-6956 | `aSynced: 'n/a'`，A 层残留 |
| E-28 | 分类 clear 只清 B（设计声明） | 7024-7068 | `aSynced: 'n/a'`，A 层保留 |
| E-29 | 审计在业务之后、无 try/catch | 6731→6733 / 6739→6740 / 6751→6754 | `fs.appendFileSync(...)` 裸调用，外层 catch 6760 → 500 |
| E-30 | `clear` / `memory-remove` **完全无审计** | 6903-7122 | 全文无 `memory-admin.log` 写入（写入点仅 6733/6740/6754） |
| E-31 | 审计无操作者 / 无结果 / 无 opId | 6733-6734 等 | 仅 `ISO时间 OP key \| text.slice(0,80)` |
| E-32 | 审计含记忆正文前 80 字 | 6734 / 6741 / 6755 | `${removed.text.slice(0, 80)}` |
| E-33 | 备份在 remove/edit 前，同名覆盖 | 6715-6717 | `copyFileSync(VEC_FILE_PATH, ...'memory-vectors-admin-backup.jsonl')` |
| E-34 | 未知 op 报错文案过时 | 6759 | `'未知 op，只支持 list/remove/edit'`（实际还支持 approve_share/reject_share） |
| E-35 | 条件式鉴权（96 处） | 6637-6639 等 | `if (req.headers['x-agent-token'] && !agentTokenOk(...))` |
| E-36 | 全局兜底：console token + timingSafeEqual | 3263-3282 | `crypto.timingSafeEqual(a, b)` (3272) |
| E-37 | 全局兜底：x-agent-call 门禁 | 3302-3318 | `/api/socialV2/*` 无令牌智能体调用一律 403 |
| E-38 | 全局兜底：Host 白名单抗 DNS rebinding | 3254-3261 | 仅允许 `127.0.0.1/localhost/[::1]:<实际端口>` |
| E-39 | 全局兜底：CSRF | 3322-3337 | 非 GET 必须 `application/json` + Origin 同源 |
| E-40 | 管理端拒绝 agent token（防越权） | 6707-6710 | `if (req.headers['x-agent-token'] !== undefined) → 403` |
| E-41 | `acquireLock` 与 `listen` 间隔整个 main 初始化 | 1390 / 7531 / 12442 | 间隔含 `loadState`、`bot.connect()` 等真实初始化 |
| E-42 | `consolePortInUse` 失败开放 | 1074-1077 | `catch { return true; }` |
| E-43 | `releaseLock` 仅按 PID 匹配删除 | 1143-1149 | `Number(...) === process.pid` |
| E-44 | 退出/异常均释放锁 | 12447-12477 | SIGINT/SIGTERM/uncaughtException/exit |
| E-45 | B 层原子写**质量良好** | 182-193 | 随机 tmp + `renameWithRetry` + `mode 0o600` + 失败清理 |
| E-46 | `v2ToolEnabled` 默认放行 | 3116-3118 | `cfg.socialV2?.tools?.[flag] !== false` |
| E-47 | `canonicalV2Key` 不改前缀 | 1370-1377 | `return \`${m[1]}:${id}\`` |
| E-48 | 私聊记忆拒绝分享 | 6892-6894 | `if (key.startsWith('private:')) → 400` |

## 3.3 MCP 层（`src/mcp-snowluma-safe.js`）

| 编号 | 问题 | 行号 | 关键代码 |
|---|---|---|---|
| E-49 | agentApi 超时 15s（< A 层 embed 30s） | 155 | `const timeoutMs = init.timeoutMs || 15000;` |
| E-50 | MCP 不再直连 A 层（统一经 bridge） | 21 / 944-945 | 注释 + `// A 层向量写入已由 bridge 的 memory-append 端点统一完成` |
| E-51 | 返回 `ok:false` 时抛错（不静默） | 165-168 | `if (body.ok === false) throw new Error(...)` |
| E-52 | `x-agent-call: 1` 恒发 | ~194 | `headers['x-agent-call'] = '1';` |
| E-53 | 无关键词降级兜底 | 全文 | `m-v.js:10-11` 声明的「embed 失败退关键词匹配」契约**未实现**（0 命中） |
| E-54 | `qq_memory_query` 先 B 后 A，A 失败整段报错 | 993 / 998 | B 层结果 `out` 被丢弃 |

---

# 第 4 部分 · 异步竞态时序表

> 前置事实：`editVectorMemory`(m-v.js:361) 与 `appendVectorMemory`(m-v.js:149) 是**唯一含 `await` 的 A 层写函数**；其余 5 个写函数（requestShare/approveShare/rejectShare/removeVectorMemoryExact/removeNAllVectorMemory/removeAllVectorMemory）为**同步**，Node 单线程下彼此不交错。
> 所有时序均基于 `loadRows` 返回**同一数组引用**（E-02）与 `loadRowsStrict` 返回**新数组**（E-03）的静态语义推演；实际触发概率 **NOT_VERIFIABLE**（禁止运行时验证）。

## R1 · edit ∥ removeAllVectorMemory（`/api/socialV2/memory-clear` 全清）→ **已删记忆复活**

| 时刻 | 控制台 edit（X） | 智能体 clear | 磁盘 A 层 |
|---|---|---|---|
| t0 | `loadRows()` → 数组 **A**=[X,Y,Z]，`cache.rows = A` | — | [X,Y,Z] |
| t1 | `await embed(clean)` ← **让出事件循环** | — | [X,Y,Z] |
| t2 | — | `loadRowsStrict()` → **新数组 B**=[X,Y,Z] | [X,Y,Z] |
| t3 | — | `kept = B.filter(key!==key)` = []；`atomicRewrite([])` | **[]** |
| t4 | embed 返回；`rows[0].text = clean`（rows 仍是 **A**） | — | [] |
| t5 | `atomicRewrite(A)` ← 用旧快照覆盖 | — | **[X',Y,Z] ❌ 全部复活** |

- 函数：`editVectorMemory` m-v.js:362/367/372；`removeAllVectorMemory` m-v.js:350-355
- 端点：`/api/memory/admin` op=edit (6739) ∥ `/api/socialV2/memory-clear` (7076)
- **后果**：**数据复活** —— 已删除/已清空的记忆重新出现且可被检索命中。若删除动机是隐私，则为隐私事故。
- **是否造成数据丢失**：否（是"多出数据"）。**是否错误覆盖**：是。

## R2 · edit ∥ removeVectorMemoryExact（`/api/memory/admin` op=remove）→ **改错记录 / 崩溃**（原报告误判为"复活"）

| 时刻 | edit（目标 idx） | remove（删除 j） | 说明 |
|---|---|---|---|
| t0 | `loadRows()` → **A**=[X,Y,Z] | — | `cache.rows = A` |
| t1 | `await embed` 让出 | — | — |
| t2 | — | `loadRows()` → 命中缓存，返回**同一个 A** | E-02 |
| t3 | — | `A.splice(j,1)` ← **就地修改 A** | A 变为 [X,Z] |
| t4 | — | `atomicRewrite(A)` | 磁盘 [X,Z] |
| t5 | `rows[idx].text = clean` | — | `rows === A`，已变短 |

| idx 与 j 的关系 | 结果 | 严重性 |
|---|---|---|
| `idx < j` | 命中原记录，edit 正确，Y 保持删除 | ✅ 无害 |
| `idx == j` | 正在编辑的记录已被删；`rows[idx]` 变成**下一条** → **新文本写入错误的记忆** | ❌ 静默数据损坏 |
| `idx > j` | 元素前移 → **改到另一条记忆** | ❌ 静默数据损坏 |
| `idx >= A.length` | `rows[idx]` 为 `undefined` → `TypeError` → catch(6760) → HTTP 500 | ⚠️ 报错但无损坏 |

- 依据：`memory-vector.js:47`（返回引用）、`:315`（`rows.splice`）、`:368`（`rows[idx].text = clean`）
- **处置**：历史报告 §9.5 时序表 t3「rows 仍是 [X,Y,Z]」**有误**，据此得出的"Y 复活"结论在此路径**不成立**，特此撤回该分支的表述。

## R3 · edit ∥ appendVectorMemory → **新写入的记忆丢失**

| 时刻 | edit | append(`/api/socialV2/memory-append`) | 磁盘 A 层 |
|---|---|---|---|
| t0 | `loadRows()` → A=[X,Y,Z] | — | [X,Y,Z] |
| t1 | `await embed` 让出 | — | [X,Y,Z] |
| t2 | — | `await embed` → `dailyBackup()` → `appendFileSync` 追加 W | [X,Y,Z,W] |
| t3 | — | `cache.mtime = 0` | [X,Y,Z,W] |
| t4 | `atomicRewrite(A)` ← 旧快照 | — | **[X,Y',Z] ❌ W 丢失** |

- 依据：`memory-vector.js:362/367/372` vs `:153/160`
- **后果**：**数据丢失**（模型刚写入的记忆消失，且调用方已收到 `ok:true`）
- **触发难度**：低。append 是最高频的 A 层写入，embed 窗口 ≤30s。

## R4 · edit ∥ edit → **丢失更新**

| 时刻 | edit-1 | edit-2 | 磁盘 |
|---|---|---|---|
| t0 | `loadRows()` → A | — | [X,Y] |
| t1 | `await embed` | `loadRows()` → A（同一引用） | [X,Y] |
| t2 | — | `await embed` | [X,Y] |
| t3 | 先返回 → `A[0].text=T1`；`atomicRewrite(A)` | — | [X(T1),Y] |
| t4 | — | 后返回 → `A[1].text=T2`；`atomicRewrite(A)` | [X(T1),Y(T2)] ✅ 恰好都生效 |

> 注意：因 `loadRows` 返回同一引用，**两次 edit 反而都可能生效**（与历史报告"后写覆盖先写"不同）。但若两次 `loadRows` 之间 `cache.mtime` 被置 0（例如中间发生过 append），则 edit-2 拿到新数组 B，后写的 `atomicRewrite(B)` 会**整体覆盖掉 edit-1 的修改** → **丢失更新** ❌。两种情况均可能，取决于缓存是否失效。

## R5 · remove ∥ clear（同为同步）→ **安全**

两个同步函数在 Node 单线程内不会交错，各自 `loadRowsStrict()` 重读磁盘，串行执行 → 无竞态。✅

## R6 · approveShare/rejectShare/requestShare ∥ edit（同步）→ **基本安全**

同步函数复用同一数组 A（E-02），edit 之后的 `atomicRewrite(A)` 会顺带保留分享状态变更 → 无丢失。✅
但若中间发生 `cache.mtime = 0` 导致新数组，则可能回退分享状态。**条件性**。

## R7 · append ∥ clear → **清空后残留 1 条**

| 时刻 | clear | append | 磁盘 |
|---|---|---|---|
| t0 | — | `await embed` 让出 | [X,Y] |
| t1 | `loadRowsStrict()` → B=[X,Y]；`kept=[]`；`atomicRewrite([])` | — | **[]** |
| t2 | — | `appendFileSync` 追加 W | **[W] ❌ 清空未生效** |

## R8 · 跨进程（第二写入者）

全仓穷举确认：`memory-vectors.jsonl` 的写入者仅 `memory-vector.js`；`bridge.js:6716` 为**拷出**（只读源）。`bridge.lock` 只保护 bridge 进程自身 → **当前不存在第二写入者**。但 `atomicRewrite` 固定 tmp 名（E-04）意味着**一旦出现第二写入者，冲突概率接近 100%**。

## 竞态汇总

| 编号 | 组合 | 数据丢失 | 记忆复活 | 错误覆盖 | 等级 |
|---|---|---|---|---|---|
| R1 | edit ∥ clear(全) / removeNAll | 否 | **是** | 是 | **P0** |
| R2 | edit ∥ remove(精确) | 否 | 否 | **是**（改错记录）/ TypeError | **P0** |
| R3 | edit ∥ append | **是** | 否 | 是 | **P0** |
| R4 | edit ∥ edit | 条件性 | 否 | 条件性 | P1 |
| R5 | remove ∥ remove/clear | 否 | 否 | 否 | ✅ |
| R6 | share ∥ edit | 条件性 | 否 | 条件性 | P2 |
| R7 | append ∥ clear | 否 | **是（残留）** | 否 | P1 |
| R8 | 跨进程 | 当前无第二写入者 | — | — | P1（潜在） |

---

# 第 5 部分 · V2.4.1 设计 vs 实现差距表

穷举命令与结果（对全部 12 个源文件，含 bridge.js 三段）：

```
memory-wal      => 0    tombstone       => 0    ShareRequest   => 0
factKey         => 0    revision        => 0    effects        => 0
_crc            => 0    embeddingStatus => 0    writer.lock    => 0
（附加）opId     => 0    idempot*        => 0    schemaVersion  => 0    memoryId => 0
```

| # | V2.4.1 设计模块 | 代码中出现 | 状态 | 现有 V1 替代 | 差距后果 |
|---|---|---|---|---|---|
| 1 | `memory-wal`（WAL） | 0 | `NOT_IMPLEMENTED` | 端点用 `aSynced/bSynced/partial` 显式回报 | 部分成功可发现但**不可自动修复**，需人工对账 |
| 2 | `tombstone`（墓碑） | 0 | `NOT_IMPLEMENTED` | 物理删除（`splice`/`filter`） | 删除无痕迹；备份恢复必然复活；R1 复活无法拦截 |
| 3 | `ShareRequest`（独立对象） | 0 | `NOT_IMPLEMENTED` | 记录在行内字段 `share_review_status` | 无法表达「群 A 批准、群 B 拒绝」；无 targetGroup 维度 |
| 4 | `factKey` | 0 | `NOT_IMPLEMENTED` | 靠 `text` 全等匹配 | 改 text 即与 B 层失联（E-21） |
| 5 | `revision`（乐观锁） | 0 | `NOT_IMPLEMENTED` | 无 | R1-R4 全部竞态无拦截手段 |
| 6 | `effects`（副作用记录） | 0 | `NOT_IMPLEMENTED` | 无 | A/B 双写无事务边界 |
| 7 | `_crc`（递归规范化校验） | 0 | `NOT_IMPLEMENTED` | `loadRows` 坏行静默 skip | 坏行在下次写操作时被永久丢弃且无告警 |
| 8 | `embeddingStatus`（状态机） | 0 | `NOT_IMPLEMENTED` | embed 失败即抛，记录不入 A 层 | 正文丢失；无重试队列、无降级（E-53） |
| 9 | `writer.lock` | 0 | `NOT_IMPLEMENTED` | 仅 `bridge.lock`（进程级，非写入级） | 进程内 async 写无互斥 |

**结论：历史报告「V2.4.1 九模块全部不存在」成立 ✅，实测确认为 0。**

---

# 第 6 部分 · 问题清单与严重程度

> 定级口径：**P0** = 已证实会造成数据丢失 / 数据损坏 / 记忆复活 / 越权访问；**P1** = 功能正确性或一致性缺陷，可导致分叉、残留或不可审计；**P2** = 健壮性/可观测性/合规改进；**P3** = 轻微或条件苛刻。

## P0

| # | 问题 | 证据 | 触发 | 后果 |
|---|---|---|---|---|
| **F-01** | `editVectorMemory` 基于 `await` 前快照做全量覆盖 | `m-v.js:362/367/372` + `:47`/`:315`/`:350` | 控制台 edit（embed ≤30s）期间发生任一其他 A 层写 | R1 **记忆复活**；R2 **改错记录/TypeError**；R3 **新记忆丢失** |

## P1

| # | 问题 | 证据 | 后果 |
|---|---|---|---|
| **F-02** | A/B 文本口径不一致：A 层不脱敏、不截断；B 层 `redactKnownTokensOnly` + 200 字截断 | `bridge.js:6658` vs `:9207/:9228/:9244` | ① content >200 字时删除匹配失败（404 或 A 残留）；② content 含会话令牌时 **A 层明文入库**（B 层已脱敏） |
| **F-03** | `aRemoved` 计算后从未上报 | `bridge.js:6962/6964/7011`（`removed: bRemoved`） | A 层残留被静默吞掉，调用方以为全删成功 |
| **F-04** | B 层自动淘汰、A 层不淘汰 | `bridge.js:9210-9215/9234/9250/9264` | 过期想法/超 20 条话题/超 50 印象在 B 消失而 A 永久残留 → 检索仍能召回已"过期"记忆 |
| **F-05** | 审计缺陷（原 P0-2，**降级**） | `bridge.js:6731→6733/6739→6740/6751→6754`（业务后写、无 try/catch）；`6903-7122`（clear/remove **零审计**）；无操作者标识 | 业务成功但返回 500；智能体侧删除行为完全无审计轨迹；无法回答"谁删的" |
| **F-06** | `atomicRewrite` 固定 tmp 名 | `m-v.js:377` `const tmp = VEC_FILE + '.tmp'` | 当前无第二写入者（已穷举确认），但一旦出现即必然冲突；对比 B 层 `atomicWriteJson`(182-193) 已用随机名 |
| **F-07** | bridge.lock 启动窗口（**加强**） | `bridge.js:1390`（acquireLock）→ `:7531`（listen），间隔为**整个 main 初始化**（含 `loadSocialV2State`:9275、`bot.connect`:12425）；`:1108` stale 判定；`:12477` exit 释放 | A 持锁但未监听期间，B 判 stale → 删锁接管 → B 端口冲突 exit(2) → 释放自己写的锁 → **A 运行时无锁文件**，第三方可再获锁 |
| **F-08** | embed 失败即丢记录，且无重试/队列/降级 | `m-v.js:153`；`mcp-snowluma-safe.js` 0 处关键词兜底（E-53） | A 层正文永久丢失，B 层已落盘 → 分叉；与 `m-v.js:10-11` 声明的降级契约不符 |
| **F-09** | `findMemoryRow` 四级模糊匹配 + "首个命中" | `m-v.js:240`（双向包含）、`:254`（归一化双向包含） | 用片段申请/审批可能命中**另一条**记忆 → 错误的记忆被共享到群（隐私）；`removeVectorMemoryExact` 反而是精确匹配，两条路径口径不一致 |
| **F-10** | append 无去重、无 opId/幂等键 | `m-v.js:149-163`；`opId`/`idempot` 全仓 0 命中；`mcp-snowluma-safe.js:155` 超时 15s < embed 30s | 模型重试/超时重试 → A 层重复条目累积（B 层有文本去重，A 层无） |
| **F-11** | 无 tombstone、无恢复函数、备份无 schema 版本 | 全仓 `restore`/`tombstone`/`schemaVersion` 0 命中 | 人工用历史备份覆盖 → 已删记忆全部复活，代码无法察觉；当前空库 + 101 条历史备份无任何保护 |

## P2

| # | 问题 | 证据 |
|---|---|---|
| F-12 | `loadRows` 坏行静默跳过；写操作会用"过滤后快照"整体重写 → 坏行**永久丢失**且无告警 | `m-v.js:53` + `:378` |
| F-13 | `loadRows` 任何异常返回 `[]` → 检索/列表静默为空 | `m-v.js:57` |
| F-14 | 向量维度/模型无校验，换模型后 `cosSim` 恒 0 → 检索静默空结果 | `m-v.js:110-120`；`:4` 仅注释 1024 |
| F-15 | A 层 rename 无 fsync、无 `renameWithRetry`（bridge 侧有）、失败留 `.tmp` | `m-v.js:378-379` vs `bridge.js:165-193` |
| F-16 | 缓存仅靠 `mtime`，同毫秒写入可能读到旧值 | `m-v.js:46-47` |
| F-17 | 审计含记忆正文前 80 字；无轮转、无大小上限 | `bridge.js:6734/6741/6755` |
| F-18 | 未知 op 报错文案过时（"只支持 list/remove/edit"） | `bridge.js:6759` |
| F-19 | `countVectorMemory` 导出但全仓无调用（死代码） | `m-v.js:202` |
| F-20 | 条件式鉴权 96 处（`if (x-agent-token && ...)`），依赖全局兜底才安全 | `bridge.js:6637-6639` 等；兜底见 E-36~E-40 |
| F-21 | 文档与产物不一致（脱敏 0 占位符 / 分卷数 8 vs 5 / 匹配级数 三 vs 四） | `05` §2.2、`01-BASELINE` 末、`m-v.js:235` |
| F-22 | `qq_memory_query`：A 层检索失败会丢弃已取到的 B 层结果 | `mcp-snowluma-safe.js:993/998` |

## P3

| # | 问题 | 证据 |
|---|---|---|
| F-23 | A 层用原始 key，B 层用 `canonicalV2Key` → 前导零可导致 key 分裂 | `bridge.js:6658` vs `:8765-8772` |
| F-24 | `dailyBackup` 用非原子 `copyFileSync`；仅在 append 触发；同日只备份第一次；`catch {}` 静默 | `m-v.js:138/146/154` |
| F-25 | `MEM_CONFIG` 在模块加载时读取一次，运行期改 `config.json` 不生效 | `m-v.js:28-32` |
| F-26 | `admin-backup.jsonl` 同名覆盖，仅保留最近一次操作前状态 | `bridge.js:6716` |

## 正向确认（无缺陷 / 历史报告疑虑已排除）

| 项 | 结论 | 证据 |
|---|---|---|
| 分享权限链（群内记忆） | **可用**。群内产生 → key=`group:X` → 申请 pending → 审批 `visibility=group_shared` + `approved` → 群检索 `key===group:X` 命中 | `m-v.js:172/177/284-286`；`bridge.js:6896/6751`；`canonicalV2Key` 不改前缀 `:1370-1377` |
| 私聊→群分享 | 设计边界，智能体侧 400 拒绝 | `bridge.js:6892-6894` |
| 控制台令牌比较 | **定长比较** `crypto.timingSafeEqual` | `bridge.js:3272` |
| 会话令牌比较 | **定长比较** | `bridge.js:3140-3145` |
| 智能体 confused-deputy | 已由 `x-agent-call` 集中门禁兜底，`/api/socialV2/*` 无令牌智能体调用 403 | `bridge.js:3302-3318`；MCP 恒发 `x-agent-call:1` |
| 管理端越权 | `/api/memory/admin` 显式拒绝任何携带 `x-agent-token` 的请求 | `bridge.js:6707-6710` |
| DNS rebinding / CSRF | Host 白名单 + JSON-only + Origin 同源 | `bridge.js:3254-3261/3322-3337` |
| 空令牌启动 | **失败关闭**（抛错拒绝启动） | `bridge.js:3132` |
| B 层原子写 | 随机 tmp + `renameWithRetry` + `0o600` + 失败清理（**优于 A 层**） | `bridge.js:182-193` |
| 无"文件被清空"路径 | 所有 `atomicRewrite` 均在成功匹配后调用；`loadRows` 异常返回 `[]` 时 `findIndex<0` 直接返回 null，不会重写 | `m-v.js:57/313-319` |
| 跨文件耦合 | `mcp-diary.js` / `mcp-host-server.js` / `qq-model-view.js` 与 Memory **零耦合** | 全文阅读确认 |
| 第二写入者 | 不存在（全仓穷举） | §4 R8 |

---

# 第 7 部分 · 关键调用链

## 7.1 写入链（A/B 双写）

```
模型 → MCP qq_memory_append (mcp-snowluma-safe.js:922)
     → agentApi POST /api/socialV2/memory-append   [:939]
       ├─ 全局中间件：Host(3254) → consoleToken(3263) → x-agent-call(3302) → CSRF(3322)
       ├─ 条件式鉴权 agentTokenOk / v2SessionAllowed / v2ToolEnabled  [6637-6639]
       ├─ ① getSocialV2State(key)  → canonical key            [:6640 → :8764]
       ├─ ② appendMemoryV2(st, ...) 改 B 内存（脱敏+截断200+淘汰）  [:6653 → :9205]
       ├─ ③ appendVectorMemory(key, `${category}: ${content}`)      [:6658 → m-v.js:149]
       │      └─ await embed(30s) → dailyBackup → appendFileSync → cache.mtime=0
       └─ ④ saveSocialV2StateStrict() → atomicWriteJson(social-v2.json) [:6668 → :8976 → :182]
             └─ 失败 → 回滚 B 内存快照（**不回滚 A**）                [:6672-6681]
```

## 7.2 删除链

```
模型 → qq_memory_remove (mcp:1014) → /api/socialV2/memory-remove  [:6903]
       ├─ ① 计算 bRemoved（B 层 text 全等计数）                    [:6920-6926]
       ├─ ② memberImpression：只删 B，A 残留，返回 aSynced:'n/a'    [:6934-6956]
       ├─ ③ removeNAllVectorMemory(key, `${category}: ${content}`, bRemoved) [:6964 → m-v.js:324]
       │      └─ loadRowsStrict → filter → atomicRewrite
       ├─ ④ 删 B 内存 → saveSocialV2StateStrict → 失败回滚 B（A 已删不可回滚）[:6983-7010]
       └─ ⑤ 响应 removed: bRemoved（**aRemoved 未上报**）           [:7011]

控制台 → /api/memory/admin op=remove  [:6730]
       ├─ copyFileSync 备份                                        [:6716]
       ├─ removeVectorMemoryExact → loadRows（共享数组）+ splice     [:6731 → m-v.js:311]
       └─ fs.appendFileSync(memory-admin.log)  ← 无 try/catch       [:6733]

控制台 → /api/memory/admin op=edit  [:6738]
       └─ await editVectorMemory → loadRows → await embed → atomicRewrite(旧快照) [:6739 → m-v.js:361]
```

## 7.3 分享审批链

```
模型 → qq_memory_share_request (mcp:953) → /api/socialV2/memory-share-request [:6882]
       ├─ 无条件三重鉴权                                            [:6888-6890]
       ├─ private: 前缀 → 400 拒绝                                  [:6892]
       └─ requestShare → findMemoryRow（四级模糊）→ status=pending   [:6896 → m-v.js:261]

控制台 → /api/memory/admin op=approve_share  [:6745]
       ├─ 拒绝 x-agent-token                                        [:6707]
       └─ approveShare → findMemoryRow → status=approved + visibility=group_shared
                                        （★ 不改 key）              [:6751 → m-v.js:277]

群检索 → /api/socialV2/memory-vector  [:6862]
       └─ searchVectorMemory → filter(r.key===key && Array.isArray(r.vec))
                             → 非私聊：visibility==='group_shared' && status==='approved'
                                                                     [:6872 → m-v.js:171]
```

## 7.4 启动/锁链

```
main() [:1386]
 ├─ loadConfig / mkdirSync / hardenStateDirAcl      [:1387-1389]
 ├─ acquireLock()                                   [:1390 → :1080]
 ├─ loadState()                                     [:1391]
 ├─ … 大量初始化 …                                   [:1392-9274]
 ├─ loadSocialV2State() / loadInitialPrompts()       [:9275/9277]
 ├─ new SnowLumaWebSocketClient / bot.connect()      [:12222/12425]
 ├─ startDshWatch() / startSnowlumaWatch()           [:12440/12441]
 └─ startConsoleServer()                            [:12442 → :3120]
        └─ server.listen(port,'127.0.0.1')          [:7531]
             └─ on('error') EADDRINUSE → exit(2)     [:7535-7540]

退出：SIGINT(12447)/SIGTERM(12453)/uncaughtException(12471)/exit(12477) → releaseLock()
```

---

# 第 8 部分 · 修复优先级建议

| 顺序 | 目标 | 最小改法 | 依赖 |
|---|---|---|---|
| 1 | **F-01**（P0） | `editVectorMemory` 在 `await embed` 之后**重新 `loadRows()` 并按 key+oldText 重新定位**；定位失败则整体放弃并报错，禁用旧索引与旧数组 | 无 |
| 2 | **F-01** 根治 | 引入进程内写串行队列（Promise 链），所有 A 层写操作入队 | 无 |
| 3 | **F-05** | `appendFileSync` 包 try/catch（失败只告警不抛）；`memory-remove`/`memory-clear` 补审计；审计加令牌指纹与结果字段 | 无 |
| 4 | **F-02 / F-03** | A 层写入复用 B 层同一份 `redactKnownTokensOnly` + 同口径截断后的文本；`memory-remove` 响应改报 `aRemoved` 并在 `aRemoved < bRemoved` 时置 `partial:true` | 无 |
| 5 | **F-06 / F-15** | A 层 `atomicRewrite` 改用 `${VEC_FILE}.${pid}.${randomBytes(6)}.tmp` + 复用式 `renameWithRetry` + try/finally 清 tmp | 无 |
| 6 | **F-07** | 先 `server.listen` 成功再 `acquireLock()`；或 stale 判定加"锁年龄/启动完成标记" | 无 |
| 7 | **F-10** | A 层 append 加 contentHash 去重；引入 opId | 可随 V2 |
| 8 | **F-11** | tombstone + 恢复函数（校验 schema → 比对 tombstone → 跳过已删 → 原子写） | 随 V2 P4 |
| 9 | **F-04 / F-08** | B 层淘汰时同步产生 A 层删除 effect；embed 失败入重试队列 | 随 V2 P2（WAL） |
| 10 | **F-09** | 审批/申请路径改用精确匹配（与 remove 同口径），或返回候选列表由操作者确认 | 无 |
| 11 | F-12~F-22 | 坏行告警、vec 维度校验、审计脱敏与轮转、文案修正、条件式鉴权改无条件 | 无 |

---

# 第 9 部分 · 尚未确认 / NOT_VERIFIABLE 清单

| # | 事项 | 原因 |
|---|---|---|
| N-1 | 「原始 SHA256」与项目原文件的一致性 | 原文件不在包内；脱敏不可逆（`05` §2.5） |
| N-2 | 脱敏是否真的移除了真实 QQ/姓名 | 导出副本无占位符也无账号，两种解释不可区分 |
| N-3 | R1-R8 竞态的**实际触发概率** | 需并发测试，本次禁止 |
| N-4 | `truncateText(text, 200)` 的确切语义 | 来自 `./md-to-plain.js`，**未包含在 12 个导出文件中**；`pendingThought` 用的 `text.slice(0,200)` 已确认 |
| N-5 | 是否有项目外的进程写 `memory-vectors.jsonl` | 静态不可排除 |
| N-6 | `mtime` 同毫秒碰撞概率、运行时缓存行为 | 需运行时观察，禁止 |
| N-7 | Ollama 实际返回维度与可用性（embed 30s 超时、模型切换） | 需环境验证，禁止 |
| N-8 | `consolePortInUse` 在无 `netstat`/`ss` 环境下的行为 | 需环境验证，禁止 |
| N-9 | `bridge.lock` 僵尸锁（PID 12412）当前是否已被清理 | 只读限制，未触碰 |
| N-10 | `mcp-snowluma-safe.js` 剩余约 1,246 行是否存在间接 Memory 调用 | 已做全文件符号穷举（0 命中 memory/socialV2 之外的耦合），但未逐行阅读 |
| N-11 | `scripts/*.mjs`（约 40 个）是否写 A 层 | 未导出；`05` §1.3 称已全仓确认无，无法独立复核 |
| N-12 | `loadRows` 返回引用导致的 R2 各分支实际占比 | 需运行时验证，禁止 |

---

# 第 10 部分 · 审查边界声明

```
✅ 仅只读：未修改任何项目代码/配置/数据/日志/锁文件
✅ 未启动、停止或重启任何服务
✅ 未执行任何业务写入测试
✅ 未调用 qq_memory_append / remove / clear / 分享审批等任何业务接口
✅ 所有源码操作均在临时工作副本上进行，行号与原文件一一对应（已用分段偏移校验）
✅ 未逐行全量阅读的部分已在 §1.3 明确列出，未声称全量审查
✅ 对历史报告的每一条结论均独立复核；发现错误已明确修正/撤回，未迎合保留

审查到此结束，未进入修复阶段，未改动任何项目文件。
```
