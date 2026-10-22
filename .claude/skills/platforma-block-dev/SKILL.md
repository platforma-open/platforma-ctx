---
name: platforma-block-dev
description: Platforma block development. Use for any work on a Platforma block — creating a block, changing its workflow (Tengo), model (TypeScript), UI (Vue) or kind, p-column specs, reading other blocks' columns, packaging software, tests, SDK upgrades, building and releasing, or a block that fails. Also use when the developer asks how blocks work.
---

# Platforma Block Development

## How To Work

Learn from real code. The workspace has the SDK source in `core/platforma/` and the public blocks in `blocks/` (all 74, or the verticals chosen at setup). They are the best guide to how a block is written today.

1. If `ORG.md` exists at the top of the workspace, read it. It wins over this file.
2. For a non-trivial change, write a short plan first (which parts change, which columns the block reads and writes, how you will test) and wait for the developer.
3. Find two or three blocks that already do something similar, and follow them. Search them:
   ```bash
   grep -rl "isSubset" blocks/*/workflow/src          # who emits subset filter columns
   grep -rl "PlDropdownRef" blocks/*/ui/src           # who uses a component
   ls -d blocks/*/kind                                # current blocks (they have a kind)
   ```
4. Check every API you use in the SDK source: `core/platforma/sdk/model/src/`, `core/platforma/sdk/workflow-tengo/src/`, `core/platforma/sdk/ui-vue/src/`, `core/platforma/sdk/block-kind/`.
5. Build and test. A change is done when the block builds and its tests pass. Say what you ran.

Blocks share patterns that are wrong. When two blocks disagree, the SDK code decides. Prefer blocks that have a `kind/` folder and use `BlockModelV3`: older blocks use APIs the SDK no longer has.

## What A Block Is

| Part | Language | Runs in | Job |
|---|---|---|---|
| kind | TypeScript | build and the Desktop App | identity; the parameters a block can be created with |
| model | TypeScript | the Desktop App, in a sandbox | turns settings (`data`) into workflow arguments (`args`), and results into UI data (`outputs`) |
| UI | Vue 3 | the Desktop App | what the user sees and sets |
| workflow | Tengo | the backend | the pipeline: reads arguments and other blocks' columns, runs software, writes p-columns |
| software | Python, R, Java, docker | jobs on the backend | the tools the workflow runs |

Blocks never call each other. A block exports p-columns into the project; other blocks find them by spec.

A block is finished when five things are defined, each one per column, page or step:

- the columns it reads: by spec, relative to the input the user picks, never by the id of the block that made them;
- the columns it exports: their full specs, because other blocks find them by spec;
- the output behind every page and plot, and what the page shows while that output is still empty;
- the resources (CPU, memory, GPU) of every software step, with a formula when they grow with the input;
- what users read about it: `block.meta` without placeholders, and a description written by `references/block-description.md`.

## Traps The Code Will Not Tell You

Checked against `@platforma-sdk/model` 1.85.2, `workflow-tengo` 6.16.0, `block-tools` 2.16.16 (2026-10-21).

**Start and build**

- Start a block with `pnpm dlx @platforma-sdk/block-tools@latest structure init ./<dir> --npm-org @<scope> --org-scope <org> --short-name <name> --no-software --non-interactive`. Not `npx` (it fails). Not the `platforma-block-boilerplate` repo or `pl-dev create-block` (an old layout without the kind).
- The kind is mandatory. A new block does not compile until you replace `NEEDS_BLOCK_PARAMS` in `kind/src/index.ts` (`Record<string, never>` for no parameters). The model is `BlockModelV3.create({ dataModel, kind })` and needs `.templateParams(...)`; fill the scaffold's empty `.args(() => ({}))` and `.templateParams(() => ({}))`. A complete example: `blocks/vj-gene-usage/kind/src/index.ts` and `model/src/index.ts`.
- There is no `pnpm build`. Use `pnpm build:dev-local` (needs Docker for Python, conda or docker software), or `env PL_BUILD_CHANNEL=dev PL_BUILD_VARIANT=binary PL_BUILD_LOCATION=local pnpm exec turbo run build` without Docker.
- Plain `pnpm watch` points software at a release. Use `env PL_BUILD_CHANNEL=dev PL_BUILD_VARIANT=binary PL_BUILD_LOCATION=local pnpm watch`.
- `block-tools structure` owns some whole files (tsconfig, lint config, the two CI workflows) and the tasks and fields it writes in `turbo.json` and each `package.json`. It rewrites them on every refresh, also during `pnpm upgrade-sdk`; what you add beside them stays. `pnpm exec block-tools structure check` shows what it would change.
- `pnpm check` and the build run `pl-tengo imports`, which deletes unused imports from Tengo files. `pnpm fmt` does not format Tengo.

**Metadata and description**

- `structure init` writes placeholders into `block/package.json`, `block.meta`: the title is the short name, the description is "<name> block", both logos are default images, the organization name is the `--org-scope` word, and its URL is `https://example.com`. `structure refresh` never changes `block.meta`, so nothing replaces them later. Replace every one before anyone else sees the block.
- The organization (name, URL, logo), `support`, the block's `url` and its logo come from `ORG.md`. Never copy them from a public block: those name MiLaboratories. If `ORG.md` does not have them, ask the developer.
- Paths in `block.meta` are relative to the `block/` folder: init puts the logos in `block/logos/` (`file:logos/block-logo.png`). Logos are PNG files: the block logo square, the organization logo a wordmark or a square.
- The Desktop App shows `organization.name` as "Developer", and `url`, `docs` and `support` in the side panel. Write `description` and `docs/description.md` (`"longDescription": "file:../docs/description.md"`) by `references/block-description.md`.
- In the Add Block dialog, a dev block always shows "Value not available" in the version list: dev blocks have no versions. It is not an error.

