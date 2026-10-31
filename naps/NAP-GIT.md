NAP-GIT
========

Runtime-Mediated Git Repositories
--------------------------------

`draft`

**NAP ID:** NAP-GIT
**Domain:** `git`
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.git`; domain presence signals availability. This is projection guidance.

## Description

NAP-GIT provides repository access through Git objects, references, and explicit
remote operations. The napplet designs browsing and editing workflows. The shell
mediates repository selection and authority. The runtime executes Git operations,
holds credentials, signs when required, and selects storage and network backends.
The same contract supports native Git libraries or executables and
[ngit](https://github.com/DanConwayDev/ngit-cli), which combines Git object
storage with Nostr repository state and collaboration.

### Boundary

A Git object is immutable content identified by its hash: a blob (file bytes),
tree (directory entries), commit (a tree and its history), or annotated tag.
A reference is a mutable name pointing to an object. These concepts are common
to both implementations; the runtime's implementation choice is not a projection.

| Concern | Owner |
|---------|-------|
| Repository access, object verification, storage, retention | Runtime |
| Local reference transactions, remote synchronization | Runtime |
| Credentials, signer selection, Nostr event construction and verification | Runtime, under shell mediation |
| Editors, comparisons, history presentation, conflict-resolution choices | Napplet |
| Repository, account, publication destinations, permission decisions | User, through the shell |
| Host paths, processes, sockets, Git configuration and command execution | Runtime internals |

Creating a commit, moving a local reference, updating a remote reference, and
submitting a proposal are separate effects. The runtime MUST NOT substitute one
for another. In particular, ngit's proposal-producing branch conventions MUST
NOT turn `push` into `propose`.

This draft exposes committed objects. Working directories, staging indexes,
checkout, merge/rebase execution, arbitrary commands, issues, reviews, repository
announcement editing, and forge administration are outside this surface.
Napplets can compare trees and construct an edited commit without host filesystem
access. Filesystem and generic Nostr access remain independently governed by
their owning domains; no other NAP domain is required by this contract.

## API Surface

Every runtime exposing `git` MUST implement `open`, `close`, `refs`, `get`, and
`read`. Additional operations are reported per repository in `operations`.
An absent operation MUST return `unsupported` when its defined request is received.
An advertised operation can still be denied by current policy.

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `open` | optional `source` (text) | `GitRepository` | `git.open` / `git.open.result` |
| `close` | `repository` (text) | `null` | `git.close` / `git.close.result` |
| `refs` | `repository` (text), optional `cursor` (text), optional `limit` (integer) | `GitRefPage` | `git.refs` / `git.refs.result` |
| `get` | `repository` (text), `oid` (text), optional `cursor` (text), optional `limit` (integer) | `GitObject` | `git.get` / `git.get.result` |
| `read` | `repository` (text), `oid` (text), `offset` (integer), `length` (integer) | `GitBytes` | `git.read` / `git.read.result` |
| `createCommit` | `repository` (text), `request` (`GitCommitRequest`) | `GitCreatedCommit` | `git.createCommit` / `git.createCommit.result` |
| `updateRef` | `repository` (text), `request` (`GitRefUpdate`) | `GitRefUpdate` | `git.updateRef` / `git.updateRef.result` |
| `fetch` | `repository` (text), `remote` (text), `ref` (text) | `GitFetchedRef` | `git.fetch` / `git.fetch.result` |
| `push` | `repository` (text), `request` (`GitPushRequest`) | `GitPublication` | `git.push` / `git.push.result` |
| `propose` | `repository` (text), `request` (`GitProposalRequest`) | `GitPublication` | `git.propose` / `git.propose.result` |

### Common values

| Value | Definition |
|-------|------------|
| Object ID (`oid`) | Full lowercase hexadecimal Git object ID: 40 characters for `sha1`, 64 for `sha256`, matching the opened repository's `objectFormat`. Abbreviations, revision expressions, and all-zero IDs are invalid. A Git ID is distinct from a Nostr event ID. |
| Reference name (`ref`) | UTF-8 text containing a full `refs/heads/…` or `refs/tags/…` name accepted by [Git's reference-name rules](https://git-scm.com/docs/git-check-ref-format). No refspecs, wildcards, revision expressions, or symbolic references. Branch mutations require `refs/heads/…`. |
| Bytes | Text containing canonical padded standard base64 under [RFC 4648](https://www.rfc-editor.org/rfc/rfc4648#section-4). Empty bytes are `""`. Whitespace, base64url, missing required padding, and nonzero pad bits are invalid. |
| Path | Bytes encoding a repository-relative path. `/` byte separates segments. Empty, absolute, trailing-slash, empty-segment, NUL, `.` and `..` paths are invalid. No Unicode normalization, case folding, URL decoding, or host path interpretation. |
| Name | Bytes encoding one nonempty tree-entry name; `/` and NUL are invalid. Names need not be UTF-8. |
| Cursor | Nonempty opaque text scoped to the endpoint, repository handle, operation, and original arguments. Not authority. |

All integers MUST be exact and within -9007199254740991 through
9007199254740991; byte counts and offsets MUST be nonnegative. List limits MUST
be positive. Omitted optional fields use the defaults stated below. `null` is
valid only where explicitly allowed. Empty lists mean no entries.

### Repository and reference schemas

`GitRepository`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `repository` | yes | text | Nonempty opaque handle for this authenticated endpoint. |
| `objectFormat` | yes | `"sha1"` or `"sha256"` | Applies to every Git object ID on this handle. |
| `operations` | yes | list of text | Supported operation names from the API table. No duplicates. |
| `remotes` | yes | list of `GitRemote` | Only remotes the caller may discover. Empty for local-only access. |

`GitRemote`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `remote` | yes | text | Nonempty opaque identifier scoped to this repository handle. |
| `operations` | yes | list of text | Supported subset of `"fetch"`, `"push"`, `"propose"`; not a grant. |

`GitRef`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ref` | yes | text | Full reference name. |
| `oid` | yes | text | Direct target; an annotated tag points to its tag object, not its peeled commit. |

