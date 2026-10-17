NAP-NAVIGATION
==============

Internal Location and History
-----------------------------

`draft`

**NAP ID:** NAP-NAVIGATION
**Domain:** `navigation`
**Depends:**
- `intent` — layering · optional — the shell MAY associate `intent.deliver`
  with the selected instance's subsequent navigation reports. No intent field,
  shared wire type, or capability requirement is added to the napplet.

## Description

NAP-NAVIGATION makes a napplet's internal location and history available to its
runtime. The napplet reports its current location and ordered history. The
runtime retains the latest report for shell location displays, history menus,
and Back/Forward controls. The runtime can ask that same instance to revisit
an entry. The napplet owns the meaning of its locations and applies the request.

A **location** identifies a view inside one napplet. A **history entry** is one
visit to a location. A **snapshot** contains the reported entries and the index
of the current entry. These records belong to one authenticated running
instance, including when several instances use the same manifest.

The runtime MUST NOT infer internal location from intent acceptance, inspect a
location to select another napplet, or treat a navigation request as proof of
arrival. Reports are napplet assertions, not runtime verification of rendered
content. The contract exposes reported history; it cannot discover unreported
application state.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `report` | `snapshot` (`NavigationSnapshot`) | `NavigationResult` | `navigation.report` / `navigation.report.result` |
| `get` | none | `NavigationResult` | `navigation.get` / `navigation.get.result` |
| `onNavigate` | handler for `NavigationRequest`, returning `NavigationOutcome`; handler for follow-up `NavigationError` | `Subscription` handle | `navigation.listen`, `navigation.navigate` / `navigation.navigate.result`, follow-up `navigation.report`, `navigation.unlisten` |

`Subscription` is a local registration handle. Closing it removes the handler.
It has no wire representation. At most one navigation handler MAY be registered
per instance; the binding MUST reject a second registration without replacing
the first. The navigation handler MAY complete asynchronously. The error handler
receives a rejection of the automatic follow-up report described below, so the
napplet can repair and report its current state.

### Schemas

`NavigationLocation` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `value` | yes | text | Non-empty, napplet-defined location. Opaque and case-sensitive. |
| `title` | no | text | Plain-text display label. |
| `state` | no | any | Optional location data: null, boolean, finite number, text, or recursively composed lists and maps of text to these values. |

The runtime MUST preserve these values without parsing, normalizing, resolving,
or executing them. A value such as `/profile/alice` is local to its instance;
it is not a host address or a convention invocation. Query-like or fragment-like
text inside `value` has no special meaning to this domain. The runtime MUST NOT
apply convention URI transposition to it. The napplet MUST validate a restored
location before using it.

`NavigationEntry` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | text | Non-empty, napplet-assigned visit identifier. Unique within the instance lifetime. |
| `location` | yes | `NavigationLocation` | Location reported for this visit. |

An entry's identifier MUST remain stable while that visit is retained. A
discarded identifier MUST NOT be reused. Separate visits to equal locations
MUST have distinct identifiers. Entry identifiers confer no authority and
MUST NOT address another instance.

`NavigationSnapshot` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `revision` | yes | integer | Positive, monotonically increasing within the instance lifetime. |
| `entries` | yes | list of `NavigationEntry` | Complete retained history, oldest to newest; at least one entry. |
| `index` | yes | integer | Zero-based current entry; from `0` through entry count minus one. |

The current location is exactly `entries[index].location`. There is no second
current-location field to reconcile. Earlier entries are backward history;
later entries are forward history. The runtime has the complete reported list,
not only its length or Back/Forward availability.

Revisions need not be consecutive. A newer snapshot MUST have a greater revision
than every snapshot previously produced by that instance. Equal revisions MUST
carry identical content. A revision MUST NOT wrap or reset within an instance.

`NavigationView` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `snapshot` | yes | `NavigationSnapshot` or null | Latest accepted snapshot; null before the first report. |
| `status` | yes | `"unreported"`, `"current"`, `"pending"`, or `"uncertain"` | Runtime knowledge of the reported location, as defined below. |
| `canNavigate` | yes | boolean | Whether the runtime currently permits requests and has a registered handler. False unless status is `"current"`. |

`"current"` means the latest report is accepted and no navigation request is
unresolved. It does not attest to the napplet's rendered content. `"pending"`
means the runtime has requested a transition and still holds the prior snapshot.
`"uncertain"` means that transition ended without a usable outcome. A shell
MUST NOT display a pending or uncertain destination as a confirmed arrival.

`NavigationResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | Whether the runtime accepted the report or completed the read. |
| `view` | yes | `NavigationView` | State after processing this request. |
| `error` | no | `NavigationError` | Required exactly when `ok` is false. |

`NavigationRequest` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `revision` | yes | integer | Revision on which the runtime selected the entry. |
| `entryId` | yes | text | Identifier of an entry in that snapshot. |

The runtime sends this record only to the instance that reported it. Back,
Forward, and history-menu selection all reduce to an existing entry identifier.
The runtime MUST NOT supply a new location or a cross-instance destination.

`NavigationOutcome` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | Whether the napplet applied the requested traversal. |
| `snapshot` | no | `NavigationSnapshot` | Required exactly when `ok` is true. |
| `error` | no | `NavigationError` | Required exactly when `ok` is false. |

`NavigationError` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `code` | yes | text | One of the codes below. |
| `message` | no | text | Plain-text diagnostic; never needed for program logic. |

| Code | Meaning |
|------|---------|
| `invalid` | Malformed record, duplicate entry identifiers, invalid index, or inconsistent revision content. |
| `stale` | A report is older than accepted state, or a traversal's base revision no longer matches. |
| `denied` | Shell policy or the napplet's navigation guard rejects the operation. |
| `unavailable` | No handler, no reported history, or the requested entry can no longer be opened. |
| `limit` | Snapshot size, entry count, or request rate exceeds runtime policy. |
| `failed` | The napplet could not apply the traversal and kept its previous location. |

### Reporting and reading

The napplet MUST report its initial location after initialization and report
every committed internal navigation transition while using this domain. A
report MUST describe the actual current location and retained history, not a
proposed destination. The napplet MUST serialize its internal transitions,
including transitions requested by the runtime.

The runtime MUST validate a report and replace the stored snapshot atomically.
It MUST reject lower revisions as `stale`. An identical retransmission of the
latest revision is idempotent; different content at that revision is `invalid`.
Rejected reports MUST NOT partially change stored history. A rejected report of
a newer revision, or different content at the accepted revision, means the
runtime cannot confirm the napplet's current state: it MUST settle any pending
request without success and mark a non-null view `uncertain` until a valid newer
report arrives. It MUST also do so when a malformed report has no usable
revision. A null view remains `unreported`. An identical retransmission MUST NOT clear `pending` or
`uncertain`; only a newer snapshot can do that.

`report` returns only after the snapshot has been processed. The napplet MUST
inspect the result and, after rejection, report a valid newer snapshot of its
actual state before relying on shell navigation controls. `get` reads the
caller's view, including before the first report. It exposes no other instance.
The shell reads the same retained view internally; no second API is required
for the runtime to access the data it holds.

The napplet MUST maintain history using these rules:

| Transition | Snapshot change |
|------------|-----------------|
| Initial location | Report the initial retained history and its current index. One entry is sufficient. |
| New visit | Discard forward entries, append a new entry with a fresh identifier, and select it. |
| Replace current location | Keep the current entry identifier and history order; replace its location. |
| Traverse | Keep entries and their identifiers unchanged; select the visited entry. |
| Reset or prune | Report the full remaining history with the actual current visit retained and a valid index. Never invent unvisited entries. |

Each changed snapshot gets a new revision, including changes to labels or
location state. Repeated locations do not collapse automatically. Resetting or
pruning MUST NOT reset revision numbers or reuse discarded entry identifiers.

### Runtime-requested traversal

1. The shell selects an entry from the instance's latest `current` view under
   user action or shell policy. The runtime MUST require `canNavigate: true`.
   Selecting the current entry is a local no-op. Otherwise it captures the
   revision and entry identifier and marks the view `pending`.
2. The runtime sends one `navigation.navigate`. It MUST keep at most one
   unresolved request per instance and MUST NOT automatically replay a request
   after failure or timeout.
3. Before changing location, the napplet MUST check that the requested revision
   equals its latest produced snapshot and the entry still exists. A mismatch
   returns `stale` without applying the traversal. The napplet MAY reject a
   traversal, for example when leaving would discard unsaved work.
4. On success, the napplet returns a newer snapshot with the same entries and
   the requested entry selected. This outcome is itself a report. It MUST NOT
   append another visit or separately report the same traversal as a new visit.
   The runtime MUST validate it as a report and also verify that its entries
   equal the request's base snapshot, including location data, and its index
   selects the requested identifier. It marks the view `current` only when it
   accepts that snapshot. A mismatched outcome is malformed, not an arrival.
5. A failed outcome MUST leave the napplet at its previous location. If it
   cannot preserve that location, it MUST separately report its actual state;
   the runtime MUST NOT infer an arrival from the failed outcome.

Newer ordinary reports supersede a pending traversal. The runtime MUST settle
that request as superseded, retain the newer view, and ignore its later outcome.
The napplet MUST cancel a superseded transition before committing a subsequent
one. Thus an older request cannot overwrite a newer local navigation or an
intent-driven transition. A failed `stale` outcome without a newer accepted
report leaves the view `uncertain`; other valid failures restore `current`.
If the runtime cannot accept a successful outcome's snapshot, it MUST settle
the request without success and leave the view `uncertain`.

The runtime MUST bound waiting with a finite timeout. Timeout, malformed
outcome, handler removal while pending, or loss of the endpoint leaves the view
`uncertain`; none is a successful traversal. A late outcome MUST NOT retroactively
complete a settled request or change history. If the napplet finishes after a
timeout, it MUST report its actual location with a newer revision. The binding
MUST send an ordinary `report` of the returned snapshot after sending every
successful handler outcome. This lets a timed-out runtime recover without
mistaking a late outcome for request success; an already accepted revision is
an idempotent report. The binding MUST expose a rejected follow-up report to the
napplet's registered error handler. A `stale` follow-up report means a newer
snapshot already won; it MUST NOT roll that snapshot back. A fresh accepted
report restores `current`.

## Wire Protocol

Messages use `navigation.*` discriminants. The projection supplies transport
and authenticated endpoint binding.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `navigation.report` | napplet -> runtime | `id`, `snapshot` |
| `navigation.report.result` | runtime -> napplet | `id`, `result` (`NavigationResult`) |
| `navigation.get` | napplet -> runtime | `id` |
| `navigation.get.result` | runtime -> napplet | `id`, `result` (`NavigationResult`) |
| `navigation.listen` | napplet -> runtime | none |
| `navigation.unlisten` | napplet -> runtime | none |
| `navigation.navigate` | runtime -> napplet | `id`, `request` (`NavigationRequest`) |
| `navigation.navigate.result` | napplet -> runtime | `id`, `outcome` (`NavigationOutcome`) |

Each initiator assigns a non-empty text `id`, unique among its outstanding
requests to that endpoint. A result MUST echo that `id`. Runtime navigation
identifiers MUST NOT be reused during the instance lifetime, so a late outcome
cannot match another request. Outcomes from another endpoint or without an
outstanding request MUST be ignored. A result is never itself acknowledged.

The binding sends `listen` after registering the handler and `unlisten` when
the registration closes. Both are idempotent and carry no `id`. The runtime
MUST NOT request traversal before `listen` or after `unlisten`. A binding that
receives an already dispatched request after handler removal MUST reply with
`unavailable`. Registering or removing a handler never erases the snapshot.

The binding MUST turn a handler's return value into one correlated outcome.
A handler exception produces `failed`; the napplet remains responsible for
reporting any actual location change. Every recognized napplet request with a
usable `id` MUST receive one result, including invalid or denied requests.

### Example

The napplet reports two visited locations, with the second current:

```json
{
  "type": "navigation.report",
  "id": "report-2",
  "snapshot": {
    "revision": 2,
    "entries": [
      { "id": "visit-1", "location": { "value": "/feed", "title": "Feed" } },
      { "id": "visit-2", "location": { "value": "/profile/alice", "title": "Alice" } }
    ],
    "index": 1
  }
}
```

After accepting the report and receiving `navigation.listen`, the runtime can
display Alice as current and Feed as a Back destination. When the user selects
Back, it sends:

```json
{
  "type": "navigation.navigate",
  "id": "back-1",
  "request": { "revision": 2, "entryId": "visit-1" }
}
```

After the napplet opens Feed, its outcome has `ok: true` and a snapshot with
revision `3`, the same two entries, and index `0`. The runtime now exposes Feed
as current and Alice as a Forward destination. A subsequent new visit discards
Alice's forward entry before appending the new entry.

## Runtime Behavior

- The runtime MUST bind every report, read, registration, and outcome to the
  authenticated endpoint. Caller-supplied manifest or instance claims MUST NOT
  select the scope. Different instances MUST have independent histories.
- The runtime MUST make the latest accepted snapshot and its status available
  to the shell while the instance remains alive, even without a handler.
- Before any report, the view is `unreported`, its snapshot is null, and
  `canNavigate` is false. Empty history MUST NOT masquerade as a known location.
- The shell MAY display a current location, Back/Forward controls, or a history
  menu. It MUST distinguish unavailable or uncertain state and MUST NOT expose
  controls as usable when `canNavigate` is false.
- Shell workspace history and history within an instance are separate. This
  domain does not choose what a global Back command does at an instance boundary.
- The runtime MAY bound entry count, serialized snapshot size, and report rate.
  It MUST reject excessive reports with `limit`, not silently truncate or alter
  accepted history. Napplets MAY prune and report a newer snapshot.
- Endpoint termination ends the instance's revision and entry-identifier scope.
  The runtime MUST remove live navigation authority for that endpoint. Any
  retained shell record MUST be marked inactive and MUST NOT be presented as a
  live current location. Retention and deletion are user-controlled shell policy.
- A new instance starts `unreported`. This contract does not silently attach an
  old history to a restarted instance, persist it across runtime restart, or
  make location values portable across napplets or artifact versions.

## Relationship to NAP-INTENT

[NAP-INTENT](NAP-INTENT.md) selects a handler and accepts responsibility for
convention delivery. NAP-NAVIGATION reports what a particular instance actually
did internally. Both domains remain usable independently.

A handler receiving `napplet:profile/open` can validate its payload, show the
profile, and report that view as a new visit or a replacement. It chooses which
transition fits its application. The shell MUST NOT create a navigation entry
merely because `intent.invoke` returned `ok: true` or `intent.deliver` was sent.
An intent might edit data without navigating, fail inside the target, or arrive
after another transition.

The optional layering dependency permits the shell to associate delivery with
the target instance it selected. It defines no correlation identifier or
completion signal for the source napplet. A navigation report MUST NOT be
treated as an intent receipt. Intent sender identity, handler selection,
convention identity, URI normalization, and payload semantics stay with their
existing owners; no intent wire type is embedded or redefined here.

Revisiting an entry MUST NOT re-invoke the intent that originally led to it.
The runtime asks that same instance to restore its location. This avoids
replaying an action or choosing a different handler after user defaults change.
Cross-napplet opens continue to use NAP-INTENT under its own policy.

## Conformance Cases

These cases apply independently to each authenticated instance.

| Case | Required result |
|------|-----------------|
| Read before the first report | `ok: true`; null snapshot, `unreported`, `canNavigate: false`. |
| Report visits A, B, C with index 1 | B is current; A is backward history; C is forward history. |
| Visit D from B | Report A, B, D with index 2; discard C and never reuse its identifier. |
| Replace B's location | Preserve B's identifier and list position; advance revision. |
| Duplicate latest report | Return the retained view without changing revision or clearing uncertainty. |
| Lower revision or equal revision with changed content | Reject as `stale` or `invalid`, respectively; do not overwrite the snapshot. |
| Same location visited twice | Retain separate visit identifiers; no automatic deduplication. |
| Back request succeeds | Accept a newer snapshot selecting the requested entry, with the same entry list; do not append a visit. |
| Guard rejects Back | Keep the prior snapshot; do not claim the target was reached. |
| Local navigation overtakes Back | Accept the newer local report; the pending request cannot later overwrite it. |
| Back times out | Mark `uncertain`; disable traversal until a newer accepted report; ignore the late outcome as a completion. |
| Successful outcome arrives after timeout | Its ordinary follow-up report can refresh the view; the timed-out request remains unsuccessful. |
| Intent accepted without target navigation | No new entry and no change to current location. |
| Second instance uses the same manifest and entry identifiers | Keep histories and traversal authority separate. |
| Report exceeds a quota | Reject atomically with `limit`; never silently trim the snapshot. |

## Security Considerations

- History can reveal private searches, profiles, and drafts. A napplet SHOULD
  report only the location data needed to identify or restore its view and
  SHOULD NOT include secrets or credentials. The shell MUST NOT publish or
  share history with another napplet without separate user authorization.
- A report proves only what its authenticated source asserted. Labels and
  location data MUST be treated as untrusted data; labels MUST render as text.
- The runtime MUST NOT fetch, execute, or invoke a location value, even when
  it resembles an external address or convention URI. Reporting a location
  grants no network, storage, or cross-napplet authority.
- The runtime MUST mediate traversal under user action or shell policy. A
  napplet cannot use this surface to inspect or navigate another instance.
- Traversal restores a view. Napplets MUST NOT replay irreversible actions
  such as publishing or payment simply because their history entry is revisited.
- Rate and size limits SHOULD prevent history reports from exhausting runtime
  resources. The shell SHOULD expose history retention and clearing to the user.

## References

- [NAP-INTENT](NAP-INTENT.md) — independent handler selection and convention delivery.
- [Web projection](../projections/web.md) — host binding and domain availability.
- [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) — normative web projection, verified manifests, and endpoint identity.

## Implementations

- None yet. This is a proposed contract; it does not establish runtime support.

## Changelog

- `fd24d88` - Introduced instance-scoped location and history reports, mediated traversal with revision and failure handling, and optional shell layering with intent delivery.
