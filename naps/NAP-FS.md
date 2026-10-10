NAP-FS
======

Virtual Filesystem Access
-------------------------

`draft`

**NAP ID:** NAP-FS
**Domain:** `fs`
**Depends:** none.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.fs`; domain presence signals availability.

## Description

NAP-FS provides napplets with shell-mediated access to a runtime-owned virtual
filesystem. The filesystem may be backed by local files, origin-private storage,
remote storage, a database, or any other runtime implementation. A napplet sees
only virtual paths, directory entries, file metadata, byte reads and writes,
user-mediated picker results, and advisory change events.

Backing storage can be shared across napplets. Each napplet still sees a
policy-bound view with its own permissions. Every operation is authorized
against the runtime-bound napplet identity. Paths, permission lists, metadata,
and options supplied by the napplet are not authority. Only runtime policy and
runtime-mediated user choice can expand the visible view.

NAP-FS defines the filesystem seam. It does not define host paths, mount sources,
picker UI, sync backends, OS permissions, or storage layout. Picker operations
return only virtual paths in the napplet's visible filesystem.

## Path Model

Paths are virtual absolute paths in the napplet's visible filesystem.

- A path MUST start with `/`.
- `/` is the only path separator.
- `/` identifies the visible virtual root.
- Empty segments are invalid except for the root `/`.
- `.` and `..` segments are invalid.
- Host drive prefixes, file URLs, and OS-native separators are invalid.
- Path segments are Unicode strings, not URLs. A runtime MUST NOT percent-decode
  a path or interpret lookalike characters as `/`.
- Control characters and bidirectional formatting characters are invalid.
- A runtime MUST apply one consistent Unicode normalization and case policy
  before authorization. Creation and move MUST fail with `conflict` when the
  destination collides with an entry under that policy.
- A runtime MUST normalize and validate paths before authorization and MUST
  revalidate the normalized path immediately before committing a mutation.
- A runtime MUST resolve any backing-store aliases, links, and mounts before
  authorization. It MUST authorize the final objects and revalidate them
  immediately before committing a mutation.
- Authorization MUST compare parsed path segments against policy roots. String
  prefix comparison is insufficient: `/shared/app` does not contain
  `/shared/app-private`.
- A runtime MUST reject any path that escapes the napplet's authorized view.

## Shared Filesystem Policy

A shared backing store does not imply shared write access. Read, list, write,
create, delete, and watch permissions are independent. A runtime that exposes a
root mutable by multiple napplets MUST apply explicit policy to every mutation.

- Shared readable roots MUST NOT become shared writable roots by implication.
- A napplet MUST NOT mutate an entry outside its authorized mutable scope, even
  when it can read or list the entry.
- Overwrite, recursive removal, and move in a shared mutable root MUST be
  explicitly allowed by runtime policy. The runtime MAY require user mediation.
- Authorization MUST use current runtime policy and the runtime-bound napplet
  identity. Entry names, metadata, and request options are not authority.
- A mutation that cannot prevent unauthorized effects across its authorization
  and commit steps MUST fail with `conflict`, `policy-denied`, or `unsupported`.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `info` | none | `FsInfo` | `fs.info` / `fs.info.result` |
| `pickFile` | optional `options` (`FsPickOptions`) | `FsPickResult` | `fs.pickFile` / `fs.pickFile.result` |
| `pickFiles` | optional `options` (`FsPickOptions`) | `FsPickResult` | `fs.pickFiles` / `fs.pickFiles.result` |
| `pickDirectory` | optional `options` (`FsPickOptions`) | `FsPickResult` | `fs.pickDirectory` / `fs.pickDirectory.result` |
| `pickSaveFile` | optional `options` (`FsPickOptions`) | `FsPickResult` | `fs.pickSaveFile` / `fs.pickSaveFile.result` |
| `stat` | `path` (text) | `FsMetadata` | `fs.stat` / `fs.stat.result` |
| `list` | `path` (text) | list of `FsDirectoryEntry` | `fs.list` / `fs.list.result` |
| `read` | `path` (text), optional `options` (`FsReadOptions`) | `FsReadResult` | `fs.read` / `fs.read.result` |
| `write` | `path` (text), `data` (text, base64-encoded bytes), optional `options` (`FsWriteOptions`) | `FsWriteResult` | `fs.write` / `fs.write.result` |
| `mkdir` | `path` (text), optional `options` (`FsMkdirOptions`) | none | `fs.mkdir` / `fs.mkdir.result` |
| `remove` | `path` (text), optional `recursive` (boolean) | none | `fs.remove` / `fs.remove.result` |
| `move` | `fromPath` (text), `toPath` (text) | none | `fs.move` / `fs.move.result` |
| `watch` | `path` (text), optional `options` (`FsWatchOptions`) | `watchId` (text) | `fs.watch` / `fs.watch.result` |
| `unwatch` | `watchId` (text) | none | `fs.unwatch` / `fs.unwatch.result` |

### Schemas

`FsPermission` values:

| Value | Notes |
|-------|-------|
| `"read"` | Read file bytes. |
| `"write"` | Write existing file bytes. |
| `"create"` | Create new entries. |
| `"delete"` | Remove entries. |
| `"list"` | List directory entries. |
| `"watch"` | Watch visible changes. |

`FsEntryKind` values:

| Value | Notes |
|-------|-------|
| `"file"` | File entry. |
| `"directory"` | Directory entry. |
| `"unknown"` | Runtime cannot safely expose file or directory semantics. |

`FsWriteMode` values:

| Value | Notes |
|-------|-------|
| `"replace"` | Replace the whole file. |
| `"append"` | Append to the file. |
| `"patch"` | Write at an offset. |

`FsChangeKind` values:

| Value | Notes |
|-------|-------|
| `"created"` | Entry was created. |
| `"modified"` | Entry was modified. |
| `"deleted"` | Entry was deleted. |
| `"moved"` | Entry was moved. |
| `"unknown"` | Runtime cannot classify the change. |

`FsError` values:

| Value | Notes |
|-------|-------|
| `"not-found"` | Entry was absent or hidden. |
| `"already-exists"` | Destination already exists. |
| `"not-a-file"` | Operation requires a file. |
| `"not-a-directory"` | Operation requires a directory. |
| `"invalid-path"` | Path violates the path model. |
| `"invalid-data"` | Byte payload is malformed. |
| `"permission-denied"` | Permission is absent. |
| `"policy-denied"` | Runtime policy denies the operation. |
| `"quota-exceeded"` | Storage quota would be exceeded. |
| `"too-large"` | Request exceeds a size limit. |
| `"unsupported"` | Runtime does not support the requested behavior. |
| `"conflict"` | Precondition, concurrency, or mutation conflict. |
| `"cancelled"` | User or runtime cancelled the operation. |
| `"io-error"` | Backing store failed. |

`FsInfo` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `roots` | yes | list of `FsRoot` | Visible roots. |
| `limits` | yes | `FsLimits` | Runtime limits. |

`FsRoot` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `path` | yes | text | Virtual absolute root path. |
| `name` | yes | text | Runtime-curated display label. |
| `permissions` | yes | list of `FsPermission` | Coarse permissions advertised for this root. |
| `description` | no | text | Runtime-curated description. |

`FsLimits` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `maxReadBytes` | yes | integer | Non-negative. Maximum decoded bytes requested by one `read`. |
| `maxWriteBytes` | yes | integer | Non-negative. Maximum decoded bytes supplied by one `write`. |
| `maxWatchCount` | no | integer | Non-negative. Maximum active watches. |
| `maxInFlightRequests` | no | integer | Non-negative. Maximum active requests. |
| `maxInFlightBytes` | no | integer | Non-negative. Maximum aggregate decoded bytes in active byte operations. |

`FsAcceptRule` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `mime` | no | text | Advisory MIME filter. |
| `extension` | no | text | Advisory extension filter. |

`FsPickOptions` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `permissions` | no | list of `FsPermission` | Requested permission intent. |
| `accept` | no | list of `FsAcceptRule` | Advisory UI filters. |
| `suggestedName` | no | text | Advisory save name. |
| `description` | no | text | Advisory picker description. |

`FsPickedEntry` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `path` | yes | text | Virtual absolute path. |
| `kind` | yes | `"file"` or `"directory"` | Picked entry kind. |
| `name` | yes | text | Entry name. |
| `permissions` | yes | list of `FsPermission` | Permissions granted for the picked entry. |
| `size` | no | integer | Non-negative. Size in bytes. |
| `modifiedAt` | no | integer | Non-negative. Runtime-defined timestamp. |

`FsPickResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `entries` | yes | list of `FsPickedEntry` | Picked entries. |

