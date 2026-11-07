# Memory V2.4.1 最终设计

- 修订时间：2026-10-31 13:05
- 前置：《Memory V2.4 最终设计修订》
- 性质：**纯设计文档。** 本轮严格只读，未修改任何代码/配置/数据，未重启服务。
- 标注：`[已确定]` / `[候选方案]` / `[待决策]`

---

# 〇、V2.4 → V2.4.1 修订对照

| #    | V2.4 问题                                                                                         | 性质        | V2.4.1 修订                                   |
| ---- | ----------------------------------------------------------------------------------------------- | --------- | ------------------------------------------- |
| P0-1 | 群聊模型用 agentToken 发起 ShareRequest，但群聊无权读私人记忆 → **授权链断裂**                                         | **安全缺口**  | 改为**主人私聊主动授权**；群聊模型只能提交不含正文的"建议"            |
| P0-2 | §12.2 允许 `approved → superseded_by_source`，但 §16 第⑧层要求 `status==='approved'` → **副本原件更新后被检索排除** | **直接矛盾**  | 拆为 `approvalStatus` + `freshnessStatus` 双状态 |
| P0-3 | WAL 只有 aStatus/bStatus，实际涉及 6 类资源                                                               | **表达力不足** | 统一 `effects[]` 数组；aStatus/bStatus 降级为兼容字段   |
| P0-4 | §6.4 说"先 tombstone 后 A 层"，§12.5 说"先 A 层后 tombstone"                                             | **顺序矛盾**  | 统一为"tombstone 先行"（检索防线优先）                   |
| P0-5 | recovery_blocked 未定义重试边界；B 层恢复缺校验；vecPurged 未进字段表                                               | 不完整       | 全部补齐 + B 层加 slotRevision/contentHash        |
| P0-6 | 所有模板 perFactKey 都是 single_current，multi_valued 分支无人使用                                           | 冗余        | 暂不实现 multi_valued；新增 `entityGroupKey` 模板声明  |
| P1-3 | C1 写成 `private:X → owner:owner:X`                                                               | **笔误**    | 修正为 `ownerId = owner:X`                     |
| P1-5 | topic 模板 allowedTypes 含 event/temporary，但 event 不该参与 supersede                                  | 策略不清      | 分类型声明 conflictPolicy                        |
| 修-6  | 副本撤销后原件不受影响，但 V2.4 未明确                                                                          | 缺失        | §13 明确                                      |

---

# 一、最终数据模型

## 1.1 A 层记录字段（V2.4.1 定稿）

| 字段                                                           | 类型          | 默认              | 必填            | 说明                                       | 状态      |
| ------------------------------------------------------------ | ----------- | --------------- | ------------- | ---------------------------------------- | ------- |
| `id`                                                         | string      | 生成              | ✅             | `mem_<base36>_<rand6>`                   | `[已确定]` |
| `schemaVersion`                                              | number      | 5               | ✅             |                                          | `[已确定]` |
| `revision`                                                   | number      | 1               | ✅             | 乐观锁                                      | `[已确定]` |
| `ownerId`                                                    | string      | —               | ✅             | `owner:<qq>` / `group:<群号>`              | `[已确定]` |
| `conversationKey`                                            | string      | —               | ✅             | `private:<qq>` / `group:<群号>`            | `[已确定]` |
| `subjectId`                                                  | string      | `unknown`       | ✅             | owner/member:<qq>/bot/topic:<slug>       | `[已确定]` |
| `visibility`                                                 | enum        | `owner_private` | ✅             | owner_private / group_shared             | `[已确定]` |
| `category`                                                   | enum        | `unknown`       | ✅             |                                          | `[已确定]` |
| `type`                                                       | enum        | `event`         | ✅             |                                          | `[已确定]` |
| `status`                                                     | enum        | `active`        | ✅             |                                          | `[已确定]` |
| `reviewStatus`                                               | enum        | `approved`      | ✅             |                                          | `[已确定]` |
| `content`                                                    | string      | —               | ✅             | purged 时 `'[deleted]'`                   | `[已确定]` |
| `contentHash`                                                | string      | 自动              | ✅             | **原始正文的 hash，purged 后不变**                | `[已确定]` |
| `contentPurged`                                              | boolean     | false           | ✅             |                                          | `[已确定]` |
| `vec`                                                        | number[]    | —               | ✅             | 见 C9                                     | `[已确定]` |
| `vecPurged`                                                  | boolean     | false           | ✅             | **新增，进字段表**                              | `[已确定]` |
| `vecModel`                                                   | string|null | `bge-m3`        | 见 C9          |                                          | `[已确定]` |
| `_crc`                                                       | string      | 自动              | ✅             |                                          | `[已确定]` |
| `createdAt` / `occurredAt` / `updatedAt` / `statusChangedAt` | number      |                 |               |                                          | `[已确定]` |
| `lifecycleEvents`                                            | object[]    | `[]`            | ✅             |                                          | `[已确定]` |
| `supersededAt` / `archivedAt` / `deletedAt`                  | number|null | null            |               |                                          | `[已确定]` |
| `expiresAt`                                                  | number|null | null            |               | temporary TTL                            | `[已确定]` |
| `factKey` / `factKeyTemplate` / `factKeyTemplateVersion`     |             | null            |               |                                          | `[已确定]` |
| `entityGroupKey`                                             | string|null | null            |               | **新增，由模板声明**                             | `[已确定]` |
| `supersedes` / `supersededBy`                                | string|null | null            |               |                                          | `[已确定]` |
| `conflictGroupId`                                            | string|null | null            |               |                                          | `[已确定]` |
| `isCandidate`                                                | boolean     | false           | ✅             |                                          | `[已确定]` |
| `source`                                                     | enum        | `model`         | ✅             | model/manual/migration/import/share_copy | `[已确定]` |
| `sourceMemoryId` / `sourceRevision` / `shareRequestId`       |             | null            | share_copy 必填 |                                          | `[已确定]` |
| `bRefs`                                                      | object[]    | `[]`            |               | `[{conversationKey, category, slotId}]`  | `[已确定]` |
| `opId`                                                       | string|null | null            |               |                                          | `[已确定]` |
| **`shareReviewStatus`**                                      | —           | —               | —             | **V2.4.1 彻底移除**（不写、不读、不返）                | `[已确定]` |

## 1.2 B 层槽位字段（新增校验字段）

| 字段                | 说明                           | 状态      |
| ----------------- | ---------------------------- | ------- |
| `slotId`          | `slot_<rand8>`，永不变           | `[已确定]` |
| `slotRevision`    | **槽位内容版本号，每次改 text 时 +1**    | `[已确定]` |
| `contentHash`     | **槽位当前 text 的 hash**（用于恢复校验） | `[已确定]` |
| `aMemoryId`       | 指向 A 层记录（可选）                 | `[已确定]` |
| `lastOperationId` | 最后一次写该槽位的 opId               | `[已确定]` |
| `text`            | 内容（可变）                       | `[已确定]` |

## 1.3 约束（V2.4.1 修正）

```
C1  【修正笔误】ownerId 与 conversationKey 同源：
      private:X → ownerId = 'owner:X'       ← 不是 owner:owner:X
      group:X   → ownerId = 'group:X'

C2  visibility=group_shared 必要条件：
      ownerId 以 'group:' 开头
      AND source='share_copy'
      AND shareRequestId ≠ null
      AND 对应 ShareRequest.approvalStatus='approved'

C3  factKey 必须匹配唯一模板；模板的 subject 或 subjectIdMapping 必须与 subjectId 一致

C4  factKey 允许的 type 由模板 decision 决定（见 §3.3 的类型策略）

C5  状态与时间字段：
      status='superseded' → supersededAt = statusChangedAt
      status='archived'   → archivedAt   = statusChangedAt
      status='deleted'    → deletedAt    = statusChangedAt
      三个时间字段不要求互斥

C6  revision 单调递增

C7  conflictGroupId ≠ null 或 isCandidate=true → 不参与自动注入

C8  source='share_copy' → sourceMemoryId/sourceRevision/shareRequestId 必填，factKey 必须 null

C9  【向量与清除的强联动】
      contentPurged=false:
        content 是真实正文（≠ '[deleted]'）
        vec.length === 1024
        vecPurged === false
        vecModel === 'bge-m3'
        contentHash === sha256(normalize(content))
      
      contentPurged=true:
        content === '[deleted]'
        vec.length === 0
        vecPurged === true
        vecModel === null
        contentHash === 【原始正文的 hash，保持不变】
        ★ 绝不用 '[deleted]' 重新计算覆盖 contentHash
      
      校验失败 → 该记录标 reviewStatus='pending'，不参与检索

C10 shareReviewStatus 字段已移除；分享状态一律从 ShareRequest 查

C11 entityGroupKey 由模板显式声明，不从 factKey 段推导
```

---

# 二、双层 cardinality 与 entityGroupKey（P0-6）

## 2.1 最终简化决定

`[已确定]`

```
perFactKeyCardinality: 当前版本恒为 single_current
  ★ 不实现 multi_valued 分支
  ★ 字段保留（schema 兼容），但所有启用模板都写 single_current
  ★ 将来需要时通过新模板版本扩展

perEntityCardinality: single | multi
  保留，用于描述"同一实体下允许多个实例"
```

**为什么砍掉 multi_valued**：V2.4 的所有模板实际都是 single_current，留一个没有模板使用的分支会增加实现和测试成本，且容易误用。

## 2.2 entityGroupKey 由模板声明（新增）

`[已确定]`

```
模板显式声明 entityGroupKey 的构造方式，程序不猜：

device 模板:
  entityGroupKey = '{subject}:{entity}'    → 'owner:device'
  instanceKey    = qualifier 段             → 'desktop'
  attribute      = attribute 段             → 'gpu'

解析示例：
  owner.device.desktop.gpu
  → subjectId        = 'owner'
  → entityGroupKey   = 'owner:device'
  → instanceKey      = 'desktop'
  → attribute        = 'gpu'
  → factKey          = 完整字符串

member_relation 模板:
  entityGroupKey = 'member:{qq}:relation'  → 'member:[OWNER_QQ]:relation'
  instanceKey    = type 段                  → 'friend'
  attribute      = null

topic_attr 模板:
  entityGroupKey = 'topic:{slug}'          → 'topic:stm32-debug'
  instanceKey    = null
  attribute      = attribute 段             → 'issue'
```

## 2.3 两个维度的实际约束

| perFactKey       | perEntity | 实际含义                                               | 用的模板                                        |
| ---------------- | --------- | -------------------------------------------------- | ------------------------------------------- |
| `single_current` | `multi`   | 每个 factKey 只能 1 个 active；同 entityGroup 下可多 factKey | device / schedule / preference / topic_attr |
| `single_current` | `single`  | 每个 factKey 1 个 active；同 entityGroup 只能 1 个 factKey | （暂无使用）                                      |

**所有启用模板当前都是 `single_current` + `multi`。**

