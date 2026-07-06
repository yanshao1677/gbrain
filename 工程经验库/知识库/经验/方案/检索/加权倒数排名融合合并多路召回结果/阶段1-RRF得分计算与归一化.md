---
title: 阶段1-RRF得分计算与归一化
level: atomic
parent: 加权倒数排名融合合并多路召回结果
status: draft
tags:
  - rrf
  - scoring
  - normalization
  - boost
created_at: 2026-07-06
updated_at: 2026-07-06
confidence: high
related:
  - ../加权倒数排名融合合并多路召回结果.md
  - 阶段2-意图感知的每路有效k调整.md
source_repo: garrytan/gbrain
source_commit:
source_paths:
  - src/core/search/hybrid.ts
---

# 阶段1：RRF 得分计算与归一化

## 触发条件

`rrfFusionWeighted`（或等权版 `rrfFusion`）被调用，入参为多路 `{ list: SearchResult[]; k: number }`，至少一路非空；即将把多路排名合成单一排序。

## 输入字段

| 字段 | 类型 | 来源 | 可选 |
|------|------|------|------|
| `lists[i].list` | `SearchResult[]` | 各路召回（vector/keyword/relational/image） | 否 |
| `lists[i].k` | `number` | 阶段 2 计算的 `effectiveRrfK(k_base, weight)` | 否 |
| `applyBoost` | `boolean` | 调用方：`detail !== 'high'` 时 true（temporal/event 跳过 boost） | 是，默认 true |
| `SearchResult.source_id` | `string?` | 召回阶段填充 | 是，fallback `'default'` |
| `SearchResult.slug` | `string` | 召回阶段填充 | 否 |
| `SearchResult.chunk_id` | `string? \| null` | 召回阶段填充 | 是 |
| `SearchResult.chunk_text` | `string` | 召回阶段填充 | 否（chunk_id 为 null 时用于 fallback key） |
| `SearchResult.chunk_source` | `string?` | 召回阶段填充 | 是（`'compiled_truth'` 触发 boost） |

## 判定规则

1. **融合去重键**（`rrfKey`）：
   ```
   source = r.source_id ?? 'default'
   key = `${source}:${r.slug}:${r.chunk_id ?? r.chunk_text.slice(0, 50)}`
   ```
   - `chunk_id` 非 null → 用 `chunk_id`
   - `chunk_id` 为 null（合成 chunkless 行）→ fallback 到 `chunk_text` 前 50 字符
   - **为什么三段式**：同 slug 不同 source 不误合并（federated 读场景）、同 slug 不同 chunk 不误合并（细粒度保留）。Pre-fix 用 `slug:chunk_id` 会让跨 source 同 slug 行合并，是已知 bug 类。

2. **单路单文档投票**（rank 从 **0** 起，不是 1）：
   ```
   rrfScore = 1 / (k + rank)      // rank=0 → 1/k；rank=k → 1/(2k)
   ```

3. **跨路累加**：同一 dedupKey 的 rrfScore 求和。第一次见到的 result 对象入 Map，后续只累加 score，**不替换 result**。

4. **归一化**（全部累加完后）：
   ```
   maxScore = max(所有 entry.score)
   if maxScore > 0:
     entry.score = entry.score / maxScore   // 归一化到 (0, 1]
   // maxScore === 0 时跳过归一化（理论不可能，所有 rrfScore 都 > 0）
   ```

5. **source-type boost**（归一化**之后**，boost 前归一化会被抹掉）：
   ```
   if applyBoost AND entry.result.chunk_source === 'compiled_truth':
     entry.score *= 2.0       // COMPILED_TRUTH_BOOST
   else:
     entry.score *= 1.0       // no-op
   ```

6. **排序**：按 `entry.score` 降序。**score 相同时的 tiebreaker 未定义**——依赖 JS `Array.prototype.sort` 的稳定性（ES2019+ 保证稳定，故同 score 的相对顺序 = 入 Map 顺序 = 路传入顺序 × 路内 rank 顺序）。

