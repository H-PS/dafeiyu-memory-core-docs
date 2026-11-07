# 06 · Interface Map（跨组件接口登记图）

- 生成：2026-11-02（R4 整合轮，父代理）
- 性质：**登记性文件**——把散落在各文档中的跨组件接口/依赖集中为一张图。除标注外全部为 **PROPOSAL/DRAFT**，无一冻结。本图不构成实施授权。
- 来源：Agent1 决策寄存器 §七（X-1~4）、整合方案 §3/§5、Agent2 统一问题清单（A/B/E 系）、PIPE-PERSONA SPEC、m01–m05 相关章节。逐条登记时以出处原文为准。

---

## 1. 接口总图（数据流视角）

```
QQ 用户 ⇄ DSH（对话运行时；不直连数据库、非权限引擎）
             │  ①Memory Context 注入（纯数据）      ← M02/C30 组装
             │  ②行为提示 hint（≤50字，有界）        ← PsyStateEngine（心理层）
             │  ③温度倍率（建议）                    ← PsyStateEngine
             │  ④回复回传（E14 强化时机依赖）        → PsyStateEngine/反馈记账
             ▼
R3 Memory Core
  M01 存储（SQLite 权威：memory.sqlite 等 9 类对象）
      ├─ affect_* 字段 + 反馈三字段（last_retrieved/confirmed/reinforce，C51 提案）
      └─ cache.sqlite（独立缓存库，可重建、可 L0 关闭）
  M02 检索编排（唯一权威：触发/融合/排序/预算/上下文构建）
      ├─ 排序插槽 ← PsyStateEngine 七步链（影子期仅 diff 日志）
      └─ 注意力容量输入 ← mind 注意力输出（插槽语义，m02 D7）
  M03 提取（EmotionExtractor 写时标注 → affect_*；只产候选不写正式记忆）
  M04 整合审核（裁决入库；Gateway 唯一实现 §6.2【D-AP-02 待裁】；Dream 候选暂定接口 §5.3）
  M05 权限（唯一判权；LLM 禁判权；Fail-Closed；不可插拔 C40）
  Dream（自治探索 → 受控接口 → M04 候选；接口有意暂定 U-3）
心理层 PsyStateEngine（可插拔槽位 #17；影子模式宿主）
  ├─ 状态持久化：三个候选落点（A01：M01 新增 KV / 独立运行时态文件 / cache 同级）——未决
  └─ G2 硬护栏：不产生记忆、不写人格、不直连记忆库、不触发记忆写入
人格层（Agent3 域，现网 Bridge）
  └─ 人格:状态:记忆 注入配比（消费方；单向依赖，不反写 R3）
```

## 2. 跨流接口契约登记表

