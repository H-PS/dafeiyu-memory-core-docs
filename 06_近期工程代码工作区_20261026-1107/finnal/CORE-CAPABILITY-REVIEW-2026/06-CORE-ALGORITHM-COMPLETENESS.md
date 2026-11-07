# F 路 —— 核心算法完整性审查（Memory Core R4.1）

## STATUS: COMPLETE

## §1 Headline

1. Memory Core R4.1 的核心算法主体（02 号 ALG-01…ALG-12）已具备完整的 17 要素规格：每个算法均有输入/输出/前置条件/步骤/公式/参数/边界/异常/幂等/测试/验收/禁止行为，参数全部标 EXP-DEFAULT 未裁决——这是本审查最重要的既有资产，检索与写入主干（ALG-01/02/03/05/06）可直接实现。
2. 但算法族覆盖面 ≠ 完整度：任务清单要求的约 90 个算法点中，多数检索/写入/去重算法达 MOSTLY_COMPLETE 以上，而摄取侧的 Confidence 计算、Importance 计算、M04 合并裁决（P4 未授权）、跨区块联合召回、Persona 动态人格上下文、FTS 分词器选型等仅 CONCEPT_ONLY 或 MISSING；21 个重点算法中 6 个存在 P0/P1 级缺口。
3. 存在系统性设计冲突：02 号 ALG-04 六特征全局线性排序与 11 号 §7.17 五阶段分组排序（禁止单一全局权重）形态矛盾；02 号 `retrieval.*`/`consolidation.*`/`context.*`/`lifecycle.*` 参数 ID 与 15 号注册表 `recall.*`/`memory.*` 三套命名并存，15 号 §7.17 虽列"待登记"，但生产配置使用前必须完成登记与 02 号回改。

## §2 全量算法清单（ALG-F01…）

按任务书 8 大算法族逐项建条目。完整度：COMPLETE / MOSTLY_COMPLETE / PARTIAL / CONCEPT_ONLY / CONFLICTED / MISSING。引用缩写：02=02-核心算法规范.md（NUL 清洗副本 EVIDENCE\B\02-clean.md，引用章节号）；09=09-记忆摄取Pipeline.md；11=11-MemoryAnchor与语义分类.md；13=13-Evidence与EmotionContext.md；14=14-ChatLLM读写API.md；15=15-参数注册表.md；05=05-可靠性与运维规范.md；06=06-测试与评估规范.md。

### 2.1 摄取族（09 号为主）

| 编号 | 名称 | 所属模块 | 完整度 | 关键缺口 |
|---|---|---|---|---|
| ALG-F-ING-01 | 原始输入处理（InputNormalizer，阶段 1） | Pipeline 协调 | MOSTLY_COMPLETE | 五类输入归一化规则+幂等键+失败处理齐备（09 §7.5）；文件解析为文本块的解析器实现规范缺（"解析为文本块"具体算法 TBD） |
| ALG-F-ING-02 | 长文本分割（多轮对话按话题边界切分） | Pipeline 协调 | PARTIAL | "按话题边界/时间窗/发言者切分"有原则（09 §7.5 输入形态 2）但无话题边界判定算法；话题边界是 LLM 判定还是规则未写 |
| ALG-F-ING-03 | 原子化（SemanticSplitter，阶段 5） | M03 | MOSTLY_COMPLETE | 6 项要求+2 禁令+正反例+幂等齐备（09 §7.9）；但切分执行者是规则还是 L1 未明说，`ingest.max_fragments_per_envelope` 与 15 号 `memory.max_units_per_input`（=20）是否同一参数未对齐 |
| ALG-F-ING-04 | 信息分类（MemoryClassifier，阶段 6） | M03 | MOSTLY_COMPLETE | 6 字段输出+互斥规则+示例齐备（09 §7.10）；分类器本体（prompt/规则混合）依赖 `ingest.model`（TBD 未选型） |
| ALG-F-ING-05 | Source Type 判定 | Pipeline 协调 | MOSTLY_COMPLETE | 五值在阶段 1 确定后不可变（09 §7.5 字段规范）；LLM 提议→ASSISTANT_INFERRED 有明文；边界（同一消息含指令+闲聊混合）处置缺 |
| ALG-F-ING-06 | Semantic Type 判定 | M03 | MOSTLY_COMPLETE | 六值单选+九宫决策树（11 §9.1 图）+模糊→NEEDS_REVIEW（09 §7.10）；六值定义依赖 11 §7 语义定义（未逐条读，标待核） |
| ALG-F-ING-07 | Anchor 提取（AnchorAssigner，阶段 9） | M03 | MOSTLY_COMPLETE | 分配规则表+2 禁令+no_anchor 拒绝齐备（09 §7.13，11 §7.13）；锚点判定执行者（LLM/规则）同样依赖未选型模型 |
| ALG-F-ING-08 | Entity 识别（EntityResolver，阶段 8） | M03 | PARTIAL | 别名/代词/昵称/关系指代/群体指代五类处理规则+硬约束齐备（09 §7.12）；但"最近先行词解析"无具体算法（距离窗口/语法依存未定义）；主体注册表（别名表）的数据结构与维护流程未见定义 |
| ALG-F-ING-09 | 关系识别（关系指代→relation_hint） | M03 | PARTIAL | 关系指代→"关系+指向"处理有（09 §7.12 表）；但 `relation_hint` 后续由谁消费、如何变成 `RELATIONSHIP`/`ABOUT` 关系记录，M04 P4 前无落地路径 |
| ALG-F-ING-10 | Evidence 绑定（EvidenceBinder，阶段 10） | M03+Pipeline | MOSTLY_COMPLETE | 绑定 6 规则+4 类失败+反向追溯+span 齐备（09 §7.14，13 §7.1/7.2）；字符区间定位由 LLM 输出的可靠性验证缺（LLM 偏移错误未给校验算法） |
| ALG-F-ING-11 | LLM 提取（LLMExtractor，阶段 4） | M03 | MOSTLY_COMPLETE | 输入输出+4 类失败+降级矩阵+幂等齐备（09 §7.8，7.23）；模型本体 TBD（ingest.model 未选型），属实施项非设计缺口 |
| ALG-F-ING-12 | 确定性验证（CandidateValidator，阶段 14） | Pipeline 协调 | COMPLETE | 9 项校验逐项判定+结构化 ValidationReport+幂等齐备（09 §7.18） |
| ALG-F-ING-13 | 候选生成与价值判定（MemoryValueEvaluator，阶段 3） | M03 | PARTIAL | 8 类判定码+禁令齐备（09 §7.7）；但 8 类判定的执行机制未定义：判定码 SAVE_STABLE_PREFERENCE"反复表达且跨会话一致"如何判定（几次算反复？查历史记忆用什么查询？）无算法 |
| ALG-F-ING-14 | TemporalResolver（7 类时间语义） | M03 | MOSTLY_COMPLETE | 7 类定义+禁令+失败处理齐备（09 §7.11）；相对时间→绝对时间的解析算法（"上周三"基准确认）标 temporal_unresolved 可接受，故不算缺失 |

### 2.2 质量族

