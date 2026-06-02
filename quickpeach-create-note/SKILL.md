---
name: quickpeach-create-note
description: >-
  Create / capture / save a note in QuickPeach the right way — structured so it CONNECTS
  into the user's knowledge graph. Use this WHENEVER the user wants to write down, capture,
  save, jot, or add a note, idea, meeting summary, snippet, or anything to their notes /
  "second brain" — even if they don't say the word "note". A note created without links and
  tags is an orphan the graph can't surface later, so always connect it.
---

# QuickPeach — create notes that connect

A note is only as useful as its connections. QuickPeach indexes notes as a
semantic + link graph (see the `quickpeach-notes` skill), so a new note that links
to related notes and carries tags becomes instantly findable and lifts the notes
around it. A wall-of-text orphan does not. Your job when creating a note is to make
it a good *node*, not just a file.

## The note format

A note is Markdown:

- **Title** — the first line as a single `# Heading`. Keep it specific and search-
  friendly ("Cold-proof sourdough timing", not "Notes").
- **Body** — the content. One idea per note (atomic) beats a sprawling dump; split
  distinct topics into separate linked notes.
- **`[[Wikilinks]]`** — link to related notes by their title: `see [[Sourdough Starter]]`.
  This is the single most valuable thing you add — it's how the user (and the graph)
  navigate between ideas. Supports `[[Title|alias]]` and `[[Title#heading]]`.
- **`#tags`** — 1–4 topical tags (`#baking #technique`). Tags group notes the way links
  connect them.

## Workflow — link-first, never orphan

1. **Find what to connect to BEFORE writing.** Query the graph for the topic
   (`context_pack` / `search` / `related` from the `quickpeach-notes` skill) to discover
   existing notes this one should reference. Linking into the existing structure is what
   makes the note discoverable later.
2. **Draft atomically.** Title → body (one idea) → `[[links]]` to the notes you found →
   `#tags`. If the content spans two clear topics, propose two linked notes instead of one.
3. **Reuse exact titles for links.** A `[[wikilink]]` only resolves if it matches an
   existing note's title — confirm the target title from your graph query, don't guess.
   Linking to a not-yet-existing title is fine too (it becomes a "to write" stub).
4. **Confirm the connection.** When you report back, name the notes you linked and the
   tags you applied so the user sees how it slots in.

## Example

Input: "save a note that the levain was ready in 6h at 26°C"

Output:
```markdown
# Levain timing — 6h at 26°C

Levain peaked at ~6 hours when kept at 26°C (warmer than the usual 8–10h at room temp).
Use the float test to confirm before mixing.

Related: [[Sourdough Starter]] · [[Dough Hydration]]

#baking #fermentation #timing
```
(After querying the graph and finding `[[Sourdough Starter]]` and `[[Dough Hydration]]`
already exist — so the new note connects instead of floating.)

## Writing it back — the real interface

QuickPeach exposes note writes through the plugin SDK:

```ts
sdk.notes.list(): Promise<{ id, title, updated_at }[]>          // discover link targets
sdk.notes.create(title: string, content?: string): Promise<{ noteId: string }>
sdk.notes.open(noteId: string): Promise<void>                   // surface it after creating
```

- `create` takes a **title** and a **markdown `content`** string — the `[[wikilinks]]` and
  `#tags` go *inside* `content`; there are no separate link/tag arguments. So assemble the
  full markdown (title can be the `# Heading` line and/or the `title` arg) and pass it.
- It requires manifest capability **`notes`** + permission **`notes-write`** (and `notes-read`
  for `list`, `navigate` for `open`). Declare them or the call is denied.
- `create` is a **mutation**, so in QuickPeach's agent runtime it runs through the
  approval-gated control path (read-only graph queries run immediately; writes ask first).
  That's expected — surface a clear title/preview so the user can approve confidently.
- Never invent file paths or touch storage directly — `sdk.notes.create` is the only
  supported way in; the vault is encrypted and host-managed.

```ts
// after finding [[Sourdough Starter]] + [[Dough Hydration]] via the graph / sdk.notes.list():
const md = `# Levain timing — 6h at 26°C\n\nPeaked at ~6h at 26°C. Float-test before mixing.\n\nRelated: [[Sourdough Starter]] · [[Dough Hydration]]\n\n#baking #fermentation #timing`;
const { noteId } = await sdk.notes.create("Levain timing — 6h at 26°C", md);
await sdk.notes.open(noteId);
```
