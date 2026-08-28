---
name: jon
description: Runs the docs-first, PO-driven "Jon" workflow for agent-assisted software development — feature stories with business rules and BDD, ADRs, and plan-then-build cycles, including autonomous backlog runs. Use when starting agent-assisted development on a new repo, or when asked to work docs-first with stories, BDD and ADRs.
---

# Jon — docs-first PO workflow

"Jon" is a docs-first, PO-driven way of building software with an agent. This is a
harness-agnostic synthesis of the method — bring it to Claude Code, Codex, Cursor or any
other agent that can read, write and run tests. No project-specific tooling is required.
Canonical: https://github.com/sterlp/ai-skill-codex/tree/main/skills

When the harness has separate Plan/Dev agents, load their role details on demand:
- Plan role: [references/plan-agent.md](references/plan-agent.md)
- Dev role: [references/dev-agent.md](references/dev-agent.md)

In a single-agent harness, skip both references — one agent plays all three roles.

## The core idea

Three layers, each with one job:

- `docs/*.md` — the **business truth**: feature stories with rules and BDD.
- `docs/adr/` — **the agent's own memory**: technical decisions and their *why*.
- code — the **technical language**: what the docs say, built.

**The docs are the long-term memory; plans are throwaway work files; code must mirror the docs.**
The docs always express the target state (SOLL), kept clearly separate from what exists today (IST).

Roles: the **PO** owns the docs and orchestrates; the **Plan** writes only the plan file; the
**Dev** writes only code. In a single-agent harness, one agent plays all three roles but respects
the same boundary: the docs are **read-only while implementing**, then reconciled afterwards.

The PO is a skeptical guardian of the docs and the **single point of contact** for Plan and Dev —
the user talks to the PO, not directly to the sub-agents. A blocker or question that is technical
or architectural (approach, plan conflict, implementation detail) is resolved by the PO directly
with Plan/Dev, no user needed. Only a genuine gap in the SOLL — a missing business decision, an
unclear use case, a goal conflict — gets escalated to the user; the docs stay coherent and always
represent the SOLL, and a question the docs cannot answer is escalated rather than guessed.

If the harness gives Dev its own write tools, a small, self-contained fix (a failing test, a
targeted correction, applying an answer to an open question) may be applied directly, without a
plan — a plan is overkill for that. Anything touching multiple files/components or new behavior
still goes through the plan.

## The agile dev cycle

1. **Design with the user.** Capture the feature as a story: a goal (the *why*) plus business
   rules. Harden each rule with a BDD — GIVEN/WHEN/THEN covering happy path, edge and failure —
   and map each to a concrete test name. Interview one branch at a time, highest-impact unknown
   first, one question per message with a recommended answer. Nothing is "designed" before it is
   written into the docs — never only in chat.

2. **Plan from the docs, then build.** Slice the ❌ backlog into small **vertical** increments
   that each build green on their own — a slice cuts through one component's layers end-to-end
   (e.g. "user service incl. UI and API"), never a single layer spread across components. Plans
   are throwaway: every decision that survives goes into the docs — the docs are the memory, the
   plan is discarded.

3. **ADRs are the agent's own.** A technical decision that does not follow from a rule/BDD gets
   `docs/adr/NNNN-<slug>.md` (Status · Context · Decision · Consequences) plus a registry entry.
   An ADR never repeats a rule or BDD — cross-link, don't copy.

4. **Build iteratively, steer.** Dev tracks progress only in the plan file / task files it
   creates — never in `docs/`. After each integration the PO reviews (plan vs. code, code vs.
   docs) and may steer. Review is one mandatory pass, then at the PO's discretion — no review
   loop of death. Implemented rules flip ❌ → ✅ only with a green BDD test **and** the PO's own
   sign-off — that flip is the PO's job, never the Dev's.

5. **Autonomy.** When told to continue without the user ("build everything specified so far",
   user away, night cycle): work the backlog one story at a time, always the full loop (1–4).
   Build only what is ❌ specified; a story still 🚧 (unclear) is left open and skipped — never
   guessed or specified solo. Take any decision derivable from docs/ADRs/code yourself, record it
   as an ADR, **and** add it to a "to confirm" list in `docs/memory.md` so the user can review it
   deliberately once back. A real SOLL gap (missing business decision, goal conflict) blocks only
   that one story — build the rest. After **every** story: clean `docs/memory.md` (resolved out,
   new open items in) and reconcile the affected docs immediately if the build surfaced a
   precision or correction to the SOLL — don't batch this for the end. Report back whenever a
   feature area is finished or fully worked through, and end with a summary: built ✅, blocked 🚧
   with the open question, decisions awaiting confirmation, next steps.

6. **End of iteration.** Reconcile and compress the docs (keep only what helps a future session —
   never echo the code); lint them (registry complete, no orphans or broken links, no stale ✅);
   clean `docs/memory.md` (open ends on top, resolved below, decisions awaiting confirmation
   flagged). When it is worth it, close the cycle with a short retro: what was learned, routed to
   durable memory, an ADR, or `memory.md`.

## File conventions

- `docs/index.md` — the story registry: one line per story (title + one-sentence goal);
  adding or renaming a story updates the registry in the same step.
- `docs/<feature>.md` — goal + rules + BDD; one feature = one name across doc, ADRs and package.
- `docs/adr/index.md` — the ADR registry.
- `docs/memory.md` — cycle notes: open ends on top, resolved below, decisions taken
  autonomously flagged for the user's confirmation; cleaned after every story/cycle.
- Status markers — exactly one per feature and per rule: 🚧 in design → ❌ specified (the
  backlog) → ✅ done (green BDD test + PO reviewed and signed off).
- Every change to a rule records what it was (As-Is), what it becomes (To-Be) and why.

## Session & compaction discipline

Persist state into `docs/memory.md` **before** compacting the session (open ends, next steps,
decisions). After compaction, restore context from `docs/index.md` + `docs/memory.md`.
The files — not the chat — are the source of truth, which is what lets the method survive
context loss and a fresh session.

## What is NOT needed

- No separate backlog file — the docs ARE the backlog: the ❌ rules.
- No process outside the loop — a chat decision that isn't in the docs doesn't exist.
- Map the roles to sub-agents when the harness has them; otherwise run them as one agent.
