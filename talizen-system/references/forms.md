---
title: Talizen Form Usage
---

# Talizen Form Usage

This document explains how to use the `talizen/form` npm package to submit forms,
and how to create complex forms in the Talizen system.
It keeps common form submission examples for quick use, but these examples are
not a replacement for package declarations. Do not fetch package declarations
for every simple submission. Call `fetch_module_types("talizen/form")` only when
you need exact parameter or return types, use an undocumented helper, see a
package version change, or hit lint/typecheck errors.

There are two complementary type sources:

- `talizen/form` provides SDK helpers such as `submitForm`.
- `/types/form.d.ts` provides the user's generated form payload types and field
  shapes.

Use `submitForm` from `talizen/form`, and type the payload with the generated
type from `/types/form.d.ts`.

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
- Call `submitForm` from client-side event handlers such as `onSubmit` or
  `onClick`.
- Type the submitted payload with generated types from `/types/form.d.ts`.
  This file is system-generated and cannot be edited.
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

NOTE: This file is system-generated and cannot be edited. A form's definition is
a file: `/platform/form/<key>.json`, where the file name is the form key used at
runtime. Edit it with the normal file tools (`list_files` / `read_file` /
`write_file` / `replace_file`) — there are no form schema tools — then re-read
this file for the latest generated types.

```json
{
  "name": "Contact form",
  "desc": "Homepage contact",
  "json_schema": {
    "type": "object",
    "required": ["email"],
    "properties": {
      "email": { "type": "string" },
      "message": { "type": "string" }
    }
  }
}
```

Writes are validated: only `name` / `desc` / `json_schema` are allowed (no `key`
— the file name is the key), `json_schema.type` must be `object`, `properties`
must be non-empty, and every `required` entry must exist in `properties`.
Notification settings live outside the file and are preserved across writes.
Renaming and deleting are refused (deleting would discard submission logs) — do
those in the platform console. Submissions stay tool-based.

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
