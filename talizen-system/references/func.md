---
title: Talizen Func Usage
---

# Talizen Func Usage

Use Func for small project-level backend workflows: booking, RSVP, lead capture,
registration/profile actions, availability checks, and JSON-table reads/writes.

Func and table data are project-scoped. Never ask for, hard-code, expose, or
send `project_id` / `site_id`; the platform supplies that context internally.

## Agent Tool Flow

When implementing a Func-backed feature:

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

- Good: `booking`, `user/auth`, `admin/report`
- Avoid: `booking.js`, `auth.login`

Call format:

- `invoke("booking", input)` calls key `booking`, method `main`
- `invoke("booking.create", input)` calls key `booking`, method `create`
- `invoke("user/auth.login", input)` calls key `user/auth`, method `login`

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

Use platform auth helpers inside Func:

```js
export function profile() {
  const user = auth.requireUser()
  return { ok: true, user }
}
```

`auth.currentUser()` returns the user or `null`. `auth.requireUser()` throws
`login required` when the visitor is not logged in.

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
