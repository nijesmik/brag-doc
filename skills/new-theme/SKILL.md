---
name: new-theme
description: Pick items out of the unclustered list in brag-doc's data/themes.json and turn them into a new theme that deep-dive can analyze. Use when the user wants to select unclustered items for further analysis; requires brag-doc scan to have run first. PR numbers or commit short hashes may be passed as arguments.
---

## Mission

Move user-chosen items out of `data/themes.json`'s `unclustered` list and append them to
`themes[]` as a new theme, so the `deep-dive` skill can analyze them later. Edit
`<repo-root>/.brag-doc/data/themes.json` yourself and re-render `overview.md` from it — no agents
are dispatched.

Resolve `<repo-root>` yourself with `git rev-parse --show-toplevel` and use the absolute path
everywhere below.

**Arguments**: the invocation may carry refs after the skill name — PR numbers (`#381` or `381`)
and/or commit short hashes (`f7g8h9i`). With refs, skip the interactive selection in Step 2; without
refs, ask the user to pick.

### Step 1: Check the data

Call `<repo-root>/.brag-doc/raw` `<rawDir>` and `<repo-root>/.brag-doc/data` `<dataDir>` below.

`<rawDir>` is required on every path — Step 2 reads the picked refs' titles from it and Step 4
computes the new theme's `기간`/`size` from it. If `<rawDir>/prs.json`, `<rawDir>/commits.json`, or
`<rawDir>/meta.json` is missing, tell the user to run the `scan` skill (re-collect) first, and
stop. Read `<rawDir>/meta.json` and note its `fallback` flag — Step 4's `규모` and the final report
depend on it.

