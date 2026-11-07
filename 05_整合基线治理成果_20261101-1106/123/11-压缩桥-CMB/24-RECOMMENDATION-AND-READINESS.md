# 真实接入准备度评估（24-RECOMMENDATION-AND-READINESS）

采集：2026-11-06 ｜ 主代理综合 7 路审查 + 自主实测

---

## 1. 逐项状态判定

**只允许以下状态**：PASS / PASS_WITH_LIMITATION / BLOCKED / FAIL / UNVERIFIED / USER_DECISION_REQUIRED / UPSTREAM_CHANGE_REQUIRED / ADAPTER_WORK_REQUIRED

---

### A. 原型质量

| 项 | 状态 | 依据 |
|---|---|---|
| CMB v0.1 是否达到独立原型完成标准 | **PASS_WITH_LIMITATION** | 78 测试可跑、核心数据流在 Mock 上自洽；但 6 个 BLOCKER 表明"原型完成"仅指"框架搭好"，不代表"可用" |
| 78 条测试是否真实对应当前代码 | **PASS** | 主代理实跑 exit=0；74 静态 + 4 循环 = 78 |
| 核心功能是否有足够测试证据 | **FAIL** | 6 个已确认 BLOCKER 全部逃过测试；0 条测试用真实 SQLite；0 条跨进程 |
| B-1 修复是否彻底 | **FAIL** | UNSUPPORTED 未覆盖（D-08）、COMMITTED 锁死（D-03）两个新死锁 |
| 是否存在新 BLOCKER | **是，6 个** | D-01/D-02/D-03/D-04/D-16/D-17/D-21 |

### B. 数据源（billion-context）

| 项 | 状态 | 依据 |
|---|---|---|
| 是否保存足够的原始消息信息 | **PASS_WITH_LIMITATION** | 文本存在 CAS blob（U-01），但：是多模态拆分后的投影（U-02）、tombstone 后媒体永久丢失（U-10） |
| CAS Blob 可否可靠恢复 | **PASS_WITH_LIMITATION** | 结构可读；但 CMB 未实现 hydrateMedia（D-12）；`algo`/`byte_len` 未校验（D-61） |
| 能否发现全部需处理的 Block/Message | **FAIL** | Block 无墓碑、物理删除（D-14）；错过窗口永久丢失 |
| Block 生命周期是否支撑可靠对账 | **FAIL** | 无历史、`supersedes` 恒 null、`covered_hash` 无人校验 |
| SQLite WAL 只读扫描是否安全 | **UNVERIFIED** | 需 E-3 实验；生产 `open()` 无 readOnly 参数（U-16） |
| 适配器版本兼容 | **FAIL** | 隐含 `user_version >= 10` 门槛无检查（D-59） |

### C. Memory Core

| 项 | 状态 | 依据 |
|---|---|---|
| Evidence Schema 冲突是什么 | **已定义** | `evidence.schema.json`（v1 基线）vs `DD-AUTHORITY-V2 §3.2`（权威）；13 个 blocking 字段 |
| 哪份契约最接近真实运行事实 | **无** | 无实现代码，"运行事实"不存在；两者都是设计期产物 |
| CMB 能否只通过 Adapter 适配 | **FAIL** | 至少 3 项必须改上游（schema 缺字段、无实现、无迁移机制） |
| M03/M04 是否具备可用接入入口 | **UPSTREAM_CHANGE_REQUIRED** | 标 `[FUTURE]`，无实现 |
| 撤销是否有完整下游传播 | **UPSTREAM_CHANGE_REQUIRED** | Memory Core 无实现；CMB 侧也缺 un-revoke |

### D. 身份与安全

| 项 | 状态 | 依据 |
|---|---|---|
| 五标识能否形成可信映射 | **FAIL** | `session_id` 真实；`tenant/bot` 部署常量；`group` 手工 Map；**`speaker` 恒 null** |
| 缺少身份时能否 fail-closed | **PASS_WITH_LIMITATION** | 会拒绝（D-01 证明），但这是"过度拒绝"——真实源全部被拒 |
| 是否可能跨群/跨租户污染 | **PASS_WITH_LIMITATION** | 值层面防住（M-1）；但 M-1 在真实数据上是死代码（D-45），真正闸门在 QQ Bridge |
| 是否可能将内部消息写入 Evidence | **FAIL** | system 提示词直接进入（D-05）；未知类型静默放行（D-07） |
| 是否有敏感数据泄露风险 | **PASS** | 日志/报告/Ledger 无正文（Agent E 实测 `NO_SECRET_LEAK_FOUND`） |

