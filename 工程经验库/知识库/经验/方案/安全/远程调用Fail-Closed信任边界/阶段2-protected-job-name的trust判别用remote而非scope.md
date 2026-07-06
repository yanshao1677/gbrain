---
title: 阶段2-protected-job-name的trust判别用remote而非scope
level: atomic
parent: ../远程调用Fail-Closed信任边界.md
status: draft
tags:
  - security
  - trust-boundary
  - fail-closed
  - protected-job
  - minion-queue
  - defense-in-depth
created_at: 2026-07-06
updated_at: 2026-07-06
confidence: high
related:
  - 阶段1-读侧数据源作用域解析.md
  - ../../检索/检索后处理阶段的fail-open姿态/阶段1-rerank双层try-catch兜住业务错与审计写错.md
source_repo: garrytan/gbrain
source_paths:
  - src/core/operations.ts
  - src/core/minions/protected-names.ts
  - src/core/minions/queue.ts
  - src/core/cycle.ts
---

# 阶段2：protected job name 的 trust 判别用 `remote` 而非 `scope`

## 触发条件

`submit_job` op 被调用，且提交的 `name` 属于 `PROTECTED_JOB_NAMES` 集合。具体在 operations.ts:2817：

```ts
if (ctx.remote !== false && isProtectedJobName(name)) {
  throw new OperationError('permission_denied', `'${name}' jobs cannot be submitted over MCP (CLI-only for security)`);
}
```

`submit_job` op 的 `scope: 'admin'`（operations.ts:2802）。本阶段揭示的关键点：**op 层的 admin scope 检查通过后，仍有第二层 `ctx.remote === false` 校验拦截 protected job**。即一个持 admin scope 的 OAuth MCP caller，能提交普通 job，但提交不了 protected job。

## 输入字段

| 字段 | 类型 | 是否可选 | 来源 |
|---|---|---|---|
| `ctx.remote` | boolean | 必填 | transport 层设置（CLI=false, MCP/HTTP=true） |
| `ctx.auth.scopes` | string[] | 可选 | OAuth token 解析 |
| `p.name` | string | 必填 | 调用方传的 job 类型名 |
| `p.data` | object | 可选 | job payload |
| `trusted` | `{ allowProtectedSubmit: true } \| undefined` | 内部派生 | operations.ts:2825 计算，传给 queue.add |

## 判定规则

### 第一层：op scope 检查（operations.ts:2802 + HTTP dispatch）

```ts
// op 定义
{
  name: 'submit_job',
  scope: 'admin',  // ← op 声明 admin scope
  ...
}
```

- HTTP transport 在 dispatch 前调 `hasScope(auth.scopes, op.scope)`（serve-http.ts:1550-1551）
- admin scope 通过 → 进入 handler
- admin scope 不通过 → 返回 `insufficient_scope`，不进 handler

**关键**：scope 是"授权边界"（authorization），不是"信任边界"（trust）。admin scope 的 MCP caller 仍在 OS 之外，不被信任。

### 第二层：protected name + remote 校验（operations.ts:2817-2819）

```ts
if (ctx.remote !== false && isProtectedJobName(name)) {
  throw new OperationError('permission_denied', ...);
}
```

- `ctx.remote !== false`：fail-closed 语义。`true`、`undefined`、任何非 `false` 值都拦
- `isProtectedJobName(name)`：`name.trim()` 后查 `PROTECTED_JOB_NAMES` 集合（protected-names.ts:69-71）
- 两条件都满足 → throw `OperationError('permission_denied')`，不进 queue

### 第三层：trusted flag 派生（operations.ts:2823-2825）

```ts
const trusted = ctx.remote === false && isProtectedJobName(name)
  ? { allowProtectedSubmit: true }
  : undefined;
```

- 仅 `ctx.remote === false`（严格 `===`，非 `!ctx.remote`）且 name 是 protected → 设 `allowProtectedSubmit: true`
- 其他情况 → `trusted = undefined`
- 传给 `queue.add(name, data, opts, trusted)`（第 4 参数）

### 第四层：queue.add 二次校验（queue.ts:90-95）

```ts
const jobName = (name || '').trim();
if (jobName.length === 0) throw new Error('Job name cannot be empty');
if (isProtectedJobName(jobName) && !trusted?.allowProtectedSubmit) {
  throw new Error(`protected job name '${jobName}' requires CLI or operation-local submitter ...`);
}
```

