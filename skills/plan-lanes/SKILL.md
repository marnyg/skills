---
name: plan-lanes
description: Interactive planning/spec'ing session that decomposes a goal, epic, or feature into a beads task graph ready for parallel execution — worker-shippable issues with acceptance criteria, read-first pointers, and declared write-sets, wired with truthful blocked-by edges and validated as parallel waves. Use before /skill:orchestrate when the user wants to plan or break down work for parallel agents — invoke on /skill:plan-lanes, "break this down for agents", "prep the task graph", "plan the next wave". Sibling to /skill:orchestrate (consumes the graph) and /skill:grill-design (use that first if the design itself is still unsettled).
---

# plan-lanes

Prepare a beads task graph good enough that `/skill:orchestrate` can
dispatch non-interactive workers against it without guessing. This
is an *interactive* session with the user — all ambiguity gets
resolved here (or parked as a spike), because workers cannot ask
questions mid-flight.

Planning is not designing. If the session keeps hitting open design
questions ("should schedules be global or place-embedded?"), stop
and run `/skill:grill-design` first; this skill assumes decisions
exist and turns them into an executable graph.

## Preconditions

```bash
bd ready            # beads db exists (bootstrap per session-start rules if not)
```

If the repo has the structured docs layout, read
`docs/desired-state/{goals,invariants,domain-model}.md` and any
relevant specs/ADRs before decomposing — issues will point workers
at these files.

## Phase 1 — Scope intake

1. Pin the unit being planned: a goal from `goals.md`, an epic, a
   feature request, or "the next wave" of an existing backlog.
2. `bd search` / `bd list` for existing overlapping issues — extend
   the graph, don't duplicate it. `bd dep tree <epic>` if an epic
   already exists.
3. List what is *known* (decided in ADRs/specs/decisions) vs
   *unknown*. Every unknown becomes either a question to the user
   now, or a `spike` issue — never a worker task.

## Phase 2 — Decompose into worker-shippable units

An issue is worker-shippable when a non-interactive agent could
complete it in one shot from the issue content plus the repo. Test
each candidate against:

- **Self-contained:** no decision left that the worker would have to
  guess. If describing the task requires "figure out whether...",
  it's a spike or needs a user decision now.
- **Checkable:** acceptance criteria a reviewer can verify
  mechanically (tests green, files exist, behavior demonstrated).
- **Sized for one lane-session:** roughly one focused agent run. Too
  big → split along test boundaries; too small → merge (tiny issues
  are coordination overhead, not parallelism).
- **Typed:** `task`/`feature`/`bug`/`chore` for workers; `spike` for
  unknowns (worked interactively, never dispatched); `epic` as the
  parent when planning a multi-wave outcome.

## Phase 3 — Annotate with the handoff contract

Use beads' native fields; `/skill:orchestrate` reads these to build
briefs:

```bash
bd create "<title>" -t task -l pi \
  -d "<what and why, 2-5 lines>" \
  --acceptance "<mechanically checkable criteria>" \
  --design "Read first: <paths to invariants/ADRs/specs>. \
Write-set: <files/dirs this issue may create or modify>. \
Must not touch: <explicit exclusions>." \
  --deps discovered-from:<epic-or-spike>
```

Rules:
- **Write-set is mandatory** and concrete (paths/globs, not "the
  backend"). It is the input to wave validation and lane
  partitioning. "TBD" is not a write-set.
- **Read first** points at repo paths, not summaries — workers have
  the full checkout.
- Echo every `bd create` before running (mutation discipline).

## Phase 4 — Wire the graph

- `bd dep add <B> --blocked-by <A>` **iff** B genuinely needs A's
  output. Both failure directions are real: missing edges cause
  collisions; fake edges (added "to be safe") serialize work that
  could run in parallel.
- Parent multi-wave plans under an `epic` issue; provenance via
  `discovered-from`.
- Ordering that exists only for merge convenience (not a real output
  dependency) is *not* an edge — it's lane assignment, decided at
  orchestration time.

## Phase 5 — Validate as waves

Simulate execution before handing over:

1. **Wave 1** = issues ready now (`bd ready` restricted to the
   plan). **Wave N+1** = issues unblocked once wave N closes.
2. Within each wave, check pairwise **write-set disjointness**. On
   overlap: add a real edge, merge the issues, or explicitly note
   they must share a lane.
3. Sanity-check wave width: >3 concurrent issues per wave exceeds
   the orchestrator's review bandwidth — that's fine (waves drain
   over multiple dispatch rounds), but say so.
4. Present the wave table (wave → issues → write-sets → blockers)
   and walk it with the user. Adjust until they sign off.

## Phase 6 — Hand over

- Fix anything the walkthrough surfaced; `bd dolt push` so the graph
  survives.
- Offer the baton: run `/skill:orchestrate` now, or record a
  `handover`-labeled note ("Graph planned for <epic>: wave 1 = X, Y.
  Dispatch via /skill:orchestrate.") for a future session.
- Do not claim issues here — claiming is the dispatcher's move.

## Failure modes to avoid

- **Design questions smuggled into tasks** — the worker will guess,
  and it will guess plausibly enough that the mistake survives
  review. Spike it or decide it now.
- **"TBD" write-sets** — wave validation silently becomes vacuous.
- **Over-decomposition** — ten 15-minute issues cost more in
  provisioning, review, and merges than two real ones.
- **Fake blocked-by edges** — every unnecessary edge deletes
  parallelism, which is the whole point.
- **Acceptance criteria a reviewer can't check mechanically** —
  "works well" is not reviewable; "PolFlowTest passes, tampered
  claim returns 403" is.
- **Planning against a stale graph** — always `bd search` before
  `bd create`; duplicates fork the truth.
