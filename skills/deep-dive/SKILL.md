---
name: deep-dive
description: Split themes from brag-doc's data/themes.json into feature/policy sub-groups and write PR-diff-based deep-dive documents under .brag-doc/deep-dive/<slug>/. Use when the user wants to dig into a scanned theme in depth; requires brag-doc scan to have run first.
---

## Mission

Pick themes from `<repo-root>/.brag-doc/data/themes.json` (a `.brag-doc/` folder at the repo root),
split each theme into feature/policy sub-groups, and produce a deep-dive folder per theme:
`deep-dive/<slug>/index.md` plus one document per sub-group.

Resolve `<repo-root>` yourself with `git rev-parse --show-toplevel` and use the absolute path
everywhere below.

### Step 1: Check the data

Call `<repo-root>/.brag-doc/raw` `<rawDir>` and `<repo-root>/.brag-doc/data` `<dataDir>` below.

`<rawDir>` is required on every path — the agents read PR bodies and diffs out of it. If
`<rawDir>/prs.json`, `<rawDir>/commits.json`, or `<rawDir>/meta.json` is missing, tell the user to
run the `scan` skill (re-collect) first, and stop.

Read `<rawDir>/meta.json` now; if its `fallback` field is `true`, the collector could not collect
PRs (`raw/prs.json` is `[]`, not missing) — tell the user that deep-dive is not supported in
fallback mode (out of scope for now), and stop **before** any migration or theme selection.

Then read `<dataDir>/themes.json`:
- **If it is missing but `<repo-root>/.brag-doc/overview.md` exists** (a legacy run), rebuild it
  first by following
  [../scan/references/rebuild-themes.md](../scan/references/rebuild-themes.md) (paths are relative
  to this skill's directory). Its `raw/` inputs are already checked above. If the rebuild
  validation fails, stop as that procedure says.
- **If neither exists**, tell the user to run the `scan` skill first, and stop.

### Step 2: Theme selection (interactive)

From `themes.json`'s `themes[]`, present the themes **without** an existing
`<repo-root>/.brag-doc/deep-dive/<slug>/index.md` and let the user pick several (in Claude Code
use AskUserQuestion with **multiSelect: true**; otherwise present a numbered list and ask for a
comma-separated pick). Put each theme's `signals` and PR count in the option descriptions.
If every theme already has an index.md, say so and stop. (For re-analysis, the user can name a
theme directly — an already-analyzed theme is simply re-analyzed from scratch; there is no
render-only mode for deep-dive documents.)

For each selected theme, take `slug`, `title`, `prs`, and `commits` straight from its object in
`themes.json` — never parse them out of overview.md.

### Step 3: Dispatch theme-grouper agents in parallel

Agent instructions: [references/theme-grouper.md](references/theme-grouper.md).

Run **one agent per selected theme, all concurrently** — count the selected themes first and spawn
exactly that many.

- **Claude Code**: dispatch that many `brag-doc:theme-grouper` agents **in a single message**.
- **Codex / others**: spawn that many `worker` agents, each told to read the instructions file above
  and follow it exactly. They must not modify any file.

Each dispatch prompt must include:
- `repoPath`: absolute path of the repo root
- `rawDir`: `<repo-root>/.brag-doc/raw` (absolute path)
- Theme info: `slug`, `title`, `prs` number array, `commits` short-hash array
  (both taken from the theme's object in `themes.json`; pass `[]` when a
  theme has none)
- `instructionsFile`: absolute path of `references/theme-grouper.md` inside this skill directory

Parse each returned JSON. If parsing fails, do not re-dispatch the agent — extract the JSON
portion from the returned text directly.

**Completeness check**: the union of the groups' `prs` must equal the theme's `prs`, **and** the
union of the groups' `commits` must equal the theme's `commits`. If any PR or commit is missing,
add it to the most relevant group yourself before Step 4.

### Step 3.5: Dispatch diff-digester agents in parallel

Agent instructions: [references/diff-digester.md](references/diff-digester.md).

Find the oversized items among the selected themes' PRs and commits (line changes > 2000):

```bash
jq '[.[] | select((.additions + .deletions) > 2000) | .number]' <rawDir>/prs.json
jq '[.[] | select((.additions + .deletions) > 2000) | .hash]' <rawDir>/commits.json
```

Intersect with the selected themes' `prs`/`commits`, then drop every item whose digest file
already exists (`<rawDir>/digests/pr-<n>.md` / `commit-<hash>.md` — merged diffs never change,
so old digests stay valid). If nothing remains, skip this step.

Run `mkdir -p <rawDir>/digests`, then **one agent per remaining item, all concurrently** —
these agents do not depend on Step 3's results, so dispatch them together with the
theme-groupers when possible:

- **Claude Code**: dispatch that many `brag-doc:diff-digester` agents **in a single message**.
- **Codex / others**: spawn that many `worker` agents, each told to read the instructions file
  above and follow it exactly.

Each dispatch prompt must include:
- `repoPath`: absolute path of the repo root
- `kind`: `"pr"` or `"commit"`, and `ref`: the PR number or commit short hash
- `outputFile`: `<rawDir>/digests/pr-<n>.md` or `<rawDir>/digests/commit-<hash>.md` (absolute path)
- `instructionsFile`: absolute path of `references/diff-digester.md` inside this skill directory

Each agent returns a JSON summary: `{ref, outputFile, files, size}`. Verify each `outputFile`
exists before Step 4; re-run the missing ones once.

### Step 4: Dispatch pr-analyzer agents in parallel

Prepare the output directories first. For each selected theme:
- `rm -rf <repo-root>/.brag-doc/deep-dive/<slug>` — clears any stale per-group docs and index.md
  from a previous run
- `mkdir -p <repo-root>/.brag-doc/deep-dive/<slug>` — recreate the empty folder

Agent instructions: [references/pr-analyzer.md](references/pr-analyzer.md).

Run **one agent per group**, across all selected themes, all concurrently — count the total groups
first and spawn exactly that many.

- **Claude Code**: dispatch that many `brag-doc:pr-analyzer` agents **in a single message**.
- **Codex / others**: spawn that many `worker` agents, each told to read the instructions file above
  and follow it exactly.

Each dispatch prompt must include:
- `repoPath`, `rawDir` (absolute paths)
- Theme info: `slug`, `title`
- Group info: `slug`, `title`, `prs` number array, `commits` short-hash array
- `outputFile`: `<repo-root>/.brag-doc/deep-dive/<theme-slug>/<group-slug>.md` (absolute path)
- `instructionsFile`: absolute path of `references/pr-analyzer.md` inside this skill directory

Each agent returns a JSON summary: `{groupSlug, outputFile, oneLiner, keyDecisions, size}`.
If a summary fails to parse, extract the JSON portion from the returned text directly.

**Verify before Step 5**: every group must have produced its `outputFile`. List the theme's
deep-dive directory and re-run the missing groups' agents before rendering index.md.

### Step 5: Render index.md yourself

For each analyzed theme, render `<repo-root>/.brag-doc/deep-dive/<slug>/index.md` **yourself —
do not delegate this to an agent** — from the theme's object in `themes.json` (`title`,
`slug`, `summary`, `prs`, `commits`, `period`, `size`), the
group objects from Step 3 (slug, title, prs, commits, summary), and the analyzer summaries
returned in Step 4 (oneLiner, keyDecisions, size).

