# 源码文件：src/memory-vector.js
- 原始路径：src/memory-vector.js
- 原始文件大小：15788 字节
- 原始 SHA256：1d6dc555920c7edcf948701faf54980efb54bf49eef03c99f48b456c01af114a
- 原始行数：380
- 是否脱敏：是（QQ号/真实姓名 → 占位符；行数与行序保持原样，可行号对照）
## 完整源码

````javascript
// 向量记忆模块：qq_memory_* 的语义检索升级版。
//
// 存储：state/memory-vectors.jsonl，一行一条 JSON：
//   { key, text, time, vec: [1024 floats] }
// 追加写入（不重写整文件）；读取带 mtime 缓存。
//
// 向量来源：本机 Ollama（默认 http://127.0.0.1:11434，模型 bge-m3）。
// 记忆文本不出本机。
//
// 降级约定：embed() 失败会 throw，调用方（mcp-snowluma-safe.js）
// 捕获后应退回原有的关键词匹配路径，本模块不静默吞错。
import fs from 'node:fs';
import path from 'node:path';
import http from 'node:http';

function projectRoot() {
  try {
    const entry = String(process.argv[1] ?? '');
    if (entry) {
      const dir = path.dirname(path.resolve(entry));
      if (path.basename(dir) === 'src') return path.dirname(dir);
    }
  } catch {}
  return process.cwd();
}

const VEC_FILE = path.join(projectRoot(), 'state', 'memory-vectors.jsonl');
const MEM_CONFIG = (() => {
  try {
    return JSON.parse(fs.readFileSync(path.join(projectRoot(), 'config.json'), 'utf8')).memoryVector ?? {};
  } catch { return {}; }
})();
const ENDPOINT = MEM_CONFIG.endpoint || 'http://127.0.0.1:11434';
const MODEL = MEM_CONFIG.model || 'bge-m3';
// 检索限流（v1.1 轻量版边界2）：全部可配置，允许空结果
const SEARCH_CFG = MEM_CONFIG.search ?? {};
const SEARCH_TOPK = Math.max(1, Number(SEARCH_CFG.topK) || 3);
const SEARCH_MAX_PER_HIT = Math.max(20, Number(SEARCH_CFG.maxPerHit) || 300);
const SEARCH_MAX_TOTAL = Math.max(200, Number(SEARCH_CFG.maxTotal) || 1200);
const SEARCH_THRESHOLD = Math.min(0.9, Math.max(0, Number(SEARCH_CFG.threshold ?? 0.3)));

let cache = { mtime: 0, rows: [] };

function loadRows() {
  try {
    const mtime = fs.existsSync(VEC_FILE) ? fs.statSync(VEC_FILE).mtimeMs : 0;
    if (mtime === cache.mtime) return cache.rows;
    if (!mtime) { cache = { mtime: 0, rows: [] }; return []; }
    const rows = [];
    const lines = fs.readFileSync(VEC_FILE, 'utf8').split('\n');
    for (const line of lines) {
      if (!line.trim()) continue;
      try { rows.push(JSON.parse(line)); } catch {}
    }
    cache = { mtime, rows };
    return rows;
  } catch { return []; }
}

// 严格读取：供 remove / clear 等「需要区分『无记录』与『文件异常』」的写路径使用。
// 与 loadRows 的关键差异：
//   ① 用 fs.existsSync 明确判断文件是否存在（不用 !mtime，避免与 cache.mtime=0 撞车）；
//   ② 文件存在时【每次重读】，不信任普通 loadRows 的缓存（它可能已把坏行跳过后的
//      不完整结果写进缓存）；
//   ③ 每条非空记录严格 JSON.parse，读取/解析失败直接抛错（不静默 skip、不吞成 []）；
//   ④ 严格解析成功后更新缓存。
// 文件不存在是「正常空结果」，重置缓存并返回 []。
function loadRowsStrict() {
  const exists = fs.existsSync(VEC_FILE);
  if (!exists) {
    cache = { mtime: 0, rows: [] };
    return [];
  }
  const mtime = fs.statSync(VEC_FILE).mtimeMs;
  const rows = [];
  const lines = fs.readFileSync(VEC_FILE, 'utf8').split('\n');
  for (const line of lines) {
    if (!line.trim()) continue;
    rows.push(JSON.parse(line));
  }
  cache = { mtime, rows };
  return rows;
}