Then read `<dataDir>/themes.json`:
- **If it is missing but `<repo-root>/.brag-doc/overview.md` exists** (a legacy run), rebuild it
  first by following
  [../scan/references/rebuild-themes.md](../scan/references/rebuild-themes.md) (paths are relative
  to this skill's directory). Its `raw/` inputs are already checked above. If the rebuild
  validation fails, stop as that procedure says.
- **If neither exists**, tell the user to run the `scan` skill first, and stop.

If `themes.json`'s `schemaVersion` is missing or is not `1`, stop **before editing anything** and
tell the user to re-run `scan` with "재수집" or "재클러스터" — Step 4 mutates this file, and the
render refuses any other schemaVersion, so editing first would leave a change the user can only
recover from by rebuilding (losing this run's theme).

If `themes.json`'s `unclustered.prs` and `unclustered.commits` are both empty, tell the user there
is nothing to pick, and stop.

### Step 2: Resolve the items (interactive when no arguments)

The pickable refs are exactly `<dataDir>/themes.json`'s `unclustered.prs` (PR numbers) and
`unclustered.commits` (commit short hashes) — never a bullet in overview.md, which may be stale.
List them with their titles from raw (`유형 \t 항목 \t 제목`, tab-separated):

```bash
jq -n -r --slurpfile t <dataDir>/themes.json \
  --slurpfile prs <rawDir>/prs.json --slurpfile commits <rawDir>/commits.json '
  ($t[0].unclustered) as $u
  | ([ $prs[0][] | select(.number as $n | $u.prs | index($n))
       | "PR\t#\(.number)\t\(.title)" ]
     + [ $commits[0][] | select(.hash as $h | $u.commits | index($h))
       | "커밋\t\(.hash)\t\(.subject)" ])
  | .[]'
```

- **With arguments**: match each token against those refs. A `#`-prefixed token is always a PR
  number. A bare all-digit token is matched against `unclustered.prs` first; if no PR matches, try
  it against `unclustered.commits` — and if it matches **both** a PR number and a commit hash, ask
  the user which one they meant instead of silently taking the PR. Any other token is a commit
  short hash and must equal an
  `unclustered.commits` hash exactly. If **any** token still matches nothing (already in a theme,
  or a typo), list the offending tokens and stop **without editing anything** — no partial
  application.
- **Without arguments**: let the user multi-select from the list above (in Claude Code use
  AskUserQuestion with **multiSelect: true**, label = ref, description = title/subject; otherwise
  present a numbered list and ask for a comma-separated pick). If the refs outnumber
  AskUserQuestion's option limit, use the numbered-list fallback there too.

Picking every ref is fine. Call the result `pickedPrs` (number array) and `pickedCommits`
(short-hash array) below; either may be empty, not both.

### Step 3: Name the theme

Ask the user for a theme title in Korean, offering `미분류 선별` as the default (in Claude Code use
AskUserQuestion with that default as the first option — the user can type their own via Other;
otherwise ask as free text, empty answer → default). If the title already appears in
`themes.json`'s `themes[].title`, ask for a different one — the `시간순 활동` tables reference
themes by title, so a duplicate would be ambiguous.

Derive the `slug` yourself: an English kebab-case translation of the title. If that slug already
appears among `themes[].slug` or as a `deep-dive/<slug>/` folder, append `-2`, `-3`, … until it is
unique.

### Step 4: Edit themes.json and re-render

Compute the new row's `기간` and `size` object from the raw files (substitute
`pickedPrs`/`pickedCommits` into `$nums`/`$hashes`):

```bash
jq -n -r --slurpfile prs <rawDir>/prs.json --slurpfile commits <rawDir>/commits.json \
  --argjson nums '[381]' --argjson hashes '["f7g8h9i"]' '
  ([ $prs[0][] | select(.number as $n | $nums | index($n)) | .mergedAt[:7] ]
   + [ $commits[0][] | select(.hash as $h | $hashes | index($h)) | .date[:7] ])
  | sort | if first == last then first else "\(first) ~ \(last)" end'
```

```bash
jq -n --slurpfile prs <rawDir>/prs.json --slurpfile commits <rawDir>/commits.json \
  --argjson nums '[381]' --argjson hashes '["f7g8h9i"]' '
  ([ $prs[0][] | select(.number as $n | $nums | index($n)) ]
   + [ $commits[0][] | select(.hash as $h | $hashes | index($h)) ])
  | {additions: (map(.additions) | add // 0), deletions: (map(.deletions) | add // 0)}'
```

This second command produces the theme's `size` object (`additions`/`deletions` integers) for
`themes.json` directly — it is not a 규모 display string.

**Fallback mode** (`fallback: true` in meta.json): skip the second command — `render-overview.md`
keys its `규모` column off `meta.json`'s `fallback` flag and renders every theme's cell as `—`
there (see the fallback note below for `$theme` itself).

Then update `<dataDir>/themes.json` — append the new theme and remove the picked refs from
`unclustered` in one jq pass (substitute the computed values into `$theme`):

```bash
jq --argjson theme '{
  "slug": "<slug>", "title": "<title>", "summary": "미분류에서 선별한 항목",
  "prs": <pickedPrs>, "commits": <pickedCommits>,
  "period": "<기간>", "size": {"additions": <additions>, "deletions": <deletions>}, "signals": []
}' '
  .themes += [$theme]
  | .unclustered.prs -= $theme.prs
  | .unclustered.commits -= $theme.commits
' <dataDir>/themes.json > <dataDir>/themes.json.tmp && mv <dataDir>/themes.json.tmp <dataDir>/themes.json
```

In fallback mode omit the `size` key from `$theme` entirely (matching the clusterer contract).

The `$theme` object sits inside a single-quoted shell argument, and the title is user-typed — if
the title (or any substituted value) contains `'` or `"`, do not inline it: write the JSON object
to `<dataDir>/theme.tmp.json`, pass it with `--argjson theme "$(cat <dataDir>/theme.tmp.json)"`,
and delete the temp file afterwards.

Then re-render `overview.md` by following
[../scan/references/render-overview.md](../scan/references/render-overview.md) — the new theme
row, the shrunken `## 미분류` section, and the re-themed `## 시간순 활동` cells all come out of
the render; do not hand-edit overview.md.

### Step 5: Final report

Report the new row — title, slug, refs, `기간`/`규모` (`규모` is `—` in fallback mode, where the
theme has no `size`) — and how many 미분류 items remain. Tell the
user the theme can now be analyzed with the `deep-dive` skill — naming it the way this platform
invokes it (`/brag-doc:deep-dive` in Claude Code, `$deep-dive` in Codex). In fallback mode, note
instead that deep-dive does not support fallback-mode data, so the row documents the grouping only.
Also mention durability: the hand-made theme now lives in `data/themes.json`, so it survives
"문서만 재렌더"; but "재수집" and "재클러스터" rebuild themes.json from scratch and will drop it.
