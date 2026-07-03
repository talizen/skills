---
title: Talizen CMS Usage
---

# Talizen CMS Usage

This document describes how to use the CMS APIs exported by `talizen/cms`.
It intentionally does not duplicate the package's TypeScript declarations,
because those declarations can change when the npm package is updated.

Use your project's `/types/cms.d.ts` as the schema source of truth, and use
`talizen/cms` for the platform-level fetch helpers.

## types/cms.d.ts file

You can find all CMS schema definitions in `/types/cms.d.ts`.

NOTE: This file is system-generated and cannot be edited, Call `create_collection` tool when a new cms collection is required. This file will be updated upon successful collection creation.

Rules:
1. Read this file before writing CMS-related code.
2. Import the needed project types from this file, for example:
   `import type { Blogs, Authors } from "./types/cms"`
3. Treat CMS fields as optional unless your generated schema guarantees
   otherwise.

Example:

```ts
export declare const CmsList: readonly [
  {
    key: "blogs"
    name: "Talizen's blogs"
    Item: Blogs
  },
  {
    key: "authors"
    name: "Talizen's authors"
    Item: Authors
  },
]

export interface Blogs {
  readonly __cmsKey: "blogs"
  slug: string
  id: string
  body: {
    title?: string
    content?: string
    author?: Authors
  }
}
```

## talizen/cms package definitions

This document keeps common CMS helper names and examples for quick use. For the
stable helpers listed below, do not fetch package declarations for every simple
use.

Fetch current package declarations only when you need an exact type/signature,
use an undocumented helper, see a package version change, or hit lint/typecheck
errors:

```ts
fetch_module_types("talizen/cms")
```

Rules:

1. Use the examples below as a quick reference for common flows.
2. Treat fetched declarations as the source of truth for exact parameter types,
   return types, generics, and optional arguments.
3. If a helper name, argument, or return type differs from an example, follow
   the fetched package declarations.
4. Use `get_import_map()` when you need to know which `talizen` package version
   or CDN URL is currently effective.

The generated `/types/cms.d.ts` file describes project content shapes. The
fetched `talizen/cms` declarations describe SDK helpers. Use both together.

Common helpers in current examples:

- `listContents` for paginated content lists.
- `getContent` for one item by slug.
- `getContentWithPrevNext` for detail pages that need adjacent items.

## List content

Use `listContents` for paginated content lists.

```ts
import { listContents } from "talizen/cms"
import type { Blogs } from "./types/cms"

export async function getServerSideProps() {
  const content = await listContents<Blogs>("blogs", {
    limit: 10,
    offset: 0,
    orderBy: "createdAt desc",
  })

  return {
    props: { content },
  }
}

export default function Page({ content }) {
  const list = content?.list ?? []

  if (list.length === 0) {
    return <main>No content yet.</main>
  }

  return (
    <main>
      {list.map((item) => (
        <article key={item.id}>{item.body?.title ?? "Untitled"}</article>
      ))}
    </main>
  )
}
```

Notes:
- The common return shape is `{ list?: T[]; total?: number }`.
- Common optional params include `limit`, `offset`, `searchKey`, `orderBy`, and
  `filter`. Fetch declarations only if you need exact parameter details.

### Order by
`orderBy` supported: `created_at`, `created_at desc`, `updated_at`, `updated_at desc`, `sort`, `sort desc`

default order by: `created_at desc`

## Filter content

Use `filter` when you need structured server-side filtering. Check the current
declarations if you need the exact supported filter shape or operators.

```ts
const content = await listContents<Blogs>("blogs", {
  limit: 10,
  filter: {
    match: "all",
    conditions: [
      { fieldId: "status", operator: "eq", value: "published" },
      { fieldId: "category", operator: "eq", value: "news" },
    ],
  },
})
```

## Get a single item

Use `getContent` when you need one content item by `slug`.

```ts
import { getContent } from "talizen/cms"
import type { Blogs } from "./types/cms"

export async function getServerSideProps(context) {
  const slug = context.params?.slug
  const content = await getContent<Blogs>("blogs", slug, {
  })

  return {
    props: { content },
  }
}
```

Notes:
- The common argument order is collection key, slug, optional params, optional
  request options. Fetch declarations only if you need exact details.

## Get current item with prev/next

Use `getContentWithPrevNext` for article detail pages that need adjacent items.

```ts
import { getContentWithPrevNext } from "talizen/cms"
import type { Blogs } from "./types/cms"

const article = await getContentWithPrevNext<Blogs>("blogs", slug, {
  prev: true,
  next: true,
  orderBy: "createdAt desc",
})
```

Common return shape:

```ts
{
  current?: Blogs
  next?: Blogs
  prev?: Blogs
}
```

## General CMS guidelines

- Keep CMS requests in `getServerSideProps` unless the project has a clear
  alternative data-loading pattern.
- Always use the generated schema in `/types/cms.d.ts` for content shape. (This file is system-generated and cannot be edited)
- Use optional chaining for nested fields, especially `body`.
- On multilingual sites, `listContents` / `getContent` / `getContentWithPrevNext` return
  each item already decoded to the current locale — just read the fields; don't read or
  merge `_i18n` yourself. See `references/i18n.md`.
- Do not rely on old helper names from legacy docs.
- When updating and creating content, follow the jsonSchema definition of the cms collection
