# 图表集（diagrams/）

采集：2026-11-06 ｜ 主代理绘制
**说明**：本文集中任务书 §26.8 要求但其他报告未涵盖的图表。已存在的图在对应报告中。

---

## 1. 系统总体架构图（含可达性标注）

```mermaid
graph TB
    subgraph External["外部系统（只读）"]
        BC["billion-context<br/>index.db + CAS blobs<br/>HEAD 0aace2f"]
        MC["Memory Core<br/>⚠️ 仅设计文档<br/>无实现代码"]
        HOST["Host<br/>QQ Bridge + DSH<br/>⚠️ DSH asar 不可读"]
    end

    subgraph CMB["CMB v0.1（隔离原型）"]
        CLI["CLI<br/>src/cli/cmb.ts"]
        ENG["Reconciliation Engine<br/>M08"]
        RES["Source Resolver<br/>M03"]
        VAL["Integrity Validator<br/>M04"]
        ADP["Evidence Adapter<br/>M05"]
        SUM["Summary Artifact<br/>M09 ⚠️ 孤立"]
        LED["Bridge Ledger<br/>M07 SQLite"]
        OBS["Observability<br/>M11"]
    end

    subgraph Adapters["适配器层"]
        BCA["BillionContextSourceAdapter<br/>⚠️ 无生产调用者"]
        MSA["MockSourceStore<br/>✅ CLI 实际使用"]
        IDP["HostIdentityProvider<br/>⚠️ speaker 恒 null"]
        MCS["MemoryCoreEvidenceAdapter<br/>⚠️ 全方法抛异常"]
        MES["MockEvidenceSink<br/>✅ 实际使用（内存 Map）"]
    end

    CLI -->|"硬编码组装"| MSA
    CLI --> ENG
    ENG --> RES
    ENG --> VAL
    ENG --> ADP
    ENG --> LED
    ENG --> OBS
    RES --> MSA
    VAL --> IDP
    ADP --> MES

    BCA -.->|"implements<br/>但从未调用"| RES
    MCS -.->|"implements<br/>但全抛异常"| ADP
    BC -.->|"只读连接<br/>UNVERIFIED"| BCA
    MC -.->|"无入口"| MCS
    HOST -.->|"身份应由此注入<br/>未实现"| IDP
    SUM -.->|"零接入"| ENG

    style MC fill:#f99
    style MCS fill:#f99
    style BCA fill:#fc9
    style IDP fill:#fc9
    style SUM fill:#fcc
    style MSA fill:#cfc
    style MES fill:#cfc
```

**图例**：绿=实际可用；橙=存在但不可达；红=空壳/孤立。

---

## 2. Evidence 生命周期图

```mermaid
stateDiagram-v2
    [*] --> NotExists : 消息存在于源库

    NotExists --> Discovered : 块覆盖该消息
    Discovered --> Resolved : getMessages 找到
    Discovered --> SourceMissing : tombstone（retired_at）

    Resolved --> TextExtracted : JSON.parse(blob)
    Resolved --> BlobMissing : content.bytes 为 null
    Resolved --> MediaUnhydrated : 含图片（⚠️ 只拿占位符）

    TextExtracted --> Validated : hash 回算通过
    TextExtracted --> HashMismatch : 回算不符
    Validated --> SpeakerRejected : ⚠️ 真实源必走此路
    Validated --> Built : speaker 可达
    Built --> Written : sink.write 成功
    Built --> WriteFailed : sink 抛错
    Written --> Ledgered : ledger.upsertMessage

    Ledgered --> Revoked : revokeScan 检测源不可见
    Revoked --> Conflict : ⚠️ 上游复活 → SINK_CONFLICT
    Conflict --> [*] : 永久卡死

    SourceMissing --> [*] : 不产生 Evidence
    BlobMissing --> [*]
    MediaUnhydrated --> Written : ⚠️ 内容不完整也写入
    HashMismatch --> [*]
    SpeakerRejected --> [*] : ⚠️ 真实源 100% 走此路
    WriteFailed --> [*] : 下轮重试

    note right of SpeakerRejected
      D-01 BLOCKER
      上游无 speaker 列
      CMB 无条件强制
    end note
    note right of Conflict
      D-15
      无 un-revoke 接口
      REVOKED 是终态
    end note
```

