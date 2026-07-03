# Talizen Skills

This repository provides Codex/agent skills for Talizen. They help agents understand Talizen project structure, CLI workflows, page development conventions, CMS/Form/SEO features, and other platform capabilities.

## What This Skill Does

- Guides agents through Talizen CLI workflows for creating projects, pulling, pushing, syncing, previewing, and publishing sites.
- Enforces Talizen page development conventions, such as `/page` routes, the `/component` directory, native `<a>` navigation, and `getServerSideProps` for data loading.
- Helps write React + Tailwind v4 pages and components that follow Talizen platform requirements.
- Provides implementation references for common platform capabilities such as CMS, form submissions, SEO metadata, sitemaps, and carousel components.
- Helps debug local-to-platform sync, preview, and publishing issues.

## Examples

```text
Add an About page to this Talizen project, following the existing page and component structure. {YOUR_TALIZEN_PROJECT_EDIT_URL like https://talizen.com/editor/project/pveao61akhoy/site/pveao646es1u}
```

```text
Connect the homepage to Talizen CMS data, using the schema and types that already exist in the project. {YOUR_TALIZEN_PROJECT_EDIT_URL like https://talizen.com/editor/project/pveao61akhoy/site/pveao646es1u}
```

```text
Optimize the project's SEO configuration, including title, description, keywords, Open Graph, and related metadata. {YOUR_TALIZEN_PROJECT_EDIT_URL like https://talizen.com/editor/project/pveao61akhoy/site/pveao646es1u}
```

## Installation

Install with `npx`:

```bash
npx skills add talizen/skills -g -y
```

Or use `bunx`:

```bash
bunx skills add talizen/skills -g -y
```

After installation, supported Codex/agent environments will automatically load the corresponding skill guidance when working with Talizen projects.

## Update

Update to the latest version with `npx`:

```bash
npx skills update talizen
```

Or use `bunx`:

```bash
bunx skills update talizen
```

Re-running the add command above also works — it is idempotent and pulls the latest.
