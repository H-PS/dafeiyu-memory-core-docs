# 当前基线说明 · 03_CURRENT_DESIGN

> **本目录无 v4.x 文件可放置**——本地不存在任何 v4.5/v4.6（判定 L3，依据见 `00_README/LOCAL_MATERIAL_INVENTORY.md`）。

## 1. 本地实际存在的"当前设计"

R4.1 · CR-08A 工作流的**本地落点**就是 `[workspace]/release/[r4-integration]\` 发布包本身（已整体复制至本包 `01_BASELINE/R4-Integration/`）。该包的内部版本为 **R4.1 定点修订版**（2026-11-02 23:06 定稿）：

- 00~08 主文档（基线/状态/计划/综合/变更/冲突登记/接口地图/评审汇总/验证报告）；
- `revised/adjudication-supplement.md`（K/X/W/CR 系增补层，R4.1 修订版）；
- `adjudication-ledger.md`（六轮裁决台账——**后于 ZIP 生成，未入 manifest/SHA256SUMS/ZIP**，属已知历史遗留）。

## 2. 历次修订材料（与 CR-08A 相关的真实历史）

| 材料 | 位置（本包内） | 性质 |
|---|---|---|
| R4.1 修订明细 | 01_BASELINE/R4-Integration/R4-adjudication-materials-change-log.md | ChatGPT 定向审核意见的落地记录（2026-11-02） |
| 勘误 | 01_BASELINE/R4-Integration/revised/errata-20261102.md | 同期勘误 |
| v3.3 → v4.6 | **无本地文件** | 仅对话历史（见 04_LATEST_V46_CANDIDATE/delta-spec 版本链） |

## 3. 与 04 目录的关系

- 03 目录证明"本地是什么"（R4.1 基线，无 v4.x）；
- 04 目录承载"对话里最新到哪了"（v4.6 收口候选）；
- **两者之间不存在中间版本**——不存在"部分修订已落盘"状态（L2 已被排除：backups/ 只含 2026-11-02 的 3 份 R4.1 修订前副本，与 CR-08A 无关）。
