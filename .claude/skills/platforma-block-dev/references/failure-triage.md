---
stability: volatile
verified: 2026-10-21
sdk: "@platforma-sdk/model 1.85.2, workflow-tengo 6.16.0, block-tools 2.16.16"
---

# Failure Triage

Open this file when a block fails. Each entry gives the symptom as it appears, the cause and the fix. Messages that come from the backend are quoted as seen in real runs.

## How To Read A Backend Error

A backend failure arrives as nested frames, one per resource on the path:

```
PlInternalError: [O] "NG:0x…/outputs/<output>": has input errors:
[I] "NG:0x…/resource": has input errors:
…more frames…
package "software/<scope>/<package>/main/1.0.0.tgz" not found in any of endpoints
```

**Start from the last line; it usually holds the diagnosis.** An error can list several causes, so read each `has input errors:` part too. The frames above it are the path. Scroll to the bottom first, every time. Look for the line after the last `has input errors:`, a `Command:` line (the exact command that ran) and a `Docker image:` line.

## Setup And Build

| Symptom | Cause | Fix |
|---|---|---|
| `ENOENT: no such file or directory, open '.../templates/static/root/.gitignore'` from `npx @platforma-sdk/block-tools structure init` | npm renames `.gitignore` inside the installed package | run it with `pnpm dlx @platforma-sdk/block-tools@<version> structure init ...` |
| `pnpm install` fails while installing `@milaboratories/pframes-rs-node` (exact message not verified) | its install script downloads a native binary from `cdn.platforma.bio` and has no fallback | allow `cdn.platforma.bio` in the proxy or firewall |
| `ERR_PNPM_RECURSIVE_EXEC_FIRST_FAIL Command failed with EACCES: build` / `spawn build EACCES` | there is no `pnpm build` in a current block | use `pnpm build:dev-local` or another named script |
| `TS2304: Cannot find name 'NEEDS_BLOCK_PARAMS'` | a new or upgraded block's kind is not filled in | set `BlockParams` in `kind/src/index.ts`; `Record<string, never>` for no parameters (see `blocks/vj-gene-usage/kind/src/index.ts`) |
| `TS2739: Type 'DataModel<...>' is missing ... dataModel, kind` | the model uses the call from before kinds | `new DataModelBuilder({ kind })`, `BlockModelV3.create({ dataModel, kind })` |
| `templateParams() not set. Every block must project its state back to its kind's params ...` | the model has no `.templateParams(...)` | add it (see `blocks/vj-gene-usage/model/src/index.ts`) |
| `unicorn(no-empty-file)` | an empty or comment-only file, often an old `test/src/wf.test.ts` | delete it or write a real test |
| `Format issues found in above 1 files. Run without --check to fix.` from `check` | `build` and `test` run `check` first, and `check` includes the format check | `pnpm fmt` |
| `... docker.sock ... driver not connecting` during `pnpm build:dev-local` | Python or conda software builds a docker image, and Docker is not running | start Docker, or build without it: `env PL_BUILD_CHANNEL=dev PL_BUILD_VARIANT=binary PL_BUILD_LOCATION=local pnpm exec turbo run build` |
| `Unsatisfied dependencies in template:<workflow>:<template>:<version>: - software:<name>` | the name in `assets.importSoftware("<name>")` is not a software package and entrypoint the workflow depends on | fix the literal; the package must be a workflow dependency |
| `'registry.storageURL' of package '...' is empty. Set it as command option, in '...' file or via environment variable 'PL_REGISTRY_<NAME>_UPLOAD_URL'` | a remote or release build has no upload URL | set `PL_RELEASE_BINARY_UPLOAD_URL` or `registry.storageURL`; through turbo, `PL_REGISTRY_<NAME>_UPLOAD_URL` arrives only for `platforma-open` |
| `Protocol ... is not supported for software registries yet. Use your own tooling for package upload` | binary upload URL is not `s3://` or `s3e://` | use S3, or upload with your own tool |
| `automatic docker login is only supported for 'public.ecr.aws', not '<host>'` | an `ecr://` push URL on another host | drop the `ecr://` prefix and run `docker login <host>` yourself |
| `These software entrypoints produce no docker image, so no kubernetes installation can run them` (CI only) | a `java`, `R` or `binary` entrypoint without a `docker` block | add a `docker` artifact, or declare `"requireDocker": false` |

## Software Does Not Reach The Backend

