---
name: lean-plan
description: |
  Lean implementation planning built for usage/per-request billed coding plans
  (5-hour windows like Claude Pro/Max, per-request billing, quota plans). Use
  whenever a task needs a coding plan, multi-step implementation, or task
  breakdown. Minimize billed requests by batching tool calls into single API
  requests and routing each slice to the cheapest capable worker; never skip
  work or verification to save a request. Works standalone, or supplements
  heavier planning skills when the user explicitly wants a full written plan
  document. Triggers: coding plan, implementation plan, task breakdown,
  multi-step implementation, 按次计费, 按次数计费, 用量计费, 省钱, 省调用,
  调用次数, 合并调用, 批量调用, 减少调用, 限额, 配额, 5-hour window,
  usage-based billing, quota, fast worker, good worker, 抵扣系数.
---

# Lean Plan

Plan and execute coding work so billed requests buy progress. The scarce
resource is **requests** (per-request billing, quota/usage windows): finish
the work with the fewest requests, but **never skip work or verification to
save one** — a broken result costs more requests later.

Three levers, in priority order:
1. **Batch** — one assistant turn = one API request. Put every independent
   tool call in the same turn. This is the biggest lever.
2. **Route cheap** — assign each slice to the cheapest worker that can do it.
   Default to fast workers.
3. **Decide once** — plan fully in one pass; no confirmation loops; one
   batched verification.

## 0. When NOT to use this skill

- User asked for a detailed written plan document or iterative confirmation.
- Task is trivially small (single file, single step) — batching still helps,
  but skip the worker machinery entirely.
- User is exploring/experimenting and expects back-and-forth dialogue.
- High-risk changes (production data migration, security-sensitive code) need
  staged review.

When in doubt, default to lean-plan.

## 1. Planning — one pass, no confirmation loop

- Entire plan in ONE response: task list + file-ownership map +
  worker-grade assignment + verification step. No "does this look right"
  stops between sections.
- No separate plan document unless the user asked for one. The todo list +
  this message IS the plan.
- Adjust only if a hard error surfaces during execution.

## 2. When to ask vs. decide (the decision ladder)

Ask ONLY when a choice is (a) materially different tradeoffs the user must
weigh AND (b) not resolvable from repo conventions/docs/history. If you must
ask, ask ALL questions in ONE `ask` call — never one at a time.

Defaults (pick the most conservative standard option, do NOT bounce back):
- Ambiguous requirement → read code/docs; match existing patterns.
- Multiple designs → simplest that satisfies behavior; reuse existing code.
- New dependency → refused; stdlib / installed / vendored.
- Abstraction/extra layer → YAGNI.
- Error handling → root cause fix, never suppress/special-case.
- Docs → update only affected existing docs; no new doc files unless asked.

## 3. Execution

### 3.1 Batch tool calls — one turn = one API request (the #1 lever)

Every assistant turn you emit is ONE billed request, no matter how many tool
calls it carries. **Put every independent tool call in the same turn.**

- **Reads:** all files you need now get read in ONE turn. Reading 10 files
  in one turn costs one request; reading them in ten turns costs ten.
- **Full-doc reads use a range selector** (`file.md:1-200`): a bare `read`
  returns a structural summary, and each follow-up page fetch it forces costs
  one wasted request. If a read still comes back truncated, fetch ALL
  missing ranges in the SAME turn, never serially.
- **Probes:** environment checks (interpreter version, git state, tool
  availability) join the first reads turn, never their own turns.
- **Writes:** independent file writes/edits go in ONE turn.
- **Verification:** build + affected tests + smoke in ONE turn (one bash,
  `&&`-joined). Never pytest in one turn and smoke in the next.
- **Fixes:** one failure costs exactly two turns — ALL edits for the defect
  batched in one turn, then ONE re-verify turn. Never edit → verify →
  edit → verify.
- **Questions:** every clarification in a single `ask` call (§2).

### 3.2 Worker grading — route cheap, prefer fast workers

Workers are a cost, not a default. Each worker costs ~3 requests fixed
(spawn + write + self-verify) plus churn risk; the main session also pays
orchestration + wait time. Spawn ONLY when the win is provable.

