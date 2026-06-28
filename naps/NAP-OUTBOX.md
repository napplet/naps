NAP-OUTBOX
==========

Outbox-Aware Relay Access
-------------------------

`draft`

**NAP ID:** NAP-OUTBOX
**Domain:** `outbox`
**Depends:**
- `relay` — layering · optional — the shell MAY use `relay` semantics internally; napplets consume the higher-level outbox surface
**Web binding (NIP-5D):** `window.napplet.outbox` · `shell.supports("outbox")`

## Description

NAP-OUTBOX provides napplets with outbox-model relay routing through the shell. A napplet can already query relays directly through NAP-RELAY, but then every napplet must discover NIP-65 relay lists, choose author write relays, fall back when lists are missing, deduplicate events, and keep live subscriptions aligned as relay intelligence changes. NAP-OUTBOX lets a napplet provide Nostr filters and intent; the runtime finds the correct relays, queries them, deduplicates results, and streams updates.

This interface is for Nostr event access where relay selection is part of the result correctness. The shell owns relay discovery, routing, fallback, deduplication, and publish fanout policy.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `getEvent` | `eventId` (`tstr`), optional `options` (`OutboxEventOptions`) | `OutboxEventResult` | `outbox.getEvent` / `outbox.getEvent.result` |
| `query` | `filters` (`NostrFilter` or list), optional `options` (`OutboxQueryOptions`) | `OutboxResult` | `outbox.query` / `outbox.query.result` |
| `subscribe` | `filters` (`NostrFilter` or list), optional `options` (`OutboxSubscribeOptions`) | `OutboxSubscription` handle | `outbox.subscribe` plus push messages |
| `publish` | `template` (`EventTemplate`), optional `options` (`OutboxPublishOptions`) | `OutboxPublishResult` | `outbox.publish` / `outbox.publish.result` |
| `resolveRelays` | `target` (`OutboxTarget`) | `OutboxRelayPlan` | `outbox.resolveRelays` / `outbox.resolveRelays.result` |

### Schemas

```cddl
NostrFilter = { * tstr => any }
NostrEvent = { * tstr => any }
EventTemplate = { * tstr => any }
OutboxStrategy = "outbox" / "inbox" / "auto"

OutboxEventOptions = {
  ? author: tstr,
  ? relays: [* tstr],
  ? strategy: OutboxStrategy,
  ? timeoutMs: uint,
}

OutboxQueryOptions = {
  ? authors: [* tstr],
  ? relays: [* tstr],
  ? strategy: OutboxStrategy,
  ? limit: uint,
  ? timeoutMs: uint,
}

OutboxSubscribeOptions = {
  ? authors: [* tstr],
  ? relays: [* tstr],
  ? strategy: OutboxStrategy,
  ? limit: uint,
  ? timeoutMs: uint,
  ? live: bool,
}

OutboxPublishOptions = {
  ? relays: [* tstr],
  ? targetAuthors: [* tstr],
  ? strategy: OutboxStrategy,
}

OutboxTarget = {
  ? authors: [* tstr],
  ? pubkey: tstr,
  ? direction: "read" / "write",
  ? strategy: OutboxStrategy,
}

OutboxRelayPlan = {
  relays: [* tstr],
  source: "nip65" / "cache" / "policy" / "fallback",
  ? missingAuthors: [* tstr],
}

OutboxEventResult = {
  ? event: NostrEvent,
  relays: [* tstr],
  ? incomplete: bool,
  ? error: tstr,
}

OutboxResult = {
  events: [* NostrEvent],
  relays: { * tstr => [* tstr] },
  ? incomplete: bool,
  ? error: tstr,
}

OutboxPublishResult = {
  ok: bool,
  ? event: NostrEvent,
  ? eventId: tstr,
  ? relays: { * tstr => bool },
  ? error: tstr,
}
```

