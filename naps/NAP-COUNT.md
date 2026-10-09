# NAP-COUNT

## Runtime-Mediated Event Counts

`draft`

**NAP ID:** NAP-COUNT
**Domain:** `count`
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.count`; domain presence signals availability.

## Description

NAP-COUNT lets napplets request counts for [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) filters without downloading
matching events. It is for counts such as reactions, replies, reposts, quotes,
reports, and follower totals where event payloads are unnecessary.

The napplet supplies filters. The runtime owns relay choice, COUNT support,
aggregation, caching, approximation policy, and refusal handling. The `count`
domain has no required NAP dependency. Using a relay as an internal source does
not require exposing the `relay` domain to the napplet.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `query` | `filters`, `options?` | `CountResult` | `count.query` / `count.query.result` |

### Schemas

`CountFilter` fields follow [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md):

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ids` | no | list of text | One or more event IDs in the upstream format. |
| `authors` | no | list of text | One or more public keys in the upstream format. |
| `kinds` | no | list of integer | One or more event kind numbers. |
| `since` | no | integer | Non-negative inclusive lower timestamp bound in seconds. |
| `until` | no | integer | Non-negative inclusive upper timestamp bound in seconds. |
| `limit` | no | integer | Non-negative upstream filter limit. |
| `#<letter>` | no | list of text | One or more tag values; the letter is a single ASCII letter, such as `#e`, `#p`, `#q`, or `#a`. |

`CountOptions` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `approximate` | no | boolean | Permit an estimated count. Defaults to false. |
| `hll` | no | boolean | Request a HyperLogLog (HLL) value when available. Defaults to false. |

Omitting `options` is equivalent to setting both options to false.

`CountResult` MUST be exactly one of `CountSuccess` or `CountFailure`.

`CountSuccess` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | MUST be true. |
| `count` | yes | integer | Non-negative count; zero is a successful result. |
| `approximate` | no | boolean | MUST be true for an estimate. Omission means false. |
| `hll` | no | text | [NIP-45](https://github.com/nostr-protocol/nips/blob/master/45.md#hll-encoding) HLL value, only when requested and available. |
| `relays` | no | list of text | One or more relay URLs used, when safe to disclose. Omit when no relay was used. |

A successful result MUST NOT contain `error` or `reason`.

`CountFailure` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | MUST be false. |
| `error` | yes | text | Non-empty machine-readable error code; see Error Handling. |
| `reason` | no | text | Human-readable explanation. |

A failed result MUST NOT contain `count`, `approximate`, `hll`, or `relays`.

`filters` MUST be a non-empty list of `CountFilter`, and every list-valued
field in each filter MUST contain at least one value. Multiple filters are ORed
and aggregated into one count, matching [NIP-45](https://github.com/nostr-protocol/nips/blob/master/45.md) `COUNT` semantics.

## Operation Rules

| Operation | Rules |
|-----------|-------|
| `query` | Counts events matching [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) filters. MUST NOT return event payloads. MAY use [NIP-45](https://github.com/nostr-protocol/nips/blob/master/45.md) `COUNT`, runtime indexes, caches, or other runtime-owned sources. |

If `options.approximate` is false, the runtime MUST NOT return an estimated
count. If it cannot provide an exact count, it MUST reject with
`"exact-count-unavailable"`. Setting this option to true permits an estimate;
it does not require one. If `options.hll` is true, the runtime MAY return an HLL
value compatible with [NIP-45](https://github.com/nostr-protocol/nips/blob/master/45.md)
when available. If false, it MUST omit `hll`. Requesting HLL data does not permit
an estimated `count` unless `options.approximate` is also true.

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
| `filters` | [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) filter objects. |
| multiple filters | [NIP-45](https://github.com/nostr-protocol/nips/blob/master/45.md) `COUNT` OR semantics with one aggregated count. |
| `approximate` | [NIP-45](https://github.com/nostr-protocol/nips/blob/master/45.md) approximate count flag. |
| `hll` | [NIP-45](https://github.com/nostr-protocol/nips/blob/master/45.md) HyperLogLog response value. |

## Wire Protocol

`count.*` messages use [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format: `{ "type": "domain.action", ...payload }`.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `count.query` | napplet -> runtime | `id`, `filters`, `options?` |
| `count.query.result` | runtime -> napplet | `id`, `ok`, `count?`, `approximate?`, `hll?`, `relays?`, `error?`, `reason?` |

The runtime MUST respond with the same `id` as the request. Result payloads
MUST satisfy `CountSuccess` or `CountFailure`; the optional markers in the wire
table are conditional on that choice, not permission to omit required fields.

### Examples

**Exact count with default options:**
```json
{ "type": "count.query", "id": "c1", "filters": [{ "kinds": [1] }] }
```
```json
{ "type": "count.query.result", "id": "c1", "ok": true, "count": 0 }
```

**Approximation explicitly permitted:**
```json
{ "type": "count.query", "id": "c2", "filters": [{ "kinds": [1] }], "options": { "approximate": true } }
```
```json
{ "type": "count.query.result", "id": "c2", "ok": true, "count": 1200, "approximate": true }
```

**Exact count unavailable for the default request:**
```json
{ "type": "count.query.result", "id": "c1", "ok": false, "error": "exact-count-unavailable", "reason": "Only an estimate is available." }
```

`{ "ok": true }` and `{ "ok": false, "count": 10 }` are invalid results.

## Error Handling

Common errors: `"invalid-filter"`, `"unsupported-filter"`,
`"count-unavailable"`, `"exact-count-unavailable"`, `"relay-refused"`,
`"too-expensive"`, `"policy-denied"`, `"timeout"`, `"unsupported"`.

The runtime MUST reject filters it cannot safely count. Rejection is preferable
to fetching large event sets or returning misleading counts.

## Runtime Behavior

- MUST accept [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) filters and preserve [NIP-45](https://github.com/nostr-protocol/nips/blob/master/45.md) OR semantics for multiple filters.
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

## Changelog

- `9d097a0` - Introduced NAP-COUNT for runtime-owned event-count queries over Nostr filters.
- `8995853` - Renamed the duplicated count operation to `query` / `count.query`.

- `5f7a0df` - Adopted injected-domain availability and linked the current upstream web binding.
- `c9e0e4c` - Defined false defaults for omitted approximation and HLL options.
- `b963eb2` - Required non-empty list-valued filter attributes.
- `c402531` - Specified the full count.query.result response type.
