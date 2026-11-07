# ADJUDICATION-COMPLIANCE-REPORT — 裁决遵循报告

> 2026-11-04 ｜ 审查者：总控设计 Agent ｜ 依据：`08-adjudication-ledger.md`（391 行全文已读）+ DD-AUTHORITY §3/§5
> 方法：台账 35+41 项裁决逐类映射到设计文档落点；对抗式查找"建议→批准"的越级表述。

## 1. 总体结论

**全部 17 条用户硬约束（任务指令）已落实；台账已决定项 0 例违反；PROPOSED/TBD/VERIFY/BLOCKED/DEFERRED 状态纪律执行（登记表 78 项中 APPROVED 38 项全部带台账/规范来源引用）。** 两处需用户裁决的真实冲突（OI-015 A 层迁移语义、OI-005 revoke 语义）已按"停止并报告"处理，未擅自裁决。

## 2. 台账裁决 → 设计落点对照（关键 20 项）

| 台账 | 裁决要点 | 设计落点 | 遵循证据 |
|---|---|---|---|
| D-AP-01 | 五模块边界；M03/M04 不可合并 | 00 §模块；02 各算法边界 | 全文职责表 |
| D-AP-02 | M04 §6.2 唯一 LLM Gateway；Embedding 归 M01 | 02 M03/M04（FUTURE）；01 EmbeddingSpace 归存储层 | AD-003 |
| D-AP-03 | P0 平台层概念接受不实施；不新增第六模块/进程 | 00 五平面（逻辑分层单进程） | AD-022 |
| D-AP-04 | 摘要+cue 索引主体；无摘要回退 | search-document.schema（content/summary/cue + 回退）；OI-05-01 | AD-006 |
| D-AP-05 | confirmed 仅 M04；reinforce 禁写；last_retrieved 非衰减权重 | SM-1/SM-2；02 §reinforce 红线专节；02 排序特征（无 last_retrieved 正向项） | AD-004/005 |
| D-AP-06 | 生活化测试集为主；收益≠实测 | 06 Benchmark；test-specs 全部 NOT RUN | AD-007 |
| D-AP-07 | C40 不可插拔；entry-only；影子 diff 判权 | 04 §7.2/7.11/7.12；permission-context.schema | AD-008 |
| D-AP-08 | 可判定触发器驱动演进 | 07 FUT-01~08 触发条件表 | — |
| D-AP-09 | 拒绝第三方托管；不声称"不出本机"；云端传输分开评估 | 04 §7.13；08 OI-08（云端传输隐私 TBD） | AD-009 |
| I-3/I-4/I-5 | L0+L2 起步；影子门禁；心理不改记忆候选 | 00 FUTURE 节；08（Psychology 边界） | AD-010/011 |
| I-7/X-6 | psy-model 暂缓；接口形态 DEFERRED | 00/02/07（psy-model 全部 DEFERRED，零实现描述） | AD-012 |
| J2/J4/J6/J7/J8 | 写时标注/15 标签配置/查询侧关闭/只预留/external 字段 | candidate.schema（affect_tags/affect_source 注释）；02 M03 | AD-013 |
| K-1/K-2 | K-1 待 K-2 不生效；导出边界未定 | 01 预留注记（PROPOSED/BLOCKED）；04 §7.10 | AD-015；OI-006 |
| K-6/CR-09 | openLoop 永不由沉默触发；禁催促/联动 | 00 FUTURE（伦理硬门禁条款） | AD-014 |
| U-1/U-2 | cache 表结构/校验不冻结 | 全套文档 **0 处** cache 表结构（grep 核验） | AD-016 |
| X-1/X-2/X-3 | delta-only 影子；Session 丢弃；Recall 不改 PAD | 00/02 FUTURE 条款 | AD-017 |
| 优化-1/3/4/5 | Token 五分类计量；Graphiti 仅登记；拔除 7 依赖；参数 ID 规范 | 02 计量；07 FUT-03；00；config/core.schema | AD-018~021 |
| PC-1/PC-2/PC-6 | 前置核验未完成项 | 全部登记 OPEN-ISSUES（OI-008 等），未声称已通过 | §3 |

## 3. 对抗式抽查（越级表述搜索）

- `grep "已批准|已裁决|用户批准"` 命中处逐一核对：均出现在"引用台账"语境（如"台账 D-AP-05 已决定"），未发现把本设计自身的 PROPOSED 项写成已批准。
- `grep "实测|已验证|通过测试"`：命中处均带 NOT RUN/VERIFY 标注或位于"验证要求"条款（要求未来验证），无虚构结果。
- 实验参数 15 项：02/06/test-specs/retrieval.schema 全部标"实验初值（EXP-DEFAULT）"，与 sf 后文 §15 一致。

## 4. 偏离与需裁决项（透明申报）

| # | 事项 | 性质 | 处理 |
|---|---|---|---|
| 1 | OI-015 A 层迁移"立即可召回" vs D-AP-05（confirmed 仅 M04） | **真实裁决冲突**（组 A 发现） | 已登记 OPEN-ISSUES OI-015，A/B 两方案待用户裁决；设计未擅自择一 |
| 2 | OI-005 revoke 语义二义 | 母本明言"语义必须明确区分"但未定义 | PD-007 给默认（可用性撤销）+ 待批 |
| 3 | OI-004 "仅 FTS5 降级"策略 | sf 后文要求批准才降级 | 未批准=不降级（保守侧），登记 TBD |
| 4 | ID 格式（UUIDv7）、13 表结构、WAL 等工程默认 | 母本/sf 未禁止 | PD-001~014 标 PROPOSED 待批 |

## 5. 声明

本设计未修改台账、未修改 m01/m02 冻结稿、未触碰 D:\qq-bridge、未启动服务、未写入任何生产代码；所有"未来能力"均标 FUTURE/DEFERRED；设计完成 ≠ 实施授权（AD-038）。
