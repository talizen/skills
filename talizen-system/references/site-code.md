# Talizen Site Code

Talizen apps are React-based websites with file-based routes under `/pages`, a
root `talizen.config.ts`, Tailwind v4 styling, generated project types, and
platform APIs for CMS, forms, metadata, import maps, previews, and publishing.

## Routing

Routing is derived from `/pages` file names by Talizen conventions:

- `/pages/Index.tsx` -> `/`
- `/pages/About.tsx` -> `/about`

For non-`Index` pages, do not guess kebab-case routes. Prefer the lowercase
canonical path returned by lint or platform validation; for example,
`/pages/BlockElementsPage.tsx` should be linked as `/blockelementspage`.

Files like `/pages/XXXX.canvas.tsx` are canvas preview entries used by the
platform editor, not normal route files to generate by hand.

For localized routing (locale prefixes and per-locale page files), see
`references/i18n.md`.

## Navigation

Use native anchors for navigation:

```tsx
<a href="/about">About</a>
```

On a multilingual site, use talizen's locale-aware `<Link>` for internal links —
it auto-prefixes the current locale, so a link never drops the visitor out of
their language (a plain `<a>` does not add the prefix). See `references/i18n.md`.

```tsx
import { Link } from "talizen"

<Link href="/about">About</Link>
```

Do not import `next/link`, `next/router`, `next/navigation`, or any other router
library. talizen's own `<Link>` is the only allowed link component.

## Data Loading

Prefer `getServerSideProps(context)` for route params and first-render data.
Read dynamic params from `context.params`; do not use client-side param hooks
when SSR params are available.

Do not read auth state in `getServerSideProps`. The render runtime deliberately
does not expose `context.auth` because HTML caching relies on explicit
dependencies and cookie-vary names; hidden user-specific auth reads would make
cache safety unpredictable. Use `useAuth()` in React UI, or Func `ctx.auth` for
protected backend actions.

Example:

```tsx
export async function getServerSideProps(context) {
  const slug = context.params?.slug;

  return {
    props: { slug },
  };
}
```

## Components

Keep page files focused on route-level composition. Put reusable UI in
`/components` or another shared components directory that already exists in the
project.

If the user asks only for a component but also wants a preview, make the
component visible in a page whenever possible. A standalone component cannot
render as a site page by itself.

## Component Registries And Effects

For carousels and slideshows, see `references/carousel.md`.

For shadcn/ui-style registry components:

- Configure `components.json` registries first.
- Use `shadcn_search_items`, `shadcn_list_items`, and `shadcn_install_item`.
- Common registries are `@spell`, `@fancy`, `@react-bits`, and
  `@talizen-sections`.
- For polished visual effects, animations, particles, cursor effects, text
  animations, hover effects, or creative backgrounds, prefer searching component
  registries before implementing custom effects.
- Ensure installed animated layers do not block content, reduce readability, or
  break responsive layouts.

Minimal `components.json` example:

```json
{
  "registries": {
    "@spell": "https://spell.sh/r/{name}.json",
    "@fancy": "https://fancycomponents.dev/r/{name}.json",
    "@react-bits": "https://reactbits.dev/r/{name}.json",
    "@talizen-sections": "https://unpkg.com/@talizen/talizen-sections/public/r/{name}.json"
  }
}
```

## Local Imports

Use relative paths for local project imports. The Talizen platform does not
support alias imports such as `@/lib/utils`; write them as relative imports from
the importing file, for example `../lib/utils`, `../../lib/utils`, or
`./lib/utils`.

Package imports and Talizen platform imports still use their normal specifiers,
such as `react`, `talizen/cms`, and import-map keys configured in
`talizen.config.ts`.

## Import Map

There are two import map views:

- `get_import_map()` returns the final effective runtime import map, including
  platform-provided modules and user-defined entries.
- `talizen.config.ts` contains only user-defined extra imports.

Edit `talizen.config.ts` `importMap.imports` when adding or changing third-party
dependencies.

Rules:

- Only add packages that are actually used.
- Prefer `esm.sh` unless compatibility requires another CDN.
- Do not add `react`, `react-dom`, or `talizen`; the platform already provides
  them.
- For React-dependent esm.sh packages, use `?external=react` so they use the
  host React copy.
- If you see duplicate-React symptoms such as `Cannot read properties of null
  (reading 'useState')` with both `react.mjs` and an esm.sh package in the stack,
  add `?external=react` to that package URL, including React wrapper subpaths
  such as `swiper/react`.
