---
name: grill-design
description: Adversarial design-discovery session for an in-progress plan, architecture, or domain model. Walks the design tree branch by branch, cross-references the codebase, and captures findings into the docs/ layout (glossary terms, invariants, ruled-out alternatives, ADRs) as they crystallize. Use when the user wants to stress-test or pin down a design before coding, says things like "stress-test this plan", "walk me through this design", "help me design X", "pin down the domain model", or invokes /skill:grill-design. Sibling skill to grill-me — same interrogation discipline, but produces durable artifacts.
---

# grill-design

Design-discovery interrogation that writes findings into `docs/` as decisions crystallize. Sibling to `grill-me`: same interrogation technique, different output.

## When to run

Reach for this when the user is **designing something they haven't built yet** and wants to pin it down before coding. Signals:

- "Stress-test this plan / design / approach"
- "Walk me through this design"
- "Help me design X"
- "Pin down the domain model for X"
- "What am I missing in this plan?"
- `/skill:grill-design`

**Distinguishing from `grill-me`:**
- `grill-me/Critique` — adversarial review of a *fixed* artifact (existing plan, code, PR). Pure conversation.
- `grill-me/Socratic` — sharpen the user's articulation of a concept they already hold. Pure conversation.
- `grill-design` — interactive design *discovery* on a plan still taking shape. Produces docs.

If the user just wants holes poked in something already-written, use `grill-me`. If they're building a design and want it captured as they go, use this.

## Interrogation discipline (from grill-me)

These rules are load-bearing — do not skip:

1. **Read first.** Look at the actual artifact (plan, existing code, related docs) before asking anything. Generic questions are worthless.
2. **One question at a time.** Wait for the answer before moving on.
3. **Read code instead of asking** whenever a question can be answered from the codebase. The user shouldn't have to tell you what `auth.ts` already does.
4. **Press on weak answers.** "It'll be fine" / "kind of like X" → push for specifics.
5. **Distinguish lazy hand-waving from genuine uncertainty.** If the user says "I don't know — that's what I'm trying to figure out," that's a signal to build the answer together, not press.
6. **No checklist-dumping.** Pick the sharpest question, not the next item on a list.
7. **No sycophancy, no pedantry, no lecturing.** Direct, skeptical, respectful.
8. **Supply your recommended answer.** Unlike `grill-me/Socratic`, here you act like a senior engineer with opinions. Format:
   > **Q — <question>.** My recommendation: **<X>**, because <reasoning>. Push back if I'm wrong.
   
   The user is the decision-maker. You're not withholding; you're proposing.

## Pre-flight: docs scaffold check

Before starting, check whether `docs/` (with our scaffold layout) exists at the repo root:

- **If yes** → proceed in **docs-attached mode**: writes happen inline per the table below.
- **If no** → proceed in **scratch mode**: accumulate findings in memory throughout the session. At wrap-up, offer to run `/skill:docs-scaffold` and write the accumulated findings into the freshly-scaffolded layout. A design session is the ideal moment to bootstrap docs — the user is producing exactly the kind of structured information that fills them.

Either way, the interrogation itself runs identically.

## Session shape

1. **Read first.** Load: relevant code, existing `docs/desired-state/*` if present, the plan/artifact the user is bringing.
2. **Identify the design tree.** What decisions are downstream of what? Roughly: domain shape → invariants/constraints → architecture → implementation. Pick a starting branch with the user.
3. **Walk one branch.** Resolve dependencies one question at a time. For each question:
   - State the question.
   - State your recommendation and reasoning.
   - Wait for the user's response.
   - Press on weak answers.
   - Capture findings inline per the table below (or accumulate in scratch mode).
4. **Branch checkpoint.** Once a branch is resolved (or productively stalled), **stop and summarize**:
   > "On this branch we've pinned: X, Y, Z. Ruled out: A, B. Open: C. Next branch or stop here?"
   
   The user decides. Don't auto-continue.
5. **Repeat** for the next branch, or stop.
6. **Wrap up** (see below).

## Inline write rules

The skill's job is to capture findings as they crystallize. Triggers and destinations:

| Finding | Write trigger | Destination | Cadence |
|---|---|---|---|
| Glossary term | A domain term gets a canonical name + definition the user confirms | `docs/desired-state/domain-model.md` (glossary section) | **Inline** |
| Invariant | User confirms "this is always true across the project" (NOT just for this feature) | `docs/desired-state/invariants.md` | **Inline**, but only after explicit "Confirm as a project-wide invariant?" prompt |
| Ruled-out alternative | An option is explicitly ruled out with a stated reason | `docs/day-to-day/exploration-log.md` | **Inline** when the rule-out happens |
| Decision (ADR-worthy) | A decision lands AND meets all three criteria below | `docs/technical/adrs/NNNN-*.md` | **End of session**, drafted as `Proposed` |
| Handoff / focus updates | Session ending | `docs/day-to-day/handoff.md`, `focus.md` | **Defer** — suggest `/skill:docs-update` at wrap-up |

