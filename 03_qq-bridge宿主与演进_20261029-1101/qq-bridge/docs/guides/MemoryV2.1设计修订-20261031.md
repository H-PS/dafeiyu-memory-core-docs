# Memory V2.1 设计修订

- 修订时间：2026-10-31 12:15
- 前置：`记忆系统审计与MemoryV2设计-20261031.md`（审计+初版设计）
- 本文件：**只修订设计，不实施代码。** 纯只读产出，未改动任何代码/配置/数据。
- 修订依据：外部审核意见 7 项

---

# 〇、V2 → V2.1 修订对照

| # | V2 设计 | V2.1 修订 | 修订原因 |
|---|---|---|---|
| 1 | `scope` 一个字段承载会话+主体+权限 | 拆为 `ownerId` / `conversationKey` / `subjectId` / `visibility` | 三语义混在一起，多主人/多群时无法区分 |
| 2 | `factKey` 模型提供即可用 | 模型提**候选** + 程序**五道验证** | 防模型乱造 key 导致错误覆盖 |
| 3 | 相似度 > 0.85 可触发合并 | 相似度**只用于发现候选**，绝不单独决定覆盖 | 语义相似 ≠ 事实相同 |
| 4 | 否定词表判断极性 | 否定词**只触发待确认**，不自动定性 | 词表判极性不可靠 |
| 5 | `status` 含 active/superseded/archived/deleted，未分审核态 | 拆出 `reviewStatus` + `shareReviewStatus` | 三种"状态"语义完全不同 |
| 6 | WAL + 队列 + 失败记录并列 | **WAL 是唯一真相，队列/失败记录均为派生视图** | 避免多套状态冲突 |
| 7 | 未提并发 | 新增进程内串行化 + 跨进程文件锁 + 行校验 | 固定 tmp 名有丢数据风险 |
| 8 | 检索过滤顺序未严格定义 | 7 层严格顺序，**禁止时间权重代替版本判断** | 时间新 ≠ 事实更新 |

---

# 一、统一数据模型

## 1.1 完整字段表

| 字段 | 类型 | 默认值 | 必填 | 职责 | 说明 |
|---|---|---|---|---|---|
| `id` | string | 自动生成 | ✅ | 稳定唯一标识 | `mem_<base36时间>_<rand6>` |
| `ownerId` | string | — | ✅ | **记忆属于谁** | `owner:<qq>` / `group:<群号>` |
| `conversationKey` | string | — | ✅ | **在哪产生** | `private:<qq>` / `group:<群号>`，沿用现有 key 格式 |
| `subjectId` | string | `'unknown'` | ✅ | **关于谁** | `owner` / `member:<qq>` / `bot` / `topic:<slug>` |
| `visibility` | enum | `'owner_private'` | ✅ | **谁能看** | `owner_private` / `group_shared`（V2.1 不开放 `global`） |
| `category` | enum | `'unclassified'` | ✅ | 原业务分类 | `activeTopic` / `pendingThought` / `memberImpression` / `manual` |
| `type` | enum | 由 category 推断 | ✅ | 语义类型 | `fact` / `event` / `preference` / `temporary` / `relationship` |
| `status` | enum | `'active'` | ✅ | 生命周期 | `active` / `superseded` / `archived` / `deleted` |
| `reviewStatus` | enum | `'approved'` | ✅ | 内容审核 | `pending` / `approved` / `rejected` |
| `shareReviewStatus` | enum | `'not_requested'` | ✅ | 分享审批 | `not_requested` / `pending` / `approved` / `rejected` |
| `content` | string | — | ✅ | 记忆正文 | 原 `text`，**不再带 category 前缀** |
| `contentHash` | string | 自动 | ✅ | 精确去重用 | `sha256(normalizedContent)` 前 16 位 |
| `createdAt` | number | now | ✅ | 写入时间 | 原 `time` |
| `occurredAt` | number\|null | `null` | ❌ | 事件发生时间 | 可为空 |
| `updatedAt` | number | now | ✅ | 最后修改 | |
| `archivedAt` | number\|null | `null` | ❌ | 归档时间 | |
| `deletedAt` | number\|null | `null` | ❌ | 删除时间 | |
| `factKey` | string\|null | `null` | ❌ | 事实键 | `subject.entity.attribute[.qualifier]`，须过验证 |
| `supersedes` | string\|null | `null` | ❌ | 本记忆替代了谁 | 指向旧记忆 id |
| `supersededBy` | string\|null | `null` | ❌ | 本记忆被谁替代 | 指向新记忆 id |
| `source` | enum | `'model'` | ✅ | 来源 | `model` / `manual` / `migration` / `import` |
| `sourceMemoryIds` | string[] | `[]` | ❌ | 关联记忆 | 预留，V2.1 不实现逻辑 |
| `bRefs` | object[] | `[]` | ❌ | B 层引用 | `[{conversationKey, category, slotId}]` |
| `vec` | number[] | — | ✅ | 向量 | 1024 维 |
| `vecModel` | string | `'bge-m3'` | ✅ | 向量模型 | 换模型时可识别需重算 |
| `schemaVersion` | number | `2` | ✅ | 结构版本 | 迁移用 |

## 1.2 四个"身份类"字段的关系（核心修订）

V2 把 `scope` 一个字段塞了三种语义，V2.1 拆开：

```
ownerId          记忆属于谁（决策权归属）
   ├─ private 会话 → owner:<该 QQ>       （记忆归这个人）
   └─ group 会话   → group:<群号>        （记忆归这个群）

conversationKey  在哪产生的（上下文归属）
   ├─ private:<qq>
   └─ group:<群号>
   ※ 与 ownerId 可不同：群里讨论"主人喜欢香菜"，ownerId=group:xxx，
     conversationKey=group:xxx，但 subjectId=owner

subjectId        关于谁（内容主体）
   ├─ owner
   ├─ member:<qq>
   ├─ bot
   └─ topic:<slug>

visibility       谁能看（访问控制）
   ├─ owner_private  → 仅 ownerId 对应的请求者可读
   └─ group_shared   → 需 shareReviewStatus=approved，群内可读
```

**为什么必须拆**：

