---
title: Talizen Platform Endpoints
---

# Talizen Platform Endpoints

The platform always serves these. You cannot replace them with static files: a
same-named file under `/public` is never reached, and nothing warns you.

| Endpoint | Built from | Customize with |
| --- | --- | --- |
| `/robots.txt` | platform defaults | `/robots.ts` |
| `/sitemap.xml` | page scan + `generateStaticParams` | `/sitemap.ts` |
| `/llms.txt` | sitemap's page list + `metadata` | `/llms.ts` |
| `/<page>.md` | the page itself | — |

A page whose filename contains `[param]` **must** export `generateStaticParams`,
or none of its URLs reach `sitemap.xml` and static export, silently. See
`sitemap.md`.

## When you must write these files

The defaults are fine for a site where every page is meant to be public. They
are wrong the moment a page is not, and nothing warns you: **every page the
platform can see goes into `sitemap.xml` and `/llms.txt` automatically**, and
the default `robots.txt` is `Allow: /`.

So whenever the site has a page that is per-user, gated, or an internal detail
(`/my-orders`, `/account`, `/dashboard`, a thank-you page, a search result
page), that page needs excluding in **all three** places, because each feeds a
different consumer:

| File | Keeps it out of | Miss it and |
| --- | --- | --- |
| `/robots.ts` → `disallow` | crawlers | it gets indexed |
| `/sitemap.ts` | the submitted URL list | you actively ask for it to be indexed |
| `/llms.ts` → `exclude` | `/llms.txt` | you hand it to AI assistants |

Doing one and forgetting the others is the common failure. Listing a personal
page under an `llms.ts` section is the inverse of excluding it: it advertises
the page to every AI crawler that reads the file.

Asked for "SEO" or "GEO" work, walk all three files, not just the one that
looks most relevant.

`/llms.txt` takes its title and summary from `metadata.title` and
`metadata.description`, so improving those updates it automatically. There is no
override field. Add `/llms.ts` only to change *structure*.

## `/robots.ts`

Default-export a function returning the Next.js `MetadataRoute.Robots` shape.

```ts
export default function robots() {
  return {
    rules: [
      { userAgent: "*", allow: "/", disallow: ["/admin"] },
      { userAgent: "BadBot", disallow: "/" },
    ],
  };
}
```

Preview domains always return `Disallow: /` and skip `/robots.ts` entirely.

## `/llms.ts`

Default-export a function (may be `async`). Return an object to declare
structure and let the platform enumerate pages, or a string to become the entire
file.

```ts
export default function llms(ctx) {
  return {
    details: "Free Markdown, placed after the summary.",
    sections: [
      { name: "Start here", links: [{ name: "/quickstart", url: `${ctx.origin}/quickstart.md` }] },
      { name: "Docs", pages: ["/docs/*"] },
    ],
    exclude: ["/internal/*"],
  };
}
```

Returning a string takes over completely — it becomes the file verbatim, so you
own the heading, the summary, and every link. `ctx.pages` is still the
platform's enumeration, so you are not starting from nothing:

```ts
export default function llms(ctx) {
  const docs = ctx.pages.filter((p) => p.path.startsWith("/docs/"));
  return [
    `# ${ctx.metadata.title}`,
    ``,
    `> ${ctx.metadata.description}`,
    ``,
    `## Docs`,
    ``,
    ...docs.map((p) => `- [${p.path}](${p.url})`),
    ``,
  ].join("\n");
}
```

Common fields, enough for most tasks:

- Section: `{ name, pages }` where `pages` are patterns — an exact path, or a
  trailing `/*` matching the prefix and everything under it. Use
  `{ name, links }` with `{ name, url }` entries to list links yourself; both
  forms may sit in one `sections` array.
- `exclude` — patterns dropped before sectioning.
- `ctx` — `{ origin, pages, metadata }`; each page is `{ path, url }`.

No `title` / `description` fields: they come from `metadata`.

## Types

The `talizen` package is the source of truth for all three files —
`SitemapFile` / `SitemapEntry`, `Robots` / `RobotsRule`, and `LLMsFile` /
`LLMsContext` / `LLMsSpec`. Read them when you need anything past the common
fields above: full field lists, ordering rules, defaults, and value constraints.

```ts
import type { LLMsFile } from "talizen";
```

Type-only imports are erased at build time, so they are safe in these files.

## Verifying

Request the endpoint itself — none of this shows up in page HTML. On a preview
domain `/robots.txt` is always `Disallow: /`, so check rules on production. If
one of these `.ts` files throws or returns the wrong shape, the endpoint returns
5xx rather than falling back to the default output.
