# 文档与代码差异表（22-DOCUMENT-CODE-DIFF）

采集：2026-11-06 ｜ 主代理直接比对 CMB `docs/` 与 `src/` 实际实现

---

## 1. README.md

| 文档位置 | 文档说法 | 代码实际 | 判定 |
|---|---|---|---|
| 模块清单 | "M01–M12 全模块" | M09 零调用、M02b/M06 不可达 | **夸大** |
| 运行环境 | "Node >= 24，零依赖" | ✅ 属实（`package.json` dependencies 空） | PASS |
| 测试命令 | `node tests/runner.ts` | ✅ 属实，78 passed | PASS |
| 限制 L1-L9 | 列出 9 条限制 | 未覆盖本轮发现的 6 个 BLOCKER | **不完整** |
| demo 输出示例 | 展示运行结果 | ✅ 属实（主代理复跑一致） | PASS |

## 2. ARCHITECTURE.md

| 文档位置 | 文档说法 | 代码实际 | 判定 |
|---|---|---|---|
| 七步流 | Discover→Resolve→Validate→Build→Write→Verify→Ledger | ✅ 流程存在，但 "Verify" 仅 `getByIdentity !== null`，非真正回读校验 | **弱化** |
| 五标识三实两缺 | 声称三实（session/group/speaker） | **speaker 实为零实**（真实源恒 null） | **错误** |
| 六不原则 | Summary 不属于 Evidence 等 | ✅ 结构性成立（M09 完全不接入） | PASS |
| batch_size/concurrency | 列为可配置 | 引擎零引用 | **未实现** |
| Evidence 与 B-1 | 声明冲突未解 | ✅ 属实 | PASS |

## 3. ADR-001 ~ ADR-007

| ADR | 声称 | 实际 | 判定 |
|---|---|---|---|
| ADR-001 | 读侧对账，不 hook 压缩 | ✅ 属实 | PASS |
| ADR-002 | direct 优先、区间恢复 | ✅ 属实 | PASS |
| ADR-003 | 三层幂等键 | 存在但 Evidence 层会撞键（R-09）、content_hash 语义错（R-02） | **失效** |
| ADR-003 | timestamp 恒 null 因 ts 二义 | ✅ 有依据（`store.ts:1031-1038`） | PASS |
| ADR-004 | 不按 active 过滤、无墓碑扫描 | ✅ 属实 | PASS |
| ADR-005 | 缺失即拒绝、不降级昵称 | M-2 缺陷下缺失被静默吸收（R-10） | **部分失效** |
| ADR-006 | Mock/真实边界清晰 | 清晰，但真实端零可用 | PASS（诚实） |
| ADR-007 | 至少一次 + 幂等消费、Ledger 非唯一事实 | 逻辑正确，但跨进程未验证（R-11） | **PASS_WITH_LIMITATION** |

## 4. TEST-REPORT.md

| 文档位置 | 文档说法 | 实际 | 判定 |
|---|---|---|---|
| §1 总览 | "78 passed" | ✅ 主代理复跑一致 | PASS |
| §5 安全 | SEC-1…12 全绿 | ✅ 属实 | PASS |
| §5 SEC-7 | "system 提示词不进 Evidence" | **该测试用 `contentType:"system"` 构造，真实上游从不用该值** | **测试无效** |
| §6 性能 | 三档数据 | ✅ 属实 | PASS |
| §8 诚实清单 | 列 12 项未覆盖 | 未包含本轮 6 个 BLOCKER | **不完整** |
| §9 结论 | "崩溃可恢复" | 全部 `:memory:`，跨进程未验证（R-11） | **高估** |

## 5. INTERFACE-VERIFICATION.md

