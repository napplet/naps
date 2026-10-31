NAP-NAVIGATION
==============

Runtime-Maintained Intent History
--------------------------------

`draft`

**NAP ID:** NAP-NAVIGATION
**Domain:** `navigation`
**Depends:**
- `intent` — capability · required — locations are validated against advertised
  intent contracts; the runtime replays a retained invocation through
  NAP-INTENT selected-instance delivery, and the napplet handles it through
  `intent.onDelivery`.

## Description

The runtime maintains a napplet instance's history and current position. The
napplet reports its committed location as an advertised convention identity and
payload. The runtime validates the report, records the visit, and makes the
complete retained history available to the shell and that napplet.

Navigation requests are intents. For Back, Forward, or a history-menu selection,
the runtime delivers the retained invocation to the same live instance. The
napplet handles it through its ordinary intent handler and reports the resulting
location. Delivery alone never moves the current position.

The napplet designs its views. The shell mediates navigation. The runtime owns
the history list, visit identifiers, current position, and retention policy.
The napplet does not submit a history list or select a retained entry by index.

A **location** is an advertised intent invocation representing a committed view.
A **visit** is one occurrence of that location in history. Two visits can contain
the same invocation. All state belongs to one authenticated running instance,
even when multiple instances use the same manifest.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `report` | `report` (`NavigationReport`) | `NavigationResult` | `navigation.report` / `navigation.report.result` |
| `get` | none | `NavigationResult` | `navigation.get` / `navigation.get.result` |

