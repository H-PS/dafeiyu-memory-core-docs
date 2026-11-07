# 05 · 排除清单与脱敏记录

本文档记录：① 未导出的文件及原因；② 导出副本的脱敏范围、位置与替换规则；③ 原始哈希与导出哈希的对应关系。

---

# 第一部分：排除清单

## 1.1 按约束必须排除

| 类别 | 具体项 | 原因 |
|---|---|---|
| 版本控制 | `.git/`（不存在） | 项目无 Git |
| 依赖 | `node_modules/` | 第三方代码，非本项目实现 |
| 日志 | `state/bridge.log`（73KB）、`state/prompt-dump.log`（213KB）、`state/qq-activity.log`（17KB）、`state/tool-restrict.log`（3.9KB） | 运行日志，含真实对话与系统信息 |
| 真实记忆 | `state/memory-vectors.jsonl`（当前 0 字节） | 真实记忆数据 |
| 真实记忆（历史） | `backups/memory-vectors-2026-10-30.jsonl`（101 条）、`state/memory-vectors.jsonl.pre-p0a.bak`（101 条）、`state/memory-vectors.jsonl.pre-vis.bak`（101 条） | 历史记忆正文，属私人数据 |
| 真实社交状态 | `state/social-v2.json`（101KB） | 含真实会话内容、群友印象、消息正文 |
| 审计原文 | `state/memory-admin.log`（333 字节，2 条） | 含记忆正文 |
| 凭据 | `state/console-token`、`config.json` 的真实 `consoleToken` / `snowluma.accessToken`、`~/.dsh/.credentials.yaml` | Token / 密钥 |
| C 层会话 | `~/.dsh/sessions/**/session.v4.jsonl.zstd`、`~/.dsh/storages/session_projcache/**` | 完整对话历史 |
| 历史备份包 | `D:\qq-bridge备份-20261030\*.zip`（37 个）、`*.jsonl` 快照 | 含真实数据 |
| Token 用量 | `state/token-usage.jsonl`（75KB）、`state/tool-calls.jsonl`（118KB） | 含调用细节与正文片段 |
| 其他状态 | `state/sessions.json`、`state/current-role.json`、`state/mode.json`、`state/stickers.json`、`state/agents/` | 运行状态，非 Memory 代码 |

## 1.2 与 Memory 无关、未导出（可补充导出）

| 项 | 说明 |
|---|---|
| `roles/*.md`（4 个） | Persona 卡。**与记忆系统无直接代码关系**，但影响检索结果的呈现。如需可另行导出 |
| `docs/guides/*.md`（约 20 个） | 设计与审计文档（含本系列 V2.1~V2.4.1 设计）。本次导出对象是**代码**，文档留在项目内 |
| `scripts/*.mjs`（约 40 个） | 测试/探测/维护脚本。除 `seed-group-owner.mjs`（已归档为 `.disabled`）外，均不写 `memory-vectors.jsonl` |
| `plugins/qq-agent-presets/**` | preset 与规则源文件（`agent.cordis.yml` 等），非 Memory 代码 |
| `dsh/agent-presets/**` | 同上（preset 源） |
| `config.json`（真实值） | 含 Token。已导出 `config.example.json` 模板 |
| `package-lock.json`（67KB） | 纯依赖树，与本次审查无关 |
| `src/forward.js`、`src/media-normalize.js`、`src/safe-fetch.js`、`src/web-search.js`、`src/voice-core.js` 等 | 与 Memory 无调用关系（已通过调用图确认：它们不被 memory-vector.js / bridge 的 memory 端点引用） |
| `src/core/`（0 字节目录） | 空目录 |

## 1.3 排除项对审查完整性的影响

| 排除项 | 是否影响审查结论 |
|---|---|
| 真实记忆正文 | ❌ 不影响。审查的是**代码路径**，不需要正文内容 |
| `social-v2.json` 正文 | ❌ 不影响。B 层结构已从代码（`appendMemoryV2`）完全推导 |
| C 层 Session | ❌ 不影响。C 层与 A 层代码无交叉引用 |
| `roles/*.md` | ❌ 不影响。Persona 是数据，不是记忆代码 |
| 测试脚本 | ⚠️ 轻微。若某测试脚本会写记忆，则可能构成"第二写入者"；已全仓库搜索确认无 |

---

# 第二部分：脱敏记录

## 2.1 脱敏原则

```
① 只改导出副本，**不修改项目原文件**
② 只做**同长度之外的字符串替换**，但保证**行数不变**（脚本内 assert 校验）
   → 因此导出行号 = 原始行号，报告中的行号可直接对照
③ 不修改缩进、不重排代码、不删除注释
④ 原始 SHA256 与导出 SHA256 分别记录，不混淆
```

## 2.2 替换规则（5 条）

