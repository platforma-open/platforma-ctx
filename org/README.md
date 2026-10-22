# Your Organization's Layer

This folder is empty in the public repo. It is the place for what is true only in your organization: where your block repos live, which package scope you publish under, where your registry is, and how your CI publishes.

A team usually keeps a private copy of platforma-ctx (a fork or a mirror in your own Git host) and commits these files here:

| File | What it holds | Who reads it |
|---|---|---|
| `org/ORG.md` | Your conventions: package scope, repo naming, registry URL, CI, review rules, backend environments | AI assistants read it before any block work (`pl-ctx setup` copies it to the top of the workspace as `ORG.md`); people read it on their first day |
| `org/repos.txt` | Your block repos, one per line: `<git url> <target dir>` | `pl-ctx setup` clones them |
| `org/verticals.txt` | The default set of public blocks to clone: `vdj`, `rnaseq` or `all` | `pl-ctx setup` |

Start from `org/ORG.example.md`.

`org/ORG.md` wins over the generic guides in platforma-ctx when the two disagree. For example, if the generic guide says "publish with `pnpm build:release`" and your `ORG.md` says "never publish from a laptop, the CI does it", the assistant follows your `ORG.md`.

Do not put secrets here: no tokens, passwords or keys. Put only names and URLs.
