# lean-plan

[中文版 Chinese version](README_CN.md)

Lean implementation planning for usage/per-request billed coding plans. Every request should buy progress: batch tool calls, route slices to the cheapest capable worker, plan once.

## Overview

Under per-request or usage-window billing, requests are the scarce resource. This skill cuts them without skipping work. Batching calls and picking cheap workers is how it saves; verification is never skipped, because a broken result costs more requests than the check would have.

## Core principles

1. **Batch.** One assistant turn is one API request, so independent tool calls share a turn. Read many files in one turn, write many in one turn, verify in one turn.
2. **Route cheap.** Give each slice to the cheapest worker that can do it. Default to fast workers (`sonic`, or `scout` for read-only); reasoning work stays in the main session.
3. **Decide once.** Plan fully in one pass, no confirmation loops, one batched verification.

### Worker grading

| Slice | Where | Why |
|---|---|---|
| Mechanical: boilerplate, config, docs, copy-paste | fast worker (`sonic`; `scout` for read-only) | ~3 requests per slice, low deduction |
| Needs judgment: logic, test interaction, argparse, edge cases | main session, do not spawn | a spawned worker costs 10+ requests plus orchestration and wait |
| A reasoning slice too big for the main session (a full subsystem) | good worker (`task`) | only then pay for full capability |

### When not to spawn workers

Estimate the single-agent batched cost first. Below 20 requests, do it yourself. Spawn workers only with 6+ independent file domains where each slice would cost 5+ requests in the main session.

## Installation

```bash
git clone https://github.com/adam-ikari/lean-plan.git ~/.agents/skills/lean-plan
```

Triggers: `coding plan`, `implementation plan`, `task breakdown`, `per-request billing`, `quota`, `5-hour window`, `save requests`, `batch calls`, `fast worker`, `good worker`.

## Benchmark data

Same task, same model, isolated workspaces in parallel. The only difference was whether lean-plan was followed. Requests and tokens are read from the `.message.usage` fields of the transcripts, not estimated.

### Task A (single package, textstats, 7 files)

| Metric | Without | With lean-plan |
|---|---|---|
| Requests | 20 | 9 |
| tokens | 583,279 | 276,813 |

### Task B (multi-module, textmon, 12 files)

| Metric | Without | With lean-plan |
|---|---|---|
| Requests | 15 | 10 |
| tokens | 325,363 | 311,204 |

### Findings

- Batching is where most of the saving comes from. Task A went from 20 requests to 9 (-55%) with tokens roughly halved; Task B from 15 to 10 (-33%). Writing seven files in one turn counts as one request.
- Fast workers are genuinely cheaper. A mechanical slice costs sonic about 3 requests; a reasoning slice costs task 13. That is why reasoning slices stay in the main session.
- The threshold matters. On Task B the first version spawned 4 workers for 39 requests, worse than the 15 of the natural run. After tuning (below ~20 requests means no workers) it correctly skipped workers and finished in 10.
- Agents underestimate their own request counts. One said 8 and used 9, another said 7 and used 10, but the measured data confirms the reduction.

### How it was measured

```bash
jq -s 'map(select(.type=="message" and .message.usage != null)) |
  {req: length, tok: (map(.message.usage.totalTokens)|add)}' <session>.jsonl
```

One usage record per assistant turn, meaning one API request.

Four runs: Baseline (natural), Lean2 (single package), Lean3 (multi-module, before the threshold), Lean4 (multi-module, after). Workspaces and specs are under `/tmp/abtest/`.

### Caveats

One run per side, so this is not a statistical result. Cost fields are all zero in this environment, so there are no dollar figures.
