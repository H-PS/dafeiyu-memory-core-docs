# Memory V2.3 设计修订

- 修订时间：2026-10-31 12:45
- 前置：`MemoryV2.2设计修订-20261031.md`
- 性质：**纯设计文档。** 本轮严格只读：未修改任何代码/配置/数据，未重启服务，未生成测试记忆。
- 标注：`[已确定]` / `[候选方案]` / `[待决策]`

---

# 〇、V2.2 → V2.3 修订对照

| # | V2.2 问题 | 性质 | V2.3 修订 |
|---|---|---|---|
| P0-1 | 通用语法 `<subject>.<entity>.<attribute>.<qualifier>` 与设备模板 `owner.device.<qualifier>.<attr>` 字段顺序冲突 | **规格矛盾** | 改为**模板驱动解析**：先匹配模板，再按模板定义的 role 解释各段。不做统一位置解析 |
| P0-2 | `keep_both` 允许多 active，又说"多 active 是异常" | **逻辑冲突** | 引入 `cardinality`（single_current / multi_valued），异常检测只对 single_current |
| P0-3 | 旧记录立即 superseded，新记录 pending → **两条都不可见** | **功能缺陷（严重）** | 两阶段更新：阶段1旧记录保持 active+approved，阶段2人工批准后原子切换 |
| P0-4 | §1.2 说三时间字段互斥，§4.1 允许 superseded→archived 保留 supersededAt | **自相矛盾** | 引入 `lifecycleEvents` 历史数组；三时间字段改为"曾经发生过"语义，不要求互斥 |
| P0-5 | tombstone 按 contentHash 永久拒绝，阻止用户重新提供同一信息 | **过度限制** | 拆 `suppression: memory_only \| content_suppression`，默认只防复活 |
| P0-6 | hard=true 只清本体，WAL/备份/Session 仍有原文，却暗示彻底删除 | **隐私声明不实** | 三级清除（logical/content/cross_store），明确 cross_store 本轮不实现 |
| P0-7 | 90 天 op-index 无法保护长期状态变更；旧操作可覆盖新状态 | **可靠性缺陷** | 引入 `revision` 乐观锁；区分 retriable / exhausted |
| P0-8 | WAL + op-index 多文件，compact 中途崩溃不一致 | **崩溃窗口** | generation + MANIFEST 原子切换 |
| P0-9 | 分享语义未定（D1 悬空） | **架构空缺** | 采用方案 A（群聊共享副本），完整定义 |
| P0-10 | PID+心跳+claim 仍无法证明原进程已停 | **安全边界不足** | 改为**单写者架构**：bridge 唯一写者，脚本走接口或停服 |
| P1-1 | 用 text 作 B 层 slotKey | 身份不稳定 | 引入 `slotId` |
| P1-2 | 冲突组全员 pending 导致最新记录也不可见 | 同上 P0-3 类问题 | 用 `conflictGroupId`，不改 reviewStatus |
| P1-3 | expiresAt 未纳入检索流程 | 过期泄漏 | 检索第②层加 expiresAt 过滤 |
| P1-4 | `<json>\t<crc32>` 不是合法 JSONL | 兼容破坏 | 改为 JSON 内 `_crc` 字段 |
| P1-5 | 合法 qualifier 区分设备时仍因相似度标 pending | 审核负担 | 只在"真冲突"时 pending |
| P1-6 | P2 回滚=删 WAL 文件 | 危险 | 回滚前必须检查所有 op 状态 |
| P1-7 | "101 条"与"0 字节"矛盾 | 需核实 | **已核实**：101 条来自 10-30 日备份，见 §19 |

---

# 一、完整数据模型

## 1.1 字段表

| 字段 | 类型 | 默认 | 必填 | 说明 | 状态 |
|---|---|---|---|---|---|
| `id` | string | 生成 | ✅ | `mem_<base36>_<rand6>` | `[已确定]` |
| `schemaVersion` | number | 3 | ✅ | 结构版本 | `[已确定]` |
| `revision` | number | 1 | ✅ | **乐观锁版本号**（新增） | `[已确定]` |
| `ownerId` | string | — | ✅ | `owner:<qq>` / `group:<群号>` | `[已确定]` |
| `conversationKey` | string | — | ✅ | 产生上下文 | `[已确定]` |
| `subjectId` | string | `unknown` | ✅ | 内容主体 | `[已确定]` |
| `visibility` | enum | `owner_private` | ✅ | `owner_private` / `group_shared` | `[已确定]` |
| `category` | enum | `unknown` | ✅ | `activeTopic`/`pendingThought`/`memberImpression`/`manual`/`unknown` | `[已确定]` |
| `type` | enum | `event` | ✅ | fact/event/preference/temporary/relationship | `[已确定]` |
| `status` | enum | `active` | ✅ | active/superseded/archived/deleted | `[已确定]` |
| `reviewStatus` | enum | `approved` | ✅ | pending/approved/rejected | `[已确定]` |
| `shareReviewStatus` | enum | `not_requested` | ✅ | not_requested/pending/approved/rejected | `[已确定]` |
| `content` | string | — | ✅ | 正文，无 category 前缀 | `[已确定]` |
| `contentHash` | string | 自动 | ✅ | sha256(normalize(content)) 全 64 位 | `[已确定]` |
| `_crc` | string | 自动 | ✅ | 行校验（JSON 内字段，见 §P1-4） | `[已确定]` |
| `createdAt` | number | now | ✅ | 写入时刻 | `[已确定]` |
| `occurredAt` | number\|null | null | ❌ | 事件发生时刻 | `[已确定]` |
| `updatedAt` | number | now | ✅ | 最后修改 | `[已确定]` |
| `statusChangedAt` | number | now | ✅ | **当前 status 的变更时刻**（新增） | `[已确定]` |
| `lifecycleEvents` | object[] | `[]` | ✅ | **状态变更历史**（新增，见 §六） | `[已确定]` |
| `supersededAt` | number\|null | null | ❌ | 曾经被替代的时刻（语义收窄，见 §六） | `[已确定]` |
| `archivedAt` | number\|null | null | ❌ | 曾经主动归档的时刻 | `[已确定]` |
| `deletedAt` | number\|null | null | ❌ | 逻辑删除时刻 | `[已确定]` |
| `expiresAt` | number\|null | null | ❌ | temporary 的 TTL | `[已确定]` |
| `factKey` | string\|null | null | ❌ | 事实键 | `[已确定]` |
| `factKeyTemplate` | string\|null | null | ❌ | 匹配到的模板 id（新增，便于审计） | `[已确定]` |
| `cardinality` | enum\|null | null | ❌ | 从模板继承：single_current/multi_valued | `[已确定]` |
| `supersedes` | string\|null | null | ❌ | 本记忆替代了谁 | `[已确定]` |
| `supersededBy` | string\|null | null | ❌ | 本记忆被谁替代 | `[已确定]` |
| `conflictGroupId` | string\|null | null | ❌ | **冲突组**（新增，替代"全员 pending"） | `[已确定]` |
| `isCandidate` | boolean | false | ✅ | **是否为待审候选**（新增） | `[已确定]` |
| `source` | enum | `model` | ✅ | model/manual/migration/import/share_copy | `[已确定]` |
| `sourceMemoryId` | string\|null | null | ❌ | **分享副本指向原件**（新增） | `[已确定]` |
| `sourceRevision` | number\|null | null | ❌ | 副本复制时的原件 revision | `[已确定]` |
| `bRefs` | object[] | `[]` | ❌ | `[{conversationKey, category, slotId}]` | `[已确定]` |
| `opId` | string\|null | null | ❌ | 产生本记录的 operationId | `[已确定]` |
| `vec` | number[] | — | ✅ | 1024 维 | `[已确定]` |
| `vecModel` | string | `bge-m3` | ✅ | 向量模型标识 | `[已确定]` |

## 1.2 关键字段约束

```
C1  ownerId 与 conversationKey 同源：
      private:X → ownerId='owner:X'   且 conversationKey='private:X'
      group:X   → ownerId='group:X'   且 conversationKey='group:X'
    例外：分享副本也是这个规则（副本 ownerId='group:X'）

C2  visibility=group_shared 必要条件：
      ownerId 以 'group:' 开头
      AND shareReviewStatus='approved'
      AND reviewStatus!='rejected'
    ★ share_copy 是唯一能产生 visibility=group_shared 的路径（见 §12）

C3  factKey 必须匹配唯一模板，且 subjectId 与该模板的 subject 段一致

C4  factKey 允许的 type 由模板的 allowedTypes 决定（不再全局写死）

C5  status 与时间字段（修订 §六）：
      status='superseded' → supersededAt ≠ null 且 = statusChangedAt
      status='archived'   → archivedAt   ≠ null 且 = statusChangedAt
      status='deleted'    → deletedAt    ≠ null 且 = statusChangedAt
      status='active'     → statusChangedAt 必须 ≥ 三个时间字段（若有值）
    三时间字段互不要求互斥（可能是历史痕迹）

C6  revision 单调递增；任何状态变更 revision += 1

C7  conflictGroupId 非 null 且 isCandidate=true 时，该记录不参与普通自动注入

C8  source='share_copy' 必须带 sourceMemoryId 和 sourceRevision
```

---

# 二、factKey 模板驱动解析（P0-1）

## 2.1 设计决定：模板驱动，不做统一位置解析

`[已确定]`

V2.2 的错误：用一条通用语法强行规定所有 factKey 的段顺序，但设备模板的 `<qualifier>.<attr>` 顺序与通用语法的 `<attribute>.<qualifier>` 顺序相反。

V2.3：**先匹配模板，再按该模板声明的 role 解释各段**。不假设任何全局位置语义。

## 2.2 模板结构（可直接实现）

