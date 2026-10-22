# Getting Started

This guide takes you through one full loop: start or open a block, build it, load it into the Desktop App, change it, see the change, and test it. Every command here was run on 2026-10-21 against `@platforma-sdk/block-tools` 2.16.16 and `@platforma-sdk/model` 1.85.2, on macOS (Apple Silicon). Windows and Linux were not tested.

If your organization has an `org/ORG.md`, read it first. It names your package scope, your repos and your registry.

## 1. What You Need

| Tool | Version | Why |
|---|---|---|
| Node.js | 22.19 or newer | the SDK packages require it |
| pnpm | through corepack (`corepack enable`); any 9.7+ works | each block pins its own pnpm version, and pnpm switches to it |
| git | any recent | |
| Docker | only for docker images | needed when a block has Python, conda or docker software and you build its image (images are built for `linux/amd64`, also on Apple Silicon) |
| Desktop App | a recent release | blocks built with a newer SDK than the app supports do not load |
| A Platforma license (`~/.pl.license`, or the `PL_LICENSE` variable) | only for a local backend | block tests need a backend you can log in to with a password, usually one on your laptop (section 6) |

Run `./platforma-ctx/pl-ctx doctor` in the workspace folder. It checks the tools and the hosts a build needs: the npm registry you use (it must serve the `@platforma-sdk` packages), `github.com`, `cdn.platforma.bio` (`pnpm install` of a block downloads a native module from this host) and `blocks.pl-open.science` (the public block registry).

## 2. Open An Existing Block, Or Start A New One

**An existing block** (yours or a reference block):

```bash
cd blocks/<name>
pnpm install
```

**A new block.** Use the SDK's scaffolding tool. It writes the current layout, with all four parts:

```bash
cd blocks
pnpm dlx @platforma-sdk/block-tools@latest structure init ./my-block \
  --npm-org @my-org --org-scope myorg --short-name my-block \
  --no-software --non-interactive
cd my-block
pnpm install
```

- Use `pnpm dlx`, not `npx`. With `npx` it fails with `ENOENT ... templates/static/root/.gitignore`.
- `--with-software --platform python` instead of `--no-software` adds a Python software package. Start without software if you can: the SDK's table engine `pt` does most table work in the workflow (filter, join, group, string length, regex), and a block without software runs on every backend. On a Kubernetes backend, software runs only as a docker image ([test-and-release.md](test-and-release.md), section 1).
- Use your organization's npm scope and a short organization word (`--org-scope`). The block's id in a registry is `<org-scope>/<short-name>`; the npm scope is not part of it. Do not use `platforma-open` or `milaboratories`.
- Do not start from the `platforma-block-boilerplate` repo or from `pl-dev create-block`. Both give an old layout without the kind.

A new block does not compile until you fill in its kind. Open `kind/src/index.ts` and replace `NEEDS_BLOCK_PARAMS`. A block that is created without parameters uses:

```ts
export type BlockParams = Record<string, never>;
```

The block's parts:

| Folder | Part | Runs in |
|---|---|---|
| `kind/` | identity and the parameters the block can be created with | build time and the Desktop App |
| `model/` | turns settings into workflow arguments, and results into UI data | the Desktop App |
| `ui/` | what the user sees | the Desktop App |
| `workflow/` | the pipeline, in Tengo | the backend |
| `software/` | tools the workflow runs (optional) | jobs on the backend |
| `block/` | the package that bundles the parts | |
| `test/` | block tests against a backend | your laptop |

The build tool owns the `package.json` files, `turbo.json`, tsconfig, lint and CI files. Do not edit them by hand: `pnpm exec block-tools structure check` shows what it would change, and `pnpm exec block-tools structure refresh` applies it.

## 3. Build

There is no `pnpm build`. Use one of the named scripts:

| Command | What it does | Docker? |
|---|---|---|
| `pnpm build:dev-local` | development build of everything, including docker images of the software | yes, if the block has Python, conda or docker software |
| `env PL_BUILD_CHANNEL=dev PL_BUILD_VARIANT=binary PL_BUILD_LOCATION=local pnpm exec turbo run build` | development build without docker images | no |
| `pnpm build:dev-no-software` | builds the block with placeholder software; the block loads, but software steps fail | no |
| `pnpm test` | builds like the second line (software for local use, no images), runs format and lint checks, then the test scripts of each package | no |
| `pnpm fmt` | formats the code | no |

