# NAP-TORRENT

## Runtime-Mediated Torrents

`draft`

**NAP ID:** NAP-TORRENT
**Domain:** `torrent`
**Depends:**
- `identity` -- capability · required -- publishing NIP-35 torrent and comment events uses the current shell-user identity as event author.
- `relay` -- capability · required -- searches, reads, comments on, and publishes NIP-35 torrent events through the runtime relay surface.
**Web binding (NIP-5D):** `window.napplet.torrent` · `shell.supports("torrent")`

## Description

NAP-TORRENT gives napplets a runtime-mediated interface for NIP-35 torrent
indexes and torrent transfers. A napplet can search, parse, publish, and comment
on NIP-35 kind `2003` torrent events, then ask the runtime to download, seed, or
inspect the referenced torrent.

The runtime owns the torrent engine, transport choice, storage, file picker,
bandwidth limits, tracker/DHT/WebTorrent policy, relay use, signing, and user
consent. Napplets never get raw TCP, UDP, WebRTC, HTTP tracker, DHT, filesystem,
or signing access through this interface.

This NAP supports both WebTorrent-style runtimes and standard BitTorrent
runtimes. A shell may use HTTP/WebRTC/WebSocket transports, TCP/UDP/DHT
transports, local torrent daemons, or another backend. The napplet requests the
intent and observes a job. It does not own the transport.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `capabilities` | none | `TorrentCapabilities` | `torrent.capabilities` / `.result` |
| `fromEvent` | `event` (`NostrEvent`) | `TorrentDescriptorResult` | `torrent.fromEvent` / `.result` |
| `toMagnet` | `descriptor` (`TorrentDescriptor`) | `TorrentMagnetResult` | `torrent.toMagnet` / `.result` |
| `search` | `query` (`TorrentSearchQuery`) | list of `TorrentSearchResult` | `torrent.search` / `.result` |
| `publish` | `request` (`TorrentPublishRequest`) | `TorrentPublishResult` | `torrent.publish` / `.result` |
| `comment` | `request` (`TorrentCommentRequest`) | `TorrentPublishResult` | `torrent.comment` / `.result` |
| `pickFiles` | optional `options` (`TorrentPickOptions`) | `TorrentFileSetResult` | `torrent.pickFiles` / `.result` |
| `add` | `request` (`TorrentAddRequest`) | `TorrentJobAck` | `torrent.add` / `.result`, then push messages |
| `status` | `jobId` (`tstr`) | `TorrentJobStatus` | `torrent.status` / `.result` |
| `jobs` | optional `filter` (`TorrentJobFilter`) | list of `TorrentJobSummary` | `torrent.jobs` / `.result` |
| `files` | `jobId` (`tstr`) | list of `TorrentFileStatus` | `torrent.files` / `.result` |
| `selectFiles` | `jobId` (`tstr`), `selection` (`TorrentFileSelection`) | `TorrentJobStatus` | `torrent.selectFiles` / `.result` |
| `pause` | `jobId` (`tstr`) | `TorrentJobStatus` | `torrent.pause` / `.result` |
| `resume` | `jobId` (`tstr`) | `TorrentJobStatus` | `torrent.resume` / `.result` |
| `remove` | `jobId` (`tstr`), optional `options` (`TorrentRemoveOptions`) | `TorrentRemoveResult` | `torrent.remove` / `.result` |

### Schemas

