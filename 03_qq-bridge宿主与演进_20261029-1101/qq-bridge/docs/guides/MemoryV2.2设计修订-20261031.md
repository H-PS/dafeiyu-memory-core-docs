# Memory V2.2 设计修订

- 修订时间：2026-10-31 12:20
- 前置：`MemoryV2.1设计修订-20261031.md`
- 性质：**纯设计文档，未实施代码。** 未修改代码/配置/数据，未重启服务。
- 标注约定：`[已确定]` / `[候选方案]` / `[待决策]`

---

# 〇、V2.1 → V2.2 修订对照

| # | V2.1 的问题 | 严重度 | V2.2 修订 |
|---|---|---|---|
| 1 | **factKey 查找嵌在语义候选之后**（流程：精确去重 → 语义候选>0.85 → factKey 验证）→ 相似度<0.85 时同 factKey 的旧版本被漏掉 | **P0 结构性错误** | factKey 查找与相似度**完全解耦**，双路径并行 |
| 2 | C4 说 factKey 只允许 fact/preference；4.1 暗示 relationship 可带 | 矛盾 | C4 改为允许 `fact\|preference\|relationship`，用模板隔离命名空间 |
| 3 | 场景 C 说 `owner.device.gpu` "格式验证失败" | **描述错误** | 该 key 是三段、格式合法；失败原因是**qualifier 业务规则**（该 attr 要求必须带 device 位） |
| 4 | superseded 复用 `archivedAt` | 语义污染 | 新增 `supersededAt`，与 `archivedAt` 互斥 |
| 5 | 删除无 tombstone，防复活只靠 WAL 检查 | 防护不足 | tombstone 表 + 四道防线 |
| 6 | WAL 单一 status，A/B 混在一起 | 无法知道补写哪层 | 拆 `operationStatus` + `aStatus` + `bStatus` |
| 7 | 崩溃恢复只查 opId 存在 | 可能误判"已完成" | 必须比对 `contentHash` 与 payload 期望状态 |
| 8 | 锁 30 秒即强制接管 | **危险** | PID+instanceId+心跳+claim 二次确认+无法确认则放弃 |
| 9 | 注册表未定义精确/模板 | 无法实施 | 模板白名单 |
| 10 | P3(supersede)/P4(delete) 先于 P2(WAL) | **违反可靠性优先** | WAL 提前到 P2，supersede/delete 推迟到 P3/P4 |

---

# 一、完整数据模型

## 1.1 字段表（V2.2 定稿）

| 字段 | 类型 | 默认 | 必填 | 说明 | 状态 |
|---|---|---|---|---|---|
| `id` | string | 生成 | ✅ | `mem_<base36(ts)>_<rand6>` | `[已确定]` |
| `schemaVersion` | number | 2 | ✅ | 结构版本 | `[已确定]` |
| `ownerId` | string | — | ✅ | 记忆归属：`owner:<qq>` / `group:<群号>` | `[已确定]` |
| `conversationKey` | string | — | ✅ | 产生上下文：`private:<qq>` / `group:<群号>` | `[已确定]` |
| `subjectId` | string | `unknown` | ✅ | 内容主体：`owner` / `member:<qq>` / `bot` / `topic:<slug>` | `[已确定]` |
| `visibility` | enum | `owner_private` | ✅ | `owner_private` / `group_shared` | `[已确定]` |
| `category` | enum | `unknown` | ✅ | `activeTopic` / `pendingThought` / `memberImpression` / `manual` / `unknown` | `[已确定]` |
| `type` | enum | `event` | ✅ | `fact` / `event` / `preference` / `temporary` / `relationship` | `[已确定]`（默认值见 §11.3） |
| `status` | enum | `active` | ✅ | `active` / `superseded` / `archived` / `deleted` | `[已确定]` |
| `reviewStatus` | enum | `approved` | ✅ | `pending` / `approved` / `rejected` | `[已确定]` |
| `shareReviewStatus` | enum | `not_requested` | ✅ | `not_requested` / `pending` / `approved` / `rejected` | `[已确定]` |
| `content` | string | — | ✅ | 正文，**不含 category 前缀** | `[已确定]` |
| `contentHash` | string | 自动 | ✅ | `sha256(normalize(content))` 全 64 位 | `[已确定]`（见 §11.2） |
| `createdAt` | number | now | ✅ | 写入时刻 | `[已确定]` |
| `occurredAt` | number\|null | null | ❌ | 事件发生时刻 | `[已确定]` |
| `updatedAt` | number | now | ✅ | 最后修改 | `[已确定]` |
| `supersededAt` | number\|null | null | ❌ | **被替代时刻（新增）** | `[已确定]` |
| `archivedAt` | number\|null | null | ❌ | **主动归档时刻（语义收窄）** | `[已确定]` |
| `deletedAt` | number\|null | null | ❌ | 删除时刻 | `[已确定]` |
| `expiresAt` | number\|null | null | ❌ | temporary 的 TTL 到期时刻（新增） | `[已确定]` |
| `factKey` | string\|null | null | ❌ | 事实键 | `[已确定]` |
| `supersedes` | string\|null | null | ❌ | 本记忆替代了谁 | `[已确定]` |
| `supersededBy` | string\|null | null | ❌ | 本记忆被谁替代 | `[已确定]` |
| `source` | enum | `model` | ✅ | `model` / `manual` / `migration` / `import` | `[已确定]` |
| `sourceMemoryIds` | string[] | `[]` | ❌ | 关联记忆 | `[已确定]`（字段保留，逻辑 P6） |
| `bRefs` | object[] | `[]` | ❌ | B 层引用 `[{conversationKey, category, slotKey}]` | `[已确定]`（逻辑 P6） |
| `opId` | string\|null | null | ❌ | 产生本记录的 operationId | `[已确定]` |
| `vec` | number[] | — | ✅ | 1024 维 | `[已确定]` |
| `vecModel` | string | `bge-m3` | ✅ | 向量模型标识 | `[已确定]` |

## 1.2 三个时间字段的严格区分（修订 V2.1 的 archivedAt 混用）

```
supersededAt  只表示"被新事实替代"
  条件：status=superseded 必须有，其他状态必须为 null
  语义：这条曾是有效事实，某个时刻被更好的版本取代了

archivedAt    只表示"主人主动归档"
  条件：status=archived 必须有，其他状态必须为 null
  语义：这条内容没问题，只是暂时不想让它参与检索

deletedAt     只表示"逻辑删除"
  条件：status=deleted 必须有，其他状态必须为 null
  语义：这条不应再出现

★ 三者互斥：一条记录的 status 只可能是其中一个值，
  因此三个时间字段最多只有一个非 null。
  违反 → 数据校验失败，标记 reviewStatus=pending。
```

**诊断价值**：如果一条记录 `supersededAt` 有值但 `status=active`，说明状态机被破坏，能立刻发现。

## 1.3 tombstone 表（新增）

独立文件 `state/memory-tombstones.jsonl`，删除时写入：

```jsonl
{"memoryId":"mem_x","ownerId":"owner:[OWNER_QQ]","contentHash":"a1b2...","deletedAt":1793400000000,"deletedBy":"console","reason":"user_request","hard":false}
```

| 字段 | 说明 |
|---|---|
| `memoryId` | 被删记忆的 id |
| `ownerId` | 归属（用于缩小查找范围） |
| `contentHash` | **防复活的核心依据** |
| `deletedAt` / `deletedBy` / `reason` | 审计 |
| `hard` | 是否物理删除过 |

**为什么独立成表**：`memory-vectors.jsonl` 可能被 compact/迁移重建，tombstone 必须独立持久化，否则一重建就丢了防复活依据。

---

# 二、factKey 规范与验证

## 2.1 格式规范（修订 member 与 qualifier）

```
结构：<subject>.<entity>.<attribute>[.<qualifier>]
      或 <subject>.<qq>.<entity>[.<qualifier>]  （member 专用）

subject 枚举：
  owner          主人
  bot            机器人自己
  member.<qq>    群友（qq = 5~12 位纯数字）
  topic.<slug>   话题（slug = 2~32 位小写字母/数字/连字符）

entity    [a-z][a-z0-9]{1,20}
attribute [a-z][a-z0-9]{1,20}
qualifier [a-z][a-z0-9]{1,20}   可选（但有必填条件，见 2.2）

三段正则（因为 member 段含数字，必须分开写）：

① /^(owner|bot)\.([a-z][a-z0-9]{1,20})\.([a-z][a-z0-9]{1,20})(\.([a-z][a-z0-9]{1,20}))?$/

② /^member\.(\d{5,12})\.([a-z][a-z0-9]{1,20})(\.([a-z][a-z0-9]{1,20}))?$/

③ /^topic\.([a-z0-9-]{2,32})\.([a-z][a-z0-9]{1,20})(\.([a-z][a-z0-9]{1,20}))?$/
```

**注意**：正则只校验**格式**。`owner.device.gpu` 三段齐全，**格式完全合法**——这是 V2.1 场景 C 的描述错误。它失败在**业务规则**（见 2.2）。

## 2.2 qualifier 合法性与必填条件

