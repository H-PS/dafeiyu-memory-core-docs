# 仓库基线记录（Repository Baselines）

采集时间：2026-11-06 08:1x（Asia/Shanghai）
采集方式：主代理直接执行 `git rev-parse` / `Get-FileHash` / `Get-ChildItem`，非子代理转述。

---

## 1. 仓库清单与状态

| 仓库 | 绝对路径 | 类型 | HEAD | 分支 | 工作区 | 本任务访问 |
|---|---|---|---|---|---|---|
| CMB v0.1 Prototype | `E:\[PROJECT]\CMB-v0.1-prototype` | 非 git（隔离目录） | — | — | — | 只读审查 |
| billion-context | `E:\[PROJECT]\refs\billion-context` | git | `0aace2f203b2da9065fd53c63f4be0c0ab03c90f` | （浅克隆） | 干净（porcelain 输出 0 行） | 只读 |
| QQ Bridge | `D:\qq-bridge` | 非 git | — | — | — | 只读 |
| Memory Core R4.1 | `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design` | 非 git | — | — | — | 只读 |
| DSH | `D:\dsh\resources\app.asar.unpacked\dsh` | asar 打包 | — | — | — | 部分可读 |

### 路径变更警告（重要）

任务书指定 `C:\Users\[USER]\Desktop\work\R4.1-memory-bank-input` **不存在**。
实际路径为 `C:\Users\[USER]\Desktop\work\Memory-Core-R4.1-Design`。
同目录下另有备份 `Memory-Core-R4.1-Design.backup-20261104-192558`（本任务不使用）。
→ 此前第三轮审查报告引用的路径已失效，但文档内容若一致则结论仍可复用（待 Agent C 核对文件哈希）。

### DSH 可读性

`D:\dsh\resources\app.asar` 为单文件（213,202,918 字节）。
`app.asar.unpacked\dsh\` 存在但仅含 `node_modules`。
→ **DSH 宿主核心代码无法直接读取**。涉及 DSH 身份的结论只能标 UNVERIFIED / STATIC_INFERENCE（依据 QQ Bridge 侧的调用证据）。
运行时上下文声明 DSH checkout 位于 `D:\dsh\resources\app.asar\dsh\`，但该目录在文件系统中不存在（asar 内部虚拟路径）。

---

## 2. 运行环境

| 项 | 值 |
|---|---|
| Node.js | v24.21.0 |
| 架构 | AMD64 |
| OS | Windows |
| CMB 依赖 | `dependencies: {}`（零第三方依赖，ADR-008） |
| CPU / RAM | **无法采集**（`Get-CimInstance` 返回 PermissionDenied）→ 性能复核中机器配置标 UNVERIFIED |

---

## 3. CMB 源码基线（16 个 TS 文件，2,014 行）

| SHA-256（前16位） | 行数 | 文件 |
|---|---|---|
| B445C6F83D92B278 | 99 | `src/evidence/evidence-sink.ts` |
| A1BA4701AD6549A2 | 357 | `src/engine/reconciliation-engine.ts` |
| 3B0ABBFC74D58F64 | 203 | `src/adapters/bc-source.ts` |
| 84D0ED770AF18055 | 258 | `src/ledger/bridge-ledger.ts` |
| 45E9755A3D2D22E2 | 153 | `src/engine/source-resolver.ts` |
| DACD357ED9C0C2AA | 163 | `src/core/domain.ts` |
| 470071BDBD863DA3 | 124 | `src/identity/host-identity.ts` |
| 7F5A4D334D17CDCA | 114 | `src/config/config.ts` |
| 6D21C8D0014E8765 | 83 | `src/engine/integrity-validator.ts` |
| 9F06AC64A54C3745 | 72 | `src/evidence/evidence-adapter.ts` |
| EA16C92C7086692A | 60 | `src/cli/cmb.ts` |
| CF9CD3D68BF3C0F6 | 89 | `src/summary/summary-artifact.ts` |
| 0E29B3D94F9FA0BB | 86 | `src/observability/observability.ts` |
| 77CD6DB59D872E0A | 79 | `src/adapters/mock-source.ts` |
| 9A1B2C0097CE679C | 35 | `src/source/source-store.ts` |
| A2A4499EE14C85A4 | 39 | `src/core/errors.ts` |

测试文件：13 个（`tests/**/*.ts`）。

---

## 4. 上游关键文件基线

### billion-context @ 0aace2f

| SHA-256（前16位） | 行数 | 文件 |
|---|---|---|
| 348CD085A1416A06 | 498 | `src/storage/schema.ts` |
| 58A8162318DF4436 | 3,092 | `src/storage/store.ts` |
| 8DEBCE0ABB3D91E1 | 665 | `src/storage/ingest.ts` |
| 65155A3D57422667 | 1,943 | `src/session.ts` |
| 1A9C74E06854ABCB | 3,994 | `src/persist.ts` |
| 976DB3FE453D2722 | 745 | `kernel/src/types.ts` |
| D3C9324EB09B822B | 176 | `kernel/src/content-store.ts` |

### Memory Core R4.1-Design

| SHA-256（前16位） | 行数 | 文件 |
|---|---|---|
| F4D087C3AADAC774 | 36 | `contracts/schemas/evidence.schema.json` |
| A204CE745D31878C | 293 | `contracts/schemas/memory-unit.schema.json` |
| A83916181B8CF75F | **465** | `DD-AUTHORITY-V2.md` |
| 8657E5204E9D3B6E | **1,034** | `13-Evidence与EmotionContext.md` |
| FD06CBCD3F1971B8 | **451** | `04-权限与治理规范.md` |

> **行数统计更正**（2026-11-06，由 Agent C 指出、主代理复核）：
> 初版使用 `Measure-Object -Line`，该命令**不统计空行**，导致行数系统性偏低。
> 正确方法为 `[System.IO.File]::ReadAllLines()` 或 `Get-Content | .Count`。
> 三个文件的正确行数已在上表更正。**哈希值不受影响，结论不受影响。**

---

## 5. 与 2026-11-04 审查基线的对比

| 项 | 2026-11-04（第三轮） | 2026-11-06（本轮） | 一致？ |
|---|---|---|---|
| billion-context HEAD | `0aace2f` | `0aace2f` | ✅ 一致 |
| Memory Core 路径 | `R4.1-memory-bank-input` | `Memory-Core-R4.1-Design` | ❌ 已变更 |
| CMB 存在性 | 尚未创建 | 16 文件 2,014 行 | 新增 |
| DSH 可读性 | 未核查 | asar 打包不可读 | 新增发现 |

**结论**：上游数据源基线一致，Memory Core 需按新路径重新核对（哈希比对见 Agent C 报告）。
