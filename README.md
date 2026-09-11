# lean-plan

[中文版 Chinese version](README_CN.md)

Lean implementation planning for usage/per-request billed coding plans. Fewer requests, same work.

## What it is

Under per-request or usage-window billing, requests are the scarce resource. This skill reduces them without skipping work. Batching calls and picking cheap workers is how it saves; verification is never skipped, because a broken result costs more requests than the check would have.

## Features

- One assistant turn is one API request, so independent tool calls share a turn. Read many files in one turn, write many in one turn, verify in one turn.
- Mechanical slices go to fast workers (`sonic`, or `scout` for read-only). Reasoning slices stay in the main session; a full subsystem that overflows the session goes to a good worker (`task`).
- Workers have a threshold. Estimate the single-agent batched cost first; below 20 requests, do it yourself. Otherwise workers need 6+ independent file domains where each slice would cost 5+ requests in the main session.
- Planning is one pass: full task list, file-ownership map, worker-grade assignment, verification step. No confirmation loops, one batched verification, one commit.
- Prompt cache stays warm. One session per task, a fixed tool set, and few workers keep the cached prefix intact, so later requests pay mostly for the delta. Measured hit rate ranged from 30% on a churny run to 73% on a clean single-session run.

> [!NOTE]
> Workers are a cost, not a default. A spawned worker costs about 3 requests fixed plus churn risk; a misgraded reasoning slice can burn 10+.


## Installation

Recommended: install for all agents with the skills CLI.

```bash
npx skills add adam-ikari/lean-plan -g -y
```

The CLI writes to `~/.agents/skills` (universal) and symlinks into each agent's skills directory. Pass `-a '*'` to force all agents, or `-a <agent>` for a single one.

Manual install per agent, clone into the agent's skills directory:

| Agent | Directory |
|---|---|
| Claude Code | `~/.claude/skills/lean-plan` |
| Codex | `~/.codex/skills/lean-plan` |
| Cursor | `~/.cursor/skills/lean-plan` |
| Gemini CLI | `~/.gemini/skills/lean-plan` |
| Kilo Code | `~/.kilocode/skills/lean-plan` |
| Roo Code | `~/.roo/skills/lean-plan` |
| Windsurf | `~/.windsurf/skills/lean-plan` |
| Amp, Antigravity, Cline, OpenCode, Warp, others | `~/.agents/skills/lean-plan` |

```bash
git clone https://github.com/adam-ikari/lean-plan.git ~/.claude/skills/lean-plan
```

The skill triggers on: `coding plan`, `implementation plan`, `task breakdown`, `per-request billing`, `quota`, `5-hour window`, `save requests`, `batch calls`, `fast worker`, `good worker`.

## Usage

1. Plan the whole task in one response: task list, file-ownership map, worker-grade assignment, verification step. No confirmation loop.
2. Batch every independent tool call into the same turn. Never emit a single tool call alone when siblings are ready.
3. Spawn workers only if the threshold says they pay off. Grade each slice by difficulty, default to fast workers.
4. Verify once over the merged result: build plus affected tests plus a smoke run, all in one turn.
5. Commit once per domain.

> [!WARNING]
> Never skip verification to save a request. An unverified "done" costs more requests later.

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
- Cache hits follow the same rules. A clean single-session run reached 73% cache reads, worker-heavy or churny runs fell to 30 to 57%. Fewer sessions, no mid-task tool changes, fewer workers, and no re-reads all keep the prefix warm.

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
