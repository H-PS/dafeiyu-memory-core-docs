# 源码文件：src/token-ledger.js
- 原始路径：src/token-ledger.js
- 原始文件大小：39568 字节
- 原始 SHA256：4d9141ce36cf037671fd65529f1aef2cbb7f732566749b0706ceceb0e7679507
- 原始行数：958
- 是否脱敏：是（QQ号/真实姓名 → 占位符；行数与行序保持原样，可行号对照）
## 完整源码

````javascript
// Token 用量账本：把 DSH 会话事件里的 provider usage 落成「按会话 × 按回合」的可查明细，
// 并在读取时用价目表折算成金额。
//
// 为什么这样设计（三条约束决定了实现形态）：
//
// 1. **权威总量来自 DSH 的 tokenUsage 投影，不来自桥接自己累加。**
//    `session/follow` 的 snapshot 帧带 `projections.values.tokenUsage`，是**整条会话日志**的
//    累计桶（含桥接启动之前的历史）。桥接只补「基线之后」的增量，所以总量是精确的，
//    不需要重放全部历史。
//
// 2. **逐轮明细靠事件折叠，且必须幂等。**
//    snapshot 帧的 `records` 是最多 maxMessages 条历史事件，重连时会**重新下发**；
//    因此同一 (session, turn, step, retry) 的采样必须「后到覆盖先到」，而不是累加。
//    这与 DSH `dsh-token-meter` 的 tokenUsage 折叠语义一致：
//      - `assistant/message` / `assistant/attempt` 带 usage 时，同 (turn, step) 覆盖上一次采样；
//      - `llm/retry-started` 结束该覆盖作用域，于是重试的第二次请求**另算一次**（retry 轴 +1）。
//
// 3. **不能把唯一副本放在会被重连清空的内存里。**
//    `pumpMux` 的 finally 会清掉所有 per-turn Map（否则重连后回复会重复累加）。
//    所以每条采样一到就 append 到 JSONL 日志，内存只是它的投影。
//
// 存储：`state/token-usage.jsonl`，两种行：
//   {"k":"base", t, sid, key, seq, model, provider, b:{cacheMiss,cacheHit,cacheWrite,output,reasoning}}
//   {"k":"turn", t, sid, key, seq, turn, step, retry, model, provider, b:{...}}
// 行数超过上限时按内存态重写（自动去重 + 丢弃被覆盖的旧采样）。
import fs from 'node:fs';
import path from 'node:path';
import {
  DEFAULT_PRICE_TABLE,
  ZERO_BUCKETS,
  addBuckets,
  costOfBuckets,
  isPeakAt,
  normalizeUsage,
  priceUsage,
  ratesAt,
  totalTokensOf
} from './model-prices.js';

/** 采样键：同一回合同一步骤同一次重试只保留一条（后到覆盖先到）。 */
function sampleKeyOf(sessionId, turn, step, retry) {
  return `${sessionId}|${turn}|${step}|${retry}`;
}

/**
 * 走势图的桶宽阶梯（毫秒）与选择规则。
 *
 * 导出给离线 fixture 复用：控制台走势图的示例数据必须和生产用**同一套桶宽**，
 * 否则「近 7 天」在离线预览里和生产里会是两种粒度，预览就失去意义了
 * （这类两边各写一份常量的漂移，正是这次体检里反复出现的问题）。
 */
export const BUCKET_LADDER_MS = [
  5 * 60 * 1000, 15 * 60 * 1000, 30 * 60 * 1000,
  60 * 60 * 1000, 2 * 60 * 60 * 1000, 3 * 60 * 60 * 1000,
  6 * 60 * 60 * 1000, 12 * 60 * 60 * 1000,
  24 * 60 * 60 * 1000, 2 * 24 * 60 * 60 * 1000, 7 * 24 * 60 * 60 * 1000
];

/** 在不超过 maxBuckets 的前提下选最细的桶。 */
export function pickBucketMs(windowMs, maxBuckets) {
  for (const candidate of BUCKET_LADDER_MS) {
    if (windowMs / candidate <= maxBuckets) return candidate;
  }
  return BUCKET_LADDER_MS[BUCKET_LADDER_MS.length - 1];
}

/**
 * 从 assistant stream 记录里取最后一个 usage 采样。
 * 与 DSH `dsh-token-meter` 的 `lastAssistantStreamChunk(stream, 'usage')` 同义。
 */
export function usageFromStream(stream) {
  if (!Array.isArray(stream)) return undefined;
  for (let i = stream.length - 1; i >= 0; i -= 1) {
    const chunk = stream[i];
    if (chunk && typeof chunk === 'object' && chunk.usage !== undefined) return chunk.usage;
  }
  return undefined;
}

/**
 * 取一条会话事件携带的 provider usage（原始桶，未归一化）。
 * 与 DSH `usageOf(event)` 一致：assistant/message 优先用 data.usage，
 * 否则回落到 assistant/attempt / assistant/message 的 stream 尾部采样。
 */
export function usageOfEvent(event) {
  if (!event || typeof event !== 'object') return undefined;
  if (event.type === 'assistant/message' && event.data?.usage !== undefined) return event.data.usage;
  if (event.type !== 'assistant/message' && event.type !== 'assistant/attempt') return undefined;
  return usageFromStream(event.data?.stream);
}

/**
 * 建立一个 token 账本。
 * @param {object} options
 * @param {string} options.file           JSONL 落盘路径
 * @param {(msg: string, err?: unknown) => void} [options.log]
 * @param {object} [options.priceTable]   价目表（`resolvePriceTable()` 的结果）
 * @param {(sessionId: string) => (string|undefined)} [options.resolveKey] 运行期 sessionId → 会话 key 解析
 * @param {number} [options.maxLines]     日志行数上限，超出即压缩重写
 * @param {() => number} [options.now]
 */
