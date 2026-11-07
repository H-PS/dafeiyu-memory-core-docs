# 07 · 三锁职责、启动/退出状态机与外部工具协议

解决 F-07、F-42。本版核心修订：**心跳在 S1 启动**（R7）、**writer.lock 职责重新定义**（R8）、**drain 语义修正**（R9）。

---

# 1. 三锁职责（重新定义，R8）

| 机制 | 保护什么 | 持有者 | 强度 |
|---|---|---|---|
| **进程内 WriteQueue** | 同进程操作的**时序正确性**（谁先谁后、原子生效） | MemoryService | 强（语言级串行） |
| **bridge.lock** | **Bridge 实例唯一性**（防双进程抢消息/抢端口） | bridge 主进程，启动→退出 | 中（PID + state + 心跳） |
| **writer.lock** | **外部维护脚本之间的互斥** | bridge（全程持有）+ 脚本（bridge 停止后） | 弱（advisory） |

## 1.1 ★ writer.lock 的真实职责（修正原设计）

```
原设计（07 §1）：
  "writer.lock：A 层存储跨进程写互斥（bridge vs 迁移/恢复/维护脚本）"
  "互斥对象：任何写 memory-store/tombstones/WAL/SR 的进程"

源码/逻辑事实：
  bridge 在启动时获取 writer.lock，【全程持有】直到退出
  ⇒ 外部脚本 acquireWriterLock 必然失败（锁被 bridge 持有）
  ⇒ 脚本必须先等 bridge 停止（07 §6 要求校验 bridge.lock.state==='stopped'）
  ⇒ "bridge vs 脚本"的隔离由【强制停服规则】保证，【不是】由 writer.lock 保证

writer.lock 的真实作用：
  · bridge 停止期间，多个维护脚本之间的互斥
  · 防"脚本 A 和脚本 B 同时迁移"
  · 防"bridge 已停但两个实例同时启动"（此时 bridge.lock 也可防，但 writer.lock 更早）
```

**为什么仍需要 writer.lock**：
- 迁移/恢复/墓碑白名单复活都可能在 bridge 之外执行
- 仅靠"约定不写"无法保证（V1 F-06 的教训）
- 它是**协作式**的，不能防恶意绕写（见 §6.3）

## 1.2 锁的获取顺序

```
固定顺序：先 bridge.lock，后 writer.lock
任何工具不得反向（防死锁）
```

---

# 2. bridge.lock 状态文件

```json
{
  "pid": 1234,
  "instanceId": "inst_01J8…",
  "state": "initializing|lock_acquired|recovering|listening|ready|shutting_down|stopped",
  "startedAt": 1730000000000,
  "heartbeatAt": 1730000050000,
  "consolePort": 3100
}
```

- 创建：`writeFileSync(flag:'wx')` 一次写入完整 JSON（沿用 V1 `bridge.js:1083-1091` 原子创建思路）
- **心跳：每 10s 更新 `heartbeatAt`**（atomicWriteJson；失败仅告警，不致命）
- 状态推进：每次状态机迁移都原子重写

## 2.1 ★ 心跳启动时机（R7 核心修正）

```
原设计：
  07 §2 "心跳：主进程每 10s 更新"（未说何时启动）
  07 §3 S4（ready）才"启动后台任务：… 心跳 …"
  ⇒ S1~S3 期间无心跳，heartbeatAt=null 或 =startedAt
  ⇒ S2 recovery 超过 GRACE 时，B 会判定 stale 并接管（即使 A 仍在正常 recovery）

修订：
  心跳在 S1（lock_acquired）成功后【立即启动】
  S4（ready）不再重复启动心跳
  ⇒ S2 recovery 期间 heartbeatAt 持续刷新 ⇒ B 的 age 判断正确
```

**实现**：
```js
// S1 成功后立即
lockHeartbeatTimer = setInterval(() => {
  try { updateLockField('heartbeatAt', Date.now()); } catch (e) { log.warn('心跳写入失败', e); }
}, 10_000);
lockHeartbeatTimer.unref();   // 不阻止进程退出
```

---

# 3. 启动状态机（六态）

```
S0  initializing
     mkdir state / 读 config / 校验 consoleToken 非空（V1 bridge.js:3132 失败关闭）

S1  ──▶ lock_acquired
     acquireBridgeLock()（§4 判定）
     ★ 立即启动心跳（§2.1）

S2  ──▶ recovering
     acquireWriterLock()
     载入 store / tombstones / SR / projection（坏行隔离，F-12）
     Recovery：扫描 WAL 未提交 op（03 §5）；有 recovery_blocked → 冻结集（服务仍可启动）
     ★ 本阶段可能耗时较长（WAL 大时），但心跳在刷新 ⇒ 不会被误判 stale

S3  ──▶ listening
     startConsoleServer() → server.listen(port, '127.0.0.1')
     listen 失败（EADDRINUSE）→ 释放两锁 → exit(2)（V1 bridge.js:7535-7542 保留）

S4  ──▶ ready
     lock.state='ready'
     启动后台任务：embedding 重试 / 审计补偿 / 备份检查 / 外部写入侦测
     ★ 心跳已在 S1 启动，此处不重复

S5  shutting_down（§5）
S6  stopped（releaseLock 完成，文件删除）
```