There is no separate navigation request handler, listener registration, or
traversal outcome operation. [NAP-INTENT](https://github.com/napplet/naps/pull/128)
owns delivery; navigation adds no fields to the intent payload or delivery.

### Schemas

`NavigationLocation` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `convention` | yes | text | Stable, queryless, fragment-free identity advertised by this napplet. |
| `payload` | no | map of text to any | Contains only advertised parameter names; values must be JSON values. |
| `title` | no | text | Plain-text display label; not an invocation parameter. |

JSON values are null, boolean, finite number, text, or recursively composed
lists and maps of text to JSON values. No opaque internal route, separate router
state, handler selector, or lifecycle hint is retained alongside the invocation.
A title is display metadata and MUST NOT be inserted into the intent payload.

`NavigationReport` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `revision` | yes | integer | Positive, increasing sequence assigned by the napplet to current-location reports. |
| `location` | yes | `NavigationLocation` or null | Actual committed view, or null when it cannot be represented by an eligible invocation. |
| `mode` | no | `"push"` or `"replace"` | Ordinary report disposition; defaults to `"push"`. A report that settles a pending revisit takes precedence. MUST be absent when location is null. |

Revisions order reports, not runtime history mutations. Each new report MUST
use a revision greater than every report previously produced by that instance,
including reports that were rejected. A retry MUST reuse the same revision and
identical content. Revisions MUST NOT wrap or reset within the instance lifetime.

A report MUST NOT contain a history list, current index, or visit identifier.
The runtime MUST reject these caller-supplied history controls as `invalid`.
A report carrying `mode` with a null location is `invalid`.

A report never says why a transition happened. The napplet reports a location
reached through an intent exactly as it reports local navigation. A guard that
declines an intent MAY reaffirm the unchanged location with a newer `replace`
report.

`NavigationEntry` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | text | Non-empty, runtime-assigned visit identifier, never reused in the instance lifetime. |
| `location` | yes | `NavigationLocation` | Validated invocation and display label for the visit. |

A retained visit keeps its identifier. Repeated invocations do not collapse into
one visit. Entry identifiers are scoped to the instance; they grant no authority
to inspect or navigate another instance.

`NavigationView` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `entries` | yes | list of `NavigationEntry` | Complete history retained by the runtime, oldest to newest. |
| `index` | yes | integer or null | Current retained entry, or null when none represents the current view. |
| `reportRevision` | yes | integer or null | Last accepted report revision; null before the first accepted report. |
| `status` | yes | `"unreported"`, `"current"`, `"pending"`, or `"uncertain"` | Runtime knowledge of the current location. |

A non-null index MUST identify an entry. The reported current location is
`entries[index].location`. An empty list requires a null index. A null index
MUST NOT be displayed as an earlier entry still being current.

`current` means the latest current-location report was accepted; it does not
prove rendered content or intent completion. `pending` means a history revisit
has been requested and its outcome is not yet established by a report.
`uncertain` means the runtime cannot treat its last accepted position as current.
`unreported` means no current-location report has been accepted.

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
| `invalid` | Malformed report, non-map payload, `mode` with a null location, or changed content at the accepted revision. |
| `unsupported` | Unadvertised convention or payload parameter name. |
| `stale` | Report revision is older than the accepted report. |
| `denied` | Shell policy rejects the operation. |
| `limit` | Location size, history capacity, or report rate exceeds runtime policy. |

### Eligible locations

The runtime MUST use the eligible intent contracts of the verified manifest
bound to the reporting endpoint. A report MUST NOT select a different manifest
or instance. Eligibility and manifest parsing belong to
[NAP-INTENT](NAP-INTENT.md#manifest-catalog-contract).

For each non-null location:

1. `convention` MUST equal an eligible advertised identity exactly. The router
   MUST NOT parse, normalize, prefix-match, or wildcard-match that identity.
2. `payload`, when present, MUST be a map whose keys are advertised parameter
   names, compared exactly.
3. An absent or empty payload is allowed. An advertisement with no parameter
   names permits no payload fields in history.
4. Names are an allowlist, not required fields. They do not declare types, value
   ranges, or nested schemas. The target validates convention semantics before
   acting on an intent.

If one convention has multiple eligible advertisements, the payload MUST fit
at least one complete advertised parameter list. The runtime MUST NOT union
separate lists into an undeclared format.

These are history-retention restrictions. They do not narrow ordinary
NAP-INTENT invocation or redefine the manifest's `i` tag as a schema.
A payload accepted for intent delivery may still be ineligible for history.

Invocation query sugar MUST be transposed before reporting. Stored identities
have no query or fragment. Query values remain text; structured values use the
explicit payload. Navigation defines no alternative URI normalization.

### Reporting and reading

The napplet MUST report its initial committed view and subsequent committed
navigation while using this domain. It MUST serialize its transitions and
reports. A report describes actual state, never a proposed destination. A view
that cannot be expressed by an eligible invocation MUST be reported as null.

The runtime MUST validate the complete report before applying a history change.
It MUST reject unsupported fields rather than silently stripping them. It MUST
reject older revisions as `stale`. A retransmission of the last accepted report
with identical content MUST return the retained view without appending a visit,
moving the index, or clearing pending/uncertain status. Changed content at that
revision is `invalid`. Map key order is insignificant; list order, value types,
field presence, and values are significant.

The runtime MUST track the highest usable report revision it has observed,
including rejected reports. A report below that revision is stale, except that
an identical retransmission of the last accepted report remains a no-op. After
a rejection, only a valid report above that observed revision can recover
current state. Repeating a rejected revision MUST NOT accept different data or
clear uncertainty; the napplet must issue a newer report. This prevents an
older in-flight report from overwriting knowledge of a later failed update.

A rejected report MUST NOT partially change history. Rejection of a newer
report, changed content at the accepted revision, or a report without a usable
revision MUST end any pending revisit without success and mark an existing view
`uncertain`. A view with no accepted report remains `unreported`. Only a valid
newer report can restore `current`.

For a report that does not settle a pending history revisit:

| Report | Runtime history change |
|--------|------------------------|
| Non-null location, `push`, equal to the selected visit's invocation | Apply as `replace`; do not create a consecutive duplicate. |
| Non-null location, `push` | Discard entries after the current index, append a new visit, and select it. With a null index, append to the retained list. |
| Non-null location, `replace` | Replace the selected visit's location without changing its id or position. With no selected visit, append and select a new visit. |
| Null location | Preserve history and set index to null; do not create an unadvertised entry. |

The runtime MUST record its resulting view atomically with the accepted report
revision. `report` returns that view. The napplet MUST inspect rejections and
submit a valid newer report of actual state before relying on shell history.
`get` returns only the caller's view; the shell reads the same runtime-owned
state internally. Neither read infers state from the napplet's router.

## Revisiting History Through Intent

1. Under user action or shell policy, the runtime selects a retained entry from a view whose status is `current`
   in the same live instance. It MUST refuse a revisit when intent is absent,
   policy denies it, or the view is not current. Selecting its current entry is a
   no-op. At most one history revisit MAY be pending per instance.
2. Before dispatch, the runtime records the selected visit id and the highest
   report revision it has observed. It marks the view `pending` and replays the
   entry's convention and payload to that endpoint through
   [NAP-INTENT selected-instance delivery](https://github.com/napplet/naps/pull/128).
   The delivery's `sender` is the napplet's own catalog identifier. The runtime
   MUST NOT add the entry id, revision, or other private fields to the payload.
3. The napplet handles the intent normally, including its navigation guard. If
   it commits a transition, it reports the resulting location like any other.
   It does not choose a history index or need to know whether this was Back,
   Forward, a menu selection, or another intent.
4. The first accepted report above the recorded revision settles the revisit.
   If its convention and payload equal the selected entry, the runtime selects
   that recorded visit. It MUST NOT append a visit or discard forward history.
   It MAY update that entry's display title from the report. `mode` does not
   override this rule.
5. A settling report of a different location, or null, does not establish the
   requested destination. The runtime applies the ordinary report rules to the
   actual location and ends the revisit without success.

Equality for invocation matching excludes the title. Payload field presence and
value types remain significant. The recorded visit id selects the exact retained
visit even when several entries hold identical convention and payload values.
The napplet never guesses which duplicate to use.

Settlement is by value, not cause. If the napplet reaches the target location
by local navigation while the revisit is pending, the revisit is reached: the
napplet is showing that view. A guard that declines and reaffirms its unchanged
location with `replace` settles the revisit without success. Intent acceptance
or delivery without a report changes no history position. A report is not a
completion receipt exposed to any napplet.

The runtime MUST bound a pending revisit by a finite timeout. Timeout, delivery
failure, or endpoint loss ends it without success. A still-live view becomes
`uncertain` after timeout or an ambiguous delivery failure; a definite failure
before delivery leaves the previous view current. The runtime MUST NOT
silently replay a failed history request.

Once a revisit is settled or ended, no later report completes it. If a late
handler actually commits a transition, its newer valid report records the actual
location under the ordinary report rules. It may therefore create a new visit,
but MUST NOT retroactively complete the old request or move back to its old
visit id. A report older than accepted state is still
`stale`. Napplets SHOULD cancel superseded asynchronous transitions before they
commit; if cancellation fails, they MUST report the actual resulting state.

The runtime MUST NOT switch handlers, substitute another instance, or restart
an instance to perform a history revisit. Cross-napplet opens continue to use
ordinary intent handler resolution.

## Wire Protocol

The projection supplies transport and authenticated endpoint binding.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `navigation.report` | napplet -> runtime | `id`, `report` (`NavigationReport`) |
| `navigation.report.result` | runtime -> napplet | `id`, `result` (`NavigationResult`) |
| `navigation.get` | napplet -> runtime | `id` |
| `navigation.get.result` | runtime -> napplet | `id`, `result` (`NavigationResult`) |

Envelope request identifiers are non-empty text, unique among the endpoint's
outstanding requests. Each recognized request with a usable identifier MUST
receive one correlated result, including rejected requests. Wrong-endpoint and
unmatched results MUST be ignored. A result is not acknowledged. Envelope ids
and runtime visit ids are distinct scopes.

### Example

Assume a napplet advertises `napplet:profile/open` with parameter `pubkey`.
After opening Bob through a local interaction, it reports:

```json
{
  "type": "navigation.report",
  "id": "report-2",
  "report": {
    "revision": 2,
    "location": {
      "convention": "napplet:profile/open",
      "payload": { "pubkey": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb" },
      "title": "Bob"
    }
  }
}
```

If Alice was its first visit, the runtime now retains Alice then Bob, with Bob
current. Back replays Alice's stored invocation to this instance as an intent.
After opening Alice, the napplet reports Alice's location with revision 3. The
report matches the pending target, so the runtime selects Alice's existing visit
and keeps Bob as Forward history. The napplet submits neither list nor index.

## Runtime Behavior

- Bind reports, reads, visit ids, and pending requests to their
  authenticated endpoint. Instances of the same manifest remain independent.
- Before a report, expose empty entries, null index, null report revision, and
  `unreported`. Expose the full retained view to the shell while the instance
  lives; distinguish pending and uncertain state from confirmed current state.
- The runtime owns retention. It MAY prune history under documented shell policy
  and user controls. It MUST preserve the current visit and any pending target;
  clearing either MUST end the pending request and make the absence explicit in
  the returned view. Pruning MUST NOT change stored invocation data or reuse ids.
- The runtime MAY bound entry count, location size, and report rate. It MUST reject a report with `limit` if it cannot accept it under
  its retention policy; it MUST NOT truncate payloads. Pruning unrelated retained
  visits is a runtime history decision, not a napplet-supplied snapshot edit.
- Retention changes do not change `reportRevision`, which identifies the last
  accepted napplet report. The view returned by every operation reflects the
  actual retained list and position.
- Keep shell workspace history separate from history inside an instance. The
  meaning of a global Back command at an instance boundary remains shell policy.
- Endpoint termination ends live navigation authority. Any retained shell record
  MUST be inactive. A new instance starts unreported; history does not silently
  transfer across instances, versions, or runtime restarts.

## Conformance Cases

| Case | Required result |
|------|-----------------|
| Read before a report | Empty history, null index/revision, `unreported`. |
| First eligible location report | Runtime creates and selects the first visit. |
| New local visit | Runtime assigns a fresh id, truncates forward history, and appends. |
| Replace current location | Preserve current visit id and list position. |
| Same invocation visited twice, not consecutively | Retain separate runtime visit ids. |
| `push` equal to the selected visit's invocation | Apply as `replace`; no consecutive duplicate. |
| `mode` with a null location | Reject as `invalid`. |
| Unadvertised convention or parameter name | Reject atomically as `unsupported`. |
| Convention belongs only to another manifest | Reject using the reporting endpoint's own verified manifest. |
| Empty advertised parameter list | Permit absent/empty payload, reject any payload field. |
| Advertised name is omitted | Do not infer a required field. |
| Advertised field contains structured JSON | Preserve it without inventing a type restriction. |
| Payload is a scalar/list, or identity has query/fragment | Reject; history uses normalized identities and parameter maps. |
| Keys fit only the union of separate advertisements | Reject; one complete advertised format must match. |
| Duplicate accepted report | No appended visit, cursor movement, or clearing of pending/uncertain state. |
| Lower revision / changed equal revision | Reject as `stale` / `invalid`; preserve history. |
| Rejected newer report | Preserve history, end pending revisit, mark state uncertain. |
| Older in-flight report arrives after a newer rejection | Reject as stale; require a revision above the observed high-water mark to recover. |
| Napplet supplies entries, index, or a visit id | Reject; history controls belong to the runtime. |
| Current view cannot be represented | Accept null location; retain history with null current index. |
| Back emits an intent but no report arrives | Do not move the position; finite timeout leaves uncertainty. |
| Back's settling report equals the selected invocation | Select the recorded visit; preserve forward entries even with identical duplicate invocations elsewhere. |
| Local navigation reaches Back's target while pending | Settle the revisit and select the recorded visit; the napplet shows that view. |
| Guard declines and reaffirms current location with replace | Preserve the current visit; end pending revisit without success. |
| Handler redirects elsewhere | Record actual location normally; do not claim the requested visit was reached. |
| Local navigation elsewhere overtakes Back | End the revisit; a later report of the target is ordinary navigation. |
| Late successful handler reports after timeout | Record actual location normally; old request stays unsuccessful. |
| Report carries an unknown field | Reject as invalid; do not strip it. |
| Intent acceptance without navigation report | No history mutation. |
| Second instance uses the same manifest | Independent history, position, id scopes, and traversal authority. |
| Retention prunes old visits | Returned view shows the retained list; current/pending targets remain valid or are explicitly cleared. |
| Report exceeds available capacity | Reject with limit; do not partially append or truncate payload. |

## Security Considerations

- History may expose searches, profiles, and drafts. Retain only invocation data
  needed to reopen views. The shell MUST NOT share it with another napplet or
  publish it without separate user authorization.
- Reports are untrusted assertions. Render labels as text. Matching an
  advertisement does not prove semantic validity, safe replay, or rendering.
- A retained invocation MUST reopen a view without repeating irreversible effects
  such as payment or publication. An advertised destructive action is not a
  history location merely because its parameter names match.
- Reporting grants no execution authority. Only a separate shell-mediated intent
  request asks a napplet to act. Reports MUST NOT dispatch their own locations.
- Settling a revisit by value grants no authority and does not prove the view
  rendered. Runtime history controls remain subject to user action and shell
  policy.

## References

- [NAP-INTENT selected-instance delivery](https://github.com/napplet/naps/pull/128) — replays a retained invocation to the same live instance without wire changes.
- [Web projection](../projections/web.md) — host binding and domain availability.
- [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) — web projection, manifests, and authenticated endpoints.

## Implementations

- None yet. This draft requires the companion intent change above.

## Changelog

- `fd24d88` - Introduced instance-scoped location and history reports, mediated traversal with revision and failure handling, and optional shell layering with intent delivery.
- `d0d7a6a` - Restricted retained locations to advertised intent invocations and parameter names, removed the separate traversal dispatcher, required intent layering, and exposed remaining intent-delivery integration gaps.
- `896c0dc` - Moved visit creation, ordering, retention and position to the runtime; replaced snapshot submissions with current-location reports and correlated intent delivery ids for revisits, supersession and timeout recovery.
