# Website Deployment

## Goal

The new `website/` directory is the docs and marketing site for Bub.

Legacy MkDocs source files may still exist in the repository during the
transition, but production deployment now targets the Astro site on
Cloudflare Workers.

## Cloudflare Workers

Connect the repository to a Cloudflare Worker using Git integration.

Recommended settings:

- Project name: `bub-website`
- Build command: `pnpm install --frozen-lockfile && pnpm build`
- Deploy command: `pnpm wrangler deploy`
- Path: `website`
- Environment variable: `SITE_URL=https://bub.build`
- Environment variable: `NODE_VERSION=22.16.0`

The repo also includes [wrangler.jsonc](./wrangler.jsonc) so local preview and
Cloudflare Workers runtime settings stay aligned:

- `main = "./node_modules/@astrojs/cloudflare/dist/entrypoints/server.js"`
- `compatibility_flags = ["nodejs_compat"]`

Production deployment is handled by Cloudflare Workers Git integration instead of
GitHub Actions.

## Current Repo State

The local developer entrypoints now target the new site:

- `just docs`
- `just docs-test`
- `just docs-preview`

The CI docs check also builds `website/` instead of MkDocs.

## GitHub Actions and Cloudflare Responsibilities

The deployment split is intentionally simple:

- `main.yml` only verifies that the website builds
- `on-release-main.yml` only handles package release tasks
- Cloudflare Workers deploys the website from the connected repository

Required Cloudflare Workers project configuration:

- Git integration enabled for this repository
- Build command set to `pnpm install --frozen-lockfile && pnpm build`
- Deploy command set to `pnpm wrangler deploy`
- Working directory set to `website`

## Cutover Later

Once the Cloudflare Pages project is live and verified, the remaining cleanup
work is:

1. remove legacy MkDocs source files once they are no longer needed
2. update repository docs that still describe the old docs toolchain
