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

## Where Blocks Run

Users run blocks on a remote backend: Platforma on Kubernetes in a cloud. Build for it from the start. A block that works only with a backend on the laptop is not finished.

- Do table work in the workflow with the SDK's table engine `pt` (`import("@platforma-sdk/workflow-tengo:pt")`): filter, join, group, string length (`strLenChars`), regex (`strContains`), any of several literal strings (`strContainsAny`). It runs on every backend, with nothing to build. Example: `blocks/cdr3-spectratype/workflow/src/main.tpl.tengo`.
- Write custom software only for what `pt` cannot do. Kubernetes runs software only as a docker image, and an image is made automatically only for `python` and `conda` software. A laptop build leaves the software on the laptop: after `pnpm test` or the watch command (binary only) the step fails on Kubernetes with "docker image is not specified"; after `pnpm build:dev-local` the image was built on the laptop but never pushed, so the cluster cannot pull it.
- To try software on a remote backend, push dev images to a registry that the cluster can pull from (see `ORG.md`): `docker login <registry>`, then `env PL_BUILD_CHANNEL=dev PL_BUILD_VARIANT=docker PL_BUILD_LOCATION=remote PL_DEV_DOCKER_PUSH_URL=<registry>/<repo> pnpm exec turbo run build`. Add `PL_DEV_DOCKER_PULL_URL` when the cluster pulls from another address. For an AWS ECR registry, write the push URL as `ecr://<host>/<repo>`: the build then logs in with the developer's AWS credentials. Do not use `pnpm build:dev-remote`: it also builds and uploads binaries, and by default it pushes to MiLaboratories' registries.
- The backend pulls only from trusted registries: the Platforma registry, its rewrite targets, and the hosts in the backend setting `PL_TRUSTED_DOCKER_REGISTRY`. An image from any other host fails with a permanent error, so the backend's admins must add the team's registry.
- To see what the cluster will run, open `software/dist/tengo/software/<entrypoint>.sw.json`. A `local` entry with a path is a laptop build. `docker.tag` is the image address that the cluster pulls.

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
- the resources (CPU, memory, GPU) of every software step, with a formula when they grow with the input, and its docker image;
- what users read about it: `block.meta` without placeholders, and a description written by `references/block-description.md`.

Before you call it finished, try it three ways: on the remote backend; with a bulk and a single-cell dataset; and while the upstream block is still computing (the options must already be there).

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

**Inputs**

- The input is a dataset, picked by its anchor. The dropdown lists anchor columns only (`pl7.app/isAnchor: "true"`): one option per dataset, for bulk and single-cell data alike, never the dataset's CDR3 or other columns. Copy the check from `blocks/cdr3-spectratype/model/src/dataset.ts` (bulk `clonotypeKey`; single-cell `scClonotypeKey`; imported sets `variantKey`, but only with the domain key `pl7.app/vdj/clonotypingRunId`, because other `variantKey` sets are peptides or other variants). Build the options in the model with `ctx.resultPool.getOptions(...)`, and leave out the block's own exports.
- Every other column is found relative to the anchor, never by block id. In the workflow: `wf.createPBundleBuilder()`, `addAnchor("main", args.datasetRef)`, then `addSingle` or `addMulti` with `axes: [{ anchor: "main", idx: 1 }]` (`blocks/cdr3-spectratype/workflow/src/main.tpl.tengo`, `wf.prepare`). In the model: `ctx.resultPool.getAnchoredPColumns({ main: ref }, [...])`.
- When a dataset has several matching columns (heavy and light chain, primary and secondary), the block picks them itself, or offers a second choice inside that dataset, such as the chain. Read the chain from the column specs (`pl7.app/vdj/scClonotypeChain` and `pl7.app/vdj/scClonotypeChain/index` in the domain), so the choice is there before the data. `isPairedDataset` in the same `dataset.ts` waits for data: do not copy it unchanged.

**Early results**

- Options must appear while upstream blocks still compute. `getOptions` reads specs only. `getAnchoredPColumns` returns `undefined` until every matching column has its data; with `{ dontWaitAllData: true }` it returns only the columns that already have data (see the comment in `blocks/clonotype-convergence/model/src/facts.ts`). To list columns while upstream still computes, use `getOptions`.
- Declare an output that the UI waits for with `.outputWithStatus(...)`. The UI then gets `{ ok: true, value, stable }` or `{ ok: false, errors, moreErrors }`. `stable: false` means it is still computing: show it as loading, not as empty. `ok: false` means it failed: show the error.
- `wf.body` runs only when everything that `wf.prepare` returns is ready, data included, and the block's exports appear only after that. Put into the bundle only the columns whose data the body reads.

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

- This is the normal way to try a block: the Desktop App connected to the backend where users work, usually the organization's DEV backend.
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
