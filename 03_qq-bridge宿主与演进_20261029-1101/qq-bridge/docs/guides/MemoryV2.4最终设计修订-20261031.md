# Memory V2.4 最终设计修订

- 修订时间：2026-10-31 12:55
- 前置：《Memory V2.3 设计修订》
- 性质：**纯设计文档。** 本轮严格只读，未修改任何代码/配置/数据，未重启服务。
- 标注：`[已确定]` / `[候选方案]` / `[待决策]`

---

# 〇、V2.3 → V2.4 修订对照

| # | V2.3 问题 | V2.4 修订 |
|---|---|---|
| P0-1 | 单一 `cardinality` 无法同时表达"多设备"和"每设备一个当前值" | 拆为 `perFactKeyCardinality` + `perEntityCardinality` 双层；member_relation 加关系类型维度 |
| P0-2 | 阶段二仍可能"旧已 superseded + 新仍 pending" | **单文件事务 = 一次原子重写**；跨文件靠状态字段过滤，不靠事务可见性 |
| P0-3 | 原件 `shareReviewStatus` 无法表达多群独立审批 | 引入独立 `ShareRequest` 对象；原件永保 `owner_private`；原件删除时级联撤销副本 |
| P0-4 | WAL 不存正文，但没定义恢复边界 | payload 最小字段表 + 五档恢复判定 + `recovery_blocked` 状态 |
| P0-5 | tombstone 未纳入检索硬过滤 | tombstone 进检索第③层；tombstone 三状态（active/restored/purged） |
| P0-6 | 端口检测有竞态 | 统一 `writer.lock`，bridge 与离线脚本互斥；原子创建 + 失败关闭 |
| P1-1 | `_crc` 计算未定义规范化 | 明确排除自身 + 字典序序列化 + legacy 识别 |
| P1-2 | lifecycleEvents 可能混入审核事件 | 严格分工：生命周期→lifecycleEvents，审核→audit |
| P1-3 | normalize 未定义 | NFC + 换行统一 + 空格折叠 + **不删数字** |
| P1-4 | topic 模板 subjectId 未映射 | 明确 `subjectId='topic:<slug>'` + `factKeyTemplateVersion` |
| P1-5 | 检索顺序未含 tombstone/分享副本有效性 | 11 层严格顺序 |
| 修-1 | §10.4 矩阵第 2 行与 C2 矛盾 | **删除该行**（原件永为 owner_private） |
| 修-5 | `vec=[]` 违反 1024 维 schema | `vec=[]` + `vecPurged=true`，字段表改为条件必填 |
| 修-6 | 批准时未关闭 conflictGroup | 明确关闭组 + 其余候选 rejected |

---

# 一、最终数据模型

## 1.1 字段表（V2.4 定稿）

| 字段 | 类型 | 默认 | 必填条件 | 说明 | 状态 |
|---|---|---|---|---|---|
| `id` | string | 生成 | 总是 | `mem_<base36>_<rand6>` | `[已确定]` |
| `schemaVersion` | number | 4 | 总是 | | `[已确定]` |
| `revision` | number | 1 | 总是 | 乐观锁版本 | `[已确定]` |
| `ownerId` | string | — | 总是 | `owner:<qq>` / `group:<群号>` | `[已确定]` |
| `conversationKey` | string | — | 总是 | 产生上下文 | `[已确定]` |
| `subjectId` | string | `unknown` | 总是 | `owner`/`member:<qq>`/`bot`/`topic:<slug>` | `[已确定]` |
| `visibility` | enum | `owner_private` | 总是 | `owner_private` / `group_shared` | `[已确定]` |
| `category` | enum | `unknown` | 总是 | activeTopic/pendingThought/memberImpression/manual/unknown | `[已确定]` |
| `type` | enum | `event` | 总是 | fact/event/preference/temporary/relationship | `[已确定]` |
| `status` | enum | `active` | 总是 | active/superseded/archived/deleted | `[已确定]` |
| `reviewStatus` | enum | `approved` | 总是 | pending/approved/rejected | `[已确定]` |
| `shareReviewStatus` | enum\|null | **null** | 总是 | **V2.4 废弃此字段**，改为 ShareRequest（见 §12） | `[已确定]` |
| `content` | string | — | 总是 | 正文；清空后为 `'[deleted]'` | `[已确定]` |
| `contentHash` | string | 自动 | 总是 | sha256(normalize(content)) 全 64 位 | `[已确定]` |
| `contentPurged` | boolean | false | 总是 | 内容是否被清除（L2） | `[已确定]` |
| `_crc` | string | 自动 | 总是 | 行校验 | `[已确定]` |
| `createdAt` | number | now | 总是 | | `[已确定]` |
| `occurredAt` | number\|null | null | — | | `[已确定]` |
| `updatedAt` | number | now | 总是 | | `[已确定]` |
| `statusChangedAt` | number | now | 总是 | | `[已确定]` |
| `lifecycleEvents` | object[] | `[]` | 总是 | 生命周期历史（见 §五） | `[已确定]` |
| `supersededAt` | number\|null | null | 见 C5 | | `[已确定]` |
| `archivedAt` | number\|null | null | 见 C5 | | `[已确定]` |
| `deletedAt` | number\|null | null | 见 C5 | | `[已确定]` |
| `expiresAt` | number\|null | null | — | temporary TTL | `[已确定]` |
| `factKey` | string\|null | null | — | | `[已确定]` |
| `factKeyTemplate` | string\|null | null | — | 模板 id | `[已确定]` |
| `factKeyTemplateVersion` | number\|null | null | — | 模板版本 | `[已确定]` |
| `perFactKeyCardinality` | enum\|null | null | — | 从模板继承 | `[已确定]` |
| `perEntityCardinality` | enum\|null | null | — | 从模板继承 | `[已确定]` |
| `supersedes` | string\|null | null | — | | `[已确定]` |
| `supersededBy` | string\|null | null | — | | `[已确定]` |
| `conflictGroupId` | string\|null | null | — | | `[已确定]` |
| `isCandidate` | boolean | false | 总是 | | `[已确定]` |
| `source` | enum | `model` | 总是 | model/manual/migration/import/share_copy | `[已确定]` |
| `sourceMemoryId` | string\|null | null | source=share_copy 时必填 | | `[已确定]` |
| `sourceRevision` | number\|null | null | source=share_copy 时必填 | | `[已确定]` |
| `shareRequestId` | string\|null | null | source=share_copy 时必填 | 指向 ShareRequest | `[已确定]` |
| `bRefs` | object[] | `[]` | — | `[{conversationKey, category, slotId}]` | `[已确定]` |
| `opId` | string\|null | null | — | | `[已确定]` |
| `vec` | number[] | — | **见 C9** | 1024 维；purged 时为 `[]` | `[已确定]` |
| `vecModel` | string\|null | `bge-m3` | 见 C9 | | `[已确定]` |
| `vecPurged` | boolean | false | 总是 | 向量是否被清除 | `[已确定]` |

## 1.2 约束（V2.4 完整）

```
C1  ownerId 与 conversationKey 同源：
      private:X → owner:owner:X   且 conversationKey:'private:X'
      group:X   → owner:group:X   且 conversationKey:'group:X'

C2  visibility=group_shared 的必要条件（修订）：
      ownerId 以 'group:' 开头
      AND source='share_copy'
      AND shareRequestId ≠ null
    ★ 原件永远是 owner_private。V2.3 的"原件+group_shared"状态在 V2.4 不存在。
    ★ 因此 V2.3 §10.4 矩阵第 2 行【删除】。

C3  factKey 必须匹配唯一模板；模板的 subject 段必须与 subjectId 一致
      （topic 模板：factKey 第二段 = subjectId 的 slug 部分）

C4  factKey 允许的 type 由模板 allowedTypes 决定

C5  状态与时间字段：
      status='superseded' → supersededAt = statusChangedAt ≠ null
      status='archived'   → archivedAt   = statusChangedAt ≠ null
      status='deleted'    → deletedAt    = statusChangedAt ≠ null
      status='active'     → statusChangedAt ≥ 所有非 null 时间字段
      三个时间字段【不要求互斥】（历史痕迹）

C6  revision 单调递增；任何变更 +1

C7  conflictGroupId 非 null 或 isCandidate=true → 不参与自动注入

C8  source='share_copy' → sourceMemoryId / sourceRevision / shareRequestId 必填，
      且 factKey 必须为 null

C9  vec 与 contentPurged 的联动（修正 V2.3 的 schema 冲突）：
      contentPurged=false → vec 必须是 1024 维数组，vecModel='bge-m3'，vecPurged=false
      contentPurged=true  → vec 必须是 []，vecPurged=true
      ★ 校验规则：`vec.length===1024 XOR vecPurged===true`
      ★ 检索时：`Array.isArray(r.vec) && r.vec.length===1024` 才参与打分

C10 shareReviewStatus 字段废弃（保留读取兼容，写时不再产生）
      分享状态一律从 ShareRequest 查（按 targetOwnerId 区分）
```

---

# 二、双层 cardinality（P0-1）

## 2.1 两个维度

`[已确定]`

```
perFactKeyCardinality   同一个 factKey 允许多少个 status=active 记录
  single_current  只允许 1 个
  multi_valued    允许多个

perEntityCardinality    同一个 (subjectId, entity 段) 下允许多少个不同 factKey 的 active
  single          只允许 1 个
  multi           允许多个
```

**两者正交**：

| perFactKey | perEntity | 含义 | 例子 |
|---|---|---|---|
| single_current | single | 整个实体只有一个当前值 | （少见） |
| single_current | multi | 实体有多个实例，每实例一个当前值 | **device**（多台设备，每台一个 GPU） |
| multi_valued | multi | 每实例可多条，实例可多个 | （关系历史场景） |

## 2.2 模板继承

```
模板声明两个字段：
  "perFactKeyCardinality": "single_current",
  "perEntityCardinality": "multi"

记录写入时，从匹配到的模板继承这两个值，存入记录
（记录自带副本，模板后续修改不影响已有记录的解释）
```

## 2.3 conflictPolicy 与两个基数的兼容规则

| perFactKey | 允许的 conflictPolicy |
|---|---|
| `single_current` | `supersede` / `supersede_with_review` / `manual_review` |
| `multi_valued` | `keep_both`（唯一） |

