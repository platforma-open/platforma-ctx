# platforma-ctx

AI context for building [Platforma](https://platforma.bio) blocks.

platforma-ctx sets up a folder that an AI coding assistant works in: GitHub Copilot, Claude Code, or any agent that reads `AGENTS.md`. The folder holds the Platforma SDK source, all public Platforma blocks, your own blocks, and short directions. The assistant learns how blocks are written from real, working code, not from long instructions. You work in the same folder, with or without the assistant.

A block is one analysis task in Platforma: its settings, its pipeline, its results and their QC. Its UI and model run in the Desktop App, its workflow runs on the Platforma backend, and its software (Python, R, Java, docker images) runs in jobs that the workflow starts.

## What Is Inside

After setup, the workspace folder looks like this. The folder itself is not a git repo. platforma-ctx, the SDK and every block are separate git repos inside it, so VS Code shows each repo on its own, and search covers all of them.

| Path | What it is | Where it comes from |
|---|---|---|
| `platforma-ctx/` | this repo: the `pl-ctx` script, the repo lists, two guides for people, and the originals of the files below | `git clone` |
| `AGENTS.md` | short directions that assistants load in every session | copied from `platforma-ctx/` by `pl-ctx setup` |
| `.claude/skills/platforma-block-dev/` | the skill: how to work on a block, the commands, and the traps that the code does not show | copied |
| `.claude/settings.json` | Claude Code permissions: builds and tests run without asking, publishing is blocked | copied |
| `ORG.md` | your organization's conventions, if your copy of platforma-ctx has `org/ORG.md` | copied |
| `core/platforma/` | the Platforma SDK source: model, UI library, workflow library, build tools, test helpers | cloned by `pl-ctx setup` |
| `support/github-ci/` | the shared GitHub Actions workflows that the CI files of a block call | cloned by `pl-ctx setup` |
| `blocks/` | the public Platforma blocks (all 74, or those of the verticals you choose), and your own block repos | cloned by `pl-ctx setup` |

`pl-ctx setup` and `pl-ctx update` replace the copies every time, so change the originals in `platforma-ctx/`. Put personal Claude Code settings in `.claude/settings.local.json`; pl-ctx does not touch that file. Setup also writes `.pl-ctx-workspace`, which marks the folder as a workspace, and `.vscode/extensions.json` if it is missing.

## Setup

You need git, Node.js 22.19 or newer, and pnpm through corepack (`corepack enable`). Docker is needed only to build docker images of block software. Tested on macOS; on Windows, use Git Bash or WSL (not tested yet).

Make a new, empty folder for the workspace, and clone platforma-ctx into it:

```bash
mkdir platforma && cd platforma
git clone https://github.com/platforma-open/platforma-ctx.git
./platforma-ctx/pl-ctx doctor
./platforma-ctx/pl-ctx setup
```

`doctor` checks the tools and the network. `setup` clones the SDK and all public blocks into the workspace folder (last commit only, to save time; about 6 minutes the first time), then your own repos if you listed any. The first setup stops if the folder holds anything other than platforma-ctx, so it never writes into a home folder or a folder of other projects.

To clone only the blocks of one area, pick a vertical:

```bash
./platforma-ctx/pl-ctx setup --vertical vdj      # immune repertoire and antibody blocks, plus common blocks
./platforma-ctx/pl-ctx setup --vertical rnaseq   # RNA-seq blocks, plus common blocks
./platforma-ctx/pl-ctx setup --vertical all      # everything (the default)
```

Common blocks (Samples & Data, Table, Graph Maker, FASTQ tools, file import) always come. Your choice is remembered in `platforma-ctx/repos/verticals.txt`. A team can set its default in `org/verticals.txt` (one line, for example `vdj`).

Add one of your own block repos, and keep everything current:

```bash
./platforma-ctx/pl-ctx add git@github.com:my-org/my-block.git
./platforma-ctx/pl-ctx update
./platforma-ctx/pl-ctx status
```

### For A Team: Set Up The Organization Once

Blocks show their organization to every user: in the Desktop App, the logo, the "Developer" name, the URL and the support contact come from `block/package.json`. Without a setup, a new block keeps the placeholders that `block-tools structure init` writes: default logos, the `--org-scope` word as the developer, and `https://example.com`.

A team keeps a private copy of platforma-ctx with an `org/` folder:

- `org/ORG.md`, from `org/ORG.example.md`: organization name and URL, support contact, package scope, block registry, CI, review rules.
- `org/organization-logo.png`, and optionally `org/block-logo.png` as the default logo for new blocks: square PNG files, 1024 x 1024.
- `org/repos.txt` with the team's block repos, and `org/verticals.txt` with its vertical.

Everyone then clones the private copy. Setup copies `ORG.md` to the top of the workspace, and assistants fill every new block from it. Steps: [org/README.md](org/README.md).

## Work With An AI Assistant

Open **the workspace folder**, not `platforma-ctx/` and not a block folder. Assistants read `AGENTS.md` and the skill only at the top of the folder they work in.

Assistants read two things there, with no extra setup:

- `AGENTS.md`: loaded in every session.
- `.claude/skills/platforma-block-dev/SKILL.md`: loaded when a task is about a block.

| Tool | How to start | How to check that the context is loaded |
|---|---|---|
| GitHub Copilot in VS Code | open the workspace folder; use the chat in Agent mode | the Agent Customizations view lists `AGENTS.md` and the skill `platforma-block-dev`; in a reply, expand "References" |
| GitHub Copilot in JetBrains | open the workspace folder; Agent mode; skills on in Settings > Tools > GitHub Copilot > Customizations | the same settings page lists the skill |
| GitHub Copilot CLI | run `copilot` in the workspace folder | `/instructions` and `/skills list` |
| Claude Code (2.1.277 or newer) | run `claude` in the workspace folder | `/context` shows `AGENTS.md`; `/skills` lists the skill |
| Codex | run `./platforma-ctx/pl-ctx setup --codex` once, then `codex` in the workspace folder | Codex reads `AGENTS.md` and the copy in `.agents/skills/` |

If your company manages Copilot centrally, policies can switch off some features, for example agent mode, workspace skills or MCP servers. In VS Code, the command "Developer: Policy Diagnostics" shows which policies apply. platforma-ctx does not need MCP.

Example requests:

- "Explain how `blocks/vj-gene-usage` finds its input dataset and what it exports."
- "Create a new block in `blocks/` that counts clonotypes per sample and shows a table. Make a plan first."
- "Upgrade `blocks/my-block` to the current SDK, then build and test it."
- "My block fails with this error: … Find the cause."

The assistant plans first, follows real blocks, checks APIs in the SDK source, and builds and tests its changes. Review what it writes; the tests are the check.

## Optional: Search By Meaning With grepai

[grepai](https://github.com/yoanbernabeu/grepai) finds code by meaning, for example "where does a block pick its input dataset", where grep needs the exact words. It runs on your computer: the index and the embedding model ([Ollama](https://ollama.com)) stay local, and no code leaves the machine. platforma-ctx works without it. When it is set up, `AGENTS.md` tells the assistant to use it.

Set it up once, in the workspace folder:

```bash
brew install yoanbernabeu/tap/grepai     # other systems: see the grepai README
ollama pull nomic-embed-text             # Ollama must be installed and running
mkdir .grepai && cp platforma-ctx/grepai.yaml .grepai/config.yaml
grepai watch                             # first index; leave it running, it then follows changes
```

`platforma-ctx/grepai.yaml` is the config that `grepai init -p ollama -b gob` writes, plus rules that skip lock files, changelogs, binary and data files. Without these rules, the first index spends about half of its time on `pnpm-lock.yaml` files. The index is a local file in `.grepai/`.

The first index reads every file once. A full workspace with `--vertical vdj` is about 30 MB of text; on a recent Mac its first index takes up to about an hour (estimated from a small test). After that, the watcher updates only the files that change. grepai skips files that a repo's `.gitignore` excludes, such as the login file that block tests write. To run it without a terminal: `grepai watch --background`, then `grepai watch --status` and `grepai watch --stop`. If grepai already watches another project on this computer, add `--log-dir <a new folder>` to these three commands.

Try it: `grepai search "where does a block pick its input dataset"`.

Connect it to the assistant:

- **Claude Code:** run `claude mcp add grepai -- grepai mcp-serve "$PWD"` in the workspace folder.
- **Copilot in VS Code:** create `.vscode/mcp.json` in the workspace folder (not tested yet):

  ```json
  {
    "servers": {
      "grepai": { "type": "stdio", "command": "grepai", "args": ["mcp-serve", "${workspaceFolder}"] }
    }
  }
  ```

- **Without MCP**, for example when a Copilot policy switches MCP off: no setup. The assistant runs `grepai search` in a terminal.

## For People

- [docs/getting-started.md](docs/getting-started.md): one full loop. Build a block, load it into the Desktop App, change it, see the change, test it.
- [docs/test-and-release.md](docs/test-and-release.md): your organization's backend, releases, SDK upgrades, forks of public blocks.

## Status

Preview. The directions were checked on 2026-10-21 against `@platforma-sdk/model` 1.85.2, `@platforma-sdk/workflow-tengo` 6.16.0 and `@platforma-sdk/block-tools` 2.16.16. Where a guide and the SDK code disagree, the code is right; please open an issue so we fix the guide.
