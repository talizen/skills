---
title: Talizen Form Usage
---

# Talizen Form Usage

This document describes the current Form API exported by `talizen/form`.
It is based on the latest npm package definitions from `talizen`.

Use `/types/form.d.ts` as the project schema source of truth, and use
`talizen/form` for the submission helper.

## Build a Form

### String field extensions (JSON Schema conventions)

Platform forms and CMS share the same JSON Schema conventions: the root uses `type: "object"`, fields live under `properties`, and required fields are listed in `required`.
**Core rule: extension fields should use `type: "string"` and express semantics via `format`, `contentMediaType`, and `accept`.**

Minimal mapping (these are the only ones AI needs to remember):

- **plain**: `{ "type": "string" }`
- **url**: `{ "type": "string", "format": "uri" }`
- **image**: `{ "type": "string", "format": "uri", "contentMediaType": "image/*", "accept": "image/*" }`
- **video**: `{ "type": "string", "format": "uri", "contentMediaType": "video/*", "accept": "video/*" }`
- **file**: `{ "type": "string", "format": "uri", "contentMediaType": "application/octet-stream" }` (can be narrowed to a specific MIME type, for example `application/pdf`)
- **richtext**: `{ "type": "string", "contentMediaType": "text/html" }` (do not add `format: "uri"`)

### Example

```json
{
  "type": "object",
  "properties": {
    "title": { "type": "string", "title": "Title" },
    "avatar": {
      "type": "string",
      "format": "uri",
      "title": "Avatar",
      "contentMediaType": "image/*",
      "accept": "image/*"
    },
    "intro": {
      "type": "string",
      "title": "Intro",
      "contentMediaType": "text/html"
    }
  },
  "required": ["title"]
}
```

## Submit a Form

- Use `submitForm` from the npm package `talizen/form` to submit forms.
  - The first argument accepts either a form key or a token string.
- Call `submitForm` from client-side event handlers such as `onSubmit` or
  `onClick`.
- Prefer generated types from `/types/form.d.ts`. (This file is
  system-generated and cannot be edited.)
- Show explicit success and error UI states after submission.
- If a field is typed as **File** in `/types/form.d.ts`, pass a raw `File` or
  `Blob` to `submitForm`. (The function uploads file values automatically.)
- For image, video, or generic file fields, use the corresponding form input
  components (for example, Image Input, Video Input, and File Input). See
  `Build a Form` -> `String field extensions` for details.

Example:

```tsx
import { submitForm } from "talizen/form";
import type { ContactForm } from "./types/form";

export default function ContactSection() {
  const handleSubmit = async (event) => {
    event.preventDefault();

    const payload: ContactForm = {
      __formKey: "contact-form",
      name: "Taylor",
      email: "taylor@example.com",
      message: "Hello from Talizen!",
      file: new File([], "file.txt"),
    };

    await submitForm<ContactForm>("contact-form", payload);
  };

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit">Send</button>
    </form>
  );
}
```

### `/types/form.d.ts`

All form payload definitions are available in `/types/form.d.ts`.

NOTE: This file is system-generated and cannot be edited. Call
`create_form({...})` when a new form is required. The file is updated after
successful form creation.

Rules:

1. Read this file before writing form-related code.
2. Import the needed payload types from this file, for example:
   `import type { ContactForm, Newsletter } from "./types/form"`
3. Keep the runtime key and payload shape aligned with the generated type.

Example:

```ts
export declare const FormList: readonly [
  {
    key: "contact-form";
    name: "Contact form";
    Payload: ContactForm;
  },
];

export interface ContactForm {
  name?: string;
  email?: string;
  message?: string;
  file?: File;
}
```
