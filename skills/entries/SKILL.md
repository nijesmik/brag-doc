---
name: entries
description: Turn existing brag-doc deep-dive folders into resume contribution-entry candidate tables under .brag-doc/entries/<slug>.md. Use when the user wants resume bullets, achievement entries, or 이력서 항목 from their analyzed contributions; requires brag-doc deep-dive to have run first.
---

## Mission

From themes that already have deep-dive folders in `<repo-root>/.brag-doc/`, generate resume
contribution-entry candidate tables (`entries/<slug>.md`, backed by
`data/entries/<slug>.json`). Follow the steps below in order.

Resolve `<repo-root>` yourself with `git rev-parse --show-toplevel` and use the absolute path
everywhere below.

### Step 1: Check the data

Call `<repo-root>/.brag-doc/data` `<dataDir>` below. Read `<dataDir>/themes.json`:
- **If it is missing but `<repo-root>/.brag-doc/overview.md` exists** (a legacy run), rebuild it
  first by following
  [../scan/references/rebuild-themes.md](../scan/references/rebuild-themes.md) (paths are relative
  to this skill's directory). That rebuild — and only it — needs `raw/`: check that
  `<repo-root>/.brag-doc/raw/prs.json`, `commits.json`, and `meta.json` all exist **before**
  starting it, and if any is missing tell the user to run the `scan` skill (re-collect) first, and
  stop. If the rebuild validation fails, stop as that procedure says.
- **If neither exists**, tell the user to run the `scan` skill and then the `deep-dive` skill
  first, and stop.

Outside that legacy path `entries` never reads `raw/` — a run with `data/themes.json` already on
disk works with `raw/` pruned.

### Step 2: Theme selection (interactive)

Legacy layout, before the selection below: for each theme whose
`<repo-root>/.brag-doc/entries/<slug>.json` exists (pre-0.3.0 runs kept the JSON next to the .md),
run `mkdir -p <repo-root>/.brag-doc/data/entries` and `mv` it to
`<repo-root>/.brag-doc/data/entries/<slug>.json`. Do this first so the offer below and every later
run see the JSON in its new home.

Present the themes in `themes.json` that have an existing
`<repo-root>/.brag-doc/deep-dive/<slug>/index.md` and let the user pick several (in Claude Code
use AskUserQuestion with **multiSelect: true**; otherwise present a numbered list and ask for a
comma-separated pick). Put each theme's title and PR count in the option descriptions.
- If no theme has a deep-dive folder, tell the user to run the `deep-dive` skill first, and stop.
- (If a selected theme already has `<repo-root>/.brag-doc/entries/<slug>.md`, ask the user per
  theme — in Claude Code use AskUserQuestion (one question per theme, or one multi-theme question
  with a per-theme option pair); otherwise a numbered choice: "재생성" — dispatch the agent
  normally, or "문서만 재전사" — dispatch it with
  `transcribeOnly: true` so it only re-renders the .md from the existing JSON. Offer "문서만 재전사"
  **only when `<repo-root>/.brag-doc/data/entries/<slug>.json` also exists** after the `mv` above —
  with no JSON there is nothing to transcribe, so 재생성 is then the only option and no question is
  asked.)

### Step 3: Dispatch entry-writer agents in parallel

For each selected theme, verify that `<repo-root>/.brag-doc/deep-dive/<slug>/index.md` exists; if missing, skip that theme and inform the user to run the `deep-dive` skill again to regenerate the folder.

Create `<repo-root>/.brag-doc/data/entries/` and `<repo-root>/.brag-doc/entries/` if missing.

Agent instructions: [references/entry-writer.md](references/entry-writer.md).

Run **one agent per selected theme**, all concurrently — count the selected themes first and spawn
exactly that many.

- **Claude Code**: dispatch that many `brag-doc:entry-writer` agents **in a single message**.
- **Codex / others**: spawn that many `worker` agents, each told to read the instructions file above
  and follow it exactly.

Each dispatch prompt must include:
- `themeDoc`: `<repo-root>/.brag-doc/deep-dive/<slug>/index.md` (absolute path — the theme's
  deep-dive index)
- `slug`, `title`
- `outputJson`: `<repo-root>/.brag-doc/data/entries/<slug>.json` (absolute path)
- `outputMd`: `<repo-root>/.brag-doc/entries/<slug>.md` (absolute path)
- `transcribeOnly`: `true` only when the user chose "문서만 재전사" in Step 2; omit otherwise
- `instructionsFile`: absolute path of `references/entry-writer.md` inside this skill directory

If an `entries/<slug>.md` already exists, it will be overwritten — note this in the final report.

### Step 4: Final report

Report the generated `entries/` file paths and the entry count per theme. Tell the user
each section of the `.md` (테마 전체, and each sub-group) opens with its own `⭐ 추천 조합` above the
table: put `x` in the `✓` cell of the rows you want, and pick one of `주도`/`구현` where both
appear. If any file was overwritten, say so.

Note that overview.md's `항목` column reflects file existence and updates on the next render
(`scan` → "문서만 재렌더").