| 场景 | V2（scope 单字段）的问题 | V2.1 |
|---|---|---|
| 群里聊主人偏好 | scope 填 `group:xxx`，但这条其实"关于主人" | ownerId=group、conversationKey=group、subjectId=owner ✅ |
| 私聊里聊群友 | scope 填 `private:owner`，但"关于群友 A" | subjectId=member:A ✅ |
| 两个群都聊同一件事 | 无法表达"属于不同群但同一主体" | ownerId 不同、subjectId 相同 ✅ |

## 1.3 约束（程序强制校验）

```
C1  ownerId 与 conversationKey 必须同源：
      conversationKey=private:X → ownerId 必须是 owner:X
      conversationKey=group:X   → ownerId 必须是 group:X
      （V2.1 不支持跨会话共享 owner，避免权限错乱）

C2  visibility=group_shared 的必要条件：
      conversationKey 是 group:*
      AND shareReviewStatus='approved'
      AND reviewStatus!='rejected'

C3  factKey 与 subjectId 必须一致：
      factKey 首段 = 'owner'  → subjectId 必须是 'owner'
      factKey 首段 = 'member' → subjectId 必须是 'member:<qq>'，且 qq 一致

C4  factKey 与 type 必须匹配：
      factKey 只允许 type ∈ {fact, preference}
      type ∈ {event, temporary, relationship} 时 factKey 必须为 null

C5  status=superseded 必须有 supersededBy 且指向存在记录
    status=deleted 必须有 deletedAt
    status=archived 必须有 archivedAt

C6  shareReviewStatus='approved' 必须有 visibility='group_shared'
    （反向：visibility='owner_private' 时 shareReviewStatus 不能是 approved）

C7  contentHash 唯一性约束只在"同 owner+category+type+status=active"内生效
```

## 1.4 不开放 global 的理由

```
global 意味着"跨所有会话共享"，但：
  ① 当前鉴权模型是"会话 token"，没有"用户身份"层
  ② 一旦开放，任何群都能读到私聊记忆，等于绕过 owner_private
  ③ 需要先有"用户级鉴权"基础设施
→ V2.1 明确不开放；字段枚举里不出现 global，程序校验拒绝
```

---

# 二、factKey 与事实更新

## 2.1 factKey 三段式结构（修订 V2 的两段式）

```
格式：<subject>.<entity>.<attribute>[.<qualifier>]

subject   : owner | member | bot | topic
entity    : device | preference | schedule | relation | ...
attribute : gpu | cpu | food | sleep | ...
qualifier : 可选，用于区分同类不同实例（desktop / lab / phone）

示例：
  owner.device.desktop.gpu     ← 主人台式机显卡
  owner.device.lab.gpu         ← 实验室电脑显卡   ← 与上面不同！
  owner.preference.food        ← 主人饮食偏好
  owner.schedule.sleep         ← 主人作息
  member.<qq>.relation         ← 与某群友的关系
  bot.persona.name             ← 机器人自己的名字
```

**这就是 RTX 案例的关键**：V2 只用了 `owner.gpu`，导致"主人电脑"和"实验室电脑"会被判为同一事实。V2.1 用 `qualifier` 区分。

## 2.2 五道验证（模型提候选，程序决定）

```
模型传 candidateFactKey（可选参数）
  ↓
① 格式验证
   /^(owner|member|bot|topic)(\.[a-z][a-z0-9]*){2,4}$/
   失败 → 丢弃 factKey，降级普通写入（不 supersede），返回 note
  ▼
② 主体一致性
   factKey 首段必须与 subjectId 映射一致（C3）
   失败 → 丢弃，降级
  ▼
③ 类型匹配
   type 必须是 fact 或 preference（C4）
   失败 → 丢弃，降级
  ▼
④ 权限一致性
   候选旧记忆与新记忆的 ownerId 必须相同
   且 visibility 必须相同
   失败 → 丢弃，降级
  ▼
⑤ 注册表白名单
   factKey 必须在 state/fact-keys.json 注册（防模型造新维度）
   未注册 → 进入"待确认"（不覆盖，标记 needs_review）
  ▼
全部通过 → 允许 supersede
```

**为什么不让模型直接定**：模型可能造出 `owner.thing.stuff` 这种无意义 key，或者把"实验室电脑"错写成 `owner.device.gpu`（漏 qualifier）。程序验证是硬闸门。

## 2.3 禁止项（审核明确要求）

```
❌ 不允许只凭余弦相似度自动覆盖
   相似度只用于"发现候选"，覆盖必须有相同 factKey
   → 两条 0.95 相似但 factKey 不同的记忆：都保留

❌ 不允许只凭否定词表决定事实极性
   否定词只用于"触发待确认"
   → 检测到极性差异 → 标记 needs_review，不自动处理

❌ 不允许相似度 + 否定词表组合决策
   即：不能写"相似度>0.85 且否定词都相同 → 覆盖"
   必须有 factKey
```

## 2.4 完整决策流程

```
新记忆写入
  ↓ 精确去重（同 owner+category+type+contentHash+status=active）
  ├─ 命中 → 更新 updatedAt，不新增，流程结束
  └─ 未命中 ↓
  ↓ 语义候选（同 owner+type，cosSim>0.85，status=active）
  ├─ 无候选 → 直接新增
  └─ 有候选 ↓
  ↓ 事实判定的三道闸门
  ┌──────────────────────────────────────────────┐
  │ 闸门1：新记忆 factKey 是否通过五道验证？        │
  │   否 → 保留原记录，新增，标 needs_review       │
  │   是 ↓                                        │
  │ 闸门2：候选中有 factKey 完全相同的？            │
  │   无 → 保留原记录，新增（不同事实）            │
  │   有 ↓                                        │
  │ 闸门3：候选的 status 是否 active？              │
  │   否（已 superseded）→ 只链接版本链，不重复标记 │
  │   是 → 执行 supersede                          │
  └──────────────────────────────────────────────┘
  ↓ supersede 动作
  旧记录：status='superseded', supersededBy=新id, archivedAt=now
  新记录：supersedes=旧id, status='active'
  写入同一条 WAL 事务（两行同 opId）
```

## 2.5 RTX 案例完整推演（含设备区分）

### 场景 A：同一台电脑升级（应该 supersede）

