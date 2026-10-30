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
runtime-owned file sets.

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

### Schemas

Primitive types:

| Name | Definition |
|------|------------|
| `HexInfoHash` | v1 BitTorrent info hash encoded as 40 lowercase hexadecimal characters |
| `MetainfoBytes` | binary contents of a `.torrent` file; the host representation is projection-defined |

Enums:

| Name | Values |
|------|--------|
| `TorrentTransport` | `webtorrent`, `bittorrent`, `auto`, or runtime-defined text |
| `TorrentMode` | `download`, `seed`, `download-and-seed` |
| `TorrentJobState` | `queued`, `metadata`, `downloading`, `seeding`, `paused`, `complete`, `cancelled`, `error` |
| `TorrentSourceType` | `magnet`, `infohash`, `metainfo`, `runtime-file-set` |
| `TorrentStorage` | `temporary`, `persistent`, `user-selected` |

#### TorrentCapabilities

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `transports` | yes | list of TorrentTransport | — |
| `modes` | yes | list of TorrentMode | — |
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
| `infoHash` | no | HexInfoHash | Required when selected by `sourceType`. |
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
| `storage` | no | TorrentStorage | — |
| `trackers` | no | list of text | — |
| `webSeeds` | no | list of text | — |
| `fileSelection` | no | TorrentFileSelection | — |

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
| `infoHash` | yes | HexInfoHash | — |
| `downloadedBytes` | yes | integer | Non-negative. |
| `uploadedBytes` | yes | integer | Non-negative. |
| `totalBytes` | yes | integer | Non-negative. |
| `progress` | yes | number | — |
| `downloadRate` | yes | number | — |
| `uploadRate` | yes | number | — |
| `ratio` | yes | number | — |
| `peers` | yes | TorrentPeerStats | — |
| `title` | no | text | — |
| `etaSeconds` | no | integer | Non-negative. |
| `storage` | no | TorrentStorage | — |
| `error` | no | text | — |
| `reason` | no | text | — |

#### TorrentJobSummary

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `jobId` | yes | text | — |
| `state` | yes | TorrentJobState | — |
| `mode` | yes | TorrentMode | — |
| `transport` | yes | TorrentTransport | — |
| `infoHash` | yes | HexInfoHash | — |
| `progress` | yes | number | — |
| `downloadRate` | yes | number | — |
| `uploadRate` | yes | number | — |
| `ratio` | yes | number | — |
| `title` | no | text | — |

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

When `TorrentAddOptions.mode` is omitted, the default is `download`.
When `TorrentAddOptions.transport` is omitted, the default is `auto`.

**`capabilities()`** -- Reports the runtime's torrent features for this
napplet. `transports` names supported backend families, not a promise that a
specific tracker, peer, or network path will work.

**`pickFiles(options?)`** -- Opens a runtime-owned user file picker and returns
a `runtimeFileSetId` handle. The handle can be used only as a
`runtime-file-set` source for a seed/upload job. It is not a filesystem path and
does not let the napplet read the selected bytes.

**`add(request)`** -- Creates a runtime transfer job from a magnet link,
info hash, metainfo bytes, or runtime-owned file set. The runtime MUST validate
the source before starting a transfer. The returned `jobId` is scoped to the
requesting napplet. Transfers run in the runtime.

`TorrentSource.sourceType` selects exactly one source field. The selected
field MUST be present; other source fields MUST be absent.

| `sourceType` | Required source field |
|--------------|-----------------------|
| `magnet` | `magnet` |
| `infohash` | `infoHash` |
| `metainfo` | `metainfo` |
| `runtime-file-set` | `runtimeFileSetId` |

For upload/seeding, callers use `sourceType: "runtime-file-set"` with
`options.mode: "seed"` or `"download-and-seed"`. The runtime may create
metainfo and start seeding. The resulting info hash is reported through job
status once known.

**`status(jobId)`** -- Returns a one-shot job status snapshot.

**`jobs(filter?)`** -- Lists the requesting napplet's torrent jobs. A runtime
MAY also expose user-approved global torrent jobs, but MUST mark them with job
ids scoped to this napplet session.

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
- `runtime-file-set` sources are runtime-owned handles, usually from a user
  picker or a prior completed download. They are not filesystem paths.

