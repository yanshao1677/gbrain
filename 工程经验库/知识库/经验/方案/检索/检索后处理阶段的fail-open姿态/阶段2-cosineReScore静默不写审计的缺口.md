---
title: 阶段2-cosineReScore 静默不写审计的缺口
level: atomic
parent: ../检索后处理阶段的fail-open姿态.md
status: reviewed
tags:
  - fail-open
  - audit-gap
  - cosine-rescore
  - error-handling
  - retrieval
created_at: 2026-07-06
updated_at: 2026-07-06
confidence: high
related:
  - 阶段1-rerank双层try-catch兜住业务错与审计写错.md
  - ../安全/远程调用Fail-Closed信任边界.md
source_repo: garrytan/gbrain
source_commit:
source_paths:
  - src/core/search/hybrid.ts
  - src/core/search/rerank.ts
  - src/core/rerank-audit.ts
  - src/core/search/autocut.ts
---

# 阶段2：cosineReScore 静默不写审计的缺口

## 触发条件

后处理阶段函数 `cosineReScore` 被调用，且满足以下全部前置：

- 主路径执行（vector 召回成功，非 no-embed / embed-fail fallback 路径）
- `queryEmbedding` 存在（hybrid.ts:1369 `if (queryEmbedding)`）
- 即：text 查询 embed 成功，或 image 查询有 text refine（unifiedEmbedding）

`cosineReScore` 是 hybrid.ts 模块内私有函数（`async function` 而非 `export async function`），仅主路径一处调用（hybrid.ts:1370）。no-embed / embed-fail 两条 fallback 路径因 `queryEmbedding=null` 跳过本阶段。

## 输入字段

| 字段 | 类型 | 是否可选 | 来源 |
|---|---|---|---|
| `engine` | BrainEngine | 必填 | 调用方传入，DB 引擎接口 |
| `results` | SearchResult[] | 必填 | RRF 融合后的结果集（`fused`） |
| `queryEmbedding` | Float32Array | 必填 | query 向量，主路径 `embedQueryBounded` 返回 |
| `column` | string | 可选，默认 `'embedding'` | 调用方传 `resolvedCol.name`（v0.36 D9 后动态解析） |

## 判定规则

### chunk_id 提取与过滤（hybrid.ts:1957-1959）

```ts
const chunkIds = results
  .map(r => r.chunk_id)
  .filter((id): id is number => id != null);
```

- 提取每条 result 的 `chunk_id`
- 过滤 `null` / `undefined`（类型守卫 `(id): id is number`）

### 早返回条件一：无 chunk_id（hybrid.ts:1961）

```ts
if (chunkIds.length === 0) return results;
```

- 触发条件：所有 result 的 `chunk_id` 都为 null
- 行为：返回输入不变，**不写审计**，不进 try

### DB 调用与 catch（hybrid.ts:1964-1973）

```ts
let embeddingMap: Map<number, Float32Array>;
try {
  embeddingMap = await engine.getEmbeddingsByChunkIds(chunkIds, column);
} catch {
  // DB error is non-fatal, return results without re-scoring
  return results;
}
```

- **单层 try/catch**，catch 块**无参数**（连 `err` 都不接收）
- 失败时静默 `return results`，**不写审计行**
- 注释只说"non-fatal"，未解释为什么不写审计

### 早返回条件二：embeddingMap 为空（hybrid.ts:1975）

```ts
if (embeddingMap.size === 0) return results;
```

- 触发条件：DB 调用成功但返回空 Map（chunk_id 都查不到 embedding）
- 行为：返回输入不变，**不写审计**

### 成功路径混合公式（hybrid.ts:1978-1993）

```ts
const maxRrf = Math.max(...results.map(r => r.score));
// ...
const cosine = cosineSimilarity(queryEmbedding, chunkEmb);
const normRrf = maxRrf > 0 ? r.score / maxRrf : 0;
const blended = 0.7 * normRrf + 0.3 * cosine;
return { ...r, score: blended };
```

魔法数字：

