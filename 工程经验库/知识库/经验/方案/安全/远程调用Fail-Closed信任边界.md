---
title: 远程调用 Fail-Closed 信任边界
level: pattern
parent:
status: draft
tags:
  - security
  - trust-boundary
  - mcp
  - fail-closed
created_at: 2026-07-05
updated_at: 2026-07-05
confidence: high
related:
  - ../../架构/用统一操作契约为CLI与Agent工具提供单一真相源.md
  - ../../架构/用正交双轴模型分离数据库与内容仓库路由.md
  - 阶段1-读侧数据源作用域解析.md
source_repo: garrytan/gbrain
source_commit:
source_paths:
  - src/core/operations.ts
  - src/mcp/dispatch.ts
  - src/mcp/server.ts
  - src/commands/serve-http.ts
---

# 远程调用 Fail-Closed 信任边界

## 1. 问题

Agent 通过 MCP/HTTP 调用后端时，攻击面包括：越权读其他 tenant 数据、用 `__all__` 逃逸 source  grant、读 private 字段（markdown fence 内敏感行）、上传任意路径文件、调用仅运维可用的 admin op。

若 trust 默认宽松（falsy = trusted），任一 transport 漏设 flag 即全局降级。

## 2. 适用约束

- 存在 local CLI + remote Agent 双入口
- 个人/公司 brain 含 private 与 world 可见性分级
- OAuth scope 或 token permission 已建模
- 子 agent 循环内调用也视为 remote（auto-link 安全），但 submitter trust 用独立字段（如 `allowedSlugPrefixes`）

## 3. 核心思路

在 **OperationContext** 上显式携带 `remote: boolean`（TypeScript 必填）；**语义：仅 `remote === false` 为 trusted local**，其余（`true`、`undefined`、cast 绕过）一律按 untrusted 处理；dispatch 默认 `remote ?? true`。

## 4. 通用结构

```
Transport          remote default    scope check
─────────────────────────────────────────────────
Local CLI          false             skip OAuth
MCP stdio          true              optional scope N/A
MCP HTTP           true              hasScope(token, op.scope)
Subagent tool      true              + slug prefix / namespace rules
```

辅助字段（按域 threading）：

- `takesHoldersAllowList` — SQL `WHERE holder = ANY($list)`，MCP 默认 `['world']`
- `auth.allowedSources` — 联邦读
- `viaSubagent` — fail-closed：true 则强制 agent 命名空间策略，不依赖 subagentId 非空

## 5. 处理流程

1. Transport 构建 `DispatchOpts`，**显式**设置 `remote`
2. `buildOperationContext`：`remote: opts.remote ?? true`；`sourceId: opts.sourceId ?? 'default'`
3. HTTP：工具列表 `operations.filter(op => !op.localOnly)`
4. HTTP CallTool：`requiredScope = op.scope || 'read'` → `hasScope(auth.scopes, requiredScope)`，失败返回 `insufficient_scope`
5. Handler 内：
   - 读路径：`sourceScopeOpts(ctx)` / `resolveRequestedSourceScope(ctx, params)`
   - 敏感 markdown：`ctx.remote === true` → strip private fences
   - 文件上传：remote → strict path confine；local → loose
6. Per-call search mode：remote 未知 mode → reject；local 未知 → loud reject（防 silent downgrade）

## 6. 异常处理

- **metaHook 失败**：吸收错误，工具调用仍 success，仅无 `_meta`（不 flip 整 call）
- **Rerank 失败**（检索域）：fail-open 返回原序，audit 记录（与 trust 域 fail-closed 对比——检索降级可接受，权限不可）
- **OperationError**：JSON `{ error, message }` + `isError: true`，禁止 plain string

## 7. 具体语言实现（TypeScript 骨架）

```typescript
interface OperationContext {
  remote: boolean; // REQUIRED — no optional
  sourceId: string; // REQUIRED
  auth?: { scopes: string[]; allowedSources?: string[] };
  takesHoldersAllowList?: string[];
}

function isTrustedLocal(ctx: OperationContext): boolean {
  return ctx.remote === false;
}

function buildOperationContext(opts: { remote?: boolean; sourceId?: string }): OperationContext {
  return {
    remote: opts.remote ?? true,
    sourceId: opts.sourceId ?? 'default',
    // ...
  };
}

function enforceHttpScope(op: { scope?: string }, tokenScopes: string[]): void {
  const required = op.scope ?? 'read';
  if (!hasScope(tokenScopes, required)) {
    throw insufficientScope(required);
  }
}

function sourceScopeOpts(ctx: OperationContext): { sourceId?: string; sourceIds?: string[] } {
  const allowed = ctx.auth?.allowedSources;
  if (allowed && allowed.length > 0) return { sourceIds: allowed };
  if (ctx.sourceId) return { sourceId: ctx.sourceId };
  return {};
}
```

## 8. 测试点

- MCP stdio 无 token：`takesHoldersAllowList=['world']`，private takes 不可见
- Remote 传 `source_id: '__all__'` → 仅返回 token grant，非全库
- Remote federated grant 不含 explicit source_id → `permission_denied`
- `localOnly` op 不出现在 HTTP ListTools
- `ctx.remote === true` 时 get_page 响应无 private facts fence 行
- CLI `gbrain call` 路径 `remote === false` 可见 full fence

## 9. 适用场景 / 不适用场景

**适用**：MCP 暴露给 LLM、多租户 OAuth、personal knowledge 含 PII  
**不适用**：纯内网 single-user、无 remote 入口——可省略 scope 层但保留 source 过滤仍建议

## 10. 风险与反模式

- 用 `!ctx.remote` 代替 `ctx.remote === false` — undefined 变 trusted
- 在 handler 内手写 source filter，绕过 `sourceScopeOpts` — 漂移泄漏
- 把 subagent trust 绑在 `remote=false` — subagent 永远 remote=true，会全拒
- MCP 请求日志写 full params — PII 持久化；应 redact + allow-list keys + bucket bytes

## 11. 标签

security, fail-closed, mcp, trust-boundary, multi-tenant

---

## 附录：来源证据（仅供溯源核实，阅读正文无需依赖此节）

| 项 | 位置 |
|----|------|
| remote REQUIRED + fail-closed 语义 | `src/core/operations.ts:288-300` |
| buildOperationContext 默认 remote=true | `src/mcp/dispatch.ts:205` |
| MCP stdio remote=true + takes 默认 world | `src/mcp/server.ts:43-45` |
| HTTP localOnly 过滤 | `src/commands/serve-http.ts:1449` |
| HTTP hasScope  enforcement | `src/commands/serve-http.ts:1550-1551` |
| summarizeMcpParams redaction | `src/mcp/dispatch.ts:75-99` |
| resolvePerCallMode remote 分支 | `src/core/operations.ts:538-543` 注释 |
