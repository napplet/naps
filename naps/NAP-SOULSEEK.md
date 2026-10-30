NAP-SOULSEEK
============

Runtime-Mediated Soulseek File Discovery and Downloads
-----------------------------------------------------

`draft`

**NAP ID:** NAP-SOULSEEK
**Domain:** `soulseek`
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.soulseek`; domain presence signals availability. This pointer is informational; the contract below is projection-neutral.

## Description

NAP-SOULSEEK lets a napplet search the Soulseek network, browse a peer's shared
files, and request downloads through the runtime. The shell mediates account
use, network activity, storage, content disclosure, and saving. The napplet
designs discovery and transfer views using bounded results and opaque handles.
The runtime executes Soulseek operations through its own client or a backend
such as slskd; backend credentials and addresses never become napplet authority.

The user configures accounts, shared folders, and upload policy in the shell.
This contract does not expose login, arbitrary protocol messages, local share
administration, chat, or unsolicited uploads. Downloading MUST NOT automatically
add a file to the user's shares. No other NAP is required. This domain owns
reads of its downloaded data; its handles are not URLs or grants to other domains.

## API Surface

An exposed `soulseek` domain MUST implement every operation below. `save` MAY
be unavailable as reported by `Info.canSave`. Availability does not imply a
connected account or a policy grant. Manifests name the bare `soulseek` domain.

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `info` | none | `Info` | `soulseek.info` / `soulseek.info.result` |
| `search` | `request` (`SearchRequest`) | `Query` | `soulseek.search` / `soulseek.search.result` |
| `browse` | `username` (text) | `Query` | `soulseek.browse` / `soulseek.browse.result` |
| `results` | `queryId` (text), optional `after` (integer), optional `limit` (integer) | `QueryPage` | `soulseek.results` / `soulseek.results.result` |
| `stop` | `queryId` (text) | `Query` | `soulseek.stop` / `soulseek.stop.result` |
| `download` | `fileId` (text) | `Transfer` | `soulseek.download` / `soulseek.download.result` |
| `transfers` | none | list of `Transfer` | `soulseek.transfers` / `soulseek.transfers.result` |
| `status` | `transferId` (text) | `Transfer` | `soulseek.status` / `soulseek.status.result` |
| `pause` | `transferId` (text) | `Transfer` | `soulseek.pause` / `soulseek.pause.result` |
| `resume` | `transferId` (text) | `Transfer` | `soulseek.resume` / `soulseek.resume.result` |
| `cancel` | `transferId` (text) | `Transfer` | `soulseek.cancel` / `soulseek.cancel.result` |
| `read` | `transferId` (text), `offset` (text), `length` (integer) | `Chunk` | `soulseek.read` / `soulseek.read.result` |
| `save` | `transferId` (text) | `Saved` | `soulseek.save` / `soulseek.save.result` |
| `release` | `kind` (`"query"` or `"transfer"`), `handle` (text) | `Released` | `soulseek.release` / `soulseek.release.result` |

### Schemas

All fields marked required MUST be present. Optional fields MUST be omitted
when unavailable; `null` is invalid everywhere. An empty list means no entries.
Requests with unknown fields, invalid types, or invalid ranges MUST fail with
`invalid_request` before effects. Receivers MUST ignore unknown result fields
for forward compatibility.

Byte counts and offsets use canonical unsigned decimal text (`"0"`, or digits
without a leading zero), from 0 through 18446744073709551615. This preserves
Soulseek's 64-bit file sizes across projections. Other integers are between 0
and 9007199254740991 unless a narrower range is specified. Text MUST be valid
Unicode without U+0000. Usernames are nonempty, case-preserving peer names;
napplets MUST NOT normalize them into Nostr identities. Handles and request
IDs are nonempty opaque text, compared exactly.

#### Info

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `state` | yes | `"unconfigured"`, `"offline"`, or `"online"` | Availability of the shell-selected Soulseek account. |
| `canSave` | yes | boolean | Whether user-mediated export is implemented; not an authorization grant. |
| `maxQueries` | yes | integer | Positive maximum retained queries per endpoint. |
| `maxTransfers` | yes | integer | Positive maximum retained transfers per endpoint, including terminal ones. |
| `maxResults` | yes | integer | Positive maximum files retained per query. |
| `maxPageSize` | yes | integer | Positive maximum entries per results page. |
| `maxReadBytes` | yes | integer | Positive maximum decoded bytes per read. |
| `maxTextBytes` | yes | integer | Positive maximum UTF-8 bytes per input text value, including IDs; excludes fixed field names. |
| `queryTimeoutMs` | yes | integer | Positive maximum query collection time in milliseconds from acceptance. |
| `pollIntervalMs` | yes | integer | Positive recommended minimum interval between polls of the same query or transfer. |

Limits are runtime policy, not constants of the Soulseek network. A runtime
MUST honor the limits captured when it accepts a query or transfer; tightening
policy requires explicit termination or revocation, not silent truncation.
`maxTextBytes` MUST remain fixed for the endpoint lifetime so issued handles
stay usable. `info` works offline. A policy denial returns an error rather than
fake limits.

#### SearchRequest

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `text` | yes | text | Nonempty search expression; at least one non-whitespace character. |
| `username` | no | text | Target one peer; omission searches the network. |

The runtime MUST pass the expression as a Soulseek search, never as a shell
command, backend URL, or local filename. It MAY apply Soulseek tokenization;
this NAP does not promise exact matching, ranking, or exhaustive discovery.

#### Query

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `queryId` | yes | text | Endpoint-owned handle. |
| `kind` | yes | `"search"` or `"browse"` | Operation that created the query. |
| `state` | yes | `"collecting"`, `"complete"`, `"stopped"`, or `"failed"` | `collecting` is the only nonterminal state. |
| `reason` | no | `"deadline"`, `"limit"`, `"peer_complete"`, or `"cancelled"` | Required for `complete` or `stopped`; forbidden otherwise. |
| `count` | yes | integer | Number of immutable retained entries. |
| `maxResults` | yes | integer | Effective collection bound at acceptance. |
| `maxPageSize` | yes | integer | Positive page bound captured at acceptance. |
| `timeoutMs` | yes | integer | Effective collection deadline duration at acceptance. |
| `error` | no | `Failure` | Required exactly when state is `failed`. |

#### File

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `fileId` | yes | text | Opaque handle bound to this query, peer, remote path, size, and account context. |
| `username` | yes | text | Peer claiming to offer the file; not an authenticated content author. |
| `path` | yes | text | Nonempty remote virtual path, preserved for display; never a local path. |
| `size` | yes | text | Advertised byte count. |
| `locked` | yes | boolean | Peer reports restricted access. `false` does not promise download permission. |
| `extension` | no | text | Peer-reported format hint; not a trusted media type. |
| `bitrateKbps` | no | integer | Positive peer-reported kilobits per second. |
| `durationSeconds` | no | integer | Peer-reported duration in seconds. |
| `sampleRateHz` | no | integer | Positive peer-reported sample rate. |
| `bitDepth` | no | integer | Positive peer-reported bits per sample. |

Unrepresentable or malformed peer entries MUST be discarded before issuing a
file handle. A browse listing that loses entries this way MUST end `failed`
with `backend_error`, preserving its valid partial results. Search responses
are best effort and MAY omit malformed peer entries.

#### QueryPage

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `query` | yes | `Query` | Snapshot taken atomically with this page. |
| `files` | yes | list of `File` | Entries starting at the requested zero-based offset. |
| `next` | yes | integer | Requested `after` plus the number returned. |

`after` defaults to 0. `limit` defaults to `Query.maxPageSize`; it MUST be positive and no greater than that bound. `after` greater
than the current count is invalid. Entries MUST be appended in arrival order,
never reordered or overwritten. Repeated listings of the same exact username,
path, and size MUST NOT create another entry in that query. File metadata is
the first retained observation. A size change creates a distinct entry.

`results` MUST return exactly the smaller of `limit` and `query.count - after`
entries from that snapshot, without waiting for more. `next < query.count` means another page is available. When equal,
the napplet can poll again if collecting, or stop if terminal. An empty page
from a collecting query does not mean the search finished.

#### Transfer

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `transferId` | yes | text | Endpoint-owned handle. |
| `file` | yes | `File` | Frozen source descriptor, retained independently of its query. |
| `state` | yes | `"queued"`, `"transferring"`, `"paused"`, `"complete"`, `"cancelled"`, or `"failed"` | See lifecycle below. |
| `maxReadBytes` | yes | integer | Positive read bound captured at admission. |
| `received` | yes | text | Contiguous bytes retained from offset zero, no greater than `file.size`. |
| `queuePosition` | no | integer | Positive peer-reported position; omitted when unavailable or not queued. |
| `bytesPerSecond` | no | integer | Observed nonnegative byte rate; omitted when unavailable. |
| `error` | no | `Failure` | Required exactly when state is `failed`. |

#### Chunk

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `offset` | yes | text | Requested offset. |
| `data` | yes | text | Standard padded base64, without whitespace, of the returned bytes. |
| `length` | yes | integer | Decoded byte count. |
| `eof` | yes | boolean | Whether offset plus returned length equals `file.size`. |

#### Saved, Released, and Failure

| Record | Field | Required | Type | Notes |
|--------|-------|----------|------|-------|
| `Saved` | `saved` | yes | boolean | Always `true`; the shell completed the export. |
| `Released` | `released` | yes | boolean | `true` if a live owned handle was removed; otherwise `false`. |
| `Failure` | `code` | yes | text | One of the codes in Error Handling. |
| `Failure` | `message` | yes | text | Nonempty diagnostic, not machine-readable; no credentials, addresses, or local paths. |

## Wire Protocol

All requests travel napplet → runtime; all results travel runtime → the same
authenticated endpoint. These envelopes are transport-neutral. The
[web projection](../projections/web.md) supplies their web binding.

Every request carries `type`, `id`, and exactly the parameters in its API row.
Every result carries the matching `type` with `.result` appended, the same `id`,
and **exactly one** of `value` (the API row's result) or `error` (`Failure`).
For example, `soulseek.transfers.result.value` is a list of `Transfer`.
The fields inside `request` for `search` follow `SearchRequest`; the other
operations place their parameters directly in the envelope.

The runtime MUST answer each validly enveloped request from a live authenticated
endpoint once, including rejected operations, except for the duplicate-ID rule
below. It MUST apply a finite policy
deadline to backend work and user prompts; expiration returns `timeout`.
Unrouteable or malformed envelopes MAY be discarded. No unsolicited messages
are defined: callers poll snapshots. Local listener helpers do not create wire
subscriptions.

IDs are unique per endpoint lifetime. Napplets MUST NOT reuse them. The runtime
MUST NOT execute a repeated ID again; it MAY replay the original result or
discard the duplicate. IDs do not provide recovery across endpoint teardown.
After a local timeout without a received `download` result, call `transfers` to reconcile the source
file before retrying. After ownership and policy checks, the runtime MUST serialize downloads per `fileId` and
return an existing non-released transfer for that exact handle, including a
terminal transfer. Releasing it allows a new download. A different file handle
does not imply the same content, even if its filename and size match.

### Examples

Network search, first page, and queue admission (acceptance is not completion):

```json
{"type":"soulseek.search","id":"q1","request":{"text":"field recording"}}
{"type":"soulseek.search.result","id":"q1","value":{"queryId":"s1","kind":"search","state":"collecting","count":0,"maxResults":500,"timeoutMs":30000,"maxPageSize":100}}
{"type":"soulseek.results","id":"q2","queryId":"s1","limit":10}
{"type":"soulseek.results.result","id":"q2","value":{"query":{"queryId":"s1","kind":"search","state":"collecting","count":1,"maxResults":500,"timeoutMs":30000,"maxPageSize":100},"files":[{"fileId":"f1","username":"field-recorder","path":"Recordings\\rain.txt","size":"3","locked":false}],"next":1}}
{"type":"soulseek.download","id":"q3","fileId":"f1"}
{"type":"soulseek.download.result","id":"q3","value":{"transferId":"t1","file":{"fileId":"f1","username":"field-recorder","path":"Recordings\\rain.txt","size":"3","locked":false},"state":"queued","received":"0","maxReadBytes":65536}}
```

Completed data can be read without granting local filesystem access. Here the
three downloaded bytes are `abc`; they are not an integrity proof:

```json
{"type":"soulseek.status","id":"q4","transferId":"t1"}
{"type":"soulseek.status.result","id":"q4","value":{"transferId":"t1","file":{"fileId":"f1","username":"field-recorder","path":"Recordings\\rain.txt","size":"3","locked":false},"state":"complete","received":"3","maxReadBytes":65536}}
{"type":"soulseek.read","id":"q5","transferId":"t1","offset":"0","length":3}
{"type":"soulseek.read.result","id":"q5","value":{"offset":"0","data":"YWJj","length":3,"eof":true}}
{"type":"soulseek.save","id":"q6","transferId":"t1"}
{"type":"soulseek.save.result","id":"q6","value":{"saved":true}}
```

Invalid input and independent content-disclosure denial have no requested effect:

```json
{"type":"soulseek.read","id":"q7","transferId":"t1","offset":"00","length":3}
{"type":"soulseek.read.result","id":"q7","error":{"code":"invalid_request","message":"Offset is not canonical decimal text"}}
{"type":"soulseek.read","id":"q8","transferId":"t1","offset":"0","length":3}
{"type":"soulseek.read.result","id":"q8","error":{"code":"denied","message":"Content disclosure is not authorized"}}
```

Stopping retains results; releasing destroys query handles. A late peer response
after `stop` MUST NOT append an entry. Already accepted transfers remain usable:

```json
{"type":"soulseek.stop","id":"q9","queryId":"s1"}
{"type":"soulseek.stop.result","id":"q9","value":{"queryId":"s1","kind":"search","state":"stopped","reason":"cancelled","count":1,"maxResults":500,"timeoutMs":30000,"maxPageSize":100}}
{"type":"soulseek.release","id":"q10","kind":"query","handle":"s1"}
{"type":"soulseek.release.result","id":"q10","value":{"released":true}}
{"type":"soulseek.download","id":"q11","fileId":"f1"}
{"type":"soulseek.download.result","id":"q11","error":{"code":"not_found","message":"File handle is unavailable"}}
```

A separate endpoint browses a peer and controls its own transfer. These example
limits are runtime choices, not protocol constants:

```json
{"type":"soulseek.info","id":"b0"}
{"type":"soulseek.info.result","id":"b0","value":{"state":"online","canSave":true,"maxQueries":4,"maxTransfers":8,"maxResults":500,"maxPageSize":100,"maxReadBytes":65536,"maxTextBytes":1024,"queryTimeoutMs":30000,"pollIntervalMs":1000}}
{"type":"soulseek.browse","id":"b1","username":"field-recorder"}
{"type":"soulseek.browse.result","id":"b1","value":{"queryId":"b1","kind":"browse","state":"collecting","count":0,"maxResults":500,"maxPageSize":100,"timeoutMs":30000}}
{"type":"soulseek.results","id":"b2","queryId":"b1"}
{"type":"soulseek.results.result","id":"b2","value":{"query":{"queryId":"b1","kind":"browse","state":"complete","count":1,"maxResults":500,"maxPageSize":100,"timeoutMs":30000,"reason":"peer_complete"},"files":[{"fileId":"f2","username":"field-recorder","path":"Recordings\\wind.txt","size":"3","locked":false}],"next":1}}
{"type":"soulseek.download","id":"b3","fileId":"f2"}
{"type":"soulseek.download.result","id":"b3","value":{"transferId":"t2","file":{"fileId":"f2","username":"field-recorder","path":"Recordings\\wind.txt","size":"3","locked":false},"state":"queued","maxReadBytes":65536,"received":"0"}}
{"type":"soulseek.pause","id":"b4","transferId":"t2"}
{"type":"soulseek.pause.result","id":"b4","value":{"transferId":"t2","file":{"fileId":"f2","username":"field-recorder","path":"Recordings\\wind.txt","size":"3","locked":false},"state":"paused","maxReadBytes":65536,"received":"0"}}
{"type":"soulseek.resume","id":"b5","transferId":"t2"}
{"type":"soulseek.resume.result","id":"b5","value":{"transferId":"t2","file":{"fileId":"f2","username":"field-recorder","path":"Recordings\\wind.txt","size":"3","locked":false},"state":"queued","maxReadBytes":65536,"received":"0"}}
{"type":"soulseek.cancel","id":"b6","transferId":"t2"}
{"type":"soulseek.cancel.result","id":"b6","value":{"transferId":"t2","file":{"fileId":"f2","username":"field-recorder","path":"Recordings\\wind.txt","size":"3","locked":false},"state":"cancelled","maxReadBytes":65536,"received":"0"}}
{"type":"soulseek.transfers","id":"b7"}
{"type":"soulseek.transfers.result","id":"b7","value":[{"transferId":"t2","file":{"fileId":"f2","username":"field-recorder","path":"Recordings\\wind.txt","size":"3","locked":false},"state":"cancelled","maxReadBytes":65536,"received":"0"}]}
```

If endpoint B presents endpoint A's `t1`, B receives `not_found` for `status`,
`read`, `save`, or transfer controls; its `transfers` result excludes A's jobs.
B may use A's request IDs without accessing A's results. Releasing A's handle
from B returns `released: false`. No message is delivered to A as a consequence.

### Error Handling

| Code | Meaning |
|------|---------|
| `invalid_request` | Invalid parameters, unknown fields, ranges, or enum values. |
| `denied` | Shell policy or the user refused this operation. |
| `unavailable` | No configured online account for a new search, browse, download, or resume requiring network work. |
| `unsupported` | `save` is not implemented, as reported by `info`. |
| `not_found` | Handle absent, released, revoked, or owned by another endpoint. These cases MUST be indistinguishable. |
| `invalid_state` | Operation is not allowed in the current query or transfer state. |
| `limit_exceeded` | Runtime capacity, storage, size, or rate bound prevents admission or terminates accepted work. |
| `peer_unavailable` | Peer cannot be reached within backend policy. |
| `peer_denied` | Peer refused the listing or transfer, including restricted shares. |
| `source_changed` | Peer-reported transfer size differs from the frozen source descriptor. |
| `timeout` | Request or browse deadline expired. |
| `cancelled` | User dismissed a save prompt, or release stopped an in-flight request. |
| `revoked` | Authority for already accepted work was withdrawn. |
| `backend_error` | Backend failed or produced unusable protocol data. |

The runtime MUST validate input before requested effects and check ownership
before disclosing handle state. A failed admission creates no query or transfer.
After an explicit error result, pending backend work MUST NOT later create that
query or transfer. A lost response is different from an explicit error.
After admission, backend failures appear in the retained `Query` or `Transfer`
snapshot; reading that snapshot is a successful operation with `value`.
Failure MUST NOT be reported as an empty successful listing or completed file.

## Shell Behavior

### Discovery and query lifetime

`search` starts one network or peer search. `browse` requests a flat snapshot of
one peer's shared files, including remote paths for folder grouping; it is not
a local directory enumeration. Both operations return after admission, without
waiting for peer replies. Only policy-authorized entries may be disclosed.

The runtime MUST issue handles within `Info.maxTextBytes`, so callers can
return them unchanged. It MUST bind each query to the initiating endpoint and the selected
account context. It MUST retain immutable results and file handles until query
release or endpoint teardown. Active work and retained metadata count against
the advertised bounds. A query reaching `maxResults` terminates `complete` with
reason `limit`; this is explicitly a truncated result set. Hitting a metadata
byte bound also terminates collection as `complete` with `limit`. Other
resource exhaustion after admission ends the affected query or transfer as
`failed` with `limit_exceeded`, preserving valid query entries.

A search terminates `complete` with reason `deadline` when its collection window
ends. That means local collection ended, not that every peer answered. A browse
terminates `complete` with `peer_complete` only after the peer's full accessible
listing arrives; browse timeout is `failed` with `timeout`, preserving partial
entries. Restricted files MAY be included as `locked`, but their metadata MUST
NOT be treated as permission to transfer them. Hidden shares need not appear.

`stop` freezes a collecting query as `stopped` with reason `cancelled`. For any
terminal query it returns the unchanged snapshot. The runtime MUST discard late
responses after any terminal transition. A stopped or failed query's retained
files remain eligible for a separately authorized download until release.

### Transfer lifetime

`download` accepts only a live file handle issued to the caller. The runtime
MUST bind the new transfer to its frozen peer, remote path, size, and account
context. The caller cannot replace these fields or supply a destination path.
The shell MUST authorize network transfer and storage before admission.
Queue admission does not promise peer availability, position, or completion.

| Current state | Trigger | Outcome |
|---------------|---------|---------|
| `queued` | Peer starts sending | `transferring`. |
| `queued` or `transferring` | `pause` | `paused`; cease data acquisition before returning. |
| `paused` | `pause` | Unchanged. |
| `paused` | `resume` | `queued`; recheck policy and request the same source. |
| `queued` or `transferring` | `resume` | Unchanged. |
| `queued`, `transferring`, or `paused` | `cancel` | `cancelled`; cease work and discard partial bytes before returning. |
| `complete`, `cancelled`, or `failed` | `cancel` | Unchanged; cancellation never deletes a completed file. |
| `complete`, `cancelled`, or `failed` | `pause` or `resume` | `invalid_state`; release and request a new transfer to retry. |
| Any nonterminal state | Unrecoverable peer/backend/storage failure | `failed` with the appropriate error. |
| `queued` or `transferring` | Exactly the advertised bytes received and retained | `complete`. |

Terminal transfer states are immutable. Operations on a transfer MUST be
serialized at their state transitions and effect commits; no serialization lock
may remain held while waiting for user input or backend work. Release and
revocation MUST be able to cancel pending work. When completion races
cancellation, whichever transition commits first determines the returned state. Automatic retries MAY keep a transfer
`queued`, but MUST be bounded by runtime policy; exhausting retries ends it
`failed`. Paused transfers do not expire merely for being paused.

The runtime MAY resume from a retained offset with the same peer or restart
from zero; `received` may decrease on restart. It MUST NOT combine bytes from
different peers. A changed advertised size MUST fail with `source_changed`.
Equal sizes do not prove equal content; offset resume cannot establish integrity.
`complete` means all expected bytes were received, not hash verification,
authenticity, safety, or media decodability. Zero-byte files may complete directly
from `queued`. Snapshots MUST NOT expose bytes beyond the advertised size.

`transfers` returns all non-released transfers owned by the current endpoint,
including terminal records, ordered by admission. `status` returns one snapshot.
Accepted transfers MUST survive release of their source query; its frozen
`file.fileId` remains descriptive but is no longer usable for new admissions.

### Reading, saving, and release

`read` requires a `complete` transfer and a separate shell grant to disclose its
content to this napplet. Transfer permission alone MUST NOT authorize reads.
`length` MUST be positive and at most the transfer's captured `maxReadBytes`.
`offset` greater than file size is invalid. A non-complete transfer returns
`invalid_state`. The runtime MUST return exactly the smaller of requested length and remaining bytes; at EOF it returns empty base64,
length 0, and `eof: true`. Reads MUST NOT initiate another network transfer.

`save` requires a `complete` transfer, otherwise it returns `invalid_state`.
The shell mediates destination selection and overwrite consent without exposing local paths. Success means the bytes
were exported, not merely that a dialog opened. Dismissal returns `cancelled`.
Saving does not grant byte disclosure to the napplet, launch an application,
execute the file, or change the user's shares. An exported copy is independent
of the transfer's temporary data. A timeout MUST NOT be followed by a late
export; the runtime MUST resolve a save committed before its deadline as success.
The export commit is the point at which the complete copy becomes available at
the approved destination. Before that point the runtime MUST recheck ownership,
state, and consent. A failed or cancelled save MUST remove its staging data and
MUST NOT replace an existing destination or leave a partial exported copy.

`release` stops work and removes the caller's query or transfer. Releasing a
query invalidates its file handles; releasing a transfer deletes its temporary
bytes and removes it from `transfers`. It MUST NOT delete exported copies or
other owners' data. Unknown or unowned handles return `released: false`.
The runtime MUST serialize release with query collection, download admission,
and transfer operations. It MUST reject pending operations on a released object
with `cancelled` and prevent late work from recreating it. Operations already
committed before release keep their result. Release is allowed even when new
work is denied, and all teardown effects complete before its result.

Handles are endpoint-scoped, nontransferable, and never reused within that
endpoint's lifetime. Endpoint teardown stops all its work and releases temporary
data. Reloading does not recover jobs; persistent background transfers are not
defined by this draft. Shells SHOULD make this lifetime clear before admission.
Other endpoints, including another instance of the same napplet, cannot recover
these handles. Shared backend sessions do not change this ownership rule.

On capability revocation or account-context change, the runtime MUST stop old
work, invalidate its handles, and dispose of temporary data. Pending requests
receive `revoked` while the endpoint remains reachable; later handle requests
receive `not_found`. New work MUST NOT silently continue on another account.
A temporary network outage is not an account change: local results, reads, saves,
and controls remain usable subject to policy. Network work may retry or fail.

## Security Considerations

- The runtime MUST authenticate the endpoint against the verified manifest and
  artifact through its projection. Payload identities, file handles alone, and
  manifest declarations do not grant authority.
- The shell MUST keep Soulseek passwords, backend tokens, listening ports, peer
  addresses, and local filesystem paths outside this interface. The user's
  Soulseek account, Nostr identity, and napplet identity are distinct; this NAP
  asserts no cryptographic binding between them.
- The runtime MUST gate searches, browsing, downloads, content disclosure, and
  saving separately under per-napplet policy. User consent may cover a bounded
  session rather than requiring a prompt for every call.
- Peer names, paths, metadata, and file bytes are untrusted. The runtime MUST
  store data under runtime-selected names and prevent path traversal, absolute
  path escape, symlink escape, and overwrites outside authorized destinations.
  Napplets SHOULD render peer metadata as text, never executable markup.
- The runtime MUST bound pending requests, query count, result metadata bytes,
  decompression, transfer storage, bandwidth, and peer connection work. It MUST
  reject excess admission with `limit_exceeded`; if an accepted browse cannot
  retain a complete listing, it MUST report a limit or failure, not `peer_complete`.
- Soulseek servers, peers, and a remote backend can observe account activity,
  search terms, filenames, or network addresses. The shell SHOULD explain that
  exposure before authorizing activity. This NAP makes no end-to-end encryption
  or anonymity guarantee.
- The runtime MUST NOT label downloaded content cryptographically verified on
  the strength of a username, filename, advertised size, or transfer completion.
  A locally calculated digest alone does not authenticate a peer's content.
- A backend adapter MUST scope privileged backend responses to the requesting
  endpoint; daemon-wide search history, transfers, shares, or tokens MUST NOT
  leak through enumeration or error messages.

## References

- [Nicotine+ Soulseek protocol documentation](https://nicotine-plus.org/doc/SLSKPROTOCOL.html)
  — implementation-maintained, unofficial protocol reference for searches,
  share listings, queued transfers, offsets, and the absence of chunk hashes.
- [slskd](https://github.com/slskd/slskd) — an example backend implementation,
  not a NAP binding or a required dependency.

## Implementations

No implementation of this NAP has been verified. Existing Soulseek clients and
backends do not imply support for the `soulseek` domain.

## Changelog

- `252a8f4` - Introduced runtime-mediated Soulseek search, share browsing, endpoint-owned downloads, bounded file reads, and user-mediated saving.
