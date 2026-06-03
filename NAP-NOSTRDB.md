NAP-NOSTRDB
============

Local Event Database
--------------------

`draft`

**NAP ID:** NAP-NOSTRDB
**Namespace:** `window.nostrdb`
**Discovery:** `shell.supports("nostrdb")`

> **Namespace note:** `window.nostrdb` is a top-level global (not `window.napplet.nostrdb`) to match the [nostrdb](https://github.com/nickhntv/nostrdb) library convention. Applications that use nostrdb outside napplet contexts can target the same namespace.

## Description

NAP-NOSTRDB provides napplets with access to a local Nostr event database maintained by the shell. Napplets can query cached events, add externally-authored events, look up events by ID, retrieve replaceable events, count matches, and subscribe to live updates. The shell typically backs this with an OPFS-based worker relay or IndexedDB store. Because napplet iframes run without `allow-same-origin`, they cannot access IndexedDB or OPFS directly — all database access is proxied through the shell via postMessage.

## API Surface

```typescript
interface NostrDb {
  query(filters: NostrFilter | NostrFilter[]): Promise<NostrEvent[]>;                              // via nostrdb.query
  add(event: NostrEvent): Promise<boolean>;                                                         // via nostrdb.add
  event(id: string): Promise<NostrEvent | undefined>;                                               // via nostrdb.event
  replaceable(kind: number, author: string, identifier?: string): Promise<NostrEvent | undefined>;  // via nostrdb.replaceable
  count(filters: NostrFilter | NostrFilter[]): Promise<number>;                                     // via nostrdb.count
  subscribe(filters: NostrFilter | NostrFilter[]): Subscription;                                   // via nostrdb.subscribe
}

interface Subscription {
  on(event: 'event', cb: (event: NostrEvent) => void): void;
  on(event: 'eose', cb: () => void): void;
  close(): void;                                                                                    // via nostrdb.unsubscribe
}
```

**`query(filters)`** — Returns events matching NIP-01 filters from the local database. Accepts a single filter or an array of filters (OR-combined).

**`add(event)`** — Inserts an externally-authored event into the local database. Events are typically received from relay subscriptions (`relay.event`) and cached locally for fast access. The napplet does NOT sign these events — they are already signed by their original author. Returns `true` on success, `false` if the event was rejected (duplicate, invalid signature, etc.).

**`event(id)`** — Retrieves a single event by its hex event ID. Returns `undefined` if not found.

**`replaceable(kind, author, identifier?)`** — Retrieves the latest replaceable event (NIP-33 semantics) for a given kind, author pubkey, and optional `d` tag identifier. Returns `undefined` if no match exists.

**`count(filters)`** — Returns the number of events matching the filters, following NIP-45 COUNT semantics.

**`subscribe(filters)`** — Opens a live subscription against the local database. Returns a handle to listen for matching events as they are added and to receive EOSE when existing stored events are exhausted. Call `close()` on the handle to end the subscription.

All methods are async because they cross the postMessage boundary. Requests include correlation IDs; the shell responds with matching IDs so the shim can resolve the correct Promise or route events to the correct subscription.

## Wire Protocol

NostrDB operations use the NIP-5D wire format. Requests include an `id` field for correlation. Subscription-scoped messages use a `subId` field to identify which subscription they belong to.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `nostrdb.query` | napplet -> shell | `id`, `filters` (NostrFilter array) |
| `nostrdb.add` | napplet -> shell | `id`, `event` (signed NostrEvent) |
| `nostrdb.event` | napplet -> shell | `id`, `eventId` (hex string) |
| `nostrdb.replaceable` | napplet -> shell | `id`, `kind` (number), `author` (hex string), `identifier`? (string) |
| `nostrdb.count` | napplet -> shell | `id`, `filters` (NostrFilter array) |
| `nostrdb.subscribe` | napplet -> shell | `id`, `subId`, `filters` (NostrFilter array) |
| `nostrdb.unsubscribe` | napplet -> shell | `id`, `subId` |
| `nostrdb.query.result` | shell -> napplet | `id`, `events` (NostrEvent array) |
| `nostrdb.add.result` | shell -> napplet | `id`, `ok` (boolean), `error`? (string) |
| `nostrdb.event.result` | shell -> napplet | `id`, `event` (NostrEvent or null) |
| `nostrdb.replaceable.result` | shell -> napplet | `id`, `event` (NostrEvent or null) |
| `nostrdb.count.result` | shell -> napplet | `id`, `count` (number) |
| `nostrdb.sub.event` | shell -> napplet | `subId`, `event` (NostrEvent) |
| `nostrdb.sub.eose` | shell -> napplet | `subId` |

Note: Subscription push messages use `nostrdb.sub.event` and `nostrdb.sub.eose` (not `nostrdb.event`) to disambiguate them from the by-ID event lookup.

### NostrFilter

Filters follow the NIP-01 filter format:

```json
{ "kinds": [1], "authors": ["<hex>"], "#e": ["<hex>"], "#p": ["<hex>"], "since": 1234, "until": 5678, "limit": 50 }
```

All filter fields are optional. Multiple filters in the `filters` array are OR-combined.

### Examples

**Query — request and result:**
```
-> { "type": "nostrdb.query", "id": "a1", "filters": [{ "kinds": [1], "authors": ["abc..."], "limit": 20 }] }
<- { "type": "nostrdb.query.result", "id": "a1", "events": [{ "id": "xyz...", "pubkey": "abc...", "kind": 1, "content": "hello", "tags": [], "created_at": 1234567890, "sig": "..." }] }
```

**Add — externally-authored event (success and rejection):**
```
-> { "type": "nostrdb.add", "id": "b2", "event": { "id": "xyz...", "pubkey": "abc...", "kind": 1, "content": "hello", "tags": [], "created_at": 1234567890, "sig": "..." } }
<- { "type": "nostrdb.add.result", "id": "b2", "ok": true }

-> { "type": "nostrdb.add", "id": "b3", "event": { "id": "xyz...", "pubkey": "abc...", "kind": 1, "content": "hello", "tags": [], "created_at": 1234567890, "sig": "..." } }
<- { "type": "nostrdb.add.result", "id": "b3", "ok": false, "error": "duplicate" }
```

**Event by ID — found and not found:**
```
-> { "type": "nostrdb.event", "id": "c4", "eventId": "xyz..." }
<- { "type": "nostrdb.event.result", "id": "c4", "event": { "id": "xyz...", "pubkey": "abc...", "kind": 1, "content": "hello", "tags": [], "created_at": 1234567890, "sig": "..." } }

-> { "type": "nostrdb.event", "id": "c5", "eventId": "missing..." }
<- { "type": "nostrdb.event.result", "id": "c5", "event": null }
```

**Replaceable — with and without identifier:**
```
-> { "type": "nostrdb.replaceable", "id": "d6", "kind": 0, "author": "abc..." }
<- { "type": "nostrdb.replaceable.result", "id": "d6", "event": { "id": "xyz...", "pubkey": "abc...", "kind": 0, "content": "{...}", "tags": [], "created_at": 1234567890, "sig": "..." } }

-> { "type": "nostrdb.replaceable", "id": "d7", "kind": 30023, "author": "abc...", "identifier": "my-article" }
<- { "type": "nostrdb.replaceable.result", "id": "d7", "event": { "id": "xyz...", "pubkey": "abc...", "kind": 30023, "content": "...", "tags": [["d", "my-article"]], "created_at": 1234567890, "sig": "..." } }
```

**Count:**
```
-> { "type": "nostrdb.count", "id": "e8", "filters": [{ "kinds": [1], "authors": ["abc..."] }] }
<- { "type": "nostrdb.count.result", "id": "e8", "count": 42 }
```

**Subscribe — subscribe, receive push events, EOSE, unsubscribe:**
```
-> { "type": "nostrdb.subscribe", "id": "f9", "subId": "sub-1", "filters": [{ "kinds": [1], "limit": 100 }] }
<- { "type": "nostrdb.sub.event", "subId": "sub-1", "event": { "id": "xyz...", "pubkey": "abc...", "kind": 1, "content": "hello", "tags": [], "created_at": 1234567890, "sig": "..." } }
<- { "type": "nostrdb.sub.eose", "subId": "sub-1" }
<- { "type": "nostrdb.sub.event", "subId": "sub-1", "event": { "id": "new...", "pubkey": "abc...", "kind": 1, "content": "new event", "tags": [], "created_at": 1234567891, "sig": "..." } }

-> { "type": "nostrdb.unsubscribe", "id": "g10", "subId": "sub-1" }
```

**Error:**
```
<- { "type": "nostrdb.add.result", "id": "b2", "ok": false, "error": "invalid signature" }
<- { "type": "nostrdb.query.result", "id": "a1", "error": "database unavailable" }
```

### Error Handling

Any result message MAY include an `error` field (string). When `error` is present, other result fields are undefined. `nostrdb.add.result` always includes `ok` (boolean); when rejection is due to a specific condition, `error` describes it.

## Shell Behavior

- The shell MUST route `nostrdb.*` messages to its local event database implementation.
- The shell MUST match responses using correlation `id` for request/result pairs and `subId` for subscription delivery.
- The shell MUST validate event signatures on `nostrdb.add` before storing — events with invalid signatures MUST be rejected with `nostrdb.add.result { ok: false, error: "invalid signature" }`.
- The shell MUST send `nostrdb.sub.eose` when stored events for a subscription are exhausted.
- The shell MUST honor `nostrdb.unsubscribe` by ceasing event delivery for that `subId`.
- The shell MUST respond to every request with a result message carrying the same `id`.
- The shell SHOULD persist the database across page reloads using OPFS, IndexedDB, or equivalent.
- The shell MAY enforce ACL checks on database access (e.g., restricting which kinds a napplet can query).
- The shell MAY scope database visibility per napplet using the napplet's `(dTag, aggregateHash)` identity.

## Security Considerations

- The shell controls what data the napplet can see. The local database MAY contain events from multiple users; the shell MAY filter results based on the napplet's ACL.
- `add()` accepts externally-authored events — the shell MUST validate event signatures before storing. Invalid signatures MUST be rejected.
- The database is shared across napplets (shared cache). Events added by one napplet are visible to others with database access. The shell MAY scope access per napplet if stricter isolation is needed.
- `subscribe` creates long-lived connections. The shell SHOULD clean up subscriptions when the napplet's iframe is removed or when the napplet disconnects.
- The shell SHOULD respond promptly to requests. Napplets SHOULD enforce request timeouts to avoid indefinite blocking.