**与 V1 的本质差异**：V1 的 `acquireLock(1390) → listen(7531)` 之间横跨整个 main（含 `loadSocialV2State:9275`、`bot.connect:12425`）。V2 把所有重型初始化放入 S2（拿锁后、监听前），且锁文件带 state + heartbeat。

## 3.1 顺序说明：为什么不是"先 listen 再拿锁"

- 先 listen 后拿锁 ⇒ 两个进程都能 listen 成功（不同时间），但只有一个拿到锁 ⇒ 后拿锁者必须关闭已开的端口（复杂）
- 当前设计：先拿锁，后 listen ⇒ 拿不到锁的直接退出，不会浪费端口
- 代价：S2 期间端口未监听（但心跳保证不被误判）

---

# 4. stale 判定算法（修订）

```
acquireBridgeLock():
  ① tryCreate(wx) 成功 → 写入 {state:'lock_acquired', instanceId, startedAt, heartbeatAt:now} → 返回
  ② EEXIST → 读取锁
     ├─ 解析失败 / 内容为空 → legacy 分支（§4.2）
     └─ 成功 → 按 state 判定：
        A. pid 不存在（ESRCH）              → stale，接管
        B. pid 存活：
           B1. state ∈ {ready, listening}    → 一定有实例 → exit(2)
           B2. state ∈ {initializing, lock_acquired, recovering}：
               age = now - max(startedAt, heartbeatAt)     ← 心跳在 S1 已启动
               B2a. age ≤ GRACE(默认90s) → **等待重试**（见 §4.1 次数计算）
               B2b. age > GRACE 且 consolePortInUse()===false → stale，接管（挂死遗留）
               B2c. age > GRACE 且 端口占用                    → 报错提示人工，**不自动删锁**
           B3. state === 'shutting_down' → 等待其完成（最多 30s），然后重判
        C. EPERM（受保护 PID）              → 需端口探测 + state 综合
  ③ 接管 = unlink + tryCreate（写入自己的 instanceId）
```

## 4.1 ★ 等待次数与 GRACE 匹配（R7 缺口②）

```
原设计：B2a "等待重试（sleep 5s，最多 6 次）" = 最多 30s
        D-06 GRACE = 90s
        ⇒ 30s 后 B 的行为未定义；而 A 可能仍在 90s GRACE 内

修订：
  maxAttempts = ceil(GRACE / 5000)        // 90s → 18 次
  循环：
    for (i = 0; i < maxAttempts; i++) {
      sleep(5000);
      重读锁；
      if (state ∈ {ready, listening}) → exit(2);
      if (pid 不存在) → stale，接管;
      if (age > GRACE) → 进入 B2b/B2c 判定;
    }
  // 循环结束仍未决 → exit(3)「启动超时，请检查另一个实例」

★ 关键：等待期间必须【重新读取锁】，而非固定睡 18 次就放弃
```

## 4.2 legacy 分支（纯 PID 锁，V1 遗留）

```
读到纯数字（无 JSON 结构）：
  视为 V1 遗留锁 → 按 V1 逻辑判定（PID 存活 + 端口探测）
  ★ 但升级为新格式：接管时写入完整 JSON
```

## 4.3 releaseLock（按 instanceId + pid 双匹配）

```js
function releaseBridgeLock() {
  const lock = readLock();
  if (lock && lock.instanceId === self.instanceId && lock.pid === process.pid) {
    // 先写 state='stopped'，再删除
    updateLockField('state', 'stopped');
    unlink(lockPath);
  }
}
```

**修 V1 场景**「B 释放了自己写的锁导致 A 裸奔」：V2 中 B 若曾错误接管，A 的锁已被删，但 A 的 instanceId 已丢失，A 退出时不会误删 B 的锁（内容不匹配）。

## 4.4 双进程时序验证

```
t0  A: acquireBridgeLock 成功（state=lock_acquired, heartbeatAt=T0）
t1  B 启动: EEXIST → pid 存活 → state=lock_acquired → age=2s ≤ GRACE → 等待 5s
t2  A: S2 recovering（重读 store、WAL 恢复）；心跳持续更新 heartbeatAt
t3  B: 再次读取 → age = now - heartbeatAt ≈ 5s ≤ GRACE → 继续等待
    （循环最多 18 次）
t4  A: S3 listening → S4 ready（state=ready）
t5  B: 下次读取 state=ready → exit(2)「已有实例」→ releaseLock（instanceId 不匹配，不删 A 的锁）
终态：A 运行、锁=A 的 instanceId、state=ready  ✅（V1 终态是"锁不存在"❌）

变体：A 在 S2 卡死（心跳停止）超过 GRACE
t6  B: age>90s 且端口空闲 → 接管（A 若稍后恢复并 listen → EADDRINUSE → exit(2)
    → A 的 releaseLock 因 instanceId 不匹配不会删 B 的锁）✅
```

---

