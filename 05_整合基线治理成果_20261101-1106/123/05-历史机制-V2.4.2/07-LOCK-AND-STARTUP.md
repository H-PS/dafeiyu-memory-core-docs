# 07 · bridge.lock、writer.lock 与启动/退出状态机

解决 F-07、F-42（consolePortInUse 失败开放）、启动顺序安全。

## 1. 两把锁的职责（任务 9）

| 维度 | bridge.lock | writer.lock |
|---|---|---|
| 目的 | **进程实例唯一性**（防止双 bridge 抢消息/抢端口） | **A 层存储跨进程写互斥**（bridge vs 迁移/恢复/维护脚本） |
| 持有者 | bridge 主进程，启动→退出全程 | bridge 主进程（启动→退出）+ 外部工具（临时） |
| 内容 | `{pid, instanceId, state, startedAt, heartbeatAt, consolePort}` | `{pid, instanceId, purpose, acquiredAt}` |
| 互斥对象 | 其他 bridge 进程 | 任何写 memory-store/tombstones/WAL/SR 的进程 |
| 与写队列关系 | 无关 | 队列保证进程内串行；锁保证进程间唯一写者。**互不可替代**（02 §6） |
| 死锁风险 | 无（单锁，不嵌套） | 获取顺序固定：先 bridge.lock 后 writer.lock，任何工具不得反向 |

> 结论：**writer.lock 是必需的**。理由：迁移（03 §7）、恢复（06 §4）、tombstone 白名单复活都可能在 bridge 之外执行，必须有协议化互斥；仅靠「约定不写」无法保证（V1 F-06 的教训）。

## 2. bridge.lock 升级为状态文件

```
state/bridge.lock （JSON，原子创建+原子更新）
{ "pid": 1234, "instanceId": "inst_01J8…", "state": "initializing|lock_acquired|recovering|listening|ready|shutting_down|stopped",
  "startedAt": 1730000000000, "heartbeatAt": 1730000050000, "consolePort": 3100 }
```

- 创建：`writeFileSync(flag:'wx')` 一次写入完整 JSON（沿用 V1 `bridge.js:1083-1091` 的原子创建思路，消除「先建空文件再写」窗口）。
- 心跳：主进程每 **10s** 更新 `heartbeatAt`（atomicWriteJson；失败仅告警，不致命）。
- 状态推进：每次状态机迁移都原子重写（见 §3）。

## 3. 启动状态机（六态 + 补充恢复态）

```
S0  initializing
     mkdir state / 读 config / 校验 consoleToken 非空（沿用 V1 bridge.js:3132 失败关闭）
S1  ──▶ lock_acquired
     acquireBridgeLock()（§4 判定逻辑）
S2  ──▶ recovering          ← V1 缺失的关键前置
     acquireWriterLock()
     载入 store/tombstones/SR（坏行隔离，F-12）
     Recovery：扫描 WAL 未提交 op（03 §5）；有 recovery_blocked → 冻结集建立（服务仍可启动）
S3  ──▶ listening
     startConsoleServer() → server.listen(port,'127.0.0.1')
     listen 失败（EADDRINUSE）→ 释放两锁 → exit(2)（V1 bridge.js:7535-7540 保留）
S4  ──▶ ready
     lock.state='ready'
     启动后台任务：embedding 重试调度 / 审计补偿 / 备份检查 / 心跳 / store 外部改动侦测
S5  shutting_down（见 §5）
S6  stopped（releaseLock 完成，文件删除）
```

**与 V1 的本质差异**：V1 的 `acquireLock(1390) → listen(7531)` 之间横跨整个 main（含 `loadSocialV2State:9275`、`bot.connect():12425`）。V2 把**所有重型初始化放在 S2（拿锁之后、监听之前）**，且锁文件中带有 state 与 heartbeat，使第二个进程能区分「正在启动」与「已死」。

## 4. stale 判定算法（修 F-07，不照搬「先 listen 再锁」）

```
acquireBridgeLock():
  tryCreate(wx) 成功 → 写入 {state:'lock_acquired', …} → 返回
  EEXIST → 读取锁（解析失败/内容为空 → 视为 legacy 或损坏 → 走 legacy 分支）
  锁内容判定：
    A. pid 不存在（ESRCH）                     → stale，接管            （V1 同）
    B. pid 存活：
       B1. state === 'ready' 或 'listening'    → 一定有实例 → exit(2)   （端口是佐证）
       B2. state ∈ {initializing/lock_acquired/recovering}：
           age = now - max(startedAt, heartbeatAt)
           B2a. age ≤ GRACE(默认 90s，08 D-06)  → **等待重试**（sleep 5s，最多 6 次），
                                                   而不是接管 ★核心修复
           B2b. age > GRACE 且 consolePortInUse()===false
                                                  → stale，接管（挂死进程遗留）
           B2c. age > GRACE 且 端口占用           → 判定挂死但持有端口 → 报错并提示人工
                                                   （kill 后重试），**不自动删锁** ★保守分支
    C. EPERM（受保护 PID）                      → 需端口探测 + 状态综合（V1 bridge.js:1112-1123
                                                   的改进：state≠ready 才允许按过期处理）
  接管 = unlink + tryCreate（写入自己的 instanceId）
```

