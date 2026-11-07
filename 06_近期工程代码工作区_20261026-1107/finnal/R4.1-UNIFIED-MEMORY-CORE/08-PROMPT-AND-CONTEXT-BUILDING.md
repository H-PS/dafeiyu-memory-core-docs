# 08-PROMPT-AND-CONTEXT-BUILDING · 上下文构建与 Event-L3 证据链

> 依据：m02 D6/D19–D22（结果格式/呈现/注入）、06-interface-map X-6、v4.6 候选五组替换文本（A–E）、A5/A6/A7 全部审查发现、F-CK-1~10。
> 状态：**Event-L3 部分为候选设计**（v4.6 未批准、九项决策未裁、F-CK 全未核查）。本文件把候选条文与 A 侧审查修订**合并呈现**，修订处逐条标注。
> **基线声明**：v4.6 候选文件为 158 行"收口替换文本"（其自述：不构成完整 v4.6）；v4.5 完整正文不存在于本地（仅 history-archive 摘录）。本节 Event-L3 设计基于**候选片段 + 摘录**，任何引用不得表述为"完整 v4.6 条文"。

---

## 一、Memory Context 构建（M02，设计已闭合）

**呈现原则（D19）**：DSH 接收的是结构化记忆内容，不是数据库查询报告。默认不提供五项：memory_id、数据库时间戳、来源编号、检索分数、内部查询过程（Core 内部保留用于校验/纠正/审计）。

**注入原则（D20 少而准）**：准确性>相关性>完整性>数量；一条够不注入多条；信息足够即停；保留时间变化、人物区别、用户纠正；允许返回少但不准。

**置信度分层（D21）**：高置信→正常提供；中等→少量提供保留不确定性；低→不进普通上下文；无可靠结果→返回空，DSH 自然表达不记得。模糊结果可作第二阶段检索线索，不得包装成确定事实。

**纯数据红线（X-6/Pluggable 不变量 5）**：Memory Context 不含人格/规则指令类字段；不改变原始记忆含义；保留事实与推断区别（Explicit/Inferred/Uncertain/Corrected）。

## 二、Event-L3 证据链（候选设计 + A 侧修订）

### 2.1 L1–L5 五级确认链

| 级 | 含义 | 可验证性 |
|---|---|---|
| L1 | 检索到该记忆 | 可 |
| L2 | 准备注入 Prompt | 可 |
| **L3** | **记忆版本确实进入最终 Prompt** | **本项目核心，可验证（候选）** |
| L4 | 模型实际使用了该记忆 | **无可靠通用信号，不在状态机内** |
| L5 | reinforce 写入 | **禁写中（E14 未落地）** |

**红线**：L3 证明"进入"不证明"使用"；不得仅凭 L3 触发 reinforce（**台账 D-AP-05 红线含双条件：E14 未落地 + B04 回传链路未确认**）。

### 2.2 两条独立证据链

1. **请求关联链**：InjectionRecord → correlationId → DSH turn。
2. **内容映射链**：memoryId + memoryVersion → 组装追踪 → PromptAssemblyEvidence。

两链独立成立、均通过才认定 L3；集合比较以 `(memoryId, memoryVersion)` 为单位。

**⚠ 结构性互斥声明（红队 RT-01 → N-14，必须先裁）**：上述链条要求 DSH 在第一阶段声明 `memoryRefsInFinalPrompt`（(memoryId,memoryVersion) 集合），但 §一/M02 D19/接口 I-6 的五项禁令**禁止这些标识出现在 Memory Context 中**——DSH 只能声明它实际收到的内容。因此存在二选一：**(a)** 版本标识随 Context 下传（破 I-6 五项禁令，需重裁该契约）；**(b)** E-1 的声明由 Bridge 侧从注入数据复制（两链退化为同源，S2-01 单点缺陷）。此互斥与 F-CK-8（字段表达力）、F-CK-10/K-1(A线)（数据来源）**均不是同一问题**，无法在设计层自行消解——**L3 证据链全部设计（§2.2 以下）的闭合性以 N-14 裁决结果为前提**。

### 2.3 两阶段事件模型（v4.6 替换文本 A §2.3.4）

```
第一阶段：实际组装观察记录（prompt/assembled，语义收窄）
  组装完成后立即产生；是 Bridge 核验的输入；不代表证据有效、不代表 L3；
  即使后续证据 invalid 也必须保留（证据无效不抹除已发生操作）
  字段：sessionId/turn/correlationId/assemblyId/memoryRefsInFinalPrompt
        （语义=DSH 观察声明的 complete 版本集合，可为空）/evidenceRef
        ＋ E-1 扩展 hasUnknownVersions〔候选〕
        ↓
Bridge 执行检查 I / II / III
        ↓
第二阶段：有效 L3 确认记录（E-2 扩展，无载体）
  仅当整体证据有效性=passed 且特定版本满足 complete 及其他 L3 条件时产生；
  仅覆盖明确列出的 complete 版本；invalid/not_executable/not_started 不得伪装成确认
```

**检查③重定义**：第一阶段观察声明集合 ≡ 证据 complete 集合（逐项严格相等）——循环依赖由此解除（记录时序循环解除；语义依赖仍在——A5 5.3 双代理确认）。

### 2.4 三类检查（v4.6 替换文本 C §2.3.13 + A5 修订）