**不兼容组合 → 模板加载失败**：
- `single_current` + `keep_both`（矛盾：单值却要都保留）
- `multi_valued` + `supersede`（矛盾：多值却要替代）

## 2.4 单 factKey 多条 active 何时合法/异常

```
合法：perFactKeyCardinality = multi_valued
异常：perFactKeyCardinality = single_current 且 active 条数 > 1
      → 打 conflictGroupId（见 §3.4），不改 reviewStatus

perEntity 多条 active：
  合法：perEntityCardinality = multi
  异常：perEntityCardinality = single 且不同 factKey 的 active > 1
        → 同样打 conflictGroupId
```

## 2.5 同一实体多个 qualifier 如何共存

```
device 模板：perEntity=multi
  entity 段 = 'device'（literal）
  qualifier 段 ∈ {desktop, lab, phone, work, home}

  owner.device.desktop.gpu  ← 实例1
  owner.device.lab.gpu      ← 实例2
  owner.device.phone.os     ← 实例2 的另一属性

  → 三个不同 factKey，各自 perFactKey=single_current（每实例每属性一个当前值）
  → 共存合法 ✅
  
  owner.device.desktop.gpu 只能有 1 个 active
  → 写第二台台式机？不行，应该用另一个 qualifier（如 'home'）
```

## 2.6 factKey 版本替代 vs 实体实例增加的区别

| 动作 | factKey 变化 | cardinality 作用 |
|---|---|---|
| 同一设备升级显卡 | **不变**（`owner.device.desktop.gpu`） | perFactKey=single → supersede |
| 新增一台设备 | **变**（新增 qualifier） | perEntity=multi → 允许共存 |
| 新增一个属性（如显示器） | **变**（新增 attribute） | perEntity=multi → 允许共存 |

## 2.7 四个案例

### 案例 A：设备多实例（合法共存）

```
t0  owner.device.desktop.gpu = '[GPU]'
    模板 device: perFactKey=single, perEntity=multi
    ②b factKey 查找 → 0 条 → 新增
    → 1 条 active

t1  owner.device.lab.gpu = 'RTX 4090'
    ②b 查找 'owner.device.lab.gpu' → 0 条 → 新增
    → 2 条 active（不同 factKey）

t2  perEntity 检查：同 subject='owner' + entity='device' 下 2 条
    → perEntity=multi → 合法 ✅
```

### 案例 B：同一设备升级（需要 supersede）

```
t0  owner.device.desktop.gpu = '[GPU]'（active, revision=1）

t1  owner.device.desktop.gpu = '[GPU]'，候选 factKey 同
    ②b 查找 → 命中 1 条（factKey 全等）
    ③a perFactKey=single_current 且命中
        conflictPolicy=supersede_with_review
        → 两阶段：
          阶段1：新记录 active+pending+isCandidate+cg_1
                 旧记录【不动】（active+approved）
          阶段2（人工批准）：
                 一个事务：旧→superseded，新→approved
```

### 案例 C：关系叠加（合法共存，需类型维度）

```
模板 member_relation：
  pattern: member.<qq>.relation.<type>
  perFactKey=single_current   （每类关系一个当前值）
  perEntity=multi             （可同时有多种关系）
  types: [classmate, teammate, friend, colleague, ex_colleague, family, online]

t0  member.<A>.relation.classmate = '大学同学'     → active
t1  member.<A>.relation.teammate  = '同一个电竞队' → active
t2  member.<A>.relation.friend    = '关系不错'     → active
→ 3 条 active，3 个不同 factKey，perEntity=multi → 全部合法 ✅

检索"和 A 什么关系" → 返回 3 条，模型能看到多重关系
```

**为什么加 `<type>` 段**：
V2.3 的 `member.<qq>.relation` 没有类型维度 → 同 qq 的所有关系共用一个 factKey → 无法叠加。V2.4 必须加。

### 案例 D：关系变化（四种情况分开处理）

| 子场景 | 具体 | 处理 |
|---|---|---|
| **D1 新增另一种关系** | 从"同事"变成"同事+朋友" | 新写 `relation.friend` → 新 factKey → 新增 ✅ |
| **D2 明确结束旧关系** | "已经从公司离职，不再是同事" | **人工** archive `relation.colleague`（或写 `relation.ex_colleague`）<br>**系统不自动 archive** |
| **D3 明确替代原关系** | "她现在是前同事" | 写 `relation.ex_colleague`（新 factKey）<br>同时人工 archive `relation.colleague`<br>★ 不是 supersede，因为是两个不同 factKey |
| **D4 只是描述发展** | "我们关系更好了" | type=event，不进关系模板<br>`factKey=null` |

**明确禁止**：仅凭自然语言语义相似自动结束旧关系。

```
反例（必须避免）：
  已有 `relation.colleague`
  新写 "离职了" → 若系统自动推断"这条否定了同事关系" → 自动 archive ❌
  
正确：
  新写 "离职了" → 作为 event 存入（无 factKey）
  人工看到后决定是否 archive colleague
```

---

# 三、factKey 模板完整定义（P0-1 配套 + P1-4）

## 3.1 模板文件

`state/fact-keys.json`：

```json
{
  "version": 4,
  "templates": [
    {
      "id": "device",
      "templateVersion": 1,
      "description": "设备配置",
      "arity": 4,
      "segments": [
        {"pos": 0, "role": "subject", "enum": ["owner"]},
        {"pos": 1, "role": "literal", "value": "device"},
        {"pos": 2, "role": "qualifier", "enum": ["desktop", "lab", "phone", "work", "home"]},
        {"pos": 3, "role": "attribute", "enum": ["gpu", "cpu", "ram", "disk", "os", "monitor"]}
      ],
      "perFactKeyCardinality": "single_current",
      "perEntityCardinality": "multi",
      "conflictPolicy": "supersede_with_review",
      "allowedTypes": ["fact"],
      "enabled": true
    },
    {
      "id": "preference",
      "templateVersion": 1,
      "arity": 3,
      "segments": [
        {"pos": 0, "role": "subject", "enum": ["owner"]},
        {"pos": 1, "role": "literal", "value": "preference"},
        {"pos": 2, "role": "attribute", "enum": ["food", "game", "music", "sleep", "social", "work"]}
      ],
      "perFactKeyCardinality": "single_current",
      "perEntityCardinality": "multi",
      "conflictPolicy": "supersede_with_review",
      "allowedTypes": ["fact", "preference"],
      "enabled": true
    },
    {
      "id": "schedule",
      "templateVersion": 1,
      "arity": 3,
      "segments": [
        {"pos": 0, "role": "subject", "enum": ["owner"]},
        {"pos": 1, "role": "literal", "value": "schedule"},
        {"pos": 2, "role": "attribute", "enum": ["sleep", "meal", "study"]}
      ],
      "perFactKeyCardinality": "single_current",
      "perEntityCardinality": "multi",
      "conflictPolicy": "supersede_with_review",
      "allowedTypes": ["fact"],
      "enabled": true
    },
    {
      "id": "member_relation",
      "templateVersion": 2,
      "arity": 4,
      "segments": [
        {"pos": 0, "role": "literal", "value": "member"},
        {"pos": 1, "role": "subjectId_qq", "pattern": "^\\d{5,12}$"},
        {"pos": 2, "role": "literal", "value": "relation"},
        {"pos": 3, "role": "qualifier",
         "enum": ["classmate", "teammate", "friend", "colleague", "ex_colleague", "family", "online"]}
      ],
      "perFactKeyCardinality": "single_current",
      "perEntityCardinality": "multi",
      "conflictPolicy": "supersede_with_review",
      "allowedTypes": ["relationship"],
      "enabled": true
    },
    {
      "id": "bot_persona",
      "templateVersion": 1,
      "arity": 3,
      "segments": [
        {"pos": 0, "role": "subject", "enum": ["bot"]},
        {"pos": 1, "role": "literal", "value": "persona"},
        {"pos": 2, "role": "attribute", "enum": ["name", "mood", "habit"]}
      ],
      "perFactKeyCardinality": "single_current",
      "perEntityCardinality": "multi",
      "conflictPolicy": "supersede_with_review",
      "allowedTypes": ["fact", "preference"],
      "enabled": true
    },
    {
      "id": "topic_attr",
      "templateVersion": 1,
      "arity": 3,
      "segments": [
        {"pos": 0, "role": "literal", "value": "topic"},
        {"pos": 1, "role": "slug", "pattern": "^[a-z0-9-]{2,32}$"},
        {"pos": 2, "role": "attribute", "enum": ["issue", "status", "deadline", "owner"]}
      ],
      "perFactKeyCardinality": "single_current",
      "perEntityCardinality": "multi",
      "conflictPolicy": "supersede_with_review",
      "allowedTypes": ["fact", "event", "temporary"],
      "enabled": true,
      "subjectIdMapping": "topic:{segment1}"
    }
  ]
}
```

## 3.2 topic 模板的 subjectId 映射（P1-4）

```
规则：模板有 subjectIdMapping 时，用它从 factKey 段推导 subjectId

topic_attr: subjectIdMapping = "topic:{segment1}"
  factKey = 'topic.stm32-debug.issue'
  → subjectId 必须是 'topic:stm32-debug'

校验（C3 的具体实现）：
  if (template.subjectIdMapping) {
    expected = template.subjectIdMapping.replace('{segment1}', segments[1]);
    if (record.subjectId !== expected) → REJECT_SUBJECT_MISMATCH
  } else {
    // 用 role=subject 的段值
    subjectSeg = segments.find(s => s.role === 'subject');
    map subjectSeg 值 → subjectId:
      'owner' → 'owner'
      'bot'   → 'bot'
    // 或 role=subjectId_qq → 'member:<qq>'
  }
```

## 3.3 模板版本管理

```
字段：
  version            模板文件整体版本（每次变更 +1）
  templateVersion    单个模板的版本（该模板变更时 +1）

记录里存 factKeyTemplate + factKeyTemplateVersion：
  记录写入时的模板快照标识

模板修改后的旧记录如何解释：
  ① 按记录里存的 templateVersion 找到历史模板
  ② 用历史模板重新验证该记录的 factKey（用于一致性检查，不改数据）
  ③ 新写入用当前版本

历史模板保留：
  state/fact-keys-history/v<version>.json
  保留最近 10 个版本

禁用模板：
  enabled=false → 新写入不能匹配（REJECT_NO_TEMPLATE）
  已有记录不受影响（factKey / templateVersion 保留，仍可检索）

修改模板的权限：
  只有 consoleToken；每次写 state/fact-keys-audit.jsonl
```