**`getEvent(eventId, options?)`** -- Fetches one event by ID through shell-owned relay routing. If `options.author` is present, the shell SHOULD use that author's outbox relays as the primary read target. If no author is known, the shell MAY use relay hints, cache, policy relays, fallback relays, or relay intelligence. The shell validates the event ID and signature before returning it.

**`query(filters, options?)`** -- Performs a one-shot outbox-aware query. The shell derives authors from `filters.authors`, `options.authors`, tag hints, or shell policy, resolves the relevant relays, queries them, deduplicates events by `id`, and returns the collected events.

**`subscribe(filters, options?)`** -- Opens a live outbox-aware subscription. The shell may add or remove relay connections as NIP-65 relay lists are discovered or updated.

**`publish(template, options?)`** -- Publishes a shell-signed event using outbox-aware relay fanout. For the shell-user's own event, this usually means the user's write relays. For directed events, the shell MAY include recipient inbox relays.

**`resolveRelays(target)`** -- Returns the relay plan the shell would use for a read or write target. This is useful for diagnostics and UI, but napplets SHOULD prefer `query`, `subscribe`, or `publish` instead of manually replaying the plan through NAP-RELAY.

## Wire Protocol

`outbox.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `outbox.getEvent` | napplet -> shell | `id`, `eventId`, `options?` |
| `outbox.getEvent.result` | shell -> napplet | `id`, `event?`, `relays`, `incomplete?`, `error?` |
| `outbox.query` | napplet -> shell | `id`, `filters`, `options?` |
| `outbox.query.result` | shell -> napplet | `id`, `events`, `relays`, `incomplete?`, `error?` |
| `outbox.subscribe` | napplet -> shell | `id`, `subId`, `filters`, `options?` |
| `outbox.event` | shell -> napplet | `subId`, `event`, `relay?` |
| `outbox.eose` | shell -> napplet | `subId` |
| `outbox.closed` | shell -> napplet | `subId`, `reason?` |
| `outbox.close` | napplet -> shell | `id`, `subId` |
| `outbox.publish` | napplet -> shell | `id`, `event`, `options?` |
| `outbox.publish.result` | shell -> napplet | `id`, `ok`, `event?`, `eventId?`, `relays?`, `error?` |
| `outbox.resolveRelays` | napplet -> shell | `id`, `target` |
| `outbox.resolveRelays.result` | shell -> napplet | `id`, `plan`, `error?` |

Key design notes:
- `filters` are NIP-01 filters, either one filter or an array of filters.
- `relays` in results records relay URLs where the shell observed returned events.
- `options.relays` is a hint or policy override, not a command to bypass shell ACLs.
- `outbox.getEvent` is a request; `outbox.event` remains the subscription push message.
- The shell may internally use NAP-RELAY semantics, but napplets consume this higher-level outbox interface.

### Examples

**Fetch a single event with an author hint:**
```
-> {
     "type": "outbox.getEvent",
     "id": "e1",
     "eventId": "ev1...",
     "options": { "author": "ab12...", "strategy": "outbox", "timeoutMs": 3000 }
   }
<- {
     "type": "outbox.getEvent.result",
     "id": "e1",
     "event": { "id": "ev1...", "pubkey": "ab12...", "kind": 1, "content": "hello", "tags": [], "created_at": 1234567890, "sig": "..." },
     "relays": ["wss://relay.example.com"]
   }
```

**Query an author's notes from their outbox relays:**
```
-> {
     "type": "outbox.query",
     "id": "q1",
     "filters": [{ "authors": ["ab12..."], "kinds": [1], "limit": 20 }],
     "options": { "strategy": "outbox", "timeoutMs": 3000 }
   }
<- {
     "type": "outbox.query.result",
     "id": "q1",
     "events": [{ "id": "ev1...", "pubkey": "ab12...", "kind": 1, "content": "hello", "tags": [], "created_at": 1234567890, "sig": "..." }],
     "relays": { "ev1...": ["wss://relay.example.com"] }
   }
