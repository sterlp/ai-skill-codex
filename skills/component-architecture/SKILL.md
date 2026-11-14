---
name: component-architecture
description: Domain-cut component architecture for Spring Boot backends and their React/TypeScript UI — facades, functions, repository, connector, shared; pages and UI components; Maven modules by domain with own api modules and generated TypeScript types; plug-in modules; transactions, naming, testability, ArchUnit/Spring Modulith. Use when structuring new code, adding a feature, component, page or module, deciding where logic, transactions or server calls belong, reviewing boundaries or naming, setting up tests, or when "component architecture", "Komponenten-Architektur", "facade", "connector", "shared", "page", "widget" or "api module" are mentioned.
---

# Component architecture

Cut code by **domain, not by technology** — in the backend, in the UI and in the Maven modules. The same
few patterns apply everywhere: a domain component has one entry surface (its service; in the UI its pages), small single-purpose
building blocks behind it, and talks to other domain components only through their entry point.
Based on the talks ["Part 1"](https://www.youtube.com/watch?v=luaDzMKyF0g) and
["Part 2"](https://www.youtube.com/watch?v=41coSMipVaA).

## Step 0: use the project's names — or ask

This skill defines **roles and rules**. The project defines the **names** (suffixes, package names,
exported packages, test names).

1. Read the project's architecture doc first (e.g. `docs/**/*architektur*`, `docs/**/*architecture*`,
   `AGENTS.md`, ADRs). **The project doc overrides the defaults below**; roles and rules stay.
2. No doc: derive the mapping from the code — count suffixes and package names. Map by
   **role, not by name**: a class the service calls for one step is a function, whatever it is called
   (e.g. `control/*Control`).
   **One pattern per role** — suffix and package belong together (`*Control` in `control/` or `*Component`
   in `component/`). Two patterns for the same role are a defect, usually an unfinished rename, not two
   concepts. Derive the winner from the code if it is clear (e.g. all classes already carry the new
   suffix → the package follows), otherwise ask the user. Never add a third variant. New code uses the
   winner; renaming existing code is its own iteration, only when the user asks for it.
3. No recognizable pattern (new project, mixed code): **ask the user** — one question per message, your
   recommendation first — proposing the defaults:
   backend `*Component` (Spring term), UI `*.component.tsx` (synonyms `.widget.tsx`, `.view.tsx`).
   Pages are always `*.page.tsx` — unambiguous, not negotiable.
4. Write the agreed mapping into the project's architecture doc, so the next agent does not guess.

## Terms (used strictly in this skill)

| Term | Meaning |
|---|---|
| **Domain component** | A domain unit: backend package `customer`, UI folder `src/customer/`. Same name on both sides, singular, case per language (`business_partner` ↔ `business-partner`). The basic unit of this skill. |
| **Function** | Building block for one use-case step inside a domain component. Default class suffix `*Component`. |
| **UI component** | Building block below a page; renders one thing. Default file suffix `*.component.tsx`. |
| **Module** | Build unit (Maven module, npm package). Holds one or more related domain components. A domain component is usually just a package, not a module. |
| **api module** | The external model (API classes) of a module; source of the generated TypeScript types. |

## Dependency rule

Source dependencies point inward only, never in a cycle, coarse → fine (workflow → function, never back
to the facade). The call flow external facade → internal facade → function → repository/connector points
outward; that is fine because the caller depends on an **interface** of repository/connector
(dependency inversion). Only the dependency direction must point inward, not the call direction.
Every entity belongs to exactly one domain component.

## Roles: backend and UI

| Role | Backend default (Spring) | UI default (React) | Owns |
|---|---|---|---|
| External facade | `*Resource`, `@RestController` | — (the UI calls it) | external ↔ internal model, versioning, authN, HTTP contract |
| Internal facade | `*Service`, `@Service` | `<name>.page.tsx` (a route) | workflow and control flow. Backend: transaction, authZ/policies, caching. UI: state, server calls, loading/error, wiring UI components |
| Function / UI component | `*Component`, `@Component` | `*.component.tsx` | one step / one thing; parameters or props in, result or callbacks out; isolated testable. A UI component makes **no** server call |
| Repository | `*Repository` | — | persistence |
| Connector | `*Connector` | `<name>.hooks.ts` (server calls) | internal → external, mirror image of the external facade |
| Entity | `*Entity` in `model/` | — | internal model |
| API class | api module, no suffix | generated `<name>-api.d.ts` | external model, shared backend ↔ UI |
| Shared | `<base>.shared.*Utils` | `src/shared/` (e.g. `server-error.component.tsx`) | technical tools without domain logic |

A page is the UI counterpart of a service: the top-level element of a domain component. A
`customer.component.tsx` that only shows one customer is the UI counterpart of a `ReadCustomerComponent`.
In the UI, extraction matters even more because UI components get reused more — and every UI component must
clearly be either **domain-owned** or **shared**. Details: [references/frontend.md](references/frontend.md).

## Access rules (backend and UI)

1. Another domain component is reached **only through its internal facade** (backend: its `*Service`;
   UI: what it explicitly exports, e.g. a reusable UI component — never its hooks or page internals).
2. Its functions, repositories, connectors and converters are private.
3. Its entities and events may be used if the package is declared public (Spring Modulith:
   `package-info.java` with `@NamedInterface`). Undeclared packages are private.
4. `shared` may be imported by everyone and knows no domain component.
5. **Plug-ins report back, the owner counts.** A domain component implementing another one's extension
   interface (e.g. a message handler) uses only that interface's public types. Cross-cutting work (metrics,
   retries, logging of the outcome) stays with the owner of the process; the plug-in only returns the
   result or reason.
