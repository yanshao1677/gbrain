---
title: 阶段1-心跳间隔TTL除以6的6x余量
level: atomic
parent: ../DB锁的心跳续期与steal-grace设计.md
status: reviewed
tags:
  - db-lock
  - heartbeat
  - ttl
  - refresh-interval
  - tuning
created_at: 2026-07-06
updated_at: 2026-07-06
confidence: high
related:
  - ../检索/检索后处理阶段的fail-open姿态/阶段1-rerank双层try-catch兜住业务错与审计写错.md
source_repo: garrytan/gbrain
source_paths:
  - src/core/db-lock.ts
---

# 阶段1：心跳间隔 = TTL/6 的 6x 余量

## 触发条件

`withRefreshingLock` 被调用，计算 refresh 间隔。具体在 db-lock.ts:813-816：

```ts
const ttlMinutes = opts.ttlMinutes ?? DEFAULT_TTL_MINUTES;  // 默认 30
const heartbeatTimeoutMs = opts.heartbeatTimeoutMs ?? 30000;
// Refresh 6x per TTL window so a missed tick doesn't expire the lock.
const refreshIntervalMs = Math.max(15000, (ttlMinutes * 60 * 1000) / 6);
```

每次 `withRefreshingLock` 调用都触发，无额外前置条件。`withRefreshingLock` 是 sync / cycle / embed-backfill 等长任务的通用锁包装器。

## 输入字段

| 字段 | 类型 | 是否可选 | 来源 |
|---|---|---|---|
| `opts.ttlMinutes` | number | 可选，默认 30 | 调用方指定，如 sync 用默认 30min |
| `opts.heartbeatTimeoutMs` | number | 可选，默认 30000 | 调用方指定，refresh 单次超时 |

## 判定规则

### TTL 解析（db-lock.ts:813）

```ts
const ttlMinutes = opts.ttlMinutes ?? DEFAULT_TTL_MINUTES;
```

- `opts.ttlMinutes` 显式传则用之
- 否则用 `DEFAULT_TTL_MINUTES = 30`（db-lock.ts:34）

### 心跳间隔公式（db-lock.ts:816）

```ts
const refreshIntervalMs = Math.max(15000, (ttlMinutes * 60 * 1000) / 6);
```

拆解：

1. `(ttlMinutes * 60 * 1000)` —— TTL 转毫秒
2. `/ 6` —— 6 等分，每个 TTL 窗口 refresh 6 次
3. `Math.max(15000, ...)` —— 下限 15 秒，防止极短 TTL 算出过于频繁的 refresh

**6x 余量的语义**：30min TTL → 5min 间隔 → 一个 TTL 窗口内有 6 次 refresh 机会。漏 1 个 tick 不会过期（还剩 5 次机会），漏 5 个 tick 才会过期。

### 下限 15s 的作用

- 极短 TTL 场景：如 `ttlMinutes = 1`，算出 `(1 * 60 * 1000) / 6 = 10000ms`，下限拉到 15000ms
- 防止心跳过于频繁打爆 direct pool
- 但短 TTL + 15s 心跳会导致 6x 余量失效（1min TTL 用 15s 间隔，4 次 refresh 就过期）—— 这是下限的代价，短 TTL 场景 steal grace 会兜底（见方案卡）

## 状态读写位置

- **读**：`opts.ttlMinutes`（输入）、`DEFAULT_TTL_MINUTES`（常量）
- **写**：`refreshIntervalMs` 局部变量，传给 `setInterval`（db-lock.ts:823）
- **DB**：无（纯计算）
- **进程状态**：`setInterval` 句柄存于 `interval` 变量（:823），finally `clearInterval`（:856）

## 正常路径

1. 解析 ttlMinutes（:813）
2. 解析 heartbeatTimeoutMs（:814）
3. 算 refreshIntervalMs = `Math.max(15000, (ttlMinutes * 60 * 1000) / 6)`（:816）
4. `tryAcquireDbLock` 抢锁（:818）
5. 抢不到 throw `LockUnavailableError`（:819）
6. 抢到 → `setInterval(refreshCallback, refreshIntervalMs)`（:823）
7. `interval.unref?.()`（:851，不阻止进程退出）
8. `await work()`（:854）
9. finally：`clearInterval(interval)` + `handle.release()`（:855-857）

## 分支路径

### A. 默认 30min TTL