## 3.4 六步验证（顺序不可换）

```
① 语法粗筛：段数 3~5，每段非空，全小写 [a-z0-9-]
② 模板匹配：遍历 enabled 模板，要求 arity 相同 + 每段 role 校验通过
   → 0 个 = REJECT_NO_TEMPLATE
   → ≥2 个 = REJECT_AMBIGUOUS（配置错误）
③ 主体验证：按 §3.2 的映射规则
④ 类型验证：type ∈ allowedTypes
⑤ 权限验证：同 factKey 已有记录的 ownerId + visibility 必须一致
⑥ 通过：写入 factKey + template + templateVersion + 两个 cardinality
```

## 3.5 各模板合法/非法示例

### device（arity=4）

| 示例 | 判定 |
|---|---|
| `owner.device.desktop.gpu` | ✅ |
| `owner.device.lab.cpu` | ✅ |
| `owner.device.phone.os` | ✅ |
| `owner.device.gpu` | ❌ NO_TEMPLATE（3 段，device 要 4 段） |
| `owner.device.server.gpu` | ❌ NO_TEMPLATE（server 不在枚举） |
| `owner.device.desktop.gpu.more` | ❌ 语法（5 段但无模板匹配） |

### preference（arity=3）

| 示例 | 判定 |
|---|---|
| `owner.preference.food` | ✅ |
| `owner.preference.game` | ✅ |
| `owner.preference.sleep` | ✅ |
| `owner.preference.desktop` | ❌ NO_TEMPLATE（desktop 不在 attribute） |
| `owner.preference` | ❌ 语法（2 段） |
| `owner.preference.food.spicy` | ❌ NO_TEMPLATE（4 段） |

### schedule（arity=3）

| 示例 | 判定 |
|---|---|
| `owner.schedule.sleep` | ✅ |
| `owner.schedule.meal` | ✅ |
| `owner.schedule.study` | ✅ |
| `owner.schedule.exercise` | ❌ NO_TEMPLATE |
| `owner.schedule` | ❌ 语法 |
| `bot.schedule.sleep` | ❌ SUBJECT_MISMATCH（subject 需 owner） |

### member_relation（arity=4，V2.4 新增 type 段）

| 示例 | 判定 |
|---|---|
| `member.[OWNER_QQ].relation.friend` | ✅ |
| `member.12345.relation.classmate` | ✅（qq 5 位边界） |
| `member.123456789012.relation.family` | ✅（qq 12 位边界） |
| `member.[OWNER_QQ].relation` | ❌ NO_TEMPLATE（3 段，需 4 段） |
| `member.abc.relation.friend` | ❌ 语法（qq 非数字） |
| `member.1234.relation.friend` | ❌ 语法（qq < 5 位） |
| `member.[OWNER_QQ].relation.boss` | ❌ NO_TEMPLATE（boss 不在枚举） |

### bot_persona（arity=3）

| 示例 | 判定 |
|---|---|
| `bot.persona.name` | ✅ |
| `bot.persona.mood` | ✅ |
| `bot.persona.habit` | ✅ |
| `owner.persona.name` | ❌ SUBJECT_MISMATCH |
| `bot.persona` | ❌ 语法 |
| `bot.persona.hobby` | ❌ NO_TEMPLATE |

### topic_attr（arity=3）

| 示例 | 判定 |
|---|---|
| `topic.stm32-debug.issue` | ✅（subjectId 必须是 `topic:stm32-debug`） |
| `topic.reset.status` | ✅ |
| `topic.stm32-debug.deadline` | ✅ |
| `topic.STM32.issue` | ❌ 语法（大写） |
| `topic.a.issue` | ❌ 语法（slug < 2） |
| `topic.stm32-debug.priority` | ❌ NO_TEMPLATE |

---

# 四、两阶段事实更新完整事务（P0-2）

## 4.1 核心机制：单文件事务 = 一次原子重写

`[已确定]`

**关键洞察**：A 层所有记录在同一个 JSONL 文件里。多记录的原子变更，只要**一次重写整个文件**就天然原子（`rename` 是文件系统原子操作）。

```
事务执行流程（A 层内部）：

① 取写锁（单写者 + 进程内串行）
② loadRowsStrict() 读全表
③ 在内存副本上：
   a. 逐条应用 changes，每条先校验：
      - record.revision === change.expectedRevision
      - record.status === change.expectedStatus（若指定）
      - record.reviewStatus === change.expectedReviewStatus（若指定）
      - 失败 → 整个事务中止，不写任何东西
   b. 全部校验通过 → 应用所有变更到内存副本
④ 序列化内存副本 → 写唯一 tmp → fsync → rename
⑤ 如果 tmp/rename 失败 → 原文件未动，事务完全未生效（失败关闭）
```

**这解决了什么**：
- V2.3 说"WAL committed 前不能暴露部分变更"——但在单文件原子重写下，**不存在"部分变更"**，要么全生效要么全不生效。
- 不需要额外的"提交标记"来标识可见性。

## 4.2 跨文件事务（A 层 + B 层 + ShareRequest）

```
A + B 的变更：

顺序：
  ① A 层一次原子重写（含所有 A 层变更）
  ② B 层一次原子重写（含所有 B 层变更）
  ③ ShareRequest 文件原子重写（若有）

可见性问题：①成功后②失败 → A 已变，B 未变

读取端如何处理：
  ★ A 层是记忆的权威来源，B 层是"当前会话的活跃态"
  ★ A 层先落地 → 记忆立即可用（正确）
  ★ B 层失败 → 会话活跃态没更新，但记忆已可检索（不阻塞）
  ★ WAL 记 aStatus=done, bStatus=failed → 重试 B
  ★ 由于 A 已可见，用户感知是"记忆写成功了"，不是不一致
  
  → 这种"部分可见"是可接受的降级，不是数据损坏
  → 前提：A 层的写入是完整的（原子重写保证）
```

**审核批准场景的可见性**（V2.3 的漏洞）：

```
V2.3 担心的场景：
  旧记录已 superseded + 新记录仍 pending
  → 两条都不可见

V2.4 用单文件原子重写彻底消除：
  approve 事务的 changes 包含【旧→superseded】和【新→approved】
  → 一次重写，两条同时生效
  → 不存在"旧已变、新未变"的中间态
  
  如果重写失败 → 两条都保持原状（旧 active+approved，新 pending）
  → 旧事实仍可用 ✅
```

## 4.3 审核批准的完整 changes

### 场景：旧 O + 候选 B + 候选 C，批准 C

```json
{
  "opId": "op_approve_1",
  "kind": "approve_fact_update",
  "conflictGroupId": "cg_1",
  "changes": [
    {
      "id": "mem_O",
      "expectedRevision": 3,
      "expectedStatus": "active",
      "set": {
        "status": "superseded",
        "supersededBy": "mem_C",
        "supersededAt": 1793400000000,
        "statusChangedAt": 1793400000000,
        "revision": 4,
        "lifecycleEvents": [..., {"from":"active","to":"superseded","at":1793400000000,"by":"console","opId":"op_approve_1"}]
      }
    },
    {
      "id": "mem_C",
      "expectedRevision": 1,
      "expectedReviewStatus": "pending",
      "set": {
        "reviewStatus": "approved",
        "isCandidate": false,
        "conflictGroupId": null,
        "supersedes": "mem_O",
        "revision": 2
      }
    },
    {
      "id": "mem_B",
      "expectedRevision": 1,
      "expectedReviewStatus": "pending",
      "set": {
        "reviewStatus": "rejected",
        "isCandidate": false,
        "conflictGroupId": null,
        "revision": 2
      }
    }
  ],
  "conflictGroupClosure": {
    "groupId": "cg_1",
    "resolvedBy": "mem_C",
    "closedAt": 1793400000000
  }
}
```

**关键点（修正 V2.3 修-6）**：
- 三条 changes 在同一事务
- 组内所有未批准的候选（B）→ `rejected`
- `conflictGroupId` 全部清空（关闭组）
- 一次原子重写 → 全部同时生效

### 拒绝 C

```json
{
  "opId": "op_reject_1",
  "kind": "reject_fact_update",
  "changes": [
    {
      "id": "mem_C",
      "expectedRevision": 1,
      "expectedReviewStatus": "pending",
      "set": {
        "reviewStatus": "rejected",
        "isCandidate": false,
        "conflictGroupId": null,
        "revision": 2
      }
    }
  ]
}
```

**O 完全不动**（保持 active+approved），B 保持 pending（继续等审）。

## 4.4 读取可见性规则

| 时刻 | 旧 O | 候选 B | 候选 C | 检索结果 |
|---|---|---|---|---|
| 阶段1完成（有候选） | active+approved | pending+cg | pending+cg | **只返回 O** ✅ |
| 批准 C 后（事务成功） | superseded | rejected | approved | **只返回 C** ✅ |
| 批准 C 事务失败 | active+approved（未变） | pending | pending | **只返回 O** ✅ |
| 拒绝 C 后 | active+approved | pending | rejected | **只返回 O** ✅ |

**没有"两条都不可见"的窗口。**

## 4.5 六个异常案例

| # | 场景 | 处理 |
|---|---|---|
| E1 | **候选审核期间又被更新** | 新写 D：同 factKey + isCandidate=true + 同 cg_1<br>C 保持 pending<br>批准 D 时：O→superseded, D→approved, B/C→rejected |
| E2 | **两个管理员并发批准不同候选** | 管理员1 批准 C（expectedRevision=1）<br>管理员2 批准 B（expectedRevision=1）<br>后者因 revision 已变（C 的 op 改了 B 的 revision）→ 冲突，返回错误，提示重读 |
| E3 | **原事实在审核期间被删除** | O 被删（status=deleted, revision=4）<br>批准 C 时：expectedRevision=3 ≠ 4 → 冲突<br>→ 拒绝批准，提示"原事实已删除，请重新评估"<br>→ C 保持 pending，等人工处理 |
| E4 | **审核操作遇 revision 冲突** | 事务整体失败，不写任何东西<br>返回 `CONFLICT_REVISION` + 当前实际 revision<br>控制台刷新后重试 |
| E5 | **WAL 已部分完成但未 committed** | `partial`：aStatus=done, bStatus=failed<br>→ A 层已原子生效（新事实可见）<br>→ B 层未更新<br>→ 重试 B（A 不动） |
| E6 | **批准的候选其 contentHash 已被 tombstone 拦截** | 批准前检查：C 的 contentHash 是否在 tombstone（content_suppression）<br>→ 是 → 拒绝批准，提示"内容已被标记删除" |

