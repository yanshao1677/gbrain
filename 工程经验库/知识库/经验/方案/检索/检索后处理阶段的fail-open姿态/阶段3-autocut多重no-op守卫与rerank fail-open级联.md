---
title: 阶段3-autocut多重no-op守卫与rerank fail-open级联
level: atomic
parent: ../检索后处理阶段的fail-open姿态.md
status: reviewed
tags:
  - fail-open
  - no-op-guard
  - autocut
  - score-discontinuity
  - cascading
  - retrieval
created_at: 2026-07-06
updated_at: 2026-07-06
confidence: high
related:
  - 阶段1-rerank双层try-catch兜住业务错与审计写错.md
  - 阶段2-cosineReScore静默不写审计的缺口.md
source_repo: garrytan/gbrain
source_commit:
source_paths:
  - src/core/search/autocut.ts
  - src/core/search/hybrid.ts
---

# 阶段3：autocut 多重 no-op 守卫与 rerank fail-open 级联

## 触发条件

后处理阶段函数 `applyAutocut` 被调用，且满足以下全部前置：

- `resolvedMode.autocut` 为 true（mode bundle 决定，tokenmax/balanced 默认开，conservative 默认关）
- `offset === 0`（仅第一页触发，分页查询跳过）
- 即：首页查询且 mode 开启 autocut

触发判定在 hybrid.ts:1509 `if (resolvedMode.autocut && offset === 0)`。两个条件任一不满足，autocut 完全不执行（不是 no-op，是根本不调用）。

`applyAutocut` 是 `src/core/search/autocut.ts` 导出的纯函数（`export function`，非 async），唯一调用点是 hybrid.ts:1510。它是 fail-open 方案里的第三种姿态：**无 try/catch，靠多重 no-op 守卫降级**——既不写审计（像 rerank 那样），也不静默吞错（像 cosineReScore 那样），而是把所有"无法判定"的情况都走 no-op 分支。

## 输入字段

| 字段 | 类型 | 是否可选 | 来源 |
|---|---|---|---|
| `results` | T[] | 必填 | 待裁剪的排名结果集（hybrid.ts 传 `returnPool`） |
| `scoreOf` | `(r: T) => number \| undefined \| null` | 必填 | 分数提取函数（hybrid.ts 传 `(x) => x.rerank_score`） |
| `cfg` | AutocutConfig | 必填 | `{ enabled, jumpRatio, minKeep }`（hybrid.ts 传 `{ enabled: true, jumpRatio: resolvedMode.autocut_jump, minKeep: 1 }`） |
| `preserve` | `(r: T) => boolean` | 可选 | 永久保留谓词（hybrid.ts 传 `(x) => x.alias_hit === true`） |

`AutocutConfig` 字段：

| 字段 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `enabled` | boolean | true | 主开关（实际生效要 mode bundle + reranker 评分前置门） |
| `jumpRatio` | number | 0.2 | 归一化 gap 阈值，gap ≥ jumpRatio 才算 cliff。**有 eval 依据**（PrecisionMemBench 校准，autocut.ts:30-31 JSDoc 明示非 magic constant） |
| `minKeep` | number | 1 | failsafe：候选 ≥1 时永不返回少于这个数 |

## 判定规则

### 守卫一：主开关或数量不足（autocut.ts:148）

```ts
if (!cfg.enabled || results.length < 2) return noOp(results);
```

- `!cfg.enabled` → no-op（signal 'none'）
- `results.length < 2` → no-op（少于 2 项无可裁剪）

### 守卫二：有限分数不足（autocut.ts:153-158）

```ts
const scores: number[] = [];
for (const r of results) {
  const s = scoreOf(r);
  if (typeof s === 'number' && Number.isFinite(s)) scores.push(s);
}
if (scores.length < 2) return noOp(results);
```

- 遍历 results，调 `scoreOf` 提取分数
- 仅保留 `typeof === 'number' && Number.isFinite()` 的分数
- 不足 2 个有限分数 → no-op

**关键级联点**：rerank fail-open 时（阶段1），`applyReranker` 返回 RRF 原序、`rerank_score` 未设置。这里 `scoreOf = (x) => x.rerank_score` 取到 undefined，过滤后 `scores.length === 0 < 2`，走 no-op。**这就是 rerank fail-open 让 autocut 级联 no-op 的机制**——不是显式标记，而是 scoreOf 取不到值自然降级。hybrid.ts:1505-1506 注释明确点出这个设计："covers the fail-open reranker path, where applyReranker returns RRF order with no scores"。

### 守卫三：分数标尺不可用（autocut.ts:160-161）