```
qualifier 的合法性来自注册表模板，不是正则。

必填条件（三条，满足任一即必须带 qualifier）：
  B1  该 (entity, attribute) 在模板中标记 qualifierRequired=true
  B2  同 ownerId 下已存在同 entity+attribute 但带 qualifier 的记录
      （防"先有 desktop 再有裸的"造成混用）
  B3  该 attribute 在模板中枚举了多个合法 qualifier 值

禁止条件：
  F1  qualifier 不在该模板的枚举白名单内
  F2  同一 (entity, attribute) 下 qualifier 混用风格
      （有 desktop/lab 又有空 → 拒绝）
```

## 2.3 注册表：模板白名单（推荐方案）

`[已确定]` 采用**模板白名单**。理由：精确白名单每加一设备就要改代码；模板白名单既能限制范围，又能扩展。

`state/fact-keys.json`：

```json
{
  "version": 1,
  "templates": [
    {
      "id": "device",
      "pattern": "owner.device.<qualifier>.<attr>",
      "qualifiers": ["desktop", "lab", "phone", "work", "home"],
      "attrs": ["gpu", "cpu", "ram", "disk", "os", "monitor"],
      "qualifierRequired": true,
      "allowedTypes": ["fact"],
      "conflictPolicy": "supersede"
    },
    {
      "id": "preference",
      "pattern": "owner.preference.<domain>",
      "domains": ["food", "game", "music", "sleep", "social", "work"],
      "qualifierRequired": false,
      "allowedTypes": ["fact", "preference"],
      "conflictPolicy": "supersede_with_review_if_negation"
    },
    {
      "id": "schedule",
      "pattern": "owner.schedule.<aspect>",
      "aspects": ["sleep", "meal", "study"],
      "qualifierRequired": false,
      "allowedTypes": ["fact", "preference"],
      "conflictPolicy": "supersede"
    },
    {
      "id": "member_relation",
      "pattern": "member.<qq>.relation",
      "qualifierRequired": false,
      "allowedTypes": ["relationship"],
      "conflictPolicy": "keep_both"
    },
    {
      "id": "bot_persona",
      "pattern": "bot.persona.<aspect>",
      "aspects": ["name", "mood", "habit"],
      "qualifierRequired": false,
      "allowedTypes": ["fact", "preference"],
      "conflictPolicy": "supersede_with_review"
    }
  ]
}
```

**`conflictPolicy` 的作用**：不同模板的冲突处理策略不同。
- `supersede`：同 factKey 直接替换
- `supersede_with_review`：替换但新记录 `reviewStatus=pending`
- `supersede_with_review_if_negation`：仅当检测到否定差异时才 pending
- `keep_both`：永不自动替换（关系类）

## 2.4 C4 修订（解决 V2.1 矛盾）

```diff
- C4  factKey 只允许 type ∈ {fact, preference}
-     type ∈ {event, temporary, relationship} 时 factKey 必须为 null

+ C4  factKey 允许的 type：fact | preference | relationship
+     ★ relationship 允许 factKey，但仅限注册表中 allowedTypes 含 relationship 的模板
+       （当前只有 member.<qq>.relation 一个）
+     event 与 temporary 不允许 factKey：
+       event 是历史事件，多次发生不构成冲突，不需要版本
+       temporary 会自然过期，不需要 supersede
+     违反 → 丢弃 factKey，降级普通写入，不改旧记录
```

**为什么 relationship 允许**：`member.<qq>.relation` 表达"与某人的关系类型"，这个会变（普通群友 → 熟人），需要版本追踪。但用 `keep_both` 策略——关系的变化不是"取代"，而是"叠加历史"。

## 2.5 合法 / 非法示例全表

| factKey | 判定 | 原因 |
|---|---|---|
| `owner.device.desktop.gpu` | ✅ 合法 | 三段 + qualifier 在枚举内 |
| `owner.device.lab.gpu` | ✅ 合法 | 不同 qualifier = 不同事实 |
| `owner.preference.food` | ✅ 合法 | qualifierRequired=false |
| `owner.schedule.sleep` | ✅ 合法 | 同上 |
| `member.[OWNER_QQ].relation` | ✅ 合法 | member 段是数字，专用正则 |
| `bot.persona.name` | ✅ 合法 | |
| `topic.stm32-debug.issue` | ✅ 合法 | topic slug 含连字符 |
| `owner.device.gpu` | ❌ 业务拒绝 | **格式合法**（三段），但 `qualifierRequired=true` 缺 device 位 |
| `owner.device.desktop.gpu.scalar` | ❌ 格式拒绝 | 超过四段 |
| `owner.device.server.gpu` | ❌ 业务拒绝 | qualifier `server` 不在枚举 |
| `member.abc.relation` | ❌ 格式拒绝 | member 段必须是 5~12 位数字 |
| `member.123.relation` | ❌ 格式拒绝 | qq 少于 5 位 |
| `Owner.Device.Desktop.GPU` | ❌ 格式拒绝 | 必须全小写 |
| `owner..gpu` | ❌ 格式拒绝 | 空段 |
| `owner.device.desktop.gpu.` | ❌ 格式拒绝 | 尾随点 |
| `owner.device.desktop` | ❌ 格式拒绝 | 只有三段但第三段被当 attr，缺 attr 位 |
| `owner.thing.stuff` | ❌ 业务拒绝 | 无匹配模板（未注册维度） |
| `event.stm32.*` | ❌ 业务拒绝 | event 类型不允许 factKey |

**关键区分**：
```
格式拒绝 = 正则不通过 → 直接丢弃 factKey
业务拒绝 = 格式通过但模板/枚举/条件不满足 → 
  未注册模板 → 进入待确认（reviewStatus=pending）
  已注册但 qualifier 缺失/非法 → 丢弃 factKey，降级写入，不 supersede
```

---

# 三、事实更新流程（核心修订）

## 3.1 V2.1 的错误（图示）

```
V2.1：
  新记忆 → 精确去重 → 【语义候选 cosSim>0.85】→ factKey 验证 → 闸门2 factKey 比对
  
  ❌ 致命问题：如果旧记录与新记录相似度只有 0.4（表述差异大），
     旧记录根本进不了候选集 → 同 factKey 的旧版本永远漏掉 → 永远无法 supersede
```

**实例**：
```
旧：{ factKey:'owner.device.desktop.gpu', content:'台式机显卡是 [GPU]' }

新：{ factKey:'owner.device.desktop.gpu', content:'那台机器换了张 5070 Ti 上去' }

cosSim(两条) ≈ 0.45 < 0.85
→ V2.1：新记录进不了"语义候选"，直接新增，旧记录仍 active
→ 结果：两条 active 同 factKey，检索时都返回，矛盾注入 ❌
```

## 3.2 V2.2 的正确流程：双路径解耦

```
新记忆写入
   │
   ├── ① 精确去重
   │     条件：同 ownerId + 同 category + 同 type + 同 contentHash + status=active
   │     命中 → 更新 updatedAt，结束
   │
   ├── ② factKey 独立查找 【★不依赖相似度】
   │     前置：新记忆的 factKey 通过了 §2 的五道验证
   │     查找：全表扫 status=active AND ownerId 相同 AND visibility 相同 AND factKey 全等
   │     （factKey 全等是硬条件，与向量无关）
   │     
   │     ├── 命中 1 条 → 进入 ③a 事实更新判断
   │     ├── 命中 ≥2 条 → 进入 ③b 多版本处理
   │     └── 命中 0 条 → 进入 ④
   │
   ├── ③a 事实更新判断（单条命中）
   │     辅助证据（计算但不作为 gate）：
   │       - cosSim（仅记录，用于日志和 review 参考）
   │       - 极性检测（否定词）
   │     
   │     决策：
   │       conflictPolicy = 'supersede'
   │         ├─ 极性一致 → supersede
   │         └─ 极性相反 → 不 supersede，新记录 pending
   │       conflictPolicy = 'supersede_with_review'
   │         └─ supersede，但新记录 reviewStatus=pending
   │       conflictPolicy = 'supersede_with_review_if_negation'
   │         ├─ 极性一致 → supersede
   │         └─ 极性相反 → 不 supersede，新记录 pending
   │       conflictPolicy = 'keep_both'
   │         └─ 永不 supersede，两条都 active
   │
   ├── ③b 多版本处理（≥2 条命中，属数据异常）
   │     见 §3.4
   │
   └── ④ 语义候选（仅用于"发现潜在冲突"，不决定 supersede）
         条件：同 ownerId + 同 type + cosSim > 0.85 + status=active
         作用：
           ├─ 有候选 → 记录到日志 + 新记录 reviewStatus=pending
           │           （提示人工：有相似但无 factKey 的记忆，可能是重复或需要合并）
           └─ 无候选 → 直接新增
         
         ★ 语义候选永不触发 supersede
```

**核心原则**：
```
factKey 命中 → 才谈 supersede
相似度高   → 只提示"可能重复"，交人工
两者独立   → 相似度不能掩盖 factKey，factKey 也不依赖相似度
```

## 3.3 完整流程图