| 编号 | 名称 | 所属模块 | 完整度 | 关键缺口 |
|---|---|---|---|---|
| ALG-F-QLT-01 | 重要性（importance 计算） | M03/M04 | MISSING | 全文档检索无 importance 计算算法：02 §7.5 3.5 只定义"只用于过滤与并列打破，不入公式"；09 §7.7 禁止把价值判定写进 importance；但 importance 本身由谁、按什么公式赋值，**无任何文档定义**（memory_objects 字段存在，计算算法缺失） |
| ALG-F-QLT-02 | 置信度（confidence 计算） | M03 | PARTIAL | MemoryClassifier 输出 confidence 0–1（09 §7.10），禁止 LLM 自报直接赋值（15 §7.5 `memory.min_confidence` A03）；但**如何由多信号合成 confidence**（ALG-07 步骤 9"规则/统计信号"仅一句话，02 §7.8）无公式、无权重、无信号清单 |
| ALG-F-QLT-03 | 可靠性（source quality S 表） | M02 | MOSTLY_COMPLETE | S 查表 8 行+禁令齐备（02 §7.5 3.3）；PROPOSED+EXP-DEFAULT，标定实验待做（OI-02-07），属参数待标定非算法缺失 |
| ALG-F-QLT-04 | 去重（DuplicateDetector，阶段 12） | M03 | MOSTLY_COMPLETE | 4 类重复+4 禁令+幂等齐备（09 §7.16）；`dedupe.semantic_threshold` 有参数（09 §8.7），语义相似度算法（向量/3-gram 未统一指定）待标定 |
| ALG-F-QLT-05 | 相似判断（3-gram Jaccard + 向量） | M02/M03 | MOSTLY_COMPLETE | 3-gram Jaccard 定义+阈值 0.9+边界齐备（02 §7.5 3.4，ALG-08 §5.1）；向量阈值 vector_dup_threshold TBD（依赖空间标定，OI-02 表内） |
| ALG-F-QLT-06 | 冲突检测（ConflictDetector，阶段 13） | M03 | MOSTLY_COMPLETE | 5 类冲突+4 禁令+不裁决边界齐备（09 §7.17）；与 ALG-08 八类冲突（02 §7.9 5.2）**粒度不一致**（5 vs 8），映射关系未定义（见 §4 C-06） |
| ALG-F-QLT-07 | 版本（版本演进/Supersede） | M01/M04 | MOSTLY_COMPLETE | ALG-01 事务五步+版本链齐备（02 §7.2）；supersede 决策属 ALG-08（FUTURE）；版本字段与审计完整 |
| ALG-F-QLT-08 | 合并（merge 六前提） | M04 | PARTIAL | 六前提+幂等+禁止行为齐备（02 §7.9 5.3）；但 M04 为 FUTURE/P4 未授权，合并执行路径当前不存在；"合并后字段覆盖比对"验收方法有（02 §7.9 16） |
| ALG-F-QLT-09 | 拆分（oversplit 处理） | M03 | MOSTLY_COMPLETE | 超 `max_fragments_per_envelope` → oversplit → 人工复核，不静默丢弃（09 §7.9）；有定义 |
| ALG-F-QLT-10 | 事实更新（fact update/supersede 链） | M01/M04 | MOSTLY_COMPLETE | 版本演进+VERSION_CONFLICT+tombstone 齐备（02 §7.2，05 §7.11，13 §7.4.6）；M04 裁决 FUTURE 依赖同 ALG-F-QLT-08 |
| ALG-F-QLT-11 | 晋升（L1/L2→L3 promotion） | M04 | CONCEPT_ONLY | X-2 明确 L1 不自动晋升（11 §7.12 规则 5）；Session Memory 不自动晋升（00 §7 371 行）；正向晋升（L2→L3）的触发条件、审批、幂等**无算法定义**（A 路已知结论 A-C003 tier 转换 0 条一致） |
| ALG-F-QLT-12 | 降级（demotion） | M04 | MISSING | 同上：L3→L2/L1 降级规则（何时、由谁、依据什么信号）零定义 |
| ALG-F-QLT-13 | 过期（expiry） | M04/Dream | PARTIAL | 过期判断 7 因素综合+禁止单一 TTL 有原则（02 §7.11 5.1）；但综合判断的**执行算法**（权重/规则/周期）无定义；`lifecycle.archive_after_days` TBD |
| ALG-F-QLT-14 | 复核（NEEDS_REVIEW 处理） | M03/M04 | PARTIAL | 候选状态机 6 态+DEFERRED→NEEDS_REVIEW 超时路径齐备（09 §7.21，7.22）；但 NEEDS_REVIEW 队列的**人工复核界面/流程/SLA** 未定义（实施项），复核通过→AUTO_ACCEPT 通道未授权（CR-01，schema const false） |
| ALG-F-QLT-15 | 淘汰（淘汰/删除） | M01/M04 | MOSTLY_COMPLETE | 删除传播 7 步+tombstone+防复活齐备（05 §7.11，06 §7.4.1 KT-M01-03，13 §7.4.6 DEL-1…7） |

### 2.3 检索族（02 号为主）

| 编号 | 名称 | 所属模块 | 完整度 | 关键缺口 |
|---|---|---|---|---|
| ALG-F-RET-01 | Query Understanding（查询规范化七步） | M02 | MOSTLY_COMPLETE | S1–S7+5.1 禁令（不删否定词/不 LLM 改写）+失败 E02-04 齐备（02 §7.3 5）；"提取显式实体/主题"的实体抽取方法未指定（NER 规则/词表/模型未定） |
| ALG-F-RET-02 | Intent Classification（9 类意图） | M02 | PARTIAL | 9 类枚举+映射表+保守回退+intent_confidence 有（11 §7.19，8.2）；但 **Intent Resolver 本体算法无定义**：9 类如何从 query 判出（关键词表/分类模型/规则）零描述，MUST NOT 用 LLM 改写查询但未禁止用 LLM 判意图（J2 只禁检索期情绪推断）——判定机制是最大缺口 |
| ALG-F-RET-03 | Anchor Router（意图→锚点路由） | M02 | MOSTLY_COMPLETE | 映射表 9 行+AnchorSet 结构+配额+互斥路由齐备（11 §7.17 阶段 1，§7.19，8.1；anchor.schema.json weights/intent_routing 已落地配置） |
| ALG-F-RET-04 | Structured Filter | M02 | MOSTLY_COMPLETE | 允许字段+硬排除+只过滤不计分（OI-022/09 结论）齐备（02 §7.3 5.2；11 §7.17 阶段 2） |
| ALG-F-RET-05 | FTS 召回 | M02 | MOSTLY_COMPLETE | 字段权重+bm25+防注入+Top-50+可替换分词（02 §7.3 5.3）；**中文分词器选型 tokenizer_id TBD**（OI-02-04），V-02 验收依赖"分词方案记录"但方案本体未定 |
| ALG-F-RET-06 | Vector 召回 | M02 | MOSTLY_COMPLETE | FLAT 八项验证+Embedding Adapter 四接口+空间隔离齐备（02 §7.3 5.4/5.5）；FLAT 执行路径 P0 VERIFY（OI-02-01），未证即 BLOCKED——设计已给判定门，不算算法缺失 |
| ALG-F-RET-07 | Embedding（文档/查询嵌入） | M01 | MOSTLY_COMPLETE | embedQuery/embedDocument/getProfile/validateCompatibility 契约+失败语义齐备（02 §7.3 5.4；embedding.schema.json async_write=true）；模型 TBD 属实施 |
| ALG-F-RET-08 | RRF 融合 | M02 | COMPLETE | 公式+示例+并列打破+NaN 防护+缺失通道=0+禁混合相加全齐备（02 §7.4），参数 k=60/权重已落 retrieval.schema.json |
| ALG-F-RET-09 | Candidate Merge（同通道去重合并） | M02 | COMPLETE | 同通道重复项合并保留最小 rank+记录合并次数（02 §7.4 步骤 2） |
| ALG-F-RET-10 | Reranking（六特征规则排序） | M02 | CONFLICTED | 02 §7.5（ALG-04）全局线性加权 `Score=αR+βE+γN+δT+εS−ζP` vs 11 §7.17 阶段 3"禁止单一固定权重全局排序，MUST 分锚点组独立配置权重"——两文档形态直接矛盾；06 §7.10 消融 ABL-C 又按 02 的全局六特征写死。冲突双方原文见 §4 R-13 |
| ALG-F-RET-11 | Importance 特征使用 | M02 | MOSTLY_COMPLETE | 只用于过滤+并列打破、MUST NOT 入公式、验收含数值对照（02 §7.5 3.5）——但前提是 importance 有值（依赖 ALG-F-QLT-01 缺失） |
| ALG-F-RET-12 | Confidence 特征使用 | M02 | PARTIAL | confidence 在返回 9 字段中（14 §8.5）；**不进入 ALG-04 六特征**（公式里无 confidence 项）——置信度对排序零影响是否符合预期未说明；作为过滤阈值 `memory.min_confidence` 只作用于摄取侧，检索侧过滤阈值 `retrieval.rank.min_importance`≠confidence |
| ALG-F-RET-13 | Reliability 特征（S 来源质量） | M02 | MOSTLY_COMPLETE | S 表 8 行+归一化+禁止替代 validation_state（02 §7.5 3.3）；06 §7.10 表中 S 定义为"来源缺失→0"，与 02"无 evidence⇒0 且 available=true"一致 |
| ALG-F-RET-14 | Recency（T 时间适配） | M02 | MOSTLY_COMPLETE | `T=exp(−λ_type·Δt)`+按类型 λ 表+历史查询改时间窗命中+禁旧记忆不可找回（02 §7.5 3.2）；λ 表 EXP-DEFAULT 待标定（OI-02-08） |
| ALG-F-RET-15 | Temporal Decay（独立衰减） | M02/M04 | PARTIAL | 排序内衰减（同上）COMPLETE 级；但作为**独立生命周期衰减**（validity 随时间衰减、触发过期）只有 D=exp(−decay_rate·Δt)（情绪侧 13 §7.9）与原则性条款（02 §7.11 5.2"衰减只影响排序不删除"）——记忆本身的 validity 衰减算法未独立成条 |
| ALG-F-RET-16 | Context Budget（Token 预算） | M02 | COMPLETE | 真实 Tokenizer+序列化完整成本+五类计量+截断优先级+估算禁令齐备（02 §7.6；14 §7.12.1 四类单位；15 §7.15 双预算先到者裁决）；Tokenizer 选型 TBD 属实施 |
| ALG-F-RET-17 | Context Assembly（GreedySelect 八步） | M02 | COMPLETE | 边际价值 U=Relevance−λ·Redundancy+分区+禁令 8 条+确定并列打破齐备（02 §7.7）；λ=0.5 已落 retrieval.schema.json redundancy_lambda |
| ALG-F-RET-18 | 权威回源九步 | M02 | COMPLETE | 九步顺序+每步失败处置+禁跳过版本/判权（02 §7.3 5.6） |
| ALG-F-RET-19 | Evidence 回源（related_evidence 六步） | M02+M01 | COMPLETE | 六步+PERM-1…4+revoked 处置齐备（13 §7.4.1—7.4.5）；`recall.evidence_expansion_limit=3` 已落 schema |
| ALG-F-RET-20 | 跨区块/多区块联合检索 | M02 | MISSING | B 路已知结论 B-C001（无自动跨区块联合召回）在本路证实：02 ALG-02 三通道（fts/vector/structured）均以单 memory_id 为单元；11 五阶段按锚点分组但组间无"同一实体的画像+事件+关系联合装配"算法；`memory_evidence_links`/`memory_relations` 不参与召回通道——跨对象联合召回无算法 |

