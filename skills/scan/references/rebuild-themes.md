# rebuild-themes — legacy overview.md → data/themes.json (one-time migration)

Reconstructs `<repo-root>/.brag-doc/data/themes.json` from a legacy overview.md that predates the
data/ directory. Followed by the **main context** of the `scan`, `new-theme`, and `deep-dive`
skills when `data/themes.json` is missing but `overview.md` exists — this is not an agent
instructions file. Requires `raw/prs.json`, `raw/commits.json`, `raw/meta.json`; the calling skill
checks they exist before invoking this.

**Inputs to resolve before starting**: `<repo-root>` (absolute), `<rawDir>` =
`<repo-root>/.brag-doc/raw`, `<dataDir>` = `<repo-root>/.brag-doc/data`.

Two overview.md layouts exist in the wild and the parsing below handles both, detected by the
theme table's header row: the **section-based** layout every release through 0.2.0 wrote, and the
**merged-table** layout only unreleased builds wrote. An unrecognized header stops the migration,
and the ref-union check below is the net for anything that parses but parses wrong.

## Parse the legacy overview.md

Read `<repo-root>/.brag-doc/overview.md` and extract:

- `stats`: **do not parse the header bullet lines** — `raw/meta.json` holds every field
  authoritatively, so read it from there:

  ```bash
  jq '{prCount: (.myPrs | tonumber), commitCount: (.myCommits | tonumber),
       directCommitCount: (if .fallback then (.myCommits | tonumber)
                           else (.myDirectCommits | tonumber) end),
       totalCommits: (.totalCommits | tonumber),
       period: (if .firstDate[:7] == .lastDate[:7] then .firstDate[:7]
                else "\(.firstDate[:7]) ~ \(.lastDate[:7])" end)}' <rawDir>/meta.json
  ```

  (`tonumber` accepts both, since the collector may have written the counts as numbers or as
  strings. The `period` shape matches the clusterer's `YYYY-MM ~ YYYY-MM` contract.
  `directCommitCount` is mode-dependent, per the clusterer contract: it counts the hashes across
  `themes[].commits` + `unclustered.commits`, which in **fallback mode** is *every* commit — the
  same population the fallback validation below enforces — so it reads `myCommits` there, not the
  filtered `myDirectCommits`. Getting this wrong renders `직접 커밋` smaller than `커밋 총`, which
  the render explicitly says cannot happen in fallback mode.)
- `themes[]`: one object per theme. **Two layouts exist — detect which by the theme table's header
  row under `## 주제별 기여`, then follow the matching branch below.** Every field lands in the
  same place either way; only where you read it from differs.

  **Layout A — section-based** (header `| # | 주제 | 기여 | 기간 | 규모 | 심층 |`). This is what
  every released version through 0.2.0 wrote, so it is the layout a real legacy run almost always
  has. The table carries only counts; the per-theme `### <n>. <title>` sections carry the rest:
  - `title` from the section heading (`### <n>. <title>`), matching the row's `주제` cell
  - `slug` from the section's `- slug: <slug>` line
  - `summary` from the paragraph between the section heading and the `- slug:` line, collapsed to
    a single line (the render writes it into a table cell)
  - `prs` from the section's `- 관련 PR:` line (`#367, #380, …` → integers); the line is **absent**
    when the theme has none → `[]`
  - `commits` from the section's `- 관련 커밋:` line (bare short hashes, comma-separated);
    absent → `[]`
  - `signals` from the section's `- 심층 분석 후보 신호:` line, comma-split; `없음` → `[]`
  - `period` from the row's `기간` cell

  **Layout B — merged table** (header `| # | 주제 | 요약 | 관련 기여 | 기간 | 규모 | 신호 | 심층 |`).
  Written only by unreleased builds between the merge of the per-theme sections and 0.3.0, so you
  will meet it mainly on a developer's checkout. There are no per-theme sections; read everything
  from the row:
  - `title` and `slug` from the `주제` cell (`<title> (`<slug>`)`)
  - `summary` from the `요약` cell (verbatim)
  - `prs` (integers) and `commits` (short hashes) from every ref in the `관련 기여` cell
  - `period` from the `기간` cell
  - `signals` from the `신호` cell, comma-split; `없음` → `[]`

  If the header matches neither, stop and tell the user the overview.md layout is unrecognized and
  they should re-run `scan` with "재수집".

  **Both layouts**, for every theme:
  - `size`: **do not parse the `규모` cell** (it may be abbreviated). Recompute exactly from raw
    (skip in fallback mode — omit the key, matching the clusterer contract):

    ```bash
    jq -n --slurpfile prs <rawDir>/prs.json --slurpfile commits <rawDir>/commits.json \
      --argjson nums '<theme prs>' --argjson hashes '<theme commits>' '
      ([ $prs[0][] | select(.number as $n | $nums | index($n)) ]
       + [ $commits[0][] | select(.hash as $h | $hashes | index($h)) ])
      | {additions: (map(.additions) | add // 0), deletions: (map(.deletions) | add // 0)}'
    ```
- `unclustered`: `{prs, commits}` from the `## 미분류` bullets' leading refs; a missing section or
  missing side → `[]`. Ignore the `심층`/`항목` checkbox cells — status is derived from file
  existence, never stored.

Assemble `{"schemaVersion": 1, "stats": ..., "themes": [...], "unclustered": {...}}` and write it
to `<dataDir>/themes.json.tmp` (`mkdir -p <dataDir>` first).

## Validate before adopting

The union of all refs must equal raw's population. Check PRs and commits (both must print `true`):

```bash
jq -n --slurpfile raw <rawDir>/prs.json --slurpfile t <dataDir>/themes.json.tmp '
  ($raw[0] | map(.number) | sort) ==
  (($t[0] | [.themes[].prs[]] + .unclustered.prs) | sort)'
```

Commit population depends on `raw/meta.json`'s `fallback`: normal mode counts direct commits only,
fallback mode counts every commit.

```bash
# normal mode
jq -n --slurpfile raw <rawDir>/commits.json --slurpfile t <dataDir>/themes.json.tmp '
  ($raw[0] | map(select(.firstParent and .pr == null and .parents < 2) | .hash) | sort) ==
  (($t[0] | [.themes[].commits[]] + .unclustered.commits) | sort)'
# fallback mode: drop the select(...) filter — the map becomes map(.hash)
```

- Both `true` → `mv themes.json.tmp themes.json` and continue.
- Any `false` → `rm themes.json.tmp`, report which refs are missing/extra, and **stop** — do not
  guess assignments. Compute both directions with the same jq expressions, replacing `==` with
  `-`: `raw - themes` lists the refs the rebuild dropped, `themes - raw` the ones it invented.

This runs once per legacy repo; afterwards `data/themes.json` is always the source of truth.
