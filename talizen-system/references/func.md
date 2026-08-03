---
title: Talizen Func Usage
---

# Talizen Func Usage

Talizen Func is the project-scoped backend runtime for work that cannot be done
safely in browser code: bookings, waitlists, RSVP, lead routing, profile
updates, availability checks, protected user actions, third-party API calls with
secrets, webhooks, payment, and simple JSON-table reads/writes.

This file is **not** the Func API reference. It holds the rules that must hold
whether or not you read anything else. The API surface — `ctx` signatures,
return shapes, integration options, defaults and limits — lives in the live
documentation below and changes with each platform release.

## Live Documentation

Single stable entry point; fetch it first, then the topic doc it lists:

```
https://www.creght.cn/api.md
```

Any page path also serves markdown with a `.md` suffix.

1. Before writing or editing Func code, open the index and read the doc for the
   topic at hand. Do not answer from memory of a `ctx` signature, default,
   limit, or config field — those are exactly what the docs update.
2. For anything involving a **third-party service, an integration, payment,
   email or verification codes, or any Func capability not stated below**: check
   the index for a matching doc *before* writing provider HTTP calls. Only when
   the index has no such doc is the capability genuinely unwrapped by the
   platform — then it is ordinary Func code with the credential in
   `process.env`.
3. If the fetch fails, do not invent an API surface. Follow only the rules
   below, and tell the user the documentation is unreachable.

## Invariants

Scope and structure

- Confirm CMS or `talizen/form` is insufficient before reaching for Func. Func
  is not for content rendering, static contact forms, detached background jobs,
  heavy file processing, timer polling, unbounded streams, self-built
  account/session systems, OAuth callbacks, or token exchange. Bounded SSE
  streaming is supported.
- Func is project-scoped. User code must not receive, hard-code, read, or branch
  on `project_id`, `site_id`, or internal table IDs.
- Files live under `/backend/func`. The extensionless path is the key
  (`/backend/func/booking.ts` → `booking`); dots are reserved for methods
  (`invoke("booking.create", input)`). Export callable methods directly — never
  hand-write a dispatcher. Use `main` only for a single-operation Func.
- Public HTTP path is `/func/<key>`. Do not take `/func/*` as a page route.

Runtime

- Reach every platform capability through `ctx`. Never use the legacy globals
  `data`, `db`, `auth`, `cache`.
- Func is not a full Node.js runtime: no Node built-ins (including
  `node:crypto`), and no `setTimeout` / `setInterval` — they cannot be used for
  delays, polling, or retries.
- Validate and normalize all input inside the Func. Return structured JSON for
  expected business states; throw only for invalid requests or unexpected
  failures.
- Import types from `talizen/func-runtime` with `import type`. Func has no
  module loader, so a runtime value import from it always fails.

Secrets and integrations

- Never hard-code an API key, bearer token, password, webhook secret, or service
  credential — not in Func files, `talizen.config.ts`, pages, components,
  examples, comments, or generated output. Secrets come from `process.env`.
- Env vars and integrations are configured by the **user** in
  `panel/backend/env` and `panel/backend/integrations`. Never claim to have
  managed them; ask the user.
- Where the platform has a managed integration, the credential stays server-side
  and the capability is used through `ctx`. Never call the provider's HTTP API or
  write an `Authorization` header for such a capability, and never work around a
  "not configured / not verified / disabled" error by going direct — that is a
  user action in the panel.
- **Which capabilities are managed is answered by the index, not by this file.**
  Email, verification codes and payment are; the list grows with each release.
  Read the matching doc before writing anything for them — the platform already
  enforces exactly the parts that go silently wrong when hand-rolled (code
  generation, expiry, single-use consumption, constant-time comparison, attempt
  caps, rate limiting; signature verification and merchant checks for payment),
  and reimplementing any of it is a security regression, not a fallback.
- Two consequences of that hold regardless of what the docs say: a verified code
  proves mailbox control, **not** a session; and a code you sent for the site's own
  purposes is not a registration proof — registration runs on the platform's
  reserved scene through `ctx.verify`.
- When registration is routed through Func (`register_entry: "func"`),
  **verification is performed by your code and the platform checks nothing**:
  `ctx.auth.register` takes no code, no ticket, and no "already verified" flag.
  Confirm the code first and return early if it fails — reaching `register` is
  itself the decision that verification passed. There is no project setting to
  turn verification on for this path, so never tell the user to enable one.
- A callback the platform does **not** wrap for you — a payment provider it has no
  integration for, or any third-party webhook — must verify the provider signature
  **over the raw body**, then the merchant identity, your local order, the amount
  and the paid state, before an idempotent update, and return the provider's exact
  acknowledgement string. Re-serialising the body before verifying breaks the
  signature; trusting a browser redirect parameter instead of the callback is how
  free goods get shipped.

Data and identity

