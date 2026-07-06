---
title: 阶段1-rerank 双层 try/catch 兜住业务错与审计写错
level: atomic
parent: ../检索后处理阶段的fail-open姿态.md
status: reviewed
tags:
  - fail-open
  - error-handling
  - rerank
  - audit
  - try-catch
created_at: 2026-07-06
updated_at: 2026-07-06
confidence: high
related:
  - ../安全/远程调用Fail-Closed信任边界.md
source_repo: garrytan/gbrain
source_commit:
source_paths:
  - src/core/search/rerank.ts
  - src/core/rerank-audit.ts
---

# 阶段1：rerank 双层 try/catch 兜住业务错与审计写错

## 触发条件

后处理阶段函数 `applyReranker` 被调用，且满足以下全部前置：

- `opts.enabled === true`（mode bundle 为 tokenmax 时默认 true，conservative/balanced 默认 false）
- `results.length > 0`（无结果不需重排）
- `opts.topNIn > 0`（mode bundle 实际从不下发 0）

任一前置不满足时走早返回（rerank.ts:63/66），不进入本阶段。

## 输入字段

| 字段 | 类型 | 是否可选 | 来源 |
|---|---|---|---|
| `query` | string | 必填 | 调用方传入，原始 query 文本 |
| `results` | SearchResult[] | 必填 | 调用方传入，RRF 融合 + cosine 重排后的结果集 |
| `opts.enabled` | boolean | 必填 | mode bundle 解析后传入 |
| `opts.topNIn` | number | 必填 | mode bundle，默认 30 |
| `opts.topNOut` | number \| null | 必填 | mode bundle，null = 不截断 |
| `opts.model` | string | 可选 | provider:model 覆盖 |
| `opts.timeoutMs` | number | 可选 | 默认 5000，透传给 gateway |
| `opts.rerankerFn` | (input) => Promise<RerankResult[]> | 可选 | 测试 seam，生产禁止设 |

## 判定规则

`RerankError` 是上游 rerank gateway（`src/core/ai/gateway.ts`）抛的错误类，含 `reason: RerankFailureReason` 字段标识失败模式。非 RerankError 的抛出（普通 Error、字符串、裸值）一律归为 `'unknown'`。

### 错误分类（rerank.ts:86-87）

```ts
const reason: RerankFailureReason =
  err instanceof RerankError ? err.reason : 'unknown';
```

- 若 `err` 是 `RerankError` 实例 → `reason` 取 `err.reason`，取值范围 `'auth' | 'rate_limit' | 'network' | 'timeout' | 'payload_too_large' | 'unknown'`（rerank-audit.ts:31-37）
- 否则 → `reason = 'unknown'`

### errorSummary 截取（rerank.ts:88）

```ts
const errorSummary = err instanceof Error ? err.message : String(err);
```

- `Error` 实例取 `.message`
- 非 Error（字符串抛出、裸值）走 `String(err)`
- 此处**不截断**；截断在 `logRerankFailure` 内部做（rerank-audit.ts:69-72，截到 200 字符）

### query_hash 计算（rerank.ts:43-44）

```ts
function hashQuery(query: string): string {
  return createHash('sha256').update(query, 'utf8').digest('hex').slice(0, 8);
}
```

- 算法：SHA-256
- 输入编码：utf8
- 截取：hex digest 前 8 字符
- 用途：隐私保护，doctor 去重同一 query 的重复失败

### 审计事件字段（rerank-audit.ts:39-58）

```ts
{
  ts: string,              // logRerankFailure 内部填充 ISO 时间戳
  model: opts.model ?? 'unknown',
  reason,                  // 上一步分类
  query_hash: hashQuery(query),
  doc_count: documents.length,   // head.length，即 topNIn 切片后长度
  error_summary: errorSummary,   // logRerankFailure 内部截到 200
  severity: 'warn',              // 永远 warn
}
```

`ts` 和 `severity` 由 `logRerankFailure` 内部填充，调用方不传。

## 状态读写位置

- **读**：`opts`（配置）、`results`（输入）、`err`（抛出的错误对象）
- **写**：审计行追加到 `~/.gbrain/audit/rerank-failures-YYYY-Www.jsonl`（ISO 周轮转，rerank-audit.ts:4）
- **DB**：无
- **进程内**：无（审计是文件系统追加，无内存状态）

跨进程语义：审计文件是跨进程共享的（多 worker 都追加同一文件），`gbrain doctor reranker_health` 读它判断降级是否在发生。若实现迁移到无共享文件系统的多机部署，需换 central log collector，否则 doctor 检查失效。

## 正常路径

按真实顺序编号：

