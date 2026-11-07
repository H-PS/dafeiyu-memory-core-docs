# 四项遗留裁决材料（14-USER-DECISIONS）

采集：2026-11-06 ｜ 主代理综合 Agent B/C/E/G 证据 + 自主核验

> **推荐不等于已批准。** 以下每项给出事实、证据、方案、风险、推荐，但最终由用户裁决。

---

## 编号映射（重要）

本文件使用 D-1…D-4 编号（重编），与历史文档的对应关系：

| 本文件 | 第三轮报告 | 议题 |
|---|---|---|
| **D-1** | B-1 / D-1 | Evidence Schema 版本冲突 |
| **D-2** | D-6 | tenant / bot 定义 |
| **D-3** | D-7 | 时间戳可空 |
| **D-4** | D-8 | Block 删除窗口 |

Agent F 在 `15-REAL-DATA-EXPERIMENT-PLAN.md §4.1` 中新增的决策项从 **D-10** 起，与本文件不重叠。

---

---

## D-1：Evidence Schema 版本冲突（原 B-1）

### 事实

| 项 | 事实 | 证据 |
|---|---|---|
| `evidence.schema.json` | 36 行，`additionalProperties: false`，required 含 `scope`/`recorded_at`，**缺 `original_text`/`speaker_id`** | `contracts/schemas/evidence.schema.json` |
| 权威文档 `DD-AUTHORITY-V2.md §3.2` | required 含 `original_text`/`speaker_id`/`visibility`，**无 `scope`/`recorded_at`/`tenant_id`** | `DD-AUTHORITY-V2.md:208-223` |
| `13-Evidence与EmotionContext.md:255` | `content_hash` **MUST** 为 `original_text` 的 sha256 | 已核验 |
| 权威序声明 | `DD-AUTHORITY-V2.md:5` 声明其含"契约"；`13-…md:1028`"与 V2 §3.2 冲突以 V2 为准" | Agent C 核验 |
| Memory Core 实现 | **无**（纯设计仓库，无 `src/`） | Agent C 核验 |
| M03/M04 | **未实现**，`api-contracts.md:89,93` 标 `[FUTURE]` | Agent C 核验 |
| 内部审查假绿 | `reviews/CROSS-DOCUMENT-CONSISTENCY-REPORT.md:39` 把 Evidence 判 PASS，但比对双方是 `01 §7.8` ↔ schema（V2 §3.2 从未进入比对） | Agent C 核验 |
| 备份物证 | schema 与备份逐字节一致 → V2 升级改文档但零改动 schema | Agent C 核验 |

### 三方字段差异统计

- 比对 32 字段
- `NO_DIFF`：4
- **blocking：12**（`evidence_id`、`source_type`、`original_text`、`recorded_at`、`timestamp`、`speaker_id`、`bot_id`、`group_id`、`scope`、`visibility`、`produced_by_run`、`produced_by_blocks`）
- 成因：CMB 有而 schema 无 8 个；schema 必填而 CMB 无 2 个；值不合法 2 个

### 方案比较

| 方案 | 做法 | 优点 | 缺点 | 阻塞解除？ |
|---|---|---|---|---|
| **A. CMB 完全适配现有 schema** | 删 `original_text`/`speaker_id`/`bot_id`/`group_id`，加 `scope`/`recorded_at` | 无上游改动 | **不可能**：删 `original_text` 等于放弃原句，违反 13 §7.3 D-01 与任务书 §15-9 | ❌ |
| **B. Memory Core 修订 schema 到 V2 标准** | 按 `DD-AUTHORITY-V2 §3.2` 重写 schema | 与权威文档一致；CMB 几乎不用改 | 需改上游；无迁移机制（无 version 字段、无 migrations 目录） | ✅ |
| **C. 版本化中间 DTO** | CMB 产出 V1 DTO，Adapter 转 V2 | 隔离演进 | 增加一层转换；仍需先定哪版权威 | 部分 |
| **D. 分阶段迁移，保留旧版读取** | 先加字段（可选），再改 required | 平滑 | 需上游支持版本共存 | 部分 |
| **E. 承认 schema 为权威，CMB 另建"扩展证据"层** | 原句存 CMB 自有库，只向 MC 传引用 | 不动上游 | 违反 D-02"优先引用而非复制"？需重新解释 | 部分 |