`FsMetadata` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `path` | yes | text | Virtual absolute path. |
| `kind` | yes | `FsEntryKind` | Entry kind. |
| `size` | no | integer | Non-negative. Size in bytes. |
| `modifiedAt` | no | integer | Non-negative. Runtime-defined timestamp. |
| `createdAt` | no | integer | Non-negative. Runtime-defined timestamp. |
| `permissions` | no | list of `FsPermission` | Coarse permissions known for this entry. |
| `revision` | no | text | Opaque write-precondition token. |

`FsDirectoryEntry` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `name` | yes | text | Entry name. |
| `path` | yes | text | Virtual absolute path. |
| `kind` | yes | `FsEntryKind` | Entry kind. |
| `size` | no | integer | Non-negative. Size in bytes. |
| `modifiedAt` | no | integer | Non-negative. Runtime-defined timestamp. |

`FsReadOptions` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `offset` | no | integer | Non-negative. Starting byte offset. |
| `length` | no | integer | Non-negative. Requested decoded byte count. |

`FsReadResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `data` | yes | text | Decoded file bytes, encoded as standard padded base64 on the JSON wire. |
| `offset` | yes | integer | Non-negative. Starting byte offset. |
| `bytesRead` | yes | integer | Non-negative. Count of decoded bytes in `data`. |
| `eof` | yes | boolean | Whether no more bytes are available after this result. |
| `size` | no | integer | Non-negative. Total file size in bytes. |

`FsWriteOptions` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `mode` | no | `FsWriteMode` | Write mode. |
| `offset` | no | integer | Non-negative. Patch byte offset. |
| `ifRevision` | no | text | Opaque revision precondition. |
| `ifAbsent` | no | boolean | Create-only precondition when true. |

`FsWriteResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `bytesWritten` | yes | integer | Non-negative. Count of decoded bytes committed. |
| `size` | no | integer | Non-negative. Resulting file size in bytes. |