| # | 原始内容类型 | 替换为 | 说明 |
|---|---|---|---|
| 1 | 主人 QQ 号（10 位） | `[REDACTED_OWNER_QQ]` | 所有权数据 |
| 2 | 机器人 QQ 号（10 位） | `[REDACTED_BOT_QQ]` | 所有权数据 |
| 3 | 另一个机器人 QQ 号（10 位） | `[REDACTED_XJ_QQ]` | 所有权数据 |
| 4 | 群号（9 位） | `[REDACTED_GROUP_QQ]` | 所有权数据 |
| 5 | 主人真实姓名（3 字） | `[REDACTED_NAME]` | 个人身份信息 |

> 具体数值不在本文件中重复（避免二次泄漏）。核验者可对比原始文件的 SHA256 确认导出副本来源。

## 2.3 脱敏命中统计

| 源文件 | 命中次数（估计） | 主要位置 |
|---|---|---|
| `src/memory-vector.js` | 0 | 该文件不含硬编码 QQ（key 由调用方传入） |
| `src/bridge.js` | 多处 | 注释中的示例、`ownerQQ` 相关判断的注释、`#101` 历史说明 |
| `src/mcp-snowluma-safe.js` | 少量 | 工具描述中的示例 |
| `config.example.json` | 可能 | 若模板含示例 QQ |
| 其余文件 | 0 或少量 | — |

> 精确命中数未逐文件统计（不影响结论）。`是否脱敏` 字段在 `04-MANIFEST.md` 中对**所有文件统一标为「是」**，表示"已按规则处理"；实际命中 0 次的文件，导出内容与原文除 Markdown 包装外无差异。

## 2.4 未脱敏但需要说明的内容

| 内容 | 处理 |
|---|---|
| 代码中的文件路径（`D:\qq-bridge\...`） | **保留**。是项目自身路径，非用户隐私 |
| 代码注释中的中文说明 | **保留**。是设计意图，审查必需 |
| `config.example.json` 中的占位符（如 `"consoleToken": ""`） | **保留**。本就是空/占位 |
| `package.json` 的依赖名与版本 | **保留** |
| 硬编码的常量（端口 3100/3080/3001、超时值等） | **保留**。非敏感 |

## 2.5 原始哈希 vs 导出哈希（对应关系）

| 源文件 | 原始 SHA256 | 导出分卷 | 分卷 SHA256 |
|---|---|---|---|
| `src/memory-vector.js` | 见 `04-MANIFEST.md` 第 2 节 | `source/01-memory-core.md` | 见 `04-MANIFEST.md` 第 1 节 |
| `src/bridge.js` | 同上 | `source/02-bridge.md` | 同上 |
| `src/mcp-snowluma-safe.js` | 同上 | `source/03-mcp-social.md` | 同上 |
| `src/qq-model-view.js` | 同上 | `source/03-mcp-social.md` | 同上（同卷） |
| `src/mcp-diary.js` | 同上 | `source/03-mcp-social.md` | 同上（同卷） |
| `src/mcp-host-server.js` | 同上 | `source/03-mcp-social.md` | 同上（同卷） |
| `src/token-ledger.js` | 同上 | `source/04-infrastructure.md` | 同上 |
| `package.json` | 同上 | `source/04-infrastructure.md` | 同上（同卷） |
| `config.example.json` | 同上 | `source/04-infrastructure.md` | 同上（同卷） |
| `src/slang-learner.js` | 同上 | `source/05-tests-and-dependencies.md` | 同上 |
| `src/self-test.js` | 同上 | `source/05-tests-and-dependencies.md` | 同上（同卷） |
| `src/sticker-lib.js` | 同上 | `source/05-tests-and-dependencies.md` | 同上（同卷） |

> ⚠️ **重要**：「分卷 SHA256」是**整个 Markdown 分卷文件**的哈希（含多个源文件、标题、元数据、围栏）。它不是任何单个源文件的哈希，**不能与原始 SHA256 直接比较**。
> 要验证「导出的源码是否忠实于原文件」，方法：去掉 Markdown 包装 + 反向替换占位符 → 应还原为原始内容 → 哈希等于原始 SHA256。
> 由于脱敏替换不可逆（不知道原值），**无法从导出副本反推出原始文件哈希** —— 这是脱敏的固有代价，已在设计上接受。

## 2.6 为什么仍然提供原始 SHA256

- SHA256 是单向哈希，**不能从哈希反推原文**
- 提供原始哈希的价值：核验者若有原始文件，可确认"这份导出对应哪个版本"
- 因此提供原始 SHA256 **不构成泄漏**

---

# 第三部分：合规声明

```
✅ 未修改任何项目原文件（12/12 核心源码 SHA256 前后一致）
✅ 未导出任何真实记忆正文
✅ 未导出任何 QQ 账号 / 群号 / 真实姓名（导出副本已替换为占位符）
✅ 未导出任何 Token / 密钥 / 密码 / Cookie / 证书
✅ 未导出任何 Session 历史 / 聊天记录
✅ 未导出任何历史备份包
✅ 导出副本的脱敏未改变行数与行序
✅ 原始哈希与导出哈希分别记录，未混淆
```