```
已有：{ content:'主人台式机用 [GPU]', factKey:'owner.device.desktop.gpu',
        type:'fact', subjectId:'owner', status:'active' }

新写：{ content:'主人台式机升级到 [GPU]', 
        candidateFactKey:'owner.device.desktop.gpu', type:'fact' }

① 精确去重：内容不同，未命中
② 语义候选：cosSim 0.92 > 0.85，候选=[旧]
③ 闸门1：factKey 五道验证通过
④ 闸门2：旧记录 factKey 完全相同 ✅
⑤ 闸门3：旧 status=active ✅
→ supersede：旧标 superseded，新 active

检索"主人显卡"：
  默认只返回 5070 Ti ✅
检索"主人以前用什么显卡"：
  includeSuperseded=true → 返回两条，标注替代关系 ✅
```

### 场景 B：两台不同电脑（不应 supersede）

```
已有：{ content:'主人台式机用 [GPU]', factKey:'owner.device.desktop.gpu' }

新写：{ content:'实验室电脑用 RTX 4090', 
        candidateFactKey:'owner.device.lab.gpu', type:'fact' }

① 精确去重：未命中
② 语义候选：cosSim 0.88 > 0.85，候选=[旧]
③ 闸门1：factKey 验证通过（lab 是合法 qualifier）
④ 闸门2：factKey 不同（desktop vs lab）❌
→ 不 supersede，两条都保留，各自 active

检索"显卡配置"：返回两条，模型能区分台式机/实验室 ✅
```

### 场景 C：模型漏了 qualifier（危险，必须拦）

```
新写：{ content:'实验室电脑用 RTX 4090', candidateFactKey:'owner.device.gpu' }

factKey 是两段（owner.device → 缺 attribute），格式验证失败
→ 丢弃 factKey，降级普通写入
→ 标记 needs_review（因为检测到"显卡"语义与现有事实相似）
→ 控制台可见待确认，不自动覆盖  ✅
```

### 场景 D：极性相反（必须拦）

```
已有：{ content:'主人喜欢香菜', factKey:'owner.preference.food' }
新写：{ content:'主人不喜欢香菜', candidateFactKey:'owner.preference.food' }

① 精确去重：未命中（文本不同）
② 语义候选：cosSim 0.94
③ 闸门1：factKey 验证通过
④ 闸门2：factKey 相同
⑤ **闸门3（新增）：极性检测**
   检测到否定词差异（"不"）
   → 不自动 supersede
   → 两条都保留，新记录 reviewStatus='pending'
   → 控制台等人工确认
```

**注意**：这里否定词只用于"触发待确认"，**不用于判断谁对谁错**。人工看到两条后决定。

---

# 三、生命周期与审核状态

## 3.1 三套状态互不干涉

```
status（生命周期，谁在管：写入/删除/归档逻辑）
  active      正常，可被检索（受其他过滤限制）
  superseded  被新事实替代，默认不检索，历史检索可见
  archived    主动归档，默认不检索，可恢复
  deleted     逻辑删除，永不检索，不可恢复（除非人工）

reviewStatus（内容审核，谁在管：写入时的可信度判断）
  pending     待人工确认（factKey 冲突/极性可疑/未注册 key）
  approved    正常（默认）
  rejected    内容有问题，不检索

shareReviewStatus（分享审批，谁在管：分享流程）
  not_requested  未申请
  pending        已申请待批
  approved       已批准（配合 visibility=group_shared）
  rejected       已拒绝
```

**为什么要拆三套**：
- V2 把 `needs_review` 和 `share_review_status` 混在 status 里想当然
- 实际它们是**三条独立的生命周期**：
  - 一条记忆可以 active 且 reviewStatus=pending（内容待确认但仍是当前事实）
  - 可以 active 且 shareReviewStatus=approved（内容没问题、已批准共享）
  - 可以 superseded 且 shareReviewStatus=approved（旧事实，但曾经批准过共享）

## 3.2 status 状态转换表

| 从 | 到 | 触发 | 附加条件 | 可逆 |
|---|---|---|---|---|
| (新建) | `active` | 写入 | 默认 | — |
| `active` | `superseded` | 新事实 supersede | factKey 验证通过 + 闸门全过 | ✅（人工） |
| `active` | `archived` | 归档操作 | — | ✅ |
| `active` | `deleted` | 删除操作 | — | ✅（逻辑删可恢复） |
| `superseded` | `active` | 恢复旧事实 | 需同时把当前 active 的改 superseded | ✅ |
| `superseded` | `archived` | 归档 | — | ✅ |
| `superseded` | `deleted` | 删除 | — | ✅ |
| `archived` | `active` | 恢复 | — | ✅ |
| `archived` | `deleted` | 删除 | — | ✅ |
| `deleted` | 任何 | — | **禁止**（需人工介入改文件） | ❌ |

**禁止从 deleted 恢复的理由**：防"删除后复活"——这是审核明确要求的。若真需恢复，走人工操作（控制台"强制恢复"，记录审计日志）。

## 3.3 reviewStatus 转换

| 从 | 到 | 触发 |
|---|---|---|
| `approved` | `pending` | 写入时检测到 factKey 未注册 / 极性可疑 / factKey 冲突但无法判定 |
| `pending` | `approved` | 控制台人工确认"这条对" |
| `pending` | `rejected` | 控制台人工确认"这条错" |
| `rejected` | `pending` | 重新提交审核 |

## 3.4 shareReviewStatus 转换

| 从 | 到 | 触发 |
|---|---|---|
| `not_requested` | `pending` | `qq_memory_share_request`（仅 group） |
| `pending` | `approved` | 控制台批准 → 同时 `visibility='group_shared'` |
| `pending` | `rejected` | 控制台拒绝 → `visibility` 保持/回退 `owner_private` |
| `approved` | `rejected` | 撤回共享 → `visibility` 回退 |

## 3.5 检索场景 × 状态矩阵（核心）

| 检索场景 | status | reviewStatus | shareReviewStatus | visibility |
|---|---|---|---|---|
| **普通私聊检索** | `active` | `approved` | 不限 | `owner_private` 或 `group_shared` |
| **普通群聊检索** | `active` | `approved` | **`approved`** | **`group_shared`** |
| **历史检索**（显式"以前/之前"） | `active` + `superseded` | `approved` | 按场景 | 按场景 |
| **控制台管理列表** | 全部 | 全部 | 全部 | 全部 |
| **待确认列表** | `active` | `pending` | — | — |
| **任何自动注入** | 排除 `archived`/`deleted` | 排除 `pending`/`rejected` | 群聊必须 approved | — |

