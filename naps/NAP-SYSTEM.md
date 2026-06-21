# NAP-SYSTEM

## Read-Only Runtime System Information

`draft`

**NAP ID:** NAP-SYSTEM
**Domain:** `system`
**Web binding (NIP-5D):** `window.napplet.system` · `shell.supports("system")`

## Description

NAP-SYSTEM gives a napplet read-only access to runtime status snapshots. It is
for diagnostics, settings panes, support views, and adaptive UI. It reports what
the runtime knows about NAP support, services, relays, caches, storage surfaces,
media surfaces, and napplet-scoped resources.

The runtime owns aggregation. A napplet asks a narrow accessor for one category;
the runtime decides which internal sources to query, how fresh the answer is,
and which details are safe to reveal. NAP-SYSTEM does not grant access to any
capability it reports.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `naps` | none | list of `SystemNapStatus` | `system.naps` / `system.naps.result` |
| `services` | none | list of `SystemServiceStatus` | `system.services` / `system.services.result` |
| `relays` | none | list of `SystemRelayStatus` | `system.relays` / `system.relays.result` |
| `eventCache` | none | `SystemStorageStatus` | `system.eventCache` / `system.eventCache.result` |
| `localStorage` | none | `SystemStorageStatus` | `system.localStorage` / `system.localStorage.result` |
| `indexedDb` | none | `SystemStorageStatus` | `system.indexedDb` / `system.indexedDb.result` |
| `media` | none | `SystemMediaStatus` | `system.media` / `system.media.result` |
| `nappletStorage` | none | `SystemStorageStatus` | `system.nappletStorage` / `system.nappletStorage.result` |
| `scopes` | none | list of `SystemScopeSummary` | `system.scopes` / `system.scopes.result` |
| `scope` | `name` (`tstr`) | `SystemScopeStatus` | `system.scope` / `system.scope.result` |

### Schemas

```cddl
SystemHealth = "ok" / "degraded" / "unavailable" / "unknown"

SystemNapStatus = {
  domain: tstr,
  supported: bool,
  ? required: bool,
  ? protocols: [* tstr],
  ? health: SystemHealth,
}

SystemServiceStatus = {
  name: tstr,
  available: bool,
  health: SystemHealth,
  ? message: tstr,
}

SystemRelayStatus = {
  url: tstr,
  read: bool,
  write: bool,
  connected: bool,
  health: SystemHealth,
  ? latencyMs: number,
}

SystemStorageStatus = {
  available: bool,
  health: SystemHealth,
  ? bytesUsed: uint,
  ? bytesQuota: uint,
  ? itemCount: uint,
  ? persistent: bool,
  ? message: tstr,
}

SystemMediaStatus = {
  available: bool,
  health: SystemHealth,
  ? audioOutput: SystemHealth,
  ? audioInput: SystemHealth,
  ? videoInput: SystemHealth,
  ? playback: SystemHealth,
  ? activeSessions: uint,
  ? message: tstr,
}

SystemScopeSummary = {
  name: tstr, ; e.g. "storage", "media", "relay", "cache"
  available: bool,
  health: SystemHealth,
}

SystemScopeStatus = {
  name: tstr,
  available: bool,
  health: SystemHealth,
  details: { * tstr => any },
}
```

Each method returns a snapshot. The runtime MAY cache, redact, or synthesize
fields. Omitted optional fields mean "not reported", not zero.

**`naps()`** returns NAP support visible to this napplet. It MAY include global
runtime support, per-napplet grants, and numbered protocol support.

**`services()`** returns named runtime service availability and health. Service
names are runtime-defined and need not map one-to-one to NAP domains.

**`relays()`** returns connected relay status. It reports relay transport health,
not user relay preferences. User relay preferences belong to NAP-IDENTITY.

**`eventCache()`** returns runtime event-cache details, such as approximate size,
quota, persistence, and health.

**`localStorage()`** returns runtime local-storage details. In a web projection
this may describe browser `localStorage`; in another projection it may describe
the runtime's equivalent local key-value surface.

**`indexedDb()`** returns runtime IndexedDB details. In a non-web projection this
MAY describe the runtime's equivalent indexed object store, or return
`available: false`.

**`media()`** returns runtime media status: playback, capture availability,
active sessions, and health. It does not grant camera, microphone, screen, or
playback control.

**`nappletStorage()`** returns storage details scoped to the requesting napplet,
such as NAP-STORAGE usage, quota, persistence, and health.

**`scopes()`** lists napplet-scoped diagnostic areas available through
`scope(name)`.

**`scope(name)`** returns details scoped to this napplet. Examples include
NAP-STORAGE usage for the current napplet, per-napplet media sessions, scoped
relay budgets, or scoped cache entries. The `details` object is runtime-defined;
the stable contract is the accessor, scope name, and read-only status envelope.

## Global vs Scoped

Global accessors describe runtime-wide systems as redacted for this napplet:

| Accessor | Reports |
|----------|---------|
| `naps()` | NAP support, grants, and numbered protocol support |
| `services()` | runtime service availability and health |
| `relays()` | connected relay health |
| `eventCache()` | event-cache health, usage, and persistence |
| `localStorage()` | local-storage health and usage |
| `indexedDb()` | IndexedDB / indexed-store health and usage |
| `media()` | media subsystem health and active session count |

Scoped accessors describe only resources attributable to the requesting napplet:

