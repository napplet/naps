# NAP-HASHTREE

## Runtime-Mediated Hashtree Document Trees

`draft`

**NAP ID:** NAP-HASHTREE
**Domain:** `hashtree`
**Depends:**
- `identity` -- capability · required -- publishing mutable Hashtree roots uses the current shell-user identity as event author.
- `relay` -- capability · required -- resolves and publishes Hashtree root events through the runtime relay surface.
- `relay` -- wire · required -- imports `RelayEventResult` for resolved mutable-root event returns.
- `resource` -- wire · optional -- imported `RelayEventResult.sidecar.resources?` carries `ResourceSidecarEntry[]`, a type owned by the `resource` domain.
**Web binding (NIP-5D):** `window.napplet.hashtree` · `shell.supports("hashtree")`

## Description

NAP-HASHTREE gives napplets a runtime-mediated interface for
[Hashtree](https://github.com/mmalmi/hashtree) document trees. A napplet can
store bytes, create directories, resolve immutable `nhash` CIDs, resolve mutable
`htree://<identifier>/<path>` roots, read files, list directories, publish root
events, and manage pins/import/export jobs.

Hashtree's upstream protocol is implemented and has a draft spec (`HTS-01`).
This NAP treats HTS-01 as the source for Hashtree bytes: SHA-256 content
addresses, deterministic MessagePack tree nodes, CHK encryption, `nhash`
identifiers, Nostr kind `30064` mutable roots, and `htree://` URLs. The NAP
defines the napplet-facing runtime API around those concepts.

The runtime owns storage backends, Blossom/FIPS/local-daemon routing, CHK key
handling, root-event signing, relay selection, pin retention, filesystem
pickers, and user consent. Napplets never receive direct Blossom, FIPS,
filesystem, or signing access through this interface.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `capabilities` | none | `HashtreeCapabilities` | `hashtree.capabilities` / `.result` |
| `parseRef` | `ref` (`tstr`) | `HashtreeRefResult` | `hashtree.parseRef` / `.result` |
| `resolve` | `ref` (`tstr`), optional `options` (`HashtreeResolveOptions`) | `HashtreeResolveResult` | `hashtree.resolve` / `.result` |
| `stat` | `ref` (`tstr`) | `HashtreeEntryResult` | `hashtree.stat` / `.result` |
| `list` | `ref` (`tstr`), optional `options` (`HashtreeListOptions`) | `HashtreeListResult` | `hashtree.list` / `.result` |
| `read` | `ref` (`tstr`), optional `options` (`HashtreeReadOptions`) | `HashtreeReadResult` | `hashtree.read` / `.result` |
| `putBytes` | `request` (`HashtreePutBytesRequest`) | `HashtreePutResult` | `hashtree.putBytes` / `.result` |
| `putTree` | `request` (`HashtreePutTreeRequest`) | `HashtreePutResult` | `hashtree.putTree` / `.result` |
| `publishRoot` | `request` (`HashtreePublishRootRequest`) | `HashtreePublishRootResult` | `hashtree.publishRoot` / `.result` |
| `pickFiles` | optional `options` (`HashtreePickOptions`) | `HashtreeFileSetResult` | `hashtree.pickFiles` / `.result` |
| `importFiles` | `request` (`HashtreeImportRequest`) | `HashtreeJobAck` | `hashtree.importFiles` / `.result`, then push messages |
| `exportFiles` | `request` (`HashtreeExportRequest`) | `HashtreeJobAck` | `hashtree.exportFiles` / `.result`, then push messages |
| `pins` | none | list of `HashtreePin` | `hashtree.pins` / `.result` |
| `pin` | `request` (`HashtreePinRequest`) | `HashtreePinResult` | `hashtree.pin` / `.result` |
| `unpin` | `ref` (`tstr`) | `HashtreePinResult` | `hashtree.unpin` / `.result` |
| `job` | `jobId` (`tstr`) | `HashtreeJobStatus` | `hashtree.job` / `.result` |
| `cancel` | `jobId` (`tstr`) | `HashtreeJobStatus` | `hashtree.cancel` / `.result` |

### Schemas

`RelayEventResult` is owned by NAP-RELAY. NAP-HASHTREE imports it by name for
root events returned while resolving mutable Hashtree references. `publishRoot`
still returns its shell-created signed event as `NostrEvent`.

```cddl
HexHash = tstr
HexKey = tstr
HexPubkey = tstr
NostrEvent = { * tstr => any }
NostrEventId = tstr
; External type: RelayEventResult, owned by NAP-RELAY.

HashtreeRefKind = "nhash" / "cid" / "htree" / "npub-path"
HashtreeEntryType = "blob" / "file" / "dir"
HashtreeVisibility = "unencrypted" / "public" / "link-visible" / "private"
HashtreeJobKind = "import" / "export"
HashtreeJobState = "queued" / "running" / "complete" / "cancelled" / "error"

HashtreeCapabilities = {
  ? canReadImmutable: bool,
  ? canResolveMutable: bool,
  ? canWrite: bool,
  ? canPublishRoots: bool,
  ? canPin: bool,
  ? canImportRuntimeFiles: bool,
  ? canExportRuntimeFiles: bool,
  ? supportsFips: bool,
  ? supportsBlossom: bool,
  ? supportsLocalStore: bool,
  ? maxReadBytes: uint,
  ? maxWriteBytes: uint,
}

HashtreeCid = {
  hash: HexHash,
  ? key: HexKey,
  ? nhash: tstr,
}

HashtreeRef = {
  kind: HashtreeRefKind,
  ? ref: tstr,
  ? cid: HashtreeCid,
  ? identifier: tstr,
  ? treePath: tstr,
  ? entryPath: tstr,
  ? visibility: HashtreeVisibility,
  ? shareSecret: HexKey,
}

HashtreeRefResult = {
  ok: bool,
  ? ref: HashtreeRef,
  ? error: tstr,
  ? reason: tstr,
}

HashtreeResolveOptions = {
  ? shareSecret: HexKey,
  ? at: uint,
}

HashtreeResolvedRoot = {
  cid: HashtreeCid,
  ? ref: HashtreeRef,
  ? rootResult: RelayEventResult,
  ? eventId: NostrEventId,
  ? author: HexPubkey,
  ? treeName: tstr,
  ? visibility: HashtreeVisibility,
}

HashtreeResolveResult = {
  ok: bool,
  ? root: HashtreeResolvedRoot,
  ? error: tstr,
  ? reason: tstr,
}

HashtreeEntry = {
  path: tstr,
  type: HashtreeEntryType,
  cid: HashtreeCid,
  ? name: tstr,
  ? size: uint,
  ? mime: tstr,
}

HashtreeEntryResult = {
  ok: bool,
  ? entry: HashtreeEntry,
  ? error: tstr,
  ? reason: tstr,
}

HashtreeListOptions = {
  ? recursive: bool,
  ? limit: uint,
}

HashtreeListResult = {
  ok: bool,
  entries: [* HashtreeEntry],
  ? root: HashtreeResolvedRoot,
  ? truncated: bool,
  ? error: tstr,
  ? reason: tstr,
}

HashtreeReadOptions = {
  ? offset: uint,
  ? length: uint,
  ? maxBytes: uint,
}

HashtreeReadResult = {
  ok: bool,
  ? blob: bstr,
  ? entry: HashtreeEntry,
  ? root: HashtreeResolvedRoot,
  ? error: tstr,
  ? reason: tstr,
}

HashtreePutOptions = {
  ? visibility: HashtreeVisibility, ; default "public"
  ? mime: tstr,
  ? filename: tstr,
  ? pin: bool,
}

HashtreePutBytesRequest = {
  data: bstr,
  ? options: HashtreePutOptions,
}

HashtreeTreeEntryInput = {
  path: tstr,
  ? data: bstr,
  ? cid: HashtreeCid,
  ? mime: tstr,
}

HashtreePutTreeRequest = {
  entries: [* HashtreeTreeEntryInput],
  ? options: HashtreePutOptions,
}

HashtreePutResult = {
  ok: bool,
  ? cid: HashtreeCid,
  ? entry: HashtreeEntry,
  ? size: uint,
  ? error: tstr,
  ? reason: tstr,
}

HashtreePublishRootRequest = {
  cid: HashtreeCid,
  treeName: tstr,
  ? content: tstr,
  ? visibility: HashtreeVisibility,
  ? shareSecret: HexKey,
  ? relays: [* tstr],
}

HashtreePublishRootResult = {
  ok: bool,
  ? eventId: NostrEventId,
  ? event: NostrEvent,
  ? ref: tstr,
  ? relays: { * tstr => bool },
  ? error: tstr,
  ? reason: tstr,
}

HashtreeImportPublishOptions = {
  treeName: tstr,
  ? content: tstr,
  ? visibility: HashtreeVisibility,
  ? shareSecret: HexKey,
  ? relays: [* tstr],
}

HashtreePickOptions = {
  ? multiple: bool,
  ? directory: bool,
  ? suggestedName: tstr,
}

HashtreeRuntimeFileSet = {
  runtimeFileSetId: tstr,
  files: [* HashtreeEntry],
  totalBytes: uint,
  ? displayName: tstr,
}

HashtreeFileSetResult = {
  ok: bool,
  ? fileSet: HashtreeRuntimeFileSet,
  ? error: tstr,
  ? reason: tstr,
}

HashtreeImportRequest = {
  runtimeFileSetId: tstr,
  ? visibility: HashtreeVisibility,
  ? pin: bool,
  ? publish: HashtreeImportPublishOptions,
}

HashtreeExportRequest = {
  ref: tstr,
  ? destination: "user-selected" / "temporary",
}

HashtreeJobAck = {
  ok: bool,
  ? jobId: tstr,
  ? state: HashtreeJobState,
  ? error: tstr,
  ? reason: tstr,
}

HashtreeJobStatus = {
  ok: bool,
  jobId: tstr,
  kind: HashtreeJobKind,
  state: HashtreeJobState,
  processedBytes: uint,
  totalBytes: uint,
  progress: number,
  ? cid: HashtreeCid,
  ? ref: tstr,
  ? runtimeFileSetId: tstr,
  ? error: tstr,
  ? reason: tstr,
}

HashtreePin = {
  ref: tstr,
  cid: HashtreeCid,
  ? label: tstr,
  ? bytesStored: uint,
  ? createdAt: uint,
}

HashtreePinRequest = {
  ref: tstr,
  ? label: tstr,
}

HashtreePinResult = {
  ok: bool,
  ? pin: HashtreePin,
  ? error: tstr,
  ? reason: tstr,
}
```

**`capabilities()`** -- Reports the runtime's Hashtree support for this
napplet. Backend flags describe runtime policy, not direct napplet access.

**`parseRef(ref)`** -- Parses `nhash`, CID text (`<hash>` or `<hash>:<key>`),
`htree://`, and `npub/path` references without fetching data. `hashtree:nhash`
MUST parse as the same immutable reference as `nhash`.

**`resolve(ref, options?)`** -- Resolves immutable or mutable references to a
root CID. For mutable references, the runtime queries Nostr root events, chooses
the newest root using HTS-01 ordering, and derives public, link-visible, or
private root keys when authorized. Returned mutable-root events use
`RelayEventResult`.

**`stat(ref)`** -- Returns metadata for a blob, file, or directory without
returning file bytes.

**`list(ref, options?)`** -- Lists directory entries. The runtime decodes and
verifies Hashtree nodes before returning entries. `recursive` can be capped by
runtime policy.

**`read(ref, options?)`** -- Reads a blob or file. The runtime fetches chunks
from local storage, Blossom, FIPS peers, or another configured backend, verifies
hashes, decrypts when a key is available, and returns bytes. It MUST reject a
directory read with `"is-directory"`.

**`putBytes(request)`** -- Stores bytes as a Hashtree blob/file and returns its
CID. The runtime applies HTS-01 chunking, node encoding, and optional CHK
encryption. It MAY upload stored objects to configured remote stores.

**`putTree(request)`** -- Builds a Hashtree directory from supplied bytes and/or
existing CIDs. Paths are logical tree paths, not filesystem paths. The runtime
normalizes and rejects traversal.

**`publishRoot(request)`** -- Publishes a Nostr kind `30064` root event with
`d`, `l`, and `hash` tags, plus the appropriate visibility tag. The runtime
signs as the current shell-user and owns relay fanout.

**`pickFiles(options?)`** -- Opens a runtime-owned file or directory picker and
returns an opaque `runtimeFileSetId`. The handle can be imported, but it does
not expose local paths or bytes to the napplet.

**`importFiles(request)`** -- Stores a runtime-owned file set as a Hashtree tree. It
is a long-running job because large directories may need chunking, hashing,
uploading, pinning, and optional root publication.

**`exportFiles(request)`** -- Materializes a Hashtree ref into a runtime-owned file
set or user-selected destination. It is a long-running job because it may need
network fetches, verification, decryption, and directory reconstruction.

**`pins()` / `pin(request)` / `unpin(ref)`** -- Manage runtime retention for
Hashtree CIDs. Pinning is a storage policy request, not a publication event.

**`job(jobId)` / `cancel(jobId)`** -- Inspect or cancel import/export jobs.

## HTS-01 Mapping

| HTS-01 concept | NAP-HASHTREE field or behavior |
|----------------|--------------------------------|
| SHA-256 content address | `HashtreeCid.hash` |
| CID key | `HashtreeCid.key` |
| `nhash` | `HashtreeCid.nhash` and immutable `parseRef` output |
| MessagePack file/dir node | runtime-owned implementation behind `putTree`, `list`, and `read` |
| CHK encryption | `visibility`, `key`, `shareSecret`, and runtime encryption policy |
| kind `30064` root event | `publishRoot`, `resolve`, `HashtreeResolvedRoot.rootResult` |
| `d` tag | `treeName` |
| `hash` tag | `HashtreeCid.hash` |
| `key`, `encryptedKey`, `selfEncryptedKey` tags | root visibility and key recovery |
| `htree://<identifier>/<path>` | mutable `HashtreeRef` |

`htree://` tree names that contain `/` MUST follow HTS-01 URL encoding guidance:
encode the tree name as one path segment before appending entry path segments.

## Wire Protocol

`hashtree.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `hashtree.capabilities` | napplet -> runtime | `id` |
| `hashtree.capabilities.result` | runtime -> napplet | `id`, `capabilities?`, `error?` |
| `hashtree.parseRef` | napplet -> runtime | `id`, `ref` |
| `hashtree.parseRef.result` | runtime -> napplet | `id`, `result` |
| `hashtree.resolve` | napplet -> runtime | `id`, `ref`, `options?` |
| `hashtree.resolve.result` | runtime -> napplet | `id`, `result` |
| `hashtree.stat` | napplet -> runtime | `id`, `ref` |
| `hashtree.stat.result` | runtime -> napplet | `id`, `result` |
| `hashtree.list` | napplet -> runtime | `id`, `ref`, `options?` |
| `hashtree.list.result` | runtime -> napplet | `id`, `result` |
| `hashtree.read` | napplet -> runtime | `id`, `ref`, `options?` |
| `hashtree.read.result` | runtime -> napplet | `id`, `result` |
| `hashtree.putBytes` | napplet -> runtime | `id`, `request` |
| `hashtree.putBytes.result` | runtime -> napplet | `id`, `result` |
| `hashtree.putTree` | napplet -> runtime | `id`, `request` |
| `hashtree.putTree.result` | runtime -> napplet | `id`, `result` |
| `hashtree.publishRoot` | napplet -> runtime | `id`, `request` |
| `hashtree.publishRoot.result` | runtime -> napplet | `id`, `result` |
| `hashtree.pickFiles` | napplet -> runtime | `id`, `options?` |
| `hashtree.pickFiles.result` | runtime -> napplet | `id`, `result` |
| `hashtree.importFiles` | napplet -> runtime | `id`, `request` |
| `hashtree.importFiles.result` | runtime -> napplet | `id`, `ack` |
| `hashtree.exportFiles` | napplet -> runtime | `id`, `request` |
| `hashtree.exportFiles.result` | runtime -> napplet | `id`, `ack` |
| `hashtree.pins` | napplet -> runtime | `id` |
| `hashtree.pins.result` | runtime -> napplet | `id`, `pins?`, `error?` |
| `hashtree.pin` | napplet -> runtime | `id`, `request` |
| `hashtree.pin.result` | runtime -> napplet | `id`, `result` |
| `hashtree.unpin` | napplet -> runtime | `id`, `ref` |
| `hashtree.unpin.result` | runtime -> napplet | `id`, `result` |
| `hashtree.job` | napplet -> runtime | `id`, `jobId` |
| `hashtree.job.result` | runtime -> napplet | `id`, `status?`, `error?` |
| `hashtree.cancel` | napplet -> runtime | `id`, `jobId` |
| `hashtree.cancel.result` | runtime -> napplet | `id`, `status?`, `error?` |
| `hashtree.progress` | runtime -> napplet | `jobId`, `status` |
| `hashtree.done` | runtime -> napplet | `jobId`, `status` |
| `hashtree.error` | runtime -> napplet | `jobId`, `error`, `reason?` |

Key design notes:
- Request/result pairs use `id` for correlation.
- Import/export push messages use `jobId`; they have no `id`.
- Binary `data` and `blob` fields cross the boundary as projection-native byte
  objects. In the web projection that means `Blob` or `ArrayBuffer`, not
  base64 strings.
- `runtimeFileSetId` is runtime-scoped. It is not a path and cannot be reused
  outside the grant that produced it.
- Root publication and resolution are part of this domain. Generic Nostr relay
  access remains in NAP-RELAY.

### Examples

**Resolve and read a public file:**

```
-> { "type": "hashtree.resolve", "id": "r1", "ref": "htree://npub1.../docs/readme.txt" }
<- { "type": "hashtree.resolve.result", "id": "r1",
     "result": { "ok": true, "root": { "cid": { "hash": "ab12..." }, "treeName": "docs", "visibility": "unencrypted" } } }
-> { "type": "hashtree.read", "id": "r2", "ref": "htree://npub1.../docs/readme.txt" }
<- { "type": "hashtree.read.result", "id": "r2",
     "result": { "ok": true, "blob": "<bytes>", "entry": { "path": "readme.txt", "type": "file", "cid": { "hash": "cd34..." }, "size": 1200, "mime": "text/plain" } } }
```

**Store bytes and publish a root:**

```
-> { "type": "hashtree.putBytes", "id": "p1",
     "request": { "data": "<bytes>", "options": { "visibility": "public", "filename": "note.txt", "pin": true } } }
<- { "type": "hashtree.putBytes.result", "id": "p1",
     "result": { "ok": true, "cid": { "hash": "ab12...", "key": "ef56...", "nhash": "nhash1..." }, "entry": { "path": "note.txt", "type": "file", "cid": { "hash": "ab12...", "key": "ef56..." }, "size": 512 } } }
-> { "type": "hashtree.publishRoot", "id": "p2",
     "request": { "cid": { "hash": "ab12...", "key": "ef56..." }, "treeName": "notes", "visibility": "public" } }
<- { "type": "hashtree.publishRoot.result", "id": "p2",
     "result": { "ok": true, "eventId": "event...", "ref": "htree://npub1.../notes" } }
```

**Import a user-selected directory:**

```
-> { "type": "hashtree.pickFiles", "id": "f1", "options": { "directory": true, "suggestedName": "site" } }
<- { "type": "hashtree.pickFiles.result", "id": "f1",
     "result": { "ok": true, "fileSet": { "runtimeFileSetId": "fs-1", "files": [], "totalBytes": 7340032, "displayName": "site" } } }
-> { "type": "hashtree.importFiles", "id": "i1",
     "request": { "runtimeFileSetId": "fs-1", "visibility": "link-visible", "pin": true,
                  "publish": { "treeName": "site", "visibility": "link-visible" } } }
<- { "type": "hashtree.importFiles.result", "id": "i1", "ack": { "ok": true, "jobId": "ht-1", "state": "queued" } }
<- { "type": "hashtree.done", "jobId": "ht-1",
     "status": { "ok": true, "jobId": "ht-1", "kind": "import", "state": "complete", "processedBytes": 7340032, "totalBytes": 7340032, "progress": 1, "cid": { "hash": "ab12...", "key": "ef56..." }, "ref": "htree://npub1.../site#k=..." } }
```

## Error Handling

Common errors: `invalid-ref`, `invalid-cid`, `invalid-tree`, `invalid-path`,
`not-found`, `missing-key`, `decrypt-failed`, `unsupported-visibility`,
`not-signed-in`, `user-denied`, `relay-timeout`, `publish-failed`,
`fetch-failed`, `store-failed`, `too-large`, `quota-exceeded`, `is-directory`,
`not-directory`, `file-pick-cancelled`, `job-not-found`, `policy-denied`,
`unsupported`.

The runtime SHOULD return structured `ok: false` results when the request was
understood but refused. It SHOULD use top-level `error` only when no typed
result can be constructed.

## Runtime Behavior

- MUST validate all returned bytes by SHA-256 before delivering them.
- MUST reject deterministic MessagePack tree nodes that violate HTS-01 ordering
  or type rules.
- MUST keep CHK keys, link-visible share secrets, and private root keys under
  runtime policy.
- MUST construct kind `30064` root events with `d`, `l`, and `hash` tags.
- MUST sign root events as the current shell-user.
- MUST choose relays, Blossom servers, FIPS peers, local caches, and fallback
  order by runtime policy.
- SHOULD merge observed mutable-root relay URLs into
  `RelayEventResult.sidecar.relayHints` when it can disclose them.
- MAY include pre-resolved resources referenced by mutable-root events in
  `RelayEventResult.sidecar.resources`; NAP-RELAY's default-off sidecar privacy
  policy and NAP-RESOURCE's fetch policy both apply.
- MUST NOT expose raw Blossom, FIPS, filesystem, or signing access through this
  interface.
- MUST sanitize logical paths and reject absolute paths, `..`, and equivalent
  traversal.
- MUST scope pins, jobs, and runtime file handles to the requesting napplet
  unless the user explicitly grants broader access.
- SHOULD cap recursive listing, read size, import size, export size, and
  concurrent jobs.
- MAY support legacy root kind `30078` when resolving, as HTS-01 permits.

## Security Considerations

- A CID with a CHK key is a read capability. Runtimes SHOULD avoid exposing keys
  unless needed by the napplet and authorized by the user.
- Link-visible roots carry a share secret in a fragment. Runtimes MUST NOT send
  that secret to relays, Blossom servers, FIPS peers, or HTTP request bodies.
- Private roots require user identity and NIP-44 decryption. Runtimes MUST reject
  private resolution when no authorized signer/decrypter is connected.
- Importing local files can exfiltrate user data. Consent UI SHOULD show file
  count, approximate bytes, visibility mode, publication target, and retention
  policy before starting.
- Exporting can materialize untrusted filenames. Runtimes MUST sanitize names and
  prevent traversal outside the chosen destination.
- FIPS and Blossom fetches can leak interest. Runtimes SHOULD disclose network
  path policy when resolving remote roots.
- Pins consume storage and may retain sensitive encrypted blobs. Runtimes SHOULD
  make pin scope and retention visible.

## References

- [Hashtree](https://github.com/mmalmi/hashtree)
- [HTS-01: hashtree Core Protocol](https://github.com/mmalmi/hashtree/blob/master/docs/HTS-01.md)
- [Hashtree URL Encoding](https://github.com/mmalmi/hashtree/blob/master/docs/URL-ENCODING.md)
- [Hashtree on FIPS](https://github.com/mmalmi/hashtree/blob/master/docs/hashtree-on-fips.md)

## Implementations

- None yet.
