NAP-RESOURCE
============

Sandboxed Resource Fetching
---------------------------

`draft`

**NAP ID:** NAP-RESOURCE
**Domain:** `resource`
**Depends:** none.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.resource`; domain presence signals availability.
**Parent:** [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)

## Description

NAP-RESOURCE lets a napplet request byte resources through the runtime:

- `resource.bytes(url)` returns one complete byte sequence.
- `resource.bytesMany(urls)` returns ordered `ResourceBytesItem` records.

These URL operations return complete, policy-approved resources. The optional
[FIPS block profile](#optional-fips-block-profile) adds selective reads of
immutable blocks under an approved root. It does not change URL resolution.

For URL operations, the napplet supplies URLs. The runtime owns fetch, scheme dispatch, policy,
MIME classification, SVG rasterization, caching, quotas, and errors. This domain
MUST NOT grant raw network access or expose upstream `Content-Type`. A projection
MUST enforce direct-network denial before exposing this domain; otherwise its
resource policy can be bypassed.

The [web resource binding](../projections/web.md#resource-binding) defines the
network-isolation prerequisite and conversion to browser `Blob` results. An
opaque sandbox origin alone does not provide that isolation.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `info` | none | `ResourceInfo` | `resource.info` |
| `bytes` | `url`, `opts?` | one complete byte sequence | `resource.bytes` |
| `bytesMany` | non-empty `urls`, `opts?` | ordered per-URL results | `resource.bytesMany` |

`info()` is optional introspection. Napplets MAY call it to adapt UI or choose
supported URL forms, but MUST NOT be required to call it before `bytes` or
`bytesMany`.

A caller MAY cancel `bytes` or `bytesMany`. Cancellation sends `resource.cancel`
for the request `id`. Late terminal envelopes for cancelled IDs MUST be dropped.

`bytesMany` reduces envelope count only. Each URL MUST be processed as if it
were an independent `bytes(url)` request. One failed URL MUST NOT discard
successful siblings.

All resource state is scoped to the napplet's verified manifest and artifact scope.
A napplet MUST NOT read another napplet's resource cache.

## Identity Scope

The runtime MUST derive scope from the authenticated endpoint's verified
manifest and artifact. For a named manifest, the manifest key is its publisher,
kind, and `d` value; for a root manifest, publisher and kind; for a snapshot,
its signed event id. The scope also includes `artifactHash`, the verified
single-artifact hash defined by [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303).
A bare `d` value MUST NOT identify a scope. Root and snapshot manifests require
no `d` tag. A napplet MUST NOT supply or override manifest or artifact identity fields in a request.
Different publishers, manifest keys, and artifact hashes MUST remain isolated.

## Wire Protocol

`resource.*` messages use [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format:
`{ "type": "domain.action", ...payload }`.

| Type | Direction | Payload |
|------|-----------|---------|
| `resource.info` | napplet -> runtime | `id` |
| `resource.info.result` | runtime -> napplet | `id`, `info` |
| `resource.info.error` | runtime -> napplet | `id`, `error`, `message?` |
| `resource.bytes` | napplet -> runtime | `id`, `url` |
| `resource.bytesMany` | napplet -> runtime | `id`, `urls` |
| `resource.cancel` | napplet -> runtime | `id` |
| `resource.bytes.result` | runtime -> napplet | `id`, `blob`, `mime` |
| `resource.bytes.error` | runtime -> napplet | `id`, `error`, `message?` |
| `resource.bytesMany.result` | runtime -> napplet | `id`, `items` |
| `resource.bytesMany.error` | runtime -> napplet | `id`, `error`, `message?` |

### Byte encoding

`ResourceBytes` is text containing canonical padded base64 as defined by
[RFC 4648, section 4](https://www.rfc-editor.org/rfc/rfc4648.html#section-4).
Every wire `blob` field MUST use it: single results, bulk items, block results, and
`ResourceSidecarEntry`. Use the standard alphabet, required `=` padding, zero
pad bits, and no whitespace or data-URL prefix. Empty bytes encode as `""`.
Receivers MUST reject non-canonical encodings. Host byte objects and integer
arrays MUST NOT appear in these wire fields.

For URL results, the decoded bytes are the complete policy-approved resource,
after any SVG rasterization. Hash verification applies to fetched content before
transformation. Block results contain unchanged stored bytes, as specified below.
Resource byte limits and quotas apply to decoded bytes, not base64 text length.
The web binding decodes this representation into `Blob`; it MUST NOT encode the
base64 characters themselves as the Blob contents.

For example, bytes `00 01 02 ff` encode as `"AAEC/w=="`:

```json
{ "type": "resource.bytes.result", "id": "r1", "blob": "AAEC/w==", "mime": "application/octet-stream" }
```

### Schemas

`ResourceSchemeInfo` fields:

| Field | Required | Type |
|-------|----------|------|
| `scheme` | yes | text |
| `enabled` | yes | boolean |

`ResourceInfo` fields:

| Field | Required | Type |
|-------|----------|------|
| `schemes` | yes | list of `ResourceSchemeInfo` |
| `maxBytes` | no | unsigned integer |
| `maxUrls` | no | unsigned integer |
| `profiles` | no | list of text |
| `maxBlockBytes` | no | integer |

`ResourceBytesItem` fields:

| Field | Required | Type |
|-------|----------|------|
| `url` | yes | text |
| `ok` | yes | boolean |
| `blob` | no | `ResourceBytes` |
| `mime` | no | text |
| `error` | no | text |
| `message` | no | text |

Rules:

- Every `resource.info`, `resource.bytes`, and `resource.bytesMany` request gets one terminal result or error envelope unless cancelled.
- `resource.info` is advisory and MUST NOT be a required preflight.
- `bytesMany.result.items` MUST preserve input order and length.
- `ok: true` items MUST include `blob` and `mime`.
- `ok: false` items MUST include `error` and MUST NOT include `blob`.
- `mime` MUST be runtime-classified by byte sniffing, never upstream header.
- Successful URL results deliver complete resources only. Block-profile results
  deliver complete blocks. Neither surface defines partial-block streaming, byte
  ranges, or progress events.
- `message?` is diagnostic only. Programmatic handling uses `error`.
- Unsupported schemes still fail per request with `unsupported-scheme`, even if
  the napplet skipped `resource.info`.

## Schemes

| Scheme | Rules |
|--------|-------|
| `data:` | MAY decode in the napplet shim. Whether decoded in the shim or runtime, it MUST be size-limited, MIME-sniffed, rasterized when SVG, quota-checked, and otherwise policy-checked. No network access. |
| `https:` | Runtime fetch. Full Default Resource Policy applies. Returned `mime` is sniffed, not upstream `Content-Type`. |
| `blossom:` | [BUD-10](https://github.com/hzrd149/blossom/blob/master/buds/10.md) Blossom URI: `blossom:<sha256>.<ext>`. Runtime MUST verify SHA-256 before delivery. BUD-10 `xs`, `as`, and `sz` hints are advisory and never bypass policy, size limits, or the hash check. Upstream hosts use `https:` policy. |
| `htree:`, `hashtree:` | Hashtree references defined below. Runtime resolves file bytes, verifies hashes and decryption before delivery, and MUST NOT leak keys to relays, storage servers, or peers. |
| `nostr:` | [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) bech32. Runtime resolves one hop and returns the referenced bytes. MUST NOT recursively follow URLs or `nostr:` references in the result. |

Unknown schemes MUST return `unsupported-scheme`. `http:` is not canonical and
MUST NOT be enabled by default.

`resource.info.schemes` reports schemes the runtime is willing to disclose to
the napplet. It does not grant fetch authority. Each `bytes` or `bytesMany` URL
still passes through scheme dispatch and policy checks.

### Hashtree references

A runtime offering Hashtree reads MUST accept these forms, subject to policy:

| Form | Normative syntax and resolution |
|------|---------------------------------|
| `hashtree:<nhash>` | Immutable identifier defined by [HTS-01 sections 6.2–6.3](https://github.com/mmalmi/hashtree/blob/master/docs/HTS-01.md#62-nhash). Bare `nhash` is an accepted alias dispatched to the same resolver. |
| `htree://<identifier>/<tree>[/<entry-path>][#k=<secret>]` | Mutable root defined by [HTS-01 section 9](https://github.com/mmalmi/hashtree/blob/master/docs/HTS-01.md#9-htree-url-profile). Portable identifiers are an `npub` or a 64-hex public key. `#private` is also accepted, subject to authorization; creation-only `#link-visible` MUST be rejected by this read API. |