```
                    ┌──────────────┐
                    │  新记忆写入   │
                    └──────┬───────┘
                           ▼
                    ┌──────────────┐
                    │ ① 精确去重    │
                    └──────┬───────┘
                   命中 ───┴─── 未命中
                    │            │
                  更新           ▼
                  updatedAt   ┌──────────────────────┐
                   结束       │ ② factKey 是否有效？  │
                              └──────┬───────────────┘
                             ┌───────┴────────┐
                           无效              有效
                             │                │
                             ▼                ▼
                    ┌────────────────┐  ┌─────────────────────┐
                    │ ④ 语义候选     │  │ ②b factKey 全表扫   │
                    │ 仅提示，不替换  │  │ (独立于相似度)      │
                    └────────┬───────┘  └──────┬──────────────┘
                      有候选 │ 无                    │
                       ▼     ▼              0条  1条    ≥2条
                 pending   新增              │    │      │
                  +新增                      ▼    ▼      ▼
                                           新增  ③a    ③b
                                              (更新) (异常)
                                                │
                                                ▼
                                        ┌──────────────────┐
                                        │ 按 conflictPolicy │
                                        └────────┬─────────┘
                                     ┌───────────┼───────────┐
                                supersede      pending     keep_both
                                     │            │            │
                                     ▼            ▼            ▼
                                 旧→superseded  两条都留    两条都 active
                                 新→active      新pending   加版本链
```

## 3.4 多个 active 同 factKey 的处理（数据异常）

```
触发：②b 查到 ≥2 条 status=active 且 factKey 相同

这是数据异常状态（正常不应出现：supersede 保证只有一个 active）
可能来源：并发写入、迁移错误、手工编辑

处理（不自动修复，避免二次破坏）：
  ① 全部标记 reviewStatus='pending'
  ② 不 supersede，新记录也 pending
  ③ 选出 updatedAt 最新的一条作为"临时当前"
     - 检索时优先返回它
     - 但结果里标注 "conflict: 存在 N 个活跃版本"
  ④ 写入 state/memory-conflicts.jsonl 报警
  ⑤ 控制台高亮显示，等人工合并

人工合并方式（P4 提供 qq_memory_manage op=merge）：
  选定保留哪条 → 其余标 superseded，supersededBy=保留条
```

## 3.5 无法确认时的处理

| 情况 | 处理 | 理由 |
|---|---|---|
| factKey 相同但检测到极性相反 | 都保留，新记录 `reviewStatus=pending` | 否定词不可靠，交人工 |
| factKey 相同但 cosSim < 0.5 | 都保留，新记录 pending | 可能误用同一 key（表达方式完全不同） |
| factKey 未注册模板 | 丢弃 factKey，降级写入，`pending` | 可能是新维度，需人工加注册表 |
| factKey 格式非法 | 丢弃 factKey，普通写入，不 pending | 纯错误，不必打扰人工 |
| 命中 ≥2 条 active | 全部 pending + 报警 | 数据异常，需人工 |

## 3.6 案例 A：RTX 设备升级

```
t0  已有：{ id:mem_001, content:'台式机显卡是 [GPU]',
            factKey:'owner.device.desktop.gpu', type:'fact',
            status:'active', ownerId:'owner:[OWNER_QQ]',
            visibility:'owner_private' }

t1  写入：content='那台机器换了张 5070 Ti 上去'
          candidateFactKey='owner.device.desktop.gpu'
          type='fact'

    ① 精确去重：contentHash 不同，未命中
    ② 五道验证：
       格式 ✅（owner.device.desktop.gpu 三段+qualifier）
       主体 ✅（首段 owner = subjectId owner）
       类型 ✅（fact 在 allowedTypes）
       权限 ✅（同 ownerId + 同 visibility）
       模板 ✅（device 模板，desktop ∈ qualifiers，gpu ∈ attrs）
    ②b factKey 全表扫：
       找到 mem_001（factKey 全等，status=active）✅
       ★ 此步不看相似度
    ③a 单条命中，conflictPolicy='supersede'
       辅助证据：cosSim=0.45（低，但【不影响】决策）
       极性检测：无否定词差异 → 一致
       → supersede

    结果：
      mem_001: status='superseded', supersededBy='mem_002', supersededAt=t1
      mem_002: status='active', supersedes='mem_001', factKey 同

t2  检索 "显卡" → 只返回 mem_002（5070 Ti）✅
t3  检索 "以前用什么显卡"（includeSuperseded）→ 返回两条 + 版本链 ✅
```

**对比 V2.1**：cosSim=0.45 < 0.85，V2.1 会漏掉；V2.2 因为 factKey 独立查找，正确 supersede。

## 3.7 案例 B：作息变化（preference 类，需 review）

```
t0  已有：{ id:mem_010, content:'主人一般凌晨 2 点睡',
            factKey:'owner.schedule.sleep', type:'fact',
            status:'active' }

t1  写入：content='主人现在 11 点就睡了'
          candidateFactKey='owner.schedule.sleep'
          type='fact'

    ① 精确去重：未命中
    ② 五道验证：全过（schedule 模板，sleep ∈ aspects）
    ②b factKey 全表扫：命中 mem_010
    ③a conflictPolicy='supersede'
       极性检测：检测到"现在"（时间转折词）但无否定词 → 判为一致
       → supersede，但新记录 reviewStatus='pending'
       
       为什么 pending：作息变化可能是真的改了，也可能是某天的偶然
       交人工确认后再 approved

t2  人工在控制台确认 → mem_011.reviewStatus='approved'
t3  检索 → 只返回新的（11 点睡）✅
```

**关键设计**：`supersede_with_review` 策略——替换但新记录待审。这样既保证了"当前事实唯一"，又允许人工纠正误判。

## 3.8 案例 C：实验室电脑（修正 V2.1 描述错误）

```
t0  已有：{ factKey:'owner.device.desktop.gpu', content:'台式机 [GPU]' }

t1  写入：content='实验室电脑用的是 RTX 4090'
          candidateFactKey='owner.device.lab.gpu'    ← 正确带 qualifier

    ② 五道验证：
       格式 ✅（三段 + qualifier lab）
       业务 ✅（lab ∈ qualifiers 白名单）
    ②b factKey 全表扫：找 'owner.device.lab.gpu' → 0 条
    ④ 语义候选：与 mem_001 cosSim=0.88 > 0.85
       → 有新相似记忆，但 factKey 不同
       → 新记录 reviewStatus='pending'（提示人工核对是否漏了 qualifier 区分）
       → 新增，不改 mem_001

    结果：两条都 active，不同 factKey
```

### 修正 V2.2 中 V2.1 场景 C 的错误描述

```
V2.1 错误说法：
  "candidateFactKey:'owner.device.gpu' 
   是两段（owner.device → 缺 attribute），格式验证失败"

V2.1 的错误：
  owner.device.gpu 是【三段】（owner / device / gpu），格式完全合法
  不是"缺 attribute"，而是"缺 qualifier"

V2.2 正确描述：
  ① 格式验证：【通过】（三段匹配 /^(owner|bot)\.entity\.attr(\.qual)?$/）
  ② 业务验证：【失败】——device 模板 qualifierRequired=true，
     要求形式必须是 owner.device.<qualifier>.<attr>，
     而 owner.device.gpu 把 gpu 放在了 attr 位、qualifier 位缺失
  ③ 处理：丢弃 factKey（不 pending，因为纯属格式用错），
     降级普通写入，新记录 reviewStatus='pending'
     （因为语义上与已有 GPU 事实相似，值得人工看一眼）
```

**为什么要区分格式拒绝与业务拒绝**：
- 格式拒绝 = 模型写错了语法 → 静默降级，不必打扰人
- 业务拒绝 = 语法对但语义不合规（如漏 qualifier）→ 需人工关注，因为可能表达了真实信息但用错了 key

---

# 四、生命周期与审核状态

## 4.1 status 转换表（含 supersededAt 分离）

| 从 | 到 | 触发 | 必填时间字段 | 其他必填 | 可逆 |
|---|---|---|---|---|---|
| (新建) | `active` | 写入 | — | — | — |
| `active` | `superseded` | 事实更新 | `supersededAt` | `supersededBy`（新 id） | ✅ |
| `active` | `archived` | 主动归档 | `archivedAt` | — | ✅ |
| `active` | `deleted` | 删除 | `deletedAt` | tombstone 写入 | ✅（仅管理员强制） |
| `superseded` | `active` | 恢复旧事实 | 清空 `supersededAt` | 原当前 active 需改 superseded | ✅ |
| `superseded` | `archived` | 归档旧版本 | `archivedAt`，**保留** `supersededAt` | — | ✅ |
| `superseded` | `deleted` | 删除 | `deletedAt` | tombstone | ✅（仅管理员） |
| `archived` | `active` | 恢复归档 | 清空 `archivedAt` | — | ✅ |
| `archived` | `deleted` | 删除 | `deletedAt` | tombstone | ✅（仅管理员） |
| `deleted` | 任何 | — | — | **默认禁止** | ❌ |

### 关于 `superseded → archived` 的特殊说明