| 检查 | 内容 | 责任方 |
|---|---|---|
| I 证据有效性 | A1–A8 结构错误、字段/类型/约束、引用绑定、fragmentDigest | DSH 生产自检 + Bridge 独立核验（非对称：invalid 可拒 L3；任何 result 不替代核验） |
| II 请求关联与集合一致性 | ①关联链 ②快照↔versions ③观察声明≡complete 集 | **Bridge 专属** |
| III 版本内容完整性 | requiredComponents/componentMappings/fragmentRefs/mappingEvidenceRef/B1–B4 | Bridge 判定（语义充分性不要求） |

**A1–A8**（结构错误→invalid）：外层结构缺失/快照不一致/ID 错配/引用错位/重复声明/无效 fragmentId/摘要校验失败/**A8 兜底（援引须六要素：错误内容/对象/字段/证据位置/归类理由/依据规则；不得作绕过 A1–A7 的入口；无法评估→not_executable→整体不得 passed）**。

**B1–B4**（证据不足→unknown）：B1 映射证据缺失或不可核验 / B2 removalBasis 缺失 / B3 部分内容暂时不可读 / B4 来源无法确认。**v4.6 收窄**：引用字段存在但对象不可取得 → 不再作普通 unknown，改 not_executable（§2.3.6.5 第 12 条）。

### 2.5 L3 版本级确认条件【修订版：五条件合取】

```
L3 确认 = 检查 I 整体 passed
        ∧ 检查 II passed
        ∧ 整体证据有效性 = passed   ← 【S5-01/V-10 修订：新增合取项，待裁】
        ∧ 该版本检查 III 判定 complete
        ∧ 第一阶段/第二阶段记录关联一致
```

**修订依据**：v4.6 13.2 原文四条件无整体门——版本甲全通过+版本乙 not_executable 时，13.4a/c 判整体 not_executable、4.3b 禁发确认记录，但 13.2 字面对甲全满足 → 甲可绕过整体门获 L3（A5 S5-01 P0，Agent-07 发现+主审双验证）。**补入整体门为唯一 P0 修复方向**（A7 §2.1：推荐 A 补入；修 4.3b 放宽会削弱 L3——不推荐）。此项为设计修订提案，**待 V-10 裁决**。

### 2.6 语义规则（v4.6 §4.5–4.6 + 修订标注）

- 有效零 complete 确认：仅当整体 passed 且全部注入版本完成确定性判定（无 unknown）。
- unknown 表达：E-1 hasUnknownVersions 标记；任何 unknown 不得解释为已确认零 complete。
- 禁止伪装：invalid 不得经空数组伪装为有效零确认——确认只存在于第二阶段。
- **4.6a 非对称缺口（A5 Agent-07）**：4.6a 排除 unknown 却不排除 0 版本/全 partial，且无法区分"零注入"与"全移除"——待 V-9 裁决。

### 2.7 invalid 下字段效力（§2.3.6.5 第 10 条）

证据整体 invalid 时版本级 completeness 仅作诊断保留，不产生确认效力；不得据此构造 complete 集/生成确认/支持 L3；整体有效性≠passed（含 not_executable）时同样不产生确认效力（效力推广）。

### 2.8 三类记录分离（§2.3.6.5 第 11 条）

实际注入操作记录（如实保留）/ 证据无效记录（保留 invalid 事实+触发 A 类编号+关联+各维度结果）/ 有效 L3 确认记录（仅第二阶段产生）。三者不得互相替代。

## 三、HintFrame 与行为提示（X-1/X-9，心理层接口）

心理层 hint ≤50 字有界状态提示（"此刻：心情偏沉。倾向：话会变短。"式——修饰语非身份语）；不进人格段；只决定长度/语气/是否展开，不决定说什么。**X-7 修订提案〔待裁〕**：温度倍率两范围作废 → 有界状态提示 + DSH 采样参数未来重裁。

## 四、验收测试规范（v4.6 替换文本 E §2.3.14——**未执行任何测试**）

T-01~T-10 + U-01~U-04 覆盖：A5 重复声明→invalid；A1 缺失→invalid；A4 错位→invalid；A7 摘要不符→invalid；result=invalid 但版本 complete→仅诊断；A4/A7 不可执行→not_executable；全 removed→有效零确认；注入但证据 invalid→l3_rejected；合法 unknown→整体可 passed 仅 complete 可确认；迟到证据→evidence_timeout 不可恢复；B 类三分各态。**此表为未来测试规范，零执行。**

## 五、F-CK 实施核查清单（全部未核查——L3 不可用的根因）

| # | 核查对象 | 未通过时 |
|---|---|---|
| F-CK-1 | DSH 能否执行全部 A 类自检 | 证据不得标通过性 result；L3 不可用 |
| F-CK-2/3 | Bridge 能否取得 A4 跨对象引用 / removalBasis.ref | 对应项 not_executable→整体不得 passed |
| F-CK-4 | 不落盘正文下 A7 摘要比对（可能无解） | A7=生产方声明→整体不得 passed |
| F-CK-5 | 最终组装阶段=F8 | 观察记录产生点不可靠→不得作 L3 证据来源 |
| F-CK-6 | 证据无效记录独立保存 | 补存储设计后 L3 方可启用 |
| F-CK-7 | 消费方强制检查证据有效性 | 修复前 L3 不可用 |
| F-CK-8 | 现有字段能否表达规则（含 E-1） | 完成扩展前 L3 不可用 |
| F-CK-9 | 第二阶段确认记录存储消费（E-2） | 完成前不得产出任何确认记录 |
| F-CK-10 | **K-1：第一阶段观察集合数据来源（复制 vs 实际追踪）** | 复制→检查③退化为自洽检查、两阶段独立性受损（最大设计缺口） |

---

*证据链设计登记完毕（候选+修订）。全部待裁项见 14-OPEN-DECISIONS。*
