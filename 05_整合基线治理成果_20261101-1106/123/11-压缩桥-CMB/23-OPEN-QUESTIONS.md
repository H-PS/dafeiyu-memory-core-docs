# 未解决问题清单（23-OPEN-QUESTIONS）

采集：2026-11-06 ｜ 主代理汇总
**本文只列真正未解决的。已解决的写结论，不写问题。**

---

## 1. 无法读取 / 无法验证的材料

| 项 | 原因 | 影响 |
|---|---|---|
| DSH 宿主核心代码 | `app.asar` 单文件 213MB，`unpacked` 仅含 node_modules；声明路径 `app.asar\dsh\` 在文件系统不存在 | 宿主身份注入时机、`currentInitiator()` 实际行为无法验证 |
| CPU / RAM 配置 | `Get-CimInstance` 返回 `PermissionDenied (0x80041003)` | 性能数据缺硬件上下文 |
| 真实 `index.db` | 本机无生产库副本 | E-1/E-2/E-3 等 10 项实验无法执行 |
| GitHub issue 实时数 | 无网络/API 访问；Agent B 尝试返回 `open_issues_count: 844`（含 PR） | 上游 issue 与 CMB 的关联仅凭代码注释 |
| `R4.1-memory-bank-input` 路径 | **不存在**，实际为 `Memory-Core-R4.1-Design` | 已用新路径，哈希已复核 |

---

## 2. 编号冲突与历史遗留

| 项 | 状态 |
|---|---|
| 实验编号 E-1…E-10 含义已变 | ✅ 已处理：采用 Agent F 新编号 E-1…E-12，旧编号弃用（`15` §0.2 有对账表） |
| 决策编号 D-1/D-6/D-7/D-8 | ✅ 已处理：`14` 用 D-1…D-4，顶部加映射表 |
| Agent G 的 X-1…X-15 与 LOG-3 的 X-1…X-6 串台 | ⚠️ **未完全解决**：两套编号含义不同，`19` §4.7 给出稳定编号 D-01…D-16，但 `01/02/03` 仍用旧表述 |
| 第三轮报告的用户裁决编号（D-6…D-9） | 保留引用，但建议用户以 `14-USER-DECISIONS.md` 为准 |

---

## 3. 需要真实数据才能回答的问题

| # | 问题 | 关联实验 | 当前状态 |
|---|---|---|---|
| Q1 | 真实 `index.db` 的 `user_version` 是多少？ | E-1 | UNVERIFIED（CMB 无版本检查，R-59） |
| Q2 | CAS blob 的实际 `algo`/`byte_len` 是否一致？ | E-2 | UNVERIFIED（R-61） |
| Q3 | 只读连接 + WAL 并发是否真的安全？ | E-3 | UNVERIFIED |
| Q4 | `messages.ts` 的实际偏差幅度？ | E-4 | UNVERIFIED |
| Q5 | message_id / block_id 重铸频率？ | E-5 | UNVERIFIED |
| Q6 | 宿主能否提供可信 tenant/bot/group/speaker？ | E-6 | UNVERIFIED（QQ Bridge 有 user_id，但未接入） |
| Q7 | Block 从产生到被删的实际窗口有多长？ | E-7 | UNVERIFIED |
| Q8 | Memory Core 真实写入入口在哪？ | E-8 | **不存在**（无实现） |
| Q9 | 真实数据中嵌套/重叠 Block 的比例？ | E-9 | UNVERIFIED |
| Q10 | 长期扫描对上游的实际影响？ | E-10 | UNVERIFIED |
| Q11 | 真实数据中消息类型分布？ | E-11 | UNVERIFIED |
| Q12 | 撤销与复活在真实数据中的频率？ | E-12 | UNVERIFIED |

---

## 4. 需要用户裁决的问题

| # | 问题 | 关联 |
|---|---|---|
| D-1 | `evidence.schema.json` 与 `DD-AUTHORITY-V2 §3.2` 哪个权威？ | BLOCKER R-17 |
| D-1b | `original_text` 必须存在吗？若必须，schema 必须加字段 | BLOCKER R-17 |
| D-1c | `source_type` 用哪套枚举（三方三套）？ | BLOCKER R-02 |
| D-2 | `tenant_id` 对单人部署如何定义？ | R-20 |
| D-2b | `bot_id` 用 `self_id`（真实可得）还是部署常量？ | R-20 |
| D-2c | 五标识只能"三实两缺"是否接受？ | R-19 |
| D-3 | Evidence 无原始时间是否接受？ | R-22 |
| D-3b | `recorded_at` 用桥接观察时间填充是否可接受？ | R-18 |
| D-4 | Block 错过即永久丢失，SLA 如何定？ | R-14 |
| **D-5（新）** | **R-01 speaker 链断裂如何解？** 五个方案：①上游补列 ②宿主注入 ③降级用 role ④放宽强制 ⑤放弃真实接入 | **BLOCKER R-01** |
| **D-6（新）** | **`content_hash` 改用 sha256(original_text) 后，旧的 blob 哈希存哪？** 新增字段？还是放弃 CAS 追溯？ | BLOCKER R-02 |

---

## 5. 技术判断分歧（Agent 之间或与主代理）

| # | 分歧 | 各方立场 | 主代理裁决 |
|---|---|---|---|
| C-1 | blob 内容是否"原始消息字节" | 主代理（初版）：是；Agent B：否，是拆分媒体后的投影 | ✅ **Agent B 正确**（`ingest.ts:194-195` 已核验） |
| C-2 | `content_hash` 是否 blocking | Agent C：`blocking:false`（结构兼容）；主代理/Agent G：BLOCKER | ✅ **BLOCKER**（13 §7.2-4 是 MUST；已修正 schema-diff.json） |
| C-3 | Memory Core 两个 schema 是否"机器互斥" | Agent C：是；Agent G：否（不带原句可同时通过） | ✅ **Agent G 更精确**：准确表述是"契约无法表达原句" |
| C-4 | 78 条测试数量算法 | 主代理：74 call site；Agent G：74+3 循环=77？ | ✅ **已核验**：74 静态 + for 循环 4 实例 = 78（`phase6:33`） |
| C-5 | `?? ""` 静默空串 CMB 是否堵住 | 主代理：堵住；Agent G：确实堵住 | ✅ **一致**（`bc-source.ts:97` + validator） |
| C-6 | M-1 交叉校验是否有效 | 主代理：有效；Agent E：真实数据上是死代码 | ✅ **Agent E 正确**（`bc-source.ts:41` 恒 null → 分支永不触发） |

---

## 6. 已知但未修复的代码缺陷（本轮不修）

按任务书要求，本轮只审查不实施。以下缺陷已确认但保留：

| 编号 | 缺陷 | 等级 |
|---|---|---|
| R-01 | speaker 链断裂 | BLOCKER |
| R-02 | content_hash 语义 | BLOCKER |
| R-03 | COMMITTED 锁死 | BLOCKER |
| R-04 | 冒号 locator | BLOCKER |
| R-05 | system 提示词进 Evidence | CRITICAL |
| R-06 | mode 不生效 | HIGH |
| R-07 | 未知类型放行 | HIGH |
| R-08 | UNSUPPORTED 卡死 | MAJOR |
| R-09 | evidence_id 撞键 | MAJOR |
| R-10 | M-2 未修 | MAJOR |
| R-12/R-42/R-43 | 媒体 hydrate 缺失 | MAJOR |
| R-15 | 撤销不可逆 | MAJOR |
| R-21 | session_type 硬编码 | BLOCKER |
| R-44 | speaker 方向反 | MAJOR |
| R-45 | M-1 死代码 | MAJOR |
| R-53 | 空 directIds | MAJOR |
| R-59/R-60/R-61 | Agent F 新发现 | MAJOR/MINOR |

---

## 7. 明确无法回答的问题

| 问题 | 为什么无法回答 |
|---|---|
| CMB 真实源下的端到端表现？ | 无真实库；且 speaker 链断裂（R-01）预示零产出 |
| Memory Core 能否接收 Evidence？ | 无实现；schema 冲突未裁决 |
| 跨进程崩解恢复是否真的工作？ | 测试全用 `:memory:`；需重写测试才能回答 |
| 多模态消息完整恢复？ | 需实现 hydrateMedia 才能验证 |
| 并发 Run 的实际危害？ | 无锁；需真实并发测试 |

---

## 8. 下一阶段必须回答的（进入 v0.2 前）

1. 是否获得真实 `index.db` 的**授权只读副本**？
2. 该副本的 `user_version` 是多少？（决定 CMB 是否需要版本分支，R-59）
3. 宿主能否导出身份映射表？（R-01 的前提）
4. 用户是否裁决 D-5（speaker 方案）与 D-6（content_hash 存储方案）？
5. 是否接受"Block 错过即丢失"的 SLA？
