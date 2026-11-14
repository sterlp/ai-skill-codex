# Model boundary: where internal and external model are converted

Two class families, told apart by suffix **and** package:

| Family | Suffix | Package | Lives in |
|---|---|---|---|
| Entity (internal model) | `*Entity` | `model/` (project may call it `entity/`) | internal facade, functions, repository |
| API class (external model) | none; versioned `V1`/`V2` on resource **and** API class | api module, `<base>.api.<component>` | from the external facade outward, and in the UI as generated types |

The suffix sits on the domain model (`PersonEntity`), so the API class `Person` needs none. "DTO
boundary" is an old synonym for "API boundary" — no `*DTO` suffix is meant.

## Convert once, at the external facade

Conversion happens **only at the external facade**, via a `*Converter` next to the resource. The
service returns an `*Entity` or an internal read model (e.g. a Spring Data projection in `model/`); the
converter turns that into the API class. A service that accepts or returns an API class is a bug.

Why not at the internal facade: services call each other. A boundary there would force Entity → API →
Entity on every service-to-service call — ceremony without insight, and every field change hits several
converters. Cross the internal/external boundary exactly once: where the process really leaves the
system. Adapters between API versions (see Versioning) stay on the external side and do not count.

**Own repository directly:** a service may call its own repository. No extra facade or DTO wrapper for
its own calls — repositories are private to the domain component, not to a function.

## Exception: version-stable value types

Enums and strong IDs (`record PersonId(String id)`) may be used from the api module in repository,
service and function — a mapping pays off only with API versioning. Full DTOs (`PersonV2`) never cross.
A projection never targets the versioned API class — otherwise a new API version would have to touch
the query.

## Versioning

Only the external model is versioned. There is exactly one `PersonEntity` and one service.
`PersonResourceV1` maps `PersonV1` → `PersonV2` and delegates to `PersonResourceV2`; only the newest
resource talks to the service. The internal model stays stable while several API versions coexist.

## Gotcha: detached entities

The transaction ends at the internal facade (see [transactions.md](transactions.md)). What the external
facade receives is **detached** — no lazy loading, no navigation over `@OneToMany`. Everything the caller
needs is loaded inside the transaction, or it becomes a projection or its own service method. That is the
price of this decision, paid deliberately.

## The connector mirrors the boundary

The connector converts the internal model into the external system's classes. Those belong to the
connector and model the foreign system — they are neither `*Entity` nor API classes of the api module, and
ArchUnit rule 7 does not apply to them.