1. 早返回检查（rerank.ts:63/66）：`!opts.enabled || results.length === 0` 或 `opts.topNIn <= 0` → 直接 `return results`，不进 try
2. 切 head/tail（rerank.ts:68-69）：`head = results.slice(0, topNIn)`，`tail = results.slice(topNIn)`
3. 构造 documents（rerank.ts:74）：`head.map(r => r.chunk_text || r.title || '')`
4. **外层 try**（rerank.ts:77）：调 `rerankerFn({query, documents, timeoutMs, ...model})`
5. 成功 → 跳出 try，进 rerank.ts:104 之后的重排逻辑（不在本阶段范围）
6. 重排完成后 `return [...reorderedHead, ...tail]`（rerank.ts:134），可选 `topNOut` 截断（:135-137）

## 分支路径

### A. 外层 catch：reranker 业务失败（rerank.ts:85-101）

1. 分类 `reason`（:86-87）
2. 取 `errorSummary`（:88）
3. **内层 try**（:89-99）：调 `logRerankFailure({model, reason, query_hash, doc_count, error_summary})`
4. **return results**（:100）：返回**输入数组不变**（RRF 顺序），不进重排逻辑

注意 `return results` 是原始输入，不是 `head` 或 `tail` 切片——完整链路继续往下走，下游（autocut 等）拿到的就是没重排的 RRF 结果。

### B. 内层 catch：审计写失败（rerank.ts:97-99）

```ts
} catch {
  // Audit logging must never break search.
}
```

- 静默吞，无语句
- 落到外层 catch 的 `return results`（:100）

实际上 `logRerankFailure` 内部已经 best-effort（rerank-audit.ts:20, 82-91），写失败只写 stderr 不抛。内层 try/catch 是**双保险**——防御 `createAuditWriter` 或 `truncateErrorSummary` 自身抛非预期错误（比如 `JSON.stringify` 遇循环结构、文件系统 API 抛 EPERM）。

### C. 畸形 shape 防御（rerank.ts:104，成功路径但不重排）

```ts
if (!Array.isArray(reranked) || reranked.length === 0) return results;
```

- 触发条件：rerankerFn resolve 但返回非数组或空数组
- 行为：返回输入不变，**不写审计**（视为业务正常，只是没结果）
- 这与外层 catch 不同——外层 catch 写审计，这条不写

## 失败处理

| 失败点 | 处理 | 是否写审计 | 是否中断检索 |
|---|---|---|---|
| rerankerFn 抛 RerankError | 外层 catch → 内层 try 写审计 → return results | 是 | 否 |
| rerankerFn 抛非 RerankError | 外层 catch，reason='unknown' → 内层 try 写审计 → return results | 是 | 否 |
| logRerankFailure 抛错 | 内层 catch 静默 → return results | 否（审计自身挂了） | 否 |
| rerankerFn resolve 畸形 shape | rerank.ts:104 早返回 return results | 否 | 否 |
| 早返回条件（enabled=false/空输入/topNIn=0） | rerank.ts:63/66 直接 return | 否 | 否 |

证据不足项：**未确认** `logRerankFailure` 在哪些边缘情况下会抛错。rerank-audit.ts:20 注释说"Write failures go to stderr but search continues"，但 `createAuditWriter` 的具体实现在 `src/core/audit/audit-writer.ts`（未读）。如果 `audit-writer.ts` 的 `log` 方法本身不抛（best-effort 到底），内层 try/catch 就是纯防御性死代码；如果它会抛（比如 JSON.stringify 循环），内层 try/catch 是真兜底。需实现时确认。

## 幂等性 / 一致性约束

- **fail-open 是无状态的**：同一输入重试，要么成功要么再写一条审计行，不累积副作用
- **审计行是 append-only**：重复失败会写多条 ts 不同的行，doctor 按 `query_hash` 去重统计
- **降级语义严格"输入不变"**：return 的是 `results` 原数组，不是 copy、不是部分结果。下游拿到的是 RRF 顺序，与"没开 reranker"等价
- **不能依赖的前提**：`results` 数组的 `rerank_score` 字段在 fail-open 后**未设置**（只有成功路径在 :119 设置）。下游 autocut 检查 `rerank_score` 是否存在来决定是否触发 noOp（hybrid.ts:1506-1507 注释），这是 fail-open 级联的关键约束——**不能在 fail-open 路径伪造一个 rerank_score**

## 代码骨架

TypeScript 实现（直接参考 rerank.ts:58-101，去掉重排逻辑只留 fail-open 骨架）：

