---
title: Talizen Func Usage
---

# Talizen Func Usage

Use Func for small project-level backend workflows: booking, RSVP, lead capture,
availability checks, protected profile-adjacent business actions, and JSON-table
reads/writes.

Func and table data are project-scoped. Never ask for, hard-code, expose, or
send `project_id` / `site_id`; the platform supplies that context internally.

## Auth First Rule

Talizen has built-in platform auth. For user login, registration, logout, and
current-user state, use the platform auth SDK/API instead of Func or JSON tables.

Do not:

- create `user`, `users`, `auth_user`, or `auth_users` tables for account identity
- store passwords or password hashes in JSON tables
- write `user/auth.login`, `user/auth.register`, or similar Func methods for
  platform account login/registration
- implement cookies, sessions, tokens, or password verification in user Func code

Do:

```tsx
import { currentUser, login, logout, register } from "talizen/auth"

await register({ email, password, name })
await login({ email, password })
const user = await currentUser()
await logout()
```

Use Func only for business logic around an authenticated user, such as booking,
orders, private profile settings, or permission checks. Inside Func, read the
visitor from platform auth:

```js
export function create(input) {
  const user = auth.requireUser()
  const row = data.insert("appointments", {
    user_id: user.id,
    date: input.date,
    time: input.time,
    status: "confirmed"
  })
  return { ok: true, id: row.id }
}
```

## Agent Tool Flow

When implementing a Func-backed feature:

0. If the requested feature is login, registration, logout, current user, or
   account identity, do not create tables or Func; use `talizen/auth` in the
   client UI.
1. `list_tables`
2. `create_table` or `update_table` with `json_schema` for fields
3. `create_table_record` only when seed/test data is needed
4. `list_funcs`
5. `create_func` or `update_func`
6. `run_func` with sample input before editing client UI
7. In page code, call `invoke("file.method", input)` from `talizen`

Useful tools:

- Tables: `list_tables`, `create_table`, `update_table`, `delete_table`
- Records: `list_table_records`, `get_table_record`, `create_table_record`,
  `update_table_record`, `delete_table_record`
- Func: `list_funcs`, `get_func`, `create_func`, `update_func`, `delete_func`,
  `run_func`

## Func Keys

Treat a Func key like an extensionless file path:

- Good: `booking`, `profile/settings`, `admin/report`
- Avoid: `booking.js`, `auth.login`

Call format:

- `invoke("booking", input)` calls key `booking`, method `main`
- `invoke("booking.create", input)` calls key `booking`, method `create`
- `invoke("admin/report.generate", input)` calls key `admin/report`, method `generate`

## Func Code

Prefer ESM exports:

```js
export function main(input) {
  return { ok: true, input }
}
```

Multiple methods in one file:

```js
export function create(input) {
  const row = data.insert("appointments", input)
  return { ok: true, id: row.id }
}

export function list(input) {
  return data.query("appointments", { limit: input.limit || 20 })
}
```

Rules:

- Do not write a manual `main` dispatcher.
- Do not use `async`/`await` unless the platform explicitly adds Promise support.
- Validate user input in Func.
- Return expected business failures as `{ ok: false, code, message }`.
- Throw only for unexpected failures.

## Data Tables

Create a JSON table before writing Func code that uses it. The table key is the
first argument of `data.*`.

Minimal table schema:

```json
{
  "type": "object",
  "properties": {
    "email": { "type": "string" },
    "date": { "type": "string" },
    "status": { "type": "string" }
  }
}
```

Data APIs:

```js
data.query("appointments", { where: { status: "confirmed" }, limit: 20 })
data.get("appointments", input.id)
data.insert("appointments", { email: input.email, status: "confirmed" })
data.update("appointments", input.id, { status: "cancelled" })
data.delete("appointments", input.id)
```

Pagination:

```js
const page = Math.max(Number(input.page || 1), 1)
const pageSize = Math.min(Number(input.pageSize || 20), 100)
return data.query("book", {
  limit: pageSize,
  offset: (page - 1) * pageSize,
  order_by: "sort desc, id desc"
})
```

## Auth In Func

Use platform auth helpers inside Func only for protected business actions:

```js
export function profile() {
  const user = auth.requireUser()
  return { ok: true, user }
}
```

`auth.currentUser()` returns the user or `null`. `auth.requireUser()` throws
`login required` when the visitor is not logged in. These helpers do not create
accounts; registration and login are platform auth operations from `talizen/auth`.

## Client Call

Use the Talizen SDK:

```tsx
import { invoke } from "talizen"

const result = await invoke("booking.create", {
  email,
  date,
  time
})
```

Client code must not include `project_id`, `site_id`, table IDs, internal
tokens, or direct database calls.

## Booking Skeleton

Table: `appointments`

Func key: `booking`

```js
export function create(input) {
  const existing = data.query("appointments", {
    where: { date: input.date, time: input.time, status: "confirmed" },
    limit: 1
  })
  if (existing.length) return { ok: false, code: "slot_taken", message: "Time unavailable" }

  const row = data.insert("appointments", {
    email: String(input.email || "").toLowerCase(),
    date: input.date,
    time: input.time,
    status: "confirmed"
  })
  return { ok: true, id: row.id }
}
```
