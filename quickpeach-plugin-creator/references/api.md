# QuickPeach extension API — full reference

Codebase-accurate (matches `src/types/extensions.ts` + `src/features/extensions/plugin-sdk.ts`).
Cross-check drift with `chub get pantakan/quickpeach-extensions --lang ts`.

## Manifest fields

| Field | Type | Notes |
|-------|------|-------|
| `schemaVersion` | `1` | always 1 |
| `id` | string | kebab-case, unique |
| `name` / `description` | string | display name + one-liner |
| `version` | string | semver |
| `author` | string | optional |
| `icon` | string | named icon (e.g. `"sparkle"`, `"dashboard"`) |
| `capabilities` | `ExtensionCapability[]` | feature surfaces used (see below) |
| `permissions` | `ExtensionPermission[]` | bridge calls allowed (see below) |
| `commands` | `Command[]` | static launcher commands |
| `preferences` | `Preference[]` | user-config fields; `secret:true` → secrets lane |
| `platform` | object | `launcher`, `views[]`, `overlayFamilies`, `dashboardWidgets`, `settingsSections` |

**Command**: `{ id, name, description, keywords: string[], action? }`.
`action` (optional, zero-code): `{ type: "open-url", url }` where `url` may use placeholders
`__APP_DATA__`, `__EXTENSIONS_DIR__`, `__QUICKPEACH_ROOT__`, `__PLUGIN_HOST_ROOT__`. Omit `action`
to handle the command inside your launcher view.

**Preference**: `{ key, label, type: "string"|"boolean"|"number"|"enum", secret?, options? }`.

## Capabilities (exact union — 13)

`launcher-commands`, `clipboard`, `browser`, `network`, `ai`, `calendar`, `notes`, `storage`,
`secrets`, `sync-state`, `preferences`, `search-files`, `events`

## Permissions (exact union — 23)

`clipboard-read`, `clipboard-write`, `clipboard-history`, `browser-open`, `browser-search`,
`network-fetch`, `ai-invoke`, `calendar-read`, `calendar-write`, `calendar-sync`, `notes-read`,
`notes-write`, `storage-read`, `storage-write`, `secrets-read`, `secrets-write`, `sync-state-read`,
`sync-state-write`, `preferences-read`, `search-files`, `toast`, `navigate`, `events`

### Capability → permissions it gates

| Capability | Permissions |
|-----------|-------------|
| `launcher-commands` | (commands; `navigate` for `sdk.ui.navigate`) |
| `clipboard` | `clipboard-read`, `clipboard-write`, `clipboard-history` |
| `browser` | `browser-open`, `browser-search` |
| `network` | `network-fetch` |
| `ai` | `ai-invoke` |
| `calendar` | `calendar-read`, `calendar-write`, `calendar-sync` |
| `notes` | `notes-read`, `notes-write` |
| `storage` | `storage-read`, `storage-write` |
| `secrets` | `secrets-read`, `secrets-write` |
| `sync-state` | `sync-state-read`, `sync-state-write` |
| `preferences` | `preferences-read` |
| `search-files` | `search-files` |
| `events` | `events` |
| (`sdk.ui.toast`) | `toast` |

## SDK API (`createPluginSdk(extensionId)`)

```ts
import { createPluginSdk } from "@peachnote/plugin-sdk";
const sdk = createPluginSdk(window.__QUICKPEACH_PLUGIN_CONTEXT__.extensionId);
```

### sdk.ui
```ts
toast(message: string, level?: "info"|"success"|"warning"|"error"): Promise<void>   // perm: toast
navigate(view: "home"|"notes"|"settings"|"dashboard"): Promise<void>                 // perm: navigate
```

### sdk.launcher
```ts
currentQuery(): PluginLauncherQuery | null
onQuery(cb: (q: PluginLauncherQuery) => void): Unsubscribe
returnItems(items: PluginLauncherListItem[]): void
onItemSelect(cb: (sel: PluginLauncherSelection) => void): Unsubscribe
// PluginLauncherQuery = { queryId, rawQuery, body, prefix? }
// PluginLauncherListItem = { id, title, subtitle?, icon?, copyText? }
```

### sdk.windows
```ts
current(): PluginWindowContext | null
openWorkspace(screen: string): Promise<void>
openUtility(tool: string): Promise<string>
openOverlay(target: { namespace, ownerId, viewId, params? }): Promise<string>
hideOverlay(namespace: string, windowLabel: string): Promise<void>
```

