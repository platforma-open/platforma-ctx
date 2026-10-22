# platforma-ctx: Instructions For AI Assistants

This folder is a workspace for building Platforma blocks. You help a developer write, change, test and release blocks.

Work from this folder, not from inside a block repo. The folder itself is not a git repo: platforma-ctx, the SDK and every block are separate git repos inside it.

## What Is Where

| Path | What it is |
|---|---|
| `ORG.md` | the organization's conventions: the name, URL, logo and support contact that every block shows, package scope, repos, registry, CI. If it exists, read it first; it wins over everything else here. |
| `blocks/` | the public Platforma blocks (read only; all of them, or those of the verticals chosen at setup), and the developer's own block repos (editable). Real, working code: learn from it. |
| `core/platforma/` | the Platforma SDK source (read only). The ground truth for every API. |
| `support/github-ci/` | the shared GitHub Actions workflows that the CI files of a block call (read only) |
| `.claude/skills/platforma-block-dev/SKILL.md` | how to work on a block: the loop, the commands, and the traps that the code does not show. **Read it before any block work.** |
| `platforma-ctx/` | the setup script `pl-ctx`, the repo lists, guides for people (`docs/`), the organization's logos (`org/*.png`), and the originals of `AGENTS.md`, `.claude/` and `ORG.md` (as `org/ORG.md`) |

## How To Work

1. Read `ORG.md` if it exists, then `.claude/skills/platforma-block-dev/SKILL.md`.
2. Find two or three blocks in `blocks/` that do something similar, and follow them.
3. Check each API you use in `core/platforma/`. When a block and the SDK disagree, the SDK is right.
4. Build and test. Say what you ran and what passed.

To find code by meaning, use grepai if it is set up (there is a `.grepai/` folder here): its MCP tools when they are connected, otherwise `grepai search "<question>"` in a terminal. Use grep for exact names.

For a non-trivial change, write a short plan first and wait for the developer.

## Rules

- **Code wins.** If a guide in platforma-ctx disagrees with the code, follow the code and tell the developer which line of the guide is wrong.
- **Say when you are not sure.** Write "not verified" instead of guessing.
- **Do not publish.** Never run `pnpm publish`, `pnpm build:release`, `pnpm build:dev-remote`, a docker push, or a push to a shared branch, unless the developer asks for exactly that. Usually only CI publishes.
- **Do not edit the reference code.** Public blocks in `blocks/` and the SDK in `core/platforma/` are for reading. Change only the developer's own blocks.
- **Do not edit the copies.** `AGENTS.md`, `ORG.md` and `.claude/` at the top of this folder are copies, and `pl-ctx update` replaces them. Change the originals in `platforma-ctx/`.
- **No real data.** Tests and examples use small synthetic data. Never put real sequences, sample or patient data into code, tests, commits or prompts to external services.
- **No secrets** in files.
