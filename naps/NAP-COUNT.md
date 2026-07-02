# NAP-COUNT

## Runtime-Mediated Event Counts

`draft`

**NAP ID:** NAP-COUNT
**Domain:** `count`
**Depends:**
- `relay` — capability · required — counts NIP-01 filter matches through relay COUNT support, runtime indexes, or runtime cache.
**Web binding (NIP-5D):** `window.napplet.count` · `shell.supports("count")`

## Description

NAP-COUNT lets napplets request counts for NIP-01 filters without downloading
matching events. It is for counts such as reactions, replies, reposts, quotes,
reports, and follower totals where event payloads are unnecessary.

The napplet supplies filters. The runtime owns relay choice, COUNT support,
aggregation, caching, approximation policy, and refusal handling.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `query` | `filters`, `options?` | `CountResult` | `count.query` / `.result` |

### Schemas

```cddl
CountFilter = {
  ? ids: [* tstr],
  ? authors: [* tstr],
  ? kinds: [* uint],
  ? since: uint,
  ? until: uint,
  ? limit: uint,
  * tstr => any, ; NIP-01 tag filters, e.g. "#e", "#p", "#q", "#a"
}

CountOptions = {
  ? approximate: bool,
  ? hll: bool,
}

CountResult = {
  ok: bool,
  ? count: uint,
  ? approximate: bool,
  ? hll: tstr,
  ? relays: [* tstr],
  ? error: tstr,
  ? reason: tstr,
}
```

`filters` MUST be a non-empty list of `CountFilter`. Multiple filters are ORed
and aggregated into one count, matching NIP-45 `COUNT` semantics.

## Operation Rules

| Operation | Rules |
|-----------|-------|
| `query` | Counts events matching NIP-01 filters. MUST NOT return event payloads. MAY use NIP-45 `COUNT`, runtime indexes, caches, or other runtime-owned sources. |

If `options.approximate` is false, the runtime SHOULD return exact counts or
reject with `"exact-count-unavailable"`. If `options.hll` is true, the runtime
MAY return a NIP-45-compatible HyperLogLog value when available.

## Common Filters

These are examples, not separate methods:

| Count | Filter |
|-------|--------|
| reactions to event | `{ "kinds": [7], "#e": [eventId] }` |
| replies to event | `{ "kinds": [1], "#e": [eventId] }` |
| reposts of event | `{ "kinds": [6], "#e": [eventId] }` |
| quotes of event | `{ "kinds": [1, 1111], "#q": [eventId] }` |
| reports of event | `{ "kinds": [1984], "#e": [eventId] }` |
| npub followers | `{ "kinds": [3], "#p": [pubkey] }` |

## NIP Mapping

| Concept | NIP tie |
|---------|---------|
| `filters` | NIP-01 filter objects. |
| multiple filters | NIP-45 `COUNT` OR semantics with one aggregated count. |
| `approximate` | NIP-45 approximate count flag. |
| `hll` | NIP-45 HyperLogLog response value. |

## Wire Protocol

`count.*` messages use NIP-5D wire format: `{ "type": "domain.action", ...payload }`.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `count.query` | napplet -> runtime | `id`, `filters`, `options?` |
| `count.query.result` | runtime -> napplet | `id`, `ok`, `count?`, `approximate?`, `hll?`, `relays?`, `error?`, `reason?` |

## Error Handling

Common errors: `"invalid-filter"`, `"unsupported-filter"`,
`"count-unavailable"`, `"exact-count-unavailable"`, `"relay-refused"`,
`"too-expensive"`, `"policy-denied"`, `"timeout"`, `"unsupported"`.

The runtime MUST reject filters it cannot safely count. Rejection is preferable
to fetching large event sets or returning misleading counts.

## Runtime Behavior

- MUST accept NIP-01 filters and preserve NIP-45 OR semantics for multiple filters.
- MUST NOT return matching event payloads through NAP-COUNT.
- MUST disclose approximation with `approximate: true`.
- MUST keep relay selection and aggregation policy runtime-owned.
- SHOULD return `relays` when useful and safe to disclose.
- MAY cache, precompute, merge relay `COUNT` responses, merge HLL values, or use
  local indexes.
- MAY refuse expensive, private, unsupported, or policy-sensitive filters.

## Security Considerations

- Counts can reveal user activity and moderation signals. Runtimes MAY restrict
  sensitive filters.
- Counts can be approximate, relay-biased, spam-inflated, or policy-filtered.
  Napplets MUST treat them as display hints, not proof.
- HLL values are mergeable estimates, not authoritative event sets.

## References

- [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) — Basic protocol flow and filters
- [NIP-45](https://github.com/nostr-protocol/nips/blob/master/45.md) — Event Counts

## Implementations

- None yet.
