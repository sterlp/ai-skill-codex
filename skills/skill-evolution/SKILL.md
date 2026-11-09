---
name: skill-evolution
description: Grow a project's own skills from cycle experience — once per cycle decide one create/update/delete on `.agents/skills`, migrate generic AGENTS*.md passages into skills, and keep a project wiki (patterns + accepted/rejected ledger) so rejected proposals are not repeated. Use at cycle close / retro, in night cycles after each finished story, or when an agent proposes or criticises a skill.
---

# Skill evolution

Use once per cycle, after the review verdict and before approval. In night cycles: after each
finished story. This check never starts another review loop.

## Where things live (per project, committed with the cycle)

- `.agents/skills/<name>/SKILL.md` — the project's executable skills: procedures, traps, review checks.
- `.agents/skills/wiki/index.md` — compact patterns (problem · root cause · proven response ·
  evidence). The folder has no `SKILL.md`, so it is never loaded as a skill.
- `.agents/skills/wiki/skill-impact.md` — ledger, one row per outcome, including `no change` and
  rejections.
- Wiki missing? Create both files with the headers below in the same cycle.

## Inputs

1. Review findings, especially "skill/instruction gap", and the agents' skill-usefulness reports.
2. Skill proposals and problems from Dev/Plan reports and compaction summaries.
3. AGENTS*.md: generic, procedure-like passages (build recipes, traps, test patterns). Migrate them
   into a skill and leave only the trigger line ("X → skill y").
4. Corrections the PO had to repeat in this cycle.

## Loop

1. **Observe:** compare plan, implementation, tests, review findings and evidence — successes too.
2. **Consolidate:** check `wiki/index.md` and the ledger first. Already rejected? Do not re-propose
   without new evidence.
3. **Decide exactly one:** `no change` · `create <skill>` · `update <skill>` (shorten rather than
   append) · `delete <skill>` (wrong, outdated, harmful, or never used despite the chance).
4. **Validate:** show that the change would have prevented the observed failure or preserves the
   success — prefer a falsifiable test or break-the-fix evidence. Deleting needs the same evidence.
5. **Gate:** keep it only with evidence and without conflicting guidance; otherwise revert.
6. **Record:** one ledger row. A wiki pattern only when the lesson is reusable but not (yet) a
   procedure.
7. **Hand over:** add the trigger line to the agent file (e.g. AGENTS-DEV.md) and name the skill
   path to Plan and Dev in the next cycle.

## Boundaries

- Skills hold procedures, applicability conditions and checks. Business rules stay in feature docs,
  decisions in ADRs, session state in `docs/memory.md`, raw traces in plans and evidence files.
- A rule about how docs are written belongs to the doc method, not to a skill.
- Cross-project behaviour rules go to the memory tool. Promote a project skill to a global one only
  after it proved useful in at least two projects.
- Agents write the skill; the PO reads every change back before accepting it.

## Wiki file headers

`wiki/index.md`:

```
# Skill knowledge

Compact catalog of reusable patterns learned from implementation iterations. Each entry states
**problem · root cause · proven response · evidence**.
```

`wiki/skill-impact.md`:

```
# Skill impact

Record accepted and rejected skill proposals so ineffective guidance is not proposed repeatedly.

| Date | Iteration | Skill | Proposal | Outcome | Evidence |
|---|---|---|---|---|---|
```