### 2.4 组织族

| 编号 | 名称 | 所属模块 | 完整度 | 关键缺口 |
|---|---|---|---|---|
| ALG-F-ORG-01 | 区块归类（画像分区整合） | M04 | PARTIAL | 11 分区枚举+profile.max_sections=11+版本保留有（15 §7.12 profile.*；12 号文档存在但未逐条读）；M04 P4 未授权，整合执行算法（从多条记忆合成一个 section）仅 12 号文档层面 |
| ALG-F-ORG-02 | 多区块关联（画像↔事件↔关系连接） | M04 | CONCEPT_ONLY | 画像与事件拆两条+用 `ABOUT`/`DERIVED_FROM` 关系连接（09 §7.13 禁令 1）；但关系**建立算法**（何时建、判定依据、幂等）在 M04 FUTURE 内无规格 |
| ALG-F-ORG-03 | 关系传播（删除/撤销传播到派生对象） | M01/M04 | MOSTLY_COMPLETE | Observation 失效三态 needs_recompute/invalidated/restricted+Relation 降级（02 §7.9 5.5）；删除传播 7 步（01 §7.16，13 §7.4.6 DEL-3） |
| ALG-F-ORG-04 | 人物关系关联（RELATIONSHIP 维护） | M04 | CONCEPT_ONLY | 关系变化冲突检测有（09 §7.17 类型 4）；但关系对象的建立/更新/关闭生命周期算法无定义（B 路已知结论 B-C 系列一致） |
| ALG-F-ORG-05 | 事件与人物关联（participants/ABOUT） | M03/M04 | PARTIAL | Event.participants MUST 经 EntityResolver（13 §7.7.1）；候选侧 subject 解析有；但 Event 对象与 MemoryUnit 的关联建立算法（event_refs 何时生成）无独立定义 |
| ALG-F-ORG-06 | 项目与事件关联（project_scoped） | M01/M05 | MOSTLY_COMPLETE | project_scoped 作为 scope 枚举+PROJECT_MISMATCH 拒绝码齐备（02 §7.10；04 号权限矩阵）；**无 Project 实体**（B 路 B-C 结论一致），项目只是 scope 标签 |
| ALG-F-ORG-07 | 跨区块检索 | M02 | MISSING | 同 ALG-F-RET-20 |
| ALG-F-ORG-08 | 优先级处理（人物优先/事件优先路由） | M02 | MOSTLY_COMPLETE | 人物 6 步顺序+三原则+4 类查询路由+差异化矩阵齐备（11 §7.14—7.18）；与 ALG-F-RET-10 的全局排序冲突需裁决（见 §4 R-13） |

### 2.5 情绪族（13 号为主；三项 DISABLED 边界清晰）

| 编号 | 名称 | 所属模块 | 完整度 | 关键缺口 |
|---|---|---|---|---|
| ALG-F-EMO-01 | 情绪提取（EmotionExtractor，阶段 11） | M03 | MOSTLY_COMPLETE | 契约完整（09 §7.15）；auto_infer=false 关闭态只接受 USER_EXPLICIT 有明文；**关闭态下 USER_EXPLICIT 情绪的写入路径**（谁调用、怎么入池）仅"条件可跳过"隐含，无正向算法规格（见 §4 E-19） |
| ALG-F-EMO-02 | 情绪分类（15 受控标签） | M03 | MOSTLY_COMPLETE | 15 标签逐字+越界→neutral+raw_label 保留+label_dict 落 emotion.schema.json（13 §7.8.1；15 §7.8 词表说明） |
| ALG-F-EMO-03 | 情绪证据（evidence_refs 强制） | M03 | MOSTLY_COMPLETE | `event_id=null` 时仍 MUST 有 evidence_refs（09 §7.15 规则 6）；EmotionContext.source_type 仅 2 值（13 §7.7.2） |
| ALG-F-EMO-04 | 强度（intensity 0–1） | M03 | MOSTLY_COMPLETE | 取值域+越界 VALIDATION_ERROR+禁止 0.5 冒充（13 §7.8.2）；intensity 本体如何从文本判定（USER_EXPLICIT 显式时）无算法——同 confidence 缺口 |
| ALG-F-EMO-05 | 时间衰减（D 系数） | Psychology 接口 | MOSTLY_COMPLETE | `D=exp(−decay_rate·Δt)`+单调+无时间 D=0（13 §7.9.1；emotion.schema.json decay_rate=0.1）；默认关闭不影响算法定义完整 |
| ALG-F-EMO-06 | 附件关联（event_id/emotion_refs） | M03 | MOSTLY_COMPLETE | SEP-2 独立对象+event_id 关联+null 禁 ΔE（13 §7.7.3） |
| ALG-F-EMO-07 | 历史检索（EMOTION_QUERY 路由） | M02 | MOSTLY_COMPLETE | 查询路由定义完整且默认关闭 const false（11 §7.19 行 7；emotion.schema.json query_enabled） |
| ALG-F-EMO-08 | 当前影响（ΔE 公式） | Psychology 接口 | MOSTLY_COMPLETE | ΔE=I×C×R×D+8 门禁+两组演算示例+缺失归零齐备（13 §7.9）；**R（话题关联度）计算算法未定义**（"由关联度计算得出（PROPOSED，参数化待标定）"一句话，无公式）；influence_enabled=false 默认关闭，启用须裁决 |
| ALG-F-EMO-09 | 与事实分离（SEP-1…8） | 全模块 | COMPLETE | 8 条硬规则+对照示例+门禁 8 齐备（13 §7.7.3，§7.10） |

### 2.6 Persona 族

| 编号 | 名称 | 所属模块 | 完整度 | 关键缺口 |
|---|---|---|---|---|
| ALG-F-PER-01 | Persona 初始化 | Persona（FUTURE） | MISSING | Persona 模块整体 FUTURE（00 §7.7 行 157；CR-08B 待单独呈报）；00–15 十六份文档内**无任何 Persona 算法正文**，仅边界条款（Memory Core 不负责人格生成，00 行 300） |
| ALG-F-PER-02 | 配置保存 | Persona | MISSING | 同上 |
| ALG-F-PER-03 | 读取（授权视图 C） | Persona | CONCEPT_ONLY | 00 权限矩阵行 329：Persona=授权视图 C 读取——仅一行边界，无算法 |
| ALG-F-PER-04 | 版本管理 | Persona | MISSING | 同 ALG-F-PER-01 |
| ALG-F-PER-05 | 动态人格上下文 | Persona | MISSING | 同上 |
| ALG-F-PER-06 | 长期记忆参与 | Persona/Memory Core | PARTIAL | 记忆→Persona 的供给面只有"授权视图 C + 宿主规则优先于记忆"（08 号行 327/338）；注入算法（选哪些记忆、预算占比 OI-08 记忆预算占比 TBD）缺 |
| ALG-F-PER-07 | EmotionContext 参与 | Persona | MISSING | 情绪默认不注入（J6/CR-02）双保险已锁（const false）；Persona 侧如何消费情绪无定义（FUTURE 一致） |
| ALG-F-PER-08 | Prompt 组装（记忆部分） | M02+Host Adapter | MOSTLY_COMPLETE | ContextPackage 结构+分区+禁令 8 条+宿主不覆盖人格规则齐备（02 §7.7，8.2；08 号 §7.4） |
| ALG-F-PER-09 | Token 预算 | M02 | COMPLETE | 同 ALG-F-RET-16 |
| ALG-F-PEP-10 | Persona 更新 | Persona | MISSING | 同 ALG-F-PER-01 |

### 2.7 存储族（05 号+02 ALG-01 为主）

