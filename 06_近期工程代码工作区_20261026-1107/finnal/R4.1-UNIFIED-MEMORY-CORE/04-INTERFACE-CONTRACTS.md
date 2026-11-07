# 04-INTERFACE-CONTRACTS · 跨流接口契约登记

> 基础：R4 06-interface-map（14 条登记）＋ B-07 §8.2 修正（逐条登记表口径 12 PROPOSAL/1 DRAFT/1 未决）＋ A 侧 Event-L3 扩展需求（E-1/E-2）。
> 修订：每条接口**失效行为必填**（B-06 发现"大多未定义"后的系统性补齐）；状态列如实标注——**0 条已批准、0 条冻结**（06-IM L64 实测）。
> 纪律：接口名称实现期确定（优化-5），但契约方向与失效行为在设计层先闭合。

---

## 一、接口总图（数据流）

```
DSH ⇄ Bridge ⇄ Memory Core（M02 检索编排 → M05 判权 → M01 权威读取）
                 Psychology（影子 slot → 仅 diff 日志）
                 Persona（注入配比消费方，单向）
                 Dream（读经 M05/M02，候选→M04 暂定接口 U-3，报告→M01）
```

## 二、跨流接口契约登记表（统一修订版）

| # | 接口 | 生产方 → 消费方 | 契约要点 | 数据分类 | **失效行为**（必填） | 状态 |
|---|---|---|---|---|---|---|
| I-1 | 状态提示 hint | PsyStateEngine → Bridge/DSH | ≤50 字有界状态提示（台账 X-7 修订方向：有界提示，温度倍率作废〔待裁〕）。**〔补读核验〕长度上限两源冲突**：50 字（PSYCH-PIPELINE §八）vs 100 字（Agent2 v2.1 Architecture §5.2/R3-Integration §8.3），且 DSH 侧 B03 上限未答——取值待裁 | 运行时态 | 心理层不可用→无 hint，正常聊天不受影响（F4） | PROPOSAL（X-7 修订待裁；长度两源冲突待裁） |
| I-2 | 注意力容量值 | PsyStateEngine → M02 D7 动态预算 | 容量数值输入（倒 U 曲线输出）。**〔补读核验〕单位=条数（Pipeline S8），不得与 token 预算混用** | 运行时态 | 缺失→M02 用静态预算默认值（F4） | PROPOSAL·**门禁未覆盖**（红队 RT-05：此通道影响入选集，是否纳入 X-4(台账) 排序插槽门禁待裁，14 §2.3） |
| I-3 | affect_valence 读取 | M01 affect_* 字段 → PsyStateEngine 七步链③ | 写时标注的情绪元数据（J2：检索期不调 LLM） | 记忆元数据 | 字段缺失→跳过情绪一致性步（F4） | PROPOSAL（J2 已裁方向，m01/m03 零承载 B-038） |
| I-4 | 排序插槽 | M02 → PsyStateEngine（L2 槽位#17） | **delta-only 影子**：只记 diff 日志，不改最终排序（台账 X-1 已裁）；晋升门禁=ME-1 语料+κ≥0.6+双签（台账 I-4/W-4） | — | 影子组件失败→跳过，用原序（F4）；diff 日志默认只记 metadata（W-2 已裁） | PROPOSAL（门禁未满足） |
| I-5 | 反馈三字段 C51 | M02/M04 记账 → 衰减v2/PSYCH | last_retrieved(M02 命中即写)/confirmed(**仅 M04 审核通道**)/reinforce(**仅 DSH 回传触发；E14 未落地且 B04 回传未确认前禁写**——台账 D-AP-05 双条件红线)；软状态可丢失 | 记忆元数据 | 可丢失不阻塞聊天（F4）；**reinforce 违规写入=红线违规**（F1） | **方向已裁（D-AP-05+W-3）；契约未冻结**；m01 D19 无字段——B-046 落点待补 |
| I-6 | Memory Context 注入格式 | M02/C30 → Bridge/DSH | 纯数据无指令层；不含 memory_id/时间戳/分数/内部过程（M02 D19 五项默认不提供） | 检索结果派生 | 构建失败→空上下文，不编造（F5） | PROPOSAL |
| I-7 | 人格:状态:记忆配比 | Bridge 组装层 → DSH prompt | 单向消费；不反写 Core。**〔补读核验拆义〕**现网 Agent3 改造对象实为"人格卡位置/频率/体量"（锚定式注入，ROLE_ANCHOR_EVERY_N 默认 20）；"配比"另指 PSYCH §八三段配额（人格300:此刻50）——**两义并存须区分** | 表达层 | 未定义（契约随 CR-08B 审批后冻结） | PROPOSAL（**CR-08B 材料齐备——四份逐项文档 D1–D3+CU-1..8 完整，W3 已核，待用户逐项审批**） |
| I-8 | 温度建议 | PsyStateEngine → DSH | **〔X-7(台账) 修订提案：倍率作废→有界状态提示，待裁〕** | 运行时态 | 修订未批前不生效 | PROPOSAL·**域内前置**（红队 RT-16 修正：温度范围统一为台账 §4.3 域内事项，责任人 Agent2——其未完成不因本提案升级而消失） |
| I-9 | 行为提示通道 | PsyStateEngine → DSH | 不进人格段（B05 红线） | 运行时态 | 未定义（长度上限 B03 未答） | PROPOSAL |
| I-10 | 回复回传（E14） | DSH → PsyStateEngine/C51 reinforce | "被使用才强化"的唯一触发源 | 用户对话数据 | **链路不存在（F6/B04）**——E14 未落地前 reinforce 禁写（F1 阻断态） | PROPOSAL（阻断中） |
| I-11 | Dream 候选接口 | Dream → M04 | 入口/载荷/校验**有意暂定**（U-3）；候选必须 inferred/uncertain、不得生成 support | Dream 产物 | 未定义（接口冻结前 Dream 不产生候选） | DRAFT（U-3 悬置） |
| I-12 | 情绪写时标注 | M03 EmotionExtractor → M01 affect_* | 15 个受控标签起步（J4）；J5 清除走 M04 修订通道。**〔补读核验 P0 修正〕落点阻断：D-AP-10 已裁 m03 保持冻结，仅 D-AP-02+W-1+PC-1 满足后方可提修订**——V1 落点改 M04 承接（m03 修订待放行）或明示仅预留 | 记忆元数据 | m03/m01 零承载（B-038）；冻结阻断叠加 | PROPOSAL（J2/J4 已裁方向；**m03 冻结阻断**） |
| I-13 | 心理状态持久化 | PsyStateEngine → M01 运行时态 KV | K-1(台账) 有条件接受 O1：M01 命名空间非权威 KV；不参与权威判定/Outbox/判权；**生效前置两项：K-2 导出边界逐项清单 + M01 D2/D10 落位设计**（台账 §3f——K-2 完成不单独构成生效条件，红队 RT-03 修正）。**〔补读核验补全〕提交契约（Agent2 R3-Integration §5.2）**：scope/key/state/version/provenance；M01 有权拒绝非法提交；不得改 schema；**备份豁免语义：不备份/不恢复/恢复后状态保持（A03/Q3，含测试 I11）** | 运行时态（第10类，B-023 缺口） | **未决阻塞**（两项前置均未完成） | UNRESOLVED |
| I-14 | Model Gateway | M04 §6.2 → M03/M02 等 | 唯一 LLM 入口（D-AP-02）；m03 §12 废弃为接口定义【PC-1 未完成前不修订冻结稿】；五条故障不变量需补齐（W-1） | — | provider 失败→路由降级→任务排队，**绝不假装成功**（F1） | PROPOSAL（W-1 前置未完成） |
| E-1 | prompt/assembled 观察事件 | DSH → Bridge | 第一阶段实际组装观察记录：sessionId/turn/correlationId/assemblyId/memoryRefsInFinalPrompt（DSH 声明的 complete 集）/evidenceRef + **hasUnknownVersions 扩展**。⚠ **结构性互斥（红队 RT-01/N-14）**：M02 D19/I-6 五项禁令禁止 memory_id 类标识进入 Memory Context——DSH 无法声明它未曾收到的版本标识；要么版本标识随 Context 下传（破 I-6 需重裁），要么 E-1 退化为 Bridge 复制（两链合一）。**此互斥先于 L3 任何实现设计，须先裁** | 事件数据 | **F3=否**：DSH 契约无此事件——扩展落地前 L3 不可用 | **候选扩展**（未实现·N-14 待裁） |
| E-2 | 第二阶段确认记录 | Bridge → 存储消费方 | 仅整体有效性 passed 且版本 complete 时产生；仅覆盖明确列出的版本；invalid/NE 不伪装确认 | 事件数据 | **F4=否**：无载体——F-CK-9 未核查前不得产出任何确认记录 | **候选扩展**（未实现） |

