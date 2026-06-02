---
name: quickpeach-harden-loop
description: >-
  Maintainer loop for continuously hardening the QuickPeach packages (quickpeach-sync-core,
  quickpeach-graph-mcp, quickpeach-skills). Use this WHENEVER the task is to "keep hardening",
  "continue improving", "make it more robust / production-ready", "research a better technique
  and apply it", or to pick up open-ended work on these repos. It defines how to choose a
  target, research the technique, implement, TEST to green, improve (and reject dead weight),
  and commit — one tight increment at a time.
---

# QuickPeach hardening loop

A repeatable loop for advancing these repos without drifting into churn. Each pass is
**one coherent, tested, committed increment**. Bias toward *fewer, higher-confidence*
changes that genuinely pull their weight.

Repos (sibling dirs under `D:/tida/`):
- `quickpeach-sync-core` — E2EE CRDT note sync (Yjs + per-device R2 manifests).
- `quickpeach-graph-mcp` — semantic+link retrieval MCP (hybrid vector+BM25 RRF, PPR, MMR).
- `quickpeach-skills` — the agent skills (this repo).

## The loop

1. **Pick the highest-value target.** Prefer: a real correctness bug > an external-dependency
   failure mode > an edge case that crashes > a perf cliff > a quality/feature gain. If unsure,
   read the repo's `ARCHITECTURE.md` "roadmap"/"risks" and the last few commits.
2. **Research before coding** (see *Research*). Confirm the technique against a real source;
   don't reinvent or guess an API.
3. **Implement one increment.** Small surface, clear why-comment, no speculative generality.
4. **Test to green** (see *Test*). All gates pass or it doesn't ship.
5. **Improve / scrutinize** (see *Improve*). Ask "does this earn its place?" — if a feature
   rarely fires or duplicates existing behavior, **revert it** and say why. That's a win.
6. **Commit** with a message stating *what + why + the test result*. Push only when the owner
   approves (their rule: stop at push and ask). No `Co-Authored-By` in public repos.
7. Repeat, or drift to the next target. Checkpoint with the owner after a meaningful increment.

## Test (the green-bar gate)

Run all of these in the target repo; **every one must pass** before committing:

```bash
npm run typecheck      # tsc --noEmit
npm test               # vitest run — add/extend a test for the change
npm run build          # tsup — ESM + CJS + d.ts
npx eslint .           # 0 errors
```

- **Always add or extend a test** that would fail without your change (or assert the new
  behavior). A hardening change with no test isn't hardened.
- **NUL-byte gotcha (this environment):** the editor occasionally injects a stray NUL byte,
  which makes git flag a `.ts` file as *binary* (`Bin x -> y` in `git diff --stat`). After
  editing, check and rewrite if needed:
  ```bash
  od -An -tx1 src/<file>.ts | tr ' ' '\n' | grep -c '^00$'   # must be 0
  ```
- For graph-mcp, smoke the bin too: `QPGRAPH_FAKE_EMBED=1 node dist/server.js --query "…" samples`.

## Improve (the scrutinize lens)

Before and after writing, question it:
- **Intent first:** is there a simpler approach that reaches the same goal? Would removing
  code beat adding it?
- **Earn its place:** does the feature actually fire in real cases? (e.g. adaptive score-cliff
  cutoff was rejected because RRF flattens gaps and unrelated notes are already pruned — it
  was dead weight.) Cut what doesn't pull its weight.
- **Hardening checklist** — walk these for any engine change:
  - empty / huge / malformed input (empty corpus, 0-length note, dup ids/titles)
  - external-dependency failure: timeout, unreachable, partial response → *actionable* error, never a hang
  - correctness under change: model/version/key swap (cache keyed by producer identity), concurrency, idempotency
  - perf: no O(n²) over the corpus; precompute once
  - output bounds: cap tool output so one huge item can't blow the agent's context
  - security/privacy: least privilege, no secrets in code, stays E2EE / on-device

## Research (don't reinvent; mine proven techniques)

1. **Local project memory first:** `chub search quickpeach` / `chub get pantakan/<doc>`
   (e.g. `quickpeach-extensions`, `agent-mcp-architecture`, `quickpeach-module-atlas`).
   Prefer registry docs over guessing the codebase.
2. **Mine the sibling indexes' techniques** — they encode what works:
   - **SocratiCode** (code index): hybrid semantic + BM25 + **RRF**, `minScore`, *search before read*.
   - **Context Hub** (chub): curated condensed docs + **progressive disclosure** / *search → get*.
   - Apply the same to notes: query a ranked **map**, drill down with `get_note`, never dump raw.
3. **Survey current art** for the specific problem (RAG reranking → MMR; CRDT merge; MCP token
   limits) via web/docs, then **apply + test + measure**. Cite the source in the commit/why-comment.
4. The north star: **use fewer tokens, more effectively** — the agent reads a curated, ranked,
   non-redundant map, not the vault.

## Kickoff prompt

To start (or continue) a session, paste [`PROMPT.md`](./PROMPT.md) — it tells the agent to run
this loop autonomously, one increment at a time, drifting across the three repos.