- trim 规范化（防 `' shell '` 绕过，queue.ts:83-85 注释）
- protected name 且无 `allowProtectedSubmit` → throw
- **防御性冗余**：即使 op 层漏判，queue 层仍拦

### 例外：cycle 内 calibration 三联旁路 queue（cycle.ts:2019-2026）

```ts
const calibrationCtx = {
  engine,
  config: calibrationConfig,
  logger: { info() {}, warn() {}, error() {} } as never,
  dryRun,
  remote: false as const,  // ← 硬编码 trusted
  sourceId: calibrationSourceId,
} as never;
await runPhaseProposeTakes(engine, calibrationCtx, ...);
```

- cycle 跑 propose_takes / grade_takes / calibration_profile 时不经 minion queue，直接构造 `remote: false as const` ctx 调 phase 函数
- 理由（cycle.ts:2009-2011 注释）：cycle 是 operator CLI / autopilot daemon，OS 已是 trust 边界，不需再经 queue 的 trust 校验
- 这是"trusted caller 旁路 queue 时的 ctx 构造决断"——但本卡聚焦 submit_job 路径，calibration 旁路作为对照存在

## 状态读写位置

- **读**：`PROTECTED_JOB_NAMES`（`protected-names.ts:15-66`，`ReadonlySet<string>` 运行时常量，模块加载即定）；`ctx.remote` / `ctx.auth.scopes`（transport 注入）
- **写**：通过校验后 `queue.add` 向 `minion_jobs` 表插入一行；`trusted` flag **不持久化**，仅用于 submit 时 gate，入库后丢弃
- **不修改**：`PROTECTED_JOB_NAMES` 集合本身——纯常量模块，禁止 side effect（`protected-names.ts:10-13` 文件头注释明确）

## 正常路径（CLI 提交 protected job）

1. CLI 调 `submit_job`，`src/cli.ts` 构造 ctx 时 `remote: false`
2. op scope admin 检查：CLI 路径绕过 HTTP `hasScope`；进入 handler
3. 第一层（operations.ts:2817）：`ctx.remote !== false` → `false`，不拦
4. 第三层（operations.ts:2825）：`ctx.remote === false && isProtectedJobName(name)` → `trusted = { allowProtectedSubmit: true }`
5. 第四层（queue.ts:90）：`isProtectedJobName && !trusted?.allowProtectedSubmit` → `!true` = false，不拦
6. `queue.add` trim 后插入 `minion_jobs` 表，返回 job 行

## 分支路径

| 条件 | 行为 |
|------|------|
| `remote=false` + protected name | 通过两层，`trusted` set，入队成功 |
| `remote=true` + protected name | 第一层 throw `permission_denied`，不进 queue |
| `remote=undefined` + protected name | 第一层 throw（fail-closed：`undefined !== false` 为 true） |
| `remote=false` + 普通 name | 通过，`trusted = undefined`，入队 |
| `remote=true` + 普通 name | 通过（admin scope 已检查），`trusted = undefined`，入队 |
| `queue.add('shell', ...)` 直接调用无 `trusted` | 第四层 throw `protected job name 'shell' requires CLI ...` |
| `queue.add(' shell ', ...)` 直接调用无 `trusted` | 第四层 throw（trim 规范化后命中 protected） |
| HTTP transport ctx 字面量漏设 `remote` + protected name | 第一层 throw（fail-closed 兜底，正是 v0.36 修复的 RCE 缺口） |
| cycle calibration 三联 | 旁路 `submit_job` 与 `queue.add`，直接 `remote: false as const` ctx 调 phase 函数 |

## 失败处理

- 第一层 `OperationError('permission_denied')`：dispatch 包装为 JSON `{ error, message }` + `isError: true`，返回 MCP/HTTP 调用方，**不进 queue、不写库**
- 第四层 `throw new Error(...)`（普通 Error，非 OperationError）：op handler 内传播，dispatch 同样包装
- 两层**互不 catch**——任一失败都向上抛，无静默降级（与检索域 fail-open 对照：权限域不可降级）
- `GBRAIN_ALLOW_SHELL_JOBS` worker-side env flag 与本卡**正交**——即使该 flag on，MCP caller 仍过不了第一层（operations.ts:2810-2811 注释）

