# 03 · M0 · 可靠记忆基础（修订版）

> 目标：把设计落地到"能跑、能崩、能恢复、权限正确"的程度。不含智能能力。
> 本版相对上一版：① 新增 `memory-permission.js`（权限模型是 M0 的一部分，不是 M1）；② 提交协议按 01 §4.4 重写；③ 验收扩到 16 项；④ 纳入凭据过渡态。

---

## 1. 范围

### 做

| # | 能力 | 关键点 |
|---|---|---|
| 1 | A/P/S 三层落地 | A 层 JSONL 追加为主；P/S 单文件 JSON |
| 2 | WriteQueue 五阶段管线 | 归一化 → embed（锁外）→ 入队 → 临界区 → 返回 |
| 3 | **per-key 互斥锁 + fencing token** | 真正的串行化（不是"写前重读"） |
| 4 | 三版本控制 | `rev` / `contentRev` / `metaRev`，单调不连续 |
| 5 | 安全删除 + Tombstone | 永久保留、不存正文 |
| 6 | WAL + 12 个崩溃/恢复场景 | 见 12 文档 §1.1.1 |
| 7 | **权限模型** | `memory-permission.js`：classify + canRead + 五维断言 |
| 8 | 数据脱敏 + CRC 校验 | 坏行不静默 |
| 9 | 两级审计 | Tier-1 拒绝式 / Tier-2 补偿 |
| 10 | 投影一致性 | per-key `convVersion`，重建 0 差异 |
| 11 | 索引写入钩子（只写不查） | 覆盖 `docId` / `generation` / `coveredRev` |
| 12 | **凭据过渡态** | token 从"每轮注入"降为"会话首轮一次" |
| 13 | S-1 ~ S-16 修订 | 见 02 文档 |

### 不做

检索路、预算、Retain、Reflect、图结构、向量模型更换（仍用 bge-m3）。

---

## 2. 数据结构变化

### 2.1 A 层

完整 Schema 见 `SCHEMA.md` §1。相对 V2.4.2 的新增字段：

```
kind / fingerprint / origin（必填）/ scope{read,originGroupId,contentClass,classReason,copies}
review{status,...} / assertion{epistemic,sourceType,assertedBy,confidence,evidence}
lifecycle 增 quarantined / duplicateOf / mergedFrom / validFrom / validUntil
embedding{status,attempts,lastError,nextRetryAt}
```

**兼容**：缺 `kind` 按 category 推导；缺 `origin` 填 `{channel:'legacy', at: time}`；缺 `scope` 按"最保守"补 `owner_only` + `uncertain`（历史数据不做批量升级）。

### 2.2 新增文件

```
state/
├── memory-impression-stats.json   S-1
├── memory-quarantine.jsonl        权限失败隔离区（M0 建结构，M2 写入）
├── memory-candidates.jsonl        候选池（M2 写入）
├── memory-index/{meta,ids,vec.f32,bm25,temporal}   M0 建目录 + 写入钩子
└── wal/{pending,committed,aborted}.jsonl
```

---

## 3. 涉及文件

| # | 文件 | 类型 | 行数 |
|---|---|---|---|
| M01 | `src/memory-store.js` | 新增 | 750 |
| M02 | `src/memory-projection.js` | 新增 | 380 |
| M03 | `src/memory-stats.js` | 新增 | 150 |
| M04 | `src/memory-permission.js` | 新增 | 220 |
| M05 | `src/memory-index.js`（写入钩子部分） | 新增 | 约 260 / 520 |
| M14 | `scripts/memory-maintenance.mjs` | 新增 | 320 |

改动：

| 文件 | 改动 |
|---|---|
| `src/memory-vector.js` | +70 / -15（S-11 三处、`atomicRewrite` tmp 名、`loadRows` 坏行上报、`edit` 定位时机、写路径委托） |
| `src/bridge.js` | 约 +260（写路径切换 + 3 个管理端点 + 凭据过渡态） |

---

## 4. 实现约束（code review checklist）