```cddl
HexInfoHash = tstr
HexPubkey = tstr
NostrEventId = tstr
NostrEvent = { * tstr => any }
NostrTag = [* tstr]

TorrentTransport = "webtorrent" / "bittorrent" / "auto" / tstr
TorrentMode = "download" / "seed" / "download-and-seed"
TorrentJobState = "queued" / "metadata" / "downloading" / "seeding" /
                  "paused" / "complete" / "cancelled" / "error"
TorrentSourceType = "nip35" / "magnet" / "infohash" / "metainfo" / "runtime-file-set"

TorrentCapabilities = {
  transports: [* TorrentTransport],
  modes: [* TorrentMode],
  ? maxActiveJobs: uint,
  ? maxTorrentSize: uint,
  ? canPublishNip35: bool,
  ? canComment: bool,
  ? canSeedRuntimeFiles: bool,
  ? canUseDht: bool,
  ? canUseTrackers: bool,
  ? canUseWebSeeds: bool,
}

TorrentFile = {
  path: tstr,
  ? size: uint,
}

TorrentReference = {
  prefix: "tcat" / "newznab" / "tmdb" / "ttvdb" / "imdb" / "mal" / "anilist" / tstr,
  value: tstr,
}

TorrentDescriptor = {
  infoHash: HexInfoHash,
  ? title: tstr,
  ? description: tstr,
  files: [* TorrentFile],
  trackers: [* tstr],
  tags: [* tstr],
  references: [* TorrentReference],
  ? magnet: tstr,
  ? eventId: NostrEventId,
  ? author: HexPubkey,
  ? createdAt: uint,
  ? event: NostrEvent,
}

TorrentDescriptorResult = {
  ok: bool,
  ? descriptor: TorrentDescriptor,
  ? error: tstr,
  ? reason: tstr,
}

TorrentMagnetResult = {
  ok: bool,
  ? magnet: tstr,
  ? error: tstr,
  ? reason: tstr,
}

TorrentSearchQuery = {
  ? text: tstr,
  ? infoHash: HexInfoHash,
  ? tags: [* tstr],
  ? references: [* TorrentReference],
  ? authors: [* HexPubkey],
  ? since: uint,
  ? until: uint,
  ? limit: uint,
}

TorrentSearchResult = {
  descriptor: TorrentDescriptor,
  event: NostrEvent,
  ? relays: [* tstr],
}

TorrentPublishRequest = {
  descriptor: TorrentDescriptor,
  ? content: tstr,
  ? relays: [* tstr],
  ? publish: bool, ; default true; false returns an unsigned preview only
}

TorrentCommentRequest = {
  torrentEventId: NostrEventId,
  torrentAuthor: HexPubkey,
  content: tstr,
  ? relays: [* tstr],
}

TorrentPublishResult = {
  ok: bool,
  ? eventId: NostrEventId,
  ? event: NostrEvent,
  ? relays: { * tstr => bool },
  ? error: tstr,
  ? reason: tstr,
}

TorrentPickOptions = {
  ? multiple: bool,
  ? suggestedName: tstr,
}

TorrentRuntimeFileSet = {
  runtimeFileSetId: tstr,
  files: [* TorrentFile],
  totalBytes: uint,
  ? displayName: tstr,
}

TorrentFileSetResult = {
  ok: bool,
  ? fileSet: TorrentRuntimeFileSet,
  ? error: tstr,
  ? reason: tstr,
}

TorrentSource = {
  sourceType: TorrentSourceType,
  ? event: NostrEvent,
  ? descriptor: TorrentDescriptor,
  ? magnet: tstr,
  ? infoHash: HexInfoHash,
  ? metainfo: bstr,
  ? runtimeFileSetId: tstr,
}

TorrentAddOptions = {
  ? mode: TorrentMode,              ; default "download"
  ? transport: TorrentTransport,    ; default "auto"
  ? startPaused: bool,
  ? seedAfterComplete: bool,
  ? downloadLimitBytesPerSecond: uint,
  ? uploadLimitBytesPerSecond: uint,
  ? storage: "temporary" / "persistent" / "user-selected",
  ? trackers: [* tstr],
  ? webSeeds: [* tstr],
  ? fileSelection: TorrentFileSelection,
}

TorrentAddRequest = {
  ? jobId: tstr,
  source: TorrentSource,
  ? options: TorrentAddOptions,
}

TorrentJobAck = {
  ok: bool,
  ? jobId: tstr,
  ? state: TorrentJobState,
  ? error: tstr,
  ? reason: tstr,
}

TorrentPeerStats = {
  peers: uint,
  seeds: uint,
  leeches: uint,
}

TorrentJobStatus = {
  ok: bool,
  jobId: tstr,
  state: TorrentJobState,
  mode: TorrentMode,
  transport: TorrentTransport,
  infoHash: HexInfoHash,
  ? title: tstr,
  downloadedBytes: uint,
  uploadedBytes: uint,
  totalBytes: uint,
  progress: number,
  downloadRate: number,
  uploadRate: number,
  ratio: number,
  ? etaSeconds: uint,
  peers: TorrentPeerStats,
  ? storage: "temporary" / "persistent" / "user-selected",
  ? error: tstr,
  ? reason: tstr,
}

TorrentJobSummary = {
  jobId: tstr,
  state: TorrentJobState,
  mode: TorrentMode,
  transport: TorrentTransport,
  infoHash: HexInfoHash,
  ? title: tstr,
  progress: number,
  downloadRate: number,
  uploadRate: number,
  ratio: number,
}

TorrentJobFilter = {
  ? states: [* TorrentJobState],
  ? modes: [* TorrentMode],
}

TorrentFileStatus = {
  index: uint,
  path: tstr,
  size: uint,
  selected: bool,
  priority: int,
  downloadedBytes: uint,
  progress: number,
}

TorrentFileSelection = {
  ? indexes: [* uint],
  ? paths: [* tstr],
  ? priority: int,
}

TorrentRemoveOptions = {
  ? deleteData: bool,
}

TorrentRemoveResult = {
  ok: bool,
  jobId: tstr,
  ? deletedData: bool,
  ? error: tstr,
  ? reason: tstr,
}
```

