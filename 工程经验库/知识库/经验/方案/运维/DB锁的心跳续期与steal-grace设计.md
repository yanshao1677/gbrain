---
title: DB 锁的心跳续期与 steal grace 设计
level: pattern
status: reviewed
tags:
  - db-lock
  - heartbeat
  - ttl
  - steal-grace
  - distributed-lock
  - resumability
  - operations
created_at: 2026-07-06
updated_at: 2026-07-06
confidence: high
source_repo: garrytan/gbrain
source_paths:
  - src/core/db-lock.ts
  - src/commands/sync.ts
---

# DB 锁的心跳续期与 steal grace 设计

## 问题

长任务（批量导入、cycle 后台循环、embed backfill）需要跨进程跨主机的互斥锁，防止两个 worker 抢同一资源（如同一 source 的 sync）。DB 锁是自然选择——已有 Postgres/PGLite，无需引入 Redis。但 DB 锁有三个失效模式必须同时处理：

1. **Holder 崩溃**：进程被 SIGKILL / OOM / 断电，没释放锁 → 锁永远占着，资源被无限期阻塞
2. **Holder 活着但 GC 暂停 / 事件循环卡顿**：CPU 密集型 import 让 setInterval 心跳漏 tick，TTL 过期 → 竞争者偷走活锁，两个 worker 同时跑
3. **DB 池耗尽**：Supavisor transaction pool 打满（EMAXCONNSESSION），心跳写不进去 → 等同于第 2 种，活锁被偷

朴素方案各有缺陷：

- 短 TTL + 高频心跳：漏一个 tick 就过期，第 2 种失效频发
- 长 TTL + 不偷锁：第 1 种失效无法自愈
- 心跳走 transaction pool：第 3 种失效无解
- TTL 过期即可偷：第 2 种失效时偷活锁

## 适用约束

- DB 是 Postgres 或 PGLite（gbrain 双引擎parity要求）
- 长任务跑几十分钟到几小时，期间事件循环可能被 CPU 工作占满
- DB 可能通过 Supavisor transaction-pooler 暴露，池有上限
- 跨主机部署（多 worker、多容器），不能用进程内锁
- 锁的偷取必须保守——偷活锁（false positive）比留死锁（false negative）代价更高，因为偷活锁导致两个 worker 同时写同一资源

## 核心想法

**四要素联动：TTL + 心跳 + heartbeat-aware steal grace + direct pool**。

1. **TTL 是最终兜底**：锁行带 `ttl_expires_at`，过期后可被竞争者 ON CONFLICT 抢占。保证 holder 崩溃后最终能自愈。
2. **心跳间隔 = TTL/6**：30min TTL → 5min 心跳。6x 余量保证漏一两个 tick 也不会过期。
3. **steal grace = 2 × 心跳间隔**：偷锁不仅看 TTL，还看 `last_refreshed_at`。即使 TTL 过期，只要 holder 在最近 2 个心跳周期内 refresh 过，就不偷——保护"活但被短暂饿死"的 holder。
4. **心跳走 direct pool**：绕开 transaction pooler，避免池耗尽时心跳写不进去。

设计公式（对 30min 默认 TTL）：

```
TTL = 30min
心跳间隔 = max(15s, TTL/6) = max(15s, 5min) = 5min
stealGrace = max(心跳间隔 × 2, 60s) = max(10min, 60s) = 10min
```

语义：holder 每 5min refresh 一次；若某次 refresh 漏了，TTL 在 30min 后过期，但因为 `last_refreshed_at` 在 10min 内（2 个心跳周期），偷锁的 ON CONFLICT WHERE 条件不满足，不会被偷；只有 holder 真的死了（连续 10min+ 没 refresh），才允许偷。

## 通用结构