---

# 五、lifecycleEvents 与审核审计分工（P1-2）

## 5.1 严格分工

`[已确定]`

| 事件类型 | 记录位置 | 理由 |
|---|---|---|
| status 变更（active↔superseded↔archived↔deleted） | `record.lifecycleEvents` | 属于记录生命周期 |
| reviewStatus 变更（pending↔approved↔rejected） | `state/memory-audit.jsonl` | 属人工审核行为，不是生命周期 |
| 分享审批（ShareRequest 状态变更） | `state/share-requests.jsonl` 的自身状态 + `memory-audit.jsonl` | 独立对象 |
| force_restore / force_write / purge | `state/memory-audit.jsonl` | 管理员强制操作 |
| conflictGroup 开启/关闭 | `state/memory-audit.jsonl` | 冲突管理行为 |

**为什么不能混**：
- lifecycleEvents 的语义是"这条记忆处于什么生命周期"
- reviewStatus 是"这条内容是否可信"
- 两者频率和用途不同：lifecycle 变更少而重要；review 可能反复变化
- 混在一起会让 lifecycleEvents 膨胀，且语义混乱

## 5.2 lifecycleEvents 结构（只含 status）

```json
"lifecycleEvents": [
  {"from": null,        "to": "active",     "at": 1793380000000, "by": "model",   "opId": "op_1"},
  {"from": "active",    "to": "superseded", "at": 1793390000000, "by": "console", "opId": "op_2", "reason": "新事实替代"},
  {"from": "superseded","to": "archived",   "at": 1793400000000, "by": "console", "opId": "op_3", "reason": "清理旧版本"}
]
```

## 5.3 audit 结构

```jsonl
{"ts":1793400000000,"action":"approve_fact_update","memoryId":"mem_C","conflictGroupId":"cg_1","by":"console","opId":"op_approve_1","reason":""}
{"ts":1793400100000,"action":"force_restore","memoryId":"mem_X","by":"console","opId":"op_r1","reason":"用户要求恢复"}
{"ts":1793400200000,"action":"purge_content","memoryId":"mem_Y","by":"console","opId":"op_p1","reason":"隐私请求"}
```

---

# 六、tombstone 最终状态模型（P0-5）

## 6.1 三状态

`[已确定]`

```
active      该 ID 已被删除，禁止复活
restored    曾被 force_restore，仍保留审计，但不再阻止该 ID
purged      tombstone 本身被清理（极少，需审计）
```

## 6.2 字段

```json
{
  "tombstoneId": "ts_xxx",
  "memoryId": "mem_x",
  "ownerId": "owner:[OWNER_QQ]",
  "contentHash": "a1b2...",
  "contentHashPrefix": "a1b2c3d4",
  "deletedAt": 1793400000000,
  "deletedBy": "console",
  "reason": "user_request",
  "suppression": "memory_only",
  "hard": false,
  "purgeLevel": "logical",
  "state": "active",
  "restoredAt": null,
  "restoredBy": null,
  "reAuthorized": [],
  "opId": "op_del_1"
}
```

| 字段 | 用途 |
|---|---|
| `contentHash` 全值 | 用于 content_suppression 的精确匹配 |
| `contentHashPrefix` | 日志/展示用（避免日志泄露太多） |
| `state` | active / restored / purged |
| `reAuthorized` | `[{at, newMemoryId, by}]` 记录重新授权 |

## 6.3 状态转换

| from | to | 触发 | 附加 |
|---|---|---|---|
| （无） | `active` | 删除时创建 | — |
| `active` | `restored` | force_restore | 写 restoredAt / restoredBy |
| `restored` | `active` | 再次删除同一 ID | 清 restoredAt |
| `active`/`restored` | `purged` | 管理员显式 purge（需审计） | 记录 purge 原因 |

## 6.4 tombstone 在检索端生效（新增，修正 V2.3）

`[已确定]`

```
检索第③层（见 §16）：
  对每条候选记录：
    查 tombstone 索引（内存 Map，启动时加载）
    ts = tombstones.get(record.id)
    if (ts && ts.state === 'active') → 排除该记录
    
为什么必须有这层：
  场景：删除操作已写 tombstone，但 A 层记录的 status 还没来得及更新
        （或崩溃导致只写了一部分）
        → 若只靠 status 过滤，该记录仍可见 → 用户"删了还能搜到"
        
  tombstone 是独立文件，写入顺序：
    ① 写 tombstone（先）
    ② 更新 A 层 status（后）
    → 即使 ② 失败，tombstone 已能拦截 ✅
```

**写入顺序（关键）**：
```
删除事务：
  ① 写 tombstone（state=active）  ← 先，作为防线
  ② A 层原子重写（status=deleted）
  ③ 都成功 → WAL committed
  ② 失败 → tombstone 已生效（记录不可见），A 层记录稍后由补偿清理
```

## 6.5 force_restore 后重新可见

```
步骤：
  ① 检查 tombstone.state === 'active'
  ② 检查是否允许恢复（consoleToken + reason）
  ③ 事务：
     a. tombstone.state → 'restored'，写 restoredAt/restoredBy
     b. A 层记录 status=deleted → active，revision+1，deletedAt 保留
     c. audit 记录 force_restore
  ④ 一次跨文件事务（tombstone + A 层）
     顺序：先改 A 层（恢复可见），再改 tombstone（否则顺序反了会有窗口）
     
     ⚠️ 顺序分析：
        若先改 tombstone → state=restored，但 A 层仍 deleted
          → 检索时：tombstone 不拦，但 status 过滤掉 → 仍不可见（安全）
        若先改 A 层 → status=active，但 tombstone 仍 active
          → 检索时：tombstone 拦截 → 仍不可见（安全）
        → 两个顺序都安全，任选。选"先 A 后 tombstone"（因为 A 是权威）
```

## 6.6 新 memoryId 重新授权 vs 旧 ID 恢复（严格区分）

| 场景 | 动作 | tombstone | 新记录 |
|---|---|---|---|
| **重新授权**（用户说"再记一次"） | 新写入，新 memoryId | state 保持 active，追加 `reAuthorized` | 新 ID，独立记录 |
| **恢复旧记录**（管理员 force_restore） | 旧记录 status→active | state → restored | 同 ID |

**区别**：
- 重新授权：旧的仍是 deleted（历史保留），新的独立存在
- 恢复：旧的复活

## 6.7 content_suppression 不可被模型绕过

```
写入检查顺序：
  ① 模型传 force=true？
     → 检查是否有 consoleToken
     → 无 → 拒绝（FORBIDDEN_FOR_MODEL）
  ② 查 tombstone（contentHash 匹配 + suppression=content_suppression）
     → 命中 → 拒绝（CONTENT_SUPPRESSED）
  ③ 查 tombstone（memoryId 匹配 + state=active）
     → 命中 → 拒绝（ID_TOMBSTONED）
  ④ 通过
```

## 6.8 删除与未完成操作的关系

| 场景 | 处理 |
|---|---|
| **尚未执行**（op status=created） | 直接置 `cancelled`，不执行 |
| **已部分执行**（partial） | 不能直接丢弃。必须先完成或补偿：<br>· 若 A 已写 → 把 A 记录标 deleted<br>· 若 B 已写 → 把 B 层槽位标记删除<br>· 然后 op → cancelled |
| **已 committed** | 执行独立删除事务（新 opId） |
| **多记录事务含被删目标** | 逐项校验：<br>· 被删的那条 → 从 changes 中移除<br>· 其他记录 → 检查是否仍需要执行<br>· 若事务语义依赖被删记录（如 supersede 的两条之一被删）→ 整事务 cancelled |

## 6.9 L2 内容清除的数据模型（修正 schema 冲突）

```
L2 执行：
  content = '[deleted]'
  contentHash 保留不变（用于 tombstone 关联和审计）
  vec = []
  vecPurged = true
  contentPurged = true
  vecModel = null

校验（C9）：
  contentPurged=true → vec.length===0 且 vecPurged=true 且 vecModel===null
  contentPurged=false → vec.length===1024 且 vecPurged=false 且 vecModel='bge-m3'

★ 这样不违反 schema：字段允许空数组，只是"purged 状态下必须是空"
```

---

# 七、删除与隐私清除边界

（V2.3 的三级不变，补充精确化）

| 级别 | 动作 | 涉及文件 |
|---|---|---|
| L1 `logical` | status=deleted + tombstone | memory-vectors, tombstones |
| L2 `content` | + content/vec 清空 | memory-vectors, tombstones |
| L3 `cross_store` | + WAL/失败记录/备份 | 全部 —— **本轮不实现** |

**明确声明（不得省略）**：
```
hard=true 不等于彻底删除所有副本。
L2 后：
  · A 层本体 content='[deleted]' ✅
  · tombstone 只有 hash，无正文 ✅
  · WAL 不存正文（设计如此）✅
  · failures 可能含正文 ⚠️（P4 改为只存 hash）
  · backups/ 日备份含原文 ⚠️（7 天后自动清理）
  · 手工备份 D:\qq-bridge备份-* 含原文 ⚠️（永久，需人工清）
  · C 层 Session 含原文 ⚠️（bridge 无写权限，完全不在范围）
```

---

# 八、WAL 最终状态机

## 8.1 三层状态（沿用 V2.3 + V2.4 补充）

```
operationStatus: created | running | partial | committed 
               | failed_retriable | failed_exhausted | recovery_blocked | cancelled
aStatus:         none | pending | done | failed | skipped
bStatus:         none | pending | done | failed | skipped
```

**新增 `recovery_blocked`**（P0-4）：两层都无正文，无法自动重建。

## 8.2 状态转换表

