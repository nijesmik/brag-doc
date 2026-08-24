# render-overview — overview.md rendering procedure

Renders `<repo-root>/.brag-doc/overview.md` from `data/themes.json` + `raw/`. Followed by the
**main context** of the `scan` and `new-theme` skills — this is not an agent instructions file.
Requires `<repo-root>/.brag-doc/data/themes.json` to exist (schemaVersion 1: the clusterer's
`stats` / `themes[]` / `unclustered` shape).

**Inputs to resolve before starting**: `<repo-root>` (absolute). Read
`<repo-root>/.brag-doc/data/themes.json`, `raw/meta.json`, `raw/prs.json`, `raw/commits.json`.

Fill the template below with `data/themes.json` + `raw/meta.json` + `raw/prs.json` and save it to
`<repo-root>/.brag-doc/overview.md`. **Render it yourself — do not delegate this to an agent.**

Extract the chronological data with (merges PRs and direct commits into one timeline, emitting the
month key and day together). Use the **normal-mode** command when `raw/meta.json`'s `fallback` is
`false`/absent; use the **fallback-mode** command when it is `true` — do not use the normal-mode
command in fallback mode, its `select` silently drops commits (a `(#N)` squash-merge subject, a
merge commit with `parents >= 2`, a commit off the first-parent line) that the clusterer still
counted into the theme table's `관련 기여` column, `미분류`, and `directCommitCount`.

Normal mode (`firstParent && pr == null && parents < 2`, i.e. direct commits only):

```bash
jq -n -r --slurpfile prs <repo-root>/.brag-doc/raw/prs.json --slurpfile commits <repo-root>/.brag-doc/raw/commits.json '
  ([ $prs[0][] | {date: .mergedAt[:10], kind: "PR", ref: "#\(.number)", title: .title} ]
   + [ $commits[0][] | select(.firstParent and .pr == null and .parents < 2)
       | {date: .date[:10], kind: "커밋", ref: .hash, title: .subject} ])
  | sort_by(.date) | .[]
  | "\(.date[:7])\t\(.date[5:])\t\(.kind)\t\(.ref)\t\(.title)"'
```

Fallback mode (no `select` — every commit is included, matching the clusterer's fallback behavior;
`prs.json` is `[]` so the PR half contributes nothing):

```bash
jq -n -r --slurpfile prs <repo-root>/.brag-doc/raw/prs.json --slurpfile commits <repo-root>/.brag-doc/raw/commits.json '
  ([ $prs[0][] | {date: .mergedAt[:10], kind: "PR", ref: "#\(.number)", title: .title} ]
   + [ $commits[0][] | {date: .date[:10], kind: "커밋", ref: .hash, title: .subject} ])
  | sort_by(.date) | .[]
  | "\(.date[:7])\t\(.date[5:])\t\(.kind)\t\(.ref)\t\(.title)"'
```

The output is 5 tab-separated columns: `월키 \t MM-DD \t 유형 \t 항목 \t 제목`. Split the monthly tables
on the first column, and put the second column in the date column.

Fill the 테마 column from the theme each PR/commit belongs to (`미분류` if unclustered).

**Checkbox columns are derived from file existence — never from a previous overview.md:**

- `심층` cell: `[x](deep-dive/<slug>/index.md)` if `<repo-root>/.brag-doc/deep-dive/<slug>/index.md`
  exists, else `[ ]`.
- `항목` column: include it (right after `심층` in the header, separator, and every row) **only if**
  `<repo-root>/.brag-doc/entries/<slug>.md` exists for at least one theme in the table. A theme's
  cell is `[x](entries/<slug>.md)` if its file exists, else `[ ]`. If no theme has an entries
  file, do not add the column.

Check existence with a single `ls` per directory (`ls <repo-root>/.brag-doc/deep-dive/*/index.md`
and `ls <repo-root>/.brag-doc/entries/*.md`, tolerating "no matches").

Template (keep the Korean headings/labels as-is; fill in the values). All per-theme data lives in
the theme table — do not add per-theme sections. Table rules:

- `주제` cell: the theme title followed by its slug in backticks, in parentheses — deep-dive uses
  the slug as the `deep-dive/` folder name.
- `요약` cell: the theme's `summary`, on a single line.
- `관련 기여` cell: the counts with every ref in parentheses —
  ``PR <prs count>개 (#367, #380, ...) · 커밋 <commits count>개 (`a1b2c3d`, ...)``, commit hashes
  in backticks. Omit whichever side is empty (and the ` · ` separator with it).
- `신호` cell: the theme's signals comma-separated; `없음` if none.

In `## 미분류`, omit whichever list is empty — PR-only or
commit-only is fine — and if both `prs` and `commits` are empty, omit the entire `## 미분류` section:

For each unclustered ref, take the title/subject from `raw/prs.json` / `raw/commits.json` and write
the one-line Korean summary yourself from the title and body — summaries are regenerated on every
render.

```markdown
# <repo> 기여 분석

- **레포**: <owner/repo> (<contributors>인 기여)
- **기간**: <stats.period>
- **규모**: PR <prCount>개, 직접 커밋 <directCommitCount>개, 커밋 총 <commitCount>개 (전체 <totalCommits>개의 ~N%)
- **계정**: <gitAuthors>, gh: <ghLogin>
- **기준 브랜치**: <meta.baseBranch> (<meta.baseRef>)
- **수집일**: <meta.collectedAt date only>

## 주제별 기여

| # | 주제 | 요약 | 관련 기여 | 기간 | 규모 | 신호 | 심층 |
|---|------|------|-----------|------|------|------|------|
| 1 | <title> (`<slug>`) | <summary> | PR 2개 (#367, #380) · 커밋 2개 (`a1b2c3d`, `e4f5g6h`) | <period> | +<additions>/-<deletions> | <signals> | [ ] |

## 미분류

- #381 <title> (one-line summary in Korean)
- `f7g8h9i` <subject> (one-line summary in Korean)

## 시간순 활동

### 2026-05

| 날짜 | 유형 | 항목 | 제목 | 테마 |
|------|------|------|------|------|
| 05-11 | PR | #384 | 블랙박스 0-byte PUT 회귀 수정 | 블랙박스 사진 업로드 파이프라인 |
| 05-11 | 커밋 | `a1b2c3d` | HEIC 변환 타임아웃 30초로 상향 | 블랙박스 사진 업로드 파이프라인 |
| 05-12 | PR | #387 | 모달/바텀시트 하드백 처리 | 웹뷰 내비게이션/뒤로가기 정책 |
```

**Fallback-mode rendering** (when `fallback` in `meta.json` is `true`): use the same template with
only these differences:

- In the theme table, the `관련 기여` column holds only `커밋 <n>개 (…)` and the `규모` column is `—`.
- Build `## 시간순 활동` with the **fallback mode** extraction command above — it must include every
  commit (no `select`) so that the theme table's `관련 기여` column, `미분류`, `directCommitCount`,
  and the chronological activity stay consistent with each other.
- On the 규모 line, `직접 커밋 <n>개` ends up equal to `커밋 총 <n>개`, because without PR records
  every commit counts as a direct commit. Even if that looks misleading, do not hide the number or
  recompute it — expose exactly what the clusterer returned.