```ts
const top = Math.max(...scores);
if (!Number.isFinite(top) || top <= 0) return noOp(results);
```

- `top` 非有限或 ≤ 0 → no-op
- 因为归一化用 `s / top`，top 不可用则归一化无意义

### 守卫四：无 cliff 越过阈值（autocut.ts:170-193）

```ts
let bestGap = -1;
let bestIdx = -1;
for (let i = minKeep - 1; i < norm.length - 1; i++) {
  const gap = norm[i] - norm[i + 1];
  if (gap > bestGap) {
    bestGap = gap;
    bestIdx = i;
  }
}
if (bestIdx < 0 || bestGap < cfg.jumpRatio) {
  return { kept: results, decision: { applied: false, signal: 'none', ..., gapRatio: bestGap < 0 ? 0 : bestGap } };
}
```

- 从 `minKeep - 1` 起遍历（保证不违反 minKeep failsafe）
- 找最大连续 gap
- `bestIdx < 0`（未找到）或 `bestGap < jumpRatio`（gap 不够陡）→ no-op，但 `gapRatio` 字段回传观测值供遥测

### 成功路径：cliff 裁剪（autocut.ts:199-220）

```ts
const threshold = sorted[bestIdx];
const kept = results.filter((r) => {
  if (preserve?.(r)) return true;
  const s = scoreOf(r);
  return typeof s === 'number' && Number.isFinite(s) && s >= threshold;
});
if (kept.length === 0) return noOp(results);  // 兜底，理论上不会发生
return { kept, decision: { applied: kept.length < results.length, signal: kept.length < results.length ? 'rerank' : 'none', ... } };
```

- threshold = cliff 处的分数
- `preserve?.(r)` 优先保留（alias_hop 精确匹配无 rerank_score 也保住）
- 否则要求 `score >= threshold`（边界 ties 一起留，保守不空）
- `kept.length === 0` 兜底返回 no-op（注释 :206-207 说"理论上不会发生，因为 top item 总会通过，但仍然 guard"）
- `applied` 字段：只有真删了才 true（`kept.length < results.length`）

## 状态读写位置

- **读**：`results`（输入）、`scoreOf(r)`（提取每条 result 的 rerank_score）、`preserve?(r)`（可选谓词）
- **写**：无（纯函数，返回新数组 `{ kept, decision }`，不改输入；无 DB、无文件、无进程状态、无审计）
- **DB**：无
- **审计**：**无** —— 与 cosineReScore 同属"不写审计"一类，但姿态不同（autocut 是纯函数无 try/catch，cosineReScore 是有 try/catch 但不写审计）

跨进程语义：autocut 是纯内存计算，无副作用。降级（no-op）通过返回值 `decision.applied === false` 表达，**调用方 hybrid.ts 把 decision 收进 `autocutDecision` 变量用于遥测**（hybrid.ts:1508, 1520），但不落盘。所以降级可观测性介于 rerank（写审计行）和 cosineReScore（完全不可观测）之间——**进程内可观测，跨进程不可观测**。

## 正常路径

按真实顺序编号：

1. 守卫一检查（:148）：`!cfg.enabled || results.length < 2` → no-op
2. 收集有限分数（:153-157）：遍历 results，`scoreOf(r)` 过滤 finite
3. 守卫二检查（:158）：`scores.length < 2` → no-op
4. 算 `top = Math.max(...scores)`（:160）
5. 守卫三检查（:161）：`!Number.isFinite(top) || top <= 0` → no-op
6. 排序副本降序（:164）：`[...scores].sort((a, b) => b - a)`
7. 归一化（:165）：`norm = sorted.map(s => s / top)`
8. 算 `minKeep = Math.max(1, cfg.minKeep)`（:167）
9. 找最大 gap（:170-178）：从 `minKeep - 1` 起遍历，记 `bestGap` 和 `bestIdx`
10. 守卫四检查（:180）：`bestIdx < 0 || bestGap < cfg.jumpRatio` → no-op（带 gapRatio 遥测）
11. 算 threshold = `sorted[bestIdx]`（:199）
12. filter（:200-204）：`preserve` 优先 + `score >= threshold`
13. 兜底检查（:208）：`kept.length === 0` → no-op
14. 返回 `{ kept, decision: { applied: kept.length < results.length, ... } }`（:210-220）

## 分支路径

### A. 守卫一：主开关关或数量不足（:148）

- 返回 `{ kept: results, decision: noOpDecision }`
- `applied: false, signal: 'none', gapRatio: 0`
- 不算失败，是正常跳过

### B. 守卫二：有限分数不足（:158）

