# quickpeach-skills

Agent **skills** for the [QuickPeach](https://github.com/Quickpeach) ecosystem — the
consistency layer that makes any AI agent (Claude Code, QuickPeach's built-in agent,
Codex, …) use QuickPeach the same way, every session.

A *skill* is a small `SKILL.md` (a pushy trigger description + a focused workflow) that
an agent consults automatically. These skills all share one discipline, proven by
SocratiCode (for code) and Context Hub (for docs): **query the index, don't read raw;
follow the conventions, don't improvise.**

## Skills

| Skill | Triggers when the user… | What it teaches |
|-------|-------------------------|-----------------|
| [`quickpeach-notes`](./quickpeach-notes/SKILL.md) | asks anything answerable from their notes | Query the semantic+link graph (`context_pack`) instead of reading raw notes. |
| [`quickpeach-create-note`](./quickpeach-create-note/SKILL.md) | wants to create / capture / save a note | Structure notes (title, `[[links]]`, `#tags`) so they connect into the graph — link-first, no orphans. |
| [`quickpeach-plugin-creator`](./quickpeach-plugin-creator/SKILL.md) | builds / scaffolds a QuickPeach extension | Manifest + SDK + view mounts + least-privilege permissions; defers to the full chub reference. |
| [`quickpeach-harden-loop`](./quickpeach-harden-loop/SKILL.md) | *(maintainer)* keeps hardening / improving the QuickPeach packages | The repeatable harden→**test**→**improve**→**research** loop + a paste-ready [kickoff prompt](./quickpeach-harden-loop/PROMPT.md). |

## Install

Each skill is a self-contained directory. Put the ones you want where your agent looks:

- **Claude Code:** copy a skill dir into `~/.claude/skills/`
  ```bash
  cp -r quickpeach-notes ~/.claude/skills/
  ```
- **Context Hub registry:** mirror under `author/skills/<name>/SKILL.md` in your
  `context-hub-content` repo, then `chub build` so `chub get` serves it.
- **QuickPeach agent:** point the agent's skill directory at this repo.

## Why skills (faster + consistent agents)

The workflow lives in the skill + the index, not in each chat's memory. So every agent,
every session, takes the same path and produces the same shape of answer — and durable
knowledge survives the session. Companion tools:

- **Semantic notes index:** [`quickpeach-graph-mcp`](https://github.com/Touutae-labs/quickpeach-graph-mcp) — the MCP server the `quickpeach-notes` skill drives.
- **Code index:** SocratiCode. **Docs/memory:** Context Hub (`chub`).

## License

MIT