- `0.7`：RRF 归一化分数权重
- `0.3`：cosine 相似度权重
- 二者和为 1.0，加权平均
- **证据不足：0.7/0.3 的来源未确认**。代码无注释说明这两个权重的校准依据（无 eval 引用、无 PR 编号、无 commit message 线索）。推测是经验值，但需实现时确认是否有过 eval 校准

`cosineSimilarity` 实现（hybrid.ts:1996-2005）：标准点积 / (模A × 模B)，分母为 0 返回 0。

## 状态读写位置

- **读**：`results`（输入）、`queryEmbedding`（输入）、`engine.getEmbeddingsByChunkIds`（DB 读）
- **写**：无（成功路径返回新数组，失败路径返回原数组；无 DB 写、无文件写、无进程内状态写）
- **DB**：读 `chunks` 表的 embedding 列（通过 `engine.getEmbeddingsByChunkIds`）
- **审计**：**无** —— 这是本阶段的核心缺口

跨进程语义：cosineReScore 是纯内存计算 + 一次 DB 读，无跨进程状态。但**降级不可观测**是跨进程问题——多 worker 部署时，cosineReScore 持续 fail-open 不会在任何监控里出现，操作员无法知道检索质量在降级。

## 正常路径

按真实顺序编号：

1. 提取 chunkIds（hybrid.ts:1957-1959）
2. 早返回检查一：`chunkIds.length === 0` → return results（:1961）
3. **单层 try**（:1964）：`engine.getEmbeddingsByChunkIds(chunkIds, column)`
4. 成功 → embeddingMap 填充
5. 早返回检查二：`embeddingMap.size === 0` → return results（:1975）
6. 算 `maxRrf`（:1978）
7. 对每条 result：取 chunkEmb → 算 cosine → 算 normRrf → 算 blended → 返回新对象 `{...r, score: blended}`（:1980-1992）
8. 按 blended 降序排序（:1993）
9. 返回新数组

## 分支路径

### A. catch：DB 调用失败（hybrid.ts:1970-1973）

```ts
} catch {
  // DB error is non-fatal, return results without re-scoring
  return results;
}
```

- **静默吞**，连 `err` 参数都不接收
- 返回**输入数组不变**（RRF 顺序）
- **不写审计行**
- 不区分错误类（network / timeout / permission / relation 不存在 全部一锅烩）

### B. 早返回一：无 chunk_id（hybrid.ts:1961）

- 返回输入不变
- 不写审计
- 这是"无可重排项"的正常情况，不算失败

### C. 早返回二：embeddingMap 为空（hybrid.ts:1975）

- 返回输入不变
- 不写审计
- 这是"DB 返回空"的边缘情况——可能是 chunk_id 在 chunks 表里查不到 embedding（数据不一致），但 cosineReScore 不报告

### D. DEBUG 模式（hybrid.ts:1988-1990）

```ts
if (DEBUG) {
  console.error(`[search-debug] ${r.slug}:${r.chunk_id} cosine=${...} norm_rrf=${...} blended=${...}`);
}
```

- DEBUG 是模块级常量，生产关闭
- 打 stderr，**不是审计行**
- 只在成功路径打，失败路径不打

## 失败处理

| 失败点 | 处理 | 是否写审计 | 是否中断检索 |
|---|---|---|---|
| `getEmbeddingsByChunkIds` 抛 DB 错 | catch 静默 → return results | **否** | 否 |
| chunkIds 全为 null | 早返回 return results | 否 | 否 |
| embeddingMap 为空 | 早返回 return results | 否 | 否 |
| `cosineSimilarity` 返回 NaN（分母为 0 已处理返回 0） | 不会发生（:2003-2004 兜底） | 不适用 | 否 |

**核心缺口**：DB 调用失败时不写审计。与阶段1 rerank 的双层 try/catch + 审计写入形成鲜明对比。

| 维度 | rerank（阶段1） | cosineReScore（本阶段） |
|---|---|---|
| try/catch 层数 | 双层（外层业务 + 内层审计） | 单层（catch 无参数） |
| 审计写入 | 是（logRerankFailure） | **否** |
| 错误分类 | 6 类 reason | 不分类 |
| doctor 检查 | `reranker_health` 读审计 | **无对应检查** |
| 降级可观测 | 是 | **否** |

