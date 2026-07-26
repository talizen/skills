# Talizen Site Code

Talizen sites are React websites with file-based routes under `/pages` or
`/page`, a root `talizen.config.ts`, Tailwind v4, generated types, and platform
APIs for CMS, forms, metadata, import maps, previews, and publishing.

Both route roots (`/pages` and `/page`) and component roots (`/components` and
`/component`) are supported. Use existing roots; for new projects, prefer
`/pages` and `/components`.

## Routing

- `/pages/Index.tsx` -> `/`
- `/pages/About.tsx` -> `/about`

For non-`Index` pages, do not guess kebab-case routes. Prefer the lowercase
canonical path from lint/platform validation, such as
`/pages/BlockElementsPage.tsx` -> `/blockelementspage`.

Do not create `*.canvas.tsx` files by hand. For localized routing, read
`references/i18n.md`.

## Navigation

Use native anchors:

```tsx
<a href="/about">About</a>
```

On multilingual sites, use Talizen's locale-aware `<Link>`:

```tsx
import { Link } from "talizen"
<Link href="/about">About</Link>
```

Do not import `next/link`, `next/router`, `next/navigation`, or other routers.

## Data Loading

Use `getServerSideProps(context)` for route params and first-render data. Read
dynamic params from `context.params`; do not use client-side param hooks when
SSR params are available.

```tsx
export async function getServerSideProps(context) {
  return { props: { slug: context.params?.slug } }
}
```

## Components

Keep page files for route composition. Put reusable UI in the existing
component root or shared component folder. If a component needs preview, make it
visible from a page when possible.

For carousels, read `references/carousel.md`.

### Visual component props

For visual configuration or multiple editable variants, define typed React
props with literal defaults and pass literal values at the component call site.
The editor supports `string`, `number`, `boolean`, and arrays of those primitive
types; literal unions become option controls, and string props named `color` or
`colors` become color controls.

For registry components, configure `components.json`, then use
`shadcn_search_items`, `shadcn_list_items`, and `shadcn_install_item`. Common
registries: `@spell`, `@fancy`, `@react-bits`, `@talizen-sections`.

## Local Imports

Use relative imports for local files: `../lib/utils`, `../../lib/utils`, or
`./lib/utils`. Alias imports like `@/lib/utils` are unsupported.

Package/platform imports keep normal specifiers, such as `react`,
`talizen/cms`, and import-map keys configured in `talizen.config.ts`.

## Import Map

The platform provides common packages such as `react`, `react-dom`, and
`talizen`; do not add them manually. Add third-party dependencies in
`talizen.config.ts` `importMap.imports`.

```ts
export default {
  importMap: {
    imports: {
      "lucide-react": "https://esm.sh/lucide-react",
    },
  },
}
```

For media assets, do not commit/import local binaries. Use complete absolute
URLs, platform CDN URLs returned by upload tools, or tiny `data:` URIs.

## talizen.config.ts

Use `export default` with a plain object. Do not import packages except
type-only imports such as `import type { Metadata } from "talizen"`. Do not use
`defineConfig` from `talizen/config`.

Common fields: `importMap.imports`, `metadata`, `redirects`, and
`customCode.head/bodyStart/bodyEnd`. Prefer structured `metadata` over duplicate
SEO in `customCode`.

## Redirects

```ts
export default {
  redirects: [
    { source: "/old", destination: "/new", permanent: true },
    { source: "/posts/:slug", destination: "/blog/:slug", permanent: false },
  ],
}
```

Use `redirects` for static site-level redirects. For data-dependent redirects,
return `redirect` from `getServerSideProps`.

## Package Types

Use `fetch_module_types(specifier)` only when exact signatures are unknown, the
user asks to verify an API, or validation reports a mismatch.