```json
{
  "version": 3,
  "templates": [
    {
      "id": "device",
      "description": "设备配置",
      "arity": 4,
      "segments": [
        {"pos": 0, "role": "subject", "enum": ["owner"]},
        {"pos": 1, "role": "literal", "value": "device"},
        {"pos": 2, "role": "qualifier", "enum": ["desktop", "lab", "phone", "work", "home"]},
        {"pos": 3, "role": "attribute", "enum": ["gpu", "cpu", "ram", "disk", "os", "monitor"]}
      ],
      "cardinality": "multi_valued",
      "conflictPolicy": "keep_both",
      "allowedTypes": ["fact"],
      "enabled": true
    },
    {
      "id": "preference",
      "arity": 3,
      "segments": [
        {"pos": 0, "role": "subject", "enum": ["owner"]},
        {"pos": 1, "role": "literal", "value": "preference"},
        {"pos": 2, "role": "attribute", "enum": ["food", "game", "music", "sleep", "social", "work"]}
      ],
      "cardinality": "single_current",
      "conflictPolicy": "supersede_with_review",
      "allowedTypes": ["fact", "preference"],
      "enabled": true
    },
    {
      "id": "schedule",
      "arity": 3,
      "segments": [
        {"pos": 0, "role": "subject", "enum": ["owner"]},
        {"pos": 1, "role": "literal", "value": "schedule"},
        {"pos": 2, "role": "attribute", "enum": ["sleep", "meal", "study"]}
      ],
      "cardinality": "single_current",
      "conflictPolicy": "supersede_with_review",
      "allowedTypes": ["fact"],
      "enabled": true
    },
    {
      "id": "member_relation",
      "arity": 3,
      "segments": [
        {"pos": 0, "role": "literal", "value": "member"},
        {"pos": 1, "role": "subjectId_qq", "pattern": "^\\d{5,12}$"},
        {"pos": 2, "role": "literal", "value": "relation"}
      ],
      "cardinality": "multi_valued",
      "conflictPolicy": "keep_both",
      "allowedTypes": ["relationship"],
      "enabled": true
    },
    {
      "id": "bot_persona",
      "arity": 3,
      "segments": [
        {"pos": 0, "role": "subject", "enum": ["bot"]},
        {"pos": 1, "role": "literal", "value": "persona"},
        {"pos": 2, "role": "attribute", "enum": ["name", "mood", "habit"]}
      ],
      "cardinality": "single_current",
      "conflictPolicy": "supersede_with_review",
      "allowedTypes": ["fact", "preference"],
      "enabled": true
    },
    {
      "id": "topic_attr",
      "arity": 3,
      "segments": [
        {"pos": 0, "role": "literal", "value": "topic"},
        {"pos": 1, "role": "slug", "pattern": "^[a-z0-9-]{2,32}$"},
        {"pos": 2, "role": "attribute", "enum": ["issue", "status", "deadline", "owner"]}
      ],
      "cardinality": "single_current",
      "conflictPolicy": "supersede_with_review",
      "allowedTypes": ["fact", "event"],
      "enabled": true
    }
  ]
}
```

**role 类型定义**：
| role | 校验方式 |
|---|---|
| `subject` | 值必须在 enum 内 |
| `literal` | 值必须严格等于 value |
| `qualifier` | 值必须在 enum 内 |
| `attribute` | 值必须在 enum 内 |
| `subjectId_qq` | 匹配 `pattern` 且与 subjectId 的 qq 段一致 |
| `slug` | 匹配 `pattern` |

## 2.3 执行顺序（六步，顺序不可换）

```
① 语法粗筛
   段数 3~5；每段非空；全小写 [a-z0-9-]
   （宽松，只是为了挡住明显垃圾，不做语义判断）
   失败 → REJECT_FORMAT

② 模板匹配
   遍历 enabled=true 的模板：
     条件：key 的段数 === template.arity
     AND 每个 pos 的段满足该 role 的校验
   结果：
     - 匹配 0 个 → REJECT_NO_TEMPLATE
     - 匹配 1 个 → 继续
     - 匹配 ≥2 个 → REJECT_AMBIGUOUS（配置错误，报警）

③ 主体验证
   模板里 role=subject 的段值，必须与记录 subjectId 一致：
     subject='owner' → subjectId='owner'
     subject='bot'   → subjectId='bot'
     role=subjectId_qq 的段值 → subjectId 必须是 'member:<同 qq>'
   失败 → REJECT_SUBJECT_MISMATCH

④ 类型验证
   记录的 type 必须 ∈ template.allowedTypes
   失败 → REJECT_TYPE_MISMATCH

⑤ 权限验证
   同 factKey 的已有记录，其 ownerId 和 visibility 必须与新记录一致
   失败 → REJECT_SCOPE_MISMATCH

⑥ 通过
   写入 factKey + factKeyTemplate + cardinality（从模板继承）
```

## 2.4 模板歧义防护

```
规则：同一 key 只能匹配一个模板

检测方式：启动时 + 每次模板变更后，遍历所有 enabled 模板，
  对每个模板生成一个"最小样例 key"和"最大样例 key"，
  交叉验证是否有多模板匹配
  发现歧义 → 拒绝加载该模板 + 报警

实际歧义示例（必须避免）：
  preference (arity=3): owner.preference.<attr>
  schedule   (arity=3): owner.schedule.<attr>
  → literal 段不同（preference vs schedule），不会歧义 ✅

  device (arity=4): owner.device.<qual>.<attr>
  topic_attr (arity=3): topic.<slug>.<attr>
  → arity 不同 + literal 不同，不会歧义 ✅
```

## 2.5 模板注册/修改/禁用的权限与版本

```
[已确定] 模板管理是控制台专属操作（需 consoleToken），机器人无权

操作：
  注册（add）     → 校验不与现有模板歧义
  修改（update）  → 已有记录保留旧 factKey 和旧 factKeyTemplate
                    新写入用新模板
  禁用（disable） → enabled=false
                    已有记录不受影响（factKey 保留，仍可检索）
                    新写入不能再用该模板
  删除（delete）  → 禁止（只能 disable）

版本管理：
  state/fact-keys.json 有 version 字段，每次变更 +1
  每次变更写 state/fact-keys-audit.jsonl：
    {ts, action, templateId, before, after, by, opId}
  历史版本保留在 state/fact-keys-history/ 下（按 version 命名）
```

## 2.6 每种模板的合法/非法示例（每模板 3+3）

### device（owner.device.\<qualifier\>.\<attribute\>，arity=4）

| 示例 | 判定 | 原因 |
|---|---|---|
| `owner.device.desktop.gpu` | ✅ | 四段，qualifier/attr 均在枚举 |
| `owner.device.lab.cpu` | ✅ | 同 |
| `owner.device.phone.os` | ✅ | 同 |
| `owner.device.gpu` | ❌ 业务拒绝 | 三段，arity 不符（**格式合法**，但没有匹配模板） |
| `owner.device.server.gpu` | ❌ 业务拒绝 | server 不在 qualifier 枚举 |
| `owner.device.desktop.gpu.x` | ❌ 语法拒绝 | 五段超 arity |

### preference（owner.preference.\<attribute\>，arity=3）

| 示例 | 判定 | 原因 |
|---|---|---|
| `owner.preference.food` | ✅ | |
| `owner.preference.game` | ✅ | |
| `owner.preference.sleep` | ✅ | |
| `owner.preference.desktop` | ❌ 业务拒绝 | desktop 不在 attribute 枚举 |
| `owner.preference` | ❌ 语法拒绝 | 两段不足 |
| `owner.preference.food.spicy` | ❌ 业务拒绝 | 四段，无模板匹配 |

### schedule（owner.schedule.\<attribute\>，arity=3）

| 示例 | 判定 | 原因 |
|---|---|---|
| `owner.schedule.sleep` | ✅ | |
| `owner.schedule.meal` | ✅ | |
| `owner.schedule.study` | ✅ | |
| `owner.schedule.exercise` | ❌ 业务拒绝 | 不在枚举（需人工加入模板） |
| `bot.schedule.sleep` | ❌ 主体不符 | subject 段值 bot 但 subjectId 若非 bot 则拒绝 |
| `owner.schedule` | ❌ 语法拒绝 | 两段不足 |

### member_relation（member.\<qq\>.relation，arity=3）

| 示例 | 判定 | 原因 |
|---|---|---|
| `member.[OWNER_QQ].relation` | ✅ | qq 10 位 |
| `member.12345.relation` | ✅ | qq 5 位（边界） |
| `member.123456789012.relation` | ✅ | qq 12 位（边界） |
| `member.abc.relation` | ❌ 语法拒绝 | qq 非数字 |
| `member.1234.relation` | ❌ 语法拒绝 | qq 少于 5 位 |
| `member.[OWNER_QQ].friend` | ❌ 业务拒绝 | 第三段必须是 literal 'relation' |

### bot_persona（bot.persona.\<attribute\>，arity=3）

| 示例 | 判定 | 原因 |
|---|---|---|
| `bot.persona.name` | ✅ | |
| `bot.persona.mood` | ✅ | |
| `bot.persona.habit` | ✅ | |
| `owner.persona.name` | ❌ 业务拒绝 | subject 段必须是 bot |
| `bot.persona` | ❌ 语法拒绝 | 两段不足 |
| `bot.persona.hobby` | ❌ 业务拒绝 | hobby 不在枚举 |

### topic_attr（topic.\<slug\>.\<attribute\>，arity=3）

| 示例 | 判定 | 原因 |
|---|---|---|
| `topic.stm32-debug.issue` | ✅ | **语法与业务都合法**（前提是 topic_attr 模板已注册） |
| `topic.reset.issue` | ✅ | |
| `topic.stm32-debug.deadline` | ✅ | |
| `topic.STM32.issue` | ❌ 语法拒绝 | 必须全小写 |
| `topic.a.issue` | ❌ 语法拒绝 | slug 少于 2 位 |
| `topic.stm32-debug.priority` | ❌ 业务拒绝 | priority 不在枚举 |

**特别说明（审核要求）**：`topic.stm32-debug.issue` 语法完全合法，但**只有在 topic_attr 模板注册且 enabled 时**才通过业务验证。V2.2 用这个例子却没说明这点，容易误以为"语法合法=可用"。

---

# 三、cardinality 与 conflictPolicy（P0-2）

## 3.1 两个概念的关系 `[已确定]`

```
cardinality 决定"同一 factKey 允许多少个 active"
conflictPolicy 决定"违反基数时怎么做"

cardinality 是主，conflictPolicy 是派生：
```

| cardinality | 允许的 conflictPolicy | 语义 |
|---|---|---|
| `single_current` | `supersede` / `supersede_with_review` / `manual_review` | 只允许 1 个 active，冲突必须处理 |
| `multi_valued` | `keep_both`（唯一） | 允许多个 active，不构成冲突 |

**`keep_both` 是否等价 `multi_valued`？**

`[已确定]` **不是等价，是绑定关系**：
- `multi_valued` 是基数声明（事实层面的性质）
- `keep_both` 是该基数下唯一合法的冲突策略（实现层面的动作）
- 若模板声明 `cardinality='multi_valued'` 但 `conflictPolicy != 'keep_both'` → **配置非法，拒绝加载**

## 3.2 完整规则表

| 模板 | cardinality | conflictPolicy | 需要审核 | 多 active |
|---|---|---|---|---|
| device | `multi_valued` | `keep_both` | ❌ | ✅ 正常（多设备） |
| preference | `single_current` | `supersede_with_review` | ✅ | ❌ 异常 |
| schedule | `single_current` | `supersede_with_review` | ✅ | ❌ 异常 |
| member_relation | `multi_valued` | `keep_both` | ❌ | ✅ 正常（关系叠加） |
| bot_persona | `single_current` | `supersede_with_review` | ✅ | ❌ 异常 |
| topic_attr | `single_current` | `supersede_with_review` | ✅ | ❌ 异常 |

