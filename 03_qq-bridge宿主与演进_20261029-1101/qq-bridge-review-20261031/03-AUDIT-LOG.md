# 03 · AUDIT LOG（只读审查执行记录）

- 审查开始：2026-10-31 13:20
- 审查结束：2026-10-31 13:58
- 执行环境：Windows，Python 3.13.12（绝对路径调用），Node 22.22.2
- 全部命令均为**只读**（读取文件、stat、hashlib、tasklist、socket connect）
- **未执行**任何写入、删除、迁移、服务启动命令

> 本日志不包含：真实记忆正文、QQ 账号、Token、密钥、私聊内容。
> 如命令输出意外包含敏感内容，已在写入本日志前脱敏。

---

## 阶段 A｜基线建立

### A1. Git 仓库状态检查

| 项 | 内容 |
|---|---|
| 命令 | `git rev-parse --is-inside-work-tree` |
| 目标 | 确认是否有 Git 版本控制 |
| 结果 | **失败（预期）** — `fatal: not a git repository (or any of the parent directories): .git` |
| 结论 | 项目无 Git；不得虚构 commit hash |
| 影响 | 后续只能用 SHA256 作为基线 |

### A2. `state/` 目录与关键文件存在性

| 项 | 内容 |
|---|---|
| 命令 | Python `os.path.exists` / `os.listdir` + `os.path.getsize` |
| 目标 | 确认 V2.4.1 设计的 9 个模块是否存在；列出 `state/` 全部文件 |
| 检查对象 | `src/memory-wal.js`、`src/writer-lock.js`、`src/normalize.js`、`state/memory-tombstones.jsonl`、`state/share-requests.jsonl`、`state/fact-keys.json`、`state/writer.lock`、`state/memory-audit.jsonl`、`state/wal/` |
| 结果 | **9 个全部不存在** |
| 结论 | V2.4.1 设计**未实施** |
| 证据 | 见 `02-AUDIT-REPORT.md` 基线部分 |

### A3. 核心源码 SHA256 基线

| 项 | 内容 |
|---|---|
| 命令 | Python `hashlib.sha256` 流式读取 + 行数统计 |
| 目标 | 建立 12 个核心文件的基线 |
| 扫描文件数 | 12 |
| 结果 | 全部成功；清单见 `01-BASELINE.md` 第 3 节 |
| 结论 | 基线建立完成，保存至 `_baseline.json`（内部中间文件） |

---

## 阶段 B｜第一批疑点补充核查

### B1. 关键符号存在性计数

| 项 | 内容 |
|---|---|
| 命令 | Python 字符串 `count()` 全文统计 |
| 目标 | 确认 V2.4.1 概念是否在代码中 |
| 检查符号 | `_crc`、`crc32`、`embeddingStatus`、`memory-wal`、`tombstone`、`ShareRequest`、`share-requests`、`writer.lock`、`effects`、`expectedRevision`、`factKey`、`revision`、`lifecycleEvents`、`vecPurged`、`contentPurged` |
| 扫描文件 | `src/bridge.js`、`src/memory-vector.js` |
| 结果 | **全部 0 次**（`canonical` 34 次但指 `canonicalV2Key`，与 CRC 无关；`status` 31 次指 `share_review_status`） |
| 结论 | 全部 `NOT_IMPLEMENTED` |

### B2. 写入口穷举（疑点 1）

| 项 | 内容 |
|---|---|
| 命令 | Python 正则全文搜索 |
| 目标 | 找出所有写 `memory-vectors.jsonl` 的代码路径 |
| 正则 | `appendFileSync\(\s*VEC_FILE`、`writeFileSync\(\s*tmp`、`renameSync\(\s*tmp`、`atomicRewrite\(` |
| 扫描范围 | `D:\qq-bridge\**\*.js` + `**\*.mjs`（排除 `node_modules`） |
| 结果 | 仅 `memory-vector.js` 内 2 条写路径：`:160` appendFileSync、`:378-379` atomicRewrite；另有 `bridge.js:6716` 单向拷出（备份） |
| 第二写入者 | **未找到**（`seed-group-owner.mjs` 已归档为 `.disabled`） |
| 结论 | 跨进程当前无第二写入者；同进程 async 是真实风险 |

### B3. atomicRewrite 调用点与 async 属性（疑点 1）

| 项 | 内容 |
|---|---|
| 命令 | Python 逐行分析函数归属 |
| 目标 | 确认 7 个调用点中哪些是 async |
| 结果 | 6 个同步（requestShare/approveShare/rejectShare/removeVectorMemoryExact/removeNAll/removeAll）；**1 个 async：`editVectorMemory`（:361）** |
| 结论 | `editVectorMemory` 的 `await embed` 是丢失更新的根因 |

