# 冻结说明 · Memory V2.4.2 修订版

- **版本标识**：`v2.4.2-revised-final`
- **冻结时间**：2026-10-31 18:00
- **性质**：**纯设计冻结**。零代码实施、零配置变更、零数据变更
- **用途**：技术参考基线。后续开发在此之上继续

---

# 1. 这份东西是什么

一套为 QQ Bridge 记忆系统设计的**完整改造方案**，经过三轮迭代：

| 轮次 | 产出 | 关键动作 |
|---|---|---|
| 原设计 V2.4.2 | `00`~`09`（10 份） | 补执行模型（写队列/WAL/effects） |
| 第一轮修订 | 加入 R1~R9 | 拆三层（A/P/S）、拆双版本、重划 M0 |
| 第二轮修订 | 加入 R10~R16 | per-key 版本、A 层 extra、metaRev、双写协议 |
| 最终审核 | `MEMORY-V2.4.2-FINAL-DESIGN-REVIEW-PACK.md` | 8 项独立交叉核查，发现 3 个 P1 |

**没有写一行代码**。所有"修复"都是设计承诺，不是已实现的事实。

---

# 2. 十五份文档清单

| 文件 | 行数 | 作用 | 什么时候看 |
|---|---|---|---|
| `00-SUMMARY.md` | 591 | 总纲：三层架构、三版本矩阵、M0/M1 划分 | **首先看** |
| `01-MAPPING.md` | 147 | F-01~F-26 + N-01~N-07 缺陷映射 | 想知道"为什么要改" |
| `02-WRITE-CONCURRENCY.md` | 503 | 8 类 op 管线、10 组并发、队列背压 | 写代码时对照 |
| `03-AB-CONSISTENCY-WAL.md` | 417 | A/P/S 一致性、WAL、5 个崩溃点、恢复 | 实现 WAL 时必读 |
| `04-LIFECYCLE-TOMBSTONE.md` | 208 | 生命周期、tombstone、级联、suppression | 实现删除时必读 |
| `05-SHARE-SECURITY.md` | 234 | ShareRequest、审批 9 步、群隔离 | M1 做分享时 |
| `06-EMBEDDING-AUDIT-BACKUP.md` | 291 | embedding 三态/六态、两级审计、备份恢复 | M0 实现 embed/审计 |
| `07-LOCK-AND-STARTUP.md` | 328 | 三锁职责、启动六态、心跳、GRACE | 实现锁时必读 |
| `08-DECISIONS-REQUIRED.md` | 130 | D-01~D-24（含 4 项新增） | **拍板前看** |
| `09-ACCEPTANCE-CRITERIA.md` | 223 | 95 条验收（标 M0/M1） | 测试时对照 |
| `10-M0-ACCEPTANCE.md` | 244 | M0 独立验收 73 条 + Checklist | M0 上线前 |
| `11-SOURCE-EVIDENCE.md` | 261 | 74 条源码证据（文件:行号） | 质疑某个结论时 |
| `12-CHANGELOG.md` | 405 | R1~R16 逐项改动记录 | 想知道"改了什么" |
| `FINAL-REVIEW.md` | 340 | 15+7 问回答、未解决问题 | 评审时 |
| `MEMORY-V2.4.2-FINAL-DESIGN-REVIEW-PACK.md` | 1,226 | **审核包**：8 项独立核查 + 10 个问题 + 修复优先级 | **接手前必读** |

---

# 3. 架构一句话

```
A 层  memory-store.jsonl     事实源（不可重建）
P 层  memory-projection.json 记忆投影（可从 A 重建，per-key convVersion）
S 层  social-v2.json         运行状态（18 类字段，不可重建）
                             + impressionStats（互动统计，不可重建）

三版本：rev（全量）/ contentRev（内容）/ metaRev（元数据）
写路径：只经 MemoryService WriteQueue
读路径：P 优先，S 降级
```

---

# 4. 后续开发的起点（最重要）

## 4.1 三个 P1，建议先修（不改架构）

| # | 问题 | 修法 | 代价 |
|---|---|---|---|
| **S-1** | `interactionCount` 存 `social-v2.json`，被 43 处整文件重写覆盖 | **拆出独立文件** `memory-impression-stats.json` | 新增 1 个 state 文件 |
| **S-2** | `e3 S_stats` 与 `e4 S_compat` 写同一文件却是两个 effect | 同上（拆文件后自然单文件单 effect） | 0（与 S-1 同批） |
| **S-3** | `clear` 后 P 层会话语义未定义 | 明确"三字段置空 + convVersion+1，不删条目" | 纯文档 + 几行代码 |

**为什么必须先修**：S-1/S-2 会让提示词里的"互动 N 次"不可靠（`bridge.js:9199` 显示该字段），用户能直接感知。S-3 会导致 clear 后 P 文件持续增长 + 提醒词出现空壳。

## 4.2 三个待批准决策（阻塞 M0 编码）

| # | 决策 | 推荐 | 未批准时默认 |
|---|---|---|---|
| **D-22** | `subjectId` 是否可编辑 | 允许（走 `edit_subject`） | 禁止，M0 关闭该 op |
| **D-23** | `reset` 是否清 A 层 | 不清（只重置会话状态） | 不清 |
| **D-21** | `clear` 屏障语义 | 方案 A（屏障） | 方案 B（V1 行为） |

## 4.3 建议实施顺序

