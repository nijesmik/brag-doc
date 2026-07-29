---
name: scan
description: Collect the user's own git/PR contributions in the current repo and cluster them into themes, writing <repo-root>/.brag-doc/overview.md. Use when the user wants to scan, inventory, or start analyzing their contributions in a repo — the first step of brag-doc, before deep-dive or entries.
---

## Mission

Analyze the user's contributions in the current repo and generate `<repo-root>/.brag-doc/overview.md`
(a `.brag-doc/` folder at the repo root). Follow the steps below in order.

Resolve `<repo-root>` yourself with `git rev-parse --show-toplevel` and use the absolute path
everywhere below.

**Output language**: overview.md is written in Korean. The template below already carries the Korean
headings and labels — keep them exactly as-is and fill in only the values.

### Step 1: Identity confirmation (interactive)

First, determine the current identity by running these yourself (do **not** rely on frontmatter
injection — these commands require permission on a fresh install):
- `git config user.name` and `git config user.email` → the local git user
- `gh api user --jq .login 2>/dev/null || echo none` → the GitHub login (`none` if not logged in)
- `git shortlog -sn HEAD | head -25` → the repo's author list

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

If `<repo-root>/.brag-doc/raw/prs.json` already exists, ask the user to choose:
- "재수집" (recommended default — picks up new PRs/commits) → proceed from Step 3
- "기존 raw 재사용" (re-cluster only) → skip Step 3 and start from Step 4

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

### Step 5: Render overview.md

Fill the template below with the theme JSON + `raw/meta.json` + `raw/prs.json` and save it to
`<repo-root>/.brag-doc/overview.md`. **Render it yourself — do not delegate this to an agent.**

Extract the chronological data with (merges PRs and direct commits into one timeline, emitting the
month key and day together). Use the **normal-mode** command when `raw/meta.json`'s `fallback` is
`false`/absent; use the **fallback-mode** command when it is `true` — do not use the normal-mode
command in fallback mode, its `select` silently drops commits (a `(#N)` squash-merge subject, a
merge commit with `parents >= 2`, a commit off the first-parent line) that the clusterer still
counted into the theme table, `관련 커밋`, `미분류`, and `directCommitCount`.

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

**If overview.md already exists**: before overwriting, collect the slugs whose deep-dive column is
checked — `[x](deep-dive/<slug>/index.md)` — in the existing theme table, and preserve the `[x]`
link **verbatim** for any matching slug in the new table.
Symmetrically, if the existing theme table has an `항목` column (added by
the `entries` skill), keep that column in the re-rendered table and preserve each
matching slug's `[x](entries/<slug>.md)` value verbatim; non-matching or new rows get
`[ ]`. If the existing overview has no `항목` column, do not add one.

Template (keep the Korean headings/labels as-is; fill in the values). Every theme section must
include its `- slug: <slug>` line — deep-dive uses it as the `deep-dive/` folder name. In the theme
table's `기여` column, omit whichever side is empty: if `commits` is empty, write only `PR <n>개`;
if `prs` is empty, write only `커밋 <n>개`. In `## 미분류`, omit whichever list is empty — PR-only or
commit-only is fine — and if both `prs` and `commits` are empty, omit the entire `## 미분류` section:

```markdown
# <repo> 기여 분석

- **레포**: <owner/repo> (<contributors>인 기여)
- **기간**: <stats.period>
- **규모**: PR <prCount>개, 직접 커밋 <directCommitCount>개, 커밋 총 <commitCount>개 (전체 <totalCommits>개의 ~N%)
- **계정**: <gitAuthors>, gh: <ghLogin>
- **기준 브랜치**: <meta.baseBranch> (<meta.baseRef>)
- **수집일**: <meta.collectedAt date only>

## 주제별 기여

| # | 주제 | 기여 | 기간 | 규모 | 심층 |
|---|------|------|------|------|------|
| 1 | <title> | PR <prs count>개 · 커밋 <commits count>개 | <period> | +<additions>/-<deletions> | [ ] |

### 1. <title>

<summary>

- slug: <slug>
- 관련 PR: #367, #380, ... (omit this line when the theme has no PRs)
- 관련 커밋: a1b2c3d, e4f5g6h (omit this line when the theme has no direct commits)
- 심층 분석 후보 신호: <signals comma-separated; "없음" if none>

## 시간순 활동

### 2026-05

| 날짜 | 유형 | 항목 | 제목 | 테마 |
|------|------|------|------|------|
| 05-11 | PR | #384 | 블랙박스 0-byte PUT 회귀 수정 | 블랙박스 사진 업로드 파이프라인 |
| 05-11 | 커밋 | `a1b2c3d` | HEIC 변환 타임아웃 30초로 상향 | 블랙박스 사진 업로드 파이프라인 |
| 05-12 | PR | #387 | 모달/바텀시트 하드백 처리 | 웹뷰 내비게이션/뒤로가기 정책 |

## 미분류

- #381 <title> (one-line summary in Korean)
- `f7g8h9i` <subject> (one-line summary in Korean)
```

**Fallback-mode rendering** (when `fallback` in `meta.json` is `true`): use the same template with
only these differences:

- In the theme table, the `기여` column holds only `커밋 <n>개` and the `규모` column is `—`.
- In each theme section, omit the `관련 PR` line and write only `관련 커밋`.
- Build `## 시간순 활동` with the **fallback mode** extraction command above — it must include every
  commit (no `select`) so that the theme table, `관련 커밋`, `미분류`, `directCommitCount`, and the
  chronological activity stay consistent with each other.
- On the 규모 line, `직접 커밋 <n>개` ends up equal to `커밋 총 <n>개`, because without PR records
  every commit counts as a direct commit. Even if that looks misleading, do not hide the number or
  recompute it — expose exactly what the clusterer returned.

### Step 6: Final report

Summarize the generated file path, the theme count, and the deep-dive candidates (themes with
signals), and mention that the user can continue with the `deep-dive` skill — naming it the way this
platform invokes it (`/brag-doc:deep-dive` in Claude Code, `$deep-dive` in Codex).