### E. 一致性与可靠性

| 项 | 状态 | 依据 |
|---|---|---|
| 当前幂等机制保证什么 | **PASS_WITH_LIMITATION** | 设计是 at-least-once + 幂等消费；但 Evidence 层会撞键（D-09）、content_hash 语义错（D-02） |
| Evidence 与 Ledger 是否存在不一致窗口 | **是（设计如此）** | 非同一事务；但有幂等补账逻辑，可收敛 |
| 能否任意阶段崩溃后恢复 | **UNVERIFIED** | 逻辑存在，测试全用 `:memory:`（D-11） |
| 撤销是否可能被重新激活 | **否（反面问题）** | 撤销不可逆；上游复活会 SINK_CONFLICT（D-15） |
| 上游删除后是否出现不可发现缺口 | **是** | Block 删除无感知（D-14）、Session 删除无感知 |

---

## 2. 是否建议进入受限真实数据验证

### 结论：**不建议现在进入。需先修 4 项前置。**

理由：
1. **D-01（speaker）会让 v0.2 验证"白跑"**：真实库上 CMB 产出 0 条 Evidence，实验会得到"零结果"，无法区分是"数据源不行"还是"CMB 不行"。
2. **D-59（user_version 门槛）会让实验直接报错**：若库是 v9，`SELECT m.ts` 失败。
3. **D-04（冒号 locator）会让撤销类实验（E-12）失效**：若 sessionId 含冒号，revoke 永远返回 0。
4. **D-06（mode 不生效）会让"只读验证"变写入**：即便 `DRY_RUN`，也会写 ledger。

**这四项都是 CMB 单方可修的，不需要用户裁决、不需要上游改动。**

---

## 3. 进入 v0.2 的前置条件（必须全部满足）

| # | 条件 | 类型 | 工作量估算 |
|---|---|---|---|
| 1 | 修复 D-01：确定 speaker 来源（需 D-5 裁决） | 用户裁决 + 开发 | 中 |
| 2 | 修复 D-59：加 `PRAGMA user_version` 检查 | CMB 单方 | 小 |
| 3 | 修复 D-04：locator 改分隔符或结构化 | CMB 单方 | 小 |
| 4 | 修复 D-06：mode 真正生效（至少 DRY_RUN 不写） | CMB 单方 | 小 |
| 5 | 获得真实 index.db 的**授权只读副本** | 用户 | — |
| 6 | 确认副本的 `user_version` | 实验 E-1 | 小 |
| 7 | 宿主侧身份导出物（group/speaker 映射） | 用户/Host | 中 |

**条件 2/3/4 可以在无授权的情况下先做（纯代码修复 + Mock 测试）。**

---

## 4. 哪些问题可先通过只读实验回答

| 问题 | 实验 | 需要真实数据 |
|---|---|---|
| user_version 是多少 | E-1 | ✅ |
| CAS 实际结构 | E-2 | ✅ |
| WAL 并发安全性 | E-3 | ✅ |
| ts 偏差幅度 | E-4 | ✅ |
| ID 重铸频率 | E-5 | ✅ |
| 宿主身份可得性 | E-6 | ✅（+授权） |
| Block 删除窗口 | E-7 | ✅（+授权） |
| Evidence 契约兼容 | E-8 | ❌（可合成，但需先裁决 schema） |
| 嵌套/重叠比例 | E-9 | ✅ |
| 长期扫描影响 | E-10 | ✅（+授权） |
| 消息类型分布 | E-11 | ✅ |
| 撤销/复活频率 | E-12 | ✅ |

**12 项中 10 项需要真实数据。**

---

## 5. 哪些问题必须用户裁决

