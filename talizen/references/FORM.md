---
title: Talizen Form Usage
---

# Talizen Form Usage

This document describes how to submit data with `talizen/form` and how to use `/types/form.d.ts` as the source of truth for payload fields.

## types/form.d.ts file

You can find all form payload definitions in `/types/form.d.ts`.

Rules:
1. Read this file before writing form-related code.
2. Import the needed payload types from this file, for example:
   `import type { ContactForm, Newsletter } from "./types/form"`
3. Use the generated `__formKey` literal to keep the runtime key and payload type aligned.

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

```ts
interface BaseFormPayload {
  readonly __formKey: string
}

export declare function submit<T extends BaseFormPayload>(
  key: T['__formKey'],
  data: Omit<T, '__formKey'>,
): Promise<void>
```

## Submit a form

```tsx
import { submit } from 'talizen/form'
import type { ContactForm } from './types/form'

export default function ContactSection() {
  const handleSubmit = async (event) => {
    event.preventDefault()

    const payload: Omit<ContactForm, '__formKey'> = {
      name: 'Taylor',
      email: 'taylor@example.com',
      message: 'Hello from Talizen!',
    }

    await submit<ContactForm>('contact-form', payload)
  }

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit">Send</button>
    </form>
  )
}
```

Guidelines:
- Keep `submit` in client-side event handlers such as `onSubmit` or `onClick`.
- Send plain serializable values only.
- Reuse the generated type so field names stay correct.
- Show a success or error state in the UI after submission.