**修正**：V2.2 把 device 设为 `supersede`（单值）是错的——主人可能有台式机、实验室电脑、手机多台，本来就该是多值。V2.3 改为 `multi_valued`。

## 3.3 关系历史的表达

`[已确定]`

```
relationship 类型的多值由 multi_valued 承载：
  多条 active 记录，各自 createdAt / occurredAt 不同
  
  supersedes/supersededBy 只用于"事实被替代"（single_current），
  绝不用于表达"关系历史"
  
为什么不能用 supersedes 表达关系历史：
  supersedes 语义是"旧的无效了"
  但关系变化是"又多了一层关系"或"关系变了"，旧的不一定无效
  
例：普通群友 → 熟人 → 朋友
  三条都 active（multi_valued），按时间排序
  而不是 A supersedes B supersedes C
```

## 3.4 多 active 异常检测（仅 single_current）

```
触发：某 factKey 的记录中，status='active' 且 reviewStatus='approved' 的条数 > 1
     且模板 cardinality='single_current'

处理（修订 V2.2 的"全员 pending"）：
  ❌ 不改 reviewStatus（会破坏已有审核结论）
  ✅ 给这组记录打同一个 conflictGroupId（如 cg_<rand8>）
  ✅ 检索时：整个冲突组默认不注入（过滤第②层排除）
  ✅ 写 state/memory-conflicts.jsonl 报警
  ✅ 控制台高亮，等人工处理

人工处理（P4 提供 op=resolve_conflict）：
  选定保留哪条 → 其余改 status='superseded'，supersededBy=保留条
  清除 conflictGroupId
  写审计

为什么不用 pending：
  pending 是"内容待确认"，与"多版本冲突"是两回事
  改 pending 会让原本 approved 的内容变成不可用，破坏已有结论
```

## 3.5 三个案例

### 案例 1：关系叠加（multi_valued，正常）

```
t0  写：'和群友 A 是普通群友关系'
    factKey='member.<A>.relation', type='relationship'
    → 模板 member_relation，cardinality=multi_valued
    → 写入，status=active, reviewStatus=approved

t1  写：'和群友 A 变成熟人了'
    factKey 相同
    → ②b factKey 查找：命中 1 条 active
    → 但 cardinality=multi_valued → 不做 supersede
    → 直接新增，两条都 active

t2  检索 '和 A 的关系'：
    → 返回两条（普通群友 / 熟人），按 createdAt 排序
    → 模型能看到关系演变 ✅
```

### 案例 2：关系变化（multi_valued，需要表达"已不是"）

```
t0  '和 A 是同事'
t1  '已经从 A 那家公司离职，不再是同事'

→ 两条都 active（multi_valued 允许）
→ 第二条的 content 自带否定语义
→ 模型读两条能理解
★ 如果希望"旧关系失效"，需要**人工显式**操作
  （控制台把旧记录 archive，不是自动 supersede）
```

**这是 multi_valued 的固有局限**：系统不会自动判断"关系结束了"。这是 `[待决策]` D-rel。

### 案例 3：设备多实例（multi_valued，正常）

```
t0  `owner.device.desktop.gpu` = '[GPU]'
t1  `owner.device.lab.gpu`     = 'RTX 4090'
t2  `owner.device.phone.os`    = 'Android 15'

→ 三条不同 factKey，各自 active
→ 检索'设备'返回三条，模型能区分 ✅

t3  写 `owner.device.desktop.gpu` = '[GPU]'
    → factKey 与 t0 相同，命中 1 条
    → cardinality=multi_valued 但同 factKey...
```

**这里需要澄清**：`multi_valued` 允许的是"不同 qualifier 的多值"，不是"同 factKey 的多值"。

`[已确定]` 修正规则：

```
"multi_valued" 的准确含义：
  同一 (entity, attribute) 下允许多个 qualifier 各自的 active 记录
  但**单一 factKey 仍只允许 1 个 active**

即：
  owner.device.desktop.gpu  → 只能 1 条 active
  owner.device.lab.gpu      → 只能 1 条 active
  两者并存 = 正常

t3 写 desktop.gpu='5070 Ti'：
  → 同 factKey 已有 active
  → 需要 supersede（不是 keep_both）
  → 但 conflictPolicy=keep_both 声明的行为是"都保留"？

矛盾！

V2.3 修正：
  把"基数"拆成两层：
  perFactKeyCardinality  单 factKey 允许多少 active  → 永远 single（1）
  perEntityCardinality   同 entity+attribute 允许多少 qualifier → multi（N）

  所有模板 perFactKeyCardinality 恒为 single（不允许同 factKey 多 active）
  perEntityCardinality 决定是"多设备"还是"单值偏好"

  → device: perEntity=multi（多设备），perFactKey=single（每设备一个当前值）
  → preference: perEntity=single（一个 food 偏好），perFactKey=single
  → member_relation: perEntity=multi（关系可多条），perFactKey=single？
```

**member_relation 的问题**：`member.<qq>.relation` 的 factKey 里没有"关系类型"这一段，所以同 qq 的所有关系记录都是同一个 factKey → perFactKey=single 就会导致新关系 supersede 旧关系，但实际上是 multi。

`[待决策]` 方案：
- 方案 a：给 member_relation 加一层 qualifier（`member.<qq>.relation.<type>`）
  → 不同关系类型 = 不同 factKey → perFactKey=single 成立
  → 但"关系变化"（同事→朋友）还是需要 supersede？
- 方案 b：member_relation 显式声明 `perFactKeyCardinality='multi'`（例外）

**推荐方案 a**（更一致），但这是 `[待决策]` D-card。

---

# 四、两阶段事实更新（P0-3）

## 4.1 V2.2 的致命缺陷

```
V2.2 流程（supersede_with_review）：
  写新记录 → 旧记录立即 status='superseded'
            → 新记录 reviewStatus='pending'
            
检索时：
  旧记录：status=superseded → 被生命周期过滤排除 ❌
  新记录：reviewStatus=pending → 被审核过滤排除 ❌
  
结果：这条事实在记忆里【完全消失】了，直到人工审核。
      如果人工一直没审 → 永久丢失。
```

**这是最严重的设计缺陷**。V2.3 必须修复。

## 4.2 V2.3 两阶段方案 `[已确定]`

### 阶段 1（写入时）

```
新记录：
  status        = 'active'
  reviewStatus  = 'pending'
  isCandidate   = true
  conflictGroupId = cg_xxx（若有旧记录）

旧记录：
  ★ 完全不动：status='active', reviewStatus='approved', isCandidate=false
  
检索效果：
  旧记录：active + approved → 正常可见 ✅
  新记录：active 但 pending + isCandidate → 不可见（待审）
  
→ 事实不会中断
```

### 阶段 2（人工批准）

```
单个 WAL 操作 op_approve：
  kind='approve_fact_update'
  changes: [
    { id: 旧, expectRevision: N, expectStatus: 'active',
      set: { status:'superseded', supersededBy:新, supersededAt:now, revision:N+1 } },
    { id: 新, expectRevision: M, expectReviewStatus: 'pending',
      set: { reviewStatus:'approved', isCandidate:false, revision:M+1 } }
  ]
  
原子性：
  - 两条 changes 在同一 opId 下
  - WAL 逐条验证 + 逐条执行
  - 任一条失败 → 整体回滚（或标记 partial，见 §9）
  - 只有两条都 done 才 committed
```

### 阶段 2'（人工拒绝）

```
单个 WAL 操作 op_reject：
  changes: [
    { id: 新, expectReviewStatus:'pending',
      set: { reviewStatus:'rejected', isCandidate:false, revision:M+1 } }
  ]
旧记录完全不动 → 继续 normal 可见
```

## 4.3 所有 conflictPolicy 的完整语义

| conflictPolicy | 需要审核 | 阶段1行为 | 阶段2（批准） | 阶段2（拒绝） |
|---|---|---|---|---|
| `supersede` | ❌ | 直接原子 supersede（旧→superseded，新→active+approved） | 不适用 | 不适用 |
| `supersede_with_review` | ✅ 强制 | 新→active+pending+isCandidate，旧不动 | 原子切换 | 新→rejected |
| `manual_review` | ✅ 强制 | 同上 | 同上 | 同上 |
| `keep_both` | ❌ | 直接新增（不同 factKey 或 multi 场景） | 不适用 | 不适用 |

**`supersede`（无需审核）何时可用？** `[待决策]` 
- 候选：只用于"确定无疑的机械更新"（如 bot.persona.name）
- 风险：无人工把关

**`supersede_with_review_if_negation`**：`[已删除]`
V2.2 有这个策略，但它依赖否定词检测来决定"要不要审核"，而 V2.3 已明确否定词不可靠。**统一改为 `supersede_with_review`**（总是审核）。

## 4.4 关键问题回答

| 问题 | 答案 |
|---|---|
| 审核通过前旧记录是否始终可检索？ | ✅ **是**（这是两阶段的核心价值） |
| 审核通过后的 supersede 是否单个 operation？ | ✅ **是**（一个 opId，changes 两条） |
| 人工审核与机器人自动判断的权限边界？ | 机器人只能写 pending 候选；批准/拒绝只能走控制台（consoleToken）；机器人无 consoleToken |
| 新事实在审核期间再次被更新？ | 见 §4.5 |

## 4.5 审核期间再次更新的处理

```
场景：
  t0 旧: memory='A', status=active, approved
  t1 写 B: isCandidate=true, pending, conflictGroupId=cg_1
  t2 写 C: 同 factKey，又要写
  
处理：
  ②b factKey 查找时，除了 status='active'，还要查 isCandidate=true 的记录
  → 发现 t1 的候选 B（同 conflictGroupId）
  
  t2 的行为：
    新候选 C：isCandidate=true, pending, conflictGroupId=cg_1（同一组）
    B 保持 pending（不删，留作历史）
    
  人工审核时看到：B（较早）、C（最新）
  批准 C → B 标 rejected（被更新的候选取代）
  或批准 B → C 标 rejected
```

**关键**：候选不自动互相替代，形成"候选队列"，由人工选择。

## 4.6 不同类型事实的策略

| 类型 | 模板 | 策略 | 理由 |
|---|---|---|---|
| **schedule**（作息） | `single_current` + `supersede_with_review` | 审核 | 作息变化可能是临时反常，需人确认 |
| **preference**（偏好） | `single_current` + `supersede_with_review` | 审核 | 口味变化需确认（也可能是开玩笑） |
| **device**（设备） | `multi_valued` per-entity | 同 factKey 冲突时审核 | 设备是客观事实，但"换卡"需确认 |
| **member_relation** | `multi_valued` | 不审核 | 关系叠加是常态 |
| **bot_persona**（机器人自己） | `single_current` | `[待决策]` 审核或不审 | 名字变化较确定，可不审 |

