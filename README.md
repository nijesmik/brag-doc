# brag-doc

A plugin that turns the contributions you made in a repo into theme-based analysis documents.
It collects the PRs and direct commits you authored, clusters them into themes, then digs into each
theme using real PR diffs and distills the result into resume contribution-entry candidates.

Works in both Claude Code and Codex.

## Install

The plugin is distributed from the [`nijesmik/plugins`](https://github.com/nijesmik/plugins)
marketplace, which registers under the name `nijesmik`.

**Claude Code**

```bash
claude plugin marketplace add nijesmik/plugins
claude plugin install brag-doc@nijesmik
```

**Codex**

```bash
codex plugin marketplace add nijesmik/plugins
```

After adding the marketplace, install `brag-doc` from the plugin list. Codex has no dedicated
install command in its CLI.

## Usage

Run these in order from inside the repo you want to analyze. Each step consumes the previous step's
output.

| Step | Claude Code | Codex | What it does |
|------|-------------|-------|--------------|
| 1 | `/brag-doc:scan` | `$scan` | Collects your contributions, clusters them into themes, writes `overview.md` |
| 2 | `/brag-doc:deep-dive` | `$deep-dive` | Splits themes into sub-groups and analyzes them against PR diffs |
| 3 | `/brag-doc:entries` | `$entries` | Extracts resume contribution-entry candidates from the deep dives |

Step 1 opens with an interactive prompt confirming the git author names you have used. Steps 2 and 3
let you pick which themes to work on.

The generated documents are written in Korean.

## Output

Everything lands under `.brag-doc/` at the repo root.

```
.brag-doc/
├── overview.md              # Contributions by theme + chronological activity
├── raw/                     # Collected source data (prs.json, commits.json, meta.json)
├── deep-dive/<theme>/
│   ├── index.md             # Theme overview and sub-group table
│   └── <group>.md           # Per-group deep dive
└── entries/<theme>.md       # Resume contribution-entry candidates
```

## Requirements

- `git` — required
- [`gh`](https://cli.github.com/) CLI, logged in — needed to collect PRs. Without it the plugin falls
  back to commits only, and deep-dive is not supported in that mode.
- `jq` — used to process the collected data