`GitRefPage`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `entries` | yes | list of `GitRef` | Local references, sorted by unsigned UTF-8 bytes of `ref`. |
| `next` | yes | text or `null` | Cursor for the next page; `null` means complete. |

`GitRefUpdate`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ref` | yes | text | Local branch reference. |
| `expected` | yes | text or `null` | Current direct object ID; `null` requires absence. |
| `oid` | yes | text or `null` | New commit ID; `null` deletes. Both `expected` and `oid` cannot be `null`. |

`GitFetchedRef`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ref` | yes | text | Requested remote reference. |
| `oid` | yes | text | Observed direct target, now available locally. |

### Object schemas

`GitObject`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `oid` | yes | text | Requested object ID. |
| `kind` | yes | `"blob"`, `"tree"`, `"commit"`, or `"tag"` | Selects the fields below. |
| `size` | yes | integer | Complete uncompressed Git object body size in bytes. |
| `entries` | conditional | list of `GitTreeEntry` | Required only for a tree; one page of immediate children. |
| `next` | conditional | text or `null` | Required only for a tree. |
| `commit` | conditional | `GitCommit` | Required only for a commit. |
| `target` | conditional | text | Required only for a tag; target object ID. |

Fields for other kinds MUST be absent. Blobs contain no byte payload in `get`;
use `read`. An annotated tag's content and signature remain available through
`read`; `target` does not assert signature validity.

`GitTreeEntry`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `name` | yes | text | Name bytes. |
| `mode` | yes | `"040000"`, `"100644"`, `"100755"`, `"120000"`, or `"160000"` | Tree, regular blob, executable blob, symlink blob, or gitlink (submodule commit). |
| `oid` | yes | text | Child object ID. A gitlink's object can be absent from this repository. |

`GitCommit`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `tree` | yes | text | Root tree object ID. |
| `parents` | yes | list of text | Parent commit IDs in stored order; empty for a root commit. |
| `author` | yes | text | Raw author-header value, encoded as bytes. Attribution, not authenticated identity. |
| `committer` | yes | text | Raw committer-header value, encoded as bytes. |
| `message` | yes | text | Exact message bytes. No character encoding is assumed. |

`GitBytes`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `data` | yes | text | Requested bytes from the uncompressed object body, without Git's object header. |
| `offset` | yes | integer | Requested offset. |
| `size` | yes | integer | Complete body size. |
| `eof` | yes | boolean | True when offset plus decoded data length reaches or exceeds `size`. |

`read` works for all four object kinds. Its raw bytes preserve headers,
signatures, non-UTF-8 messages, and fields omitted from the structured view.
The runtime MUST verify the object's Git hash, including its type/size header,
before returning any part of it. Replacement objects and grafted history MUST
NOT alter the requested object's view.

### Commit schemas

