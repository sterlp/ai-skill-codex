# Transaction management

A cross-cutting concern of the **internal facade** (`*Service`). Never at the external facade, never in
the connector, only by exception at the repository (bulk updates, native queries).

## Why there

A transaction represents a use case ("place order"), not a single DB call. The internal facade
orchestrates several repositories/functions — that is where all-or-nothing must hold.

```java
@Service
public class OrderService {
    @Transactional(readOnly = false)
    public void placeOrder(PlaceOrder command) {
        orderRepository.save(order);
        reduceStockComponent.execute(order);
        // no connector call here, unless there is a deliberate rollback reason (see below)
    }
}
```

## Functions join, never open: `Propagation.MANDATORY`

Functions (`*Component`) annotate `@Transactional(propagation = Propagation.MANDATORY, readOnly = …)`. Called without a
running transaction they fail with `IllegalTransactionStateException` — this makes a missing boundary
visible instead of silently running each repository call in its own transaction. Tests of such functions
need their own transaction (see [testing.md](testing.md)).

## `readOnly` is mandatory

Every `@Transactional` sets `readOnly` explicitly (`true` for reads, `false` for writes) — never rely on
the default. Reason: the reader sees the intent, and `readOnly = true` lets Hibernate/the JDBC driver skip
dirty checking and flushes. A project meta-annotation can bundle this (see naming-conventions.md).

## Connector calls and transactions

Default: **no connector call inside a transaction.** Network I/O keeps locks and the connection open, and
a rollback cannot undo what the external system already did.

Allowed only with a **deliberate rollback reason**, written next to the call: e.g. the external system must
accept the data, otherwise our change must not be committed either. Then:

1. The external call is the **last** step before commit, so a failure rolls back everything.
2. Set an explicit transaction timeout and an explicit client timeout on the connector.
3. A caller of a transactional endpoint waits **longer** than that endpoint's transaction timeout (buffer,
   e.g. +1 s). Otherwise it gives up while the server may still commit, and nobody knows the outcome.
4. Accept the remaining risk: the external system succeeded, our commit fails → the external side needs an
   idempotent retry or a compensation.

Everything else: commit first, then call the connector — from an `AFTER_COMMIT` listener, preferably
`@Async`, so the caller does not wait for the external system (see "Events and transactions").

## Events and transactions

Details and tests: skill `spring-transactional-event-listeners`. The listener is a method of the
listening domain component's internal facade (`*Service`).

- **DB writes: join the publisher's transaction, `BEFORE_COMMIT`.** One connection, one commit, a failure
  rolls back everything:
  ```java
  @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT, fallbackExecution = true)
  @Transactional(readOnly = false)   // REQUIRED: joins; opens one only in the fallback case
  public void onOrderCreated(OrderCreatedEvent event) { ... }
  ```
- **No DB write, only after a successful commit** (connector call, mail): `AFTER_COMMIT`, preferably
  `@Async`. Changes started there are not part of the finished commit.
- Set `fallbackExecution` explicitly (default `false`): with `false`, an event published **outside** a
  transaction is silently dropped. Use `true` unless you deliberately want that.
- **`REQUIRES_NEW` is never your own decision.** It suspends the business transaction, holds a second
  connection (pool exhaustion under load) and decouples the commit. Use it only if the user has already
  decided it, or tell the user and get the decision first; the reason goes into a comment at the
  annotation.
- **"Must not join" (other module, own lifecycle, independent of the caller's outcome)** → `@Async`
  (`AFTER_COMMIT` listener or async service call), not `REQUIRES_NEW`: the work runs in its own thread with its
  own transaction, the caller's connection is not held twice.
- **What a valid reason looks like** — both must hold, and the user confirms it: (1) the commit must be
  independent of the caller's outcome, **and** (2) it must run synchronously in the same thread. Example:
  an audit entry must survive the rollback of the business process and must be written before the
  process continues. Only (1) → `@Async`. Neither → join.

## Spring pitfalls

- **Self-invocation:** `this.otherMethod()` inside the same class bypasses the proxy and the transaction.
  Only public methods of the internal facade, called from outside, are transactional.
- **Swallowed exceptions:** catching an exception inside a `@Transactional` method without rethrowing
  prevents the rollback. Checked exceptions need `rollbackFor`.
- `@Transactional` is local to one DB; across modules/services use sagas/compensation.
