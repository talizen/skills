---
name: talizen
mandatory: true
description: >
  MANDATORY platform baseline for ALL website work in Talizen/Creght — read and follow
  BEFORE creating or editing any project file. Sites are built as Talizen React apps
  (pages under /pages or /page + a root talizen.config.ts) by default; a genuinely
  self-contained single HTML file goes under public/, never a project-root index.html.
  Covers pages/routes, CMS/form/Func, Auth, Tailwind v4, SEO/metadata, importMap, domains,
  DNS/SSL, publishing, analytics, env vars, and platform tools. Not for greetings or chat.
tags:
  - talizen
  - platform
  - website
triggers:
  - build a website
  - create a page
  - standalone HTML
  - single HTML file
  - double-click to open
  - landing page
  - poster
  - 独立HTML网页
  - 路演/展示页
  - 保存文件双击即可打开
examples:
  - Build a website for a coffee shop
  - Create an About page
  - 制作一套独立HTML网页用于路演展示
---

# Talizen

Talizen apps are React websites with file-based routes under `/pages` or
`/page`, a root `talizen.config.ts`, Tailwind v4, platform CMS/forms/Func/auth,
metadata, import maps, previews, linting, and versioning. This is the operating
checklist; `## References` maps each topic to its `references/*.md`.

## Hard Rules

- Talizen is not a browser SPA or Next.js app. Never add `react-router-dom`,
  `next/link`, `next/router`, `next/navigation`, `getStaticProps`, or
  `getStaticPaths`. Navigate with native `<a href="/path">`, or Talizen's
  locale-aware `<Link>` on multilingual sites.
- Keep the project's existing `/pages` or `/page` and `/components` or
  `/component` roots. Prefer plural roots only for new projects.
- Use `getServerSideProps(context)` for route params and public first-render
  data. Do not read auth, call Func, or import browser SDKs in SSR.
- Only platform built-in importMap packages may appear in a page's module graph.
  A dependency added to `talizen.config.ts` resolves in the browser but not in
  SSR, silently dropping the page to client-only rendering; `lint` misses it.
  See `references/site-code.md` "SSR Availability".
- Use relative imports for local files; aliases such as `@/lib/utils` are
  unsupported.
- Prefer Tailwind v4 utilities. Use `/index.css` only for tokens, keyframes,
  complex selectors, or custom utilities. No inline `style` or page `<style>`.
- `/index.css` is loaded on every page automatically. Never import a `.css` file
  from a page or component; it breaks the render.
- Static files, including a self-contained standalone HTML file, go under
  `public/` (served at the domain root); a project-root `index.html` is NOT
  served.
- Do not commit/import local binaries. Use absolute URLs, tiny `data:` URIs, or
  the platform CDN URL from `upload_attachment`.
- Do not create `*.canvas.ts(x)` unless explicitly asked; they are platform
  generated.
- For visual configuration, including multiple variants, use typed React
  component props with defaults; the editor exposes component props as visual
  controls.
- CMS collection / form / JSON table / auth provider **definitions are files**:
  `/platform/{cms,form,table,auth}/<key>.json`. Read and edit them with the normal
  file tools; there are no schema tools. File name = resource key. Content
  entries, table records and form logs are still tool-based. See the matching
  `references/*.md`.
- `/platform/**` is live project state, not site source: it is **not** covered by
  site versions or `revert_version`, and it is shared by every site in the
  project. A rollback restores source files only — a schema change stays applied.
- Never edit `/types/cms.d.ts` or `/types/form.d.ts`. They regenerate from
  `/platform/cms/*.json` and `/platform/form/*.json`; re-read after a schema edit.
- CMS rich-text body fields (e.g. an article `body`) are HTML, not Markdown;
  Markdown renders as literal text.
- Use structured `metadata`, not custom `seo` fields or raw SEO tags it can
  express.
- Use Func for small backend workflows and persistent writes. Never fake
  persistence in React state or expose secrets/project IDs in page code.
- For a reported Func/runtime failure, read `references/error-handling.md` and
  `references/func.md` before editing. For deadline errors, inspect the caller's
  timeout; never infer a platform hard limit from a `run_func` self-test.
