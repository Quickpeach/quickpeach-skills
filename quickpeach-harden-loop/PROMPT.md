# Kickoff prompt — QuickPeach hardening loop

Paste this to start/continue an autonomous hardening session. Edit the repo line to focus it.

---

Keep hardening the QuickPeach packages — `quickpeach-sync-core`, `quickpeach-graph-mcp`,
`quickpeach-skills` (sibling dirs under `D:/tida/`). Follow the **quickpeach-harden-loop**
skill. Work **one tight, tested increment at a time**, and drift across the repos where you
see the most value.

Each increment:
1. **Pick** the highest-value target (correctness bug > external-dep failure > crashing edge
   case > perf cliff > quality/feature). Skim the repo's `ARCHITECTURE.md` roadmap/risks first.
2. **Research** the technique before coding — `chub get pantakan/<doc>`, mine SocratiCode/Context
   Hub patterns, or survey current art; cite the source. Don't guess an API or reinvent.
3. **Implement** a small, well-scoped change with a why-comment.
4. **Test to green** — `npm run typecheck && npm test && npm run build && npx eslint .` all pass,
   add/extend a test that would fail without the change, and verify no stray NUL byte
   (`od -An -tx1 <file> | tr ' ' '\n' | grep -c '^00$'` → 0).
5. **Scrutinize** — does it earn its place? If a change doesn't pull its weight, **revert it and
   say why** (that's a valid outcome).
6. **Commit** with *what + why + test result*. **Do not push** — stop and ask me first. No
   `Co-Authored-By` in public repos. Don't touch `D:/tida/quickpeach` (the app) unless I ask.
7. Report the increment in 3–5 lines, then continue to the next target (or checkpoint with me).

North star: **fewer tokens, more effective** — the agent reads a curated, ranked, non-redundant
map, not the raw vault. Terse updates, no trailing fluff. Go.
