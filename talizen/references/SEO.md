---
title: Talizen SEO & Metadata
---

# Talizen SEO & Metadata

This document describes how to configure SEO on the Talizen platform using the `metadata` model. It applies both to site-level configuration in `talizen.config.ts` and page-level configuration in React page components.

Talizen's SEO model is heavily inspired by Next.js `Metadata`. Prefer structured `metadata` over ad‑hoc `seo` blocks or raw `<meta>` tags.

## Two-Level Metadata Model

Talizen combines two metadata sources:
- **Site-level metadata** in `talizen.config.ts`
- **Page-level metadata** exported from each page component file

Both support standard Next.js metadata fields, including:
- `title`, `description`, `keywords`, `creator`, `publisher`, `applicationName`, `generator`, `referrer`
- `formatDetection`
- `openGraph` (with `title`, `description`, `url`, `siteName`, `images`, `videos`, `audio`, `locale`, `type`)

Talizen converts these structured fields into `<title>`, `<meta>` and Open Graph tags in the rendered HTML.

## Site-Level Metadata (`talizen.config.ts`)

Site-wide defaults are defined in `talizen.config.ts`:

```ts
// talizen.config.ts
import type { Metadata } from 'talizen'

export default {
  metadata: {
    title: {
      template: '%s | Merry Gradient',
      default: 'Merry Gradient',
    },
    description:
      'Merry Gradient is a professional CSS gradient & light glow studio for modern web designers and developers.',
    keywords: [
      'CSS Gradient',
      'Light Glow',
      'Background Texture',
      'CSS Generator',
      'UI Design',
    ],
    openGraph: {
      title: 'Merry Gradient',
      description:
        'Professional visual tool for designing sophisticated CSS background textures and lighting effects.',
      url: 'https://example.com',
      siteName: 'Merry Gradient',
      images: [
        {
          url: 'https://example.com/og.png',
          width: 1200,
          height: 630,
        },
      ],
      type: 'website',
    },
  } satisfies Metadata,
}
```

Guidelines:
- Always use the `metadata` field (not a custom `seo` object) for SEO.
- Place Open Graph data under `metadata.openGraph`.
- Use absolute URLs for Open Graph images, videos, and audio.

## Page-Level Metadata (PAGE.tsx / Page.tsx)

Each page can export its own `metadata` object, or export `generateMetadata` for dynamic values. Both participate in merge rules with the site-level metadata. Prefer `generateMetadata` when page SEO depends on route params or query data.

```tsx
// PAGE.tsx or Page.tsx
import type { Metadata } from 'talizen'

export const metadata: Metadata = {
  description: 'The React Framework for the Web',
}

export default function Page() {
  return <main>...</main>
}
```

Dynamic example:

```tsx
// PAGE.tsx or Page.tsx
import type { Metadata } from 'talizen'
import { getContent } from 'talizen/cms'
import type { Blogs } from './types/cms'

export async function generateMetadata({ params }): Promise<Metadata> {
  const content = await getContent<Blogs>('blogs', params.slug, {
    builtinRef: true,
  })

  return {
    title: content?.body?.title ?? params.slug,
    description: content?.body?.description ?? 'Blog detail page',
  }
}
```

This will produce tags such as:

```html
<title>About</title>
<meta name="description" content="The React Framework for the Web" />
```

Other fields (e.g. `openGraph`) fall back to `talizen.config.ts` unless overridden at the page level.

## Merge Rules: Site vs Page

When both levels define the same field:

- **Literal page title wins**:

  ```tsx
  // Page.tsx
  export const metadata = {
    title: 'About Us',
  }

  // talizen.config.ts
  export default {
    metadata: {
      title: 'Index',
    },
  }
  ```

  Result:

  ```html
  <title>About Us</title>
  ```

- **Template at root + literal on page**:

  ```tsx
  // Page.tsx
  export const metadata = {
    title: 'About Us',
  }

  // talizen.config.ts
  export default {
    metadata: {
      title: {
        template: '%s | Acme',
        default: 'Acme',
      },
    },
  }
  ```

  Result:

  ```html
  <title>About Us | Acme</title>
  ```

  If the page does not define `title`, the `default` value is used:

  ```html
  <title>Acme</title>
  ```

