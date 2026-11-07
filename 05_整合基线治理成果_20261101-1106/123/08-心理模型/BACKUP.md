# 备份与回滚

> 这个项目迭代快，参数天天改。**每次改动前后都要快照。**

---

## 为什么必须备份

这个项目有三类"改坏了很难恢复"的风险：

| 风险 | 后果 |
|---|---|
| 调参调糊涂 | 不知道上次"感觉对"的版本长什么样 |
| JS/Python 漂移 | 对拍失败后找不回一致的那一版 |
| 机制叠加 | 麻木、饱和、溢出互相干扰，改一个崩三个 |

**已经发生过一次**：连续修改 `mind.mjs`（emoCap、spillToMood、麻木去重）时没有备份，
如果其中一步改错，前面几步都得重做。

---

## 用法

### 打快照

```bash
node backup.mjs snap "改了什么"
```

输出：

```
✓ 快照 20261102-135037-decay-integrated
  39 个文件
```

### 看有哪些

```bash
node backup.mjs list
```

```
快照（新→旧）：

  20261102-135037-decay-integrated
    说明：decay-integrated
    39 个文件 · 2026-11-02 13:50:37
  20261102-135000-baseline
    ...
```

### 和快照比

```bash
node backup.mjs diff
```

```
对比快照：20261102-135037-decay-integrated

  M mind.mjs
  M memory.mjs
  + decay.mjs
  + test-decay.mjs

  改 2 · 新增 2 · 删除 0
```

### 回滚

```bash
node backup.mjs restore 20261102-135037-decay-integrated
```

**回滚前会自动把当前状态存成 `before-restore` 快照**，所以回滚错了还能回来。

---

## 什么时候必须打

| 时机 | 例子 |
|---|---|
| **改算法逻辑前** | 加溢出机制、改麻木叠加方式 |
| **调参数后觉得"对了"** | 麻木率调到 0.83 那个版本 |
| **对拍通过后** | JS/Python 一致的那一刻 |
| **接新模块前** | 接 decay.mjs、接向量库 |
| **大重构前** | 改 hint() 的输出格式 |

### 一个实用习惯

改之前先打一个，叫 `pre-<改什么>`：

```bash
node backup.mjs snap "pre-emoCap"
# ... 改 ...
node backup.mjs snap "post-emoCap-麻木率0.83"
```

这样出问题能精确回到"改之前"。

---

## 快照里有什么

**只存源码和文档**：

- `.mjs` / `.js` / `.py`
- `.md`
- `.json`
- `.html` / `.css`

**不存**：

- `_backup/` 自己
- `node_modules/`
- `__pycache__/`
- `out/`（生成的图和数据）

### 保留策略

**最近 30 个**，更早的自动删除。

一个快照约 200KB（39 个文件），30 个约 6MB，可以接受。

---

## 当前基线

`20261102-135000-baseline` 是第一个快照，状态为：

```
✓ JS/Python 对拍一致
✓ test-pipeline   OK
✓ test-attention  OK
✓ test-memory     OK
✓ test-vector     OK
✓ stress          OK
✓ tuning          OK
```

**如果后面改崩了，回到这一版至少能跑。**

---

## 注意

### 快照不是 git

它没有分支、没有合并、没有 commit message 的规范。
它只是"某个时刻的完整拷贝"，够用但很粗。

**如果项目继续长大，应该换成 git。** 现在的规模（39 个文件）用快照够了。

### 回滚会覆盖

`restore` 是**全量覆盖**：快照里有而当前没有的文件，不会被删除；
但快照里有的文件，当前会被覆盖成快照的版本。

**所以回滚前务必确认快照是对的**（用 `list` 看时间和说明）。
