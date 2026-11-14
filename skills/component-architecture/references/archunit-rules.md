# Architecture rules: Spring Modulith and ArchUnit

What to check and why. Rules 1–2 are covered by Spring Modulith; rules 3–7 are own ArchUnit tests.

## When to set it up

**Not with only one domain component** — there is no boundary yet, and the prototype exception
(`SKILL.md`, access rule 7) is allowed. **Set it up when the second domain component arrives**, and resolve
the prototype exception at the same time.

In an existing project with cycles: set up what passes now (e.g. an ArchUnit rule for private building
blocks, with a self-check on a sample violation), record the rest as an open point, and switch on
`verify()` once the cycles are resolved. Generating the diagrams is useful even before that.

## Spring Modulith

Every direct sub-package of the application class's package is an "application module" in Modulith terms —
in this skill that is a **domain component**, not a Maven module.

```java
@SpringBootTest   // or a plain test — ApplicationModules does not need the context
class ModularityTest {
    @Test
    void verifyModules() {
        var modules = ApplicationModules.of(Application.class);
        new Documenter(modules).writeModulesAsPlantUml().writeIndividualModulesAsPlantUml();
        modules.verify();
    }
}
```

- `verify()` fails on cycles between domain components and on access to another domain component's
  internal packages. Public packages are declared per `package-info.java`:
  ```java
  @org.springframework.modulith.NamedInterface("entity")
  package com.example.app.customer.model;
  ```
  The top-level package of a module (service, events placed there) is public by default; sub-packages are
  private unless named.
- `Documenter` writes PlantUML diagrams — valuable **even when `verify()` is not active**: an agent can
  read unexpected arrows between domain components directly. Put them into the module's `docs/`.
- For Maven modules, the compiler already enforces the direction between modules; Modulith checks
  inside a module.

## ArchUnit rules

### Rules 1–2 — layer access and no cycles

Between domain components Modulith `verify()` covers both: no cycles (a cycle makes a later module split
impossible) and no access to another component's private packages. Without Modulith: `slices()` over the
domain packages. **Inside** a domain component the direction (resource → service → function →
repository/connector, never back) needs an ArchUnit `layeredArchitecture()` rule — Modulith does not
check it.

### Rule 3 — naming matches package

Every `*Repository` lives in `..repository..`, no `*Resource` in the service package, etc. — per the
project's mapping (naming-conventions.md).

### Rule 4 — transactions only at the internal facade

- `@Transactional` only on internal facade classes/methods; functions only with `Propagation.MANDATORY`;
  repositories only on an allow-list (bulk update, native query).
- Every `@Transactional` sets `readOnly` explicitly.
- Every `@TransactionalEventListener` sets `fallbackExecution` explicitly and lives on an internal facade.
- `Propagation.REQUIRES_NEW` only on an allow-list; each entry decided by the user, reason in a comment at the annotation.
See [transactions.md](transactions.md).

### Rule 5 — entities only through their domain component

An `@Entity` is changed only by its own domain component. Allowed: one deliberate one-sided relation
(`SKILL.md`, access rule 6) — the rule then checks that no second domain component references the same
foreign entity. `*Event` classes consumed by listeners are no entity reference.

### Rule 6 — `shared`

- Every `*Utils` lives in `..shared..`.
- `..shared..` depends on no domain package.
- UI counterpart (lint rule or test): `src/shared/` imports nothing from a domain folder.

### Rule 7 — API classes only from the external facade on

Classes of the api module are not referenced by services, functions or repositories. Exception:
enums and strong IDs. A service returning an `*Entity` is **correct**; only the DTO is forbidden there.
See [model-boundary.md](model-boundary.md).