1. **单写者**：A 层只由 `memory-store.js` 写。切换后 `memory-vector.js` 的写函数内部改为 `throw new Error('已迁移到 memory-store')`，迫使暴露漏网调用。
2. **锁**：所有写路径必须 `withKeyLock(key)`；`clear` 用同一把锁；锁实现是 `Map<key, Promise>` 链，不是文件锁。
3. **fencing**：提交前校验 `clearEpoch`；不通过则 abort。
4. **embed 在锁外**：30s 超时不得持锁。
5. **effect 唯一**：`validateEffects()` 构造期拒绝同 type 重复。
6. **tmp 名唯一**：`file + '.' + pid + '.' + rand + '.tmp'`。
7. **坏行不静默**：读路径返回 `{rows, errors}`；写路径（remove/clear）用严格读直接抛错。
8. **无删除权**：Reflect 相关接口不存在于 M0；`memory-store` 的 `setLifecycle` 拒绝 `deleted` 之外的批量删除请求（除显式 remove）。

---

## 5. 风险

| # | 风险 | 级 | 缓解 |
|---|---|---|---|
| R1 | 写路径切换漏调用方 → 双写 | 高 | 旧函数 throw；grep 红线 1 |
| R2 | 恢复逻辑写错 | **阻断** | 12 场景隔离目录演练，先于切换 |
| R3 | CRC 逐行校验拖慢读取 | 中 | 只在写路径与 `verify-crc` 命令校验 |
| R4 | 权限分类器误判导致大量隔离 | 中 | 分类器规则表可单测；上线首周监控 quarantine 增长率 |
| R5 | 投影重建期间读半成品 | 高 | tmp + rename；失败降级"无投影"不阻塞回复 |

R2 是唯一"会毁数据"的风险，因此顺序必须是：`恢复演练 → 合并代码 → 真实崩溃注入`。

---

## 6. 验收（16 项）

完整判据见 `11-ACCEPTANCE.md` §5。摘要：

| # | 项 | 判据 |
|---|---|---|
| M0-01 | append 后 A/P/索引一致 | 同 id 版本相等 |
| M0-02 | edit 后向量重算 | `vecContentRev === contentRev` |
| M0-03 | remove 后三路不命中 | 语义/关键词/时间 |
| M0-04 | clear 后投影与墓碑 | P 无该 key；`clearEpoch` +1 |
| M0-05 | 删除不存在 = no_op | 响应无正文 |
| M0-06 | 并发 append 150 条 | 全落盘；rev 单调唯一 |
| M0-07~11 | 5 崩溃点 | 恢复后三者一致 |
| M0-12 | 投影重建 0 差异 | diff = 0 |
| M0-13 | 单写者静态检查 | grep |
| M0-14 | 索引一致性 | `索引集合 == active ∪ 尾部`，不相交 |
| M0-15 | 降级四场景 | 可用且注入文本有标注 |
| M0-16 | **权限字段推导** | public→origin_group；含凭证→owner_only+quarantine；无 groupId→quarantine |

**准入门槛**：M0-01~11 全过才能合并；M0-12~16 允许带缺陷进 M1，但 M1 结束前清零。

---

## 7. 迁移

记忆库当前 **0 条**（设计冻结时清空），本次是"建结构"不是"转数据"：

1. 建空文件结构。
2. `social-v2.json` 若残留 `memberImpressions[*].interactionCount` → 抄到 stats 文件，旧字段置 null。
3. `memory-tombstones.jsonl` 保留不动。
4. `memory-vectors.jsonl` 若存在 → 一次性导入器（Schema 转换），导入后改名 `.imported` 保留 30 天；历史 `review.status` 保持原映射，`contentClass` 统一置 `uncertain`（保守）。

---

## 8. 顺序（约 16 人日，比上一版 +2 天，因新增权限模块）

```
① memory-store.js（A 层 + WAL）              3.5d
② 恢复演练（隔离目录，12 场景）                1.5d   ← 必须早于 ③
③ bridge.js 写路径切换                        2d
④ memory-projection.js + 差异报告             2d
⑤ memory-stats.js（S-1）                      0.5d
⑥ memory-permission.js（classify + canRead）   1.5d
⑦ memory-vector.js 补丁（S-11 + tmp + edit）   1d
⑧ memory-index.js 写入钩子（docId/generation） 1.5d
⑨ 凭据过渡态（session_once）                    1d
⑩ 崩溃注入 12 场景 + 16 项验收                  2d
```

**② 在 ③ 之前**：先切写路径再发现恢复逻辑有问题，回滚成本是双倍的。
