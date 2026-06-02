---
name: quickpeach-plugin-creator
description: >-
  Build, scaffold, or debug a QuickPeach extension / plugin. Use this WHENEVER the user wants
  to make a QuickPeach plugin or extension, add a launcher command / overlay window / dashboard
  widget / settings section to QuickPeach, or asks about the extension manifest, plugin SDK,
  view mounts, or permissions. Always pull the full API reference with
  `chub get pantakan/quickpeach-extensions --lang ts` before writing real code — don't guess
  the manifest or SDK shape.
---

# Build a QuickPeach extension

QuickPeach extensions are sandboxed iframe plugins that add surfaces (launcher commands,
overlay windows, dashboard widgets, settings sections) and talk to the host through a
permissioned SDK bridge. This skill orients you; the **authoritative API reference** is
`chub get pantakan/quickpeach-extensions --lang ts` — fetch it before writing code so the
manifest schema, SDK methods, and permission names are exact (they evolve).

## Scaffold

```
my-extension/
  manifest.json     # identity, capabilities, permissions, commands, surfaces
  src/index.ts      # entry point (mount fn)
  dist/             # build output
  package.json      # optional build tooling
```

## Manifest (the contract)

```json
{
  "schemaVersion": 1,
  "id": "my-extension",
  "name": "My Extension",
  "version": "0.1.0",
  "capabilities": ["launcher-commands"],
  "permissions": ["toast"],
  "commands": [
    { "id": "hello", "name": "Say Hello", "keywords": ["greet"] }
  ],
  "platform": {
    "launcher": { "prefix": "@mine", "placeholder": "Search…", "viewId": "launcher-view", "dynamic": true }
  }
}
```

- **`capabilities`** declare what surfaces you use; **`permissions`** declare what bridge
  calls you may make. Request the **fewest** that work — the user sees these at install,
  and the host enforces them. (Full lists: the chub reference.)
- Surfaces live under `platform`: `launcher` (a `@prefix` command view), `views`
  (overlay windows), dashboard widgets, settings sections.
- Secrets go in `preferences` with `"secret": true` (stored in the host secrets lane),
  never hard-coded.

## Entry point + SDK

```ts
import { createPluginSdk } from "@peachnote/plugin-sdk";

const ctx = window.__QUICKPEACH_PLUGIN_CONTEXT__;
const sdk = createPluginSdk(ctx.extensionId);

export default async function mount({ mount }) {
  mount.innerHTML = "<p>Extension loaded!</p>";
  await sdk.ui.toast("Hello from my extension!", "success");
}
```

The SDK is the **only** way to touch the host (toasts, storage, windows, clipboard, …),
each call gated by a declared permission. There is no direct filesystem / network escape
hatch from the iframe — that's the security model, not a limitation to work around.

## Workflow

1. **Fetch the reference first:** `chub get pantakan/quickpeach-extensions --lang ts`
   (manifest schema, full SDK API, view mount types, permission catalog, security model).
2. **Pick the surface** (launcher command? overlay window? dashboard widget? settings?)
   and set `capabilities` + matching `platform` entry.
3. **Declare least-privilege `permissions`** — only what your SDK calls need.
4. **Write `mount`**, drive everything through `sdk.*`, handle the no-permission /
   denied case gracefully.
5. **Test locally** via QuickPeach's Local Dev install (install-from-folder / from-zip),
   then iterate.

## Don'ts

- Don't hard-code secrets — use `preferences` (`secret: true`).
- Don't request broad permissions "just in case" — the install dialog shows them; trust is
  the marketplace currency.
- Don't reach outside the SDK bridge or assume host internals — pin to the documented API.