| 文档位置 | 文档说法 | 实际 | 判定 |
|---|---|---|---|
| VERIFIED_STATIC 22 项 | 含上游字段/DDL | ✅ 大体准确，但 `content_hash` 语义描述不足 | MINOR |
| MOCK_ONLY 10 项 | Mock 能力清单 | ✅ 属实 | PASS |
| UNVERIFIED 12 项 | 未验证清单 | 缺少 speaker 链断裂、media hydrate | **不完整** |
| UNSUPPORTED 17 项 | 不支持清单 | ✅ 属实 | PASS |

## 6. RISK-REGISTER.md（CMB 自己的）

| 项 | 实际 | 判定 |
|---|---|---|
| 35 项风险 | 未包含本轮 6 个 BLOCKER | **严重不完整** |
| P0 = 9 项 | 需用户裁决 4 项（D-6~D-9） | ✅ 属实但已过时（本轮扩为 6 项） |

## 7. INTEGRATION-READINESS.md

| 文档回答 | 实际 | 判定 |
|---|---|---|
| 问题 6「可接入真实 billion-context」 | 答"不可" | ✅ 判断正确，但**理由不足**（真正原因是 speaker 链断裂，非仅 session_type） |
| 问题 7「可接入 Memory Core」 | 答"不可" | ✅ 正确 |
| 问题 10「是否建议下一阶段」 | 建议"受限真实数据验证" | ✅ 但仍成立，只是前置条件更多 |

## 8. PERF-REPORT.md 与 PERFORMANCE 相关

| 文档说法 | 实际 | 判定 |
|---|---|---|
| 吞吐 373-847 msgs/s | 口径未说明，可高估 2 倍 | MINOR |
| 幂等重扫快 194 倍 | 实为"不做任何工作的开销" | **表述误导** |
| 峰值 RSS | 仅 2 次采样 | MINOR |

## 9. PHASE6-CHANGES.md

| 文档说法 | 实际 | 判定 |
|---|---|---|
| B-1 修复完成 | ✅ 部分（UNSUPPORTED 漏了） | **PASS_WITH_LIMITATION** |
| M-2 列为"裁决不修" | ✅ 诚实记录，但**未评估数据完整性后果** | MINOR |
| M-4 列为"接入前必修" | ✅ 诚实 | PASS |

## 10. security-review-agentE.md（被审对象外）

| 文档说法 | 实际 | 判定 |
|---|---|---|
| B-1 BLOCKER 描述 | ✅ 准确 | PASS |
| M-1~M-6 | ✅ 准确 | PASS |
| 建议修复 | 部分已修（M-1/M-5），部分未修（M-2/M-4） | ✅ 诚实 |

---

## 11. 汇总：文档可信度评级

| 文档 | 与代码一致性 | 评级 |
|---|---|---|
| `README.md` | 主要事实正确，模块完整性夸大 | **B** |
| `ARCHITECTURE.md` | 流程正确，"三实"错误 | **B-** |
| `ADR-001~007` | 大多准确，ADR-003/005 有失效 | **B** |
| `TEST-REPORT.md` | 数字正确，SEC-7 无效，"崩溃恢复"高估 | **B-** |
| `INTERFACE-VERIFICATION.md` | 大体准确，未验证清单不全 | **B+** |
| `RISK-REGISTER.md`（CMB 版） | **严重不完整** | **C** |
| `INTEGRATION-READINESS.md` | 结论方向对，理由不足 | **B** |
| `PERF-REPORT.md` | 数据真实，表述有误导 | **B-** |
| `PHASE6-CHANGES.md` | 诚实 | **A-** |

## 12. 最严重的文档问题

1. **`RISK-REGISTER.md` 缺 6 个 BLOCKER** —— 外部审阅者若只看这一份，会严重低估风险
2. **`ARCHITECTURE.md` 的"三实两缺"** —— speaker 实际是"零实"，误导读者以为身份可用
3. **`TEST-REPORT.md` SEC-7 无效测试** —— 声称验证了 system 排除，实际构造了上游不存在的值
4. **`TEST-REPORT.md` "崩溃可恢复"** —— 全部用 `:memory:`，从未跨进程验证