## 2.4 四个场景（最终校验）

### 场景 1：不同设备共存

```
owner.device.desktop.gpu = '[GPU]'
  entityGroupKey='owner:device', instanceKey='desktop', attribute='gpu'
  factKey 查重 → 0 条 → 新增

owner.device.lab.gpu = 'RTX 4090'
  entityGroupKey='owner:device', instanceKey='lab'
  factKey 查重 → 0 条（factKey 不同）→ 新增
  perEntity=multi → 允许同 entityGroup 下多 factKey ✅

→ 2 条 active
```

### 场景 2：同设备升级

```
已有 owner.device.desktop.gpu = '[GPU]'（active, rev=1）

写 owner.device.desktop.gpu = '[GPU]'
  factKey 查重 → 命中 1 条（全等）
  perFactKey=single_current → 必须处理
  conflictPolicy=supersede_with_review
  → 两阶段：
    阶段1：新 active+pending+isCandidate+cg_1，旧【不动】
    阶段2（批准）：一次原子重写，旧→superseded，新→approved
```

### 场景 3：同群友多关系

```
member.<A>.relation.classmate = '大学同学'  → active
member.<A>.relation.teammate  = '队友'      → active
member.<A>.relation.friend    = '关系不错'  → active

三个不同 factKey（instanceKey 不同：classmate/teammate/friend）
entityGroupKey 都是 'member:<A>:relation'
perEntity=multi → 共存 ✅
perFactKey=single → 同 type 只能 1 条 active
```

### 场景 4：同类型关系变化

```
member.<A>.relation.friend = '一般朋友'（active, rev=1）

写 member.<A>.relation.friend = '很熟的朋友'
  factKey 全等 → perFactKey=single → supersede_with_review
  → 两阶段

★ 这表达"友情的程度变化"，不是"新增另一种关系"
```

---

# 三、factKey 模板最终表

## 3.1 六个模板

| id                | arity | 段结构                                 | perFactKey | perEntity | entityGroupKey         | conflictPolicy        | allowedTypes           |
| ----------------- | ----- | ----------------------------------- | ---------- | --------- | ---------------------- | --------------------- | ---------------------- |
| `device`          | 4     | subject.literal.qualifier.attribute | single     | multi     | `{subject}:device`     | supersede_with_review | fact                   |
| `preference`      | 3     | subject.literal.attribute           | single     | multi     | `{subject}:preference` | supersede_with_review | fact, preference       |
| `schedule`        | 3     | subject.literal.attribute           | single     | multi     | `{subject}:schedule`   | supersede_with_review | fact                   |
| `member_relation` | 4     | literal.qq.literal.qualifier        | single     | multi     | `member:{qq}:relation` | supersede_with_review | relationship           |
| `bot_persona`     | 3     | subject.literal.attribute           | single     | multi     | `bot:persona`          | supersede             | fact, preference       |
| `topic_attr`      | 3     | literal.slug.attribute              | single     | multi     | `topic:{slug}`         | 按 type（见 3.3）         | fact, event, temporary |

## 3.2 枚举值

```
device.qualifier  : desktop, lab, phone, work, home
device.attribute  : gpu, cpu, ram, disk, os, monitor
preference.attribute : food, game, music, sleep, social, work
schedule.attribute   : sleep, meal, study
member_relation.qualifier : classmate, teammate, friend, colleague, ex_colleague, family, online
bot_persona.attribute     : name, mood, habit
topic_attr.attribute      : issue, status, deadline, owner
```

## 3.3 topic 模板的类型策略（P1-5）

`[已确定]` **按 type 分策略**：

| type        | conflictPolicy          | 是否参与 supersede | 理由                      |
| ----------- | ----------------------- | -------------- | ----------------------- |
| `fact`      | `supersede_with_review` | ✅              | 话题的事实属性（如 issue 内容）会变   |
| `event`     | `keep_both`             | ❌              | 历史事件不冲突（"改过一次 bug"可以多次） |
| `temporary` | `keep_both`             | ❌              | 靠 expiresAt 生命周期处理      |

**实现**：

```
topic_attr 模板的 conflictPolicy 不是单一值，而是按 type 查表：
  if (record.type === 'fact')      → supersede_with_review
  if (record.type === 'event')     → keep_both
  if (record.type === 'temporary') → keep_both

★ 这是唯一一个"按 type 决策"的模板。其他模板 conflictPolicy 固定。
★ 模板字段需支持：conflictPolicyByType: {fact: '...', event: '...', temporary: '...'}
```

## 3.4 模板歧义防护

```
规则：同一 key 只能匹配一个 enabled 模板

验证时机：
  ① 启动加载时
  ② 每次模板变更后

验证方法：
  对每个模板，构造"最小合法 key"和"最大合法 key"，检查是否只匹配自己
  发现歧义 → 拒绝加载该模板 + 写 audit 报警 + 控制台红标
```

## 3.5 topic 的 subjectId 映射（P1-4）

`[已确定]`

```
factKey = 'topic.stm32-debug.issue'
→ slug 段 = 'stm32-debug'
→ subjectId 必须是 'topic:stm32-debug'

模板声明：
  "subjectIdMapping": "topic:{segment1}"

校验：
  expected = 'topic:' + segments[1]
  if (record.subjectId !== expected) → REJECT_SUBJECT_MISMATCH
```


# 五、ShareRequest 主人授权链（P0-1）

## 5.1 核心原则

`[已确定]`

```
① 只有记忆的 owner（私聊主人）才能授权分享自己的记忆
② 群聊模型不能读取、枚举、猜测私人 memoryId
③ 控制台审批 ≠ 主人授权（审批是流程，授权是意愿）
④ 目标群必须是已验证的已知群
```

## 5.2 两种发起路径

### 路径 A：主人主动授权（唯一能绑定私人记忆的路径）

```
主人在私聊说"把这条记忆分享到 X 群"

① bridge 在私聊会话收到 → conversationKey='private:<ownerQQ>'
② 模型调 qq_memory_share_request(
     key='private:<ownerQQ>',
     token=<私聊 token>,
     sourceMemoryId='mem_xxx',       ← 必须是本会话可读的私人记忆
     targetGroupId='<群号>'          ← 必须显式指定
   )
③ 权限校验（见 5.4）
④ 创建 ShareRequest：
     requestedBy: 'model'
     requestedByConversationKey: 'private:<ownerQQ>'
     authorizedBy: 'owner:<ownerQQ>'      ← 授权者是主人
     targetOwnerId: 'group:<群号>'
     approvalStatus: 'pending'
⑤ 主人可通过控制台看到这条 pending
```

**关键**：`sourceMemoryId` 必须是私聊会话能访问的记录。群聊会话调这个工具时，无法提供合法的 private memoryId（因为它读不到）。

### 路径 B：群友建议（不含正文，不绑定记忆）

```
群友在群里说"这个可以记下来给大家看"

① bridge 在群聊会话收到 → conversationKey='group:X'
② 模型可调 qq_memory_suggest_share(
     key='group:X',
     token=<群 token>,
     topic='关于主人显卡的配置'      ← 只是描述，不含正文、不含 memoryId
   )
③ 创建 ShareSuggestion（不是 ShareRequest）：
     suggestionId, conversationKey, topic, createdAt, status='noted'
④ 控制台显示"群 X 有个分享建议"，主人可自行决定是否授权
⑤ ShareSuggestion 永不自动转成 ShareRequest
```

**为什么需要路径 B**：模型可以提示"这里可能有值得分享的内容"，但**不能**因此获得访问私人记忆的权限。

## 5.3 完整权限检查顺序（6 步）

```
① 发起者身份
   token 反查 → conversationKey + ownerId
   若 conversationKey 是 group:* → 只能走路径 B（suggest），不能建 ShareRequest
   若 conversationKey 是 private:<A> → 可走路径 A

② 记忆归属
   sourceMemoryId 必须存在
   且 record.ownerId === 'owner:<A>'（发起者本人）
   且 record.conversationKey === 当前私聊
   ★ 不能分享别人的记忆

③ 记忆状态
   record.status === 'active'
   record.reviewStatus === 'approved'
   record.contentPurged === false    ← 已清空正文的不能分享
   record.source !== 'share_copy'    ← 副本不能再分享

④ 目标群验证
   targetGroupId 必须：
     · 是 bridge 已知的群（在 allow.groups 或历史会话中出现过）
     · 不是当前私聊
     · 不是自己已分享过的群（幂等检查）
   ★ 不能由模型随意指定不存在的群

⑤ revision 校验
   record.revision 必须等于请求时的快照（防基于旧版本分享）

⑥ 幂等检查
   按 (sourceMemoryId, targetOwnerId)：
     · 有 approved 且副本 active → already_shared
     · 有 pending → already_pending
     · 有 rejected → 允许新建
     · 有 revoked → 允许新建
```

## 5.4 控制台审批 ≠ 主人授权

```
ShareRequest 有两个"人"的信息：
  authorizedBy:  string | null    ← 主人授权（路径 A 才有）
  reviewedBy:    string | null    ← 控制台审批

规则：
  ① 只有 authorizedBy ≠ null 的 request 才能被批准
  ② 群友建议（ShareSuggestion）没有 authorizedBy，不能直接批准
     → 主人必须先"授权"（转成 ShareRequest 或手动指定）
  ③ 控制台可以审批但不可以代替授权
     → 这是防"控制台管理员滥用"
  ④ 若未来有多管理员，authorizedBy 仍是主人
```

## 5.5 ShareRequest 字段（V2.4.1）

```json
{
  "requestId": "sr_<rand8>",
  "sourceMemoryId": "mem_orig",
  "sourceRevision": 3,
  "sourceContentHash": "abc...",
  "targetOwnerId": "group:[GROUP_ID]",
  "targetConversationKey": "group:[GROUP_ID]",
  "approvalStatus": "pending",
  "freshnessStatus": "current",
  "requestedAt": 1793400000000,
  "requestedBy": "model",
  "requestedByConversationKey": "private:[OWNER_QQ]",
  "authorizedBy": "owner:[OWNER_QQ]",
  "reviewedAt": null,
  "reviewedBy": null,
  "copyMemoryId": null,
  "revokedAt": null,
  "revokedReason": null,
  "reason": "",
  "expiresAt": null,
  "revision": 1,
  "opId": "op_share_1"
}
```

## 5.6 取消未完成的申请

```
主人取消：
  ① 控制台或私聊调 qq_memory_manage(op='cancel_share', requestId)
  ② 校验调用者是 authorizedBy 对应的主人（私聊 token）或控制台
  ③ request.approvalStatus: 'pending' → 'rejected'
     reason: 'cancelled_by_owner'
  ④ 写 audit
  ⑤ 若已有副本（不应有，因为 pending 时无副本）→ 报错
```

## 5.7 过期申请