- **rerank fail-open 级联入口**：rerank 失败时 `rerank_score` 全 undefined，过滤后 `scores.length === 0`
- 返回 no-op，`results` 原样
- 不写审计，不报错
- 与阶段1的级联关系：rerank fail-open → rerank_score 未设置 → autocut scoreOf 取 undefined → 守卫二触发 → autocut no-op

### C. 守卫三：top 不可用（:161）

- 返回 no-op
- 罕见：所有有限分数都 ≤ 0，或 Math.max 返回非有限（理论上 scores 已过滤 finite，此守卫是防御性冗余）

### D. 守卫四：无 cliff 越阈值（:180-193）

- 返回 `{ kept: results, decision: { applied: false, signal: 'none', gapRatio: bestGap } }`
- **gapRatio 回传**：供遥测观察"差一点就 cut"的情况
- 这是最常见的 no-op 路径——多数查询本就无 cliff

### E. 兜底：kept.length === 0（:208）

- 返回 no-op
- 注释 :206-207："a degenerate threshold could in theory keep 0 (it cannot here, since the top item always passes), but guard anyway"
- 防御性冗余守卫

## 失败处理

| 失败点 | 处理 | 是否写审计 | 是否中断检索 |
|---|---|---|---|
| `scoreOf(r)` 抛错 | **不会发生**（scoreOf 是 `(x) => x.rerank_score`，属性访问不抛） | 不适用 | 否 |
| `Math.max(...scores)` 溢出 | 守卫三拦截（top 非有限 → no-op） | 否 | 否 |
| `sort` 比较函数异常 | 不会发生（纯数值比较） | 不适用 | 否 |
| rerank 上游 fail-open 导致 score 全空 | 守卫二拦截（scores.length < 2 → no-op） | 否 | 否 |
| `preserve?.(r)` 抛错 | 可能抛出（preserve 是可选谓词，若调用方传错实现） | 否 | **是**（无 try/catch 兜） |

**核心姿态**：autocut 假设所有输入都是"良构"的——`scoreOf` 和 `preserve` 是调用方提供的同步纯函数，不应抛错。因此**无 try/catch**，靠多重 no-op 守卫处理"数据不可用"（而非"代码出错"）的情况。这是与 rerank（有 try/catch 兜 upstream flaky）、cosineReScore（有 try/catch 兜 DB flaky）的根本区别：autocut 的依赖都是进程内同步函数，没有 flaky 外部依赖需要兜。

`preserve` 抛错时不兜底是有意设计——调用方 bug 不该由被调用方静默吞，让异常上抛暴露问题更利于发现。这与 rerank 内层 catch 兜审计写错的逻辑不同：rerank 兜的是"基础设施失败"（审计文件写不进去），autocut 不兜的是"调用方逻辑错"（谓词实现有 bug），两者性质不同。

## 幂等性 / 一致性约束

- **纯函数无副作用**：同一输入永远同一输出，可重复调用
- **no-op 语义严格"输入不变"**：返回 `kept: results`（同一引用），下游拿到原数组
- **不改 `rerank_score` 字段**：autocut 只读 `rerank_score` 做裁剪决策，不改任何字段
- **minKeep failsafe 不可绕过**：循环从 `minKeep - 1` 起（:172），保证裁剪后至少保留 minKeep 项；兜底 `kept.length === 0` 再守一道
- **preserve 优先于 score**：`preserve?(r) === true` 的项无视 score 保留（:201）。hybrid.ts 用此保护 alias_hop 注入的精确匹配（无 rerank_score 也保住，Codex P1 修复）
- **边界 ties 一起留**：`s >= threshold`（:203，非 `>`），cliff 边界的同分项一起保留，保守不空
- **级联关系不可依赖的前提**：autocut 的级联 no-op 依赖 rerank fail-open 时 `rerank_score` 字段确实为 undefined。若 rerank 实现改为"fail-open 时填一个哨兵分数"（如 0 或 -1），会破坏级联——守卫二不再触发，autocut 会对哨兵分数做裁剪。这是 rerank 与 autocut 之间的隐式契约
- **可观测性**：`AutocutDecision` 回传 `applied / signal / cut / kept / total / gapRatio` 六字段，hybrid.ts 收进 `autocutDecision` 变量用于遥测。但不落盘审计，跨进程不可观测

## 代码骨架

TypeScript 实现（直接参考 autocut.ts:101-221，保留全部守卫）：

