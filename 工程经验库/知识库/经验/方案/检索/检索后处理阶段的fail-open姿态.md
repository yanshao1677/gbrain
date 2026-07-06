---
title: 检索后处理阶段的 fail-open 姿态
level: pattern
parent:
status: draft
tags:
  - fail-open
  - error-handling
  - retrieval
  - resiliency
  - audit
created_at: 2026-07-06
updated_at: 2026-07-06
confidence: high
related:
  - ../安全/远程调用Fail-Closed信任边界.md
source_repo: garrytan/gbrain
source_commit:
source_paths:
  - src/core/search/rerank.ts
  - src/core/search/hybrid.ts
  - src/core/search/relational-recall.ts
  - src/core/search/autocut.ts
  - src/core/rerank-audit.ts
---

# 检索后处理阶段的 fail-open 姿态

## 1. 问题

检索系统在 RRF 融合后会串接多个后处理阶段（重排、关系召回、自动截断、余弦重排等），每个阶段都可能失败：上游 LLM 服务 flaky、DB 瞬态故障、provider 返回畸形 shape、表结构缺失。

若任一阶段失败就让整个检索 throw，会导致：

- 单个 flaky upstream 让全库检索不可用
- 用户看到"查询失败"而非"质量略降的结果"
- 故障面从"一个阶段降级"扩大到"整条检索链路中断"

不解决会怎样：检索可用性绑定到每一个后处理阶段的可用性上，MTBF 等于所有阶段 MTBF 的倒数之和。

## 2. 适用约束

成立需要哪些前提：

- 检索链路有多个串接阶段，每个阶段是"锦上添花"而非"必需"
- 各阶段有明确的"输入不变"降级语义（重排失败→用 RRF 顺序、关系召回失败→空臂、截断失败→不截断）
- 团队能接受"质量降级但不中断"的语义，且有外部观测手段（审计行、doctor 检查）发现降级在发生
- 不适用于：阶段失败会让结果**错误**（而非**质量降级**）的场景。比如 source 隔离失败必须 fail-closed，因为返回错误 source 的结果是数据泄漏，不是质量降级

## 3. 核心思路

每个后处理阶段用 try/catch 包裹，失败时返回"输入不变"或"空贡献"，并写一条审计行供外部观测；检索链路永远继续往下走，绝不因后处理阶段失败而中断。审计写入本身也要包 try/catch，保证审计写失败不会反过来让检索中断。

## 4. 通用结构

| 角色 | 职责 |
|---|---|
| 阶段函数 | 接收 `T[]` 输入，返回 `T[]` 输出；失败时返回输入不变或空 |
| 外层 try/catch | 捕获阶段业务错，调审计写入，返回降级结果 |
| 审计写入器 | best-effort 写入 JSONL 文件，写失败只写 stderr 不抛 |
| 内层 try/catch | 包审计写入器调用，捕获审计自身抛的非预期错误 |
| 健康检查 | 读审计文件窗口内事件数，结合阶段 enabled 标志判断"健康"还是"降级" |

## 5. 处理流程

```mermaid
flowchart TB
  In[阶段输入 T[]] --> Try{try 阶段函数}
  Try --|成功| Out[阶段输出 T[]]
  Try --|抛错| OuterCatch[外层 catch]
  OuterCatch --> Classify[分类错误原因]
  Classify --> InnerTry{try 审计写入}
  InnerTry --|成功| InnerDone[审计已写]
  InnerTry --|抛错| InnerCatch[内层 catch 静默]
  InnerDone --> Degrade[返回降级结果]
  InnerCatch --> Degrade
  Degrade --> NextStage[下一阶段]
  Out --> NextStage
```

关键分支：

- 阶段成功 → 正常输出，进入下一阶段
- 阶段抛错 → 外层 catch → 写审计 → 返回降级结果（输入不变或空）→ 下一阶段
- 审计写也抛错 → 内层 catch 静默 → 仍返回降级结果 → 下一阶段