### 需用户裁决的具体问题

1. **`evidence.schema.json` 与 `DD-AUTHORITY-V2 §3.2`，哪个是权威？**
   - 若 V2 权威 → schema 是 bug，需修
   - 若 schema 权威 → 文档错了，且 `original_text` 不能存（但 13 号文档说 MUST 存）
2. **`original_text` 必须存在吗？** 13 §7.3 D-01 说 MUST。若存，schema 必须加字段。
3. **`bot_id`/`group_id` 是否应成为 Evidence 的正式维度？** 两者都不在 schema 也不在 V2 §3.2。
4. **`source_type` 用哪套枚举？** 三方三套（CMB `chat` / schema 12 值 / DOC 5 值）。

### 推荐

**推荐方案 B**（Memory Core 修订 schema 到 V2 标准），理由：
- V2 是文档声明的权威（`DD-AUTHORITY-V2.md:5` + `13-…md:1028`）
- schema 缺 `original_text` 与 `speaker_id` 且 `additionalProperties:false`，等于**禁止存原句**，与整个 13 号文档的核心要求矛盾
- 内部审查"假绿"证明这是升级遗漏，非有意设计
- 但：**必须先建立迁移机制**（version 字段 + migrations 目录），否则修改无兼容性保障

**若用户认为不宜改上游**，则退为方案 C（版本化中间 DTO），但需明确"CMB 侧放弃 `original_text` 落地"的语义后果。

---

## D-2：tenant / bot 定义（原 D-6）

### 事实

| 身份 | 上游有吗 | 宿主有吗 | CMB 怎么拿 | 可信度 |
|---|---|---|---|---|
| `tenant_id` | ❌ | ❌（QQ Bridge 全仓 0 次） | 部署常量 | 手工配置，可从没被验证 |
| `bot_id` | ❌ | ⚠️ 仅 `event.self_id`（Bot 自身 QQ） | 部署常量 | 同上 |
| `group_id` | ❌ | ✅ `convKey` 里有群号 | 手工 Map | 取决于 Map 正确性 |
| `session_id` | ✅ | ✅ | 真实值 | 会轮换 |
| `speaker_id` | ❌ | ✅ `event.user_id` | **恒 null** | 不可用 |

### 证据
- `bc-source.ts:39,41,42`：`tenant_id`/`bot_id`/`group_id` 恒 null，`session_type` 硬编码 `"unknown"`
- `D:\qq-bridge\src\bridge.js:1220`：`convKey = (kind, id) => \`${kind}:${id}\``
- `bridge.js:11238`：`const isOwner = String(event.user_id) === String(cfg.ownerQQ ?? '')` → user_id 存在于事件中
- `D:\qq-bridge` 全仓检索 `tenant_id`/`tenantId` = **0 次**

### 需用户裁决

1. **`tenant_id` 对单人部署有意义吗？** 若只有一个租户，是否可用固定常量 `"default"`？
2. **`bot_id` 用 `self_id`（Bot 的 QQ 号）还是部署常量？**
3. **`group_id` 的权威来源是哪？** QQ Bridge 的 `convKey` 能否作为唯一真相？
4. **五标识只能"三实两缺"（session/group/speaker 可得，tenant/bot 不可得），是否接受？**

### 推荐

- `tenant_id`：单人部署用固定常量 `"default"`，并在 Evidence 中标注 `verification_level: "DEPLOYMENT_CONSTANT"`
- `bot_id`：用 `self_id`（它是真实可得的），但需 QQ Bridge 写入某处供 CMB 读取
- **必须做**：`Session.tenant_id` 若非 null 而与部署常量不一致，应拒绝（当前 CMB 完全不读这个字段）