| from | to | 触发 | 条件 |
|---|---|---|---|
| （无） | created | 操作发起 | — |
| created | running | 取锁成功 | — |
| running | committed | 全部 changes 成功 | aStatus/bStatus 都 done/none |
| running | partial | 部分成功 | 一 done 一 failed |
| running | failed_retriable | 全部失败 | 可重试错误 |
| partial | running | 重试 | retryCount < 3 |
| partial | failed_exhausted | 重试超限 | retryCount >= 3 |
| running | **recovery_blocked** | 崩溃恢复时两层无正文 | 无法重建 |
| running | cancelled | 收到删除/撤销 | — |
| partial | cancelled | 收到删除 | — |
| failed_retriable | running | 人工 retry | — |
| failed_retriable | cancelled | 收到删除 | — |
| failed_exhausted | cancelled | 人工取消 | — |
| recovery_blocked | cancelled | 人工取消 | — |
| 任意终态 | — | 不可回退（除人工显式重置） | — |

## 8.3 WAL payload（不含正文）

### append

```json
{
  "opId": "op_x",
  "ts": 1793400000000,
  "kind": "append",
  "memoryId": "mem_x",
  "contentHash": "a1b2c3...",
  "ownerId": "owner:[OWNER_QQ]",
  "conversationKey": "private:[OWNER_QQ]",
  "category": "activeTopic",
  "type": "event",
  "subjectId": "owner",
  "visibility": "owner_private",
  "contentLength": 42,
  "expectedRevision": null,
  "aStatus": "pending",
  "bStatus": "pending"
}
```

**最小必要字段**（用于恢复校验和重建 B 层）：
`memoryId, contentHash, ownerId, conversationKey, category, type, subjectId, visibility, contentLength`

**为什么需要 contentLength**：校验"从 A 层读出的正文是否与原始长度一致"（防截断）。

### 状态变更（如 approve / delete / supersede）

```json
{
  "opId": "op_y",
  "kind": "state_change",
  "changes": [
    {"id": "mem_A", "expectedRevision": 3, "expectedStatus": "active",
     "set": {"status": "superseded", "revision": 4}},
    {"id": "mem_B", "expectedRevision": 1, "expectedReviewStatus": "pending",
     "set": {"reviewStatus": "approved", "revision": 2}}
  ]
}
```

**不存正文字段**，`set` 里也不含 content。

---

# 九、WAL 不存正文的恢复协议（P0-4）

## 9.1 最终决定

`[已确定]` **WAL 不保存原始正文**。

理由：
- 隐私：删除后 WAL 不残留原文
- 可行：append 的正文一定在 A 层或 B 层之一（除非两层都失败）

## 9.2 恢复判定表

| A 层 | B 层 | 恢复动作 | 结果 |
|---|---|---|---|
| done | done | 校验 contentHash 与记录一致 → committed | ✅ |
| done | pending | 从 A 层读 content → 补写 B 层 | ✅ committed |
| pending | done | 从 B 层读 text → 重算 embedding → 补写 A 层 | ✅ committed |
| pending | pending | **两层都无正文** → `recovery_blocked` | ⚠️ 需重新提交 |
| done | failed | 同 done/pending（重试 B） | ✅ |
| failed | failed | `failed_retriable`（可重试，因为 A 可能有正文）<br>但若 A 从未写过正文 → `recovery_blocked` | 视情况 |
| 任一层 done 但 status=deleted | — | **cancelled**（已被删除，不回复） | ✅ |

## 9.3 recovery_blocked 的处理

```
触发条件：
  aStatus ∈ {pending, failed} 且 bStatus ∈ {pending, failed}
  且 A/B 层均查不到 contentHash 对应的正文

状态：
  operationStatus = 'recovery_blocked'
  
行为：
  ① 不自动重试（进入后台重试队列的排除名单）
  ② 写入 state/memory-failures.jsonl：
     {opId, kind, memoryId, contentHash, blockedAt, reason:'no_content_source'}
  ③ 控制台可见（"需重新提交"列表）
  ④ HTTP 响应（若有请求方）：`{ok:false, code:'RECOVERY_BLOCKED', message:'需要重新提交内容'}`

恢复：
  模型/用户重新提交 → 新 opId + 新 memoryId
  旧 op 保持 recovery_blocked（终态），仅审计
  ★ 不复活旧 opId
```

## 9.4 重新提交的规则

```
重新提交时：
  ① 新 opId（不复用）
  ② 新 memoryId（不复用）
  ③ contentHash 可能相同（内容一样）→ append 去重会命中
     → 若已有 active 记录（说明上次其实成功了）→ 返回 already_exists
     → 若无 → 正常写入
  ④ 旧 op 保持 recovery_blocked，不重试

为什么不复用旧 ID：
  旧 ID 可能已被 tombstone 记录（若之前删过）
  复用会导致"防复活"逻辑混乱
```

## 9.5 正文可恢复但向量缺失

```
场景：A 层记录存在但 vec=[]（vecPurged=true 或写入时 embed 失败）
恢复：
  ① 从 A 层读 content
  ② 调用 embed(content) 重算
  ③ 更新 vec + vecModel，revision+1
  ④ 若 embed 仍失败 → failed_retriable（下次再试）
```

## 9.6 状态机变化（V2.3 → V2.4）

```
V2.3:  created → running → {committed|partial|failed|cancelled}
V2.4:  created → running → {committed|partial|failed_retriable|recovery_blocked|cancelled}
                          partial → {running|failed_exhausted|cancelled}
```

---

# 十、revision 乐观锁与多记录事务

（V2.3 已定，V2.4 补充）

## 10.1 校验时机

```
在写锁内、应用变更前，逐条校验：
  record.revision === change.expectedRevision
  record.status === change.expectedStatus（若提供）
  record.reviewStatus === change.expectedReviewStatus（若提供）

任一失败 → 整体中止（一条都不改）
返回：{code:'CONFLICT_REVISION', id, expected, actual}
```

## 10.2 并发冲突案例

```
t0  mem_X: revision=5, status=active
t1  管理员A 批准（expectedRevision=5）→ 成功，revision=6
t2  管理员B 拒绝（expectedRevision=5）→ 冲突，返回 actual=6
    → B 刷新后看到 revision=6，决定不做
```

## 10.3 回滚与 revision

```
任何回滚操作也必须走 revision 校验：
  回滚 = 一次新的 state_change 事务
  不能直接改文件（会用旧 revision 覆盖新状态）

★ 审核要求"所有状态回滚必须考虑 revision"即此
```

---

# 十一、generation + MANIFEST 最终恢复协议

（V2.3 已定，V2.4 补充精确性）

## 11.1 文件与顺序

```
state/wal/
  MANIFEST.json
  wal-0007.jsonl
  index-0007.json
  wal-0006.jsonl（上一代）

compact 顺序（严格）：
  ① 写 wal-0008.jsonl.<pid>.<rand>.tmp → fsync → rename → wal-0008.jsonl
  ② 写 index-0008.json.<pid>.<rand>.tmp → fsync → rename
  ③ 计算两文件 checksum
  ④ 写 MANIFEST.<pid>.<rand>.tmp：
     {activeGen:7, pendingGen:8, gens:[8(complete:false),7,6]}
     → fsync → rename → MANIFEST.json
  ⑤ 重新写 MANIFEST：
     {activeGen:8, pendingGen:null, gens:[8(complete:true),7]}
     → fsync → rename
  ⑥ 删除 gen 6
```

## 11.2 恢复判定（补充精确条件）

```
读 MANIFEST：
  pendingGen === null：
    → 用 activeGen
    → 清理所有非 activeGen 且非上一代的孤儿文件

  pendingGen === 8：
    → 检查 wal-0008.jsonl 和 index-0008.json 是否都存在且 checksum 正确
      ├─ 都正确 → 切换未完成但产物完整 → 补写 MANIFEST activeGen=8, pendingGen=null
      │           然后继续用 8
      └─ 任一缺失/校验失败 → 回滚：
            · 删除 wal-0008 / index-0008 / 相关 tmp
            · MANIFEST 改 activeGen=7, pendingGen=null
            · 用 7
```

## 11.3 未完成操作的保护

```
compact 步骤明确要求：
  "保留所有非终态 op（完整保留）"

非终态 = operationStatus ∈ {created, running, partial, failed_retriable, recovery_blocked}
终态   = {committed, failed_exhausted, cancelled}

★ recovery_blocked 也算非终态（保留，供审计和人工处理）
★ compact 不因"太旧"丢弃任何非终态 op
```

---

# 十二、ShareRequest 与共享副本（P0-3）

## 12.1 独立对象

`[已确定]` 新增 `state/share-requests.jsonl`：

```json
{
  "requestId": "sr_<rand8>",
  "sourceMemoryId": "mem_orig",
  "sourceRevision": 3,
  "sourceContentHash": "abc...",
  "targetOwnerId": "group:[GROUP_ID]",
  "targetConversationKey": "group:[GROUP_ID]",
  "status": "pending",
  "requestedAt": 1793400000000,
  "requestedBy": "model",
  "requestedByConversationKey": "group:[GROUP_ID]",
  "reviewedAt": null,
  "reviewedBy": null,
  "copyMemoryId": null,
  "revokedAt": null,
  "revokedReason": null,
  "reason": "",
  "opId": "op_share_1",
  "revision": 1
}
```

## 12.2 status 状态机

```
pending → approved（批准，创建副本）
pending → rejected（拒绝）
approved → revoked（撤销共享 / 原件删除）
approved → superseded_by_source（原件已更新，副本过期）

终态：rejected / revoked / superseded_by_source
```

| from | to | 触发 | 副作用 |
|---|---|---|---|
| （无） | pending | share_request | 写 WAL |
| pending | approved | 控制台批准 | 创建副本 + 写 WAL |
| pending | rejected | 控制台拒绝 | 无副本 |
| approved | revoked | 撤销 / 原件删除 | 副本 status=deleted |
| approved | superseded_by_source | 原件被更新（revision 变） | 副本保留但标记过期（见 §12.6） |

## 12.3 分享事务（批准）

