---
title: 阶段1-计算 KnobsHash 缓存键
level: atomic
parent: 命名能力包与版本化缓存指纹防污染
status: draft
tags:
  - cache-key
  - search
  - hashing
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - ../命名能力包与版本化缓存指纹防污染.md
source_repo: garrytan/gbrain
source_commit:
source_paths:
  - src/core/search/mode.ts
  - src/core/search/query-cache.ts
---

# 阶段1：计算 KnobsHash 缓存键

## 触发条件

`hybridSearch`（或任何写/读 `query_cache` 的路径）已完成 `resolveSearchMode`，即将执行 cache lookup 或 cache write。

## 输入字段

| 字段 | 类型 | 来源 |
|------|------|------|
| `knobs` | `ResolvedSearchKnobs` | resolveSearchMode 输出 |
| `ctx.embeddingColumn` | `string?` | 当前查询使用的向量列名 |
| `ctx.embeddingModel` | `string?` | provider:model 字符串 |
| `ctx.schemaPack` | `string?` | 活跃 schema pack 名 |
| `ctx.schemaPackVersion` | `string?` | pack 版本 |

## 判定规则

1. 初始化字符串数组 `parts`，**第一项固定** `v=${KNOBS_HASH_VERSION}`（当前值 **11**）
2. 按**固定顺序** append 所有影响结果集的 knob 键值。源码实际顺序（mode.ts `knobsHash` 函数体）：
   `v / mode / cache / sim / ttl / iw / tb / exp / lim / rr / rrm / rri / rro / rrt / fr / cmbt / cmbi / iqt / iqi / um / umo / lli / col / prov / gs / pack / pver / cr / crd / tib / ac / acj / rel / reld`
   - 注意 parts 里的**键名缩写**：embedding 列是 `col=`（非 `embCol=`）、provider:model 是 `prov=`（非 `embModel=`）；schema pack 拆成 `pack=` 和 `pver=` 两个独立 part，**不是** `pack@ver` 拼接
3. `ctx.embeddingColumn` 缺省 → 字面量 `'embedding'`
4. `ctx.embeddingModel` 缺省 → `'default'`
5. `ctx.schemaPack` 缺省 → `'none'`；`schemaPackVersion` 缺省 → `'none'`（注意：不是 `'0'`）
6. `hash = SHA256(parts.join('|'))` 的 hex digest，再 **`.slice(0, 16)`** 截断为 16 位
7. **纪律**：新增 knob 到 parts → **必须** `KNOBS_HASH_VERSION += 1`，否则 persisted cache 跨语义 serve

## 状态读写位置

- **读**：`query_cache` 表，`WHERE knobs_hash = $hash`（与 source_id、embedding 相似度 AND）
- **写**：INSERT/UPSERT 行写入同一 `knobs_hash` 列
- **无**进程内缓存；hash 每 query 重算

## 正常路径

1. `resolved = resolveSearchMode(...)`
2. `hash = knobsHash(resolved, { embeddingColumn, embeddingModel, schemaPack, schemaPackVersion })`
3. cache.get(scope, queryVec, hash, threshold, ttl)
4. miss → run pipeline → cache.set(..., hash, results)

## 分支路径

| 条件 | 行为 |
|------|------|
| cache hit（sim < threshold 且 hash 相等） | 跳过 pipeline，返回 stored results |
| hash 不匹配 | miss，即使 query 文本相同 |
| 历史行 `knobs_hash IS NULL` | lookup 排除（pre-version 行不可 serve） |
| `ctx` 未传 embedding 上下文 | 使用 default 字面量，与 legacy 路径 stable |

## 失败处理

- SHA256 计算不会失败
- cache DB 错误：lookup miss（catch 吞掉，走全 pipeline）
- cache write 错误：catch 吞掉，注释明确「cache write must never break the search hot path」

## 幂等性 / 一致性约束

- 相同 resolved knobs + 相同 ctx → 相同 hash（确定性）
- hash **全局** per brain，非 per-provider 分区；version bump 触发全库一次性 miss spike（可接受 tradeoff）
- 后续阶段不得用不含 hash 的 cache key 读结果

## 代码骨架

```typescript
export const KNOBS_HASH_VERSION = 11;

export function knobsHash(
  knobs: ResolvedSearchKnobs,
  ctx?: KnobsHashContext,
): string {
  // 固定顺序、append-only。新增字段必须 bump KNOBS_HASH_VERSION。
  const parts = [
    `v=${KNOBS_HASH_VERSION}`,
    `mode=${knobs.resolved_mode}`,
    `cache=${knobs.cache_enabled ? 1 : 0}`,
    `sim=${knobs.cache_similarity_threshold.toFixed(4)}`,
    `ttl=${knobs.cache_ttl_seconds}`,
    `iw=${knobs.intentWeighting ? 1 : 0}`,
    `tb=${knobs.tokenBudget ?? 'none'}`,
    `exp=${knobs.expansion ? 1 : 0}`,
    `lim=${knobs.searchLimit}`,
    `rr=${knobs.reranker_enabled ? 1 : 0}`,
    // rrm / rri / rro / rrt —— reranker 配置
    `fr=${knobs.floor_ratio === undefined ? 'none' : knobs.floor_ratio.toFixed(4)}`,
    // cmbt / cmbi / iqt / iqi / um / umo / lli —— cross-modal
    `col=${ctx?.embeddingColumn ?? 'embedding'}`,      // 键名 col，非 embCol
    `prov=${ctx?.embeddingModel ?? 'default'}`,         // 键名 prov，非 embModel
    `gs=${knobs.graph_signals ? 1 : 0}`,
    `pack=${ctx?.schemaPack ?? 'none'}`,                // pack 与 pver 独立，非 pack@ver
    `pver=${ctx?.schemaPackVersion ?? 'none'}`,
    // cr / crd / tib / ac / acj —— contextual / title-boost / autocut
    `rel=${knobs.relationalRetrieval ? 1 : 0}`,
    `reld=${knobs.relational_retrieval_depth ?? 2}`,
  ];
  // 关键：返回 16 位 hex 截断，非完整 64 位
  return createHash('sha256').update(parts.join('|')).digest('hex').slice(0, 16);
}
```

## 最小验证清单

- conservative vs tokenmax 同 query text → hash 不同
- 同 knobs 连续两次 → hash 相同
- version 10 行在 version 11 代码下 lookup miss
- 换 schemaPack 名 → hash 变
- cache INSERT 后 SELECT 必须带相同 hash 才 hit

---

## 附录：来源证据（仅供溯源核实，阅读正文无需依赖此节）

| 项 | 位置 |
|----|------|
| KNOBS_HASH_VERSION | `src/core/search/mode.ts:750` |
| KnobsHashContext 接口 | `src/core/search/mode.ts:763-779` |
| knobsHash 函数 | `src/core/search/mode.ts:781+` |
| cache SQL knobs_hash 谓词 | `src/core/search/query-cache.ts:163` |
| cache write swallow | `src/core/search/query-cache.ts:263-265` |