- `refreshIntervalMs = Math.max(15000, (30 * 60 * 1000) / 6) = Math.max(15000, 300000) = 300000ms = 5min`
- 6x 余量：5min 间隔，30min TTL，6 次 refresh 机会
- 漏 1 tick：还剩 5 次机会，TTL 不会过期

### B. 极短 TTL（如 1min）

- `refreshIntervalMs = Math.max(15000, (1 * 60 * 1000) / 6) = Math.max(15000, 10000) = 15000ms = 15s`
- 6x 余量失效：15s 间隔，1min TTL，4 次 refresh 机会
- 下限 15s 胜出，牺牲余量换 pool 压力可控

### C. 长 TTL（如 120min）

- `refreshIntervalMs = Math.max(15000, (120 * 60 * 1000) / 6) = Math.max(15000, 1200000) = 1200000ms = 20min`
- 6x 余量：20min 间隔，120min TTL，6 次 refresh 机会
- 余量保持，但单次 refresh 失败后要等 20min 才下个 tick—— steal grace 必须覆盖这个间隔

## 失败处理

本阶段是纯计算，无失败路径。但 `refreshIntervalMs` 的取值直接影响下游 refresh 的失败容错：

| 场景 | refreshIntervalMs | 6x 余量 | 漏 tick 容忍度 |
|---|---|---|---|
| 30min TTL（默认） | 5min | 有效 | 漏 5 个才过期 |
| 1min TTL | 15s（下限） | 失效 | 漏 3 个就过期 |
| 120min TTL | 20min | 有效 | 漏 5 个才过期 |

**关键约束**：`refreshIntervalMs` 与 `resolveStealGraceSeconds` 必须联动——steal grace = 2 × refreshIntervalSec（db-lock.ts:65-66）。改 refreshIntervalMs 必须同步改 steal grace，否则保护窗口错位。

## 幂等性 / 一致性约束

- **纯函数计算**：同一 `ttlMinutes` 永远算出同一 `refreshIntervalMs`
- **6x 余量是设计契约**：注释 db-lock.ts:815 "Refresh 6x per TTL window so a missed tick doesn't expire the lock" 明示这个 6 不是 magic number，是"漏一个 tick 不过期"的余量设计
- **下限 15s 不可移除**：移除会让短 TTL 算出过频繁的 refresh，打爆 direct pool
- **6 与 steal grace 的 2 的关系**：steal grace = 2 × refreshInterval = 2 × (TTL/6) = TTL/3。即死 holder 在 TTL 过期后还要等 TTL/3 才被偷。改 6 必须改 2，否则保护窗口与偷锁窗口错位
- **不能依赖的前提**：setInterval 在 Node.js/Bun 中不保证准时——事件循环忙时 tick 会延迟。6x 余量正是为此设计，但不保证 100% 不漏 tick（极端卡顿下仍可能漏 5 个）

## 代码骨架

```ts
const DEFAULT_TTL_MINUTES = 30;

export async function withRefreshingLock<T>(
  engine: BrainEngine,
  lockId: string,
  work: () => Promise<T>,
  opts: WithRefreshingLockOpts = {},
): Promise<T> {
  const ttlMinutes = opts.ttlMinutes ?? DEFAULT_TTL_MINUTES;
  const heartbeatTimeoutMs = opts.heartbeatTimeoutMs ?? 30000;
  // 6x 余量：一个 TTL 窗口 refresh 6 次，漏 1 个 tick 不会过期
  const refreshIntervalMs = Math.max(15000, (ttlMinutes * 60 * 1000) / 6);

  const handle = await tryAcquireDbLock(engine, lockId, ttlMinutes);
  if (!handle) throw new LockUnavailableError(lockId);

  let healthOk = true;
  const interval = setInterval(() => {
    void (async () => {
      try {
        const timeout = new Promise<never>((_, reject) =>
          setTimeout(() => reject(new Error('refresh_timeout')), heartbeatTimeoutMs)
        );
        await Promise.race([handle.refresh(), timeout]);
        healthOk = true;
      } catch (err) {
        const msg = err instanceof Error ? err.message : String(err);
        process.stderr.write(`[lock-refresh] ${lockId}: ${msg}; will retry next tick\n`);
        healthOk = false;
        // 不 clearInterval，下个 tick 重试
      }
    })();
  }, refreshIntervalMs);
  interval.unref?.();

  try {
    return await work();
  } finally {
    clearInterval(interval);
    try { await handle.release(); } catch { /* idempotent */ }
    if (!healthOk) {
      process.stderr.write(`[lock-refresh] ${lockId}: completed with degraded heartbeat\n`);
    }
  }
}
```