**魔法数字追问**：
- `k_base = 60`：源码 `hybrid.ts:47` 注释无解释。RRF 论文常用 60，业界共识值。改小 → top 排名陡峭；改大 → 平坦。**证据不足：60 的具体来源未在代码注释中确认**，推测是论文默认值。
- `COMPILED_TRUTH_BOOST = 2.0`：`hybrid.ts:48`。`compiled_truth` 是 brain 的"已合成真相"chunk（区别于原始 markdown chunk），作者认为其质量值得 2x 加权。**具体 2.0 而非 1.5 或 3.0 的依据未在注释中说明**。

## 状态读写位置

- **读**：无持久化；纯函数从入参 `lists` 读取
- **写**：无持久化；产出新 `SearchResult[]`（`{ ...result, score }` 浅拷贝，不 mutate 原结果）
- **进程内状态**：`Map<string, {result, score}>`，函数局部，调用结束即销毁
- **跨进程语义**：无需变化——纯函数，无副作用，无共享状态

## 正常路径

1. 初始化 `scores = new Map()`
2. 遍历每路 `{ list, k }`：
   - 遍历 `list` 中 `rank` 从 0 到 `list.length - 1`：
     - 算 `key = rrfKey(r)`
     - 算 `rrfScore = 1 / (k + rank)`
     - Map 有 key → `existing.score += rrfScore`，**不替换 result**
     - Map 无 key → `scores.set(key, { result: r, score: rrfScore })`
3. `entries = Array.from(scores.values())`
4. `entries.length === 0` → 返回 `[]`
5. `maxScore = max(entries.map(e => e.score))`
6. `maxScore > 0` 时遍历 entries：`e.score /= maxScore`；若 `applyBoost && chunk_source === 'compiled_truth'` → `e.score *= 2.0`
7. `entries.sort((a,b) => b.score - a.score)`
8. 返回 `entries.map(({result, score}) => ({ ...result, score }))`

**先后顺序约束**：
- 归一化必须在所有路累加完成后（不能边累加边归一化）
- boost 必须在归一化之后（否则 boost 被归一化抹掉）
- 排序必须在 boost 之后

## 分支路径

| 条件 | 行为 |
|------|------|
| 某路 `list` 为空数组 | 该路循环 0 次，不影响其他路；不报错 |
| 所有路都为空 | `entries.length === 0` → 返回 `[]`，不抛错 |
| `maxScore === 0`（理论不可能） | 跳过归一化与 boost，直接排序输出（全 0 score，顺序 = 入 Map 顺序） |
| 同 dedupKey 在多路出现 | 第一次入 Map 的 result 被保留；后续只累加 score，不替换 result 对象 |
| `chunk_id` 为 null | fallback 到 `chunk_text.slice(0, 50)` 作为 key 的一部分 |
| `applyBoost === false` | 跳过 boost，仅归一化 |
| `chunk_source !== 'compiled_truth'` | boost 系数 = 1.0（no-op） |

## 失败处理

- **本阶段不会抛异常**：纯计算，无 IO、无外部调用
- **数值边界**：`1/(k+rank)` 当 k 极小且 rank 极大时趋近 0，但不会溢出（k 最小为 1，rank 实际上限 = 召回 limit，典型 ≤100）
- **tiebreaker 未定义**：同 score 的相对顺序由 JS sort 稳定性保证（ES2019+）。若运行时是旧版 JS 引擎或非稳定 sort，同 score 顺序不确定——**证据不足：gbrain 运行时为 Bun，其 sort 稳定性是否依赖 V8/JSC 的 ES2019 实现，需运行时确认**。但生产中同 score 罕见（连续浮点），实际影响可忽略。

## 幂等性 / 一致性约束

- **确定性**：相同 `lists`（相同路顺序、相同路内顺序）+ 相同 `k` → 产出完全相同的输出（含 score 与顺序）
- **路传入顺序敏感**：路顺序影响 (a) 同 dedupKey 的 result 保留、(b) 同 score 的 tiebreak。调用方必须保证路顺序稳定
- **不 mutate 输入**：返回的是 `{ ...result, score }` 浅拷贝；原 `SearchResult` 对象的 `score` 字段不被修改
- **后续阶段约束**：
  - 归一化后的 score ∈ (0, 1]（boost 后可能 >1，如 compiled_truth 归一化 1.0 × 2.0 = 2.0）
  - 后续 `cosineReScore` 会用 `0.7*rrf + 0.3*cosine` 混合，要求 rrf 已归一化——本阶段保证
  - 后续 `dedup` 阶段不可重新打开已融合的 chunk 粒度（dedup 在 page 粒度，本阶段在 chunk 粒度）