| 编号 | 名称 | 所属模块 | 完整度 | 关键缺口 |
|---|---|---|---|---|
| ALG-F-STO-01 | SQLite 权威写入 | M01 | COMPLETE | ALG-01 十七要素全齐（02 §7.2）：事务七步（注释内文本为"事务五步+乐观并发"，步骤实列 7 条，见 §4 S-21 命名差异）+异常+幂等+禁止行为 |
| ALG-F-STO-02 | Outbox | M01 | COMPLETE | 同事务写+状态机+积压续跑+dead_letter+监控最老事件年龄（02 §7.2；05 §7.5；06 KT-M01-01/02） |
| ALG-F-STO-03 | FTS 更新 | M01 Worker | MOSTLY_COMPLETE | Outbox→Worker→FTS5 驱动+旧版本晚到丢弃+重建九步齐备（05 §7.4.1）；**分词器 tokenizer_id TBD**（OI-02-04）+索引文本投影"无摘要回退策略 TBD"（05 §7.4.1 步 3，D-AP-04 遗留） |
| ALG-F-STO-04 | LanceDB 更新 | M01 Worker | MOSTLY_COMPLETE | 同上+空间隔离+八项验证（05 §7.4.2；02 §7.3 5.5）；FLAT P0 VERIFY |
| ALG-F-STO-05 | Embedding 空间迁移 | M01 | MOSTLY_COMPLETE | 换模型新建空间+全量重建+旧空间退役前保留+禁混入（05 §7.4.2；vector.schema 组；B-011 旧空间退役保留）——**新旧空间双写并存期的查询路由**（查旧还是查新）未明说，仅"退役前保留" |
| ALG-F-STO-06 | 删除 | M01/M04 | COMPLETE | 逻辑删除+tombstone+传播 7 步+防复活+幂等删除（05 §7.11；06 KT-M01-03；13 DEL-1…7） |
| ALG-F-STO-07 | 恢复 | M01 | COMPLETE | 十步恢复+四分一致性+对账脚本+备份规范（05 §7.3/7.6/7.7） |
| ALG-F-STO-08 | 索引重建 | M01 | COMPLETE | FTS5 九步+LanceDB 九步+回滚能力（05 §7.4） |
| ALG-F-STO-09 | 幂等 | 全链 | COMPLETE | 四层幂等键表（09 §7.22）+ALG-01 幂等+Outbox 幂等+M05 复用域 |
| ALG-F-STO-10 | 失败重试 | 全链 | COMPLETE | 可重试/不可重试错误清单+指数退避+DEFERRED 落盘+TTL→NEEDS_REVIEW（09 §7.22；05 §7.5；indexing.outbox.retry_max=8 已落 schema） |

## §3 总表（重点算法 12 维度概览）

| # | 名称 | 输入 | 输出 | 公式/步骤 | 参数（唯一注册？） | 边界（空/重/冲突） | 异常 | 测试 | 完整度 | 可直接实现？ |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | ALG-01 M01 写入 | Memory Object+Provenance+idempotency_key | {memory_id,version,outbox_event_id} | 事务五步/七步（02 §7.2 5） | storage.* 4 个（15 §7.17 待登记，P1） | 幂等重放/版本冲突/部分提交全覆盖 | E02-01…03 | T02-01…04+T-M01-01…11 | COMPLETE | 是 |
| 2 | ALG-02 多路检索 | RetrievalRequest | RetrievalResponse+trace | 规范化七步+三通道+回源九步 | retrieval.*（02 §7.3 13）→ recall.* 映射（15 §7.1.3） | 空查询/通道失败/权限拒绝全覆盖 | E02-04…07 | T02-05…08 | MOSTLY_COMPLETE | 是（分词器除外） |
| 3 | ALG-03 RRF | 通道结果集 | ScoredCandidate[] | RRF(d)=Σ w_r/(k+rank_r(d))（02 §7.4 5） | recall.rrf_k=60 已注册；通道权重待登记 | 缺失通道=0/并列打破/NaN 防护 | E02-08 | T02-08/09 手算一致 | COMPLETE | 是 |
| 4 | ALG-04 六特征排序 | RRF 候选+元数据 | ScoredCandidate[]（final_score） | Score=αR+βE+γN+δT+εS−ζP（02 §7.5） | 6 权重+λ 表+S 表（15 §7.17 待登记） | 特征缺失权重重分配/max=min/全缺失 Score=0 | E02-09 | T02-10…12 | **CONFLICTED** | 否（先裁决 02 vs 11 排序形态） |
| 5 | ALG-05 Token Budget | 已排序候选+budget_tokens | selected/skipped+TokenUsage | Cost(d)=TokenCount(Serialize(d))（02 §7.6） | recall.budget_tokens=1200 已注册+hard_cap 待登记 | 超长单条/预算耗尽/Tokenizer 不可用 | E02-10/11 | T02-13/14 | COMPLETE | 是（Tokenizer 选型后） |
| 6 | ALG-06 Context Builder | 已判权已计量候选 | ContextPackage | GreedySelect 八步 U=R−λ·Red（02 §7.7） | redundancy_lambda=0.5/max_items=20 已注册 | 冲突并列保留/版本不一致剔除/全剔除空 Context | E02-12/13 | T02-15/16 | COMPLETE | 是 |
| 7 | ALG-07 M03 候选提取（FUTURE） | 授权来源 | Candidate[] | 十步+质量十项（02 §7.8） | extraction.*（未入 15 域集合） | 空来源/模型失败/整批失败 | INTERNAL_ERROR | T02-17 | MOSTLY_COMPLETE（P4 后实施） | 部分（需 M03 授权+模型选型） |
| 8 | ALG-08 M04 整合（FUTURE） | Candidate+既有记忆 | ConsolidationDecision | 9 步+重复四类+冲突八类+六前提（02 §7.9） | consolidation.* 4 个（未入 15 域集合） | 决策失败 hold/Observation 失效/幂等 | VERSION_CONFLICT | T02-18 | MOSTLY_COMPLETE（P4 后实施） | 部分（confirmed 通道未授权） |
| 9 | ALG-09 M05 权限（FUTURE 契约预留） | PermissionContext | PermissionDecision 四字段 | 十步+默认拒绝十原则+派生交集（02 §7.10） | permission.* 3 个已注册（m05_mode=contract_deny） | error/timeout/未知全拒+派生最严格交集 | POLICY_ERROR/TIMEOUT | T02-19/20+P-01…11 | COMPLETE（契约级） | 是（contract_deny 模式即可跑） |
| 10 | ALG-10 生命周期 | 状态转换请求 | 转换结果+audit | 授权矩阵+七步（02 §7.11） | lifecycle.*（未入 15 域集合；archive_after_days TBD） | 非法转换/越权/状态已变 | PERMISSION_DENIED | T02-21 | MOSTLY_COMPLETE | 部分（T4 授权矩阵依赖 M04/M05 落地） |
| 11 | ALG-12 评估 | 测试集 | 指标集+五态报告 | Recall/HitRate/Precision/MRR/NDCG 公式+消融 A–F+参数七步（02 §7.13；06 §7.9/7.10） | eval.* 待登记（P3） | 空 gold→null/未命中 0/环境漂移 NOT RUN | E02-16 | T02-23 | MOSTLY_COMPLETE | 是（测试集冻结后） |
| 12 | ΔE 情绪影响（13 §7.9） | EmotionContext+话题关联 | ΔE（仅运行时） | ΔE=I×C×R×D+8 门禁（13 §7.9） | emotion.* 7 个已注册（3 DISABLED const） | 无事件 ΔE=0/R 低于阈值归零/缺失归零 | — | 门禁 8 条单测（emotion.schema gates） | MOSTLY_COMPLETE | 公式可算，**R 算法缺**，默认关闭 |
| 13 | 16 阶段摄取 Pipeline | IngestRequest 五类 | memory_id/COMMITTED 回执 | 阶段 1–16 逐段规格（09 §7.5–7.20） | ingest.* 6 个已注册+memory.* 4 个已注册 | DEFERRED/REJECTED/降级矩阵全覆盖 | E-09-01… | 09 §12+T-M01 系列 | MOSTLY_COMPLETE | 是（模型选型+话题切分除外） |
| 14 | 五阶段检索装配（11 §7.17） | query+身份 | ReadResponse | 意图→锚点→组内排序→权限→证据扩展 | anchor.* 4+recall.* 已注册 | EMOTION 组恒 0/低置信保守路由 | degraded 标记 | 11 §12 锚点路由用例 | MOSTLY_COMPLETE | 是（与 ALG-04 的冲突裁决后） |
| 15 | related_evidence 反向追溯 | memory_id+read_mode | 原句集 | 六步（13 §7.4.1） | recall.evidence_expansion_limit=3 已注册 | revoked/权限/哈希不符 | evidence_missing/integrity_error | 13 §12.1 | COMPLETE | 是 |
| 16 | Outbox→索引同步 | outbox_events | FTS5/LanceDB 文档 | KT-M01-02 旧版本丢弃+重建九步（06 §7.4.1；05 §7.4） | indexing.* 3 个已注册 | 重复事件/晚到事件/删除幂等 | dead_letter | T-M01-09/10/11 | MOSTLY_COMPLETE | 是（分词器除外） |

