# gbrain hybrid search 模块深读笔记（模板场景二）

> 模块：src/core/search/ 整个目录，聚焦 relational recall / rerank fail-open / autocut / cosineReScore 四个子机制
> 仓库：gbrain v0.42.56.0
> 性质：中间产物，不入 `知识库/经验/`，与 `gbrain-代码地图.md` 同级

---

## 1. 模块背景

hybrid search 是 gbrain 的核心检索层，负责把"用户的自然语言 query"变成"带引用的合成答案所需的 chunk 列表"。它在 hybrid.ts:809 的 `hybridSearch` 里组织向量召回、BM25 关键词召回、关系图召回、可选 LLM 扩展召回四路结果，用 RRF 融合后再做一系列后处理（cosine 重排、rerank、autocut、token budget 截断），最终输出带 evidence/safety 戳的 SearchResult[]。

本次深读聚焦四个尚未提炼的子机制：relational recall（关系召回第四臂）、rerank fail-open（重排失败降级）、autocut（结果自动截断）、cosineReScore（融合后余弦重排）。RRF 融合本身已经做过原子卡（`阶段1-RRF得分计算与归一化.md`），不在本次范围。

## 2. 核心职责

负责：

- 把一条 query 串变成按相关性排序的 SearchResult 列表
- 协调向量 / 关键词 / 关系 / LLM 扩展多路召回
- 在融合前后做 boost、去重、重排、截断、token 预算控制
- 给每条结果打 evidence/create_safety 戳，让下游 agent 不只看分数
- 缓存查询结果，用 knobs_hash 防止 cross-mode 串读

不负责：

- 生成最终合成答案（这是 `query` op 上层 think/gather 做的）
- embedding 向量化（委托给 engine 的 embed 接口）
- 实际 SQL 执行（委托给 engine.searchVector / searchKeyword / relationalFanout / getEmbeddingsByChunkIds）
- 配置 mode 的持久化（mode.ts 只负责解析，写 config 是 config.ts 的事）

## 3. 核心对象

### 入口函数

| 对象 | 文件:行号 | 作用 |
|---|---|---|
| `hybridSearch` | `src/core/search/hybrid.ts:809` | bare 主入口，签名 `(engine, query, opts?): Promise<SearchResult[]>`；eval 路径和 think/gather 直调它 |
| `hybridSearchCached` | `src/core/search/hybrid.ts:1569` | op 层实际调的入口（`operations.ts:1461, 1635`），cache miss 时转调 bare `hybridSearch`（`:1759`） |
| `HybridSearchOpts` | `src/core/search/hybrid.ts:703-742` | extends `SearchOpts`，加 `expansion / onRelationalMeta / mode / expandFn / rrfK / dedupOpts / onMeta / _queryEmbedDeadline` |

### 四个子机制

| 对象 | 文件:行号 | 签名 |
|---|---|---|
| `buildRelationalArm` | `src/core/search/relational-recall.ts:153` | `(engine, query, opts?: RelationalArmOpts): Promise<SearchResult[]>`，`RelationalArmOpts` 见 `:33-40` |
| `parseRelationalQuery` | `src/core/search/relational-intent.ts:222` | `(query, vocab?): RelationalQuery \| null`，纯函数 regex 解析器 |
| `applyReranker` | `src/core/search/rerank.ts:58` | `(query, results, opts: RerankerOpts): Promise<SearchResult[]>`，`RerankerOpts` 见 `:25-40` |
| `applyAutocut` | `src/core/search/autocut.ts:134` | `<T>(results, scoreOf, cfg: AutocutConfig, preserve?): { kept: T[]; decision: AutocutDecision }`，纯函数 |
| `cosineReScore` | `src/core/search/hybrid.ts:1951` | `(engine, results, queryEmbedding, column='embedding'): Promise<SearchResult[]>`，模块内私有（非 export） |

### 底层引擎契约

| 对象 | 文件:行号 | 作用 |
|---|---|---|
| `engine.relationalFanout` | `src/core/engine.ts:1242` | `(seeds: string[], opts?): Promise<RelationalFanoutRow[]>`，关系图 BFS 遍历；`RelationalFanoutOpts` 在 `src/core/types.ts:1235-1251` |
| `engine.getEmbeddingsByChunkIds` | `src/core/engine.ts:943` | cosineReScore 的底层调用，按 chunk_id 批量取 embedding |
| `gateway.rerank` | `src/core/ai/gateway.ts`（未读全） | rerank.ts:78-84 调用，cross-encoder 重排 |
| `logRerankFailure` | `src/core/rerank-audit.ts` | rerank 失败审计写入，`RerankFailureEvent` 见 `:39-58` |