**状态汇总（红队 RT-10 修正后如实口径）**：16 条（14 原有 + E-1/E-2 扩展）——批准 0、冻结 0；PROPOSAL 13、DRAFT 1、UNRESOLVED 1（I-13）、候选扩展 2。**失效行为：明确定义 11/16；悬置语义定义 5/16**（I-7/I-9 未定义、I-11 悬置定义、I-12 待补、I-13 未决——均如实标注于行内，不虚计为"已定义"）。相对 R4 基线"大多未定义"仍是系统性补齐。

**表头说明（红队 RT-09 修正）**："生产方 → 消费方"列登记的是**主生产方与主消费方**——I-5（M02/M04 联合记账）、I-6（M02 产/C30 为 M02 内上下文构建能力=Matrix C30，非独立模块）等多生产/多消费情形如实按主次登记，不声称绝对唯一。C30 = M02 上下文构建能力（Capability-Ownership-Matrix C30），本表 I-6 的"生产方"即其承载。

**〔交叉复核修正〕状态滞后说明**：06-interface-map 原表对 I-4/I-5/I-12/I-14（原 X-4/X-5/X-12/X-14）的"待裁"状态列**滞后于台账**——影子门禁（台账 I-4）、反馈三字段（D-AP-05）、写时标注（J2）、Gateway 收敛（D-AP-02）六轮均已裁方向；"0 批准 0 冻结"指的是**接口契约文本**未经批准冻结，不等于相关事项无裁决。本登记表状态列按台账最新裁决方向修正。计数差异（11/2/1 vs 12/1/1）属 I-14 归组口径不一，本表不重复该矛盾，按 16 条实数登记。