## §4 21 个重点算法逐条结论

标注：EXISTING DESIGN 给文件+章节；DESIGN COMPLETENESS 按输入/输出/步骤/条件/模块责任/边界评估；RECOMMENDED ADDITION 均标 NEW PROPOSAL（未批准）。

**1. 摄取与原子化（SemanticSplitter）**
- EXISTING DESIGN：09 §7.9（6 项要求 S1–S6、2 禁令、正反例 4 条、幂等键 `(envelope_id, splitter_version, extraction_result_hash)`）。
- DESIGN COMPLETENESS：MOSTLY_COMPLETE——输出（Fragment 字符区间+context_ref）、边界（oversplit→人工复核不丢弃）、异常（context_lost→REJECTED）齐备；但"切分依据只允许语义边界"的执行者未指定（L1 模型输出 or 规则后处理），且 `ingest.max_fragments_per_envelope`（09 §8.7）与 15 号已注册 `memory.max_units_per_input`（默认 20，15 §7.5）疑似同一参数两个 ID，违反 OI-023"同一参数不得有第二个 ID"。
- RECOMMENDED ADDITION（NEW PROPOSAL）：①在 09 §8.7 将该上限参数改为引用 `memory.max_units_per_input`；②补一节"切分执行模式"：L1 抽取输出候选断言边界+确定性校验器按 S1–S6 复核，复核失败回退整段。

**2. 分类与 Anchor（MemoryClassifier+AnchorAssigner）**
- EXISTING DESIGN：09 §7.10（6 字段、互斥、多标签示例）、09 §7.13（分配规则表+2 禁令）；11 §7.13（8 锚点定义）。
- DESIGN COMPLETENESS：MOSTLY_COMPLETE——输入/输出/边界（越界 semantic_type→REJECTED 禁就近猜）齐备；缺口与上条同源：分类与锚点判定执行者依赖 `ingest.model`（TBD），且 11 §7.10 要求 `semantic_type` 判定"MUST NOT 与价值判定码混用"，两者 prompt 边界未写。
- RECOMMENDED ADDITION（NEW PROPOSAL）：在 10 号 LLM 调用体系补"分类/锚点/时间三类 L1 任务的输出 Schema 与 prompt_version 登记"，并给 `semantic_type` 六值判定的一页判定表（含边界例）。

**3. Evidence 绑定（EvidenceBinder）**
- EXISTING DESIGN：09 §7.14（6 规则+4 失败）；13 §7.1/7.2（双层结构、字段表、七问）；`memory_evidence_links` 主键+反向索引（13 §7.1.3）。
- DESIGN COMPLETENESS：MOSTLY_COMPLETE——模块责任（Pipeline 编排+M03 草案+M01 持久化）、权限（visibility 取来源交集）、失效传播（revoked→evidence_missing）齐备。缺口：碎片→字符区间的定位若由 LLM 产生，**无偏移校验算法**（错位即静默绑定错句）。
- RECOMMENDED ADDITION（NEW PROPOSAL）：绑定后增加确定性校验步：对 `original_text[source_locator]` 与碎片做相似度核对（如 ≥0.8 或包含关系），不符→DEFERRED（`span_mismatch`）。

**4. 重要性与置信度**
- EXISTING DESIGN：仅字段与使用边界（02 §7.5 3.5 importance 只过滤/并列；09 §7.10 confidence 输出；15 §7.5 三阈值）。
- DESIGN COMPLETENESS：**缺失**——importance 无赋值算法（谁写、按什么公式、初始值、是否随事件更新）；confidence 无合成公式（ALG-07 步骤 9 仅一句"规则/统计信号"）。两字段是 memory_objects 权威字段（01 §8.3），权威字段无写入算法属 P0。
- RECOMMENDED ADDITION（NEW PROPOSAL，P0）：新增《重要性/置信度计算规范》一章：confidence = f(source_type 基础分, LLM 自报折扣, 重复一致性, 冲突扣减)，全部权重 EXP-DEFAULT；importance 初版可定义为常量默认值+M04 整合时更新（两档），避免与排序公式耦合。

**5. 去重（DuplicateDetector）**
- EXISTING DESIGN：09 §7.16（4 类+4 禁令）；02 §7.5 3.4（3-gram Jaccard）；13 §7.4.2（Evidence content_hash 去重）。
- DESIGN COMPLETENESS：MOSTLY_COMPLETE——跨主体/跨 scope 禁合并、只检测不合并边界清晰。缺口：semantic_threshold 的相似度算法未统一（09 §7.16 用"语义相似度"，02 用 3-gram/向量两条路径），跨层一致性需一个基准实现说明。
- RECOMMENDED ADDITION（NEW PROPOSAL，P2）：在 15 号登记 `dedupe.similarity_algorithm`（枚举 jaccard3/vector）作为配置项，两文档引用同一 ID。

**6. 冲突处理（检测+裁决分离）**
- EXISTING DESIGN：09 §7.17（5 类，检测不裁决）；02 §7.9 5.2（8 类，M04 裁决）。
- DESIGN COMPLETENESS：MOSTLY_COMPLETE——"检测/裁决分离+旧记忆保留至 M04"红线清晰；但 5 类与 8 类**无映射表**（如 09 `state_change` 对应 02 `temporal_conflict` 还是独立类？09 无 `numeric_conflict`/`source_credibility`），两套枚举并存将导致 trace 字段不可对齐。
- RECOMMENDED ADDITION（NEW PROPOSAL，P1）：在 02 §7.9 或 09 §7.17 增加映射表：09 五类 ⊂ 02 八类，注明 09 阶段 13 只产出 02 八类中的子集标志位；统一 `conflict_kind` 枚举到 contracts/schemas。

**7. 版本（版本链与 supersede）**
- EXISTING DESIGN：02 §7.2（ALG-01 步骤 4 版本快照）；02 §7.9（supersede 决策）；05 §7.11；06 KT-M01-02（旧版本事件晚到丢弃）。
- DESIGN COMPLETENESS：MOSTLY_COMPLETE——乐观并发+版本单调+审计齐备；supersede 的触发判定在 M04（FUTURE）内，V0.9 若不带 M04，则事实更新只能走"新建+手工裁决"，需在 API 层明确该限制。
- RECOMMENDED ADDITION（NEW PROPOSAL，P2）：14 号写明 V0.9 无 M04 时 `propose` 的 supersede 类候选一律落 NEEDS_REVIEW 终态，不自动演进。

**8. Tier 晋升降级**
- EXISTING DESIGN：11 §7.12（3×6 矩阵、L1 不自动晋升 X-2、tier≠优先级）；09 §7.21（CR-01 AUTO_ACCEPT≠confirmed）。
- DESIGN COMPLETENESS：**缺失（与 A 路 A-C003 一致）**——矩阵只是"典型基准"；L2→L3 晋升与 L3→L2 降级的触发条件/审批者/幂等/审计零定义；`lifecycle_candidate`（M03 输出）→ 最终 tier（M04 决定）之间无转换算法。
- RECOMMENDED ADDITION（NEW PROPOSAL，P1）：V0.9 最小方案：tier 在创建时由 M04 整合通道一次性确定（lifecycle_candidate 直接采纳或人工改），之后**不支持自动升降级**（写明 invariant），升降级算法留 P4+。补在哪：01 §7.7 lifecycle 矩阵+DD-AUTHORITY-V2 §2.5。

**9. 过期复核**
- EXISTING DESIGN：02 §7.11 5.1（7 因素综合+禁单一 TTL）；`lifecycle.expiry_policy.{type}`/`archive_after_days` TBD（02 §7.11 13/14）。
- DESIGN COMPLETENESS：PARTIAL——原则齐、算法无：7 因素无权重/无规则表/无扫描周期（Dream FUTURE，任务八类默认关闭 02 §7.12 14）。V0.9 若无 Dream，过期复核无执行体。
- RECOMMENDED ADDITION（NEW PROPOSAL，P1）：V0.9 只做"读取时惰性过期判定"：检索回源九步第 5 步已检查 lifecycle_state，可在此追加 valid_to/expires_at 检查（纯读取时判定，无扫描器），把主动过期留给 Dream。

**10. FTS（全文召回）**
- EXISTING DESIGN：02 §7.3 5.3（字段权重+bm25+转义+Top-50）；05 §7.4.1（重建九步）；06 T-M02-01/V-02。
- DESIGN COMPLETENESS：MOSTLY_COMPLETE——边界（防注入、禁混 BM25 与 cosine）齐；**tokenizer_id TBD 是硬缺口**：中文分词失败=V-02 直接 FAIL，且 OI-02-04 说"随宿主模型"，而宿主 tokenizer 属 08 号 OI-08-10 亦未定——两处 TBD 串联，需一个决策。
- RECOMMENDED ADDITION（NEW PROPOSAL，P0）：在 15 号登记 `recall.fts_tokenizer`（候选：jieba-wasm / nodejieba / SQLite ICU；含版本与可替换契约"分词方案 MUST 可替换且版本入日志"02 §7.3 5.3 已给契约），按 06 §7.16.2"其它真实环境验证项"先实测两项候选。

