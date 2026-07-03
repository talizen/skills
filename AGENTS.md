# AGENTS.md — how the Talizen / Creght skills relate

There are **three sibling agent-skills** for the same platform. They share most
guidance but target different audiences and must be kept consistent. Read this
before editing any skill file.

## The three skills

| Skill | Location | Audience | Tooling model |
| --- | --- | --- | --- |
| **talizen-system** | `talizen-system/` (this repo) | The **in-platform agent** that runs inside the Talizen / Creght editor | Has platform tools: `lint`, `create_version`, `diff_patch_file`, `fetch_module_types`, import-map, preview. **No CLI workflow.** |
| **talizen** (external) | `skills/talizen/` (this repo) | End users / general-purpose agents working **locally via the Talizen CLI** | CLI workflow (pull / push / watch / preview). Platform tools only *"if the environment exposes them, otherwise use the CLI"*. |
| **creght** (external) | `skills/creght/` in the separate **`creght-dev/skills`** repo | End users of **Creght** | Same as external `talizen`, but Creght-branded. |

- **Creght integrates Talizen's capabilities**, and Talizen is also operated as
  an independent platform. Creght and Talizen therefore share the same code
  model and SDK — the only differences are the platform name and branding.
- **`talizen-system` is the source of truth.** It is the newest and most
  complete. The two external skills are downstream and are kept in sync with its
  *audience-neutral* content.

## Syncing content from `talizen-system` → the external skills

Do **not** blind-copy. When porting guidance from `talizen-system`:

1. **Filenames** — `talizen-system` uses `UPPERCASE.md` under `references/`
   (e.g. `CMS.md`); the external skills use `lowercase.md` (`cms.md`, `forms.md`,
   `error-handling.md`). Keep each skill's own naming — `SKILL.md` links files by
   name.
2. **Keep external-only files** — `cli.md` and `site-code.md` exist only in the
   external skills (they document the CLI / local workflow). `talizen-system` has
   no equivalent; never delete them during a sync.
3. **Filter platform-only content** — strip or rewrite anything that assumes the
   in-platform agent: the `lint` / `create_version` / `diff_patch_file` /
   `fetch_module_types` / import-map tools, the `BROWSER_ERROR_RENDER` signal, and
   preview-runtime specifics. Re-express as CLI / general guidance, or drop it.
4. **`error-handling` is intentionally two documents** — `talizen-system` keeps
   the platform/preview version (`BROWSER_ERROR_RENDER`, preview-runtime, the
   built-in React runtime). The external skills carry a **CLI-oriented** version.
   Keep them separate; do not merge them back into one.

## Creght rebranding (external `creght` only)

Creght runs on Talizen, so only **prose brand words** change:

- Prose: `Talizen` → `Creght`; `talizen.com` → `creght.cn`.
- **Unchanged**: package imports stay `talizen`, `talizen/cms`, `talizen/form`;
  the config file stays `talizen.config.ts`; the locale cookie stays
  `CREGHT_LOCALE` (it is already Creght-named upstream).