// 本机 Ollama 的 embedding。失败 throw，调用方负责降级。
export function embed(text) {
  return new Promise((resolve, reject) => {
    const body = JSON.stringify({ model: MODEL, prompt: String(text ?? '').slice(0, 4000) });
    const req = http.request(`${ENDPOINT}/api/embeddings`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'Content-Length': Buffer.byteLength(body) },
      timeout: 30000,
    }, (res) => {
      let buf = '';
      res.on('data', (c) => { buf += c; });
      res.on('end', () => {
        try {
          const out = JSON.parse(buf);
          if (Array.isArray(out.embedding) && out.embedding.length) resolve(out.embedding);
          else reject(new Error('embedding 返回为空'));
        } catch (e) { reject(e); }
      });
    });
    req.on('error', reject);
    req.on('timeout', () => { req.destroy(new Error('embedding 请求超时')); });
    req.end(body);
  });
}

export function cosSim(a, b) {
  if (!Array.isArray(a) || !Array.isArray(b) || a.length !== b.length) return 0;
  let dot = 0, na = 0, nb = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    na += a[i] * a[i];
    nb += b[i] * b[i];
  }
  const denom = Math.sqrt(na) * Math.sqrt(nb);
  return denom > 0 ? dot / denom : 0;
}

// 追加一条向量记忆。返回写入的记录。
// 1) 信息量检查：太短/太少汉字/纯符号的拒绝（防"喵"、纯表情进库）
// 2) 每日自动备份：当天第一次写入前，把整个库快照到 backups/（保留最近 7 份）
function infoOK(text) {
  const t = String(text ?? '').replace(/\[CQ:[^\]]*\]/g, '').trim();
  if (t.length < 6) return false;
  const han = (t.match(/[\u4e00-\u9fff]/g) || []).length;
  return han >= 3;
}
function dailyBackup() {
  try {
    if (!fs.existsSync(VEC_FILE)) return;
    const today = new Date().toISOString().slice(0, 10);
    const dir = path.join(projectRoot(), 'backups');
    const dest = path.join(dir, `memory-vectors-${today}.jsonl`);
    fs.mkdirSync(dir, { recursive: true });
    if (!fs.existsSync(dest)) fs.copyFileSync(VEC_FILE, dest);
    // 只保留最近 7 份
    for (const f of fs.readdirSync(dir)) {
      const m = f.match(/^memory-vectors-(\d{4}-\d{2}-\d{2})\.jsonl$/);
      if (!m) continue;
      const age = (Date.now() - new Date(m[1]).getTime()) / 86400000;
      if (age > 7) fs.unlinkSync(path.join(dir, f));
    }
  } catch {}
}

export async function appendVectorMemory(key, text, time) {
  const clean = String(text ?? '').trim().slice(0, 1000);
  if (!clean) throw new Error('记忆内容为空');
  if (!infoOK(clean)) throw new Error('信息量不足（太短或无实义），不入库');
  const vec = await embed(clean);
  dailyBackup();
  const row = {
    key, text: clean, time: time ?? Date.now(), vec,
    share_review_status: 'not_requested',   // 分享审核：未申请（独立于 needs_review）
  };
  fs.mkdirSync(path.dirname(VEC_FILE), { recursive: true });
  fs.appendFileSync(VEC_FILE, JSON.stringify(row) + '\n', 'utf8');
  cache.mtime = 0; // 下次读强制重载
  return { text: clean, time: row.time };
}

