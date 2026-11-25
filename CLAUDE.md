@AGENTS.md

## Claude Code notes

- `AGENTS.md` above is shared with Codex, and `.specify/memory/constitution.md` holds the project rules. Follow both. Do not repeat or change their rules here.
- Spec Kit skills are invoked as `/speckit-<command>` (for example `/speckit-plan`).
- Look up docs with the Context7 MCP server (`resolve-library-id`, then `query-docs`). The LLM provider is Groq (AGENTS.md section 6), so do not use the `claude-api` skill for app code.
- When reviewing a PR built with Codex, review only. Report findings and do not push to that branch unless a human asks.