```
一条 superseded 记录被归档时：
  status 变成 archived（覆盖 superseded）
  但 supersededAt 和 supersededBy 保留（历史链不能丢）
  archivedAt 同时设置
  
→ 状态判定：status=archived 优先于 superseded
→ 检索：archived 不返回（除非 includeArchived）
→ 语义："这条既是被替代的旧版本，又被主动归档了"
```

## 4.2 deleted 的 tombstone 与恢复

```
删除流程（默认逻辑删除）：
  ① 原记录：status='deleted', deletedAt=now, content 保留
  ② tombstone：{ memoryId, ownerId, contentHash, deletedAt, deletedBy, reason, hard:false }

hard 删除（可选，需显式参数）：
  ① 原记录：content='[deleted]', vec=[]  （内容清空，保留骨架用于引用完整性）
  ② tombstone：hard:true

恢复（仅管理员）：
  前置：status=deleted
  动作：
    ① status → 'active', deletedAt → null
    ② 保留 tombstone（标记 restoredAt，不删除）
       ← 这样防复活检查仍能看到"曾被删过"
    ③ 写审计日志 state/memory-audit.jsonl：
       { action:'force_restore', memoryId, by, at, reason, originalDeletedAt }
  限制：
    - 只能通过控制台（需 consoleToken），机器人无此权限
    - 必须提供 reason
```

## 4.3 逻辑删除 vs 物理删除的边界

| 类型 | 数据 | 可恢复 | 何时用 |
|---|---|---|---|
| **逻辑删除**（默认） | 记录保留，status=deleted | ✅ 管理员 | 用户说"忘了这件事" |
| **硬删除**（hard=true） | content/vec 清空，骨架保留 | ⚠️ 仅骨架 | 隐私敏感（含 QQ 号/真名） |
| **物理删除**（当前不支持） | 从文件移除 | ❌ | 暂不提供 |

**为什么不提供物理删除**：
- 从 JSONL 中物理移除记录需要 rewrite 整个文件，与并发/崩溃恢复冲突
- 只要有 tombstone 在，"逻辑删除 + 内容清空"在隐私效果上等同物理删除
- 若将来合规要求，可提供"离线 compact 物理清理"（需停服务）

## 4.4 防已删除记忆复活：四道防线

```
防线 ①：tombstone 表
  写入前查：同 ownerId + 同 contentHash 是否在 tombstone？
  命中 → 
    ├─ 默认：拒绝写入，返回 "该内容已被删除"
    └─ 若调用方显式 force=true：写入，但 reviewStatus='pending' + 审计日志

防线 ②：WAL 重试前置检查
  每次重试前查目标 memoryId 的 status
  status='deleted' → 取消 op（stage='cancelled'）

防线 ③：备份恢复
  导入备份时逐条对比 tombstone
  已删的记录 → 跳过，不导入

防线 ④：迁移/重跑
  迁移脚本必须读 tombstone
  已删的记录 → 跳过
  脚本要幂等（按 id 去重）
```

**四道防线覆盖的场景**：

| 场景 | 拦截者 |
|---|---|
| 模型重复调 append 写同样内容 | 防线 ① |
| WAL 里有旧任务重试 | 防线 ② |
| 用 backups/ 里的快照恢复 | 防线 ③ |
| 重跑迁移脚本 | 防线 ④ |

## 4.5 检索场景 × 状态矩阵

| 检索场景 | status | reviewStatus | shareReviewStatus | visibility |
|---|---|---|---|---|
| 普通私聊 | `active` | `approved` | 不限 | 见权限矩阵 |
| 普通群聊 | `active` | `approved` | `approved` | `group_shared` |
| 历史检索（includeSuperseded） | `active` + `superseded` | `approved` | 同上 | 同上 |
| 归档检索（includeArchived） | `active` + `archived` | `approved` | 同上 | 同上 |
| 控制台管理 | 全部 | 全部 | 全部 | 全部 |
| 待确认列表 | `active` | `pending` | — | — |
| 自动注入上下文 | 排除 archived/deleted | 排除 pending/rejected | 群聊必须 approved | 按权限 |

---

# 五、A/B/C 关联方案

## 5.1 关联载体（显式 ID，非文本）

```
A 层记录：
  bRefs: [{ conversationKey, category, slotKey }]
  slotKey 定义：
    activeTopic      → 该 topic 的 text（B 层槽位用 text 作 key）
    pendingThought   → text
    memberImpression → 群友名

B 层记录：
  aMemoryId: "mem_xxx"   （可选字段，B 层每条记忆可带）

关联建立：
  同一 operationId 内产生的 A+B → 自动互相关联
  单独写入的 → 一方可为 null
```

## 5.2 关联关系图

```
┌──────────── C 层：Session 原始历史 ────────────┐
│  session-<uuid>/session.v4.jsonl.zstd        │
│  bridge 维护映射：conversationKey → sessionId │
│  单向：C 不引用 A/B                          │
└───────────────────┬──────────────────────────┘
                    │ conversationKey（间接）
┌───────────────────▼──────────────────────────┐
│  B 层：SocialV2                               │
│  activeTopics[{ text, ..., aMemoryId? }]      │
│  pendingThoughts[{ text, ..., aMemoryId? }]   │
│  memberImpressions[name]{ traits, ...,        │
│                          aMemoryId? }         │
└───────────────────┬──────────────────────────┘
                    │ 显式 ID 双向（非文本）
┌───────────────────▼──────────────────────────┐
│  A 层：向量长期记忆                            │
│  { id, bRefs:[{conversationKey,category,slotKey}] } │
└──────────────────────────────────────────────┘
```

## 5.3 B 层引用失效时的读取行为（补充细节 5）

```
情况：A 层记录的 bRefs 指向 B 层某槽位，但该槽位已被删

读取行为：
  ① B 层不会主动检查 aMemoryId 有效性（避免每次读都扫 A 层）
  ② 检索 A 层时，若 bRefs 指向的槽位不存在：
     - 不报错
     - 记录到日志（debug 级）
     - 结果里不提示（避免噪音）
  ③ 定期维护（P6）：
     qq_memory_manage op=prune_refs
     扫描所有 bRefs，删除悬空引用，写审计
  ④ 若 aMemoryId 指向不存在的 A 记录：
     - B 层读取照常（当 null 处理）
     - 不阻断

★ 原则：引用失效是"软失败"，不阻断任何正常读写。
```

## 5.4 不默认同步 / 不靠文本匹配

| 规则 | 说明 |
|---|---|
| 不自动双向同步 | 改 A 不自动改 B，改 B 不自动改 A |
| 不靠文本匹配 | 关联只用 id，绝不用 `text` 相等判断 |
| 删除不级联 | 删 A 层时，B 层保留，响应带 `warnings: ["有 N 处 B 引用"]` |
| 显式同步工具 | `qq_memory_sync(ref)` 按 id 同步，需显式调用 |

---

# 六、WAL 完整状态机

## 6.1 三层状态分离（修订 V2.1 的单状态）

```
operationStatus  操作整体状态
  created     已创建，未开始
  running     执行中
  partial     部分成功（一层 done 一层 failed）
  committed   全部成功
  failed      全部失败或重试超限
  cancelled   被删除/撤销

aStatus          A 层子状态
  none        不需要写 A
  pending     待写
  done        已写
  failed      写失败
  skipped     跳过（如 A 层不可用但允许降级）

bStatus          B 层子状态
  none        不需要写 B
  pending     待写
  done        已写
  failed      写失败
  skipped     跳过
```

**为什么要分开**：
```
V2.1 的单一 status='partial' 无法回答"到底该重试哪层"
V2.2 的 aStatus=failed + bStatus=done → 明确只重试 A 层
```

## 6.2 组合语义表

| operationStatus | aStatus | bStatus | 含义 | 下一步 |
|---|---|---|---|---|
| `committed` | done | done | 全成功 | 完成 |
| `committed` | done | none/skipped | 只写 A（如纯 A 层记忆） | 完成 |
| `committed` | none/skipped | done | 只写 B | 完成 |
| `partial` | done | failed | A 成了 B 败 | 重试 B |
| `partial` | failed | done | B 成了 A 败 | 重试 A |
| `partial` | failed | failed | 都败 | 重试两层 |
| `failed` | failed/failed | 重试超限 | 移入 failures |
| `cancelled` | 任意 | 任意 | 被撤销 | 停止所有重试 |
| `running` | 任意非终态 | 任意非终态 | 执行中/崩溃 | 进入 recovering |

## 6.3 状态转换表（完整）

