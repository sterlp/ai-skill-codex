# Branch Review Workflow

0. `git status`; if peon-plan/*.md exists, validate planned vs actual changes.
1. Pick next changed file (source before tests).
2. Classify before acting:
   - docs match code -> skip, log "verified-by-docs" in review-progress.md
   - docs contradict code -> confirmed bug, go to 3 (skip discard-if-passes check)
   - test exists, docs silent -> docs-bug: extend docs, log in review-progress.md, no issue file
   - no docs/test/clear intent -> append to open-points.md (file+line), skip, do not guess
3. Confirmed bug only: create issue-xx.md, write test asserting doc-defined behavior. Passes unexpectedly -> discard, log "unconfirmed". Fails -> go to 4.
4. Fix, verify test passes, verify full compile, document in issue-xx-fix.md.
5. Update review-progress.md (reviewed files, findings, remaining scope), compactSession with next target. Loop to 1 until no scope remains.
6. Write summary-commit.md: files reviewed, issues fixed, docs-bugs fixed, unconfirmed issues, verification status.
7. Show open-points.md if it exists.

Rules: fix as found, don't batch. Issue numbers = highest existing + 1. Verify compile after every change. Skip style nitpicks unless critical. Apply TDD/SOLID, reuse shared Util code. Docs lead over code when unambiguous; if unsure, defer to open-points.md, never fabricate intent.