// 语义检索：本会话（key）的记忆里找与 topic 最相近的条目。
// 权限过滤（v1.1 + 分享审核）：
//   私聊（主人）查询 → 全部可见（owner_private + group_shared 均可）
//   群聊查询        → 只返回 visibility='group_shared' 且
//                     share_review_status='approved' 的条目
//                     （pending/rejected/not_requested 一律不给 —— 程序级过滤）
export function searchVectorMemory(key, topic, topKOverride) {
  const rows = loadRows().filter((r) => r.key === key && Array.isArray(r.vec));
  if (!rows.length) return Promise.resolve([]);
  const isOwnerPrivate = key.startsWith('private:');
  const visible = isOwnerPrivate
    ? rows
    : rows.filter((r) => r.visibility === 'group_shared' && r.share_review_status === 'approved');
  if (!visible.length) return Promise.resolve([]);
  const topK = Math.max(1, Number(topKOverride) || SEARCH_TOPK);
  return new Promise((resolve, reject) => {
    embed(topic).then((qv) => {
      const scored = visible
        .map((r) => ({ r, score: cosSim(qv, r.vec) }))
        .filter((x) => x.score > SEARCH_THRESHOLD)
        .sort((a, b) => b.score - a.score)
        .slice(0, topK);
      // 单条裁剪 + 总长裁剪（只裁返回内容，不改库）
      let total = 0;
      const out = [];
      for (const { r, score } of scored) {
        const text = String(r.text ?? '').slice(0, SEARCH_MAX_PER_HIT);
        if (total + text.length > SEARCH_MAX_TOTAL) break;
        total += text.length;
        out.push({ text, time: r.time, score: Number(score.toFixed(3)), category: r.category || 'unclassified' });
      }
      resolve(out);
    }).catch(reject);
  });
}

// 会话的全部向量记忆条数（控制台/调试用）。
export function countVectorMemory(key) {
  return loadRows().filter((r) => r.key === key).length;
}

// ── P0-B 管理能力（仅控制台 HTTP 调用，机器人无删除权）──────────────
export const VEC_FILE_PATH = VEC_FILE;

// 列出记忆（可按 key / category / share_review_status 过滤；不返回 vec）
export function listVectorMemory(filter = {}) {
  const rows = loadRows();
  return rows
    .filter((r) => (!filter.key || r.key === filter.key))
    .filter((r) => (!filter.category || r.category === filter.category))
    .filter((r) => (!filter.subjectId || r.subjectId === filter.subjectId))
    .filter((r) => (filter.needs_review === undefined || (r.needs_review ?? true) === filter.needs_review))
    .filter((r) => (!filter.share_review_status || (r.share_review_status || 'not_requested') === filter.share_review_status))
    .map((r) => ({
      key: r.key, text: r.text, time: r.time, category: r.category || 'unclassified',
      subjectId: r.subjectId || 'unknown', subjectName: r.subjectName || 'unknown',
      subjectType: r.subjectType || 'unknown', sourceType: r.sourceType || 'unknown',
      visibility: r.visibility || 'owner_private', needs_review: r.needs_review ?? true,
      share_review_status: r.share_review_status || 'not_requested',
      share_requested_at: r.share_requested_at ?? null,
      share_reviewed_at: r.share_reviewed_at ?? null,
      share_review_note: r.share_review_note ?? null,
    }));
}

