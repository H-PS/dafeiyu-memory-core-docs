# 04 · MANIFEST（导出清单与哈希）

- 生成时间：2026-10-31 15:00（重整为 5 卷）
- 项目根：`D://qq-bridge`（无 Git 仓库）
- 导出根：`D://qq-bridge-review-20261031`
- 源码组织：**5 个分卷**，每卷含多个完整源文件，用一级标题（`# 源码文件：<路径>`）分隔。
- 脱敏：**全部源文件均已脱敏**（4 个 QQ 号 + 真实姓名 → 占位符）。脱敏只改导出副本，**行数与行序不变**，因此报告中的行号可直接在导出源码中定位。
- 「导出 SHA256」= 该源码所在 **Markdown 分卷文件**的整体哈希；单文件校验请用「原始 SHA256」对照项目原文件。

## 1. 分卷总览

| # | 分卷 | 大小(字节) | 内含源文件 | 导出 SHA256 |
|---|---|---|---|---|
| 1 | `source/01-memory-core.md` | 19,117 | memory-vector.js | `f8c33d9f6522c96e00d59a22079def4467da7f3e49f3b9ac5688f32b953c2bde` |
| 2 | `source/02-bridge.md` | 706,443 | bridge.js | `54a8b1ccfad7cd93d33219803ddeeed9511652695ae6c63838c398d3595183ca` |
| 3 | `source/03-mcp-social.md` | 105,151 | mcp-snowluma-safe.js + qq-model-view.js + mcp-diary.js + mcp-host-server.js | `24376a4ffda49eaf88e619387ce315be8adc7c338c8546ed3ff38700c847b701` |
| 4 | `source/04-infrastructure.md` | 54,772 | token-ledger.js + package.json + config.example.json | `70b896f9e6e09a8bfa50f85813c46be8e7d7433c38f507237e540ef3ce569643` |
| 5 | `source/05-tests-and-dependencies.md` | 40,801 | slang-learner.js + self-test.js + sticker-lib.js | `af59f58df29b42577834c62a4626365111e47368b17398d35812609d98f35965` |

## 2. 逐文件清单

| # | 原始相对路径 | 大小(字节) | 行数 | 原始 SHA256 | 所在分卷 | 行段 | 脱敏 |
|---|---|---|---|---|---|---|---|
| 1 | `src/memory-vector.js` | 15,788 | 380 | `1d6dc555920c7edcf948701faf54980efb54bf49eef03c99f48b456c01af114a` | `source/01-memory-core.md` | 全文 | 是 |
| 2 | `src/bridge.js` | 701,164 | 12482 | `47bdeb20e2416d72bdec70fd07db719343441259da74e8a6ee66013b1f4d2e8a` | `source/02-bridge.md` | 1-4200;4201-8400;8401-12482 | 是 |
| 3 | `src/mcp-snowluma-safe.js` | 86,093 | 1465 | `803a05cc8c6c8483f2c78b84c2857065f6b1eb6bc3d68af86127c78506b7c622` | `source/03-mcp-social.md` | 全文 | 是 |
| 4 | `src/qq-model-view.js` | 5,855 | 134 | `5beddd47897c98b2485d497e3b6c54d9226f66c60ef2606cb729dced6770982e` | `source/03-mcp-social.md` | 全文 | 是 |
| 5 | `src/mcp-diary.js` | 4,097 | 113 | `0010dd1e7ab643f455c348fbae03760343a8e9820a3e6743294ef5d98db303a8` | `source/03-mcp-social.md` | 全文 | 是 |
| 6 | `src/mcp-host-server.js` | 4,597 | 93 | `68cc06ecb8f45aeeacb4185a3f06120b283ac997b06dec042d70a98868373ce0` | `source/03-mcp-social.md` | 全文 | 是 |
| 7 | `src/token-ledger.js` | 39,568 | 958 | `4d9141ce36cf037671fd65529f1aef2cbb7f732566749b0706ceceb0e7679507` | `source/04-infrastructure.md` | 全文 | 是 |
| 8 | `package.json` | 3,774 | 72 | `b6a395123fcc655022c1e0ebebfc0fa52fd3664a408ce67ab746c09ee8baeadd` | `source/04-infrastructure.md` | 全文 | 是 |
| 9 | `config.example.json` | 7,449 | 279 | `e1346f09b2d4f4bef32c316616a0486867f0233065d26044fad6af5fcee87435` | `source/04-infrastructure.md` | 全文 | 是 |
| 10 | `src/slang-learner.js` | 16,534 | 382 | `8d51fa76144f068c8299112a10642d5e840615a56992e786c8696e9e1b296b60` | `source/05-tests-and-dependencies.md` | 全文 | 是 |
| 11 | `src/self-test.js` | 5,005 | 118 | `6fc5f2201d43050e0c59de0597b6526e78a3b7e77a1d79f9a7485d0a3829c770` | `source/05-tests-and-dependencies.md` | 全文 | 是 |
| 12 | `src/sticker-lib.js` | 15,260 | 302 | `58d3c3084e5246901ded36e8e0c4698862351a7b19550bd0553760706cee4f0e` | `source/05-tests-and-dependencies.md` | 全文 | 是 |