```ts
interface AutocutConfig {
  enabled: boolean;
  jumpRatio: number;  // 有 eval 依据（PrecisionMemBench 校准），非 magic constant
  minKeep: number;
}

interface AutocutDecision {
  applied: boolean;
  signal: 'rerank' | 'none';
  cut: number;
  kept: number;
  total: number;
  gapRatio: number;
}

function noOp<T>(results: T[]): { kept: T[]; decision: AutocutDecision } {
  return {
    kept: results,
    decision: { applied: false, signal: 'none', cut: results.length, kept: results.length, total: results.length, gapRatio: 0 },
  };
}

export function applyAutocut<T>(
  results: T[],
  scoreOf: (r: T) => number | undefined | null,
  cfg: AutocutConfig,
  preserve?: (r: T) => boolean,
): { kept: T[]; decision: AutocutDecision } {
  // 守卫一：主开关或数量不足
  if (!cfg.enabled || results.length < 2) return noOp(results);

  // 守卫二：收集有限分数，不足 2 个 no-op（rerank fail-open 级联入口）
  const scores: number[] = [];
  for (const r of results) {
    const s = scoreOf(r);
    if (typeof s === 'number' && Number.isFinite(s)) scores.push(s);
  }
  if (scores.length < 2) return noOp(results);

  // 守卫三：top 不可用 no-op
  const top = Math.max(...scores);
  if (!Number.isFinite(top) || top <= 0) return noOp(results);

  const sorted = [...scores].sort((a, b) => b - a);
  const norm = sorted.map((s) => s / top);
  const minKeep = Math.max(1, cfg.minKeep);

  // 找最大 gap，从 minKeep - 1 起（保证 failsafe）
  let bestGap = -1;
  let bestIdx = -1;
  for (let i = minKeep - 1; i < norm.length - 1; i++) {
    const gap = norm[i] - norm[i + 1];
    if (gap > bestGap) {
      bestGap = gap;
      bestIdx = i;
    }
  }

  // 守卫四：无 cliff 越阈值 no-op（带 gapRatio 遥测）
  if (bestIdx < 0 || bestGap < cfg.jumpRatio) {
    return {
      kept: results,
      decision: { applied: false, signal: 'none', cut: results.length, kept: results.length, total: results.length, gapRatio: bestGap < 0 ? 0 : bestGap },
    };
  }

  // cliff 裁剪：preserve 优先 + score >= threshold（边界 ties 一起留）
  const threshold = sorted[bestIdx];
  const kept = results.filter((r) => {
    if (preserve?.(r)) return true;
    const s = scoreOf(r);
    return typeof s === 'number' && Number.isFinite(s) && s >= threshold;
  });

  // 兜底：理论上不会发生（top item 总会通过），但 guard
  if (kept.length === 0) return noOp(results);

  return {
    kept,
    decision: {
      applied: kept.length < results.length,
      signal: kept.length < results.length ? 'rerank' : 'none',
      cut: kept.length,
      kept: kept.length,
      total: results.length,
      gapRatio: bestGap,
    },
  };
}
```

## 最小验证清单

可执行断言：

1. `cfg.enabled === false` 时直接返回 no-op，`kept` 是输入原引用，`applied === false`
2. `results.length < 2` 时直接返回 no-op，不调 `scoreOf`
3. `scoreOf` 对所有项返回 undefined 时（rerank fail-open 场景），`scores.length === 0 < 2`，返回 no-op，`kept` 是输入原引用
4. `scoreOf` 只有 1 项返回有限分数时，返回 no-op
5. 所有有限分数都 ≤ 0 时（top ≤ 0），返回 no-op
6. `Math.max(...scores)` 返回非有限值时（理论上 scores 已过滤，此为防御），返回 no-op
7. 最大 gap < `cfg.jumpRatio` 时，返回 no-op，但 `decision.gapRatio` 回传观测到的 bestGap
8. 最大 gap ≥ `cfg.jumpRatio` 时，`applied === true`，`signal === 'rerank'`，`kept.length < results.length`
9. `minKeep = 3` 时，循环从 `i = 2` 起，裁剪后 `kept.length >= 3`
10. `preserve(r) === true` 的项即使 `scoreOf(r)` 返回 undefined 也被保留
11. cliff 边界有 ties（多项同分等于 threshold）时，所有 ties 一起保留（`>=` 非 `>`）
12. 兜底：构造 threshold 使所有项都不通过（理论不可能，构造异常 preserve 实现），`kept.length === 0` 时返回 no-op
13. 级联测试：rerank fail-open → rerank_score 全 undefined → autocut scoreOf 全 undefined → 守卫二触发 → autocut no-op（验证隐式契约）
14. `applied` 字段：`kept.length === results.length` 时（无项被删）`applied === false`，即使走了 cliff 路径

## 与阶段1、阶段2的对照（本卡的核心价值）

本卡揭示 fail-open 方案内的**三种姿态**：

