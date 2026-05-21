---
name: docs-update
description: Updates the structured docs/ tree after a meaningful unit of work — refreshes handoff.md and focus.md, prunes notes.md, evaluates exploration-log.md entries for deletion when ADRs land, prompts for new ADRs on significant decisions, and checks the change against invariants. Use after completing a feature, hitting a milestone, before opening a PR, when the user says "wrap up" / "update docs", or when invoked as /skill:docs-update.
---

# docs-update

Heuristic + manual trigger. Maintains the `day-to-day/` and `technical/adrs/` parts of the docs tree so cross-session memory stays fresh.

## When to run

Reach for this skill when **any** of these is true:

- A meaningful unit of work just completed (feature, non-trivial bug fix with investigation, milestone reached).
- About to open a PR.
- User signals end-of-session: "we're done", "wrap up", "stash this", "update docs".
- User explicitly invokes: `/skill:docs-update`.

**Skip** if the work was trivial (typo fix, dependency bump, doc-only edit, single-line refactor). Triviality is the agent's judgment, but err on the side of skipping — empty updates create noise.

## Prerequisites

The repo must have the `docs/` layout from `docs-scaffold`. If `docs/day-to-day/` does not exist, stop and tell the user to run `/skill:docs-scaffold` first.

## Steps

### 1. Load state

Read:
- `docs/desired-state/invariants.md` (and any sub-scope invariants for paths touched this session) — needed for the contradiction check
- `docs/desired-state/goals.md` — for the focus.md goal link
- `docs/day-to-day/handoff.md`, `focus.md`, `notes.md`, `exploration-log.md` — current state
- `docs/technical/adrs/` — list existing ADRs (filenames are enough)
- `git diff` / `git log` since the last handoff timestamp (or last session) — what actually changed
- For monorepos: identify if work touched sub-scopes with their own `docs/`. If yes, you will repeat steps 2–4 in each affected scope.

**Parent–child invariant sanity check.** While loading invariants from multiple scopes, scan for obvious contradictions between a sub-scope invariant and a parent invariant (e.g. parent says "all auth uses SSO", sub-scope says "this service uses API keys"). Semantic comparison is hard — don't be exhaustive, just flag anything that looks contradictory at a glance:

> "Possible split-brain invariants: parent `<path>` says '<X>', sub-scope `<path>` says '<Y>'. These look contradictory. Reconcile before continuing — invariants are invariant."

Do not proceed until resolved.

### 2. Contradiction check (do this first — it can halt the rest)

Scan the diff against every relevant `invariants.md`. If any change appears to violate an invariant:

> **STOP.** Report the apparent violation to the user:
> - Which invariant: quote it.
> - Which change: file + line range.
> - Ask: "Is the invariant wrong (update `desired-state/invariants.md`) or is this change wrong (revert / revisit)?"

Do not proceed with the rest of the update until the user decides. Silent resolution is forbidden.

### 3. Update `day-to-day/handoff.md`

**Overwrite** the file. Sections:

- **Last session** — 1–3 bullets of what was actually done. Reference specific files/PRs where useful.
- **Loose threads** — things half-done, deferred decisions, open questions, anything that would surprise the next session.
- **Suggested next steps** — 1–3 concrete things the next session can pick up immediately.

Keep total under ~50 lines. If you find yourself writing more, that content probably belongs in `notes.md`, an ADR, or a guide.

### 4. Update `day-to-day/focus.md`

Read current `focus.md`. If the focus is unchanged, skip. If it has shifted (the work has moved to a new feature/area/goal), **replace** with:

- **Now:** one line — current focus.
- **Toward goal:** one line — which `desired-state/goals.md` entry this serves. If no goal fits, that's a signal — either add a goal or question whether this work belongs.
- **Out of scope:** 1–2 bullets explicitly excluding nearby work.

Total ~20 lines max.

### 5. Update `day-to-day/notes.md`

Two passes:

**Append:** Any operational weather discovered this session — current-state quirks, environment oddities, in-flight migrations, "don't touch X right now" warnings. Each entry begins with `YYYY-MM-DD — `.

Skip things that belong elsewhere:
- Durable rules → `AGENTS.md`
- Landed architecture → ADR
- How something works → `technical/guides/`