| # | 接口 | 生产方 | 消费方 | 契约要点 | 数据分类 | 失效行为 | 状态 | 出处 |
|---|---|---|---|---|---|---|---|---|
| X-1 | mind.hint() | PsyStateEngine（Agent2） | M02/Bridge | 此刻状态 ≤50 字 | 运行时态（类别未决，E01） | 未定义 | PROPOSAL | Agent1 寄存器 §七 |
| X-2 | 注意力容量值 | PsyStateEngine | M02 D7 动态预算 | 容量数值输入 | 运行时态 | 未定义 | PROPOSAL | 同上 |
| X-3 | 情绪一致性评分输入（记忆侧 valence） | M01 affect_* 字段 | PsyStateEngine 七步链③ | 读 affect_valence，同向/反向 | 记忆元数据 | 未定义 | PROPOSAL | 整合方案 §3 |
| X-4 | 排序插槽 | M02 | PsyStateEngine（L2 槽位 #17） | 影子期仅 diff；晋升需 ≥2 周+一致率 ≥75% | — | 降级链：重置中性→六步链→纯 RRF | PROPOSAL（影子强制 I-4 待裁） | 整合方案 §5.1 |
| X-5 | 反馈三字段（C51） | M02/M04 记账 | 衰减 v2、PsyStateEngine、D 阶段实验 | last_retrieved/confirmed/reinforce；异步、软状态可丢失；W5 条件：reinforce 仅由 DSH 回传触发，E14 落地前 schema 冻结、写入禁用 | 记忆元数据 | 可丢失不阻塞 | PROPOSAL（D-AP-05 待裁） | D-CH-04；W5 议题 5 |
| X-6 | Memory Context 注入格式 | M02/C30 | DSH/Bridge（Agent3） | 纯数据、无指令层；格式稳定性承诺 | 检索结果派生 | 未定义 | PROPOSAL | X-3（Agent1 寄存器） |
| X-7 | 人格:状态:记忆配比 | Bridge 组装层 | DSH prompt | 单向消费；配比待 DSH 侧答复 | 表达层 | 未定义 | PROPOSAL | PIPE-PERSONA/Agent2 §6 |
| X-8 | 温度倍率 | PsyStateEngine | DSH | 倍率（非绝对值）——B02 未答 | 运行时态 | 未定义 | PROPOSAL（待 Agent3） | 统一问题清单 B02 |
| X-9 | 行为提示通道 | PsyStateEngine | DSH | 不进人格段（B05 红线）；长度上限 B03 未答 | 运行时态 | 未定义 | PROPOSAL（待 Agent3） | B01/B03/B05 |
| X-10 | 回复回传 | DSH | PsyStateEngine/E14 强化 | "被使用才强化"依赖此回传（B04 未答） | 用户对话数据 | 未定义 | PROPOSAL（待 Agent3） | B04/E14 |
| X-11 | Dream 候选接口 | Dream | M04 | 入口/载荷/校验**有意暂定** | Dream 产物 | 未定义 | DRAFT（U-3 悬置） | dream.md/m04 §5.3 |
| X-12 | EmotionExtractor 写时标注 | M03 | M01 affect_* 落库 | 检索期不调 LLM 红线；V1 词表 15 个 | 记忆元数据 | 未定义 | PROPOSAL（J2 待裁） | A 方案/J2 |
| X-13 | 心理状态持久化位置 | PsyStateEngine | M01 或独立文件 | 三候选（O1 M01 新增 KV / O2 独立运行时态文件 / O3 cache 同级）；E01 主张新增第三类数据 | **运行时态（类别缺口）** | 未定义 | **未决（A01/E01，阻塞项）** | 统一问题清单 A01/E01 |
| X-14 | Model Gateway | M04 §6.2 唯一实现 | M03/M02 等 LLM 调用方 | m03 §12 废弃为接口定义【需改冻结稿】 | — | 五条约束（D-CH-07） | PROPOSAL（D-AP-02 待裁） | D-CH-01 |

## 3. 权限与数据分类缺口（登记，不裁决）

1. **运行时态类别缺口（E01）**：M01 D2 的 9 类对象全部是"关于用户的数据"；心理状态是"关于 AI 自身的数据"（可丢弃、不备份、不回滚、与权限无关）。X-13 因此悬空。→ 已入冲突登记 K-1。
2. **E05 伦理缺口**：心理状态是否属用户数据（导出/删除权、删号时是否连带删除）未定义，影响 L2 数据模型。→ 已入冲突登记 K-2。
3. **回传链缺口（B04/E14）**：C51 的 confirmed/reinforce 与心理层"被使用才强化"（N15）都依赖 DSH 回传，B04 未答前两个闭环都断。→ 待 Agent3 答复。
4. **插件旁路检查**：可插拔组件（含 PsyStateEngine 槽位 #17）不得成为判权输入的生产方；W5 已确认两条绕过路径——(A) 可插拔组件可伪造可信上下文代上游发起判权；(B) 影子模式 diff 日志可能含未脱敏 deny 内容。补强条件（manifest 增 `trusted_context_source` 与 `diff_log_classification` 字段）已入裁决增补层 W-2。

## 4. 契约状态汇总

- 14 条跨流接口：**0 条已批准、0 条冻结**；PROPOSAL 11 条、DRAFT/悬置 2 条、未决阻塞 1 条（X-13）。
- 每条接口的"失效行为"字段大多未定义——这是设计文档的系统性空白，建议进入裁决材料（不阻塞架构裁决，但阻塞实现）。
