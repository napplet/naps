NAP-FS
======

Virtual Filesystem Access
-------------------------

`draft`

**NAP ID:** NAP-FS
**Domain:** `fs`
**Web binding (NIP-5D):** `window.napplet.fs` · `shell.supports("fs")`

## Description

NAP-FS provides napplets with shell-mediated access to a runtime-owned virtual
filesystem. The filesystem may be backed by local files, origin-private storage,
remote storage, a database, or any other runtime implementation. A napplet sees
only virtual paths, directory entries, file metadata, byte reads and writes, and
advisory change events.

Backing storage can be shared across napplets. Each napplet still sees a
policy-bound view with its own permissions. Every operation is authorized
against the runtime-bound napplet identity; no request payload can widen that
view.

NAP-FS defines the filesystem seam. It does not define host paths, mount sources,
picker UI, sync backends, OS permissions, or storage layout.

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
| `stat` | `path` (`tstr`) | `FsMetadata` | `fs.stat` / `fs.stat.result` |
| `list` | `path` (`tstr`) | list of `FsDirectoryEntry` | `fs.list` / `fs.list.result` |
| `read` | `path` (`tstr`), optional `options` (`FsReadOptions`) | `FsReadResult` | `fs.read` / `fs.read.result` |
| `write` | `path` (`tstr`), `data` (`bstr`), optional `options` (`FsWriteOptions`) | `FsWriteResult` | `fs.write` / `fs.write.result` |
| `mkdir` | `path` (`tstr`), optional `options` (`FsMkdirOptions`) | none | `fs.mkdir` / `fs.mkdir.result` |
| `remove` | `path` (`tstr`), optional `recursive` (`bool`) | none | `fs.remove` / `fs.remove.result` |
| `move` | `fromPath` (`tstr`), `toPath` (`tstr`) | none | `fs.move` / `fs.move.result` |
| `watch` | `path` (`tstr`), optional `options` (`FsWatchOptions`) | `watchId` (`tstr`) | `fs.watch` / `fs.watch.result` |
| `unwatch` | `watchId` (`tstr`) | none | `fs.unwatch` / `fs.unwatch.result` |

### Schemas

```cddl
FsPermission = "read" / "write" / "create" / "delete" / "list" / "watch"
FsEntryKind = "file" / "directory" / "unknown"
FsWriteMode = "replace" / "append" / "patch"
FsChangeKind = "created" / "modified" / "deleted" / "moved" / "unknown"
FsError = "not-found" / "already-exists" / "not-a-file" / "not-a-directory" /
  "invalid-path" / "permission-denied" / "policy-denied" / "quota-exceeded" /
  "too-large" / "unsupported" / "conflict" / "io-error"

FsInfo = {
  roots: [* FsRoot],
  limits: FsLimits,
}

FsRoot = {
  path: tstr,
  name: tstr,
  permissions: [* FsPermission],
  ? description: tstr,
}

FsLimits = {
  maxReadBytes: uint,
  maxWriteBytes: uint,
  ? maxWatchCount: uint,
  ? maxInFlightRequests: uint,
  ? maxInFlightBytes: uint,
}

FsMetadata = {
  path: tstr,
  kind: FsEntryKind,
  ? size: uint,
  ? modifiedAt: uint,
  ? createdAt: uint,
  ? permissions: [* FsPermission],
  ? revision: tstr,
}

FsDirectoryEntry = {
  name: tstr,
  path: tstr,
  kind: FsEntryKind,
  ? size: uint,
  ? modifiedAt: uint,
}

FsReadOptions = {
  ? offset: uint,
  ? length: uint,
}

FsReadResult = {
  data: bstr,
  offset: uint,
  bytesRead: uint,
  eof: bool,
  ? size: uint,
}

FsWriteOptions = {
  ? mode: FsWriteMode,
  ? offset: uint,
  ? ifRevision: tstr,
  ? ifAbsent: bool,
}

FsWriteResult = {
  bytesWritten: uint,
  ? size: uint,
}

FsMkdirOptions = {
  ? recursive: bool,
}

FsWatchOptions = {
  ? recursive: bool,
}

FsChange = {
  watchId: tstr,
  path: tstr,
  kind: FsChangeKind,
  ? fromPath: tstr,
}
```

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
chunk. `length` MUST NOT exceed `FsLimits.maxReadBytes`. The runtime MAY return
fewer bytes than requested. `eof: true` means no more bytes are available after
this result. Reading a directory MUST fail with `not-a-file`.