### `package "…/<pkg>/<version>.tgz" not found in any of endpoints`

The descriptor points at a registry package that the backend cannot find. Usual causes:

- **The block was built without the dev variables.** Plain `pnpm watch`, or `turbo run build` without `PL_BUILD_*`, writes a release descriptor such as `"registry": "platforma-open", "package": "software/<scope>/<pkg>/main/1.0.0.tgz"`. Unless that exact release is published, the software cannot be found. Rebuild with `env PL_BUILD_CHANNEL=dev PL_BUILD_VARIANT=binary PL_BUILD_LOCATION=local ...` and press "Update Dev Block".
- The software package still builds with the old `pl-pkg build`, which ignores the `PL_BUILD_*` variables. The current build script is `block-tools software build`; run `pnpm exec block-tools structure refresh` to switch to it.
- A release was published, but the backend has no `--package-registry <name>=<url>` for the registry name in the descriptor.

### `registry "no-software-placeholder" is not found in packageLoader controller configuration`

The block was built with `pnpm build:dev-no-software`, which writes placeholder descriptors (`core/platforma/lib/node/package-builder-lib/src/defaults.ts`). Rebuild with software.

### `docker image is not specified`

A Kubernetes backend runs only images and has no fallback to a binary (`core/platforma/lib/node/package-builder-lib/src/core/docker-coverage.ts`). The software was built without an image, or as a dev-local folder. This also hits a dev block on a remote backend: its dev-local software is a path on your machine. Push an image the cluster can pull: `docker login <host>`, then `env PL_BUILD_CHANNEL=dev PL_BUILD_VARIANT=docker PL_BUILD_LOCATION=remote PL_DEV_DOCKER_PUSH_URL=<host>/<repo> pnpm exec turbo run build` (not run for this guide). The workflow and model of a dev block do reach a remote backend; its UI works only on your machine, so do not share projects with dev blocks.

### The image is refused at once

The backend pulls only from trusted hosts: the default Platforma registry, `--docker-rewrite` / `--default-docker-registry` targets, and `--trusted-docker-registry=<host>`. Other hosts "fail with a permanent error" (`platforma --help --show-hidden`, 4.2.14; the exact message was not verified). The backend owner must add your host.

### `pnpm build:dev-remote` is all green, but nothing changed

A green build means the build worked, not that the block uses it. Check that the new image tag is in the `docker.tag` field of `software/dist/tengo/software/<entrypoint>.sw.json`, and update the block in the project.

## Workflow

| Symptom | Cause and fix |
|---|---|
| `expected only results and exports in the returned map, found ...` | `wf.body` must return a map with exactly the keys `outputs` and `exports`. Add `exports: {}` even when empty (`core/platforma/sdk/workflow-tengo/src/workflow/index.lib.tengo`) |
| `file 'x' was not saved, add saveFile() call to the exec builder` | `getFile("x")` without `saveFile("x")`; the same pair exists for `getFileContent` / `saveFileContent` (`core/platforma/sdk/workflow-tengo/src/exec/index.lib.tengo`) |
| `type schema validation error: params: value "{…}" does not contain key "X"` | the value is `undefined`, which Tengo reads as a missing key. In an old project this is a `BlockData` field added after the project last ran: fix the stored data with a migration step |
| `unrecognized arguments: --x` | the Tengo call and the tool's argument parser drifted apart. Nothing checks across that line; change both in one commit |
| `not callable: undefined` (wording not verified) | the code calls a function that the imported module does not have; check the name |
| `lost blob "NG:0x…" data: no incarnations found in any of storages`, often after `max attempts for notification '…' reached` | data made by one step was gone before a later step used it. A step in between must keep a reference to it (cause not verified) |
| an `if` on a tool's result never takes the expected branch | a template body cannot see a tool's output; pass the output reference to a child template rendered with `render.create`; it runs when its inputs are ready (`blocks/mixcr-clonotyping/workflow/src/run-mixcr.tpl.tengo` passes `getFileContent(...)` to `run-steps.tpl.tengo`) |

## Model, UI And Tables