`secret` is exactly 64 hex characters, as defined by HTS-01. Immutable
`nhash` input MUST validate its Bech32 checksum and HTS-01 TLV fields.

Tree names and entry segments MUST follow upstream
[URL encoding](https://github.com/mmalmi/hashtree/blob/master/docs/URL-ENCODING.md):
split segments before decoding; a tree name containing `/` occupies one encoded
segment. Query strings, malformed escapes, unknown fragments, and colon-separated
mutable paths MUST return `invalid-request`. Directories do not yield file bytes
and MUST return `invalid-request`.

HTS-01's `self` and petname aliases MAY be accepted only when runtime policy can
resolve them unambiguously; otherwise return `invalid-request`. Bare hex CIDs
and unspecified “compatible” forms are not inputs to this API. Hashtree support
MUST advertise both `htree` and `hashtree` in `info.schemes` when disclosed;
`nhash` is an alias, not a scheme. Keys embedded in either form remain local to
the runtime resolver. Scheme support does not grant access to private roots.

## Optional FIPS block profile

`fips-hashtree/1` is an optional, read-only profile within `resource`. A runtime
MAY use FIPS internally to resolve existing Hashtree URLs without exposing this
profile. The profile is for napplets that choose which blocks of a large file to
read, such as a map selecting only the tiles in view. A block is one complete
stored object addressed by SHA-256; it is not a byte range or an assembled file.

The napplet chooses the root and traversal order. The shell mediates access. The
runtime authenticates providers and verifies every block and descendant link.
Application discovery schemas, mutable-root selection, file assembly and domain
logic remain outside this profile. No new NAP domain, URL scheme, socket, upload,
RPC, payment or signing API is introduced.

### Availability and operations

A runtime exposing the profile MUST implement `info` and include
`"fips-hashtree/1"` in `info.profiles`. Absence of `profiles`, or of this value,
means the profile is unavailable. Presence of `resource` alone is insufficient.
This is additive: URL callers still need no introspection preflight. A runtime
implementing these operations MUST return `unsupported-profile` for an unknown
profile. Existing runtimes without these operations are not required to handle
new message types; callers MUST discover support before using them.

`info.maxBlockBytes`, when disclosed, is a positive integer describing the
current stored-block limit. It is independent of the assembled-resource
`maxBytes` limit. Introspection grants no authority.

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `open` | `request: ResourceBlockRequest`, `opts?` | `ResourceBlockGrant` | `resource.open` |
| `block` | `grant`, `hash`, `opts?` | one complete verified stored block | `resource.block` |
| `release` | `grant` | acknowledgement | `resource.release` |

| Type | Direction | Payload |
|------|-----------|---------|
| `resource.open` | napplet -> runtime | `id`, `request` |
| `resource.open.result` | runtime -> napplet | `id`, `grant: ResourceBlockGrant` |
| `resource.open.error` | runtime -> napplet | `id`, `error`, `message?` |
| `resource.block` | napplet -> runtime | `id`, `grant`, `hash` |
| `resource.block.result` | runtime -> napplet | `id`, `hash`, `blob` |
| `resource.block.error` | runtime -> napplet | `id`, `error`, `message?` |
| `resource.release` | napplet -> runtime | `id`, `grant` |
| `resource.release.result` | runtime -> napplet | `id` |
| `resource.release.error` | runtime -> napplet | `id`, `error`, `message?` |

Each request receives one terminal result or error unless cancelled. Request IDs
share the resource domain's request namespace. In `block` and `release`, `grant`
is the `ResourceBlockGrant.id` text, not the entire grant record.
`resource.cancel` also cancels
`open` and `block`; cancelling an open MUST leave no newly created grant, even
when approval completed concurrently. Cancelling a block MUST remove queued work
and abandon its active transfer when no other authorized reader needs it. Other
readers MUST remain unaffected. Late results MUST be discarded.

### Schemas

All integers in this profile MUST be exact and within `0..9007199254740991`.
A `hash` is exactly 64 lowercase hexadecimal characters encoding SHA-256.

`ResourceBlockRequest`:

| Field | Required | Type | Meaning |
|-------|----------|------|---------|
| `profile` | yes | text | Exactly `"fips-hashtree/1"`. |
| `root` | yes | text | Immutable root hash. |
| `rootType` | yes | `"blob"` or `"file"` | Raw blob or unencrypted Hashtree File node. |
| `bytes` | yes | integer | Logical file length; exact stored length for a blob root. Zero is valid only for a blob root. |
| `providers` | yes | list of `ResourceBlockProvider` | Non-empty set of proposed provider identities. |
| `expires` | yes | integer | Future Unix timestamp in seconds; latest permitted grant expiry. |

`ResourceBlockProvider`:

| Field | Required | Type | Meaning |
|-------|----------|------|---------|
| `peerId` | yes | text | 33-byte compressed secp256k1 public key: `02` or `03`, then 64 lowercase hex characters; MUST decode to a valid curve point. This is a FIPS provider identity, not a Nostr x-only public key. |
| `seedUrls` | no | list of text | Advisory connection hints. The runtime MAY use its own approved routes. Hints never replace provider authentication or grant authority. |

`ResourceBlockGrant`:

| Field | Required | Type | Meaning |
|-------|----------|------|---------|
| `id` | yes | text | Non-empty opaque runtime-issued handle bound to this endpoint and its verified manifest/artifact scope. |
| `expires` | yes | integer | Effective expiry, no later than the request's expiry. |
| `maxBlockBytes` | yes | integer | Positive maximum stored bytes per block, including encoded tree nodes. |
| `timeoutMs` | yes | integer | Positive finite deadline for each block request, measured from runtime receipt, including queue time. |

The runtime MUST validate the request before prompting or connecting. It MUST
reject unknown request/provider fields, invalid identities, unsupported root
types and invalid lengths with `invalid-request`. An already expired
request returns `expired`. A grant MUST NOT silently widen its root, provider
set or lifetime. A runtime MAY approve a subset of providers, shorten expiry or
apply smaller quotas. Changing the approved set requires a new grant.

For example, this exchange reads the four bytes `00 01 02 ff`. The provider key
and limits are illustrative; no live provider or fixed policy limit is implied.
The runtime supplies approved routes when no seed hints are present.

```json
{
  "type": "resource.open", "id": "o1",
  "request": {
    "profile": "fips-hashtree/1",
    "root": "3d1f57c984978ef98a18378c8166c1cb8ede02c03eeb6aee7e2f121dfeee3e56",
    "rootType": "blob", "bytes": 4, "expires": 2147483647,
    "providers": [{ "peerId": "0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798" }]
  }
}
```

```json
{ "type": "resource.open.result", "id": "o1", "grant": { "id": "g1", "expires": 2147483647, "maxBlockBytes": 2097152, "timeoutMs": 30000 } }
```

```json
{ "type": "resource.block", "id": "b1", "grant": "g1", "hash": "3d1f57c984978ef98a18378c8166c1cb8ede02c03eeb6aee7e2f121dfeee3e56" }
```

```json
{ "type": "resource.block.result", "id": "b1", "hash": "3d1f57c984978ef98a18378c8166c1cb8ede02c03eeb6aee7e2f121dfeee3e56", "blob": "AAEC/w==" }
```

```json
{ "type": "resource.release", "id": "r2", "grant": "g1" }
```

```json
{ "type": "resource.release.result", "id": "r2" }
```

### Provider access and mediation

The shell MUST authorize the root and provider set before any connection for
that request. Approval MAY follow an explicit user decision or an existing
user-controlled rule. A signed discovery record authenticates an advertisement;
it does not authorize a connection. The shell MUST let the user revoke access.
A denied request returns `blocked-by-policy` without contacting its seeds or
providers. Public reads MUST NOT require a Nostr account or a signature.

The runtime owns transport identities, routing, connection limits, scheduling,
retries and caches. It MUST authenticate the end provider against the approved
`peerId`, independently of any seed or intermediary. It MUST NOT expose transport
secret keys or reuse the user's Nostr signing key. This profile grants neither
serving cached blocks nor forwarding traffic for other peers.

The provider read protocol is [Hashtree Blob protocol v1 over FIPS/TCP](https://github.com/mmalmi/hashtree/blob/master/docs/NETWORKING.md#blob-protocol-v1),
service 39018. The runtime MUST request HTL 0: the selected provider may serve its
own data but is not asked to forward the blob lookup. Ordinary FIPS routing is
unaffected. Arbitrary service ports and application request bodies are outside
this profile. Seed hints are untrusted destinations subject to the projection's
connection policy; the provider identity is not an exemption from that policy.

A verified reply from any approved provider may satisfy a read. `NoResult` from
one provider is not global absence. Return `not-found` only when all selected
providers give `NoResult`; if no verified block arrives and any attempt is
inconclusive, return `timeout` or `network-error` as appropriate. Hash mismatch
is `decode-failed`, not `not-found`. Missing results MUST NOT be cached as
permanent global absence. Retries of these immutable reads MUST remain within
the same grant and caller cancellation/deadline.

### Root authority and integrity

A grant initially authorizes only its root. The runtime MUST check the current
grant and known descendant membership before every read, including cache hits.
A hash supplied by the napplet is not a membership proof. The runtime learns
children only by validating a previously authorized File node under
[HTS-01's stored-object and node format](https://github.com/mmalmi/hashtree/blob/master/docs/HTS-01.md#3-tree-node-wire-format).
Before caching or delivering a block, the runtime MUST verify that SHA-256 of
its exact stored bytes equals the requested hash. It MUST NOT hash a
decoded/re-encoded representation. Cache reuse still requires this grant's
membership, type and length checks.

For this profile:

- Blob length MUST equal the length recorded in the root or verified parent.
- File nodes MUST have type File and a non-empty ordered link list. Each child
  MUST have a positive exact size. Child logical sizes MUST sum to the parent's
  logical size without overflow. File link order remains application-visible.
- Directory links, names and CHK keys MUST be rejected. Only unencrypted Blob
  and File children are accepted. Missing link type means Blob; unknown explicit
  types MUST be rejected. An encrypted/private tree needs a separate profile.
- Conflicting type/size claims for the same hash within a grant MUST be rejected.
  Depth, node size, link count and retained membership state MUST be bounded.
  Exceeding a local policy bound returns `too-large` or `quota-exceeded`.
- Only after the entire parent passes validation may its child hashes become
  authorized. Reads of an unlearned child return `blocked-by-policy`; callers
  must read and await the parent before scheduling its children.

`resource.block.result.hash` MUST equal the requested hash. Its `blob` MUST use
the existing `ResourceBytes` wire encoding and contain the complete stored bytes
verified against that hash. The runtime MUST check authority again before
completion so revoked/expired grants cannot receive a late success.

Block results are deliberately distinct from rendered resources: no MIME claim,
SVG rasterization, decompression or file assembly occurs. Transforming a block
would change its hash. This is an explicit exception to file MIME policy for
`resource.block` only; `bytes`, `bytesMany` and sidecars keep their existing rules.
A projection MUST expose block bytes as inert binary data, never automatically
navigate to them, render them or treat them as executable code. Direct-network
denial remains mandatory throughout the napplet's lifetime.

### Lifecycle, limits and errors

`release` is idempotent within the endpoint scope and never accesses another
endpoint's grant. It MUST revoke authority, cancel unfinished reads and discard
late successes before acknowledging. Releasing an absent, expired or already
released handle acknowledges without revealing another scope's state. Malformed
handles return `invalid-request`.

Expiry, endpoint disposal, artifact replacement, and a shell account change MUST
revoke grants and pending work. Revocation MUST take effect before cache access
or delivery. Cache retention MUST NOT preserve authority. The runtime MUST NOT
share a transport identity between different napplet scopes or account lifetimes.
It MUST retire transport identities and connections when their scope ends.

The runtime MUST bound grants, queued requests, active transfers, total bytes,
cache and membership memory. Policy limits are not fixed protocol constants.
The size of an entire file MUST NOT cause an eager download or count as one
block allocation. A read MUST terminate within the grant's `timeoutMs` or its
remaining lifetime, whichever is shorter; cancelling or expiring a request MUST
release its resources. These operations introduce no
napplet-visible progress or connection-state messages; shell diagnostics remain
shell-owned.

The following error codes apply to the block profile. `release` of a well-formed
absent handle remains a successful acknowledgement. A handle owned by another
endpoint MUST be indistinguishable from an unknown one. `info` remains advisory:
a profile listed earlier may still be denied or unavailable at request time.

| Code | Meaning |
|------|---------|
| `invalid-request` | Malformed payload, invalid identity, hash, handle or length. |
| `unsupported-profile` | `open` requested an unsupported profile. |
| `expired` | Requested expiry has passed, or a known grant expired during use. |
| `invalid-grant` | `block` used an unknown or revoked grant; forgotten expired handles may also return this. |
| `blocked-by-policy` | Root, provider, seed or descendant access was denied. |
| `not-found` | Every selected provider returned `NoResult`. |
| `timeout` | The read deadline elapsed without a verified result. |
| `network-error` | Connection or transport failure prevented a conclusive result. |
| `decode-failed` | Hash, node encoding, type or logical length failed verification. |
| `too-large` | Block or tree shape exceeds a runtime policy bound. |
| `quota-exceeded` | Grant, queue, transfer, byte or memory budget exhausted. |

### Privacy and acceptance

Before first approval, the shell MUST explain that seeds and routing peers can
observe connections and that providers learn the requested block hashes. FIPS
transport authentication/encryption and an ephemeral identity do not imply
anonymity. The root, provider set and effective expiry MUST be available in shell
controls. Napplets MUST NOT receive other napplets' connection or cache state.

Independent implementations should exercise these cases before claiming support:

| Case | Required outcome |
|------|------------------|
| Profile absent | Existing URL reads keep working; block controls are unavailable. |
| Root, parent, selected leaf | Only requested complete blocks are fetched; each hash and length is checked. |
| Guessed hash, unlearned child, cross-endpoint handle | No network/cache disclosure; deny without expanding authority. |
| Wrong hash, invalid tree, inconsistent length | No success or newly authorized descendants. |
| One missing provider; another returns data | Accept verified data; do not report global absence. |
| Missing plus timed-out provider | Report an inconclusive failure, not `not-found`. |
| Revocation, expiry or account change during read | Abort/discard pending work and late results, including cached results. |
| Cancel while approving or queued; shared active read | No leaked grant/work; another authorized reader still completes. |
| Denied root or disallowed seed | No connection; direct-network sandbox restrictions remain enforced. |
| Binary block round-trip | Exact stored bytes survive the projection; URL MIME/SVG policy remains unchanged. |
| Empty blob; explicit File root | Accept a verified empty blob; never infer root type from size or a preferred chunk size. |
| Queue consumes the deadline | Terminate within the granted deadline; do not start a fresh timeout on dequeue. |

## Default Resource Policy

The following file-processing rules apply to URL operations and sidecars.
The block profile retains destination isolation, authorization and bounded work,
but uses its explicit stored-byte verification and limits instead of MIME
classification, SVG rasterization and whole-resource limits.

| Policy | Level | Rule |
|--------|-------|------|
| Private IP block | MUST | Enforce after DNS resolution and before connection. Re-check every redirect. Block RFC1918, loopback, link-local, ULA, and `169.254.169.254`. |
| MIME sniffing | MUST | Classify bytes by sniffing. Enforce scheme-appropriate allowlists. Never pass upstream `Content-Type` through. |
| SVG rasterization | MUST | Raw `image/svg+xml` MUST NOT be delivered. Rasterize to PNG/WebP in an isolated execution context with no network access. |
| Blossom hash check | MUST | Hash mismatch returns `decode-failed`. |
| Hashtree verification | MUST | Hashtree results verify the resolved root, tree nodes, chunks, and CHK decryption before delivery. Hash/key mismatch returns `decode-failed`. |
| Response size cap | SHOULD | Recommended 10 MiB. Exceed returns `too-large`. |
| Fetch timeout | SHOULD | Recommended 30 s per URL. Exceed returns `timeout`. |
| Concurrency/rate limit | SHOULD | Recommended 10 in-flight and 60 requests/minute per napplet. Bulk counts per URL, not per envelope. |
| Bulk URL cap | SHOULD | Recommended 100 URLs. Exceed returns top-level `resource.bytesMany.error` with `too-large`. |
| Redirect cap | SHOULD | Recommended <= 5 hops. Re-check private IP policy per hop. |
| Blob quota | SHOULD | Recommended 50 MiB outstanding per napplet. Exceed returns `quota-exceeded`. |
| Single-flight cache | SHOULD | Concurrent same-URL requests MAY share one in-flight fetch. |

SVG rasterization SHOULD cap input bytes, output dimensions, and wall-clock
time. Recommended caps: 5 MiB input, 4096 x 4096 output, 2 s raster time.

## Sidecar Pre-Resolution

The runtime MAY hydrate resource cache entries before the napplet asks for them.
The sidecar entry type is owned here. This NAP does not define a carrier field
or enable sidecar delivery on another domain. A carrier MUST separately declare
its optional `resource` wire dependency and import this type before carrying it.
Concrete carrier attachments are deferred to the importing spec changes.

`ResourceSidecarEntry` fields:

| Field | Required | Type |
|-------|----------|------|
| `url` | yes | text |
| `blob` | yes | `ResourceBytes` |
| `mime` | yes | text |

Sidecar rules:

- Sidecars are optional and carrier-policy gated.
- Sidecar `mime` follows the same sniffing rule as `resource.bytes.result`.
- SVG sidecars MUST already be rasterized to PNG/WebP.
- Shims MUST hydrate sidecars before invoking the napplet event handler.
- Sidecars do not change cache scope: the verified manifest and artifact scope still applies.

## URL And Cache Keys

Cache lookup keys are byte-equal URL strings as supplied by the napplet. This
NAP does not require URL canonicalization. Napplets that need deduplication
SHOULD pass canonical URL strings.

## Coexistence

- NAP-IDENTITY profile `picture` and `banner` URLs are fetched through
  `resource.bytes` or `resource.bytesMany`.
- NAP-MEDIA artwork URLs are fetched through `resource.bytes` or
  `resource.bytesMany`.

## Error Codes

| Code | Emitted by | Meaning |
|------|------------|---------|
| `invalid-request` | top-level or item error | Malformed payload, missing field, or empty `urls` is a top-level error. An invalid URL/reference fails only its bulk item, or the single `bytes` request. |
| `not-found` | item or `bytes` error | Resource does not exist. |
| `blocked-by-policy` | item or `bytes` error | Runtime policy rejected the fetch. |
| `timeout` | item or `bytes` error | Fetch or rasterization timeout. |
| `too-large` | item, `bytes` error, or bulk top-level error | Response, rasterization, or bulk URL cap exceeded. |
| `unsupported-scheme` | item or `bytes` error | Scheme is not supported. |
| `decode-failed` | item or `bytes` error | MIME sniff, Blossom hash, or rasterization decode failed. |
| `network-error` | item or `bytes` error | DNS, TCP, TLS, or upstream network failure. |
| `quota-exceeded` | item or `bytes` error | Per-napplet decoded-byte quota exceeded. |

## Security Considerations

- The runtime is an SSRF boundary. DNS-time private-IP checks are mandatory.
- Resource bytes are visible to the host page and browser tooling. They are not
  confidential once delivered through this channel.
- Upstream `Content-Type` is attacker-controlled and MUST NOT be trusted.
- Raw SVG is an active XML surface and MUST be rasterized for URL results and
  sidecars. Block results are the explicit inert-byte exception described above.
- Hashtree fragment and embedded keys are read capabilities. Runtimes MUST keep them
  client-side and MUST NOT forward them to relays, storage servers, FIPS peers,
  or HTTP request bodies.
- Sidecar prefetch can leak user interest to resource hosts. It is optional and
  carrier-policy gated.
- Cache scope comes from runtime-bound napplet identity, never napplet payload.
- `resource.info` can reveal enabled schemes and coarse policy limits. Runtimes
  MAY redact schemes or round limits for untrusted napplets.

## Implementations

- [napplet.soy / soyLI experimental implementation](https://github.com/soytec/napplet-soy/blob/808044555443d6e2dede98d9494f985e0957cb3b/docs/FIPS-RESOURCES.md):
  deployed root/provider grants, verified descendant reads, cancellation and
  scoped diagnostics, used by the Atlas map consumer. This is design and test
  evidence, **not conformance to this draft**: its `soy.fips/1` binding uses
  different envelopes, transferred binary values and inferred root types.
  Browser seed destination enforcement needs separate verification; public URL
  syntax checks alone do not satisfy DNS-time policy. Its application RPC adapter
  is outside this proposal. The draft requires migration and independent-shell
  acceptance.

## Changelog

- `558f6cc` - Introduced NAP-RESOURCE as a runtime-mediated resource fetch surface for https, blossom, nostr, and data URLs with shell-owned policy and sidecar metadata.
- `c876e7b` - Added `bytesMany` for batched resource reads while preserving per-URL policy, MIME, cache, quota, and error handling.
- `23cf33c` - Removed downstream implementation-location guidance so the spec stays implementation-neutral.
- `1c41cb6` - Added Hashtree URL support as a runtime-fetchable resource scheme.
- `8e75ead` - Added `resource.info` scheme introspection for supported schemes, MIME policy, caps, and bulk limits.
- `8c0645d` - Changed resource sidecars to reference relay-owned `RelayEventResult` instead of redefining relay event shape.

- `3dca448` - Adopted injected-domain availability and verified manifest/artifact scopes for all napplet kinds, with publisher isolation.
- `2b61364` - Limited terminal-response requirements to operations with result/error envelopes and exempted cancellation.
- `c5eda99` - Made SVG rasterization isolation host-neutral.
- `c0cffe4` - Required equivalent data-URL policy checks in the shim and runtime.
- `2c8fc73` - Required enforced projection network isolation, standardized base64 wire bytes and web Blob conversion, specified Hashtree reference forms, and deferred concrete sidecar carrier attachments.
- `2a911aa` - Replaced the `blossom:sha256:<hex>` form with BUD-10 `blossom:<sha256>.<ext>` URIs and made BUD-10 discovery hints advisory.
