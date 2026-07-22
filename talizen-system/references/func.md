---
title: Talizen Func Usage
---

# Talizen Func Usage

Use Talizen Func for small project-level backend workflows that cannot be safely
done in browser code: bookings, waitlists, RSVP, lead routing, profile updates,
availability checks, protected user actions, third-party API calls with secrets,
and simple JSON-table reads/writes.

Func is project-scoped. User code must not receive, hard-code, read, or branch
on `project_id`.

## Workspace

Func files live under `/backend/func`. `/backend/func/booking.ts` maps to Func
key `booking`. Edit Funcs as files; use `run_func` only for self-tests with
sample input.

Do not use Func for normal CMS rendering, static contact forms covered by
`talizen/form`, long-running jobs, streaming, heavy file processing, custom
identity/session systems, OAuth callbacks, or token exchange. For auth UI, use
`talizen/auth` and read `references/auth.md`.

## Keys And Calls

Func keys are extensionless paths:

- Good: `booking`, `booking/admin`, `profile/settings`
- Avoid: `booking.js`, `user/auth.ts`, `auth.login`

Dots are reserved for method calls:

```ts
invoke("booking.create", input) // key booking, method create
invoke("profile/settings.update", input)
invoke("booking", input) // key booking, method main
```

Use `main` only for a single-operation Func. For related operations, export
multiple methods from one file.

## Func Code Rules

1. Export ESM functions: `export function method(input, ctx)`.
2. Put all platform runtime access behind `ctx`.
3. Import only TypeScript types from `talizen/func-runtime` when needed.
4. Validate and normalize all input inside the Func.
5. Return structured JSON for expected business states; throw only for invalid
   requests or unexpected failures.
6. Never hard-code API keys, bearer tokens, passwords, webhook secrets, or
   service credentials.
7. Do not use legacy globals such as `data`, `db`, `auth`, or `cache`.
8. Do not manually dispatch methods; export callable methods directly.
9. Func is not a full JavaScript runtime. Timer APIs such as `setTimeout` /
   `setInterval` are unsupported — do not use them for delays, polling, or
   retries inside Func.

```ts
import type { TalizenFuncContext } from "talizen/func-runtime"

export function create(input, ctx: TalizenFuncContext) {
  return ctx.db.insert("appointments", input)
}
```

Common helpers:

- `ctx.db.get/query/insert/update/delete(tableKey, ...)`
- `ctx.auth.currentUser()` and `ctx.auth.requireUser()`
- `ctx.assets.upload({ filename, mimeType, base64 })`
- `ctx.cache.get/set/del/incr/expire(...)`
- `ctx.request.host/ip/method/path`
- `ctx.response.status(code)`

Func HTTP responses return `{ "result": ... }` or `{ "error": "..." }`.
Browser `invoke()` unwraps successful `result` values and throws
`TalizenFuncError` for errors.

## Secrets And Environment

Func code reads secrets from `process.env.NAME`. The user must add variables in
the Creght Backend / Env panel at `panel/backend/env`; agents should not claim
to manage platform env vars.

Never put secrets in `talizen.config.ts`, Func files, pages, components,
examples, comments, or generated output.

## Auth In Func

Use Func auth only for protected backend actions:

```ts
import type { TalizenFuncContext } from "talizen/func-runtime"

export function create(input, ctx: TalizenFuncContext) {
  const user = ctx.auth.requireUser()
  return ctx.db.insert("orders", { ...input, userId: user.id })
}
```

React UI must use `useAuth()` from `talizen/auth`. Do not implement passwords,
sessions, login, registration, or OAuth callbacks in Func.

## JSON Tables

Func stores simple persistent project data through project JSON tables. The
table must exist before writes. Use JSON Schema only to describe/validate record
shape; do not design dynamic SQL migrations or table DDL in Func.

For user-specific data, store the platform `user.id`, not emails as identity
keys. Do not create account identity tables such as `users` or `auth_users`.

## Assets

When Func creates large runtime assets, upload them with:

```ts
ctx.assets.upload({ filename, mimeType, base64 })
```

Store returned URL/path/size metadata in JSON tables. Do not store or return
large base64 payloads.

## Payment

Payment integrations are custom server-side Func work. Keep provider secrets in
env vars, validate webhooks/signatures, and return explicit order states. There
is no built-in payment SDK.

## Calling From Pages

Use `talizen/func` from browser-side pages/components.

`invoke` defaults to a **5s** timeout (`timeoutMs: 5000`). Override it when the
Func needs longer — for example model image or video generation often needs
**60s+**. Errors such as `context deadline exceeded` or `context timeout`
usually mean the default timeout is too short; raise `timeoutMs`.

```tsx
import { invoke, TalizenFuncError } from "talizen/func"

try {
  const result = await invoke("booking.create", input)
} catch (error) {
  const message = error instanceof TalizenFuncError ? error.message : "Unable to submit."
}
```

Set a longer timeout when needed:

```tsx
import { invoke } from "talizen/func"

try {
  const result = await invoke("image.generate", input, {
    timeoutMs: 60000, // e.g. model image / video generation
  })
} catch (error) {
  // timeout or other Func errors
}
```

Call Func from event handlers for mutations. Keep persistent writes inside Func,
not React state alone.

## SSR Boundary

Do not call Func from `getServerSideProps`. SSR receives request/cookie helpers
for public or cookie-vary-safe first render data, but intentionally does not
expose `ctx.auth`, `ctx.func`, `ctx.db`, or `ctx.cache`.

```ts
import type { TalizenServerSideContext } from "talizen/server-runtime"

export async function getServerSideProps(ctx: TalizenServerSideContext) {
  return { props: { host: ctx.request.host } }
}
```

Keep auth, private data, writes, and cache/db logic in Func/browser flows.

## Checklist

1. Confirm CMS or `talizen/form` is insufficient.
2. Create or verify required JSON tables.
3. Add Func code under `/backend/func`.
4. Validate input and keep secrets in `process.env`.
5. Call with `invoke("key.method", input)` from UI.
6. Use `run_func` for sample backend tests when useful.
7. Run lint after page/component edits.