**关键规则**：
```
archived 和 deleted 永不参与任何自动检索
  → 只有控制台显式查时才可见
superseded 只在"历史检索"时返回
  → 判定依据：查询文本含时间语义词（以前/之前/原来/曾经/旧），
     或调用方显式传 includeSuperseded=true
```

---

# 四、A/B/C 跨层关系

## 4.1 category 与 type 的职责分离（审核重点）

```
category（业务分类，与 B 层槽位一一对应）
  activeTopic      → B 层 st.activeTopics[]
  pendingThought   → B 层 st.pendingThoughts[]
  memberImpression → B 层 st.memberImpressions[name]
  manual           → 无 B 层对应（纯 A 层）
  作用：决定"这条在 B 层放哪个槽"

type（语义类型，影响检索与事实更新）
  fact        稳定事实（可带 factKey，可被 supersede）
  event       历史事件（不可 supersede，可累积）
  preference  偏好（可带 factKey，可 supersede）
  temporary   短期状态（有 TTL）
  relationship 关系（可带 factKey）
  作用：决定"这条怎么被检索、能不能被覆盖、要不要过期"
```

**两者独立映射（不是一一对应）**：

| category | 常见 type | 也可以是 |
|---|---|---|
| activeTopic | event | fact（讨论出的事实）/ temporary |
| pendingThought | temporary | — |
| memberImpression | relationship | preference（"他喜欢猫"） |
| manual | fact / preference | event |

**保留两者的理由**：
- 只用 category → 无法表达"这个 activeTopic 其实是个稳定事实" → 事实更新失效
- 只用 type → 无法与 B 层槽位对齐 → B 层同步失效

## 4.2 三层关联图

```
┌──────────────── C 层：Session 原始历史 ────────────────┐
│  session-<uuid>/session.v4.jsonl.zstd                │
│  对应关系：bridge 维护 conversationKey → sessionId     │
│            （存在 social-v2.json 的会话状态里或映射表） │
└────────────────────────┬─────────────────────────────┘
                         │ 单向引用（C → 不引用 A/B）
                         │ 通过 conversationKey 间接关联
┌────────────────────────▼─────────────────────────────┐
│  B 层：SocialV2 短期状态                              │
│  st.activeTopics[{ text, lastMentionAt, ...,          │
│                    aMemoryId?  }]     ← 可选反向引用   │
│  st.memberImpressions[name]{ traits, ...,             │
│                               aMemoryId? }            │
└────────────────────────┬─────────────────────────────┘
                         │ 显式 ID 双向引用（不靠文本）
┌────────────────────────▼─────────────────────────────┐
│  A 层：向量长期记忆                                   │
│  { id, conversationKey, category, type,               │
│    bRefs: [{conversationKey, category, slotId}] }     │
└───────────────────────────────────────────────────────┘
```

## 4.3 关联方式（审核要求：不默认双向同步、不靠文本匹配）

```
关联载体：显式 ID
  A 层 bRefs[] 指向 B 层槽位
  B 层每条记忆可带 aMemoryId 指回 A 层

建立时机：写入时，A/B 同一次操作产生 → 互相关联（同 operationId）
         不同操作产生的记录 → 不自动关联

单向 vs 双向：
  A → B：通过 bRefs（A 知道自己在 B 层哪几个槽有对应）
  B → A：通过 aMemoryId（B 知道自己是哪条 A 记忆的投影）
  两者都可缺失（历史数据、单独写入的都可能是 null）

禁止：
  ❌ 不靠 text 匹配建立关联（文本会变，改一处另一处失联）
  ❌ 不默认双向同步（改 A 层不自动改 B 层，反之亦然）
  ❌ 删除不级联（删 A 层不删 B 层，但要在响应的 warnings 里提示
     "有 N 处 B 层引用仍存在"）

提供显式同步工具（V2.1 只设计不实现）：
  qq_memory_sync(ref)  → 按 id 把 A 层 content 同步到 B 层槽位
  或反向。必须显式调用，不是自动。
```

## 4.4 删除时的关联检查

```
删除 A 层记忆 id=mem_x：
  ① 查 bRefs
  ② 若 bRefs 非空：
     - 默认：只删 A，B 层保留，响应带 warnings
     - 可选参数 cascadeB=true：同时删 B 层对应槽位（需显式）
  ③ 查 supersedes/supersededBy：
     - 若有 supersededBy → 该新记忆的 supersedes 需清空（否则悬空引用）
     - 若有 supersedes → 该旧记忆的 supersededBy 需清空
  ④ 写入时同 opId 事务，保证引用一致性
```

---

# 五、可靠性状态机（统一，避免冲突）

## 5.1 核心原则（修订 V2 的三套并列）

```
WAL 是唯一真相来源（single source of truth）
  state/memory-wal.jsonl  ← 追加写，只增不改

派生视图（可随时从 WAL 重建，不单独维护状态）：
  待重试队列 = WAL 中 status ∈ {a_failed, b_failed, retrying} 的 op
  失败记录   = WAL 中 status = failed 且 retryCount >= max
  完成标记   = WAL 中 status = committed

这样避免"WAL 说完成、队列说待重试、失败记录又说失败"的三方冲突。
```

## 5.2 operationId 状态机（完整）

```
                    ┌──────────────────────────────────────┐
                    │                                      │
  [created] ──► [a_pending] ──► [a_done] ──► [b_pending] ──► [b_done] ──► [committed]
                    │              │            │              │
                    │ 失败          │ 不适用      │ 失败          │ 失败
                    ▼              │            ▼              ▼
               [a_failed] ◄────────┘       [b_failed] ◄────────┘
                    │                          │
                    │ 重试                      │ 重试
                    ▼                          ▼
               [retrying] ──► 成功 ──► [a_done] / [b_done]
                    │
                    │ 超次数
                    ▼
                [failed]  ←── 移入 failures（人工处理）

任何阶段收到「删除」：
  [cancelled]  ← 队列任务跳过，不再重试
                    │
                    └─► 若已 committed，删除走独立的 delete 操作

任何阶段崩溃：
  重启时扫 WAL：
    status 不是 committed/cancelled/failed 的 op → 进入 [recovering]
    recovering 判定：
      ① 检查 A 层是否已有该 op 的记录（按 opId 查）
         ├─ 有 → 标记 a_done
         └─ 无 → 回到 a_pending
      ② 检查 B 层是否已有该 op 的标记
         ├─ 有 → 标记 b_done
         └─ 无 → 回到 b_pending
      ③ 检查该 memoryId 是否已被删除（status=deleted）
         ├─ 是 → 直接 [cancelled]（防复活）★关键
         └─ 否 → 继续
```