```
单次事务：

阶段 1（预检）：
  ① 检查 sourceMemoryId 存在且 status=active
  ② 检查 sourceRevision === 记录当前 revision（防基于旧版本分享）
  ③ 检查 targetOwnerId 与请求者 conversationKey 一致
  ④ 幂等检查：已存在 (sourceMemoryId, targetOwnerId) 的 active 副本 → 返回 already_shared
  ⑤ 检查是否已有 pending request 同 (source, target) → 返回 already_pending

阶段 2（执行，跨文件但顺序明确）：
  ① 生成 copyMemoryId
  ② 写 A 层（原子重写，追加副本记录）：
     {
       id: copyMemoryId,
       ownerId: targetOwnerId,
       conversationKey: targetConversationKey,
       visibility: 'group_shared',
       source: 'share_copy',
       sourceMemoryId, sourceRevision, shareRequestId,
       content: 复制原件,
       contentHash: 同原件,
       vec: 复制原件 vec（不重算）,
       factKey: null,           ← 副本不带 factKey
       reviewStatus: 同原件,
       status: 'active'
     }
  ③ 写 share-requests（原子重写）：
     request.status = 'approved', copyMemoryId, reviewedAt, reviewedBy, revision+1
  ④ 写 WAL committed

若 ②成功 ③失败：
  · A 层已有副本（可见）
  · share-request 仍 pending
  · WAL partial
  · 重试 ③（A 不动）
  · 期间副本可见 → 可接受（因为已经批准了，只是记录没更新）
```

## 12.4 撤销事务

```
触发：控制台撤销 / 原件删除 / 原件更新

事务：
  ① 查 share-request（status=approved）
  ② A 层原子重写：副本 status='deleted', deletedAt, lifecycleEvent
  ③ 写 tombstone（副本 id）
  ④ share-requests：status='revoked', revokedAt, revokedReason
  ⑤ WAL committed
```

## 12.5 原件删除时的级联撤销（特别决定）

`[已确定]` **默认策略：原件删除时级联撤销所有 active 副本。**

```
用户删除原件 mem_orig：
  ① 查 share-requests：sourceMemoryId=mem_orig 且 status=approved 的所有 request
  ② 收集所有 copyMemoryId
  ③ 一个大事务（A 层一次原子重写）：
     changes: [
       {id: mem_orig, set: {status:'deleted', ...}},
       {id: copy_1,   set: {status:'deleted', ...}},
       {id: copy_2,   set: {status:'deleted', ...}},
       ...
     ]
  ④ 写 tombstone（原件 + 所有副本）
  ⑤ share-requests：所有相关 request → revoked
  ⑥ WAL committed

若 ③ 失败：
  → 整体不生效（原子重写保证）
  → 返回失败，"删除未完成"
  
若 ③成功但 ④/⑤ 失败：
  → 副本已不可见（A 层 status=deleted）
  → tombstone 可能缺（但 status 已拦截）
  → share-request 仍 approved（但副本已 deleted）
  → WAL partial，重试 ④⑤
  → 不宣称"完整删除"（返回 partial:true）
```

**为什么默认级联撤销**：
```
若不撤销：
  用户删了私聊原件，但群里仍能搜到副本
  → 用户预期"删了就没了"被违反
  → 隐私风险
```

**例外讨论** `[待决策]` D-cascade：
```
例外 1：用户明确说"只删我这边，群里留着"
  → 提供参数 cascadeCopies=false
  → 但必须在响应里警告"群内副本仍可见"

例外 2：原件是 archive 而非 delete
  → 不级联（archive 只是不想检索，不表示撤回共享）
  
例外 3：用户主动撤回分享（revoke）而非删除原件
  → 只撤销副本，原件不动
```

## 12.6 原件更新后副本的处理

```
原件 revision 变化（内容改了）：
  ① 查相关 share-request（approved）
  ② share-request.status → 'superseded_by_source'，记 sourceRevision 差异
  ③ 副本保持 active（不自动删，群里已经看到的内容不能突然消失）
  ④ 但副本标记 'stale': true 或在 share-request 里记 superseded
  ⑤ 若用户想更新群里的内容 → 重新申请分享（新 request）

★ 不自动同步（V2.3 已确定）
```

## 12.7 副本不允许再次分享

```
校验：share_request 的 sourceMemoryId 指向的记录
      if (record.source === 'share_copy') → 拒绝
      理由：副本 ownerId=group:X，再分享到 group:Y 会造成跨群传播
      正确做法：从原件再申请到 group:Y
```

## 12.8 重复申请幂等

```
share_request 时：
  查 (sourceMemoryId, targetOwnerId)：
    ├─ 有 approved + 副本 active → 返回 {ok:true, status:'already_shared', copyMemoryId}
    ├─ 有 pending → 返回 {ok:true, status:'already_pending', requestId}
    ├─ 有 rejected → 允许重新申请（新建 request）
    └─ 无 → 新建 pending
```

---

# 十三、分享权限矩阵（修订 V2.3）

## 13.1 完整矩阵

| # | 请求者 | 目标 ownerId | 目标 visibility | 额外条件 | 允许 |
|---|---|---|---|---|---|
| 1 | 私聊 A | `owner:A` | `owner_private` | active+approved | ✅ |
| 2 | 私聊 A | `owner:A` | `group_shared` | — | ❌ **不存在此状态**（C2） |
| 3 | 私聊 A | `owner:B` | 任意 | — | ❌ |
| 4 | 私聊 A | `group:X` | 任意 | — | ❌ |
| 5 | 群 X | `group:X` | `group_shared` | 有 approved ShareRequest + 副本 active+approved | ✅ |
| 6 | 群 X | `group:X` | `owner_private` | — | ❌ |
| 7 | 群 X | `group:X` | `group_shared` | ShareRequest 非 approved | ❌ |
| 8 | 群 X | `group:X` | `group_shared` | ShareRequest approved 但副本已 revoked | ❌ |
| 9 | 群 X | `group:Y` | 任意 | — | ❌ |
| 10 | 群 X | `owner:A` | 任意 | — | ❌ |
| 11 | 控制台 | 任意 | 任意 | 任意 | ✅（写审计） |
| 12 | 机器人 | 任意 | 任意 | — | ❌（无管理权） |

**修复**：V2.3 的第 2 行删除（原件永为 owner_private）。

## 13.2 关键语义

```
① group_shared 只存在于 share_copy 记录上
② 副本 ownerId = 目标群，所以天然不跨群（矩阵第 9 行）
③ 副本 ownerId ≠ 个人，所以群读不到个人原件（第 10 行）
④ 每个群的审批独立（ShareRequest 按 targetOwnerId 分开）
⑤ shareReviewStatus 字段废弃（改用 ShareRequest 查询）
```

---

# 十四、单写者与离线维护互斥（P0-6）

## 14.1 统一 writer lock

`[已确定]`

```
锁文件：state/writer.lock
内容：
{
  "pid": 12345,
  "instanceId": "inst_a1b2c3d4",
  "createdAt": 1793400000000,
  "mode": "bridge" | "maintenance",
  "purpose": "启动 bridge" | "迁移脚本" | ...
}
```

## 14.2 bridge 启动流程

```
① 尝试 fs.openSync(lockPath, 'wx')
   ├─ 成功 → 写入 {pid, instanceId, mode:'bridge', createdAt}
   │          → 启动心跳（每 10s 更新，可选）
   │          → 继续启动
   └─ EEXIST → 读锁文件
              ├─ 解析失败 → 【失败关闭】退出，提示人工检查
              ├─ mode='bridge' → 已有 bridge 在跑 → 【退出】(code 2)
              └─ mode='maintenance' → 有维护脚本在跑
                 → 【退出】，提示"维护中，请稍后再试"
```

**关键**：bridge **不允许**为了启动而强占维护锁。宁可启动失败，也不能与维护脚本并发写。

## 14.3 离线脚本启动流程

```
① 检查 state/writer.lock 是否存在
   ├─ 存在：读内容
   │   ├─ mode='bridge' → 【拒绝执行】"服务正在运行，请先停止或改用 HTTP 接口"
   │   ├─ mode='maintenance' → 【拒绝执行】"另一个维护脚本正在运行"
   │   └─ 解析失败 → 【失败关闭】"锁文件损坏，请人工检查"
   └─ 不存在：↓
② 尝试 wx 创建锁（mode='maintenance'）
   ├─ 成功 → 执行
   └─ EEXIST（竞态：刚被 bridge 抢到）→ 回到 ①
```

**关键**：不靠"端口连不上"判断服务已停（有竞态），只看锁文件。

## 14.4 竞态案例

| # | 场景 | 处理 |
|---|---|---|
| R1 | 脚本检查锁不存在 → bridge 启动抢锁 → 脚本 wx 失败 | 脚本回到①，看到 bridge 锁 → 拒绝执行 ✅ |
| R2 | bridge 持锁 → 脚本尝试 → 见 bridge 锁 → 拒绝 | ✅ |
| R3 | 脚本持锁 → bridge 启动 → 见 maintenance 锁 → bridge 退出 | ✅（bridge 报错退出，不是数据损坏） |
| R4 | 两个脚本同时启动 | 一个 wx 成功，另一个失败 → 拒绝 ✅ |
| R5 | bridge 崩溃，锁残留 | 下次启动时读到 mode=bridge + pid 已死<br>→ **不自动删除**，失败关闭，提示人工确认<br>→ 人工确认后可手动删除锁文件（需记录） |

## 14.5 bridge 崩溃后的锁处理

```
启动时发现 mode='bridge' 的残留锁：
  ① 检查 pid 是否存活（process.kill(pid, 0)）
     ├─ 存活 → 【退出】"另一个 bridge 在跑"
     └─ 不存活 → 【仍退出】，提示：
        "发现上次异常退出的锁（pid=X, instanceId=Y, 创建于 Z）。
         请确认没有其他实例后，手动删除 state/writer.lock 再启动。
         删除前请检查 state/memory-wal/ 是否有未完成操作。"
        
  为什么不自动清理：
    ① PID 可能被复用（存活检查不可靠）
    ② 崩溃时可能留下未完成的 WAL 操作，需要先检查
    ③ 自动清理可能掩盖真正的问题（如磁盘故障导致崩溃）
```

## 14.6 锁文件损坏

```
读取时 JSON.parse 失败：
  ① bridge 启动 → 【退出】"锁文件损坏"
  ② 脚本 → 【拒绝执行】"锁文件损坏"
  ③ 都提示人工检查 + 手动删除
  
不猜测、不自动重建。
```