### 配置与缓存

| 对象 | 文件:行号 | 作用 |
|---|---|---|
| `resolveSearchMode` | `src/core/search/mode.ts` | 解析 per-call → config → MODE_BUNDLES 的链路 |
| `knobsHash` | `src/core/search/mode.ts` | 33 字段固定顺序拼接 + SHA256 截 16 位 hex，已有原子卡 |
| `KNOBS_HASH_VERSION = 11` | `src/core/search/mode.ts:750` | 每加 knob 必须 bump |
| `SemanticQueryCache` | `src/core/search/query-cache.ts:38` | `DEFAULT_SIMILARITY_THRESHOLD = 0.92`，`DEFAULT_TTL_SECONDS = 3600` |

## 4. 内部流程

### 主路径调用链（bare `hybridSearch`，vectorLists 非空时）

```mermaid
flowchart TB
  Entry[hybridSearch hybrid.ts:809] --> RelPre[relational recall 预构建 :1027]
  RelPre --> Branch{vector 是否成功}
  Branch --|失败/无 embed| NoEmbed[no-embed 路径 :1050<br/>relational+keyword 走 rrfFusionWeighted]
  Branch --|embed 失败| EmbedFail[embed-fail 路径 :1284<br/>同 no-embed]
  Branch --|成功| Main[主路径 :1276]
  Main --> RelArm[relational 第四臂注入 allLists :1359]
  RelArm --> RRF[rrfFusionWeighted :1363]
  RRF --> Cosine[cosineReScore :1370]
  Cosine --> Post[runPostFusionStages :1377<br/>backlink/salience/recency/title/graph/alias]
  Post --> Exact[applyExactMatchBoost :1381]
  Exact --> TwoPass[two-pass expansion 可选 :1400]
  TwoPass --> Dedup[dedupResults :1437]
  Dedup --> Rerank[applyReranker fail-open :1461]
  Rerank --> AliasHop[applyAliasHop :1468]
  AliasHop --> Evidence[stampEvidence :1477]
  Evidence --> Adaptive[applyAdaptiveReturn 可选 :1491]
  Adaptive --> Autocut[applyAutocut :1509]
  Autocut --> Slice[slice + enforceTokenBudget + stampContentFlags :1523]
  Slice --> Done[返回 SearchResult[]]
  NoEmbed --> Done
  EmbedFail --> Done
```

### 四个子机制的调用顺序与位置

按行号顺序：

1. **relational recall** 在所有路径之前预构建（`hybrid.ts:1027-1036`），主路径在 `:1359-1361` 作为第四臂 push 进 `allLists`；no-embed 路径 `:1050-1056`、embed-fail 路径 `:1284-1290` 也注入
2. RRF 融合 `rrfFusionWeighted(allLists, ...)`（`:1363`）
3. **cosineReScore**（`:1369-1371`）—— 紧跟 RRF 之后、post-fusion 之前
4. `runPostFusionStages`（`:1377-1378`）—— backlink/salience/recency/title/graph-signals/alias-resolved
5. `applyExactMatchBoost`（`:1381-1383`）
6. two-pass expansion（`:1400-1429`，可选）
7. `dedupResults`（`:1437`）
8. **rerank fail-open** `applyReranker`（`:1461-1463`）
9. `applyAliasHop`（`:1468-1471`）
10. `stampEvidence`（`:1477`）
11. adaptive return `applyAdaptiveReturn`（`:1491-1495`，可选）
12. **autocut** `applyAutocut`（`:1509-1518`）
13. slice + `enforceTokenBudget` + `stampContentFlags`（`:1523-1529`）

注意：relational recall 与 cosineReScore 之间隔了 RRF；relational 是"融合前注入候选"，cosineReScore 是"融合后重排分数"。两者不直接交互。

### cached wrapper 的额外层

`hybridSearchCached`（`:1569`）在 cache miss 时转调 bare `hybridSearch`（`:1759`），并叠加一层 cache lookup + 同一份 token budget。eval 路径（`eval-replay.ts:257`、`eval-longmemeval.ts:668`、`eval.ts:249`、`correctness-gate.ts:135`）和 `think/gather.ts:110`、`brainstorm/orchestrator.ts:578` 直接调 bare `hybridSearch` —— 这是 [CDX-5+6] 的关键决断：mode 解析必须在 bare 入口生效，eval 才能测到与生产一致的 mode 行为。

