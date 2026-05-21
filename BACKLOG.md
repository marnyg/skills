# Skill Backlog

Skills to design, workshop, and tighten — one at a time.

Status legend: 📝 raw idea · 🛠 in workshop · ✅ shipped to `skills/`

---

## ✅ docs-scaffold + docs-update

Shipped to `skills/docs-scaffold/` and `skills/docs-update/`. See those SKILL.md files. Workshop notes preserved below for context.

## 📝 docs (workshop notes)

**Goal:** Describe the desired docs structure and layout for a repo, and help the agent navigate/maintain it.

Proposed structure (`docs/` in repo root):

- **Desired-state folder** — the "north star"
  - project context
  - ideal state of the system
  - hard constraints / values
  - higher-order goals
  - entity relationship diagrams
- **Technical docs**
  - sequence diagrams
  - code docs (APIs / user stories / technical guides)
  - ADRs (architecture decision records)
- **Day-to-day**
  - handoffs (next steps, loose threads)
  - milestones that frame tasks to goals
  - tasks, and which higher-order goal they attach to
  - implementation details to be aware of (current-state notes that don't belong in higher-order docs but matter — the stuff usually dumped in CLAUDE.md, e.g. "use the Azure DevOps CLI to run pipelines", "this repo is attached to these external systems: pipelines, Argo apps, kube context, etc.")

**Open questions for workshop:**
- Is this one skill or two? (a "scaffold docs structure" skill vs. a "maintain/navigate docs" skill)
- How does the agent decide what goes where? (heuristics for desired-state vs. day-to-day)
- Templates per doc type?
- How does it interact with existing CLAUDE.md / AGENTS.md / pi memory?

---

## 📝 domain-modeling

**Goal:** Help the user model a domain rigorously before/during implementation.

Components:
- Entity diagram (ERD)
- Sequence diagram
- Blast radius analysis (what breaks if X changes/fails)

**Open questions for workshop:**
- One skill with sub-modes, or three small skills?
- Output format — Mermaid? PlantUML? Just structured markdown?
- Where do diagrams live (the `docs/` skill above?)
- Does "blast radius" belong here or in a separate risk/change-impact skill?

---

## 📝 grill-me (refine existing)

Current version lives at `~/.claude/skills/grill-me/skill.md` — already two modes:
- **Critique mode** — adversarial stress-test of a plan/code/decision
- **Socratic mode** — sharpen articulation/understanding of a concept

**Workshop targets:**
- Compare against Matt Pocock's variant: https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md
  — his is design/conceptual outlining oriented; ours leans interrogation. Decide whether to merge, fork, or keep separate.
- Tighten mode-selection signal so the agent doesn't ask the disambiguation question unnecessarily.
- Decide if "grill-with-docs" (design-outlining flavor) becomes a third mode or its own skill.

---

## ✅ grill-design (was: grill-with-docs)

Shipped to `skills/grill-design/`. Renamed from `grill-with-docs` to avoid collision with our `docs-*` skill namespace and pair symmetrically with `grill-me`. Adapted Matt Pocock's approach to write into our richer `docs/` layout (glossary → domain-model, hard constraints → invariants, ruled-out alternatives → exploration-log, decisions → ADRs). Workshop notes preserved below.

## 📝 grill-with-docs (workshop notes)

**Goal:** Use grilling as a design/conceptual outlining tool — produce docs as a byproduct of being grilled.

**Workshop targets:**
- Read Matt's SKILL.md and extract the parts worth borrowing.
- Decide relationship to `grill-me` (merge as mode, or separate skill that calls it).
- Define the output artifact (where do the docs land — `docs/desired-state/`? scratch?).

---

## Workshop order (proposed)

1. **docs** — foundational; other skills will want to write into this structure.
2. **grill-me** refinement — already exists, quick wins, sharpens our workshop technique itself.
3. **grill-with-docs** — once `docs` and `grill-me` are settled, this becomes the bridge between them.
4. **domain-modeling** — builds on `docs` for diagram placement.

(Re-order freely.)
