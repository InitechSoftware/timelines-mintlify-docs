# TimelinesAI API Documentation (Mintlify)

This repository holds the [Mintlify](https://mintlify.com) documentation site published at
**[timelines.ai/docs](https://timelines.ai/docs/)**.

It contains the MDX pages, navigation (`docs.json`), and **copies of the OpenAPI specs** that Mintlify
renders into the interactive API reference. The specs are authored in a separate upstream repo and pulled
in here by the sync workflow (below). This README documents how **this** repo works; it does not cover the
upstream authoring process.

## Repository layout

| Path | Purpose |
|---|---|
| `docs.json` | Navigation, theming, and OpenAPI references (Mintlify config; replaces the older `mint.json`) |
| `openapi/*.yaml` | Copies of the four OpenAPI specs that Mintlify renders (pulled by the sync workflow) |
| `public-api-reference/*.mdx` | Public API endpoint pages |
| `partner-api-reference/*.mdx` | Partner API endpoint pages |
| `webhook-reference/*.mdx` | Public webhook event pages |
| `guides/*.mdx` | Narrative guides |
| `public-api-reference/overview.mdx` | Custom overview page with **hardcoded** cards and an endpoint table |
| `.github/workflows/sync-openapi.yml` | The spec-sync workflow (see below) |

### The four OpenAPI specs

| Spec (`openapi/`) | Mintlify tab | Has a `servers` block? |
|---|---|---|
| `public_api_spec.yaml` | Public API | Yes (rewritten to absolute) |
| `webhook_spec.yaml` | Public API Webhooks | No |
| `partner_api_spec.yaml` | Partner API | Yes (rewritten to absolute) |
| `partner_api_webhook_spec.yaml` | Partner API Webhooks | No |

## How OpenAPI specs get here — the `sync-openapi` workflow

The `openapi/*.yaml` files are **not authored in this repo**; they are copies pulled from the upstream
`InitechSoftware/timelines` repo. Syncing is a **pull** performed by a GitHub Action **in this repo** —
the upstream repo is only read, never modified.

`.github/workflows/sync-openapi.yml` is triggered **manually**:

- **Actions tab** → *sync-openapi* → *Run workflow* (optionally set the `source_ref`, default `production`), or
- CLI: `gh workflow run sync-openapi.yml --repo InitechSoftware/timelines-mintlify-docs -f source_ref=production`

It:

1. Shallow-clones `InitechSoftware/timelines` at `source_ref` using the read-only PAT in the
   `TIMELINES_RO_TOKEN` secret.
2. Copies all four YAMLs into `openapi/` (full-file copy — every path, schema, description, example).
3. **Rewrites the `servers` URL to absolute** on the two specs that have one (see below), and **fails the
   job** if the expected line can't be rewritten (no silent regressions).
4. Commits **only if `openapi/` actually changed** and pushes to `master`.

Pushing to `master` triggers Mintlify's GitHub App, which **auto-deploys** the site within a couple of
minutes (plus CDN cache).

### Why the `servers` URL is rewritten

The upstream source specs use a **relative** `servers` URL (e.g. `/integrations/api`). Mintlify builds the
"Try it" playground request from the OpenAPI `servers` field and resolves a relative URL against the **docs
host**, not the API host — so with a relative URL the playground breaks with
*"A valid request URL is required to generate request example."* This repo's copy therefore **must** carry an
absolute URL. The workflow rewrites:

| Spec | Relative (upstream) | Absolute (this repo) |
|---|---|---|
| `public_api_spec.yaml` | `/integrations/api` | `https://app.timelines.ai/integrations/api` |
| `partner_api_spec.yaml` | `/partner/api/v1` | `https://app.timelines.ai/partner/api/v1` |

The rewrite is **idempotent** (a no-op once absolute) and the workflow **fails loudly** if the upstream
`servers` line ever changes format/path so the rewrite can't be applied — rather than shipping a spec with
no valid base URL. If that failure happens, update the relative/absolute pairs in `sync-openapi.yml`.

> Never hand-edit the `servers` URL in `openapi/*.yaml` to fix it permanently — the next sync would overwrite
> your edit. The rewrite belongs in the workflow.

## Adding, updating, or removing an endpoint page

Endpoint content comes from the OpenAPI YAML; MDX files are a thin override/placement layer. The rendering
hierarchy:

```
YAML spec (content: paths, schemas, descriptions, examples)
  └─ MDX frontmatter (overrides: title, description)
      └─ MDX body (additions: <Warning>, <Note>, custom text)
          └─ docs.json (structure: tabs, groups, page order)
```

### Add an endpoint page

1. Make sure the endpoint exists in the relevant `openapi/*.yaml` (comes from a sync).
2. Create an MDX file in the matching directory (`public-api-reference/`, `partner-api-reference/`,
   `webhook-reference/`). Filenames are conventionally derived from the spec's `summary`, lowercased and
   hyphenated. Frontmatter, matching the style already used in that directory:

   ```mdx
   ---
   openapi: get /chats
   title: "List Chats"
   description: "Short one-sentence summary."
   ---
   ```

   - `openapi` — `method path` matching the spec exactly (e.g. `get /chats`, `post /workspace/invitations`).
     Webhook pages use this repo's convention `openapi: "webhook <event>"` (e.g. `openapi: "webhook message:received"`).
     Mintlify resolves the method+path against the four specs listed in the top-level `openapi` array in
     `docs.json`, so a per-page spec prefix is normally unnecessary (keep method+path unique across specs).
   - `title` — short human form used for the page heading and nav.
   - Custom content (e.g. `<Warning>` for deprecation) goes below the closing `---`.
3. Add the page path (without `.mdx`) to `docs.json` → `navigation.tabs` → the correct tab → group.
4. If it belongs in the hardcoded lists in `public-api-reference/overview.mdx`, update those too.

### Update an endpoint

- Content/schema/description changes come from the spec → **run a sync**; the MDX auto-reflects.
- If the spec `summary` changed and you renamed the MDX file, update the path in `docs.json`.

### Remove an endpoint

1. Ensure the path is gone from the spec (upstream), then sync.
2. Delete the MDX file.
3. Remove the page reference from `docs.json`.
4. Check `public-api-reference/overview.mdx` for hardcoded references.

## Repo conventions & gotchas

- **`docs.json` navigation is fully manual** — nothing auto-adds pages to the nav.
- **`public-api-reference/overview.mdx` has hardcoded cards and an endpoint table** — update it when adding
  or removing endpoints; it does not auto-generate.
- **Every endpoint MDX needs `openapi`, `title`, `description`** in frontmatter.
- **Filenames derive from the spec `summary`** — changing a summary and renaming a file orphans the old path
  in `docs.json`.
- **Don't hand-edit `openapi/*.yaml`** for content — those are sync copies and will be overwritten. Content
  edits happen upstream; only the `servers` rewrite is a deliberate this-repo difference (applied by the
  workflow, not by hand).
- **Encoding:** edit YAML/MDX in place; don't pipe files through PowerShell (it can corrupt encoding / inject
  BOM).

## Verifying a change

After a sync + Mintlify deploy:

```bash
# Absolute servers URL is live
curl -sL https://timelinesai.mintlify.dev/docs/openapi/public_api_spec.yaml | grep -A2 '^servers:'
# A specific page renders
curl -sL https://timelinesai.mintlify.dev/docs/public-api-reference/<page-slug>.md
```

Also open an endpoint page (e.g. **List Chats**) and confirm the "Try it" playground shows the
`app.timelines.ai` base URL rather than the error message.

| URL | What it is | Verify against it? |
|---|---|---|
| `timelinesai.mintlify.dev/docs/` | Mintlify origin | **Yes** |
| `timelines.ai/docs/` | Vercel proxy (customer-facing); may cache and can break root-relative static assets | No |
| `timelinesai.mintlify.app/` | Legacy subdomain | No (stale) |

## The `TIMELINES_RO_TOKEN` secret

- A **fine-grained PAT** scoped to **only** `InitechSoftware/timelines`, **Contents: Read-only**.
- Stored as a repo Actions secret (Settings → Secrets and variables → Actions) — GitHub-side metadata,
  never committed to the repo.
- **Max expiry is 366 days.** When it lapses, `sync-openapi` fails at the clone step and docs stop updating.
  **Rotate before expiry:** create a new fine-grained PAT with the same scope and update the secret:
  `gh secret set TIMELINES_RO_TOKEN --repo InitechSoftware/timelines-mintlify-docs`

## Local development

Install the [Mintlify CLI](https://www.npmjs.com/package/mint) and run at the repo root (where `docs.json` lives):

```
npm i -g mint
mint dev
```

Preview at `http://localhost:3000`.

## Publishing

Changes deploy to production automatically after pushing to the default branch (`master`), via the Mintlify
GitHub App. For **spec** updates, prefer running the `sync-openapi` workflow rather than hand-editing the
`openapi/` copies, so the upstream source stays authoritative.