## 5. 对外接口

### 给 op 层暴露

- `hybridSearchCached` —— `query`/`search` op（`operations.ts:1461, 1635`）的实际入口
- `resolveSearchMode` —— op 层在调 hybridSearch 前先用它解析 mode

### 给 eval 路径暴露

- `hybridSearch`（bare）—— `eval-replay.ts`、`eval-longmemeeval.ts`、`eval.ts`、`correctness-gate.ts` 直接调
- `think/gather.ts:110`、`brainstorm/orchestrator.ts:578` 也直调 bare

### 给跨模态暴露

- `by-image.ts` 的 image-as-query 检索走独立的 embedding_image 列路径，但仍调 bare `hybridSearch` 做融合

### 不对外暴露

- `cosineReScore` 是模块内私有（`async function` 而非 `export async function`）
- `applyAutocut`、`applyReranker`、`buildRelationalArm` 是 export 的，但实际只被 hybrid.ts 内部调用（grep 全仓确认）

## 6. 扩展点

新增一个"融合后处理阶段"要改的地方：

1. 在 `hybrid.ts` 主路径的步骤 4-13 之间插入新阶段，注意它的位置约束（rerank 之前 vs 之后语义不同）
2. 如果新阶段需要 mode 旋钮，在 `mode.ts` 的三个 MODE_BUNDLES（conservative/balanced/tokenmax）各加字段，并 bump `KNOBS_HASH_VERSION`（mode.ts:750）
3. 如果新阶段有 fail-open 行为，参考 rerank.ts 的双层 try/catch 模式（外层捕获业务错，内层捕获审计写错）
4. 如果新阶段改变结果集大小，要更新 autocut 的 `preserve` 谓词（hybrid.ts:1514-1518 现在保护 alias_hit）

新增一个召回臂要改的地方：

1. 在 `hybrid.ts:1027` 附近的预构建区加新臂的构建
2. 在三条路径（主/no-embed/embed-fail）的 `allLists.push` 处都加注入（参考 relational recall 的三处注入）
3. 在 `intent-weights.ts` 加新臂的 RRF 权重映射
4. 如果新臂有 SQL，在 engine.ts 加方法签名，postgres-engine.ts 和 pglite-engine.ts 各实现一份（parity）

新增一个 search mode 旋钮要改的地方：

1. `mode.ts` 的 `SearchOpts` 接口加字段
2. `mode.ts` 三个 MODE_BUNDLES 各加默认值
3. `knobsHash` 的 parts 数组加字段（mode.ts 内，33 字段那个数组）
4. bump `KNOBS_HASH_VERSION`
5. `test/search-mode.test.ts` 加 drift guard 用例

## 7. 错误处理

### A. relational recall — fail-open（空臂 + 审计行）

`relational-recall.ts:226-231`：

```ts
} catch (err) {
  const msg = err instanceof Error ? err.message : String(err);
  failureWriter.log({ error_summary: truncate(msg), query_kind: parsed.kind });
  meta.errored = true;
  return finish([]);   // 返回空列表，纯 no-op
}
```

`failureWriter` 见 `:57-61`（featureName `'relational-recall-failures'`）。还有几处更细的 fail-open：

- `:92` `resolveSeedScoped` 里 `if (!r || r.source === 'fallback_slugify') continue` —— 置信度闸门（D3 tier-1），fallback_slugify 解析的 seed 直接丢弃
- `:596-599` `applyAliasResolvedBoost` 内部 `try/catch` → `return`（pre-v104 brain 无 slug_aliases 表时 no-op）
- `:647-651` `applyAliasHop` 内部 `resolveAliases` 失败 → `return results`（pre-v110 表缺失或瞬态错误都 fail-open）
- `:674-678` alias inject 时 `getPage` 失败 → `continue`（跳过该 ref）

### B. rerank — fail-open（返回 RRF 原序 + 审计行）

`rerank.ts:85-101`：

```ts
} catch (err) {
  const reason: RerankFailureReason =
    err instanceof RerankError ? err.reason : 'unknown';
  const errorSummary = err instanceof Error ? err.message : String(err);
  try {
    logRerankFailure({ ... });
  } catch {
    // Audit logging must never break search.
  }
  return results;   // 返回输入不变（RRF 顺序）
}
```