In **scratch mode** (no docs scaffold), accumulate these in memory in the same structure; flush at wrap-up after scaffolding.

### ADR three-criteria gate (from grill-with-docs)

Only draft an ADR if **all three** are true:

1. **Hard to reverse** — cost of changing your mind later is meaningful.
2. **Surprising without context** — a future reader will wonder "why did they do it this way?"
3. **Real trade-off** — there were genuine alternatives and one was picked for specific reasons.

If any of the three is missing, do not draft an ADR. A small or obvious decision goes in the glossary, invariants, or just the handoff.

### Invariant vs feature-decision

Easy to confuse. Ask before writing to `invariants.md`:

> "You said `<X>`. Is that a property that must hold for the *whole project* (invariant), or only for *this feature* (decision)?"

Project-wide → invariants. Feature-specific → glossary entry or ADR. Default: ask, never assume.

## Cross-referencing rules

- **Glossary conflict.** If the user uses a term that conflicts with an existing definition in `domain-model.md`, surface it immediately:
  > "Glossary defines '<term>' as <X>, but you seem to mean <Y>. Which is right?"
- **Code-vs-claim mismatch.** If the user states how something works and the code disagrees, surface it:
  > "You said cancellation is partial, but `orders.ts:N` cancels the whole order. Which is right?"
- **Invariant violation.** If the design under discussion would violate an existing invariant, halt and escalate — same rule as `docs-update`. Don't resolve silently.
- **Fuzzy language.** If the user uses an overloaded term, propose a precise canonical one:
  > "You're saying 'account' — do you mean Customer or User? Those are different entities."

## Branch checkpoints

After each design-tree branch is resolved:

1. **Summarize**: pinned / ruled-out / open items, briefly.
2. **Ask**: "Next branch or stop?"
3. **Honor the answer.** Do not push for more branches if the user wants to stop.

This is the stop criterion — prevents runaway sessions.

## Wrap-up

When the session ends (user stops, or branch tree exhausted):

1. **If scratch mode**: offer to run `/skill:docs-scaffold` and write the accumulated findings into the new layout. Be explicit about what will be written where.
2. **Draft ADRs** for any decisions that passed the three-criteria gate. Pre-fill `Considered Options` from the exploration-log entries on the same topic. Status: `Proposed`. User promotes to `Accepted` separately.
3. **Suggest** `/skill:docs-update` to refresh `handoff.md` and `focus.md` with the session's state.
4. **Final summary**: print what was written where (glossary entries added, invariants added, exploration-log sections touched, ADRs drafted, sub-scopes affected).

## Self-verification

Before each branch checkpoint summary AND before wrap-up, **audit your own grilling** against this checklist. If any item fires, fix it before continuing or finishing. The user's agreeableness and your own are both failure modes here — catch them.

### Dismissed findings

- A weak answer you accepted because the user seemed satisfied (satisfaction ≠ precision)
- An alternative you mentioned but didn't actually probe ("X was considered" without why-not)
- A term you let pass without canonicalizing ("account" / "user" / "customer" used interchangeably)
- A claim about the codebase you didn't verify by reading code
- An invariant you wrote to `invariants.md` without the explicit confirm step
- An ADR drafted before the decision was actually firm, or without all three criteria met
- A branch you marked "resolved" without populating pinned/ruled-out/open

### Premature closure phrases

If any of these appear in your reasoning or output, re-open:

- _"The user seems satisfied with this answer"_ — satisfaction is not articulation. Did they state the answer with precision, or just nod?
- _"We've explored enough alternatives"_ — list them. If the list is "X (chosen) and... not sure what else", you haven't explored.
- _"This is probably an invariant"_ — ask the confirm question. "Probably" is not "confirmed project-wide invariant".
- _"The decision feels solid, let's write the ADR"_ — ADRs are end-of-session. Mid-session "solid" is wishful.
- _"The codebase probably supports X"_ — read the code. Don't probably.
- _"The user implied Y"_ — implication is not statement. Ask.
- _"We can resolve this offline"_ — offline is a deferral. Either pin it now or explicitly add it to the open list for the branch summary.
- _"This term is close enough to the existing glossary entry"_ — close enough is glossary drift. Either match the existing term or sharpen the distinction.

## Failure modes to avoid

- **Writing invariants without confirmation.** Every `invariants.md` write must follow an explicit confirm step.
- **Inline ADRs.** ADRs are end-of-session only. An "ADR" written mid-decision is a wish, not a record.
- **Following every tangent.** The branch checkpoint exists to prevent this. Use it.
- **Asking what you can read.** If the codebase or existing docs answer the question, read instead of asking.
- **Withholding opinions.** This skill supplies recommended answers. Stay opinionated. The user can disagree.
- **Sycophancy / pedantry / checklist-dumping.** Standard interrogation anti-patterns. Same as `grill-me`.
- **Auto-running docs-update at the end.** Suggest it; don't invoke it without the user's nod.