`FsMkdirOptions` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `recursive` | no | boolean | Whether missing parent directories may be created. |

`FsWatchOptions` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `recursive` | no | boolean | Whether directory watches include visible descendants. |

`FsChange` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `watchId` | yes | text | Runtime-generated watch id. |
| `path` | yes | text | Virtual absolute path. |
| `kind` | yes | `FsChangeKind` | Change kind. |
| `fromPath` | no | text | Previous virtual path for moved entries. |

**`info()`** — Returns visible roots, coarse root permissions, and runtime limits.
This is advisory discovery, not authorization. Permissions can change during a
session, a root can contain paths with narrower policy, and any operation can
still fail.

`info()` MUST NOT expose host paths, volume names, backing-store types, usernames,
exact ACLs, free disk space, or runtime-internal mount identifiers. Root names
and descriptions MUST be runtime-curated labels safe to disclose to the napplet.
They MUST NOT reveal account names, device names, private folder names, storage
providers, organizations, or user-specific labels unless runtime policy
explicitly permits that disclosure.

**`pickFile(options?)`** — Asks the runtime to let the user select one file.
Picker operations are user-mediated authority requests. The napplet supplies
intent and advisory filters only. It MUST NOT supply host paths, file URLs,
OS-native paths, common-folder identifiers, volume names, mount identifiers, or
storage-provider identifiers. The runtime MAY show UI, require a user gesture,
deny the request, or return `cancelled` when the user cancels. A successful
`pickFile` result MUST contain exactly one file entry.