---

# 五、生命周期与历史时间（P0-4）

## 5.1 方案选择：lifecycleEvents + 当前状态

`[已确定]` 采用 **lifecycleEvents 数组**，同时保留三个便捷时间字段（冗余，便于查询）。

```json
{
  "status": "archived",
  "statusChangedAt": 1793400000000,
  "supersededAt": 1793390000000,
  "archivedAt": 1793400000000,
  "deletedAt": null,
  "lifecycleEvents": [
    {"from": null, "to": "active", "at": 1793380000000, "by": "model", "opId": "op_1"},
    {"from": "active", "to": "superseded", "at": 1793390000000, "by": "model", "opId": "op_2", "reason": "新事实替代"},
    {"from": "superseded", "to": "archived", "at": 1793400000000, "by": "console", "opId": "op_3", "reason": "清理旧版本"}
  ]
}
```

## 5.2 为什么这样设计

| 问题 | lifecycleEvents 如何解决 |
|---|---|
| 当前状态唯一确定 | `status` 字段唯一 |
| 历史不丢失 | `lifecycleEvents` 完整追加 |
| 三时间字段语义清晰 | 各自只在对应事件时写入，不要求互斥 |
| 校验规则与转换表一致 | 校验只看"当前 status 与 statusChangedAt + events 末条一致" |

## 5.3 字段校验规则（替代 V2.2 的互斥规则）

```
V1  当前 status 必须与 lifecycleEvents 最后一条的 to 一致
    （空数组时 status 必须是 'active' 且来自创建）

V2  statusChangedAt 必须等于 lifecycleEvents 最后一条的 at

V3  时间字段规则（不要求互斥，只要求"有值则必须与历史一致"）：
      supersededAt ≠ null ⟺ lifecycleEvents 中存在 to='superseded' 的事件
      且 supersededAt = 该事件的 at（若有多次，取最近一次）
      archivedAt   ≠ null ⟺ 存在 to='archived' 的事件
      deletedAt    ≠ null ⟺ 存在 to='deleted' 的事件

V4  status='superseded' → supersededAt ≠ null 且 = statusChangedAt
    status='archived'   → archivedAt   ≠ null 且 = statusChangedAt
    status='deleted'    → deletedAt    ≠ null 且 = statusChangedAt
    status='active'     → statusChangedAt ≥ 所有非 null 时间字段

V5  lifecycleEvents 只追加，不修改、不删除（除 compact 时合并连续相同事件）

V6  任何状态变更必须同时：更新 status + statusChangedAt + 追加 event + revision+1
    四者原子
```

## 5.4 完整状态转换表

| from | to | 触发 | 必填 | lifecycleEvent |
|---|---|---|---|---|
| (无) | `active` | 创建 | — | `{from:null, to:'active'}` |
| `active` | `superseded` | 事实更新批准 | `supersededBy` | `{from:'active', to:'superseded', reason:'新事实'}` |
| `active` | `archived` | 主动归档 | — | `{from:'active', to:'archived'}` |
| `active` | `deleted` | 删除 | tombstone | `{from:'active', to:'deleted'}` |
| `superseded` | `active` | 恢复旧事实（管理端） | 同时把当前 active 改 superseded | `{from:'superseded', to:'active'}` |
| `superseded` | `archived` | 归档旧版本 | — | `{from:'superseded', to:'archived'}` |
| `superseded` | `deleted` | 删除 | tombstone | `{from:'superseded', to:'deleted'}` |
| `archived` | `active` | 恢复归档 | — | `{from:'archived', to:'active'}` |
| `archived` | `deleted` | 删除 | tombstone | `{from:'archived', to:'deleted'}` |
| `archived` | `superseded` | 归档的旧版本又被替代 | 罕见 | `{from:'archived', to:'superseded'}` |
| `deleted` | 任何 | **禁止**（除 force_restore） | — | — |
| `deleted` | `active` | force_restore（管理员） | 审计日志 + reason | `{from:'deleted', to:'active', reason:'force_restore'}` |

## 5.5 归档/删除不会自动恢复

```
规则：
  ① archived / deleted 永不参与自动检索（过滤第②层硬排除）
  ② 恢复只能通过显式操作（qq_memory_manage op=restore / op=force_restore）
  ③ 恢复必须带 reason，写审计
  ④ 恢复后 revision += 1（防止旧操作覆盖）
  ⑤ TTL 到期 → archived（不是 active，不是 deleted）
```

---

# 六、tombstone 与重新授权（P0-5）

## 6.1 两个目的严格分开

`[已确定]`

| 目的 | 手段 | 默认 |
|---|---|---|
| **A. 防旧记录复活** | 按 `memoryId` 阻止 | ✅ 默认开启 |
| **B. 防相同内容重新写入** | 按 `contentHash` 阻止 | ❌ **默认关闭**，需人工开启 |

## 6.2 tombstone 字段与生命周期

```json
{
  "memoryId": "mem_x",
  "ownerId": "owner:[OWNER_QQ]",
  "contentHash": "a1b2...",
  "deletedAt": 1793400000000,
  "deletedBy": "console",
  "reason": "user_request",
  "suppression": "memory_only",
  "hard": false,
  "purgeLevel": "logical",
  "restoredAt": null,
  "reAuthorized": []
}
```

| 字段 | 说明 |
|---|---|
| `suppression` | `memory_only`（只防该 id 复活）/ `content_suppression`（同内容也拒） |
| `hard` | 是否做过内容清除 |
| `purgeLevel` | `logical` / `content` / `cross_store`（见 §七） |
| `restoredAt` | 被 force_restore 过的时间（仍保留 tombstone） |
| `reAuthorized` | 用户重新授权的记录 `[{at, newMemoryId, by}]` |

**生命周期**：
```
创建：删除时
更新：
  - force_restore → 写 restoredAt（不删 tombstone）
  - 用户重新授权 → 追加 reAuthorized 条目
  - 升级 suppression → 从 memory_only 改为 content_suppression（需人工）
删除（清理）：
  - 默认永久保留（tombstone 很小）
  - 除非管理员显式 purge（需审计）
```

## 6.3 memory_only vs content_suppression

```
memory_only（默认）：
  检查：写入时若 record.id ∈ tombstones → 拒绝（该 id 已被删）
        但相同 content 用新 id 写入 → ✅ 允许
  适用：普通删除（用户说"忘了这事"）
  
content_suppression（需人工开启）：
  检查：写入时若 contentHash ∈ tombstones → 拒绝
  适用：隐私敏感（用户说"这个内容以后别再记了"）
  开启方式：删除时显式传 suppression='content_suppression'
  限制：只有控制台能开启（机器人不行）
```

## 6.4 重新授权流程

```
场景：用户删了"主人喜欢香菜"（memory_only），之后又说"我确实喜欢香菜"

流程：
  ① 写入检测：contentHash 匹配 tombstone，但 suppression='memory_only'
  ② 允许写入，生成新 memoryId
  ③ tombstone 追加：reAuthorized: [{at:now, newMemoryId:'mem_new', by:'model'}]
  ④ 新记录 source='model'（不是 share_copy 等），reviewStatus='approved'
     （因为用户明确重新提供了，可信）
  ⑤ 检索：只返回新记录（旧的 status=deleted 不参与）
```

**如果 suppression='content_suppression'**：
```
① 写入检测：contentHash 匹配
② 拒绝，返回错误："该内容已被明确标记为不再记录"
③ 若模型坚持（force=true）→ 仍需 consoleToken，机器人无法 force
④ 控制台可手动解除 suppression 或强制写入（写审计）
```

## 6.5 force 的权限

```
[已确定]
机器人（agentToken）：      ❌ 无 force 权限
控制台（consoleToken）：    ✅ 可 force，必须提供 reason，写审计
管理员（本地改文件）：      ✅ 可改，但绕过所有校验（记录在案）

审计记录 state/memory-audit.jsonl：
  {ts, action:'force_write'|'force_restore'|'purge_tombstone', 
   memoryId, contentHash, by, reason, opId}
```

## 6.6 备份恢复/迁移/WAL 重试如何检查 tombstone

| 路径 | 检查方式 |
|---|---|
| WAL 重试 | 执行前查 `tombstones[memoryId]` → 存在则 cancelled |
| 备份恢复 | 逐条导入前查：`tombstones[memoryId]` 存在 → 跳过<br>`tombstones[contentHash]` + suppression=content_suppression → 跳过 |
| 迁移重跑 | 同备份恢复 |
| 写入 | 查 memoryId（总是）+ contentHash（仅 content_suppression） |

## 6.7 删除与未完成操作的关系

```
删除操作（memoryId=X）时：
  ① 查 WAL：是否有 op 的 targetMemoryId === X 且非终态？
     ├─ 有 → 那些 op 标 cancelled
     └─ 无 → 继续
  ② 查 WAL：是否有 op 的 payload.memoryIds 包含 X？
     ├─ 有 → 该 op 标 cancelled（整个事务取消）
     └─ 无 → 继续
  ③ 执行删除（status=deleted + tombstone）
  ④ 写 WAL：删除本身是一个 op
  
顺序保证：先取消依赖 X 的操作，再删 X
```

## 6.8 删除本身可恢复、可审计、不依赖原文

```
可恢复：
  status=deleted 的记录完整保留（content 未清空时）
  force_restore 可恢复
  审计日志记录全过程

可审计：
  tombstone 有 deletedAt/deletedBy/reason
  audit 有完整操作链

不依赖删除后查不到的原文：
  即使 content 已被清空（hard=true），tombstone 仍有 contentHash
  通过 contentHash 可追溯"曾有这条内容"
```

---

# 七、隐私清除边界（P0-6）

## 7.1 三级清除

`[已确定]` 分三级，**本轮只实现前两级的设计，第三级不实现**。

| 级别 | 名称 | 范围 | 实现状态 |
|---|---|---|---|
| L1 | `logical`（逻辑删除） | 只改 `status='deleted'` | P4 实现 |
| L2 | `content`（内容清除） | 本体 `content='[deleted]'`、`vec=[]` | P4 实现 |
| L3 | `cross_store`（全链路） | 本体 + WAL + 失败记录 + 备份 | **本轮不实现** |

## 7.2 各存储的内容残留

