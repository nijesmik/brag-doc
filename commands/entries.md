---
description: Generate resume contribution-entry candidates from deep-dive folders into <repo-root>/.brag-doc/entries/<slug>.md
---

## Context

- Repo root: !`git rev-parse --show-toplevel`

## Mission

From themes that already have deep-dive folders in `<repo-root>/.brag-doc/`, generate resume
contribution-entry candidate tables (`entries/<slug>.md`, backed by
`entries/<slug>.json`). Follow the steps below in order.

### Step 1: Check the overview

Read `<repo-root>/.brag-doc/overview.md`. **If it does not exist**, tell the user to run
`/brag-doc:scan` and then `/brag-doc:deep-dive` first, and stop.

### Step 2: Theme selection (interactive)

Present the themes whose deep-dive column (`심층`) is `[x]` in the theme table — i.e. a deep-dive
folder exists — via AskUserQuestion (**multiSelect: true**). Put each theme's title and PR count in
the option descriptions.
- If no theme has `[x]` in the 심층 column, tell the user to run `/brag-doc:deep-dive` first, and stop.
- (For re-generation, the user can name an already-generated theme directly.)

### Step 3: Dispatch entry-writer agents in parallel

For each selected theme, verify that `<repo-root>/.brag-doc/deep-dive/<slug>/index.md` exists; if missing, skip that theme and inform the user to run `/brag-doc:deep-dive` again to regenerate the folder.

Create the `<repo-root>/.brag-doc/entries/` directory if it is missing. Dispatch one
`brag-doc:entry-writer` agent **per selected theme**, all **in a single message**. Each
dispatch prompt must include:
- `themeDoc`: `<repo-root>/.brag-doc/deep-dive/<slug>/index.md` (absolute path — the target of the
  theme's 심층 link in overview.md)
- `slug`, `title`
- `outputBase`: `<repo-root>/.brag-doc/entries/<slug>` (absolute path; the agent appends `.json`/`.md`)

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