```

**Live subscription:**
```
-> {
     "type": "outbox.subscribe",
     "id": "s1",
     "subId": "sub-1",
     "filters": [{ "authors": ["ab12..."], "kinds": [1], "limit": 50 }],
     "options": { "strategy": "outbox", "live": true }
   }
<- { "type": "outbox.event", "subId": "sub-1", "relay": "wss://relay.example.com", "event": { "id": "ev1...", "pubkey": "ab12...", "kind": 1, "content": "hello", "tags": [], "created_at": 1234567890, "sig": "..." } }
<- { "type": "outbox.eose", "subId": "sub-1" }
```

**Publish through the user's write relays:**
```
-> {
     "type": "outbox.publish",
     "id": "p1",
     "event": { "kind": 1, "content": "hello from a napplet", "tags": [], "created_at": 1234567890 },
     "options": { "strategy": "outbox" }
   }
<- {
     "type": "outbox.publish.result",
     "id": "p1",
     "ok": true,
     "event": { "id": "ev2...", "pubkey": "user...", "kind": 1, "content": "hello from a napplet", "tags": [], "created_at": 1234567890, "sig": "..." },
     "eventId": "ev2...",
     "relays": { "wss://relay.example.com": true }
   }
```

**Resolve relay plan:**
```
-> { "type": "outbox.resolveRelays", "id": "r1", "target": { "pubkey": "ab12...", "direction": "read" } }
<- { "type": "outbox.resolveRelays.result", "id": "r1", "plan": { "relays": ["wss://relay.example.com"], "source": "nip65" } }
```

### Error Handling

Result messages MAY include `error`. Common errors include `"not found"`, `"no authors"`, `"relay list unavailable"`, `"relay timeout"`, `"policy denied"`, `"publish denied"`, and `"invalid filter"`.

If the shell returns partial results because some relay lists or relay connections failed, it SHOULD set `incomplete: true` and still return events it found.

## Shell Behavior

- The shell MUST resolve relay plans according to NIP-65 relay list metadata when available.
- The shell MUST verify that an event returned by `outbox.getEvent` matches the requested `eventId`.
- The shell MUST deduplicate events by event ID before returning `outbox.query.result`.
- The shell MUST validate event signatures before delivering events to napplets.
- The shell MUST sign event templates submitted through `outbox.publish`; napplets do not receive signing keys.
- The shell MUST respond to every request with a result or lifecycle message carrying the same `id` or `subId`.
- The shell SHOULD cache relay lists and refresh them according to shell policy.
- The shell SHOULD fall back to configured relays when NIP-65 data is absent, stale, or unreachable.
- The shell MAY use NIP-66 or other relay intelligence to choose among candidate relays.
- The shell MAY enforce ACL checks for outbox read, subscribe, publish, relay override, and relay discovery operations.
- The shell MAY include recipient inbox relays when publishing directed events such as replies, reactions, direct messages, or zaps.

## Security Considerations

- Relay selection leaks user interest. Shells SHOULD limit broad outbox queries and apply user or napplet-specific policy for sensitive authors and filters.
- `options.relays` MUST be treated as a hint subject to shell validation. Napplets MUST NOT be able to force connections to private network relays or disallowed hosts.
- Event ID lookups without an author can become broad relay searches. Shells SHOULD enforce timeouts, fallback limits, and policy checks before expanding beyond hinted or known relays.
- Unbounded author sets, missing limits, and broad live subscriptions can exhaust shell and relay resources. Shells SHOULD enforce limits, timeouts, and maximum subscription counts.
- Publishing remains shell-mediated. Shells SHOULD show consent UI or require prior policy approval before signing and publishing events.
- Relay plans may reveal user relay preferences. Shells MAY redact relay URLs or return a policy-derived plan when a napplet is not trusted to inspect relay metadata.

## References

- [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md)
- [NIP-65](https://github.com/nostr-protocol/nips/blob/master/65.md)
- [NIP-66](https://github.com/nostr-protocol/nips/blob/master/66.md)

## Implementations

- (none yet)