| 存储 | 含原始 content | L1 后 | L2 后 | L3 后 |
|---|---|---|---|---|
| `state/memory-vectors.jsonl` | ✅ | 保留 | 清空 | 清空 |
| `state/memory-wal.jsonl` | ⚠️ 见 §7.3 | 保留 | 保留 | 清除/脱敏 |
| `state/memory-failures.jsonl` | ⚠️ 可能 | 保留 | 保留 | 清除/脱敏 |
| `state/memory-tombstones.jsonl` | ❌ 只有 hash | — | — | — |
| `backups/memory-vectors-*.jsonl` | ✅ | 保留 | 保留 | 按策略清理 |
| `D:\qq-bridge备份-*\*.jsonl` | ✅ | 保留 | 保留 | 人工清理 |
| **C 层 Session** | ✅ | **完全无关** | **完全无关** | **不自动纳入** |

## 7.3 WAL 是否允许保存原始 content

`[待决策]` D-wal-content

| 方案 | 优点 | 缺点 |
|---|---|---|
| a. WAL 存完整 content | 崩溃恢复能重写 | 删除后 WAL 里还有原文 |
| b. WAL 只存 contentHash + memoryId | 无隐私残留 | 崩溃恢复必须从别处取 content（但 content 就在待写的记录里，通常够） |
| c. 存前 N 字符（如 100） | 折中 | 仍可能含敏感信息 |

**推荐 b**（只存 hash + id）。理由：
- append 时 content 本来就在内存里，崩溃后可以要求重新提交
- 但若崩溃后模型不重提，这条记忆就丢了（可接受，因为本来就没写成功）

**若选 a**，删除后必须对 WAL 做脱敏（改写历史行 → 破坏"只追加"语义，需 rewrite WAL 为一个新 generation，见 §八）。

## 7.4 备份保留策略

```
当前实际（已核实）：
  backups/memory-vectors-YYYY-MM-DD.jsonl  保留 7 天（dailyBackup）
  D:\qq-bridge备份-20261030\  手工备份，永久（无自动清理）

V2.3 设计：
  backups/ 日备份：保留 7 天（不变）
  L2 内容清除后：不主动改历史备份（成本高），但在 tombstone 记 purgeLevel
  提供手动工具：qq_memory_manage op=purge_backups memoryId=X
    → 扫描所有备份文件，把该 id 的 content 清空
    → 需 consoleToken + reason
    → 本轮不实现
```

## 7.5 C 层 Session 不在 A 层删除范围

```
[已确定]
A 层删除（任何级别）都【不】自动触碰 C 层 Session：
  原因：① Session 由 DSH 管理，bridge 无直接写权限
       ② Session 是完整对话流，删除单条记忆在技术上困难
       ③ 删 Session 会破坏对话连续性

→ 若用户要求"彻底清除某段对话"，需：
   ① 删 A 层记忆（L2）
   ② 手动处理 Session（DSH 管理端 / 重置会话）
   ③ 清理备份
   这是三步人工流程，不是单次删除能完成的
```

## 7.6 隐私声明（必须写进文档和 UI）

```
⚠️ 明确声明：
  hard=true ≠ 彻底删除所有副本
  L2（内容清除）后：
    - A 层本体的 content 已清空 ✅
    - WAL、备份、Session 中可能仍有原文 ⚠️
    - 需要 L3（本轮未实现）才能全链路清除
  
  在 L3 实现前，不得在任何文档/UI 中宣称"彻底删除"
```

---

# 八、WAL 状态机与长期幂等（P0-7）

## 8.1 三层状态（保留 V2.2 已认可的部分）

```
operationStatus: created | running | partial | committed | failed_retriable | failed_exhausted | cancelled
aStatus:         none | pending | done | failed | skipped
bStatus:         none | pending | done | failed | skipped
```

**修订 V2.2**：`failed` 拆为 `failed_retriable` / `failed_exhausted`。

## 8.2 状态转换表（完整）

| operationStatus | 含义 | 允许的下一步 |
|---|---|---|
| `created` | 已创建，未开始 | running / cancelled |
| `running` | 执行中 | partial / committed / failed_retriable / cancelled |
| `partial` | 部分成功 | running（重试）/ committed / failed_retriable / failed_exhausted / cancelled |
| `committed` | 全成功 | （终态） |
| `failed_retriable` | 失败但可重试 | running（重试）/ cancelled |
| `failed_exhausted` | 重试耗尽 | cancelled / （人工 retry 重置为 running） |
| `cancelled` | 已取消 | （终态） |

**关键规则**：
```
① partial 只能由 running 进入（不能从 created 直接到 partial）
② failed_exhausted 只能由 partial + retryCount>=3 进入
③ cancelled 可以从任何非终态进入
④ committed / cancelled / failed_exhausted 是终态
⑤ 终态不可回退（除人工 retry 显式重置）
```

## 8.3 append 幂等（长期）

```
三重保障：
  ① 稳定 memoryId（写入前生成，写进记录）
  ② contentHash 精确去重（不依赖 WAL 保留期）
  ③ 正文二次确认（hash 相同也要比对 content）

流程：
  写入时：
    生成 memoryId
    计算 contentHash
    查同 ownerId + category + type + contentHash + status=active 的记录
      ├─ 找到 → 比对 content 全等
      │   ├─ 全等 → 更新 updatedAt，不新增（幂等）
      │   └─ 不等 → 视为 hash 碰撞（极罕见），新增 + warn
      └─ 未找到 → 新增
  
  ★ 这个机制不依赖 WAL，所以 90 天窗口外仍有效
```

## 8.4 状态变更幂等（revision 乐观锁）

```
每条记录有 revision（单调递增）

任何状态变更操作必须携带：
  targetMemoryId
  expectedRevision
  expectedStatus（或 expectedReviewStatus）
  newRevision = expectedRevision + 1
  opId

执行前校验（在 WITH WRITE LOCK 内）：
  record = 读 targetMemoryId
  if (record.revision !== expectedRevision) → CONFLICT_REVISION
  if (record.status !== expectedStatus)     → CONFLICT_STATUS
  → 拒绝，写 WAL {stage:'conflict', expected, actual}

执行：
  record.revision = newRevision
  record.status = newStatus
  ...
```

**这解决"旧操作覆盖新状态"**：

```
t0  记录 R: revision=5, status=active
t1  操作 A（删除）: expectedRevision=5 → 成功 → revision=6, status=deleted
t2  迟到的操作 B（supersede）: expectedRevision=5
    → 校验：实际 revision=6 ≠ 5 → 拒绝 ✅
    
→ B 不能把已删除的记录改回 superseded
```

## 8.5 compact 后仍能识别执行结果

```
因为校验靠 revision（在记录里），不靠 opId 索引
→ WAL compact 掉旧 op 不影响状态变更的正确性

op-index 只用于"append 幂等"的辅助（90 天窗口）
append 幂等的主保障是 contentHash（永久）
```

## 8.6 多 memoryId 事务的逐项验证与补偿

```
supersede / approve 等事务涉及多条记录：

payload:
{
  "opId": "op_x",
  "kind": "approve_fact_update",
  "changes": [
    {"id": "mem_old", "expectedRevision": 3, "expectedStatus": "active",
     "set": {"status": "superseded", "supersededBy": "mem_new", "supersededAt": 123}},
    {"id": "mem_new", "expectedRevision": 1, "expectedReviewStatus": "pending",
     "set": {"reviewStatus": "approved", "isCandidate": false}}
  ]
}

执行：
  for each change:
    校验 → 执行 → 记录该条的 done/failed
  全部 done → committed
  部分 done → partial（重试只补 failed 的）
  全 failed → failed_retriable
```

**补偿逻辑**：
- 不做"自动回滚已成功的部分"（复杂且危险）
- 而是**继续重试未完成的部分**，直到全部 done 或重试耗尽
- 重试耗尽 → `failed_exhausted` + 人工介入

## 8.7 无法安全恢复时失败关闭

```
recovering 时若遇到：
  ① revision 不匹配（说明有其他进程/操作改过）
  ② expectedStatus 不匹配
  ③ 记录不存在
  ④ 内容 hash 不匹配
  → 不猜测，标 failed_exhausted + 写 failures
  → 控制台可见，人工处理
```

## 8.8 force_restore 是新 operation

```
force_restore 不复用任何旧 opId：
  生成新 opId
  kind='force_restore'
  expectedStatus='deleted'
  set={status:'active', revision: N+1}
  
→ 已 cancelled 的旧 append 不会被重新激活
→ 旧 append 的 opId 保持 cancelled 终态
```

---

# 九、WAL checkpoint / generation（P0-8）

## 9.1 文件结构

```
state/wal/
  MANIFEST.json              ← 提交标记（唯一真相入口）
  wal-0007.jsonl             ← 当前 generation
  index-0007.json            ← 当前 op 索引
  wal-0006.jsonl             ← 上一代（备份）
  index-0006.json
```

## 9.2 MANIFEST 结构

```json
{
  "version": 1,
  "activeGen": 7,
  "pendingGen": null,
  "gens": [
    {"gen": 7, "createdAt": 1793400000000, "opCount": 1234, "checksum": "sha256...", "complete": true},
    {"gen": 6, "createdAt": 1793390000000, "opCount": 1100, "checksum": "sha256...", "complete": true}
  ]
}
```

## 9.3 compact 流程（含崩溃点）

```
前提：取得写锁（单写者架构下必然成功）

① 读当前 activeGen=7 的 wal + index
② 计算新内容：
   - 保留所有非终态 op（完整保留）
   - 最近 30 天的终态 op（完整保留）
   - 更早的终态 op → 只保留摘要进 index
③ 写 wal-0008.jsonl（唯一 tmp → rename）
   ⚠️ 崩溃点 A：写了但没 rename → 只有 .tmp，无影响
④ fsync wal-0008
   ⚠️ 崩溃点 B：fsync 失败 → 删 tmp，中止
⑤ 计算 checksum
⑥ 写 index-0008.json（唯一 tmp → rename）
   ⚠️ 崩溃点 C：index 写失败 → 删除 wal-0008，中止
⑦ fsync index-0008
⑧ 原子写 MANIFEST：
   {activeGen: 8, pendingGen: null, gens: [8, 7, 6]}
   ⚠️ 崩溃点 D：MANIFEST 写失败 → 旧 MANIFEST 仍指向 7，8 是新孤儿
      恢复时清理孤儿
⑨ 删除 gen 5 及更早
   ⚠️ 崩溃点 E：删除失败 → 只是垃圾文件，无害
```

## 9.4 两阶段提交（可选，更严格）

若要求"MANIFEST 写入本身也可恢复"：

```
步骤 ⑧ 改为两阶段：
  ⑧a 写 MANIFEST {activeGen:7, pendingGen:8, gens:[...]}
  ⑧b fsync
  ⑧c 写 MANIFEST {activeGen:8, pendingGen:null, gens:[8,7]}
  ⑧d fsync
  
恢复时：
  pendingGen != null：
    ├─ 检查 gen=8 的文件是否完整（checksum 正确）
    │  ├─ 完整 → 认为切换已完成，补写 MANIFEST activeGen=8
    │  └─ 不完整 → 回滚，清除 pendingGen，用 gen=7
    └─ 删除不完整的 gen 文件
```

