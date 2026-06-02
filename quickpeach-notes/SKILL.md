---
name: quickpeach-notes
description: >-
  Answer from the user's QuickPeach notes / knowledge base / "second brain" using the
  quickpeach-graph-mcp tools instead of reading note files directly. Use this WHENEVER a
  question could be answered from the user's own notes — "what did I write about X", "find
  my notes on Y", "summarize my thinking on Z", "what's linked to this note", "do I have
  anything about…" — even if the user doesn't name a tool. Querying the note graph is far
  cheaper and more accurate than dumping raw notes into context.
---

# QuickPeach Notes — query the graph, don't read raw notes

The user's notes are indexed as a **semantic + link knowledge graph** (the
`quickpeach-graph-mcp` MCP server). Treat it like SocratiCode treats code:
**query before reading.** A single `context_pack` call returns the most relevant
notes — ranked by hybrid vector search + link-graph expansion + personalized
PageRank — with snippets, a *reason* each surfaced, and the connecting subgraph.
That is dramatically cheaper and more accurate than reading note files one by one,
and it surfaces connected/central notes that plain text search misses.

If the `quickpeach-graph-*` tools are not available, the server isn't connected —
say so and fall back to normal file reading; do not pretend.

## Core rule

**Start with `context_pack`, then `get_note` only what you must open.** This is the
search→get discipline that keeps you token-cheap:

1. `context_pack(query)` returns a compact **map** — ranked notes with title, score,
   reason, and a query-focused **snippet** (not the full note).
2. Answer from the snippets. Call **`get_note(id)`** to pull a note's *full* markdown ONLY
   when its snippet isn't enough and you've decided that specific note matters. Never read
   notes speculatively to find out if they're relevant — that's what the map is for.

```
context_pack(query, limit?, minScore?) → ranked map (title, score, reason, snippet) + subgraph
get_note(idOrTitle)                     → full markdown of ONE note (the drill-down)
```

Retrieval is **hybrid** (semantic vector + BM25 keyword, RRF-fused) then graph-PageRank
re-ranked — so exact terms/names land too, not just fuzzy meaning. Each hit's `reason`
("matched query · 3 backlinks · central via links") and blended `score` explain *why* it
surfaced; cite notes by title. Use `minScore` (0..1) to drop weak hits when you want a
tight, low-token answer (the pack reports an `omitted` count).

## Goal → Tool

| Goal | Tool |
|------|------|
| Answer a question / find context from the notes | `context_pack(query)` ← default |
| Hybrid "find notes about/mentioning X" | `search(query, k?)` |
| Read ONE note in full (after the map) | `get_note(idOrTitle)` |
| "What's related to this note?" | `related(note, k?)` |
| "What links here?" / find references | `backlinks(note)` |
| "What's directly connected?" | `neighbors(note)` |
| "How are A and B connected?" | `path_between(a, b)` |
| "What's tagged #X?" / pivot by topic | `notes_by_tag(tag)` |
| Size/shape of the knowledge base | `graph_stats()` |

`note` accepts an id **or** a title. All tools are read-only — they never modify notes.

## Workflow

1. **Reframe the user's question as a query** and call `context_pack`. Prefer a natural-
   language query ("how I handle dough hydration") over keywords — the vector layer wants
   meaning, not just terms.
2. **Read the pack, not the vault.** Answer from the snippets + reasons. If one note is
   clearly central and you need more than its snippet, fetch *that* note's full text.
3. **Follow the graph when the question is relational** — "what depends on this idea",
   "what did I link from my project note" → `neighbors` / `backlinks` / `related`, not a
   fresh search.
4. **Cite.** Name the notes you used; the user navigates by title.
5. **Tune `limit`** down (3–5) for a focused answer, up (10+) for a survey/overview.

## Why this is the right default

Reading raw notes burns tokens, has no ranking, and misses the *links* the user
deliberately made between ideas. The graph encodes both meaning (embeddings) and the
user's own structure (`[[wikilinks]]`/backlinks/tags), and PageRank lifts the notes that
are central to a topic — so a small `context_pack` usually beats reading ten notes. This
is the same "index is a map; raw reading is expensive" principle SocratiCode applies to
code and Context Hub applies to docs.

## Source + related skills

The `context_pack` / `search` / `related` / `backlinks` / `neighbors` / `path_between` /
`graph_stats` tools are served by the **quickpeach-graph-mcp** MCP server (the notes graph
index). To read a note's *full* text beyond the pack snippet, open it (host note-open /
`sdk.notes.open`). To **create** a note that connects into this graph, use the
`quickpeach-create-note` skill — it links-first so new notes land findable here.
