---
title: Talizen Form Usage
---

# Talizen Form Usage

This document describes the current Form API exported by `talizen/form`.
It is based on the latest npm package definitions from `talizen@0.0.7`.

Use `/types/form.d.ts` as the project schema source of truth, and use
`talizen/form` for the submission helper.

## types/form.d.ts file

You can find all form payload definitions in `/types/form.d.ts`.

Rules:
1. Read this file before writing form-related code.
2. Import the needed payload types from this file, for example:
   `import type { ContactForm, Newsletter } from "./types/form"`
3. Keep the runtime key and payload shape aligned with the generated type.

Example:

```ts
export declare const FormList: readonly [
  {
    key: "contact-form"
    name: "Contact form"
    Payload: ContactForm
  },
]

export interface ContactForm {
  readonly __formKey: "contact-form"
  name?: string
  email?: string
  message?: string
}
```

## talizen/form package definition

`talizen@0.0.7` currently exports:

```ts
import { type TalizenRequestOptions } from "talizen/core"

export interface FormRecord {
  readonly __formKey?: string
  [key: string]: unknown
}

export declare function submitForm<T extends FormRecord>(
  keyOrToken: T["__formKey"] | string,
  payload: T,
  options?: TalizenRequestOptions,
): Promise<"ok">
```

Important:
- The current API name is `submitForm`.
- Older docs that mention `submit` or `SubmitForm` are outdated unless the
  project defines its own wrapper.
- The first argument accepts either a form key or a token string.
- The payload is sent directly; the helper does not strip `__formKey` for you.

## Optional request config

If needed, configure Talizen globally:

```ts
import { setTalizenConfig } from "talizen/core"

setTalizenConfig({
  baseUrl: "https://www.talizen.com",
})
```

Or pass request options per call:

```ts
await submitForm<ContactForm>(
  "contact-form",
  {
    __formKey: "contact-form",
    email: "hi@example.com",
  },
  { baseUrl: "https://www.talizen.com" },
)
```

## Submit a form

```tsx
import { submitForm } from "talizen/form"
import type { ContactForm } from "./types/form"

export default function ContactSection() {
  const handleSubmit = async (event) => {
    event.preventDefault()

    const payload: ContactForm = {
      __formKey: "contact-form",
      name: "Taylor",
      email: "taylor@example.com",
      message: "Hello from Talizen!",
    }

    await submitForm<ContactForm>("contact-form", payload)
  }

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit">Send</button>
    </form>
  )
}
```

Alternative when you only have a token:

```ts
await submitForm("form-token", {
  email: "taylor@example.com",
  message: "Hello from Talizen!",
})
```

## Form guidelines

- Keep `submitForm` in client-side event handlers such as `onSubmit` or
  `onClick`.
- Prefer the generated type from `/types/form.d.ts`.
- Include `__formKey` in the typed payload when your project schema provides it.
- Use a token when the integration gives you a token instead of a stable form
  key.
- Show explicit success and error UI states after submission.

## Talizen AI Form operations

When running inside Talizen's built-in AI assistant, use the platform tools
below to inspect forms and create new form definitions:

### `list_form()`

List all forms in the current project.

Typical use:
- Discover available form keys.
- Confirm form names and schema before coding.

Response shape:

```json
{
  "total": 1,
  "list": [
    {
      "id": "123",
      "key": "contact-form",
      "name": "Contact form",
      "desc": "",
      "created_at": "2026-04-02 12:30:00",
      "updated_at": "2026-04-02 12:30:00",
      "json_schema": {}
    }
  ]
}
```

### `create_form({ key, name, desc?, json_schema? })`

Create a new form in the current project.

Input example:

```json
{
  "key": "contact-form",
  "name": "Contact form",
  "desc": "Contact entry from site footer",
  "json_schema": {
    "type": "object",
    "properties": {
      "email": { "type": "string" },
      "message": { "type": "string" }
    },
    "required": ["email"]
  }
}
```

Response shape:

```json
{
  "id": "123"
}
```

> Note: form log tools may be temporarily unavailable depending on platform
> registration strategy.

### (Temporarily disabled) `list_form_log({ key, limit?, offset? })`

List submission logs for a specific form key.

Input example:

```json
{
  "key": "contact-form",
  "limit": 20,
  "offset": 0
}
```

Response shape:

```json
{
  "total": 2,
  "list": [
    {
      "id": "456",
      "form_id": "123",
      "uid": "u_abc",
      "ua": "Mozilla/5.0",
      "ip": "127.0.0.1",
      "form_url": "/contact",
      "body": {
        "email": "taylor@example.com",
        "message": "Hello"
      },
      "created_at": "2026-04-02 12:35:00",
      "updated_at": "2026-04-02 12:35:00"
    }
  ]
}
```

### (Temporarily disabled) `get_form_log({ key, id })`

Get one specific submission log by form key and log id.

Input example:

```json
{
  "key": "contact-form",
  "id": "456"
}
```

Response shape:

```json
{
  "item": {
    "id": "456",
    "form_id": "123",
    "uid": "u_abc",
    "ua": "Mozilla/5.0",
    "ip": "127.0.0.1",
    "form_url": "/contact",
    "body": {
      "email": "taylor@example.com",
      "message": "Hello"
    },
    "created_at": "2026-04-02 12:35:00",
    "updated_at": "2026-04-02 12:35:00"
  }
}
```

If the form key or log id is not found, the tool may return an `error` field in
the response.

### Recommended order in AI tasks

1. Call `list_form()` first to discover existing form keys.
2. Call `create_form({...})` when a new form is required.
