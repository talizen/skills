---
name: talizen
description: Guidance for building and configuring apps on the Talizen platform. Use this skill when working with Talizen projects, especially when editing `talizen.config.ts`, configuring file-based routes and CMS, enforcing Tailwind v4 styling rules, setting up SEO and `metadata`, or following platform-specific tool and reply conventions.
---

# Talizen

## Overview

This skill explains how to work inside the Talizen site builder from code: how to configure `talizen.config.ts`, respect the custom runtime and file-based routing rules, use the `talizen/cms` and `talizen/form` APIs, wire third-party imports via `importMap`, and correctly set up SEO using the `metadata` pattern. It also documents Talizen-specific tool usage (`diff_patch_file`, `lint`, `create_version`) and reply rules for the built-in AI assistant.

Talizen apps are built with React components and a root configuration file `talizen.config.ts`. SEO is configured in two layers: global site-level metadata in `talizen.config.ts` and page-level metadata exported from each page component file.

## Runtime & Routing Rules

Talizen uses its own runtime for rendering; it is not a standard browser frontend or Next.js runtime. You must follow these structural rules or the program will break:

- This is a **file-based website system**. All pages are React components written in `.tsx` files.
- The project is **not** a React single-page app. Do **not** use `react-router-dom` or similar client-side routing libraries.
- Routing is derived from the `/page` directory（NextJs-style）:
  - `/page/Index.tsx` → `/`
  - `/page/About.tsx` → `/about`
- The `/page` directory is required.
- Files like `/page/XXXX.canvas.tsx` are canvas preview entries used by the platform to render page previews in its canvas-based editor.
- In general, do not hand-write `/page/*.canvas.tsx` files; the platform usually generates them automatically.
- A typical generated `/page/XXXX.canvas.tsx` file looks like:

```tsx
import React from 'react'
import Page from './Index.tsx'

export default function Canvas() {
  return (
    <div>
      <div
        style={{
          position: 'absolute',
          width: 1200,
        }}
      >
        <Page />
      </div>
    </div>
  )
}
```

- When you need reusable view pieces (for example a `Hero` section), put them in `/component` (or another shared components directory), and keep actual route entries under `/page`.

When editing or generating files, ensure you preserve this structure and do not introduce SPA-style routing assumptions.

## Quick Start

When a user asks things like:
- “在 Talizen 里帮我优化页面的 SEO 标题和描述”
- “给这个项目加上 framer-motion 依赖”
- “在页面里设置 Open Graph 信息”
- “帮我接上 CMS 内容”

you should:

1. Locate the project root `talizen.config.ts`.
2. For third-party libraries, update `importMap.imports` with the correct CDN URL.
3. For SEO, prefer using the `metadata` field in `talizen.config.ts` plus per-page `export const metadata` instead of ad-hoc `seo` fields or raw `<meta>` tags.
4. For page-specific SEO, open the page component (for example `Page.tsx`/`PAGE.tsx`) and add or edit its exported `metadata`.
5. For CMS data, use `talizen/cms` in `getServerSideProps` to fetch content and pass it into the page as props.
6. For form submission, use `talizen/form` and read `/types/form.d.ts` before wiring payload fields.
6. Keep styling in Tailwind v4 utility classes; do not introduce inline styles or separate CSS files.

## CMS Usage (`talizen/cms`)

Talizen provides a CMS client via the `talizen/cms` package. Use it inside `getServerSideProps` to fetch content for pages, then pass the results into the page as props.

High-level rules:
- Use `ListContent` for lists (for example blog indexes) and `GetContent` for single items (for example blog detail pages).
- Always treat CMS data as optional; use optional chaining (`?.`) and provide user-friendly fallbacks.
- Keep CMS access in `getServerSideProps` and pass plain serializable props to the page component.

For full examples and patterns, see:
- `references/CMS.md`

## Form Usage (`talizen/form`)

Talizen provides a Form client via the `talizen/form` package. Use it when a page needs to submit a named form payload back to the platform.

High-level rules:
- Read `/types/form.d.ts` before writing form code so you know the exact `key` and payload shape.
- Import the payload type from `./types/form` and keep the submitted data aligned with that type.
- Use the stable form `key` from the editor, not the display name.

For full examples and patterns, see:
- `references/FORM.md`

## talizen.config.ts Basics

Talizen supports a root configuration file `talizen.config.ts` to control:
- Third-party dependencies via `importMap`
- Global SEO via `metadata`
- Arbitrary custom code snippets injected into `<head>` / `<body>` via `customCode`

### Basic structure

```ts
// talizen.config.ts
import type { Metadata } from 'next'

export default {
  importMap: {
    imports: {
      // Example: add external packages
      'framer-motion': 'https://esm.sh/framer-motion',
      'lucide-react': 'https://esm.sh/lucide-react',
    },
  },
  customCode: {
    head: '<script>window.__HEAD__=1</script>',
    body: '<script>window.__BODY__=1</script>',
  },
  // Prefer using `metadata` instead of custom `seo` fields
  metadata: {
    title: 'My Site',
    description: 'My Site Description',
    keywords: ['Next.js', 'React', 'JavaScript'],
  } satisfies Metadata,
}
```

