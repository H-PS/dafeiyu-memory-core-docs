# 00 · README（交付说明与核验结论）

## 这是什么

对 `D:\qq-bridge` 项目 **Memory 系统**的一次**只读取证 + 完整源码导出**，供 ChatGPT 独立复核。

- 项目：QQ ↔ DeepSeek-Harness 桥接（大肥鱼）
- 审查对象：A 层向量记忆 + B 层 SocialV2 + 权限/分享/锁/备份/审计
- 版本基线：**无 Git**，以 SHA256 为唯一基线（12 个核心文件，审查前后完全一致）
- 设计背景：此前已完成 Memory V2.1 ~ V2.4.1 设计（纯设计，未实施）

## 最重要的一句话结论

> **V2.4.1 设计的 9 个模块在代码中全部不存在**（`memory-wal` / `tombstone` / `ShareRequest` / `factKey` / `revision` / `effects` / `_crc` / `embeddingStatus` / `writer.lock` 均 0 次出现）。
> 仓库里是 **V1 实现**：`memory-vector.js`（380 行）+ `bridge.js`（12,482 行）。
> 因此 8 项审查中 5 项为 `NOT_IMPLEMENTED`；另 3 项为 `PARTIAL`（有 V1 雏形）。
> 同时发现 **V1 本身的 3 个 P0 缺陷**，其中 1 个是本轮新发现（删除复活路径）。

## 目录结构

```text
D:\qq-bridge-review-20261031\
├── 00-README.md                  ← 本文件：导航 + 核验结论
├── 01-BASELINE.md                ← 基线：无 Git、审查时间、12 文件 SHA256、state 清单
├── 02-AUDIT-REPORT.md            ← ★ 核心：10 项审查 + 4 处修正 + 3 个 P0
├── 03-AUDIT-LOG.md               ← 审查日志：实际执行的每条只读命令与结论
├── 04-MANIFEST.md                ← 清单：12 个源文件的路径/大小/行数/原始哈希/导出哈希
├── 05-EXCLUSIONS-REDACTIONS.md   ← 排除清单 + 脱敏规则与记录
├── SHA256SUMS.txt                ← 全部交付文件的哈希（可用 `sha256sum -c` 校验）
└── source/                       ← 5 个分卷，每卷含多个完整源文件
    ├── 01-memory-core.md         ← memory-vector.js（380）＋ 数据模型/存储/检索算法索引
    ├── 02-bridge.md              ← bridge.js（12,482，分 3 段）＋ 端点/鉴权/key 流转链索引
    ├── 03-mcp-social.md          ← mcp-snowluma-safe.js（1,465）＋ qq-model-view.js（134）
    │                                ＋ mcp-diary.js（113）＋ mcp-host-server.js（93）
    │                                ＋ MCP 工具清单 / SocialV2 位置 / B 层字段语义
    ├── 04-infrastructure.md      ← token-ledger.js（958）＋ package.json（72）
    │                                ＋ config.example.json（279）＋ 锁/原子写/备份索引
    └── 05-tests-and-dependencies.md ← slang-learner.js（382）＋ self-test.js（118）
                                       ＋ sticker-lib.js（302）＋ 调用链/写入口穷举
```

**分卷规则**：每卷可含多个源文件，每个源文件用一级标题 `# 源码文件：<相对路径>` 开头，`---` 分隔。`bridge.js` 因 12,482 行过长，按 1-4200 / 4201-8400 / 8401-12482 三段**连续拆分**（无重叠无遗漏）。

## 阅读顺序建议

| 目的 | 读什么 |
|---|---|
| 快速了解结论 | 本文件 → `02-AUDIT-REPORT.md` 的「第三部分 汇总」与「第四部分 P0」 |
| 核对某个结论 | `02-AUDIT-REPORT.md` 对应章节 → `source/` 中对应文件的行号 |
| 验证没改项目 | `01-BASELINE.md` 第 3 节（12/12 哈希一致） |
| 确认没泄漏数据 | `05-EXCLUSIONS-REDACTIONS.md` |
| 自己复算哈希 | `SHA256SUMS.txt` + `04-MANIFEST.md` |

## 10 项审查结果速览

| # | 审查项 | 状态 | 等级 |
|---|---|---|---|
| 1 | CRC 递归规范化 | `NOT_IMPLEMENTED` | — |
| 2 | embeddingStatus | `NOT_IMPLEMENTED` | — |
| 3 | WAL effects + revision | `NOT_IMPLEMENTED` | — |
| 4 | tombstone 检索屏障 | `NOT_IMPLEMENTED` | — |
| 5 | ShareRequest 孤儿副本 | `NOT_IMPLEMENTED` | — |
| 6 | L2 删除与 B 层 | `PARTIAL` | P1 |
| 7 | writer.lock 生命周期 | `PARTIAL` | P1 |
| 8 | 特权操作审计 | `PARTIAL` | **P0** |
| 9 | 写入幂等性 | **`FAIL`** | **P0** |
| 10 | 备份恢复防复活 | `PARTIAL` | P1 |