**`pickFiles(options?)`** — Asks the runtime to let the user select one or more
files. A successful result MUST contain at least one file entry. Cancellation
MUST be reported with `cancelled`, not as an empty successful result.

**`pickDirectory(options?)`** — Asks the runtime to let the user select one
directory. A successful result MUST contain exactly one directory entry.

**`pickSaveFile(options?)`** — Asks the runtime to let the user select or name one
file destination. A successful result MUST contain exactly one file entry. The
entry MAY be absent in the backing store until the napplet writes it, but the
returned virtual path MUST be authorized for the returned permissions.

Picker options are hints, not authority. Requested permissions, accepted MIME
types, accepted extensions, suggested names, and descriptions do not widen the
napplet's view. The runtime decides the actual selected objects and permissions
according to user choice and runtime policy. `accept` filters are advisory UI
filters only; runtimes and napplets MUST NOT treat them as content validation.
When `permissions` is absent, the requested permission intent defaults to
`read` for `pickFile` and `pickFiles`, `read` and `list` for `pickDirectory`, and
`write` and `create` for `pickSaveFile`.

Selected files or directories MAY originate outside the napplet's current visible
virtual filesystem. Before returning success, the runtime MUST expose each
selected object through a virtual absolute path in the napplet's visible
filesystem. The returned path MUST obey the NAP-FS path model. Subsequent
`stat`, `list`, `read`, `write`, `mkdir`, `remove`, `move`, `watch`, and
`unwatch` operations on that path use normal NAP-FS authorization.

Picker results MUST NOT expose host paths, file URLs, OS-native separators,
volume names, usernames, mount identifiers, backing-store identifiers, or storage
provider identifiers. They MUST NOT reveal whether the selected object was
already inside an existing visible root, mounted from outside it, copied into the
filesystem, proxied, or backed by another runtime mechanism.

Picker grants are not durable authority. A runtime MAY make a picked entry
visible only for the current session, MAY persist it across sessions, or MAY
revoke it at any time according to runtime policy. A napplet MUST NOT assume that
a picked path remains available after remount, reload, or runtime restart. It
MUST rediscover available roots with `info()` or ask the user to pick again.

**`stat(path)`** — Returns metadata for a visible file or directory. Metadata is
intentionally coarse. It omits host-specific identifiers such as inode, device,
owner, group, mode bits, symlink target, and backing-store metadata. `revision`,
when present, is an opaque token for write preconditions. Napplets MUST compare
it only for equality and MUST NOT infer ordering or content from it. A runtime
that exposes revisions MUST change the revision when file content changes.

**`list(path)`** — Lists direct children of a visible directory. Result ordering
is unspecified. Listing a file MUST fail with `not-a-directory`.

**`read(path, options?)`** — Reads bytes from a file. Range reads are mandatory.
`offset` defaults to `0`. `length` defaults to the runtime's maximum readable
chunk. `length` counts decoded bytes and MUST NOT exceed
`FsLimits.maxReadBytes`. The runtime MAY return fewer bytes than requested but,
for a non-zero `length`, MUST return at least one byte or set `eof: true`.
`eof: true` means no more bytes are available after this result. Reading a
directory MUST fail with `not-a-file`.

**`write(path, data, options?)`** — Writes bytes to a file. Range writes are
mandatory. `mode` defaults to `replace`. `replace` replaces the whole file and
MUST NOT carry `offset`. `append` appends to the file and MUST NOT carry
`offset`. `patch` writes at `offset` and MUST carry `offset`. The decoded length
of `data` MUST NOT exceed `FsLimits.maxWriteBytes`. A successful write MUST
commit all supplied bytes atomically and report `bytesWritten` equal to the
decoded data length. If the runtime cannot do so, it MUST fail without changing
the file. Concurrent appends MUST NOT interleave their bytes.