**〔红队 RT-04 修正〕编号体系声明**：本表接口编号自 v1.1 起改用 **I-xx** 前缀（原 06-interface-map 的 X-1~X-14 改号为 I-1~I-14），与裁决台账的 X 系（X-1~X-8，心理层/情绪整合裁决项）彻底区隔。引用历史材料中的"X-4 插槽/X-12 标注"等接口义时，一律标注"接口 I-x（原 X-x）"；台账义标注"台账 X-x"。

## 三、Core 内部接口（模块间，设计已闭合）

| 接口 | 方向 | 契约 | 状态 |
|---|---|---|---|
| Storage API | M01 → M02/M03/M04/M05 | 统一读写入口；Repository+IndexAdapter 分层；禁 SQL 拼接 | 设计定（名称实现期） |
| processCandidates | M03 → M04 | candidate_refs/subject_id/policy_version → task_id/accepted/pending/rejected（字段未冻结） | m04 §12.1 |
| updateObservation / buildView | M04 内部→M01 | expected_version 乐观并发；原子发布 | m04 §12.2/12.3 |
| listNeighbors | M04 → M02 | 一跳邻居查询；**不接收 depth/budget**；M02 持扩展预算 | m04 §12.4（已冻结职责分工） |
| 权限判定 | M01–M04 → M05 | request_id/actor/action/resource/scope → decision/reason_code/policy_version/permission_version | m05 §10（已冻结） |
| memory_search | DSH → Bridge → M02 | 八条实施约束（D16）；每次调用新请求边界 | m02 D16（已冻结） |

## 四、接口层未决问题（B-07 §13 前置的接口部分）

| # | 问题 | 阻断 | 去向 |
|---:|---|---|---|
| 1 | M02→M01 检索接口签名/参数/返回结构未冻结 | V1 | 14 号 + 实现设计 |
| 2 | M04→M01 字段级写入契约（B-046） | V1 | 14 号 |
| 3 | M03→M04 候选内容可达性（B-048） | V1 | 14 号 |
| 4 | Embedding 路由：M01 全权 vs M02 感知 index_space（B-011，未裁） | 条件 | 14 号 |
| 5 | C51 三字段在 m01/m02/m04 的落点（B-046/I-5） | V1 | 14 号 |
| 6 | I-13 心理状态持久化（前置两项：K-2 清单 + M01 D2/D10 落位） | 心理层 V1 | 14 号 |
| 7 | **E-1 版本标识下传与 I-6 五项禁令互斥（N-14，红队 RT-01）** | **L3 全链** | 14 号第一层 |
| 8 | **M04 合并时 affect_* 聚合规则（Agent2 Q16/Q17）**——多来源情绪元数据合并（intensity 加权平均/peak-end 0.4/0.25/0.35 为 Agent2 候选）无规则，M04 为冻结稿零承载 | V1 条件 | 14 §4.5 |

---

*登记完毕。Event-L3 详细证据链设计见 08；全部未决汇总见 14。*
