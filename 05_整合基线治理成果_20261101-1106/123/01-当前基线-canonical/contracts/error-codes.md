# Memory Core R4.1 — 统一错误码契约（Error Codes）

> 文件：`contracts/error-codes.md` ｜ 版本 1.0 ｜ 状态：APPROVED（结构依据 sf 后文 §4.4/§4.5；冲突裁决 DD-AUTHORITY R1）
> 规则：错误码不得由各模块自行命名；本表是唯一登记处。错误响应 MUST NOT 泄露私人记忆正文。

## 1. 规范错误码表

每个错误码 MUST 具备七属性：code / category / message 模板 / retryable / severity / handling / exposure_policy。

| code | category | message 模板 | retryable | severity | handling | exposure_policy |
|---|---|---|---|---|---|---|
| VALIDATION_ERROR | validation | 输入未通过 Schema 校验：{field} | false | warning | 调用方修正后重发 | host_visible |
| NOT_FOUND | not_found | 目标对象不存在：{target} | false | info | 确认 ID 或刷新索引 | host_visible |
| VERSION_CONFLICT | conflict | 版本冲突：expected={v1} actual={v2} | true | warning | 重新读取权威数据后再提交 | host_visible |
| PERMISSION_DENIED | permission | 无权执行 {action}（reason_code={rc}） | false | warning | 检查授权；不得重试绕过 | host_visible（不含权限细节推理链） |
| PERMISSION_ERROR | permission | 权限服务故障/超时，默认拒绝 | true | error | fail-closed；恢复后重试 | host_visible |
| STORAGE_ERROR | storage | SQLite 权威存储故障 | true | critical | 停止权威读写；事务回滚；告警 | host_visible |
| INDEX_ERROR | index | 派生索引故障：{fts/lancedb} | true | error | 进入 Outbox 重试；不得污染权威数据 | host_visible |
| EMBEDDING_ERROR | embedding | Embedding 生成失败/超时 | true | error | 保留 Outbox 待重试；禁止伪造向量 | host_visible |
| TIMEOUT | timeout | 操作超时：{component} | true | warning | 幂等重试 | host_visible |
| CONFIGURATION_ERROR | configuration | 配置校验失败：{item} | false | critical | 启动失败；修复配置 | internal_only |
| INTERNAL_ERROR | internal | 未预期错误（trace_id={tid}） | false | critical | 记录并隔离；人工介入 | internal_only |
| DUPLICATE_REQUEST | conflict | 幂等键重复：{idempotency_key} | false | info | 返回原结果 | host_visible |
| BUDGET_EXCEEDED | validation | Token 预算不足：need={n} budget={b} | false | info | 精简上下文或提高预算（需授权） | host_visible |
| TASK_NOT_FOUND | not_found | 异步任务不存在：{task_id} | false | info | 确认 task_id | host_visible |
| RETRIEVAL_FAILED | index | 检索失败：{channel} | true | error | 按批准降级策略；未批准则报错不降级 | host_visible |
| UNAVAILABLE | storage | 服务不可用（health 相关） | true | critical | 等待恢复 | host_visible |

## 2. 兼容别名映射（DD-AUTHORITY R1 裁决）

m.md §15.5 名称 → 本表规范码：

| 旧名（m.md） | 规范码 | 备注 |
|---|---|---|
| INVALID_ARGUMENT | VALIDATION_ERROR | |
| UNAUTHORIZED | PERMISSION_DENIED | 未认证情形；响应不含数据 |
| FORBIDDEN | PERMISSION_DENIED | 已认证但无权 |
| VALIDATION_FAILED | VALIDATION_ERROR | |
| STORAGE_UNAVAILABLE | STORAGE_ERROR（severity=critical） | 或 UNAVAILABLE（health 层） |
| INDEX_UNAVAILABLE | INDEX_ERROR | |
| EMBEDDING_FAILED | EMBEDDING_ERROR | |
| RETRIEVAL_FAILED | RETRIEVAL_FAILED（保留同名） | |
| BUDGET_EXCEEDED | BUDGET_EXCEEDED（保留同名） | |
| TASK_NOT_FOUND | TASK_NOT_FOUND（保留同名） | |
| INTERNAL_ERROR | INTERNAL_ERROR（保留同名） | |
| NOT_FOUND / VERSION_CONFLICT / DUPLICATE_REQUEST | 同名保留 | |

契约 `error.schema.json` 的 enum 已包含两侧名称全集；规范化输出 MUST 使用本表规范码。

## 3. 错误处理协议（模块必读）

1. 每个错误 MUST 携带 trace_id 与 request_id（可关联）。
2. `exposure_policy=user_visible` 的错误模板 MUST NOT 包含记忆正文、Scope 推断、权限策略细节。
3. 重试：仅 `retryable=true` 的错误可自动重试，且必须幂等（idempotency_key）。
4. fail-closed 三联：PERMISSION_ERROR / TIMEOUT（权限路径）/ 未知状态 ⇒ 一律拒绝，不得降级为放行。
5. 检索通道失败：INDEX_ERROR/EMBEDDING_ERROR 进入 Outbox 重试；是否"部分结果继续返回"必须由已批准降级策略决定，未批准 ⇒ status=failed 并说明；部分结果 MUST 标 `status=partial` + `degraded=true`。
6. 禁止用空数组、默认值或虚假成功掩盖失败（sf 后文 §16）。
7. 错误码新增：MUST 更新本表 + error.schema.json enum + 03 文档错误协议节，并在 CHANGELOG 登记（变更申请十要素）。
