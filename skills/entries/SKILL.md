---
name: entries
description: Turn existing brag-doc deep-dive folders into resume contribution-entry candidate tables under .brag-doc/entries/<slug>.md. Use when the user wants resume bullets, achievement entries, or 이력서 항목 from their analyzed contributions; requires brag-doc deep-dive to have run first.
---

## Mission

From themes that already have deep-dive folders in `<repo-root>/.brag-doc/`, generate resume
contribution-entry candidate tables (`entries/<slug>.md`, backed by
`entries/<slug>.json`). Follow the steps below in order.

Resolve `<repo-root>` yourself with `git rev-parse --show-toplevel` and use the absolute path
everywhere below.

### Step 1: Check the overview

Read `<repo-root>/.brag-doc/overview.md`. **If it does not exist**, tell the user to run
the `scan` skill and then the `deep-dive` skill first, and stop.

### Step 2: Theme selection (interactive)

Present the themes whose deep-dive column (`심층`) is `[x]` in the theme table — i.e. a deep-dive
folder exists — and let the user pick several (in Claude Code use AskUserQuestion with
**multiSelect: true**; otherwise present a numbered list and ask for a comma-separated pick).
Put each theme's title and PR count in the option descriptions.
- If no theme has `[x]` in the 심층 column, tell the user to run the `deep-dive` skill first, and stop.
- (For re-generation, the user can name an already-generated theme directly.)

### Step 3: Dispatch entry-writer agents in parallel

For each selected theme, verify that `<repo-root>/.brag-doc/deep-dive/<slug>/index.md` exists; if missing, skip that theme and inform the user to run the `deep-dive` skill again to regenerate the folder.

Create the `<repo-root>/.brag-doc/entries/` directory if it is missing.

Agent instructions: [references/entry-writer.md](references/entry-writer.md).

Run **one agent per selected theme**, all concurrently — count the selected themes first and spawn
exactly that many.

- **Claude Code**: dispatch that many `brag-doc:entry-writer` agents **in a single message**.
- **Codex / others**: spawn that many `worker` agents, each told to read the instructions file above
  and follow it exactly.

Each dispatch prompt must include:
- `themeDoc`: `<repo-root>/.brag-doc/deep-dive/<slug>/index.md` (absolute path — the target of the
  theme's 심층 link in overview.md)
- `slug`, `title`
- `outputBase`: `<repo-root>/.brag-doc/entries/<slug>` (absolute path; the agent appends `.json`/`.md`)
- `instructionsFile`: absolute path of `references/entry-writer.md` inside this skill directory

If an `entries/<slug>.md` already exists, it will be overwritten — note this in the final report.

### Step 4: Update the overview

In `overview.md`'s theme table (header `| # | 주제 | 기여 | 기간 | 규모 | 심층 |`), ensure a `항목`
column exists immediately after the `심층` column (add it to the header row, the separator row, and
every data row — existing rows get `[ ]`). Set each generated theme's `항목` cell to
`[x](entries/<slug>.md)`. Leave the rest as `[ ]`.

### Step 5: Final report

Report the generated `entries/` file paths and the entry count per theme. Tell the user
each section of the `.md` (테마 전체, and each sub-group) opens with its own `⭐ 추천 조합` above the
table: put `x` in the `✓` cell of the rows you want, and pick one of `주도`/`구현` where both
appear. If any file was overwritten, say so.
