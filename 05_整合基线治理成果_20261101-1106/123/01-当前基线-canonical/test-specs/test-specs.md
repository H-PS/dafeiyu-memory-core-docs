# Memory Core R4.1 — 测试规格与验收标准（Test Specs）

> 文件：`test-specs/test-specs.md` ｜ 版本 1.0 ｜ 状态：APPROVED 结构（依据 sf 后文 §7 + 06 文档；判定法与阈值标 EXP-DEFAULT/PROPOSED）
> 总则：结果五态 PASS/FAIL/BLOCKED/NOT RUN/MOCKED；禁止虚假验收七条；Mock 仅限接口测试并标 MOCKED；**测试数据一律合成，禁止使用真实私人记忆**。

## TS-1 M01 存储测试 [P1]

| ID | 场景 | 步骤 | 预期 | 判定 |
|---|---|---|---|---|
| TC-M01-001 | 正常创建 | createMemory 合法对象 | 返回对象，version=1，Outbox 事件 pending | PASS 条件：DB 行+事件行同事务存在 |
| TC-M01-002 | Schema 非法 | content 空 / scope 非枚举 | VALIDATION_ERROR，无写入 | 事务零副作用 |
| TC-M01-003 | 乐观并发 | 两个 updateMemory(expectedVersion=1) 并发 | 一个成功，一个 VERSION_CONFLICT | 禁止 LWW |
| TC-M01-004 | **事务原子性（关键）** | 注入 Outbox 写入失败（fault injection） | memory+version+outbox 全部回滚 | **MUST 整体回滚** |
| TC-M01-005 | 版本历史 | 连续 3 次更新 | listMemoryVersions 返回 v1–v3，旧版本保留 | 禁止原地覆盖 |
| TC-M01-006 | **旧事件晚到（关键）** | 先后产生 v2、v3 事件，故意先处理 v3 再处理 v2 | v2 被 SkipOrSuperseded，索引保持 v3 | 索引哈希=v3 |
| TC-M01-007 | **删除幂等（关键）** | 删除后重复投递 DELETE 事件两次 | 索引无复活，两次均幂等成功/跳过 | 数据不得恢复 |
| TC-M01-008 | 删除传播 | 删除→等 Worker | FTS5+LanceDB 查无此 id；verifyDeleted 通过 | 残留=FAIL |
| TC-M01-009 | 重复请求 | 相同 idempotency_key 两次 create | 第二次 DUPLICATE_REQUEST（返回原结果） | 不产生重复行 |
| TC-M01-010 | 备份恢复 | backup→损坏→restore→重建索引 | 恢复后 verifyIntegrity 通过，索引可重建 | 版本一致 |
| TC-M01-011 | Integrity | 手工篡改一行 content | verifyIntegrity 报告差异 | 可检测 |

## TS-2 M02 检索测试 [P2]

| ID | 场景 | 预期 |
|---|---|---|
| TC-M02-001 | FTS5 中文召回 | 合成中文记忆命中 Top-K；unicode61 局限记录在案（VERIFY 替代分词器） |
| TC-M02-002 | FLAT 精确性 | 对比暴力余弦基准：LanceDB 返回顺序与基准一致（全部 QUERY）——**不一致 ⇒ BLOCKED** |
| TC-M02-003 | 空间隔离 | 两个 embedding_space 的向量混查被拒/隔离返回 | 
| TC-M02-004 | RRF 可复现 | 同输入两次检索：候选顺序与 rrf_score 完全一致 |
| TC-M02-005 | 缺失通道 | 仅一路命中：RRF 只计该路；缺失贡献 0 |
| TC-M02-006 | 特征缺失无 NaN | 全部特征缺失时 finalScore 有限值，不抛异常 |
| TC-M02-007 | 惩罚分离 | duplicate_penalty 不进入正向归一化；正分与罚分独立计算 |
| TC-M02-008 | 稳定排序 | 同分候选按 memory_id ASC；跨运行一致 |
| TC-M02-009 | 回源覆盖 | 索引正文≠权威正文时，上下文使用 SQLite 版本 + STALE_VERSION 处理 |
| TC-M02-010 | Token 预算 | 1200 预算下 sum(token_cost)≤1200；estimated/exhaust 区分记录 |
| TC-M02-011 | 贪心选择 | 高相关优先入选；REDUNDANT 排除不重复注入 |
| TC-M02-012 | 空结果合法 | 全被排除时 status=empty，不注入凑数内容 |
| TC-M02-013 | 通道故障 | 注入 LanceDB 故障：未批准降级 ⇒ status=failed；批准降级 ⇒ partial+degraded=true |

## TS-3 权限测试（12 场景）[P2 契约 / P6 完整]