```
若 request.expiresAt ≠ null 且 now > expiresAt：
  ① 控制台维护任务（或检索时惰性检查）把 approvalStatus 改 'rejected'
     reason: 'expired'
  ② 不自动删除记录（保留审计）
  ③ 主人可重新发起

默认 expiresAt = null（不过期），除非显式设置
```

## 5.8 五个异常场景（P0-1 专项）

| #  | 场景                                             | 处理                                                                                                     |
| -- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| A1 | 群聊模型尝试用群 token 发起 ShareRequest，指定一个私人 memoryId | ① 检查 conversationKey='group:X' → 只能走路径 B → 拒绝 ShareRequest 创建，返回 `{code:'GROUP_CANNOT_SHARE_PRIVATE'}` |
| A2 | 群聊模型猜了一个 memoryId（如 `mem_0001`）尝试分享            | 同 A1，在步骤①就被拦（不进入 memoryId 校验）                                                                          |
| A3 | 主人私聊想分享**别人**的记忆                               | 步骤②：record.ownerId ≠ 发起者 → 拒绝 `{code:'NOT_OWNER'}`                                                     |
| A4 | 主人私聊分享自己的记忆到不存在的群                              | 步骤④：targetGroupId 不在已知群列表 → 拒绝 `{code:'UNKNOWN_TARGET_GROUP'}`                                         |
| A5 | 控制台管理员试图批准一条没有 authorizedBy 的 request（群友建议转来的） | 步骤①（审批时校验）：authorizedBy === null → 拒绝 `{code:'NO_OWNER_AUTHORIZATION'}`                                |

---

# 六、approvalStatus 与 freshnessStatus（P0-2）

## 6.1 两个独立维度

`[已确定]`

```
approvalStatus   审批状态（决定能不能检索）
  pending    等待批准
  approved   已批准（副本可检索）
  rejected   被拒绝
  revoked    已撤销（副本不可检索）

freshnessStatus  副本新旧（不影响能否检索）
  current    副本对应的原件版本仍是当前
  stale      原件已更新，但副本保留

★ 关键：检索只看 approvalStatus，不看 freshnessStatus
```

## 6.2 完整状态转换表

### approvalStatus

| from       | to         | 触发                | 副作用                           |
| ---------- | ---------- | ----------------- | ----------------------------- |
| （无）        | `pending`  | 主人授权发起            | 写 WAL，无副本                     |
| `pending`  | `approved` | 控制台批准             | 创建副本（新 memoryId）              |
| `pending`  | `rejected` | 控制台拒绝 / 主人取消 / 过期 | 无副本                           |
| `approved` | `revoked`  | 主人撤回 / 原件删除级联     | 副本 status=deleted + tombstone |

**终态**：`rejected` / `revoked`

### freshnessStatus

| from      | to        | 触发                    | 影响                                     |
| --------- | --------- | --------------------- | -------------------------------------- |
| `current` | `stale`   | 原件 revision 变化（内容改了）  | **副本仍可检索**，但标记过期                       |
| `stale`   | `current` | 主人重新分享（新 request，新副本） | 新 request 的 current，旧 request 保持 stale |

**`freshnessStatus` 不影响 approvalStatus**。

## 6.3 原件更新后的行为（修正 V2.4 的矛盾）

`[已确定]`

```
原件 mem_orig 被编辑（revision 3 → 4）：

① 查所有 sourceMemoryId=mem_orig 且 approvalStatus='approved' 的 request
② 每个 request：
     freshnessStatus: 'current' → 'stale'
     ★ approvalStatus 保持 'approved'（不改为 superseded_by_source）
     ★ 副本 status 保持 'active'（不删）
③ 副本内容不自动同步（保持旧快照）
④ 写 audit：{action:'share_stale', requestId, oldRevision:3, newRevision:4}

检索影响：
  副本仍可检索（approvalStatus='approved' 未变）✅
  但结果里可标注 "此内容基于原件的旧版本（rev 3）"
```

**为什么副本要保持可见**：

- 群里已经看到的内容不能突然消失（会造成困惑）
- 用户撤回才应该消失（approvalStatus → revoked）

## 6.4 检索判定表

| 副本状态 | approvalStatus | freshnessStatus | 可检索    | 说明       |
| ---- | -------------- | --------------- | ------ | -------- |
| 正常   | `approved`     | `current`       | ✅      | 正常       |
| 原件更新 | `approved`     | `stale`         | ✅      | 可检索但标注过期 |
| 主人撤回 | `revoked`      | 任意              | ❌      | 撤销       |
| 原件删除 | `revoked`      | 任意              | ❌      | 级联撤销     |
| 待批   | `pending`      | —               | ❌（无副本） | —        |
| 被拒   | `rejected`     | —               | ❌（无副本） | —        |

**检索代码**：

```js
if (record.source === 'share_copy') {
  const req = shareRequests.get(record.shareRequestId);
  if (!req || req.approvalStatus !== 'approved') return EXCLUDE;
  // freshnessStatus 不影响可见性，只影响展示
}
```

## 6.5 策略对比（为什么不用 V2.4 的单状态）

| 方案                          | 原件更新后     | 问题           |
| --------------------------- | --------- | ------------ |
| V2.4：`superseded_by_source` | 副本不可检索    | ❌ 群里内容"莫名消失" |
| V2.4.1：`approved` + `stale` | 副本可检索，标过期 | ✅ 符合直觉       |

---

# 七、统一 WAL effects 状态机（P0-3）

## 7.1 核心结构

`[已确定]`

```json
{
  "opId": "op_xxx",
  "kind": "append|approve_fact_update|reject_fact_update|cascade_delete|share_copy|revoke_share|force_restore|compact",
  "createdAt": 1793400000000,
  "updatedAt": 1793400000500,
  "operationStatus": "created|running|partial|committed|failed_retriable|failed_exhausted|recovery_blocked|cancelled",
  "retryCount": 0,
  "effects": [ ... ],
  "aStatus": "done",
  "bStatus": "done"
}
```

**`aStatus` / `bStatus` 保留为兼容字段**（从 effects 派生，便于旧代码读）：

```js
aStatus = effects.find(e => e.resource === 'memory_vectors')?.status ?? 'none'
bStatus = effects.find(e => e.resource === 'social_v2')?.status ?? 'none'
```

## 7.2 effect 完整字段

| 字段            | 类型          | 必填 | 说明                                                   |
| ------------- | ----------- | -- | ---------------------------------------------------- |
| `effectId`    | string      | ✅  | `eff_<opId后缀>_<序号>`，op 内唯一                           |
| `resource`    | enum        | ✅  | 见 7.3                                                |
| `action`      | enum        | ✅  | `create` / `batch_update` / `append` / `delete_file` |
| `targetIds`   | string[]    | ✅  | 目标对象 ID（记录 id / slotId / requestId）                  |
| `requirement` | enum        | ✅  | `required` / `optional`                              |
| `order`       | number      | ✅  | 执行顺序（小到大）                                            |
| `status`      | enum        | ✅  | `pending` / `done` / `failed` / `skipped`            |
| `expect`      | object[]    | ❌  | 执行前校验条件（revision/status）                             |
| `set`         | object[]    | ❌  | 执行后的目标状态（用于验证）                                       |
| `effectHash`  | string|null | ❌  | 执行后计算（用于验证内容）                                        |
| `attempts`    | number      | ✅  | 尝试次数                                                 |
| `verifiedAt`  | number|null | ❌  | 最后一次验证通过的时间                                          |
| `lastError`   | string|null | ❌  | 最后错误                                                 |

## 7.3 resource 合法枚举

`[已确定]`

| resource            | 对应文件                            | required 默认       |
| ------------------- | ------------------------------- | ----------------- |
| `memory_tombstones` | `state/memory-tombstones.jsonl` | ✅ required        |
| `memory_vectors`    | `state/memory-vectors.jsonl`    | ✅ required        |
| `social_v2`         | `state/social-v2.json`          | 视情况               |
| `share_requests`    | `state/share-requests.jsonl`    | ✅ required        |
| `memory_audit`      | `state/memory-audit.jsonl`      | **optional**      |
| `fact_keys`         | `state/fact-keys.json`          | ✅ required（模板变更时） |


**为什么 audit 是 optional**：审计是"记录效果"，不是"业务效果"。审计失败不影响业务正确性，但要报警。  
**但**：审计失败时 `operationStatus` 加 `auditWarn: true`，控制台红标。

## 7.4 验证方式（每个 resource 具体怎么验）

| resource            | 验证方法                                                                |
| ------------------- | ------------------------------------------------------------------- |
| `memory_tombstones` | 读文件，确认 targetIds 每个都有 `state` 符合 `set` 的记录                          |
| `memory_vectors`    | 读文件，确认 targetIds 每个的 `status`/`reviewStatus`/`revision` 符合 `set`    |
| `social_v2`         | 读文件，确认 targetIds（slotId）每个存在且 `slotRevision`/`contentHash` 符合 `set` |
| `share_requests`    | 读文件，确认 targetIds 每个的 `approvalStatus`/`freshnessStatus` 符合 `set`    |
| `memory_audit`      | 读文件末尾 N 行，确认存在 `opId` 匹配的记录                                         |
| `fact_keys`         | 读文件，确认 `version` 符合 `set`                                           |

**关键**：验证是**读完整个文件后逐条比对**，不是"看返回值"。

## 7.5 committed 的充分必要条件

`[已确定]`

```
operationStatus = 'committed'
当且仅当：
  ① 所有 requirement='required' 的 effect.status === 'done'
  ② 每个 required effect 的 verify() 通过（见 7.4）
  ③ 没有 effect.status === 'failed' 且 requirement='required'

注意：
  ✗ 不能因为 aStatus=done 且 bStatus=done 就 committed
    （可能 share_requests 还没完成）
  ✗ 不能因为"写文件返回成功"就认为是 done
    （必须 verify）
```

## 7.6 重试条件

| effect.status                     | 可重试 | 条件               |
| --------------------------------- | --- | ---------------- |
| `pending`                         | ✅   | 执行顺序到了           |
| `failed` + `requirement=required` | ✅   | retryCount < 3   |
| `failed` + `requirement=optional` | ✅   | 但失败不阻止 committed |
| `done`                            | ❌   | 已成功，不重复          |
| `skipped`                         | ❌   | 显式跳过             |

**重试只重试 status ≠ 'done' 的 effect**，已 done 的不动（幂等）。

## 7.7 降级规则

```
哪个 effect 允许降级（failed 但仍 committed）：
  · memory_audit（optional）→ 降级，加 auditWarn
  · social_v2 在某些场景（如纯 A 层记忆无 B 层槽位）→ requirement='optional'

哪个绝不允许降级：
  · memory_tombstones（required）— 删除的防线
  · memory_vectors（required）— 主体数据
  · share_requests（required）— 分享状态
```

## 7.8 崩溃恢复流程（effects 版）

