NAP-STORAGE
===========

Scoped Key-Value Storage
------------------------

`draft`

**NAP ID:** NAP-STORAGE
**Namespace:** `window.napplet.storage`
**Discovery:** `shell.supports("storage")`

## Description

NAP-STORAGE provides napplets with an async localStorage-like API. Without `allow-same-origin`, iframes have opaque origins and cannot access localStorage directly. This interface routes storage operations through the shell via postMessage, which scopes data by napplet identity — a composite key of `(dTag, aggregateHash)` — to enforce isolation between napplets. Different napplet types and different versions of the same napplet have completely separate storage namespaces.

## API Surface

```typescript
interface NappletStorage {
  getItem(key: string): Promise<string | null>;    // via storage.get
  setItem(key: string, value: string): Promise<void>; // via storage.set
  removeItem(key: string): Promise<void>;           // via storage.remove
  keys(): Promise<string[]>;                        // via storage.keys
}
```

**`getItem(key)`** — Retrieves a stored value by key. Returns `null` if the key does not exist, matching localStorage semantics.

**`setItem(key, value)`** — Stores a key-value pair. Throws if the napplet exceeds its storage quota.

**`removeItem(key)`** — Deletes a stored key. No-op if the key does not exist.

**`keys()`** — Lists all keys stored by this napplet.

All methods are async because they cross the postMessage boundary. Each request includes a correlation ID (`id`); the shell responds with a matching ID so the napplet can resolve the correct Promise.

## Wire Protocol

Storage operations use the NIP-5D wire format. Each request includes an `id` field for correlation.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `storage.get` | napplet -> shell | `id`, `key` |
| `storage.set` | napplet -> shell | `id`, `key`, `value` |
| `storage.remove` | napplet -> shell | `id`, `key` |
| `storage.keys` | napplet -> shell | `id` |
| `storage.get.result` | shell -> napplet | `id`, `value` (string or null) |
| `storage.set.result` | shell -> napplet | `id` |
| `storage.remove.result` | shell -> napplet | `id` |
| `storage.keys.result` | shell -> napplet | `id`, `keys` (string array) |

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

### Error Handling

Any result message MAY include an `error` field (string). When `error` is present, other result fields are undefined.

```
<- { "type": "storage.set.result", "id": "b2", "error": "quota exceeded" }
```

## Shell Behavior

- The shell MUST scope storage by composite key `(dTag, aggregateHash)`. Different napplet types and different versions of the same napplet MUST have isolated storage. The shell maps each napplet's `(dTag, aggregateHash)` identity to an isolated storage namespace.
- The shell MUST enforce a per-napplet storage quota. A recommended default is 512 KB measured in UTF-8 byte count.
- The shell MUST persist storage so data survives page reloads. How the shell stores data (localStorage, IndexedDB, etc.) is an implementation detail.
- The shell MUST respond to every request with a result message carrying the same `id`.
- The shell MUST return an `error` field in the result message for quota violations and invalid requests.

## Security Considerations

- Storage is scoped by composite key `(dTag, aggregateHash)`. The shell MUST enforce isolation so a napplet cannot read or write another napplet's data.
- Storage quota enforcement prevents a single napplet from consuming unbounded host storage.
- Storage values are strings only. The shell SHOULD NOT attempt to parse or execute stored content.
- The shell MAY enforce ACL checks on `storage:read` and `storage:write` capabilities before processing storage requests.
- Storage keys and values are visible to the shell. Napplets that need confidential storage should encrypt values before storing.
