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

The filesystem can be global across napplets. The runtime still controls each
napplet's view and permissions. Every operation is authorized against the
runtime-bound napplet identity; no request payload can widen that view.

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
- A runtime MUST normalize and validate paths before authorization.
- A runtime MUST reject any path that escapes the napplet's authorized view.

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
}

FsMetadata = {
  path: tstr,
  kind: FsEntryKind,
  ? size: uint,
  ? modifiedAt: uint,
  ? createdAt: uint,
  ? permissions: [* FsPermission],
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
exact ACLs, free disk space, or runtime-internal mount identifiers.

**`stat(path)`** — Returns metadata for a visible file or directory. Metadata is
intentionally coarse. It omits host-specific identifiers such as inode, device,
owner, group, mode bits, symlink target, and backing-store metadata.

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
NOT exceed `FsLimits.maxWriteBytes`. The runtime MAY write fewer bytes than
supplied and report `bytesWritten`.

**`mkdir(path, options?)`** — Creates a directory. If `recursive` is absent or
`false`, the parent directory MUST already exist. If `recursive: true`, the
runtime creates missing parent directories within the napplet's authorized view.
Recursive creation MUST NOT cross hidden parents, mount boundaries, or any path
outside the napplet's authorized view.

**`remove(path, recursive?)`** — Removes a file or directory. Removing a
non-empty directory without `recursive: true` MUST fail with `conflict`.
Recursive removal MUST be authorized for every visible entry it removes and MUST
NOT remove hidden descendants outside the napplet's authorized view.

**`move(fromPath, toPath)`** — Moves or renames a file or directory. The runtime
MUST authorize both source and destination. Moving an entry MUST NOT escape the
napplet's authorized view. If the destination exists, the runtime MAY reject with
`already-exists` or `conflict` according to runtime policy.

**`watch(path, options?)`** — Starts an advisory watch and returns `watchId`.
`recursive` defaults to `false`. If `recursive: false`, directory watches cover
the directory entry and direct children. If `recursive: true`, directory watches
cover the directory and visible descendants. `recursive` has no effect for file
watches. A runtime MAY reject recursive watches with `unsupported` or
`policy-denied`.

Watch events are invalidation signals. They MAY be coalesced, duplicated,
reordered, dropped, or reported as `unknown`. Napplets SHOULD call `stat`,
`list`, or `read` after receiving `fs.changed`.

**`unwatch(watchId)`** — Stops a watch. Unknown `watchId` values MAY be treated
as successful no-ops.

## Wire Protocol

`fs.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `fs.info` | napplet -> runtime | `id` |
| `fs.info.result` | runtime -> napplet | `id`, `info`, `error?` |
| `fs.stat` | napplet -> runtime | `id`, `path` |
| `fs.stat.result` | runtime -> napplet | `id`, `metadata?`, `error?` |
| `fs.list` | napplet -> runtime | `id`, `path` |
| `fs.list.result` | runtime -> napplet | `id`, `entries`, `error?` |
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
- Result messages MAY include `error`. When `error` is present,
  operation-specific result fields SHOULD be absent.
- A runtime MAY return `not-found` instead of `permission-denied` to avoid
  revealing hidden paths.

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
- The runtime MUST NOT trust any path, permission, root, or watch identifier
  supplied by the napplet beyond using it as an untrusted request parameter.
- The runtime MUST enforce the napplet's view on `stat`, `list`, `read`, `write`,
  `mkdir`, `remove`, `move`, `watch`, and `unwatch`.
- The runtime MUST NOT expose host absolute paths or backing-store identifiers.
- The runtime MUST enforce `FsLimits.maxReadBytes` and `FsLimits.maxWriteBytes`.
- The runtime SHOULD enforce per-napplet limits on active watches.
- The runtime MAY use any persistence, sync, conflict-resolution, or backing-store
  strategy as long as the napplet-observed contract is preserved.
- The runtime MAY report stale metadata when its backing store is eventually
  consistent.
- The runtime MAY revoke permissions during a session. Subsequent operations MUST
  reflect the new policy.

## Security Considerations

- The filesystem can be shared globally across napplets. The runtime MUST enforce
  per-napplet views so one napplet cannot read, write, list, remove, move, or
  watch entries outside its authorized view.
- Host paths are sensitive. Exposing them leaks usernames, device layout, mounted
  volumes, and runtime internals. NAP-FS uses virtual paths only.
- Path traversal is a primary attack surface. Runtimes MUST reject `.` and `..`,
  normalize before authorization, and prevent mount or symlink escape.
- Recursive watch can leak hidden descendants if implemented naively. Runtimes
  MUST emit only paths visible to the watching napplet and MUST NOT reveal hidden
  paths through event timing or event payloads where avoidable.
- Large reads, writes, recursive operations, and watches are denial-of-service
  surfaces. Runtimes SHOULD enforce quotas, chunk limits, watch limits, and
  operation timeouts.
- `info()` is not an authorization token. Napplets MUST handle operation failures
  even when `info()` advertised a matching permission.
- Concurrent writes can conflict. Runtimes MAY reject conflicting writes with
  `conflict` or apply runtime-defined ordering. Napplets SHOULD treat metadata and
  watch events as potentially stale.
- File bytes are visible to the runtime and cross the projection boundary.
  Napplets SHOULD NOT treat NAP-FS as confidential storage from the runtime.

## Implementations

- (none yet)

## Changelog

- `pending` - Introduced NAP-FS for shell-mediated virtual filesystem access.