覆盖所有错误类（auth/network/timeout/rate-limit/payload-too-large/unknown），见 `:11` 注释。还有两处防御性 fail-open：

- `:104` `if (!Array.isArray(reranked) || reranked.length === 0) return results` —— provider 返回畸形 shape 时透传
- `:97-99` 内层 try/catch 包住 `logRerankFailure`，审计写失败也绝不 break search

### C. autocut — fail-open（no-op 透传）

`autocut.ts` 是纯函数，无 try/catch；"失败"语义是"无悬崖可切"时返回 `noOp(results)`（`:101-113`）。触发条件：

- `:148` `if (!cfg.enabled || results.length < 2) return noOp(results)`
- `:158` `if (scores.length < 2) return noOp(results)` —— <2 条带有限 rerank_score
- `:161` `if (!Number.isFinite(top) || top <= 0) return noOp(results)` —— 分数标尺不可用
- `:180-193` 最大 gap < jumpRatio → 不切，返回原 list + gapRatio 上报
- `:208` `if (kept.length === 0) return noOp(results)` —— 退化阈值兜底

注意调用方 `hybrid.ts:1506-1507` 注释明确："applyAutocut additionally no-ops when <2 items carry a finite rerank_score (covers the fail-open reranker path, where applyReranker returns RRF order with no scores)" —— rerank fail-open 后 rerank_score 全空，autocut 自动级联 no-op。

### D. cosineReScore — fail-open（不重排直接返回）

`hybrid.ts:1970-1973`：

```ts
} catch {
  // DB error is non-fatal, return results without re-scoring
  return results;
}
```

还有两处早返回：

- `:1961` `if (chunkIds.length === 0) return results` —— 无可重排的 chunk_id
- `:1975` `if (embeddingMap.size === 0) return results` —— 拿不到任何 embedding

注意 `cosineReScore` 内部不写审计行；失败完全静默。这是与 rerank/graph-signals/relational-recall 的一个差异点（后三者都有 audit writer）。

### 四机制 fail-open 姿态对比

| 机制 | 失败时返回 | 写审计行 | 双层 try/catch |
|---|---|---|---|
| relational recall | 空臂 `[]` | 是（`relational-recall-failures`） | 否（单层） |
| rerank | 输入不变（RRF 顺序） | 是（`logRerankFailure`） | 是（外层业务 + 内层审计） |
| autocut | noOp（原 list） | 否（纯函数无 IO） | 不适用 |
| cosineReScore | 输入不变 | **否**（静默） | 否（单层） |

## 8. 设计优点

每条都有代码证据，不用形容词：

1. **relational recall 在三条 fallback 路径都注入，而非只在主路径**（hybrid.ts:1027/1050/1284/1359）。代码证据：vector 不可用时 relational 答案价值最大，所以 no-embed 和 embed-fail 两条 fallback 也注入。反直觉的"fallback 路径也要带特性"决断。

2. **rerank 的双层 try/catch 把审计写失败也兜住**（rerank.ts:85-101）。代码证据：外层 catch 捕获 rerank 业务错，内层 try/catch 包 `logRerankFailure`，注释 `// Audit logging must never break search.`。这保证 flaky upstream 或 audit 文件系统满盘都不会让 search 挂。

3. **autocut 自动级联 no-op 覆盖 rerank fail-open 路径**（hybrid.ts:1506-1507 注释 + autocut.ts:158 `scores.length < 2`）。代码证据：rerank fail-open 后 rerank_score 全空，autocut 的 `<2 条带分数` 条件自动触发 noOp，不需要显式 `if (rerankFailed)` 分支。

4. **autocut 的 `preserve` 谓词保护 alias-hop 注入的页**（hybrid.ts:1514-1518 传入 `(x) => x.alias_hit === true`）。代码证据：alias-hop 在 rerank 之后注入（`:1468`），所以没有 rerank_score；不加 preserve 它会被 autocut 按"无分数"误删。autocut 给 alias-hop 开后门，而不是让 alias-hop 伪造一个 rerank_score —— 这是两个独立特性交互时谁让步的清晰决断。

5. **mode 解析在 bare `hybridSearch` 生效而非只在 cached wrapper**（hybrid.ts:809 主入口内调 `resolveSearchMode`）。代码证据：eval 路径（eval-replay.ts:257 等）直调 bare `hybridSearch`，如果 mode 只在 cached wrapper 生效，eval 测不到生产 mode 行为。这是 [CDX-5+6] 的硬约束。