export function createTokenLedger({
  file,
  log = () => {},
  priceTable = DEFAULT_PRICE_TABLE,
  resolveKey,
  maxLines = 20000,
  now = () => Date.now()
} = {}) {
  if (typeof file !== 'string' || !file) throw new Error('createTokenLedger: file is required');

  /** sessionId -> 最新一条投影基线 */
  const baselines = new Map();
  /** sampleKey -> 采样记录 */
  const samples = new Map();
  /** sessionId -> { turn, step, retry, seq }：重试轴，用于区分同一步骤的多次计费请求 */
  const retryAxis = new Map();
  /** sessionId -> 最近一次见到的会话 key（会话退役后仍可归属到原 QQ 会话） */
  const sessionKeys = new Map();
  /** sessionId -> 被淘汰（只剩累计值、无法再拆到某一轮）的采样汇总 */
  const evicted = new Map();
  let evictedDirty = false;
  /** sessionId -> 最近一次活动时间 */
  const sessionSeen = new Map();
  /** 落盘行数（含被覆盖的旧采样），用于决定何时压缩 */
  let journalLines = 0;
  let lastIngestAt = 0;
  let lastCompactedAt = 0;
  let droppedLines = 0;

  function keyOf(sessionId) {
    let key;
    try {
      key = resolveKey?.(sessionId);
    } catch {
      key = undefined;
    }
    if (key) {
      sessionKeys.set(sessionId, key);
      return key;
    }
    return sessionKeys.get(sessionId);
  }

  // ── 载入 ────────────────────────────────────────────────────────────────
  function applyRecord(record, { fromDisk }) {
    if (!record || typeof record !== 'object') return false;
    const sid = record.sid;
    if (typeof sid !== 'string' || !sid) return false;
    if (record.key) sessionKeys.set(sid, record.key);
    const seq = Number.isFinite(Number(record.seq)) ? Number(record.seq) : -1;
    const t = Number.isFinite(Number(record.t)) ? Number(record.t) : 0;
    const buckets = normalizeUsage(record.b);

    // 被淘汰的逐轮采样：只保留累计桶与花费，不保留回合归属。
    if (record.k === 'evict') {
      const prevEvicted = evicted.get(sid);
      if (prevEvicted && prevEvicted.dropped >= (Number(record.dropped) || 0)) return false;
      evicted.set(sid, {
        sessionId: sid,
        key: record.key,
        buckets,
        cost: Number(record.cost) || 0,
        byTier: {
          peak: Number(record.peak) || 0,
          off: Number(record.off) || 0
        },
        estimated: record.estimated === true,
        dropped: Number(record.dropped) || 0,
        maxSeq: Number.isFinite(Number(record.maxSeq)) ? Number(record.maxSeq) : -1
      });
      if (t > (sessionSeen.get(sid) ?? 0)) sessionSeen.set(sid, t);
      return true;
    }

    if (record.k === 'base') {
      const prev = baselines.get(sid);
      if (prev && prev.seq > seq) return false; // 旧基线不得覆盖新基线
      baselines.set(sid, {
        sessionId: sid,
        key: record.key,
        seq,
        t,
        model: record.model,
        provider: record.provider,
        buckets
      });
      if (t > (sessionSeen.get(sid) ?? 0)) sessionSeen.set(sid, t);
      return true;
    }

    if (record.k === 'turn') {
      const turn = Number(record.turn);
      const step = Number(record.step);
      const retry = Number.isFinite(Number(record.retry)) ? Number(record.retry) : 0;
      if (!Number.isFinite(turn) || !Number.isFinite(step)) return false;
      const sk = sampleKeyOf(sid, turn, step, retry);
      const prev = samples.get(sk);
      // 幂等关键：只有 seq **严格更新**才允许覆盖。
      // 相等 seq 只可能是同一条事件的重放（重连时 snapshot 会重发），
      // 若放行「相等 seq 覆盖」，旧快照就能把新数据改小 —— 花费会凭空缩水。
      if (prev && seq >= 0 && prev.seq >= seq) return false;
      samples.set(sk, {
        sessionId: sid,
        key: record.key,
        seq,
        t,
        turn,
        step,
        retry,
        model: record.model,
        provider: record.provider,
        buckets
      });
      const axis = retryAxis.get(sid);
      if (!axis || axis.seq <= seq) retryAxis.set(sid, { turn, step, retry, seq });
      if (t > (sessionSeen.get(sid) ?? 0)) sessionSeen.set(sid, t);
      return true;
    }
    return false;
  }

  function load() {
    baselines.clear();
    samples.clear();
    retryAxis.clear();
    sessionKeys.clear();
    sessionSeen.clear();
    journalLines = 0;
    droppedLines = 0;
    let raw;
    try {
      raw = fs.readFileSync(file, 'utf8');
    } catch {
      return; // 首次运行：文件还不存在
    }
    const lines = raw.split('\n');
    for (const line of lines) {
      if (!line) continue;
      journalLines += 1;
      let record;
      try {
        record = JSON.parse(line);
      } catch {
        droppedLines += 1; // 进程中断留下的半行：跳过
        continue;
      }
      try {
        applyRecord(record, { fromDisk: true });
      } catch (error) {
        droppedLines += 1;
        log('token 账本：丢弃无法应用的行', error?.message ?? error);
      }
    }
  }

  // ── 落盘 ────────────────────────────────────────────────────────────────
  function appendLine(record) {
    try {
      fs.mkdirSync(path.dirname(file), { recursive: true });
      fs.appendFileSync(file, `${JSON.stringify(record)}\n`, 'utf8');
      journalLines += 1;
      return true;
    } catch (error) {
      log('token 账本：写入失败', error?.message ?? error);
      return false;
    }
  }

  /**
   * 把内存态整体重写回磁盘：顺带丢掉被覆盖的旧采样，控制文件体积。
   */
  function compact() {
    const out = [];
    for (const base of baselines.values()) {
      out.push({
        k: 'base', t: base.t, sid: base.sessionId, key: base.key, seq: base.seq,
        model: base.model, provider: base.provider, b: base.buckets
      });
    }
    for (const s of samples.values()) {
      out.push({
        k: 'turn', t: s.t, sid: s.sessionId, key: s.key, seq: s.seq,
        turn: s.turn, step: s.step, retry: s.retry,
        model: s.model, provider: s.provider, b: s.buckets
      });
    }
    for (const e of evicted.values()) {
      out.push({
        k: 'evict', t: sessionSeen.get(e.sessionId) ?? 0, sid: e.sessionId, key: e.key,
        b: e.buckets, cost: e.cost, peak: e.byTier.peak, off: e.byTier.off,
        estimated: e.estimated, dropped: e.dropped, maxSeq: Number(e.maxSeq) >= 0 ? Number(e.maxSeq) : -1
      });
    }
    try {
      fs.mkdirSync(path.dirname(file), { recursive: true });
      const tmp = `${file}.${process.pid}.${Math.random().toString(16).slice(2, 10)}.tmp`;
      fs.writeFileSync(tmp, out.map((r) => `${JSON.stringify(r)}\n`).join(''), { encoding: 'utf8', mode: 0o600 });
      fs.renameSync(tmp, file);
      journalLines = out.length;
      evictedDirty = false;
      lastCompactedAt = now();
      return true;
    } catch (error) {
      log('token 账本：压缩失败', error?.message ?? error);
      return false;
    }
  }

  /**
   * 淘汰最旧的逐轮采样，把内存与磁盘都压在 maxLines 条以内。
   *
   * 为什么必须有淘汰：`compact()` 只是把内存态「重写」一遍 —— 如果内存本身无限增长，
   * 那么一旦条数越过上限，**之后每一条采样**都会触发一次整文件同步重写（多 MB 的
   * JSONL 写在 QQ 事件路径上），而且内存会一直涨。所以压缩必须真的丢东西。
   *
   * 关键：**丢明细不能丢总量**。被淘汰的采样先把桶累加进 `evicted`（连同它自己那一次的
   * 计价结果），读取时照样算进权威累计；只是这些旧回合不再能拆到具体某一轮，控制台会
   * 把它归进「未逐轮归属」。这样即使某个会话还没有投影基线，总量也依然精确。
   */
  function evictOldestSamples() {
    const overflow = samples.size - maxLines;
    if (overflow <= 0) return 0;
    const ordered = [...samples.entries()].sort((a, b) => a[1].t - b[1].t);
    for (let i = 0; i < overflow; i += 1) {
      const [sampleId, sample] = ordered[i];
      samples.delete(sampleId);
      const acc = evicted.get(sample.sessionId) ?? {
        sessionId: sample.sessionId,
        key: sample.key,
        buckets: { ...ZERO_BUCKETS },
        cost: 0,
        byTier: { peak: 0, off: 0 },
        estimated: false,
        dropped: 0,
        // 被淘汰采样里最大的 seq：一旦更新的投影基线覆盖到它，这些桶就已经包含在
        // 基线里了，再相加就是重复计费（见 evictionCovered()）。
        maxSeq: -1
      };
      acc.buckets = addBuckets(acc.buckets, sample.buckets);
      const sampleSeq = Number(sample.seq);
      if (Number.isFinite(sampleSeq) && sampleSeq >= 0) {
        acc.maxSeq = Math.max(Number(acc.maxSeq) >= 0 ? Number(acc.maxSeq) : -1, sampleSeq);
      }
      const p = priceUsage(sample.buckets, { model: sample.model, at: sample.t || now(), table: priceTable });
      acc.cost += p.cost.total;
      acc.byTier[p.rates.tier] += p.cost.total;
      acc.estimated ||= p.rates.estimated;
      acc.dropped += 1;
      if (sample.key) acc.key = sample.key;
      evicted.set(sample.sessionId, acc);
      evictedDirty = true;
    }
    return overflow;
  }

  /**
   * 被淘汰的累计桶是否已经被更**新的投影基线**整块覆盖。
   *
   * 为什么必须判断：基线（session/follow 快照的 tokenUsage）本身就是"整条会话日志"的
   * 权威累计，包含 seq <= asOfSeq 的全部事件；而淘汰桶没有 seq，旧代码无条件把它和基线
   * 相加 —— 一旦重连后拿到的基线覆盖了被淘汰的采样，这部分 token 与花费就被算了两遍
   * （实测 +33%，且只有 reset() 能清掉）。
   */
  function evictionCovered(entry) {
    if (!entry) return false;
    const base = baselines.get(entry.sessionId);
    const maxSeq = Number(entry.maxSeq);
    if (!base || !Number.isFinite(maxSeq) || maxSeq < 0) return false;
    return Number(base.seq) >= maxSeq;
  }

  function maybeCompact() {
    if (journalLines <= maxLines && samples.size <= maxLines && !evictedDirty) return;
    const dropped = evictOldestSamples();
    if (dropped > 0) log(`token 账本：逐轮明细超过 ${maxLines} 条，已淘汰最旧的 ${dropped} 条（累计总量不受影响）`);
    compact();
  }

  // ── 摄入 ────────────────────────────────────────────────────────────────

  /**
   * 摄入一条会话事件。
   * @param {object} args
   * @param {string} args.sessionId
   * @param {object} args.event       SessionEvent（含 seq/type/data）
   * @param {'live'|'snapshot'} [args.origin] 事件来源；仅用于统计，两路都走同一套幂等折叠
   * @param {number} [args.at]        事件时间（缺省用事件自带的 time）
   * @param {string} [args.model]     实际模型（调用方按会话跟踪；缺省尝试从事件里取）
   * @param {string} [args.provider]
   * @returns {boolean} 是否产生了新的账本记录
   */
  function ingestEvent({ sessionId, event, origin = 'live', at, model, provider } = {}) {
    if (typeof sessionId !== 'string' || !sessionId || !event || typeof event !== 'object') return false;
    // 时间优先取事件自带的 `time`（Unix 毫秒，由 DSH 写入），它决定峰谷档位；
    // 取不到才回落到「收到时间」，避免机器时钟/重放顺序影响计价。
    const t = Number.isFinite(Number(at)) ? Number(at)
      : (Number.isFinite(Number(event.time)) ? Number(event.time) : now());
    const seq = Number.isFinite(Number(event.seq)) ? Number(event.seq) : -1;
    const eventModel = model ?? event.data?.model ?? event.data?.usage?.model;
    const eventProvider = provider ?? event.data?.provider;

    if (event.type === 'llm/retry-started') {
      const turn = Number(event.data?.turn);
      const step = Number(event.data?.step);
      if (!Number.isFinite(turn) || !Number.isFinite(step)) return false;
      const retry = Number.isFinite(Number(event.data?.retry)) ? Number(event.data.retry) : 1;
      const axis = retryAxis.get(sessionId);
      if (axis && axis.seq > seq) return false;
      retryAxis.set(sessionId, { turn, step, retry, seq });
      return false; // 轴本身不入账
    }

    if (event.type !== 'assistant/message' && event.type !== 'assistant/attempt') return false;
    const raw = usageOfEvent(event);
    if (!raw) return false;
    const turn = Number(event.data?.turn);
    const step = Number(event.data?.step);
    if (!Number.isFinite(turn) || !Number.isFinite(step)) return false;
    const axis = retryAxis.get(sessionId);
    const retry = axis && axis.turn === turn && axis.step === step ? axis.retry : 0;
    const buckets = normalizeUsage(raw);
    if (totalTokensOf(buckets) === 0) return false;

    const key = keyOf(sessionId);
    const sk = sampleKeyOf(sessionId, turn, step, retry);
    const prev = samples.get(sk);
    if (prev && seq >= 0 && prev.seq >= seq) return false;

    const record = {
      k: 'turn', t, sid: sessionId, key, seq,
      turn, step, retry,
      model: eventModel ?? prev?.model,
      provider: eventProvider ?? prev?.provider,
      b: buckets
    };
    samples.set(sk, {
      sessionId, key, seq, t, turn, step, retry,
      model: record.model, provider: record.provider, buckets
    });
    if (t > (sessionSeen.get(sessionId) ?? 0)) sessionSeen.set(sessionId, t);
    lastIngestAt = now();
    appendLine(record);
    maybeCompact();
    return true;
  }

  /**
   * 摄入 `session/follow` snapshot 帧的投影基线（整条会话日志的权威累计桶）。
   * 语义：基线与「seq 大于 baseline.seq 的采样」相加才是会话总量。
   */
  function ingestBaseline({ sessionId, projection, at, model, provider } = {}) {
    if (typeof sessionId !== 'string' || !sessionId || !projection) return false;
    const values = projection.values ?? projection;
    const tokenUsage = values?.tokenUsage;
    if (!tokenUsage) return false;
    const seq = Number.isFinite(Number(projection.asOfSeq)) ? Number(projection.asOfSeq) : -1;
    const t = Number.isFinite(Number(at)) ? Number(at) : now();
    const buckets = normalizeUsage(tokenUsage);
    const prev = baselines.get(sessionId);
    if (prev && prev.seq > seq) return false;
    const key = keyOf(sessionId);
    const same = prev && prev.seq === seq
      && prev.buckets.cacheMiss === buckets.cacheMiss
      && prev.buckets.cacheHit === buckets.cacheHit
      && prev.buckets.cacheWrite === buckets.cacheWrite
      && prev.buckets.output === buckets.output;
    baselines.set(sessionId, { sessionId, key, seq, t, model, provider, buckets });
    if (t > (sessionSeen.get(sessionId) ?? 0)) sessionSeen.set(sessionId, t);
    // 内容与 seq 都没变的重复基线（重连重发）不再落盘：否则每重连一次就多一行，
    // 而 follow 集合只会增长，日志会被无意义地推大。
    if (same) return false;
    lastIngestAt = now();
    appendLine({
      k: 'base', t, sid: sessionId, key, seq,
      model, provider, b: buckets
    });
    // 基线也要参与压缩判断：重连风暴会在每个会话上各追加一行基线，
    // 只靠 ingestEvent 触发压缩的话，这些行可能永远等不到一次压缩。
    maybeCompact();
    return true;
  }

  /** 摄入 snapshot 的全部历史记录（逐轮明细回填）。 */
  function ingestSnapshotRecords({ sessionId, records, at } = {}) {
    if (!Array.isArray(records)) return 0;
    let n = 0;
    for (const entry of records) {
      const event = entry?.type === 'event' ? entry.event : entry;
      if (!event || typeof event !== 'object') continue;
      if (ingestEvent({ sessionId, event, origin: 'snapshot', at })) n += 1;
    }
    return n;
  }

  // ── 读取 ────────────────────────────────────────────────────────────────

  /** 按 sessionId 取「基线 + 基线之后采样」的权威桶。 */
  /**
   * 单遍索引：一次扫过所有采样，算出每个会话需要的全部聚合量。
   *
   * 为什么必须这么写：控制台每 3 秒拉一次 `/api/tokens/summary`，而它跑在**和 QQ 事件
   * 同一个事件循环**里。原先 summary 对每个会话都重扫一遍全部采样（O(会话数 × 采样数)，
   * 而且 sessionTotals/sessionCost/turnRows 各扫一遍 = 3 倍），几千条采样、几十个会话
   * 就是百万级迭代。这里压成单遍 O(采样数)。
   *
   * 基线单独计价（它没有逐时明细，只能整体按快照时刻的峰谷算），并只把
   * `seq > 基线 seq` 的采样算进「权威累计」，避免和基线重复计费。
   */
  function buildIndex() {
    const index = new Map();
    for (const s of samples.values()) {
      let entry = index.get(s.sessionId);
      if (!entry) {
        entry = {
          liveBuckets: { ...ZERO_BUCKETS },
          liveCost: 0,
          liveByTier: { peak: 0, off: 0 },
          allCost: 0,
          allByTier: { peak: 0, off: 0 },
          estimated: false,
          turns: new Map(),
          models: new Set()
        };
        index.set(s.sessionId, entry);
      }
      const base = baselines.get(s.sessionId);
      const isLive = !base || s.seq > base.seq;
      const p = priceUsage(s.buckets, { model: s.model, at: s.t || now(), table: priceTable });
      entry.estimated ||= p.rates.estimated;
      entry.allCost += p.cost.total;
      entry.allByTier[p.rates.tier] += p.cost.total;
      if (isLive) {
        entry.liveBuckets = addBuckets(entry.liveBuckets, s.buckets);
        entry.liveCost += p.cost.total;
        entry.liveByTier[p.rates.tier] += p.cost.total;
      }
      let row = entry.turns.get(s.turn);
      if (!row) {
        row = {
          sessionId: s.sessionId,
          key: s.key,
          turn: s.turn,
          steps: 0,
          attempts: 0,
          startedAt: s.t,
          endedAt: s.t,
          model: s.model,
          provider: s.provider,
          buckets: { ...ZERO_BUCKETS },
          cost: 0,
          tier: null
        };
        entry.turns.set(s.turn, row);
      }
      row.buckets = addBuckets(row.buckets, s.buckets);
      row.steps += 1;
      row.attempts += s.retry > 0 ? 1 : 0;
      if (s.t < row.startedAt) row.startedAt = s.t;
      if (s.t > row.endedAt) row.endedAt = s.t;
      if (s.model) { row.model = s.model; entry.models.add(s.model); }
      if (s.provider) row.provider = s.provider;
      row.cost += p.cost.total;
      row.tier = row.tier === null || row.tier === p.rates.tier ? p.rates.tier : 'mixed';
    }
    return index;
  }

  /** 把一个会话的索引项 + 投影基线合成「会话视图」。 */
  function sessionView(key, sessionIds, index) {
    const view = {
      key,
      sessionIds,
      sessions: sessionIds.length,
      buckets: { ...ZERO_BUCKETS },
      cost: 0,
      unattributedCost: 0,
      attributedCost: 0,
      byTier: { peak: 0, off: 0 },
      turns: 0,
      steps: 0,
      estimated: false,
      lastAt: 0,
      firstAt: 0,
      models: []
    };
    const models = new Set();
    for (const sid of sessionIds) {
      const base = baselines.get(sid);
      const entry = index.get(sid) ?? null;
      const rawGone = evicted.get(sid) ?? null;
      const gone = evictionCovered(rawGone) ? null : rawGone;
      const liveBuckets = entry?.liveBuckets ?? { ...ZERO_BUCKETS };
      const cumulative = addBuckets(base?.buckets ?? ZERO_BUCKETS, gone?.buckets ?? ZERO_BUCKETS);
      const total = addBuckets(cumulative, liveBuckets);
      view.buckets = addBuckets(view.buckets, total);

      let cost = (entry?.liveCost ?? 0) + (gone?.cost ?? 0);
      view.byTier.peak += (entry?.liveByTier.peak ?? 0) + (gone?.byTier.peak ?? 0);
      view.byTier.off += (entry?.liveByTier.off ?? 0) + (gone?.byTier.off ?? 0);
      view.estimated ||= (entry?.estimated ?? false) || (gone?.estimated ?? false);
      if (base) {
        const p = priceUsage(base.buckets, { model: base.model, at: base.t || now(), table: priceTable });
        cost += p.cost.total;
        view.byTier[p.rates.tier] += p.cost.total;
        view.estimated ||= p.rates.estimated;
      }
      view.cost += cost;
      if (entry) {
        for (const model of entry.models) models.add(model);
        for (const row of entry.turns.values()) {
          view.steps += row.steps;
          if (!view.firstAt || row.startedAt < view.firstAt) view.firstAt = row.startedAt;
          if (row.endedAt > view.lastAt) view.lastAt = row.endedAt;
        }
        view.turns += entry.turns.size;
      }
      // 能拆到具体某一轮的花费 = 所有采样的花费之和；差额是只剩累计值的历史。
      const attributed = entry?.allCost ?? 0;
      view.attributedCost += attributed;
      view.unattributedCost += Math.max(0, cost - attributed);
    }
    view.models = [...models];
    for (const sid of sessionIds) {
      const t = sessionSeen.get(sid);
      if (t && t > view.lastAt) view.lastAt = t;
    }
    return view;
  }

  /** 某个会话的逐轮明细（新的在前）。 */
  function turnsOf(sessionIds, index) {
    const rows = [];
    for (const sid of sessionIds) {
      const entry = index.get(sid);
      if (entry) rows.push(...entry.turns.values());
    }
    rows.sort((a, b) => b.turn - a.turn || b.endedAt - a.endedAt);
    return rows;
  }

  /** 账本里出现过的全部 sessionId（基线、采样、被淘汰汇总、活动记录四处的并集）。 */
  function allSessionIds() {
    const ids = new Set([...baselines.keys(), ...sessionSeen.keys(), ...evicted.keys()]);
    for (const s of samples.values()) ids.add(s.sessionId);
    return ids;
  }

  /** 汇总：全局 + 每个会话。 */
  function summary({ limit = 200 } = {}) {
    const index = buildIndex();
    const byKey = new Map();
    for (const sid of allSessionIds()) {
      const key = keyOf(sid) ?? (baselines.get(sid)?.key) ?? null;
      const bucketKey = key ?? `未归属:${sid.slice(0, 8)}`;
      if (!byKey.has(bucketKey)) byKey.set(bucketKey, []);
      byKey.get(bucketKey).push(sid);
    }
    const conversations = [];
    for (const [key, sids] of byKey) conversations.push(sessionView(key, sids, index));
    conversations.sort((a, b) => b.cost - a.cost || totalTokensOf(b.buckets) - totalTokensOf(a.buckets));

    const totals = {
      buckets: { ...ZERO_BUCKETS },
      cost: 0,
      attributedCost: 0,
      unattributedCost: 0,
      byTier: { peak: 0, off: 0 },
      turns: 0,
      steps: 0,
      conversations: conversations.length,
      estimated: false
    };
    for (const c of conversations) {
      totals.buckets = addBuckets(totals.buckets, c.buckets);
      totals.cost += c.cost;
      totals.attributedCost += c.attributedCost;
      totals.unattributedCost += c.unattributedCost;
      totals.byTier.peak += c.byTier.peak;
      totals.byTier.off += c.byTier.off;
      totals.turns += c.turns;
      totals.steps += c.steps;
      totals.estimated ||= c.estimated;
    }
    const totalTok = totalTokensOf(totals.buckets);
    const inputTok = totals.buckets.cacheMiss + totals.buckets.cacheHit + totals.buckets.cacheWrite;
    return {
      ok: true,
      generatedAt: now(),
      priceVersion: priceTable.version,
      currency: priceTable.currency,
      symbol: priceTable.symbol,
      nowTier: ratesAt(undefined, now(), priceTable),
      priceTable: {
        version: priceTable.version,
        currency: priceTable.currency,
        symbol: priceTable.symbol,
        unit: priceTable.unit,
        peakMultiplier: priceTable.peakMultiplier,
        peakWindows: priceTable.peakWindows,
        peakWeekdays: priceTable.peakWeekdays,
        holidayCount: (priceTable.holidays ?? []).length,
        models: Object.fromEntries(Object.entries(priceTable.models).map(([name, m]) => [name, { label: m.label, off: m.off }]))
      },
      totals: {
        ...totals,
        totalTokens: totalTok,
        inputTokens: inputTok,
        cacheHitRate: inputTok > 0 ? totals.buckets.cacheHit / inputTok : 0,
        avgCostPerTurn: totals.turns > 0 ? totals.cost / totals.turns : 0,
        avgCostPerStep: totals.steps > 0 ? totals.cost / totals.steps : 0
      },
      conversations: conversations.slice(0, Math.max(1, Math.min(500, Number(limit) || 200)))
    };
  }

  /** 单会话明细：汇总 + 逐轮（新的在前）。 */
  function conversation(key, { turnLimit = 200 } = {}) {
    if (typeof key !== 'string' || !key) return null;
    const sids = [];
    for (const sid of allSessionIds()) {
      if (keyOf(sid) === key) sids.push(sid);
    }
    if (!sids.length) return null;
    const index = buildIndex();
    const view = sessionView(key, sids, index);
    const rows = turnsOf(sids, index);
    return {
      ok: true,
      generatedAt: now(),
      currency: priceTable.currency,
      symbol: priceTable.symbol,
      ...view,
      turnLimit,
      turns: rows.slice(0, Math.max(1, Math.min(1000, Number(turnLimit) || 200)))
    };
  }

  /**
   * 按时间分桶的走势数据（控制台走势图用）。
   *
   * 三个刻意的设计：
   * 1. **单遍扫描**：控制台每 3 秒拉一次，不能是 O(会话×采样) 或多次遍历。
   * 2. **空桶补零**：没有调用的时段必须出现在图里（值为 0），否则"那段时间很安静"
   *    会被画成"那两个小时之间什么都没有"，图会撒谎。
   * 3. **桶的峰谷档位**：桶里只有一种档位就用它；混了就标 `mixed`；**空桶按该时刻
   *    实际属于高峰还是空闲来标** —— 这样即使某段时间完全没用量，图上也能看出
   *    "这段时间本来就是 2 倍价"。
   * 桶宽阶梯与选择规则在模块级（`BUCKET_LADDER_MS` / `pickBucketMs`），离线 fixture 复用同一份。
   */
  function series({ hours = 24, maxBuckets = 48, at } = {}) {
    const endAt = Number.isFinite(Number(at)) ? Number(at) : now();
    const windowHours = Math.min(24 * 90, Math.max(1, Number(hours) || 24));
    const windowMs = windowHours * 60 * 60 * 1000;
    const bucketMs = pickBucketMs(windowMs, Math.max(6, Math.min(120, Number(maxBuckets) || 48)));
    // 桶边界对齐到 bucketMs 的整数倍，这样刷新之间桶不会"漂移"。
    const end = Math.floor(endAt / bucketMs) * bucketMs + bucketMs;
    const start = end - Math.ceil(windowMs / bucketMs) * bucketMs;
    const count = Math.round((end - start) / bucketMs);

    const rows = [];
    for (let i = 0; i < count; i += 1) {
      const t = start + i * bucketMs;
      rows.push({
        t,
        cost: 0,
        tokens: 0,
        cacheMiss: 0,
        cacheHit: 0,
        cacheWrite: 0,
        output: 0,
        turns: 0,
        steps: 0,
        tier: null,
        keys: new Set()
      });
    }
    const totals = { cost: 0, tokens: 0, turns: 0, steps: 0 };
    let estimated = false;
    let countedSamples = 0;
    // 逐桶的「轮次」去重集合（键 = sessionId|turn）。
    // steps 数的是计费请求（同一回合的多个 step / 重试各算一条），turns 数的是真正的对话轮次；
    // 旧代码把去重集合的键写成 sampleKeyOf(sid, turn, step, retry) —— 那**正是 samples 的键**，
    // 遍历 samples 时每个键天然唯一，去重永不命中，于是 row.turns 恒等于 row.steps：
    // 图上标的「轮次」其实一直是「请求数」，且这层去重是纯粹的死代码。
    const turnSeen = new Set();
    const rowTurnSeen = rows.map(() => new Set());
    // 权威总额（含窗口之外）的三个组成部分，按 summary() 的同一口径在**同一次遍历**里累加：
    //   ① 被淘汰采样的累计（evicted）② 投影基线自身 ③ 仍然逐轮可拆、且没被基线覆盖的采样（seq > 基线 seq）。
    // 非 live 采样绝不能计入：基线已经包含了它的历史，再算一次就是重复计费，offChartCost 会凭空变大。
    let ledgerCost = 0;
    let ledgerBuckets = { ...ZERO_BUCKETS };
    for (const e of evicted.values()) {
      if (evictionCovered(e)) continue; // 已被投影基线覆盖：再加一次就是重复计费
      ledgerCost += Number(e.cost) || 0;
      ledgerBuckets = addBuckets(ledgerBuckets, e.buckets);
    }
    for (const base of baselines.values()) {
      // 与 sessionView() 同口径：基线没有逐时明细，按快照时刻（缺失时取当前）整体计价。
      ledgerCost += priceUsage(base.buckets, { model: base.model, at: base.t || now(), table: priceTable }).cost.total;
      ledgerBuckets = addBuckets(ledgerBuckets, base.buckets);
    }
    for (const s of samples.values()) {
      // 这条采样的计价只算一次：窗口内的桶与窗口外的"总额"共用同一个 p。
      const p = priceUsage(s.buckets, { model: s.model, at: s.t || endAt, table: priceTable });
      const tokens = totalTokensOf(s.buckets);
      const base = baselines.get(s.sessionId);
      if (!base || s.seq > base.seq) {
        ledgerCost += p.cost.total;
        ledgerBuckets = addBuckets(ledgerBuckets, s.buckets);
      }
      if (s.t < start || s.t >= end) continue;
      const idx = Math.min(count - 1, Math.max(0, Math.floor((s.t - start) / bucketMs)));
      const row = rows[idx];
      row.cost += p.cost.total;
      row.tokens += tokens;
      row.cacheMiss += s.buckets.cacheMiss;
      row.cacheHit += s.buckets.cacheHit;
      row.cacheWrite += s.buckets.cacheWrite;
      row.output += s.buckets.output;
      row.steps += 1;
      row.tier = row.tier === null || row.tier === p.rates.tier ? p.rates.tier : 'mixed';
      row.keys.add(keyOf(s.sessionId) ?? s.key ?? null);
      const turnId = `${s.sessionId}|${s.turn}`;
      if (!rowTurnSeen[idx].has(turnId)) {
        rowTurnSeen[idx].add(turnId);
        row.turns += 1;
      }
      // 窗口级轮次：跨桶的同一回合只算一次（所以各桶 turns 之和 ≥ totals.turns）。
      if (!turnSeen.has(turnId)) {
        turnSeen.add(turnId);
        totals.turns += 1;
      }
      totals.cost += p.cost.total;
      totals.tokens += tokens;
      totals.steps += 1;
      estimated ||= p.rates.estimated;
      countedSamples += 1;
    }
    for (const row of rows) {
      // 空桶也要有档位：用该时刻真实的峰谷归属，而不是留空。
      if (row.tier === null) row.tier = isPeakAt(row.t, priceTable) ? 'peak' : 'off';
      row.keys = [...row.keys].filter(Boolean);
    }

    let busiest = null;
    for (const row of rows) {
      if (row.cost <= 0) continue;
      if (!busiest || row.cost > busiest.cost) busiest = row;
    }
    // 时间轴之外的那部分（更早的回合 / 投影基线覆盖的历史）。
    //
    // 旧实现在这里调 summary({ limit: 1 }) 只为拿一个总额 —— 那是又一次 buildIndex()（全量采样、
    // 每条约一次 priceUsage、还建两层 Map）加逐会话聚合，把控制台每 3 秒轮询一次的最热端点成本
    // 直接翻倍，也违背了本函数头「单遍扫描」的约定。现在用上面同一次遍历累出的权威总额。
    const ledgerTokens = totalTokensOf(ledgerBuckets);
    return {
      ok: true,
      generatedAt: endAt,
      currency: priceTable.currency,
      symbol: priceTable.symbol,
      hours: windowHours,
      bucketMs,
      from: start,
      to: end,
      buckets: rows.map(({ keys, ...rest }) => ({ ...rest, keys })),
      totals,
      busiest: busiest ? { t: busiest.t, cost: busiest.cost, tokens: busiest.tokens } : null,
      estimated,
      countedSamples,
      // 时间轴之外的部分：图上画不出来，控制台会单独说明，避免"图上加起来 ≠ 总花费"引起误会。
      offChartCost: Math.max(0, ledgerCost - totals.cost),
      offChartTokens: Math.max(0, ledgerTokens - totals.tokens)
    };
  }

  /** 最近采样流水（实时观察用，按时间倒序）。 */
  function recent(limit = 50) {
    const n = Math.max(1, Math.min(500, Number(limit) || 50));
    const all = [...samples.values()].sort((a, b) => b.t - a.t);
    const out = [];
    for (const s of all.slice(0, n)) {
      const p = priceUsage(s.buckets, { model: s.model, at: s.t || now(), table: priceTable });
      out.push({
        time: s.t,
        key: keyOf(s.sessionId) ?? s.key ?? null,
        sessionId: s.sessionId,
        turn: s.turn,
        step: s.step,
        retry: s.retry,
        model: s.model ?? null,
        tokens: totalTokensOf(s.buckets),
        ...s.buckets,
        cost: p.cost.total,
        tier: p.rates.tier,
        estimated: p.rates.estimated
      });
    }
    return out;
  }

  function stats() {
    return {
      file,
      journalLines,
      maxLines,
      droppedLines,
      sessions: new Set([...baselines.keys(), ...sessionSeen.keys()]).size,
      baselines: baselines.size,
      samples: samples.size,
      lastIngestAt,
      lastCompactedAt,
      priceVersion: priceTable.version
    };
  }

  /** 清空账本（控制台手动重置）。 */
  function reset() {
    baselines.clear();
    samples.clear();
    retryAxis.clear();
    sessionKeys.clear();
    sessionSeen.clear();
    // 被淘汰采样汇总是「权威累计」的一部分：漏清它，控制台点完重置仍会报出旧花费，
    // 用户以为没清掉。更隐蔽的是 evictedDirty 若保持 true，下一次 maybeCompact() 会走
    // compact()，把内存里这份 **已经被删掉** 的 evicted 重新序列化回磁盘 ——
    // 「删除」的总额在下一个采样到来时原地复活，而且文件被写坏成"重置前的量"。
    evicted.clear();
    evictedDirty = false;
    droppedLines = 0;
    lastCompactedAt = 0;
    journalLines = 0;
    try {
      fs.mkdirSync(path.dirname(file), { recursive: true });
      fs.writeFileSync(file, '', { encoding: 'utf8', mode: 0o600 });
    } catch (error) {
      log('token 账本：重置失败', error?.message ?? error);
    }
    lastIngestAt = 0;
  }

  load();

  return {
    ingestEvent,
    ingestBaseline,
    ingestSnapshotRecords,
    summary,
    conversation,
    series,
    recent,
    stats,
    reset,
    compact,
    hasSession: (sid) => baselines.has(sid) || sessionSeen.has(sid),
    file
  };
}
````


