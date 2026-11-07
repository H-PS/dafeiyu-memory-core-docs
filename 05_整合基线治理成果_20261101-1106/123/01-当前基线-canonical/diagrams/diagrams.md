# Memory Core R4.1 — 图集（Diagrams）

> 文件：`diagrams/diagrams.md` ｜ 版本 1.0 ｜ 共 15 张，均为 Mermaid，与 00–08 正文一致。禁止出现图中有、正文无的接口或状态。

## D-01 系统总体架构图

```mermaid
flowchart TB
    U[用户 / 群组 / 外部事件]
    HA[Host Adapter / QQ Bridge]
    DSH[DSH / Agent / LLM]
    API[Memory API /api/v1]
    M02[M02 Retrieval & Context]
    M05[M05 Permission contract_deny]
    M03[M03 Extraction FUTURE]
    M04[M04 Consolidation FUTURE]
    M01[M01 Storage]
    SQ[(SQLite Authority)]
    FTS[(FTS5 Derived)]
    LDB[(LanceDB Derived)]
    OB[Outbox]
    WK[Index Worker]
    DR[Dream FUTURE]
    AU[Audit]
    EV[Evaluation]

    U --> HA
    HA <--> DSH
    HA --> API
    API <--> M02
    M02 --> M05
    M05 -. contract_deny .-> M02
    HA -->|authorized sources| API
    API --> M03
    M03 --> M04
    M04 --> M01
    M01 --> SQ
    M01 --> OB
    OB --> WK
    WK --> FTS
    WK --> LDB
    M02 --> M01
    DR -.-> M03
    DR -.-> M04
    AU -.observes.-> M01
    EV -.measures.-> M02
```

## D-02 M01 数据写入流程（算法：事务五步）

```mermaid
flowchart TD
    A[输入 Schema 校验] -->|FAIL| ERR1[VALIDATION_ERROR]
    A -->|PASS| B[校验 expected_version]
    B -->|冲突| ERR2[VERSION_CONFLICT]
    B -->|OK| C[写 memory_objects]
    C --> D[写 memory_versions]
    D --> E[写 outbox_events]
    E --> F{COMMIT}
    F -->|任一步失败| G[ROLLBACK 全部]
    F -->|成功| H[权威提交完成]
    H --> I[Worker 异步消费 Outbox]
```

## D-03 M01 事务与 Outbox 一致性

```mermaid
sequenceDiagram
    participant App as 调用方
    participant M01 as M01
    participant DB as SQLite
    participant OB as outbox_events
    participant W as Worker
    participant IDX as FTS5+LanceDB
    App->>M01: updateMemory(id,patch,expectedVersion)
    M01->>DB: BEGIN
    M01->>DB: write memory(version+1)
    M01->>DB: write memory_versions
    M01->>OB: insert outbox_event(pending,v_new)
    M01->>DB: COMMIT
    alt 任一步失败
        M01->>DB: ROLLBACK（Outbox 同回滚，绝无"记忆已提交事件丢失"）
    end
    W->>OB: claim pending
    W->>DB: read latest version
    alt v_event < v_current
        W->>OB: markSuperseded
    else
        W->>IDX: upsert FTS5 + LanceDB(v_new,hash)
        W->>OB: markCompleted(仅校验通过后)
    end
```

## D-04 Worker 索引同步流程

```mermaid
flowchart TD
    A[poll pending/next_retry_at<=now] --> B{deleted or absent?}
    B -->|yes| C[FTS5 delete + LanceDB delete + verify]
    C --> D[markCompleted]
    B -->|no| E{v_event < v_current?}
    E -->|yes| F[SkipOrSupersede]
    E -->|no| G[build SearchDocument]
    G --> H[FTS5 upsert]
    H --> I[embedDocument]
    I -->|失败| J[retry_wait: 记录失败,不伪造向量,事件不完成]
    I -->|成功| K[validateVector: 维度/空间/哈希/归一化]
    K -->|fail| J
    K -->|pass| L[LanceDB upsert]
    L --> M{verify consistency}
    M -->|pass| D
    M -->|fail| N[retry_wait]
    N --> O{retry_count > max?}
    O -->|yes| P[dead_letter + 告警人工修复]
```

