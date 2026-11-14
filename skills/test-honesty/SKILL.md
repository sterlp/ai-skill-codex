---
name: test-honesty
description: Recurring shapes of dishonest tests in this repo — green-without-feature, hand-built stand-ins for external exception shapes, falsifiability per test, surface pin inventories, green-before declarations, stub-LLM both directions — with origin stories. The one-liner rules stay in AGENTS.md/AGENTS-DEV.md.
---

# Test honesty — recurring shapes & origin stories (migrated from AGENTS-DEV.md, 2026-10-20)

**Trigger:** writing tests, reporting an increment green, mutation checks, test pin inventories.

The one-liner rules live in `AGENTS.md` / `AGENTS-DEV.md`; this skill carries the recurring shapes
and the origin stories. These bit us repeatedly in this repo — check them before reporting an
increment green.

## A test that would also pass without the feature is not a test

Recurring shapes here (2026-09-03, cycles 2b-1…3b):

- cleanup in `finally` running *before* the assertion
- assertions on a constant suffix only
- a test that builds the object graph *directly* and thereby bypasses the resolution chain it
  claims to prove

## Keyed on an external exception shape → reproduce it through the real wrapping layer

- **Keyed a rule on the shape of an external exception (JDK/library wrapper)? One test must
  reproduce that shape through the real wrapping layer** — a hand-built exception of the same class
  is not the production case.
  (Evidence 2026-10-29, R-AEM-3, `story/aem-error-url`: 6 green tests built `new RuntimeException()`
  themselves; the production shape — a `CompletionException` envelope carrying the cause's class name
  as its message — never reached the trigger, so the feature was dead in the reported case until the
  shape test. Review B1/B2, REJECTED → fix.)

## Falsifiability per test

- **Prove falsifiability per test, naming the path the mutation hits** — not "all red under mutation
  X". A mutation that only flips the filter says nothing about the cropping.
- **New branch in the production code ⇒ one test per branch** — the plan's test inventory derives
  from the *branching of the NEW code* (null guard, case normalisation, default fallback, suffix
  match), not only from the doc's BDD bullets. Where a mutation proof is warranted (next point), each
  mutation (drop the null guard, drop `toLowerCase`, `endsWith` → `equals`) must turn **exactly its
  own test** red — a branch whose mutation leaves the suite green is unpinned, whatever the doc said.
  (Origin 2026-10-24, bug-triage T-2 R-WEB-13: 4 of 6 new branches unpinned; Review F1.)
- **Mutation proof is risk-based, not per branch by default:** demand it where no honest
  negative/edge test can pin the rule (concurrency, caching, secrets, data loss); elsewhere a red-first
  test is the cheaper, durable proof. (Six reviews 2026-11 flagged blanket mutation lists as ceremony.)
- **Red first, per new test:** every new test shows its own red run before the fix — one red test
  does not vouch for four others. A test that cannot be red first is declared a characterization test.
  (I1-D 2026-11-13: only 1 of 5 new tests was shown red before the fix.)

## Absence and negative tests need a valid neighbour

- **"X is not shown/not restored/not rendered" is self-green** if the code path never draws anything.
  Pair every absence assertion with a valid neighbour on the same path that IS drawn (the safe
  picture next to the unsafe one, the named image before the unnamed one).
  (Büroklammer review 2026-11-13: `unsafePicturesAreNotRendered`, `anImageNeverBorrowsAnEarlierName`,
  `compactDoesNotRestoreRemovedPictures` — all green without the feature.)
- In jsdom assert on the element and its identity (`toBeTruthy()` on `.anticon-down`), never
  `.not.toBeNull()` (true on `undefined`) or "some `svg`"; scope selectors to the owning widget's
  DOM, not to `document`. (K5 review 2026-11-13.)

## Transitions, rendezvous and timing

- **Assert the transition, not a snapshot:** state before the trigger, the trigger, the end state —
  a snapshot that is already in the target state proves nothing. Fix the expected value BEFORE the
  trigger, never derive it from the actual result.
- **Every future/await is asserted:** read the outcome via `get`/`await` with a short deadline; a
  missed rendezvous must fail, never time out silently into green. (K1, K2, L1, Büroklammer reviews.)
- **A test exceeding its 1000 ms budget is split** (e.g. `test.each`), never given a longer timeout;
  a timeout that cannot fire within the test is no assertion.

## Surface format changed → pin inventory

- **Grep-inventory ALL test pins on that surface** — never trust the plan/doc list as complete
  (2026-09-19 read-numbers: plan named 4 raw-read pins, IST had 13; all adaptations purely
  mechanical).

## "Green before the change" declarations

- **Only for tests that can actually run against the pre-change state.** A new test coupled to the
  NEW type (e.g. asserts `instanceof Combo` where the old code had `CCombo`) is swap-falsifiable by
  construction and is declared as such — never as a regression guard that "was green before".
  (Origin 2026-09-12, config-cycle-2: all 3 R-A4 tests were red before the swap; the plan had
  declared 2 of them "green before and after".)
- If a test IS green before the fix, say so before building it — declare it a
  characterization/regression test in the plan; never present it as proof of the rule. Reporting this
  as a blocker is the correct move, not a failure.

## Stub/mocked-LLM tests: both directions

- **Assert BOTH directions: what is SENT and what is RECEIVED.** Assert the captured request payload
  (the actual built messages/parameters), not just that a call happened — whenever the send path
  contains logic (dedup, filtering, truncation, mapping), that logic IS the feature under test.
  (Origin 2026-09-11: `AiCompressorAgent` dedup was inverted → compact input always empty; both
  existing tests asserted only the system prompt / the response side and stayed green for the broken
  send path.)

## Removing a gate or trigger → inventory tests by their USAGE

- **When removing a gate/trigger (hook, flag, pre-turn check, event listener), inventory the affected
  tests by their USAGE of the removed thing, not by its name** — tests that need the gate as their
  trigger (no tool round / alternate path of their own) go red or NPE.
  (Origin 2026-10-28, R-CC-18.5: removing the per-turn compact flag left
  `AbstractAgentCompactResultTest.autoCompactRetriesAfterFailure` outside the plan inventory — second
  occurrence of this shape, first one 2026-10-19 with two tests missing from the same kind of list.)