**`capabilities()`** -- Reports the runtime's torrent features for this
napplet. `transports` names supported backend families, not a promise that a
specific tracker, peer, or network path will work.

**`fromEvent(event)`** -- Validates a NIP-35 kind `2003` event and converts its
tags into a `TorrentDescriptor`. It MUST read `x` as the v1 BitTorrent info
hash, `title` as the title, `file` tags as file path plus optional size,
`tracker` tags as tracker hints, `t` tags as search tags, and `i` tags as
prefixed references. It MUST reject events without a valid `x` tag.

**`toMagnet(descriptor)`** -- Builds a magnet URI using
`xt=urn:btih:<infoHash>`. It MAY include `dn` from `title` and `tr` values from
`trackers`. It MUST NOT require a `.torrent` file, because NIP-35 indexes do not
store torrent files on Nostr.

**`search(query)`** -- Searches for NIP-35 kind `2003` events through runtime
relay policy, indexes, or cache. It returns normalized descriptors plus source
events. Search matching is runtime-owned; the stable contract is the query
shape and returned descriptor.

**`publish(request)`** -- Builds a NIP-35 kind `2003` torrent event from a
descriptor, signs it as the current shell-user, and publishes it through the
runtime relay surface. `publish: false` returns a preview event without
publishing. The runtime owns tag formatting and SHOULD preserve only NIP-35
fields it can validate.

**`comment(request)`** -- Publishes a NIP-35 kind `2004` comment that replies
to a torrent event. It MUST follow NIP-10 reply tagging for the referenced
torrent.

**`pickFiles(options?)`** -- Opens a runtime-owned user file picker and returns
a `runtimeFileSetId` handle. The handle can be used only as a
`runtime-file-set` source for a seed/upload job. It is not a filesystem path and
does not let the napplet read the selected bytes.

**`add(request)`** -- Creates a runtime transfer job from a NIP-35 event,
descriptor, magnet link, info hash, metainfo bytes, or runtime-owned file set.
The returned `jobId` is scoped to the requesting napplet. Transfers run in the
runtime, not inside the napplet.

For upload/seeding, callers use `sourceType: "runtime-file-set"` with
`options.mode: "seed"` or `"download-and-seed"`. The runtime may create
metainfo, start seeding, and optionally return a descriptor through job status
once the info hash is known.

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

## NIP-35 Mapping

| NIP-35 element | NAP-TORRENT field |
|----------------|-------------------|
| kind `2003` | torrent index event consumed by `fromEvent`, `search`, `publish`, and `add` |
| `content` | `TorrentDescriptor.description` or `publish.content` |
| `["title", value]` | `TorrentDescriptor.title` |
| `["x", hash]` | `TorrentDescriptor.infoHash` |
| `["file", path, size?]` | `TorrentDescriptor.files[]` |
| `["tracker", url]` | `TorrentDescriptor.trackers[]` |
| `["t", tag]` | `TorrentDescriptor.tags[]` |
| `["i", "prefix:value"]` | `TorrentDescriptor.references[]` |
| kind `2004` | `comment` result event |

`TorrentReference.prefix` follows the NIP-35 prefix set:
`tcat`, `newznab`, `tmdb`, `ttvdb`, `imdb`, `mal`, and `anilist`. Unknown
prefixes MAY be preserved.

## Wire Protocol

`torrent.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `torrent.capabilities` | napplet -> runtime | `id` |
| `torrent.capabilities.result` | runtime -> napplet | `id`, `capabilities?`, `error?` |
| `torrent.fromEvent` | napplet -> runtime | `id`, `event` |
| `torrent.fromEvent.result` | runtime -> napplet | `id`, `result` |
| `torrent.toMagnet` | napplet -> runtime | `id`, `descriptor` |
| `torrent.toMagnet.result` | runtime -> napplet | `id`, `result` |
| `torrent.search` | napplet -> runtime | `id`, `query` |
| `torrent.search.result` | runtime -> napplet | `id`, `results?`, `error?` |
| `torrent.publish` | napplet -> runtime | `id`, `request` |
| `torrent.publish.result` | runtime -> napplet | `id`, `result` |
| `torrent.comment` | napplet -> runtime | `id`, `request` |
| `torrent.comment.result` | runtime -> napplet | `id`, `result` |
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

**Search NIP-35, convert to a magnet, then start a download:**

```
-> { "type": "torrent.search", "id": "s1", "query": { "text": "dune", "tags": ["movie"], "limit": 10 } }
<- { "type": "torrent.search.result", "id": "s1",
     "results": [{ "descriptor": { "infoHash": "0123...", "title": "Dune", "files": [], "trackers": ["udp://tracker.example:1337"], "tags": ["movie"], "references": [] }, "event": { "kind": 2003, "tags": [] } }] }
