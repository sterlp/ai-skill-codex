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

## Page = controller and composer

- Top-level element of a domain component, reachable via the router.
- Owns **every** server call of its UI components — loads **and** actions (save, delete, status change) —
  through hooks, plus state, loading/error handling and the workflow between UI components.
- Decides **when** to load (also lazily, e.g. only when a tab opens), requests each data source **once**
  per load/reload even if several UI components show it, and after an action takes over the result or
  reloads in **one** place.
- Has exactly one error area; every load or action error of the page and its UI components shows there.
- Grows like a service: delegate rendering to UI components until only wiring and control flow are left.
  Price: a bit more glue code — that is the cost of reusable, server-free UI components, accept it.

## UI component = function of the UI

- Renders one thing from props; reports user actions via callbacks (`onSave`, `onCancel`, `onDelete`).
- **No server calls**, neither loading nor saving, preferably no global state — it gets what it shows.
  That makes it reusable and testable without a server.
- One per file, same size rules as in the backend.
- Default suffix `*.component.tsx`; `.widget.tsx` and `.view.tsx` are synonyms — one suffix per project
  (`SKILL.md`, Step 0). Whether a UI component is reusable or page-specific is expressed by its **place**
  (see below), not by a second suffix.

## Dialogs

A dialog editing the page's data is a UI component: it gets `onSave`/`onCancel`, the page runs the request
and reloads. The dialog shows the error from the rejected `onSave` promise once, directly below its
headline, and stays open. An `afterSaved` callback for the reload is a smell — the page already knows.

Only a **self-contained** dialog showing another domain component's data loads and saves on its own (rare).
It is that domain component's exported UI, reached like any other foreign domain component.

## Tabs

A tab sharing data, requests or actions with its host is part of the host page. A tab is an **embedded
page** only if it owns all its requests and gets nothing but IDs or the host's already loaded context —
check question: could it be mounted as its own route with the same parameters? An embedded page follows
every page rule and has its own error area directly below the tab bar; a host without own requests has none.

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

- Only from pages (or embedded pages), through hooks of the own domain component, typed with the
  generated api types.
- Loading, success and error are handled the same way in every page (project pattern, e.g. a shared
  server-error UI component in the page's error area).
- Smells of a UI component that loads on its own: the same data requested several times per page load,
  `reloadKey` props to re-trigger a child's request, an error shown inside a section instead of the
  page's error area.

## Tests

1. **Page first:** the page is the glue and holds the important behaviour. Test it with stubbed server
   responses (the project's tool): every data source requested **exactly once** per load and reload,
   errors in the page's error area, reload after an action.
2. **UI components directly with props:** feed every state (loading, empty, each business status, error
   from a rejected callback) — no server stub needed, because a UI component loads nothing.
3. **UI component logic in isolation** (formatting, validation, conditional rendering) — like a
   `<Name>ComponentTest` in the backend.