**`write(path, data, options?)`** — Writes bytes to a file. Range writes are
mandatory. `mode` defaults to `replace`. `replace` replaces the whole file and
MUST NOT carry `offset`. `append` appends to the file and MUST NOT carry
`offset`. `patch` writes at `offset` and MUST carry `offset`. `data` length MUST
NOT exceed `FsLimits.maxWriteBytes`. A successful write MUST commit all supplied
bytes atomically and report `bytesWritten` equal to the data length. If the
runtime cannot do so, it MUST fail without changing the file. Concurrent appends
MUST NOT interleave their bytes.

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

`fs.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `fs.info` | napplet -> runtime | `id` |
| `fs.info.result` | runtime -> napplet | `id`, `info?`, `error?` |
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

**Range read:**
```
-> { "type": "fs.read", "id": "r1", "path": "/shared/video.bin", "options": { "offset": 1048576, "length": 65536 } }
<- { "type": "fs.read.result", "id": "r1", "result": { "data": <bytes>, "offset": 1048576, "bytesRead": 65536, "eof": false, "size": 9000000 } }
```

**Replace file:**
```
-> { "type": "fs.write", "id": "w1", "path": "/shared/note.txt", "data": <bytes>, "options": { "mode": "replace" } }
<- { "type": "fs.write.result", "id": "w1", "result": { "bytesWritten": 12, "size": 12 } }
```

**Patch file:**
```
-> { "type": "fs.write", "id": "w2", "path": "/shared/db.bin", "data": <bytes>, "options": { "mode": "patch", "offset": 4096 } }
<- { "type": "fs.write.result", "id": "w2", "result": { "bytesWritten": 512, "size": 8192 } }
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
- The runtime MUST NOT trust any path, permission, root, or watch identifier
  supplied by the napplet beyond using it as an untrusted request parameter.
- The runtime MUST enforce the napplet's view on `stat`, `list`, `read`, `write`,
  `mkdir`, `remove`, `move`, `watch`, and `unwatch`.
- The runtime MUST NOT expose host absolute paths or backing-store identifiers.
- The runtime MUST enforce `FsLimits.maxReadBytes` and `FsLimits.maxWriteBytes`.
- The runtime SHOULD enforce per-napplet limits on active watches, in-flight
  requests, aggregate in-flight bytes, recursive work, and operation duration.
- The runtime MAY reject work beyond those limits with `too-large`,
  `quota-exceeded`, or `policy-denied`.
- The runtime MAY use any persistence, sync, conflict-resolution, or backing-store
  strategy as long as the napplet-observed contract is preserved.
- The runtime MAY report stale metadata when its backing store is eventually
  consistent.
- The runtime MAY revoke permissions during a session. Subsequent operations MUST
  reflect the new policy.

## Security Considerations

- Backing storage can be shared across napplets. Shared mutation is destructive
  authority, not a consequence of visibility. The runtime MUST enforce each
  napplet's mutable scope and explicit policy for destructive shared operations.
- Host paths are sensitive. Exposing them leaks usernames, device layout, mounted
  volumes, and runtime internals. NAP-FS uses virtual paths only.
- Path traversal is a primary attack surface. Runtimes MUST reject `.` and `..`,
  normalize before authorization, and prevent mount or symlink escape.
- Recursive watch can leak hidden descendants if implemented naively. Runtimes
  MUST derive event delivery only from activity visible to the watching napplet.
  Event payloads, filtering, coalescing, and drop behavior MUST NOT reveal hidden
  entries. Runtimes SHOULD batch and rate-limit events to reduce timing channels.
- Large reads, writes, recursive operations, and watches are denial-of-service
  surfaces. Runtimes SHOULD enforce quotas, chunk limits, aggregate byte limits,
  request limits, watch limits, and operation timeouts.
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