> `doctor` 是 gbrain 的运维自检命令（`gbrain doctor`），`reranker_health` 是其中一个检查项，它读取 rerank-audit JSONL 文件统计近期失败率并向操作员报告。cosineReScore 没有对应的 doctor 检查项，因为没有任何审计文件可读。

## 幂等性 / 一致性约束

- **fail-open 是无状态的**：同一输入重试，要么成功要么静默返回原数组
- **降级语义严格"输入不变"**：return 的是 `results` 原数组，下游拿到 RRF 顺序
- **不修改 `rerank_score` 字段**：cosineReScore 改的是 `score`（RRF 分数），不动 `rerank_score`。与阶段1 rerank 改 `rerank_score` 形成字段分工
- **不能依赖的前提**：cosineReScore 失败时 `score` 字段保持 RRF 原值，下游 autocut 不读 `score` 而读 `rerank_score`（autocut.ts:9-16 注释），所以 cosineReScore fail-open 不会让 autocut 级联 noOp —— 这与 rerank fail-open 的级联行为不同
- **可观测性缺口**：cosineReScore 持续 fail-open 时，操作员无法从任何 audit 文件或 doctor 检查发现。只能通过"检索质量下降"的间接信号察觉，而质量下降难以归因到具体阶段

## 代码骨架

TypeScript 实现（直接参考 hybrid.ts:1951-1994，保留缺口）：

```ts
async function cosineReScore(
  engine: BrainEngine,
  results: SearchResult[],
  queryEmbedding: Float32Array,
  column: string = 'embedding',
): Promise<SearchResult[]> {
  const chunkIds = results
    .map(r => r.chunk_id)
    .filter((id): id is number => id != null);

  // 早返回一：无 chunk_id
  if (chunkIds.length === 0) return results;

  let embeddingMap: Map<number, Float32Array>;
  try {
    embeddingMap = await engine.getEmbeddingsByChunkIds(chunkIds, column);
  } catch {
    // 缺口：此处不写审计，降级不可观测
    // 对比 rerank.ts:89-99 的双层 try/catch + logRerankFailure
    return results;
  }

  // 早返回二：DB 返回空
  if (embeddingMap.size === 0) return results;

  const maxRrf = Math.max(...results.map(r => r.score));

  return results.map(r => {
    const chunkEmb = r.chunk_id != null ? embeddingMap.get(r.chunk_id) : undefined;
    if (!chunkEmb) return r;

    const cosine = cosineSimilarity(queryEmbedding, chunkEmb);
    const normRrf = maxRrf > 0 ? r.score / maxRrf : 0;
    const blended = 0.7 * normRrf + 0.3 * cosine;  // 证据不足：0.7/0.3 来源未确认

    return { ...r, score: blended };
  }).sort((a, b) => b.score - a.score);
}

function cosineSimilarity(a: Float32Array, b: Float32Array): number {
  let dot = 0, magA = 0, magB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    magA += a[i] * a[i];
    magB += b[i] * b[i];
  }
  const denom = Math.sqrt(magA) * Math.sqrt(magB);
  return denom === 0 ? 0 : dot / denom;
}
```

## 最小验证清单

可执行断言：

1. `cosineReScore` 在 `results` 全部 `chunk_id=null` 时直接返回输入，不调用 `getEmbeddingsByChunkIds`
2. `cosineReScore` 在 `chunkIds.length=0` 时直接返回输入，不调用 `getEmbeddingsByChunkIds`
3. `getEmbeddingsByChunkIds` 抛错时，`cosineReScore` 返回输入不变（引用相等），**不写任何审计文件**
4. `getEmbeddingsByChunkIds` resolve 空 Map 时，`cosineReScore` 返回输入不变，不写审计
5. 成功路径返回的 result 对象 `score` 字段被改为 blended 值，`rerank_score` 字段未定义
6. 成功路径返回的数组按 blended 降序排序
7. `maxRrf === 0` 时（所有 result score 为 0），`normRrf = 0`，blended = `0.3 * cosine`
8. `cosineSimilarity` 在零向量输入时返回 0（分母为 0 兜底）
9. `column` 参数透传给 `getEmbeddingsByChunkIds`（v0.36 D9 后必须传 resolvedCol.name，否则 alt-column 排序 corrupt）
10. catch 块不接收 `err` 参数（静默吞，连错误对象都不留）
11. 对比测试：同样失败场景下，rerank 写审计行 + doctor `reranker_health` 能检测；cosineReScore 不写审计 + 无 doctor 检查能检测 —— 验证缺口存在

