# 01 · BASELINE（统一代码基线）

## 1. Git 状态

| 项 | 结果 |
|---|---|
| 仓库路径 | `D:\qq-bridge` |
| 是否为 Git 仓库 | ❌ **否** |
| 依据 | `git rev-parse --is-inside-work-tree` → `fatal: not a git repository (or any of the parent directories): .git` |
| `.git` 目录 | 不存在 |
| 当前分支 | **无法提供** |
| HEAD commit hash | **无法提供** |
| 工作区是否干净 | **无法判定**（无版本控制） |
| 未提交修改 | **无法判定** |
| 未跟踪文件 | **无法判定** |

> ⚠️ 本报告**不虚构任何 commit hash**，也不声称具有 Git 版本保证。全部代码按「当前工作区唯一版本」对待。
> 历史对比只能依赖 `D:\qq-bridge备份-20261030\*.zip`（37 个手工备份）与 `state/*.pre-*.bak`。

## 2. 审查时间

| 项 | 时间 |
|---|---|
| 审查开始 | 2026-10-31 13:20 |
| 审查结束 | 2026-10-31 13:58 |
| 导出完成 | 2026-10-31 14:00 |

## 3. 核心源码 SHA256（开始 vs 结束）

**审查期间项目未被修改：12/12 文件哈希完全一致，变化 0 个。**

| # | 相对路径 | 大小(字节) | 行数 | SHA256（开始 = 结束） |
|---|---|---|---|---|
| 1 | `src/memory-vector.js` | 15,788 | 380 | `1d6dc555920c7edc…`（完整值见 `04-MANIFEST.md`） |
| 2 | `src/bridge.js` | 701,164 | 12,482 | `47bdeb20e2416d72…` |
| 3 | `src/mcp-snowluma-safe.js` | 86,093 | 1,465 | `803a05cc8c6c8483…` |
| 4 | `src/qq-model-view.js` | 5,855 | 134 | `5beddd47897c98b2…` |
| 5 | `src/mcp-diary.js` | 4,097 | 113 | `0010dd1e7ab643f4…` |
| 6 | `src/mcp-host-server.js` | 4,597 | 93 | `68cc06ecb8f45aee…` |
| 7 | `src/slang-learner.js` | 16,534 | 382 | `8d51fa76144f068c…` |
| 8 | `src/sticker-lib.js` | 15,260 | 302 | `58d3c3084e524690…` |
| 9 | `src/token-ledger.js` | 39,568 | 958 | `4d9141ce36cf0376…` |
| 10 | `src/self-test.js` | 5,005 | 118 | `6fc5f2201d43050e…` |
| 11 | `package.json` | 3,774 | 72 | `b6a395123fcc6550…` |
| 12 | `config.example.json` | 7,449 | 279 | `e1346f09b2d4f4be…` |

> 完整 64 位 SHA256 见 `04-MANIFEST.md` 第 2 节表格。

## 4. `state/` 目录清单（只读元数据，未读取正文）

| 文件 | 大小(字节) | 说明 |
|---|---|---|
| `agents/` | (dir) | 子目录 |
| `bridge.lock` | 5 | 内容 `12412`（**僵尸锁**，见第 6 节） |
| `bridge.log` | 73,431 | 运行日志（未导出） |
| `console-token` | 48 | 控制台令牌（**未导出**） |
| `current-role.json` | 46 | 当前角色（含角色名） |
| `memory-admin.log` | 333 | **审计日志，2 条记录**（含记忆正文，未导出原始内容） |
| `memory-vectors.jsonl` | **0** | A 层记忆库，已清空 |
| `memory-vectors.jsonl.pre-p0a.bak` | 2,162,534 | 历史备份（101 条，**未导出**） |
| `memory-vectors.jsonl.pre-vis.bak` | 2,076,154 | 历史备份（101 条，**未导出**） |
| `mode.json` | 53 | 运行模式 |
| `prompt-dump.log` | 213,006 | 提示词转储（未导出） |
| `qq-activity.log` | 17,213 | 活动日志（未导出） |
| `sessions.json` | 322 | 会话映射 |
| `social-v2.json` | 101,136 | B 层状态（含真实会话内容，**未导出**） |
| `stickers.json` | 559 | 表情库元数据 |
| `token-usage.jsonl` | 75,616 | token 用量（未导出） |
| `tool-calls.jsonl` | 118,504 | 工具调用记录（未导出） |
| `tool-restrict.log` | 3,920 | 工具限制日志（未导出） |

## 5. 数据状态摘要（只读统计，不含正文）

| 项 | 值 |
|---|---|
| A 层记忆条数 | **0**（文件 0 字节） |
| B 层会话数 | 2（主人私聊 + 1 个群） |
| B 层记忆字段 | `activeTopics`=0、`pendingThoughts`=0、`memberImpressions`=0（全部为空） |
| B 层运行数据 | `recentMessages` 100 / 46 条，`unread` 4 / 25 条（**属运行数据，不是长期记忆**） |
| 历史备份中的 A 层条数 | `backups/memory-vectors-2026-10-30.jsonl` = 101 条；两个 `.pre-*.bak` = 101 条 |

## 6. `bridge.lock` 状态（脱敏结构信息）

| 项 | 值 |
|---|---|
| 路径 | `state/bridge.lock` |
| 大小 | 5 字节 |
| 内容 | `12412`（纯 PID 数字，无 `instanceId`、无 `createdAt`、无 `mode` 字段） |
| PID 12412 是否存活 | **否**（`tasklist /FI "PID eq 12412"` → 无匹配） |
| 端口 3100 是否占用 | **否**（TCP connect 失败） |
| 判定 | **僵尸锁**（上次非正常退出未释放） |
| 是否清理 | **未清理**（本任务禁止修改/删除锁文件） |
| 影响 | 按 `bridge.js:1094-1138` 逻辑，下次启动会检测到 PID 不存在 → `stale=true` → 自动删除并重试，**不阻碍启动** |

## 7. 基线一致性声明

```
✅ 12/12 核心源码文件在审查结束后 SHA256 与开始时完全一致
✅ 未执行任何 git checkout / reset / clean / stash / commit
✅ 未修改、创建、删除任何项目文件
✅ 未清理锁文件
✅ 未读取或复制真实记忆正文（只读元数据与条数统计）
```

---

*基线信息对应导出包中 `source/` 下 8 个 Markdown 文件的原始文件。导出副本已脱敏，但行号与原始文件完全一致。*