| 维度 | rerank（阶段1） | cosineReScore（阶段2） | autocut（本阶段） |
|---|---|---|---|
| try/catch | 双层 | 单层（catch 无参数） | **无** |
| 审计写入 | 是 | 否 | 否 |
| 降级机制 | catch 返回原数组 | catch 返回原数组 | **多重 no-op 守卫** |
| 依赖类型 | flaky upstream（LLM gateway） | flaky DB（embedding 查询） | **进程内同步纯函数** |
| 失败可观测 | doctor `reranker_health` | 完全不可观测 | 进程内 decision 字段，跨进程不可观测 |
| 与上游级联 | 独立 | 独立（不读 rerank_score） | **隐式契约级联**（依赖 rerank fail-open 时 rerank_score 为 undefined） |
| 魔法数字 | 无（reason 分类是枚举） | 0.7/0.3（**无 eval 依据**） | jumpRatio 0.2（**有 eval 依据**，PrecisionMemBench 校准） |

**三种姿态的分层逻辑**：

1. **有 flaky 外部依赖 + 失败需可观测** → rerank 模式（双层 try/catch + 审计）
2. **有 flaky 外部依赖 + 失败不需可观测** → cosineReScore 模式（单层 try/catch 静默吞）—— 这是**缺口**，与 rerank 不一致
3. **无 flaky 外部依赖（纯函数）** → autocut 模式（无 try/catch，多重 no-op 守卫）—— 这是**合理设计**，因依赖不会抛

**结论**：autocut 的"无 try/catch + 多重 no-op"是**与依赖类型匹配的合理姿态**，不是缺口。理由：

1. `scoreOf` 和 `preserve` 是调用方提供的同步纯函数，属性访问 `(x) => x.rerank_score` 不会抛
2. 多重 no-op 守卫覆盖所有"数据不可用"情况（开关关、数量不足、分数不足、top 不可用、无 cliff）
3. 唯一能抛的点是 `preserve?.(r)` 谓词（若调用方传错实现），但这是调用方 bug，不该由 autocut 兜底
4. `jumpRatio` 有 eval 依据（PrecisionMemBench），与 cosineReScore 的 0.7/0.3 形成对照——后者无依据才是缺口

**隐式契约的风险**：autocut 的级联 no-op 依赖 rerank fail-open 时 `rerank_score` 为 undefined。这是**未显式声明的契约**——hybrid.ts:1505-1506 注释提了一句，但代码层面无断言。若 rerank 实现改为填哨兵分数，autocut 会静默对哨兵做裁剪，产出错误结果。建议（超出本卡范围）：在 `applyReranker` 的 fail-open 路径加断言或注释明示"绝不设置 rerank_score 字段"。

## 来源证据（附录，不进正文）

| 项 | 位置 |
|----|------|
| applyAutocut 函数定义 | `src/core/search/autocut.ts:134-221` |
| 守卫一（开关/数量） | `src/core/search/autocut.ts:148` |
| 守卫二（有限分数不足，级联入口） | `src/core/search/autocut.ts:153-158` |
| 守卫三（top 不可用） | `src/core/search/autocut.ts:160-161` |
| 守卫四（无 cliff 越阈值） | `src/core/search/autocut.ts:170-193` |
| cliff 裁剪 + preserve 优先 | `src/core/search/autocut.ts:199-204` |
| kept.length === 0 兜底 | `src/core/search/autocut.ts:208` |
| applied 字段判定 | `src/core/search/autocut.ts:213-214` |
| noOp 辅助函数 | `src/core/search/autocut.ts:101-113` |
| jumpRatio 0.2 默认 + eval 依据 | `src/core/search/autocut.ts:30-31, 45` |
| minKeep 1 默认 | `src/core/search/autocut.ts:46` |
| JSDoc 说明读 rerank_score 不读 RRF/cosine | `src/core/search/autocut.ts:9-16` |
| 调用方上下文（scoreOf 传 rerank_score） | `src/core/search/hybrid.ts:1510-1518` |
| 调用方触发条件（offset === 0） | `src/core/search/hybrid.ts:1509` |
| 调用方注释说明级联 rerank fail-open | `src/core/search/hybrid.ts:1505-1506` |
| 调用方注释说明 alias_hop preserve（Codex P1） | `src/core/search/hybrid.ts:1514-1516` |
| AutocutDecision 接口六字段 | `src/core/search/autocut.ts:49-59` |
| DEFAULT_AUTOCUT 冻结对象 | `src/core/search/autocut.ts:43-47` |

---

*生成依据：《开源项目工程经验提炼提示词模板》场景 4C。待过场景五审查后入库。*