- **Absolute page title ignores template**:

  ```tsx
  // Page.tsx
  export const metadata = {
    title: {
      absolute: 'About',
    },
  }

  // talizen.config.ts
  export default {
    metadata: {
      title: {
        template: '%s | Acme',
        default: 'Acme',
      },
    },
  }
  ```

  Result:

  ```html
  <title>About</title>
  ```

## OG & Keywords Mapping

Example metadata:

```ts
export const metadata = {
  title: 'Acme',
  description: 'A demo site',
  keywords: ['Next.js', 'React', 'JavaScript'],
  openGraph: {
    title: 'Next.js',
    description: 'The React Framework for the Web',
    url: 'https://nextjs.org',
    siteName: 'Next.js',
    images: [
      {
        url: 'https://nextjs.org/og.png',
        width: 800,
        height: 600,
      },
    ],
    type: 'website',
  },
}
```

Generates tags like:

```html
<meta name="keywords" content="Next.js,React,JavaScript" />
<meta property="og:title" content="Next.js" />
<meta property="og:description" content="The React Framework for the Web" />
<meta property="og:image" content="https://nextjs.org/og.png" />
```

When users ask to “加 OG” or “加 keywords”, prefer:
- Editing `metadata.openGraph` for Open Graph data.
- Editing `metadata.keywords` (array of strings) for keywords.

Avoid writing raw `<meta>` tags for things that `metadata` already covers.

## Migrating from `seo` + Custom Head to `metadata`

Legacy Talizen projects may use a custom `seo` block and embed `<meta>` tags directly via `customCode.head`:

```ts
export default {
  importMap: {
    imports: {
      'lucide-react': 'https://esm.sh/lucide-react',
    },
  },
  seo: {
    title: 'Merry Gradient | Professional CSS Gradient & Light Glow Studio',
    description:
      'Craft sophisticated multi-layered CSS background textures and stunning lightglow gradients.',
  },
  customCode: {
    head: `
      <meta name="keywords" content="CSS Gradient, Light Glow, Web Design Tool">
      <meta property="og:type" content="website">
      <meta property="og:title" content="Merry Gradient | Professional CSS Gradient & Light Glow Studio">
      <meta property="og:description" content="Professional visual tool for designing sophisticated CSS background textures and lighting effects.">
      <meta name="twitter:card" content="summary_large_image">
    `,
  },
}
```

Migration steps:

1. Move `seo.title` and `seo.description` into `metadata.title` and `metadata.description`.
2. Convert Open Graph `<meta>` tags into `metadata.openGraph`.
3. Convert `keywords` `<meta>` into `metadata.keywords` (array of strings).
4. Remove duplicated SEO tags from `customCode.head`, keeping only tags that cannot be represented via `metadata` (for example certain verification tags or platform-specific meta).

Example migrated config:

```ts
import type { Metadata } from 'talizen'

export default {
  importMap: {
    imports: {
      'lucide-react': 'https://esm.sh/lucide-react',
    },
  },
  metadata: {
    title: 'Merry Gradient | Professional CSS Gradient & Light Glow Studio',
    description:
      'Craft sophisticated multi-layered CSS background textures and stunning lightglow gradients.',
    keywords: [
      'CSS Gradient',
      'Light Glow',
      'Web Design Tool',
      'Background Texture',
      'Multi-layered Gradient',
      'UI Design',
      'CSS Generator',
    ],
    openGraph: {
      type: 'website',
      title: 'Merry Gradient | Professional CSS Gradient & Light Glow Studio',
      description:
        'Professional visual tool for designing sophisticated CSS background textures and lighting effects.',
      // Optionally add url, siteName, images, etc.
    },
  } satisfies Metadata,
  customCode: {
    head: `
      <meta name="twitter:card" content="summary_large_image">
    `,
  },
}
```

## AI SEO Workflow

When AI is asked things like:
- “基于当前的代码，帮我优化一下页面的 SEO 标题和描述”
- “帮我补全 OG 和 keywords”

Recommended steps:

1. Inspect `talizen.config.ts` to understand global `metadata`.
2. Inspect the relevant page component to see if it exports `metadata`.
3. Propose and apply changes primarily in:
   - `metadata.title` / `metadata.description`
   - `metadata.keywords`
   - `metadata.openGraph`
4. Only use `customCode.head` for SEO when something cannot be expressed through `metadata`.
