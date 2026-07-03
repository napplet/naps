NAP-STORAGE
===========

Scoped Key-Value Storage
------------------------

`draft`

**NAP ID:** NAP-STORAGE
**Domain:** `storage`
**Web binding (NIP-5D):** `window.napplet.storage` · `shell.supports("storage")`

## Description

NAP-STORAGE provides napplets with an async localStorage-like API. Without `allow-same-origin`, iframes have opaque origins and cannot access localStorage directly. This interface routes storage operations through the shell via postMessage, which scopes data by napplet identity — a composite key of `(dTag, aggregateHash)` — to enforce isolation between napplets. Different napplet types and different versions of the same napplet have completely separate storage namespaces.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `getItem` | `key` (`tstr`) | `tstr` or `null` | `storage.get` / `storage.get.result` |
| `setItem` | `key` (`tstr`), `value` (`tstr`) | none | `storage.set` / `storage.set.result` |
| `removeItem` | `key` (`tstr`) | none | `storage.remove` / `storage.remove.result` |
| `keys` | none | list of `tstr` keys | `storage.keys` / `storage.keys.result` |
| `instance.getItem` | `key` (`tstr`) | `tstr` or `null` | `storage.get` / `storage.get.result` with `scope: "instance"` |
| `instance.setItem` | `key` (`tstr`), `value` (`tstr`) | none | `storage.set` / `storage.set.result` with `scope: "instance"` |
| `instance.removeItem` | `key` (`tstr`) | none | `storage.remove` / `storage.remove.result` with `scope: "instance"` |
| `instance.keys` | none | list of `tstr` keys | `storage.keys` / `storage.keys.result` with `scope: "instance"` |

**`getItem(key)`** — Retrieves a stored value by key. Returns `null` if the key does not exist, matching localStorage semantics.

**`setItem(key, value)`** — Stores a key-value pair. Throws if the napplet exceeds its storage quota.

**`removeItem(key)`** — Deletes a stored key. No-op if the key does not exist.

**`keys()`** — Lists all keys stored by this napplet.

All methods are async because they cross the postMessage boundary. Each request includes a correlation ID (`id`); the shell responds with a matching ID so the napplet can resolve the correct Promise.

**`instance`** — The same four methods, scoped to the *calling instance* instead of shared across all instances of the napplet. A napplet open more than once (e.g. several feeds) uses `instance.*` for per-window state and the top-level methods for shared state. Scope is per call: instancing is a property of the data, not the napplet. See [Instance Scope](#instance-scope).

## Wire Protocol

Storage operations use the NIP-5D wire format. Each request includes an `id` field for correlation.

Every request carries an optional `scope`: `"shared"` (default) or `"instance"`. It is the only wire-level addition for per-instance storage; the `instance.*` API is sugar that sets `scope: "instance"`.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `storage.get` | napplet -> shell | `id`, `key`, `scope?` |
| `storage.set` | napplet -> shell | `id`, `key`, `value`, `scope?` |
| `storage.remove` | napplet -> shell | `id`, `key`, `scope?` |
| `storage.keys` | napplet -> shell | `id`, `scope?` |
| `storage.get.result` | shell -> napplet | `id`, `value` (string or null) |
| `storage.set.result` | shell -> napplet | `id` |
| `storage.remove.result` | shell -> napplet | `id` |
| `storage.keys.result` | shell -> napplet | `id`, `keys` (string array) |

Result messages carry no `scope`; the correlation `id` already identifies the request.

### Examples

**Get:**
```
-> { "type": "storage.get", "id": "a1", "key": "theme" }
<- { "type": "storage.get.result", "id": "a1", "value": "dark" }
```

If the key does not exist, `value` is `null`:
```
-> { "type": "storage.get", "id": "a2", "key": "missing-key" }
<- { "type": "storage.get.result", "id": "a2", "value": null }
```

**Set:**
```
-> { "type": "storage.set", "id": "b2", "key": "theme", "value": "light" }
<- { "type": "storage.set.result", "id": "b2" }
```

**Remove:**
```
-> { "type": "storage.remove", "id": "c3", "key": "theme" }
<- { "type": "storage.remove.result", "id": "c3" }
```

**Keys:**
```
-> { "type": "storage.keys", "id": "d4" }
<- { "type": "storage.keys.result", "id": "d4", "keys": ["theme", "language"] }
```

Any request MAY add `"scope": "instance"` to address the calling instance's namespace instead of the shared one.

### Error Handling

Any result message MAY include an `error` field (string). When `error` is present, other result fields are undefined.

```
<- { "type": "storage.set.result", "id": "b2", "error": "quota exceeded" }
```

## Instance Scope

When a napplet is open more than once (e.g. several feeds), `scope: "instance"`
isolates a key to the calling instance while `scope: "shared"` (the default)
stays common to every instance. The choice is per call, so per-instance state
(criteria, scroll) and shared state (theme, preferences) coexist without a
napplet-wide mode.

The instance namespace is defined by two guarantees:

- **Unique** — instances MUST NOT share an instance namespace or read each
  other's instance-scoped keys.
- **Stable** — the same instance MUST resolve to the same namespace across
  reloads and sessions, for as long as it exists.

How the shell identifies an instance and keeps it stable is **unspecified** —
the shell's concern. A napplet never sees or names an instance identifier; it
sets `scope` and trusts the two guarantees.

**Lifetime.** Instance storage lives as long as the instance; the shell MAY
reclaim it on destroy. State that must outlive the instance belongs in `shared`.

## Shell Behavior

- The shell MUST scope storage by composite key `(dTag, aggregateHash)`. Different napplet types and different versions of the same napplet MUST have isolated storage. The shell maps each napplet's `(dTag, aggregateHash)` identity to an isolated storage namespace.
- For `scope: "instance"` requests, the shell MUST further isolate the namespace per instance, satisfying the Unique and Stable guarantees above. Requests with `scope` absent or `"shared"` MUST address the napplet-wide namespace, preserving the behavior of napplets that never set `scope`.
- The shell MUST enforce a per-napplet storage quota. A recommended default is 512 KB measured in UTF-8 byte count. Whether instance namespaces draw from the same per-napplet budget or a per-instance budget is an implementation detail.
- The shell MUST persist storage so data survives page reloads. How the shell stores data (localStorage, IndexedDB, etc.) is an implementation detail.
- The shell MUST respond to every request with a result message carrying the same `id`.
- The shell MUST return an `error` field in the result message for quota violations and invalid requests.

## Security Considerations

- Storage is scoped by composite key `(dTag, aggregateHash)`. The shell MUST enforce isolation so a napplet cannot read or write another napplet's data. For `scope: "instance"`, the shell MUST additionally enforce isolation between instances of the same napplet.
- Storage quota enforcement prevents a single napplet from consuming unbounded host storage.
- Storage values are strings only. The shell SHOULD NOT attempt to parse or execute stored content.
- The shell MAY enforce ACL checks on `storage:read` and `storage:write` capabilities before processing storage requests.
- Storage keys and values are visible to the shell. Napplets that need confidential storage should encrypt values before storing.

## Changelog

- `fe4a523` - Introduced NAP-STORAGE for scoped key-value storage.