```
启动时扫 WAL：
  for each op where operationStatus ∉ 终态:
    ① 对每个 effect：
       status='done' → 重新 verify()，通过则保持 done
                       verify 失败 → 改 pending（内容不符）
       status='pending'/'failed' → 保持，待执行
    ② 检查所有 targetIds 是否已被删除（查 tombstone）
       若某个 targetId 已 deleted 且 effect 是"创建/更新"→ 该 effect 标 skipped
    ③ 重新计算 operationStatus：
       全部 required done+verified → committed
       部分 done → partial
       全部 failed → failed_retriable
       targetIds 全部 deleted 且 effect 无意义 → cancelled
```

## 7.9 五种操作的完整 effects 示例

### 7.9.1 append

```json
{
  "opId": "op_append_1",
  "kind": "append",
  "effects": [
    {
      "effectId": "eff_a1",
      "resource": "memory_vectors",
      "action": "batch_update",
      "targetIds": ["mem_new"],
      "requirement": "required",
      "order": 1,
      "expect": [],
      "set": [{"id": "mem_new", "status": "active", "reviewStatus": "approved", "revision": 1}],
      "effectHash": "sha256(record)"
    },
    {
      "effectId": "eff_a2",
      "resource": "social_v2",
      "action": "batch_update",
      "targetIds": ["slot_x1"],
      "requirement": "required",
      "order": 2,
      "set": [{"slotId": "slot_x1", "slotRevision": 1, "contentHash": "abc"}]
    },
    {
      "effectId": "eff_a3",
      "resource": "memory_audit",
      "action": "append",
      "targetIds": ["mem_new"],
      "requirement": "optional",
      "order": 3
    }
  ]
}
```

**注意**：若 category 无 B 层对应（如 `manual`），`social_v2` effect 的 requirement 改为 `optional` 或省略。

### 7.9.2 approve_fact_update

见 §4.2（memory_vectors 一个 required + audit 一个 optional）。

### 7.9.3 cascade_delete（原件 + 2 个副本）

```json
{
  "opId": "op_del_1",
  "kind": "cascade_delete",
  "effects": [
    {
      "effectId": "eff_d1",
      "resource": "memory_tombstones",
      "action": "create",
      "targetIds": ["mem_orig", "mem_copy_1", "mem_copy_2"],
      "requirement": "required",
      "order": 1,
      "set": [
        {"memoryId": "mem_orig", "state": "active", "suppression": "memory_only"},
        {"memoryId": "mem_copy_1", "state": "active"},
        {"memoryId": "mem_copy_2", "state": "active"}
      ]
    },
    {
      "effectId": "eff_d2",
      "resource": "memory_vectors",
      "action": "batch_update",
      "targetIds": ["mem_orig", "mem_copy_1", "mem_copy_2"],
      "requirement": "required",
      "order": 2,
      "expect": [
        {"id": "mem_orig", "revision": 5, "status": "active"},
        {"id": "mem_copy_1", "revision": 2, "status": "active"},
        {"id": "mem_copy_2", "revision": 2, "status": "active"}
      ],
      "set": [
        {"id": "mem_orig", "status": "deleted", "revision": 6},
        {"id": "mem_copy_1", "status": "deleted", "revision": 3},
        {"id": "mem_copy_2", "status": "deleted", "revision": 3}
      ]
    },
    {
      "effectId": "eff_d3",
      "resource": "share_requests",
      "action": "batch_update",
      "targetIds": ["sr_1", "sr_2"],
      "requirement": "required",
      "order": 3,
      "expect": [
        {"requestId": "sr_1", "revision": 2, "approvalStatus": "approved"},
        {"requestId": "sr_2", "revision": 2, "approvalStatus": "approved"}
      ],
      "set": [
        {"requestId": "sr_1", "approvalStatus": "revoked", "revokedReason": "source_deleted", "revision": 3},
        {"requestId": "sr_2", "approvalStatus": "revoked", "revokedReason": "source_deleted", "revision": 3}
      ]
    },
    {
      "effectId": "eff_d4",
      "resource": "memory_audit",
      "action": "append",
      "targetIds": ["mem_orig"],
      "requirement": "optional",
      "order": 4
    }
  ]
}
```

**order 1 = tombstone 永远最先**（P0-4 修正）。

### 7.9.4 share_copy

```json
{
  "opId": "op_share_1",
  "kind": "share_copy",
  "effects": [
    {
      "effectId": "eff_s1",
      "resource": "memory_vectors",
      "action": "batch_update",
      "targetIds": ["mem_copy_new"],
      "requirement": "required",
      "order": 1,
      "set": [{
        "id": "mem_copy_new", "ownerId": "group:[GROUP_ID]",
        "visibility": "group_shared", "source": "share_copy",
        "status": "active", "reviewStatus": "approved", "revision": 1
      }]
    },
    {
      "effectId": "eff_s2",
      "resource": "share_requests",
      "action": "batch_update",
      "targetIds": ["sr_new"],
      "requirement": "required",
      "order": 2,
      "expect": [{"requestId": "sr_new", "revision": 1, "approvalStatus": "pending"}],
      "set": [{"requestId": "sr_new", "approvalStatus": "approved", "copyMemoryId": "mem_copy_new", "freshnessStatus": "current", "revision": 2}]
    },
    {
      "effectId": "eff_s3",
      "resource": "memory_audit",
      "action": "append",
      "targetIds": ["mem_copy_new"],
      "requirement": "optional",
      "order": 3
    }
  ]
}
```

**顺序分析**：先写副本（可见），再更新 request。

- 若 ①成功 ②失败 → 副本可见但 request 还 pending → 检索第⑧层会排除（因为 request 不是 approved）→ **安全**
- 若 ①失败 → 无副本，request 保持 pending → 安全

★ 所以 share_copy 的顺序是"先副本后 request"，与 cascade_delete 的"先 tombstone"不同。  
原因：share_copy 的"安全"依赖 request 状态，所以 request 必须最后写（写成功才放行）。

### 7.9.5 revoke_share

```json
{
  "opId": "op_revoke_1",
  "kind": "revoke_share",
  "effects": [
    {
      "effectId": "eff_r1",
      "resource": "memory_tombstones",
      "action": "create",
      "targetIds": ["mem_copy_1"],
      "requirement": "required",
      "order": 1,
      "set": [{"memoryId": "mem_copy_1", "state": "active"}]
    },
    {
      "effectId": "eff_r2",
      "resource": "memory_vectors",
      "action": "batch_update",
      "targetIds": ["mem_copy_1"],
      "requirement": "required",
      "order": 2,
      "expect": [{"id": "mem_copy_1", "revision": 2, "status": "active"}],
      "set": [{"id": "mem_copy_1", "status": "deleted", "revision": 3}]
    },
    {
      "effectId": "eff_r3",
      "resource": "share_requests",
      "action": "batch_update",
      "targetIds": ["sr_1"],
      "requirement": "required",
      "order": 3,
      "expect": [{"requestId": "sr_1", "revision": 2, "approvalStatus": "approved"}],
      "set": [{"requestId": "sr_1", "approvalStatus": "revoked", "revokedAt": 1793400000000, "revision": 3}]
    }
  ]
}
```

## 7.10 不得声称的（审核要求）

```
❌ WAL 不能让多个普通文件具备数据库事务的原子性
   真实情况：
     · 单文件：atomicRewrite（rename 原子）
     · 多文件：顺序写 + 失败重试 + 最终验证
     · 中间态可能被读到（但通过状态字段过滤，业务上安全）
   
✅ 可以声称的：
     · 每个文件内部是原子的
     · 跨文件最终一致（靠 effects 重试）
     · 任何中间态在检索端都是安全的（因为状态过滤）
```

---

# 八、统一删除与级联撤销顺序（P0-4）

## 8.1 统一原则

`[已确定]`

```
先保护检索不可见（tombstone），再完成数据层状态变更。
```

**这修正了 V2.4 §6.4 与 §12.5 的顺序矛盾。**

## 8.2 完整删除流程（12 步）

```
① 取 bridge 写锁（进程内串行）
② 创建 WAL operation（operationStatus='created'）
③ 读原件记录 + 所有 sourceMemoryId=原件 且 approvalStatus='approved' 的 ShareRequest
④ 收集所有副本 memoryId（从 request.copyMemoryId）
⑤ 校验所有相关 revision（原件 + 副本 + request）
   任一不符 → 事务中止，返回 CONFLICT_REVISION
⑥ 【effect 1】一次原子写 tombstone（原件 + 所有副本，整批）
   → 更新内存 tombstone 索引
   ★ 此后所有相关记录在检索端已不可见（即使 A 层未改）
⑦ 【effect 2】一次原子重写 A 层：原件 + 所有副本 → status='deleted'
⑧ 【effect 3】原子重写 ShareRequest：所有相关 request → approvalStatus='revoked'
⑨ 【effect 4，optional】写 audit
⑩ 验证所有 required effects（读文件逐条比对）
⑪ 全部通过 → operationStatus='committed'
   任一失败 → 'partial'，按 effects 重试
⑫ 释放锁
```

## 8.3 各失败点的具体处理

| 失败点                | 状态                 | 检索可见性                         | 后续                                     |
| ------------------ | ------------------ | ----------------------------- | -------------------------------------- |
| ⑥ tombstone 写失败    | 无变化（文件未改）          | 全部可见                          | operationStatus='failed_retriable'，重试⑥ |
| ⑦ A 层写失败           | tombstone 已生效      | **全部不可见**（tombstone 拦）        | partial，重试⑦                            |
| ⑧ ShareRequest 写失败 | tombstone + A 层已生效 | 副本不可见（tombstone + status 双保险） | partial，重试⑧                            |
| ⑨ audit 失败         | 业务已完成              | 不可见                           | committed + auditWarn                  |
| ⑪ 验证失败             | 同对应 effect 失败      | —                             | partial，重试                             |

**关键**：⑥ 一旦成功，即使后续全失败，**删除在检索上已生效**。这是把 tombstone 放第一位的原因。

## 8.4 重试是否幂等

```
重试⑥：tombstone 可能已存在 → 按 memoryId 查重，已存在则跳过（幂等）
重试⑦：A 层记录可能已 deleted → 检查 status，已是 deleted 则跳过
重试⑧：request 可能已 revoked → 检查 approvalStatus，已是 revoked 则跳过

★ 每个 effect 的执行都是"幂等设置目标状态"而非"增量修改"
```

## 8.5 删除期间新建的 ShareRequest 如何处理

```
场景：删除事务执行中（tombstone 已写，A 层未改），有新的 share_request 到达

处理：
  ① share_request 的权限检查第③步会查 record.status
     此时 record.status 可能仍是 'active'（A 层未改）
  ② 但必须额外查 tombstone：
     若 targetIds 中有 tombstone.state='active' → 拒绝
     → 返回 {code:'SOURCE_BEING_DELETED'}
③ 所有分享/写入操作都必须先查 tombstone（不只是删除路径）
```

**这是必须补的校验**：V2.4 只在写入和检索查 tombstone，分享申请没查。

## 8.6 删除期间新建的副本如何阻止

