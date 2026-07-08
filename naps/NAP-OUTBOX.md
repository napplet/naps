NAP-OUTBOX
==========

Outbox-Aware Relay Access
-------------------------

`draft`

**NAP ID:** NAP-OUTBOX
**Domain:** `outbox`
**Depends:**
- `relay` — wire · required — imports `RelayEventResult` for raw Nostr event returns.
- `resource` — wire · optional — imported `RelayEventResult.sidecar.resources?` carries `ResourceSidecarEntry[]`, a type owned by the `resource` domain.
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

`RelayEventResult` is owned by NAP-RELAY. NAP-OUTBOX imports it by name for
every raw Nostr event it returns. Its sidecar carries optional
`resources` and `relayHints`; relay hints replace NAP-OUTBOX's former top-level
`relays` result map for returned events.

Primitive references:

| Name | Meaning |
|------|---------|
| `NostrFilter` | NIP-01 filter object. |
| `NostrEvent` | NIP-01 signed event object. |
| `EventTemplate` | Unsigned event template for shell signing. |
| `RelayEventResult` | External type owned by NAP-RELAY. |

Enumerations:

| Name | Values |
|------|--------|
| `OutboxTarget.direction` | `read`, `write` |
| `OutboxRelayPlan.source` | `nip65`, `cache`, `policy`, `fallback` |

`OutboxEventOptions`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `author` | `tstr` | no | Author pubkey hint for relay discovery. |
| `relays` | list of `tstr` | no | Relay URL hints or policy override candidates. |
| `timeoutMs` | `uint` | no | Request timeout in milliseconds. |

`OutboxQueryOptions`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `authors` | list of `tstr` | no | Author pubkey hints for relay discovery. |
| `relays` | list of `tstr` | no | Relay URL hints or policy override candidates. |
| `limit` | `uint` | no | Maximum number of events to return. |
| `timeoutMs` | `uint` | no | Request timeout in milliseconds. |

`OutboxSubscribeOptions`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `authors` | list of `tstr` | no | Author pubkey hints for relay discovery. |
| `relays` | list of `tstr` | no | Relay URL hints or policy override candidates. |
| `limit` | `uint` | no | Maximum stored events to backfill. |
| `timeoutMs` | `uint` | no | Initial request timeout in milliseconds. |

`OutboxPublishOptions`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `relays` | list of `tstr` | no | Explicit relay URL fanout candidates, subject to shell validation. |
| `toOutbox` | `bool` | no | Defaults to `true`. Publish to the shell-user's NIP-65 write relays. |
| `toInboxes` | list of `tstr` | no | Author pubkeys whose NIP-65 read relays are required publish fanout targets. |

`OutboxTarget`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `authors` | list of `tstr` | no | Author pubkeys for relay discovery. |
| `pubkey` | `tstr` | no | Single pubkey target. |
| `direction` | `read` or `write` | no | Relay direction to resolve. |

`OutboxRelayPlan`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `relays` | list of `tstr` | yes | Candidate relay URLs. |
| `source` | `nip65`, `cache`, `policy`, or `fallback` | yes | Source of the relay plan. |
| `missingAuthors` | list of `tstr` | no | Authors whose relay lists could not be resolved. |

`OutboxEventResult`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `result` | `RelayEventResult` | no | Matching event result when found. |
| `incomplete` | `bool` | no | True when lookup results are partial. |
| `error` | `tstr` | no | Error reason when lookup failed. |

`OutboxResult`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `events` | list of `RelayEventResult` | yes | Deduplicated event results. |
| `incomplete` | `bool` | no | True when results are partial. |
| `error` | `tstr` | no | Error reason when the query failed. |

`OutboxSubscription`:

| Member | Type | Required | Description |
|--------|------|----------|-------------|
| `on("event", cb)` | function | yes | Registers a callback for `outbox.event` deliveries. The callback receives `RelayEventResult`. |
| `on("closed", cb)` | function | yes | Registers a callback for `outbox.closed`. The callback receives optional `reason`. |
| `close()` | function | yes | Closes the subscription by sending `outbox.close` for the handle's subscription ID. |

