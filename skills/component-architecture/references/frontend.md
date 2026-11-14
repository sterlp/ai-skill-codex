# UI (React / TypeScript)

Same roles and rules as in the backend (see `SKILL.md`, "Roles"), different building blocks. Names are a
project convention (Step 0); only `*.page.tsx` is fixed.

## Folder per domain component

```
src/
  customer/                         same name as the backend domain component
    customer.page.tsx               page = route, entry point (counterpart of CustomerService)
    customer-list.page.tsx          a second page of the same domain component
    customer-api.d.ts               generated from the api module, never edited
    hooks/customer.hooks.ts         server calls (counterpart of a connector)
    model/customer.model.ts         UI-only state types (not server types)
    components/
      customer/customer.component.tsx         shows one customer
      customer-edit/customer-edit.component.tsx
  shared/
    server-error/server-error.component.tsx   no domain
    phone-input/phone-input.component.tsx     no domain
```

## Page = internal facade of the UI

- Top-level element of a domain component, reachable via the router.
- Owns state, server calls (through hooks), loading/error handling and the workflow between UI components.
- Grows like a service: delegate rendering to UI components until only wiring and control flow are left.

## UI component = function of the UI

- Renders one thing from props; reports user actions via callbacks.
- **No server calls**, preferably no global state — it gets what it shows. That makes it reusable and
  testable without a server.
- One per file, same size rules as in the backend.
- Default suffix `*.component.tsx`; `.widget.tsx` and `.view.tsx` are synonyms — one suffix per project
  (`SKILL.md`, Step 0). Whether a UI component is reusable or page-specific is expressed by its **place**
  (see below), not by a second suffix.

## Owned or shared — decide explicitly

Extraction matters more in the UI than in the backend because UI components get reused more. Every UI component
answers one question: **whose is it?**

| Situation | Place |
|---|---|
| Used by one page only | inside the domain component |
| Domain-specific, reused by other domain components (`customer.component.tsx`) | stays with the owner, exported on purpose (the owner decides its props) |
| No domain (`phone-input`, `server-error`, `loading`) | `src/shared/` |

Smell: a `shared/` UI component importing domain types or a domain's generated api types → it belongs to that
domain. Another domain component's hooks, model or non-exported UI components are private.

## Server calls

- Only through hooks of the own domain component, typed with the generated api types.
- Loading, success and error are handled the same way in every page (project pattern, e.g. a shared
  server-error UI component).

## Tests

1. **Page first:** the page wires the UI components and holds the important behaviour. Test it with stubbed
   server responses (the project's tool).
2. **Every UI component: a render test.** Cheap and catches a crash while rendering (undefined access, bad
   props).
3. **UI component logic in isolation** (formatting, validation, conditional rendering) — like a
   `<Name>ComponentTest` in the backend.