## D-05 M02 多路检索流程

```mermaid
flowchart TD
    A[current_message] --> B[Unicode/空白/标点规范化]
    B --> C[keywords/named_entities/time_constraints 提取]
    C --> D[buildRetrievalScope]
    D --> E1[SQLite structured filter]
    D --> E2[FTS5 MATCH BM25 Top50]
    D --> E3[embedQuery]
    E3 --> F[validateVector]
    F --> G[LanceDB FLAT searchExact Top50 校验 space_id]
    E1 --> H[合并 C=Cs∪Cf∪Cv 按 memory_id 去重保通道证据]
    E2 --> H
    G --> H
    H --> I[RRF k=60 w=1.0/1.0 rank从1 缺失=0]
    I --> J[六特征规则排序 归一化+惩罚分离]
    J --> K[状态检查+冲突标识]
    K --> L[稳定排序 finalScore DESC, memory_id ASC]
    L --> M[Top 20]
```

## D-06 M02 权限终检流程

```mermaid
flowchart TD
    A[候选 Top20] --> B[SQLite 回源 getLatest]
    B -->|NOT_FOUND| X1[excluded NOT_FOUND]
    B -->|deleted| X2[excluded DELETED]
    B -->|lifecycle/availability 不符| X3[excluded INELIGIBLE]
    B -->|OK| C[M05 canRead]
    C -->|ALLOW| D[继续]
    C -->|DENY| X4[excluded PERMISSION_DENIED]
    C -->|INDETERMINATE/异常/超时| X5[excluded fail-closed]
    D --> E[chooseRepresentation]
    E --> F[TokenCount exact/estimated]
    F -->|超预算| X6[excluded BUDGET_EXCEEDED]
    F -->|OK| G[边际价值 U=Relevance-λ·Redundancy]
    G -->|<=0| X7[excluded REDUNDANT]
    G -->|>0| H[selected]
```

## D-07 M02 Context Builder 流程

```mermaid
flowchart TD
    A[selected 列表] --> B[分区: DirectlyRelevant/StableFacts/HistoricalEvents/ActivePlans/RelevantBackground]
    B --> C[仅注入 ALLOW 项 携带 memory_id/version/type/confidence/source_refs/scope]
    C --> D{冲突记忆?}
    D -->|yes| E[保留冲突标记 不合成结论]
    D -->|no| F[正常组装]
    E --> G[总预算校验 sum<=B]
    F --> G
    G --> H[MemoryContext + selection_trace + permission_trace]
    H --> I[context.assembled 事件 + prompt_injection_event_id L3]
```

## D-08 M03 候选生成流程（FUTURE）

```mermaid
flowchart TD
    A[授权来源: 对话/摘要/明确保存/导入/授权系统事件] --> B[Evidence Capture]
    B --> C[Candidate Extraction]
    C --> D[Type Classification]
    D --> E[Scope Proposal]
    E --> F[Source Validation]
    F --> G[Quality Check 十项]
    G --> H[Candidate Queue candidate/pending_review]
    H -.->|仅 M04 通道| I[M04 review]
```

## D-09 M04 合并与冲突处理流程（FUTURE）

```mermaid
flowchart TD
    A[candidate/记忆组] --> B{精确重复?}
    B -->|yes| C[去重: 合并证据引用]
    B -->|no| D{语义重复?}
    D -->|yes| E[MergeProposal: 主体一致/Scope兼容/来源链保留/时间明确]
    D -->|no| F{冲突?}
    F -->|attribute/temporal/subject/scope/source_conflict| G[标记 disputed 保留双方 不自行裁决]
    F -->|compatible| H[建立 CONFIRMS/RELATED_TO 关系]
    E --> I{授权执行?}
    I -->|yes| J[M01 新版本 + SUPERSEDES + provenance]
    I -->|no| K[保持 pending]
```

## D-10 M05 权限判断流程（契约期恒 DENY）