| ID | 场景 | 输入 | 预期 |
|---|---|---|---|
| TC-PM-001 | owner_private 跨主体 | 用户 B 请求用户 A 私有记忆 | DENY（decision_id 留痕） |
| TC-PM-002 | group_shared 越群 | 非授权群请求 | DENY |
| TC-PM-003 | session_only 跨会话 | 新会话请求 | DENY |
| TC-PM-004 | project_scoped 跨项目 | 其他项目请求 | DENY |
| TC-PM-005 | system_internal 暴露 | 普通用户请求 | DENY |
| TC-PM-006 | 群管理员读成员私人 | group_administrator 角色请求 | DENY（角色≠所有权） |
| TC-PM-007 | Bot Owner 读私人正文 | bot_owner 维护操作 | DENY（维护≠读取正文） |
| TC-PM-008 | 权限服务超时 | 注入 timeout | INDETERMINATE ⇒ 排除（fail-closed） |
| TC-PM-009 | 权限服务错误 | 注入异常 | 同上 |
| TC-PM-010 | 缺少身份 | actor.id 空 | DENY |
| TC-PM-011 | 删除后索引残留 | 删除后立即检索 | 召回结果不出现该 id（权威排除+索引清理双保险） |
| TC-PM-012 | 派生数据越权 | 用受限记忆构造 Observation 请求 | 派生对象 scope=来源最严格交集；越权请求 DENY |

## TS-4 污染测试（6 条）[P2/P3]

1. 错误事实不自动升级：候选 unverified 永不进入 active 召回。
2. 幻觉不成事实：LLM 生成内容无 evidence ⇒ quality_flags 含 no_source ⇒ 不入 pending_review。
3. 重复错误不提高置信度：同一错误文本 100 次注入，confidence 不上调。
4. 冲突不静默覆盖：矛盾双记录均保留，disputed，检索返回冲突标记。
5. 无来源推断不伪装事实：content_kind=inference 保持；序列化标记"推断"。
6. 越权记忆不进最终 Prompt：权限 DENY 的记忆在 context.selected 中不存在（抽查 selected 全 ALLOW）。

## TS-5 Benchmark 规格 [P3]

- 数据集：`benchmark/`（queries/memories/relevance/permission_cases/conflict_cases/expected_results 六文件），**dataset_version 固定**；≥15 类查询类型各≥2 条（精确/语义/同义/专有名词/时间/历史事件/长期事实/计划状态/多条相似/冲突/删除/权限隔离/超预算/跨空间/同步延迟）。
- 指标：Recall@K（K=5,10,20）、HitRate@K、Precision@K、MRR、NDCG@K（分级 0–3）、Context Precision、Context Coverage、Token Efficiency、P50/P95/P99。
- 分通道消融 A–F：仅FTS5 / 仅LanceDB / 双路 / +RRF / +规则排序 / 完整链。**必须分通道报告**。
- 参数版本登记：algorithm_version + config_version + dataset_version + 结果（可追溯）。
- 性能：先实测建立基线；**未实测前禁止宣称任何延迟数值**。
- 验收阈值：P3 功能性=TS-1~TS-4 全 PASS；质量阈值待基线后由用户裁决（OPEN）。

## TS-6 故障注入与恢复 [P3]

| ID | 注入 | 预期 |
|---|---|---|
| TC-FR-001 | SQLite 写失败 | 停止权威读写；错误 STORAGE_ERROR；恢复后 Outbox 续跑 |
| TC-FR-002 | LanceDB 不可用 | 不静默换引擎；按批准策略（未批准=failed） |
| TC-FR-003 | FTS5 不可用 | 记录不完整检索状态 |
| TC-FR-004 | Embedding 超时 | 事件 retry_wait；无伪造向量 |
| TC-FR-005 | 进程崩溃（Worker 处理中） | 租约超时→事件回 pending→重处理→幂等 |
| TC-FR-006 | 重复事件×N | 终态一致 |
| TC-FR-007 | 恢复中断 | 再次恢复可续；不得跳过一致性校验 |

## TS-7 E2E（12 场景）[P3]

明确保存→自动候选（MOCKED 提取）→重复→补充→冲突→过期意图→私人检索→群聊隔离→权限拒绝→删除后不可召回→索引重建→预算控制。每场景给：前置数据（合成）/操作序列/预期上下文与 trace/判定。链路五分（Retrieved/Authorized/Injected/Model Used/Answer Correct）中，前三分可测；后两分 FUTURE（需额外评估机制，不得用注入日志冒充）。

## TS-8 验收标准汇总

- **P0**：05 环境验证清单 10 项全 PASS + SQLite/LanceDB 驱动能力验证记录（VERIFY→裁决）。
- **P1**：TS-1 全 PASS（三大关键测试 MUST PASS）。
- **P2**：TS-2/TS-3/TS-4 全 PASS；FLAT 精确性（TC-M02-002）MUST PASS（否则 BLOCKED）。
- **P3**：TS-5 跑通并出具基线报告 + TS-6/TS-7 全 PASS + 一致性对账 0 差异 + 交付报告与偏差清单。
- **通用**：MOCKED 结果不得计入真实验收；未运行=NOT RUN 不得写 PASS。