`GitCommitRequest`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `parent` | yes | text or `null` | Parent commit, or `null` for a root commit with an initially empty tree. |
| `changes` | yes | list of `GitChange` | Nonempty, distinct paths; applied to the parent's tree. |
| `message` | yes | text | Nonempty UTF-8 commit message; NUL is invalid. |

`GitChange`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `path` | yes | text | Path bytes. |
| `action` | yes | `"put"` or `"delete"` | Delete removes one existing non-tree entry. |
| `mode` | conditional | `"100644"` or `"100755"` | Required only for `put`. |
| `data` | conditional | text | File bytes; required only for `put`, including empty files. |

`GitCreatedCommit`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `oid` | yes | text | Created commit object ID. |

### Publication schemas

`GitPushRequest`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `remote` | yes | text | Remote identifier from `GitRepository.remotes`. |
| `ref` | yes | text | Exact destination branch. |
| `oid` | yes | text | Locally available commit to publish. |

`GitProposalRequest`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `remote` | yes | text | Remote identifier supporting `propose`. |
| `target` | yes | text | Intended destination branch. |
| `base` | yes | text | Locally available commit; ancestor of `head`. |
| `head` | yes | text | Locally available commit, distinct from `base`. |
| `title` | yes | text | Nonempty plain text. |
| `description` | yes | text | Plain text; may be empty. |

`GitPublication`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `outcome` | yes | `"complete"`, `"failed"`, `"partial"`, or `"unknown"` | Derived from receipts under the rules below. |
| `receipts` | yes | list of `GitReceipt` | Nonempty; every required publication effect is represented. |

`GitReceipt`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `effect` | yes | `"objects"`, `"ref"`, or `"proposal"` | Object availability, reference publication, or proposal publication. |
| `destination` | yes | text | Nonempty opaque destination identifier, stable within this repository handle. |
| `status` | yes | `"accepted"`, `"rejected"`, or `"unknown"` | Positive acknowledgement, known no effect at this destination, or uncertain effect. |
| `oid` | yes | text | Requested push commit or proposal head. |
| `ref` | conditional | text | Required only for `ref` effects; exact destination branch. |
| `locator` | no | text | Credential-free absolute URI identifying the published artifact, when available. Informational; conveys no authority. |

## Operation Semantics

### Repository access and lifetime