### B4. canonicalV2Key 语义（疑点 2）

| 项 | 内容 |
|---|---|
| 命令 | 读取 `bridge.js:1370-1376` |
| 目标 | 确认 key 规范化是否改变 `private`/`group` 前缀 |
| 结果 | 正则 `^(group\|private):(\d+)$`，返回 `${m[1]}:${id}` —— **只去前导零，不改前缀** |
| 结论 | `private:X` 与 `group:X` 永远是不同 key |

### B5. appendVectorMemory 写入字段（疑点 2）

| 项 | 内容 |
|---|---|
| 命令 | 读取 `memory-vector.js:149-163` |
| 目标 | 确认写入时是否带 `visibility` |
| 结果 | 只写 `key, text, time, vec, share_review_status` —— **无 visibility 字段** |
| 结论 | 群记录初始 `visibility=undefined` → 群检索初始不可见（正确）；审批后才变 `group_shared` |

### B6. 分享审批的可见性判断（疑点 2）

| 项 | 内容 |
|---|---|
| 命令 | 读取 `memory-vector.js:171-199` |
| 目标 | 确认群检索的过滤条件 |
| 结果 | `:172` key 全等过滤 → `:177` `visibility==='group_shared' && share_review_status==='approved'` |
| 结论 | 群记录（key=group:X）审批后，key 匹配 + 条件满足 → **能被本群检索** |
| **修正** | 撤回第一批报告「分享功能实质不可用」的结论 |

### B7. 锁获取与端口监听顺序（疑点 3）

| 项 | 内容 |
|---|---|
| 命令 | Python 正则搜索 `acquireLock()`、`.listen(`、`releaseLock()` |
| 目标 | 确认锁与监听的执行顺序 |
| 结果 | `:1390 acquireLock()` → `:7531 server.listen()`（相隔 6141 行）；`:7535-7542 server.on('error')` 处理 EADDRINUSE；`:12447-12477` 三个退出钩子 |
| 结论 | 存在启动窗口；但端口是第二道防线，**不会真双实例** |

### B8. consolePortInUse 保守策略（疑点 3）

| 项 | 内容 |
|---|---|
| 命令 | 读取 `bridge.js:1064-1078` |
| 目标 | 确认探测失败时的行为 |
| 结果 | `catch { return true; }` —— 探测失败**保守认为"有人在跑"** |
| 结论 | 无 netstat 环境下会判定"已有实例"，需人工删锁（P1） |

### B9. 审计写入顺序（疑点 4）

| 项 | 内容 |
|---|---|
| 命令 | 读取 `bridge.js:6730-6762` |
| 目标 | 确认业务与审计的先后 |
| 结果 | `:6731` 业务 → `:6733` 审计（remove）；`:6739` → `:6740`（edit）；`:6750` → `:6754`（approve/reject）；外层 catch 在 `:6760` |
| 结论 | **全部"业务→审计"**；`appendFileSync` 无 try/catch → 失败时返回 500 但业务已生效 |

### B10. clear 端点审计检查（疑点 4）

| 项 | 内容 |
|---|---|
| 命令 | 扫描 `bridge.js:7014-7123` 内的 `memory-admin.log` / `appendFileSync` |
| 目标 | 确认 clear 是否有审计 |
| 结果 | **无任何审计写入**（只有 `:7121` 的 sendJson） |
| 结论 | clear 完全无审计记录（P1） |

### B11. bridge.lock 实际内容与 PID 存活（疑点 3 附加）

| 项 | 内容 |
|---|---|
| 命令 | Python 读文件 + `tasklist /FI "PID eq 12412" /FO CSV /NH` + socket connect 3100 |
| 目标 | 确认当前锁状态 |
| 结果 | 内容 `12412`；tasklist 无匹配（PID 不存在）；端口 3100 未监听 |
| 结论 | **僵尸锁**，但不阻碍下次启动（代码有 stale 检测） |
| 注意 | **未删除、未修改锁文件** |

---

## 阶段 C｜新增审查项目

### C1. 备份/恢复函数搜索（第 10 项）

| 项 | 内容 |
|---|---|
| 命令 | Python 正则搜索 `restore`、`importMemor`、`loadBackup`、`fromBackup`、`copyFileSync.*memory-vectors` |
| 结果 | 19 处命中，**全部与 memory-vectors 恢复无关**（`restore` 多为 preset 恢复、计时器恢复） |
| 结论 | **代码中无 memory-vectors 恢复函数**，恢复只能人工 copy |

