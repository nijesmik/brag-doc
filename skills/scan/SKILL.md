---
name: scan
description: Collect the user's own git/PR contributions in the current repo and cluster them into themes, writing <repo-root>/.brag-doc/overview.md. Use when the user wants to scan, inventory, or start analyzing their contributions in a repo — the first step of brag-doc, before deep-dive or entries.
---

## Mission

Analyze the user's contributions in the current repo and generate `<repo-root>/.brag-doc/overview.md`
(a `.brag-doc/` folder at the repo root). Follow the steps below in order.

Resolve `<repo-root>` yourself with `git rev-parse --show-toplevel` and use the absolute path
everywhere below.

**Output language**: overview.md is written in Korean. The rendering procedure in
[references/render-overview.md](references/render-overview.md) carries the Korean headings and
labels — keep them exactly as-is and fill in only the values.

### Step 1: Identity confirmation (interactive)

First, determine the current identity by running these yourself (do **not** rely on frontmatter
injection — these commands require permission on a fresh install):
- `git config user.name` and `git config user.email` → the local git user
- `gh api user --jq .login 2>/dev/null || echo none` → the GitHub login (`none` if not logged in)
- `git shortlog -sne HEAD | head -25` → the repo's top authors, then
  `git shortlog -sne HEAD | grep -iF <user.name or user.email>` → the user's entries even
  when they sit far down the list

Also detect the repo's **default branch** (`baseBranch`) — all collection is anchored to it.
Take the first that succeeds:
- `gh repo view --json defaultBranchRef --jq .defaultBranchRef.name 2>/dev/null`
- `git rev-parse --abbrev-ref origin/HEAD 2>/dev/null | sed 's@^origin/@@'`
- `git remote show origin 2>/dev/null | sed -n 's/.*HEAD branch: //p'`
- `git rev-parse --abbrev-ref HEAD` (final fallback — current branch)

Show the detected `baseBranch` to the user (display only, no selection), and pass it to the
collector in Step 3.

Then confirm with the user (in Claude Code use AskUserQuestion with **multiSelect: true**;
otherwise present a numbered list and ask for a comma-separated pick):
- Present the authors matching or similar to git user.name as default candidates, and let the user
  **multi-select the author names they have used** (one person commonly uses several names).
- If the gh login is `none`, tell the user the run will proceed with commits only, without PR collection.

### Step 2: Re-run check

If `<repo-root>/.brag-doc/raw/prs.json` already exists, ask the user to choose (in Claude Code use
AskUserQuestion; otherwise a numbered list):
- "재수집" (recommended default — picks up new PRs/commits) → proceed from Step 3
- "재클러스터 (기존 raw 재사용)" → skip Step 3 and start from Step 4
- "문서만 재렌더" — only offer this option when `data/themes.json` **or** a legacy `overview.md`
  exists. Zero agent dispatches: if `data/themes.json` is missing, first rebuild it from the
  legacy overview.md by following
  [references/rebuild-themes.md](references/rebuild-themes.md), then jump straight to Step 5
  (render) and Step 6. Content stays identical; only the document format is refreshed — this is
  the choice to use after a plugin update.

(The quoted strings are the option labels shown to the user — keep them in Korean.)

### Step 3: Dispatch the collector agent

Agent instructions: [references/collector.md](references/collector.md).

- **Claude Code**: dispatch the `brag-doc:collector` agent.
- **Codex / others**: spawn one `worker` agent told to read the instructions file above and follow
  it exactly.

The dispatch prompt must include:
- `repoPath`: absolute path of the repo root
- `outputDir`: `<repo-root>/.brag-doc/raw` (absolute path)
- `ghLogin`: the confirmed gh login (`none` if unavailable)
- `gitAuthors`: the author names confirmed in Step 1
- `baseBranch`: the default branch detected in Step 1
- `instructionsFile`: absolute path of `references/collector.md` inside this skill directory

### Step 4: Dispatch the clusterer agent

Agent instructions: [references/clusterer.md](references/clusterer.md).

- **Claude Code**: dispatch the `brag-doc:clusterer` agent.
- **Codex / others**: spawn one `worker` agent told to read the instructions file above and follow
  it exactly. It must not modify any file.

The dispatch prompt must include the absolute `rawDir` path and `instructionsFile` (absolute path of
`references/clusterer.md` inside this skill directory).
Parse the returned JSON. If parsing fails, do not re-dispatch the agent — extract the JSON portion
from the returned text directly.

Then persist it: run `mkdir -p <repo-root>/.brag-doc/data` and save the parsed clusterer JSON to
`<repo-root>/.brag-doc/data/themes.json`, adding `"schemaVersion": 1` as the first top-level key.
This file — not the in-memory return — is the source of truth every later step and skill reads;
overview.md is a rendered artifact derived from it.

### Step 5: Render overview.md

Render `<repo-root>/.brag-doc/overview.md` by following
[references/render-overview.md](references/render-overview.md) exactly — **render it yourself; do
not delegate this to an agent.** It reads `data/themes.json` (saved in Step 4) plus the `raw/`
files, and derives the `심층`/`항목` checkbox columns from file existence.

### Step 6: Final report

Summarize the generated file path, the theme count, and the deep-dive candidates (themes with
signals), and mention that the user can continue with the `deep-dive` skill — naming it the way this
platform invokes it (`/brag-doc:deep-dive` in Claude Code, `$deep-dive` in Codex).
