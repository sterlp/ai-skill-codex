---
name: review-bugs-and-fix-issues
description: Systematic bug review and issue resolution workflow. Use when reviewing multiple issues, debugging complex problems, or conducting code quality audits with documentation tracking.
---

# Review Bugs and Fix Issues

## When to use
Load this skill when:
- Multiple related issues need systematic review
- Bug triage requires verification before fixing
- Session context management is critical for long-running reviews
- Documentation of findings is required
- Issues must be processed in strict severity order

## Workflow

### Phase 1: Setup and Planning

**Input:** Receive the plan text from planning session (via `plan-issue-review` skill output). This is a single chat message containing:
- Overall severity assessment
- Complete issue list sorted by severity
- File references with line numbers
- Impact analysis and recommendations

```markdown
- [ ] Parse incoming plan structure from chat message
- [ ] Create PLAN.md file for status tracking (recommended)
- [ ] Optionally create per-issue files for detailed notes during review
- [ ] Verify issue count matches between plan text and your tracking
- [ ] Confirm severity ordering is correct (Critical → Low)
```

**File structure (if creating):**
```text
issue-tracker/
├── PLAN.md              # Status table, decisions log, final summary
├── issue-01-name.md     # Per-issue detail during active review
└── ...
```

**Note:** The plan agent provides structured text — you have file tools and should create tracking documents if desired.

**Severity order:**
1. Critical
2. High
3. Medium
4. Low

**Required PLAN.md fields:**
- Issue ID
- Title
- Severity
- Current status
- Reproduced? (yes / no / partial)
- Code changed? (yes / no)
- Fix status
- Notes / blocker

**Allowed issue statuses:**
- Pending
- In review
- Reproduced
- Fixed
- Reproduced but not fixable by agent
- Not reproducible
- Not an issue
- Blocked
- Skipped

### Phase 2: Issue-by-Issue Review

Process issues strictly from **highest severity to lowest severity**.

For **each issue**, follow this sequence:

```markdown
- [ ] Read the issue file completely
- [ ] Mark issue as "In review"
- [ ] Verify whether the report describes an actual issue
- [ ] Attempt reproduction / verification
    - [a] Reproduced
    - [b] Not reproducible
    - [c] Not an issue during verification
- [ ] Attempt to create a verification test
    - [a] If possible: write and run test
    - [b] If not possible: document why and continue
- [ ] If reproduced and fixable by the agent: implement the fix
- [ ] If reproduced but not fixable by the agent: document why and mark accordingly
- [ ] If any code change was made: run all tests of the affected component before compacting
- [ ] Update status in both the issue file and PLAN.md
- [ ] Compact session with preserve parameter (see below)
```

### Decision rules per issue

**1. Not an issue during verification**
Use this outcome when the reported behavior is confirmed during review but is by design, expected, already covered by existing constraints, or based on a misunderstanding of intended behavior.

Required documentation:
- Why it is not considered a bug
- Evidence checked
- Any relevant code path, config, test, or product rule
- Whether user clarification or documentation update is recommended

Set status to:
- `Not an issue`

**2. Not reproducible**
Use this outcome when the issue cannot be reproduced with the available code, configuration, data, and evidence.

Required documentation:
- Reproduction attempts made
- Environment / inputs used
- Missing information, logs, or setup required
- Whether follow-up is needed

Set status to:
- `Not reproducible`

**3. Reproduced and fixable by agent**
Use this outcome when the issue is verified and the required change is within agent capability and repository scope.

Required actions:
- Write a verification test when possible
- Implement the smallest valid fix
- Re-run the verification test if created
- Run all tests of the affected component before compacting
- Record changed files and outcome

Set status to:
- `Fixed`

**4. Reproduced but not fixable by agent**
Use this outcome when the issue is real, but the agent cannot safely complete the fix.

Examples:
- Missing external system access
- Requires production-only validation
- Requires architectural, product, legal, security, or business decision
- Requires credentials, infrastructure access, or manual migration
- Risk is too high for autonomous change

Required documentation:
- Why the issue is real
- Why the agent cannot fix it
- What exact next action is required
- Whether a partial mitigation exists

Set status to:
- `Reproduced but not fixable by agent`

### Phase 3: Session Management

**Critical:** Use the session compaction tool with `preserve` parameter between issues to maintain context across cycles. Proactive compaction prevents loss of critical information when context limits are approached.

The tool is named `compactSession`. Do not rename or replace it.

Use this pattern:

```json
{
  "preserve": "Current issue, severity order, completed issues, issue statuses, file paths modified, tests executed and results, unresolved blockers, next issue to review, important decisions and rationale"
}
```

**What to preserve:**
- Current issue ID and title
- Severity ordering and next issue to process
- Completed issues and final statuses
- File paths modified
- Tests run and whether they passed
- Whether code was changed for the issue
- Open blockers or required user input
- Key decisions and rationale

### Phase 4: Verification and Cleanup

After all issues are processed:

```markdown
- [ ] Run final build verification across all affected projects or components
- [ ] Confirm every issue is reflected correctly in PLAN.md
- [ ] Add final summary and follow-up recommendations to PLAN.md
- [ ] Delete detailed per-issue markdown files
- [ ] Keep PLAN.md as the permanent overview artifact
```

**Cleanup rule:**
- Remove temporary per-issue files after review is complete
- Preserve only the consolidated overview document unless the user explicitly asks to keep detailed issue files

## Key Principles

1. **Review in severity order** — process the most important issues first
2. **Verify before fixing** — not every reported issue is actually a bug
3. **Not reproducible is a valid outcome** — document it clearly
4. **Not an issue is a valid outcome** — by-design behavior must not be "fixed"
5. **Test when possible** — automated verification reduces regressions
6. **Run full component tests after code changes** — do this before compacting
7. **Document everything** — findings, rationale, blockers, and outcomes
8. **Preserve context** — use `compactSession` between issues
9. **One issue at a time** — avoid mixing investigations

## Common Pitfalls

- **Fixing before verification**: do not change code before confirming the issue
- **Ignoring severity**: low-severity issues must not displace critical issues
- **Treating non-reproducible issues as resolved**: they must be marked explicitly
- **Treating by-design behavior as a bug**: verify intent before changing logic
- **Skipping component-wide tests after a fix**: local success is not enough
- **Context loss after long sessions**: preserve decisions, file paths, and next steps when compacting
- **Unsafe autonomous fixes**: mark issues as reproduced but not fixable by agent when escalation is required