# 源码文件：package.json
- 原始路径：package.json
- 原始文件大小：3774 字节
- 原始 SHA256：b6a395123fcc655022c1e0ebebfc0fa52fd3664a408ce67ab746c09ee8baeadd
- 原始行数：72
- 是否脱敏：是（QQ号/真实姓名 → 占位符；行数与行序保持原样，可行号对照）
## 完整源码

````javascript
{
  "name": "qq-bridge",
  "version": "0.2.1-r1",
  "private": true,
  "license": "MIT",
  "description": "Bridge between SnowLuma (OneBot v11 / QQ) and DeepSeek Harness Web API: QQ messages become DSH agent prompts, agent replies go back to QQ. Targets the DSH Cookie-auth / slash-RPC / remote.mux protocol generation, so it deliberately names no DSH version here.",
  "type": "module",
  "engines": {
    "node": ">=22.13.0"
  },
  "scripts": {
    "start": "node src/bridge.js",
    "test:audit": "node scripts/test-audit.mjs",
    "test:token": "node scripts/test-token-economy.mjs && node scripts/test-reply-wait.mjs && node scripts/test-qq-model-view.mjs && node scripts/test-qq-preset-contract.mjs",
    "test:tokens": "node scripts/test-token-usage.mjs",
    "test:hardening": "node scripts/test-hardening.mjs",
    "test:reply-wait": "node scripts/test-reply-wait.mjs",
    "self-test": "node src/self-test.js",
    "test-md": "node scripts/test-md-to-plain.mjs",
    "test-wait": "node scripts/test-v2-wait.mjs",
    "test:initial-prompt": "node scripts/test-v2-initial-prompt.mjs",
    "test:regressions": "node scripts/test-audit-regressions.mjs",
    "test:media-search": "node scripts/test-media-search.mjs",
    "test:media-normalize": "node scripts/test-media-normalize.mjs",
    "test:issues": "node scripts/test-issue-regressions.mjs",
    "test-wait:e2e": "node scripts/test-dsh-v2-wait.mjs",
    "test-vision": "node scripts/test-vision-media.mjs",
    "test-forward": "node scripts/test-forward.mjs",
    "test-slang": "node scripts/test-slang-learn.mjs",
    "test-stickers": "node scripts/test-stickers.mjs",
    "test-voice": "node scripts/test-voice.mjs",
    "test-voice:gui": "node scripts/test-voice-gui.mjs",
    "test-voice:system": "node scripts/test-voice-system.mjs",
    "test-encoding": "node scripts/test-encoding.mjs",
    "send-voice": "node scripts/send-voice.mjs",
    "voice-gui": "cd ../voice-tool && node voice-gui.mjs",
    "voice-cli": "cd ../voice-tool && node voice-cli.mjs",
    "test-console": "node scripts/test-console.mjs",
    "test:console-ui": "node scripts/test-console-ui.mjs",
    "test:preset-prompt": "node scripts/test-preset-prompt.mjs",
    "test:role-card": "node scripts/test-role-card.mjs",
    "preview:console": "node scripts/console-ui-fixture.mjs",
    "verify:adaptation": "node scripts/verify-dsh-021-adaptation.mjs",
    "test:dsh-auth": "node scripts/test-dsh-auth.mjs",
    "test:dsh-inbox": "node scripts/test-dsh-inbox-projection.mjs",
    "test:session-retire:e2e": "node scripts/test-dsh-session-retire.mjs",
    "verify:persona": "node scripts/verify-persona-config.mjs",
    "test:mcp-servers": "node scripts/test-mcp-servers-stdio.mjs",
    "test:plugin": "node scripts/test-qq-mode-plugin.mjs",
    "test:setup-idempotent": "node scripts/test-setup-dsh-idempotent.mjs",
    "test:mux-reconnect": "node scripts/test-mux-reconnect.mjs",
    "probe:events": "node scripts/probe-events.mjs",
    "probe:tools": "node scripts/probe-tools.mjs",
    "probe:protocol": "node scripts/probe-020-protocol.mjs",
    "probe:settings": "node scripts/probe-020-settings.mjs",
    "probe:heartbeat": "node scripts/probe-snowluma-heartbeat.mjs",
    "scan:tool-names": "node scripts/scan-dsh-tool-names.mjs",
    "postinstall": "node scripts/patch-snowluma-sdk.mjs",
    "privacy-scan": "node scripts/privacy-scan.mjs",
    "diagnose:acl": "node scripts/diagnose-acl.mjs",
    "harden-acl": "node scripts/harden-state-acl.mjs"
  },
  "dependencies": {
    "@deepseek-ai/dsh-host-apiproxy": "0.1.1-rc.2",
    "@deepseek-ai/schemastery": "^3.18.1",
    "@modelcontextprotocol/sdk": "^1.30.0",
    "@snowluma/mcp": "^1.14.20",
    "@snowluma/sdk": "^1.14.20",
    "js-yaml": "^4.3.2",
    "zod": "^4.4.3"
  }
}
````