```
Step 0  拍板 D-22 / D-23（D-21 可延后）
Step 1  修 S-1/S-2/S-3（stats 拆文件 + clear 语义）
Step 2  M0 编码（按 00 §8.1 的 20 项缺陷）
Step 3  M0 验收（10-M0-ACCEPTANCE.md 的 73 条 + 8 条红线）
Step 4  M1（ShareRequest / 六态 / 备份 envelope / 淘汰联动）
```

**M0 关键路径**：`atomicWriteLines → 数据模型 → WAL → WriteQueue → tombstone → P 层 → 三态 embedding`。后四项可并行。

---

# 5. 不要重复做的事

```
✗ 不要再重新审计 V1 源码 —— 11-SOURCE-EVIDENCE.md 有 74 条带行号的证据
✗ 不要再纠结"要不要三层" —— 18 个 S 层字段不可重建，这是硬事实（03 §1.2）
✗ 不要再纠结"要不要双版本" —— 单版本会让 pending edit 被 embedding 刷新打断（02 §4.2 有推演）
✗ 不要照抄 design 当已实现 —— M0 未写一行代码
✗ 不要把 01-MAPPING 的"关闭"理解为"已修复" —— 它是"设计已覆盖"
```

---

# 6. 源码基线（设计行号的锚点）

**设计文档里所有 `bridge.js:6731` 这类行号，都绑定下表的代码状态。**

若后续修改了这些文件，行号会失效，需重新核对 `11-SOURCE-EVIDENCE.md`。

| 文件 | 字节 | 行数 | SHA256（前16） | 最后修改 |
|---|---|---|---|---|
| `src/memory-vector.js` | 15,788 | 380 | `1d6dc555920c7edc` | 10-31 04:19:45 |
| `src/bridge.js` | 701,164 | 12,482 | `47bdeb20e2416d72` | 10-31 04:33:27 |
| `src/mcp-snowluma-safe.js` | 86,093 | 1,465 | `803a05cc8c6c8483` | 10-31 04:21:58 |
| `src/qq-model-view.js` | 5,855 | 134 | `5beddd47897c98b2` | — |
| `src/mcp-diary.js` | 4,097 | 113 | `0010dd1e7ab643f4` | — |
| `src/mcp-host-server.js` | 4,597 | 93 | `68cc06ecb8f45aee` | — |
| `src/slang-learner.js` | 16,534 | 382 | `8d51fa76144f068c` | — |
| `src/sticker-lib.js` | 15,260 | 302 | `58d3c3084e524690` | — |
| `src/token-ledger.js` | 39,568 | 958 | `4d9141ce36cf0376` | — |
| `src/self-test.js` | 5,005 | 118 | `6fc5f2201d43050e` | — |
| `package.json` | 3,774 | 72 | `b6a395123fcc6550` | — |
| `config.example.json` | 7,449 | 279 | `e1346f09b2d4f4be` | — |

**验证方法**：
```bash
python -c "import hashlib;print(hashlib.sha256(open(r'D:/qq-bridge/src/bridge.js','rb').read()).hexdigest())"
# 应为 47bdeb20e2416d72...（完整值见下）
```

**完整 SHA256**：
```
memory-vector.js      1d6dc555920c7edcf948701faf54980efb54bf49eef03c99f48b456c01af114a
bridge.js             47bdeb20e2416d72bdec70fd07db719343441259da74e8a6ee66013b1f4d2e8a
mcp-snowluma-safe.js  803a05cc8c6c8483f2c78b84c2857065f6b1eb6bc3d68af86127c78506b7c622
```

**源码快照位置**（若需对照）：
- `D:\qq-bridge备份-20261030\p0-final-after.zip`（4 个核心文件，04:44，与上表一致）
- `D:\qq-bridge-review-20261031\source\`（脱敏分卷，5 卷）

---

# 7. 当前状态快照（冻结时的真实数据）

| 项 | 值 |
|---|---|
| A 层 `memory-vectors.jsonl` | **0 字节**（空） |
| B/S 层 `social-v2.json` | 101,136 字节，2 个会话，记忆三字段全空 |
| 记忆总数 | **0 条** |
| `state/bridge.lock` | 5 字节（内容 `12412`，**僵尸锁**，PID 已不存在，不阻碍启动） |
| 服务状态 | 全部停止 |
| DSH 版本 | 0.2.1-rc.2 |

**含义**：M0 迁移时**无数据可迁**（这是最好的时机）。

---

# 8. 与已有备份的关系

| 备份 | 内容 | 与本包的关系 |
|---|---|---|
| `p0-final-after.zip`（04:44） | 4 个源码文件 | 源码基线，与本包 §6 一致 |
| `D:\qq-bridge-review-20261031\`（14:00） | 审查报告 + 5 卷脱敏源码 | 上一轮的审查证据 |
| `D:\memory-v242-independent-review\`（17:40） | 独立审查报告 | 第一轮核查 |
| **本包** `v25-memory-v242-design.zip` | 15 份设计 + 本说明 | **设计冻结基线** |

---

# 9. 一句话交接

> 这是一套**未实施的记忆系统改造设计**。三层架构、三版本、WAL、tombstone、M0 边界都已定义清楚，74 条源码证据可追溯。
> 接手前先读 `MEMORY-V2.4.2-FINAL-DESIGN-REVIEW-PACK.md` 的 §8（8 项核查）和 §9（10 个问题）。
> 开工前先修 3 个 P1（stats 拆文件、clear 语义、删除幂等），再拍板 D-22/D-23。
> **当前记忆库为空，是最好的迁移时机。**

---

*本说明为冻结文档，不建议修改。若需变更设计，请改对应的 00~12，并更新 `12-CHANGELOG.md`。*
