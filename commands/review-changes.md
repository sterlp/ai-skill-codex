Review all changes on this branch using systematic issue tracking:

0. Run `git status` in base directory - if peon-plan/*.md exists, use it to validate that all planned changes are present and no unplanned changes were introduced
1. Select next changed file/class to review (prioritize source over tests)
2. If issue found: create `<project>/issue-xx.md`, write a test that reproduces it — if the test passes (not reproducible), discard it and document as "unconfirmed" in review-progress.md; if the test fails, proceed with fix
3. Apply fix, verify test now passes, verify full compile, document in `<project>/issue-xx-fix.md`
4. After each issue-fix cycle completes - update `<project>/review-progress.md` with reviewed files, findings, and remaining scope, then call `compactSession` with summary of progress
5. Repeat from step 1 until every changed file is reviewed and review-progress.md confirms no remaining scope
6. Create <project>/summary-commit.md listing all reviewed files, issues found/fixed, unconfirmed issues, and verification status

Rules:
- Fix issues as you find them (don't batch)
- Use issue numbering starting from highest existing + 1
- Verify fixes compile after each change
- Only flag real problems - skip style nitpicks unless critical
- If you fix issues apply TDD, clean code, SOLID principles and verify if shared code is properly resued in Util or shared classes