Writing an existing file requires `write` permission. Creating an absent file
requires `create` permission and authorization on its parent directory. Runtime
policy MAY require both permissions. `ifRevision`, when present, requires the
current file revision to match. `ifAbsent: true` requires the path to be absent.
`ifAbsent: false` has no effect. An unmet precondition MUST fail with `conflict`
without changing the file. A runtime that does not expose revisions MUST reject
`ifRevision` with `unsupported`.

**`mkdir(path, options?)`** — Creates a directory. If `recursive` is absent or
`false`, the parent directory MUST already exist. If `recursive: true`, the
runtime creates missing parent directories within the napplet's authorized view.
Recursive creation MUST NOT cross hidden parents, mount boundaries, or any path
outside the napplet's authorized view. Creation of every missing directory
requires `create` permission and authorization on its existing parent.

**`remove(path, recursive?)`** — Removes a file or directory. Removing a
non-empty directory without `recursive: true` MUST fail with `conflict`. Removal
requires `delete` permission on the target. Recursive removal MUST be authorized
for every visible entry it removes and MUST NOT remove hidden descendants outside
the napplet's authorized view. Before deleting any entry, the runtime MUST
determine the complete removal set from an authorized stable snapshot or use an
equivalent mechanism that reauthorizes each entry immediately before removal. It
MUST NOT remove an entry added or moved into the subtree unless that entry is
separately authorized. If the tree changes during the operation and safe
completion is uncertain, the runtime MUST fail with `conflict`.

**`move(fromPath, toPath)`** — Moves or renames a file or directory. The runtime
MUST atomically authorize the source entry, every descendant, the destination
parent, and replacement of any destination entry. It MUST revalidate that
authorization immediately before commit. Moving an entry MUST NOT escape the
napplet's authorized view. Cross-root or mount-boundary moves MUST fail with
`unsupported` or `policy-denied` unless runtime policy explicitly authorizes the
complete operation. The source requires `delete` permission. An absent
destination requires `create` permission and authorization on its parent.
Replacing an existing destination requires `delete` permission on that entry.
Runtime policy MAY require additional permissions. Policy attached to the moved
entry MUST be preserved or re-evaluated before it becomes visible at the
destination. A race or policy change MUST fail with `conflict` without moving the
entry. If the destination exists, the runtime MAY reject with `already-exists` or
`conflict` according to runtime policy.

**`watch(path, options?)`** — Starts an advisory watch and returns `watchId`.
`recursive` defaults to `false`. If `recursive: false`, directory watches cover
the directory entry and direct children. If `recursive: true`, directory watches
cover the directory and visible descendants. `recursive` has no effect for file
watches. A runtime MAY reject recursive watches with `unsupported` or
`policy-denied`. Watch permission is independent from read and list permission.
Recursive watch requires explicit runtime policy.

Watch events are invalidation signals. They MAY be coalesced, duplicated,
reordered, dropped, or reported as `unknown`. Napplets SHOULD call `stat`,
`list`, or `read` after receiving `fs.changed`. The runtime MUST filter invisible
entries before generating, coalescing, or dropping events. It SHOULD batch and
rate-limit events to reduce timing disclosure.

**`unwatch(watchId)`** — Stops a watch. Unknown `watchId` values MAY be treated
as successful no-ops. The runtime MUST scope the identifier to the requesting
napplet before lookup. An identifier owned by another napplet MUST be
indistinguishable from an unknown identifier.

## Wire Protocol