**11. Vector（向量召回）**
- EXISTING DESIGN：02 §7.3 5.4/5.5（Adapter 四接口+FLAT 八项验证）；05 §7.4.2；embedding.schema.json（provider/active_space/dimension 校验/async_write）。
- DESIGN COMPLETENESS：MOSTLY_COMPLETE——未证 FLAT 即 BLOCKED 的门禁设计优于一般设计文档。缺口同 ALG-F-STO-05：换模型过渡期"旧空间仍可查、新空间渐次写入"的路由规则未写（只有"退役前保留"）。
- RECOMMENDED ADDITION（NEW PROPOSAL，P2）：在 embedding.schema.json 的 spaces[] 增加过渡态语义（active/draining），M02 查询仅在 active 空间检索（切换瞬时完成，不做双空间并查）——避免复杂化，与 INV-03 一致。

**12. RRF**
- EXISTING DESIGN：02 §7.4（十七要素+示例表）；retrieval.schema.json fusion 节（rank_start=1、missing_channel_contribution=0 const）。
- DESIGN COMPLETENESS：COMPLETE——可直接实现，含手算验收（T02-08 ±1e-6）。
- 无 NEW PROPOSAL 需要。structured 不计入 RRF（OI-022）已在 schema description 固化。

**13. Reranking（六特征规则排序）**
- EXISTING DESIGN：02 §7.5（全局线性公式）vs 11 §7.17 阶段 3（"禁止单一固定权重全局排序…组内权重 MUST 独立配置…MUST NOT 用一个全局分对所有锚点统一排序"）；06 §7.10 消融表按 02 全局六特征固化。
- DESIGN COMPLETENESS：**CONFLICTED**——冲突双方：①02-clean.md §7.5（ALG-04）"Score(d)=α·R+β·E+γ·N+δ·T+ε·S−ζ·P…α=0.55…全部 EXP-DEFAULT"（一套权重打所有候选）；②11-MemoryAnchor与语义分类.md §7.17 阶段 3"3.2 组内打分：在每个锚点组内用**该组特征集**打分…组内权重 MUST 独立配置；3.4 跨组装配 MUST 按 anchor_plan 顺序+配额，而 MUST NOT 用一个全局分"。11 号还禁止 tier 入权重（§7.17 禁令 2），02 号公式无 tier 项（不直接违反），但 E 特征（02 定义"实体/书名号/引号精确命中"）与 06 号消融表 E 定义（"词面匹配含规范化前后"）存在口径差。三份文档必须收敛到一份排序规格。
- RECOMMENDED ADDITION（NEW PROPOSAL，P0）：裁决方向建议采用 11 号分组装配为**外壳**（锚点组顺序+配额+组内排序），ALG-04 六特征公式**降级为组内打分函数之一**（R/E/T 保留，N 在组内恒 1 可省，S/P 保留）；06 号消融 ABL-C/D/E/F 随之改写为"组内特征消融"。需用户裁决后方可改 02/06。

**14. Temporal Decay**
- EXISTING DESIGN：02 §7.5 3.2（λ 按类型表+半衰期）；02 §7.11 5.2（衰减只影响排序不删除+last_retrieved 禁入 v2 正向权重）；emotion 侧 D 公式（13 §7.9）。
- DESIGN COMPLETENESS：PARTIAL——**排序内衰减**完整（公式+缺失处理+历史查询改时间窗命中）；**记忆有效性衰减**（触发降级/过期的独立机制）无算法（同 ALG-F-QLT-13）。λ 表 7 组值未标定（OI-02-08），但公式形态已裁决 R10，属参数待标定非不可实现。
- RECOMMENDED ADDITION（NEW PROPOSAL，P2）：与第 9 条合并处理——V0.9 只做读取时惰性判定；λ 表标定列入 ALG-12 首轮 Benchmark。

**15. 跨区块检索**
- EXISTING DESIGN：无（B 路 B-C001 一致）。
- DESIGN COMPLETENESS：**MISSING**——02 ALG-02 三通道召回均以单 memory_id 为单元；11 五阶段按锚点**分组取候选**（阶段 2"按 anchor_plan 逐锚点取候选"），这是最接近"多区块联合"的机制，但组间只有配额装配、没有"同一实体的画像条+其事件+其关系"的**联合召回**（如 `ABOUT`/`RELATIONSHIP` 关系不作为召回通道，`memory_relations` 两端只有 memory_id 也无法按实体聚合——B 路 B-C002 一致）。长文本分割块之间的关联召回同样无算法（检索对象是最小 MemoryUnit，非区块）。
- RECOMMENDED ADDITION（NEW PROPOSAL，P1）：V0.9 不做自动联合召回；以"锚点分组装配"近似覆盖（人物查询天然聚合画像组）。如需事件→人物联动，给 `getRelated(memory_id)` 手动二跳（14 §7.4 场景 3 已有该接口链），并把它写进 14 号场景 2/3 的 MUST 链路（当前是"（不足时）"软措辞，改为 MUST 二跳）。

**16. Evidence 回源**
- EXISTING DESIGN：13 §7.4.1 六步+§7.4.4 PERM-1…4+§7.4.2 哈希校验；`recall.evidence_expansion_limit=3`（15 §7.7，已落 retrieval.schema.json context.evidence_expansion_limit）。
- DESIGN COMPLETENESS：COMPLETE——先判权后反查、revoked 不返回正文、错误不回显，全齐备。可直接实现。
- 无 NEW PROPOSAL。

**17. Token Budget**
- EXISTING DESIGN：02 §7.6（十七要素）；14 §7.12.1（四类单位+四分项预算）；15 §7.15（token/char 双预算先到者裁决+禁换算）；retrieval.schema.json（token_budget=1200、budget_chars=2400、truncate_original_content=false const）。
- DESIGN COMPLETENESS：COMPLETE——唯一依赖是 Tokenizer 选型（OI-02-04，宿主模型定）；`allow_estimate=false` 已给降级开关。可直接实现。
- RECOMMENDED ADDITION（NEW PROPOSAL，P2）：15 号 §7.17 待登记表已有 storage/tokenizer 项，补 `retrieval.tokenizer.id/version` 的登记时点为"宿主模型确认后立即"，避免与 08 号宿主 tokenizer 验证（OI-08-10）脱节。

**18. Context Assembly**
- EXISTING DESIGN：02 §7.7（GreedySelect 八步+4 分区+禁令 8 条+幂等）；retrieval.schema.json context 节（selection=greedy const、redundancy_lambda=0.5、deduplicate=true、permission_fail_policy=deny const）。
- DESIGN COMPLETENESS：COMPLETE——与 11 号 §7.17 阶段 5"分区记录 Token（画像/事件/证据/情绪分别计入）"存在**两个分区口径**（02 的 must/relevant/background/uncertain vs 11 的 profile/event/evidence/emotion），14 §7.12.1 又要求 profile_tokens/event_tokens/evidence_tokens/emotion_tokens 四分项——三处需统一为一个双层结构（分区×内容类型）。
- RECOMMENDED ADDITION（NEW PROPOSAL，P1）：定义 TokenUsage 双维：zone（02 四值）×kind（11/14 四值），加总核对两条轴各自成立；改 02 §8.3 与 14 §7.12.1 同步。

**19. 情绪关联与衰减**
- EXISTING DESIGN：09 §7.15（提取契约+关闭态）；13 §7.7/7.8/7.9（分离 8 规则+15 标签+ΔE 公式+8 门禁）；emotion.schema.json（3 开关 const false+gates const+label_dict 15 标签）。
- DESIGN COMPLETENESS：MOSTLY_COMPLETE——默认关闭三开关以 const 锁定（超出普通文档纪律）；ΔE 公式有演算示例。缺口：①**R（话题关联度）无计算算法**（13 §7.9.1 表"由关联度计算得出（PROPOSED，参数化待标定）"）；②`emotion.auto_infer=false` 关闭态下，USER_EXPLICIT 情绪由**谁**写入（宿主侧识别"我当时很紧张"→调 propose？还是 EmotionExtractor 仍运行但只接受显式？）——09 §7.15"只接受 USER_EXPLICIT"隐含 EmotionExtractor 仍运行，但触发词/判定算法无定义。
- RECOMMENDED ADDITION（NEW PROPOSAL，P1）：①R 算法留待启用前标定（写明"启用 CR-03 裁决时必须先提交 R 计算规格"）；②在 09 §7.15 补关闭态正向规格：EmotionExtractor 仅当输入 envelope 含 `explicit_emotion_marker`（宿主侧/规则识别显式情绪句）时运行，输出仅 USER_EXPLICIT；无 marker 即 skipped_reason=emotion_disabled。

