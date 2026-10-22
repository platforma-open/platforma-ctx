# <Organization> Block Development Conventions

Copy this file to `org/ORG.md` and fill it in. Keep it short and factual. AI assistants read it before any block work, and it wins over the generic guides in platforma-ctx.

## Blocks We Own

| Block | Repo | Started from | What it does |
|---|---|---|---|
| `<block-name>` | `<git url>` | `<public block it was forked from, or "new">` | `<one sentence>` |

## Package Names

- Form: `@<npm-scope>/<organization>.<block>`, for example `@acme-bio/acme.region-qc`. The build tool requires the dot. `block-tools structure init` writes it from `--npm-org @acme-bio --org-scope acme --short-name region-qc`, and names the part packages `@acme-bio/acme.region-qc.model`, `.ui`, `.workflow`, `.kind`.
- npm scope: `@<npm-scope>`. Organization word: `<organization>`.
- A registry identifies a block as `<organization>/<block>`; the npm scope is not part of it. Never use `platforma-open` or `milaboratories` as the organization word: the id would clash with public blocks.

## Where Blocks Are Published

- Block registry: `<https URL of your registry>`
- Software images (docker), if any: `<registry host>`
- Who publishes: `<for example: only CI, on merge to main>`

## Review And Release

- Branches: `<for example: push to a feature branch, open a pull request>`
- Review: `<who reviews>`
- Versions: changesets (`pnpm changeset`) in every pull request that changes a block.

## Backend Environments

| Environment | Address | Use it for |
|---|---|---|
| dev | `<host>` | trying dev builds of a block |
| test | `<host>` | checking a release before production |
| production | `<host>` | real work |

## Data Rules

- `<for example: tests use synthetic sequences only; never commit real sequences or datasets>`