## 5.3 WAL 记录格式

```jsonl
{"opId":"op_abc123","ts":1793400000000,"stage":"created","kind":"append","payload":{"ownerId":"owner:[OWNER_QQ]","conversationKey":"private:[OWNER_QQ]","content":"...","category":"activeTopic","type":"event","memoryId":"mem_xyz"}}
{"opId":"op_abc123","ts":1793400000030,"stage":"a_done"}
{"opId":"op_abc123","ts":1793400000050,"stage":"b_done"}
{"opId":"op_abc123","ts":1793400000060,"stage":"committed"}
```

**只增不改**：每个状态转换追加一行。重建时按 opId 归并，取最后 stage。

## 5.4 幂等保证

```
写入幂等：
  写 A 层前先查：WAL 中该 opId 是否已 a_done？
    ├─ 是 → 跳过 A 层写入（避免重复）
    └─ 否 → 写入
  同时在记录里存 opId，重建时可检测"同 opId 已存在记录"→ 跳过

删除幂等：
  删除前查 status：
    ├─ 已是 deleted → 直接返回成功（幂等）
    └─ 否则 → 标记 deleted

重试幂等：
  重试任务执行前必须查：
    ① memoryId 是否 status=deleted → 是则取消任务
    ② opId 是否已 committed → 是则丢弃任务
```

## 5.5 三者的具体实现关系

| 文件 | 作用 | 写入时机 | 大小管理 |
|---|---|---|---|
| `state/memory-wal.jsonl` | 唯一真相 | 每次状态转换 | 定期 compact（只保留未完成 op + 最近 7 天已完成） |
| `state/memory-queue.json` | **不单独存**，启动时从 WAL 重建 | — | — |
| `state/memory-failures.jsonl` | 超次数失败的 op 归档 | 重试超限时 | 人工清理 |
| `state/memory-vectors.jsonl` | 记忆本体 | 写入/编辑/删除 | 定期 compact |

**为什么队列不单独存**：单独存就会出现"WAL 和队列不一致"的问题。从 WAL 派生是幂等的、可重放的。

---

# 六、并发与数据完整性

## 6.1 进程内串行化（必须有，当前完全没有）

```js
// 设计：单一 Promise 链，所有写操作排队
let writeChain = Promise.resolve();

function withWriteLock(fn) {
  const task = writeChain.then(fn, fn);   // 前一个失败也继续
  writeChain = task.catch(() => {});      // 吞掉错误避免链断
  return task;
}

// 所有写路径包一层：
appendVectorMemory → withWriteLock(async () => { ... })
removeAllVectorMemory → withWriteLock(...)
editVectorMemory → withWriteLock(...)
approveShare / rejectShare → withWriteLock(...)
```

**为什么必须**：当前 `atomicRewrite` 固定 `.tmp` 名，两个并发写必然冲突（缺陷 N2）。

**副作用**：写操作串行 → 高并发时延迟增加。但记忆写入频率低（一条消息最多一次 append），可接受。

## 6.2 跨进程文件锁

```
锁文件：state/memory.lock
获取：fs.openSync(lockPath, 'wx')  ← 原子创建，已存在则失败
内容：{ pid, opId, acquiredAt }
超时：30 秒未释放 → 视为僵尸锁，强制接管（记录日志）
释放：unlink（在 finally 里）
```

**为什么需要**：bridge 进程 + 可能的手工脚本（如迁移脚本）同时写。虽然罕见，但代价很低。

**副作用**：僵尸锁可能导致短暂阻塞。超时接管机制缓解。

## 6.3 tmp 文件名修复

```js
// 当前（缺陷）: memory-vector.js:377
const tmp = VEC_FILE + '.tmp';

// V2.1（对齐 bridge.js:184 的正确做法）
const tmp = `${VEC_FILE}.${process.pid}.${crypto.randomBytes(6).toString('hex')}.tmp`;
try {
  fs.writeFileSync(tmp, ...);
  renameWithRetry(tmp, VEC_FILE);   // 复用 bridge 的 EPERM/EBUSY 重试
} catch (e) {
  try { fs.unlinkSync(tmp); } catch {}
  throw e;
}
```

## 6.4 JSONL 尾行损坏：检测、校验、恢复

### 损坏的三种形态

```
① 尾行半截（最可能）：进程写入时被 kill
   {"id":"mem_1","content":"你好"}
   {"id":"mem_2","cont        ← 半截

② 中间坏行：磁盘位翻转 / 手工编辑错误
   {"id":"mem_1",...}
   {{{坏
   {"id":"mem_3",...}

③ 多行粘连：上一次写没换行，下次 append 直接接上
   {"id":"mem_1",...}{"id":"mem_2",...}
```

### 校验方案

```js
// 每行末尾加校验后缀（不破坏 JSONL 兼容性：仍是单行文本）
// 格式：<json>\t<crc32(json)的8位hex>
{"id":"mem_1",...}\ta1b2c3d4
```

**为什么用 CRC32 而不是 SHA256**：CRC32 足够检测"半截/损坏"，且计算快 10 倍以上。SHA256 用于内容去重（contentHash），目的不同。

### 读取策略（三档）

| 场景 | 策略 | 行为 |
|---|---|---|
| **普通检索** | `loadRowsSafe` | 坏行跳过，**记录到 `memory-corrupt.log`**，返回其余（不能因为一行坏就查不了） |
| **写操作** | `loadRowsStrict` | 坏行**抛错**，拒绝写入（避免基于不完整数据改写，扩大损坏） |
| **修复模式** | `compactFile` | 读出所有好行 → 写新文件（唯一 tmp）→ rename；坏行存入 `memory-quarantine.jsonl` |

### 尾行粘连防护