### C2. dailyBackup 实现（第 10 项）

| 项 | 内容 |
|---|---|
| 命令 | 读取 `memory-vector.js:131-147` + 搜索调用点 |
| 结果 | 仅在 `:154`（append 内）调用一次；保留 7 天；`if (!fs.existsSync(dest))` 同一天只备份一次；`catch {}` 静默 |
| 结论 | 删除/编辑不触发备份；备份失败无提示 |

### C3. 幂等性检查（第 9 项）

| 项 | 内容 |
|---|---|
| 命令 | 搜索 `opId`、`idempot`；读取 `appendVectorMemory` 与 `editVectorMemory` |
| 结果 | `opId` 0 次；`appendVectorMemory` 无查重；`editVectorMemory` 有 `await embed` |
| 结论 | 无幂等机制；存在删除复活路径（详见报告 9.5） |

### C4. 反引号连续长度检查（导出准备）

| 项 | 内容 |
|---|---|
| 命令 | Python 正则 `\`+` 统计最长连续 |
| 目标 | 选择安全的 Markdown 围栏长度 |
| 结果 | `bridge.js` / `memory-vector.js` / `mcp-snowluma-safe.js` 最长均为 **1** |
| 结论 | 用 4 个反引号作围栏，安全 |

---

## 阶段 D｜导出与复核

### D1. 源码 Markdown 生成

| 项 | 内容 |
|---|---|
| 命令 | Python 脚本：读原文件 → 脱敏替换 → 按段切分 → 写 Markdown → 计算导出 SHA256 |
| 脱敏 | 5 条替换规则（见 `05-EXCLUSIONS-REDACTIONS.md`） |
| 行数校验 | 脚本内 `assert len(redacted_lines) == original_lines` —— **全部通过** |
| 输出 | 8 个 Markdown，12 个源码条目 |
| 结果 | 全部成功 |

### D2. 索引追加

| 项 | 内容 |
|---|---|
| 命令 | Python 追加写入 `04/05/06` 三个 md |
| 目标 | 补充 SocialV2 / 锁备份 / 分享权限的**行号索引**（这些模块的代码在 bridge.js 内，已完整导出在 `02-bridge.md`） |
| 结果 | 成功 |

### D3. 结束 SHA256 复核

| 项 | 内容 |
|---|---|
| 命令 | 重新计算 12 个核心文件 SHA256，与 `_baseline.json` 对比 |
| 结果 | **12/12 完全一致，变化 0 个** |
| 结论 | 审查期间项目**零修改** |

### D4. 导出文件完整性核验

| 项 | 内容 |
|---|---|
| 命令 | 检查每个 md 的围栏配对数、源码段行数、是否含省略标记 |
| 结果 | 见 `00-README.md` 的核验结论 |

---

## 未执行的命令（明确标注）

| 命令 / 操作 | 原因 |
|---|---|
| `git status` / `git log` / `git diff` | 无 Git 仓库 |
| 启动 bridge / DSH / Ollama | 任务禁止 |
| 向 bridge 发 HTTP 请求（测试写入） | 任务禁止 |
| 并发写测试 | 任务禁止（会改数据） |
| 多实例启动验证锁竞争 | 任务禁止 |
| 删除 / 清理 `bridge.lock` | 任务禁止（只允许读） |
| 读取 `memory-vectors.jsonl` 正文 | 当前 0 字节，无需读取；历史备份正文按约束不读取 |
| 读取 `state/social-v2.json` 正文 | 按约束排除（只统计字段条数） |
| 解压 `~/.dsh/sessions/**` | C 层真实聊天内容，按约束排除 |

---

## 仍未验证 / 证据不足

| # | 项 | 原因 |
|---|---|---|
| 1 | 并发写损坏的实际概率 | 需并发测试（禁止） |
| 2 | `consolePortInUse` 在无 netstat 环境行为 | 需环境测试（禁止） |
| 3 | mtime 同毫秒碰撞概率 | 需构造测试（禁止） |
| 4 | 锁竞争窗口的实际触发频率 | 需多次启动观察（禁止） |
| 5 | `mcp-snowluma-safe.js` 其余 1400 行 | 只抽查记忆相关工具 |
| 6 | 外部进程（非本项目）是否写 memory-vectors | 无法从本项目代码排除 |

---

## 日志完整性声明

```
✅ 上述所有命令均实际执行过，输出摘要如实记录
✅ 未执行的命令已在上表明确标注
✅ 未伪造任何命令输出
✅ 日志不含记忆正文、QQ 号、Token、私钥
✅ 检查的文件数量：源码 12 个（导出）+ 数据文件元数据 18 个
```
