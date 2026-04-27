# rrmadmin/.github — Shared GitHub Actions workflows

Org-level reusable workflows for the rrmadmin ecosystem.

## Workflows

### `astro-cf-deploy.yml` — Astro + Cloudflare Pages deploy

Reusable workflow that builds an Astro site, runs the `site-ssot` prebuild
(if enabled), and deploys to Cloudflare Pages via wrangler.

**Why it exists:** CF Pages projects can be silently converted to "Direct
Upload" mode whenever `wrangler pages deploy` is run against them (one-way
conversion per CF docs / API error 8000069). Once converted, the native
GitHub App auto-deploy webhook stops firing and cannot be reconnected
without recreating the project (which destroys the `*.pages.dev` URL,
env vars, deploy history, and bindings).

This workflow replaces the broken native integration with a CI-driven
push-to-deploy pattern. Same UX (`git push origin main` → site updates),
runs on GitHub Actions instead of CF Pages' build infra.

**Side benefit:** ecosystem-wide consistency. Every Astro site in the
ecosystem references the same workflow → updates to the build pipeline
(new prebuild step, new validator, etc.) propagate to all sites
automatically on next push. No need to update 50 workflow files when
the build process changes.

### Usage

In a consumer repo, add `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: rrmadmin/.github/workflows/astro-cf-deploy.yml@main
    with:
      project-name: my-project-name  # CF Pages project name
    secrets: inherit
```

That's it. ~10 lines of YAML per site instead of ~80.

### Inputs

| Input | Required | Default | What |
|-------|----------|---------|------|
| `project-name` | yes | — | CF Pages project slug (e.g., `neofertility-ie`) |
| `branch` | no | `main` | Branch passed to `wrangler pages deploy --branch` |
| `ssot-enabled` | no | `true` | Run the `site-ssot` prebuild + check out rrm-tools + ecosystem-identity. Set `false` for sites that don't use site-ssot. |
| `build-staging-block` | no | `"1"` | `ASTRO_STAGING_BLOCK` env passed to build (1 = staging robots/headers, 0 = production). |
| `node-version` | no | `"22"` | Node version for the runner. |

### Per-repo secrets / vars (set once per consumer site)

`rrmadmin` is a User account, so org-level secret inheritance is not available. Each consumer site sets these at the repo level:

**Secrets:**
- `CLOUDFLARE_API_TOKEN` — CF API token with **Pages: Edit** scope on the ecosystem account(s).
- `GH_TOOLS_PAT` — GitHub PAT with read access to `rrmadmin/rrm-tools` and `rrmadmin/ecosystem-identity` (the two repos this workflow checks out at runtime). Same PAT can be reused across all consumer sites.

**Vars:**
- `CLOUDFLARE_ACCOUNT_ID` — the CF account that owns the consumer's Pages project.

Onboarding a new site: ~30 seconds of `gh secret set` / `gh variable set` per consumer.

### What the workflow does

1. **Checkout caller repo** (the Astro site).
2. **Checkout `rrmadmin/rrm-tools`** (only if `ssot-enabled` — contains `site-ssot`, `standards-gate`, etc.).
3. **Checkout `rrmadmin/ecosystem-identity`** (only if `ssot-enabled` — contains `people.json`, `organizations.json`, `credentials.json` registry).
4. **Stage ecosystem layout** — symlink the two checkouts to the paths that local builds expect (`~/iCode/tools/...` and `~/iCode/config/ecosystem-identity/`), so scripts that hardcode those paths Just Work in CI.
5. **Install npm deps** for the caller and for `site-ssot`.
6. **`npm run build`** (with `ASTRO_STAGING_BLOCK` and `SITE_SSOT_ENABLED` env vars set per inputs).
7. **`wrangler pages deploy`** the resulting `dist/`.

### Migrating an existing site to this workflow

1. Confirm the Astro site builds locally with `npm run build`.
2. Confirm a CF Pages project exists for the site (project-name slug).
3. Add `.github/workflows/deploy.yml` (snippet above).
4. Push to `main`. Watch Actions tab for the run.

### Caveats

- Sites that don't use `site-ssot` (yet) should pass `ssot-enabled: false` to skip the rrm-tools / ecosystem-identity checkouts.
- The PAT auth pattern is a 2026 short-term choice; graduating to a GitHub App token (short-lived, no rotation) is on the roadmap once the surface area justifies it.
- Concurrency is grouped per `project-name + branch`, so two simultaneous pushes to the same branch don't race; queued, not cancelled.