| Accessor | Reports |
|----------|---------|
| `nappletStorage()` | NAP-STORAGE-style usage for this napplet |
| `scopes()` | which scoped areas can be queried |
| `scope(name)` | runtime-defined details for one scoped area |

The runtime MUST NOT expose another napplet's scoped details through
NAP-SYSTEM.

## Wire Protocol

`system.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `system.naps` | napplet -> runtime | `id` |
| `system.naps.result` | runtime -> napplet | `id`, `naps?`, `error?` |
| `system.services` | napplet -> runtime | `id` |
| `system.services.result` | runtime -> napplet | `id`, `services?`, `error?` |
| `system.relays` | napplet -> runtime | `id` |
| `system.relays.result` | runtime -> napplet | `id`, `relays?`, `error?` |
| `system.eventCache` | napplet -> runtime | `id` |
| `system.eventCache.result` | runtime -> napplet | `id`, `status?`, `error?` |
| `system.localStorage` | napplet -> runtime | `id` |
| `system.localStorage.result` | runtime -> napplet | `id`, `status?`, `error?` |
| `system.indexedDb` | napplet -> runtime | `id` |
| `system.indexedDb.result` | runtime -> napplet | `id`, `status?`, `error?` |
| `system.media` | napplet -> runtime | `id` |
| `system.media.result` | runtime -> napplet | `id`, `status?`, `error?` |
| `system.nappletStorage` | napplet -> runtime | `id` |
| `system.nappletStorage.result` | runtime -> napplet | `id`, `status?`, `error?` |
| `system.scopes` | napplet -> runtime | `id` |
| `system.scopes.result` | runtime -> napplet | `id`, `scopes?`, `error?` |
| `system.scope` | napplet -> runtime | `id`, `name` |
| `system.scope.result` | runtime -> napplet | `id`, `scope?`, `error?` |

Key design notes:
- Every operation is read-only.
- Request/result pairs use `id` for correlation.
- There is no push surface. Napplets that need fresh status call the accessor
  again.
- The runtime MAY redact, bucket, or omit sensitive fields.
- The runtime MAY return stale-but-useful status if live probing is expensive.

### Examples

**NAP support:**

```
-> { "type": "system.naps", "id": "n1" }
<- {
     "type": "system.naps.result",
     "id": "n1",
     "naps": [
       { "domain": "shell", "supported": true, "required": true, "health": "ok" },
       { "domain": "storage", "supported": true, "health": "ok" },
       { "domain": "inc", "supported": true, "protocols": ["NAP-4"], "health": "ok" }
     ]
   }
```

**Relay and event-cache status:**

```
-> { "type": "system.relays", "id": "r1" }
<- {
     "type": "system.relays.result",
     "id": "r1",
     "relays": [
       { "url": "wss://relay.example.com", "read": true, "write": true, "connected": true, "health": "ok", "latencyMs": 82 }
     ]
   }

-> { "type": "system.eventCache", "id": "c1" }
<- {
     "type": "system.eventCache.result",
     "id": "c1",
     "status": { "available": true, "health": "ok", "bytesUsed": 7340032, "persistent": true }
   }
```

**Scoped storage status for this napplet:**

```
-> { "type": "system.nappletStorage", "id": "s1" }
<- {
     "type": "system.nappletStorage.result",
     "id": "s1",
     "status": { "available": true, "health": "ok", "bytesUsed": 4096, "itemCount": 12 }
   }

-> { "type": "system.scope", "id": "s2", "name": "media" }
<- {
     "type": "system.scope.result",
     "id": "s2",
     "scope": {
       "name": "media",
       "available": true,
       "health": "ok",
       "details": { "activeSessions": 1 }
     }
   }
```

## Error Handling

Result messages include an `error` string when the runtime cannot fulfill the
request. Common errors: `"unsupported"`, `"policy denied"`, `"unavailable"`,
`"unknown scope"`, and `"probe failed"`.

When a result includes `error`, it SHOULD still include a conservative fallback
when possible, such as `available: false` with `health: "unknown"`.

## Runtime Behavior

- The runtime MUST respond to every request with a result message carrying the
  same `id`.
- The runtime MUST NOT mutate runtime state to answer a system accessor, except
  for incidental measurement work required to produce the snapshot.
- The runtime MUST aggregate system details itself. Napplets MUST NOT be
  required to call lower-level NAPs to assemble a system view.
- The runtime MUST scope `scope(name)` to the requesting napplet.
- The runtime MUST NOT expose another napplet's scoped storage, cache, media,
  relay, or service details.
- The runtime MAY redact URLs, counts, byte totals, service names, latency, or
  health messages by policy.
- The runtime MAY rate-limit system accessors.
- The runtime MAY return approximate values.

## Security Considerations

System information is a fingerprinting and privacy surface. NAP-SYSTEM is
read-only, but it can reveal runtime topology, relay choices, storage size,
media availability, and service health.

- The runtime SHOULD expose the least detail needed for the requesting napplet.
- The runtime SHOULD bucket or omit exact byte counts and latency values when
  exact values are not needed.
- The runtime MUST NOT expose private event contents, storage keys, stored
  values, media metadata, raw IndexedDB data, raw localStorage data, or another
  napplet's scoped data.
- The runtime MUST NOT treat `system.naps` as an authorization mechanism.
  Capability authorization remains with NAP-SHELL and runtime policy.
- Napplets MUST treat system snapshots as advisory. A status can change
  immediately after it is read.

## Implementations

- None yet.