## 9.5 崩溃恢复判定表

| 崩溃点 | MANIFEST 状态 | 磁盘文件 | 恢复动作 |
|---|---|---|---|
| A（wal tmp 未 rename） | activeGen=7 | `wal-0008.jsonl.*.tmp` | 删 tmp，用 7 ✅ |
| B（fsync 失败） | activeGen=7 | 同 A | 同 A |
| C（index 写失败） | activeGen=7 | `wal-0008.jsonl` 存在，index 不存在 | 删 wal-0008，用 7 ✅ |
| D（MANIFEST 写失败） | activeGen=7, pendingGen=null | wal-0008 + index-0008 都存在且完整 | 删 8（孤儿），用 7 ✅ |
| D'（两阶段中 ⑧c 失败） | activeGen=7, pendingGen=8 | 8 完整 | 补写 activeGen=8 ✅ |
| E（删旧 gen 失败） | activeGen=8 | 5/6/7 都在 | 无害，下次 compact 再删 ✅ |

## 9.6 何时删旧 WAL

```
规则：
  ① 新 gen（8）已 committed 到 MANIFEST
  ② 新 gen 的 wal 包含旧 gen 所有【非终态】op（步骤②保证）
  ③ 至少保留上一代（gen 7）用于回滚
  → 可以删 gen 6 及更早

禁止：
  ✗ 在仍有非终态 op 时删包含它们的 gen
  ✗ 在 MANIFEST 未确认前删任何 gen
```

## 9.7 仍有未完成操作时的保护

```
步骤② 明确要求：
  "保留所有非终态 op（完整保留）"
  
即：compact 不因"太旧"而丢弃未完成的 op
  即使一个 op 是 100 天前的 partial，也完整复制到新 gen
  
这样保证：compact 后仍能恢复所有这些 op
```

---

# 十、分享副本（P0-9）

## 10.1 方案 A `[已确定]`

```
批准后创建群聊共享副本：

原件（不变）：
  id: mem_orig
  ownerId: owner:A
  conversationKey: private:A
  visibility: owner_private
  shareReviewStatus: approved  ← 标记"曾被批准分享"
  revision: N+1

副本（新建）：
  id: mem_copy（新 id）
  ownerId: group:X
  conversationKey: group:X
  visibility: group_shared
  shareReviewStatus: approved
  reviewStatus: approved（继承原件的）
  source: 'share_copy'
  sourceMemoryId: mem_orig
  sourceRevision: N（复制时的原件 revision）
  content: 复制原件
  vec: 复制原件（不重算，省 token/时间）
  factKey: 若原件有，副本保留（但 factKey 的模板要求 subject 一致，需校验）
```

## 10.2 完整定义（10 项）

| # | 项目 | 定义 |
|---|---|---|
| 1 | share_request 引用方式 | 必须传 `memoryId`（不再接受 text 匹配）。若传 text，先用它定位到 id，再走 id 流程；定位失败 → 404 |
| 2 | 目标群号确定 | 从请求者的 `conversationKey` 取（群 token 对应群）。不接受显式传群号（防越权） |
| 3 | 谁发起/谁批准 | 发起：模型（群上下文，agentToken）<br>批准：控制台（consoleToken）<br>机器人不能批准自己的申请 |
| 4 | 创建副本的 WAL 操作 | `kind='share_copy'`，changes 两条：<br>① 原件 set `{shareReviewStatus:'approved', revision:N+1}`<br>② 新建副本（不在 changes 里，作为"create"类型） |
| 5 | 审批失败时状态 | 原件 `shareReviewStatus='rejected'`，不创建副本 |
| 6 | 撤销共享副本 | 副本 `status='deleted'` + tombstone（suppression=memory_only）<br>原件 `shareReviewStatus` 改回 `not_requested`（可选，或保留 approved 留痕） |
| 7 | 原件修改后是否同步副本 | **❌ 不同步**（默认）。副本是快照，sourceRevision 记录它是哪个版本 |
| 8 | 副本再次分享 | **禁止**。副本的 `ownerId=group:X`，要分享到群 Y 必须从原件再申请一次 |
| 9 | 重复申请去重 | 按 `(sourceMemoryId, targetOwnerId)` 查是否已有 active 副本<br>已有 → 返回 `already_shared` |
| 10 | 群内删除副本是否影响原件 | **❌ 不影响**。原件独立存在 |

## 10.3 副本的 factKey 处理

```
[待决策] D-copy-factkey

问题：原件的 factKey 可能是 `owner.device.desktop.gpu`（subject=owner）
      副本 ownerId 变成 group:X，但 factKey 的 subject 段还是 owner
      → 违反 C3（factKey subject 必须与 subjectId 一致）

方案 a：副本不带 factKey
  → 副本不参与事实更新（合理，它是共享快照）
  → 推荐

方案 b：副本 factKey 改写成 group scope
  → 需要新的模板体系
  → 复杂，不推荐
```

**推荐方案 a**：副本 `factKey=null`，`factKeyTemplate=null`。

## 10.4 修订后的权限矩阵

| # | 请求者 | 目标 ownerId | visibility | 状态要求 | 允许 |
|---|---|---|---|---|---|
| 1 | 私聊 A | `owner:A` | `owner_private` | active+approved | ✅ |
| 2 | 私聊 A | `owner:A` | `group_shared` | active+approved | ✅（自己的原件） |
| 3 | 私聊 A | `owner:B` | 任意 | — | ❌ |
| 4 | 私聊 A | `group:X` | 任意 | — | ❌（群副本不进私聊） |
| 5 | 群 X | `group:X` | `group_shared` | active+approved+shareApproved | ✅ |
| 6 | 群 X | `group:X` | `owner_private` | — | ❌ |
| 7 | 群 X | `group:X` | `group_shared` | shareReviewStatus≠approved | ❌ |
| 8 | 群 X | `group:Y` | 任意 | — | ❌（不跨群） |
| 9 | 群 X | `owner:A` | 任意 | — | ❌（群不读个人原件） |
| 10 | 控制台 | 任意 | 任意 | 任意 | ✅（写审计） |
| 11 | 机器人（agentToken） | 任意 | 任意 | — | ❌（无管理权） |

**关键**：因为采用方案 A（副本），矩阵第 4 行和第 9 行自然成立——群副本的 ownerId 是 group，个人原件的 ownerId 是 owner，两者天然隔离。

---

# 十一、单写者架构（P0-10）

## 11.1 设计决定 `[已确定]`

**采用单写者架构**：bridge 是唯一能写 A/B 文件的进程。

理由（直接回应审核意见）：
```
跨进程锁的根本问题：无法证明"原持有者真的停止写入了"
  - PID 存活 ≠ 在写（可能卡住，也可能已死但 PID 被复用）
  - 心跳停止 ≠ 已死（可能只是 event loop 阻塞）
  - claim 等待 ≠ 原进程不会恢复
  - 任何基于"超时"的判定都可能误判

单写者消除这个问题：根本不存在第二个写者
```

## 11.2 写入通道约束

```
唯一写者：bridge 进程（src/bridge.js）

其他需要写记忆的：
  ① 迁移脚本
  ② 维护脚本（compact、purge）
  ③ 手工修数据
  
约束：
  ✅ 必须通过 bridge HTTP 接口（走三重鉴权 / consoleToken）
  ✅ 或在服务停止时直接改文件
  ❌ 不允许服务运行时直接改 JSONL

检测违规：
  bridge 启动时检查 state/memory.lock（若存在说明上次异常或有人绕过）
  → 记录日志，不自动删除
```

## 11.3 脚本如何判断"服务在跑"

```
脚本启动时：
  ① 尝试连接 127.0.0.1:3100
     ├─ 连接成功 → 服务在跑 → 拒绝执行，提示"请先停止服务或改用 HTTP 接口"
     └─ 连接失败 → 可能停着，也可能端口改了
  ② 读 config.json 的 consolePort（确认端口）
  ③ 再次尝试连接
  ③' 仍失败 → 检查 state/memory.lock：
     ├─ 存在 → 说明上次可能是异常退出，提示人工确认
     └─ 不存在 → 可以执行
```

**这是"协作式"而非"抢占式"**：脚本主动让路，不争夺锁。

## 11.4 如果必须支持跨进程（离线迁移）

`[候选方案]` 保留一个"离线模式"：

```
离线迁移工具的强制流程：
  ① 检查端口 → 必须连不上（服务已停）
  ② 检查 lock 文件 → 必须不存在，或存在但 PID 已死且 instanceId 不匹配
  ③ 创建自己的 lock（记录 instanceId）
  ④ 执行迁移
  ⑤ 释放 lock

如果 ② 无法确认：
  → 失败关闭，要求人工确认
  → 不猜测
```

## 11.5 进程内锁（仍然需要）

```
即使单写者，进程内仍有并发（多个 HTTP 请求同时到）
→ 必须用 Promise 链串行化

let writeChain = Promise.resolve();
function withWriteLock(fn) {
  const task = writeChain.then(fn, fn);
  writeChain = task.catch(() => {});
  return task;
}
```

## 11.6 回应对审核的 8 个问题

| # | 问题 | 单写者架构下的答案 |
|---|---|---|
| 1 | 如何证明锁所有权 | 不需要——只有一个写者 |
| 2 | 如何防止原持有者恢复后继续写 | 不存在第二个持有者 |
| 3 | PID 复用 | 不涉及 |
| 4 | 心跳暂停但进程活着 | 不涉及 |
| 5 | 多 claim 竞争 | 不存在 |
| 6 | WAL running 状态不完整 | 由崩溃恢复流程处理（验证 contentHash） |
| 7 | 无法证明锁失效时失败关闭 | 离线模式下做不到就失败关闭 |
| 8 | 接管后防旧持有者提交 | 不存在接管 |

**结论**：单写者是唯一能真正闭合这些问题的方案。

---

# 十二、A/B/C 关联与 slotId（P1-1）

## 12.1 B 层稳定 slotId

```
B 层每条记忆加 slotId：
  activeTopics[i]     → { slotId: 'slot_<rand8>', text, lastMentionAt, ... }
  pendingThoughts[i]  → { slotId: ..., text, createdAt, expiresAt, ... }
  memberImpressions   → { [name]: { slotId: ..., traits, interactionCount, ... } }

规则：
  slotId 在创建时生成，永不变
  text 可改（改时 slotId 不变）
  slotId 生命周期 = B 层该条记录的生命周期
  
为什么不用 text：
  text 会变（memory-update），变后 bRefs 就失联了
```

## 12.2 A 层 bRefs

