# Testing conventions

Order of importance — same in backend and UI: **entry point first** (resource end-to-end / page), then
**variants in isolation** (function / UI component).

## End-to-end test: what it means here

- A **real HTTP request** against the `*Resource` of the running app (`@SpringBootTest(webEnvironment =
  RANDOM_PORT)`), through service → function → repository → in-memory DB.
- **One Spring context** for the whole suite (see below).
- **External systems are stubs inside the test context**, never a real sandbox system (rate limits,
  instability, cost per call).
- Name: project convention (Step 0).

Calling a service method directly is not end-to-end: it skips security, serialization, validation and the
error mapping of the resource.

## Stubs

Name `*Stub` (e.g. `PaymentProviderStub`). Two forms, both registered in the one test context:

| Form | What it replaces | Tests along |
|---|---|---|
| **Servlet stub** (HTTP endpoint on the same test server, the connector's base URL points to it) | the external system | connector logic: serialization, retries, error mapping, timeouts |
| **Bean stub** (implements the connector's interface, `@Primary`/test config) | the connector | everything above the connector |

Prefer the servlet stub when the connector has logic worth testing; the bean stub is fine for simple
connectors (e.g. mail).

A stub offers:

- **recorded/configurable answers**,
- **introspection** — what it received and how often (e.g. a retry happened exactly twice),
- **fault injection** — error answers and timeouts, to test the failure path, not only the happy path.

### Stubs implement `Resetable` — reset is the default

Stubs reset themselves **before every test** (`@BeforeEach` in the abstract test base), so answers, error
configuration and recorded calls do not leak between tests. A stub's state is pure test support without
domain meaning, so a full reset is harmless. (Contrast with test data below — on purpose.)

## Isolated tests of functions: `<Name>ComponentTest`

Variants and edge cases are tested at the function, not by multiplying end-to-end tests. Plain unit test
(POJO, Mockito optional) or with the Spring context.

### Transaction in a Spring context test: `TransactionTemplate`, not `@Transactional`

A function with `Propagation.MANDATORY` (see [transactions.md](transactions.md)) needs a transaction in
the test.

- `@Transactional` on a test method **rolls back** at the end — the test is green even if the real
  commit would fail (constraints, flush errors).
- `TransactionTemplate` commits for real; assertions run afterwards against committed data.

```java
@Autowired TransactionTemplate trx;

@Test
void reduceStock_persists() {
    trx.executeWithoutResult(status -> reduceStockComponent.execute(order));
    // assertions outside the transaction, against committed data
}
```

A plain unit test has no Spring proxy — `MANDATORY` does not apply there. That is why plain unit tests
are often the simpler choice for variants.

## Stable suites

- **Random test data** (IDs, values without meaning for the test), not hard-coded — tests do not collide.
  Random data must not decide the outcome: a value that matters for the assertion is set explicitly.
- **One Spring context.** A second context (different `@ActiveProfiles`, `@MockitoBean`,
  `@TestPropertySource` per class) is a **bug in the test setup**, not normal — it slows the suite down and
  shows inconsistent setups. Put all stubs and overrides into one shared test configuration / abstract base.

## Test data reset: targeted, not global

A `TestDataHelper` implementing `Resetable` is possible, but a global reset of all helpers before every
test is the **exception**. Preferred: reset exactly the data the test needs in its *given* part. A global
reset hides what a test really depends on and slows the suite.

## UI

See [frontend.md](frontend.md), "Tests": page first, a render test per UI component, UI component logic in isolation.

