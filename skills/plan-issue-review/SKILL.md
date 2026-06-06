---
name: plan-issue-review
description: Creates a high-level issue review plan as a single chat message output. Use when preparing to review multiple issues and need an overview with severity assessment, file references, and impact analysis before the development agent begins work.
---

# Plan Issue Review

## When to use
Load this skill when:
- Multiple issues need systematic organization before review
- A high-level plan must be created for a development agent session
- You need to assess overall severity and identify affected files upfront

**Critical constraint:** This agent has NO file modification tools. The final chat response IS the plan that gets passed to the review agent.

## Workflow

### Phase 1: Gather All Issues

```markdown
- [ ] Collect all source issues (tickets, bug reports, memory notes)
- [ ] Extract issue title, description, reproduction steps if available
- [ ] Identify affected components and file paths from context or codebase exploration
```

### Phase 2: Assign Severity and Assess Impact

| Severity | Criteria |
|----------|----------|
| **Critical** | Data loss, security vulnerability, system crash, blocking production |
| **High** | Major feature broken, significant user impact, no workaround |
| **Medium** | Feature degraded but functional, workaround exists |
| **Low** | Cosmetic issues, minor inconveniences, nice-to-have fixes |

### Phase 3: Create Comprehensive Plan Output

Generate ONE structured chat message containing:

1. **Overall assessment** — summary of scope and severity distribution
2. **Complete issue list** — sorted by severity (Critical first)
3. **File references** — all affected files with line numbers if known
4. **Impact analysis** — how bad each issue is, dependencies between issues
5. **Recommendations** — suggested approach for review agent

### Phase 4: Verification Before Output

```markdown
- [ ] All issues included and numbered sequentially (01, 02, ...)
- [ ] Every issue has assigned severity
- [ ] File references are accurate (verify with codebase exploration if possible)
- [ ] Severity ordering is correct throughout
- [ ] Impact assessment is honest — be conservative when uncertain
- [ ] Open questions are clearly stated for review agent follow-up
```

## Required Output Format

```markdown
# ISSUE REVIEW PLAN

## Overall Assessment
[Summary: number of issues, severity distribution, estimated effort]
[Example: "6 issues identified: 1 Critical, 2 High, 2 Medium, 1 Low. 
The critical race condition in AbstractChatService requires immediate attention."]

---

## Issue List (by Severity)

### CRITICAL

**Issue 01: [Title]**
- **Severity:** Critical
- **Confidence:** High/Medium/Low
- **Description:** [brief description]
- **Affected files:** `/path/to/File.java` line(s) (Verified/Likely/Unknown)
- **Impact:** [how bad this is, what breaks if unfixed]
- **Notes:** [any context or constraints]

---

### HIGH

**Issue 02: [Title]**
...

---

## File Reference Summary
| File | Issues Affecting It | Evidence Status |
|------|---------------------|-----------------|
| `/path/to/File.java` | Issue 01, Issue 03 | Verified |
| `/path/to/Other.java` | Issue 02 | Likely |

---

## Recommendations for Review Agent
- [Any suggested order beyond severity]
- [Dependencies between issues to be aware of]
- [Known constraints or risks]
```

## Key Principles

1. **One message only** — everything must fit in the final chat response
2. **Be conservative with severity** — when in doubt, rate higher
3. **Include ALL file references** — this saves review agent from searching later
4. **Honest impact assessment** — don't minimize issues; document real risk
5. **Use clear separators** — `---` between sections for easy parsing

## Common Pitfalls

- **Incomplete file references**: missing paths forces review agent to waste time searching
- **Underestimating severity**: better to over-report than miss critical issues
- **Vague descriptions**: be specific about what the issue is and where it occurs
- **Missing dependencies**: if fixing Issue A requires understanding Issue B, document this