```
副本创建（share_copy 事务）：
  effect 1 写副本前，必须检查：
    ① 原件 status === 'active'
    ② 原件不在 tombstone（state='active'）
    ③ 原件 revision === request.sourceRevision
  任一不符 → 拒绝创建
```

## 8.7 已撤销副本再次删除

```
场景：副本已被 revoke（status=deleted + tombstone），用户又发起删除

处理：
  ① 删除流程第③步：查 ShareRequest 时只看 approvalStatus='approved' 的
     → 已 revoked 的不在列表里
  ② 第④步收集副本 → 不含已删的
  ③ 第⑥步写 tombstone → 已存在，幂等跳过
  ④ 第⑦步改 A 层 → 已 deleted，跳过
  → 结果：只有原件被删，已删副本不受影响 ✅
```


## 8.8 并发：阻止同时产生新分享

```
必须保证：
  删除原件 与 新建 ShareRequest/副本 互斥

实现：
  ① 都在同一个 bridge 写入队列（withWriteLock）
  ② 删除事务在锁内完成 12 步
  ③ 分享事务也在锁内
  → 天然串行，不会交错

若分享先执行（锁内）：
  → 分享完成 → 副本存在 → 删除事务会收集到它 → 级联删 ✅
若删除先执行：
  → tombstone 已写 → 后续分享被 8.5 的检查拒绝 ✅
```

## 8.9 部分完成的删除如何在检索端保持不可见

```
关键：tombstone 是检索第③层（见 §14）
  即使 A 层 status 还是 active，tombstone 会拦

风险：内存 tombstone 索引
  若 tombstone 文件写了但内存索引没更新（崩溃）
  → 重启时从文件重建索引
  → 重建完成前，检索不可用（启动期）
  → 不暴露不一致状态
```

---

# 九、recovery_blocked / B 层恢复 / 向量清除（P0-5）

## 9.1 recovery_blocked 最终定义

`[已确定]`

```
状态语义：
  operationStatus = 'recovery_blocked'
  
行为约束：
  · 自动重试 = 禁止（不进重试队列）
  · 自动执行 = 终止（该 op 不再被任何定时器处理）
  · 审计保留 = 是（WAL 记录永久保留，标 blocked）
  · 重新提交 = 新 opId + 新 memoryId（不复用）

不标为 failed_retriable：
  failed_retriable 会被自动重试
  recovery_blocked 不会
```

**控制台"重新提交"入口**：

```
① 控制台显示 recovery_blocked 列表
② 点击"重新提交" → 弹窗要求主人提供 content（因为原文已丢）
③ 提供后 → 创建新 op（新 opId + 新 memoryId）
④ 旧 op 保持 recovery_blocked 终态

★ 入口只能创建新 operation，不能复用旧 opId
```

## 9.2 B 层恢复的完整校验（V2.4 缺失）

`[已确定]` B 层槽位必须带 `slotRevision` + `contentHash`。

**恢复 A 层正文时（B 层有、A 层无）的六项校验**：

```
① slotId 正确
   B 层槽位存在，slotId 与 WAL payload 记录的一致

② lastOperationId 对应
   slot.lastOperationId === WAL.opId
   ★ 若不等于 → 说明该槽位已被后续操作覆盖 → 不能作为恢复源

③ contentHash 一致
   slot.contentHash === WAL.payload.contentHash
   ★ 若不符 → 内容已变 → 不能作为恢复源

④ slotRevision 符合预期
   slot.slotRevision === WAL.payload.expectedSlotRevision
   ★ 若不符 → 槽位被改过 → 不能作为恢复源

⑤ 槽位未被更新为另一条记忆
   slot.aMemoryId === WAL.payload.memoryId（或都是 null）
   ★ 若指向别的 memoryId → 不能作为恢复源

⑥ 槽位当前 text 的 normalize 后 hash === contentHash
   防止"文件被手工改过但 hash 字段没改"
```

**六项任一失败 → 进入冲突处理，不猜正文**：

```
冲突处理：
  ① operationStatus = 'recovery_blocked'
  ② 写 failures：{opId, reason: 'b_slot_conflict', detail: {哪一项失败}}
  ③ 控制台显示"B 层槽位已变化，无法恢复 A 层正文"
  ④ 等人工/重新提交
```

**恢复成功后**：

```
① 用 slot.text 作为 A 层 content
② 计算 contentHash（应与 ⑥ 一致）
③ 调用 embed(content) 重算向量
④ 写入 A 层记录（新 revision）
⑤ effect status='done'
```

## 9.3 向量清除的 schema（最终）

`[已确定]`

```
contentPurged=false:
  content      = 真实正文
  vec.length   = 1024
  vecPurged    = false
  vecModel     = 'bge-m3'
  contentHash  = sha256(normalize(content))

contentPurged=true:
  content      = '[deleted]'
  vec          = []
  vecPurged    = true
  vecModel     = null
  contentHash  = 【原始正文的 hash，保持不变】

★ 绝不用 '[deleted]' 重算 contentHash 覆盖
★ 检索时：只有 contentPurged=false 且 vec.length===1024 才参与打分
```

## 9.4 恢复操作与 contentPurged

| 场景                                      | 能否 force_restore | 处理                                                                                                       |
| --------------------------------------- | ---------------- | -------------------------------------------------------------------------------------------------------- |
| contentPurged=false                     | ✅                | 正常恢复（status → active）                                                                                    |
| contentPurged=true（正文已清）                | ⚠️ **需重新提供正文**   | ① 恢复 status=active<br />② content 仍是 '[deleted]'<br />③ **禁止参与检索**（因为没有正文）<br />④ 需主人/模型重新提交内容 → 走"补充"流程 |
| contentPurged=true + tombstone restored | 同上               | 同上                                                                                                       |

**关键规则**：

```
contentPurged=true 的记录：
  · 可以被 force_restore（status → active）
  · 但【永远不参与检索】（因为 content='[deleted]' 无意义）
  · 检索过滤加一条：if (contentPurged) return EXCLUDE
  
  除非通过"补充正文"操作：
    qq_memory_manage(op='supplement_content', memoryId, content)
    → content 更新，contentPurged=false，vec 重算
    → 才能参与检索
```

---

# 十、tombstone 最终规则（沿用 V2.4，补充）

## 10.1 三状态

```
active     该 ID 已删除，禁止复活
restored   曾被 force_restore，保留审计，不再阻止该 ID
purged     tombstone 本身被清理（极少，需审计）
```

## 10.2 检索拦截位置

`[已确定]` **检索第③层**（权限之后、生命周期之前）。

```js
// 伪代码
for (const record of candidates) {
  // ① 权限
  if (!checkPermission(record, requester)) continue;
  // ② 
  // ③ tombstone
  const ts = tombstoneIndex.get(record.id);
  if (ts && ts.state === 'active') continue;   // 排除
  // ④ 生命周期
  ...
}
```

## 10.3 写入顺序（与删除流程一致）

```
删除时：
  ① 写 tombstone（state=active）
  ② 更新内存索引
  ③ 改 A 层 status=deleted
```

## 10.4 force_restore

```
① tombstone.state: 'active' → 'restored'，写 restoredAt/restoredBy
② A 层 status: 'deleted' → 'active'，revision+1，deletedAt 保留
③ 顺序：先 A 层（恢复 status），再 tombstone（否则窗口期不可见——但两个顺序都安全，见 §6.5 分析）
④ 写 audit

★ 恢复后 tombstone 仍保留（state='restored'），不再阻止该 ID
```

---

# 十一、CRC 与 normalize（P1-6 / P1-7）

## 11.1 normalize（最终定义）

`[已确定]`

```js
function normalize(content) {
  return String(content ?? '')
    .normalize('NFC')                 // Unicode NFC 规范化
    .replace(/\r\n?/g, '\n')          // 统一换行符为 \n
    .replace(/[ \t]+/g, ' ')          // 连续空格/制表符 → 单个空格
    .replace(/\n{3,}/g, '\n\n')       // 3+ 连续换行 → 2 个
    .trim();                          // 去首尾空白
}
```

**明确不做的事**：

```
❌ 不删除数字        → '[GPU]' 与 '[GPU]' 保持不同 ✅
❌ 不删除标点        → '喜欢'与'喜欢！'保持不同
❌ 不删除否定词      → '不喜欢'不会变成'喜欢'
❌ 不做同义词替换
❌ 不删除设备型号/版本号
❌ 不转大小写        → 英文大小写保留（型号区分）
```

**为什么这样**：

```
V2.2/V2.3 的 `findMemoryRow` 第 4 级归一化删了所有数字（memory-vector.js:251）
→ '[GPU]' 和 '[GPU]' 会撞车
→ 这是已确认的真实缺陷
→ V2.4.1 的 normalize 明确不删数字
```

## 11.2 contentHash

```
contentHash = sha256(normalize(content)) 的完整 64 位 hex

为什么用完整 64 位：
  截断会有碰撞风险；64 字符成本可忽略

hash 命中后的二次确认：
  即便 contentHash 相同，也要比对 normalize(content) 全等
  → 防 hash 碰撞（极罕见但要有）

purged 后：
  contentHash 保持原值（不重算）
  用于 tombstone 关联
```

## 11.3 \_crc 规范

`[已确定]`

```js
// 写入时
function computeCrc(record) {
  const { _crc, ...rest } = record;
  const canonical = JSON.stringify(rest, Object.keys(rest).sort());
  return crc32(canonical).toString(16).padStart(8, '0');
}

// 读取时
function verifyCrc(line) {
  const record = JSON.parse(line);
  if (!record._crc) return { status: 'legacy' };       // 旧数据
  const expected = computeCrc(record);
  if (record._crc !== expected) return { status: 'corrupt' };
  return { status: 'ok' };
}
```

**关键点**：

```
① 计算时排除 _crc 字段本身
② 序列化用【键名字典序】保证确定性（sort keys）
③ CRC32（不是 SHA256）—— 只做意外损坏检测，不做安全认证
④ _crc 缺失 = legacy（旧数据），不是损坏
⑤ _crc 不符 = 损坏（写路径拒绝，读路径跳过+日志）
```

**明确声明**：`_crc` **不是**防篡改认证。恶意修改者可以同时改内容和 \_crc。它只检测"传输/写入过程中的意外损坏"。

## 11.4 读取三档

| 档位 | 函数               | legacy | corrupt        | 用在哪  |
| -- | ---------------- | ------ | -------------- | ---- |
| 宽松 | `loadRowsSafe`   | 接受     | 跳过+日志          | 检索   |
| 严格 | `loadRowsStrict` | 接受     | **抛错**         | 写操作  |
| 修复 | `compactFile`    | 接受     | 隔离到 quarantine | 手工维护 |

## 11.5 Windows 文件操作限制