**Test**

- `pnpm test` runs format and lint checks first (`pnpm fmt` fixes formatting). Turbo caches results; add `--force` to run again. Python tests are separate (`pytest`).
- The first test file in a model, UI or workflow package runs only after `pnpm exec block-tools structure refresh` and `pnpm install` add that package's test script. A kind test needs its `test` script and `vitest` added by hand (see `blocks/vj-gene-usage/kind/package.json`).
- The `test/` package that `structure init` creates is not connected to the block. Add the block as a `workspace:` dependency before writing a block test (pattern: `blocks/redefine-clonotypes/test/package.json`).
- Block tests need a backend you can log in to with a password (or one without login), and a license. A backend with single sign-on only cannot run them; run one on the laptop: `references/local-backend.md`.

**Workflow (Tengo)**

- A template starts with `self := import("@platforma-sdk/workflow-tengo:tpl")`. Inside a block, `":tpl"` means the block's own package.
- A template must give the same result every time: iterate over arrays, not over map keys. `json.encode` of a map can change key order.
- In a trace step, `type` is a string and `id` is `wf.blockId().getDataAsJson()`. `wf.blockId().getDataAsJson().package` crashes at run time.
- Without `.prerunArgs()` in the model, the prerun gets `args`. A separate prerun template needs `wf.setPreRun(...)`.
- The main run sees only the blocks that its `args` point to (`PlRef`) and their upstream blocks. The model's dropdowns see every block above, and this block's own exports too: leave those out of input choices.

**Columns**

- A subset filter column (`pl7.app/isSubset: "true"`) filters by which rows exist. A 0/1 column with a row for every key filters nothing.
- A block's own id goes into the spec's domain (`pl7.app/blockId`). The SDK's `BlockScopedDomain` list and project templates expect it there.
- Antibody regions: amino-acid FR4 is `FR4InFrame` from MiXCR Clonotyping and `FR4` from Import V(D)J Data; a germline-imputed region differs from an observed one only by the annotation `pl7.app/vdj/imputed`. More: `references/reference-blocks.md`.

**Model and UI**

- Do not copy a model output back into the block's `data` from the UI without a guard: the new `data` changes the output again, and it can loop. If you must (for example a default label), write only when the value really differs, as `blocks/vj-gene-usage/ui/src/app.ts` does.
- `init` builds the data of a new block, and of a block the user resets. It does not change the stored data of existing blocks: add a migration to the data model for that.
- The model runs in a sandbox: 8 MiB memory, 10 seconds per call. Heavy work belongs in the workflow.

**Release**

- Never run `pnpm publish` in a block: its `prepublishOnly` publishes to MiLaboratories' registry. Publish with `block-tools publish -r <your s3 url> --registry-serve-url <your registry url>`, usually from CI.
- `pnpm build:dev-remote` and `pnpm build:release` push images and archives to MiLaboratories' registries unless these variables point elsewhere: `PL_DEV_DOCKER_PUSH_URL` (dev images), `PL_RELEASE_DOCKER_PUSH_URL` and `PL_RELEASE_DOCKER_PULL_URL` (release images), `PL_DEV_BINARY_UPLOAD_URL` / `PL_RELEASE_BINARY_UPLOAD_URL` (archives, S3 only). Run `docker login` to the registry first. Do not run these builds unless the developer asks.
- Publishing reads the built files, not the sources: build the whole block before you publish. Users see a registry after they add its URL in the Desktop App (Settings, Additional Registries).
- A registry identifies a block as `<organization>/<name>`, from the package name `@<scope>/<organization>.<name>`. The npm scope is dropped.
- `pnpm upgrade-sdk` rewrites `.github/workflows/build.yaml` and `mark-stable.yaml` with MiLaboratories' CI. After an upgrade, check `git diff .github/`.
- Old blocks (`BlockModel.create`, `defineApp`) use APIs removed in SDK 1.83. Moving one to the current SDK needs a move to `BlockModelV3` and `defineAppV3` first. After `pnpm upgrade-sdk`, a block made before the kind usually needs four fixes: delete an empty test file, fill the kind, create the data model and the block with the kind, add `.templateParams(...)`.

**A dev block on a remote Kubernetes backend**

- Load it in the Desktop App: Settings, Development Block Folders. After a rebuild, the block shows **Update Dev Block**.
- Workflow and model work. The UI works only on the developer's machine. Software works only as a docker image from a registry the backend trusts.

## References

Read one when you need it:

| File | For |
|---|---|
| `references/failure-triage.md` | an error message or a block that fails |
| `references/local-backend.md` | a backend on the laptop for block tests |
| `references/reference-blocks.md` | which blocks to read for common tasks; the columns that producers export |
| `references/block-description.md` | the short description and `docs/description.md`: what to write, in which order, and an example |

For everything else, read the real blocks in `blocks/` and the SDK in `core/platforma/`.

When this file or a reference is wrong, tell the developer which line, so it can be fixed.