---

## 3. 故障恢复流程图

```mermaid
flowchart TD
    Start["run() 开始"] --> StartRun["ledger.startRun()<br/>end_time=NULL"]
    StartRun --> LS{"store.listSessions()"}
    LS -->|"抛错"| Crash1["❌ 无 try<br/>run() 崩溃<br/>run_ledger 半开"]
    LS -->|"ok"| LB["store.listBlocks()"]
    LB -->|"抛错"| Crash1
    LB -->|"ok"| Loop["遍历块"]

    Loop --> GS{"ledger.getBlock()"}
    GS --> Idem{"COMMITTED 且<br/>version 未变?"}
    Idem -->|"是"| Skip["skipped++<br/>continue ✅"]
    Idem -->|"否"| Term{"REVOKED/<br/>SUPERSEDED?"}
    Term -->|"是"| Skip2["terminal skip ✅"]
    Term -->|"否"| Reset{"复位到 DISCOVERED"}

    Reset -->|"PARTIAL/READY/<br/>RESOLVING/VALIDATING"| ViaFailed["→FAILED→DISCOVERED ✅"]
    Reset -->|"MISSING/CORRUPTED/FAILED"| Direct["→DISCOVERED ✅"]
    Reset -->|"UNSUPPORTED"| Fail1["❌ UNSUPPORTED→FAILED 非法<br/>STATE_RESET_FAIL"]
    Reset -->|"COMMITTED (payload 变)"| Fail2["❌ COMMITTED→FAILED 非法<br/>STATE_RESET_FAIL"]

    ViaFailed --> Process["resolve→validate→write"]
    Direct --> Process
    Fail1 --> Stuck1["块永久停在 UNSUPPORTED"]
    Fail2 --> Stuck2["块永久停在 COMMITTED<br/>summary 更新永不生效"]

    Process --> WriteLoop{"sink.write()"}
    WriteLoop -->|"成功"| Verify{"getByIdentity"}
    WriteLoop -->|"抛错"| WF["writeFailures++<br/>下轮重试 ✅"]
    Verify -->|"null"| VerifyFail["throw → catch<br/>下轮重试 ✅"]
    Verify -->|"ok"| UpsertLedger["ledger.upsertMessage()"]
    UpsertLedger -->|"成功"| Count["evidence_written++ ✅"]
    UpsertLedger -->|"抛错"| WF

    Count --> Persist["persistBlockRow()"]
    Persist --> Loop
    WF --> Persist
    Stuck1 --> Loop
    Stuck2 --> Loop

    Loop -->|"结束"| Finish["ledger.finishRun()<br/>end_time=now ✅"]
    Finish --> Report["返回报告"]

    style Crash1 fill:#f99
    style Fail1 fill:#f99
    style Fail2 fill:#f99
    style Stuck1 fill:#fcc
    style Stuck2 fill:#fcc
    style Skip fill:#cfc
    style Skip2 fill:#cfc
```

---

## 4. 身份映射图（五标识来源与断链）

