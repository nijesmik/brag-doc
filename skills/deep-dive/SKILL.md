---
name: deep-dive
description: Split themes from an existing brag-doc overview.md into feature/policy sub-groups and write PR-diff-based deep-dive documents under .brag-doc/deep-dive/<slug>/. Use when the user wants to dig into a scanned theme in depth; requires brag-doc scan to have run first.
---

## Mission

Pick themes from `<repo-root>/.brag-doc/overview.md` (a `.brag-doc/` folder at the repo root),
split each theme into feature/policy sub-groups, and produce a deep-dive folder per theme:
`deep-dive/<slug>/index.md` plus one document per sub-group.

Resolve `<repo-root>` yourself with `git rev-parse --show-toplevel` and use the absolute path
everywhere below.

### Step 1: Check the overview

Read `<repo-root>/.brag-doc/overview.md`. **If it does not exist**, tell the user to run
the `scan` skill first, and stop.

### Step 2: Theme selection (interactive)

Present the themes whose deep-dive column is `[ ]` in the theme table and let the user pick several
(in Claude Code use AskUserQuestion with **multiSelect: true**; otherwise present a numbered list and
ask for a comma-separated pick).
Put each theme's deep-dive candidate signals and PR count in the option descriptions.
If every theme is already `[x]`, say so and stop. (For re-analysis, the user can name a theme directly.)

If `raw/prs.json` is missing, the agents cannot read PR bodies — check before dispatching and,
if missing, tell the user to run the `scan` skill (re-collect) first.

Also read `<repo-root>/.brag-doc/raw/meta.json`; if its `fallback` field is `true`, the collector
could not collect PRs (`raw/prs.json` is `[]`, not missing) — tell the user that deep-dive is not
supported in fallback mode (out of scope for now), and stop.

For each selected theme, read both `관련 PR:` and `관련 커밋:` from its section in overview.md.
A theme may have only PRs, only commits, or both — pass whichever exist, using `[]` for the other.

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
  (both extracted from the theme's section in the overview; pass `[]` when a theme has none)
- `instructionsFile`: absolute path of `references/theme-grouper.md` inside this skill directory

Parse each returned JSON. If parsing fails, do not re-dispatch the agent — extract the JSON
portion from the returned text directly.

**Completeness check**: the union of the groups' `prs` must equal the theme's `prs`, **and** the
union of the groups' `commits` must equal the theme's `commits`. If any PR or commit is missing,
add it to the most relevant group yourself before Step 4.

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
do not delegate this to an agent** — from the theme's section in overview.md, the
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
numeric (`1234567`, `1e23456`) cannot break the YAML.

The `PR·커밋` cell of the `하위 그룹` table lists, in a single cell, **every** PR number (`#367`)
and direct-commit short hash (`` `a1b2c3d` ``) belonging to that group: the group object's `prs`
from Step 3 first, then its `commits`, each in chronological order, separated by `, `. Omit
whichever side is empty — a group can never have both empty.

### Step 6: Update the overview

After rendering, change the deep-dive column of each analyzed theme in the overview.md
theme table to `[x](deep-dive/<slug>/index.md)`.

### Step 7: Final report

Report the generated `deep-dive/<slug>/` folders (index.md + group files) and each group's
`oneLiner` returned by the agents.