| 当前 | 事件 | 转换到 | WAL 追加 |
|---|---|---|---|
| （无） | append 开始 | `created` | `{opId, kind, payload}` |
| `created` | 取得锁 | `running` | `{opId, stage:'begin'}` |
| `running` | A 层写入成功 | `running` (aStatus=done) | `{opId, aStatus:'done', aHash:<contentHash>}` |
| `running` | A 层写入失败 | `running` (aStatus=failed) | `{opId, aStatus:'failed', err}` |
| `running` | B 层写入成功 | `running` (bStatus=done) | `{opId, bStatus:'done', bHash:<contentHash>}` |
| `running` | B 层写入失败 | `running` (bStatus=failed) | `{opId, bStatus:'failed', err}` |
| `running` | 两层都 done | `committed` | `{opId, stage:'committed'}` |
| `running` | 一 done 一 failed | `partial` | `{opId, stage:'partial'}` |
| `running` | 都 failed | `failed` | `{opId, stage:'failed'}` |
| `partial` | 重试成功补全 | `committed` | `{opId, stage:'committed'}` |
| `partial` | 重试仍失败 | `partial`（retryCount+1） | `{opId, retryCount}` |
| `partial` | retryCount >= 3 | `failed` | `{opId, stage:'failed', exhausted:true}` |
| 任意非终态 | 收到删除 | `cancelled` | `{opId, stage:'cancelled', reason}` |
| 任意非终态 | 进程崩溃 | 重启后 → `recovering` | — |

## 6.4 崩溃恢复：验证内容而非仅 opId

```
V2.1 的错误：只看 "opId 是否存在"
问题：opId 已写入 WAL 但实际记录没写进去（或写了一半）→ 误判为已完成

V2.2 恢复流程：
  ① 读 WAL，找出 operationStatus ∉ {committed, cancelled, failed} 的 op
  ② 对每个 op，逐层验证：
     
     A 层验证：
       查 A 层是否已有记录满足：
         - record.opId === op.opId  （优先）
         - 或 record.id === payload.memoryId（兼容旧数据无 opId）
       若找到：
         - 比较 record.contentHash === payload.contentHash
           ├─ 一致 → aStatus = done
           └─ 不一致 → aStatus = failed（内容不该是这样，需重写）
       若未找到 → aStatus = pending
     
     B 层验证：
       查 B 层对应会话的对应槽位：
         - 是否存在且其 aMemoryId === payload.memoryId
         - 或 slotKey + text 匹配 payload.content
       若匹配 → bStatus = done
       若未匹配 → bStatus = pending
     
  ③ 关键闸门：检查 memoryId 是否已 deleted
       查 tombstone 或记录 status
       ├─ 已删 → operationStatus = cancelled，不重做  ★防复活
       └─ 未删 → 继续
     
  ④ 根据验证结果恢复：
       aStatus=done + bStatus=done → committed
       aStatus=done + bStatus=pending → 继续写 B
       aStatus=pending + bStatus=done → 继续写 A
       aStatus=failed → 重试
```

## 6.5 多个 memoryId 属于同一 operation

```
supersede 操作涉及两条记录（旧 + 新）：

payload 结构：
{
  "opId": "op_x",
  "kind": "supersede",
  "memoryIds": ["mem_old", "mem_new"],
  "changes": [
    { "id": "mem_old", "expect": { "status": "active" },
      "set": { "status": "superseded", "supersededBy": "mem_new", "supersededAt": 123 } },
    { "id": "mem_new", "expect": { "status": null },
      "set": { "status": "active", "supersedes": "mem_old", "contentHash": "..." } }
  ]
}

恢复时：
  逐个 memoryId 验证其当前 state 是否等于 changes[].set 的期望值
  ├─ 全部符合 → operationStatus=committed
  ├─ 部分符合 → 对不符合的重做（只重做那一部分）
  └─ 全部不符合 → 整体重做

★ 关键：changes[] 是幂等的（设成固定值，重复执行结果相同）
```

## 6.6 WAL compact 与幂等保证

```
compact 规则：
  ① 最近 30 天的 op：完整保留（所有 stage 行）
  ② 30 天前的 op，按 operationStatus 分：
     - committed / cancelled → 只保留摘要到 memory-op-index.json
     - failed → 保留到 failures（不 compact）
     - partial → 保留（可能仍需重试）
  ③ 一个 op 的多行合并成一行（取最终状态）

memory-op-index.json 结构：
{
  "version": 1,
  "ops": {
    "op_abc": { "contentHash": "a1b2...", "committedAt": 1793400000000 },
    "op_def": { "contentHash": "c3d4...", "committedAt": 1793400100000 }
  }
}

索引保留策略：
  最多保留 10000 条 或 最近 90 天（取较大者）
  超出 → 淘汰最旧的
  ★ 这意味着幂等窗口是 90 天。超过 90 天的重复写入不会被识别为重复。

幂等检查顺序：
  ① 查 WAL（30 天内）
  ② 查 op-index（90 天内）
  ③ 查 tombstone（永久）
  ④ 都没有 → 视为新操作
```

**副作用**：90 天后的重复写入会真的重复。但配合 contentHash 精确去重（P3），实际影响很小——因为去重不依赖 opId。

## 6.7 失败 / 取消 / 删除 / 恢复的关系

```
失败（failed）
  = 重试超限，不再自动重试
  → 记录保留在 WAL + failures
  → 人工可 run qq_memory_manage op=retry 重新激活

取消（cancelled）
  = 主动撤销（因删除导致）
  → 记录保留，但永不重试
  → 不可重新激活（因为记忆已删）

删除（deleted）
  = 记忆本身的生命周期状态，与 operationStatus 不同层
  → 一条记录可以 operationStatus=committed 但 status=deleted

恢复（force_restore）
  = 管理员把 status 从 deleted 改回 active
  → 不改变任何 WAL 记录
  → 若该记忆还有 pending 的 op，恢复后 op 可能重新执行（需检查）
```

## 6.8 五个崩溃恢复案例

### 案例 R1：A 写完、B 未写时崩溃

```
崩溃前 WAL：
  {op:'op_1', kind:'append', payload:{memoryId:'mem_1', contentHash:'h1', ...}}
  {op:'op_1', stage:'begin'}
  {op:'op_1', aStatus:'done', aHash:'h1'}

重启恢复：
  ① op_1 的 operationStatus 是 running
  ② A 层验证：找到 mem_1，contentHash='h1' === payload 期望 ✅ → aStatus=done
  ③ B 层验证：未找到对应槽位 → bStatus=pending
  ④ tombstone 检查：mem_1 未删除 ✅
  ⑤ 继续写 B → bStatus=done → committed
```

### 案例 R2：A 写完但内容不符（写了一半）

```
崩溃前 WAL：
  {op:'op_2', payload:{memoryId:'mem_2', contentHash:'h2'}}
  {op:'op_2', aStatus:'done'}     ← WAL 说 done

实际 A 层：
  mem_2 存在但 contentHash='h2_truncated' ≠ 'h2'

恢复：
  ① A 层验证：找到 mem_2 但 hash 不符
  ② 判定 aStatus=failed（内容不对，不能信 WAL 的 done）
  ③ 删除损坏的 mem_2，重写 → aStatus=done
  ★ 这就是 V2.1"只看 opId"会误判为完成的场景
```

### 案例 R3：写入过程中被删除（防复活）

```
崩溃前 WAL：
  {op:'op_3', payload:{memoryId:'mem_3'}, aStatus:'failed'}

崩溃后用户删除：
  mem_3.status = 'deleted'
  tombstone: {memoryId:'mem_3', contentHash:'h3'}

重启恢复：
  ① 扫到 op_3 是 partial（aFailed + bDone）
  ② 恢复流程走到"闸门③"：查 mem_3 status = deleted
  ③ → operationStatus = cancelled，不重试
  ★ 记忆保持删除状态，不复活
```

### 案例 R4：supersede 只完成一半

```
崩溃前：
  mem_old: status='active'（还没改）
  mem_new: status='active'（已写）
  WAL: {op:'op_4', kind:'supersede', memoryIds:['mem_old','mem_new'],
        changes:[{id:'mem_old', set:{status:'superseded',...}},
                 {id:'mem_new', set:{status:'active',...}}],
        aStatus:'done'}

恢复：
  ① 逐条验证 changes：
     mem_new: status='active' ✅ 符合期望
     mem_old: status='active' ❌ 期望 superseded
  ② 只重做 mem_old 的变更 → status='superseded'
  ③ 全部符合 → committed
  
  ★ 幂等：重做 mem_new 也无害（设成固定值）
```

### 案例 R5：WAL 自身尾行损坏

```
memory-wal.jsonl 尾行：
  {"op":"op_5","stage":"begin"}{"op":"op_5","aStat     ← 半截

恢复：
  ① 读 WAL 时逐行校验 CRC
  ② 尾行损坏 → 跳过 + 日志
  ③ op_5 只有 begin 记录 → 视为 created 状态
  ④ 进入正常恢复流程（验证 A/B 实际状态）
  ⑤ compact 时移除损坏行
  
  ★ 只丢"最后一条状态记录"，不丢整个 op
```

---

# 七、权限矩阵

## 7.1 请求者身份定义

```
requester = {
  ownerId: 'owner:<qq>' | 'group:<群号>',   ← 由 agentToken 反查会话得到
  conversationKey: 'private:<qq>' | 'group:<群号>',
  isGroup: boolean
}
```

## 7.2 完整权限矩阵

