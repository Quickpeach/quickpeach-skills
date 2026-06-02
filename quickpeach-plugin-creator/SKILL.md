---
name: quickpeach-plugin-creator
description: >-
  Build, scaffold, or debug a QuickPeach extension / plugin. Use this WHENEVER the user wants to
  make a QuickPeach plugin or extension, add a launcher command / overlay window / dashboard widget
  / settings section, expose data through the plugin SDK, or asks about the extension manifest,
  capabilities, permissions, view mounts, or the sandbox/security model. Extensions are sandboxed
  iframes that reach the host ONLY through a permissioned SDK bridge — get the manifest, capability,
  and permission names exact (see references/api.md) instead of guessing; a wrong permission means a
  silent denied bridge call.
---

# Build a QuickPeach extension

A QuickPeach extension is an **ES module loaded into a sandboxed iframe** that adds UI surfaces
(launcher commands, overlay windows, dashboard widgets, settings sections, workspace screens) and
talks to the host **only** through `@peachnote/plugin-sdk`. Every SDK call is checked against the
`permissions` you declared in the manifest — there is no filesystem or direct-network escape hatch.

This SKILL.md is the working guide. For the **exhaustive, codebase-accurate** API — the full SDK
namespaces with signatures, the complete capability/permission lists, command-action types, view-mount
declarations, and placeholder variables — read [`references/api.md`](references/api.md). For anything
that may have changed, cross-check with `chub get pantakan/quickpeach-extensions --lang ts`.

## Scaffold

```
my-extension/
  manifest.json     # identity + capabilities + permissions + commands + platform surfaces
  src/index.ts      # entry: a mount() function
  dist/view.js      # build output (the manifest points views[].entry here)
  package.json      # optional (TypeScript / Vite / esbuild → dist/)
```

## Manifest — the trust contract

```json
{
  "schemaVersion": 1,
  "id": "my-extension",
  "name": "My Extension",
  "description": "One line.",
  "version": "0.1.0",
  "author": "You",
  "icon": "sparkle",
  "capabilities": ["launcher-commands", "notes"],
  "permissions": ["notes-read", "notes-write", "toast"],
  "commands": [
    { "id": "do-it", "name": "Do It", "description": "…", "keywords": ["go"],
      "action": { "type": "open-url", "url": "__APP_DATA__/notes" } }
  ],
  "preferences": [
    { "key": "apiKey", "label": "API Key", "type": "string", "secret": true }
  ],
  "platform": {
    "launcher": { "prefix": "@mine", "placeholder": "Search…", "viewId": "launcher-view", "dynamic": true },
    "views": [{ "viewId": "launcher-view", "entry": "dist/view.js", "exportName": "LauncherView", "mount": "launcher" }]
  }
}
```

**Two-layer gate — get this right or calls silently fail:**
- **`capabilities`** = which feature *surfaces*/namespaces the extension uses (e.g. `notes`, `network`).
- **`permissions`** = which specific *bridge calls* are allowed (e.g. `notes-read`, `network-fetch`).

Pair them: to call `sdk.notes.create(...)` you need capability `notes` **and** permission `notes-write`.
Request the **fewest** that work — the install dialog shows them to the user; over-asking erodes trust
and the host rejects undeclared calls at runtime.

Capability list (13) and permission list (23): see [`references/api.md`](references/api.md) — they are the
exact `ExtensionCapability` / `ExtensionPermission` unions from the app, so copy names verbatim.

Static `commands[].action` can be `open-url` (with placeholders `__APP_DATA__`, `__EXTENSIONS_DIR__`,
`__QUICKPEACH_ROOT__`, `__PLUGIN_HOST_ROOT__`) for zero-code commands; omit `action` to handle the
command in your launcher view instead.

## Entry point protocol

```ts
import { createPluginSdk } from "@peachnote/plugin-sdk";

export default async function mount({ root, mount, extensionId, context }) {
  const sdk = createPluginSdk(extensionId);   // or window.__QUICKPEACH_PLUGIN_CONTEXT__.extensionId
  mount.innerHTML = `<div id="app">Hello</div>`;
  await sdk.ui.toast("loaded", "success");
}
```

The host resolves the mount function in this order: **named export matching `exportName` → `default`
→ `mount` → export named after `viewId`.** `context` is the `PluginWindowContext` (bridgeToken, viewId,
props…). Drive *everything* host-side through `sdk.*`.

## Goal → SDK (the 80%)

| Goal | SDK |
|------|-----|
| Toast / navigate | `sdk.ui.toast(msg, level)` · `sdk.ui.navigate(view)` |
| Dynamic launcher results | `sdk.launcher.onQuery(cb)` → `sdk.launcher.returnItems(items)` |
| Open a window | `sdk.windows.openWorkspace/openUtility/openOverlay/hideOverlay` |
| HTTP (only path out) | `sdk.network.fetch(url, opts)` · `sdk.network.provider(id)` |
| Read/create notes | `sdk.notes.list()` · `sdk.notes.create(title, content?)` · `sdk.notes.open(id)` |
| Per-extension KV / blobs | `sdk.storage.{get,set,keys,remove}` · `sdk.storage.blobs.*` |
| Encrypted secrets | `sdk.secrets.{get,set,keys,remove}` |
| User preferences | `sdk.settings.get(key)` · `sdk.settings.onChange(key, cb)` |
| Cross-device synced KV | `sdk.syncState.*` |
| Clipboard / browser / calendar / AI / crypto / events | see references/api.md |

## View mount types

`launcher` (palette results, `dynamic:true`) · `workspace` (full screen, `openWorkspace`) · `overlay`
(floating window, declare `platform.overlayFamilies`, `openOverlay`) · `dashboard-widget` (declare
`platform.dashboardWidgets` with slot/position/priority) · `settings-section` (auto-rendered form,
declare `platform.settingsSections` with field defs). Details + declaration shapes in references/api.md.

## Security model (don't fight it)

- iframe sandbox = `allow-scripts allow-forms`; CSP `connect-src 'none'` → **all** network goes through
  `sdk.network.fetch`; no external scripts, no nested iframes.
- Storage is scoped per extension; secrets encrypted at rest (XChaCha20-Poly1305); asset paths confined
  to the extension root; bridge tokens validated per request; every bridge call permission-checked.
- So: never hard-code secrets (`preferences` with `secret:true` → `sdk.secrets`), never assume host
  internals, never try to reach the filesystem/network outside the SDK. These are the rules, not bugs.

## Local development

1. `manifest.json` + build to `dist/`.
2. QuickPeach → **Settings → Extensions → "Install from folder"** → pick the folder. Appears
   immediately, no restart. Installed copy lives at `~/.quickpeach/extensions/<id>/`.
3. Edit → rebuild → reload the iframe (devtools F5 / reopen the window).

## Workflow

1. **Decide the surface** (launcher command? overlay? widget? settings? workspace?) → set the matching
   `capabilities` + `platform` entry.
2. **List the bridge calls** you'll make → declare exactly those `permissions` (pair with capabilities).
3. **Write `mount`**, route all host access through `sdk.*`, and handle denied/missing-permission paths.
4. **Build → install from folder → iterate.** Confirm each `sdk.*` call's permission is declared if it
   no-ops.
5. If unsure of an exact name/signature, read [`references/api.md`](references/api.md) — don't guess.