## 幂等性 / 一致性约束

- `PROTECTED_JOB_NAMES` 是 `ReadonlySet`，运行时不可变；`isProtectedJobName` 是纯函数（trim + has），同输入恒同输出
- op 层 `name.trim()`（operations.ts:2804）与 queue 层 `jobName.trim()`（queue.ts:86）**都 trim**——防 `' shell '` 在 op 层 trim 后命中 protected，到 queue 层因不 trim 而绕过（queue.ts:83-85 注释明确这个双 trim 的理由）
- `trusted` flag 仅在 submit 路径派生，不缓存、不跨调用复用——每次 `submit_job` 重新计算
- 严格 `=== false` 而非 `!ctx.remote`——防 `undefined` 被当 trusted（operations.ts:2813-2816 F7b 注释）
- queue.add 第四层是**防御性冗余**：即使 op 层漏判或绕过（如未来新增 in-process handler 直接调 `queue.add`），queue 层仍拦

## 代码骨架

```typescript
// submit_job handler（operations.ts:2803-2825 精简）
async (ctx, p) => {
  const name = typeof p.name === 'string' ? p.name.trim() : '';
  if (ctx.dryRun) return { dry_run: true, action: 'submit_job', name };

  const { isProtectedJobName } = await import('./minions/protected-names.ts');
  // 第一层：fail-closed，非严格 false 即拦
  if (ctx.remote !== false && isProtectedJobName(name)) {
    throw new OperationError('permission_denied',
      `'${name}' jobs cannot be submitted over MCP (CLI-only for security)`);
  }

  const queue = new MinionQueue(ctx.engine);
  // 第三层：trusted flag 仅 local CLI + protected 才 set
  const trusted = ctx.remote === false && isProtectedJobName(name)
    ? { allowProtectedSubmit: true }
    : undefined;

  return queue.add(name, p.data, opts, trusted);
}

// queue.add 第四层（queue.ts:86-95 精简）
const jobName = (name || '').trim();
if (jobName.length === 0) throw new Error('Job name cannot be empty');
if (isProtectedJobName(jobName) && !trusted?.allowProtectedSubmit) {
  throw new Error(`protected job name '${jobName}' requires CLI or operation-local submitter ...`);
}
```

## 最小验证清单

- `remote=true` + `name='shell'` → `permission_denied`，minion_jobs 表无新行
- `remote=false` + `name='shell'` + admin scope → 入队成功，job.name = 'shell'
- `remote=true` + `name='extract-atoms-drain'` → `permission_denied`（非 shell 但同 protected，防只盯 shell 的回归）
- `remote=undefined` + `name='shell'` → `permission_denied`（fail-closed 兜底）
- `remote=true` + `name='embed'`（普通 name） → 通过，`trusted = undefined`
- `queue.add('shell', data, opts)` 无第 4 参 → throw（防御性冗余）
- `queue.add(' shell ', data, opts)` 无第 4 参 → throw（trim 规范化生效）
- HTTP transport ctx 字面量漏设 `remote` + `name='shell'` → 第一层 throw（v0.36 RCE 修复回归点）
- cycle 跑 calibration_profile 不调用 `submit_job`、不写 minion_jobs 表（旁路验证）

---

## 附录：来源证据（仅供溯源核实，阅读正文无需依赖此节）

| 项 | 位置 |
|----|------|
| submit_job op 声明 admin scope | `src/core/operations.ts:2802` |
| 第一层 remote + protected 校验 | `src/core/operations.ts:2817-2819`（注释 :2807-2816 含 F7b fail-closed 理由） |
| trusted flag 派生 | `src/core/operations.ts:2825`（注释 :2823-2824） |
| PROTECTED_JOB_NAMES 集合（11 个 name） | `src/core/minions/protected-names.ts:15-66` |
| isProtectedJobName（trim + has） | `src/core/minions/protected-names.ts:69-71` |
| queue.add 第四层二次校验 | `src/core/minions/queue.ts:90-95`（trim 注释 :83-85） |
| cycle calibration 旁路 queue | `src/core/cycle.ts:2019-2026`（注释 :2009-2011） |
| HTTP hasScope admin 检查 | `src/commands/serve-http.ts:1550-1551` |
| 与 worker-side GBRAIN_ALLOW_SHELL_JOBS 正交 | `src/core/operations.ts:2810-2811` |
