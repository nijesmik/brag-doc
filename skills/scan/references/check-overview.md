# check-overview — mark a theme's checkbox in overview.md

Flips the `deep-dive` or `entries` cell of one or more theme rows in
`<repo-root>/.brag-doc/overview.md` to `[x]` right after those files are written. Followed by the
**main context** of the `deep-dive` and `entries` skills — this is not an agent instructions file.

This is a surgical in-place patch, not a re-render: it touches only the target cells and leaves
every other byte of overview.md alone. A full re-render belongs to
[render-overview.md](render-overview.md), which needs `raw/` that `entries` never reads.

**Inputs to resolve before starting**: `<repo-root>` (absolute), `column` (`deep-dive` or
`entries`), and the list of theme `slug`s whose files were just written.

### Step 1: Preconditions

If `<repo-root>/.brag-doc/overview.md` does not exist, skip this procedure entirely and tell the
user to run the `scan` skill to render it.

Read the header row of the `## 주제별 기여` table. If its last two columns are not
`| deep-dive | entries |` — a pre-0.4.0 overview still has `| 심층 |` and an optional `| 항목 |` —
do **not** hand-patch it. Skip the update and tell the user to re-render once with the `scan`
skill's "문서만 재렌더" option, which produces the current column layout.

### Step 2: Patch one row per slug

For each slug, find the single row of that table whose line contains `` (`<slug>`) `` — that
literal appears only in a theme row's `주제` cell. If
no row matches (the theme was renamed or re-clustered since the last render), skip that slug and
say so in the final report.

Locate the target cell **from the end of the line, not by counting cells from the left** — a
free-form Korean `주제`/`요약`/`신호` cell can itself contain a `|`, so a left-to-right split is not
reliable. Every such row ends with exactly two checkbox cells:

```
… | <deep-dive cell> | <entries cell> |
```

and each of those two cells is always either `[ ]` or `[x](<path>)` — nothing else. Take the last
two `|`-delimited fields of the line on that basis, and replace the target one with:

- `column` = `deep-dive` → `[x](deep-dive/<slug>/index.md)`
- `column` = `entries` → `[x](entries/<slug>.md)`

If the trailing two fields do not both look like `[ ]` or `[x](…)`, treat the row as unrecognized:
skip that slug and report it rather than guessing.

Rewrite that one line only; keep every other cell of the row byte-identical. Do not touch other
rows, the `## 미분류` section, or `## 시간순 활동` (whose `항목` column is unrelated — it holds
PR/commit refs).

This procedure only ever sets `[x]` — it never clears a cell back to `[ ]`. A checkbox left `[x]`
by a folder that was since deleted is corrected by the next full render, which stays the sole
authority on checkbox state.

### Step 3: Report

Report which slugs were checked, and name any slug that was skipped and why.
