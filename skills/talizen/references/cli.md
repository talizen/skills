# Talizen CLI

The Talizen CLI is a Go command-line tool that acts as a local bridge for
Talizen site code.

It supports:

- Logging in to Talizen from the terminal.
- Listing projects and sites.
- Pulling remote site files into a local directory.
- Pushing the current local site file snapshot back to Talizen.
- Watching local site files and syncing changes back to Talizen.
- Opening or creating a platform preview.
- Publishing the current site state or a specific commit.
- Uploading local files through the Talizen site asset flow.

The CLI does not render sites locally. Rendering, CMS, assets, and realtime
preview are handled by the Talizen backend and web app.

## Install

Install the CLI from npm:

```bash
npm install -g talizen-cli
```

The package installs the `talizen` command. It requires Node.js 18 or newer and
supports macOS, Linux, and Windows on x64 or arm64.

Verify the install before using the CLI:

```bash
talizen version
```

## Endpoints

The CLI uses the Talizen production endpoint by default:

```text
API: https://talizen.com
Web: https://talizen.com
```

Omit `--api` and `--web` unless the user explicitly provides a different
Talizen endpoint for their environment.

## Common Commands

```bash
talizen login
talizen projects
talizen pull --site_id=<project_id>/<site_id> --dir=./mysite
talizen push --site_id=<project_id>/<site_id> --dir=./mysite
talizen sync --site_id=<project_id>/<site_id> --dir=./mysite
talizen preview --site_id=<project_id>/<site_id>
talizen publish
talizen publish --commit=<commit>
```

## Working With Site Files

If the site is not already local, use `talizen projects` to find the project and
site ID, then `talizen pull` into a target directory. Edit files locally, then
use `talizen push` to upload the current local snapshot back to Talizen.

Use `talizen sync` when you want watch mode. `sync` first pushes the current
local snapshot, then keeps running and automatically listens for later local
file changes.

Both `push` and `sync` are one-way local-to-remote workflows. They do not pull
Web editor changes back to the local directory. If the same site was edited in
the Web editor, run `talizen pull` manually or restart from a clean local copy
before continuing.

When verification depends on platform rendering, use `talizen preview` instead
of starting a local renderer. A Talizen site is not expected to render locally by
default.

## Platform Data Commands

General-purpose agents should use CLI commands for platform data operations such
as CMS collections, CMS content, forms, and generated types. Do not assume
Talizen-system-only tools such as `create_collection`, `create_form`, or direct
internal patch helpers exist outside the Talizen system agent.

CMS collections:

```bash
talizen cms collections --site_id=<project_id>/<site_id>
talizen cms collection get --site_id=<project_id>/<site_id> (--id=<id> | --key=<key>)
talizen cms collection create --site_id=<project_id>/<site_id> --key=<key> --name=<name> --schema=./schema.json
talizen cms collection update --site_id=<project_id>/<site_id> (--id=<id> | --key=<key>) [--new-key=<key>] [--name=<name>] [--desc=<desc>] [--schema=./schema.json]
talizen cms collection delete --site_id=<project_id>/<site_id> (--id=<id> | --key=<key>)
```

CMS content:

```bash
talizen content list --site_id=<project_id>/<site_id> --collection=<key> [--limit=20] [--offset=0] [--filter=./filter.json]
talizen content get --site_id=<project_id>/<site_id> --collection=<key> (--id=<id> | --slug=<slug>)
talizen content create --site_id=<project_id>/<site_id> --collection=<key> --data=./content.json [--slug=<slug>] [--sort=0]
talizen content update --site_id=<project_id>/<site_id> --collection=<key> --id=<id> --data=./content.json [--slug=<slug>] [--publish=true]
talizen content delete --site_id=<project_id>/<site_id> --collection=<key> --id=<id>
```

Forms:

```bash
talizen form list --site_id=<project_id>/<site_id>
talizen form get --site_id=<project_id>/<site_id> (--id=<id> | --key=<key>)
talizen form create --site_id=<project_id>/<site_id> --key=<key> --name=<name> --schema=./schema.json
talizen form update --site_id=<project_id>/<site_id> (--id=<id> | --key=<key>) [--new-key=<key>] [--name=<name>] [--desc=<desc>] [--schema=./schema.json] [--setting=./setting.json]
talizen form delete --site_id=<project_id>/<site_id> (--id=<id> | --key=<key>)
talizen form logs --site_id=<project_id>/<site_id> (--id=<id> | --key=<key>) [--limit=20] [--offset=0]
talizen form log get --site_id=<project_id>/<site_id> (--id=<form_id> | --key=<form_key>) --log_id=<log_id>
talizen form log delete --site_id=<project_id>/<site_id> (--id=<form_id> | --key=<form_key>) --log_id=<log_id>
talizen form submit --site_id=<project_id>/<site_id> --key=<form_key> --data=./payload.json
```

Prefer file-based JSON input for schemas and content payloads. It is easier for
agents to inspect, validate, edit, and retry than deeply escaped JSON passed
inline on the command line.

After creating or changing collections/forms, pull or refresh generated files
such as `/types/cms.d.ts` and `/types/form.d.ts` before writing code that imports
those types.

## Asset Upload

Use `talizen upload` when site code needs a local file to become a Talizen-hosted
asset.

```bash
talizen upload --site_id=<project_id>/<site_id> --file=./image.png
talizen upload --site_id=<project_id>/<site_id> --file=./image.png --name=hero.png --json
```

By default the command prints the public file URL. With `--json`, it prints the
full upload metadata, including `site_path`, a stable `/_assets/...` path that
can be referenced from Talizen site code. Prefer `site_path` in site code when a
stable project-relative asset path is better than a CDN URL.
