# NAP-NZB

Runtime-mediated NZB downloads
-----------------------------

`draft`

**NAP ID:** NAP-NZB
**Domain:** `nzb`
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)), informational:** `window.napplet.nzb`; domain presence signals availability.

## Description

The shell provides NZB download jobs. An NZB is an XML document identifying
Usenet articles that contain file segments; see the
[NZB format specification](https://sabnzbd.org/wiki/extra/nzb-spec).
The napplet submits an NZB, observes jobs, and requests job actions. The runtime
owns downloader selection, credentials, article retrieval, storage, repair,
extraction, and configured processing. It may implement the downloader itself
or adapt a user-configured service such as NZBGet or SABnzbd. The same contract
applies in each case; backend RPC methods and paths never become napplet APIs.

The common surface is job management. Reading completed output is optional:
a shell can control a remote downloader without having access to its files.
This NAP has no dependencies on other NAP domains. Article browsing/posting,
indexer search, global downloader administration, and media playback are outside
its scope. No other domain gains a new NZB source form through this proposal.

Every runtime MAY omit `nzb`. A runtime exposing it MUST implement discovery,
submission, job listing, and status; configured downloaders may be read-only.
Optional actions and file reads MUST be advertised explicitly. Manifests name
`nzb`; domain presence and advertised features are not permission grants.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `downloaders` | none | `DownloadersResult` | `nzb.downloaders` / `nzb.downloaders.result` |
| `add` | `request` (`AddRequest`) | `AddResult` | `nzb.add` / `nzb.add.result` |
| `jobs` | `request` (`JobsRequest`) | `JobsResult` | `nzb.jobs` / `nzb.jobs.result` |
| `status` | `jobId` (text) | `Job` | `nzb.status` / `nzb.status.result` |
| `control` | `request` (`ControlRequest`) | `ControlResult` | `nzb.control` / `nzb.control.result` |
| `files` | `request` (`FilesRequest`) | `FilesResult` | `nzb.files` / `nzb.files.result` |
| `read` | `request` (`ReadRequest`) | `ReadResult` | `nzb.read` / `nzb.read.result` |

### Common values

Omitted optional fields use their stated defaults. `null` is valid only where
explicitly allowed. Unknown request fields MUST produce `invalid_request`;
receivers MUST ignore unknown result fields and unsupported advertised enum
values. Each advertised list contains distinct values.

| Value | Type | Rules |
|-------|------|-------|
| Identifier | text | Nonempty opaque shell-issued value. `downloaderId`, `jobId`, `categoryId`, and `fileId` are scoped to the current user and verified napplet identity. Possession alone grants no authority. |
| Count / offset / length | integer | 0–9007199254740991; positive where stated. No fractions or numeric text. |
| Timestamp | integer | Nonnegative Unix time in whole seconds. |
| Base64 | text | Standard padded [RFC 4648 §4](https://www.rfc-editor.org/rfc/rfc4648.html#section-4) encoding without whitespace; lengths count decoded bytes. Invalid alphabet, padding, or pad bits are rejected. |
| `OperationKey` | text | Nonempty caller-chosen identifier for one mutation; separate from wire `id`. See [Mutation replay](#mutation-replay). |

Identifiers MUST NOT expose credentials, connection URLs, host filesystem paths,
or raw backend IDs. The runtime MUST NOT reassign a handle to an unrelated
backend, account, job, or file when backend identifiers are recycled. Handles
for retained jobs SHOULD survive endpoint reloads under the same verified
napplet identity and user; otherwise the caller rediscovers them with `jobs`.

### Downloader discovery

`DownloadersResult`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `downloaders` | yes | list of `Downloader` | Only discoverable configured downloaders, with unique IDs. Empty means none are exposed. Order has no meaning. |

`Downloader`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `downloaderId` | yes | text | Identifies one shell-owned engine or configured downloader account. |
| `label` | yes | text | User-facing label. |
| `canAdd` | yes | boolean | Whether the adapter can submit and track jobs. False makes `add` unsupported. |
| `sourceKinds` | yes | list of `"bytes"` or `"url"` | MUST include `"bytes"` when `canAdd` is true; empty otherwise. |
| `addOptions` | yes | list of `"categoryId"`, `"priority"`, `"startPaused"`, or `"password"` | Optional submission fields the adapter can honor. |
| `categories` | yes | list of `Category` | Allowed category choices; empty when category selection is unsupported. |
| `actions` | yes | list of `JobAction` | Actions the adapter can implement for at least some jobs. |
| `canReadFiles` | yes | boolean | Whether the shell can enumerate and read completed output through authorized file mappings. |
| `maxNZBBytes` | yes | integer | Positive bound on decoded NZB input, including fetched URL content. |
| `maxPageSize` | yes | integer | Positive maximum for `jobs` and `files` page sizes. |
| `maxReadBytes` | yes | integer | Positive read bound when `canReadFiles` is true; zero otherwise. |
| `replaySeconds` | yes | integer | Positive minimum mutation-record lifetime after dispatch terminates; reservation begins before dispatch. |

`Category`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `categoryId` | yes | text | Shell-issued choice, mapped internally to backend configuration. |
| `label` | yes | text | Human-readable category name. |
| `description` | yes | text | Explains relevant destination and processing behavior without exposing paths or secrets. |

Discovery describes the configured adapter's enforceable features, not a product
name's theoretical features. A remote control-only adapter returns
`canReadFiles: false`. Native engines use the same discovery fields. A caller
MUST NOT infer behavior from `label` or require a particular downloader brand.
Runtime limits may change; an operation exceeding a current limit fails with
`limit_exceeded`. Unsupported features fail with `unsupported` before dispatch.

### Submission

`AddRequest`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `downloaderId` | yes | text | Explicit destination; no silent fallback to another downloader. |
| `operationKey` | yes | `OperationKey` | One logical submission. |
| `source` | yes | `NZBSource` | Exactly one NZB document. |
| `name` | no | text | Nonempty display-name suggestion, not a path or command. Omission uses a runtime-selected label. |
| `categoryId` | no | text | Advertised category. Omission uses the user's configured default. |
| `priority` | no | `"low"`, `"normal"`, or `"high"` | Relative scheduling preference; omission uses the configured default. Never overrides a pause, quota, or policy. |
| `startPaused` | no | boolean | Omitted means false. True requires the download hold to exist before any article retrieval. |
| `password` | no | text | Nonempty archive password; omitted means none supplied by this request. Never returned in status or errors. |

`NZBSource`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `kind` | yes | `"bytes"` or `"url"` | Selects the source form. |
| `dataBase64` | for `"bytes"` | text | One uncompressed NZB XML document. Forbidden for `"url"`. |
| `url` | for `"url"` | text | Absolute HTTPS URL without userinfo or fragment. Forbidden for `"bytes"`. |

An explicitly supplied optional submission field MUST be supported even if its
value resembles a default. Unknown category IDs produce `not_found`; the shell
MUST NOT silently select a substitute. Defaults, categories, and processing
remain user-controlled. The shell MUST authorize the effective configuration,
including retention and configured processing, before submission.

The runtime MUST validate and bound the NZB before handing it to a downloader.
It MUST accept well-formed NZB documents with the documented namespace and at
least one file with article segments, subject to size and policy limits. It
MUST disable external entity resolution and network DTD loading, reject internal
entity declarations, and bound XML depth and element counts. An external
DOCTYPE declaration MAY be accepted inertly. Metadata extensions are data and
MUST NOT select arbitrary scripts, paths, credentials, or network targets.
Archives containing NZBs and arbitrary backend upload formats are unsupported.

For URL sources, the runtime MUST apply fetch policy to the initial target and
every redirect, enforce response/decompression limits, and validate the obtained
NZB under the same rules as bytes. Fetching MUST NOT inherit unrelated ambient
credentials. Query tokens MUST be treated as secrets. An adapter may delegate
fetching only when it can enforce these obligations; otherwise the shell fetches
the NZB and submits bytes. Fetch failure MUST NOT create a job. A successfully
fetched document is fixed for that mutation's replay record.

`AddResult`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `job` | yes | `Job` | Identified, accepted logical download job. May already have advanced or failed. |
| `replayUntil` | yes | timestamp | Earliest expiry of this mutation's replay record; the runtime may extend it. |

An add success means the runtime or backend accepted an identifiable job, not
that any file was downloaded. A duplicate rule may create a `skipped` job. If
the backend declines submission without a job, return `backend_rejected`.
The runtime MUST NOT invent a queued job from an uncorrelated success boolean,
or identify a job by matching only its display name.

### Jobs and state

`JobsRequest`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `downloaderId` | yes | text | Downloader to inspect. |
| `after` | no | text | Exclusive last `jobId` from the preceding page. |
| `limit` | no | integer | Positive; defaults to `min(100, maxPageSize)`. |

`JobsResult`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `jobs` | yes | list of `Job` | First matching jobs after `after`, in ascending UTF-8 byte order of `jobId`, up to `limit`. |
| `next` | yes | text or null | Last returned ID if more visible jobs existed at evaluation; otherwise `null`. |

The runtime MUST merge its visible queue and history into logical jobs without
duplicates. It MUST NOT omit eligible jobs within a page. An empty page has
`next: null`. Repeat `downloaderId` and `limit` with `after: next` to continue.
Pages are live observations: jobs added before `after` can be missed, and jobs
can change or disappear between pages. Start from the first page to refresh.
Listing is not ownership: policy may expose jobs submitted elsewhere while
withholding control or file access. Backend unavailability is an error, not an
empty list or a claim that jobs disappeared.

`status` returns a fresh observation of one authorized job. A job missing from
both accessible queue and history produces `not_found` only after a successful
lookup; connection failures produce `unavailable`. Conflicting or unmappable
backend state is `unknown`, never assumed success.

`Job`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `jobId` | yes | text | Opaque logical job handle. |
| `downloaderId` | yes | text | Owning downloader. |
| `name` | yes | text | Current display label. |
| `state` | yes | `JobState` | Normalized observation below. |
| `observedAt` | yes | timestamp | When the runtime obtained this observation. |
| `actions` | yes | list of `JobAction` | Currently implementable actions for this job and phase; subset of downloader actions. Still subject to authorization and races. |
| `canReadFiles` | yes | boolean | True only for a completed job with an authorized output mapping and downloader file-read support; false is independent of job success. |
| `categoryId` | no | text | Present only when mapped to a discoverable category. |
| `downloadPaused` | no | boolean | Known per-job download hold. Omitted when not observable. Required when pause/resume is advertised for this job. |
| `downloadProgress` | no | number | Backend's transfer estimate from 0 to 1. Excludes later processing; omitted if unavailable. |
| `downloadedBytes` | no | integer | Reported transferred bytes, including retries when the backend counts them. Not output file size. |
| `totalBytes` | no | integer | Reported expected transfer size; may change during repair. |
| `downloadRate` | no | number | Nonnegative bytes per second attributable to this job; omit if only a global rate is known. |
| `stage` | no | text | Sanitized display-only detail, such as repair or extraction. Not a second state machine. |
| `message` | no | text | Sanitized display-only explanation; never a raw log or secret. |

`JobState`:

| Value | Meaning |
|-------|---------|
| `"queued"` | Waiting for scheduling, backend-wide resume, resources, or another prerequisite. |
| `"downloading"` | Retrieving article data, possibly alongside processing. |
| `"paused"` | Per-job download hold is active and no processing is currently running. |
| `"processing"` | Repair, extraction, moving, configured processing, or waiting for such work; download hold does not imply this work is paused. |
| `"completed"` | Backend reports successful completion of its configured workflow. May reflect an external manual success override; not independent file verification. |
| `"failed"` | Backend reports terminal failure. |
| `"cancelled"` | Execution was stopped by cancellation. No claim about later data retention. |
| `"skipped"` | Backend intentionally omitted the download, for example due to duplicate policy. |
| `"unknown"` | A backend observation exists but cannot be mapped reliably. |

For overlapping stages, use `downloading` while article transfer is active,
otherwise `processing` while processing is active or pending, then `paused`
when held, else `queued`. An explicit terminal outcome takes precedence.
Queue/history location, 100% transfer progress, or a final path alone MUST NOT
be mapped to `completed`. A completed job may still have no readable output.

### Job control

`JobAction` is one of `"pause"`, `"resume"`, `"cancel"`, `"retry"`, `"forget"`,
or `"delete"`.

`ControlRequest`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `jobId` | yes | text | One job; no wildcard, batch, or global target. |
| `operationKey` | yes | `OperationKey` | One logical control mutation. |
| `action` | yes | `JobAction` | Must be supported for the actual job state when executed. |

| Action | Required effect for success |
|--------|-----------------------------|
| `pause` | Set this job's download hold. Does not promise to pause repair/extraction or other processing. No global pause. |
| `resume` | Clear this job's download hold. Does not override global pause, scheduler, quota, or user policy. |
| `cancel` | Stop this job's execution, including processing, without deleting its retained files. Keep a cancellation record visible while retained. If the backend cannot preserve files or stop the current phase, do not advertise this action. |
| `retry` | Accept a new attempt of a failed or cancelled job with the same NZB and effective configuration. The backend may reuse retained data or download it again. Preserve existing retained files; do not silently force past duplicate policy. Unsupported if the source/configuration cannot be recovered. |
| `forget` | Remove a terminal job from this NAP's visible queue/history without deleting data. Backend archival is allowed. Later lookup of that handle is `not_found`. |
| `delete` | For a terminal job, delete its exclusively owned retained download/output data and forget its record. Requires a separate deletion grant. Unsupported if safe ownership or deletion completion cannot be established. |

Terminal jobs are `completed`, `failed`, `cancelled`, or `skipped`. `unknown`
is never presumed terminal. A hold already set/cleared satisfies pause/resume
without new backend work. The runtime MUST recheck state and authorization
before dispatch. A valid action unsupported by the adapter is `unsupported`;
a supported action no longer applicable to the job is `invalid_state`.

`ControlResult`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `jobId` | yes | text | Original target handle. |
| `action` | yes | `JobAction` | Requested action. |
| `job` | for pause, resume, cancel, retry | `Job` | Observed resulting job; retry may return a new `jobId` if the backend creates a successor. Omitted for forget/delete. |
| `replayUntil` | yes | timestamp | Earliest expiry of this mutation's replay record; the runtime may extend it. |

Success MUST reflect the effect above, established by reliable acknowledgement
or follow-up observation, not merely command dispatch or an unconditional
backend boolean. Retry success is acceptance of a new attempt, not download
completion. Original and successor IDs MUST NOT be confused or silently rebound.
The runtime MUST NOT emulate a job action using a global downloader command.
If effects may have occurred but cannot be confirmed within the request deadline,
return `outcome_unknown`. If some effects are known to have occurred but the
required action did not finish, return `mutation_failed`. Neither case permits
automatic redispatch; retain the replay record and refresh job state.

### Optional completed-file access

`files` and `read` require an actual authorized mapping to completed output,
independent of backend management access. Unsupported adapters return
`unsupported`. A backend path or a list of NZB segments is not such a mapping.
No host path, credential-bearing URL, or arbitrary path parameter is exposed.

`FilesRequest`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `jobId` | yes | text | Completed job to inspect. |
| `after` | no | text | Exclusive last `fileId` from the previous page. |
| `limit` | no | integer | Positive; defaults to `min(100, maxPageSize)`. |

`FilesResult`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `files` | yes | list of `OutputFile` | Actual completed output files, sorted by `fileId` in UTF-8 byte order. Same first-page/continuation rules as `jobs`. |
| `next` | yes | text or null | Last returned file ID if more existed at evaluation; otherwise `null`. |

`OutputFile`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `fileId` | yes | text | Identifies one version of one job output file. |
| `name` | yes | text | Display name, not a lookup path. |
| `byteLength` | yes | integer | Exact completed file size. |
| `mediaType` | no | text | Best-effort media type; not an instruction to execute content. |

A readable empty output set returns an empty list. Missing mapping produces
`unsupported`; output still being produced produces `not_ready`; previously
mapped files no longer present produce `not_found`. File handles MUST NOT be
reused for replacement content. An adapter advertising reads MUST ensure that
all successful reads of a `fileId` address the same file version; if it cannot,
it MUST invalidate the handle and return `file_changed`.

`ReadRequest`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `jobId` | yes | text | Job owning the file. |
| `fileId` | yes | text | Handle returned by `files` for this job. |
| `offset` | yes | integer | Zero-based byte offset. |
| `length` | yes | integer | Positive maximum requested bytes, at most `maxReadBytes`. |

`ReadResult`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `dataBase64` | yes | text | Exact requested file bytes. |
| `byteLength` | yes | integer | Exact decoded count. |
| `eof` | yes | boolean | True exactly when `offset + byteLength` equals the file size. |

A successful read returns exactly `min(length, file size - offset)` bytes.
`offset` equal to file size returns empty data with `eof: true`; a larger offset
is `invalid_range`. Failed reads return no partial data. Reads MUST NOT resume
a job, trigger article retrieval, run processing, or execute a downloaded file.
Deletion, forgetting, or loss of authorization invalidates file access before
subsequent data delivery.

### Mutation replay

`add` and `control` MUST use a durable replay record, scoped to the current user,
verified napplet identity, and downloader. The runtime reserves `operationKey`
with the operation and exact validated parameters before any external effect.
Record comparison ignores object member order and wire `id`, but does not
normalize NZB XML, URLs, names, or option omissions. The reservation MUST survive
runtime restart. It MUST NOT expire or permit redispatch while the initial
fetch/dispatch remains active, and MUST remain for at least `replaySeconds`
after that work terminates. `replayUntil` is a guaranteed minimum retention
boundary and MUST NOT move earlier. Replays alone need not extend retention.

During that interval, the same key and parameters MUST return the recorded
outcome without redispatch. Different parameters or a different operation with
the same key produce `conflict`. An unresolved attempt returns `outcome_unknown`
with `replayUntil`; it may later resolve only from reliable evidence. If the
runtime crashes between dispatch and acknowledgement, it MUST preserve the
uncertainty rather than dispatch again. Backend content-duplicate detection is
not a substitute for this record.

Cached outcomes MUST be authorized again before delivery. An add/control success
replayed after a job changed or disappeared remains its original observation,
not a new job or current status; use `status` to refresh. Records and handles
needed to interpret a replay MUST remain available until expiry even after
forget/delete. A missing public job remains `not_found` despite that internal
record.

After expiry, the runtime MAY treat the key as new. Napplets MUST NOT automatically
retry beyond the known replay window, or use a fresh key after an uncertain
outcome, without reconciliation or renewed user intent. This is bounded request
deduplication, not exactly-once execution across an external downloader.

## Wire Protocol

The projection carries objects with `type` and nonempty text `id`. Browser
mechanics belong to the [web projection](../projections/web.md).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `nzb.downloaders` | napplet → shell | `id` |
| `nzb.downloaders.result` | shell → napplet | `id`, `result` (`DownloadersResult`) or `error` |
| `nzb.add` | napplet → shell | `id`, `request` (`AddRequest`) |
| `nzb.add.result` | shell → napplet | `id`, `result` (`AddResult`) or `error` |
| `nzb.jobs` | napplet → shell | `id`, `request` (`JobsRequest`) |
| `nzb.jobs.result` | shell → napplet | `id`, `result` (`JobsResult`) or `error` |
| `nzb.status` | napplet → shell | `id`, `jobId` |
| `nzb.status.result` | shell → napplet | `id`, `result` (`Job`) or `error` |
| `nzb.control` | napplet → shell | `id`, `request` (`ControlRequest`) |
| `nzb.control.result` | shell → napplet | `id`, `result` (`ControlResult`) or `error` |
| `nzb.files` | napplet → shell | `id`, `request` (`FilesRequest`) |
| `nzb.files.result` | shell → napplet | `id`, `result` (`FilesResult`) or `error` |
| `nzb.read` | napplet → shell | `id`, `request` (`ReadRequest`) |
| `nzb.read.result` | shell → napplet | `id`, `result` (`ReadResult`) or `error` |

Every routable recognized request with a valid `id` MUST receive one matching
result while its endpoint remains live. Results contain exactly one of `result`
or `error` (`NZBError`). IDs are unique among outstanding requests per endpoint;
equal IDs from different endpoints are independent and MUST NOT cross-route.
A duplicate outstanding ID MUST NOT dispatch another operation or receive a
second result. Unauthenticated/unroutable messages and unknown types are ignored.
`operationKey`, not `id`, controls mutation replay across completed requests.
There are no unsolicited messages or mandatory subscriptions; napplets poll
`jobs` or `status`, subject to runtime rate limits.

### Examples

Two downloaders: a native engine with file access, and a remote control adapter.
The limits and supported options are illustrative, not product guarantees.

```json
{"type":"nzb.downloaders","id":"d1"}
{"type":"nzb.downloaders.result","id":"d1","result":{"downloaders":[{"downloaderId":"native","label":"Local downloads","canAdd":true,"sourceKinds":["bytes","url"],"addOptions":["startPaused"],"categories":[],"actions":["pause","resume","cancel","retry","forget","delete"],"canReadFiles":true,"maxNZBBytes":1048576,"maxPageSize":100,"maxReadBytes":65536,"replaySeconds":86400},{"downloaderId":"remote","label":"Home downloader","canAdd":true,"sourceKinds":["bytes"],"addOptions":[],"categories":[],"actions":["pause","resume","forget"],"canReadFiles":false,"maxNZBBytes":1048576,"maxPageSize":100,"maxReadBytes":0,"replaySeconds":86400}]}}
```

Submit one NZB as bytes. The XML describes one illustrative article segment;
acceptance does not establish that the article exists.

```json
{"type":"nzb.add","id":"a1","request":{"downloaderId":"remote","operationKey":"submit-1","source":{"kind":"bytes","dataBase64":"PG56YiB4bWxucz0iaHR0cDovL3d3dy5uZXd6YmluLmNvbS9EVEQvMjAwMy9uemIiPjxmaWxlIHBvc3Rlcj0iZXhhbXBsZUBleGFtcGxlLm5ldCIgZGF0ZT0iMTc5MzM2MTYwMCIgc3ViamVjdD0iZXhhbXBsZS50eHQiPjxncm91cHM+PGdyb3VwPmFsdC50ZXN0PC9ncm91cD48L2dyb3Vwcz48c2VnbWVudHM+PHNlZ21lbnQgYnl0ZXM9IjgiIG51bWJlcj0iMSI+ZXhhbXBsZUBleGFtcGxlLm5ldDwvc2VnbWVudD48L3NlZ21lbnRzPjwvZmlsZT48L256Yj4="},"name":"Example"}}
{"type":"nzb.add.result","id":"a1","result":{"job":{"jobId":"job-a","downloaderId":"remote","name":"Example","state":"queued","observedAt":1793361600,"actions":["pause"],"canReadFiles":false,"downloadPaused":false},"replayUntil":1793448000}}
```

The same key with a new wire ID replays that result. Changing `name` under the
same key produces `conflict`. If acknowledgement were lost before the job could
be identified, the sole result would instead be:

```json
{"type":"nzb.add.result","id":"a1","error":{"code":"outcome_unknown","message":"Submission may have been accepted; reconcile using the same operation key","replayUntil":1793448000}}
```

Pause the accepted remote job, without changing the global downloader state:

```json
{"type":"nzb.control","id":"c0","request":{"jobId":"job-a","operationKey":"pause-1","action":"pause"}}
{"type":"nzb.control.result","id":"c0","result":{"jobId":"job-a","action":"pause","job":{"jobId":"job-a","downloaderId":"remote","name":"Example","state":"paused","observedAt":1793361610,"actions":["resume"],"canReadFiles":false,"downloadPaused":true},"replayUntil":1793448010}}
```

The following observations assume the user subsequently resumed the job.

List and inspect a remote job during extraction, then observe completion without
file access. Backend history placement does not determine these states.

```json
{"type":"nzb.jobs","id":"j1","request":{"downloaderId":"remote","limit":10}}
{"type":"nzb.jobs.result","id":"j1","result":{"jobs":[{"jobId":"job-a","downloaderId":"remote","name":"Example","state":"processing","observedAt":1793361700,"actions":[],"canReadFiles":false,"downloadProgress":1,"stage":"Extracting"}],"next":null}}
{"type":"nzb.status","id":"s1","jobId":"job-a"}
{"type":"nzb.status.result","id":"s1","result":{"jobId":"job-a","downloaderId":"remote","name":"Example","state":"completed","observedAt":1793361800,"actions":["forget"],"canReadFiles":false}}
{"type":"nzb.files","id":"f1","request":{"jobId":"job-a"}}
{"type":"nzb.files.result","id":"f1","error":{"code":"unsupported","message":"Completed output is not exposed by this adapter"}}
```

A denied deletion has no effect despite the native adapter supporting deletion:

```json
{"type":"nzb.control","id":"c1","request":{"jobId":"job-local","operationKey":"delete-1","action":"delete"}}
{"type":"nzb.control.result","id":"c1","error":{"code":"permission_denied","message":"Deleting download data is not authorized"}}
```

Read an authorized native output file (`SGVsbG8uDQo=` is eight bytes):

```json
{"type":"nzb.files","id":"f2","request":{"jobId":"job-local"}}
{"type":"nzb.files.result","id":"f2","result":{"files":[{"fileId":"file-a","name":"example.txt","byteLength":8,"mediaType":"text/plain"}],"next":null}}
{"type":"nzb.read","id":"r1","request":{"jobId":"job-local","fileId":"file-a","offset":0,"length":8}}
{"type":"nzb.read.result","id":"r1","result":{"dataBase64":"SGVsbG8uDQo=","byteLength":8,"eof":true}}
```

Additional consistency cases:

| Situation | Required observation |
|-----------|----------------------|
| `read.length` is zero | `invalid_request`; no file bytes. |
| `read.length` exceeds the downloader's limit | `limit_exceeded`; no truncated success. |
| Two endpoints both send `id: "s1"` | Each result goes only to its requesting endpoint; neither learns the other's handles. |
| Endpoint closes after a backend accepted `add` | No late delivery to the closed endpoint; the job persists and the replay record remains. |
| Access is revoked during a file read | Suppress late bytes; return `permission_denied` if the endpoint remains live. |
| A remote adapter cannot preserve data when stopping a phase | Do not advertise `cancel`; an attempted unsupported cancellation has no backend effect. |

### Error Handling

`NZBError`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `code` | yes | text | One of the codes below. |
| `message` | yes | text | Sanitized explanation; not machine-parsed. |
| `replayUntil` | when a mutation record exists | timestamp | Earliest expiry; omitted for requests rejected before reservation. |

| Code | Meaning |
|------|---------|
| `invalid_request` | Wrong shape, enum, encoding, NZB, or nonpositive size. No effects dispatched. |
| `permission_denied` | User or shell denied the operation, including revoked access. |
| `not_found` | Handle, category, job, or file absent or concealed from the caller. |
| `unsupported` | Adapter cannot enforce the requested feature or lacks file mapping. |
| `invalid_state` | Action no longer applicable to the actual job phase. |
| `invalid_range` | Read offset exceeds the output size. |
| `not_ready` | Job output is still being produced. |
| `file_changed` | File handle no longer identifies the same output version. |
| `limit_exceeded` | Structurally valid input exceeds a current size/page/resource limit. |
| `rate_limited` | Runtime rate or outstanding-request bound reached. |
| `authentication_required` | Backend credentials need attention in the shell. |
| `unavailable` | Backend/fetch failure or deadline expiry with no possible mutation effect. |
| `backend_rejected` | Backend conclusively declined the requested mutation without applying it. |
| `mutation_failed` | Mutation stopped after known partial effects, such as deleting only some files. Reconcile before any new attempt. |
| `backend_error` | Malformed/inconsistent response without a possible mutation effect. |
| `conflict` | Retained operation key reused with different operation or parameters. |
| `outcome_unknown` | Mutation may have taken effect but cannot yet be identified or confirmed. |

The runtime validates and authorizes before effects. It MAY return `not_found`
to conceal inaccessible objects. After a possible mutation, a missing or
malformed acknowledgement, timeout, or revocation race MUST NOT be reported as
a conclusive non-effect: use `outcome_unknown` unless the outcome is known.
Known incomplete mutations use `mutation_failed`, never a no-effect error.
No error response carries partial file bytes. Revocation can suppress sensitive
cached results without implying that an accepted backend job was undone.

## Shell Behavior

- The shell MUST bind every request to its verified napplet endpoint and current
  user. Read, control, file access, and data deletion are distinct permissions;
  discovering a foreign job MUST NOT grant control of it.
- Jobs belong to the runtime/backend, not an endpoint lifetime. Authorized jobs
  MAY continue after the napplet closes. The shell MUST disclose this retention
  when accepting a submission. Closing an endpoint MUST NOT implicitly cancel,
  forget, delete, or resubmit a job. The runtime stops delivery to that endpoint.
- User/profile changes MUST invalidate the old endpoint's access and cached
  deliveries. Retained jobs continue under their existing owner; they MUST NOT
  become the new user's jobs. Reopening uses `jobs` under fresh authorization.
- The runtime MUST serialize conflicting mutations for a logical job and recheck
  actual backend state before acting. Other backend clients can still change
  jobs; results are observations, not locks on external clients.
- The runtime MUST impose finite request deadlines, bounded input/output sizes,
  bounded outstanding work, and polling limits. Long downloads are jobs, not
  requests held open until completion. It MUST preserve uncertain mutations
  beyond request/endpoint termination through the replay record.
- Authorization MUST be rechecked before file/data delivery and before effects.
  Revocation stops new mediated operations and late data delivery; it does not
  promise to retract work already accepted by an external service.

## Security Considerations

Downloader configuration, NNTP credentials, API keys, and backend administration
remain with the runtime. An NZB URL selects input data, never a downloader or
credential scope. The shell MUST restrict network targets under user policy,
including DNS resolution and redirects, and MUST NOT follow metadata-supplied
network targets outside that policy. Backend connections MUST use authenticated,
protected transport or an explicitly user-trusted local channel; a napplet
cannot request a cleartext downgrade or choose backend endpoints.

NZB subjects, metadata, filenames, and archive contents are untrusted. A native
engine MUST confine untrusted file writes and extraction to authorized storage,
preventing path traversal and link escapes. For an external downloader, that
service is the execution and storage trust boundary. The shell MUST authorize
this delegation and disclose relevant configured processing, including scripts
and output destinations. An adapter MUST NOT claim to sandbox a remote service
merely because it mediates its API.

Napplet inputs MUST NOT enable scripts or destinations outside the authorized
effective configuration, select arbitrary executables, or alter global settings.
Choosing an advertised category may select user-approved processing; it is not
a general script-execution capability. Explicit request options take precedence
over NZB metadata. Metadata MUST NOT override the selected/default category or
its authority; display and extraction-password hints MAY be used under shell
policy. Display names MUST NOT be passed as backend path parameters.

File access requires the runtime to resolve and verify an owned output mapping.
It MUST NOT trust a path reported by a backend as permission to read arbitrary
local files. Deletion MUST preserve files shared with other jobs and reject work
whose ownership cannot be established. Unsupported adapters MUST fail explicitly
rather than weaken file-retention, isolation, or deletion semantics.

Passwords, URL tokens, private paths, and raw backend logs MUST NOT appear in
results, diagnostics sent to napplets, or another caller's discovery. Replay
records containing sensitive inputs MUST receive the same protection as other
runtime secrets. Download completion is not a signature, malware assessment,
license claim, or integrity guarantee beyond the configured downloader's work.

## Adapter Guidance (informational)

This table explains the common boundary; it does not freeze backend API versions.
Adapters must verify the behavior of their configured backend and credentials.

| Concern | Native engine | NZBGet adapter | SABnzbd adapter |
|---------|---------------|----------------|-----------------|
| Submission | Create an owned job from validated NZB bytes. | Adapt `append` and retain its job identity. | Adapt `addfile` and retain the returned job identity. |
| Observation | Expose actual engine stages. | Combine queue and history observations. | Include processing work reported in history. |
| Control | Enforce one job's state and storage boundaries. | Use per-job queue/history operations. | Use per-job operations; global pause is a different action. |
| Files | Expose owned completed output if authorized. | Requires a separate trusted output mapping. | Requires a separate trusted output mapping. |

Backend references: [NZBGet API](https://nzbget.com/documentation/api/),
[NZBGet submission](https://nzbget.com/documentation/api/append/),
[NZBGet queue control](https://nzbget.com/documentation/api/editqueue/),
[NZBGet history](https://nzbget.com/documentation/api/history/), and
[SABnzbd API](https://sabnzbd.org/wiki/configuration/5.1/api).

Backend cancellation/deletion defaults differ; do not equate an operation named
“delete” with NAP-NZB `cancel`. Likewise, an API success boolean may require
follow-up observation. A backend lacking introspection does not prevent this
NAP: the adapter reports only behavior it has established. This specification
does not require backends to implement NAP messages or native idempotency keys.

## Implementations

No NAP-NZB implementation is verified by this proposal. NZBGet and SABnzbd are
reference integration targets, not claimed NAP implementations.

## Changelog

- `53d2b0e` - Defined native and external NZB downloader jobs, advertised controls and optional file reads, mutation replay, and runtime authority boundaries.