Key points:
- `importMap.imports` keys are module specifiers used in `import` statements; values are CDN URLs.
- `customCode.head` / `customCode.body` are literal HTML strings injected into the document; keep them small and focused (analytics, verification tags, etc.).
- Use `metadata` for SEO instead of ad-hoc `seo` fields.

## SEO & Metadata Overview

Talizen follows a two-level metadata model similar to Next.js:
- Site-level metadata in `talizen.config.ts`
- Page-level metadata exported from each page component file

High-level behaviour:
- Prefer `metadata` (global + per page) instead of ad-hoc `seo` fields.
- Use `metadata.openGraph` for Open Graph data and `metadata.keywords` for keyword lists.
- When both site and page define a title:
  - A simple page `title: 'About Us'` overrides a simple root `title: 'Index'`.
  - If the root defines a `title.template`, Talizen combines template + page literal.
  - A page `title.absolute` ignores the root template and is used as-is.
- Use absolute URLs for Open Graph images, videos, and audio.

For complete field lists, examples, and migration guidance from legacy `seo` + custom `<meta>` tags to `metadata`, see:
- `references/SEO.md`

## ImportMap Usage

When users want to add or update third-party libraries:

1. Open `talizen.config.ts`.
2. Add or edit entries under `importMap.imports`:

```ts
export default {
  importMap: {
    imports: {
      'framer-motion': 'https://esm.sh/framer-motion',
      foo: 'https://cdn.example/foo',
    },
  },
}
```

3. In Talizen pages/components, import using the same specifier:

```tsx
import { AlertCircle } from 'lucide-react'
```

Keep importMap clean:
- Only add packages that are actually used.
- Prefer `esm.sh` as the package CDN/provider unless there is a clear compatibility reason to use something else.
- Do not add `react` or `react-dom` to `importMap.imports`; Talizen already provides them and redefining them can cause runtime issues.
- Update URLs carefully when bumping versions.

## Styling Rules (Tailwind v4)

Talizen uses Tailwind CSS v4 built-in. Follow these rules:

- Use Tailwind utility classes for styling.
- Do not use inline styles (`style` props), separate CSS files, or `<style>` tags.
- When adding transitions on color, ensure the color is driven from a single source to avoid “double transitions”:
  - Bad: parent changes text color on hover and child also has its own hover color, causing a jarring effect.
  - Good: either keep the child color fixed and let it inherit the parent change, or control the child color only via a single hover/group-hover rule.

Express color transitions through a consistent set of Tailwind classes on the same element or the same group.

## When to Use Custom Head/Body Code

Use `customCode.head` / `customCode.body` for:
- Analytics snippets
- Third-party verification tags
- Custom scripts that are not covered by `metadata`

Avoid duplicating metadata:
- Do not emit `<title>` or `<meta name="description">` manually when they are already covered by `metadata`.
- Prefer structured `metadata` for anything that fits into the Next.js metadata schema; treat `<meta>` tags in `customCode` as a last resort.

## Task Planning & Tools (Talizen AI Assistant)

Within the Talizen platform, the AI assistant can read and write project files and has access to platform tools. When acting as that assistant:

- Before working on a task, if the project contains an `AGENTS.md` file, read it and follow its instructions as primary guidance.
- If the user wants to import an existing project or codebase into the Talizen platform, preserve the full source experience:
  - Include all pages, content, and functional behaviour from the uploaded code.
  - Recreate the original design with high fidelity unless the user explicitly asks for design changes.
  - Ensure the imported result is compatible with the Talizen platform.
- For editing files, prefer the `diff_patch_file` tool for applying changes; avoid wholesale rewrites when a patch is sufficient.
- Whenever a task involves changes to page or component code, run the `lint` tool before completing the task to ensure code correctness.
- After completing a task that modified files, use the `create_version` tool to create a new version, unless the user has explicitly said not to create a version.

Treat these tools as the standard workflow: plan changes, patch files, lint, then version.

## Reply Rules (End User Communication)

The end user of the Talizen AI assistant is typically non-technical:

- Do not expose raw code snippets in final user-facing responses unless explicitly requested.
- Do not show internal tool outputs (`diff_patch_file`, `lint`, `create_version`) directly to the user.
- Communicate in product and content terms: describe what changed on the site, how the page behaves, and what the user can do next, instead of describing implementation details.

## Resources (optional)

Current references:

- `references/SEO.md` — full SEO and `metadata` configuration guide for `talizen.config.ts` and per-page metadata, including merge rules and migration from legacy `seo` configs.

Add more reference or script files under `references/` or `scripts/` only when Talizen introduces new runtime behaviours or tools that require detailed documentation or automation.