### sdk.network   (perm: network-fetch)
```ts
fetch<T>(url, options?: { method?, headers?, body? }): Promise<PluginFetchResponse<T>>
provider(providerId): { get<T>(path,opts?), post<T>(path,body?,opts?), request<T>(path,opts?) }
// PluginFetchResponse = { url, ok, status, headers, bodyText, json }
```

### sdk.notes   (perm: notes-read / notes-write)
```ts
list(): Promise<{ id, title, updated_at }[]>            // notes-read
create(title: string, content?: string): Promise<{ noteId: string }>  // notes-write
open(noteId: string): Promise<void>                     // navigate
```

### sdk.storage   (perm: storage-read / storage-write)
```ts
get<T>(key): Promise<T|null>;  keys(prefix?): Promise<string[]>;  set(key,value): Promise<void>;  remove(key): Promise<void>
blobs.get(path): Promise<{ metadata, bytes: Uint8Array }|null>
blobs.list(prefix?): Promise<BlobMetadata[]>;  blobs.stat(path): Promise<BlobMetadata|null>
blobs.put(path, input, { contentType?, syncClass? }): Promise<BlobMetadata>   // syncClass: "local-only"|"cloud-replicated"|"cache"
blobs.remove(path): Promise<boolean>
```

### sdk.secrets   (perm: secrets-read / secrets-write) — XChaCha20-Poly1305, per-extension
```ts
get<T>(key): Promise<T|null>; keys(prefix?): Promise<string[]>; set(key,value): Promise<void>; remove(key): Promise<void>
```

### sdk.settings   (perm: preferences-read) — reads manifest-declared preferences
```ts
get<T>(key): Promise<T|null>;  onChange<T>(key, cb): Unsubscribe
```

### sdk.syncState   (perm: sync-state-read / sync-state-write) — synced across devices
```ts
get<T>(key); keys(prefix?); set(key,value); remove(key); onChange<T>(key, cb): Unsubscribe
```

### sdk.clipboard   (perm: clipboard-*)
```ts
read(): Promise<string>;  write(text): Promise<void>;  history(limit?): Promise<string[]>
```

### sdk.browser   (perm: browser-open / browser-search)
```ts
open(url): Promise<void>;  search(query, provider?): Promise<void>
```

### sdk.calendar   (perm: calendar-read / calendar-write / calendar-sync)
```ts
listConnectors(); listSources(); upsertSource(input); removeSource(id); sync(sourceIds?)
listEvents(query?: { sourceIds?, from?, to?, query?, limit? }): Promise<CalendarEvent[]>
```

### sdk.events   (perm: events)
```ts
emit(event, payload?): Promise<void>;  on<T>(event, cb): Unsubscribe
```

### sdk.crypto   (no permission — pure local primitives)
```ts
sha256Hex(input); sha256Base64(input); randomUUID(); randomHex(bytesLength?)
seal(scope, input, aad?): Promise<CryptoEnvelope>;  open(scope, envelope, aad?): Promise<Uint8Array>
```

### sdk.ai   (perm: ai-invoke)
```ts
invoke(prompt: string, model?: string): Promise<string>
```

## View mount types + declaration

| Mount | Declare in `platform.` | Open via |
|-------|------------------------|----------|
| `launcher` | `launcher` (prefix, viewId, `dynamic:true`) + `views[]` | command palette `@prefix` |
| `workspace` | `views[]` (`mount:"workspace"`) | `sdk.windows.openWorkspace(screen)` |
| `overlay` | `overlayFamilies` (multiInstance, alwaysOnTop, size) + `views[]` | `sdk.windows.openOverlay({...})` |
| `dashboard-widget` | `dashboardWidgets` (slot, position main/side/bottom, priority, minHeight) | dashboard |
| `settings-section` | `settingsSections` (field defs: string/boolean/number/enum) | Settings |

`views[]` entry: `{ viewId, entry: "dist/x.js", exportName, mount }`.

## Entry mount signature
```ts
export default async function mount({ root, mount, extensionId, context }) { ... }
// resolution order: exportName → default → mount → export named like viewId
// context: PluginWindowContext { bridgeToken, viewId, props, ... }
```

## Security model
- iframe sandbox: `allow-scripts allow-forms` only.
- CSP: `connect-src 'none'` (no direct net — use `sdk.network.fetch`), no external scripts, no nested iframes.
- Per-extension scoped storage; encrypted secrets; asset paths confined to extension root.
- Session bridge tokens validated per request; every bridge call checked against manifest `permissions`.

## Local dev
1. `manifest.json` + build to `dist/`.
2. Settings → Extensions → "Install from folder". No restart.
3. Installed at `~/.quickpeach/extensions/<id>/`. Edit → rebuild → reload iframe.