**20. Persona Prompt 组装**
- EXISTING DESIGN：08 号 §7.4（宿主拼装最终 Prompt：人格+系统规则+记忆注入块；宿主规则优先 EX-08-05）；02 §7.7（ContextPackage）；E2E-10 宿主注入边界 MOCKED（06 §7.14）。
- DESIGN COMPLETENESS：MOSTLY_COMPLETE（记忆注入侧）——Memory Core 侧组装契约完整；**Persona 本体（人设存储/版本/动态上下文）0 算法**（CR-08B 待单独呈报，00 OI-00-07），属明确登记的 FUTURE 而非隐性遗漏。
- RECOMMENDED ADDITION（NEW PROPOSAL，P2）：维持 FUTURE 不补；仅在 08 号 VERIFY-08-02（注入点）落地时确认 Context Adapter 与 Persona 的接口边界（Persona 是 Context Adapter 的消费方，非 Memory Core 职责）。

**21. Outbox 索引更新**
- EXISTING DESIGN：02 §7.2（同事务写 Outbox）；05 §7.5（积压续跑/dead_letter/最老事件年龄）；06 KT-M01-01/02/03（三关键测试）；indexing.* 3 参数已注册（poll_ms=1000、retry_max=8、consistency_check=true APPROVED）。
- DESIGN COMPLETENESS：MOSTLY_COMPLETE——幂等（event_id）+重试上限+版本丢弃齐备。缺口：①FTS5 文本投影"无摘要回退策略 TBD"（05 §7.4.1 步 3，D-AP-04 明文遗留）——回退策略不定，Worker 实现无法写；②事件 payload schema 的 `payload_version` 兼容矩阵（旧消费者读新事件）未定义。
- RECOMMENDED ADDITION（NEW PROPOSAL，P0）：①补"索引文本投影优先级"：summary 存在→summary+cue；否则→content 截断至 `indexing.projection_max_chars`（新参数，EXP-DEFAULT，禁语义截断，截断边界按句）；②payload_version 兼容规则：Worker 忽略未知字段（forward-compatible），拒绝未知 event_type（fail-closed）。

## §5 完整度统计与十个最关键缺口

### 5.1 统计（按 §2 全量 76 条）

| 完整度 | 条数 | 占比 |
|---|---:|---:|
| COMPLETE | 13 | 17% |
| MOSTLY_COMPLETE | 43 | 57% |
| PARTIAL | 12 | 16% |
| CONCEPT_ONLY | 4 | 5% |
| CONFLICTED | 1 | 1% |
| MISSING | 3 | 4% |

（Persona 族 10 条中 5 条 MISSING、1 条 CONCEPT_ONLY 均因模块整体 FUTURE 且 CR-08B 已登记；视为"显式未设计"而非隐性遗漏。）

### 5.2 十个最关键缺口（按对 V0.9 长期记忆聊天 AI 的影响排序）

| # | 缺口 | 涉及条目 | 影响 |
|---|---|---|---|
| 1 | importance 赋值算法缺失（权威字段无写入者） | ALG-F-QLT-01 | 过滤/并列打破失效；字段永远空值 |
| 2 | confidence 合成公式缺失（仅 LLM 自报） | ALG-F-QLT-02 | 三阈值分流（AUTO_ACCEPT/NEEDS_REVIEW/PENDING）输入不可信 |
| 3 | 02 vs 11 排序形态冲突（全局 vs 分组） | ALG-F-RET-10 | 实现者无所适从；消融基线失效 |
| 4 | FTS 中文分词器 tokenizer_id TBD（串联宿主 tokenizer TBD） | ALG-F-RET-05 / S-21 | V-02 中文检索验收无法执行 |
| 5 | Outbox 索引文本投影无摘要回退策略 TBD | S-21 / ALG-F-STO-03 | Worker 无法实现；D-AP-04 明文遗留 |
| 6 | Tier 晋升/降级算法缺失（与 A 路 A-C003 一致） | ALG-F-QLT-11/12 | memory_tier 只有标签轴（A-C001） |
| 7 | Intent Resolver 判定算法缺失（9 类意图如何判出） | ALG-F-RET-02 | 锚点路由输入不可得，五阶段检索退化 |
| 8 | 跨区块/联合召回缺失（B-C001 本路证实） | ALG-F-RET-20 / ORG-07 | 人物理解查询只能靠锚点分组近似 |
| 9 | 冲突枚举两套（5 类 vs 8 类）无映射 | ALG-F-QLT-06 | M03 trace 与 M04 决策字段不可对齐 |
| 10 | 关键参数未入 15 号注册表（retrieval.* 权重类 11 项、consolidation.* 4 项、lifecycle.*、context.* 部分） | 15 §7.17 | 生产配置使用即违反"未登记不得使用"（15 §7.17 MUST） |

## §6 最小补齐清单（按 V0.9 阻塞排序）

| 序 | 缺什么 | 补在哪份文档 | 是否改 schema | 能否实验确定 |
|---|---|---|---|---|
| 1 | 排序形态裁决（全局 ALG-04 vs 分组装配）→ 建议分组外壳+组内六特征 | 02 §7.5 + 11 §7.17 双向修订；06 §7.10 消融表 | 是（retrieval.schema.json reranking.weights 改为按锚点组嵌套或保留全局+组内覆盖） | 权重值可实验（消融 A–F）；形态本身须用户裁决 |
| 2 | importance 算法（V0.9 最小：常量默认+M04 人工更新两档） | 01 §8.3 字段表注 + 新章节于 02 或 12 号 | 否（字段已存在） | 部分可（分档阈值可测）；公式形态须裁决 |
| 3 | confidence 合成公式（信号清单+权重 EXP-DEFAULT） | 02 ALG-07 §5 步骤 9 展开为独立小节 | 否 | 是（标注集扫描 Precision/Recall，15 §7.5 A15 已给方法） |
| 4 | FTS 分词器选型 + `recall.fts_tokenizer` 登记 | 15 §7.7/§7.17 + 02 §7.3 5.3 引用 | 是（retrieval.schema.json fts 节加 tokenizer 字段） | 是（两项候选实测，06 §7.16.2） |
| 5 | 索引文本投影无摘要回退策略 + `indexing.projection_max_chars` | 05 §7.4.1 步 3 + 15 §7.11 | 是（storage/worker.schema.json） | 截断上限可实验；策略本身须裁决 |
| 6 | Intent Resolver 判定机制（规则词表+优先级+保守回退，禁 LLM 判意图或明示允许） | 11 §7.19 后新增小节 | 否 | 是（intent 标注集 κ 测试，I-8 口径） |
| 7 | Tier 一次性确定+禁自动升降级 invariant | 01 §7.7 + DD-AUTHORITY-V2 §2.5 | 否（schema 已有 tier 字段） | 否（不变量裁决项） |
| 8 | 冲突枚举映射表（09 五类 ⊂ 02 八类） | 02 §7.9 5.2 表后附映射 | 是（conflict 枚举入 contracts/schemas 单一来源） | 否 |
| 9 | 未登记参数补登记（11 项权重+consolidation 4+lifecycle 2+context 2） | 15 §7.17→§7.4–7.12 正式行（先 TBD 值） | 是（对应 config schema） | 是（逐项按 §7.14 八步） |
| 10 | EmotionExtractor 关闭态正向规格（explicit_emotion_marker 触发） | 09 §7.15 | 否 | 否（边界裁决项，量小） |
| 11 | 跨区块最小方案：getRelated MUST 二跳写入场景链 | 14 §7.4 场景 2/3 | 否 | 否 |
| 12 | TokenUsage 双维口径（zone×kind） | 02 §8.3 + 14 §7.12.1 + 15 §7.15 | 是（retrieval.schema.json context 节） | 否 |

## §7 端到端验收链（一次会话陈述 → L3 记忆 → 可回溯原句 → 保留规则过期）

以一句真实链路验证各算法衔接。场景：用户说"记住：我不吃香菜"（会话 S1，owner_private）。