| Symptom | Cause and fix |
|---|---|
| `outputs.X` is `undefined` in the model | first check that the block ran and finished: an output is `undefined` until then. If it did, the workflow output key does not match `ctx.outputs?.resolve("X")`; match the strings exactly |
| `getDataAsJson()` returns nothing for a `getFile(...)` output | `getFile` / `saveFile` give a file; `getFileContent` / `saveFileContent` give inline content. Use the pair that matches how the model reads it |
| the UI works with one client and acts strangely with two | an output written back into the data that produced it |
| `Key not found ctl/file/blobInfo` from table creation | raised in `core/platforma/lib/node/pl-tree/src/snapshot.ts`. Seen when column discovery reached columns of the result pool whose data lacks that field; narrow what the table may discover (cause not verified) |
| `Invalid PTable handle` | the handle is not registered in the table pool (`core/platforma/lib/node/pf-driver/src/ptable_def_pool.ts`); cause not verified |
| `unable to find column "X"; valid columns: [...]` | looks like a Polars error from a table step or your own Python code: a column used in a select is missing from the file (not verified) |

## Tests

| Symptom | Cause and fix |
|---|---|
| `can't resolve platform address (checked test_config.json file and PL_ADDRESS environment var)` | no backend address; see [local-backend.md](local-backend.md) |
| `No auth information found in config ...` / `Server require no auth, but test user name or test password are provided ...` | `PL_TEST_USER` / `PL_TEST_PASSWORD` do not match the backend's login setting |
| `RpcError: Deadline exceeded after 0.500s` | the test request timeout is 500 ms; add `?request-timeout=5000` to `PL_ADDRESS` |
| `ModelAPIVersionMismatchError: Model API version mismatch for block <id>: 1 !== 2` | `setBlockArgs` on a current block; use `mutateBlockStorage(blockId, { operation: "update-block-data", value })` |
| `Unexpected block status, block not calculating anything at the moment: NotCalculated` | `awaitBlockDone` before `runBlock` |
| `Aborted while awaiting block done.` | the wait's timeout was too short; the default is 5000 ms |
| a variable such as `PL_TEST_REQUEST_TIMEOUT` has no effect under `pnpm test` | turbo passes only `PL_ADDRESS`, `PL_TEST_USER`, `PL_TEST_PASSWORD`, `PL_TEST_PROXY`, `DEBUG` |
| the test package cannot import `this-block` | the scaffold does not wire it; add the `workspace:` dependency (pattern: `blocks/redefine-clonotypes/test/package.json`) |
| every test passes, nothing is checked | the scaffolded `expect(true).toBe(true)` is still the only test |

## Not Your Block

- `The workload was stopped before it ever became ready to run … the pod was evicted or preempted` (wording not verified): infrastructure. Run again; if it stays, tell whoever runs the backend.
- `ImagePullBackOff`: the cluster could not pull the software image. First check that the image tag in the block's descriptor was really pushed to the registry (a build can point at an image that was never pushed), then that the cluster may pull from that registry.
- An archive for `macosx-aarch64` cannot run on a Linux server. If a Mac-built artifact reached a remote backend, the build path is wrong, not the code.

## Silent Failures: No Error At All

| Symptom | Cause |
|---|---|
| a pure template runs again instead of using the cache | map iteration without sorted keys, or `json.encode` on a map; its key order is random. Use a JSON literal, `canonical.encode(map)`, and `for k in maps.getKeys(m)` |
| a JSON cell arrives as base64 text, often ending in `=` | bytes from `json.encode` put into another `json.encode`; wrap the inner one in `string(...)` |
| rows missing after an xsv import | `allowNA: false` on a value column drops rows where that value is missing (on axes `allowNA` is ignored) (`core/platforma/sdk/workflow-tengo/src/pframes/xsv-import-pt.lib.tengo`) |
| a schema check passes but a key is ignored | `validation.assertType` accepts unknown keys unless the schema has `"__options__,closed"` |
| a selector matches other columns after a refactor | `matchPColumn` matches axes exactly by default; `matchMultiColumnSelector` matches partially (`core/platforma/lib/model/common/src/drivers/pframe/spec/selectors.ts`) |
| a domain-anchored query matches too much | `.ignoreMissingDomains()` on the bundle builder applies to all its queries, not one |
| batches are bigger than requested, or the run fails on `maxBatches` | in batch mode the SDK makes batches bigger to stay under `maxBatches`, and fails when the number of isolation scopes alone is above it: raise `maxBatches` |

## Before You Report A Failure

- **Run it again from a clean state.** A stale cache has made a green suite look red.
- **Check that the command really ran.** An empty search result proves nothing if the tool was missing and its error went to `/dev/null`.
- **Read to the last frame** before you form a theory.