6. **Mutual needs:** pick one direction. Only the owner holds the entity relation (`@ManyToOne(fetch = LAZY)`),
   never both ways. Changes only through the other facade, never via cascade. Back direction via `*Event`
   or via ID (`orderService.setTrackingId(...)`), never via an entity reference. The event belongs to the
   component that is depended on (it publishes); the depending component listens — importing the event
   follows the existing direction, so no source cycle arises.
7. **A connector needed by a second domain component:** first look for the real cause — usually a hidden,
   never-extracted service. If the need is real, the connector becomes its own domain component with a
   service. Agreed with the user, its own preparation iteration before the feature.
8. **Prototype exception** (only one domain component): the external facade may read the repository
   directly. Ends when the second domain component arrives.
9. Enforce with ArchUnit/Spring Modulith, not by convention: [references/archunit-rules.md](references/archunit-rules.md).

## Modules

Cut Maven modules by domain, the same way as packages:

- **Domain module** (`customer`): related domain components of one domain. As soon as there is more than one
  domain module, each gets its own **api module** (`customer-api`). TypeScript types are generated
  (`typescript-generator-maven-plugin`) per domain component into its UI folder — the UI never hand-writes
  server types.
- **Plug-in module**: a capability unrelated to the domain (audit, docs hosting, docs assistant). Extract it
  early: own package root, Spring Boot auto-configuration, own tests and stubs, own `docs/`, the host owns
  security, config and secrets.
- From **≥ 10 domain components** in one module: with every new one, check whether a module split is the better
  choice. Only ever a proposal — **never decide alone**, discuss it with the user, and do it as its own
  preparation iteration before the feature.

Module kinds, api module, TS generation, plug-in rules, module docs: [references/modules.md](references/modules.md).

## Model boundary

Internal model `*Entity`, external model = API classes in the api module. Convert **once, at the external
facade** — a service that accepts or returns an API class is a bug; a service returning an `*Entity` is
correct. Enums and strong IDs may cross. Why, versioning, detached entities:
[references/model-boundary.md](references/model-boundary.md).

## Transactions, authorization, caching

The transaction boundary is the internal facade — never the external facade or the connector. Functions
join it (`Propagation.MANDATORY`); a repository opens its own only by exception (bulk update, native query). Every `@Transactional` sets `readOnly` explicitly. Connector calls stay outside
the transaction unless there is a deliberate rollback reason. Details:
[references/transactions.md](references/transactions.md).

## Testability

Entry point first, then variants in isolation — backend and UI alike: an end-to-end HTTP test against the
`*Resource` (one Spring context, external systems as stubs in that context), variants in an isolated
`<Name>ComponentTest`; in the UI the page first, a render test per UI component.
Details: [references/testing.md](references/testing.md), UI: [references/frontend.md](references/frontend.md).