- Keep the import specifier exactly the same as the `imports` key.
- Talizen's compiler supports Vite-style local asset queries for relative imports:
  `import assetUrl from "./asset.ext?url"` returns a browser-accessible Blob URL,
  and `import source from "./file.ext?raw"` returns the original file contents as
  a string. Use these for worker source, WASM/worker helper scripts, and other
  local assets that need URL or raw-string semantics.
- For assets generated at runtime inside Func, such as AI-generated images, use
  `ctx.assets.upload({ filename, mimeType, base64 })` from the Func and store
  only the returned URL/path/size metadata. Do not store base64 payloads in JSON
  tables or return them from list endpoints.

Example:

```ts
export default {
  importMap: {
    imports: {
      "framer-motion": "https://esm.sh/framer-motion@12.34.5?external=react",
    },
  },
};
```

## talizen.config.ts

Use `export default` with a plain object. Do not import packages in this file
except type-only imports such as `import type { Metadata } from "talizen"`.
Do not use `defineConfig` from `talizen/config`.

`talizen.config.ts` can define:

- `importMap.imports` for user-defined third-party dependencies.
- `metadata` for global SEO.
- `viewport` for site-level initial viewport tags, similar to Next.js App
  Router's `viewport` object. Page-level `export const viewport` /
  `generateViewport` is not supported.
- `customCode.head` and `customCode.body` for small analytics, verification, or
  script snippets not covered by structured metadata.
- `redirects` for site-level URL redirects (see below).

Do not duplicate metadata in `customCode`; prefer structured `metadata` whenever
the field fits.

## Viewport

Configure the site's default viewport in `talizen.config.ts` with a plain
`viewport` object. Do not put viewport settings under `metadata`, and do not
write a raw `<meta name="viewport">` in `customCode.head`.

If omitted, Talizen renders:

```html
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

Example:

```ts
export default {
  viewport: {
    width: 1200,
    initialScale: null,
    maximumScale: 1,
    userScalable: false,
    interactiveWidget: "resizes-visual",
    themeColor: "#111111",
    colorScheme: "dark",
  },
};
```

Supported fields:

- `width` / `height`: string or number, e.g. `"device-width"` or `1200`.
- `initialScale` / `minimumScale` / `maximumScale`: number. Use `null` to
  remove a default field; e.g. `initialScale: null` removes
  `initial-scale=1`.
- `userScalable`: boolean; `false` emits `user-scalable=no`, `true` emits
  `user-scalable=yes`.
- `interactiveWidget`: string, e.g. `"resizes-visual"`.
- `themeColor`: emits `<meta name="theme-color">`.
- `colorScheme`: emits `<meta name="color-scheme">`.

## Redirects

Configure site-level redirects with the `redirects` array in `talizen.config.ts`.
This is the Talizen equivalent of Next.js `redirects()`. Because the config must
be a plain object, `redirects` is an array literal, not an `async` function.

Each rule has:

- `source`: path to match. Supports exact matches (`/old-page`) and a trailing
  wildcard segment (`/blog/*`).
- `destination`: target path. May be an internal path (`/new-page`), a wildcard
  backreference (`/posts/*`), an absolute URL, or a protocol-relative URL.
- `permanent`: `true` for a 308 permanent redirect (best for SEO), `false` for a
  307 temporary redirect.

Rules match in order and the first match wins; a rule whose target would match
itself is skipped. Redirects run before page routing, so they take precedence
over pages and `/public` files, and the original query string is preserved for
internal targets.

```ts
import type { Redirect } from "talizen";

export default {
  redirects: [
    { source: "/old-page", destination: "/new-page", permanent: true },
    { source: "/blog/*", destination: "/posts/*", permanent: false },
  ] satisfies Redirect[],
};
```

For per-request or conditional redirects that depend on data, return `redirect`
from `getServerSideProps` instead of using `redirects`.

## Package Types

Use `fetch_module_types(specifier)` only when exact package signatures are
needed, a less common package/subpath is involved, the effective URL/version
changed, the user explicitly asks to verify the API, or lint/typecheck suggests
the quick reference is stale.

Do not fetch module types for common packages already covered by model knowledge,
such as `react`.

Keep package declarations separate from generated project types:

- `fetch_module_types("specifier")` describes package exports and helper
  signatures, such as `talizen/cms` or `talizen/form`.
- `/types/cms.d.ts` and `/types/form.d.ts` describe generated project schemas and
  payload fields.