6. **relational recall 的 seed 置信度闸门**（relational-recall.ts:92）。代码证据：`fallback_slugify` 解析的 seed 直接 `continue` 丢弃，宁可不出关系候选也不从"编造的 slug"出发遍历。这是"精度优先于召回"的典型决断。

## 9. 设计代价

1. **`hybrid.ts` 单文件 2006 行，主路径调用链长 13 步**。新人要追一条 query 的完整流程需要在 7 个函数之间跳读，认知负担高。代价是换来 mode 旋钮的集中解析和 cache 的统一接入。

2. **四个 fail-open 机制姿态不一致**。relational/rerank 写审计，cosineReScore 静默；rerank 双层 try/catch，其他单层。这种不一致让"加新 fail-open 阶段时该照哪个模板"变成需要判断的事，容易写错。

3. **relational recall 三路径注入导致代码重复**。hybrid.ts:1027/1050/1284/1359 四处都调 `buildRelationalArm`，三处 push 进 allLists。如果未来加第五条路径（比如 image-only），要记得也注入 relational，否则 image-only 查询会丢关系答案。

4. **autocut 依赖 rerank_score 存在，但 rerank 是可选的**。当 reranker 关闭或 fail-open 时，autocut 自动 no-op，结果集大小只靠 token budget 截断。这是"两个特性耦合但通过隐式 signal 解耦"的代价 —— 解耦干净，但需要读注释才知道为什么 autocut 在没 rerank 时也 no-op。

5. **mode.ts 的 knobs_hash 维护成本**。每加一个 knob 必须同步改 parts 数组 + bump KNOBS_HASH_VERSION，漏一处就是 cross-mode 串读 bug。已有原子卡和 drift guard 测试覆盖，但仍是一条需要纪律的扩展点。

## 10. 可提炼候选

| # | 候选 | 层级 | 衔接 |
|---|---|---|---|
| 1 | relational recall 的 seed 置信度闸门（`fallback_slugify` 直接丢弃） | 方案层 → 原子层 | 新方案《关系召回的 seed 解析与置信度闸门》的阶段 1 |
| 2 | relational recall 的解析顺序 connects → intro → who_at → who_rel（first-match wins） | 原子层 | 候选 1 的阶段 2 |
| 3 | autocut 为什么读 rerank_score 而非 RRF/cosine 分数（PrecisionMemBench 实测硬证据） | 原子层 | 补全总目录"下一步计划"的 autocut 项 |
| 4 | autocut 的 jumpRatio=0.2 不是魔法数字，是 eval 校准起点 + per-mode 旋钮 | 原子层 | 候选 3 的姊妹卡 |
| 5 | reranker fail-open 覆盖所有错误类 + 审计写失败也绝不 break search | 方案层 → 原子层 | 新方案《检索后处理阶段的 fail-open 姿态》的阶段 1 |
| 6 | cosineReScore 是 fail-open 但不写审计行 —— 与 rerank 形成对比 | 原子层 | 候选 5 的阶段 2，或独立卡"fail-open 的审计一致性缺口" |
| 7 | relational recall 在三条 fallback 路径都注入，而非只在主路径 | 原子层 | 候选 1 的阶段 3 |
| 8 | autocut 的 `preserve` 谓词保护 alias-hop 注入的页 | 原子层 | 新方案《特性交互时的 preserve 谓词设计》或候选 3 的姊妹卡 |

候选 1+2+7 串起来可以做一个完整方案《关系召回的 seed 解析与置信度闸门》，分 3 个阶段。
候选 3+4+8 串起来可以做一个完整方案《autocut 的悬崖截断》，分 3 个阶段。
候选 5+6 串起来可以做一个完整方案《检索后处理阶段的 fail-open 姿态》，分 2 个阶段，并指出 cosineReScore 的审计缺口。

这三条方案都直接补全总目录"下一步计划"里 RRF 剩余项。

## 11. 已读证据

实际打开过（Read 工具）的文件：