**Threshold — do the arithmetic first:** estimate the single-agent batched
cost. If it is below ~20 requests, do it yourself with batching — NO workers.
Spawn workers only when ALL hold:
- ≥6 genuinely independent file domains, AND
- each slice is large enough that its ~3-request worker fixed cost amortizes
  (it would cost ≥5 requests done in the main session), AND
- the slices are truly independent (disjoint files, contracts stated).

**Grading:**

| Slice difficulty | Where it runs | Why |
|---|---|---|
| Truly mechanical: boilerplate, config/data, docs, copy-paste edits | **fast worker** (`sonic`; `scout` for read-only) | Cheap (~3 requests), low deduction coefficient. |
| Needs judgment: logic, tests interacting with code, argparse wiring, edge cases | **main session, batched — do NOT spawn** | A spawned good worker costs ~10+ requests + orchestration + wait; main can do it batched for the same cost with zero overhead. |
| A reasoning slice so big the main session can't absorb it (a full subsystem) | **good worker** (`task`) | Only then pay for full capability. |

- **When in doubt, keep it in the main session.** Do NOT spawn a good worker
  for a slice the main session could batch. Do NOT send a reasoning slice to
  a fast worker — it will fail or churn, costing more than doing it in main.
- Merge trivial slices into one worker rather than spawning one per file.
- Each worker task is self-contained in ONE message: Target (exact
  files/symbols, explicit non-goals) + Change (step-by-step) + Acceptance
  (observable result). State the shared contracts in the batch context;
  workers never negotiate them.
- One worker = one file-ownership domain. Never two workers on the same file.


### 3.3 Worker resilience (flaky/throttled upstreams)

- Workers never commit; the main session commits.
- If a worker hits rate limits/throttle (402/429): stop immediately, report
  current state, end the turn. No spinning retries.
- Worker state report (when stopping early): completed files, owned-file
  list, in-flight change, blocker, contracts respected.

## 4. Verification — one batched pass, never zero

- Each worker verifies its own slice (compile/run) before reporting back.
- Main session runs ONE batched verification over the merged result — build +
  affected tests + smoke, all tool calls in one turn. Never split it into
  serial re-checks.
- Never skip verification to save a request. An unverified "done" costs more
  requests later.

## 5. Commit — main session only

- Workers never commit. Main session groups changes by domain and commits
  once per domain.
- Commit message reflects what actually changed; no filler.

## 6. Cache-preserving execution

Prompt cache reuses the stable prefix: system prompt + history up to the last
turn. Every request in a stable session pays only for the delta (measured 83%
cache hits for a clean single-session run vs 30% for one with churn). Hit
rate is a byproduct of turn count — more turns mean a longer cached prefix —
so never add turns to raise it. Batching (§3.1) stays the #1 lever; these
rules cost zero extra requests. Preserve the prefix:

- **One session per task, run to completion.** Never compact, /clear, or hand
  off mid-task — each reset re-sends the whole context cold (0% reuse).
- **Freeze the tool surface.** Do not mount/unmount MCP servers, load/unload
  skills, or start/stop LSP between turns. Tool schemas are part of the
  prefix; any change invalidates it.
- **Fewer workers = more cache.** A spawned worker starts a fresh context
  with zero reusable prefix. This reinforces the §3.2 threshold: below ~20
  requests, single-session batching both saves requests and keeps hits high.
- **Read once, in order.** Fetch spec/context early, batched (§3.1), and do
  not re-read the same file later unless it changed. Identical reads ride the
  cache; changed tool results are normal appends.
- **Workers get context from the task text, not by reading.** Subagent reads
  of long files truncate and force serial re-reads — each pays fresh input (a
  run re-read a 165-line skill across 4 turns). Embed needed content in the
  worker prompt; workers never re-fetch long docs.
- **Keep volatile data in tool results, not in your plan text.** Timestamps,
  git status, env snapshots go at the end of the turn. Content your later
  turns repeat must be stable or the prefix breaks.