## 3. 分卷内容说明

- `source/01-memory-core.md`
  **A 层记忆核心**：`src/memory-vector.js`（380 行，全文）+ 数据模型/存储格式/检索算法/函数清单索引
- `source/02-bridge.md`
  **桥接主程序**：`src/bridge.js`（12,482 行，分 3 段连续导出：1-4200 / 4201-8400 / 8401-12482，无重叠无遗漏）+ 全部记忆 HTTP 端点、鉴权三要素、key 流转链索引
- `source/03-mcp-social.md`
  **MCP 工具层 + SocialV2**：`mcp-snowluma-safe.js`(1,465) + `qq-model-view.js`(134) + `mcp-diary.js`(113) + `mcp-host-server.js`(93)；附 MCP 记忆工具清单、SocialV2 逻辑位置、B 层三字段语义
- `source/04-infrastructure.md`
  **基础设施**：`token-ledger.js`(958) + `package.json`(72) + `config.example.json`(279)；附锁/原子写/备份的完整函数与行号索引、配置要点
- `source/05-tests-and-dependencies.md`
  **测试与依赖**：`slang-learner.js`(382) + `self-test.js`(118) + `sticker-lib.js`(302)；附写入/检索完整调用链、写入口穷举、async 函数清单

## 4. 未导出的依赖及原因

| 未导出项 | 原因 |
|---|---|
| `node_modules/` | 第三方依赖，非本项目代码 |
| `state/memory-vectors.jsonl`（0 字节） | 真实记忆数据 |
| `state/social-v2.json`（101KB） | 真实会话内容 |
| `state/memory-admin.log` | 含记忆正文（脱敏摘录见 `05-EXCLUSIONS-REDACTIONS.md`） |
| `backups/*.jsonl`、`state/*.pre-*.bak` | 历史记忆正文（101 条 × 3 份） |
| `D://qq-bridge备份-*//` | 历史备份包 |
| `~/.dsh/sessions/**` | C 层会话历史（真实聊天内容） |
| `roles/*.md` | Persona 卡（数据，非记忆代码） |
| `docs/guides/*.md` | V2.1~V2.4.1 设计文档（本次导出对象是代码） |
| `scripts/*.mjs`（约 40 个） | 测试/探测脚本；`seed-group-owner.mjs` 已归档为 `.disabled` |
| `plugins/qq-agent-presets/**`、`dsh/agent-presets/**` | preset 与规则配置 |
| `config.json` 真实值、`.credentials.yaml` | Token/密钥（`config.example.json` 模板已导出） |

## 5. 完整性核验

- 项目核心源码审查前后 SHA256：**12/12 一致**（变化 0 个）
- 源码导出无省略：脚本内 `assert len(redacted_lines)==original_lines` 全部通过
- 行数逐一对齐：`bridge.js` 3 段 = 4200+4200+4082 = 12,482；其余 11 个文件全文导出，代码块行数 = 原始行数
- 代码围栏：4 个反引号（源文件内最长反引号连续为 1，不会提前闭合）
- 分卷内源文件用一级标题 `# 源码文件：<相对路径>` 分隔，`---` 作为分隔线