---

## D-3：时间戳可空（原 D-7）

### 事实

| 时间字段 | 上游 | 语义 | CMB 用法 |
|---|---|---|---|
| `messages.ts` | ✅ | **二义**：首见戳 或 批采样时刻（无列可区分） | 读入 `SourceMessage.timestamp` |
| `Evidence.timestamp` | — | — | **恒 null** |
| `blocks.created_at` | ✅ | 块创建时间 | 读入 `CompressionBlock.created_at` |
| `Evidence.recorded_at` | — | schema required | **CMB 没有** |

### 证据
- `store.ts:1031-1038`：`batchTs = Date.now()`，无 first-sight stamp 时用它
- `MessageRow.ts` 注释："Ingest input: taken from the message's first-sight stamp when present, else the per-batch sampled now"
- 13 号文档 §7.2 字段纪律第 2 条：`timestamp` 未知 **MUST** 为 null，**MUST NOT** 填推测时间

### 需用户裁决

1. **接受 Evidence 无原始时间吗？** 下游 M03/M04 若无时间，能否做时序推理？
2. **`recorded_at`（记录时间）可以用桥接观察时间填充吗？** 它是 schema required。
3. **`messages.ts` 能否用作"近似时间"？** 若可以，偏差可能达数小时。

### 推荐

- 短期：接受 `timestamp = null`，同时补 `recorded_at` = 桥接运行时间（这满足 schema 的 required）
- 中期：向 billion-context 请求区分"首见戳"与"批采样时刻"（加一列标记）
- **不要**用 `messages.ts` 冒充原始时间

---

## D-4：Block 删除窗口（原 D-8）

### 事实

| 项 | 事实 | 证据 |
|---|---|---|
| Block 墓碑 | **无** | 上游 `blocks` 表无 `retired_at` 类列 |
| Block 删除 | 物理 `DELETE`（不在当轮 `state.blocks` 即删） | `store.ts:1077-1087` |
| Block 历史 | 仅 `orphans_json`（文本）+ `metadata_json.preCompactionArchive`（id/时间/原因） | Agent B 核验 |
| 消息墓碑 | 有 `retired_at`，可复活 | `store.ts:925,949` |
| 媒体释放 | tombstone 时立即 | `store.ts:951-955` |
| Session 删除 | 物理删除全部 | `store.ts:1694-1696` |

### 影响

- CMB 若停机超过 N 轮压缩，**错过的 Block 永不可重建**
- 无法知道"历史上曾有哪些 Block"
- 无法知道"某个 Block 覆盖关系如何演化"

### 需用户裁决

1. **接受"错过即永久丢失"吗？**
2. **是否要求 billion-context 提供 Block 生命周期事件或保留策略？**（UPSTREAM_CHANGE_REQUIRED）
3. **CMB 是否需要"高频扫描"来缩小窗口？** 代价是持续 I/O 压力。
4. **是否接受用 `preCompactionArchive`（若有）作为部分补偿？**

### 推荐

- 短期：接受限制，但**必须在扫描间隔与丢失风险间明确 SLA**（例如"每 5 分钟扫一次，最多丢失 5 分钟内的块"）
- 中期：请求上游提供 block 墓碑或至少保留 N 天历史
- **不推荐**高频扫描（会与上游争 I/O，且 Block 删除是"当轮不在就删"，高频也未必能捕获）

---

## 汇总：四项裁决的相互依赖

```
D-1（Schema 权威） ──┬──> 决定 Evidence 字段集
                     └──> 影响 D-2（tenant/bot 是否有地方存）
                     └──> 影响 D-3（recorded_at 是否必须）

D-4（Block 窗口） ────> 决定 CMB 的数据完整性上限
```

**建议裁决顺序**：D-1 → D-2 → D-3 → D-4
（先定契约，再定字段归属，再定时间语义，最后定数据窗口）
