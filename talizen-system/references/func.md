---
title: Talizen Func Usage
---

# Talizen Func Usage

Use Talizen Func for small project-level backend workflows that cannot be done
safely in browser code: appointments, waitlists, RSVP, lead routing,
registration-like submissions, profile updates, availability checks, and simple
JSON-table reads or writes.

Func is project-scoped. The platform runs each Func inside the current project
context, so user code must not receive, hard-code, read, or branch on
`project_id`. A site may call a Func through its own domain, but the Func and
its JSON data belong to the project.

## When To Use Func

Use Func when the user asks for behavior such as:

- Save an appointment, booking, RSVP, signup, or lead.
- Check availability before creating a record.
- Update an existing JSON-table record.
- Read a small list of project records for an interactive UI.
- Run validation that must not be trusted to the browser.

Do not use Func for:

- Normal content rendering that CMS already covers.
- Static contact forms that only need `talizen/form`.
- Long-running jobs, streaming, file processing, or heavy computation.
- Direct database, Redis, network, token, or credential access from page code.
- Custom identity/session systems, including OAuth callbacks and token
  exchange. Use the platform project auth APIs instead.

## Func Keys And Methods

Each Func has a project-unique key. Treat the key like an extensionless file
path:

- Good: `booking`, `booking/admin`, `user/auth`
- Avoid: `booking.js`, `user/auth.ts`, `auth.login`

Dots are reserved for method invocation. The client call format is:

```ts
invoke("file.method", input)
```

Examples:

- `invoke("booking.create", input)` calls Func key `booking`, method `create`.
- `invoke("user/auth.login", input)` calls Func key `user/auth`, method `login`.
- `invoke("booking", input)` calls Func key `booking`, method `main`.

If a Func contains only one operation, implement `main`. If a feature has
closely related operations, put multiple exported methods in one Func file.

## Writing Func Code

Current Func runtime executes synchronous JavaScript with `goja`.

Rules:

1. Export methods with CommonJS-style `exports.method = function(input, ctx)`.
2. Use `exports.main = function(input, ctx)` for the default method.
3. Do not use `async`, `await`, `Promise`, ESM `import`, or ESM `export`.
4. Do not write a manual `main` dispatcher. Export each callable method
   directly.
5. Validate and normalize all input inside the Func.
6. Return structured JSON. Use expected business results such as
   `{ ok: false, code: "slot_taken" }` instead of throwing.
7. Throw only for unexpected failures or invalid requests that should surface as
   errors.

Available globals:

- `data.get(tableKey, id)`
- `data.query(tableKey, query)`
- `data.insert(tableKey, body)`
- `data.update(tableKey, id, body)`
- `data.delete(tableKey, id)`
- `auth.currentUser()`
- `auth.requireUser()`
- `console.log/warn/error`
- `ctx.user_id`, `ctx.trace_id`, and optional `ctx.extra`

`ctx.project_id` and `ctx.site_id` are intentionally not visible to Func code.
The platform uses them internally for project isolation.

Redis helpers may exist behind the same Func runtime, but only use them after
the project confirms Redis proxy support is enabled. For ordinary booking and
lead flows, prefer `data.*`.

## Project Auth

Use `talizen/auth` on the page for login state:

```ts
import {
  currentUser,
  listAuthProviders,
  login,
  loginWithOAuth,
  logout,
  register
} from "talizen/auth"

await register(input)
await login(input)
const providers = await listAuthProviders()
await loginWithOAuth("github", { redirectUrl: "/account" })
const user = await currentUser()
await logout()
```

Before writing auth payloads, read the `talizen/auth` type definitions from the
`talizen` version used by the current project. Do not create user tables, store
passwords, implement sessions, or write OAuth callback Funcs.

Inside Func, use the injected auth helper:

```js
exports.create = function(input) {
  const user = auth.requireUser()
  return data.insert("appointments", { userId: user.id, date: input.date })
}
```

## JSON Tables

Func stores persistent project data through project JSON tables. A table must
exist before a Func writes to it. The table key is the string passed to
`data.*`, for example `appointments`.

Use JSON Schema only to describe and validate table record shape. Do not design
Func features that require dynamic SQL migrations or table DDL.

Common query shape:

```js
data.query("appointments", {
  where: { email: "person@example.com" },
  limit: 20,
  offset: 0,
  order_by: "created_at desc"
})
```

Keep query payloads small and predictable. Use simple top-level fields for
filters that the platform can index later.

## Minimal Booking Example

Func key: `booking`

```js
function required(value, field) {
  const text = String(value || "").trim()
  if (!text) throw new Error(field + " is required")
  return text
}

exports.create = function(input) {
  const date = required(input.date, "date")
  const time = required(input.time, "time")
  const existing = data.query("appointments", {
    where: { date: date, time: time, status: "confirmed" },
    limit: 1
  })
  if (existing.length > 0) {
    return { ok: false, code: "slot_taken", message: "This time is unavailable." }
  }

  const inserted = data.insert("appointments", {
    name: required(input.name, "name"),
    email: required(input.email, "email").toLowerCase(),
    date: date,
    time: time,
    status: "confirmed",
    createdAt: new Date().toISOString()
  })
  return { ok: true, id: inserted.id }
}
```

Expected table key: `appointments`. Its JSON Schema should include at least
`name`, `email`, `date`, `time`, `status`, and `createdAt`.

## Calling Func From A Page

Use the SDK exported by `talizen`. For exact declarations, fetch package types
only when needed:

```ts
fetch_module_types("talizen")
```

Common client-side call:

```tsx
import { invoke, TalizenFuncError } from "talizen"

type BookingResult =
  | { ok: true; id: string }
  | { ok: false; code: "slot_taken"; message: string }

try {
  const result = await invoke<BookingResult>("booking.create", {
    name,
    email,
    date,
    time
  })
  if (!result.ok) {
    setMessage(result.message)
    return
  }
  setMessage("Your booking is confirmed.")
} catch (error) {
  setMessage(error instanceof TalizenFuncError ? error.message : "Unable to book.")
}
```

Rules for page code:

- Call Func from event handlers for mutations.
- Keep all persistent writes inside Func, not in React state alone.
- Import from `talizen`, not from a relative SDK path.
- Use `invoke("file.method", input)` for normal use.
- Use `invoke("file", input)` only when the Func exports `main`.
- Handle expected `{ ok: false, code, message }` business responses separately
  from thrown errors.
- Do not include `project_id`, `site_id`, internal tokens, or table IDs in the
  client payload.

## Agent Checklist

Before building a Func-backed feature:

1. Identify the project JSON table keys required by the workflow.
2. Create or update the JSON Schema for those tables if the platform tools
   expose table management.
3. Create a project-level Func with an extensionless key.
4. Export one method per operation.
5. Call the method with `invoke("key.method", input)` from the page.
6. Run lint after editing page/component code.

For a simple appointment workflow, the normal shape is:

- Table: `appointments`
- Func key: `booking`
- Methods: `create`, optionally `listByEmail` or `cancel`
- Client call: `invoke("booking.create", input)`
