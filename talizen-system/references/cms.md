---
title: Talizen CMS Usage
---

# Talizen CMS Usage

`/types/cms.d.ts` is the source of truth for content shapes; `talizen/cms` is
the fetch API. Read the generated file before writing CMS code and import from
it: `import type { Blogs, Authors } from "./types/cms"`. It is system-generated
and cannot be edited — change the schema with `create_collection` /
`update_collection`, then re-read it. Each item is
`{ __cmsKey, id, slug, body: { …fields } }`; treat fields as optional unless the
schema guarantees otherwise and use optional chaining, especially on `body`.

The helpers below cover the common cases: `listContents` (paginated lists),
`getContent` (one item by slug), `getContentWithPrevNext` (detail pages needing
neighbours). Call `fetch_module_types("talizen/cms")` only for an exact
signature, an undocumented helper, a package version change, or a
lint/typecheck error — fetched declarations always win over these examples.
`get_import_map()` shows the effective `talizen` version.

## Existing content operations

When the user asks to change existing CMS content, use the platform CMS tools
directly when the field exists and is already rendered: preserve unrelated
fields, update once, then re-read the entry to verify. If the field is missing
or not bound in the page, make the minimal schema or rendering change instead.

To reorder the CMS admin list, set each entry's `sort` via `update_content`
(bigger first); for the order visitors see, pass `orderBy` in `listContents`.
Never delete and recreate entries to reorder: ids change and `delete_content`
cannot be undone (site versions do not snapshot CMS content).

## List content

```ts
import { listContents } from "talizen/cms"
import type { Blogs } from "./types/cms"

export async function getServerSideProps() {
  const content = await listContents<Blogs>("blogs", {
    limit: 10,
    offset: 0,
    orderBy: "created_at desc",
  })
  return { props: { content } }
}
```

Returns `{ list?: T[]; total?: number }`. Optional params: `limit`, `offset`,
`searchKey`, `orderBy`, `filter`.

### Order by

`orderBy` is comma-separated `<field>[ asc|desc]`, default `asc`. Fields are
system columns (`id`, `created_at`, `updated_at`, `sort`) or body fields written
`body.<key>` (nested: `body.meta.rank`), e.g.
`orderBy: "body.date desc, created_at desc"`. Default is `sort desc, id desc`,
matching the CMS admin list.

- A bare body name (`"date desc"`) is a 400; unsupported fields fail loudly
  instead of silently falling back to the default order.
- Values compare as stored, so dates only sort correctly in ISO form
  (`2026-07-17`), which `format: date` produces.
- Entries missing the field sort last.
- Sort server-side; do not fetch a large `limit` and re-sort in JS.

## Filter content

Use `filter` for structured server-side filtering.

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

```ts
const content = await getContent<Blogs>("blogs", context.params?.slug)
```

Arguments: collection key, slug, optional params, optional request options.

## Get current item with prev/next

```ts
const article = await getContentWithPrevNext<Blogs>("blogs", slug, {
  prev: true,
  next: true,
})
```

Returns `{ current?: T; next?: T; prev?: T }`.

Known limitation: neighbours are resolved against the default
`sort desc, id desc` order only; any other `orderBy` still returns
default-order neighbours.

## Writing content bodies (rich text is HTML, not Markdown)

Rich-text body fields (e.g. an article `body`) store HTML, not Markdown: the
editor produces HTML, pages render it via `dangerouslySetInnerHTML`, and TOCs
are derived by scanning for `<h2>` / `<h3>`. Markdown put there renders as
literal text and yields no TOC.

Author them as HTML: `<h2>`/`<h3>` (no `<h1>` — the title is its own field),
`<p>`, `<strong>`, `<em>`, `<ul>`/`<ol>` with `<li><p>…</p></li>`,
`<blockquote>`, `<code>`, `<pre><code>` (HTML-escape `<`, `>`, `&` inside),
`<table>` with `<thead>`/`<tbody>`, and
`<a target="_blank" rel="noopener noreferrer nofollow">`.

Plain string fields (title, description, slug, …) stay plain text. When unsure,
read an existing entry with `list_contents` / `get_content` and match its `body`
format.

## General CMS guidelines

- Keep CMS requests in `getServerSideProps` unless the project has a clear
  alternative data-loading pattern.
- Follow the collection's jsonSchema when creating or updating content.
- On multilingual sites, `listContents` / `getContent` / `getContentWithPrevNext`
  return each item already decoded to the current locale — just read the fields;
  don't read or merge `_i18n` yourself. See `references/i18n.md`.
- A field's `contentMediaType` picks the editor control: `text/markdown` → the
  Markdown editor (use it with `type: "string"` for Markdown bodies),
  `text/html` → rich text, `image/*` / `video/*` → URL/upload, anything else →
  a generic file field.
- Do not rely on old helper names from legacy docs.