`open` without `source` asks the shell to let the user choose a repository or
create an empty runtime-managed repository. With `source`, it requests access to
that repository location. The shell MAY offer a choice, require consent, use an
existing authorized store, or deny the request. Local host paths and `file:`
URLs are invalid. Supported remote URL schemes are runtime policy; unsupported
ones return `unsupported`. `nostr:` locations use the upstream
[NIP-34 clone URL format](https://github.com/nostr-protocol/nips/blob/master/34.md#nostr-clone-url-format).

Success means the runtime established repository access and can service core
reads. It does not promise that every remote object is cached. `open` MAY fetch
under shell policy, but MUST NOT publish or alter an existing repository's refs.
The runtime MUST NOT equate matching URLs, root commits, or object hashes with
identical repository authority. The returned remote identifiers bind the approved
logical repository and its authority, not an arbitrary Git server named by data.

Handles, remotes, and cursors MUST be scoped to the authenticated endpoint.
Copying a handle to another endpoint grants nothing. Policy MAY revoke access.
Unknown, foreign, closed, or revoked handles return `invalid-handle` without
revealing whether another endpoint owns them.

`close` prevents new operations on the handle and waits for already-started
operations to settle before returning `null`. Repeated close on the caller's
already-closed handle succeeds. Close releases access and cursors; it does not
delete repositories, undo commits, or retract publications. Endpoint teardown
invalidates all handles and cursors. The runtime MUST stop work that has not
crossed its effect boundary and MUST NOT start further publication steps under
the revoked endpoint. Already-issued effects may finish; the runtime MUST retain
their outcome for shell inspection and MUST NOT send late results to another
endpoint. Policy revocation follows the same effect rule.

Objects read, created, or fetched through a handle MUST remain available for
that handle's lifetime unless access is revoked or storage fails. References
may be shared with other authorized callers; the handle is not a private clone.

### Reads and pagination

`refs` lists authorized local branch and tag references. It performs no remote
refresh. Remote state is obtained with `fetch`. A first page captures one local
reference snapshot; subsequent pages MUST continue that snapshot, even if refs
move. An empty repository returns `entries: []`, `next: null`.
The runtime MUST NOT normalize or lossily decode reference names. A visible
reference whose name cannot be represented as UTF-8 makes listing fail with
`unsupported` rather than silently hiding that reference.

`get` returns a verified object's structured view. Tree entries are sorted
lexicographically by unsigned name bytes. `cursor` and `limit` are valid only
for trees. For both paginated operations, omission of `limit` selects a runtime
default. The runtime MAY cap the page size but MUST use `next` rather than
silently truncate. Continuations MUST repeat the original arguments, including
the same limit or its omission. Invalid or expired cursors return `invalid-cursor`;
the caller restarts from the first page. An incomplete page MUST advance.

`read` returns exactly `min(length, max(0, size - offset))` bytes. An offset at
or beyond the end returns empty bytes and `eof: true`. Zero length is valid.
The runtime MUST reject an excessive requested length with `too-large`, rather
than shorten a successful read. It MAY obtain authorized missing objects from
approved backends; inability to obtain an object returns `not-found` or
`unavailable`. Symlinks and gitlinks are data; the runtime MUST NOT follow them
to host files or submodule repositories.

### Local mutations

`createCommit` constructs blobs, trees, and one commit. Parent ancestry MUST be
preserved exactly. It does not update a reference, staging index, working
directory, or remote. The shell selects the author, committer, timestamps, and
any Git signing identity under user policy. No napplet field selects a signer.
The returned object's stored headers are inspectable with `get` and `read`.
The message MUST be stored as its exact UTF-8 bytes, without normalization or
line-ending changes.
Git signatures and Nostr event signatures are separate mechanisms.

The runtime MUST validate all changes before constructing the commit. Changes
MUST NOT overlap as ancestor/descendant paths. `put` replaces or creates only a
regular file; replacing a tree, symlink, or gitlink returns `conflict`. Missing
parent directories are created. Traversing a non-tree returns `conflict`.
Deleting a missing entry returns `not-found`; deleting the last entry removes
empty ancestor trees. Unchanged entries and their modes MUST be preserved.
The runtime MUST reject `.git` path segments, ASCII case-insensitively, and
backslash bytes in changes. These restrictions do not hide existing tree names
from reads. Invalid input or failure MUST NOT update refs; unreachable objects
left by failed creation MAY be garbage-collected.

`updateRef` is a local compare-and-swap: the runtime MUST atomically compare
the current direct target with `expected` and apply `oid`, or return `conflict`
without changing that reference. It MUST validate a non-null target as a local
commit. Symbolic destinations MUST be rejected. Rewrites and deletions require
explicit shell policy. The runtime MUST NOT change a branch attached to a
working directory; it MUST return `conflict`. Success returns the applied
request record. Runtime-managed bare repositories avoid this restriction.
This transaction makes no statement about remote state.

### Remote effects

`fetch` resolves one named remote reference according to that remote's authority,
obtains its target and reachable Git objects, and returns the observed target.
Success requires complete reachable history and trees, excluding gitlink targets.
A runtime limited to incomplete history returns `unavailable`. A missing ref
returns `not-found`. Fetch MUST NOT move local branches, tags, or working
directories. Internal caches and tracking metadata MAY change. A concurrent
remote change does not invalidate the returned observation.

`push` requests publication of exactly `oid` to the destination branch, with
its reachable objects. Only branch creation and fast-forward updates are in
this draft. A fast-forward retains the previous commit as an ancestor of the
new commit. The runtime MUST check the authorized remote state before
publication and apply any stronger checks the backend supports. A known
non-fast-forward returns `conflict`. Force, deletion, mirror refspecs, and
automatic merge are excluded. A backend that cannot perform the requested
reference update MUST return `unsupported`; it MUST NOT silently submit a proposal.

Remote publication is not the local `updateRef` transaction. Neither a prior
`fetch` nor a successful `push` guarantees exclusive ownership or global
compare-and-swap on a distributed remote. The runtime MUST NOT claim such a
guarantee. It MUST preserve the exact requested ref and object ID when signing
or sending a publication. Competing signed state can later supersede it.
When a backend represents several refs in one state document, the runtime MUST
preserve unrelated refs from the resolved state. This preserves the observed
state; it cannot prevent an unseen concurrent update from being superseded.

`propose` submits the commits after `base` through `head` for the named target
branch, with the supplied title and description. The runtime MUST verify
ancestry and preserve that range. It MUST NOT merge or update the target branch.
The runtime owns representation as patches, a remote branch plus pull request,
or Nostr proposal events. If it cannot preserve the requested range or metadata,
it returns `unsupported`. Native Git alone need not support proposals.
This operation creates a new proposal; revisions and lifecycle changes are
outside this draft. A proposal receipt SHOULD supply its address as `locator`.

Before `push` or `propose`, the shell MUST authorize the complete publication
plan: destinations, disclosed history, signer/account, and each required effect.
The runtime MUST NOT add destinations after authorization without renewed
mediation. Supporting object uploads count as publication even if a later
reference or proposal step fails. For patches carrying all required content,
a separate object-upload effect is unnecessary.

After publication begins, the runtime MUST return `GitPublication`, including
on timeout or partial failure, rather than hide receipts in an ordinary error.
Every required effect/destination pair MUST have one receipt. Unattempted steps
are `rejected`; acceptance requires positive acknowledgement or verified
preexisting satisfaction. Outcome is derived in this order:

| Condition | `outcome` |
|-----------|-----------|
| Any receipt is `unknown` | `"unknown"` |
| All receipts are `accepted` | `"complete"` |
| All receipts are `rejected` | `"failed"` |
| Otherwise | `"partial"` |

An accepted `objects` receipt means the required objects are available at that
destination. An accepted `ref` receipt means the selected ref authority accepted
the exact update; for signed distributed state, selected relay acceptance is
the evidence. An accepted `proposal` receipt means the proposal was accepted
by its selected service or relay. `push` requires at least one `ref` receipt and
no `proposal` receipts. `propose` requires at least one `proposal` receipt and
MUST NOT include a `ref` receipt for the target branch. Supporting proposal
branches MAY have their own `ref` receipts. `complete` does not promise global
consensus, permanent availability, replication to unselected recipients, or merge.

## Native Git and ngit Mapping

This table is implementation guidance, not a claim of NAP support. Upstream
formats remain owned by [Git](https://git-scm.com/docs) and
[NIP-34](https://github.com/nostr-protocol/nips/blob/master/34.md).

| Contract | Native Git implementation | ngit implementation |
|----------|---------------------------|---------------------|
| `open` | Approved local object store or managed clone | Resolve approved Nostr repository identity; manage a Git object store |
| `get`, `read`, `createCommit` | Git object database | The same Git object model; Nostr event IDs never replace Git IDs |
| `refs`, `updateRef` | Local refs and atomic local transactions | Local refs and the same local transactions; no implicit event publication |
| `fetch` | Resolve remote refs and fetch objects | Resolve authorized signed state and obtain objects from approved Git servers |
| `push` | Publish objects and update exact remote branch | Publish objects where needed and authorized signed branch state |
| `propose` | Optional forge or patch-delivery integration | Construct and publish the appropriate Nostr proposal representation |

ngit separates signed repository state from Git object hosting, as described
in its [architecture guide](https://ngit.dev/how-it-works). The runtime MUST
verify signatures and the selected repository's maintainer authority before
using signed remote state. An announcement or server's possession of objects
alone does not authorize its refs. The runtime MUST follow its selected
backend's state-selection rules; unresolved conflicting authority returns
`conflict`. It MUST NOT substitute a reachable object's host as the state
authority when signed state is unavailable.
Before publishing signed branch state, the runtime MUST verify that the selected
signer has maintainer authority for the opened repository. It MUST return
`denied` before publication if that authority is absent.

ngit's [proposal branch behavior](https://ngit.dev/pull-requests) belongs behind
`propose`. An implementation may use a remote helper internally, but MUST
prevent helper naming rules, implicit tracking configuration, or automatic
fallbacks from changing the requested effect. Separate receipts expose Git
server uploads and relay publication; one successful step does not establish
the other. No generic relay or signer surface is exposed to the napplet by `git`.

## Wire Protocol

The following language-neutral messages carry this contract. The
[web projection](../projections/web.md) maps them to the
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) envelope. `?` marks
optional fields; it is not part of a field name.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `git.open` | napplet -> runtime | `id`, `source?` |
| `git.open.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `git.close` | napplet -> runtime | `id`, `repository` |
| `git.close.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `git.refs` | napplet -> runtime | `id`, `repository`, `cursor?`, `limit?` |
| `git.refs.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `git.get` | napplet -> runtime | `id`, `repository`, `oid`, `cursor?`, `limit?` |
| `git.get.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `git.read` | napplet -> runtime | `id`, `repository`, `oid`, `offset`, `length` |
| `git.read.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `git.createCommit` | napplet -> runtime | `id`, `repository`, `request` |
| `git.createCommit.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `git.updateRef` | napplet -> runtime | `id`, `repository`, `request` |
| `git.updateRef.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `git.fetch` | napplet -> runtime | `id`, `repository`, `remote`, `ref` |
| `git.fetch.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `git.push` | napplet -> runtime | `id`, `repository`, `request` |
| `git.push.result` | runtime -> napplet | `id`, `result?`, `error?` |
| `git.propose` | napplet -> runtime | `id`, `repository`, `request` |
| `git.propose.result` | runtime -> napplet | `id`, `result?`, `error?` |

`id` is nonempty text, unique among in-flight requests from that endpoint.
The runtime MUST return one corresponding result with the same `id` for every
recognized request with a valid `id` from a live authenticated endpoint.
Results MUST be routed only to that endpoint. There are no unsolicited messages
or wire subscriptions in this draft. Unrecognized message types and messages
from unauthenticated endpoints follow the projection's rules.

A successful envelope MUST contain `result` of the operation's declared type
and omit `error`. A failed envelope MUST contain `error` and omit `result`.
A publication result can report an unsuccessful effect through `outcome`;
successful delivery of the result does not establish successful publication.
Unknown request fields MUST be rejected with `invalid-request` before effects,
so misspelled authority or precondition fields cannot silently disappear.
Receivers MAY ignore unknown result fields.

### Error Handling

`GitError` (`error`):

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `code` | yes | text | One value from the table below. |
| `message` | no | text | Safe human-readable detail. Not for programmatic branching. |

| Code | Meaning |
|------|---------|
| `invalid-request` | Missing, malformed, unexpected, or contradictory fields; invalid path, ref, object ID, base64, or range. |
| `invalid-handle` | Repository access is absent or ended. |
| `invalid-cursor` | Cursor is absent from its scope, expired, or used with different arguments. |
| `not-found` | Authorized object, ref, remote, or entry is absent or hidden. |
| `denied` | Shell policy or user choice refused the request. |
| `cancelled` | User cancelled selection, or work stopped before its observable mutation/publication. |
| `unsupported` | Operation, source scheme, object format, or requested semantics cannot be provided. |
| `conflict` | Local precondition failed, ancestry is unsuitable, paths conflict, or remote authority cannot be resolved. |
| `too-large` | Request, object, or requested read exceeds runtime limits. |
| `busy` | Runtime work or storage quota prevents the operation. |
| `unavailable` | Required data, transport, credentials, or signer is unavailable. |
| `integrity` | Object hash, structure, signature, or authority verification failed. |

The runtime MUST validate and authorize before observable mutations or
publication. Policy MAY return `not-found` to hide unauthorized data. Ordinary
errors establish that the requested mutation/publication did not occur; internal
cache fills and unreachable objects are permitted. Started publication follows
the receipt rules instead. Invalid or missing `id` cannot be correlated and MAY
be discarded. A caller MUST NOT reuse an in-flight ID; a runtime detecting a
duplicate MUST NOT execute a second effect or emit a second result for that ID.

Correlation IDs are not persistent deduplication keys. After losing a result,
the caller MUST treat mutation outcome as uncertain. It SHOULD inspect local
refs or remote state and use shell-mediated review before retrying publication.
Repeating `propose` can create duplicate public proposals. There is no automatic
retry guarantee, reattachment, or exactly-once remote publication in this draft.

### Examples

The SHA-1 fixture changes `README.md` from `Hi` plus LF to `Hello` plus LF.
The root message is `Initial` plus LF. Both commits use author and committer
`Example <example@example.org> 0 +0000`, with no extra headers. These are test
objects, not a live repository. Encoded paths and bytes are literal base64.

**Open, then capture local refs:**

```text
-> { "type": "git.open", "id": "o1", "source": "https://git.example.org/tools.git" }
<- { "type": "git.open.result", "id": "o1", "result": { "repository": "repo-1", "objectFormat": "sha1", "operations": ["open", "close", "refs", "get", "read", "createCommit", "updateRef", "fetch", "push"], "remotes": [{ "remote": "remote-1", "operations": ["fetch", "push"] }] } }
-> { "type": "git.refs", "id": "r1", "repository": "repo-1" }
<- { "type": "git.refs.result", "id": "r1", "result": { "entries": [{ "ref": "refs/heads/main", "oid": "eb7f6a0e5a12e427bec3f9af5fd505fc2efc1fa4" }], "next": null } }
```

**Create an edited commit, then install it with a local precondition:**

```text
-> { "type": "git.createCommit", "id": "c1", "repository": "repo-1", "request": { "parent": "eb7f6a0e5a12e427bec3f9af5fd505fc2efc1fa4", "changes": [{ "path": "UkVBRE1FLm1k", "action": "put", "mode": "100644", "data": "SGVsbG8K" }], "message": "Update README\n" } }
<- { "type": "git.createCommit.result", "id": "c1", "result": { "oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c" } }
-> { "type": "git.updateRef", "id": "u1", "repository": "repo-1", "request": { "ref": "refs/heads/main", "expected": "eb7f6a0e5a12e427bec3f9af5fd505fc2efc1fa4", "oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c" } }
<- { "type": "git.updateRef.result", "id": "u1", "result": { "ref": "refs/heads/main", "expected": "eb7f6a0e5a12e427bec3f9af5fd505fc2efc1fa4", "oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c" } }
```

**Stale local precondition; reference unchanged by this request:**

```text
-> { "type": "git.updateRef", "id": "u2", "repository": "repo-1", "request": { "ref": "refs/heads/main", "expected": "eb7f6a0e5a12e427bec3f9af5fd505fc2efc1fa4", "oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c" } }
<- { "type": "git.updateRef.result", "id": "u2", "error": { "code": "conflict" } }
```

**Inspect the committed tree and read a blob:**

```text
-> {"type": "git.get", "id": "g1", "repository": "repo-1", "oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c"}
<- {"type": "git.get.result", "id": "g1", "result": {"oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c", "kind": "commit", "size": 202, "commit": {"tree": "dc0a29da1d9b3f68dcd56af0e34f8df4fbf8b24f", "parents": ["eb7f6a0e5a12e427bec3f9af5fd505fc2efc1fa4"], "author": "RXhhbXBsZSA8ZXhhbXBsZUBleGFtcGxlLm9yZz4gMCArMDAwMA==", "committer": "RXhhbXBsZSA8ZXhhbXBsZUBleGFtcGxlLm9yZz4gMCArMDAwMA==", "message": "VXBkYXRlIFJFQURNRQo="}}}
-> {"type": "git.get", "id": "g2", "repository": "repo-1", "oid": "dc0a29da1d9b3f68dcd56af0e34f8df4fbf8b24f"}
<- {"type": "git.get.result", "id": "g2", "result": {"oid": "dc0a29da1d9b3f68dcd56af0e34f8df4fbf8b24f", "kind": "tree", "size": 37, "entries": [{"name": "UkVBRE1FLm1k", "mode": "100644", "oid": "e965047ad7c57865823c7d992b1d046ea66edf78"}], "next": null}}
-> {"type": "git.read", "id": "b1", "repository": "repo-1", "oid": "e965047ad7c57865823c7d992b1d046ea66edf78", "offset": 0, "length": 6}
<- {"type": "git.read.result", "id": "b1", "result": {"data": "SGVsbG8K", "offset": 0, "size": 6, "eof": true}}
```

**Fetch records an observation without moving the local branch:**

```text
-> {"type": "git.fetch", "id": "f1", "repository": "repo-1", "remote": "remote-1", "ref": "refs/heads/main"}
<- {"type": "git.fetch.result", "id": "f1", "result": {"ref": "refs/heads/main", "oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c"}}
```

**ngit publication: object upload accepted, signed-state acceptance uncertain.**
Assume `repo-n` and `remote-n` were opened and authorized, and the source commit
is a fast-forward of the observed destination.

```text
-> { "type": "git.push", "id": "p1", "repository": "repo-n", "request": { "remote": "remote-n", "ref": "refs/heads/main", "oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c" } }
<- {"type": "git.push.result", "id": "p1", "result": {"outcome": "unknown", "receipts": [{"effect": "objects", "destination": "object-store-1", "status": "accepted", "oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c"}, {"effect": "ref", "destination": "state-relay-1", "status": "unknown", "oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c", "ref": "refs/heads/main"}]}}
```

**Proposal submission is independent of branch update.** Assume this remote
supports `propose`, and both commits are present with the stated ancestry.

```text
-> { "type": "git.propose", "id": "p2", "repository": "repo-n", "request": { "remote": "remote-n", "target": "refs/heads/main", "base": "eb7f6a0e5a12e427bec3f9af5fd505fc2efc1fa4", "head": "800cea77c0e8f383c0c6fb6c52b16980d634c17c", "title": "Update README", "description": "Clarify the greeting." } }
<- {"type": "git.propose.result", "id": "p2", "result": {"outcome": "complete", "receipts": [{"effect": "proposal", "destination": "proposal-relay-1", "status": "accepted", "oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c"}]}}
```

**Malformed input and denial despite declared domain availability:**

```text
-> { "type": "git.read", "id": "e1", "repository": "repo-1", "oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c", "offset": -1, "length": 20 }
<- { "type": "git.read.result", "id": "e1", "error": { "code": "invalid-request" } }
-> { "type": "git.push", "id": "e2", "repository": "repo-1", "request": { "remote": "remote-1", "ref": "refs/heads/main", "oid": "800cea77c0e8f383c0c6fb6c52b16980d634c17c" } }
<- { "type": "git.push.result", "id": "e2", "error": { "code": "denied" } }
```

**Release access; subsequent reads fail:**

```text
-> {"type": "git.close", "id": "x1", "repository": "repo-1"}
<- {"type": "git.close.result", "id": "x1", "result": null}
-> {"type": "git.read", "id": "x2", "repository": "repo-1", "oid": "e965047ad7c57865823c7d992b1d046ea66edf78", "offset": 0, "length": 6}
<- {"type": "git.read.result", "id": "x2", "error": {"code": "invalid-handle"}}
```

A second endpoint using `repo-1` also receives `invalid-handle`, even while the
first endpoint's handle is live. Equal request IDs on different endpoints are
independent; neither receives the other's result. If the first endpoint ends
while publication is in flight, the shell retains receipts and delivers no
late result to the second endpoint.

## Shell Behavior

- The shell MUST bind access to the projection-authenticated endpoint and its
  verified manifest/artifact identity. `git` declarations and handles do not
  establish user identity or publication permission.
- The runtime MUST enforce repository, object/history, reference, remote, and
  operation scope on every request. Possession of an object ID is not read
  authority. A policy unable to authorize disclosed history MUST deny access.
  The runtime MUST deny an object read rather than redact or rewrite the content
  or structured fields of a hash-identified object.
- The shell MUST keep account and repository selection under user control.
  A source URL, Nostr pubkey, author header, or remote identifier MUST NOT select
  credentials or grant maintainer authority by itself.
- The runtime MUST enforce finite work, byte, pagination, storage, and operation
  duration limits. It MAY choose their values. Overload MUST produce `busy` or
  `too-large`; publication deadlines MUST preserve receipts and uncertainty.
- The runtime MAY implement this contract using executables, libraries, local
  stores, or services. It MUST preserve the same observable semantics.

## Security Considerations

- Git content and configuration are untrusted. The runtime MUST prevent
  repository-controlled hooks, filters, credential helpers, diff drivers, Git
  aliases, command substitutions, and arbitrary remote helpers from executing.
  Only runtime-approved integrations may run. Napplet fields MUST NOT become
  command options, environment settings, executable names, or shell code.
- The runtime MUST contain object parsing and storage independently of host
  path interpretation. It MUST bound decompression and graph traversal, reject
  malformed objects, and prevent traversal through aliases, symlinks, alternates,
  submodules, or malicious repository configuration into unauthorized stores.
- Network destinations from source URLs, redirects, announcements, server
  responses, or Git metadata are untrusted hints. The runtime MUST apply network
  policy to every connection and redirect, including discovered object hosts and
  relays. It MUST NOT send credentials to an unapproved destination.
- Publication can disclose entire reachable history, including deleted secrets.
  The shell MUST authorize that history, not merely the currently edited files.
  Permission to read or create local commits does not grant upload permission.
- Repository handles, remote metadata, receipts, locators, and errors MUST NOT
  expose host paths, credentials, private keys, signer tokens, or another
  endpoint's repositories. Private repository names and destinations require
  explicit disclosure policy. The shell MUST retain signing authority.
- Object hashes prove content identity, not author identity, maintainer
  authority, code safety, or reproducibility of a napplet artifact. Signed
  repository state also does not make referenced code safe to execute.

## Implementations

No implementation of this NAP is verified. Git and ngit are implementation
targets; their existing interfaces do not establish support for this draft.

## Changelog

- `bb71090` - Defined shared Git object access, commit creation, local reference transactions, remote fetch/push, optional proposals, publication receipts, and native/ngit authority boundaries.