`fs.*` messages use the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `fs.info` | napplet -> runtime | `id` |
| `fs.info.result` | runtime -> napplet | `id`, `info?`, `error?` |
| `fs.pickFile` | napplet -> runtime | `id`, `options?` |
| `fs.pickFile.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `fs.pickFiles` | napplet -> runtime | `id`, `options?` |
| `fs.pickFiles.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `fs.pickDirectory` | napplet -> runtime | `id`, `options?` |
| `fs.pickDirectory.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `fs.pickSaveFile` | napplet -> runtime | `id`, `options?` |
| `fs.pickSaveFile.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `fs.stat` | napplet -> runtime | `id`, `path` |
| `fs.stat.result` | runtime -> napplet | `id`, `metadata?`, `error?` |
| `fs.list` | napplet -> runtime | `id`, `path` |
| `fs.list.result` | runtime -> napplet | `id`, `entries?`, `error?` |
| `fs.read` | napplet -> runtime | `id`, `path`, `options?` |
| `fs.read.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `fs.write` | napplet -> runtime | `id`, `path`, `data`, `options?` |
| `fs.write.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `fs.mkdir` | napplet -> runtime | `id`, `path`, `options?` |
| `fs.mkdir.result` | runtime -> napplet | `id`, `error?` |
| `fs.remove` | napplet -> runtime | `id`, `path`, `recursive?` |
| `fs.remove.result` | runtime -> napplet | `id`, `error?` |
| `fs.move` | napplet -> runtime | `id`, `fromPath`, `toPath` |
| `fs.move.result` | runtime -> napplet | `id`, `error?` |
| `fs.watch` | napplet -> runtime | `id`, `path`, `options?` |
| `fs.watch.result` | runtime -> napplet | `id`, `watchId?`, `error?` |
| `fs.changed` | runtime -> napplet | `change` |
| `fs.unwatch` | napplet -> runtime | `id`, `watchId` |
| `fs.unwatch.result` | runtime -> napplet | `id`, `error?` |

Key design notes:

- Request/result pairs use `id` for correlation.
- `fs.changed` is runtime-pushed and has no `id`.
- `watchId` is generated by the runtime and scoped to the requesting napplet.
- Every request receives one result message with the same `id`.
- A successful result MUST omit `error` and include every operation-specific
  success field shown in the API surface. A failed result MUST include `error`
  and MUST omit all operation-specific success fields.
- A runtime MAY return `not-found` instead of `permission-denied` to avoid
  revealing hidden paths. It SHOULD normalize error detail and timing for hidden
  paths.

### Byte Encoding

`fs.write.data` and `FsReadResult.data` carry file bytes as base64 text in the
JSON wire envelope.

- The encoding MUST use the RFC 4648 standard base64 alphabet with padding.
- Empty bytes MUST be encoded as the empty string.
- Runtimes MUST reject malformed base64, omitted padding, whitespace, base64url,
  and non-canonical encodings with `invalid-data`.
- `bytesRead`, `bytesWritten`, `FsLimits.maxReadBytes`,
  `FsLimits.maxWriteBytes`, and `FsLimits.maxInFlightBytes` count decoded bytes,
  not the encoded JSON string length.
- Runtimes MAY separately enforce encoded envelope size limits. Those limits are
  runtime policy and do not change the `FsLimits` byte counts.

### Examples

**Info:**
```
-> { "type": "fs.info", "id": "i1" }
<- {
     "type": "fs.info.result",
     "id": "i1",
     "info": {
       "roots": [
         { "path": "/shared", "name": "Shared files", "permissions": ["read", "list", "write", "create", "delete", "watch"] },
         { "path": "/media", "name": "Media", "permissions": ["read", "list", "watch"] }
       ],
       "limits": { "maxReadBytes": 1048576, "maxWriteBytes": 1048576, "maxWatchCount": 16 }
     }
   }
```

**Pick text file:**
```
-> { "type": "fs.pickFile", "id": "p1", "options": { "permissions": ["read"], "accept": [{ "mime": "text/plain" }, { "extension": ".md" }] } }
<- { "type": "fs.pickFile.result", "id": "p1", "result": { "entries": [{ "path": "/picked/report.md", "kind": "file", "name": "report.md", "permissions": ["read"], "size": 1200 }] } }
```

