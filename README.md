# lean-plan

面向按次 / 用量计费编码计划的精简实现规划技能。让每个请求都买到进展：批量合并工具调用，把切片交给最便宜的 worker，计划一次成型。

Lean implementation planning for usage/per-request billed coding plans. Every request should buy progress: batch tool calls, route slices to the cheapest capable worker, plan once.

## 简介 / Overview

按次计费或用量窗口下，稀缺资源是请求。这个技能减少请求数，但不靠跳过工作来省：省请求的办法是合并调用、选便宜的 worker，验证不能省，坏结果会花更多请求补。

Under per-request or usage-window billing, requests are the scarce resource. This skill cuts them without skipping work. Batching calls and picking cheap workers is how it saves; verification is never skipped, because a broken result costs more requests than the check would have.

## 核心原则 / Core principles

1. 批量。一次 assistant 回合就是一次 API 请求，所有独立的工具调用都放进同一回合。读多个文件、写多个文件、验证，各用一次回合完成。
   Batch. One assistant turn is one API request, so independent tool calls share a turn. Read many files in one turn, write many in one turn, verify in one turn.
2. 路由便宜。切片交给最便宜的 worker。默认 fast worker（`sonic`，只读用 `scout`），需要推理的活留在主会话。
   Route cheap. Give each slice to the cheapest worker that can do it. Default to fast workers (`sonic`, or `scout` for read-only); reasoning work stays in the main session.
3. 决定一次。计划一次成型，不反复确认，验证一次做完。
   Decide once. Plan fully in one pass, no confirmation loops, one batched verification.

### Worker 分级 / Worker grading

| 切片难度 Slice | 在哪跑 Where | 原因 Why |
|---|---|---|
| 纯机械：样板、配置、文档、复制粘贴 / boilerplate, config, docs, copy-paste | fast worker（`sonic`；只读 `scout`） | 约 3 请求一片，抵扣系数低 / ~3 requests, low deduction |
| 需判断：逻辑、测试交互、argparse、边界 / logic, test interaction, argparse, edge cases | 主会话批量做，不派 / main session, do not spawn | 派出去要 10+ 请求加编排加等待，主会话批量同价零开销 / a spawned worker costs 10+ requests plus orchestration and wait |
| 大到主会话装不下的推理片（完整子系统）/ a reasoning slice too big for the main session | good worker（`task`） | 才值得付全能力 / only then pay for full capability |

### 防滥用阈值 / When not to spawn workers

先估算单代理批量成本。低于 20 请求就自己做完，不派 worker。要派 worker，得有至少 6 个互不相干的文件域，且每个切片在主会话里做也要花 5 个以上请求。

Estimate the single-agent batched cost first. Below 20 requests, do it yourself. Spawn workers only with 6+ independent file domains where each slice would cost 5+ requests in the main session.

## 安装 / Installation

```bash
git clone https://github.com/adam-ikari/lean-plan.git ~/.agents/skills/lean-plan
```

触发词 / Triggers: `coding plan`、`implementation plan`、`task breakdown`、`多步实现`、`按次计费`、`省调用`、`调用次数`、`合并调用`、`批量调用`、`减少调用`、`限额`、`配额`、`fast worker`、`good worker`、`抵扣系数`。

## 实验数据 / Benchmark data

同一任务、同模型、隔离工作区并行跑，唯一差别是是否执行 lean-plan。请求数和 tokens 从转录的 `.message.usage` 字段实采，不是估算。

Same task, same model, isolated workspaces in parallel. The only difference was whether lean-plan was followed. Requests and tokens are read from the `.message.usage` fields of the transcripts, not estimated.

### 任务 A / Task A — 单包 textstats，7 文件

| 指标 Metric | 无 skill Without | 使用 lean-plan With |
|---|---|---|
| 请求数 Requests | 20 | 9 |
| tokens | 583,279 | 276,813 |

### 任务 B / Task B — 多模块 textmon，12 文件

| 指标 Metric | 无 skill Without | 使用 lean-plan With |
|---|---|---|
| 请求数 Requests | 15 | 10 |
| tokens | 325,363 | 311,204 |

### 结论 / Findings

- 批量合并是主要的收益来源。任务 A 请求从 20 降到 9（-55%），tokens 差不多减半；任务 B 从 15 降到 10（-33%）。一个回合写完 7 个文件只算一次请求。
  Batching is where most of the saving comes from. Task A went from 20 requests to 9 (-55%) with tokens roughly halved; Task B from 15 to 10 (-33%). Writing seven files in one turn counts as one request.
- fast worker 确实便宜。sonic 处理纯机械切片大约 3 个请求，task 处理推理切片要 13 个。所以推理切片留在主会话。
  Fast workers are genuinely cheaper. A mechanical slice costs sonic about 3 requests; a reasoning slice costs task 13. That is why reasoning slices stay in the main session.
- 阈值有用。任务 B 第一次按「3 个文件域」就派 worker，4 个 worker 花了 39 个请求，比自然跑法的 15 个还多。改成「估算低于 20 不派」之后，正确选择不派 worker，10 个请求完成。
  The threshold matters. On Task B the first version spawned 4 workers for 39 requests, worse than the 15 of the natural run. After tuning (below ~20 requests means no workers) it correctly skipped workers and finished in 10.
- 代理会低估自己的请求数。自己估 8 实际 9，估 7 实际 10，但实测数据确认下降存在。
  Agents underestimate their own request counts. One said 8 and used 9, another said 7 and used 10, but the measured data confirms the reduction.

### 测量方法 / How it was measured

```bash
jq -s 'map(select(.type=="message" and .message.usage != null)) |
  {req: length, tok: (map(.message.usage.totalTokens)|add)}' <session>.jsonl
```

一条带 usage 的记录就是一个 assistant 回合，也就是一次 API 请求。

One usage record per assistant turn, meaning one API request.

4 次运行：Baseline（自然）、Lean2（单包）、Lean3（多模块，调阈值前）、Lean4（多模块，调阈值后）。工作区和规格留在 `/tmp/abtest/`。

Four runs: Baseline (natural), Lean2 (single package), Lean3 (multi-module, before the threshold), Lean4 (multi-module, after). Workspaces and specs are under `/tmp/abtest/`.

### 局限 / Caveats

每组只有一次运行，不是统计结论。本环境 token 成本字段全为 0，给不出美元数字。

One run per side, so this is not a statistical result. Cost fields are all zero in this environment, so there are no dollar figures.