```mermaid
graph LR
    subgraph Upstream["billion-context index.db"]
        S1["sessions.session_id ✅"]
        S2["sessions.parent_session_id ✅"]
        S3["⚠️ 无 tenant_id"]
        S4["⚠️ 无 bot_id"]
        S5["⚠️ 无 group_id"]
        S6["⚠️ messages 表无 speaker 列"]
    end

    subgraph Bridge["QQ Bridge"]
        B1["convKey = kind:id<br/>bridge.js:1220 ✅"]
        B2["event.user_id<br/>bridge.js:11238 ✅"]
        B3["event.self_id ✅"]
        B4["⚠️ 无 tenant 概念<br/>全仓 0 次"]
    end

    subgraph CMBIdentity["CMB HostIdentityProvider"]
        I1["resolveTenant → 部署常量"]
        I2["resolveBot → 部署常量"]
        I3["resolveGroup → 手工 Map"]
        I4["resolveSpeaker → hint ?? bound"]
        I5["resolveVisibility → 依 session_type"]
    end

    subgraph Evidence["Evidence"]
        E1["tenant_id (常量)"]
        E2["bot_id (常量)"]
        E3["group_id"]
        E4["⚠️ speaker_id = null"]
        E5["visibility"]
    end

    S1 --> I5
    B1 --> I3
    B3 --> I2
    B4 -.->|"不存在"| I1
    S6 -.->|"空"| I4
    B2 -.->|"未接入"| I4

    I1 --> E1
    I2 --> E2
    I3 --> E3
    I4 --> E4
    I5 --> E5

    style E4 fill:#f99
    style S6 fill:#f99
    style B4 fill:#f99
    style I1 fill:#fc9
    style I2 fill:#fc9
    style I3 fill:#fc9
```

---

## 5. v0.1 → v1.0 演进图

```mermaid
graph LR
    V01["v0.1<br/>独立原型<br/>✅ 已完成<br/>78 测试<br/>6 BLOCKER"]
    V02["v0.2<br/>真实数据只读验证<br/>⛔ 需先修 4 项<br/>需授权副本"]
    V03["v0.3<br/>真实 Source Adapter<br/>⛔ 需 D-5 speaker 裁决"]
    V04["v0.4<br/>隔离 Evidence Adapter<br/>⛔ 需 D-1 schema 裁决"]
    V05["v0.5<br/>端到端沙盒<br/>⛔ 需 Memory Core 实现"]
    V10["v1.0<br/>生产接入<br/>⛔ 需全部 18 项阻塞解决"]

    V01 -->|"修 D-59/D-04/D-06<br/>+D-02/D-05/D-07/D-08/D-10"| V02
    V02 -->|"E-1/E-2/E-3 通过<br/>+D-5 裁决"| V03
    V03 -->|"E-6/E-11 通过<br/>+D-1 裁决"| V04
    V04 -->|"E-8 通过<br/>+Memory Core 实现"| V05
    V05 -->|"E-7/E-9/E-10/E-12<br/>+权限/审计/回滚"| V10

    style V01 fill:#cfc
    style V02 fill:#ffc
    style V03 fill:#ffc
    style V04 fill:#fcc
    style V05 fill:#fcc
    style V10 fill:#f99
```

---

## 6. 数据流与三种数据存储的关系

```mermaid
graph TB
    subgraph SourceSide["源侧（billion-context）"]
        IDX["index.db<br/>SQLite + WAL<br/>sessions/messages/blocks/content"]
        LEG["legacy JSON 文件<br/>（加密模式下唯一存储）"]
    end

    subgraph CMBSide["CMB 侧"]
        ML["CMB 数据源适配器<br/>readOnly 连接"]
        CL["CMB Ledger<br/>独立的 index-like SQLite<br/>block_ledger/message_ledger/run_ledger"]
        SA["Summary Artifact Store<br/>独立 SQLite（零接入）"]
        MS["MockEvidenceSink<br/>进程内 Map"]
    end

    subgraph Dest["目标侧"]
        MCD["Memory Core 数据<br/>⚠️ 不存在"]
    end

    IDX -->|"SELECT（只读）"| ML
    LEG -.->|"⚠️ CMB 不读"| ML
    ML --> CL
    ML --> SA
    ML --> MS
    MS -.->|"⚠️ 无实现"| MCD

    style LEG fill:#fc9
    style MCD fill:#f99
    style SA fill:#fcc
```