## 与阶段1的对照（本卡的核心价值）

本卡不是独立方案，而是阶段1 rerank fail-open 姿态的**对照卡**，揭示同一方案内的不一致：

| 维度 | rerank（阶段1，标杆） | cosineReScore（本阶段，缺口） |
|---|---|---|
| try/catch 层数 | 双层 | 单层 |
| 审计写入 | 是 | 否 |
| 错误分类 | 6 类 | 不分类 |
| doctor 可检测 | 是 | 否 |
| 代码注释解释姿态 | 是（rerank.ts:10-13 JSDoc） | 部分（hybrid.ts:1971 "non-fatal" 但未解释为何不审计） |

**结论**：cosineReScore 的 fail-open 是**姿态不一致**的缺口，不是有意设计。理由：

1. 同一方案（检索后处理 fail-open）内，rerank 有审计 cosineReScore 没有，无架构理由
2. cosineReScore 的 DB 失败与 rerank 的 upstream 失败同属"flaky 依赖导致降级"，可观测性需求相同
3. hybrid.ts:1971 注释只说"non-fatal"未解释"non-audited"，说明是疏漏而非权衡
4. v0.36 (D9) 之前 cosineReScore 用固定 `'embedding'` 列静默 corrupt alt-column ranks（hybrid.ts:1367-1368 注释），这种"静默出错"历史模式说明该阶段历来审计意识薄弱

**证据不足项**：未确认是否有 PR 或 commit 显式讨论过"cosineReScore 不写审计"的决定。需查 git log `hybrid.ts` 的 `cosineReScore` 函数历史确认是疏漏还是有意。

## 来源证据（附录，不进正文）

| 项 | 位置 |
|----|------|
| cosineReScore 函数定义 | `src/core/search/hybrid.ts:1951-1994` |
| 单层 try/catch（缺口核心） | `src/core/search/hybrid.ts:1964-1973` |
| 静默 catch 块 | `src/core/search/hybrid.ts:1970-1973` |
| 早返回条件一（无 chunk_id） | `src/core/search/hybrid.ts:1961` |
| 早返回条件二（空 Map） | `src/core/search/hybrid.ts:1975` |
| 混合公式 0.7/0.3 | `src/core/search/hybrid.ts:1986`（**权重来源未确认**，见判定规则） |
| cosineSimilarity 实现 | `src/core/search/hybrid.ts:1996-2005` |
| DEBUG stderr 打印（非审计） | `src/core/search/hybrid.ts:1988-1990` |
| 调用方上下文 | `src/core/search/hybrid.ts:1369-1371` |
| v0.36 D9 alt-column 修复注释 | `src/core/search/hybrid.ts:1365-1368, 1965-1968` |
| queryEmbedding 赋值点 | `src/core/search/hybrid.ts:1193, 1239, 1257, 1698, 1704` |
| 对照：rerank 双层 try/catch | `src/core/search/rerank.ts:85-101` |
| 对照：rerank 审计写入 | `src/core/rerank-audit.ts:85-91` |
| 对照：rerank JSDoc 声明姿态 | `src/core/search/rerank.ts:10-13` |
| 对照：rerank 错误分类 6 类 | `src/core/rerank-audit.ts:31-37` |
| 级联论证：autocut 读 rerank_score 不读 score | `src/core/search/autocut.ts:9-16` |

---

*生成依据：《开源项目工程经验提炼提示词模板》场景 4C。待过场景五审查后入库。*