-> { "type": "torrent.add", "id": "a1",
     "request": { "source": { "sourceType": "nip35", "descriptor": { "infoHash": "0123...", "title": "Dune", "files": [], "trackers": ["udp://tracker.example:1337"], "tags": ["movie"], "references": [] } },
                  "options": { "mode": "download-and-seed", "transport": "auto", "storage": "user-selected" } } }
<- { "type": "torrent.add.result", "id": "a1", "ack": { "ok": true, "jobId": "tor-1", "state": "metadata" } }
<- { "type": "torrent.progress", "jobId": "tor-1",
     "status": { "ok": true, "jobId": "tor-1", "state": "downloading", "mode": "download-and-seed", "transport": "bittorrent", "infoHash": "0123...", "title": "Dune", "downloadedBytes": 1048576, "uploadedBytes": 0, "totalBytes": 734003200, "progress": 0.0014, "downloadRate": 2000000, "uploadRate": 0, "ratio": 0, "peers": { "peers": 24, "seeds": 8, "leeches": 16 } } }
```

**Publish a NIP-35 torrent index:**

```
-> { "type": "torrent.publish", "id": "p1",
     "request": { "descriptor": { "infoHash": "abcd...", "title": "archive", "description": "public data set", "files": [{ "path": "archive.tar", "size": 42 }], "trackers": [], "tags": ["archive"], "references": [] } } }
<- { "type": "torrent.publish.result", "id": "p1",
     "result": { "ok": true, "eventId": "event...", "event": { "kind": 2003, "content": "public data set", "tags": [["title", "archive"], ["x", "abcd..."], ["file", "archive.tar", "42"], ["t", "archive"]] } } }
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
<- { "type": "torrent.pause.result", "id": "x1", "status": { "ok": true, "jobId": "tor-1", "state": "paused", "mode": "download-and-seed", "transport": "bittorrent", "infoHash": "0123...", "downloadedBytes": 1048576, "uploadedBytes": 0, "totalBytes": 734003200, "progress": 0.0014, "downloadRate": 0, "uploadRate": 0, "ratio": 0, "peers": { "peers": 0, "seeds": 0, "leeches": 0 } } }
-> { "type": "torrent.remove", "id": "x2", "jobId": "tor-1", "options": { "deleteData": true } }
<- { "type": "torrent.remove.result", "id": "x2", "result": { "ok": true, "jobId": "tor-1", "deletedData": true } }
```

## Error Handling

Common errors: `invalid-event`, `invalid-infohash`, `invalid-magnet`,
`unsupported-source`, `unsupported-transport`, `unsupported-mode`,
`not-signed-in`, `user-denied`, `relay-timeout`, `publish-failed`,
`metadata-timeout`, `tracker-failed`, `dht-unavailable`, `no-peers`,
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
- MUST validate NIP-35 kind `2003` events before treating them as torrent
  descriptors.
- MUST construct NIP-35 kind `2003` publish events from validated descriptor
  fields, not from arbitrary caller-supplied tag arrays.
- MUST construct kind `2004` comments with NIP-10 reply tags.
- MUST perform all signing and publishing through runtime-owned identity and
  relay policy.
- MUST scope jobs and storage visibility to the requesting napplet unless the
  user explicitly grants broader access.
- MUST keep `runtimeFileSetId` handles opaque and scoped to the requesting
  napplet.
- MUST enforce runtime policy for trackers, DHT, peer discovery, web seeds,
  bandwidth, storage, retention, and seeding.
- SHOULD report progress through `torrent.progress` at a throttled cadence.
- MAY use WebTorrent, standard BitTorrent, a local daemon, or a remote runtime
  service as the backend.
- MAY require user consent before starting downloads, seeding, publishing
  torrent indexes, deleting data, or exposing existing jobs.

## Security Considerations

- Torrent transfers are network egress and ingress. Runtimes MUST treat every
  transfer as untrusted until policy and consent checks pass.
- Torrent metadata can name misleading paths. Runtimes MUST sanitize paths and
  MUST NOT allow archive-style traversal or absolute paths to escape the chosen
  storage root.
- Seeding reveals interest and can upload user data. Runtimes SHOULD show
  torrent title, info hash, size, mode, backend, tracker/DHT policy, and seeding
  policy before starting a job.
- NIP-35 events are unsigned claims by their authors. The info hash is an
  integrity key, but titles, descriptions, file names, tags, references, and
  tracker URLs are untrusted.
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
- [NIP-10](https://github.com/nostr-protocol/nips/blob/master/10.md) -- Text Notes and Threads
- [BEP-53](https://www.bittorrent.org/beps/bep_0053.html) -- Magnet URI extension

## Implementations

- None yet.
