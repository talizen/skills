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
