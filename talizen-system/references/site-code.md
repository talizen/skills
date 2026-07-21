# Talizen Site Code

Talizen apps are React websites with file-based routes under `/pages` or
`/page`, a root `talizen.config.ts`, Tailwind v4, generated types, and platform
APIs for CMS, forms, metadata, import maps, previews, and publishing.

Both route roots (`/pages` and `/page`) and component roots (`/components` and
`/component`) are supported. Use the project's existing roots; for new projects,
prefer `/pages` and `/components`.

## Routing

Routes derive from page file names:

- `/pages/Index.tsx` -> `/`
- `/pages/About.tsx` -> `/about`

For non-`Index` pages, do not guess kebab-case routes. Prefer the lowercase
canonical path returned by lint/platform validation, such as
`/pages/BlockElementsPage.tsx` -> `/blockelementspage`.

Do not create `*.canvas.tsx` files by hand. For localized routing, read
`references/i18n.md`.

## Navigation

Use native anchors for normal navigation:

```tsx
<a href="/about">About</a>
```

On multilingual sites, use Talizen's `<Link>` so internal links keep the current
locale:

```tsx
import { Link } from "talizen"

<Link href="/about">About</Link>
```

Do not import `next/link`, `next/router`, `next/navigation`, or other router
libraries.

## Data Loading

Use `getServerSideProps(context)` for route params and public first-render data.
Read dynamic params from `context.params`; do not use client-side param hooks
when SSR params are available.

Type the context — don't use `any` or an untyped `context`. Type the whole
function with `GetServerSideProps<Props, Params>` and the page props with
`InferGetServerSidePropsType`, both imported from `talizen`:

```tsx
import type { GetServerSideProps, InferGetServerSidePropsType } from "talizen"

export const getServerSideProps: GetServerSideProps<{ slug: string }, { slug: string }> = async (context) => {
  return { props: { slug: context.params.slug } }
}

export default function Page(props: InferGetServerSidePropsType<typeof getServerSideProps>) {
  return <main>{props.slug}</main>
}
```

Fields: `params`, `searchParams`, `request` (`host` / `headers.get()`), `cookies`
(`get`/`has`/`set`/`delete`), and `locale` / `locales` / `defaultLocale` /
`routingDefaultLocale`. `req`, `query`, and `request.cookies` are deprecated aliases.

Do not read auth or call Func in `getServerSideProps`. Use `useAuth()` in React
UI and Func `ctx.auth` for protected backend actions.

## Components

Keep page files for route composition. Put reusable UI in the project's existing
component root or shared component folder.

If a user asks only for a component but expects a preview, make it visible from
a page when possible.

For carousels, read `references/carousel.md`.

## Component Registries And Effects

For shadcn/ui-style registry components:

- Configure `components.json` registries first.
- Use `shadcn_search_items`, `shadcn_list_items`, and `shadcn_install_item`.
- Common registries: `@spell`, `@fancy`, `@react-bits`, `@talizen-sections`.
- For polished effects or animations, prefer registry search before custom code.
- Ensure animated layers do not block content, readability, or responsiveness.

Minimal `components.json`:

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

Use relative paths for local project imports: `../lib/utils`,
`../../lib/utils`, or `./lib/utils`. Alias imports like `@/lib/utils` are not
supported.

Package imports and Talizen platform imports keep normal specifiers, such as
`react`, `talizen/cms`, and import-map keys configured in `talizen.config.ts`.

## Import Map

The platform provides common packages such as `react`, `react-dom`, and
`talizen`; do not add them manually.

Add or change third-party dependencies in `talizen.config.ts`
`importMap.imports`. Use exact ESM URLs where possible.

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
URLs, platform CDN URLs returned by upload tools, or tiny `data:` URIs. Func
runtime assets should use `ctx.assets.upload(...)` and store returned metadata.

## talizen.config.ts

Use `export default` with a plain object. Do not import packages in this file
except type-only imports such as `import type { Metadata } from "talizen"`.
Do not use `defineConfig` from `talizen/config`.

Common fields:

- `importMap.imports` for user-defined dependencies
- `metadata` for global SEO
- `viewport` for site viewport
- `redirects` for site redirects
- `customCode.head` / `customCode.bodyStart` / `customCode.bodyEnd` for snippets
  not covered by structured metadata

Prefer structured `metadata` over duplicate SEO in `customCode`.

## Viewport

Configure the default viewport in `talizen.config.ts` with a plain `viewport`
object. Do not put viewport settings under `metadata`, page exports, or raw head
tags. Read `references/seo.md` for field details.

## Redirects

Configure site redirects with `redirects` in `talizen.config.ts`:

```ts
export default {
  redirects: [
    { source: "/old", destination: "/new", permanent: true },
    { source: "/posts/:slug", destination: "/blog/:slug", permanent: false },
  ],
}
```

Use `redirects` for site-level static redirects. For per-request redirects that
depend on data, return `redirect` from `getServerSideProps`.

## Package Types

Use `fetch_module_types(specifier)` only when exact package signatures are
unknown, the user asks to verify an API, or lint/typecheck reports a mismatch.
Do not fetch types for common packages already covered by model knowledge.