### Examples

**Start a download from a magnet link:**

```
-> { "type": "torrent.add", "id": "a1",
     "request": { "source": { "sourceType": "magnet", "magnet": "magnet:?xt=urn:btih:0123456789abcdef0123456789abcdef01234567&dn=archive" },
                  "options": { "mode": "download-and-seed", "transport": "auto", "storage": "user-selected" } } }
<- { "type": "torrent.add.result", "id": "a1", "ack": { "ok": true, "jobId": "tor-1", "state": "metadata" } }
<- { "type": "torrent.progress", "jobId": "tor-1",
     "status": { "ok": true, "jobId": "tor-1", "state": "downloading", "mode": "download-and-seed", "transport": "bittorrent", "infoHash": "0123456789abcdef0123456789abcdef01234567", "title": "archive", "downloadedBytes": 1048576, "uploadedBytes": 0, "totalBytes": 734003200, "progress": 0.0014, "downloadRate": 2000000, "uploadRate": 0, "ratio": 0, "peers": { "peers": 24, "seeds": 8, "leeches": 16 } } }
```

**Pick local files and start a seed/upload job:**

```
-> { "type": "torrent.pickFiles", "id": "f1", "options": { "multiple": true, "suggestedName": "archive" } }
<- { "type": "torrent.pickFiles.result", "id": "f1",
     "result": { "ok": true, "fileSet": { "runtimeFileSetId": "fs-1", "files": [{ "path": "archive.tar", "size": 42 }], "totalBytes": 42, "displayName": "archive" } } }
-> { "type": "torrent.add", "id": "u1",
     "request": { "source": { "sourceType": "runtime-file-set", "runtimeFileSetId": "fs-1" },
                  "options": { "mode": "seed", "transport": "auto", "uploadLimitBytesPerSecond": 500000 } } }
<- { "type": "torrent.add.result", "id": "u1", "ack": { "ok": true, "jobId": "tor-seed-1", "state": "seeding" } }
```

**Pause and remove a transfer:**

```
-> { "type": "torrent.pause", "id": "x1", "jobId": "tor-1" }
<- { "type": "torrent.pause.result", "id": "x1", "status": { "ok": true, "jobId": "tor-1", "state": "paused", "mode": "download-and-seed", "transport": "bittorrent", "infoHash": "0123456789abcdef0123456789abcdef01234567", "downloadedBytes": 1048576, "uploadedBytes": 0, "totalBytes": 734003200, "progress": 0.0014, "downloadRate": 0, "uploadRate": 0, "ratio": 0, "peers": { "peers": 0, "seeds": 0, "leeches": 0 } } }
-> { "type": "torrent.remove", "id": "x2", "jobId": "tor-1", "options": { "deleteData": true } }
<- { "type": "torrent.remove.result", "id": "x2", "result": { "ok": true, "jobId": "tor-1", "deletedData": true } }
```

## Error Handling

Common errors: `invalid-source`, `invalid-infohash`, `invalid-magnet`,
`unsupported-source`, `unsupported-transport`, `unsupported-mode`,
`user-denied`, `metadata-timeout`, `tracker-failed`, `dht-unavailable`, `no-peers`,
`storage-denied`, `quota-exceeded`, `too-large`, `job-not-found`,
`file-pick-cancelled`, `unsupported-state`, `policy-denied`, `network-error`,
`unsupported`.

The runtime SHOULD return structured `ok: false` results when an operation was
understood but refused. It SHOULD use top-level `error` only when no typed
result can be constructed.

## Runtime Behavior

- MUST keep torrent network transports runtime-owned.
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
- [BEP-53](https://www.bittorrent.org/beps/bep_0053.html) -- Magnet URI extension

## Implementations

- None yet.

## Changelog

- `3330324` - Introduced NAP-TORRENT for runtime-owned torrent jobs.
- `3bc9c3a` - Changed torrent source events to use relay-owned result shape and declared the relay dependency.

- `c0200f0` - Adopted injected-domain availability and linked the current upstream web binding.
- `fe66d8e` - Restricted the contract to torrent transfers with generic, validated sources; removed Nostr discovery, publishing, comments, event conversion, event schemas, and dependencies, and kept discovery integration informative.