# 5. 退出状态机（修订 drain）

```
SIGINT / SIGTERM / exit / uncaughtException（沿用 V1 bridge.js:12447-12477 四挂点）
  S5 shutting_down:
    ① 停止接受新请求（server.close）
    ② WriteQueue.drain(drainTimeoutMs)     ← 见 §5.1
       超时未执行的排队 op → WAL 标 aborted(reason='shutdown')  ★不是 recovery_blocked
    ③ saveState() / saveSocialV2State()（非记忆状态）
    ④ 停止心跳定时器
    ⑤ releaseWriterLock()
    ⑥ lock.state='stopped' → releaseBridgeLock()
  S6 stopped
uncaughtException 例外：先 log → ①② 尽力 → ④⑤⑥ → exit(1)
```

## 5.1 ★ drain 与 P1 的关系（R9 修正）

```
原设计（07 §5）：
  "WriteQueue.drain(timeout 5s)；超时未落定的 op → WAL 标 recovery_blocked"
问题：
  P1 embed 最长 30s，但 drain 只等 5s
  ⇒ 排队中的 op（P1 已完成、等 P3）5s 内可能排不完
  ⇒ 每次正常退出都产生 recovery_blocked（污染恢复流程，需人工裁决）

修订：
  ① drain 分两段：
     a. 等待 P3 执行中的 op 结束（同步 IO，通常 < 1s）
     b. 等待排队中的 op 进入 P3 并完成，上限 drainTimeoutMs
  ② drainTimeoutMs 默认 5000，但动态调整：
     if (队列非空 && 最近 op 处于 P1) → 延长至 min(30000, 队列长度 × 200ms + 2000)
  ③ 超时仍未执行的排队 op → 标 **aborted(reason='shutdown')**
     ★ recovery_blocked 只用于"P3 中途 IO 失败"，不用于"没来得及执行"
```

**语义澄清**：
- `aborted(shutdown)` = 该 op 从未执行，无副作用，不需要人工处理
- `recovery_blocked` = 该 op 执行到一半失败，可能有部分 effect，必须人工裁决

---

# 6. 外部工具协议（修订）

## 6.1 脚本写 A 层的前提

```
任何 scripts/*.mjs 欲写 memory-store / tombstones / WAL / SR / projection：
  ① 校验 bridge.lock：state==='stopped' 或文件不存在（**强制停服**，D-08）
  ② acquireWriterLock(purpose:'migrate|restore|maintain', timeout 30s)
  ③ 读写 …（遵守 02 §3 原子写规范）
  ④ releaseWriterLock()（finally 中，防异常泄漏）
```

## 6.2 为什么要求停服（D-08）

- bridge 全程持有 writer.lock ⇒ 脚本拿不到 ⇒ 必须等 bridge 停
- 即使无视 writer.lock 强行写，也会与 bridge 的 WriteQueue 冲突（无跨进程协调）
- **停服是最可靠的隔离手段**

## 6.3 违规的缓解（非保证）

```
bridge 每 30s 核对：
  · store 文件 mtime
  · 首行 meta（writerInstanceId / storeVersion / lastCommitAt）
  发现「writer.lock 空闲但 store 被外部改动」→ 全量重载 + Tier-1 审计告警（EXTERNAL_WRITE）

★ NOT_VERIFIABLE：无法防御故意绕写（恶意进程可直接改文件）
```

## 6.4 脚本的锁释放（必须 finally）

```js
async function withMaintenanceLock(purpose, fn) {
  const lock = await acquireWriterLock(purpose);   // 失败则抛
  try {
    return await fn();
  } finally {
    try { releaseWriterLock(lock); } catch (e) { log.error('释放 writer.lock 失败', e); }
  }
}
```

**要求**：所有脚本必须用此包装，禁止手动 acquire 不 release。

---

# 7. consolePortInUse（保留失败开放，但收窄触发面）

```js
function consolePortInUse() {
  try {
    // Windows: netstat -ano -p TCP | findstr LISTENING:<port>
    // POSIX: ss -ltn || netstat -ltn
    ...
  } catch {
    return true;   // 探测失败 → 保守认为有人在跑（宁可不启动，不冒双实例）
  }
}
```

**V2 的改进**：`state` 字段已能区分大多数场景，`netstat` 不可用只影响 B2b/B2c 的裁决（→ exit(2) + 明确提示，不再"永久无法启动且原因不明"）。

---

# 8. 验收要点

1. 双进程启动演练（§4.4 全时序 + 卡死变体）：终态锁归属正确、无双实例、无"运行中无锁"。
2. **A 在 S2 recovery 耗时 120s（> GRACE 90s）但心跳正常** → B 必须等待，**不得接管**。
3. kill -9 于 S0~S5 各态：重启 stale 判定符合 §4 表。
4. drain 超时注入：在途 P3 op 标 recovery_blocked；未执行的排队 op 标 aborted(shutdown)。
5. 篡改/清空 bridge.lock：legacy 分支可恢复启动。
6. 外部工具不取锁直写 store：bridge 侦测 EXTERNAL_WRITE + 重载 + 审计告警。
