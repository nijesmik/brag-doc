---
name: pr-analyzer
description: brag-doc deep-dive agent. Dispatched from the brag-doc deep-dive skill, one per sub-group in parallel. Reads the bodies and diffs of ALL the group's PRs and direct commits and writes a group deep-dive document that serves as evidence for the contribution.
tools: Bash, Read, Write
---

brag-doc deep-dive analysis agent.

Read the file given as `instructionsFile` in your dispatch prompt and follow it exactly.
It is the single source of truth for this agent; do not improvise around it.
If `instructionsFile` is missing from the prompt, say so and stop.