- Use `talizen/auth` `useAuth()` for auth UI, and read the installed `talizen`
  types first to match that version's request/response models. Never build
  users, passwords, sessions, OAuth callbacks, or account identity with Func or
  JSON tables.

## Default Workflow

1. Locate the project root; read `AGENTS.md` if present.
2. Read `talizen.config.ts` when config, imports, SEO, custom code, site CSS, or
   platform behavior may be involved.
3. Inspect relevant page/component/backend files and preserve local conventions.
4. Versions are automatic. The platform snapshots the site before your first
   file change and again when the task ends, so a rollback point always exists.
   Never call `create_version` yourself "to be safe" — that call costs a full
   round trip and buys nothing. Call it only if the user explicitly asks to save
   or mark a version.
5. Make focused edits. Writes return the patched region and that file's
   `syntax_errors` — do not re-read a file to confirm an edit landed.
6. Run `lint` once after the last edit, never per edit; it is a compile, config
   and cross-file check, and its `browser_check` renders the page client-side
   the way the editor canvas does, not through the server-rendered preview
   route.
   Do not follow it with a routine `browser` call — a screenshot is not how a
   task ends. Open a route only when something is genuinely unverified: a
   brand-new route, a reported failure (add `?dev`), or a subjective aesthetic
   request you cannot read off the code.
7. On any lint, typecheck, build, preview, browser, or runtime failure, follow
   `references/error-handling.md` immediately.

Editing an existing CMS entry is a platform-data operation, not a source edit:
when the field exists and the page already renders it, update it with the CMS
tools and verify the entry — no version, patch, or lint. Touch source only when
the field is missing or the page does not render it.

## References

Paths below are relative to this skill's `references/`. Read one only when you
reach its topic; do not read the others.

- `site-code.md` — routes, navigation, SSR data loading, local imports,
  importMap, `talizen.config.ts`, redirects, package types, components,
  `public/` static files and one-file artifacts (deck/poster/preview).
- `cms.md` — CMS content operations, schema types, fetch patterns.
- `css.md` — Tailwind v4 and `/index.css` conventions.
- `i18n.md` — multilingual routes, language switchers, locale APIs, `_i18n`,
  slugs, translations, localized CMS.
- `forms.md` — platform form schema and `talizen/form`.
- `auth.md` — login, registration, logout, current user, OAuth, account UI,
  protected flows.
- `func.md` — Func invariants: code and keys, JSON tables, secrets, asset
  uploads, third-party API keys, managed integrations, auth inside Func,
  `invoke(...)`, `/func/<key>`. It deliberately does **not** enumerate the `ctx`
  surface or which capabilities are managed — that set changes with each release.
  The API reference is live at `https://www.creght.cn/api.md`; read the matching
  doc there before writing Func code instead of relying on remembered signatures,
  defaults, or limits.
- `seo.md` — `metadata`, viewport, Open Graph, keywords, favicon, legacy `seo`
  migration.
- `carousel.md` — carousel/slideshow setup.
- `sitemap.md` — root-level sitemap generation.
- `platform-endpoints.md` — `/robots.txt`, `/sitemap.xml`, `/llms.txt` and their
  `/robots.ts`, `/llms.ts` customization files. Read before writing any of those
  three filenames anywhere, including under `/public`.
- `analytics.md` — visit analytics and custom event tracking.
- `canvas.md` — `*.canvas.tsx` artboards, frame layout, staging a draft next to
  the page.
- `console-operations.md` — domains, DNS/SSL, publishing, analytics, env vars,
  members, plans. Question-only: answer directly, no versions, edits, or lint.
- `error-handling.md` — any lint, build, preview, browser, runtime, or
  external-resource failure, and pages that render wrong, empty, or not-found
  while the data exists (re-fetch with `?dev` for full diagnostics first).

## Reply Rules

Talizen assistant end users are often non-technical:

- Do not expose raw code snippets unless requested.
- Do not show raw `diff_patch_file`, `lint`, or `create_version` outputs.
- Describe what changed on the site, how it behaves, and what the user can do
  next.