| # | 请求者 | 目标 ownerId | 目标 visibility | 允许 | 说明 |
|---|---|---|---|---|---|
| 1 | 私聊 A | `owner:A` | `owner_private` | ✅ | 自己的私有记忆 |
| 2 | 私聊 A | `owner:A` | `group_shared` | ✅ | 自己的记忆，即使标了共享，自己也能读 |
| 3 | 私聊 A | `owner:B` | 任意 | ❌ | 跨用户 |
| 4 | 私聊 A | `group:X` | 任意 | ❌ | 群记忆不进私聊 |
| 5 | 私聊 A | `owner:A` | 任意 + status=archived | ❌ | 归档不参与自动检索 |
| 6 | 群 X | `group:X` | `group_shared` + reviewStatus=approved | ✅ | 本群已审批的共享记忆 |
| 7 | 群 X | `group:X` | `owner_private` | ❌ | 群内未共享 |
| 8 | 群 X | `group:X` | `group_shared` + 未 approved | ❌ | 待审/被拒 |
| 9 | 群 X | `group:Y` | 任意 | ❌ | **不跨群** |
| 10 | 群 X | `owner:A` | 任意 | ❌ | 群不能读个人记忆 |
| 11 | 控制台（consoleToken） | 任意 | 任意 | ✅ | 管理权限，但写审计 |
| 12 | 机器人（agentToken） | 任意 | 任意 | ❌ | 机器人无管理权 |

## 7.3 关键语义澄清（审核要求）

```
① group_shared ≠ 跨群共享
   含义：这条记忆可以在【它所属的那个群】里被引用
   不是：所有群都能读
   实现：ownerId='group:X' + visibility='group_shared' 
         → 只有 conversationKey='group:X' 的请求者能读

② group_shared ≠ 自动进入私聊
   含义：私聊 owner A 不能读 ownerId='group:X' 的记忆
   即使这条是"主人说过的话"被分享到群里
   实现：矩阵第 4 行 ❌

③ 个人记忆不因群内出现相似内容而自动共享
   场景：群里有人说了句和主人私聊记忆一样的话
   行为：那条私聊记忆的 visibility 仍是 owner_private
         不会因为"群里有相似内容"就变成 group_shared
   实现：visibility 只能通过显式 share_request + approve 改变

④ 跨会话复制必须显式授权
   若真要把私聊记忆放进群：
     ① 调 qq_memory_share_request（群 token）→ pending
     ② 主人控制台 approve → visibility=group_shared
     ③ 但 ownerId 仍是 owner:A（不是 group:X）
     
     ★ 这里有个设计选择：
       方案 a（推荐）：ownerId 不变，visibility 改 group_shared
                      → 矩阵第 2 行让主人能读，矩阵第 6 行让群能读？
                        冲突！因为第 6 行要求 ownerId='group:X'
       方案 b：approve 时"复制"一条新记录 ownerId='group:X'
              → 原记录保持 owner_private
              → 这就是"跨会话复制"，需显式授权
```

### 关于第 ④ 点的方案选择 `[待决策]`

```
当前 V2.1 的实现（approveShare 只改 visibility，不改 key）导致：
  private key + group_shared + approved
  → 群检索时 key 不匹配（扫不到）
  → 这是 P0 审计发现的"分享机制自相矛盾"

V2.2 必须先定方案：
  方案 a：审批 = 复制
    原记录不动（owner:owner, owner_private）
    新增一条（owner:group:X, group_shared, approved）
    两条通过 sourceMemoryIds 关联
    优点：语义清晰，权限矩阵无冲突
    缺点：数据冗余，改一处要同步两处
    
  方案 b：审批 = 改 ownerId + visibility
    原记录的 ownerId 从 owner:A 改成 group:X
    优点：无冗余
    缺点：原主人还能读吗？矩阵第 2 行失效，主人反而看不到自己的记忆
    
  方案 c：引入"共享副本"概念（介于两者）
    原记录不变
    在 group 的检索空间里维护一个"共享索引"指向原记录
    优点：无冗余，权限清晰
    缺点：检索要双路合并（A 调研过，复杂度高）
```

**★ 这是本设计最重要的待决策项**，因为它决定了 approveShare 的语义，进而决定整个权限矩阵的实现。

---

# 八、并发与数据完整性

## 8.1 进程内串行化

```js
let writeChain = Promise.resolve();
function withWriteLock(fn) {
  const task = writeChain.then(fn, fn);
  writeChain = task.catch(() => {});
  return task;
}
```
适用范围：append / remove / clear / edit / approve / reject / archive / restore

## 8.2 跨进程锁（修订 30 秒强制接管）

### 锁文件结构

```json
{
  "pid": 12345,
  "instanceId": "inst_a1b2c3d4",
  "opId": "op_x",
  "acquiredAt": 1793400000000,
  "heartbeatAt": 1793400005000,
  "hostname": "DESKTOP-XXX"
}
```

`instanceId`：进程启动时生成的随机值，**用于区分"PID 复用"**（同一 PID 可能是重启后的新进程）。

### 获取流程（修订）

```
① 尝试 fs.openSync(lockPath, 'wx')
   ├─ 成功 → 持有锁，启动心跳定时器（每 10s 更新 heartbeatAt）
   └─ EEXIST ↓

② 读锁文件
   ├─ 读取失败 / 格式损坏 → 【不接管】返回错误
   │   "锁文件损坏，请人工检查 state/memory.lock"
   └─ 成功 ↓

③ PID 存活检查（process.kill(pid, 0)）
   ├─ ESRCH（不存在）→ 僵尸锁，可接管
   ├─ EPERM（存在但无权限）→ 视为存活 ↓
   └─ 成功（存活）↓

④ 心跳检查
   age = now - heartbeatAt
   ├─ age < 30s → 锁活跃，等待重试（最多等 10s，每 1s 试一次）
   └─ age >= 30s → 可疑 ↓

⑤ 可疑但 PID 存活 → 【不直接接管】，走 claim 流程
   ① 写 state/memory.lock.claim
      { pid, instanceId, claimAt, targetLock:{pid, instanceId} }
   ② 等待 5s
   ③ 再读锁文件，检查 heartbeatAt
      ├─ 已更新（> 原 heartbeatAt）→ 原持有者恢复，放弃 claim，回到步骤④
      └─ 仍未更新 → 判定失活，接管
   ④ 接管时：
      - 记录审计日志：谁接管了谁的锁、何时、原因
      - 删除 .claim 文件

⑥ 无法确认（任何一步异常）→ 【不接管】抛错退出
   提示："无法确认锁状态，请人工检查。若确认无其他进程，可删除 state/memory.lock"
```

### 为什么 30 秒不能作为唯一依据

```
反例：进程在做一次大 compact（重写 10000 条记录），耗时 40 秒。
     心跳定时器被 event loop 阻塞（同步 fs 操作），30 秒内没更新。
     → V2.1：判定失活，接管
     → 结果：两个进程同时写，数据损坏 ❌

V2.2 的修正：
  ① claim 二次确认（再等 5s）
  ② 即使接管，也先检查"是否有多条 WAL 在 running 状态"
     ├─ 有多条 running → 说明确实有活进程在写，放弃接管
     └─ 无 → 可以接管
  ③ 接管后先做一致性检查（验证 A/B 层状态），再继续
```

### Windows PID 复用问题

```
场景：
  进程 A（pid=1000）持锁后崩溃
  Windows 快速复用 pid=1000 给进程 B
  → process.kill(1000, 0) 成功（B 活着）
  → 误判"A 还活着"→ 永远等不到锁

V2.2 的解法：
  instanceId + heartbeat：
    - 锁文件里的 instanceId 是 A 的
    - B 如果也想拿锁，B 的 instanceId 不同
    - 但 B 不会更新 A 的锁文件（因为 B 不是锁持有者）
    - 所以 heartbeatAt 停在 A 崩溃的时刻
    - → 步骤④判定可疑 → 步骤⑤ claim → 接管 ✅
  
  即：不依赖 PID 唯一性，而依赖心跳活性。
```

## 8.3 tmp 文件修复

```js
const tmp = `${VEC_FILE}.${process.pid}.${crypto.randomBytes(6).toString('hex')}.tmp`;
try {
  fs.writeFileSync(tmp, payload, { mode: 0o600 });
  fs.fsyncSync(fs.openSync(tmp, 'r'));   // 保证落盘
  renameWithRetry(tmp, VEC_FILE);        // 复用 bridge 的重试逻辑
} catch (e) {
  try { fs.unlinkSync(tmp); } catch {}
  throw e;
}
```

## 8.4 JSONL 行格式与校验

```
每行格式：<json>\t<crc32(json)的 8 位 hex>

示例：
{"id":"mem_1","content":"主人喜欢白米饭",...}\ta1b2c3d4

为什么用 CRC32 而非 SHA256：
  CRC32 足够检测"半截/字节翻转"
  计算快 10 倍以上
  SHA256 用于 contentHash（去重），目的不同
```

### 三档读取（修订）

| 档位 | 函数 | 坏行行为 | 用在哪 |
|---|---|---|---|
| 宽松 | `loadRowsSafe` | 跳过 + 写 `memory-corrupt.log` | 普通检索 |
| 严格 | `loadRowsStrict` | 抛错，拒绝操作 | 所有写操作 |
| 修复 | `compactFile` | 隔离到 `memory-quarantine.jsonl` | 手工触发的维护 |