```ts
// 配置
const TTL_MINUTES = 30;  // 兜底
const REFRESH_INTERVAL_MS = Math.max(15000, (TTL_MINUTES * 60 * 1000) / 6);  // 6x 余量
const STEAL_GRACE_SECONDS = Math.max(
  Math.floor(Math.max(15, (TTL_MINUTES * 60) / 6) * 2),
  60
);  // 2 个心跳周期

// 抢锁（INSERT ON CONFLICT，带 steal grace 守卫）
async function tryAcquire(engine, lockId, ttlMinutes) {
  const stealGrace = resolveStealGraceSeconds(ttlMinutes);
  // INSERT ... ON CONFLICT DO UPDATE
  //   WHERE ttl_expires_at < NOW()
  //     AND (last_refreshed_at IS NULL OR last_refreshed_at < NOW() - stealGrace)
  // 返回行 = 抢到；0 行 = 有人占着
}

// 心跳（setInterval，走 direct pool，不清 clearInterval on transient fail）
async function withLock(engine, lockId, work) {
  const handle = await tryAcquire(engine, lockId, TTL_MINUTES);
  if (!handle) throw new LockUnavailableError(lockId);

  let healthOk = true;
  const interval = setInterval(() => {
    void (async () => {
      try {
        await Promise.race([
          handle.refresh(),  // 走 engine.executeRawDirect，绕开 transaction pool
          timeout(30_000),
        ]);
        healthOk = true;
      } catch (err) {
        stderr(`[lock-refresh] ${lockId}: ${err.message}; will retry next tick`);
        healthOk = false;
        // 不 clearInterval！下个 tick 继续重试，TTL 是兜底
      }
    })();
  }, REFRESH_INTERVAL_MS);

  try {
    return await work();
  } finally {
    clearInterval(interval);
    try { await handle.release(); } catch { /* idempotent */ }
    if (!healthOk) stderr(`[lock-refresh] ${lockId}: completed with degraded heartbeat`);
  }
}
```

## 流程

```mermaid
flowchart TB
  Start[tryAcquireDbLock] --> Insert[INSERT ON CONFLICT WHERE ttl_expires_at < NOW AND last_refreshed_at < NOW - stealGrace]
  Insert --> Rows{返回行?}
  Rows --|0 行| Null[返回 null → LockUnavailableError]
  Rows --|有行| Handle[返回 DbLockHandle]
  Handle --> SetInt[setInterval refresh 每 TTL/6]
  SetInt --> Work[执行 work]
  Work -->|每 tick| Race[Promise.race refresh, 30s timeout]
  Race -->|成功| HealthTrue[healthOk = true]
  Race -->|失败| HealthFalse[healthOk = false, 不 clearInterval, 下 tick 重试]
  Work -->|完成| Finally[finally: clearInterval + release]
  Work -->|抛错| Finally
  HealthFalse -.TTL 兜底.-> Stolen[死 holder 被 ON CONFLICT 偷]
```

## 异常处理矩阵

| 失效模式 | 检测 | 处理 | 数据后果 |
|---|---|---|---|
| Holder 崩溃（SIGKILL/OOM） | `ttl_expires_at` 过期 + `last_refreshed_at` 老于 stealGrace | 竞争者 ON CONFLICT 偷锁 | 旧 holder 若复活已无锁，可能重复跑（依赖业务侧幂等） |
| Holder 活但事件循环卡顿 | TTL 过期但 `last_refreshed_at` 新于 stealGrace | **不偷**（heartbeat-aware 保护） | 无 |
| DB transaction pool 耗尽 | refresh 抛 EMAXCONNSESSION | catch + healthOk=false，下 tick 重试 | 无（走 direct pool，不依赖 transaction pool） |
| Refresh 单次超时（>30s） | `Promise.race` timeout | catch + healthOk=false，下 tick 重试 | 无 |
| Refresh 持续失败到 TTL 过期 | `last_refreshed_at` 老于 stealGrace | 竞争者偷锁 | 同 holder 崩溃 |
| Release 抛错 | finally catch | 吞错（idempotent，注册的 cleanup 会兜底） | 锁行可能残留，TTL 过期后自愈 |

## 测试要点