## 14.7 人工解除锁

```
需人工操作（无工具）：
  ① 确认没有 bridge / 脚本在跑（任务管理器）
  ② 检查 state/memory-wal/MANIFEST.json 的未完成操作
  ③ 删除 state/writer.lock
  ④ 启动 bridge（它会重建锁）

审计：
  人工操作本身无法自动审计（没有工具介入）
  → 建议：bridge 启动时如果发现"锁被删过"（无 lock 但有 WAL 非终态）
    → 在日志里记录 + 触发恢复流程
```

## 14.8 Windows 文件操作的实际限制

`[已确定]`

| 操作 | Windows 限制 | 应对 |
|---|---|---|
| `rename` 覆盖已存在文件 | 可能 EBUSY/EPERM（文件被打开时） | 复用 `renameWithRetry`（EPERM/EBUSY/EACCES 重试 4 次） |
| `fsync` | 映射到 FlushFileBuffers，可用 | 正常调用 |
| 目录 fsync | **不支持** | 跳过（不调用） |
| `wx` 原子创建 | ✅ 支持 | 用作锁获取 |
| 文件锁（flock） | ❌ 无 POSIX flock | 不用 flock，用 `wx` + 锁文件 |
| 删除被打开的文件 | 失败（EBUSY） | 重试或跳过（垃圾无害） |
| 路径长度 | MAX_PATH 260（未启用长路径） | 保持路径短 |

---

# 十五、A/B/C 与 slotId 关联

（V2.3 已定，V2.4 补充）

## 15.1 slotId

```
B 层每条记忆：
  { slotId: 'slot_<rand8>', text, aMemoryId?, lastOperationId?, ... }

slotId 创建时生成，永不变
text 可变（改时 slotId 不变）
```

## 15.2 关联

```
A 层 bRefs: [{conversationKey, category, slotId}]
B 层 aMemoryId: 'mem_xxx'
B 层 lastOperationId: 'op_xxx'（用于 WAL 恢复验证）
```

## 15.3 WAL 恢复验证

```
恢复 B 层时：
  ① 按 slotId 找记录
  ② 比对 lastOperationId === 期望 opId
  ③ 比对 text 内容（仅校验，不作身份）
```

---

# 十六、最终检索过滤顺序

`[已确定]` 11 层，顺序不可换。

```
输入：query, requester{ownerId, conversationKey}, options

① 请求者身份验证
   token 有效 → 反查 ownerId + conversationKey
   失败 → 403

② 权限验证（硬过滤，先于任何计算）
   record.ownerId === requester 允许的范围
   AND visibility 规则（见 §13.1）
   不通过 → 不可见

③ tombstone 检查（新增，P0-5）
   ts = tombstones.get(record.id)
   if (ts && ts.state === 'active') → 排除

④ 生命周期过滤
   排除 status ∈ {archived, deleted}
   （includeArchived/includeSuperseded 显式放开，仅控制台路径）

⑤ TTL 过滤
   排除 expiresAt !== null && expiresAt < now

⑥ reviewStatus 过滤
   排除 pending / rejected

⑦ isCandidate / conflictGroupId 过滤
   排除 isCandidate === true
   排除 conflictGroupId !== null

⑧ 分享副本有效性检查（群上下文）
   若 record.source === 'share_copy'：
     查 ShareRequest：status 必须 = 'approved'
     且 copyMemoryId === record.id
     → 否则排除

⑨ 语义相似度
   cosSim(queryVec, record.vec)
   仅对 vec.length === 1024 的记录计算（C9）

⑩ 加权排序
   finalScore = score × timeWeight(createdAt) × typeWeight(type)
   ★ 时间权重只影响排序

⑪ topK + 内容裁剪
```

## 16.1 三种查询模式严格分开

| 模式 | 谁可用 | 放开的过滤 | 用途 |
|---|---|---|---|
| **自动注入**（默认） | 模型（agentToken） | 无（全开） | 每轮上下文 |
| **历史检索** | 模型（显式参数） | ④ 放开 superseded | 问"以前" |
| **管理查询** | 控制台（consoleToken） | ④⑤⑥⑦ 全放开 | 控制台列表 |

**禁止**：模型通过参数打开"管理查询"级别的过滤。

```js
// 校验
if (requester.type === 'agent' && (options.includeArchived || options.includeCandidates)) {
  return 403;  // 模型不允许
}
```

---

# 十七、异常场景预演（38 个）

| # | 场景 | 预期 | 机制 |
|---|---|---|---|
| S1 | 同句写两次 | 去重，更新 updatedAt | contentHash |
| S2 | 语义相似但 factKey 不同且合法 | 都保留，不 pending | factKey 独立 |
| S3 | 同 factKey 相似度低 | factKey 命中 → 按策略 | 独立查找 |
| S4 | 设备多实例（desktop/lab） | 不同 factKey，都 active | perEntity=multi |
| S5 | 同设备升级 | supersede（两阶段） | perFactKey=single |
| S6 | 关系叠加（同学+队友+朋友） | 3 条 active | relation.<type> |
| S7 | 关系结束（离职） | 人工 archive，不自动 | 禁止语义推断 |
| S8 | 极性相反（喜欢/不喜欢香菜） | 都保留，新 pending | 两阶段 |
| S9 | **审核期间旧事实仍可见** | 返回旧记录 | 两阶段阶段1 |
| S10 | 批准后原子切换 | 一次重写全生效 | 单文件原子 |
| S11 | 拒绝后旧完全不变 | O 保持 active+approved | 只改候选 |
| S12 | 批准时关闭整个 conflictGroup | 其余候选 rejected | changes 含全部 |
| S13 | 审核期间又来新候选 | 同 cg，队列 | §4.5 E1 |
| S14 | 两管理员并发批准不同候选 | 后者 revision 冲突 | revision |
| S15 | 原事实审核期间被删 | 批准时 revision 冲突 → 拒绝 | revision |
| S16 | 多 active 同 factKey（single） | 打 cg，整组不注入 | conflictGroup |
| S17 | tombstone 拦截已删记录 | 检索排除 | 第③层 |
| S18 | 删除写 tombstone 后崩溃 | 记录已不可见（tombstone 拦） | 写入顺序 |
| S19 | force_restore | tombstone→restored，记录→active | §6.5 |
| S20 | 重新授权同内容（memory_only） | 允许，新 id | §6.6 |
| S21 | 重新授权同内容（content_suppression） | 拒绝 | §6.7 |
| S22 | 模型尝试 force=true | 403 | §6.7 |
| S23 | 原件删除级联撤销副本 | 大事务全删 | §12.5 |
| S24 | 级联撤销部分失败 | 返回 partial，不宣称完整 | §12.5 |
| S25 | 群 Y 读群 X 的副本 | 拦截（ownerId 不同） | 权限第② |
| S26 | 群读个人原件 | 拦截 | 权限第② |
| S27 | 原件更新后副本 stale | 副本保留，request 标 superseded | §12.6 |
| S28 | 重复分享申请 | already_shared / already_pending | §12.8 |
| S29 | 副本再次分享 | 拒绝 | §12.7 |
| S30 | **A完成B未完成，WAL无正文** | 从 A 读正文补写 B | §9.2 |
| S31 | **B完成A未完成** | 从 B 读正文重算向量 | §9.2 |
| S32 | **两层都无正文** | recovery_blocked，不重试 | §9.3 |
| S33 | 正文有但向量缺 | 重算 embedding | §9.5 |
| S34 | WAL compact 崩溃 | 按 MANIFEST 判定 | §11.2 |
| S35 | 锁文件损坏 | 失败关闭，人工处理 | §14.6 |
| S36 | 脚本与 bridge 竞态启动 | 一方 wx 失败 | §14.4 |
| S37 | JSONL 尾行半截 / _crc 不符 | 写拒绝，读跳过 | P1-1 |
| S38 | 旧代码读新数据（superseded/deleted） | **需数据回滚**（移出文件） | §19.3 |

---

# 十八、旧工具兼容

## 18.1 签名（全可选）

```js
qq_memory_append(key, token, category, content, extra?,
                 type?, candidateFactKey?, occurredAt?, subjectId?)
qq_memory_query(key, token, category?, topic?, includeSuperseded?)
qq_memory_remove(key, token, category, content?, target?, memoryId?,
                 suppression?, hard?)
qq_memory_clear(key, token, category?, mode?)
qq_memory_share_request(key, token, memoryId?, text?)   ← 优先 memoryId
qq_memory_manage(op, targetId?, reason?, ...)            ← 新增
```

## 18.2 shareReviewStatus 废弃的兼容

```
读：
  if (record.shareReviewStatus) {
    // 旧字段存在 → 转换为"虚拟 ShareRequest"用于展示，但不写回
  }
  实际判断一律查 ShareRequest 表

写：
  不再写 shareReviewStatus 字段
```

## 18.3 旧数据双读

```js
function normalizeRow(raw) {
  const content = raw.content ?? stripPrefix(raw.text);
  const purged = raw.contentPurged ?? false;
  return {
    ...raw,
    schemaVersion: raw.schemaVersion ?? 1,
    revision: raw.revision ?? 1,
    ownerId: raw.ownerId ?? deriveOwner(raw.key),
    conversationKey: raw.conversationKey ?? raw.key,
    content,
    contentHash: raw.contentHash ?? sha256(normalize(content)),
    contentPurged: purged,
    vecPurged: raw.vecPurged ?? false,
    reviewStatus: raw.reviewStatus ?? (raw.needs_review === true ? 'pending' : 'approved'),
    status: raw.status ?? 'active',
    isCandidate: false,
    conflictGroupId: null,
    lifecycleEvents: raw.lifecycleEvents ?? [{from:null, to:'active', at:raw.time??raw.createdAt}],
    statusChangedAt: raw.statusChangedAt ?? (raw.time ?? raw.createdAt),
    shareReviewStatus: null,   // 废弃
  };
}
```

---

# 十九、P0-P6 实施与回滚

## 19.1 依赖图

