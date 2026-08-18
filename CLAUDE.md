# CLAUDE.md

brag-doc is a plugin of markdown prompt files — no executable code. It must work in **both
Claude Code and Codex**.

## Dual-runtime rules

- **Skills are the only entry points**: `skills/<name>/SKILL.md` (Claude `/brag-doc:<name>`,
  Codex `$<name>`). No `commands/` directory.
- **Agent instructions live once**, in `skills/*/references/<agent>.md`. `agents/*.md` are thin
  Claude-only stubs that just read the `instructionsFile` given in their dispatch prompt — never
  put real instructions in them, only a `tools:` allowlist covering everything the reference
  file does (a new capability in a reference file may need a `tools:` update, or it silently
  fails in Claude Code only).
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

- Generated documents are in Korean; skill/reference prose is in English.
- `slug` values: English kebab-case. All output lands under `<repo-root>/.brag-doc/`.
- Agents return compact JSON or count summaries, never file contents, to the main context.