```json
"bRefs": [
  {"conversationKey": "private:[OWNER_QQ]", "category": "activeTopic", "slotId": "slot_a1b2c3d4"}
]
```

## 12.3 B 层反向引用

```json
{
  "slotId": "slot_a1b2c3d4",
  "text": "主人在调试 STM32",
  "aMemoryId": "mem_xyz",
  "lastOperationId": "op_abc"
}
```

## 12.4 WAL 恢复按 slotId + opId 验证

```
恢复 B 层时：
  ① 按 slotId 查记录
  ② 比对 lastOperationId === 期望 opId
  ③ 比对 content/text 作为内容校验（不作为身份依据）
  
不再用 text 匹配作为身份判断
```

## 12.5 文本只能用于内容校验

| 用途 | 允许 |
|---|---|
| 身份识别（找哪条） | ❌ 必须用 id/slotId |
| 内容校验（确认是同一内容） | ✅ 可用 contentHash 或 text |
| 用户搜索 | ✅ 可用（但不是身份） |

---

# 十三、检索过滤顺序（完整）

```
① 权限过滤（最先，先于任何计算）
   ownerId 匹配 + visibility 允许 + 群上下文需 shareReviewStatus=approved

② 生命周期过滤
   排除 status ∈ {archived, deleted}（includeArchived 显式放开 archived）
   includeSuperseded 显式放开 superseded
   
③ TTL 过滤（新增）
   排除 expiresAt ≠ null 且 expiresAt < now
   ★ 不依赖清理任务是否运行

④ 审核过滤
   排除 reviewStatus ∈ {pending, rejected}
   
⑤ 候选/冲突过滤（新增）
   排除 isCandidate=true
   排除 conflictGroupId ≠ null（冲突组整体不注入）

⑥ 语义相似度
   cosSim > threshold

⑦ 加权排序
   score × timeWeight × typeWeight
   ★ 时间权重只影响排序

⑧ topK + 裁剪
```

**为什么 ③⑤ 必须在 ⑥ 之前**：
- TTL：过期内容不该参与打分（也避免侧信道）
- 冲突组：有争议的内容不该被注入

---

# 十四、异常场景预演（28 个）

| # | 场景 | V2.3 预期 | 机制 |
|---|---|---|---|
| S1 | 同句写两次 | contentHash 去重 → 更新 updatedAt | append 幂等 |
| S2 | 语义相似但 factKey 不同且都合法 | 都保留，**不 pending**（只记日志） | P1-5 |
| S3 | 同 factKey 相似度低 | factKey 命中 → 按策略处理 | factKey 独立查找 |
| S4 | 两台设备不同 qualifier | 不同 factKey → 都 active | cardinality |
| S5 | 极性相反 | 不 supersede，新记录 pending | 两阶段 |
| S6 | **新事实审核期间** | **旧记录仍可见** | 两阶段阶段1 |
| S7 | 审核批准后 | 原子 supersede（单 op） | 两阶段阶段2 |
| S8 | 审核拒绝 | 新 rejected，旧完全不变 | 两阶段阶段2' |
| S9 | 审核期间又来新候选 | 形成候选队列，同 conflictGroupId | §4.5 |
| S10 | 多 active 同 factKey（single_current） | 打 conflictGroupId，整组不注入 | §3.4 |
| S11 | 关系叠加（multi_valued） | 都 active，无冲突 | cardinality |
| S12 | 两用户相同内容 | ownerId 隔离 | 权限 |
| S13 | 私有记忆被群查 | 拦截 | 权限第① |
| S14 | 群共享副本被其他群查 | 拦截（ownerId 不同） | 权限第① |
| S15 | 个人原件被群查 | 拦截（ownerId 不同） | 权限第① |
| S16 | 删除后 WAL 重试 | 查 tombstone → cancelled | 防线② |
| S17 | 删除后备份恢复 | 对比 tombstone → 跳过 | 防线③ |
| S18 | 删除后重新写同内容（memory_only） | **允许**，生成新 id，记 reAuthorized | §6.4 |
| S19 | 删除后重新写同内容（content_suppression） | 拒绝 | §6.3 |
| S20 | 迟到的 supersede 覆盖已删记录 | revision 不匹配 → 拒绝 | §8.4 |
| S21 | 两个操作并发改同一记录 | 后到的 revision 不匹配 → 拒绝 | §8.4 |
| S22 | WAL compact 中途崩溃 | 按 MANIFEST + 文件完整性判定 | §9.5 |
| S23 | WAL 尾行损坏 | CRC 失败 → 跳过该行 + 日志 | P1-4 |
| S24 | JSONL 尾行半截 | _crc 缺失/不符 → 写路径拒绝，读路径跳过 | P1-4 |
| S25 | 脚本在服务运行时改文件 | 端口检测 → 拒绝 | 单写者 |
| S26 | 模板歧义匹配 | 拒绝加载模板 + 报警 | §2.4 |
| S27 | 分享副本原件被改 | 副本不变（快照），sourceRevision 标差异 | §10.2③ |
| S28 | 群内删副本 | 原件不受影响 | §10.2⑩ |

### S6 详细（核心修复验证）

```
t0  旧记录 O: status=active, reviewStatus=approved, content='主人凌晨2点睡'
t1  写新记录 N: content='主人现在11点睡', factKey 同, conflictPolicy=supersede_with_review
    阶段1：
      N: status=active, reviewStatus=pending, isCandidate=true, conflictGroupId=cg_1
      O: **完全不变**（active + approved）✅

t2  用户提问"我几点睡"：
     检索：
       ①权限 ok
       ②生命周期：O active ✅，N active ✅
       ③TTL ok
       ④审核：O approved ✅，N pending ❌排除
       ⑤候选：N isCandidate ❌排除
       ⑥打分：O 命中
     → 返回 O（凌晨2点睡）✅
     → 事实不中断（虽然可能过时，但至少有答案）

t3  人工批准：
     单 op 原子：
       O: status=superseded, supersededBy=N, revision+1
       N: reviewStatus=approved, isCandidate=false, revision+1

t4  再检索：
     O 被 status 过滤掉，N 返回（11点睡）✅
```

### S20 详细（revision 防护）

```
t0  R: revision=5, status=active
t1  操作 A（用户删除）：
      expectedRevision=5 → 校验通过 → revision=6, status=deleted
t2  操作 B（1 分钟前排队但延迟的 supersede）：
      expectedRevision=5
      → 校验：实际 revision=6 ≠ 5
      → 拒绝，WAL 记 conflict
      → R 保持 deleted ✅

若没有 revision：
  B 会把 deleted 改成 superseded
  → 用户删掉的记忆"复活"成"被替代的旧版本"
  → 检索 includeSuperseded 时可能被看到 ❌
```

---

# 十五、旧工具接口兼容

## 15.1 签名扩展（全可选）

```js
qq_memory_append(key, token, category, content, extra?,
                 type?, candidateFactKey?, occurredAt?, subjectId?)
qq_memory_query(key, token, category?, topic?, includeSuperseded?, includeArchived?)
qq_memory_remove(key, token, category, content?, target?, memoryId?, 
                 suppression?, hard?)
qq_memory_clear(key, token, category?, mode?)
qq_memory_share_request(key, token, memoryId?, text?)   ← 优先 memoryId
qq_memory_manage(op, ..., reason?)                       ← 新增
```

## 15.2 字段映射

| 旧 | 新 |
|---|---|
| `key` | `conversationKey`（格式不变） |
| `text` | `content`（去 category 前缀） |
| `time` | `createdAt` |
| `needs_review`(bool) | `reviewStatus`（true→pending） |
| `share_review_status` | `shareReviewStatus` |

## 15.3 旧数据双读

```js
function normalizeRow(raw) {
  const content = raw.content ?? stripCategoryPrefix(raw.text);
  return {
    ...raw,
    id: raw.id ?? legacyId(raw),
    schemaVersion: raw.schemaVersion ?? 1,
    revision: raw.revision ?? 1,
    ownerId: raw.ownerId ?? deriveOwner(raw.key),
    conversationKey: raw.conversationKey ?? raw.key,
    content,
    contentHash: raw.contentHash ?? sha256(normalize(content)),
    reviewStatus: raw.reviewStatus ?? (raw.needs_review === true ? 'pending' : 'approved'),
    status: raw.status ?? 'active',
    isCandidate: false,
    conflictGroupId: null,
    lifecycleEvents: raw.lifecycleEvents ?? [{from:null,to:'active',at:raw.time??raw.createdAt}],
    statusChangedAt: raw.statusChangedAt ?? (raw.time ?? raw.createdAt),
  };
}

// 必须确定性
function legacyId(raw) {
  return 'mem_legacy_' + sha256(`${raw.key}|${raw.text}|${raw.time}`).slice(0,12);
}
```

---

# 十六、P0-P6 实施与回滚（修订）

## 16.1 依赖图

```
P0 基础设施（锁/tmp/_crc/单写者约束）
 └─► P1 数据模型 + 双读 + 基础过滤
      ├─► P2 WAL + revision + 幂等 + generation  ★所有后续依赖
      │    ├─► P3 去重 + factKey 模板 + 两阶段事实更新
      │    └─► P4 删除 + tombstone + 归档 + 审计
      │         └─► P6 分享副本 + slotId + 跨层引用
      └─► P5 TTL 过滤 + 加权排序 + 历史检索
```

## 16.2 各批次详情与回滚

### P0：基础设施

| 项 | 内容 |
|---|---|
| 改动 | `memory-lock.js`（进程内链 + 单写者检测）；修 `atomicRewrite` tmp 名；`_crc` 计算/校验函数 |
| 验收 | 并发 100 次写入不丢；`_crc` 能检出篡改；现有测试全过 |
| **代码回滚** | 覆盖文件（纯新增+小改） |
| **数据回滚** | 不需要（无格式变化） |
| **人工处理** | 无 |

### P1：数据模型

| 项 | 内容 |
|---|---|
| 改动 | 字段补齐；`normalizeRow` 双读；`searchVectorMemory` 加 status/review/TTL 过滤；`findMemoryRow` 修正数字被抹 |
| 验收 | 新记录带全字段；旧记录可读；archived/deleted/pending 不检索 |
| **代码回滚** | 覆盖文件 |
| **数据回滚** | 不需要（只加字段，旧代码读时忽略新字段） |
| **人工处理** | 无 |

### P2：WAL + revision（关键）