```js
// append 前检查文件末尾
const stat = fs.statSync(VEC_FILE);
if (stat.size > 0) {
  const fd = fs.openSync(VEC_FILE, 'r');
  const buf = Buffer.alloc(1);
  fs.readSync(fd, buf, 0, 1, stat.size - 1);
  fs.closeSync(fd);
  if (buf[0] !== 0x0a) {              // 不是 \n
    fs.appendFileSync(VEC_FILE, '\n'); // 先补换行
    log('检测到尾行无换行，已补');
  }
}
```

### 落盘策略

```
append（单条新增）：
  fs.appendFileSync（单系统调用，原子性较好）
  + 可选 fs.fsyncSync（保证落盘，代价小因为频率低）

atomicRewrite（批量改）：
  写唯一 tmp → fsync(tmp) → rename → （可选）fsync(dir)

WAL 写入：
  同样 append + 可选 fsync
```

---

# 七、检索过滤顺序（严格）

## 7.1 七层顺序（不可调换）

```
输入：query, requester{ownerId, conversationKey, token}, options

① 权限过滤（最高优先，先于任何打分）
   ownerId === requester.ownerId
   AND visibility 允许（owner_private 需同 owner；group_shared 需 approved）
   失败 → 该记录直接不可见，连分数都不算

② 生命周期过滤
   status === 'active'
   （includeSuperseded=true 时额外允许 'superseded'）
   排除 archived / deleted

③ 内容审核过滤
   reviewStatus === 'approved'
   排除 pending / rejected
   （控制台管理视图可关闭此层）

④ 分享审批过滤（仅群聊场景）
   若 requester 是 group 上下文：
     visibility === 'group_shared' AND shareReviewStatus === 'approved'

⑤ 语义相似度打分
   cosSim(queryVec, record.vec)
   过滤 score <= threshold(0.3)

⑥ 加权排序
   finalScore = score
              × timeWeight(createdAt)      // 轻微衰减，不是主导
              × typeWeight(type)           // fact/relationship 略高，temporary 略低
   注意：★ 时间权重只影响排序，绝不用来判断"哪条是当前事实"
         当前事实由 status 决定，不由时间决定

⑦ topK 截断 + 内容裁剪
```

## 7.2 明确禁止（审核要求）

```
❌ 禁止以时间权重代替事实版本判断
   错误做法：两条 GPU 记忆，因为新的 createdAt 更近 → 排前面 → 视为当前
   正确做法：靠 status=superseded 明确标记，与时间无关

❌ 禁止把权限过滤放在打分之后
   错误做法：先算所有分数，再过滤无权限的
   风险：时间/资源侧信道（分数计算时已触碰到不该读的数据）
   正确：先过滤，再只对可见记录打分

❌ 禁止把 archived/deleted 靠"分数低"隐式排除
   必须显式 status 过滤
```

## 7.3 各层为什么要独立

| 层 | 解决什么 | 如果合并会怎样 |
|---|---|---|
| 权限 | 越权读取 | 群友能读到主人私聊记忆 |
| 生命周期 | 过时/已删记忆注入 | 模型引用已删内容 |
| 内容审核 | 可疑内容进入上下文 | 错误事实被当真 |
| 分享审批 | 未经批准的私有内容进群 | 隐私泄漏 |
| 相似度 | 相关性 | 返回无关内容 |
| 加权 | 结果质量 | 重要记忆被淹没 |

**每层都是独立的失败模式，合并会失去精确定位能力。**

---

# 八、异常场景预演（18 个）

| # | 场景 | 触发 | 当前表现 | V2.1 预期 | 主要防护 |
|---|---|---|---|---|---|
| S1 | 同一句连续写两次 | 模型重复调用 | A 层 2 条重复 | contentHash 精确去重 → 更新 updatedAt | 第一层去重 |
| S2 | 语义相似但事实不同 | 两条不同设备 | 都返回，混淆 | 语义候选 → factKey 不同 → 都保留 | 闸门2 |
| S3 | 新事实覆盖旧事实（同设备） | GPU 升级 | 都返回 | factKey 相同 → 旧 superseded | 五道验证+三闸门 |
| S4 | 两台不同电脑 | 台式机/实验室 | 可能误合并 | qualifier 不同 → 不合并 | 三段式 factKey |
| S5 | 极性相反 | 喜欢/不喜欢香菜 | 都返回，模型混乱 | 检测否定差异 → 都保留 + pending | 只触发待确认 |
| S6 | 两个用户说相同内容 | 不同 owner | 各自存 | ownerId 隔离 | 约束 C1 |
| S7 | 私有记忆被群聊检索 | 群 token 查私聊 | 被拦（现有） | 同 + 权限层最先 | 过滤第①层 |
| S8 | Ollama 不可用 | 停服务 | aSynced:false，无重试 | 同 + WAL 记录 → 队列重试 | 状态机 |
| S9 | 向量写入失败 | 磁盘满/只读 | 抛错 500 | WAL 保护，可从 recovering 重放 | WAL |
| S10 | JSONL 尾行半截 | 写入时 kill | loadRows 静默跳过 | CRC 校验 → 隔离 + 日志；写路径拒绝 | 三档读取 |
| S11 | 多行粘连 | append 未换行 | 两条记录变一条坏 | 写入前补 \n | 尾行检查 |
| S12 | 删除中崩溃 | 逻辑删落盘后 kill | 留 .tmp 垃圾 | 唯一 tmp + 逻辑删先落盘 → 不复活 | tmp 修复 |
| S13 | 已删记忆被重试重写 | 队列里有旧任务 | 无队列（不适用） | 重试前查 status=deleted → 取消 | **★关键闸门** |
| S14 | 归档记录被检索 | — | 无归档概念 | 过滤第②层排除 | 状态过滤 |
| S15 | 两操作并发改同一条 | 双请求 | 固定 tmp → 丢数据 | withWriteLock 串行化 | 写入锁 |
| S16 | B 层状态过期 | 24h/expiresAt | 自动过滤 | 同（保留现有机制） | 已有 |
| S17 | Session 历史缺失 | 删 session 文件 | A 层仍在 | 同（三层独立） | 架构 |
| S18 | 记忆被 superseded 后想恢复 | 用户改主意 | 无此概念 | status 改回 active，当前 active 的改 superseded | 状态机 |