- TTL=30min 时，refreshIntervalMs === 300000（5min），stealGraceSeconds === 600（10min）
- TTL=1min 时，refreshIntervalMs === 15000（下限 15s 生效），stealGraceSeconds === 60（下限 60s 生效）
- `GBRAIN_LOCK_STEAL_GRACE_SECONDS=120` 时，resolveStealGraceSeconds 返回 120（env 覆盖）
- 抢锁 SQL：holder 活（last_refreshed_at 新）时 INSERT ON CONFLICT 返回 0 行
- 抢锁 SQL：holder 死（last_refreshed_at 老）时 INSERT ON CONFLICT 返回 1 行
- refresh 走 `engine.executeRawDirect`，不走 transaction pool
- refresh 单次失败不清 clearInterval
- release 抛错被吞，不传播
- `interval.unref()` 被调用（不阻止进程退出）
- finally 里 clearInterval + release 都执行

## 风险与权衡

- **心跳走 direct pool 的前提**：引擎必须暴露 `executeRawDirect` 接口。PGLite 无 pool 概念，direct = 普通 query；Postgres 走独立 session pool。若引擎没这个接口，本设计退化为"心跳走 transaction pool"，第 3 种失效无解。
- **steal grace 是 2× 心跳间隔的经验值**：保护 2 个心跳周期的容错。若事件循环卡顿超过 2 个心跳周期（10min），仍会被偷。对 CPU 密集型任务可能不够，但拉长会延长死锁自愈时间。
- **偷锁后旧 holder 复活**：旧 holder 若从长时间 GC 中恢复，已无锁，可能重复跑。本设计不解决这个——依赖业务侧幂等（sync 的 checkpoint 让重复跑是安全的）。
- **TTL 30min 默认的取舍**：长任务（几小时）用 30min TTL 合理；短任务（几秒）用 30min TTL 会在崩溃后等 30min 才自愈。调用方可通过 `opts.ttlMinutes` 覆盖。
- **不门控 read pool 健康度**：v0.42.x #1794 之前，代码先 `SELECT 1` 探 read pool 健康度，失败就 clearInterval。这恰恰是池耗尽时让活锁被偷的根因——心跳本该是"证明我活着"，不应被"DB 暂时病了"阻止。v0.42.x 移除此门控。

## 来源证据（附录，不进正文）

| 项 | 位置 |
|----|------|
| withRefreshingLock 主函数 | `src/core/db-lock.ts:807-864` |
| refreshIntervalMs = TTL/6 公式 | `src/core/db-lock.ts:815-816` |
| 心跳 setInterval + 不 clearInterval on fail | `src/core/db-lock.ts:823-847` |
| v0.42.x #1794 注释（refresh 即心跳，走 direct pool） | `src/core/db-lock.ts:826-835` |
| Promise.race refresh + 30s timeout | `src/core/db-lock.ts:836-839` |
| catch 块 healthOk=false 不 clearInterval | `src/core/db-lock.ts:841-845` |
| interval.unref (#1633) | `src/core/db-lock.ts:848-851` |
| finally clearInterval + release 吞错 | `src/core/db-lock.ts:855-863` |
| DEFAULT_TTL_MINUTES = 30 | `src/core/db-lock.ts:34` |
| DEFAULT_STEAL_GRACE_SECONDS = 600 | `src/core/db-lock.ts:56` |
| resolveStealGraceSeconds 函数 | `src/core/db-lock.ts:58-67` |
| steal grace = 2× refreshSec 公式 | `src/core/db-lock.ts:64-66` |
| steal grace JSDoc（#1794 heartbeat-aware） | `src/core/db-lock.ts:45-55` |
| tryAcquireDbLock ON CONFLICT WHERE | `src/core/db-lock.ts:208-221` (postgres), `:257-271` (pglite) |
| handle.refresh 走 executeRawDirect | `src/core/db-lock.ts:231-243` |
| handle.release + registerCleanup | `src/core/db-lock.ts:223-228, 244-250` |
| syncLockId 命名 `gbrain-sync:${sourceId}` | `src/core/db-lock.ts:722` |

---

*生成依据：《开源项目工程经验提炼提示词模板》场景 4B。待过场景五审查后入库。*
