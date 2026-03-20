---
title: Talizen CMS Usage
---

# Talizen CMS Usage

This document describes the current CMS API exported by `talizen/cms`.
It is based on the latest npm package definitions from `talizen@0.0.7`.

Use your project's `/types/cms.d.ts` as the schema source of truth, and use
`talizen/cms` for the platform-level fetch helpers.

## types/cms.d.ts file

You can find all CMS schema definitions in `/types/cms.d.ts`.

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

## talizen/cms package definition

`talizen@0.0.7` currently exports the following CMS types:

```ts
import { type TalizenRequestOptions } from "talizen/core"

export interface BaseCmsItem {
  readonly __cmsKey: string
  slug: string
  id: string
  body: Record<string, unknown>
}

export interface GetContentListFilterCondition {
  fieldId?: string
  operator?: string
  value?: any
}

export interface GetContentListFilter {
  match?: "any" | "all"
  conditions?: GetContentListFilterCondition[]
}

export interface ListContentParams {
  limit?: number
  offset?: number
  searchKey?: string
  orderBy?: string
  filter?: GetContentListFilter
}

export interface GetContentParams {
}

export interface GetContentWithPrevNextParams extends GetContentParams {
  prev?: boolean
  next?: boolean
  searchKey?: string
  orderBy?: string
  filter?: GetContentListFilter
}

export interface ListResponse<T> {
  list?: T[]
  total?: number
}

export interface ContentWithPrevNext<T extends BaseCmsItem> {
  current?: T
  next?: T
  prev?: T
}

export declare function listContents<T extends BaseCmsItem>(
  key: T["__cmsKey"],
  params?: ListContentParams,
  options?: TalizenRequestOptions,
): Promise<ListResponse<T>>

export declare function getContent<T extends BaseCmsItem>(
  key: T["__cmsKey"],
  slug: string,
  params?: GetContentParams,
  options?: TalizenRequestOptions,
): Promise<T>

export declare function getContentWithPrevNext<T extends BaseCmsItem>(
  key: T["__cmsKey"],
  slug: string,
  params?: GetContentWithPrevNextParams,
  options?: TalizenRequestOptions,
): Promise<ContentWithPrevNext<T>>
```

Important:
- The current package uses `listContents`, `getContent`, and
  `getContentWithPrevNext`.
- Older names like `ListContent` and `GetContent` should be treated as outdated
  documentation unless the project provides its own wrapper.
- All three APIs support an optional third or fourth `options` argument from
  `talizen/core`.

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
- `listContents` returns `{ list?: T[]; total?: number }`.
- `searchKey`, `orderBy`, and `filter` are supported in addition
  to `limit` and `offset`.

## Filter content

Use `filter` when you need structured server-side filtering.

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
- The second argument is the item `slug`.
- `getContent` returns a single typed item, not a `{ list, total }` wrapper.

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

Return shape:

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
- Always use the generated schema in `/types/cms.d.ts` for content shape. (!DON'T EDIT IT)
- Use optional chaining for nested fields, especially `body`.
- Do not rely on old helper names from legacy docs.
- When updating and creating content, follow the jsonSchema definition of the cms collection