### S3 详细推演（核心场景）

```
t0: 已有 mem_001 { factKey:'owner.device.desktop.gpu', content:'[GPU]', status:'active' }

t1: 模型写「主人台式机升级到 [GPU]」，传 candidateFactKey='owner.device.desktop.gpu'
    opId=op_x

    WAL: {opId:op_x, stage:'created', memoryId:mem_002}
    ① 精确去重 → 未命中
    ② 语义候选 → [mem_001] cosSim 0.92
    ③ 五道验证 → 全过
    ④ 闸门2 → factKey 相同
    ⑤ 闸门3 → mem_001.status=active ✅
    ⑥ 极性检测 → 无否定词差异 ✅
    ⑦ 执行 supersede（同 opId 两行 WAL）：
         mem_001 → status:superseded, supersededBy:mem_002, archivedAt:t1
         mem_002 → status:active, supersedes:mem_001, factKey:同上
    WAL: {opId:op_x, stage:'a_done'}
    WAL: {opId:op_x, stage:'b_done'}
    WAL: {opId:op_x, stage:'committed'}

t2: 检索"主人显卡" → 只返回 mem_002 ✅
t3: 检索"主人以前显卡" → includeSuperseded → 返回两条，带版本链 ✅
```

### S13 详细推演（防复活，最关键）

```
t0: op_y 写入失败 → WAL: {opId:op_y, stage:'a_failed', retryCount:1}
t1: 用户删除该记忆（或删除整 key）
    → mem_002 标记 status:deleted
t2: 定时器扫 WAL，发现 op_y 待重试
    → 重试前检查：
       ① 查 memoryId=mem_002 的 status
       ② status === 'deleted' → **取消 op_y，不写 A 层**
       WAL: {opId:op_y, stage:'cancelled'}
t3: 结果：记忆确实被删除，没有被重试任务复活 ✅
```

**没有这道闸门会怎样**：重试任务把已删记忆重新写回 A 层 → 用户删了但检索还能看到 → 隐私/信任崩塌。

---

# 九、旧工具接口兼容方案

## 9.1 MCP 工具签名扩展（不破坏现有调用）

```js
// 现有
qq_memory_append(key, token, category, content, extra?)
qq_memory_query(key, token, category?, topic?)
qq_memory_remove(key, token, category, content?, target?)
qq_memory_clear(key, token, category?)
qq_memory_share_request(key, token, text)

// V2.1（新增参数全部可选，旧调用完全兼容）
qq_memory_append(key, token, category, content, extra?, 
                 type?, candidateFactKey?, occurredAt?, subjectId?)   ← 新增可选
qq_memory_query(key, token, category?, topic?, 
                includeSuperseded?, includeArchived?)                ← 新增可选
qq_memory_remove(key, token, category, content?, target?, 
                 memoryId?, mode?)      ← memoryId 优先，mode: soft|hard
qq_memory_clear(key, token, category?, mode?)
qq_memory_manage(op, ...)               ← 新增：archive/restore/review
```

## 9.2 兼容映射表

| 旧字段/概念 | V2.1 映射 | 说明 |
|---|---|---|
| `key` | `conversationKey` | 保持格式不变（private:xxx / group:xxx） |
| `text` | `content` | 去掉 category 前缀；读旧数据时按前缀拆 |
| `time` | `createdAt` | 语义不变（都是写入时刻） |
| `subjectId`（已有） | 保持 | 兼容 |
| `subjectName`/`subjectType` | 保留 | 展示用 |
| `sourceType` | `source` | 值域兼容（observed/manual → model/manual） |
| `needs_review`（布尔） | `reviewStatus` | true→pending, false→approved |
| `share_review_status` | `shareReviewStatus` | 值域不变 |
| `visibility` | 保持 | 值域不变 |
| `category`（已存在于部分记录） | 保持 | 不变 |
| `vec` | 保持 | 不变 |

## 9.3 旧数据读取兼容（双读）

```js
function normalizeRow(raw) {
  // 旧记录没有 id/type/status → 补默认值
  return {
    id: raw.id ?? generateLegacyId(raw),           // 确定性生成（同记录每次相同）
    ownerId: raw.ownerId ?? deriveOwner(raw.key),
    conversationKey: raw.conversationKey ?? raw.key,
    subjectId: raw.subjectId ?? 'unknown',
    visibility: raw.visibility ?? 'owner_private',
    category: raw.category ?? inferCategoryFromTextPrefix(raw.text),
    type: raw.type ?? inferType(raw.category),
    status: raw.status ?? 'active',
    reviewStatus: raw.reviewStatus ?? (raw.needs_review === true ? 'pending' : 'approved'),
    shareReviewStatus: raw.shareReviewStatus ?? raw.share_review_status ?? 'not_requested',
    content: raw.content ?? stripCategoryPrefix(raw.text),
    contentHash: raw.contentHash ?? sha256(normalize(raw.content)),
    createdAt: raw.createdAt ?? raw.time,
    occurredAt: raw.occurredAt ?? null,
    updatedAt: raw.updatedAt ?? raw.createdAt ?? raw.time,
    ...
  };
}
```

**`generateLegacyId` 必须确定性**：同一旧记录每次生成相同 id（用 `sha256(key+text+time)` 前 12 位），否则每次读都不同，关联会乱。

## 9.4 控制台 UI 兼容

```
listVectorMemory 返回结构：
  保留旧字段名（key/text/time/category/visibility/...）
  + 新增字段（id/type/status/reviewStatus/...）
  → 旧 UI 读旧字段仍工作，新 UI 读新字段
```

---

# 十、分批实施与回滚方案

## 10.1 现有代码需要修改的位置（精确到行）