`OutboxPublishResult`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ok` | `bool` | yes | True when publish succeeded. |
| `event` | `NostrEvent` | no | Signed event produced by the shell. |
| `eventId` | `tstr` | no | Event ID of the signed event. |
| `relays` | map of relay URL `tstr` to `bool` | no | Per-relay publish result. |
| `error` | `tstr` | no | Error reason when publish failed. |

**`getEvent(eventId, options?)`** -- Fetches one event by ID through shell-owned relay routing. If `options.author` is present, the shell SHOULD use that author's outbox relays as the primary read target. If no author is known, the shell MAY use relay hints, cache, policy relays, fallback relays, or relay intelligence. The shell validates the event ID and signature before returning it as `RelayEventResult`.

**`query(filters, options?)`** -- Performs a one-shot outbox-aware query. The shell derives authors from `filters.authors`, `options.authors`, tag hints, or shell policy, resolves the relevant relays, queries them, deduplicates events by `id`, merges relay hints into each `RelayEventResult.sidecar.relayHints`, and returns the collected results.

**`subscribe(filters, options?)`** -- Opens a live outbox-aware event stream. Outbox routing may span multiple relays and may change over time, so the napplet-facing lifecycle is the subscription handle plus `outbox.close` and `outbox.closed`.

**`publish(template, options?)`** -- Signs the template once and publishes that signed event through outbox-aware relay fanout. Unless `options.toOutbox` is `false`, the shell includes the shell-user's NIP-65 write relays. When `options.toInboxes` is non-empty, the shell MUST resolve those authors' NIP-65 read relays and include each resolved, policy-allowed relay in the fanout set. The shell also includes validated `options.relays` entries. The final publish target set is the deduplicated union of requested own-outbox relays, requested inbox relays, and validated explicit relay URLs. If the shell cannot satisfy a required publish fanout target, it MUST report the failure through `OutboxPublishResult`.

**`resolveRelays(target)`** -- Returns the relay plan the shell would use for a read or write target. This is useful for diagnostics and UI, but napplets SHOULD prefer `query`, `subscribe`, or `publish` instead of manually replaying the plan through NAP-RELAY.

## Wire Protocol

`outbox.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `outbox.getEvent` | napplet -> shell | `id`, `eventId`, `options?` |
| `outbox.getEvent.result` | shell -> napplet | `id`, `result?`, `incomplete?`, `error?` |
| `outbox.query` | napplet -> shell | `id`, `filters`, `options?` |
| `outbox.query.result` | shell -> napplet | `id`, `events`, `incomplete?`, `error?` |
| `outbox.subscribe` | napplet -> shell | `id`, `subId`, `filters`, `options?` |
| `outbox.event` | shell -> napplet | `subId`, `result` |
| `outbox.closed` | shell -> napplet | `subId`, `reason?` |
| `outbox.close` | napplet -> shell | `id`, `subId` |
| `outbox.publish` | napplet -> shell | `id`, `event`, `options?` |
| `outbox.publish.result` | shell -> napplet | `id`, `ok`, `event?`, `eventId?`, `relays?`, `error?` |
| `outbox.resolveRelays` | napplet -> shell | `id`, `target` |
| `outbox.resolveRelays.result` | shell -> napplet | `id`, `plan`, `error?` |

Key design notes:
- `filters` are NIP-01 filters, either one filter or an array of filters.
- `RelayEventResult.sidecar.relayHints` records relay URLs where the shell observed returned events, when the shell can disclose them.
- `RelayEventResult.sidecar.resources` carries resource sidecars when the shell pre-resolves bytes under NAP-RESOURCE policy.
- `OutboxPublishOptions.toOutbox` defaults to `true`.
- `OutboxPublishOptions.toInboxes` is a delivery contract for recipient inbox fanout, not a hint.
- `options.relays` is an explicit relay URL fanout set for `publish`, and a hint or policy override candidate for read surfaces. It never bypasses shell ACLs.
- `outbox.getEvent` is a request; `outbox.event` remains the subscription push message.
- The shell may internally use NAP-RELAY semantics, but napplets consume this higher-level outbox interface.

### Examples

**Fetch a single event with an author hint:**
```
-> {
     "type": "outbox.getEvent",
     "id": "e1",
     "eventId": "ev1...",
     "options": { "author": "ab12...", "timeoutMs": 3000 }
   }
<- {
     "type": "outbox.getEvent.result",
     "id": "e1",
     "result": {
       "event": { "id": "ev1...", "pubkey": "ab12...", "kind": 1, "content": "hello", "tags": [], "created_at": 1234567890, "sig": "..." },
       "sidecar": { "relayHints": ["wss://relay.example.com"] }
     }
   }
```

**Query an author's notes from their outbox relays:**
```
-> {
     "type": "outbox.query",
     "id": "q1",
     "filters": [{ "authors": ["ab12..."], "kinds": [1], "limit": 20 }],
     "options": { "timeoutMs": 3000 }
   }
<- {
     "type": "outbox.query.result",
     "id": "q1",
     "events": [
       {
         "event": { "id": "ev1...", "pubkey": "ab12...", "kind": 1, "content": "hello", "tags": [], "created_at": 1234567890, "sig": "..." },
         "sidecar": { "relayHints": ["wss://relay.example.com"] }
       }
     ]
   }
```

**Live subscription:**
```
-> {
     "type": "outbox.subscribe",
     "id": "s1",
     "subId": "sub-1",
     "filters": [{ "authors": ["ab12..."], "kinds": [1], "limit": 50 }],
     "options": { "timeoutMs": 3000 }
   }
<- { "type": "outbox.event", "subId": "sub-1", "result": { "event": { "id": "ev1...", "pubkey": "ab12...", "kind": 1, "content": "hello", "tags": [], "created_at": 1234567890, "sig": "..." }, "sidecar": { "relayHints": ["wss://relay.example.com"] } } }
```