| 步 | 动作 | 算法 ID | 文档依据 | 通过判据 | 链路状态 |
|---|---|---|---|---|---|
| 1 | Chat LLM 提交 `memory.propose`（11 必填字段） | ALG-F-ING-05 前置 | 14 §7.5.1 | candidate.status=PENDING；不直接写库 | ✅ 设计闭环 |
| 2 | InputNormalizer 归一化（source_type=USER_INSTRUCTION 确定不可变） | ALG-F-ING-01 | 09 §7.5 | envelope 含原句+时间戳；幂等键生成 | ✅ |
| 3 | RulePreprocessor 命中明确记忆指令触发词 | ALG-F-ING-01/11 | 09 §7.6 | is_explicit_instruction=true；高优先队列 | ✅ |
| 4 | MemoryValueEvaluator：SAVE_EXPLICIT 快速通过 | ALG-F-ING-13 | 09 §7.7 | bypass_reason=explicit_instruction 记录 | ✅ |
| 5 | LLMExtractor（经 M04 Gateway）抽取 | ALG-F-ING-11 | 09 §7.8 | units[]+schema 校验；禁输出 validation_state | ⚠️ 依赖 ingest.model 选型（TBD） |
| 6 | SemanticSplitter：一碎片一核心语义（"不吃香菜"限定保留） | 算法 1（§4-1） | 09 §7.9 | 否定词"不"未丢失（S5）；原句区间可定位 | ✅ |
| 7 | MemoryClassifier：semantic_type=PREFERENCE+semantic_tags=[SELF_EXPLICIT,…] | 算法 2（§4-2） | 09 §7.10 | 六字段齐；互斥规则满足 | ⚠️ 依赖模型选型 |
| 8 | TemporalResolver：无时间表达→字段 null+temporal_unresolved 不适用 | ALG-F-ING-14 | 09 §7.11 | 未知时间不猜测 | ✅ |
| 9 | EntityResolver：subject=OWNER owner:<id> | ALG-F-ING-08 | 09 §7.12 | resolution_basis 记录 | ✅ |
| 10 | AnchorAssigner：anchor=[SELF_EXPLICIT,PREFERENCE] | 算法 2（§4-2） | 09 §7.13 | 画像与事件未混；≥1 锚点 | ✅ |
| 11 | EvidenceBinder：绑定原句 Evidence+span | 算法 3（§4-3） | 09 §7.14；13 §7.1 | content_hash=sha256(original_text)；反向可查 | ✅（偏移校验为 NEW PROPOSAL） |
| 12 | EmotionExtractor：无显式情绪→跳过 | ALG-F-EMO-01 | 09 §7.15 | skipped_reason=emotion_disabled 记录 | ✅ |
| 13 | DuplicateDetector：4 类重复检测（哈希+语义） | 算法 5（§4-5） | 09 §7.16 | 无既有重复→通过 | ✅ |
| 14 | ConflictDetector：5 类冲突检测（不裁决） | 算法 6（§4-6） | 09 §7.17 | conflict_flags=[] 或标记 | ✅ |
| 15 | CandidateValidator：9 项校验 | ALG-F-ING-12 | 09 §7.18 | ValidationReport 全 pass；confidence≥0.55 | ⚠️ confidence 合成缺（缺口 #2） |
| 16 | PermissionGate：M05 canWrite（当前 contract_deny 模式） | ALG-09 | 09 §7.19；02 §7.10 | decision_id 携带；ALLOW | ⚠️ P0–P3 M05=contract_deny，链路只能验证"拒绝路径" |
| 17 | M04 裁决（P4）→ tier 确定 lifecycle_candidate→L3 | 算法 8（§4-8）+缺口 #7 | 09 §7.20；11 §7.12 | validation_state≠confirmed（CR-01）；tier=L3 | ⚠️ M04 未授权，V0.9 需人工通道替代 |
| 18 | M01 事务五步：memory+version+outbox 同事务 | ALG-F-STO-01/02 | 02 §7.2 | 三表一致；version=1 | ✅ 可实现 |
| 19 | Outbox→Worker→FTS5/LanceDB 同步 | 算法 21（§4-21） | 05 §7.5；06 KT-M01-02 | 索引文档 version=1；分词命中 | ⚠️ 依赖缺口 #4/#5（分词器+投影回退） |
| 20 | 用户问"我能吃香菜吗"→search | ALG-02 规范化七步（S3 保留否定） | 02 §7.3 5 | normalized_query 保留"不"（T02-05） | ✅ |
| 21 | RRF 融合→六特征排序→权威回源九步→M05 终检 | ALG-03/04+算法 13 | 02 §7.4/7.5/7.3 5.6 | 候选带 channel_evidence；正文=SQLite 版 | ⚠️ 排序形态冲突（缺口 #3） |
| 22 | Token Budget 计量→GreedySelect→分区注入 | ALG-05/06 | 02 §7.6/7.7 | token_usage 五类+双预算；PREFERENCE 内容带 source_type 标注 | ✅（Tokenizer 选型后） |
| 23 | 可回溯原句：getEvidence(memory_id, related_evidence) | 算法 16（§4-16） | 13 §7.4.1 | 返回"我不吃香菜"原句+source_locator；先判权（步 2） | ✅ 可实现 |
| 24 | 保留规则过期：读取时惰性检查 valid_to/expires_at（NEW PROPOSAL 缺口 #9 的 V0.9 方案）或 Dream 扫描（FUTURE） | 算法 9（§4-9）+ALG-10 | 02 §7.11 5.1 | PREFERENCE λ=0.001 半衰期≈693 天（排序层）；lifecycle 过期需触发体 | ⚠️ 主动过期无执行体（缺口 #6） |

链路结论：**24 步中 17 步设计闭环可直接实现，7 步（5/7/15/16/17/19/24）存在依赖缺口**——其中 3 步属"参数/策略待裁决"（缺口 #3/#4/#5），3 步属"权威字段/算法缺失"（缺口 #1/#2/#7），1 步属"执行体 FUTURE"（M04/过期扫描）。链路不断裂的 V0.9 最小路径：M04 裁决用人工通道替代+confidence 给保守合成公式+分词器/投影回退按 §6 第 4/5 项补齐。

## §8 方法附录

1. **阅读顺序与范围**：①02-clean.md（EVIDENCE\B 清洗副本，grep `^#{1,3}` 定位 12 个 ALG+17 要素结构后按段读 131–1553 行）；②09 号 §7.4–7.24+§8.7（16 阶段逐段）；③11 号 §7.12/§7.14/§7.17+§8（tier 矩阵、人物优先、五阶段排序、锚点路由）；④14 号 §7.4/§7.12（读取场景、Token 单位）；⑤13 号 §7.1–7.9（Evidence 绑定+情绪算法，grep 定位）；⑥15 号 §7.1/§7.4–7.17（参数注册全表）；⑦05 号 §7.1–7.11（故障矩阵/重建/Outbox）；⑧06 号 §7.1–7.17（测试层级/污染/Benchmark/消融/FLAT 八项）；⑨contracts/config/*.schema.json 六份（memory/retrieval/anchor/emotion/embedding/test——PS 5.1 解析中文 description 因编码报错，改以文本阅读确认字段与 const）；⑩11-ALGORITHM-REVIEW-SCOPE.md（前轮范围：可替代性九问+D-AP 裁决索引，本轮不重复其结论，仅取"不得把'有名字'当'算法完整'"纪律）。
2. **审计来源**：E:\[PROJECT]\AUDIT-FORENSIC-2026 下 09-CONFLICT-REGISTER.md、10-GAP-AND-RISK-REGISTER.md 未逐条读（按硬性要求第 4 条控制总量）；与已知结论 A-C001…A-C004、B-C001…B-C009 的交叉验证通过 01/02 号两份路报告（同目录）完成，本路独立证实 B-C001（跨区块无联合召回）与 A-C003（tier 转换 0 条）。
3. **完整度判据**：COMPLETE=17 要素齐且无跨文档矛盾；MOSTLY_COMPLETE=核心要素齐但有 1–2 个待标定参数或执行者待选型；PARTIAL=有原则/枚举但执行算法缺；CONCEPT_ONLY=仅边界条款或一行提及；CONFLICTED=两文档直接矛盾且双方为规范正文；MISSING=任务书要求但全文档集检索无定义。
4. **未实测≠不可实现**：全部 EXP-DEFAULT 参数（15 号 32 项）判 MOSTLY_COMPLETE 而非 CONCEPT_ONLY；仅当"公式形态+输入输出+边界"任一缺位才降档。
5. **引用纪律**：02 号一律引章节号（清洗副本行号有偏移）；其余文档引文件+章节；冲突条目列双方原文要点（§4-13）。
6. **限制**：12-人物画像整合.md、10-LLM调用体系.md、01/03/04/07/08 号未逐段读（按标题+grep 关键词取证）；FORENSIC 49 条 FC 未通读；Persona 族结论基于 00/06/08/DESIGN-DECISION-REGISTER 的 FUTURE 边界条款，若 CR-08B 呈报材料中另有 Persona 算法正文，本路未覆盖。
- §1 Headline
- §2 全量算法清单（ALG-F01…，紧凑表：编号/名称/所属模块/完整度/关键缺口）
- §3 总表（名称/输入/输出/公式/步骤/参数/边界/异常/测试/完整度/可直接实现？）
- §4 21 个重点算法逐条结论
- §5 完整度统计与十个最关键缺口
- §6 最小补齐清单（按 V0.9 阻塞排序）
- §7 端到端验收链
- §8 方法附录

（章节内容将按完成顺序逐章追加。）