| 项 | 内容 |
|---|---|
| 改动 | `memory-wal.js`；generation + MANIFEST；revision 乐观锁；三层状态；崩溃恢复 |
| 验收 | 5 个崩溃恢复案例全过；并发冲突被 revision 拒绝；compact 崩溃可恢复 |
| **代码回滚** | ⚠️ **不能直接删 WAL**。流程：<br>① 检查所有 op 是否终态<br>② 有非终态 → 必须先处理（完成或人工 cancelled）<br>③ 全部终态 → 保留 WAL 文件（新代码不读，文件在）<br>④ 覆盖代码 |
| **数据回滚** | WAL 文件保留不动（旧代码不读，无害） |
| **人工处理** | 若有非终态 op 无法处理 → 停服，人工检查 WAL，决定完成或取消 |

**为什么不能删 WAL**：删了就丢失"哪些操作已执行"的依据，可能导致重复执行。

### P3：去重 + factKey + 两阶段

| 项 | 内容 |
|---|---|
| 改动 | 精确去重；模板解析；两阶段更新；`fact-keys.json` |
| 验收 | S3/S4/S6/S7/S8 全过；factKey 模板歧义检测 |
| **代码回滚** | 覆盖代码 + 删 `fact-keys.json` |
| **数据回滚** | ⚠️ **关键**：旧代码会把 `status=superseded` 当 active 读<br>→ 必须把 superseded 改回 active（保留 supersededAt 供追溯）<br>→ 把 isCandidate=true 的记录标 rejected 或删除 |
| **人工处理** | 若回滚时发现有 pending 候选 → 人工决定批准还是拒绝（旧代码不支持候选概念） |

### P4：删除 + tombstone

| 项 | 内容 |
|---|---|
| 改动 | 逻辑删除；tombstone；归档；force_restore + 审计 |
| 验收 | S16-S19 全过；四道防线验证 |
| **代码回滚** | 覆盖代码 |
| **数据回滚** | ⚠️ **tombstone 必须保留**（不删！）<br>否则回滚后防复活失效<br>已 deleted 的记录保持 deleted（旧代码读时按不存在处理——但旧代码没有 status 过滤，会读出来！）<br>→ 需把 deleted 记录物理移出文件，或标特殊 content |
| **人工处理** | 与 P3 类似，需处理 deleted 记录的可见性 |

**P4 回滚的难题**：旧代码没有 `status` 字段的过滤逻辑，会把 `status=deleted` 的记录当正常记忆读出来。
→ 解法：回滚前把 deleted 记录移到 `state/memory-deleted-archive.jsonl`（单独文件），主文件不含它们。

### P5：TTL + 加权

| 项 | 内容 |
|---|---|
| 改动 | TTL 过滤；timeWeight/typeWeight |
| 验收 | 过期不注入；排序合理；不影响版本判断 |
| **代码回滚** | 覆盖代码 |
| **数据回滚** | 不需要（无格式变化） |

### P6：分享副本 + slotId

| 项 | 内容 |
|---|---|
| 改动 | share_copy 流程；slotId；bRefs |
| 验收 | S27/S28 过；副本权限正确；原件不受影响 |
| **代码回滚** | 覆盖代码 |
| **数据回滚** | ⚠️ 清空 `bRefs` 和 `aMemoryId`（旧代码不读，但会残留）<br>分享副本：`source=share_copy` 的记录可删除（旧代码不区分来源，会把副本当普通群记忆） |
| **人工处理** | 记录哪些 bRefs 被清（便于恢复） |

---

# 十七、数据矛盾核实结果（P1-7）

## 17.1 核实结论

`[已确定]` **101 条与 0 字节不是矛盾，是不同时间的不同文件。**

| 项 | 结果 |
|---|---|
| **101 条的来源文件** | `D:\qq-bridge\backups\memory-vectors-2026-10-30.jsonl` |
| **文件大小** | 2,076,124 字节 |
| **条数** | 101（实测，逐行解析） |
| **needs_review 分布** | `true` = 101/101（**全部 true**） |
| **visibility 分布** | `owner_private` 71 / `group_shared` 30 |
| **字段** | key/text/time/vec/category/subjectId/subjectName/subjectType/sourceType/visibility/needs_review（11 个） |
| **统计时间** | 文件 mtime = 2026-10-30（dailyBackup 当天首次写入前快照） |
| **是否当前运行数据** | ❌ **否**。当前 `state/memory-vectors.jsonl` = 0 字节（10-31 05:18:44） |
| **是否历史备份** | ✅ **是**。位于项目 `backups/` 目录，由 `dailyBackup()` 产生 |
| **是否需要迁移** | ❌ 不需要。当前无运行数据 |

## 17.2 时间线（实测的各快照）

```
10-30          101 条   backups/memory-vectors-2026-10-30.jsonl   ← V2.2 §11.7 引用的就是这个
10-31 00:22    112 条   D:\qq-bridge备份-*\memory-vectors-执行前全量-20261031.jsonl
10-31 00:42    101 条   D:\qq-bridge备份-*\memory-vectors-P0迁移前-20261031.jsonl
10-31 03:07    102 条   D:\qq-bridge备份-*\memory-vectors-P0前-20261031.jsonl（P0 审计时的状态）
10-31 05:15     9 条   D:\qq-bridge备份-*\memory-vectors-移除的身份冲突-20261031-051541.jsonl
10-31 05:17    18 条   D:\qq-bridge备份-*\memory-vectors-移除的群友与冲突-20261031-051744.jsonl
10-31 05:18    75 条   D:\qq-bridge备份-*\memory-vectors-全部清空前-20261031-051844.jsonl
10-31 05:18     0 条   state/memory-vectors.jsonl（当前）
```

## 17.3 V2.2 表述的问题

```
V2.2 §11.7 写："needs_review = true，共 101 条"
未说明这是【历史备份】的数据，容易被误读为当前状态。

V2.3 修正：
  明确标注数据来源 = backups/memory-vectors-2026-10-30.jsonl
  明确标注非当前运行数据
  明确当前为 0 条
```

## 17.4 本轮未做

```
✅ 只读核实（读文件 + 统计），未恢复、未导入、未修改任何数据
✅ 未把备份数据导入当前库
✅ 未修改任何文件
```

---

# 十八、尚未确定的问题

| # | 问题 | 候选 | 影响 | 优先级 |
|---|---|---|---|---|
| D1 | `multi_valued` 的准确语义：per-factKey 还是 per-entity | a) per-factKey=single + per-entity=multi<br>b) 模板显式 per-factKey 基数 | 决定 device 和 member_relation 的行为 | **高** |
| D2 | `member_relation` 是否需要 qualifier（`member.<qq>.relation.<type>`） | a) 加<br>b) 保持 + 声明 per-factKey=multi 例外 | 关系变化能否表达 | **高** |
| D3 | 哪些策略可以不审核（`supersede` 直接执行） | a) 仅 bot.persona<br>b) 全部需审核 | 安全 vs 效率 | 中 |
| D4 | WAL 是否存原始 content | a) 完整<br>b) 只存 hash<br>c) 前 100 字 | 隐私 vs 恢复能力 | **高** |
| D5 | 分享副本是否带 factKey | a) 不带（推荐）<br>b) 改写 scope | factKey 一致性 | 中 |
| D6 | 离线迁移模式是否保留 | a) 保留（带强制检查）<br>b) 只允许停服改文件 | 运维便利 vs 安全 | 中 |
| D7 | tombstone 是否永久保留 | a) 永久<br>b) 90 天后清理 | 防复活时效 | 中 |
| D8 | `_crc` 是否强制 | a) 强制（新记录）<br>b) 可选 | 兼容性 | 低 |
| D9 | conflictGroup 是否整组不注入 | a) 整组排除（推荐）<br>b) 只排除 isCandidate | 冲突可见性 | 中 |
| D10 | force_restore 后旧的 superseded 关系是否恢复 | a) 不恢复<br>b) 一并恢复 | 语义完整性 | 低 |

---

# 十九、无法闭合的设计问题（如实声明）

| # | 问题 | 为什么无法闭合 |
|---|---|---|
| N1 | **C 层 Session 的隐私擦除** | bridge 无 DSH Session 的写权限；只能通过 DSH 管理端或重置会话。A 层删除永远无法覆盖 C 层 |
| N2 | **备份的隐私擦除** | 历史备份是静态文件，改写它们需要重写（破坏"备份不可变"原则）。只能提供手动工具，且需人工确认 |
| N3 | **multi_valued 与"关系结束"的表达** | 系统无法自动判断"不再是同事"。只能靠人工 archive 或模型用自然语言表达 |
| N4 | **事实更新的"对错"判断** | 系统无法验证"11 点睡"是否真的是新作息。只能靠人工审核 |
| N5 | **旧代码读新数据** | P3/P4 回滚后，旧代码会把 superseded/deleted 当 active。必须配合数据回滚（移出文件），不能纯代码回滚 |
| N6 | **跨进程的绝对排除** | 单写者靠"协作"（脚本自觉让路），无法物理阻止恶意进程直接改文件 |

---

# 二十、本轮交付确认

## 20.1 相对 V2.2 的实质改动（10 项）

1. **factKey 模板驱动解析**（不再统一位置语法），六步验证顺序，模板歧义防护
2. **cardinality（single_current/multi_valued）** 替代含混的 keep_both，异常检测只对 single
3. **两阶段事实更新**：阶段 1 旧记录保持可见，阶段 2 人工批准后原子切换 —— 修复"两条都不可见"的致命缺陷
4. **lifecycleEvents** 替代三时间字段互斥规则，历史不丢失
5. **tombstone 拆 suppression**（memory_only / content_suppression），默认只防复活
6. **隐私清除三级**（logical/content/cross_store），明确 hard ≠ 彻底删除
7. **revision 乐观锁**，旧操作不能覆盖新状态；failed 拆 retriable/exhausted
8. **WAL generation + MANIFEST**，compact 崩溃可判定
9. **分享副本方案 A** 完整定义，权限矩阵重写
10. **单写者架构**替代跨进程锁

## 20.2 仍需人工决策（10 项）

见 §十八，优先级最高的是 D1（multi_valued 语义）、D2（member_relation qualifier）、D4（WAL 是否存原文）。

## 20.3 无法闭合的设计问题（6 项）

见 §十九，全部是"技术上做不到"或"依赖外部系统"，不是设计缺陷。

## 20.4 严格只读声明

```
✅ 本轮只读：
   - 读取了 src/memory-vector.js（全文）、src/bridge.js（相关段落）、
     state/memory-vectors.jsonl、state/social-v2.json、
     backups/memory-vectors-2026-10-30.jsonl、D:\qq-bridge备份-* 下的快照
   - 只执行了纯读取的统计脚本（读文件、计数、算 md5）
   
✅ 未修改：代码 / 配置 / 数据 / Persona / rules / preset
✅ 未清空、未迁移、未生成测试记忆
✅ 未启动或重启任何服务（QQ Bridge / DSH / Ollama 均未触碰）
✅ 只新增了本文档一个文件
```

---

*本文档完成后立即停止。不进入实施阶段，等待人工审核。*