## 代码骨架

```typescript
const RRF_K_BASE = 60;
const COMPILED_TRUTH_BOOST = 2.0;

interface SearchResult {
  slug: string;
  chunk_id: string | null;
  chunk_text: string;
  source_id?: string;
  chunk_source?: string;
  score: number;
}

function rrfKey(r: SearchResult): string {
  const source = r.source_id ?? 'default';
  return `${source}:${r.slug}:${r.chunk_id ?? r.chunk_text.slice(0, 50)}`;
}

export function rrfFusionWeighted(
  lists: Array<{ list: SearchResult[]; k: number }>,
  applyBoost = true,
): SearchResult[] {
  const scores = new Map<string, { result: SearchResult; score: number }>();

  for (const { list, k } of lists) {
    for (let rank = 0; rank < list.length; rank++) {
      const r = list[rank];
      const key = rrfKey(r);
      const rrfScore = 1 / (k + rank);           // rank 从 0 起
      const existing = scores.get(key);
      if (existing) {
        existing.score += rrfScore;               // 累加，不替换 result
      } else {
        scores.set(key, { result: r, score: rrfScore });
      }
    }
  }

  const entries = Array.from(scores.values());
  if (entries.length === 0) return [];

  const maxScore = Math.max(...entries.map(e => e.score));
  if (maxScore > 0) {
    for (const e of entries) {
      e.score = e.score / maxScore;               // 归一化到 (0, 1]
      const boost = applyBoost && e.result.chunk_source === 'compiled_truth'
        ? COMPILED_TRUTH_BOOST : 1.0;
      e.score *= boost;                           // boost 在归一化后
    }
  }

  return entries
    .sort((a, b) => b.score - a.score)
    .map(({ result, score }) => ({ ...result, score }));
}
```

## 最小验证清单

- 两路都含文档 X（一路 rank=0，一路 rank=2，k=60）→ X 的 score = 1/60 + 1/62 ≈ 0.0328
- 单路单文档 rank=0 k=60 → score = 1/60 ≈ 0.0167；归一化后 = 1.0（自己是 max）
- 同 dedupKey 在两路出现 → result 对象是第一路传入的，score 是两路之和
- `compiled_truth` 文档归一化后 score=1.0 → boost 后 = 2.0
- `applyBoost=false` 时 `compiled_truth` 不 boost，score 保持归一化值
- 全空入参 → 返回 `[]`，不抛错
- 路 A rank=0 文档与路 B rank=0 文档 dedupKey 相同 → score = 2/k（共识奖励）
- `chunk_id=null` + `chunk_text="abc..."`（≥50 字符）→ key 含 `chunk_text.slice(0,50)`
- 输入 `SearchResult` 对象的 `score` 字段在调用后未被修改（不 mutate）

---

## 附录：来源证据（仅供溯源核实，阅读正文无需依赖此节）

| 项 | 位置 |
|----|------|
| rrfFusionWeighted 实现 | `src/core/search/hybrid.ts:1860-1896` |
| rrfFusion（等权版，逻辑相同） | `src/core/search/hybrid.ts:1903-1945` |
| rrfKey 三段式 + 注释 | `src/core/search/hybrid.ts:1818-1831` |
| RRF_K = 60 | `src/core/search/hybrid.ts:47` |
| COMPILED_TRUTH_BOOST = 2.0 | `src/core/search/hybrid.ts:48` |
| rank 从 0 起（循环 `rank = 0; rank < list.length`） | `src/core/search/hybrid.ts:1867, 1907` |
| 归一化（÷ maxScore） | `src/core/search/hybrid.ts:1884-1891, 1925-1939` |
| boost 在归一化后 | `src/core/search/hybrid.ts:1888-1890, 1932-1933` |
| applyBoost 来自 `detail !== 'high'` | `src/core/search/hybrid.ts:1363` |