Template (keep the Korean headings/labels as-is; fill in the values):

```markdown
---
theme: "<theme title>"
prs: [367, 380]
commits: ["a1b2c3d", "e4f5g6h"]
period: "2026-05 ~ 2026-07"
size: "+4.2k/-1.6k"
---

# <theme title>

## 개요

(2~4 paragraphs in Korean weaving the group one-liners into one theme narrative)

## 하위 그룹

| 그룹 | PR·커밋 | 규모 | 요약 |
|------|---------|------|------|
| [<group title>](<group-slug>.md) | #367, #380, `a1b2c3d` | +1.2k/-0.4k | <oneLiner> |

## 핵심 결정

- (bullet list merging the groups' keyDecisions, in Korean)
```

The frontmatter starts on line 1 of the document and keeps the key order of the template above.
`prs` is a YAML array of the theme's PR numbers (bare integers) and `commits` one of its
direct-commit short hashes, **both exhaustive**; when the theme has none of that kind, leave an
empty array `[]` rather than dropping the key. Every other value — `theme`, each commit hash,
`period`, `size` — must be double-quoted, so that a title containing `:` or a hash that looks
numeric (`1234567`, `1e23456`) cannot break the YAML. The `size` value is the abbreviated display
string rendered from the theme's `size.additions`/`size.deletions` integers in `themes.json` —
abbreviate each value at ≥1000 to one decimal with `k`, below 1000 keep the raw integer
(`{"additions": 4200, "deletions": 160}` → `"+4.2k/-160"`); the same rule applies to the
`하위 그룹` table's `규모` cells.

The `PR·커밋` cell of the `하위 그룹` table lists, in a single cell, **every** PR number (`#367`)
and direct-commit short hash (`` `a1b2c3d` ``) belonging to that group: the group object's `prs`
from Step 3 first, then its `commits`, each in chronological order, separated by `, `. Omit
whichever side is empty — a group can never have both empty.

### Step 6: Check the themes off in overview.md

For the themes that produced an index.md in Step 5, mark their `deep-dive` cell in
`<repo-root>/.brag-doc/overview.md` by following
[../scan/references/check-overview.md](../scan/references/check-overview.md) (paths are relative to
this skill's directory) with `column` = `deep-dive` and those themes' slugs. **Patch it yourself —
do not delegate this to an agent.**

### Step 7: Final report

Report the generated `deep-dive/<slug>/` folders (index.md + group files) and each group's
`oneLiner` returned by the agents, plus the overview.md result from Step 6.
