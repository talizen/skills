---
title: Talizen SEO & Metadata
---

# Talizen SEO & Metadata

Use Talizen's structured `metadata` model for SEO. It applies at two levels:

- Site defaults in `talizen.config.ts`. May be written as `metadata: (ctx) => ({…})`
  to branch on locale or host — see `site-code.md`.
- Page metadata exported from each page component.

Avoid custom `seo` objects and raw tags when `metadata` can express the same
thing.

## Metadata Shape

Supported fields mirror the common Next.js `Metadata` shape: `title`,
`description`, `keywords`, `creator`, `publisher`, `applicationName`,
`generator`, `referrer`, `formatDetection`, `openGraph`, and `icons`.

Place Open Graph values under `metadata.openGraph`; use absolute URLs for OG
images, videos, and audio. Store keywords as a string array.

Minimal site-level config:

```ts
import type { Metadata } from "talizen"

export default {
  metadata: {
    title: { template: "%s | Acme", default: "Acme" },
    description: "Acme makes better tools.",
    keywords: ["Acme", "Tools"],
    openGraph: {
      title: "Acme",
      description: "Acme makes better tools.",
      url: "https://example.com",
      siteName: "Acme",
      images: [{ url: "https://example.com/og.png", width: 1200, height: 630 }],
      type: "website",
    },
  } satisfies Metadata,
}
```

## Page Metadata

Pages may export `metadata` or `generateMetadata`. Use `generateMetadata` when
SEO depends on params, query data, or CMS content.

**Every new page needs its own.** A page without `metadata` silently inherits
the site title, so a five-page site ships five identical `<title>` tags and one
of the cheapest SEO wins is gone. Write it when you create the page, not in a
later "SEO pass" — by then nobody remembers which pages were added. Detail pages
generated from one template are the usual casualty: the template has no
`metadata`, so every instance shares the site title.

```tsx
import type { Metadata } from "talizen"

export const metadata: Metadata = {
  title: "About",
  description: "About Acme.",
}
```

```tsx
import type { Metadata } from "talizen"
import { getContent } from "talizen/cms"

export async function generateMetadata({ params }): Promise<Metadata> {
  const content = await getContent("blogs", params.slug, { builtinRef: true })
  return {
    title: content?.body?.title ?? params.slug,
    description: content?.body?.description ?? "Blog detail page",
  }
}
```

## Merge Rules

- A literal page `title` overrides the site title.
- A site title template such as `{ template: "%s | Acme", default: "Acme" }`
  wraps literal page titles.
- A page title object with `{ absolute: "About" }` bypasses the site template.
- `title.template` in the site layer wraps literal page titles.
- Other page fields override the same site fields when defined; otherwise site
  fields fall back.
- `metadata.icons` merges by slot: page `shortcut`, `icon`, `apple`, or `other`
  replaces that slot only when non-empty.

Site `metadata.title` and `metadata.description` also drive `/llms.txt`; never
hand-write one. See `platform-endpoints.md`.

## Icons

`metadata.icons` emits favicon/touch links in this order: `shortcut`, `icon`,
`apple-touch-icon`, `other`.

Each `shortcut`, `icon`, and `apple` value may be a string URL, an object
`{ url, media?, sizes?, type? }`, or an array. `other` may be `{ rel, url }` or
an array.

```ts
export const metadata = {
  icons: {
    icon: [{ url: "/icon.png" }, { url: "/icon-dark.png", media: "(prefers-color-scheme: dark)" }],
    shortcut: "/shortcut-icon.png",
    apple: { url: "/apple-icon.png", sizes: "180x180", type: "image/png" },
    other: { rel: "apple-touch-icon-precomposed", url: "/apple-precomposed.png" },
  },
}
```

## Viewport

Viewport is site-level config, separate from `metadata`.

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
}
```

If omitted, Talizen renders:

```html
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

Do not use `metadata.viewport`, page-level `viewport`, `generateViewport`, or
raw viewport tags in `customCode.head`. Supported fields: `width`, `height`,
`initialScale`, `minimumScale`, `maximumScale`, `userScalable`,
`interactiveWidget`, `themeColor`, and `colorScheme`. Use `initialScale: null`
to remove the default `initial-scale=1`.

## Legacy SEO Migration

When migrating old `seo` fields or SEO tags in `customCode.head`:

1. Move `seo.title` and `seo.description` to `metadata.title` and
   `metadata.description`.
2. Convert Open Graph tags to `metadata.openGraph`.
3. Convert keywords meta tags to `metadata.keywords`.
4. Keep `customCode.head` only for tags that cannot be represented by
   `metadata`, such as some verification or platform-specific tags.

For requests like "add OG" or "add keywords", edit `metadata.openGraph` and
`metadata.keywords`; do not add duplicate raw tags.

## Structured Data (JSON-LD)

`metadata` cannot express schema.org, so JSON-LD goes in as a raw
`application/ld+json` script — via `customCode.head` for site-wide facts, or
inside the page for page-specific ones.

Pick the type from what the page *is*, not from what the site is. The common
mistake is stamping the same business object on every page:

| Page | Type |
| --- | --- |
| home / contact | `LocalBusiness` or a subtype (`CafeOrCoffeeShop`, `Store`…) |
| one product, one dish, one plan | `Product` with `offers` |
| an article | `Article` / `BlogPosting` |
| a page with real Q&A on it | `FAQPage` |
| a multi-step how-to | `HowTo` |

A product detail page carrying only the shop's `LocalBusiness` block tells
search engines nothing about the product. Either give it a `Product` block, or
nest the products as `makesOffer` on the business object — both are valid, but
one of them has to be there.

Two rules that break it silently:

- **Every fact in the JSON-LD must be visible on the page.** Prices, hours,
  ratings and answers that appear only in the markup are grounds for a
  structured-data penalty, not a bonus.
- **Keep it in sync with the copy.** When the visible FAQ or price changes, the
  JSON-LD is a second place holding the same string. Leave a comment at both
  sites saying so.

## AI SEO Workflow

1. Inspect `talizen.config.ts` for global `metadata`.
2. Inspect the page component for `metadata` or `generateMetadata`.
3. Apply changes primarily to title, description, keywords, and Open Graph.
4. Add or correct JSON-LD per the table above — check the type actually matches
   each page.
5. Walk `/robots.ts`, `/sitemap.ts` and `/llms.ts`; see
   `platform-endpoints.md` "When you must write these files". Asked for GEO
   specifically, `/llms.txt` is the file AI assistants read.
6. Use `customCode.head` only for unsupported SEO tags.