| 操作          | 限制                   | 应对                               |
| ----------- | -------------------- | -------------------------------- |
| `rename` 覆盖 | 目标被打开时 EBUSY/EPERM   | `renameWithRetry`（4 次，40ms 递增等待） |
| `fsync`     | 可用（FlushFileBuffers） | 正常调用                             |
| 目录 fsync    | **不支持**              | 跳过                               |
| `wx` 原子创建   | ✅                    | 用于 writer.lock                   |
| flock       | ❌ 无                  | 不用                               |
| 删除被打开文件     | EBUSY                | 重试或留垃圾（无害）                       |

---

# 十二、检索过滤最终顺序（V2.4.1）

`[已确定]` 12 层，顺序不可换。

```
① 请求者身份验证
   token → conversationKey + ownerId
   失败 → 403

② 查询模式校验
   agentToken 只能用"自动注入"或"历史检索"
   不能用"管理查询"级别的参数
   违反 → 403

③ 权限硬过滤（先于任何计算）
   ownerId 匹配 + visibility 规则（见 §13）

④ tombstone 检查
   tombstoneIndex.get(record.id)?.state === 'active' → 排除

⑤ contentPurged 检查
   record.contentPurged === true → 排除（无正文不可检索）

⑥ 生命周期过滤
   排除 archived / deleted
   includeSuperseded 时放开 superseded

⑦ TTL 过滤
   expiresAt ≠ null && expiresAt < now → 排除

⑧ reviewStatus 过滤
   排除 pending / rejected

⑨ isCandidate / conflictGroupId 过滤
   排除 isCandidate=true
   排除 conflictGroupId ≠ null

⑩ 分享副本有效性（仅群上下文）
   record.source === 'share_copy'：
     查 ShareRequest：approvalStatus 必须 'approved'
     ★ 不看 freshnessStatus

⑪ 语义相似度
   仅对 vec.length===1024 计算

⑫ 加权排序 + topK + 裁剪
   score × timeWeight × typeWeight
```

**三种查询模式**：

| 模式         | ⑥⑦⑧⑨ 放开情况       |
| ---------- | --------------- |
| 自动注入（模型默认） | 全不放开            |
| 历史检索（模型显式） | ⑥ 放开 superseded |
| 管理查询（控制台）  | ⑥⑦⑧⑨ 全放开        |

---

# 十三、权限矩阵（最终）

| #  | 请求者  | 目标 ownerId | visibility      | 额外条件                                   | 允许         |
| -- | ---- | ---------- | --------------- | -------------------------------------- | ---------- |
| 1  | 私聊 A | `owner:A`  | `owner_private` | active+approved                        | ✅          |
| 2  | 私聊 A | `owner:A`  | `group_shared`  | —                                      | ❌ 不存在此状态   |
| 3  | 私聊 A | `owner:B`  | 任意              | —                                      | ❌          |
| 4  | 私聊 A | `group:X`  | 任意              | —                                      | ❌          |
| 5  | 群 X  | `group:X`  | `group_shared`  | ShareRequest.approvalStatus='approved' | ✅          |
| 6  | 群 X  | `group:X`  | `owner_private` | —                                      | ❌          |
| 7  | 群 X  | `group:X`  | `group_shared`  | approvalStatus≠approved                | ❌          |
| 8  | 群 X  | `group:Y`  | 任意              | —                                      | ❌          |
| 9  | 群 X  | `owner:A`  | 任意              | —                                      | ❌          |
| 10 | 控制台  | 任意         | 任意              | 任意                                     | ✅（写 audit） |
| 11 | 机器人  | 任意         | 任意              | —                                      | ❌（无管理权）    |

**补充规则**：

```
原件恢复后，共享副本不自动恢复（副本已 revoked，需重新申请）
副本撤销后，原件不受影响（原件 owner_private，独立）
删除原件后，所有副本按级联策略撤销（§8）
stale 副本在 approvalStatus=approved 时可检索 ✅
revoked 副本不可检索 ❌
```

---

# 十四、writer.lock 单写者（沿用 V2.4）

## 14.1 锁结构

```json
{
  "pid": 12345,
  "instanceId": "inst_a1b2c3d4",
  "createdAt": 1793400000000,
  "mode": "bridge",
  "purpose": "启动"
}
```

## 14.2 bridge 启动

```
① wx 创建 lock
   ├─ 成功 → 写入 → 继续启动
   └─ EEXIST → 读锁
              ├─ 解析失败 → 退出（提示人工）
              ├─ mode='bridge' → 退出（已有实例）
              └─ mode='maintenance' → 退出（维护中）
```

## 14.3 维护脚本

```
① 读 lock
   ├─ 存在 mode='bridge' → 拒绝
   ├─ 存在 mode='maintenance' → 拒绝
   ├─ 解析失败 → 拒绝
   └─ 不存在 → wx 创建（mode='maintenance'）
```

## 14.4 崩溃后

```
启动时发现 mode='bridge' 的残留锁：
  ① 检查 pid 存活
     ├─ 存活 → 退出
     └─ 不存活 → 仍退出，提示：
        "发现上次异常退出的锁（pid=X）。请人工确认后删除 state/writer.lock。
         删除前请检查 state/wal/ 是否有未完成操作。"
  ② 不自动清理（防 PID 复用误判 + 可能遗留未完成操作）
```

---

# 十五、实施批次（重排，修正 P4/P6 依赖）

## 15.1 依赖图

```
P0 基础设施（writer.lock / tmp / normalize / CRC）
 └─► P1 数据模型（字段 / 双读 / 权限硬过滤 / 基础检索）
      └─► P2 WAL 基础（effects / revision / generation / 恢复）
           ├─► P3 事实管理（factKey / cardinality / 去重 / 两阶段）
           └─► P4 生命周期与分享基础（tombstone / 删除 / 归档 / ShareRequest / 副本撤销）
                └─► P5 检索优化（TTL / 加权 / 历史）
                     └─► P6 跨层关联与 UX（slotId / bRefs / 同步 / 管理工具）
```

**关键修正**：ShareRequest 基础模型 + 副本撤销从 P6 提前到 **P4**。  
理由：P4 的删除要级联撤销副本，必须有 ShareRequest 能力。不能等 P6。

## 15.2 各批次详情

### P0 基础设施

| 项    | 内容                                                                       |
| ---- | ------------------------------------------------------------------------ |
| 代码   | 新增 `writer-lock.js`；修 `atomicRewrite` tmp 名；新增 `normalize.js` / `crc.js` |
| 数据格式 | 无变化                                                                      |
| 验收   | ① 并发 100 写不丢 ② bridge 与脚本互斥 ③ `_crc` 检出篡改 ④ normalize 不删数字               |
| 回滚   | 覆盖代码（无数据变化）                                                              |

### P1 数据模型

| 项    | 内容                                                                |
| ---- | ----------------------------------------------------------------- |
| 代码   | 字段补齐；`normalizeRow` 双读；检索加 ③④⑤⑥⑧⑨ 基础层（④⑤预留空实现）；`findMemoryRow` 修正 |
| 数据格式 | 新记录多字段；旧记录补默认值                                                    |
| 验收   | 新记录字段齐；旧记录可读；pending/archived/deleted 不检索                         |
| 回滚   | 覆盖代码（只加字段，旧代码忽略）                                                  |
| 回滚数据 | 不需要                                                               |

### P2 WAL 基础

| 项      | 内容                                                                                                       |
| ------ | -------------------------------------------------------------------------------------------------------- |
| 代码     | `memory-wal.js`；effects 数组；revision 校验；generation+MANIFEST；恢复协议                                          |
| 数据格式   | 新增 `state/wal/` 目录                                                                                       |
| 验收     | ① 5 种操作的 effects 正确 ② 崩溃恢复验证 contentHash ③ revision 冲突拒绝 ④ compact 崩溃可判定                                 |
| **回滚** | ⚠️ **不能删 WAL**：<br />① 检查所有 op 是否终态<br />② 有非终态 → 必须先完成或人工 cancelled<br />③ 全部终态 → 保留 WAL 文件<br />④ 覆盖代码 |
| 人工     | 非终态无法处理 → 停服人工检查                                                                                         |

### P3 事实管理

| 项        | 内容                                                                                                                    |
| -------- | --------------------------------------------------------------------------------------------------------------------- |
| 代码       | 模板解析；`fact-keys.json`；entityGroupKey；精确去重；两阶段更新；conflictGroup                                                         |
| 数据格式     | 新记录带 factKey/template/版本；新增 `state/fact-keys.json`                                                                    |
| 验收       | ① 四个设备/关系场景正确 ② 两阶段旧事实可见 ③ 批准原子切换 ④ 模板歧义检测                                                                            |
| **回滚代码** | 覆盖 + 删 `fact-keys.json`                                                                                               |
| **回滚数据** | ⚠️ superseded → active（保留 supersededAt）；isCandidate 记录 → 移到 `state/memory-candidates-archive.jsonl`；清 conflictGroupId |
| 人工       | pending 候选需人工决定                                                                                                       |


### P4 生命周期与分享基础

| 项        | 内容                                                                                                                                |
| -------- | --------------------------------------------------------------------------------------------------------------------------------- |
| 代码       | tombstone；逻辑删除；L2；归档；**ShareRequest 基础模型**；**share_copy**；**revoke_share**；级联删除；audit                                             |
| 数据格式     | 新增 `state/memory-tombstones.jsonl`、`state/share-requests.jsonl`                                                                   |
| 验收       | ① 级联删除完整 ② tombstone 检索拦截 ③ 主人在私聊授权分享 ④ 群聊不能发起 ShareRequest ⑤ revoke 正确                                                           |
| **回滚代码** | 覆盖                                                                                                                                |
| **回滚数据** | ⚠️ tombstone 必须保留（不删）；deleted 记录 → 移到 `state/memory-deleted-archive.jsonl`；share_copy 记录 → 移到 `state/memory-copies-archive.jsonl` |
| 人工       | 已分享内容需人工处理                                                                                                                        |

### P5 检索优化

| 项    | 内容                                  |
| ---- | ----------------------------------- |
| 代码   | TTL 过滤；timeWeight/typeWeight；历史检索参数 |
| 数据格式 | 无变化                                 |
| 验收   | ① 过期不注入 ② 排序合理 ③ 历史检索能取 superseded  |
| 回滚   | 覆盖代码                                |

### P6 跨层关联与 UX

| 项      | 内容                                                             |
| ------ | -------------------------------------------------------------- |
| 代码     | slotId；slotRevision；bRefs；aMemoryId；同步工具；重新提交界面；维护工具           |
| 数据格式   | B 层加 slotId/slotRevision/contentHash/aMemoryId/lastOperationId |
| 验收     | ① A/B 互链 ② 删除时 warnings ③ B 层恢复六项校验 ④ 重新提交入口                   |
| **回滚** | 清空 bRefs/aMemoryId（记录哪些被清，便于恢复）                                |
| 人工     | —                                                              |

## 15.3 并行约束