- Table definitions are files: `/platform/table/<key>.json`, file name = table
  key. Create and edit them with the normal file tools; there are no table
  schema tools. Records stay tool-based (`list_table_records`,
  `create_table_record`, …). The table must exist before writes.
- Store the platform `user.id` as the ownership key. Never use an email as an
  identity key, and never create account identity tables such as `users` or
  `auth_users`.
- Use `ctx.auth.requireUser()` for protected backend actions. Auth UI uses
  `useAuth()` from `talizen/auth` (see `references/auth.md`); never hand-roll
  password hashing, session tokens, or OAuth callbacks in Func.
- **Sign-in can be a Func**: `ctx.auth.login(ref)` issues a session for an existing
  user, which is the only way to build passwordless (emailed-code) login or to apply
  your own rules — bans, required onboarding steps, tenant checks — at sign-in. The
  platform still mints the session; Func never sees the token, and direct
  `useAuth().login()` keeps working alongside it.
  **The ref must come from a fact the server just verified** — the user returned by
  `ctx.users.find` after `ctx.email.verifyCode` succeeded, or the account whose
  password `ctx.users.checkPassword` just accepted. Never
  `ctx.auth.login({ email: input.email })`: that is "whoever the browser claims to
  be", and unlike a bad password change it leaves the real user no symptom.
  Every session issued this way is recorded with the Func file that issued it.
- **Password reset / change has no platform endpoint — write it in Func.** Two
  steps: `ctx.users.find({ email })` then `ctx.email.sendCode`, and later
  `ctx.email.verifyCode` then `ctx.users.setPassword({ email, password })`.
  `setPassword` takes no code or ticket: verifying first *is* the authorization.
  Three hard rules, because getting them wrong is account takeover or an
  enumeration oracle: look the user up **before** sending a code; return the
  **same** response whether or not the user exists; never pass `setPassword`'s
  404 back to the browser. The platform hashes and revokes all of that user's
  sessions — including the caller's, so route them to login afterwards.
- A **signed-in** user changing their own password confirms the old one with
  `ctx.users.checkPassword`, and must be pointed at by the id from
  `ctx.auth.requireUser()` — never a `userId`/`email` taken from the request body,
  which would let any logged-in visitor change somebody else's password. Do not
  "verify" the old password by calling login from page code: the Func can be
  called directly, so that check is not a boundary.

Boundaries

- Do not call Func from `getServerSideProps`. SSR intentionally exposes no
  `ctx.auth`, `ctx.func`, `ctx.db`, or `ctx.cache`.
- Call Func from browser event handlers for mutations, and keep persistent
  writes in Func — never fake persistence in React state.
- A browser-selected `File`/`Blob` uploads through `uploadAsset` from
  `talizen/assets`. Do not base64-encode browser files through `invoke()` just
  to reach `ctx.assets.upload`, which is only for bytes generated inside Func.
  Never store or return large base64 payloads.

Timeouts

- `invoke` has a short default timeout; bounded long work needs an explicit
  `timeoutMs` at the call site. Detached background work is not an option.
- For `context deadline exceeded` / `context timeout`: inspect the page's actual
  `invoke(..., { timeoutMs })`, then re-run the same input and model with a
  larger timeout. `run_func.timeout_ms` affects only that self-test and is never
  evidence of a platform hard limit. Never "fix" a deadline by shortening the
  requested output, lowering `max_tokens`, switching models or providers,
  creating tables, or faking a background job.

## ctx Surface

**Not listed here on purpose.** The set of `ctx` namespaces grows with each
platform release, so any copy in this file goes stale silently — and a stale list
is worse than none: it tells you a capability does not exist when it does, and you
then write `fetch` against a provider the platform already wraps. Get the current
set from the index above; `ctx` is also fully typed, so
`import type { TalizenFuncContext } from 'talizen/func-runtime'` gives it to you in
the editor.

The runtime facts around `ctx` do belong here, because they hold whatever the
surface looks like:

- Reach capabilities only through `ctx` — never the legacy globals, never a
  hand-rolled equivalent of something the platform wraps.
- `fetch`, `Response`, `TextDecoder` and Web Crypto (`crypto`) are globals; a
  returned `Response` bypasses the JSON envelope, which is what webhook and payment
  callbacks need in order to answer with the provider's exact acknowledgement.
- `ctx.trace_id` correlates one call across logs.

## Workflow

1. Confirm CMS or `talizen/form` cannot do it.
2. Read the relevant doc from the index above.
3. Create or verify the JSON tables under `/platform/table`.
4. Write the Func under `/backend/func`; validate input, secrets from
   `process.env`.
5. Call it with `invoke("key.method", input)`; native Fetch + SSE only for
   streaming.
6. Use `run_func` with sample input for backend self-tests.
7. Run `lint` after page/component edits, then verify the real path: success,
   expected business failure, signed-out, third-party failure, timeout.

On any runtime failure, read `references/error-handling.md`.