// ── 分享审核（任务一）────────────────────────────────────────────────
// 定位辅助：三级匹配（她复述时可能丢前缀/拿的是裁剪片段）
//   1) key + text 全匹配
//   2) 双向包含（她给的片段 ⊂ 库里原文，或反过来）
//   3) 去掉分类前缀后包含
function findMemoryRow(rows, key, text) {
  const t = String(text ?? '').trim();
  if (!t) return -1;
  let idx = rows.findIndex((r) => r.key === key && r.text === t);
  if (idx >= 0) return idx;
  idx = rows.findIndex((r) => r.key === key && (r.text.includes(t) || t.includes(r.text)));
  if (idx >= 0) return idx;
  idx = rows.findIndex((r) => {
    if (r.key !== key) return false;
    const bare = r.text.replace(/^(activeTopic|pendingThought|memberImpression):\s*/, '');
    return bare.includes(t) || t.includes(bare);
  });
  if (idx >= 0) return idx;
  // 4级：归一化（去空白/标点/引号/分类前缀）后比较，抗轻微改写
  const norm = (s) => String(s).replace(/^(activeTopic|pendingThought|memberImpression):\s*/, '')
    .replace(/[\s，。、；：""''「」\(\)（）！？,.!?:;'"-]/g, '')
    .replace(/\d+/g, '');
  const nt = norm(t);
  if (nt.length >= 8) {
    idx = rows.findIndex((r) => r.key === key && (norm(r.text).includes(nt) || nt.includes(norm(r.text))));
    if (idx >= 0) return idx;
  }
  return -1;
}

// 模型提交分享申请：定位已有记忆（三级匹配），标记为 pending。
export function requestShare(key, text) {
  const rows = loadRows();
  const idx = findMemoryRow(rows, key, text);
  if (idx < 0) return null;
  const r = rows[idx];
  const status = r.share_review_status || 'not_requested';
  if (status === 'approved') return { status: 'already_approved', text: r.text };
  if (status === 'pending') return { status: 'already_pending', text: r.text };
  r.share_review_status = 'pending';
  r.share_requested_at = Date.now();
  cache.mtime = 0;
  atomicRewrite(rows);
  return { status: 'pending', text: r.text, share_requested_at: r.share_requested_at };
}

// 管理端批准：status=approved 且 visibility=group_shared。
export function approveShare(key, text, note) {
  const rows = loadRows();
  const idx = findMemoryRow(rows, key, text);
  if (idx < 0) return null;
  const r = rows[idx];
  const status = r.share_review_status || 'not_requested';
  if (status === 'approved') return { status: 'already_approved', text: r.text };
  r.share_review_status = 'approved';
  r.visibility = 'group_shared';
  r.share_reviewed_at = Date.now();
  if (note) r.share_review_note = String(note).slice(0, 200);
  cache.mtime = 0;
  atomicRewrite(rows);
  return { status: 'approved', text: r.text, visibility: r.visibility };
}

// 管理端拒绝：status=rejected，visibility 回退 owner_private。
export function rejectShare(key, text, note) {
  const rows = loadRows();
  const idx = findMemoryRow(rows, key, text);
  if (idx < 0) return null;
  const r = rows[idx];
  const status = r.share_review_status || 'not_requested';
  if (status === 'rejected') return { status: 'already_rejected', text: r.text };
  r.share_review_status = 'rejected';
  r.visibility = 'owner_private';
  r.share_reviewed_at = Date.now();
  if (note) r.share_review_note = String(note).slice(0, 200);
  cache.mtime = 0;
  atomicRewrite(rows);
  return { status: 'rejected', text: r.text, visibility: r.visibility };
}

// 精确删除：key + text 全匹配才算命中（不用模糊匹配）。返回删除的记录。
export function removeVectorMemoryExact(key, text) {
  const rows = loadRows();
  const idx = rows.findIndex((r) => r.key === key && r.text === String(text));
  if (idx < 0) return null;
  const [removed] = rows.splice(idx, 1);
  cache.mtime = 0;
  atomicRewrite(rows);
  return removed;
}

// 按 key + text 精确匹配，最多删除 n 条。返回实际删除数量（0 表示无匹配或 n 非正）。
// 供 remove 端点的 A/B 数量对齐：A 层删除数量 ≤ B 层本次实际删除数量。
// 使用严格读取：文件读取/解析失败抛错（不把异常当"无匹配"）。
export function removeNAllVectorMemory(key, text, n) {
  const rows = loadRowsStrict();
  // n 必须是正整数：非数字 / 负数 / 0 / NaN 一律按 0 处理（不删除）。
  const limit = Number.isInteger(n) && n > 0 ? n : 0;
  if (limit === 0) return 0;
  const targetText = String(text ?? '');
  let removed = 0;
  const kept = [];
  for (const r of rows) {
    if (removed < limit && r.key === key && r.text === targetText) {
      removed++;
      continue;
    }
    kept.push(r);
  }
  if (removed > 0) {
    cache.mtime = 0;
    atomicRewrite(kept);
  }
  return removed;
}

// 删除指定 key 的全部记录。返回实际删除数量。
// 仅供 clear 端点使用；严格匹配 r.key === key，不影响其他 key。
// 使用严格读取：文件读取/解析失败抛错。
export function removeAllVectorMemory(key) {
  const rows = loadRowsStrict();
  const kept = rows.filter((r) => r.key !== key);
  const removed = rows.length - kept.length;
  if (removed > 0) {
    cache.mtime = 0;
    atomicRewrite(kept);
  }
  return removed;
}

// 修正：改文本并重算 embedding（旧向量对不上新文本，必须重算）。
export async function editVectorMemory(key, oldText, newText) {
  const rows = loadRows();
  const idx = rows.findIndex((r) => r.key === key && r.text === String(oldText));
  if (idx < 0) throw new Error('未找到要修正的记忆（key+text 需全匹配）');
  const clean = String(newText ?? '').trim().slice(0, 1000);
  if (!clean) throw new Error('新文本为空');
  const vec = await embed(clean);
  rows[idx].text = clean;
  rows[idx].vec = vec;
  rows[idx].edited = true;
  cache.mtime = 0;
  atomicRewrite(rows);
  return { text: clean };
}

function atomicRewrite(rows) {
  const tmp = VEC_FILE + '.tmp';
  fs.writeFileSync(tmp, rows.map((r) => JSON.stringify(r)).join('\n') + '\n', 'utf8');
  fs.renameSync(tmp, VEC_FILE);
}
````



---

# 附：A 层记忆数据模型（代码依据）

## 存储格式

`state/memory-vectors.jsonl`，一行一条 JSON：

````json
{"key":"<会话key>","text":"<正文>","time":<写入时刻ms>,"vec":[1024 floats],"share_review_status":"not_requested"}
````

| 字段 | 来源 | 说明 |
|---|---|---|
| `key` | 调用方传入 | `private:<QQ>` 或 `group:<群号>`，经 `canonicalV2Key` 规范化（只去前导零） |
| `text` | 调用方传入，**含 category 前缀** | 格式为 `` `${category}: ${content}` ``（由 MCP 层拼接） |
| `time` | `Date.now()` | **写入时刻**，不是事件发生时刻 |
| `vec` | Ollama `bge-m3` | 1024 维；`embed()` 失败则整条不写 |
| `share_review_status` | 固定 `'not_requested'` | 后续由 `requestShare`/`approveShare`/`rejectShare` 改 |
| `visibility` | **写入时不存在** | 只有 `approveShare` 才加 `'group_shared'` |
| `category`/`subjectId`/`subjectName`/`subjectType`/`sourceType`/`needs_review` | 部分记录有 | V1 后期补的字段，不保证全量存在 |

## 关键函数清单（本卷源码中的位置）

| 功能 | 函数 | 行号 |
|---|---|---|
| 宽松读（坏行静默跳过） | `loadRows()` | 44-58 |
| 严格读（坏行抛错） | `loadRowsStrict()` | 68-83 |
| embedding（Ollama HTTP） | `embed(text)` | 86-108 |
| 余弦相似度 | `cosSim(a,b)` | 110-120 |
| 信息量门槛 | `infoOK(text)` | 125-130 |
| 每日备份 | `dailyBackup()` | 131-147 |
| 追加写入 | `appendVectorMemory(key,text,time)` | 149-163 |
| 语义检索 | `searchVectorMemory(key,topic,topK)` | 171-199 |
| 条数统计 | `countVectorMemory(key)` | 202-204 |
| 列表（管理端） | `listVectorMemory(filter)` | 210-228 |
| 三级/四级匹配 | `findMemoryRow(rows,key,text)` | 235-258 |
| 提交分享申请 | `requestShare(key,text)` | 261-274 |
| 批准分享 | `approveShare(key,text,note)` | 277-291 |
| 拒绝分享 | `rejectShare(key,text,note)` | 294-308 |
| 精确删除 | `removeVectorMemoryExact(key,text)` | 311-319 |
| 删 N 条 | `removeNAllVectorMemory(key,text,n)` | 324-344 |
| 删整 key | `removeAllVectorMemory(key)` | 349-358 |
| 编辑（重算向量） | `editVectorMemory(key,old,new)` | 361-374 |
| **原子重写（缺陷点）** | `atomicRewrite(rows)` | **376-380** |

## 检索算法（当前实现）

```
searchVectorMemory(key, topic, topK):
  ① loadRows()                              ← 容错读，坏行静默跳过
  ② filter(r => r.key === key && Array.isArray(r.vec))    ← key 全等硬过滤
  ③ isOwnerPrivate = key.startsWith('private:')
  ④ private → 全部可见；group → 只留 visibility==='group_shared' && status==='approved'
  ⑤ embed(topic) → cosSim(qv, r.vec)
  ⑥ filter(score > 0.3) → sort desc → slice(0, topK)
  ⑦ 单条裁剪 300 字 / 总长裁剪 1200 字
```

**参数默认值**（`config.json` 的 `memoryVector.search`）：`topK=3`、`threshold=0.3`、`maxPerHit=300`、`maxTotal=1200`。