```mermaid
flowchart TD
    A[PermissionContext] --> B{身份字段齐全?}
    B -->|no| R1[DENY 身份不明]
    B -->|yes| C{action 合法?}
    C -->|no| R2[DENY]
    C -->|yes| D{scope/subject/sensitivity/会话 检查}
    D -->|缺关键字段| R3[DENY fail-closed]
    D -->|齐| E{策略判定}
    E -->|ALLOW| F[decision_id+审计]
    E -->|DENY| R4[DENY]
    E -->|error/timeout/未知| R5[INDETERMINATE=拒绝]
    style R1 fill:#fdd
    style R2 fill:#fdd
    style R3 fill:#fdd
    style R4 fill:#fdd
    style R5 fill:#fdd
```

## D-11 Dream 维护流程（FUTURE）

```mermaid
flowchart TD
    A[Scheduler 触发] --> B[Task Planner]
    B --> C[Permission Check]
    C -->|denied| X[记录并跳过]
    C -->|allowed| D[经模块接口调用 M03/M04/M01/M02]
    D --> E[Result Validation]
    E --> F[Authorized Commit]
    F --> G[Audit Event + dream_run_id 记录]
    A -.只读.-> H[一致性检查 P0-P3 唯一允许项]
```

## D-12 数据删除传播流程

```mermaid
flowchart TD
    A[deleteMemory] --> B[SQLite 事务: 标记 deleted + tombstone + Outbox DELETE]
    B --> C[检索层立即按权威状态排除]
    C --> D[FTS5 delete]
    D --> E[LanceDB delete]
    E --> F[验证索引不存在]
    F --> G[缓存失效]
    G --> H[Observation/Relation 失效或受限]
    H --> I[审计 + 事件 completed]
    D -.失败.-> J[Outbox retry_wait 权威删除不回退]
    E -.失败.-> J
```

## D-13 SQLite 与索引恢复流程

```mermaid
flowchart TD
    A[故障/损坏] --> B[Stop Writers]
    B --> C[Restore SQLite Backup API 快照]
    C --> D[Validate Schema/Integrity]
    D -->|fail| Z[不得标记完全恢复]
    D -->|pass| E[Read Index Registry]
    E --> F[Rebuild or Reconcile FTS5]
    E --> G[Rebuild or Reconcile LanceDB]
    F --> H[Verify Versions + 重放 tombstone]
    G --> H
    H --> I[Process Pending Outbox]
    I --> J[Acceptance Checks 检索回归]
    J -->|pass| K[Resume Service]
    J -->|fail| Z
```

## D-14 Embedding 模型迁移流程（B-011）

```mermaid
stateDiagram-v2
    [*] --> backfilling: 创建新 embedding_space_id
    backfilling --> validating: 覆盖率校验通过
    validating --> dual_running: 检索对比验证通过
    dual_running --> switching: 追赶完成(Outbox=0 且版本一致)
    switching --> monitoring: 路由已切换
    monitoring --> retired: 观察期通过
    backfilling --> rolled_back: 失败
    validating --> rolled_back: 失败
    dual_running --> rolled_back: 失败
    switching --> rolled_back: 失败
    monitoring --> rolled_back: 超阈值回滚
    rolled_back --> [*]
    retired --> [*]
    note right of dual_running
        禁止: 同维度混合/未回填就切换/
        未验证就删旧空间/静默降级旧模型
    end note
```

## D-15 DSH Context Adapter 调用时序图

```mermaid
sequenceDiagram
    participant U as 用户消息
    participant HB as QQ Bridge(Host Adapter)
    participant DSH as DSH/LLM
    participant API as Memory API
    participant M02 as M02
    participant M05 as M05(contract_deny)
    participant M01 as M01/SQLite
    U->>HB: 消息
    HB->>HB: 解析身份 user/group/session/bot/tenant
    HB->>API: POST /memories/recall (retrieval-request)
    API->>M02: retrieve+rank+buildContext
    M02->>M01: 回源 getLatest
    M02->>M05: canRead 逐条
    M05-->>M02: contract_deny=false时按策略; 恒不 allow-all
    M02-->>API: MemoryContext(授权上下文+trace)
    API-->>HB: 响应(八字段)
    HB->>DSH: 注入上下文(携带 memory_id/version/type/confidence/source_refs/scope)
    DSH-->>HB: 回复
    HB->>API: POST /memories/retain(授权来源候选)
    API-->>HB: task_id / candidate_ids
    HB->>API: context.assembled → prompt_injection_event_id (L3 仅证明进入 Prompt)
```
