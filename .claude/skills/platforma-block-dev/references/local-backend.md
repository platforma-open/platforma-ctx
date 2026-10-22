---
stability: volatile
verified: 2026-10-21
sdk: "@platforma-sdk/model 1.85.2, workflow-tengo 6.16.0, block-tools 2.16.16"
---

# A Local Backend For Tests

Template tests (`tplTest`) and block tests (`blockTest`) need a running Platforma backend. Unit tests of Tengo, TypeScript or Python code do not. This file shows how to run a backend on your own machine and point the tests at it.

## Why A Local Backend

- **The tests log in with a user name and a password** (or with nothing, on a backend without login: then set no user and no password). `core/platforma/lib/node/pl-client/src/test/test_config.ts` calls the password login. A backend that offers only SSO cannot run them. A shared backend works only if it also has a password login (htpasswd or LDAP) and you have a test user there.
- **A dev build points at folders on your machine.** The software descriptor of a dev-local build holds an absolute path. Only a backend on the same machine can run that software.
- **Every backend needs a Platforma license.** Get one at `https://platforma.bio/getlicense`. The backend reads `~/.pl.license` when you give neither `--license` nor `--license-file` (`platforma --help`).

## Get The Backend Binary

The backend is a public download. The SDK builds the URL like this (`core/platforma/tools/pl-bootstrap/src/platforma.ts`):

```
https://cdn.platforma.bio/software/pl/<os>/pl-<version>-<arch>.tgz
   <os>:   macos | linux | windows
   <arch>: arm64 | amd64
```

This is the formula of the `pl-bootstrap` tool. Other SDK code downloads a `.zip` for Windows (not verified).

The same files are on `https://cdn-ga.pl-open.science/software/pl/...`; the SDK tries that host when the first one fails (`core/platforma/lib/node/pl-deployments/src/common/pl_binary_download.ts`). Use the backend version your organization runs (see `ORG.md` at the top of the workspace); tests against another version can pass locally and fail there.

The archive unpacks to `binaries/platforma` plus `binaries/libs/`. For example, on a Mac with Apple Silicon:

```bash
mkdir -p ~/pl-<version>
curl -fSL https://cdn.platforma.bio/software/pl/macos/pl-<version>-arm64.tgz | tar -xz -C ~/pl-<version>   # (not verified)
~/pl-<version>/binaries/platforma --version
```

The `curl | tar` line was not run. The `binaries/platforma` layout was seen in a downloaded 3.0.3 archive and in an unpacked 4.2.14 backend.

`platforma --debug-help=-` prints the backend's own guide for block developers: work folders, block logs, log levels and local package registries.

## Run It

```bash
mkdir -p ~/pl-dev
printf 'default:default\nadmin:admin\n' > ~/pl-dev/users.htpasswd
# 32 random bytes, base64url. Create it once and keep it.
openssl rand -base64 32 | tr '+/' '-_' | tr -d '=' > ~/pl-dev/master-secret

~/pl-<version>/binaries/platforma \
  --main-root ~/pl-dev/data \
  --auth-htpasswd ~/pl-dev/users.htpasswd \
  --master-secret "$(cat ~/pl-dev/master-secret)" \
  --admin-user admin \
  --license-file ~/.pl.license
```

| Flag | What it does (from `platforma --help`, 4.2.14) |
|---|---|
| `--main-root` | folder for all local data: database, downloaded software, work folders |
| `--auth-htpasswd` | users and passwords from an htpasswd file; the `htpasswd` tool can write it |
| `--master-secret` | secret for signing tokens and resources, base64url, at least 32 bytes decoded. Required with htpasswd or LDAP login. `--master-secret-file` reads it from a file |
| `--admin-user` | gives this login the admin role |
| `--license-file`, `--license` | the license; default file `~/.pl.license` |
| `--listen-address`, `--listen-port` | default `127.0.0.1` and `6345` |
| `--package-registry <name>=<url>` | where to find software of a registry name; `dir://<path>` or `http(s)://...`; repeatable |

Checked on 2026-10-21: backend 4.2.14 started with these flags (on another port, with the license in `~/.pl.license`) accepted the plain `user:password` lines above, and the test of a block made by `block-tools structure init` passed against it.

Keep the same master secret between restarts. The test client caches its login token in `.test_auth.json`, in the folder where the tests run. After you change the secret or the users, delete that file.

Python software makes the backend download a Python run environment the first time, from `bin.pl-open.science`. The macOS arm64 archive of `runenv-python-3.12.10` is about 0.9 GB.

## Point The Tests At It

```bash
export PL_ADDRESS=http://127.0.0.1:6345
export PL_TEST_USER=default
export PL_TEST_PASSWORD=default
pnpm test            # in the block root
```

How a test finds the backend (`core/platforma/lib/node/pl-client/src/test/test_config.ts`):

1. `test_config.json` in the folder where the test process runs: `test/` for block tests, `workflow/` for template tests. Fields: `address`, `test_user`, `test_password`, `test_proxy`, `test_admin_user`, `test_admin_password`. It holds a password, so keep it out of git.
2. These variables override the file: `PL_ADDRESS`, `PL_TEST_USER`, `PL_TEST_PASSWORD`, `PL_TEST_PROXY`, `PL_TEST_ADMIN_USER`, `PL_TEST_ADMIN_PASSWORD`.

| Situation | Result |
|---|---|
| no address anywhere | `can't resolve platform address (checked test_config.json file and PL_ADDRESS environment var)` |
| backend needs login, no user or password set | `No auth information found in config ...` |
| backend has no login, but a user or password is set | `Server require no auth, but test user name or test password are provided ...` |

**turbo passes only five variables to tests:** `PL_ADDRESS`, `PL_TEST_USER`, `PL_TEST_PASSWORD`, `PL_TEST_PROXY` and `DEBUG` (the `test` task in `turbo.json`). Others, such as `PL_TEST_REQUEST_TIMEOUT` or the admin variables, do not arrive when you run `pnpm test`. Put the admin user and password in `test_config.json`. `test_config.json` has no timeout field: put the timeout into the address, for example `PL_ADDRESS=http://127.0.0.1:6345?request-timeout=5000`.

`PL_ADDRESS` forms (`core/platforma/lib/node/pl-client/src/core/config.ts`):

| Form | Meaning |
|---|---|
| `127.0.0.1:6345` | plain connection, no TLS |
| `http://host:port` | plain connection |
| `https://host` | TLS; port 443 unless given |
| `grpc://host:port`, `tls://host:port` | the port is required |

An address with a scheme can carry options as a query, for example `https://pl.example.com?request-timeout=5000`. The test client's request timeout is 500 ms. That is enough on one machine and often too short for a remote backend; raise it with `request-timeout`, which reaches the tests through `PL_ADDRESS`.

## `pl-dev`

`pl-dev` is a public command-line tool in `@platforma-sdk/bootstrap` (`pnpm dlx --package=@platforma-sdk/bootstrap pl-dev --help`). It can download and start a backend. Its traps:

- Its default backend version is 3.5.0 (`"pl-version"` in `core/platforma/lib/node/pl-deployments/package.json`). Always pass `--version <version>`.
- `pl-dev start local s3` and `pl-dev start docker` need Docker: the S3 variant starts MinIO with docker compose (`core/platforma/tools/pl-bootstrap/src/core.ts`).
- Starting a 4.x backend with `pl-dev start local` was not verified. Its source has no master-secret handling, and 4.x requires a master secret with htpasswd login.
- `pl-dev create-block` downloads the old block boilerplate. Start new blocks with `block-tools structure init`.

The direct binary command above is the verified way.