**Pick directory:**
```
-> { "type": "fs.pickDirectory", "id": "p2", "options": { "permissions": ["read", "list", "watch"], "description": "Choose media folder" } }
<- { "type": "fs.pickDirectory.result", "id": "p2", "result": { "entries": [{ "path": "/picked/media", "kind": "directory", "name": "media", "permissions": ["read", "list"] }] } }
```

**Pick save destination:**
```
-> { "type": "fs.pickSaveFile", "id": "p3", "options": { "permissions": ["write", "create"], "suggestedName": "export.json", "accept": [{ "mime": "application/json" }] } }
<- { "type": "fs.pickSaveFile.result", "id": "p3", "result": { "entries": [{ "path": "/picked/export.json", "kind": "file", "name": "export.json", "permissions": ["write", "create"] }] } }
```

**Pick cancelled:**
```
-> { "type": "fs.pickFile", "id": "p4" }
<- { "type": "fs.pickFile.result", "id": "p4", "error": "cancelled" }
```

**Range read:**
```
-> { "type": "fs.read", "id": "r1", "path": "/shared/video.bin", "options": { "offset": 1048576, "length": 65536 } }
<- { "type": "fs.read.result", "id": "r1", "result": { "data": "SGVsbG8gd29ybGQh", "offset": 1048576, "bytesRead": 12, "eof": false, "size": 9000000 } }
```

**Replace file:**
```
-> { "type": "fs.write", "id": "w1", "path": "/shared/note.txt", "data": "SGVsbG8gd29ybGQh", "options": { "mode": "replace" } }
<- { "type": "fs.write.result", "id": "w1", "result": { "bytesWritten": 12, "size": 12 } }
```

**Patch file:**
```
-> { "type": "fs.write", "id": "w2", "path": "/shared/db.bin", "data": "AAECAw==", "options": { "mode": "patch", "offset": 4096 } }
<- { "type": "fs.write.result", "id": "w2", "result": { "bytesWritten": 4, "size": 8192 } }
```

**Recursive mkdir:**
```
-> { "type": "fs.mkdir", "id": "m1", "path": "/shared/projects/new", "options": { "recursive": true } }
<- { "type": "fs.mkdir.result", "id": "m1" }
```

**List directory:**
```
-> { "type": "fs.list", "id": "l1", "path": "/shared" }
<- { "type": "fs.list.result", "id": "l1", "entries": [{ "name": "note.txt", "path": "/shared/note.txt", "kind": "file", "size": 12 }] }
```

**Recursive watch:**
```
-> { "type": "fs.watch", "id": "x1", "path": "/shared", "options": { "recursive": true } }
<- { "type": "fs.watch.result", "id": "x1", "watchId": "watch-1" }
<- { "type": "fs.changed", "change": { "watchId": "watch-1", "path": "/shared/note.txt", "kind": "modified" } }
```

**Permission denied:**
```
-> { "type": "fs.read", "id": "r2", "path": "/hidden/private.txt" }
<- { "type": "fs.read.result", "id": "r2", "error": "not-found" }
```

## Runtime Behavior

- The runtime MUST authorize every request against the napplet identity assigned
  at creation time by the projection.
- The runtime MUST respond to every request with a result message carrying the
  same `id`.
- The runtime MUST NOT trust any path, permission, root, watch identifier, picker
  option, or picker filter supplied by the napplet beyond using it as an
  untrusted request parameter.
- The runtime MUST enforce the napplet's view on `pickFile`, `pickFiles`,
  `pickDirectory`, `pickSaveFile`, `stat`, `list`, `read`, `write`, `mkdir`,
  `remove`, `move`, `watch`, and `unwatch`.
- The runtime MUST NOT expose host absolute paths or backing-store identifiers.
- The runtime MUST mediate picker operations through runtime policy and, when
  required, user choice.