| 文件 | 范围 |
|---|---|
| `src/core/search/hybrid.ts` | 全文 2006 行 |
| `src/core/search/relational-recall.ts` | 全文 233 行 |
| `src/core/search/relational-intent.ts` | 全文 244 行 |
| `src/core/search/autocut.ts` | 全文 222 行 |
| `src/core/search/rerank.ts` | 全文 139 行 |
| `src/core/search/mode.ts` | 全文 1138 行 |
| `src/core/search/token-budget.ts` | 前 113 行 |
| `src/core/search/return-policy.ts` | 前 132 行 |
| `src/core/search/graph-signals.ts` | 前 60 行 + 130-419 行 |
| `src/core/search/keyword.ts` | 全文 11 行 |
| `src/core/search/vector.ts` | 全文 11 行 |
| `src/core/search/sql-ranking.ts` | 前 60 行 |
| `src/core/search/dedup.ts` | 前 60 行 |
| `src/core/search/query-cache.ts` | 前 60 行 |
| `src/core/search/query-cache-gate.ts` | 前 60 行 |
| `src/core/search/query-intent.ts` | 前 60 行 |
| `src/core/search/intent-weights.ts` | 前 80 行 + 80-145 行 |
| `src/core/search/two-pass.ts` | 前 60 行 |
| `src/core/search/evidence.ts` | 前 60 行 |
| `src/core/search/alias-normalize.ts` | 全文 57 行 |
| `src/core/search/title-match.ts` | 前 60 行 |
| `src/core/search/source-boost.ts` | 前 60 行 |
| `src/core/search/expansion.ts` | 前 60 行 |
| `src/core/search/telemetry.ts` | 前 60 行 |
| `src/core/search/embedding-column.ts` | 前 60 行 |
| `src/core/search/by-image.ts` | 前 60 行 |
| `src/core/search/recency-decay.ts` | 前 60 行 |
| `src/core/search/explain-formatter.ts` | 前 60 行 |
| `src/core/search/llm-intent.ts` | 前 60 行 |
| `src/core/search/mode-switch-ux.ts` | 前 60 行 |
| `src/core/search/image-loader.ts` | 前 60 行 |
| `src/core/search/eval.ts` | 前 60 行 |
| `src/core/rerank-audit.ts` | 前 60 行 |
| `src/core/types.ts` | 1235-1251 行（`RelationalFanoutOpts`） |
| `src/core/engine.ts` | 1240-1289 行（`relationalFanout` / `getAdjacencyBoosts` / `getContentFlagsByPageIds` 声明） |

另用 Grep 全仓扫描过 `hybridSearch` / `buildRelationalArm` / `applyReranker` / `applyAutocut` / `cosineReScore` / `relationalFanout` / `getEmbeddingsByChunkIds` / `getAdjacencyBoosts` 的所有调用点。

## 12. 待深读问题

1. `query-cache.ts` 的 `lookup`/`store` 完整实现未读 —— cache 行何时拒绝写入？page_generations 快照构造细节？
2. `query-cache-gate.ts` 的两层失效闸门完整逻辑未读 —— Layer 1 clock vs Layer 2 snapshot 的 fallback 顺序？空 `{}` 在 v0.41.19.0+ 的语义反转？
3. `dedup.ts` 的 4 层去重完整实现未读 —— Jaccard 相似度计算？compiled_truth guarantee 的具体逻辑？
4. `by-image.ts` 的 image + text refine 加权 RRF 合并未读 —— Phase 3 unified_multimodal 的 fallback 路径？
5. `embedding-column.ts` 的 `isCacheSafe` 比较逻辑未读全 —— 直接关系 cache 能否安全启用
6. `query-intent.ts` 的 regex pattern bank 和 `isAmbiguousModalityQuery` 未读 —— intent-weights 和 adaptive-return 的输入源
7. `postgres-engine.ts` / `pglite-engine.ts` 的 `relationalFanout` SQL 实现（:3039 / :2971）未读 —— BFS SQL 形状、source 隔离 WHERE、depth 硬上限 3 的执行位置
8. `src/core/ai/gateway.ts` 的 `rerank` / `RerankError` 定义未读 —— reason 分类逻辑、payload_too_large 触发条件
9. `operations.ts` 的 `query` op 完整调用上下文（`:1635` 调 `hybridSearchCached` 的参数构造）未读 —— `ctx.remote` 信任边界如何影响 mode 透传
10. `src/core/entities/resolve.ts` 的 `resolveEntitySlugWithSource` 未读 —— relational-recall.ts:91 调用它决定 seed 何时落到 `fallback_slugify`

---

*生成依据：《开源项目工程经验提炼提示词模板》场景二。未经场景五审查，不作为经验库条目入库。*