- **consolePortInUse 失败开放保留**（V1 `bridge.js:1074-1077` return true）：宁可不启动也不冒双实例。但其触发面在 V2 收窄：`state` 字段已能区分大多数场景，`netstat` 不可用只影响 B2b/B2c 的裁决（→ exit(2) + 明确提示，不再「永久无法启动且原因不明」）。
- **releaseLock 按 instanceId+pid 双匹配**：`read lock; if (lock.instanceId===self && lock.pid===self.pid) unlink`。修 V1 场景「B 释放了自己写的锁导致 A 裸奔」——V2 中 B 若曾错误接管，A 的锁已被删，但 A 的 instanceId 已丢失，A 退出时不会误删 B 的锁（内容不匹配）。同时 B 在 listen 失败退出时正确删自己的锁。

### 4.1 双进程时序验证（F-07 场景重演）

```
t0  A: acquireBridgeLock 成功（state=lock_acquired, startedAt=T0）
t1  B 启动: EEXIST → pid 存活 → state=lock_acquired → age=2s ≤ GRACE → 等待（5s 重试）
t2  A: S2 recovering（重读 store、WAL 恢复）→ state=recovering → heartbeat 持续更新
t3  B: 仍在等待（heartbeat 新鲜）
t4  A: S3 listening → S4 ready（state=ready）
t5  B: 下次重试读到 state=ready → exit(2)「已有实例」 → 释放（无锁可删，A 的锁保留）✅
终态：A 运行、锁文件= A 的 instanceId、state=ready  ✅（V1 终态是「锁不存在」❌）

变体：A 在 S2 卡死超过 GRACE
t6  B: age>90s 且端口空闲 → 接管（A 若稍后恢复并 listen → EADDRINUSE → exit(2)
     → A 的 releaseLock 因 instanceId 不匹配不会删 B 的锁）✅
```

## 5. 退出状态机

```
SIGINT / SIGTERM / exit / uncaughtException（沿用 V1 bridge.js:12447-12477 四挂点）
  S5 shutting_down:
    ① WriteQueue.drain(timeout 5s) —— 在途写操作必须落定（WAL 终态化）
       超时未落定的 op → WAL 标 recovery_blocked（下次启动人工裁决，03 §5.4）
    ② saveState() / saveSocialV2State()（非记忆状态照旧）
    ③ lock.state='shutting_down'
    ④ releaseWriterLock()
    ⑤ lock.state='stopped' → releaseBridgeLock()（instanceId+pid 匹配删除）
  S6 stopped
uncaughtException 例外：先 log（V1 EPIPE 兜底逻辑保留）→ ①② 尽力 → ④⑤ → exit(1)
```

> drain 超时的 op 必须显式 `recovery_blocked`，**不允许**静默丢弃（与 V1 `/api/restart` 路径 `bridge.js:7514-7522` 的 500ms 窗口丢改动问题同源——V2 把该窗口从 500ms 提升到 5s 且有 WAL 兜底）。

## 6. 外部工具的锁协议

```
任何 scripts/*.mjs 欲写 memory-store/tombstones/WAL/SR：
  ① acquireWriterLock(purpose:'migrate|restore|maintain', timeout 30s)
  ② 校验 bridge.lock：state==='stopped' 或文件不存在（要求 bridge 停止，08 D-08）
  ③ 读写 …（同样遵守 02 §3 原子写规范）
  ④ releaseWriterLock()
违规（不取锁直接写）→ 无法强制阻止；缓解：bridge 每 30s 核对 store mtime 与
  内存 storeVersion/lastCommitAt，发现外部改动 → 全量重载 + Tier-1 审计告警（EXTERNAL_WRITE）
  （NOT_VERIFIABLE：无法防御故意绕写）
```

## 7. 验收要点

1. 双进程启动演练（§4.1 全时序 + 卡死变体）：终态锁文件归属正确、无双实例、无「运行中无锁」。
2. kill -9 注入于 S0~S5 各状态：重启后 stale 判定符合 §4 表；WAL 恢复正确。
3. drain 超时注入：在途 op 标 recovery_blocked 而非丢失。
4. 篡改/清空 bridge.lock：legacy 分支可恢复启动。