**Publish a reply through the user's outbox and referenced inboxes:**
```
-> {
     "type": "outbox.publish",
     "id": "p1",
     "event": { "kind": 1111, "content": "reply from a napplet", "tags": [["p", "ab12..."]], "created_at": 1234567890 },
     "options": { "toInboxes": ["ab12..."] }
   }
<- {
     "type": "outbox.publish.result",
     "id": "p1",
     "ok": true,
     "event": { "id": "ev2...", "pubkey": "user...", "kind": 1111, "content": "reply from a napplet", "tags": [["p", "ab12..."]], "created_at": 1234567890, "sig": "..." },
     "eventId": "ev2...",
     "relays": { "wss://relay.example.com": true }
   }
```

**Publish to explicit relay targets without the user's public outbox:**
```
-> {
     "type": "outbox.publish",
     "id": "p2",
     "event": { "kind": 1059, "content": "...", "tags": [["p", "ab12..."]], "created_at": 1234567890 },
     "options": { "toOutbox": false, "relays": ["wss://dm.example.com"] }
   }
<- {
     "type": "outbox.publish.result",
     "id": "p2",
     "ok": true,
     "event": { "id": "ev3...", "pubkey": "user...", "kind": 1059, "content": "...", "tags": [["p", "ab12..."]], "created_at": 1234567890, "sig": "..." },
     "eventId": "ev3...",
     "relays": { "wss://dm.example.com": true }
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
- The shell MUST publish at most one signed event for each `outbox.publish` request.
- The shell MUST include the shell-user's NIP-65 write relays in `outbox.publish` fanout unless `OutboxPublishOptions.toOutbox` is `false`.
- The shell MUST resolve NIP-65 read relays for every author in `OutboxPublishOptions.toInboxes` when `toInboxes` is non-empty.
- The shell MUST include every resolved, policy-allowed `toInboxes` relay in `outbox.publish` fanout or report failure through `OutboxPublishResult`.
- The shell MUST deduplicate `outbox.publish` relay URLs across own-outbox relays, inbox relays, and validated explicit relay URLs before publishing.
- The shell MUST respond to every request with a result or lifecycle message carrying the same `id` or `subId`.
- The shell MUST deliver matching subscription events as `outbox.event` until the napplet sends `outbox.close` or the shell terminates the stream with `outbox.closed`.
- The shell SHOULD merge observed relay URLs into `RelayEventResult.sidecar.relayHints` after deduplication when it can disclose them.
- The shell MAY include pre-resolved byte resources in `RelayEventResult.sidecar.resources`; NAP-RELAY's default-off sidecar privacy policy and NAP-RESOURCE's fetch policy both apply.
- The shell SHOULD cache relay lists and refresh them according to shell policy.
- The shell SHOULD fall back to configured relays when NIP-65 data is absent, stale, or unreachable.
- The shell MAY use NIP-66 or other relay intelligence to choose among candidate relays.
- The shell MAY enforce ACL checks for outbox read, subscribe, publish, relay override, and relay discovery operations.

## Security Considerations

- Relay selection leaks user interest. Shells SHOULD limit broad outbox queries and apply user or napplet-specific policy for sensitive authors and filters.
- `options.relays` MUST be subject to shell validation. Napplets MUST NOT be able to force connections to private network relays or disallowed hosts.
- Event ID lookups without an author can become broad relay searches. Shells SHOULD enforce timeouts, fallback limits, and policy checks before expanding beyond hinted or known relays.
- Unbounded author sets, missing limits, and broad subscriptions can exhaust shell and relay resources. Shells SHOULD enforce limits, timeouts, and maximum subscription counts.
- Publishing remains shell-mediated. Shells SHOULD show consent UI or require prior policy approval before signing and publishing events.
- Relay plans may reveal user relay preferences. Shells MAY redact relay URLs or return a policy-derived plan when a napplet is not trusted to inspect relay metadata.

## References

- [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md)
- [NIP-65](https://github.com/nostr-protocol/nips/blob/master/65.md)
- [NIP-66](https://github.com/nostr-protocol/nips/blob/master/66.md)

## Implementations

- (none yet)

## Changelog

- `a1968b8` - Introduced NAP-OUTBOX for outbox-aware relay routing.
- `20379e8` - Added single-event retrieval through `outbox.getEvent`.
- `57db924` - Changed outbox event returns to use relay-owned result shape and declared the relay dependency.
- `70515a5` - Removed `strategy`, `live`, and `outbox.eose` caller-visible controls from outbox options, messages, and examples.
- `e575906` - Restored the `OutboxSubscription` handle definition for `event`, `closed`, and `close()` lifecycle methods.