## 6. 异常处理

| 失败类 | 阶段行为 | 审计 | 是否中断检索 |
|---|---|---|---|
| 阶段业务错（auth/network/timeout/rate-limit/payload-too-large/unknown） | 返回降级结果 | 写 warn 行 | 否 |
| 阶段返回畸形 shape | 返回输入不变 | 否（视为业务错走审计） | 否 |
| 审计写入失败 | 不影响阶段结果 | 写 stderr | 否 |
| 早返回条件（输入空、配置关、无可处理项） | 直接返回 | 否 | 否 |

幂等性：fail-open 是无状态的，同一输入重试要么成功要么再写一条审计行，不会累积副作用。

## 7. 具体语言实现

本方案卡是容器，可运行的代码骨架在阶段1原子卡《rerank 双层 try/catch 兜住业务错与审计写错》中给出 TypeScript 实现。

## 8. 测试点

必须验证的关键行为：

- 阶段函数抛错时，检索链路返回的是输入不变而非 throw
- 审计写入器抛错时，检索链路仍返回降级结果而非 throw
- 早返回条件（输入空、配置关）不触发审计写入
- 审计行的字段完整（ts/model/reason/query_hash/doc_count/error_summary/severity）
- 健康检查能从审计文件窗口内事件数判断"降级在发生"

## 9. 适用场景 / 不适用场景

适用：

- 检索后处理（重排、关系召回、自动截断、余弦重排）
- 富化阶段（alias 解析、graph signals）
- 任何"质量增强"类阶段，失败时返回原结果是安全降级

不适用：

- 信任边界 enforcement（source 隔离、protected phase 校验）—— 必须 fail-closed，见已有方案《远程调用 Fail-Closed 信任边界》
- 写入路径（page upsert、fact 写入）—— 失败必须显式报告，不能静默吞
- 预算闸门 —— 超支必须 clean abort，不能"放行降级"

## 10. 风险与反模式

| 反模式 | 后果 | 对策 |
|---|---|---|
| 阶段 fail-open 但不写审计 | 降级在发生但无人知道 | 强制每个 fail-open 阶段配一个审计写入器 |
| 审计写入不在 try/catch 内 | 审计写失败（磁盘满、权限）反中断检索 | 内层 try/catch 兜底 |
| 阶段返回"部分结果"而非"输入不变" | 降级语义不明，下游难推断 | 严格"输入不变"或"空贡献"二选一 |
| 用 fail-open 处理信任边界错 | 安全漏洞（source 隔离失败时返回错误 source 的结果） | 信任边界永远 fail-closed，本方案不适用 |
| 健康检查只读事件数不读 enabled 标志 | 阶段关闭时"无事件"被误判为健康 | 健康检查先读 enabled，再解读事件数 |

## 11. 标签

fail-open, error-handling, retrieval, resiliency, audit, degradation, best-effort

---

## 附录：来源证据（仅供溯源核实，阅读正文无需依赖此节）

| 项 | 位置 |
|----|------|
| rerank fail-open 双层 try/catch | `src/core/search/rerank.ts:85-101` |
| rerank JSDoc 声明 fail-open 姿态 | `src/core/search/rerank.ts:10-13, 47-56` |
| 审计写入器 best-effort 语义 | `src/core/rerank-audit.ts:20, 82-91` |
| 审计事件字段定义 | `src/core/rerank-audit.ts:39-58` |
| cosineReScore 静默 fail-open（反例） | `src/core/search/hybrid.ts:1970-1973` |
| relational recall fail-open + 审计 | `src/core/search/relational-recall.ts:226-231` |
| autocut 纯函数 noOp 透传 | `src/core/search/autocut.ts:101-113, 148` |
| hybrid.ts 主路径调用顺序 | `src/core/search/hybrid.ts:1461-1463` |