### 尾行粘连防护

```js
// append 前检查末尾字节
const size = fs.statSync(VEC_FILE).size;
if (size > 0) {
  const buf = Buffer.alloc(1);
  const fd = fs.openSync(VEC_FILE, 'r');
  fs.readSync(fd, buf, 0, 1, size - 1);
  fs.closeSync(fd);
  if (buf[0] !== 0x0a) fs.appendFileSync(VEC_FILE, '\n');
}
```

---

# 九、检索过滤顺序

```
① 权限过滤（最先，先于任何打分）
   ownerId 匹配 + visibility 允许 + 群聊需 shareReviewStatus=approved

② 生命周期过滤
   status='active'（includeSuperseded/includeArchived 时放宽）
   排除 deleted 永远

③ 内容审核过滤
   reviewStatus='approved'（控制台可关）

④ 分享审批过滤
   群上下文额外要求 visibility='group_shared' AND shareReviewStatus='approved'

⑤ 语义相似度
   cosSim > threshold(0.3)

⑥ 加权排序
   finalScore = score × timeWeight(createdAt) × typeWeight(type)
   ★ timeWeight 只影响排序，绝不影响"哪条是当前事实"

⑦ topK + 裁剪
```

---

# 十、异常场景预演（22 个）

| # | 场景 | V2.2 预期 | 防护机制 |
|---|---|---|---|
| S1 | 同句连续写两次 | contentHash 去重 → 更新 updatedAt | 精确去重 |
| S2 | 语义相似但事实不同 | factKey 不同 → 都保留，新记录 pending | factKey 独立查找 |
| S3 | **同 factKey 但相似度低**（RTX 换卡，表述差很多） | factKey 命中 → 正确 supersede | **★ 本版核心修复** |
| S4 | 两台不同电脑 | qualifier 不同 → 不合并 | 三段式 + 模板 |
| S5 | 极性相反 | 都不 supersede，新 pending | 否定词只触发 pending |
| S6 | 两个用户说相同内容 | ownerId 隔离 | 权限层 |
| S7 | 私有记忆被群聊检索 | 拦截 | 权限层第① |
| S8 | group_shared 记忆被其他群读 | 拦截（ownerId 不同） | 矩阵第 9 行 |
| S9 | 个人记忆因群内相似内容自动共享 | 不会（visibility 只能显式改） | 矩阵 §7.3③ |
| S10 | Ollama 不可用 | aStatus=failed，WAL 记录，队列重试 | 状态机 |
| S11 | 向量写入失败 | WAL 保护 | WAL |
| S12 | JSONL 尾行半截 | CRC 检出，写路径拒绝，读路径跳过+日志 | 行校验 |
| S13 | 多行粘连 | 写入前补 \n | 尾行检查 |
| S14 | 删除中崩溃 | 逻辑删先落盘 → 不复活 | tombstone |
| S15 | 已删记忆被 WAL 重试 | 重试前查 status=deleted → cancelled | 防线② |
| S16 | 已删记忆被备份恢复 | 对比 tombstone → 跳过 | 防线③ |
| S17 | 已删记忆被迁移重跑 | 对比 tombstone → 跳过 | 防线④ |
| S18 | 已删记忆被模型重写同样内容 | tombstone contentHash 命中 → 拒绝 | 防线① |
| S19 | 归档记录被检索 | status 过滤排除 | 过滤第② |
| S20 | 两进程并发写 | 文件锁 + claim 二次确认 | 锁机制 |
| S21 | 锁持有者进程卡住 40 秒 | claim 后检查 running ops → 放弃接管 | 锁修订 |
| S22 | supersede 只完成一半 | WAL 逐条验证 changes → 只补未完成部分 | 状态机 §6.5 |

### S3 详细（核心修复验证）

```
旧：content='台式机显卡 [GPU]', factKey='owner.device.desktop.gpu'
新：content='那台机器换 5070 Ti 了',  candidateFactKey='owner.device.desktop.gpu'

cosSim(旧, 新) = 0.45

V2.1 流程：
  语义候选要求 cosSim>0.85 → 0.45 不达标 → 无候选 → 直接新增
  → 两条 active 同 factKey ❌

V2.2 流程：
  ②b factKey 全表扫（不看相似度）→ 命中旧记录
  ③a conflictPolicy=supersede → 极性一致 → supersede ✅
  相似度 0.45 只记日志，不影响决策
```

### S21 详细（锁修订验证）

```
t0   进程 A 拿锁，开始 compact（同步写 10000 条，耗时 45s）
t10  A 的心跳定时器因事件循环阻塞，未更新 heartbeatAt
t35  进程 B 尝试拿锁：
      ① wx 失败
      ② 读锁：pid=A.pid, heartbeatAt=t0
      ③ pid 存活（A 还在跑）
      ④ age=35s > 30s → 可疑
      ⑤ 走 claim：写 .claim，等 5s
      ⑥ 再读：heartbeatAt 仍是 t0（A 被阻塞）
      ⑦ V2.1 会在这里接管 ❌
      ⑧ V2.2 额外检查：WAL 里有 running 状态的 op？
         → 有（A 的 compact op）→ 判定 A 确实在写 → 放弃接管 ✅
t45  A 完成，更新心跳，释放锁
t46  B 重试成功拿锁
```

---

# 十一、补充细节（审核要求的 8 项）

## 11.1 id 生成与碰撞处理

```
格式：mem_<base36(Date.now())>_<rand6>
例：mem_lx8k2p_a1b2c3

长度：约 20 字符
熵：36^6 ≈ 21.7 亿（同毫秒内）
碰撞概率：可忽略，但仍需处理

碰撞处理：
  写入前查该 id 是否已存在（在内存 rows 里 findIndex）
  ├─ 碰撞 → 重新生成 rand6（最多重试 5 次）
  └─ 仍碰撞 → 抛错（极端情况，说明 RNG 有问题）

为什么不只用时间戳：同毫秒多条写入会冲突
为什么不用 uuid：太长，JSONL 里占空间多
```

## 11.2 contentHash 碰撞处理

```
格式：sha256(normalize(content)) 全 64 位 hex

为什么全 64 位而非截断：
  截断到 16 位（64 bit）理论碰撞 2^32 条后出现
  全 64 位（256 bit）实际上不可能碰撞
  存储成本可忽略（64 字符 vs 16 字符，差 48 字节/条）

去重时的二次确认：
  即便 contentHash 相同，也要比对 content 全等
  ├─ content 也相同 → 确认重复，更新 updatedAt
  └─ content 不同 → hash 碰撞（理论上不可能），视为不同记录，
                     记 warn 日志
  ★ "hash 相同" 不等于 "内容相同"，必须二次确认
```

## 11.3 type 无法推断时的默认值

```
推断规则（按 category）：
  activeTopic      → event
  pendingThought   → temporary
  memberImpression → relationship
  manual           → event（最保守）
  unknown          → event

为什么默认 event：
  event 是唯一"不可 supersede、不参与事实冲突"的类型
  → 即使推断错了，也不会造成错误覆盖
  → 最安全

★ 宁可漏掉一次 supersede，也不要错误覆盖
```

## 11.4 temporary 的 expiresAt 与 TTL

```
expiresAt 默认值：
  pendingThought 类 → now + 2h（对齐 B 层默认）
  manual temporary  → now + 24h
  可由 extra.expiresAtMs 覆盖

过期行为：
  检索时：expiresAt < now → 排除（但不改 status）
  清理时：定期扫描（P4）→ status 改 archived，archivedAt=now
         理由：不用 deleted（可能还想看历史）

与 B 层的关系：
  B 层 pendingThoughts 已有 expiresAt 字段，语义一致
  A/B 同步时，expiresAt 取同一个值
```

## 11.5 B 层引用失效后的读取行为

见 §5.3，要点：
```
软失败：不报错、不阻断、不提示用户
定期 prune（P6）
```

## 11.6 WAL compact 与幂等保留策略

见 §6.6，要点：
```
WAL 保留 30 天完整
op-index 保留 90 天 / 10000 条
幂等窗口 = 90 天
超过窗口后的重复写入靠 contentHash 去重兜底（不依赖 opId）
```

## 11.7 统一 needs_review 字段命名

```
旧字段（当前代码）：needs_review（布尔）
  - 101 条全部 true，完全失效
  - 只在 listVectorMemory 过滤，检索不看

V2.2 统一为：reviewStatus（枚举）
  映射：
    needs_review=true  → reviewStatus='pending'
    needs_review=false → reviewStatus='approved'
    字段缺失           → reviewStatus='approved'

迁移时：
  读旧记录时转换成 reviewStatus
  写回时只写 reviewStatus，不再写 needs_review
  保留读取兼容（listVectorMemory 返回时可同时给两个字段名）
```

## 11.8 修正场景 C 的描述

见 §3.8 末尾。核心：
```
owner.device.gpu 是【三段】，格式合法
失败原因是【业务规则】：device 模板 qualifierRequired=true，缺 qualifier 位

区分：
  格式拒绝 → 静默丢弃 factKey
  业务拒绝 → 丢弃 + 新记录 pending（提示人工）
```