# 源码文件：config.example.json
- 原始路径：config.example.json
- 原始文件大小：7449 字节
- 原始 SHA256：e1346f09b2d4f4bef32c316616a0486867f0233065d26044fad6af5fcee87435
- 原始行数：279
- 是否脱敏：是（QQ号/真实姓名 → 占位符；行数与行序保持原样，可行号对照）
## 完整源码

````javascript
{
  "dsh": {
    "baseUrl": "http://127.0.0.1:3080",
    "provider": "deepseek-official",
    "model": "deepseek-flash",
    "reasoningEffort": "max",
    "authToken": "",
    "authHeader": "authorization",
    "authPrefix": "Bearer",
    "allowRemote": false
  },
  "role": {
    "maxInjectChars": 6000
  },
  "snowluma": {
    "wsUrl": "ws://127.0.0.1:3001",
    "httpUrl": "http://127.0.0.1:3000",
    "accessToken": "",
    "followDiscoveredEndpoint": true,
    "silenceWarnMs": 600000,
    "launcherPath": "",
    "homeDir": ""
  },
  "ownerQQ": null,
  "sessionCwd": "",
  "agentPreset": "qq-chat",
  "workspaceTitle": "QQ 聊天",
  "allow": {
    "private": [],
    "groups": []
  },
  "deny": {
    "private": [],
    "groups": []
  },
  "allowAllWhenEmpty": false,
  "ackMessage": "",
  "sendDelayMs": 300,
  "questionTimeoutMs": 300000,
  "consolePort": 3100,
  "consoleToken": "",
  "pricing": {
    "models": {
      "my-relay-model": {
        "label": "我的中转模型",
        "off": {
          "cacheHit": 0.5,
          "cacheMiss": 3,
          "output": 9
        }
      }
    }
  },
  "security": {
    "interceptNotify": true
  },
  "slang": {
    "enabled": true,
    "extractMinMessages": 10,
    "extractCooldownMs": 300000,
    "inferenceThresholds": [
      2,
      4,
      8
    ],
    "injectMax": 8,
    "learnerPreset": "qq-chat",
    "workspaceTitle": "QQ 黑话学习",
    "autoResearch": true
  },
  "social": {
    "enabled": true,
    "maxReplyChars": 500,
    "mustReplyKeywords": [
      "deepseek",
      "小鲸鱼",
      "大肥鱼",
      "鲸鱼",
      "d指导",
      "在吗"
    ],
    "contextWindow": 20,
    "triggerProbability": 0.1,
    "activeCheckMinMs": 8000,
    "activeCheckMaxMs": 30000,
    "idleWindowMs": 360000,
    "idleRetryProbability": 0.4,
    "proactiveEnabled": true,
    "proactiveIdleThresholdMs": 1800000,
    "proactiveCheckMinMs": 2700000,
    "proactiveCheckMaxMs": 5400000,
    "proactiveProbability": 0.2,
    "activeReplyDelayMinMs": 1000,
    "activeReplyDelayMaxMs": 5000,
    "activeDurationEnabled": true,
    "activeDurationMinMs": 1200000,
    "activeDurationMaxMs": 3000000,
    "skipProbability": 0.05,
    "surrenderProbability": 0,
    "idleRetryWaitMs": 120000,
    "burstEnabled": true,
    "burstIntervalMinMs": 1000,
    "burstIntervalMaxMs": 3000,
    "longGapProbability": 0.2,
    "longGapMinMs": 5000,
    "longGapMaxMs": 10000
  },
  "socialV2": {
    "enabled": true,
    "autoReplyCheckMs": 30000,
    "agentPreset": "qq-chat-v2",
    "provideRecommendations": true,
    "tools": {
      "getPrompt": true,
      "getUnread": true,
      "getRecent": true,
      "socialState": true,
      "sendGroup": true,
      "sendPrivate": true,
      "reply": true,
      "sendBurst": true,
      "sendMessage": true,
      "waitMessages": true,
      "feedback": true,
      "getMyRecent": true,
      "getMessageDetail": true,
      "getActiveMembers": true,
      "setWakeConfig": true,
      "markRead": true,
      "memory": true,
      "slangQuery": true,
      "slangSubmit": true,
      "getImages": true,
      "getForwardMsg": true,
      "sendPoke": true,
      "listStickers": true,
      "getStickerImage": true,
      "sendSticker": true,
      "setStickerRemark": false,
      "stickerNote": true,
      "collectSticker": true,
      "getSelfImage": true,
      "sendVoice": false
    },
    "wake": {
      "defaultMode": "diving",
      "preSleepWaitEnabled": true,
      "preSleepWaitMs": 300000,
      "staleChatMinutes": 10,
      "staleSilentMinutes": 120,
      "recommendedDefaultInfinite": true,
      "sleepMinMs": 60000,
      "sleepMaxMs": 0,
      "recommendedSleepMinMs": 300000,
      "recommendedSleepMaxMs": 7200000,
      "recommendedProbability": 0.05,
      "recommendedKeywords": [
        "小鲸鱼",
        "DeepSeek",
        "deepseek",
        "DS",
        "D老师",
        "d老师",
        "D指导",
        "d指导",
        "D师傅",
        "d师傅",
        "深度求索",
        "大肥鱼",
        "鲸鱼",
        "DeepSeek V3",
        "DeepSeek R1",
        "R1"
      ],
      "recommendedAtMention": true,
      "recommendedNameMention": true,
      "recommendedQuestion": true,
      "recommendedPoke": true,
      "recommendedHint": "如果你要潜水，推荐先调用 qq_wait_for_messages(timeoutMs=300000) 完成一次沉睡前观察：5 分钟内没人说话就可以设置下一次唤醒并沉睡；若期间有人发新消息，先查看 newMessages，判断不需要你参与可直接沉睡，若参与了则下次想睡需再等观察窗口。潜水时长推荐 5~120 分钟，普通消息概率 0.05；@/名字/关键词/提问唤醒建议保持开启。需要等特定某人/某几人时，可额外设置 triggers.speakerIds。",
      "batchWindowMs": 8000,
      "maxWakePerMinute": 1,
      "maxWakePerHour": 12,
      "noActionLimit": 3,
      "maxWakeConfigReminders": 2
    },
    "send": {
      "burstEnabled": true,
      "burstMaxMessages": 8,
      "burstIntervalMinMs": 1000,
      "burstIntervalMaxMs": 3000,
      "longGapProbability": 0.2,
      "longGapMinMs": 5000,
      "longGapMaxMs": 10000,
      "maxSendPerMinute": 8,
      "maxSendPerHour": 60,
      "maxMessageChars": 500,
      "maxGapMs": 10000,
      "gapBaseMs": 800,
      "gapPerCharMs": 20,
      "recommendedHint": "单条消息不要用空格断句，中文短句用逗号/顿号或直接连写；分条时每条都要读起来完整，不要把同一句话拆到两条里；普通闲聊建议一次 1~3 条，条间 1~3 秒；讲故事/回忆可以 5~10 秒间隔；不要连续刷屏。"
    },
    "wait": {
      "defaultMs": 30000,
      "minMs": 5000,
      "maxMs": 600000,
      "defaultQuietMs": 8000,
      "minQuietAfterNewMs": 10000
    },
    "sticker": {
      "enabled": true,
      "syncTtlMs": 60000,
      "maxListCount": 100,
      "includeInPrompt": true,
      "promptMaxStickers": 8,
      "collect": {
        "enabled": true,
        "maxPerMinute": 2,
        "maxPerHour": 10,
        "maxRemarkChars": 20
      }
    },
    "voice": {
      "enabled": false,
      "dir": "audio",
      "maxSeconds": 300,
      "maxBytes": 20971520,
      "maxPerMinute": 2,
      "maxPerHour": 10,
      "allowAbsolutePath": false,
      "defaultVolume": 1,
      "loudness": "normal",
      "ffmpegPath": "",
      "autoReadback": true
    },
    "proactive": {
      "enabled": true,
      "checkIntervalMinMs": 1800000,
      "checkIntervalMaxMs": 5400000,
      "idleThresholdMs": 900000,
      "probability": 0.3
    },
    "feedback": {
      "maxLength": 500,
      "notifyOwnerOnError": false
    },
    "context": {
      "recentLimit": 100,
      "unreadLimit": 30,
      "contextWindow": 20,
      "inlineWakeMessages": true,
      "wakeMessageLimit": 12,
      "wakeRecentLimit": 4,
      "wakeMaxChars": 6000
    },
    "initialPromptMaxChars": 2000,
    "media": {
      "maxBytes": 8388608,
      "maxPixels": 64000000,
      "maxCount": 5,
      "normalize": true,
      "maxModelBytes": 1048576,
      "maxLongEdge": 1280,
      "gifFrames": 6,
      "gifTile": 240,
      "normalizeTimeoutMs": 25000
    }
  },
  "webSearch": {
    "provider": "auto",
    "apiKey": "",
    "apiKeyEnv": "DEEPSEEK_API_KEY",
    "baseURL": "",
    "model": "deepseek-v4-flash",
    "maxUses": 5,
    "maxTokens": 4096,
    "maxResults": 8,
    "timeoutMs": 35000
  }
}
````



