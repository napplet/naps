# NAP-TORRENT

## Runtime-Mediated Torrents

`draft`

**NAP ID:** NAP-TORRENT
**Domain:** `torrent`
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.torrent`; domain presence signals availability.

## Description

NAP-TORRENT gives napplets a runtime-mediated interface for torrent
transfers. A napplet can request downloads or seeding, select files, and
observe transfer jobs using magnet links, info hashes, metainfo bytes, or
runtime-owned file sets. It also provides verified file reads, torrent metadata
export, live settings, and recovery of retained jobs.

The runtime owns the torrent engine, transport choice, storage, file picker,
bandwidth limits, tracker/DHT/WebTorrent policy, and user consent. Napplets
never get raw TCP, UDP, WebRTC, HTTP tracker, DHT, or filesystem access through
this interface.

This NAP supports both WebTorrent-style runtimes and standard BitTorrent
runtimes. A shell may use HTTP/WebRTC/WebSocket transports, TCP/UDP/DHT
transports, local torrent daemons, or another backend. The napplet requests the
transfer and observes a job. It does not own the transport.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `capabilities` | none | `TorrentCapabilities` | `torrent.capabilities` / `.result` |
| `pickFiles` | optional `options` (`TorrentPickOptions`) | `TorrentFileSetResult` | `torrent.pickFiles` / `.result` |
| `add` | `request` (`TorrentAddRequest`) | `TorrentJobAck` | `torrent.add` / `.result`, then push messages |
| `status` | `jobId` (`text`) | `TorrentJobStatus` | `torrent.status` / `.result` |
| `jobs` | optional `filter` (`TorrentJobFilter`) | list of `TorrentJobSummary` | `torrent.jobs` / `.result` |
| `files` | `jobId` (`text`) | list of `TorrentFileStatus` | `torrent.files` / `.result` |
| `selectFiles` | `jobId` (`text`), `selection` (`TorrentFileSelection`) | `TorrentJobStatus` | `torrent.selectFiles` / `.result` |
| `pause` | `jobId` (`text`) | `TorrentJobStatus` | `torrent.pause` / `.result` |
| `resume` | `jobId` (`text`) | `TorrentJobStatus` | `torrent.resume` / `.result` |
| `remove` | `jobId` (`text`), optional `options` (`TorrentRemoveOptions`) | `TorrentRemoveResult` | `torrent.remove` / `.result` |

| `metadata` | `jobId` (`text`) | `TorrentMetadataResult` | `torrent.metadata` / `.result` |
| `read` | `request` (`TorrentReadRequest`) | `TorrentReadResult` | `torrent.read` / `.result` |
| `exportMetainfo` | `jobId` (`text`) | `TorrentMetainfoResult` | `torrent.exportMetainfo` / `.result` |
| `saveFile` | `jobId` (`text`), `fileIndex` (`integer`) | `TorrentFileActionResult` | `torrent.saveFile` / `.result` |
| `openFile` | `jobId` (`text`), `fileIndex` (`integer`) | `TorrentFileActionResult` | `torrent.openFile` / `.result` |
| `cancel` | `requestId` (`text`) | `TorrentCancelResult` | `torrent.cancel` / `.result` |
| `update` | `request` (`TorrentUpdateRequest`) | `TorrentSettingsResult` | `torrent.update` / `.result` |

### Schemas

Primitive types:

| Name | Definition |
|------|------------|
| `HexV1Hash` | text: 40 lowercase hexadecimal characters, the full v1 info hash |
| `HexV2Hash` | text: 64 lowercase hexadecimal characters, the full v2 info hash |
| `Base64Bytes` | text: standard padded base64 encoding of bytes; size limits count decoded bytes |
| `MetainfoBytes` | binary contents of a `.torrent` file; the host representation is projection-defined |

Enums:

| Name | Values |
|------|--------|
| `TorrentTransport` | `webtorrent`, `bittorrent`, `auto`, or runtime-defined text |
| `TorrentMode` | `download`, `seed`, `download-and-seed` |
| `TorrentJobState` | `queued`, `metadata`, `downloading`, `seeding`, `paused`, `complete`, `cancelled`, `error` |
| `TorrentSourceType` | `magnet`, `infohash`, `metainfo`, `runtime-file-set` |
| `TorrentStorage` | "temporary", "persistent", or "user-selected" |
| `TorrentFormat` | "v1", "v2", or "hybrid" |
| `TorrentRetention` | "session" or "persistent" |

#### TorrentCapabilities

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `transports` | yes | list of TorrentTransport | — |
| `modes` | yes | list of TorrentMode | — |
| `formats` | yes | list of TorrentFormat | Supported transfer and metainfo formats. |
| `createFormats` | yes | list of TorrentFormat | Formats the runtime can create; empty if unsupported. |
| `retentions` | yes | list of TorrentRetention | Supported job lifetimes. |
| `mutableSettings` | yes | list of text | Supported field names from TorrentSettingsPatch. |
| `maxReadBytes` | yes | integer | Positive; at most 1048576 decoded bytes per read. |
| `maxReadWaitMs` | yes | integer | Non-negative; at most 30000. |
| `maxPendingReads` | yes | integer | Positive per-endpoint bound. |
| `maxMetainfoBytes` | yes | integer | Positive bound on decoded metainfo input and export. |
| `canSaveFiles` | yes | boolean | Supports user-mediated export of completed files. |
| `canOpenFiles` | yes | boolean | Supports user-mediated opening or handoff. |
| `maxActiveJobs` | no | integer | Non-negative. |
| `maxTorrentSize` | no | integer | Non-negative. |
| `canSeedRuntimeFiles` | no | boolean | — |
| `canUseDht` | no | boolean | — |
| `canUseTrackers` | no | boolean | — |
| `canUseWebSeeds` | no | boolean | — |

#### TorrentFile

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `path` | yes | text | — |
| `size` | no | integer | Non-negative. |

#### TorrentPickOptions

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `multiple` | no | boolean | — |
| `suggestedName` | no | text | — |

#### TorrentRuntimeFileSet

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `runtimeFileSetId` | yes | text | — |
| `files` | yes | list of TorrentFile | — |
| `totalBytes` | yes | integer | Non-negative. |
| `displayName` | no | text | — |

#### TorrentFileSetResult

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | — |
| `fileSet` | no | TorrentRuntimeFileSet | — |
| `error` | no | text | — |
| `reason` | no | text | — |

#### TorrentSource

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `sourceType` | yes | TorrentSourceType | Selects the source field. |
| `magnet` | no | text | Required when selected by `sourceType`. |
| `hashes` | no | TorrentHashes | Required for the "infohash" source. |
| `metainfo` | no | MetainfoBytes | Required when selected by `sourceType`. |
| `runtimeFileSetId` | no | text | Required when selected by `sourceType`. |

#### TorrentAddOptions

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `mode` | no | TorrentMode | — |
| `transport` | no | TorrentTransport | — |
| `startPaused` | no | boolean | — |
| `seedAfterComplete` | no | boolean | — |
| `downloadLimitBytesPerSecond` | no | integer | Non-negative. |
| `uploadLimitBytesPerSecond` | no | integer | Non-negative. |
| `storage` | no | TorrentStorage | Storage location; independent of job retention. |
| `trackers` | no | list of text | — |
| `webSeeds` | no | list of text | — |
| `fileSelection` | no | TorrentFileSelection | — |
| `retention` | no | TorrentRetention | Default "session". |
| `createFormat` | no | TorrentFormat | Runtime-file-set sources only; default "v1". |

#### TorrentAddRequest

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `source` | yes | TorrentSource | — |
| `jobId` | no | text | — |
| `options` | no | TorrentAddOptions | — |

#### TorrentJobAck

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | — |
| `jobId` | no | text | — |
| `state` | no | TorrentJobState | — |
| `error` | no | text | — |
| `reason` | no | text | — |

#### TorrentPeerStats

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `peers` | yes | integer | Non-negative. |
| `seeds` | yes | integer | Non-negative. |
| `leeches` | yes | integer | Non-negative. |

#### TorrentJobStatus

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | — |
| `jobId` | yes | text | — |
| `state` | yes | TorrentJobState | — |
| `mode` | yes | TorrentMode | — |
| `transport` | yes | TorrentTransport | — |
| `hashes` | no | TorrentHashes | Present once an identity is known; omitted before creation completes. |
| `downloadedBytes` | yes | integer | Non-negative. |
| `uploadedBytes` | yes | integer | Non-negative. |
| `totalBytes` | no | integer | Non-negative; omitted until size is known. |
| `progress` | no | number | 0..1; omitted until meaningful. |
| `downloadRate` | yes | number | — |
| `uploadRate` | yes | number | — |
| `ratio` | yes | number | — |
| `peers` | yes | TorrentPeerStats | — |
| `title` | no | text | — |
| `etaSeconds` | no | integer | Non-negative. |
| `storage` | no | TorrentStorage | Effective storage choice. |
| `settings` | yes | TorrentSettings | Current effective settings and revision. |
| `retention` | yes | TorrentRetention | Effective job lifetime. |
| `error` | no | text | — |
| `reason` | no | text | — |

#### TorrentJobSummary

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `jobId` | yes | text | — |
| `state` | yes | TorrentJobState | — |
| `mode` | yes | TorrentMode | — |
| `transport` | yes | TorrentTransport | — |
| `hashes` | no | TorrentHashes | Present once an identity is known; omitted before creation completes. |
| `progress` | no | number | 0..1; omitted until meaningful. |
| `downloadRate` | yes | number | — |
| `uploadRate` | yes | number | — |
| `ratio` | yes | number | — |
| `title` | no | text | — |
| `retention` | yes | TorrentRetention | Effective lifetime for owned jobs; delegated aliases are session-scoped. |

#### TorrentJobFilter

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `states` | no | list of TorrentJobState | — |
| `modes` | no | list of TorrentMode | — |

#### TorrentFileStatus

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `index` | yes | integer | Non-negative. |
| `path` | yes | text | — |
| `size` | yes | integer | Non-negative. |
| `selected` | yes | boolean | — |
| `priority` | yes | integer | — |
| `downloadedBytes` | yes | integer | Non-negative. |
| `progress` | yes | number | — |

#### TorrentFileSelection

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `indexes` | no | list of integer | Each index is non-negative. |
| `paths` | no | list of text | — |
| `priority` | no | integer | — |

#### TorrentRemoveOptions

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `deleteData` | no | boolean | — |

#### TorrentRemoveResult

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | — |
| `jobId` | yes | text | — |
| `deletedData` | no | boolean | — |
| `error` | no | text | — |
| `reason` | no | text | — |

#### TorrentHashes

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `v1` | no | HexV1Hash | At least one hash is required. |
| `v2` | no | HexV2Hash | Hybrid metadata has both hashes. |

#### TorrentMetadataResult

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | Result discriminator. |
| `format` | conditional | TorrentFormat | Required on success. |
| `hashes` | conditional | TorrentHashes | Required on success; verified from metainfo. |
| `name` | no | text | Untrusted display name. |
| `private` | conditional | boolean | Required on success; private-torrent flag. |
| `magnet` | conditional | text | Required on success; shareable identity and permitted hints. |
| `trackers` | no | list of text | Effective tracker URLs, when disclosure is permitted. |
| `webSeeds` | no | list of text | Effective web-seed URLs, when disclosure is permitted. |
| `endpointsRedacted` | conditional | boolean | Required on success; true if either URL list is withheld. |
| `metainfoReady` | conditional | boolean | Required on success; complete export is available subject to policy. |
| `error` | conditional | text | Required on failure. |
| `reason` | no | text | Diagnostic only. |

#### TorrentReadRequest

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `jobId` | yes | text | Authorized job. |
| `fileIndex` | yes | integer | Non-negative index returned by files. |
| `offset` | yes | integer | Non-negative byte offset within the file. |
| `length` | yes | integer | Positive; at most maxReadBytes. |
| `waitMs` | no | integer | 0..maxReadWaitMs; default 0. |

#### TorrentReadResult

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | Result discriminator. |
| `offset` | conditional | integer | Requested offset; required on success. |
| `dataBase64` | conditional | Base64Bytes | Required on success; verified bytes only. |
| `byteLength` | conditional | integer | Decoded byte count; required on success. |
| `totalBytes` | conditional | integer | File length; required on success. |
| `eof` | conditional | boolean | Required on success; true exactly when offset + byteLength equals totalBytes. |
| `error` | conditional | text | Required on failure. |
| `reason` | no | text | Diagnostic only. |

#### TorrentMetainfoResult

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | Result discriminator. |
| `dataBase64` | conditional | Base64Bytes | Complete .torrent file; required on success. |
| `byteLength` | conditional | integer | Decoded byte count; required on success. |
| `hashes` | conditional | TorrentHashes | Hashes of the exported info dictionary; required on success. |
| `error` | conditional | text | Required on failure. |
| `reason` | no | text | Diagnostic only. |

#### TorrentFileActionResult

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | Result discriminator. |
| `jobId` | conditional | text | Required on success. |
| `fileIndex` | conditional | integer | Required on success. |
| `outcome` | conditional | "saved" or "accepted" | Required on success; saved for saveFile, accepted for openFile. |
| `error` | conditional | text | Required on failure. |
| `reason` | no | text | Diagnostic only. |

#### TorrentCancelResult

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | Result discriminator. |
| `cancelled` | conditional | boolean | Required on success; whether a pending request was cancelled. |
| `error` | conditional | text | Required on failure. |
| `reason` | no | text | Diagnostic only. |

#### TorrentSettings

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `revision` | yes | integer | Non-negative monotonic settings revision for this job. |
| `downloadLimitBytesPerSecond` | yes | integer | Non-negative; 0 means no requested limit, subject to shell limits. |
| `uploadLimitBytesPerSecond` | yes | integer | Non-negative; 0 means no requested limit, subject to shell limits. |
| `seedAfterComplete` | yes | boolean | Whether to continue uploading after download completion. |
| `trackers` | no | list of text | An explicit list replaces all configured tracker hints. |
| `webSeeds` | no | list of text | An explicit list replaces all configured web-seed hints. |
| `endpointsRedacted` | yes | boolean | True if either endpoint list is withheld by disclosure policy. |

#### TorrentSettingsPatch

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `downloadLimitBytesPerSecond` | no | integer | Non-negative; 0 means no requested limit, subject to shell limits. Omission leaves it unchanged. |
| `uploadLimitBytesPerSecond` | no | integer | Non-negative; 0 means no requested limit, subject to shell limits. Omission leaves it unchanged. |
| `seedAfterComplete` | no | boolean | Whether to continue uploading after download completion. Omission leaves it unchanged. |
| `trackers` | no | list of text | An explicit list replaces all configured tracker hints. Omission leaves it unchanged. |
| `webSeeds` | no | list of text | An explicit list replaces all configured web-seed hints. Omission leaves it unchanged. |

#### TorrentUpdateRequest

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `jobId` | yes | text | Authorized job. |
| `expectedRevision` | yes | integer | Revision obtained from status or a prior update. |
| `patch` | yes | TorrentSettingsPatch | Non-empty set of supported settings. |

#### TorrentSettingsResult

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | Result discriminator. |
| `settings` | conditional | TorrentSettings | Required on success and revision-conflict. |
| `error` | conditional | text | Required on failure. |
| `reason` | no | text | Diagnostic only. |

### Operation behavior

When `TorrentAddOptions.mode` is omitted, the default is `download`.
When `TorrentAddOptions.transport` is omitted, the default is `auto`.
When `retention` is omitted, the default is `session`. Storage location does not
change this default. Initial bandwidth limits default to 0; effective limits
remain subject to shell policy. `seedAfterComplete` defaults to true for
`download-and-seed` and `seed`, and false for `download`. A conflicting
explicit value is invalid. `startPaused` defaults to false.

**`capabilities()`** -- Reports the runtime's torrent features for this
napplet. `transports` names supported backend families, not a promise that a
specific tracker, peer, or network path will work.

**`pickFiles(options?)`** -- Opens a runtime-owned user file picker and returns
a `runtimeFileSetId` handle. The handle can be used only as a
`runtime-file-set` source for a seed/upload job. It is not a filesystem path.
A selection grant authorizes seeding only; it MUST NOT implicitly authorize
`read`, `saveFile`, or `openFile` on the resulting job. Those operations require
separate disclosure or handoff authorization, including for local seed data.

**`add(request)`** -- Creates a runtime transfer job from a magnet link,
info hash, metainfo bytes, or runtime-owned file set. The runtime MUST validate
the source before starting a transfer. The returned `jobId` is scoped to the
requesting napplet. Transfers run in the runtime.

`TorrentSource.sourceType` selects exactly one source field. The selected
field MUST be present; other source fields MUST be absent.

| `sourceType` | Required source field |
|--------------|-----------------------|
| `magnet` | `magnet` |
| `infohash` | `hashes` |
| `metainfo` | `metainfo` |
| `runtime-file-set` | `runtimeFileSetId` |

For upload/seeding, callers use `sourceType: "runtime-file-set"` with
`options.mode: "seed"` or `"download-and-seed"`. The runtime may create
metainfo and start seeding in `createFormat` (default "v1"). Unsupported
formats fail explicitly. The resulting hashes appear in status once known.
`metadata` and `exportMetainfo` become available when their required metadata
is complete. File-set handles remain endpoint-scoped and expire on endpoint
closure; an accepted persistent job retains its own authorized source lease.

**`status(jobId)`** -- Returns a one-shot job status snapshot.

**`jobs(filter?)`** -- Lists the requesting napplet's torrent jobs. A runtime
MAY also expose user-approved global torrent jobs through session-scoped
aliases. Such aliases are delegated access, not ownership; they expire with
the endpoint and MUST NOT bypass the underlying job's permissions. Owned
persistent jobs use the stable identifiers described under Job lifetime and
recovery.

**`files(jobId)`** -- Returns known file status. Before metadata is available,
the runtime MAY return an empty list.

**`selectFiles(jobId, selection)`** -- Changes file selection or priority for a
job. Selection applies to download jobs only. Seed jobs MAY reject with
`"unsupported-state"`.

**`pause(jobId)` / `resume(jobId)`** -- Pause or resume a transfer. Paused jobs
retain metadata and partial data according to runtime storage policy.

**`remove(jobId, options?)`** -- Removes a job. `deleteData: true` requests
runtime deletion of downloaded data, but the runtime MAY deny it if the data is
user-owned or shared by another job.

## Content access and streaming

**`read(request)`** returns a bounded range from a file in an authorized job.
The runtime MUST validate the file index and range with checked integer
arithmetic. It MUST enforce `maxReadBytes`, `maxReadWaitMs`, and
`maxPendingReads` before allocating buffers or scheduling work. These limits
are required capability fields; introspection is advisory, not a permission.

The range is `[offset, min(offset + length, file size))`. Offset equal to the
file size succeeds with empty bytes and `eof: true`; an offset beyond it is
`invalid-range`. A successful response contains the entire clipped range,
with `byteLength` equal to its decoded length. No partial success is returned.
Metadata must be known; otherwise return `metadata-unavailable`.

Only bytes verified against the torrent's hashes may cross the seam. With
`waitMs: 0`, an unavailable range returns `not-ready`. With a positive wait,
the runtime temporarily prioritizes its missing pieces within bandwidth and
quota policy. It returns `timeout` if the range remains unavailable when the
wait expires. This priority expires on completion, cancellation, or timeout;
it MUST NOT change persistent file selection or priority. An unselected file
returns `file-not-selected`. Paused jobs may serve verified local bytes but
MUST NOT resume networking to satisfy a read; missing bytes return `not-ready`.

A napplet can stream by requesting successive ranges, and seek by cancelling
an outstanding read and requesting another offset. This supplies verified
content; it does not supply a decoder or a direct network URL. The napplet
owns decoding and playback when it consumes the bytes. Returned bytes remain
untrusted file content and MUST NOT be executed by the runtime.

**`saveFile(jobId, fileIndex)`** asks the shell to export a complete, verified
file through a user-approved destination picker. Success means the export
has finished, with `outcome: "saved"`. It returns no filesystem path. An
incomplete file returns `not-ready`; the caller can wait for download progress
and retry. Cancellation or failure MUST leave no partial final destination.
Existing destinations MUST NOT be overwritten without explicit user approval.

**`openFile(jobId, fileIndex)`** asks the shell to open the file or hand it to
a user-selected handler. The shell owns handler choice, recipient permission,
and any transfer of access. Success with `outcome: "accepted"` acknowledges
opening responsibility; it does not prove playback or handler completion.
The shell MAY stream an incomplete file only if the selected handler can
consume verified ranges under these same rules; otherwise return `not-ready`.
No reusable URL, global file handle, or cross-napplet grant is returned.

Save/open support is advertised by `canSaveFiles` and `canOpenFiles`. An
unsupported action returns `unsupported`. A napplet may separately forward
bytes it is authorized to read using communication capabilities available to
it. This NAP does not define portable zero-copy handoff or extend another
domain's source schema.

**`cancel(requestId)`** applies to a pending `read`, `exportMetainfo`, or
`saveFile` from this authenticated endpoint. It has its own request `id`.
The runtime MUST serialize cancellation against the target's terminal outcome.
If cancellation wins, it returns `cancelled: true`, ends target work, and sends
one target result with `ok: false, error: "cancelled"`. If the target already
finished, it returns `cancelled: false`. It MUST NOT undo a committed save or
cancel another endpoint's request. Unknown target IDs return false without
revealing other endpoints. Unsupported target operations return `unsupported`.
Bindings MUST discard duplicate terminal messages and late bytes after a
successful cancellation. A cancelled save MUST remove its temporary output.

## Metadata and torrent formats

**`metadata(jobId)`** returns verified identity and shareable discovery hints
for the torrent. It returns `metadata-unavailable` until its info dictionary
is verified. File indexes and paths come from `files()` and MUST remain stable
for the job once metadata is known. Tracker/web-seed lists are effective
runtime settings, not authenticated content. Private URLs and credentials
MUST be withheld unless disclosure is separately authorized. Withheld lists
are omitted and `endpointsRedacted` is true; an empty list means known empty.
The returned magnet MUST omit withheld hints.

**`exportMetainfo(jobId)`** returns a complete `.torrent` file bounded by
`maxMetainfoBytes`. Export MUST preserve the exact hash-bearing info dictionary
and include all required metadata, including v2 piece layers when applicable.
Incomplete metadata returns `metadata-unavailable`; excess size returns
`too-large`. The runtime MUST NOT fabricate a complete torrent from an info
dictionary alone. Export is a snapshot of permitted effective tracker and
web-seed settings. Policy may redact non-identity fields, but MUST deny export
if this would invalidate the torrent or disclose unauthorized credentials.
A completed export MUST be reusable as a `metainfo` source with the same hashes.

`TorrentHashes` carries full identities. At least one of `v1` or `v2` is
required. A verified v1 torrent has only `v1`, v2 only `v2`, and hybrid both.
An input hash set may identify one side of a hybrid torrent; once metadata is
known, both identities are reported. The runtime MUST verify every supplied
identity against the received metainfo. Conflicting hashes fail with
`hash-mismatch`; truncated v2 hashes are invalid.

Formats follow [BEP-3](https://www.bittorrent.org/beps/bep_0003.html) and
[BEP-52](https://www.bittorrent.org/beps/bep_0052.html). Hybrid support requires
validation that both formats describe the same content. Advertised `formats`
means full validation of those formats, not silent fallback to another one.
A discovered unsupported format returns `unsupported-format`; discovery after
job acceptance ends the job with that error. Creation uses `createFormats`.

Magnets follow [BEP-9](https://www.bittorrent.org/beps/bep_0009.html): `btih`
for v1, `btmh` for v2, and both for a hybrid torrent. The runtime validates every
supplied identity; it MUST NOT interpret an arbitrary hash length as an
algorithm. A hash learned from an input is a claimed identity until metadata
verification succeeds. Status may report it without claiming metadata readiness.

## Live settings

**`update(request)`** atomically patches an existing job. Omitted fields stay
unchanged; supplied arrays replace their complete lists, and an empty array
clears that list. Clearing tracker hints does not disable DHT. Unknown or
unsupported fields return `unsupported-setting`; malformed values return
`invalid-request`. A stale `expectedRevision` returns `revision-conflict` and
current settings. Any failure leaves all settings unchanged.

The runtime MUST validate the complete patch under current policy before
applying it. It may clamp bandwidth requests to shell limits, and MUST return
the resulting effective settings. A denied tracker, web seed, or seeding
change rejects the entire patch rather than silently applying a subset.
URL disclosure follows the metadata rules, including `endpointsRedacted`.
Any effective change, including one from shell policy, increments `revision`.
A no-op retains the revision. Status exposes current settings after reconnect.

Changing `seedAfterComplete` on a download-origin job updates its reported
mode to `download` or `download-and-seed`. Turning it off during seeding stops
uploading and enters `complete`. Turning it on for a completed download starts
seeding under current policy. Paused jobs keep their paused state; resume
uses the new mode, continues any incomplete download, and seeds completed
content only when the setting is true. Otherwise completed content remains
`complete`. For an originally seed-only job changing this setting returns
`unsupported-state`; pause controls that job. All mutations for a job MUST be
serialized; `remove` wins against later queued updates. Updates never grant
new raw transport authority.

## Job lifetime and recovery

The runtime binds each job to its verified napplet principal and user profile;
neither is caller-supplied. An accepted `jobId` is stable for the job's retained
lifetime, unique within that ownership scope, and MUST NOT be recycled. It is
an identifier, not authorization. A requested `jobId` is only a preference;
collision returns `job-conflict`, and a retry MUST NOT mutate an existing job.
Reattached endpoints recheck current permissions before listing or operating
on jobs. Another profile or napplet cannot acquire a job by guessing its ID.
A shared backend engine MUST keep per-napplet grants and settings distinct;
removing one job MUST NOT revoke another owner's job or delete shared data
without the separate deletion authorization described by `remove`.
Artifact upgrades require continuity verified by runtime policy, not a
matching caller-supplied name.

`retention: "session"` binds a job to its creating endpoint. Closing that
endpoint stops transfers and retires the job. Data retention follows storage
policy and does not grant a later endpoint access. `retention: "persistent"`
requires advertised support and explicit shell policy admission. Admission
MUST persist the job record, identity, settings, source lease, and recoverable
transfer state before acknowledging creation. Such jobs can outlive endpoint
closure and runtime restarts; backend state is reconciled before serving reads.
Loss of source data or permission is reported as an error, never a fabricated
successful recovery. Persistent records remain discoverable until removed by
the user or an authorized caller. Revoked access prevents rediscovery.

A new endpoint recovers retained jobs with `jobs()` and refreshes each with
`status()`. No event replay is implied. State/progress/error messages are sent
only to currently authorized endpoints. On permission revocation, networking
and pending work subject to that grant stop. Pending reads, exports, and saves
terminate with `permission-revoked`; removal terminates them with
`job-not-found`. Endpoint closure cancels its pending requests and temporary
range priorities without stopping otherwise-authorized persistent jobs.

Unknown hashes, totals, and progress MUST be omitted before they are known;
zero is a known value, not an unknown sentinel. An unknown/unauthorized job
returns `job-not-found`. Every live-endpoint request receives exactly one
terminal result or error; `reason` is diagnostic only. Successful `add` returns
`jobId` and `state`; failed acknowledgements MUST NOT imply a job was created.

## Discovery and Nostr Integration

This section is informative. Discovery, event conversion, publishing, and
comments are outside the `torrent` domain.

A napplet can discover a torrent through existing Nostr capabilities, convert
its metadata locally, then pass a magnet link or info hash to `add`. For
example, a [NIP-35](https://github.com/nostr-protocol/nips/blob/master/35.md)
search napplet can use `outbox` for reads and publishing, with `relay` as an
explicit escape hatch. Those operations remain subject to their owning
contracts and shell policy. Conversion can live in a napplet library or a
shell adapter. A shell can also integrate discovery internally.

These integrations do not add dependencies to `torrent`. The transfer contract
is the same regardless of where the napplet discovered the torrent.

## Wire Protocol

`torrent.*` messages use the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `torrent.capabilities` | napplet -> runtime | `id` |
| `torrent.capabilities.result` | runtime -> napplet | `id`, `capabilities?`, `error?` |
| `torrent.pickFiles` | napplet -> runtime | `id`, `options?` |
| `torrent.pickFiles.result` | runtime -> napplet | `id`, `result` |
| `torrent.add` | napplet -> runtime | `id`, `request` |
| `torrent.add.result` | runtime -> napplet | `id`, `ack` |
| `torrent.status` | napplet -> runtime | `id`, `jobId` |
| `torrent.status.result` | runtime -> napplet | `id`, `status?`, `error?` |
| `torrent.jobs` | napplet -> runtime | `id`, `filter?` |
| `torrent.jobs.result` | runtime -> napplet | `id`, `jobs?`, `error?` |
| `torrent.files` | napplet -> runtime | `id`, `jobId` |
| `torrent.files.result` | runtime -> napplet | `id`, `files?`, `error?` |
| `torrent.selectFiles` | napplet -> runtime | `id`, `jobId`, `selection` |
| `torrent.selectFiles.result` | runtime -> napplet | `id`, `status?`, `error?` |
| `torrent.pause` | napplet -> runtime | `id`, `jobId` |
| `torrent.pause.result` | runtime -> napplet | `id`, `status?`, `error?` |
| `torrent.resume` | napplet -> runtime | `id`, `jobId` |
| `torrent.resume.result` | runtime -> napplet | `id`, `status?`, `error?` |
| `torrent.remove` | napplet -> runtime | `id`, `jobId`, `options?` |
| `torrent.remove.result` | runtime -> napplet | `id`, `result` |
| `torrent.metadata` | napplet -> runtime | `id`, `jobId` |
| `torrent.metadata.result` | runtime -> napplet | `id`, `result` |
| `torrent.read` | napplet -> runtime | `id`, `request` |
| `torrent.read.result` | runtime -> napplet | `id`, `result` |
| `torrent.exportMetainfo` | napplet -> runtime | `id`, `jobId` |
| `torrent.exportMetainfo.result` | runtime -> napplet | `id`, `result` |
| `torrent.saveFile` | napplet -> runtime | `id`, `jobId`, `fileIndex` |
| `torrent.saveFile.result` | runtime -> napplet | `id`, `result` |
| `torrent.openFile` | napplet -> runtime | `id`, `jobId`, `fileIndex` |
| `torrent.openFile.result` | runtime -> napplet | `id`, `result` |
| `torrent.cancel` | napplet -> runtime | `id`, `requestId` |
| `torrent.cancel.result` | runtime -> napplet | `id`, `result` |
| `torrent.update` | napplet -> runtime | `id`, `request` |
| `torrent.update.result` | runtime -> napplet | `id`, `result` |
| `torrent.state` | runtime -> napplet | `jobId`, `state`, `status?` |
| `torrent.progress` | runtime -> napplet | `jobId`, `status` |
| `torrent.error` | runtime -> napplet | `jobId`, `error`, `reason?` |

Key design notes:
- Request/result pairs use `id` for correlation.
- Transfer push messages use `jobId`; they have no `id`.
- `torrent.add.result` acknowledges job creation only. Completion, seeding, or
  failure arrives through `torrent.state`, `torrent.progress`, or
  `torrent.error`.
- `progress` is `0..1`. Rates are bytes per second. Ratio is
  `uploadedBytes / downloadedBytes`, or `0` when no bytes have been downloaded.
- `transport: "auto"` lets the runtime choose between supported engines.
- `runtime-file-set` sources are runtime-owned handles from `pickFiles`.
  They are not filesystem paths and do not survive endpoint closure.

### Examples

**Start a download from a magnet link:**

```
-> {"type": "torrent.add", "id": "a1", "request": {"source": {"sourceType": "magnet", "magnet": "magnet:?xt=urn:btih:0123456789abcdef0123456789abcdef01234567&dn=archive"}, "options": {"mode": "download-and-seed", "transport": "auto", "storage": "user-selected"}}}
<- {"type": "torrent.add.result", "id": "a1", "ack": {"ok": true, "jobId": "tor-1", "state": "metadata"}}
<- {"type": "torrent.progress", "jobId": "tor-1", "status": {"ok": true, "jobId": "tor-1", "state": "downloading", "mode": "download-and-seed", "transport": "bittorrent", "title": "archive", "downloadedBytes": 1048576, "uploadedBytes": 0, "totalBytes": 734003200, "progress": 0.0014, "downloadRate": 2000000, "uploadRate": 0, "ratio": 0, "peers": {"peers": 24, "seeds": 8, "leeches": 16}, "hashes": {"v1": "0123456789abcdef0123456789abcdef01234567"}, "settings": {"revision": 0, "downloadLimitBytesPerSecond": 0, "uploadLimitBytesPerSecond": 0, "seedAfterComplete": true, "trackers": [], "webSeeds": [], "endpointsRedacted": false}, "retention": "session"}}
```

**Pick local files and start a seed/upload job:**

```
-> {"type": "torrent.pickFiles", "id": "f1", "options": {"multiple": true, "suggestedName": "archive"}}
<- {"type": "torrent.pickFiles.result", "id": "f1", "result": {"ok": true, "fileSet": {"runtimeFileSetId": "fs-1", "files": [{"path": "archive.tar", "size": 42}], "totalBytes": 42, "displayName": "archive"}}}
-> {"type": "torrent.add", "id": "u1", "request": {"source": {"sourceType": "runtime-file-set", "runtimeFileSetId": "fs-1"}, "options": {"mode": "seed", "transport": "auto", "uploadLimitBytesPerSecond": 500000}}}
<- {"type": "torrent.add.result", "id": "u1", "ack": {"ok": true, "jobId": "tor-seed-1", "state": "seeding"}}
```

**Pause and remove a transfer:**

```
-> {"type": "torrent.pause", "id": "x1", "jobId": "tor-1"}
<- {"type": "torrent.pause.result", "id": "x1", "status": {"ok": true, "jobId": "tor-1", "state": "paused", "mode": "download-and-seed", "transport": "bittorrent", "downloadedBytes": 1048576, "uploadedBytes": 0, "totalBytes": 734003200, "progress": 0.0014, "downloadRate": 0, "uploadRate": 0, "ratio": 0, "peers": {"peers": 0, "seeds": 0, "leeches": 0}, "hashes": {"v1": "0123456789abcdef0123456789abcdef01234567"}, "settings": {"revision": 0, "downloadLimitBytesPerSecond": 0, "uploadLimitBytesPerSecond": 0, "seedAfterComplete": true, "trackers": [], "webSeeds": [], "endpointsRedacted": false}, "retention": "session"}}
-> {"type": "torrent.remove", "id": "x2", "jobId": "tor-1", "options": {"deleteData": true}}
<- {"type": "torrent.remove.result", "id": "x2", "result": {"ok": true, "jobId": "tor-1", "deletedData": true}}
```

**Read verified file bytes:**

Here `tor-text` is an authorized job containing the complete five-byte file `hello.txt`.

```
-> {"type": "torrent.read", "id": "read-1", "request": {"jobId": "tor-text", "fileIndex": 0, "offset": 0, "length": 5, "waitMs": 1000}}
<- {"type": "torrent.read.result", "id": "read-1", "result": {"ok": true, "offset": 0, "dataBase64": "aGVsbG8=", "byteLength": 5, "totalBytes": 5, "eof": true}}
```

**Cancel a buffering read before seeking:**

```
-> {"type": "torrent.read", "id": "read-2", "request": {"jobId": "tor-1", "fileIndex": 0, "offset": 0, "length": 65536, "waitMs": 1000}}
-> {"type": "torrent.cancel", "id": "cancel-1", "requestId": "read-2"}
<- {"type": "torrent.cancel.result", "id": "cancel-1", "result": {"ok": true, "cancelled": true}}
<- {"type": "torrent.read.result", "id": "read-2", "result": {"ok": false, "error": "cancelled"}}
-> {"type": "torrent.read", "id": "read-3", "request": {"jobId": "tor-1", "fileIndex": 0, "offset": 1048576, "length": 65536, "waitMs": 0}}
<- {"type": "torrent.read.result", "id": "read-3", "result": {"ok": false, "error": "not-ready"}}
```

**Export a newly created torrent:**

The exported metainfo describes the same `hello.txt` bytes as the read example.

```
-> {"type": "torrent.metadata", "id": "meta-1", "jobId": "tor-text"}
<- {"type": "torrent.metadata.result", "id": "meta-1", "result": {"ok": true, "format": "v1", "hashes": {"v1": "0eb2d544ef0c9ae3c0fbba90a71cd742a44aa81d"}, "name": "hello.txt", "private": false, "magnet": "magnet:?xt=urn:btih:0eb2d544ef0c9ae3c0fbba90a71cd742a44aa81d", "trackers": [], "webSeeds": [], "endpointsRedacted": false, "metainfoReady": true}}
-> {"type": "torrent.exportMetainfo", "id": "export-1", "jobId": "tor-text"}
<- {"type": "torrent.exportMetainfo.result", "id": "export-1", "result": {"ok": true, "dataBase64": "ZDQ6aW5mb2Q2Omxlbmd0aGk1ZTQ6bmFtZTk6aGVsbG8udHh0MTI6cGllY2UgbGVuZ3RoaTE2Mzg0ZTY6cGllY2VzMjA6qvTGHdzF6KLavt4PO0gs2a6pQ01lZQ==", "byteLength": 91, "hashes": {"v1": "0eb2d544ef0c9ae3c0fbba90a71cd742a44aa81d"}}}
```

**Update limits and detect a stale revision:**

```
-> {"type": "torrent.update", "id": "update-1", "request": {"jobId": "tor-1", "expectedRevision": 0, "patch": {"uploadLimitBytesPerSecond": 500000}}}
<- {"type": "torrent.update.result", "id": "update-1", "result": {"ok": true, "settings": {"revision": 1, "downloadLimitBytesPerSecond": 0, "uploadLimitBytesPerSecond": 500000, "seedAfterComplete": true, "trackers": [], "webSeeds": [], "endpointsRedacted": false}}}
-> {"type": "torrent.update", "id": "update-2", "request": {"jobId": "tor-1", "expectedRevision": 0, "patch": {"trackers": []}}}
<- {"type": "torrent.update.result", "id": "update-2", "result": {"ok": false, "error": "revision-conflict", "settings": {"revision": 1, "downloadLimitBytesPerSecond": 0, "uploadLimitBytesPerSecond": 500000, "seedAfterComplete": true, "trackers": [], "webSeeds": [], "endpointsRedacted": false}}}
```

**Save or hand off a completed file:**

```
-> {"type": "torrent.saveFile", "id": "save-1", "jobId": "tor-text", "fileIndex": 0}
<- {"type": "torrent.saveFile.result", "id": "save-1", "result": {"ok": true, "jobId": "tor-text", "fileIndex": 0, "outcome": "saved"}}
-> {"type": "torrent.openFile", "id": "open-1", "jobId": "tor-text", "fileIndex": 0}
<- {"type": "torrent.openFile.result", "id": "open-1", "result": {"ok": true, "jobId": "tor-text", "fileIndex": 0, "outcome": "accepted"}}
```

**Create a retained job and rediscover it after reconnect:**

The `jobs` request comes from a new endpoint admitted as the same napplet principal and profile. Unknown progress is omitted.

```
-> {"type": "torrent.add", "id": "persist-1", "request": {"source": {"sourceType": "infohash", "hashes": {"v1": "0eb2d544ef0c9ae3c0fbba90a71cd742a44aa81d"}}, "options": {"retention": "persistent", "storage": "persistent"}}}
<- {"type": "torrent.add.result", "id": "persist-1", "ack": {"ok": true, "jobId": "retained-1", "state": "metadata"}}
-> {"type": "torrent.jobs", "id": "recover-1"}
<- {"type": "torrent.jobs.result", "id": "recover-1", "jobs": [{"jobId": "retained-1", "state": "metadata", "retention": "persistent", "mode": "download", "transport": "bittorrent", "hashes": {"v1": "0eb2d544ef0c9ae3c0fbba90a71cd742a44aa81d"}, "downloadRate": 0, "uploadRate": 0, "ratio": 0}]}
```

## Error Handling

Common errors: `invalid-request`, `invalid-range`, `file-not-selected`,
`not-ready`, `metadata-unavailable`, `timeout`, `cancelled`, `permission-revoked`,
`unsupported-format`, `hash-mismatch`, `unsupported-setting`,
`revision-conflict`, `job-conflict`, `source-unavailable`, `invalid-source`, `invalid-infohash`, `invalid-magnet`,
`unsupported-source`, `unsupported-transport`, `unsupported-mode`,
`user-denied`, `metadata-timeout`, `tracker-failed`, `dht-unavailable`, `no-peers`,
`storage-denied`, `quota-exceeded`, `too-large`, `job-not-found`,
`file-pick-cancelled`, `unsupported-state`, `policy-denied`, `network-error`,
`unsupported`.

Typed result records use `ok: false` with a required machine-readable `error`
on failure and no success-only fields. `TorrentSettingsResult.settings` is
also present on `revision-conflict`. On success, `ok: true` requires every
success field and excludes `error`. Operations whose wire result has optional
`status`, `jobs`, `files`, or `capabilities` return exactly one of that payload
or a top-level `error`. Failure MUST NOT fabricate a status snapshot.

## Runtime Behavior

- MUST keep torrent network transports runtime-owned.
- MUST verify content before reads, exports, or handoff, and separately authorize
  disclosure of local seed data. File selection and seeding are not read grants.
- MUST enforce private-torrent restrictions on discovery and peer exchange;
  tracker changes MUST NOT bypass these restrictions.
- MUST bound metadata size, pending operations, decoded buffers, and total
  retained storage per verified napplet principal.
- MUST NOT expose arbitrary TCP, UDP, WebRTC, HTTP, tracker, DHT, or filesystem
  access through this interface.
- MUST scope jobs and storage visibility to the requesting napplet unless the
  user explicitly grants broader access.
- MUST keep `runtimeFileSetId` handles opaque and scoped to the requesting
  napplet.
- MUST enforce runtime policy for trackers, DHT, peer discovery, web seeds,
  bandwidth, storage, retention, and seeding.
- SHOULD report progress through `torrent.progress` at a throttled cadence.
- MAY use WebTorrent, standard BitTorrent, a local daemon, or a remote runtime
  service as the backend.
- MAY require user consent before starting downloads, seeding, deleting data,
  or exposing existing jobs.

## Security Considerations

- Torrent transfers are network egress and ingress. Runtimes MUST treat every
  transfer as untrusted until policy and consent checks pass.
- Torrent metadata can name misleading paths. Runtimes MUST sanitize paths and
  MUST NOT allow archive-style traversal or absolute paths to escape the chosen
  storage root.
- Seeding reveals interest and can upload user data. Runtimes SHOULD show
  torrent title, info hash, size, mode, backend, tracker/DHT policy, and seeding
  policy before starting a job.
- Torrent metadata and source hints are untrusted. An info hash identifies
  torrent content; it does not authenticate its publisher or establish trust
  in file names, titles, or tracker URLs.
- Trackers, DHT, peers, and web seeds can observe the user's IP address unless
  the runtime provides network isolation. Runtimes SHOULD expose that policy in
  consent UI.
- `runtime-file-set` handles can represent user-selected local files. The
  runtime MUST NOT let napplets infer local paths or enumerate user files beyond
  explicit grants.
- Deleting data is destructive. Runtimes SHOULD require user confirmation for
  `deleteData: true`.

## References

- [NIP-35](https://github.com/nostr-protocol/nips/blob/master/35.md) -- Torrents
- [BEP-3](https://www.bittorrent.org/beps/bep_0003.html) -- BitTorrent v1
- [BEP-9](https://www.bittorrent.org/beps/bep_0009.html) -- Metadata exchange and magnets
- [BEP-52](https://www.bittorrent.org/beps/bep_0052.html) -- BitTorrent v2 and hybrid torrents
- [BEP-53](https://www.bittorrent.org/beps/bep_0053.html) -- Magnet URI extension

## Implementations

- None yet.

## Changelog

- `3330324` - Introduced NAP-TORRENT for runtime-owned torrent jobs.
- `3bc9c3a` - Changed torrent source events to use relay-owned result shape and declared the relay dependency.

- `c0200f0` - Adopted injected-domain availability and linked the current upstream web binding.
- `fe66d8e` - Restricted the contract to torrent transfers with generic, validated sources; removed Nostr discovery, publishing, comments, event conversion, event schemas, and dependencies, and kept discovery integration informative.