---

# 十二、P0-P6 实施顺序（重排）

## 12.1 依赖关系图

```
P0 基础设施加固（锁/tmp/CRC）
 └──► P1 数据模型扩展 + 双读 + 状态过滤
       ├──► P2 WAL + 状态机 + 幂等  ★所有后续可靠性依赖它
       │     ├──► P3 去重 + factKey 独立查找 + supersede
       │     └──► P4 逻辑删除 + tombstone + 归档 + 审计
       │           └──► P6 跨层引用 + 同步工具
       └──► P5 加权排序 + 历史检索
```

## 12.2 批次详情

### P0：基础设施加固（无依赖）

| 项 | 内容 |
|---|---|
| 改动 | 新增 `memory-lock.js`（进程内链 + 文件锁 + claim）；修 `atomicRewrite` tmp 名；加 CRC 校验函数 |
| 依赖 | 无 |
| 验收 | ① 并发 100 次写入不丢数据 ② 锁被占用时能正确等待/放弃 ③ 现有全部测试通过 |
| 回滚 | 覆盖文件（纯新增+小改，无数据格式变化） |
| 风险 | 低 |

### P1：数据模型扩展 + 双读 + 状态过滤

| 项 | 内容 |
|---|---|
| 改动 | 字段表补齐；`normalizeRow` 双读；`searchVectorMemory` 加 status/reviewStatus 过滤；`findMemoryRow` 修正数字被抹的缺陷 |
| 依赖 | P0 |
| 验收 | ① 新记录带全字段 ② 旧记录能读（补默认值）③ archived/deleted 不被检索 ④ 现有功能不回退 |
| 回滚 | 覆盖代码；数据无损（只加字段） |
| 风险 | 中（改检索核心） |

### P2：WAL + 状态机 + 幂等（★关键批次）

| 项 | 内容 |
|---|---|
| 改动 | 新增 `memory-wal.js`；拆 operationStatus/aStatus/bStatus；崩溃恢复（验证 contentHash）；op-index |
| 依赖 | P1（字段定义） |
| 验收 | ① 5 个崩溃恢复案例全过 ② 重复 opId 不重复写 ③ WAL compact 后幂等仍有效 |
| 回滚 | 删 WAL 文件（无 WAL 时降级为 P1 行为） |
| 风险 | 高（核心可靠性） |
| **前置强制** | 必须在 P3/P4 之前完成——因为 P3 引入 superseded 状态、P4 引入 deleted 状态，都是"需要可靠恢复"的状态 |

### P3：去重 + factKey + supersede

| 项 | 内容 |
|---|---|
| 改动 | 精确去重；factKey 五道验证 + 模板白名单；**factKey 独立查找**；多版本处理；`fact-key.js` + `state/fact-keys.json` |
| 依赖 | P2（supersede 需要 WAL 保护两记录的原子变更） |
| 验收 | ① S3 案例（相似度 0.45 仍正确 supersede）② S4 两设备不合并 ③ 同文不重复 |
| 回滚 | 代码回滚 + `state/fact-keys.json` 删除；superseded 记录需手工改回 active |
| 风险 | 中高 |

### P4：逻辑删除 + tombstone + 归档

| 项 | 内容 |
|---|---|
| 改动 | status=deleted + tombstone 表；archived；force_restore + 审计；四道防线；`qq_memory_manage` 工具 |
| 依赖 | P2（删除涉及状态变更，需 WAL） |
| 验收 | ① 删除后可恢复 ② 四道防线全通（S15-S18）③ 审计日志完整 |
| 回滚 | 代码回滚 + tombstone 保留（无害） |
| 风险 | 中 |

### P5：加权排序 + 历史检索

| 项 | 内容 |
|---|---|
| 改动 | timeWeight / typeWeight；includeSuperseded / includeArchived |
| 依赖 | P1（字段）+ P3（superseded 状态） |
| 验收 | ① 新旧事实同分时新的优先排序 ② 历史检索能取到 superseded ③ 时间权重不影响"哪条是当前" |
| 回滚 | 代码回滚（不影响数据） |
| 风险 | 低 |

### P6：跨层引用 + 同步工具

| 项 | 内容 |
|---|---|
| 改动 | bRefs 写入；aMemoryId；`qq_memory_sync`；prune_refs |
| 依赖 | P3（id 稳定）+ P4（删除时检查引用） |
| 验收 | ① A/B 同操作产生时正确互链 ② 删除 A 时 warnings 正确 ③ 悬空引用可 prune |
| 回滚 | 清空 bRefs/aMemoryId 字段（无害） |
| 风险 | 中 |

## 12.3 关键约束（审核要求）

```
✓ WAL 与幂等（P2）先于依赖它的 P3/P4
✓ 不允许先引入无法可靠恢复的数据状态，再在后续批次补可靠性
  → superseded（P3）和 deleted（P4）都在 P2 之后 ✅
✓ 每批次独立验收、独立回滚
✓ P0-P2 可以纯代码回滚（只加字段不改旧字段）
✓ P3 之后的回滚需要数据配合（superseded/deleted 状态）
```

---

# 十三、兼容旧工具接口方案

## 13.1 签名扩展（全部可选参数）

```js
qq_memory_append(key, token, category, content, extra?,
                 type?, candidateFactKey?, occurredAt?, subjectId?)
qq_memory_query(key, token, category?, topic?,
                includeSuperseded?, includeArchived?)
qq_memory_remove(key, token, category, content?, target?, memoryId?, hard?)
qq_memory_clear(key, token, category?, mode?)     // mode: soft|hard
qq_memory_manage(op, ...)                          // 新增：archive/restore/review/merge/prune
```

## 13.2 字段映射

| 旧 | 新 | 规则 |
|---|---|---|
| `key` | `conversationKey` | 格式不变 |
| `text` | `content` | 去 category 前缀 |
| `time` | `createdAt` | 语义不变 |
| `needs_review` | `reviewStatus` | true→pending, false→approved |
| `share_review_status` | `shareReviewStatus` | 值域不变 |
| `sourceType` | `source` | observed→model, manual→manual |
| `key` 推导 | `ownerId` | private:X → owner:X；group:X → group:X |

## 13.3 旧数据读取

```js
function normalizeRow(raw) {
  return {
    ...raw,
    id: raw.id ?? legacyId(raw),          // 确定性生成
    ownerId: raw.ownerId ?? deriveOwner(raw.key),
    conversationKey: raw.conversationKey ?? raw.key,
    content: raw.content ?? stripPrefix(raw.text),
    contentHash: raw.contentHash ?? sha256(normalize(content)),
    reviewStatus: raw.reviewStatus ?? (raw.needs_review === true ? 'pending' : 'approved'),
    ...
  };
}

// legacyId 必须确定性：同一条旧记录每次算出相同 id
function legacyId(raw) {
  return 'mem_legacy_' + sha256(`${raw.key}|${raw.text}|${raw.time}`).slice(0, 12);
}
```

---

# 十四、尚未确定问题

| # | 问题 | 候选 | 影响 | 优先级 |
|---|---|---|---|---|
| **D1** | **approveShare 语义：复制 / 改 ownerId / 共享索引** | a 复制 / b 改 ownerId / c 索引 | **决定整个权限矩阵实现** | **最高** |
| D2 | `factKey` 注册表初始内容 | 预置设备/偏好/作息 / 模型提议+人工批 | 决定事实替换覆盖面 | 高 |
| D3 | 多 active 同 factKey 的"临时当前"选取 | updatedAt 最新 / createdAt 最新 / 不选（都不返回） | 异常状态下的检索行为 | 中 |
| D4 | `occurredAt` 来源 | 模型推断 / 仅显式 | 推断有错 vs 模型负担 | 中 |
| D5 | `timeWeight` 半衰期 | 7 天 / 30 天 / 不衰减 | 旧记忆权重 | 中 |
| D6 | `typeWeight` 具体值 | fact 1.2 / event 1.0 / temporary 0.8 | 排序倾向 | 中 |
| D7 | 幂等窗口 90 天是否够 | 90 天 / 更长 | 极长期重复写入 | 低 |
| D8 | 是否提供物理删除 | 不提供 / 离线 compact | 合规需求 | 低 |
| D9 | `expiresAt` 到期后 archived 还是 deleted | archived（可查历史）/ deleted | 过期记忆的可见性 | 中 |
| D10 | 跨进程锁超时阈值 | 30s / 60s | 误接管 vs 死等 | 低 |

---

# 十五、本设计明确未覆盖

```
✗ 向量库替换（Faiss/Qdrant）——记忆量小
✗ Reranker 重排序——成本不划算
✗ BM25 混合——P7 可选
✗ 多主人架构——当前单主人
✗ global 可见性——明确不开放
✗ 知识图谱——超出范围
✗ 自动从 C 层提炼到 A 层——待决策
✗ 物理删除工具——暂不提供
```

---

*本文件为设计文档，未修改任何代码、配置、数据，未重启任何服务。等待人工审核。*