**Prune:** For each existing entry older than 30 days, add a trailing `<!-- stale? -->` comment if not already present. Do **not** delete — the user prunes.

### 6. Update `day-to-day/exploration-log.md`

Two passes:

**Resolve:** For each existing section, check if a corresponding ADR has landed (look in `docs/technical/adrs/`). If yes, ask:

> "ADR-NNNN appears to resolve the exploration for '<topic>'. Delete that section from the exploration log?"

If user confirms, delete the section.

**Append:** If this session involved strategy-level pivots (tried an approach, ruled it out, switched), add entries. Granularity bar: *would another agent re-attempt this in a future session if it wasn't recorded?* If no, don't log it.

Format per entry: `- YYYY-MM-DD — Tried <X>. Ruled out: <reason>.` (or "Considered", "Paused", "Landed on").

### 7. Detect significant decisions → prompt for ADR

A "significant decision" means: a choice between non-trivial alternatives that has broad reach (architecture, protocol, library with API surface, design pattern affecting multiple modules). Library version bumps and tool swaps don't count.

If a significant decision was made this session:

> "This session made a significant decision: <one-line summary>. Draft an ADR?"

If yes:
- Compute next ADR number: max of existing `NNNN-*.md` filenames + 1, zero-padded to 4 digits.
- Create `docs/technical/adrs/NNNN-kebab-title.md` in MADR format (template below).
- Pre-fill `Considered Options` from the matching exploration-log section if one exists.
- Set Status to `Proposed`. User changes to `Accepted` when ready.

### 8. Recurse into sub-scopes

If the diff touched files under a sub-scope with its own `docs/` (e.g. `services/auth/docs/`), repeat steps 2–7 against that sub-scope's `day-to-day/` and `technical/`. Only recurse into scopes actually touched.

### 9. Budget check

For each scope whose `desired-state/` you loaded this session, count total lines across `goals.md`, `invariants.md`, `domain-model.md`. If any scope exceeds **~300 lines**, surface:

> "Scope `<path>` desired-state is <N> lines (budget ~300). Consider pruning or splitting before it becomes noisy to load every session."

Also flag if you traversed more than ~3 scope levels deep to reach the work — that's a signal the tree may be over-nested.

Don't auto-prune. Surface and let the user decide.

### 10. Report

Print a tight summary:
- Files updated (path + 1-line reason)
- Files left alone (and why, if non-obvious)
- Contradictions found (if any — should already have halted at step 2 or step 1's parent–child check)
- ADRs drafted
- Sub-scopes recursed into
- Budget warnings (from step 9)
- Anything that needs human attention (stale notes pruning, exploration-log deletions awaiting confirmation)

## MADR ADR template

```markdown
# ADR-NNNN: <Short title in sentence case>

- Status: Proposed
- Date: YYYY-MM-DD
- Supersedes: <ADR-NNNN if applicable, else omit>

## Context and Problem Statement

<!-- 2-4 sentences. What is the problem? Why are we deciding now? -->

## Decision Drivers

<!-- The forces. Constraints, invariants, goals this decision must respect. -->

- <!-- driver -->
- <!-- driver -->

## Considered Options

<!-- Each option gets a sub-section. Pre-fill from exploration-log if available. -->

### Option A: <name>

<!-- One paragraph description. -->

- Pros: <!-- ... -->
- Cons: <!-- ... -->

### Option B: <name>

- Pros: <!-- ... -->
- Cons: <!-- ... -->

## Decision Outcome

Chosen: **<Option X>**, because <!-- key reason tied to the drivers -->.

### Consequences

- <!-- What becomes easier -->
- <!-- What becomes harder -->
- <!-- What follow-up work this implies -->

### Confirmation

<!-- How will we know this decision was right? What would invalidate it? -->
```

## Failure modes to avoid

- **Empty updates.** If nothing meaningfully changed, don't write. Skip the skill.
- **Silent invariant violations.** Never just "fix" a contradiction — always surface it.
- **Tasks in handoff.** Handoff is what happened + what's next. It is not a task list. Tasks belong in the user's tracker.
- **Decision drift.** If you can't tie current `focus.md` to a goal, surface that — don't invent a goal.
- **Logging tool-level pivots.** "Used X instead of Y" at the tool level is noise. Only log strategy-level dead ends.
