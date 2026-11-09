# Purpose

## Origin

Adapted as a lightweight project workflow from *WikiSkill: Compiling Agent Experience into
Persistent Knowledge for Skill Evolution* (Tang et al., 2026; https://arxiv.org/html/2608.27454v1).

## Patterns addressed

- Useful lessons disappearing in archived plans, chat history or ever-growing AGENTS*.md files.
- Repeating rejected skill changes because their outcome was not retained.
- Turning one-off project decisions into overly broad global instructions.
- Loading accumulated diagnostic history into agents that only need an active procedure.

## Adaptation

Every project keeps three layers: plans and evidence files as the raw layer, `.agents/skills/wiki/`
as the compact knowledge layer, and `.agents/skills/*/SKILL.md` as the executable layer. The base
loop (one skill outcome per cycle) also lives in the PO prompt, so it runs without this skill; the
skill adds the wiki, the ledger and the validation gate. Automated benchmarks and multi-agent
optimisation are deliberately omitted until their cost is justified.
