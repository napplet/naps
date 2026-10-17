NAP-NAVIGATION
==============

Internal Location and Intent History
------------------------------------

`draft`

**NAP ID:** NAP-NAVIGATION
**Domain:** `navigation`
**Depends:**
- `intent` — layering · required — the runtime checks retained entries against
  the reporting napplet's advertised intent contracts and uses intent delivery
  for navigation requests. Navigation defines no second dispatcher.

## Description

NAP-NAVIGATION makes a napplet's reported current location and retained history
available to its runtime. Each location is an invocation of an intent advertised
by that napplet: a stable convention identity and a payload containing only its
advertised parameter names. Internal routes that cannot be expressed this way
MUST NOT be retained as locations.

The napplet reports the views it actually committed. The runtime retains the
reported history for the shell. Navigation requests are intents; the napplet
handles them through its intent handler and reports the resulting location.
Intent acceptance or delivery alone MUST NOT create an entry or move the current
position.

A **history entry** is one visit to an advertised intent invocation. A
**snapshot** contains the retained entries and the current position. Snapshots
belong to one authenticated running instance, including when several instances
use the same manifest. A report asserts application state; it does not prove
what was rendered.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `report` | `snapshot` (`NavigationSnapshot`) | `NavigationResult` | `navigation.report` / `navigation.report.result` |
| `get` | none | `NavigationResult` | `navigation.get` / `navigation.get.result` |

There is no navigation request handler, listener registration, or traversal
outcome operation. [NAP-INTENT](NAP-INTENT.md) owns intent delivery.

### Schemas

`NavigationEntry` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | text | Non-empty visit identifier assigned by the napplet; unique within its instance lifetime. |
| `convention` | yes | text | Stable, queryless, fragment-free convention identity advertised by this napplet. |
| `payload` | no | map of text to any | Only advertised parameter names. Values must be JSON values as defined below. |
| `title` | no | text | Plain-text display label; not an invocation parameter. |

JSON values are null, boolean, finite number, text, or recursively composed
lists and maps of text to JSON values. No opaque route string, separate router
state, handler selector, or lifecycle hint is part of a retained invocation.
The entry's `id` and `title` are history metadata; they MUST NOT be inserted into
the intent payload.

An identifier MUST remain stable while its visit is retained. A discarded
identifier MUST NOT be reused within the instance lifetime. Separate visits to
the same convention and payload have distinct identifiers. Identifiers confer
no authority over another instance.

`NavigationSnapshot` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `revision` | yes | integer | Positive and monotonically increasing within the instance lifetime. |
| `entries` | yes | list of `NavigationEntry` | Complete retained history, oldest to newest. May be empty. |
| `index` | yes | integer or null | Zero-based current entry, or null when the current view has no retained invocation. |

A non-null index MUST identify an entry. An empty list requires a null index.
The reported current location is the selected entry's convention and payload.
There is no additional current-location string or hidden state to reconcile.
A null index MUST NOT be displayed as the last retained entry being current.

Revisions need not be consecutive. Every changed snapshot MUST have a revision
greater than every snapshot previously produced by that instance. Equal
revisions MUST contain identical data. Revisions MUST NOT reset within the
instance lifetime.

`NavigationView` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `snapshot` | yes | `NavigationSnapshot` or null | Latest accepted snapshot; null before the first accepted report. |
| `status` | yes | `"unreported"`, `"current"`, or `"uncertain"` | Whether the runtime has an accepted report of the current state. |

`current` means the latest report was accepted. It does not mean a requested
intent was handled. A snapshot with a null index can be current: the napplet has
reported that its current view is not represented in retained history.
`uncertain` means a rejected report prevents the runtime from treating its last
accepted snapshot as current. The runtime MUST keep that distinction visible to
the shell.

`NavigationResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | Whether the report was accepted or the read completed. |
| `view` | yes | `NavigationView` | State after processing the request. |
| `error` | no | `NavigationError` | Required exactly when `ok` is false. |

`NavigationError` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `code` | yes | `"invalid"`, `"unsupported"`, `"stale"`, `"denied"`, or `"limit"` | Meaning below. |
| `message` | no | text | Optional plain-text diagnostic. |

| Code | Meaning |
|------|---------|
| `invalid` | Malformed snapshot, non-map payload, duplicate identifiers, invalid index, or changed content at the accepted revision. |
| `unsupported` | An entry uses an unadvertised convention or parameter name. |
| `stale` | The report's revision is lower than the accepted revision. |
| `denied` | Shell policy rejects the operation. |
| `limit` | Entry count, payload size, or report rate exceeds runtime policy. |

### Advertised intents and parameter names

The runtime MUST use the eligible intent contracts of the verified manifest
bound to the reporting endpoint. It MUST NOT accept a manifest, handler, or
instance identifier supplied in the report as an alternative authority.
Eligibility and manifest parsing are defined by
[NAP-INTENT](NAP-INTENT.md#manifest-catalog-contract).

For every retained entry:

1. Its `convention` MUST equal an eligible advertised convention exactly.
   The router MUST NOT parse, normalize, prefix-match, or wildcard-match it.
2. Its payload, when present, MUST be a map. Every key MUST be an advertised
   parameter name for that convention, compared exactly.
3. An absent payload or an empty map is permitted. An advertisement with no
   parameter names permits no payload fields.
4. Advertised names are an allowlist, not a list of required fields. They do not
   declare value types, ranges, or nested schemas. The target remains responsible
   for validating convention semantics before acting on a delivery.

These restrictions govern what navigation may retain. They do not tighten
NAP-INTENT's general invocation rules or redefine an `i` tag as a payload schema.
A payload valid for ordinary intent delivery can still be ineligible for history.

If a convention has multiple eligible advertisements, a payload MUST fit the
parameter names of at least one complete advertisement. The runtime MUST NOT
union incompatible parameter lists to create an undeclared format.

Invocation query sugar MUST be transposed before constructing a report. Stored
conventions contain neither query nor fragment. Query-derived values remain
text; structured values use the explicit payload. Navigation introduces no
alternative URI normalization rules.

### Reporting and reading

A napplet using this domain MUST report its initial state and every committed
navigation transition. A report MUST describe actual state, never a proposed
destination. When the current view cannot be represented by an eligible intent
invocation, the napplet MUST report a null index rather than retain an
unadvertised location or continue claiming its previous view is current.

The runtime MUST validate every entry and replace the snapshot atomically.
One unsupported entry rejects the entire report; the runtime MUST NOT silently
remove entries or payload fields. Lower revisions are `stale`. An identical
retransmission of the accepted revision is idempotent; changed content at that
revision is `invalid`. Map key order is not significant for equality; list
order, value types, field presence, and values are significant.

Rejection MUST NOT mutate accepted history. Rejection of a newer report,
changed data at the accepted revision, or a report without a usable revision
MUST mark a non-null view `uncertain`. A null view remains `unreported`.
An identical retransmission MUST NOT clear uncertainty; a valid newer report
restores `current`.

`report` returns after validation. The napplet MUST inspect the result and
repair a rejected report before relying on the shell's current-location view.
`get` returns only the caller's retained view. The shell reads that same view
internally; the runtime does not need to invoke a napplet to read data it holds.

| Committed transition | Reported change |
|----------------------|-----------------|
| New visit from a selected entry | Discard forward history, append a fresh identifier, and select the new entry. |
| Replace the selected invocation | Preserve its identifier and list position; replace convention, payload, or label. |
| Revisit a retained entry | Preserve the entries and identifiers; select that visit. |
| Current view is not representable | Set index to null; do not add an undeclared invocation. |
| New visit with no selected entry | Append a fresh identifier to the retained list and select it. |
| Prune or clear | Report the complete remaining list and actual current position, or null if the current visit is no longer retained. |

Each changed snapshot advances the revision. Repeated invocations do not
collapse automatically. Pruning or clearing MUST NOT reset revisions or permit
identifier reuse.

## Navigation Through Intent

A shell requests a retained location by delivering that entry's convention and
payload through intent. It MUST NOT invent a second navigation command or add
private fields to the convention payload. The target validates the invocation,
applies its navigation guard, and reports any committed transition.

A history revisit addresses the instance that owns the history. It MUST NOT
silently choose another handler when user defaults change or restart a terminated
instance as if its old history remained live. Cross-napplet opens continue to use
ordinary handler resolution under NAP-INTENT.

Intent acceptance, delivery, or refusal alone MUST NOT change the reported
history or current position. A navigation report is an assertion of current
state, not an intent completion receipt. The shell MUST NOT label a requested
destination as reached without an accepted report selecting it.

### Intent integration still to specify

The current NAP-INTENT contract does not fully define this history-revisit flow.
[The existing-instance delivery proposal](https://github.com/napplet/naps/pull/128)
addresses shell provenance and fixed endpoint targeting without a new dispatcher.
It does not yet settle retained-visit identity. Before traversal can be
implemented interoperably, the intent contract must cover:

- Shell-originated delivery to a particular existing instance, including
  truthful sender provenance and no fallback to another instance.
- How the target distinguishes a retained visit from a new visit when two
  entries contain identical convention and payload values. Any delivery
  metadata belongs to intent; it MUST NOT be smuggled into advertised payload
  fields or create a mutual wire dependency.

This draft does not pretend that `behavior.reuse` guarantees either property.
Reporting and reading are specified here. Traversal remains incomplete until
that intent contract is settled; a runtime MUST NOT claim conforming traversal
by adding a private navigation protocol.

## Wire Protocol

The projection supplies transport and authenticated endpoint binding.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `navigation.report` | napplet -> runtime | `id`, `snapshot` |
| `navigation.report.result` | runtime -> napplet | `id`, `result` (`NavigationResult`) |
| `navigation.get` | napplet -> runtime | `id` |
| `navigation.get.result` | runtime -> napplet | `id`, `result` (`NavigationResult`) |

Request identifiers are non-empty text, unique among the endpoint's outstanding
requests. Every recognized request with a usable identifier MUST receive one
result carrying that identifier, including rejected requests. Results from the
wrong endpoint or without an outstanding request MUST be ignored. A result is
not itself acknowledged.

### Example

Assume the verified napplet advertises `napplet:profile/open` with the parameter
name `pubkey`. It reports two visited profiles, with the second current:

```json
{
  "type": "navigation.report",
  "id": "report-2",
  "snapshot": {
    "revision": 2,
    "entries": [
      {
        "id": "visit-1",
        "convention": "napplet:profile/open",
        "payload": { "pubkey": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa" },
        "title": "Alice"
      },
      {
        "id": "visit-2",
        "convention": "napplet:profile/open",
        "payload": { "pubkey": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb" },
        "title": "Bob"
      }
    ],
    "index": 1
  }
}
```

The runtime can expose Bob as current and Alice as an earlier visit. An entry
with an unadvertised `tab` payload field is rejected, even if the napplet uses
that field internally. An opaque `/profile/alice` route is not a retained intent.

## Runtime Behavior

- Bind reports and reads to their authenticated endpoint. Histories from two
  instances of the same manifest MUST remain independent.
- Make the full accepted snapshot and its status available to the shell while
  the instance lives. Before the first accepted report, return a null snapshot
  and `unreported` status.
- Keep shell workspace history separate from history within an instance. The
  meaning of a global Back command at an instance boundary is shell policy.
- Enforce limits by rejecting with `limit`, never by silently trimming entries
  or payloads. Napplets MAY prune and submit a newer snapshot.
- End live history authority when the endpoint terminates. Any retained shell
  record MUST be marked inactive. A new instance starts `unreported`; history
  does not silently transfer across instances, versions, or runtime restarts.

## Conformance Cases

| Case | Required result |
|------|-----------------|
| Read before a report | Successful read; null snapshot and `unreported`. |
| Eligible convention and advertised payload keys | Accept the complete snapshot. |
| Unadvertised convention, including a near match | Reject the complete snapshot as `unsupported`. |
| Advertisement belongs to another napplet | Reject; consult only the reporting endpoint's verified manifest. |
| An unadvertised payload key appears in any entry | Reject atomically as `unsupported`; do not strip the key. |
| No parameter names are advertised | Accept absent or empty payload; reject any payload field. |
| A payload omits an advertised name | Do not infer that it is required. |
| Payload is a scalar or list | Reject as `invalid`; retained payloads are parameter maps. |
| Advertised field has a structured JSON value | Preserve it without inventing a type restriction; the convention defines semantic validity. |
| Query or fragment appears in a stored convention | Reject; only stable convention identities are retained. |
| Payload keys fit only a union of separate advertisements | Reject; one complete advertised format must match. |
| Two visits have the same invocation | Preserve separate identifiers and the explicit current index. |
| Lower revision or changed content at equal revision | Reject as `stale` or `invalid`; preserve accepted history. |
| Invalid newer report | Preserve the snapshot, mark it `uncertain`; an identical old retransmission does not recover it. |
| Current view has no retained invocation | Accept a null index; never display an earlier entry as current. |
| Intent accepted without a navigation report | Do not append history or move the current position. |
| Second instance uses the same entry identifiers | Keep views and authority separate. |
| Snapshot exceeds a quota | Reject atomically with `limit`. |

Traversal conformance cases await the NAP-INTENT integration described above.

## Security Considerations

- History payloads can reveal searches, profiles, and drafts. A napplet SHOULD
  retain only data needed to reopen the view. The shell MUST NOT publish or share
  history with another napplet without separate user authorization.
- Treat reports as untrusted assertions and render labels as text. Advertisement
  matching does not prove that a payload is safe or that a view was rendered.
- Advertising an action does not authorize replaying its effects. Retained
  invocations MUST reopen views without repeating irreversible actions such as
  payment or publication. A destructive invocation is not a history location
  merely because its convention is advertised.
- A report grants no execution authority. Only a separate shell-mediated intent
  request can ask the napplet to act. Reports MUST NOT automatically dispatch
  their own entries.

## References

- [NAP-INTENT](NAP-INTENT.md) — advertised contracts, invocation and delivery.
- [Web projection](../projections/web.md) — host binding and domain availability.
- [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) — web projection, manifests and endpoint identity.

## Implementations

- None yet. Reporting and reading are proposed; intent-based history traversal
  still requires the integration contract above.

## Changelog

- `fd24d88` - Introduced instance-scoped location and history reports, mediated traversal with revision and failure handling, and optional shell layering with intent delivery.
- `d0d7a6a` - Restricted retained locations to advertised intent invocations and parameter names, removed the separate traversal dispatcher, required intent layering, and exposed remaining intent-delivery integration gaps.