```
P0 基础设施（writer.lock / tmp / _crc / normalize）
 └─► P1 数据模型 + 双读 + 基础过滤（含 tombstone 层预留）
      ├─► P2 WAL + revision + generation + 恢复协议
      │    ├─► P3 双层 cardinality + factKey 模板 + 两阶段更新
      │    └─► P4 tombstone + 删除 + 归档 + 级联撤销
      │         └─► P6 ShareRequest + slotId + bRefs
      └─► P5 TTL + 加权排序
```

## 19.2 各批次

### P0

| 项 | 内容 |
|---|---|
| 改动 | `writer-lock.js`；修 `atomicRewrite` tmp；`_crc` 与 `normalize`；`renameWithRetry` 复用 |
| 验收 | ① 并发 100 写不丢 ② 锁互斥（bridge vs 脚本）③ `_crc` 检出篡改 |
| 回滚 | 覆盖代码（无数据格式变化） |
| 人工 | 无 |

### P1

| 项 | 内容 |
|---|---|
| 改动 | 字段补齐；`normalizeRow`；检索加 status/review/TTL 过滤；`findMemoryRow` 修正数字 |
| 验收 | 新字段齐全；旧记录可读；pending/archived/deleted 不检索 |
| 回滚 | 覆盖代码（只加字段） |

### P2

| 项 | 内容 |
|---|---|
| 改动 | `memory-wal.js`；generation+MANIFEST；revision；恢复协议；recovery_blocked |
| 验收 | 5 个崩溃案例；revision 冲突拒绝；compact 崩溃可恢复 |
| **代码回滚** | ⚠️ 不能删 WAL。<br>① 检查所有 op 是否终态<br>② 有非终态 → 先完成或人工 cancelled<br>③ 全部终态 → 保留 WAL 文件<br>④ 覆盖代码 |
| **数据回滚** | WAL 保留不动（旧代码不读） |
| **人工** | 非终态 op 无法处理 → 停服人工检查 |

### P3

| 项 | 内容 |
|---|---|
| 改动 | 双层 cardinality；模板（含 member_relation v2）；两阶段更新；conflictGroup |
| 验收 | S4-S16 全过；模板歧义检测；两阶段可见性正确 |
| **代码回滚** | 覆盖 + 删 `fact-keys.json` |
| **数据回滚** | ⚠️ **必须**：<br>① superseded → active（保留 supersededAt）<br>② isCandidate=true 的记录 → 移到 `state/memory-candidates-archive.jsonl`<br>③ 清空 conflictGroupId |
| **人工** | pending 候选需人工决定批准或删除 |

### P4

| 项 | 内容 |
|---|---|
| 改动 | tombstone；逻辑删除；L2；归档；级联撤销；audit |
| 验收 | S17-S24 全过；tombstone 检索拦截；级联撤销完整 |
| **代码回滚** | 覆盖代码 |
| **数据回滚** | ⚠️ **tombstone 必须保留**（否则防复活失效）<br>deleted 记录 → 移到 `state/memory-deleted-archive.jsonl`<br>（旧代码无 status 过滤，会把 deleted 当正常读出来） |
| **人工** | 归档/恢复操作需人工重做 |

### P5

| 项 | 内容 |
|---|---|
| 改动 | TTL 过滤；timeWeight/typeWeight |
| 验收 | 过期不注入；排序合理 |
| 回滚 | 覆盖代码（无数据变化） |

### P6

| 项 | 内容 |
|---|---|
| 改动 | ShareRequest；share_copy；slotId；bRefs |
| 验收 | S23-S29 全过；权限矩阵正确 |
| **代码回滚** | 覆盖代码 |
| **数据回滚** | ⚠️：<br>① `source=share_copy` 的记录 → 移到 `state/memory-copies-archive.jsonl`<br>② 清空 bRefs / aMemoryId（记录被清内容）<br>③ share-requests.jsonl 保留（旧代码不读，无害） |
| **人工** | 已分享的群内容需人工重新处理 |

## 19.3 关键约束（审核要求）

```
✓ P2 先于 P3/P4（它们会引入需要可靠恢复的状态）
✓ 不允许先引入不可恢复状态再补可靠性
✓ 每批独立验收、独立回滚
✓ 回滚必须考虑 revision（不能直接改文件）
✓ 不能让旧代码把 superseded/deleted/pending 当正常记忆
  → P3/P4 回滚必须配合数据迁移（移到独立文件）
✓ 不能把"原子写文件"等同"跨文件事务原子性"
  → 单文件原子（rename）；跨文件靠顺序 + 状态过滤
```

---

# 二十、待决策问题

| # | 问题 | 候选 | 影响 | 优先级 |
|---|---|---|---|---|
| D1 | 原件删除是否默认级联撤销副本 | a) 默认级联（推荐）<br>b) 默认不级联 + 警告 | 隐私 vs 群内容连续性 | **高** |
| D2 | member_relation 的 type 枚举初始值 | 预置 7 个 / 人工扩展 | 关系表达覆盖度 | **高** |
| D3 | recovery_blocked 后是否提供一键重新提交 | a) 不提供（模型自己重提）<br>b) 控制台按钮生成新 op | 运维便利 | 中 |
| D4 | 锁心跳是否需要（bridge 长时间 compact 时） | a) 不需要（单写者）<br>b) 需要（防误判） | 复杂性 | 低 |
| D5 | `supersede`（无审核）是否保留 | a) 只给 bot.persona<br>b) 全部取消（都审核） | 安全 vs 效率 | 中 |
| D6 | 副本 vec 是否复制（不重算） | a) 复制（省时）<br>b) 重算（更准） | 成本 vs 质量 | 中 |
| D7 | conflictGroup 整组不注入 vs 只排除候选 | a) 整组（推荐）<br>b) 只排除候选 | 冲突期可见性 | 中 |
| D8 | tombstone 是否永久保留 | a) 永久<br>b) 90 天后 purged | 防复活窗口 | 中 |

---

# 二十一、无法安全闭合的问题

| # | 问题 | 原因 |
|---|---|---|
| N1 | C 层 Session 的隐私擦除 | bridge 无 DSH Session 写权限 |
| N2 | 历史备份的隐私擦除 | 备份是静态文件，改写破坏不可变原则 |
| N3 | 关系"结束"的自动判断 | 系统无法验证；只能人工 |
| N4 | 事实"对错"的自动判断 | 系统无法验证；只能人工 |
| N5 | 旧代码读新数据的兼容 | 必须配合数据迁移，不能纯代码回滚 |
| N6 | 恶意进程直接改文件 | 单写者是协作式，无法物理阻止 |
| N7 | 跨文件事务的严格原子性 | 文件系统无跨文件事务；只能"顺序 + 状态过滤 + 重试" |
| N8 | WAL 无正文时两层都失败的恢复 | 信息已丢失，无法重建 |

---

# 二十二、设计冻结状态

## 22.1 可以冻结（可进入实施）

| 项 | 理由 |
|---|---|
| 数据模型（§1） | 字段齐备，约束明确，无待决策 |
| 双层 cardinality（§2） | 四案例全过，规则完整 |
| factKey 模板（§3） | 六模板定义完整，含示例 |
| 两阶段更新（§4） | 单文件原子重写，无中间态 |
| lifecycleEvents 分工（§5） | 语义清晰 |
| tombstone 模型（§6） | 三状态 + 检索拦截 |
| WAL 状态机（§8-9） | 含 recovery_blocked |
| revision 乐观锁（§10） | 校验时机明确 |
| generation/MANIFEST（§11） | 恢复判定表完整 |
| ShareRequest（§12） | 独立对象，状态机完整 |
| 权限矩阵（§13） | 修正矛盾后自洽 |
| 单写者互斥（§14） | 锁流程明确，5 个竞态案例 |
| slotId（§15） | |
| 检索顺序（§16） | 11 层严格 |

## 22.2 不能冻结（需先决策）

| 项 | 阻塞于 |
|---|---|
| 级联撤销的默认行为 | D1 |
| member_relation 的 type 列表 | D2 |
| recovery_blocked 的 UI 处理 | D3 |
| 副本 vec 策略 | D6 |

## 22.3 建议

```
建议进入最终实施评审，但：
  ① 先决策 D1（级联撤销）——它影响 P4 的实现
  ② 先决策 D2（关系 type 枚举）——它影响 P3 的模板
  ③ D3/D6 可在 P3/P6 实施前再定

不建议一次性实施全部 6 批：
  P0 → P1 → P2 必须串行（可靠性地基）
  P3 和 P4 可以并行（不同文件）
  P5 最简单，可随时插入
  P6 依赖 P3/P4，最后做
```

---

# 二十三、本轮交付确认

## 23.1 相对 V2.3 的实质改动（14 项）

1. 双层 cardinality（perFactKey + perEntity）
2. member_relation 加 type 维度（arity 3→4）
3. 单文件事务 = 一次原子重写（消除中间态）
4. 批准事务含关闭 conflictGroup + 其余候选 rejected
5. ShareRequest 独立对象，废弃 shareReviewStatus
6. 原件删除级联撤销副本
7. 删除 tombstone 矩阵第 2 行（原件永不 group_shared）
8. WAL recovery_blocked 状态
9. WAL 恢复判定表（五档）
10. tombstone 进检索第③层
11. tombstone 三状态（active/restored/purged）
12. writer.lock 统一互斥（bridge + 脚本）
13. vec/vecPurged/contentPurged 的 schema 联动（C9）
14. `_crc` 规范化 + normalize 定义（不删数字）

## 23.2 仍需决策（8 项）

见 §二十，最关键是 D1（级联撤销）、D2（关系 type 枚举）。

## 23.3 无法闭合（8 项）

见 §二十一，全部是架构/外部系统边界。

## 23.4 只读声明

```
✅ 本轮只读：
   - 读 MemoryV2.3 文档（核实 C2 与矩阵矛盾、vec=[] 与 1024 维冲突）
   - 读 memory-vector.js / bridge.js 相关段落（前几轮已读，本轮未重读全部）
   - 未执行任何统计脚本（本轮仅读文档，无需核实数据）

✅ 未修改：代码 / 配置 / 数据 / Persona / rules / preset / WAL / tombstone / 备份
✅ 未迁移、未恢复、未清理、未生成测试记忆
✅ 未启动或重启 QQ Bridge / DSH / Ollama
✅ 只新增本文档一个文件
```

---

*本文档完成后立即停止。不进入实施阶段，等待人工审核与最终实施评审。*