---

# 附：锁 / 原子写 / 备份（本卷与 `02-bridge.md` 中的位置）

| 功能 | 函数 | 位置 |
|---|---|---|
| 单实例锁路径 | `LOCK_FILE` | bridge.js:1054 |
| 端口占用探测 | `consolePortInUse()` | bridge.js:1064-1078 |
| 获取锁 | `acquireLock()` | bridge.js:1080-1142 |
| 释放锁 | `releaseLock()` | bridge.js:1143-1149 |
| 取锁调用点 | `acquireLock()` | bridge.js:**1390** |
| 端口监听 | `server.listen()` | bridge.js:**7531** |
| EADDRINUSE 处理 | `server.on('error')` | bridge.js:7535-7542 |
| 原子写 JSON（正确范例） | `atomicWriteJson()` | bridge.js:182-193 |
| 原子写文本 | `atomicWriteText()` | bridge.js:196-210 |
| rename 重试 | `renameWithRetry()` | bridge.js:165-179 |
| **A 层原子重写（缺陷点）** | `atomicRewrite(rows)` | memory-vector.js:**376-380** |
| 每日备份 | `dailyBackup()` | memory-vector.js:131-147 |
| 备份唯一调用点 | `dailyBackup()` | memory-vector.js:154（仅 append 内） |
| admin 前备份 | `copyFileSync(VEC_FILE_PATH, ...admin-backup.jsonl)` | bridge.js:6716 |

