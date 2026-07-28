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
- `ctx.email.send/sendCode/verifyCode(...)` (requires an email integration)
- `ctx.request.host/ip/method/path`
- `ctx.response.status(code)`

Web Crypto is available through the global `crypto`. AES-GCM and AES-CBC
encryption/decryption are supported; AES-CBC uses a 16-byte IV and automatic
PKCS#7 padding. Do not import `node:crypto`.

Ordinary Func returns produce `{ "result": ... }` or `{ "error": "..." }`.
Browser `invoke()` unwraps successful `result` values and throws
`TalizenFuncError` for errors. For webhooks or other callers that require an
exact HTTP body, return the global Web-compatible `Response`; it bypasses the
JSON envelope:

```ts
export async function notify(_input, ctx) {
  await verifyWebhook(await ctx.request.text());
  return new Response("success");
}
```

`new Response("success")` returns status 200 with
`text/plain;charset=UTF-8`. Pass `{ status, headers }` only when customization
is required. `Response` is a runtime global and must not be imported as a
runtime value; `talizen/func-runtime` exports type-only `Response` and
`ResponseInit` aliases for explicit annotations. JSON, form-encoded, text, and binary POST bodies can reach Func;
non-JSON requests receive `{}` as `input` while exact bytes remain available
through the one-shot `ctx.request.text()` / `arrayBuffer()` readers. Use native
`fetch()` rather than `invoke()` for a method that returns `Response`.

## Secrets And Environment

Func code reads secrets from `process.env.NAME`. The user must add variables in
the Creght Backend / Env panel at `panel/backend/env`; agents should not claim
to manage platform env vars.

Never put secrets in `talizen.config.ts`, Func files, pages, components,
examples, comments, or generated output.

Some third-party services have a managed **integration** instead: the user
connects them once in the Creght Backend / Integrations panel at
`panel/backend/integrations`, and the credential stays server-side rather than
being exposed as an env var. Email (Resend) works this way — see
"Email And Verification Codes". Agents should not claim to manage integrations
either; ask the user to connect one.

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

## Email And Verification Codes

Sending email and email verification codes go through `ctx.email`. This requires
the user to have connected an email integration (currently Resend) in
`panel/backend/integrations`. The provider credential stays on the server:
**never write an API key, `fetch("https://api.resend.com/...")`, or an
`Authorization` header for email in Func code.**

```ts
import type { TalizenFuncContext } from "talizen/func-runtime";

export function sendLoginCode(input, ctx: TalizenFuncContext) {
  ctx.email.sendCode({ to: input.email, scene: "login" });
  return { ok: true };
}

export function verifyLoginCode(input, ctx: TalizenFuncContext) {
  const ok = ctx.email.verifyCode({
    to: input.email,
    scene: "login",
    code: input.code,
  });
  if (!ok) throw new Error("invalid or expired code");
  return { ok: true };
}
```

`ctx.email.send({ to, subject, html, text })` sends an arbitrary message and
returns `{ id, provider }`; `to` accepts one address or an array.

The platform already enforces code generation, expiry, single-use consumption,
constant-time comparison, the wrong-attempt cap, and per-recipient rate limiting.
**Do not reimplement any of it** — no `Math.random()` codes, no `ctx.cache`-based
code storage, no hand-rolled attempt counters. `scene` namespaces codes by
purpose, so a login code and a password-reset code for the same address never
collide.

If `ctx.email` reports that no integration is configured, verified, or enabled,
that is a user action in `panel/backend/integrations` — do not work around it by
calling a provider API directly.

A verified code is proof of mailbox control, not a session. Establish login state
through project auth; see "Auth In Func".

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
Signed browser uploads are available on both preview and published site domains.

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
is no built-in payment SDK, and unlike email there is **no managed payment
integration and no `ctx.payment`** — payment providers differ too much to share
one interface, so write the checkout and webhook logic as ordinary Func code. A payment callback must verify the provider
signature, merchant/app identity, local order, amount, and paid state before an
idempotent update, then return the provider's exact acknowledgement with
`Response`.

For Alipay OpenAPI AES content encryption, Base64-decode the console AES key,
import it as `AES-CBC`, and use a 16-byte zero IV. Encrypt the UTF-8
`biz_content`, Base64-encode the ciphertext, set `encrypt_type=AES`, then RSA2
sign the complete request parameters. AES does not replace RSA2 signing or
asynchronous-notification verification. A standard `alipay.trade.page.pay`
asynchronous notification remains form data: verify it with RSA2 and do not
AES-decrypt the whole body. Decrypt only a field explicitly documented as
encrypted. Verify an encrypted OpenAPI response's RSA2 signature before
decrypting it.

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
5. For email or verification codes, use `ctx.email` and require an email
   integration; never call a mail provider API directly.
6. Call with `invoke("key.method", input)` from UI.
7. Use `run_func` for sample backend tests when useful.
8. Run lint after page/component edits.
