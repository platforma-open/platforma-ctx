# Block Description

How to write the two texts that describe a block to its users: the short `description` and the long description in `docs/description.md`. Users read them in the Desktop App when they choose a block. Write for a biologist who has never seen the block, not for a developer.

Most public blocks in `blocks/` do not follow these rules yet. Do not copy their descriptions as a model.

## Where The Texts Appear

| Field | Where it lives | What the user sees |
|---|---|---|
| `description` | `block/package.json`, `block.meta.description` | the card in the block list: plain text, cut after 3 lines, or after 1 line when the title wraps |
| `longDescription` | `docs/description.md`, set as `"longDescription": "file:../docs/description.md"` | the detail page, after a click on the card: markdown, scrolls |

The detail page shows, from the top: the title, the tags, the short description, then `docs/description.md`. So the file starts right under a title and a summary that the user has just read. Do not repeat them. Headings, paragraphs, lists, bold, code, links, tables and block quotes show; images do not.

The side panel shows other fields: Developer (`organization.name`), URL (`url`), Docs (`docs`), Support (`support`), and the supported systems (`supportedPlatforms`). Do not repeat them in the text.

## The Short Description

- One or two sentences, 200 characters or fewer, plain text.
- The most important part first: the text must make sense when it is cut to one line.
- Say what the block produces, not how. Do not start with the block title.

Good: "Groups similar antibody or peptide sequences into clusters by sequence identity, and gives one representative sequence per cluster."

## The Long Description: Six Paragraphs, No Headings

Write plain paragraphs in this order. No headings: not `# Overview`, and no section headings. The order is the structure. Aim for 150 to 400 words.

1. **What it does.** Two to four sentences: what the block does, and which biological question it answers. Start with the action, not with the tool. Good: "Groups similar sequences into clusters, so related clonotypes that may share a target can be treated as one lead." Bad: "This block wraps MMseqs2."
2. **What it needs.** The input, in the user's words. Name the block that makes it by its title in the block list. When several blocks can make it, name the common ones and add "or any block that produces …". If the block takes files from disk, say so.
3. **Which data it supports, and what not.** Bulk or single cell; the sequence types (VHH, scFv, paired chains, TCR, BCR, peptides, synthetic libraries); the largest input really tested ("tested up to 2 million clonotypes", not "scales well"). Then one line that is never left out: `**Not supported:** …`, or `**Not supported:** No known limits.` when there are none.
4. **What it gives.** The main columns and plots, what the numbers mean, and their units. Say what kind of block can use the result next (for example "blocks that work with abundance"), not a list of named blocks: a list goes stale and makes other blocks look unsupported.
5. **How it works.** The method in three to six sentences, enough to trust the result and to write the methods section of a paper. Then the one or two settings that change the answer, and what changing them does.
6. **Tools and citation.** For every external tool the block runs: its name, its authors, its license, and the paper to cite, in a block quote. Users in companies must know a tool's license before they run it. Leave this paragraph out only when the block runs no external software.

When a fact is not known (for example the largest tested input), write the sentence with a visible gap and tell the developer. Do not invent it.

## Never Write

| Do not write | Because |
|---|---|
| `PColumn`, `PFrame`, `result pool`, `exports`, `Tengo`, `workflow`, `spec` | developer words; a biologist reads "columns" or "table" |
| "This block wraps <tool>" as the first line | the user wants a result, not a wrapper |
| the supported systems, a docs link, a support email, the changelog, the version | the side panel already shows them |
| images | they do not show |
| "powerful", "state of the art", "seamless" | they say nothing |

## Example

The public Sequence Clustering block (`blocks/clonotype-clustering`), written in this form:

```markdown
Groups similar sequences into clusters, so that clones which may share a target or a function can be handled as one group instead of thousands of separate rows.

A dataset of variants, produced by **MiXCR Clonotyping**, **Peptide Profiling** or any block that outputs per-clonotype or per-variant sequences.

Supports antibody and TCR clonotypes, and peptide variants. Amino acid or nucleotide sequences, chosen in the settings. Works on bulk and on single cell input.

**Not supported:** clustering across different sequence types in one run.

Every input sequence gets a cluster. For each cluster there is one centroid sequence, plus a reference centroid, which is always a real observed member of the cluster, so it can be synthesized directly. There is also the abundance of each cluster per sample, the distance of each member to its centroid, and the cluster radius. Bubble plots and histograms show the cluster sizes. The result can go to blocks that work with abundance, such as enrichment or distribution analysis, to look at clusters instead of single clones.

Sequences are clustered with MMseqs2, using Easy Cluster or Easy Linclust. Sequences that meet an identity threshold go into one cluster. The centroid is either a consensus of the aligned members, or the member that MMseqs2 chose as representative. Two settings change the answer. **Sequence identity** sets how similar two sequences must be to share a cluster. **Residue weighting** decides whether every clonotype counts once or counts by its abundance; keep the default, equal weight, unless the centroid should reflect clonal expansion.

Clustering uses MMseqs2, from the Söding lab and the Steinegger group (https://github.com/soedinglab/MMseqs2), under the GPL-3.0 license.

> Steinegger M and Söding J. MMseqs2 enables sensitive protein sequence searching for the analysis of massive data sets. Nature Biotechnology 35, 1026–1028 (2017). https://doi.org/10.1038/nbt.3988
```

The largest tested input is still missing in this example: only someone who ran the block can add it.
