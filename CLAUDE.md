# CLAUDE.md

brag-doc is a plugin of markdown prompt files — no executable code. It must work in **both
Claude Code and Codex**.

## Dual-runtime rules

- **Skills are the only entry points**: `skills/<name>/SKILL.md` (Claude `/brag-doc:<name>`,
  Codex `$<name>`). No `commands/` directory.
- **Agent instructions live once**, in `skills/*/references/<agent>.md`. (`references/` also holds
  shared **main-context** procedures — e.g. `render-overview.md`, `rebuild-themes.md` — that belong
  to no agent and need no stub; their headers say so.) `agents/*.md` are thin
  Claude-only stubs that just read the `instructionsFile` given in their dispatch prompt — never
  put real instructions in them, only config frontmatter: a `tools:` allowlist covering
  everything the reference file does (a new capability in a reference file may need a `tools:`
  update, or it silently fails in Claude Code only) and a `model:` tier (Codex ignores it).
- **Every dispatch step in a SKILL.md gives both branches**: Claude Code → named
  `brag-doc:<agent>` agent; Codex / others → generic `worker` agent told to read the
  instructions file. Both branches get the absolute `instructionsFile` path.
- **Pass all context explicitly in dispatch prompts** (repoPath, outputDir, …). Never rely on
  frontmatter injection or Claude-specific context — including path variables like
  `${CLAUDE_PLUGIN_ROOT}`; describe paths relative to the skill directory instead.
- **Claude-only tools need a fallback**: e.g. "in Claude Code use AskUserQuestion; otherwise
  present a numbered list". Reference files use only portable shell tools (`git`, `gh`, `jq`).
- **Two manifests, kept in sync**: `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json`
  — every shared field (name, version, description, author, keywords) must match; bump both on
  release. `skills`, `interface`, `repository` are Codex-only.

## Conventions

- Documents generated for the user are in Korean; agent-to-agent intermediates (e.g.
  `raw/digests/`) and skill/reference prose are in English.
- `slug` values: English kebab-case. All output lands under `<repo-root>/.brag-doc/`.
- Agents return compact JSON or count summaries, never file contents, to the main context.
- Structured state lives in `.brag-doc/data/*.json` (source of truth); `.md` outputs are render
  artifacts. Skills communicate through the JSON — never by parsing rendered markdown, except the
  deep-dive documents (their prose and frontmatter *are* the source), the one-time
  `rebuild-themes` migration, and render-overview's `계정` carry-over from a pre-0.3.0
  overview.md. The `심층`/`항목` checkboxes in overview.md are derived from file existence at
  render time.
