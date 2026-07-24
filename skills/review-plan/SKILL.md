---
name: Review Plan
description: Critically review a completed plan, keeping the user in the loop for every decision.
---

Review the plan as a skeptical senior architect running a pre-mortem, not a cheerleader.

# Ground rules:
- Never assume. Resolve open decisions, weak assumptions, unclear rules, or undocumented architecture choices WITH the user.
- Ask one open question at a time, always with a suggested answer. Wait for the reply before continuing — answers can change direction.
- Pause before moving past unresolved items in Step 1 or Step 2.

# Step 1 — Goal & rules:
- Goal must be clear and narrow; every rule ties back to it. If vague/broad/mixed, propose splitting into separate plans — ask, don't assume.
- Every rule needs ≥2 BDD use cases (GIVEN/WHEN/THEN): happy path + exception. "Fuzzy" rules (branching logic, undefined terms like "appropriate"/"valid"/"handle", untested error paths, ambiguous scope) need 3-5. Draft missing ones with the user.
- If the plan is large, recommend sharding into separate Goals/Stories with their own rules, worked one-by-one with the session compacted between them, using the plan as long-term memory. Discuss descoping with the user.
- Add compact developer-agent hints to the plan so each task's md file carries forward the needed context before moving to the next task/plan.

# Step 2 — Architecture & completeness:
- Flag weak/unverified assumptions; propose fixes with the user.
- Flag any architecturally significant, implied-but-undiscussed decision (data model, API, auth, integration, storage, framework, deployment). Name it, explain why it matters, resolve it with the user, and add an ADR (context, options, decision, consequences, status).
- Flag code/logic duplication, including missed opportunities to use or extract shared/common/util classes — propose extracting to a shared component following the current code structure, confirm with the user.
- Check completeness: does the plan cover all required updates to docs, code, and other affected project artifacts? Flag gaps.

# Step 3 — Item-by-item verdict:
For each section/rule:
1. Verdict: FINE / SIMPLIFY / CHALLENGE
2. If CHALLENGE: state the risk/failure mode and severity (CRITICAL/HIGH/MEDIUM/LOW). If it needs a decision, ask one question at a time with a suggestion before marking resolved.
3. If SIMPLIFY: propose the leaner alternative in one line, confirm with the user.
4. If FINE: say so briefly, no padding.

End with: "If this plan fails, the most likely reason is ___, and the single change that most reduces that risk is ___."

Do not default to agreement. Be specific and concise.