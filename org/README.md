# Your Organization's Layer

This folder is empty in the public repo. It is the place for what is true only in your organization: who publishes the blocks, with which logo and support contact, where your block repos live, which package scope you publish under, where your registry is, and how your CI publishes.

A team usually keeps a private copy of platforma-ctx (a fork or a mirror in your own Git host) and commits these files here:

| File | What it holds | Who reads it |
|---|---|---|
| `org/ORG.md` | Your conventions: organization name and URL, support contact, package scope, repo naming, registry URL, CI, review rules, backend environments | AI assistants read it before any block work (`pl-ctx setup` copies it to the top of the workspace as `ORG.md`); people read it on their first day |
| `org/organization-logo.png` | Your organization's logo, a PNG: a wide wordmark or a square; the Desktop App draws it at most 196 x 64 pixels | copied into every new block as `block/logos/organization-logo.png` |
| `org/block-logo.png` | Optional: the logo for a new block until it has its own, a square PNG (1024 x 1024) | copied into a new block as `block/logos/block-logo.png` |
| `org/repos.txt` | Your block repos, one per line: `<git url> <target dir>` | `pl-ctx setup` clones them |
| `org/verticals.txt` | The default set of public blocks to clone: `vdj`, `rnaseq` or `all` | `pl-ctx setup` |

Without these files, a new block keeps the placeholders that `block-tools structure init` writes: the default logos, an organization named after the `--org-scope` word, the URL `https://example.com`, and no support contact. The Desktop App shows them to every user of the block.

## Set It Up Once

1. Make a private copy of platforma-ctx in your Git host.
2. Copy `org/ORG.example.md` to `org/ORG.md` and fill it in. The section "Block Metadata" holds the values that every block shows to its users.
3. Add `org/organization-logo.png`, and `org/block-logo.png` if you want a default block logo.
4. List your block repos in `org/repos.txt` (format: `repos/local.example.txt`), and put your vertical in `org/verticals.txt`.
5. Commit and push. Everyone then clones the private copy instead of the public repo. To get updates of platforma-ctx, merge the public repo into the private copy now and then.

`org/ORG.md` wins over the generic guides in platforma-ctx when the two disagree. For example, if the generic guide says "publish with `pnpm build:release`" and your `ORG.md` says "never publish from a laptop, the CI does it", the assistant follows your `ORG.md`.

Do not put secrets here: no tokens, passwords or keys. Put only names, URLs and logos.
