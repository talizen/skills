---
name: talizen
description: >
  Guidance for Talizen platform apps and Creght editor operations. Use before
  editing project files or giving platform-specific advice about /pages or /page routes,
  talizen.config.ts, CMS/form/Func, Auth, Tailwind v4, SEO/metadata, importMap,
  domains, DNS/SSL, publishing, website analytics, environment variables, or
  other platform tools. Not required for greetings or unrelated chat.
---

# Talizen

Talizen apps are React websites with file-based routes under `/pages` or
`/page`, a root `talizen.config.ts`, Tailwind v4, platform CMS/forms/Func/auth,
metadata, import maps, previews, linting, and versioning.

Keep this file as the operating checklist. Load only the relevant
`references/*.md` file for implementation details.

## Hard Rules

- Talizen is not a browser SPA or Next.js app. Do not add `react-router-dom`,
  `next/link`, `next/router`, `next/navigation`, `getStaticProps`, or
  `getStaticPaths`.
- Preserve the project's existing `/pages` or `/page` root and `/components` or
  `/component` root. Prefer plural roots only for new projects.
- Use native `<a href="/path">` links, or Talizen's locale-aware `<Link>` for
  multilingual internal links. Never use router libraries.
- Use `getServerSideProps(context)` for route params and public first-render
  data. Do not read auth, call Func, or import browser SDKs in SSR.
- Do not create `*.canvas.ts(x)` files unless explicitly asked; they are platform
  generated.
- Prefer Tailwind v4 utility classes. Use `/index.css` only for tokens,
  keyframes, complex selectors, or custom utilities. No inline `style` or page
  `<style>` blocks.
- Use relative imports for local project files. Alias imports such as
  `@/lib/utils` are unsupported.
- Do not commit/import local binaries. Reference media by absolute URL, data URI
  for tiny inline assets, or platform CDN URL from `upload_attachment`.
- Use structured `metadata`, not custom `seo` fields or raw SEO tags that
  `metadata` can express.
- Use Talizen Func for small backend workflows and persistent writes. Never fake
  persistence in React state or expose secrets/project IDs in page code.
- Use `talizen/auth` `useAuth()` in React auth UI. Do not build users/passwords,
  sessions, OAuth callbacks, or account identity with Func or JSON tables.
- Before using `talizen/auth`, read the installed `talizen` type definitions and
  follow that version's request/response models.

## Default Workflow

1. Locate the project root. Read `AGENTS.md` if present.
2. Read `talizen.config.ts` when config, imports, SEO, custom code, site CSS, or
   platform behavior may be involved.
3. Inspect relevant page/component/backend files and preserve local conventions.
4. In the Talizen AI assistant, create one rollback `create_version` before
   source edits unless this is a copy-only edit.
5. Make focused edits, preferably with `diff_patch_file` in that assistant.
6. Run `lint` after page/component/source changes.
7. If lint, typecheck, build, preview, browser render, or runtime validation
   fails, immediately follow `references/error-handling.md`.
8. After successful edits, create one final `create_version`. A task should make
   at most two versions: before and after.

## Reference Routing

- `references/site-code.md`: routes, navigation, SSR data loading, local imports,
  import maps, `talizen.config.ts`, redirects, package type lookup, components.
- `references/css.md`: Tailwind v4 and `/index.css` conventions.
- `references/cms.md`: CMS schema types and fetch patterns.
- `references/i18n.md`: multilingual routing, locale APIs, `_i18n`, slugs, and
  translation workflow.
- `references/forms.md`: platform form schema and `talizen/form`.
- `references/auth.md`: login, registration, logout, current user, OAuth,
  account UI, and protected auth patterns.
- `references/func.md`: project Func code, JSON tables, secrets, asset uploads,
  auth inside Func, and browser `invoke("file.method")`.
- `references/seo.md`: `metadata`, viewport, Open Graph, keywords, and migration
  from legacy SEO/custom head.
- `references/carousel.md`: carousel/slideshow setup.
- `references/sitemap.md`: root-level sitemap generation.
- `references/analytics.md`: visit analytics and custom event tracking.
- `references/console-operations.md`: question-only editor guidance for domains,
  DNS/SSL, publishing, analytics, env vars, members, and plans.
- `references/error-handling.md`: bounded recovery for lint, build, preview,
  browser, runtime, and external-resource errors.

## Read Before

- Auth UI or protected flows: read `references/auth.md`.
- Func, JSON tables, backend actions, third-party API keys, `invoke(...)`, or
  `/api/func`: read `references/func.md`.
- SEO, OG, keywords, favicon, viewport, or legacy `seo` migration: read
  `references/seo.md`.
- Multilingual routes, language switchers, translations, or localized CMS:
  read `references/i18n.md`.
- Editor-navigation questions only: read `references/console-operations.md` and
  answer directly without versions, edits, or lint.
- Custom event tracking: read `references/analytics.md`.

## Common Patterns

- Use SSR only for public or cookie-vary-safe first-render data. Keep login
  state, private user data, writes, and Func calls in browser SDK/Func/API flows.
- Password-gated pages should render a public gate, verify through Func/API, set
  a signed access cookie, then fetch protected content from Func/API.
- Article lists with fast-changing counters should SSR/cache the list and fetch
  counters after hydration.
- If asked for only a component but a preview is expected, make it visible from a
  page whenever possible.
- When importing an existing project, preserve source behavior and design unless
  the user asks for changes.

## Reply Rules

Talizen assistant end users are often non-technical:

- Do not expose raw code snippets unless requested.
- Do not show raw `diff_patch_file`, `lint`, or `create_version` outputs.
- Describe what changed on the site, how it behaves, and what the user can do
  next.
