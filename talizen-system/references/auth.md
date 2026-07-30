---
title: Talizen Auth Usage
---

# Talizen Auth Usage

Talizen auth is project-scoped platform auth for site visitors. Use the
browser-side `talizen/auth` SDK for login, registration, logout, current-user
state, OAuth/social login, and protected UI.

Do not build auth with Func or JSON tables. Do not store passwords, session
tokens, OAuth tokens, OAuth callback state, or project auth identities in custom
tables.

## Core Rule

React UI must use `useAuth()` for auth state and auth mutations:

```tsx
import { useAuth } from "talizen/auth"

export function AccountBar() {
  const { user, loading, login, register, logout, updateProfile } = useAuth()

  if (loading) return <p>Loading...</p>
  if (!user) return <button onClick={() => login("alice", "secret")}>Sign in</button>

  return <button onClick={() => logout()}>Sign out</button>
}
```

`useAuth()` returns `{ user, loading, error, isAuthenticated, isInitialized,
refresh, login, register, logout, updateProfile }`. Hook mutations update all
subscribers, so UI depending on `useAuth()` refreshes without remounting the
whole app.

Do not import top-level `login` or `logout` from `talizen/auth`; they are not
public SDK exports. Use `useAuth().login()` and `useAuth().logout()` in React
components.

`account` is the login identifier. It does not have to be an email address.

## SSR And Func

`getServerSideProps` does not expose `ctx.auth`. Do not import `talizen/auth`,
and do not call login, registration, logout, current-user reads, or Func from
`getServerSideProps`.

Why: render HTML is cached by route, query, dependencies, and learned
cookie-vary names. A direct auth/user read would make HTML depend on hidden
user-specific state and can break cache safety. Cookie reads are explicit
through `ctx.cookies.get(...)`; cookie writes are no-store.

For protected backend actions, use Func for the business action and read the
signed-in project user with `ctx.auth.currentUser()` or
`ctx.auth.requireUser()`. Func auth is not for implementing login,
registration, logout, or OAuth.

A password-gated page follows the same boundary: render a public gate, verify
through Func/API, set a signed access cookie, then fetch the protected content
from Func/API.

## Registration And Profile

In React UI, prefer `useAuth().register(...)` and
`useAuth().updateProfile(...)` so subscribers update automatically. The
top-level `register(input)` helper exists for low-level non-hook flows only.

Registration requires `account` and `password`; `email`, `phone`, `name`,
`avatar`, and `profile` are optional profile fields.

## Verified Contacts

Whether registration requires a proven email (or phone) is **project policy**,
not a call argument. Read it at `/platform/auth.json`:

```json
{
  "open_register": true,
  "register_requires": ["email"],
  "register_entry": "sdk",
  "expose_account_exists": false
}
```

- `register_requires` empty → nothing changes; `register(...)` behaves as before.
- `register_requires: ["email"]` → the page must prove the address first. The
  proof lives server-side; the browser only carries an httpOnly cookie, so
  **never try to read, store, or pass a proof yourself**:

```tsx
import { useAuth, startVerification, confirmVerification } from "talizen/auth"

await startVerification({ channel: "email", to: email, purpose: "register" })
await confirmVerification({ channel: "email", to: email, purpose: "register", code })
await useAuth().register({ account: email, email, password })   // no code argument
```

- The proof is bound to that exact address and to `purpose`, is single-use, and
  expires in 10 minutes. Registering with a different address than the one proven
  fails.
- `register_entry: "func"` → direct `useAuth().register(...)` **and** direct
  `startVerification(...)` both return 403; the project registers through its own
  Func (`ctx.auth.register`). Use it when the site has its own server-side rules
  (invite codes, domain allowlists) — rules written in page code are only advisory.
  On that path verification is performed by the Func's own code and there is no
  project setting for it, so the settings below describe page-code registration
  only.
- There is no `email_verified_at` on the user. Do not read, display, or gate on
  one, and do not add a table column to imitate it — verification is a step at
  registration, not a per-user flag.