## 正确范例（完整函数，未省略）

````javascript
// bridge.js:165-179
function renameWithRetry(from, to, attempts = 4) {
  let lastError;
  for (let i = 0; i < attempts; i += 1) {
    try {
      fs.renameSync(from, to);
      return;
    } catch (error) {
      lastError = error;
      if (error?.code !== 'EPERM' && error?.code !== 'EBUSY' && error?.code !== 'EACCES') throw error;
      try { Atomics.wait(new Int32Array(new SharedArrayBuffer(4)), 0, 0, 40 * (i + 1)); } catch {}
    }
  }
  throw lastError;
}
````

## 缺陷点（完整函数，未省略）

````javascript
// memory-vector.js:376-380  ★ 固定 tmp 名，无 pid/随机，无 rename 重试，无 fsync
function atomicRewrite(rows) {
  const tmp = VEC_FILE + '.tmp';
  fs.writeFileSync(tmp, rows.map((r) => JSON.stringify(r)).join('
') + '
', 'utf8');
  fs.renameSync(tmp, VEC_FILE);
}
````

## 备份事实

| 项 | 实际 |
|---|---|
| 触发 | 仅 `appendVectorMemory` 内、`appendFileSync` 之前 |
| 粒度 | 每日一次（`if (!fs.existsSync(dest))` 同一天只备份第一次） |
| 保留 | 7 天（`age > 7` 删除） |
| 失败处理 | `catch {}` 静默，无提示 |
| 是否检查 schema | **否** |
| 是否有恢复函数 | **否**（全仓库无 `restore memory-vectors` 实现） |
| 删除/编辑是否触发 | **否** |

## 配置要点（`config.example.json`）

| 键 | 默认 | 说明 |
|---|---|---|
| `memoryVector.enabled` | `true` | A 层开关 |
| `memoryVector.endpoint` | `http://127.0.0.1:11434` | Ollama 地址 |
| `memoryVector.model` | `bge-m3` | embedding 模型 |
| `memoryVector.search.topK` | `3` | 检索返回条数 |
| `memoryVector.search.threshold` | `0.3` | 相似度阈值 |
| `socialV2.tools.memory` | `true` | 记忆工具总开关 |
| `consolePort` | `3100` | 控制台端口（锁探测依赖它） |