## 最小验证清单

可执行断言：

1. `opts.ttlMinutes` 未传时，`ttlMinutes === 30`（DEFAULT_TTL_MINUTES）
2. `opts.ttlMinutes = 30` 时，`refreshIntervalMs === 300000`（5min）
3. `opts.ttlMinutes = 1` 时，`refreshIntervalMs === 15000`（下限 15s 生效，非 10000）
4. `opts.ttlMinutes = 120` 时，`refreshIntervalMs === 1200000`（20min）
5. `opts.ttlMinutes = 0.1`（6 秒）时，`refreshIntervalMs === 15000`（下限生效）
6. `refreshIntervalMs` 永远 ≥ 15000
7. 30min TTL 时，`(ttlMinutes * 60 * 1000) / refreshIntervalMs === 6`（一个 TTL 窗口 6 次 refresh）
8. setInterval 的回调被 `refreshIntervalMs` 调度（不是别的值）
9. `interval.unref?.()` 被调用（防 timer 阻止进程退出）
10. 联动测试：`resolveStealGraceSeconds(30) === 600 === 2 * (300000/1000)`（steal grace = 2 × refreshIntervalSec）
11. 联动测试：`resolveStealGraceSeconds(1) === 60`（下限 60s 生效，因 refreshSec = max(15, 10) = 15，15×2=30 < 60）

## 设计依据（本卡的核心价值）

**为什么是 6，不是 3 或 12？**

6x 余量的设计目标：**漏 1 个 tick 不过期，且 refresh 频率不对 direct pool 产生压力**。

- **3x 余量**（如 30min TTL → 10min 间隔）：漏 1 个 tick 还剩 2 次机会，但 10min 间隔意味着 refresh 失败后要等 10min 才重试—— steal grace 要拉到 20min，死锁自愈慢
- **6x 余量**（30min TTL → 5min 间隔）：漏 1 个 tick 还剩 5 次机会，5min 间隔对 direct pool 压力可接受，steal grace = 10min 自愈合理
- **12x 余量**（30min TTL → 2.5min 间隔）：容错更强但 refresh 频率翻倍，direct pool 压力加倍，收益递减

6 是"容错足够 + 压力可控"的折中。代码注释 db-lock.ts:815 没解释为什么是 6 而非 3/12，只说"so a missed tick doesn't expire the lock"——这暗示"漏 1 个 tick 不过期"是硬约束。3 倍也满足此硬约束（漏 1 个还剩 2 次机会），6 倍额外提供了"漏 2-5 个 tick 仍不过期"的容错，代价是 refresh 频率比 3 倍翻倍。选择 6 而非 3，是偏向容错；选择 6 而非 12，是偏向 pool 压力。

**证据不足项**：未找到 PR 或 commit 显式讨论"为什么是 6 而非 3 或 12"。6 的选择可能是经验值。需查 git log `db-lock.ts` 的 `refreshIntervalMs` 历史确认是否有量化依据。

## 来源证据（附录，不进正文）

| 项 | 位置 |
|----|------|
| withRefreshingLock 主函数 | `src/core/db-lock.ts:807-864` |
| ttlMinutes 解析 | `src/core/db-lock.ts:813` |
| refreshIntervalMs 公式（6x 余量） | `src/core/db-lock.ts:815-816` |
| 6x 余量注释 | `src/core/db-lock.ts:815` |
| DEFAULT_TTL_MINUTES = 30 | `src/core/db-lock.ts:34` |
| setInterval 调度 | `src/core/db-lock.ts:823` |
| interval.unref (#1633) | `src/core/db-lock.ts:848-851` |
| 联动：resolveStealGraceSeconds = 2× refreshSec | `src/core/db-lock.ts:64-66` |
| heartbeatTimeoutMs = 30000 默认 | `src/core/db-lock.ts:814` |
| Promise.race refresh + timeout | `src/core/db-lock.ts:836-839` |

---

*生成依据：《开源项目工程经验提炼提示词模板》场景 4C。待过场景五审查后入库。*