- The runtime MUST expose picked entries only as virtual paths in the napplet's
  visible filesystem.
- The runtime MAY require a user gesture before showing picker UI.
- The runtime MUST enforce `FsLimits.maxReadBytes` and `FsLimits.maxWriteBytes`
  as decoded byte limits.
- The runtime SHOULD enforce per-napplet limits on active watches, in-flight
  requests, aggregate in-flight bytes, recursive work, and operation duration.
- The runtime MAY reject work beyond those limits with `too-large`,
  `quota-exceeded`, or `policy-denied`.
- The runtime MAY use any persistence, sync, conflict-resolution, or backing-store
  strategy as long as the napplet-observed contract is preserved.
- The runtime MAY report stale metadata when its backing store is eventually
  consistent.
- The runtime MAY revoke permissions or picked access during a session.
  Subsequent operations MUST reflect the new policy.

## Security Considerations

- Backing storage can be shared across napplets. Shared mutation is destructive
  authority, not a consequence of visibility. The runtime MUST enforce each
  napplet's mutable scope and explicit policy for destructive shared operations.
- Host paths are sensitive. Exposing them leaks usernames, device layout, mounted
  volumes, and runtime internals. NAP-FS uses virtual paths only.
- Picker results are subject to the same host-path secrecy rules. They MUST NOT
  reveal host paths, volume names, storage providers, or backing-store details.
- Path traversal is a primary attack surface. Runtimes MUST reject `.` and `..`,
  normalize before authorization, and prevent mount or symlink escape.
- Recursive watch can leak hidden descendants if implemented naively. Runtimes
  MUST derive event delivery only from activity visible to the watching napplet.
  Event payloads, filtering, coalescing, and drop behavior MUST NOT reveal hidden
  entries. Runtimes SHOULD batch and rate-limit events to reduce timing channels.
- Large reads, writes, recursive operations, and watches are denial-of-service
  surfaces. Runtimes SHOULD enforce quotas, chunk limits, aggregate byte limits,
  request limits, watch limits, and operation timeouts.
- Picker UI can be abused for prompt spam and focus disruption. Runtimes SHOULD
  rate-limit picker requests and MAY require user activation.
- Picker filters are advisory. Runtimes and napplets MUST NOT infer safe content
  from MIME types, extensions, suggested names, or picker descriptions.
- Picker results reveal user choices and may reveal names, sizes, timestamps, and
  kinds. Runtimes SHOULD minimize returned metadata and SHOULD avoid denial
  detail that reveals whether a hidden or host object exists.
- `info()` is not an authorization token. Napplets MUST handle operation failures
  even when `info()` advertised a matching permission.
- File and directory names, sizes, timestamps, kinds, and change timing are
  sensitive even when file bytes are unavailable. Runtimes SHOULD expose only
  policy-authorized metadata and SHOULD omit or coarsen timestamps and sizes when
  exact values are unnecessary.
- Concurrent writes can conflict. Napplets SHOULD use `revision` and write
  preconditions when lost updates matter. Without a precondition, the runtime MAY
  apply runtime-defined ordering. Metadata and watch events can be stale.
- An `unknown` entry kind grants no implied operation. The runtime MUST reject
  `read` or `list` unless it can enforce the corresponding safe file or directory
  semantics.
- File bytes are visible to the runtime and cross the projection boundary.
  Napplets SHOULD NOT treat NAP-FS as confidential storage from the runtime.

## Implementations

- (none yet)

## Changelog

- `e63e73b` - Introduced NAP-FS for shell-mediated virtual filesystem access.
- `pending` - Hardened shared mutation, concurrency, metadata, and watch policy.
- `pending` - Added user-mediated picker operations that expose selected files and
  directories as virtual paths with runtime-defined persistence.
- `pending` - Defined base64 byte encoding, decoded-byte limits, schema tables,
  and `invalid-data` for malformed byte payloads.

- `3ad611a` - Adopted injected-domain availability and linked the current upstream web binding.