```
默认串行：P0 → P1 → P2 → P3 → P4 必须串行验收

可并行（需明确文件所有权）：
  · P5 与 P3/P4（不同代码路径，但都依赖 P1/P2）
  · P6 内部（slotId 与 bRefs 可以分两次）

禁止：
  ✗ 把 P3+P4 合并成一次大改（回滚无法分离）
  ✗ 跳过 P2 直接做 P3（verification 需要 WAL）

不允许为了缩短时间合并阶段
```

---

# 十六、45 个异常场景预演

> 格式：初始 → 触发 → WAL/effects → 资源变化 → 检索可见性 → 崩溃恢复 → 最终

## 分享授权（S1-S8）

**S1｜主人私聊授权分享**

- 初始：`mem_orig`（owner:A, private, active+approved, rev=3）
- 触发：主人私聊 `qq_memory_share_request(sourceMemoryId=mem_orig, targetGroupId=X)`
- WAL：`kind='share_request'`，effects=`[share_requests: create pending]`
- 资源：`sr_1`（pending, authorizedBy=owner:A, current）
- 检索：无副本，不可见（正常）
- 崩溃恢复：request 已写 → 保持 pending
- 最终：等控制台审批

**S2｜群聊机器人尝试访问个人原件**

- 初始：`mem_orig`（owner:A, private）
- 触发：群 X 的 agentToken 调 `qq_memory_query(key='private:A')`
- WAL：无（在鉴权层被拒）
- 资源：无变化
- 检索：403 `agentTokenOk` 失败
- 崩溃：无
- 最终：拒绝，审计可疑行为

**S3｜未授权的 share_request**

- 初始：`mem_orig`（owner:A）
- 触发：群 X 发起 `share_request(sourceMemoryId=mem_orig)`
- WAL：无（步骤①拦）
- 资源：无
- 检索：403 `GROUP_CANNOT_SHARE_PRIVATE`
- 最终：拒绝

**S4｜两个群分别申请同一原件**

- 初始：`mem_orig`（owner:A）
- 触发：主人授权到群 X 和群 Y
- WAL：两个独立 op，各 create 一个 request
- 资源：`sr_1`(group:X, pending), `sr_2`(group:Y, pending)
- 检索：都无副本
- 最终：两个独立审批

**S5｜一群批准、一群拒绝**

- 初始：`sr_1`, `sr_2` 都 pending
- 触发：批准 sr_1，拒绝 sr_2
- WAL：op1 `share_copy`（effects: vectors create + share_requests approve）；op2 `reject`
- 资源：`mem_copy_1`(group:X, active)；`sr_1`(approved, current)；`sr_2`(rejected)
- 检索：群 X 可见 mem_copy_1；群 Y 不可见
- 崩溃恢复：op1 若 partial → 重试补
- 最终：群 X 有内容，群 Y 无

**S6｜原件更新后副本 stale**

- 初始：`mem_orig` rev=3，`mem_copy_1`（sourceRevision=3），`sr_1`(approved, current)
- 触发：编辑原件 → rev=4
- WAL：`kind='edit'`，effects=`[memory_vectors: update mem_orig;  share_requests: freshnessStatus→stale]`
- 资源：`sr_1`(approved, **stale**)；`mem_copy_1` 内容不变
- 检索：群 X **仍可检索** mem_copy_1（approvalStatus=approved）
- 崩溃恢复：若 share_requests 更新失败 → partial → 重试
- 最终：副本可见但标 stale

**S7｜stale 副本继续检索**

- 初始：`sr_1`(approved, stale)
- 触发：群 X 检索
- WAL：无
- 检索：第⑩层查 approvalStatus=approved → **通过** ✅
- 最终：返回副本，结果里标"基于旧版本(rev 3)"

**S8｜用户撤销分享**

- 初始：`sr_1`(approved)，`mem_copy_1`(active)
- 触发：主人 `revoke_share(sr_1)`
- WAL：`kind='revoke_share'`，effects=`[tombstones: create copy_1; vectors: copy_1→deleted; share_requests: sr_1→revoked]`
- 资源：`mem_copy_1`(deleted)；tombstone(copy_1, active)；`sr_1`(revoked)
- 检索：群 X 不可见（tombstone + status 双保险）
- 崩溃恢复：若 vectors 失败但 tombstone 成功 → 仍不可见 → 重试补齐
- 最终：副本撤销，原件不受影响

## 删除与级联（S9-S14）

**S9｜用户删除原件**

- 初始：`mem_orig`(rev=5)，`mem_copy_1`(rev=2)，`sr_1`(approved)
- 触发：删除原件
- WAL：`kind='cascade_delete'`，4 个 effects（tombstone→vectors→share_requests→audit）
- 资源：tombstone(orig+copy_1)；两者 deleted；`sr_1`(revoked)
- 检索：原件、副本都不可见
- 崩溃恢复：若 tombstone 成功但 vectors 失败 → 仍不可见 → 重试
- 最终：全部不可见

**S10｜删除时存在多个副本**

- 初始：3 个副本（group X/Y/Z）
- 触发：删除原件
- WAL：effects 的 targetIds 含全部 4 个 id
- 资源：4 个 tombstone + 4 个 deleted + 3 个 request revoked
- 检索：全不可见
- 最终：一次事务完成

**S11｜删除过程中有新 share_request**

- 初始：删除事务进行中（tombstone 已写，vectors 未改）
- 触发：主人又发起分享申请
- WAL：新 request 被拒（在权限检查第③步查 tombstone）
- 资源：无新 request
- 检索：—
- 最终：拒绝 `SOURCE_BEING_DELETED`

**S12｜副本重复删除**

- 初始：`mem_copy_1` 已 deleted + tombstone
- 触发：再次删除
- WAL：effect 幂等（tombstone 存在则跳过，status 已 deleted 则跳过）
- 资源：无变化
- 检索：不可见
- 最终：成功（幂等）

**S13｜ShareRequest 成功、副本写入失败**

- 初始：`sr_1` pending
- 触发：批准
- WAL：effects order 1(vectors create copy) 失败 → order 2(share_requests) 不执行
- 资源：无副本；`sr_1` 仍 pending
- 检索：无副本可见
- 崩溃恢复：partial → 重试 order 1
- 最终：重试成功后完成

**S14｜副本写入成功、ShareRequest 失败**

- 初始：同上
- 触发：effects order 1 成功，order 2 失败
- WAL：partial
- 资源：副本存在（active），`sr_1` 仍 pending
- 检索：**副本不可见**（第⑩层查 request 不是 approved）✅
- 崩溃恢复：重试 order 2
- 最终：重试成功后可见

## WAL 与恢复（S15-S20）

**S15｜WAL effects 部分完成**

- 初始：op 有 3 个 required effects
- 触发：effect 1 done，2 failed，3 pending
- WAL：operationStatus='partial'
- 恢复：重启扫 WAL → effect 1 verify 通过保留 done → 重试 2、3
- 最终：全 done 后 committed

**S16｜tombstone 写成功、A 层失败**

- 初始：删除事务
- 触发：effect 1(tombstone) done，effect 2(vectors) failed
- WAL：partial
- 资源：tombstone 有效；`mem_orig.status` 仍 active
- 检索：**不可见**（tombstone 第④层拦）✅
- 崩溃恢复：重试 effect 2
- 最终：补齐后 committed

**S17｜A 层成功、ShareRequest 失败**

- 初始：删除事务
- 触发：effect 2 done，effect 3 failed
- WAL：partial
- 资源：原件 deleted；`sr_1` 仍 approved
- 检索：副本不可见（status=deleted + tombstone）✅
- 崩溃恢复：重试 effect 3
- 最终：补齐

**S18｜audit 失败**

- 初始：删除事务，前 3 个 required done
- 触发：effect 4(audit, optional) failed
- WAL：committed + `auditWarn: true`
- 资源：业务完成；audit 缺失
- 检索：不可见
- 最终：committed，控制台红标提示补 audit

**S19｜WAL compact 中途崩溃**

- 初始：gen 7，正在 compact 到 8
- 触发：写完 wal-0008 后崩溃（MANIFEST 未更新）
- WAL：MANIFEST activeGen=7，pendingGen=null，存在孤儿 wal-0008
- 恢复：清理孤儿 8，用 7
- 最终：数据完整

**S20｜MANIFEST 两阶段切换失败**

- 初始：MANIFEST 第 1 次写成功（activeGen=7, pendingGen=8），第 2 次失败
- 触发：重启
- 恢复：检查 gen 8 文件完整 → 补写 activeGen=8；不完整 → 回滚用 7
- 最终：一致

## 并发与 revision（S21-S24）

**S21｜revision 并发冲突**

- 初始：`mem_X` rev=5
- 触发：操作 A（expected=5）成功→6；操作 B（expected=5）到达
- WAL：B 的 effect 校验失败 → `failed` + `lastError=CONFLICT_REVISION`
- 资源：`mem_X` 保持 A 的结果
- 检索：正常
- 最终：B 返回冲突，需重读

**S22｜两管理员并发批准不同候选**

- 初始：`mem_O`(active, rev=3)，候选 B、C(pending, rev=1)
- 触发：管理员1 批准 C；管理员2 批准 B
- WAL：op1（O→superseded rev4, C→approved rev2, **B→rejected rev2**）  
  op2（B expected rev=1 但实际 2）→ 冲突
- 资源：C approved，B rejected
- 检索：只返回 C
- 最终：op2 失败，提示"B 已被其他审核处理"

**S23｜候选组已关闭却收到迟到批准**

- 初始：`cg_1` 已关闭（C approved，B rejected，conflictGroupId=null）
- 触发：迟到的批准 B
- WAL：B 的 expect `{reviewStatus:'pending'}` 但实际 `rejected` → 失败
- 资源：无变化
- 最终：拒绝 `CANDIDATE_ALREADY_RESOLVED`

**S24｜旧事实已删除、迟到 supersede**

- 初始：`mem_O` 已 deleted（rev=6）
- 触发：迟到的 approve（expected rev=3）
- WAL：expect 失败
- 资源：无变化
- 检索：O 不可见
- 最终：拒绝 `CONFLICT_REVISION`

## 恢复与正文（S25-S30）

**S25｜recovery_blocked**

- 初始：op append，aStatus=pending, bStatus=failed，WAL 无正文
- 触发：崩溃恢复，两层都查不到 contentHash 对应记录
- WAL：operationStatus='recovery_blocked'
- 资源：无记录创建
- 检索：无
- 崩溃恢复：不再自动重试
- 最终：控制台可见，需重新提交

**S26｜A 有正文、B 无正文**

- 初始：op append，A 已写，B 未写
- 触发：恢复
- WAL：读 A 层 → contentHash 匹配 → 取正文
- 恢复动作：用 A 的 content 补写 B 层槽位
- 最终：committed

**S27｜B 有正文但 slotRevision 不匹配**