## 3 个 P0（必须优先处理）

| # | 问题 | 位置 | 触发 | 后果 |
|---|---|---|---|---|
| **P0-1** | **embed 期间删除 → 记录复活** | `memory-vector.js:361-374` | `editVectorMemory` 的 `await embed`（最长 30s）期间执行 remove/clear | 已删记忆重新出现在检索结果 |
| **P0-2** | **审计在业务后 + 无 try/catch** | `bridge.js:6731-6734 / 6739-6741 / 6750-6755` | 磁盘满 / ACL 变更 | 记忆已删/已改，但返回 HTTP 500，且无审计痕迹 |
| **P0-3** | **无 WAL → 崩溃后 A/B 不一致不可恢复** | 全局 | 进程崩溃 | 永久不一致，无自动发现 |

## 对第一批报告的 4 处修正（重要）

| # | 原结论 | 修正 |
|---|---|---|
| 1 | 「approveShare 只改 visibility 不改 key → 分享功能实质不可用」 | **撤回**。群内记忆审批后**本群能正常检索**（key 保持 group:X 恰好正确）；私聊→群不支持（已被 `bridge.js:6841` 400 拒绝，属设计边界） |
| 2 | 「atomicRewrite 固定 tmp 名 → 并发写必然损坏」 | **细化**。跨进程当前无第二写入者；真实风险是**同进程 async 丢失更新** |
| 3 | 「P0：分享功能不可用」 | **降级为无缺陷** |
| 4 | 未提及 embed 期间删除 | **新增 P0-1**（可单线程复现） |

## 文件核验结论

```
✅ 交付文件总数：12（6 个报告 md + 5 个 source 分卷）+ 1 个 SHA256SUMS.txt
✅ 源码文件数：12（覆盖 Memory 全部调用链：核心/桥接/工具/社交/存储/配置/依赖）
✅ 所有 Markdown 可正常读取
✅ 代码围栏配对：5/5 分卷均无未闭合围栏（4 反引号，源文件内最长连续反引号为 1）
✅ 源码完整性：
   · bridge.js  12,482 行 → 3 段（1-4200 / 4201-8400 / 8401-12482），无重叠无遗漏
   · 其余 11 个文件全文导出，代码块行数与原始行数**逐一精确匹配**
✅ 无省略标记（未出现 `...省略` / `其余不变` / `此处省略`）
✅ 脱敏未改变行数（脚本内 assert 保证）→ 报告行号可直接对照导出源码
✅ 项目核心源码审查前后 SHA256：12/12 一致，变化 0
✅ 未修改任何项目文件（代码/配置/数据/Persona/preset）
✅ 未启动或重启任何服务
✅ 未写入测试记忆
```

## 如何独立复核

### 1. 验证导出文件未被篡改

```bash
cd D:\qq-bridge-review-20261031
sha256sum -c SHA256SUMS.txt      # Git Bash / WSL
# 或 PowerShell:
# Get-Content SHA256SUMS.txt | ForEach-Object { ... }
```

### 2. 验证项目未被修改

对照 `01-BASELINE.md` 第 3 节的 12 个 SHA256，重新计算 `D:\qq-bridge` 下对应文件：

```bash
sha256sum D:/qq-bridge/src/{memory-vector.js,bridge.js,mcp-snowluma-safe.js,...}
```

### 3. 定位某段代码

1. 在 `02-AUDIT-REPORT.md` 找到「文件:行号」
2. 打开 `source/` 中对应 md
3. 按段标题的行号范围直接跳转（`bridge.js` 分 3 段，每段标题写明起始行）

### 4. 注意脱敏占位符

导出源码中的 `[REDACTED_OWNER_QQ]` / `[REDACTED_NAME]` 等是占位符。它们**不改变行数**，因此行号精确。原值不在任何交付文件中。

## 已知限制

| # | 限制 |
|---|---|
| 1 | 项目无 Git，无法提供 commit / 分支 / diff；只能靠 SHA256 + 手工备份 |
| 2 | 并发相关的实际概率需运行时测试，本任务禁止执行 |
| 3 | `mcp-snowluma-safe.js` 只审查了记忆相关工具，其余 1400 行未逐行审查 |
| 4 | 无法排除"本项目之外的其他进程写 memory-vectors.jsonl" |
| 5 | 脱敏后无法从导出副本反推原始文件哈希（已分别记录两者） |

---

*本次为只读审查。未修改项目、未启动服务、未写入测试数据。*