- Do not gate login on a verified email; existing users would be locked out.
  Gate specific actions inside Func instead.
- `expose_account_exists` decides what happens when the address already has an
  account. Off (default): the request gets the **same response as any other** and
  the address receives an "you already have an account" email instead of a code —
  so nobody can use the endpoint to probe which emails have accounts. On:
  `startVerification` fails with 409 and a clear message. Off is not a bug to work
  around: do not add your own "is this email taken" check in page code, which is
  exactly the probe the default prevents.

## Sign-in Through Func

`useAuth().login()` stays the normal path. A project only needs its own Func when
sign-in has to do something the SDK cannot express — **passwordless login with an
emailed code**, or server-side rules at sign-in (bans, required onboarding, tenant
checks). That Func uses `ctx.users.checkPassword` / `ctx.email.verifyCode` and then
`ctx.auth.login`; see `references/func.md`.

There is no setting that closes direct `/auth/login`, so a site can have both. Page
code calls the Func through `invoke()` and treats a resolved promise as signed in —
the session cookie arrives with that response. Every sign-in is recorded (source,
Func file, IP) and the owner can read it per user in the editor.

## Forgot / Change Password

There is **no SDK call and no platform endpoint for this** — `useAuth()` has no
`resetPassword`, and adding one is not pending. The flow is written in the site's
own Func with `ctx.users.find` / `ctx.users.checkPassword` / `ctx.users.setPassword`,
and the page just calls it through `invoke()`. See `references/func.md`.

Page code's only jobs: collect the address, then the code plus the new password,
and **show the same message whether or not the account exists** — the Func returns
an identical response by design, so do not add an "is this email registered" check
to make the UI more informative. After a successful reset the user's sessions are
gone (including the current one), so send them to the login page rather than
assuming they are still signed in.

**You may tighten this file, not loosen it.** Adding a channel, switching
`register_entry` to `func`, or closing registration is allowed. Removing a
channel, switching back to `sdk`, reopening registration, or turning on
`expose_account_exists` is rejected — tell
the user to change it in the editor under Auth settings. Turning on `email`
requires the project to already have a verified email integration; the write
fails with a message pointing at Backend → Integrations.

## OAuth

OAuth providers are configured in project Auth settings. Page code should not
hard-code client secrets, token endpoints, or callback logic.

A provider's configuration is a file: `/platform/auth/<key>.json`, where the file
name is the provider key (`github`, `google`, …). Edit it with the normal file
tools — there are no provider tools:

```json
{
  "name": "GitHub",
  "status": "enable",
  "client_id": "Iv1.xxx",
  "redirect_url": "https://example.com/api/auth/callback/github",
  "scopes": ["user:email"],
  "endpoint": {
    "auth_url": "https://github.com/login/oauth/authorize",
    "token_url": "https://github.com/login/oauth/access_token",
    "userinfo_url": "https://api.github.com/user"
  }
}
```

- **`client_secret` is never in the file** and writing it is rejected — files are
  diffable and get shipped around. After creating a provider this way, tell the
  user to set the secret in the platform console; login stays inactive until then
  (`ready_for_login` shows the current state and is read-only).
- Required on write: `name`, `client_id`, `redirect_url`, and
  `endpoint.auth_url` / `token_url` / `userinfo_url`. All URLs must be http(s).
  `status` is `enable` or `disable`.
- An existing secret is preserved across file writes.
- Renaming is refused; deleting a provider file removes the provider (config
  only, no user data).

```ts
import { getOAuthLoginUrl, listAuthProviders, loginWithOAuth } from "talizen/auth"
```

- `listAuthProviders()` returns enabled providers.
- `loginWithOAuth(provider, { redirectUrl })` redirects the browser to provider
  login.
- `getOAuthLoginUrl(provider, { redirectUrl })` returns a provider login URL
  for custom redirect flows.

`redirectUrl` must be a same-site relative path such as `/account`.