- 初始：op append，A 未写，B 已写但槽位被后续改过（slotRevision 2 vs 期望 1）
- 触发：恢复
- 校验：① slotId ✅ ② lastOperationId ❌（被新 op 覆盖）
- WAL：operationStatus='recovery_blocked'，reason='b_slot_conflict'
- 最终：不猜正文，等人工

**S28｜A/B 都没有正文**

- 初始：op append，两层都 pending，WAL 无正文
- 触发：恢复
- WAL：'recovery_blocked'
- 最终：需重新提交

**S29｜向量缺失但正文存在**

- 初始：`mem_X` 存在但 vec=[]（vecPurged=true 或 embed 曾失败）
- 触发：恢复需要向量
- 恢复动作：读 content → embed() → 更新 vec + vecModel，rev+1
- 若 embed 失败：failed_retriable（下次再试）
- 最终：补齐向量

**S30｜contentPurged 后恢复**

- 初始：`mem_X`(deleted, contentPurged=true, content='[deleted]', vec=[])
- 触发：force_restore
- WAL：`kind='force_restore'`，effects=`[vectors: status→active; tombstones: state→restored]`
- 资源：`mem_X`(active, **contentPurged 仍 true**)
- 检索：**不可见**（第⑤层 contentPurged 过滤）✅
- 最终：status 恢复但无正文，需"补充正文"才能检索

## tombstone 与重新授权（S31-S33）

**S31｜tombstone 已 restored 后迟到旧操作**

- 初始：`mem_X` 被删 → tombstone(active) → force_restore → tombstone(restored)
- 触发：迟到的删除 op（expected rev=5，实际 rev=7）
- WAL：revision 冲突 → 失败
- 资源：`mem_X` 保持 active
- 检索：可见
- 最终：迟到操作被拒

**S32｜content_suppression 拦截重新写入**

- 初始：tombstone(contentHash='abc', suppression='content_suppression')
- 触发：写入 normalize 后 hash='abc' 的新内容
- WAL：写入前检查 → 拦截
- 资源：无新记录
- 检索：无
- 最终：拒绝 `CONTENT_SUPPRESSED`

**S33｜用户重新授权同一内容**

- 初始：tombstone(memory_only, contentHash='abc')
- 触发：写入 hash='abc' 的新内容
- WAL：检查 suppression=memory_only → 允许
- 资源：新记录 `mem_new`；tombstone 追加 `reAuthorized: [{newMemoryId:'mem_new'}]`
- 检索：返回 mem_new（旧的 deleted）
- 最终：成功

## B 层（S34-S35）

**S34｜B 层槽位改名**

- 初始：slot_a1b2(text='旧话题', slotId='slot_a1b2')
- 触发：memory-update 改 text
- 资源：`slot_a1b2`(text='新话题', slotRevision+1, contentHash 更新)
- 关键：**slotId 不变**，所以 bRefs 仍有效 ✅
- 检索：A 层 bRefs 指向 slot_a1b2 仍能找到

**S35｜slotId 对应内容被更新**

- 初始：A 层 bRefs 指向 slot_a1b2（内容是"话题 A"）
- 触发：该槽位被改成完全不同的"话题 B"
- 资源：slotRevision+1，contentHash 变
- 影响：A 层与 B 层语义脱节
- 处理：P6 的 prune_refs 检测到 contentHash 不匹配 → 标记 dangling → 报警
- 最终：引用保留但不自动同步

## 事实管理（S36-S40）

**S36｜同 factKey 多 active**

- 初始：两条 active 同 factKey（数据异常，如迁移导致）
- 触发：检索
- 资源：都打 `conflictGroupId=cg_1`
- 检索：第⑨层整组排除 → 都不可见
- 最终：控制台人工合并

**S37｜不同设备 qualifier 共存**

- 初始：无
- 触发：写 desktop.gpu 和 lab.gpu
- 资源：两条 active，不同 factKey
- 检索：都能返回
- 最终：正常

**S38｜同设备 GPU 升级**

- 初始：desktop.gpu='5060Ti'(active)
- 触发：写 desktop.gpu='5070Ti'
- 资源：新记录 pending+isCandidate+cg_1；旧记录不动
- 检索：**只返回 5060Ti**（旧的仍可见）
- 批准后：旧的 superseded，新的 approved
- 最终：只见 5070Ti

**S39｜同一群友多关系叠加**

- 初始：classmate 关系
- 触发：写 teammate、friend
- 资源：3 条 active（不同 factKey）
- 检索：3 条都返回
- 最终：正常

**S40｜关系结束但模型无法确定**

- 初始：colleague 关系 active
- 触发：主人说"我离职了"
- 处理：模型写 `type='event', factKey=null`（不碰 colleague）
- 资源：新增 event 记录；colleague 仍 active
- 检索：两条都返回
- 最终：人工决定是否 archive colleague

## 其他（S41-S45）

**S41｜临时记忆 TTL 到期**

- 初始：`mem_temp`(type=temporary, expiresAt=now-1h, status=active)
- 触发：检索
- 检索：第⑦层 TTL 过滤 → 排除 ✅
- 资源：status 仍 active（未清理）
- 最终：不注入（不依赖清理任务）

**S42｜CRC 不一致**

- 初始：某行 `_crc` 被改
- 触发：读
- 读路径：跳过该行 + 日志
- 写路径：`loadRowsStrict` 抛错，拒绝写入
- 最终：写被阻断，需修复

**S43｜旧 JSONL 兼容读取**

- 初始：旧格式记录（无 id/status/\_crc，有 needs_review）
- 触发：读
- 处理：`normalizeRow` 补默认值 + 生成确定性 legacyId
- 资源：内存中转换，不写回
- 最终：正常参与检索

**S44｜维护脚本与 bridge 竞争 writer.lock**

- 初始：无锁
- 触发：脚本和 bridge 同时尝试 wx 创建
- 结果：一方成功，另一方 EEXIST → 退出
- 资源：无数据变化
- 最终：互斥成立

**S45｜删除完成后检索缓存仍有旧记录**

- 初始：删除完成（tombstone + status=deleted）
- 触发：检索（内存 rows 缓存可能还有旧数据）
- 处理：**每次检索前检查 cache.mtime**，删除后设 `cache.mtime=0` 强制重载
- 另：检索第④层查 tombstone 内存索引（删除时已更新）
- 检索：不可见 ✅
- 最终：缓存不造成泄漏

---

# 十七、待决策

| #  | 问题                                      | 候选                                         | 影响          |
| -- | --------------------------------------- | ------------------------------------------ | ----------- |
| D1 | 原件删除是否默认级联撤销副本                          | a) 默认级联<br />b) 默认不级联+警告                   | 隐私 vs 群内容连续 |
| D2 | member_relation 的 type 枚举初始值            | 7 个预置 / 可扩展                                | 关系表达覆盖      |
| D3 | recovery_blocked 是否提供控制台重新提交入口          | a) 提供<br />b) 不提供                          | 运维便利        |
| D4 | 群友建议（ShareSuggestion）是否要这个功能            | a) 要（路径 B）<br />b) 不要（只保留路径 A）             | 功能范围        |
| D5 | `bot_persona` 是否允许无审核 supersede         | a) 允许<br />b) 也要审核                         | 安全 vs 效率    |
| D6 | 副本 vec 是否复制原件（不重算）                      | a) 复制<br />b) 重算                           | 成本 vs 质量    |
| D7 | tombstone 是否永久保留                        | a) 永久<br />b) 90 天后 purged                 | 防复活窗口       |
| D8 | topic 的 event/temporary 是否也进 factKey 体系 | a) 进（但 keep_both）<br />b) 不进（factKey=null） | 语义一致性       |

---

# 十八、无法安全闭合

| #  | 问题               | 原因               |
| -- | ---------------- | ---------------- |
| N1 | C 层 Session 隐私擦除 | bridge 无 DSH 写权限 |
| N2 | 历史备份隐私擦除         | 静态文件，改写破坏不可变     |
| N3 | 关系"结束"自动判断       | 需人工              |
| N4 | 事实"对错"判断         | 需人工              |
| N5 | 旧代码读新数据          | 需配合数据迁移          |
| N6 | 恶意进程绕过锁          | 协作式，非强制          |
| N7 | 跨文件严格原子性         | 文件系统不支持          |
| N8 | 两层都无正文的恢复        | 信息已丢失            |
|    |                  |                  |

---

# 十九、交付确认

## A. 相对 V2.4 的实质改动（11 项）

1. **主人授权链**：改为私聊主动授权，群聊只能提交建议（不含正文/memoryId）
2. **approvalStatus / freshnessStatus 拆分**：修复"原件更新后副本不可检索"
3. **统一 effects 状态机**：6 类资源，required/optional，verify 逐条比对
4. **统一删除顺序**：tombstone 永远第一（修正 §6.4/§12.5 矛盾）
5. **committed 判定**：所有 required effects done + verified（不再只看 a/bStatus）
6. **recovery_blocked**：不自动重试，不标 failed_retriable
7. **B 层恢复六项校验**：slotId/lastOpId/contentHash/slotRevision/aMemoryId/text-hash
8. **vecPurged 进字段表** + C9 强联动校验
9. **砍掉 multi_valued**：所有模板 single_current，避免无用的复杂分支
10. **entityGroupKey 模板声明**：程序不猜
11. **C1 笔误修正**：`ownerId = owner:X`（不是 owner:owner:X）

## B. 决策清单（8 项）

见 §十七。最影响实施的是 D1（级联撤销）、D2（关系类型枚举）、D4（是否要群友建议功能）。

## C. 冻结清单

**可冻结（可进入实施）**：数据模型、双层 cardinality + entityGroupKey、factKey 模板、两阶段更新、ShareRequest 授权链、approvalStatus/freshnessStatus、effects 状态机、删除顺序、recovery_blocked、tombstone、CRC/normalize、检索 12 层、writer.lock、批批次划分。

**不可实施（阻塞于决策）**：级联撤销默认行为（D1）、关系 type 枚举（D2）、recovery_blocked UI（D3）、群友建议功能（D4）。

## D. 实施建议

```
具备进入最终实施评审的条件。

前置建议：
  ① 先定 D1、D2（影响 P3/P4 实现）
  ② D3/D4 可在对应批次前再定

实施顺序必须：
  P0 → P1 → P2 → P3 → P4 串行验收
  P5、P6 按依赖安排

不允许：
  ✗ 跳过 P2 做 P3（无 WAL 保护的两阶段更新不可靠）
  ✗ 合并 P3+P4（回滚无法分离）
```

## E. 只读确认

```
✅ 本轮只读：
   - 读取 MemoryV2.4 文档（核实 superseded_by_source 冲突、C1 笔误、§6.4/§12.5 顺序）
   - 未修改任何代码、配置、数据
   - 未修改 Persona / rules / preset / 插件
   - 未生成任何测试记忆
   - 未启动或重启 QQ Bridge / DSH / Ollama
   - 只新增 V2.4.1 设计文档一个文件
```

---

*文档完成，停止。不进入实施阶段。代码实施必须由主人另行明确批准。*
