---
name: talizen
description: >
  Use when working with Talizen sites or the Talizen CLI, including pulling site
  code locally, pushing local changes, running watch-mode sync, writing
  Talizen-compatible React page/component code, CMS and form integration,
  routing, styling, metadata, previewing, publishing, or debugging
  local-to-platform workflows.
---

# Talizen

Talizen sites are React websites rendered by the Talizen platform. General
agents usually use the Talizen CLI to pull site files, edit locally, then push,
sync, or preview through Talizen. Do not assume Talizen-system-only tools are
available; use them if exposed, otherwise inspect files and use the CLI.

## Core Model

- The CLI handles login, project/site discovery, pull, push, sync, preview,
  publish, platform data, and asset operations.
- `pull` downloads remote site files into a local directory.
- `push` uploads the current local snapshot and exits.
- `sync` pushes once, then watches local changes and pushes in realtime.
- The CLI does not render sites locally; rendering, CMS, assets, preview, and
  publication are backend/web-app responsibilities.
- Site code must follow Talizen platform rules; do not treat it as generic Vite,
  Next.js, or browser SPA code.
- If the user provides no actionable requirement, ask what to build or change.
- Authenticated backend data operations such as login/registration are not yet
  supported. If requested, implement only frontend UI and state the backend
  limitation clearly.

## Default Workflow

1. Locate the site directory. Pull with the CLI if it is not local.
2. Read local `AGENTS.md` if present.
3. Read `talizen.config.ts` when imports, metadata, custom code, or site-level
   CSS may be involved.
4. Inspect relevant page/component files, `/types`, and root configs.
5. Apply focused edits that preserve local conventions.
6. Validate with available local checks or platform checks. Without a local
   renderer, use `talizen push`, `talizen sync`, or `talizen preview`.
7. On lint, typecheck, build, preview, browser, or runtime errors, immediately
   read `references/error-handling.md` before fixing.

## Hard Platform Rules

- Preserve existing `/pages` or `/page` route root and `/components` or
  `/component` UI root. Prefer plural roots only for new projects.
- Do not add `react-router-dom`, `next/link`, `next/router`,
  `next/navigation`, `getStaticProps`, or `getStaticPaths`.
- Use native anchors for navigation. On multilingual sites, use Talizen's
  locale-aware `<Link>`; never use router libraries.
- Use `getServerSideProps(context)` for route params and first-render data.
  Read route params from `context.params`.
- Do not create `*.canvas.ts(x)` files unless explicitly asked.
- Prefer Tailwind v4 utilities. Avoid inline `style` and ad-hoc page `<style>`
  blocks.
- Use relative imports for local files; aliases such as `@/lib/utils` are
  unsupported.
- Use structured `metadata`, not custom `seo` fields or duplicate raw SEO tags.
- Do not implement authenticated backend data operations yet.
- On validation/runtime errors, read `references/error-handling.md` before
  speculative code changes.

## Reference Routing

- `references/cli.md`: CLI install/use, endpoint defaults, platform data,
  publish, and asset upload.
- `references/site-code.md`: routes, pages/components, SSR, imports, import
  maps, `talizen.config.ts`, redirects, package types.
- `references/css.md`: Tailwind v4 and `/index.css`.
- `references/cms.md`: CMS schema types and fetch patterns.
- `references/forms.md`: form schema and `talizen/form`.
- `references/seo.md`: `metadata`, Open Graph, keywords, legacy SEO migration.
- `references/i18n.md`: multilingual routing, locale APIs, `_i18n`, messages.
- `references/sitemap.md`: root-level sitemap.
- `references/carousel.md`: carousel/slideshow setup.
- `references/error-handling.md`: bounded recovery for validation/runtime errors.

For most site-authoring tasks, read `references/site-code.md` first, then load
only the topic reference the task touches.
