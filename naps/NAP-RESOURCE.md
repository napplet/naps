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

The napplet supplies URLs. The runtime owns fetch, scheme dispatch, policy,
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
Every wire `blob` field MUST use it: single results, bulk items, and
`ResourceSidecarEntry`. Use the standard alphabet, required `=` padding, zero
pad bits, and no whitespace or data-URL prefix. Empty bytes encode as `""`.
Receivers MUST reject non-canonical encodings. Host byte objects and integer
arrays MUST NOT appear in these wire fields.

The decoded bytes are the complete policy-approved resource, after any SVG
rasterization. Hash verification applies to fetched content before transformation.
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
- Successful results deliver complete byte sequences only. No streaming, chunks, ranges,
  or progress fields are defined by this NAP.
- `message?` is diagnostic only. Programmatic handling uses `error`.
- Unsupported schemes still fail per request with `unsupported-scheme`, even if
  the napplet skipped `resource.info`.

## Schemes

| Scheme | Rules |
|--------|-------|
| `data:` | MAY decode in the napplet shim. Whether decoded in the shim or runtime, it MUST be size-limited, MIME-sniffed, rasterized when SVG, quota-checked, and otherwise policy-checked. No network access. |
| `https:` | Runtime fetch. Full Default Resource Policy applies. Returned `mime` is sniffed, not upstream `Content-Type`. |
| `blossom:` | Canonical form `blossom:sha256:<hex>`. Runtime MUST verify SHA-256 before delivery. Upstream hosts use `https:` policy. |
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

## Default Resource Policy

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
- Raw SVG is an active XML surface and MUST be rasterized before delivery.
- Hashtree fragment and embedded keys are read capabilities. Runtimes MUST keep them
  client-side and MUST NOT forward them to relays, storage servers, FIPS peers,
  or HTTP request bodies.
- Sidecar prefetch can leak user interest to resource hosts. It is optional and
  carrier-policy gated.
- Cache scope comes from runtime-bound napplet identity, never napplet payload.
- `resource.info` can reveal enabled schemes and coarse policy limits. Runtimes
  MAY redact schemes or round limits for untrusted napplets.

## Implementations

- (none yet)

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
