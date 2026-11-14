# Naming conventions

**The project defines the names** (see `SKILL.md`, Step 0). This table is the default to propose when the
project has no recognizable pattern, and the mapping key when it has one: map these defaults to the
project's names once, in the project's architecture doc — do not carry variants through the code.

Why the suffix sits on the entity: see [model-boundary.md](model-boundary.md).

## Default names

| Role | Backend default | Spring annotation | UI default | Legacy (JEE) |
|---|---|---|---|---|
| External facade | `*Resource` | `@RestController` | — | `*BF` / `*BusinessFacade` (remote interface) |
| Internal facade | `*Service` | `@Service` | `<name>.page.tsx` (fixed) | `*BM` / `*BusinessManager` / `*Manager` (local interface) |
| Function / UI component | `*Component` | `@Component` | `*.component.tsx` (or `.widget.tsx` / `.view.tsx`) | `*BA` / `*BusinessActivity` / `*Activity` |
| Connector | `*Connector` | `@Component` | `<name>.hooks.ts` | `*BC` / `*BusinessConnector` / `*ESI` |
| Repository | `*Repository` | `@Repository` | — | `*DAO` |
| Converter | `*Converter` | `@Component` | — | – |
| Entity | `*Entity` | `@Entity` | — | `*BE` / `*BusinessEntity` |
| API class | no suffix; versioned `…V1`/`…V2` | – | generated `<name>-api.d.ts` | – |
| Event | `*Event` | – (no bean) | – | – |
| Shared utility | `*Utils` in `shared` | – (no bean unless needed) | `src/shared/<x>/<x>.component.tsx` | – |
| Stub | `*Stub` | test config | project tool | – |
| Function test | `<Name>ComponentTest` | – | `<name>.component.test.tsx` | – |

Folders/packages per domain component: backend `api/` (resource + converter), `model/`, `repository/`,
`component/` (functions), `connector/`, service at the top level (no `facade` package). UI: `components/`, `hooks/`, `model/`, pages at the top
level. Projects often name these differently (`entity/`, `control/`) — that is fine once it is written
down, as long as each role has exactly **one** name. Synonyms side by side are a defect (`SKILL.md`, Step 0).

Module names: domain name, same as backend package and UI folder (case per language:
`business_partner` ↔ `business-partner`). api module: `<name>-api`.

## Legacy names (JEE)

External facade = remote interface, internal facade = local interface — that pairing is the reason for two
facades. `*Manager` and `*Service` are the same layer; which name applies is team convention.

## Events

`*Event` classes (e.g. `OrderCreatedEvent`), published via `ApplicationEventPublisher`, resolve the back
direction between two domain components without a direct reference.

- Default package: `model` of the publishing domain component. A separate `event` package only once there
  are many events.
- Declare the package public for other domain components if needed (Spring Modulith `@NamedInterface`).
- A common marker interface (e.g. `DomainEvent`) makes all events findable by one type search.

## Recommendations

- Only the API side is versioned (`PersonV2` + `PersonResourceV2`); the entity stays unversioned.
- Project meta-annotations (e.g. bundling `@Transactional(readOnly = …)` with timing/metrics) instead of
  bare Spring annotations make later cross-cutting changes a one-place change.
- A function ideally has one public entry method (`execute()`), private helpers allowed — easier to test
  and reuse (cf. command pattern).
