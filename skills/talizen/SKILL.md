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

Talizen sites are React-based websites rendered by the Talizen platform. Local
agents usually work by using the Talizen CLI to pull site files, editing those
files locally, then pushing, syncing, or previewing through Talizen.

This skill is for general-purpose agents. Do not assume Talizen-system-only
tools are available. If the current environment exposes Talizen tools such as
linting, schema creation, module type fetching, versioning, or patch helpers,
use them when appropriate; otherwise inspect local files and use the CLI.

## Core Model

- The CLI handles login, project creation and discovery, pull, push, sync,
  preview, and publish workflows, plus platform data and asset operations.
- `pull` downloads remote site files into a local directory.
- `push` uploads the current local directory snapshot to Talizen and exits.
- `sync` is watch mode: it pushes the current local snapshot, then keeps
  listening for local file changes and pushes them in realtime.
- The CLI does not render sites locally.
- Rendering, CMS, assets, realtime preview, and publication are handled by the
  Talizen backend and web app.
- Site code must follow Talizen platform rules; do not treat a Talizen site as a
  generic Vite, Next.js, or browser SPA project.

## Default Workflow

1. Locate the site directory. If it is not local yet, use the CLI pull workflow.
2. Read local project guidance such as `AGENTS.md` if present.
3. Read `talizen.config.ts` when imports, metadata, custom code, or site-level
   styling may be involved.
4. Inspect the relevant `/page`, `/component`, `/types`, and root config files
   before editing.
5. Apply focused changes that match existing project conventions.
6. Validate with available local checks or Talizen platform checks. If no local
   renderer exists, use `talizen push`, `talizen sync`, or `talizen preview` as
   the verification path. Use `talizen push` for a one-time upload and
   `talizen sync` when you want watch mode.

## Hard Platform Rules

- Pages live in `/page` as `.tsx` React components.
- Keep reusable UI in `/component` or another shared components directory.
- Do not introduce `react-router-dom`, `next/link`, `next/router`,
  `next/navigation`, `getStaticProps`, or `getStaticPaths`.
- Use native anchors such as `<a href="/about">...</a>` for navigation.
- Prefer `getServerSideProps(context)` for route params and first-render data.
- Read route params from `context.params` when SSR params are available.
- Do not proactively create `*.canvas.ts` or `*.canvas.tsx` files. They are
  platform editor preview entries. Edit existing canvas files only when the user
  explicitly asks.
- Use Tailwind v4 utility classes for component styling. Avoid inline `style`
  props and ad-hoc `<style>` tags in page components.
- Use structured `metadata` for SEO instead of custom `seo` fields or raw
  `<title>` / `<meta name="description">` tags.

## Reference Map

- `references/cli.md`: CLI install/use, endpoint defaults, platform data
  commands, and asset upload commands.
- `references/site-code.md`: routing, page/component structure, import maps,
  and config rules.
- `references/cms.md`: CMS data fetching and generated schema usage.
- `references/forms.md`: form submissions and payload typing.
- `references/seo.md`: site and page metadata.
- `references/css.md`: Tailwind v4 and `index.css` conventions.
- `references/sitemap.md`: root-level sitemap generation.
- `references/carousel.md`: carousel/slideshow default approach.

For most site-authoring tasks, read `references/site-code.md` first, then load
the specific topic reference only if the task touches that area.