```ts
import { createHash } from 'crypto';
import { logRerankFailure, RerankError, type RerankFailureReason } from './rerank-audit.ts';

export interface RerankerOpts {
  enabled: boolean;
  topNIn: number;
  topNOut: number | null;
  model?: string;
  timeoutMs?: number;
  rerankerFn?: (input: { query: string; documents: string[]; timeoutMs?: number; model?: string })
    => Promise<Array<{ index: number; relevanceScore: number }>>;
}

function hashQuery(query: string): string {
  return createHash('sha256').update(query, 'utf8').digest('hex').slice(0, 8);
}

export async function applyReranker<T extends { chunk_text?: string; title?: string }>(
  query: string,
  results: T[],
  opts: RerankerOpts,
): Promise<T[]> {
  // 早返回：不进 try，不写审计
  if (!opts.enabled || results.length === 0) return results;
  if (opts.topNIn <= 0) return results;

  const head = results.slice(0, opts.topNIn);
  const documents = head.map(r => r.chunk_text || r.title || '');

  let reranked: Array<{ index: number; relevanceScore: number }>;
  try {
    const rerankerFn = opts.rerankerFn ?? defaultReranker;
    reranked = await rerankerFn({
      query,
      documents,
      timeoutMs: opts.timeoutMs,
      ...(opts.model ? { model: opts.model } : {}),
    });
  } catch (err) {
    // 外层 catch：业务错
    const reason: RerankFailureReason =
      err instanceof RerankError ? err.reason : 'unknown';
    const errorSummary = err instanceof Error ? err.message : String(err);
    try {
      // 内层 try：审计写错也兜住
      logRerankFailure({
        model: opts.model ?? 'unknown',
        reason,
        query_hash: hashQuery(query),
        doc_count: documents.length,
        error_summary: errorSummary,
      });
    } catch {
      // Audit logging must never break search.
    }
    return results;  // 输入不变
  }

  // 畸形 shape 防御（不写审计）
  if (!Array.isArray(reranked) || reranked.length === 0) return results;

  // ... 重排逻辑（不在本阶段范围） ...
  return results;  // 占位，实际是 [...reorderedHead, ...tail]
}
```

注意骨架里的 `<T extends { chunk_text?: string; title?: string }>` 泛型约束——生产代码用具体的 `SearchResult` 类型，骨架用结构类型让读者看清依赖的是哪两个字段。

## 最小验证清单

可执行断言：

1. `applyReranker` 在 `opts.enabled=false` 时直接返回输入，不调用 rerankerFn
2. `applyReranker` 在 `results.length=0` 时直接返回输入，不调用 rerankerFn
3. `applyReranker` 在 `opts.topNIn=0` 时直接返回输入，不调用 rerankerFn
4. rerankerFn 抛 RerankError(reason='timeout') 时，applyReranker 返回输入不变（引用相等），且 `logRerankFailure` 被调用一次，event.reason='timeout'
5. rerankerFn 抛普通 Error 时，applyReranker 返回输入不变，event.reason='unknown'，event.error_summary=err.message
6. rerankerFn 抛字符串 `'boom'` 时，event.error_summary='boom'（String(err) 兜底）
7. logRerankFailure 抛错时（mock 让它 throw），applyReranker 仍返回输入不变，不向上抛
8. rerankerFn resolve `[]` 时，applyReranker 返回输入不变，**logRerankFailure 不被调用**（畸形 shape 不写审计）
9. rerankerFn resolve 非数组时，applyReranker 返回输入不变，logRerankFailure 不被调用
10. 成功路径返回的 head 元素 `rerank_score` 字段被设置；fail-open 路径返回的元素 `rerank_score` 字段**未定义**（下游 autocut 依赖此约束）
11. query_hash 是 8 字符 hex（SHA-256 前 8 字符）
12. error_summary 超过 200 字符时被截断到 199 + '…'（rerank-audit.ts:69-72）

## 来源证据（附录，不进正文）

| 项 | 位置 |
|----|------|
| applyReranker 函数体 | `src/core/search/rerank.ts:58-138` |
| 早返回条件 | `src/core/search/rerank.ts:63, 66` |
| 外层 try/catch | `src/core/search/rerank.ts:77-101` |
| 内层 try/catch（审计兜底） | `src/core/search/rerank.ts:89-99` |
| reason 分类 | `src/core/search/rerank.ts:86-87` |
| hashQuery 实现 | `src/core/search/rerank.ts:43-44` |
| 畸形 shape 防御 | `src/core/search/rerank.ts:104` |
| JSDoc 声明 fail-open + Never throws | `src/core/search/rerank.ts:10-13, 47-56` |
| RerankFailureReason 联合类型 | `src/core/rerank-audit.ts:31-37` |
| RerankFailureEvent 字段定义 | `src/core/rerank-audit.ts:39-58` |
| logRerankFailure best-effort 语义 | `src/core/rerank-audit.ts:20, 82-91` |
| truncateErrorSummary 截断规则 | `src/core/rerank-audit.ts:69-72` |
| 审计文件路径与轮转 | `src/core/rerank-audit.ts:4, 60-63` |
| 调用方（hybrid.ts 主路径） | `src/core/search/hybrid.ts:1461-1463` |
| 下游 autocut 依赖 rerank_score 存在 | `src/core/search/hybrid.ts:1506-1507` 注释 + `src/core/search/autocut.ts:158` |
| createAuditWriter 实现（未读） | `src/core/audit/audit-writer.ts`（路径待核实） |

---

*生成依据：《开源项目工程经验提炼提示词模板》场景 4C。待过场景五审查后入库。*