| 文件:行 | 当前 | V2.1 改动 | 批次 |
|---|---|---|---|
| `memory-vector.js:44-58` | `loadRows` 容错读 | 加 CRC 校验 + 坏行日志；保留容错语义 | P1 |
| `memory-vector.js:68-83` | `loadRowsStrict` | 加 CRC 校验，坏行抛错 | P1 |
| `memory-vector.js:149-163` | `appendVectorMemory` | 加 id/type/status 等字段；加去重；包 writeLock | P1+P2 |
| `memory-vector.js:171-199` | `searchVectorMemory` | 加 7 层过滤；改 `key`→`ownerId`+`conversationKey` | P1 |
| `memory-vector.js:210-228` | `listVectorMemory` | 返回新字段（保留旧字段别名） | P1 |
| `memory-vector.js:235-258` | `findMemoryRow` | **修正数字被抹掉的缺陷**；id 优先 | P1 |
| `memory-vector.js:311-319` | `removeVectorMemoryExact` | 改逻辑删除（默认） | P3 |
| `memory-vector.js:324-358` | `removeNAll`/`removeAll` | 改逻辑删除 | P3 |
| `memory-vector.js:361-374` | `editVectorMemory` | 加 updatedAt；包 writeLock | P1 |
| `memory-vector.js:376-380` | `atomicRewrite` | **修 tmp 名**；加 fsync；包 writeLock | P1 |
| 新增 `memory-wal.js` | — | WAL 读写 + 状态机 | P4 |
| 新增 `memory-lock.js` | — | 进程内串行 + 文件锁 | P1 |
| 新增 `fact-key.js` | — | factKey 五道验证 + 注册表 | P2 |
| `bridge.js:6628-6694` | append 端点 | 适配新字段 + 返回 opId | P1 |
| `bridge.js:6903-6980` | remove 端点 | 加 memoryId 支持 + 逻辑删除 | P3 |
| `bridge.js:7014-7123` | clear 端点 | 同上 | P3 |
| `bridge.js:6646-6714` | admin 端点 | 加 archive/restore/review 操作 | P3 |
| `mcp-snowluma-safe.js` append/query/remove/clear | 工具 | 加可选新参数 | P1 |
| `mcp-snowluma-safe.js` | 新增 | `qq_memory_manage` 工具 | P3 |

## 10.2 批次划分与验收标准

| 批次 | 内容 | 验收标准 | 风险 |
|---|---|---|---|
| **P0** | 只加 `memory-lock.js` + 修 `atomicRewrite` tmp 名 | 并发写测试不丢数据；现有功能全通 | 低 |
| **P1** | 数据模型扩展 + 双读兼容 + 7 层过滤 | 新记录带全字段；旧记录能读；archived/deleted 不检索 | 中（改检索核心） |
| **P2** | 精确去重 + factKey 五道验证 + supersede | 同文不重复；GPU 案例正确；不同设备不合并 | 中高（改写入核心） |
| **P3** | 逻辑删除 + 归档 + 恢复 + manage 工具 | 删除可恢复；归档不检索；恢复可用 | 中 |
| **P4** | WAL + 状态机 + 崩溃恢复 + 重试队列 | 模拟崩溃能恢复；已删不复活 | 高 |
| **P5** | 加权排序（时间/类型权重） | 排序质量提升；不影响版本判断 | 低 |
| **P6** | （可选）BM25 混合 | 精确词命中提升 | 中 |

## 10.3 回滚方案

```
每批次前：
  备份 memory-vector.js + bridge.js + mcp-snowluma-safe.js
  备份 state/memory-vectors.jsonl（即便当前为空也备份，证明状态）
  备份 state/social-v2.json

回滚：
  P0-P2：直接覆盖文件即可（未改数据格式，或只加字段）
  P3-P4：需要 data rollback
    ① 停服务
    ② 覆盖代码
    ③ 数据：新格式记录被旧代码读时会保留多余字段（无害，因为旧代码只读已知字段）
       但 status=deleted 的记录旧代码会当正常记录读出来 → 需恢复数据快照
    ④ 启动验证

关键：P1 的"只加字段不改旧字段"策略让 P0-P2 可以纯代码回滚
```

## 10.4 迁移方案（当前无数据，但仍要备）

```
当前状态（已核实）：
  memory-vectors.jsonl = 0 字节
  social-v2.json 三个记忆字段 = 0

迁移步骤（若届时已有数据）：
  ① 备份原文件 → memory-vectors.pre-v2.jsonl
  ② 逐行 normalizeRow → 补字段
  ③ 生成确定性 id
  ④ 写新文件（唯一 tmp → rename）
  ⑤ 校验：行数一致、id 唯一、contentHash 全有
  ⑥ 抽样对比 10 条新旧内容一致
  ⑦ 保留原文件 7 天

幂等：按 id 去重，可重复执行
```

---

# 十一、尚未确定、需人工决策

| # | 问题 | 选项 | 影响 |
|---|---|---|---|
| D1 | `factKey` 注册表初始内容 | 谁填？预置常用维度 / 模型提议+人工批 | 决定事实替换覆盖哪些场景 |
| D2 | superseded 默认是否进普通检索 | 不进（严格）/ 进但标注 | "她还记得旧事吗"的体验 |
| D3 | `occurredAt` 来源 | 模型推断 / 只接受显式 | 推断有错风险 vs 模型负担 |
| D4 | 时间衰减曲线参数 | 半衰期 7 天 / 30 天 / 不衰减 | 旧记忆的权重 |
| D5 | `typeWeight` 具体值 | fact 1.2 / event 1.0 / temporary 0.8？ | 排序倾向 |
| D6 | WAL compact 频率与保留期 | 7 天 / 30 天 / 按条数 | 磁盘占用与可追溯性 |
| D7 | 逻辑删除能否转物理删除 | 提供 hard 模式 / 永不物理删 | 合规与可恢复性的取舍 |
| D8 | 跨进程锁超时时间 | 30s / 60s / 更长 | 僵尸锁风险 vs 误接管风险 |
| D9 | 是否自动从 C 层提炼到 A 层 | 不做 / 定期批量 / 按需 | token 消耗 vs 记忆完整性 |
| D10 | `bRefs` 双向引用是否 P1 就实现 | 只留字段位 / 立即实现同步 | 复杂度 |

---

# 十二、本设计未覆盖（明确说明）

```
✗ 向量库替换（Faiss/Qdrant）——记忆量小，不需要
✗ Reranker 重排序——收益不足，成本高
✗ BM25 混合——P6 可选，非必做
✗ 多用户/多主人架构——当前单主人，V2.1 不假设多主人
✗ global 可见性——V2.1 明确不开放
✗ 知识图谱——超出记忆系统范畴
✗ 自动记忆抽取（C→A）——D9 待定
```

---

*本文件为设计文档，未修改任何代码、配置、数据，未重启任何服务。等待审核。*