见 `14-USER-DECISIONS.md` + `23-OPEN-QUESTIONS.md §4`：
- D-1/D-1b/D-1c（schema 权威与字段）
- D-2/D-2b/D-2c（tenant/bot/group 定义）
- D-3/D-3b（时间戳）
- D-4（Block 窗口 SLA）
- **D-5（新）speaker 来源**
- **D-6（新）content_hash 存储方案**

**共 10 项。**

---

## 6. 哪些必须修改上游

| # | 问题 | 需要什么 |
|---|---|---|
| 1 | Memory Core 无实现 | 实现 M01/M03/M04 |
| 2 | evidence.schema.json 缺 original_text/speaker_id | 修订 schema（+ 迁移机制） |
| 3 | Block 无墓碑 | 增加 retired 机制或保留策略 |
| 4 | speaker 无列 | 增加 speaker 列（或宿主注入方案） |
| 5 | `covered_hash` 无校验 | 增加校验或提供替代凭据 |
| 6 | `user_version` 版本演进 | CMB 适配（不需上游改） |

前 5 项中，**1/2 是 Memory Core**，**3/4/5 是 billion-context**。

---

## 7. 哪些可以留到真实 Adapter 阶段

| 问题 | 理由 |
|---|---|
| hydrateMedia 实现（D-12） | 不影响文本消息验证 |
| 并发 Run 保护（R-29） | v0.2 单进程足够 |
| 性能优化（R-28 重复查询） | 正确性优先 |
| 数据库世代标识（R-30） | 重建场景罕见 |
| 日志 events 上限（R-39） | 长跑才暴露 |

---

## 8. 推荐的下一阶段执行顺序

```
第 0 步（无授权可做，1-2 天）
  ├─ 修 D-59（user_version 检查）
  ├─ 修 D-04（locator 分隔符）
  ├─ 修 D-06（mode 生效）
  ├─ 修 D-02（content_hash 改 sha256(original_text)）
  ├─ 修 D-05（system 改查 role）
  ├─ 修 D-07（白名单）
  ├─ 修 D-08（UNSUPPORTED 复位）
  ├─ 修 D-10（Math.max）
  └─ 补测试：冒号 sessionId、真实文件 ledger、跨进程

第 1 步（需用户裁决 D-5/D-6）
  ├─ D-5 决定 speaker 方案 → 修 D-01
  ├─ D-6 决定 content_hash 存储 → 完成 D-02
  └─ 决定 D-1（schema 权威）

第 2 步（需授权）
  ├─ 获得 index.db 只读副本
  ├─ 执行 E-1（结构 + user_version）
  ├─ 执行 E-2（CAS 结构）
  └─ 执行 E-3（WAL 并发）

第 3 步
  ├─ 修 D-45（M-1 死代码：适配器提供真实 group_id）
  ├─ 修 D-21（session_type 注入）
  └─ 执行 E-6（身份映射）+ E-11（类型分布）

第 4 步
  ├─ 执行 E-7/E-9/E-12（生命周期类）
  └─ 评估数据完整性上限

第 5 步
  └─ 裁决 D-1 后执行 E-8（契约兼容，合成库）
```

---

## 9. 最终判断

| 问题 | 答案 |
|---|---|
| CMB v0.1 通过原型阶段审查了吗？ | **PASS_WITH_LIMITATION**：框架完成、Mock 自洽，但存在 6 BLOCKER + 1 CRITICAL，不可用于任何真实数据 |
| 最重要的 BLOCKER | **D-01 speaker 链断裂** —— 它让"真实接入"在架构上不可能，且需要用户裁决 |
| 建议进入受限真实数据验证吗？ | **否，先修 4 项 CMB 单方可修的缺陷**（D-59/D-04/D-06 + D-02/D-05/D-07/D-08/D-10） |
| 修完后建议吗？ | **是**，但需用户先裁决 D-5（speaker 方案） |
| 距真实接入（v1.0）还差什么 | 6 BLOCKER + Memory Core 实现 + schema 修订 + 身份体系 |

### 一句话

**CMB v0.1 是一个诚实的骨架：它清楚地知道自己不能做什么（Memory Core adapter 全抛异常），但不知道自己有几个致命问题（6 个 BLOCKER 全部逃过 78 条测试）。它的价值在于把"接入前必须解决的清单"从模糊变成具体——这份清单现在有 58 项，其中 18 项阻塞接入。**
