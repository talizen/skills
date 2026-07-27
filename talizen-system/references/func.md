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
`talizen/form`, detached background jobs, heavy file processing, custom
identity/session systems, OAuth callbacks, or token exchange. Bounded SSE
streaming is supported. For auth UI, use `talizen/auth` and read
`references/auth.md`.

## Keys And Calls

Func keys are extensionless paths:

- Good: `booking`, `booking/admin`, `profile/settings`
- Avoid: `booking.js`, `user/auth.ts`, `auth.login`

Dots are reserved for method calls:

```ts
invoke("booking.create", input); // key booking, method create
invoke("profile/settings.update", input);
invoke("booking", input); // key booking, method main
```

Use `main` only for a single-operation Func. For related operations, export
multiple methods from one file.

## Func Code Rules

1. Export ESM functions: `export function method(input, ctx)`.
2. Put all platform runtime access behind `ctx`.
3. Import TypeScript types from `talizen/func-runtime` when needed.
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
import type { TalizenFuncContext } from "talizen/func-runtime";

export function create(input, ctx: TalizenFuncContext) {
  return ctx.db.insert("appointments", input);
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
import type { TalizenFuncContext } from "talizen/func-runtime";

export function create(input, ctx: TalizenFuncContext) {
  const user = ctx.auth.requireUser();
  return ctx.db.insert("orders", { ...input, userId: user.id });
}
```

React UI must use `useAuth()` from `talizen/auth`. Do not implement passwords,
sessions, login, registration, or OAuth callbacks in Func.

## JSON Tables

Func stores simple persistent project data through project JSON tables. The
table must exist before writes. Use JSON Schema only to describe/validate record
shape; do not design dynamic SQL migrations or table DDL in Func.

A table's definition is a file: `/platform/table/<key>.json`, where the file name
is the table key. Create or edit it with the normal file tools — there are no
table schema tools — using the same shape as a CMS collection file:
`{ "name": …, "desc": …, "json_schema": { "type": "object", "properties": {…} } }`.
Writes are validated the same way (allowed fields only, no `key`, object schema,
non-empty `properties`, `required` must reference declared fields). Renaming is
refused; deleting is refused while the table still has records. Records stay
tool-based (`list_table_records` / `create_table_record` / …).

For user-specific data, store the platform `user.id`, not emails as identity
keys. Do not create account identity tables such as `users` or `auth_users`.

## Assets

Choose the upload path based on where the bytes originate.

For a `File` or `Blob` selected in the browser, use the CDN signed-upload client:

```tsx
import { uploadAsset } from "talizen/assets";

const asset = await uploadAsset(file, {
  onFileUploadProcess(fileName, progress) {
    console.log(fileName, progress);
  },
});

// asset.fileUrl, asset.url (same value), asset.size
```

This hashes the file, requests a short-lived upload URL, uploads the bytes
directly from the browser to CDN storage, and confirms the upload. Do not send
browser files or base64 through `invoke()` just to call `ctx.assets.upload`.
When passing a `Blob` instead of a `File`, set `{ fileName: "image.webp" }`.
Signed browser uploads are available on published site domains; preview domains
currently reject them.

When the bytes are generated inside Func and cannot originate in the browser,
use:

```ts
const asset = ctx.assets.upload({ filename, mimeType, base64 });
// asset.fileUrl === asset.url
```

`ctx.assets.upload()` returns `{ fileUrl, url, size }`; `url` is a compatibility
alias of `fileUrl`. It does not expose the internal storage path. Store the URL
and size when needed. Do not store or return large base64 payloads. Func asset
uploads are limited to 20 MiB; prefer browser signed upload for user-selected
files.

## Payment

Payment integrations are custom server-side Func work. Keep provider secrets in
env vars, validate webhooks/signatures, and return explicit order states. There
is no built-in payment SDK.

## Calling From Pages

Use `talizen/func` from browser-side pages/components.

`invoke` defaults to **5s**; the runner's default maximum is **300s**. A bounded
third-party request may use a longer timeout; detached background work may not.

For `context deadline exceeded` / `context timeout`, before editing:

1. Inspect the page's actual `invoke(..., { timeoutMs })`.
2. Re-run the same input and model with a larger timeout. `run_func.timeout_ms`
   controls only that self-test; it is not evidence of a platform hard limit.
3. If the larger timeout succeeds, raise the caller's `timeoutMs` and verify the
   production path. Do not shorten the requested output, reduce `max_tokens`,
   switch models/providers, create tables, or add background jobs as the fix.

```tsx
import { invoke, TalizenFuncError } from "talizen/func";

try {
  const result = await invoke("booking.create", input);
} catch (error) {
  const message =
    error instanceof TalizenFuncError ? error.message : "Unable to submit.";
}
```

Set a longer timeout when needed:

```tsx
import { invoke } from "talizen/func";

try {
  const result = await invoke("image.generate", input, {
    timeoutMs: 60000, // e.g. model image / video generation
  });
} catch (error) {
  // timeout or other Func errors
}
```

For bounded incremental output, send SSE events in the Func and consume the
native Fetch stream; ordinary `invoke` expects one JSON result:

```ts
// /backend/func/writer.ts
export async function main(input, ctx) {
  ctx.sse.send("token", { text: "Hello" });
  return { ok: true };
}

// page/component
const response = await fetch("/func/writer?stream=1&timeout_ms=120000", {
  method: "POST",
  headers: { Accept: "text/event-stream", "Content-Type": "application/json" },
  body: JSON.stringify(input),
});
if (!response.ok || !response.body) throw new Error("Func stream failed");
const reader = response.body.getReader();
const decoder = new TextDecoder();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  const sseChunk = decoder.decode(value, { stream: true });
  // Parse `event:` / `data:` frames separated by a blank line.
}
```

Read chunks are arbitrary byte boundaries, not complete events; buffer across
reads and split SSE frames only on a blank line.

The platform finishes with `done` or `error`. Timeout still applies, and cookies
cannot be changed after the first event.

Call Func from event handlers for mutations. Keep persistent writes inside Func,
not React state alone.

## SSR Boundary

Do not call Func from `getServerSideProps`. SSR receives request/cookie helpers
for public or cookie-vary-safe first render data, but intentionally does not
expose `ctx.auth`, `ctx.func`, `ctx.db`, or `ctx.cache`.

```ts
import type { TalizenServerSideContext } from "talizen/server-runtime";

export async function getServerSideProps(ctx: TalizenServerSideContext) {
  return { props: { host: ctx.request.host } };
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
