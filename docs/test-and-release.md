# Test In Your Environment, Release, Upgrade

This guide is for the steps after the first loop in [getting-started.md](getting-started.md): trying a block on your organization's backend, publishing it to your own registry, upgrading the SDK, and keeping forks of public blocks up to date. Checked on 2026-10-21 against `@platforma-sdk/block-tools` 2.16.16. Your organization's actual hosts, registries and CI are in `org/ORG.md`.

## 1. A Dev Block On Your Organization's Backend

The Desktop App can load a dev block while it is connected to a remote backend. This section is about backends on Kubernetes, which is how organizations usually run Platforma. What works:

| Part | Works on a remote backend? |
|---|---|
| Workflow and model | yes: the app sends them from your laptop to the backend |
| UI | yes, but only on your machine. A colleague who opens the same project does not have your folder. Do not share projects that contain dev blocks. |
| Software | on Kubernetes, only as a docker image that the cluster can pull |

A local development build (`build:dev-local`, `pnpm test`, the watch command) leaves the software on your laptop: a binary path, or a docker image that was built but not pushed. The cluster can use neither, so software steps fail there. A Kubernetes backend runs software only from images. So, for a block with software:

1. Log in to your image registry: `docker login <your-registry-host>`. For an AWS ECR registry, skip this and write the push URL below as `ecr://<host>/<repo>`: the build then logs in with your AWS credentials.
2. Build and push dev images to it:
   ```bash
   env PL_BUILD_CHANNEL=dev PL_BUILD_VARIANT=docker PL_BUILD_LOCATION=remote \
       PL_DEV_DOCKER_PUSH_URL=<your-registry-host>/<repo> \
       pnpm exec turbo run build
   ```
3. The backend must trust your registry host: the admins add it to the backend setting `PL_TRUSTED_DOCKER_REGISTRY`. An image from a host that is not trusted fails with a permanent error. The cluster's nodes must also be able to pull from it. Both are set up by whoever runs your backend. Ask them before the first try. If the cluster pulls from another address than the one you push to, also set `PL_DEV_DOCKER_PULL_URL`.

To check what the cluster will run, open `software/dist/tengo/software/<entrypoint>.sw.json` in the software package. `docker.tag` is the image address that the cluster pulls. A `local` entry with a path on your laptop means a laptop build, which cannot run on the cluster.

Do not use `pnpm build:dev-remote` for this. It builds binaries as well and uploads them, and without these variables it pushes to MiLaboratories' registries.

A block without software (only Tengo and the SDK's table engine) needs none of this. It runs on a remote backend directly from your dev build.

Block tests (`test/`) log in with a user name and password, or with nothing if the backend has no login. If your organization's backends allow only single sign-on, run block tests against a backend on your laptop instead (see getting-started, section 6).

## 2. Release

In most organizations only CI publishes. If yours does, the steps for a developer are:

1. Make the change on a branch.
2. Add a changeset: `pnpm changeset`. Pick the packages you changed and the kind of version bump. Every change to a block needs one.
3. Open a pull request. After review and merge, CI builds the release and publishes it.

What the CI (or a release manager) runs, for reference:

- A release build first. A local development build carries paths from a laptop, and the publish command does not refuse it.
- Images for the software, pushed to your registry: `PL_BUILD_CHANNEL=release PL_BUILD_VARIANT=docker PL_BUILD_LOCATION=remote` with `PL_RELEASE_DOCKER_PUSH_URL` and `PL_RELEASE_DOCKER_PULL_URL` set to your registry.
- The block, published to your block registry:
  ```bash
  cd block
  pnpm exec block-tools publish \
    -r 's3://<bucket>/<prefix>/?region=<region>' \
    --registry-serve-url 'https://<your-registry-host>/'
  ```
  `-r` accepts `s3://`, `file://` or a plain folder path. For other storage, publish to a folder and copy it with your own tool. By default the publish also marks the version as stable and refreshes the registry's overview; `--unstable` and `--no-refresh` turn that off. It publishes the kind first and stops if the model was built against a different kind version.

**Never run `pnpm publish` in a block.** The block package's `prepublishOnly` script publishes to MiLaboratories' public registry by default, and the build tool writes it back on every refresh.

Users see your registry when they add its URL in the Desktop App: **Settings**, **Additional Registries**. Settings are per user.

A block's id in a registry is `<organization>/<name>`, taken from the package name `@scope/<organization>.<name>`. The npm scope is dropped. If your block has the same organization and name as a public block, users see two blocks with the same id from two registries. Use your own organization word.

## 3. Upgrade The SDK

```bash
pnpm upgrade-sdk
```

It updates the SDK versions, re-installs, and refreshes all files the build tool owns. Then build and test. Blocks made before the kind became mandatory need up to four manual fixes:

| Error | Fix |
|---|---|
| `unicorn(no-empty-file)` on a test file | delete the empty test file |
| `Cannot find name 'NEEDS_BLOCK_PARAMS'` | fill `BlockParams` in `kind/src/index.ts` |
| `Type 'DataModel<...>' is missing ... dataModel, kind` | create the data model and the block with the kind: `new DataModelBuilder({ kind })`, `BlockModelV3.create({ dataModel, kind })` |
| `templateParams() not set` | add `.templateParams(...)` to the model; `.templateParams(() => ({}))` for a block without parameters |

How to tell an old block: `BlockModel.create(` in `model/src/` and `defineApp(` in `ui/src/` is the old model API. The SDK removed it in `@platforma-sdk/model` 1.83.0. Such a block must be moved to `BlockModelV3` and `defineAppV3` before the upgrade can build. Ask an AI assistant in platforma-ctx to plan that move; it is a larger change.

**After every upgrade, check your CI files.** The build tool rewrites `.github/workflows/build.yaml` and `mark-stable.yaml` with MiLaboratories' CI on every refresh. These generated workflows stay active: they run on pull requests and pushes and ask for MiLaboratories' runners and secrets, so in your repos they fail or wait. If your organization has its own CI:

- keep it in other workflow files, so a refresh does not overwrite it;
- switch the two generated workflows off in GitHub (the Actions tab, the workflow, "Disable workflow"), or delete them again after each refresh;
- check `git diff .github/` after every `pnpm upgrade-sdk`.

## 4. Keep A Fork Of A Public Block Up To Date

If your block started as a copy of a public block, the public original keeps changing: new SDK versions, fixes, new features.

```bash
git remote add upstream https://github.com/platforma-open/<repo>.git
git fetch upstream
git merge upstream/main
```

- Merge often, for example monthly and before every SDK upgrade. Small merges are easy; large ones are not.
- Keep your own changes small and in few files.
- A change that is useful to everyone can go back to the public original as a pull request. Then your fork no longer has to carry it.
