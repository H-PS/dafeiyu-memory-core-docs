# 实施前事实核查清单（F-CK-1 ～ F-CK-9）· R4.1 · CR-08A

> **文件性质**：审查导航摘要。**全部条目均未核查**——依赖 DSH/Bridge 实际代码与运行环境，不能由文档审查代替。核查前 L3 保持不可用（属既有「L3 实现暂缓」的组成部分）。
>
> 相关代码位置背景（源自 CR-08A 第二阶段定向核查，只读证据，路径 `[bridge-root]`）：
> - `node_modules\@deepseek-ai\dsh-host-apiproxy\lib\types\api\sessions.d.ts`（prompt 契约：sessionId/mode/content/clientTimeZone——**无关联字段**）
> - `node_modules\@deepseek-ai\dsh-session\lib\types\types.d.ts`（SessionEventMap：turn/start 仅 `{turn:number}`；**无 prompt/assembled**）
> - `node_modules\@deepseek-ai\dsh-agent\lib\index.js` L273（system-prompt/assemble 钩子）
> - `node_modules\@deepseek-ai\dsh-llm\lib\types\call-config.js`（LlmCallConfig 含 temperature——内部字段，会话 API 不暴露）

| # | 核查对象 | 实际责任人 | 证据来源 | 通过条件 | 未通过时的处理 |
|---|---|---|---|---|---|
| F-CK-1 | DSH 能否执行全部 A 类自检（A1–A8） | DSH 实现方 | 发出前自检日志或等价机制 | 自检覆盖 A1–A8 且 result 与之一致 | 证据不得标通过性 result；L3 不可用 |
| F-CK-2 | Bridge 能否取得 A4 跨对象引用（mappingEvidenceRef 绑定对象） | DSH 接口 + Bridge 实现方 | 接口文档 + 实测 | Bridge 可解析并核对绑定 | 该项 not_executable → 整体不得 passed |
| F-CK-3 | Bridge 能否取得 removalBasis.ref 对象 | 同上 | 同上 | 同上 | 同上 |
| F-CK-4 | Bridge 能否在**不落盘正文**前提下完成 A7 摘要比对 | DSH 接口 + Bridge 实现方 | 校验机制设计 + 实测 | 存在不落盘的摘要重算或等价核验路径 | A7 为生产方声明 → 整体不得 passed |
| F-CK-5 | 最终 Prompt 实际组装阶段（= 事实表 F8） | DSH 源码核实 | 源码定位 | 观察记录产生于过滤/截断/拼接全部完成后（system-prompt/assemble 是否该阶段——**待核实**） | 不得作 L3 证据来源 |
| F-CK-6 | 证据无效记录能否独立保存 | 存储设计方 | 审计存储设计 | invalid 记录独立于确认事件存在 | 补存储设计后 L3 方可启用 |
| F-CK-7 | 事件消费方是否强制检查证据有效性 | 消费方实现方 | 消费路径代码审查 | 全部消费路径以整体有效性为前置 | 修复前 L3 不可用 |
| F-CK-8 | 现有版本/事件/状态字段能否表达本节规则（含 E-1 扩展 hasUnknownVersions） | DSH + Bridge 实现方 | 契约核对 | 全部所需字段存在或扩展已实现 | 完成扩展前 L3 不可用 |
| **F-CK-9** | **第二阶段确认记录的存储与消费机制（E-2）** | Bridge 实现方 + 存储设计 | 记录/消费路径设计 + 实测 | 确认记录可独立产生、按 correlationId/assemblyId 关联第一阶段、消费方可区分两阶段 | 完成前不得产出任何有效确认记录；L3 不可用 |

## 附加核查点（v4.6 增量规范识别，尚无编号——供 Claude 评估是否应并入 F-CK）

**F-CK-10（候选）：第一阶段观察集合的数据来源**——`memoryRefsInFinalPrompt` 是从最终组装后的实际内容来源追踪中取得（独立观察），还是从 DSH 自己的 completeness/evidence 字段复制。若为复制，检查 ③ 的独立性前提受损。**此为 v4.6 候选明确标记的未解决问题（delta-spec 问题一），建议 Claude 评估其定级与归属。**

## 其他既存待核实事实（源自事实表，非 F-CK 编号体系）

- `~/.dsh/settings.yaml` provider 层温度配置（未核查）；
- DSH agent loop 内部能力性表述过滤（B05，需源码确认）；
- 运行中 DSH 实例版本（桥接注释按 0.1.5-rc.1 协议复核）；
- `role.maxInjectChars` 截断代码精确位置；配置读取时机；
- hint Token 实测；字素簇计数实现选型；
- TTL 数值、去重表参数、摘要密钥管理、prompt-dump 位置/期限/执行者（均实施前待定）。