`pnpm build:dev-remote` and `pnpm build:release` push images and archives to registries. By default they point at MiLaboratories' registries. Do not run them on your laptop; see [test-and-release.md](test-and-release.md).

## 4. Load The Block Into The Desktop App

Try the block on the backend where its users work: usually your organization's DEV backend, which runs on Kubernetes in the cloud. Connect the Desktop App to it.

1. In the Desktop App, open **Settings** and find **Development Block Folders**.
2. Add the folder of your block repo, for example `blocks/my-block` in the workspace folder. One folder per line.
3. Open a project and add a block. Your block appears with a `dev` tag.

The app reads the built files from your disk, so build before you add the block. It sends the workflow and the model to the backend; the UI runs on your machine. A block without its own software runs on the remote backend straight from this build. A block with software needs docker images in a registry that the cluster can pull from: [test-and-release.md](test-and-release.md), section 1.

## 5. Change, Rebuild, See

Rebuild on every change with the development settings:

```bash
env PL_BUILD_CHANNEL=dev PL_BUILD_VARIANT=binary PL_BUILD_LOCATION=local pnpm watch
```

Plain `pnpm watch` builds the software for release, and runs then fail with "not found in any of endpoints". Use the line above.

The app checks your block's workflow and model files every 1.5 seconds or so. When they change, the block card shows **Update Dev Block**. Click it to load the new version. The app does not look at the UI files, so a change only in the UI does not show the button. Rebuild with `--force` (for example `pnpm build:dev-local --force`), which rewrites the model file too.

Things to try on your first loop:

- Change a label in `ui/src/` and see it in the app.
- Change a default in `model/src/` and see it in a new block.
- Change the workflow in `workflow/src/main.tpl.tengo`, run the block, and see the new result.
- Make the workflow fail on purpose (for example `ll.panic("test")`) and find the error in the block's log in the app.

## 6. Test

| Layer | Where | Needs a backend? | Command |
|---|---|---|---|
| Workflow logic | `workflow/src/*.test.tengo` | no | part of `pnpm test` (`pl-tengo test`) |
| Model and UI logic | `model/src/*.test.ts`, `ui/src/*.test.ts` | no | part of `pnpm test` (vitest) |
| Python software | `software/tests/` | no | `pytest` in the software folder (not part of `pnpm test`) |
| The whole block | `test/src/*.test.ts` | yes | `pnpm test` with `PL_ADDRESS` set |

`pnpm test` runs format and lint checks first. If it fails on formatting, run `pnpm fmt`. Turbo caches test results: to run the tests again with nothing changed (for example against another backend), use `pnpm test -- --force` (`pnpm test --force` fails with "Unknown option").

A test file is part of `pnpm test` only when its package has a test script. The build tool adds that script when it sees test files: after you add the first `*.test.tengo` or `*.test.ts` to a package, run `pnpm exec block-tools structure refresh`, then `pnpm install`. Python tests are not part of `pnpm test`; run `pytest` in the software folder.

Block tests log in with a user name and password (or with nothing, if the backend has no login). A backend that allows only single sign-on cannot run them, so run a backend on your laptop. You need a license. The full recipe is in `.claude/skills/platforma-block-dev/references/local-backend.md`. In short:

```bash
export PL_ADDRESS=http://127.0.0.1:6345 PL_TEST_USER=default PL_TEST_PASSWORD=default
pnpm test
```

A new block's `test/` package is not connected to the block yet. Before you write a block test, add the block as a dependency of the test package; see `blocks/redefine-clonotypes/test/package.json` for the pattern.

## 7. Next

- Testing in your organization's environment, releasing, SDK upgrades: [test-and-release.md](test-and-release.md).
- How blocks work inside: `.claude/skills/platforma-block-dev/SKILL.md`. It is written for AI assistants, but people can read it too. Then the real blocks in `blocks/`.
- Real blocks to learn from: `.claude/skills/platforma-block-dev/references/reference-blocks.md`.
