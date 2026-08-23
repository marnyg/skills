---
name: orchestrate
description: Act as an orchestrator that fans repo work out to parallel worker agents in herdr-managed git worktrees, tracks lanes via beads, reviews and merges their output, and reports back. Use when the user asks to parallelize work, spin up multiple agents, act as an orchestrator/coordinator, or run tasks in parallel lanes — invoke on /skill:orchestrate, "spin up agents", "work these in parallel", "orchestrate this". Requires herdr (HERDR_ENV=1) and bd; degrade to single-lane work if either is missing. Sibling to /skill:plan-lanes, which prepares the task graph this skill dispatches.
---

# orchestrate

One session (this one) plays **orchestrator**: it owns the beads
graph, lane allocation, review, merges, and the final push. Worker
agents run in herdr-managed git worktrees, never touch `bd` or
`git push`, and report back via `REPORT.md` + lifecycle state.

Proven shape: 2–3 lanes, each a `pi` agent in its own worktree
branch, dispatched with a self-contained brief, polled until
settled, reviewed, merged `--no-ff`, cleaned up.

## Preconditions (check, don't assume)

```bash
test "${HERDR_ENV:-}" = 1        # inside a herdr pane
herdr status                     # server running, compatible
bd ready                         # beads db exists
```

If herdr is unavailable, say so and fall back to doing the work
serially yourself. If `bd` is missing, lanes can still run but
tracking degrades to the report files.

Herdr CLI syntax drifts; when unsure run the command group bare
(`herdr agent`, `herdr worktree`) — never bare `herdr` (launches the
TUI) and never probe mutating subcommands by omitting args.

## Phase 1 — Plan lanes (before touching anything)

If the issues were prepared by `/skill:plan-lanes`, they carry the
handoff contract in native beads fields — `acceptance` (verifiable
done-criteria), `design` (Read first / Write-set / Must not touch) —
and the dep graph is already validated as waves: lane partitioning
is mechanical and briefs mostly write themselves from `bd show`.
Otherwise, derive the same information now:

1. `bd ready` + `bd dep list` the candidates. **Fix lies in the
   graph first**: if issue B practically needs issue A's output but
   shows ready, add `bd dep add B --blocked-by A`. Parallel dispatch
   off a lying ready-queue is how agents collide.
2. Partition into lanes with **disjoint write-sets** (e.g. code lane
   vs docs/spec lane, or separate modules). If two candidate issues
   touch the same files, same lane or defer one.
   For sprawling backlogs, consider pausing to run `/skill:plan-lanes`
   with the user instead of inferring write-sets solo.
3. Cap at 2–3 lanes. More lanes = more review debt; the orchestrator
   is the bottleneck, and that's the point.
4. Present the lane table (issue(s), branch, deliverable, worker
   kind/model) and **get user confirmation before spawning** —
   workers cost money and mutate the repo.

## Phase 2 — Provision

Per approved lane:

```bash
bd update <issue> --claim
herdr worktree create --cwd "$REPO" --branch agent/<lane> --base main \
  --label "lane X: <desc>" --no-focus
# parse .result.root_pane.pane_id from the JSON
herdr agent start <lane>-worker --kind pi --pane <pane-id>
```

Rules:
- One branch per lane, named `agent/<lane>`.
- Never commandeer existing panes/agents you didn't create without
  explicit user permission.
- `--no-focus` everywhere; the user keeps their focus.

## Phase 3 — Brief and dispatch

Workers run non-interactively from your perspective and **cannot ask
you questions mid-flight**. The brief must be self-contained:

- The task + explicit acceptance criteria (tests green, files
  produced).
- What to read first (invariants, domain model, relevant ADRs/specs
  — point at repo paths; the worker has the full checkout).
- **Hard rules block** (verbatim, adapt paths):
  - work only in this worktree on branch `agent/<lane>`
  - commit with clear messages; do NOT `git push`
  - do NOT use `bd`/beads at all — orchestrator owns tracking;
    explicitly override any repo AGENTS.md instructions about bd
    and about pushing at session end
  - state what the lane must NOT touch (e.g. "no edits under
    `docs/`" for the code lane) — this is what keeps merges clean
  - when done, write `REPORT.md` at the worktree root: status, what
    was built, test output summary, key decisions, open questions

Dispatch with `herdr agent prompt <name> "<brief>"` — **without**
`--wait` (lanes run tens of minutes; poll instead).

## Phase 4 — Monitor

Poll every ~30 s until a lane settles:

```bash
herdr agent list   # filter your workers; states: working|idle|done|blocked|unknown
```

- `done`/`idle` → review (Phase 5).
- `blocked` → `herdr agent read <name> --source recent-unwrapped
  --lines 120`, inspect the dialog, **escalate to the user** before
  answering it.
- `unknown` does not prove completion — read the pane.
- Peeking mid-flight with `agent read` is cheap and catches derailed
  workers early (e.g. one fighting the environment instead of the
  task).

## Phase 5 — Review and integrate (per finished lane)

Trust nothing in the report until verified:

1. In the worktree: read `REPORT.md`, `git log main..agent/<lane>`,
   `git diff --stat`.
2. **Independently re-run the acceptance check** (tests, build) in
   the worktree.
3. Check the diff against `docs/desired-state/invariants.md` and the
   lane's must-not-touch list.
4. Merge: `git merge --no-ff agent/<lane> -m "Merge lane X: ..."`.
5. Capture the report's essence as `bd note <issue> "..."`, then
   drop `REPORT.md` from main (it's a lane artifact, not repo docs).
6. Re-run the full test suite on merged main after the last code
   lane lands.

Merge docs-only lanes as they finish; they can't break code lanes.

## Phase 6 — Cleanup and wrap-up

```bash
herdr worktree remove --workspace <ws-id>   # per lane
git branch -d agent/<lane>                  # merged branches only
git worktree prune
```

Then the normal session close: file follow-up issues from the
workers' open-questions sections (`--deps discovered-from:<issue>`),
close lane issues **with user confirmation**, run `/skill:docs-update`
if the repo has the docs layout, and push (`git push` + `bd dolt
push` where applicable).

## Failure modes to avoid

- **Dispatching off an unfixed dependency graph** — fix `blocked-by`
  edges first; `bd ready` must be truthful before it drives
  allocation.
- **Workers with overlapping write-sets** — disjointness is decided
  at planning time, not discovered at merge time.
- **`agent prompt --wait` on long lanes** — it times out; poll
  `agent list` instead.
- **Trusting the worker's own test claims** — always re-run.
- **Letting workers touch bd or push** — repo AGENTS.md often
  mandates both at session end; the brief must explicitly override.
- **Silently answering a blocked worker's approval dialog** —
  inspect and escalate.
- **Leaving worktrees/branches behind** — cleanup is part of the
  workflow, not optional.
