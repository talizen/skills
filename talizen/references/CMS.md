---
title: Talizen CMS Usage
---

# Talizen CMS Usage

This document describes how to use the Talizen CMS APIs from code. It focuses on fetching content via `talizen/cms` inside `getServerSideProps` and safely rendering it in page components.

Talizen provides at least two primary helpers:
- `ListContent` — fetch a paginated list of content items
- `GetContent` — fetch a single content item by identifier

All fields returned from CMS may be missing or null, so code must use optional chaining and defensive access.

## types/cms.d.ts file
You can find all cms type definition in `/types/cms.d.ts` file.

Rules:
1. Read this file before writing CMS-related code. Fields like `__cmsKey`,
   `slug`, `id`, and the exact `body` shape are useful when implementing logic.
2. When writing code, import the needed types from this file, for example:
   `import type { Blogs, Authors } from "./types/cms"`

example:
```
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

export interface Authors {
  readonly __cmsKey: "authors"
  slug: string
  id: string
  body: {
    name?: string
    avatar?: string
  }
}
```

## talizen/cms package definition
```
export interface ListContentParams {
    limit?: number
    offset?: number
    searchKey?: string
}

interface BaseCmsItem {
    readonly __cmsKey: string
}

export declare function ListContent<T extends BaseCmsItem>(
    key: T['__cmsKey'],
    params: ListContentParams,
): Promise<T[]>

export declare function GetContent<T extends BaseCmsItem>(
    key: T['__cmsKey'],
    slug: string
): Promise<T>
```

## Listing Content

Use `ListContent` to fetch lists such as blog posts, articles, or products.

```ts
import { ListContent } from 'talizen/cms'

export async function getServerSideProps(context) {
  const res = await ListContent('blogs', { page: 1, offset: 0 }) // { total, list }

  return {
    props: { content: res },
  }
}

export default function Page({ content }) {
  const list = content?.list

  // Always guard for null/undefined values
  if (!list || list.length === 0) {
    return <main>No content yet.</main>
  }

  return (
    <main>
      {list.map((item) => {
        const title = item?.body?.title ?? 'Untitled'
        return <article key={item?.id}>{title}</article>
      })}
    </main>
  )
}
```

Guidelines:
- The first argument to `ListContent` is the collection name (for example `'blogs'`).
- The second argument is an options object; typical fields include `page` and `offset`.
- The return shape includes at least `total` and `list`; treat `list` as optional.

## Getting a Single Content Item

Use `GetContent` when you need one specific item, for example a blog detail page.

```ts
import { GetContent } from 'talizen/cms'

export async function getServerSideProps(context) {
  const slug = context.params?.slug
  const res = await GetContent('blogs', slug)

  return {
    props: { content: res },
  }
}

export default function Page({ content }) {
  const title = content?.body?.title ?? 'Untitled'
  const body = content?.body?.content ?? ''

  return (
    <main>
      <h1>{title}</h1>
      <div>{body}</div>
    </main>
  )
}
```

Guidelines:
- The first argument is the collection name; the second is the identifier (for example `slug` from the route).
- Always treat `content` and nested fields as optional; use `?.` and default values.

## General CMS Best Practices

- Keep CMS access inside `getServerSideProps`, and pass plain serializable props into the page component.
- Use optional chaining (`?.`) for all CMS-derived fields to avoid runtime errors when fields are missing.
- Provide user-friendly fallbacks (for example “Untitled”, “No content yet”, or empty strings).
- Avoid assuming specific schema fields beyond what is clearly documented in the project; if unsure, inspect existing code or sample responses before using new fields.

