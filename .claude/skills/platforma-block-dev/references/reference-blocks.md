---
stability: volatile
verified: 2026-10-21
sdk: "@platforma-sdk/model 1.84.1 to 1.85.2 across the blocks below"
---

# Reference Blocks

Real, current, public blocks that platforma-ctx clones into `blocks/`. Use them to see how a working block does something before you write it yourself.

Every block below uses the current model API (`BlockModelV3.create` with a kind), has a `kind/` package, and has tests. They are not perfect. When two blocks do the same thing in different ways, check the SDK source in `core/platforma/` to find out which way is right.

## Which Block To Read For What

| You want to see | Read | Where to look first |
|---|---|---|
| The smallest complete analysis block | `blocks/vj-gene-usage/` | `workflow/src/main.tpl.tengo`, `model/src/index.ts`, `kind/src/index.ts`, `ui/src/pages/` |
| A table engine (`pt`) computation and importing its result as p-columns | `blocks/vj-gene-usage/` | `workflow/src/usage.lib.tengo` (`xsv.importFile`), `workflow/src/specs.lib.tengo` |
| Reading region sequences (CDR3 and others), paired chains, plots | `blocks/cdr3-spectratype/` | `workflow/src/main.tpl.tengo`, `model/src/index.ts` |
| Running a packaged tool once per sample | `blocks/fastqc/` | `workflow/src/`, `model/src/index.ts` |
| Turning a user's file into p-columns; a test that builds a dataset from a TSV | `blocks/xsv-import/` | `test/src/wf.test.ts` |
| A per-clonotype score that other blocks can filter on; a subset filter as input; a results table | `blocks/repertoire-score/` | `model/src/index.ts`, `ui/src/pages/MainPage.vue` |
| Python software in a block; a subset filter; template tests that feed hand-made columns into a workflow template | `blocks/generation-probability/` | `software/`, `workflow/src/subset.test.ts`, `workflow/src/test/` |
| A Python package with a full test suite (unit, integration, regression) | `blocks/titeseq-analysis/` | `software/tests/` |
| Reading all seven antibody regions and handling producer differences; Tengo unit tests | `blocks/sequence-properties/` | `workflow/src/main.tpl.tengo`, `workflow/src/*.test.tengo` |

## Blocks That Produce Common Inputs

Read these to learn the exact specs of the columns your block will consume. Do not copy their style blindly: they are large and old in places.

| Block | What it exports | Where the specs are built |
|---|---|---|
| `blocks/mixcr-clonotyping/` | clonotype datasets: per-region sequences, genes, abundances, CDR3 length | `workflow/src/calculate-export-specs.lib.tengo` |
| `blocks/import-vdj-data/` | clonotype datasets from AIRR, MiXCR, 10x and table files, including amino-acid input | `workflow/src/infer-columns-common.lib.tengo`, `workflow/src/bare-set-specs.lib.tengo` |
| `blocks/redefine-clonotypes/` | a regrouped clonotype dataset, region numbering by ANARCI (IMGT, Kabat, Chothia) | `workflow/src/process.tpl.tengo` |

## Antibody Region Columns: What Every Consumer Must Know

Most antibody blocks read the region sequences. All producers above use one column name, `pl7.app/vdj/sequence`, with the region in the domain key `pl7.app/vdj/feature` (for example `FR1`, `CDR3`) and the alphabet in `pl7.app/alphabet` (`aminoacid` or `nucleotide`). Differences that break naive code:

- The amino-acid FR4 is called `FR4InFrame` by MiXCR Clonotyping, but `FR4` by Import V(D)J Data. Accept both.
- A germline-imputed region has the same name and domain as an observed one. Only the annotation `pl7.app/vdj/imputed: "true"` tells them apart.
- A missing region is usually NA. MiXCR writes the text `region_not_covered` in its own export files; MiXCR Clonotyping turns it into NA, but a block that imports such files may carry the text. Treat NA, an empty string and `region_not_covered` all as missing.
- In bulk data the chain is in the axis domain. In paired data (single-cell, and also paired sequences imported from a table) it is a column domain, `pl7.app/vdj/scClonotypeChain` (`A` or `B`), with `pl7.app/vdj/scClonotypeChain/index` (`primary` or `secondary`). Do not conclude from these keys that the data is single-cell.
- Amino-acid input can carry a numbering scheme in `pl7.app/vdj/numberingSchema` (`imgt`, `kabat`, `chothia`). Region boundaries, and so lengths, differ between schemes.
- A MiXCR run with CDR3 as its explicit assembling feature exports CDR3 only; no other region columns exist for it. A preset without an assembling feature (RNA-seq style) exports all seven regions, with NA where a region is not covered (`blocks/mixcr-clonotyping/workflow/src/calculate-export-specs.lib.tengo`).

`blocks/sequence-properties/workflow/src/main.tpl.tengo` handles most of these; read it before you write a consumer.

## A Block That Consumes Scores And Filters

`blocks/antibody-tcr-lead-selection/` reads the score columns (`pl7.app/isScore`) and subset filter columns (`pl7.app/isSubset`) that other blocks export, and lets the user rank and filter clonotypes with them. Read its `model/src/util.ts` to see how an exported score or filter becomes a filter in its UI. The SKILL.md traps list explains the `isSubset` rule.

## Known Differences From A New Block

The reference blocks are older than `block-tools structure init` 2.16. Both forms below work; a new block gets the second one.

| In most reference blocks | In a block made by `structure init` today |
|---|---|
| the block package is named `@<scope>/<org>.<name>` | it is named `@<scope>/<org>.<name>.block` |
| `.changeset/config.json` has an empty `fixed` list | the block's packages are in one `fixed` group, so they are released together |

## Not Good References

- Blocks that still use the old model API (`BlockModel.create` without a kind). The SDK no longer supports writing them.
- Blocks whose CI runs no tests.