## Design principles

- **Open/closed via strategy:** for functions, UI components, connectors and — where it fits — services: a new variant is
  a new class behind the same interface, never a new `if`/`switch` branch in existing code (new payment
  provider → new connector; new discount rule → new function). A facade that is pure workflow is exempt;
  its exchangeable steps are not.
- **Size:** methods max ~60 lines, classes/files max ~500 lines, in every layer (pages and UI components too).
  Before going over, check:
  ```
  - [ ] Does everything belong here? -> extract a function/UI component or a new domain component
  - [ ] Is the abstraction level right? -> move details into a function/UI component
  - [ ] Foreign orchestration in here? -> back to the responsible service/page/module
  - [ ] A hidden, unnamed building block? -> extract it as its own building block
  ```
  Only if all four are "no" is going over a rare exception — re-check on every change.
- **Internal facade maturity:** as it grows, it delegates to functions until only control flow (if/loop,
  transaction boundary) is left. The transaction boundary stays with it.
- **Docs:** a module's docs live in the module (`<module>/docs/`); comments only for what the code cannot tell.

## Workflow: new domain component

```
- [ ] Step 0 done: project names known (or agreed with the user and written down)
- [ ] Name = domain term, identical in backend package and UI folder
- [ ] >= 10 domain components in the module? Propose a module split to the user first
- [ ] Entities with relations only inside this domain component (exception: one one-sided relation to a foreign entity, access rule 6)
- [ ] Backend: repository, functions (+ <Name>ComponentTest), service, resource, converter
- [ ] API classes in the api module; TS types regenerated by the build, not hand-written
- [ ] UI: <name>.page.tsx, hooks for server calls, UI components without server calls
- [ ] UI components: domain-owned or shared/ — decided explicitly
- [ ] Connector only if needed, private to the domain component; stub for it in the test context
- [ ] Technical helpers to shared, never domain logic there
- [ ] Mutual need: one direction, back via event or ID
- [ ] Dependency graph: no cycles, no jump back
- [ ] End-to-end test over the stack; page test in the UI
- [ ] From the 2nd domain component: ArchUnit/Modulith, resolve the prototype exception
```

## Workflow: extract a module

```
- [ ] Agreed with the user; its own iteration, not mixed with a feature
- [ ] Kind decided: domain module (+ api module) or plug-in module (references/modules.md)
- [ ] Moved, not copied; old package removed in the same iteration
- [ ] Dependencies between modules acyclic (the compiler now enforces it)
- [ ] Module docs/ with feature doc, architecture doc, ADRs; root docs only reference it
- [ ] Module tests run on their own (mvn -pl <module> verify)
- [ ] Host wiring documented: security, config, secrets, extension beans
```

## Common mistakes (not obvious from the rules)

- Names guessed instead of taken from the project doc or asked (Step 0).
- Synonyms treated as different roles (`*Control` vs. `*Component`), or a new synonym introduced.
- `@Transactional` on a Spring test method — the rollback hides persistence bugs; use `TransactionTemplate`.
- `REQUIRES_NEW` chosen without the user — never your own decision. Writes from a listener join (`BEFORE_COMMIT`); work that must not join goes `@Async`.
- More than one Spring test context (different profiles/`@MockitoBean` per class) — a setup bug, not normal.
- Plug-in does the owner's cross-cutting work (counts, retries) instead of reporting back.
- `src/shared/` UI component or `*Utils` importing domain types — it belongs to that domain.
- Module split or connector extraction done alone or mixed into a feature — always the user's decision,
  its own iteration.

## References

1. [modules.md](references/modules.md) — module kinds, api module, TS generation, plug-in modules, module docs
2. [frontend.md](references/frontend.md) — pages, UI components, hooks, shared, UI tests
3. [model-boundary.md](references/model-boundary.md) — entity vs API class, versioning, detached entities
4. [transactions.md](references/transactions.md) — MANDATORY, readOnly, connector calls, events
5. [testing.md](references/testing.md) — end-to-end, stubs, `TransactionTemplate`, test data
6. [naming-conventions.md](references/naming-conventions.md) — default names, legacy JEE mapping, events
7. [archunit-rules.md](references/archunit-rules.md) — Spring Modulith, ArchUnit rules
