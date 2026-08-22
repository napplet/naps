NAP-RESOURCE
============

Sandboxed Resource Fetching
---------------------------

`draft`

**NAP ID:** NAP-RESOURCE
**Domain:** `resource`
**Web binding (NIP-5D):** `window.napplet.resource` · `shell.supports("resource")`
**Parent:** NIP-5D

## Description

NAP-RESOURCE lets a napplet request byte resources through the runtime:

- `resource.bytes(url, options?) -> Blob`
- `resource.bytesMany(requests, options?) -> list of ResourceBytesItem`

The napplet supplies URLs. The runtime owns fetch, scheme dispatch, policy,
MIME classification, SVG rasterization, caching, quotas, and errors. Napplets
MUST NOT receive raw network access or upstream `Content-Type`.

NAP-RESOURCE is the fetch path for the NIP-5D web sandbox. The contract is
projection-neutral; `Blob` is the web projection result type.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `info` | none | `ResourceInfo` | `resource.info` |
| `bytes` | `url`, optional `servers` | one Blob result | `resource.bytes` |
| `bytesMany` | non-empty list of `ResourceBytesRequest` | ordered per-request results | `resource.bytesMany` |
| `bytesAsObjectURL` | `url` | `{ url, revoke }` helper | helper over `bytes` |

`info()` is optional introspection. Napplets MAY call it to adapt UI or choose
supported URL forms, but MUST NOT be required to call it before `bytes` or
`bytesMany`.

The web projection places `servers` in the second argument to `bytes`. Each
`bytesMany` entry carries its own `servers`; no batch-wide server list exists:

```js
resource.bytes("blossom:sha256:<hash>", {
  servers: ["https://cdn.hzrd149.com", "https://blossom.primal.net"],
});

resource.bytesMany([
  {
    url: "blossom:sha256:<first-hash>",
    servers: ["https://cdn.hzrd149.com"],
  },
  {
    url: "blossom:sha256:<second-hash>",
    servers: ["https://blossom.primal.net"],
  },
]);
```

The web projection MAY also expose `options.signal` to abort `bytes` or
`bytesMany`. Abort sends `resource.cancel` for the request `id`. Late terminal
envelopes for cancelled IDs MUST be dropped.

`bytesMany` reduces envelope count only. Each entry MUST be processed as if it
were an independent `bytes(request.url, { servers: request.servers })` request.
One failed entry MUST NOT discard successful siblings.

All resource state is scoped to the napplet's `(dTag, aggregateHash)` identity.
A napplet MUST NOT read another napplet's resource cache.

## Wire Protocol

`resource.*` messages use NIP-5D wire format:
`{ "type": "domain.action", ...payload }`.

| Type | Direction | Payload |
|------|-----------|---------|
| `resource.info` | napplet -> runtime | `id` |
| `resource.info.result` | runtime -> napplet | `id`, `info` |
| `resource.info.error` | runtime -> napplet | `id`, `error`, `message?` |
| `resource.bytes` | napplet -> runtime | `id`, `url`, `servers?` |
| `resource.bytesMany` | napplet -> runtime | `id`, `requests` |
| `resource.cancel` | napplet -> runtime | `id` |
| `resource.bytes.result` | runtime -> napplet | `id`, `blob`, `mime` |
| `resource.bytes.error` | runtime -> napplet | `id`, `error`, `message?` |
| `resource.bytesMany.result` | runtime -> napplet | `id`, `items` |
| `resource.bytesMany.error` | runtime -> napplet | `id`, `error`, `message?` |

Single request with location hints:

```json
{
  "type": "resource.bytes",
  "id": "resource-1",
  "url": "blossom:sha256:<hash>",
  "servers": ["https://cdn.hzrd149.com"]
}
```

Batch request with per-resource hints:

```json
{
  "type": "resource.bytesMany",
  "id": "resource-2",
  "requests": [
    {
      "url": "blossom:sha256:<first-hash>",
      "servers": ["https://cdn.hzrd149.com"]
    },
    {
      "url": "blossom:sha256:<second-hash>",
      "servers": ["https://blossom.primal.net"]
    }
  ]
}
```

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
| `maxServers` | no | unsigned integer |

`ResourceBytesRequest` fields:

| Field | Required | Type |
|-------|----------|------|
| `url` | yes | text |
| `servers` | no | list of text |

`ResourceBytesItem` fields:

| Field | Required | Type |
|-------|----------|------|
| `url` | yes | text |
| `ok` | yes | boolean |
| `blob` | no | bytes |
| `mime` | no | text |
| `error` | no | text |
| `message` | no | text |

Rules:

- Every request gets one terminal result or error envelope.
- `resource.info` is advisory and MUST NOT be a required preflight.
- `bytesMany.result.items` MUST preserve request order and length.
- `ok: true` items MUST include `blob` and `mime`.
- `ok: false` items MUST include `error` and MUST NOT include `blob`.
- `mime` MUST be runtime-classified by byte sniffing, never upstream header.
- Successful results deliver complete Blobs only. No streaming, chunks, ranges,
  or progress fields are defined by this NAP.
- `message?` is diagnostic only. Programmatic handling uses `error`.
- Unsupported schemes still fail per request with `unsupported-scheme`, even if
  the napplet skipped `resource.info`.

## Schemes

| Scheme | Rules |
|--------|-------|
| `data:` | MAY decode in the napplet shim. If sent to the runtime, it MUST be decoded and policy-checked. No network access. |
| `https:` | Runtime fetch. Full Default Resource Policy applies. Returned `mime` is sniffed, not upstream `Content-Type`. |
| `blossom:` | Canonical form `blossom:sha256:<hex>`. Runtime MUST verify SHA-256 before delivery. Advisory servers are request metadata, not part of the URL. Upstream hosts use `https:` policy. |
| `htree:` | Hashtree reference (`htree://...`, `nhash`, or compatible immutable form). Runtime resolves the referenced file bytes, verifies every Hashtree hash before delivery, and MUST NOT leak fragment keys to relays, storage servers, or peers. |
| `nostr:` | NIP-19 bech32. Runtime resolves one hop and returns the referenced bytes. MUST NOT recursively follow URLs or `nostr:` references in the result. |

Unknown schemes MUST return `unsupported-scheme`. `http:` is not canonical and
MUST NOT be enabled by default.

`resource.info.schemes` reports schemes the runtime is willing to disclose to
the napplet. It does not grant fetch authority. Each `bytes` or `bytesMany` URL
still passes through scheme dispatch and policy checks.

### Blossom Location Hints

`servers` is an ordered, advisory list of locations for a `blossom:` resource.
It has no meaning for other schemes and MUST be ignored for them. Advisory means
the list cannot force network access, bypass a cache hit, or override runtime or
user policy.

After a cache miss, a runtime MUST resolve a `blossom:` resource in this order:

1. Accepted request `servers`, in supplied order.
2. Runtime or user default servers, in configured order.
3. Public fallback servers selected by the runtime.

Empty tiers are skipped.

Each server entry MUST be an HTTPS public origin: scheme, host, and optional
port only, with no credentials, query, or fragment. A trailing `/` is allowed.
The runtime MUST discard invalid or disallowed entries, deduplicate equivalent
origins while preserving first occurrence, and consider only the first entries
up to a finite per-resource cap. `resource.info.maxServers` MAY disclose that
cap. Every accepted origin still passes the Default Resource Policy, including
DNS-time private-IP checks, redirect checks, timeouts, quotas, and rate limits.

For each server, HTTP `404` and `410` are definitive misses and MUST continue to
the next server. If all reachable servers return a definitive miss and no
attempt is inconclusive, the runtime MUST return `not-found`. If no server
succeeds and a DNS, TCP, TLS, or upstream transport failure leaves the result
inconclusive, the runtime MUST return `network-error` unless a more specific
defined error applies.

The runtime MUST verify the requested SHA-256 for every candidate body before
caching or delivery. A mismatched body MUST NOT be cached or delivered and
returns `decode-failed`.

## Default Resource Policy

| Policy | Level | Rule |
|--------|-------|------|
| Private IP block | MUST | Enforce after DNS resolution and before connection. Re-check every redirect. Block RFC1918, loopback, link-local, ULA, and `169.254.169.254`. |
| MIME sniffing | MUST | Classify bytes by sniffing. Enforce scheme-appropriate allowlists. Never pass upstream `Content-Type` through. |
| SVG rasterization | MUST | Raw `image/svg+xml` MUST NOT be delivered. Rasterize to PNG/WebP in a no-network sandboxed Worker. |
| Blossom hash check | MUST | Hash mismatch returns `decode-failed`. |
| Hashtree verification | MUST | `htree:` results verify the resolved root, tree nodes, chunks, and CHK decryption before delivery. Hash/key mismatch returns `decode-failed`. |
| Blossom server hints | MUST | Accept only public HTTPS origins. Deduplicate and cap entries. Apply the full network policy to every attempt. |
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
The sidecar entry type is owned here. The carrier field is owned by the carrier
domain, such as `RelayEventResult.sidecar.resources?` in NAP-RELAY.

`ResourceSidecarEntry` fields:

| Field | Required | Type |
|-------|----------|------|
| `url` | yes | text |
| `blob` | yes | bytes |
| `mime` | yes | text |

Sidecar rules:

- Sidecars are optional and carrier-policy gated.
- Sidecar `mime` follows the same sniffing rule as `resource.bytes.result`.
- SVG sidecars MUST already be rasterized to PNG/WebP.
- Shims MUST hydrate sidecars before invoking the napplet event handler.
- Sidecars do not change cache scope: `(dTag, aggregateHash)` still applies.

## URL And Cache Keys

Cache lookup keys are byte-equal URL strings as supplied by the napplet. This
NAP does not require URL canonicalization. Napplets that need deduplication
SHOULD pass canonical URL strings.

`servers` MUST NOT participate in the cache key. Requests for the same canonical
`blossom:sha256:<hex>` URL use the same cache entry even when their hints differ.
Napplets and projection shims MUST carry server hints in `servers`, not append
[BUD-10 `xs` discovery parameters](https://github.com/hzrd149/blossom/blob/master/buds/10.md)
to the resource URL.

## Coexistence

- NAP-RELAY MAY carry `ResourceSidecarEntry[]` on
  `RelayEventResult.sidecar.resources?`. NAP-RELAY owns that field and its
  default-off privacy policy.
- NAP-IDENTITY profile `picture` and `banner` URLs are fetched through
  `resource.bytes` or `resource.bytesMany`.
- NAP-MEDIA artwork URLs are fetched through `resource.bytes` or
  `resource.bytesMany`.

## Error Codes

| Code | Emitted by | Meaning |
|------|------------|---------|
| `invalid-request` | top-level errors | Malformed payload, missing field, or empty `requests`. |
| `not-found` | item or `bytes` error | Resource does not exist. |
| `blocked-by-policy` | item or `bytes` error | Runtime policy rejected the fetch. |
| `timeout` | item or `bytes` error | Fetch or rasterization timeout. |
| `too-large` | item, `bytes` error, or bulk top-level error | Response, rasterization, or bulk URL cap exceeded. |
| `unsupported-scheme` | item or `bytes` error | Scheme is not supported. |
| `decode-failed` | item or `bytes` error | MIME sniff, Blossom hash, or rasterization decode failed. |
| `network-error` | item or `bytes` error | DNS, TCP, TLS, or upstream network failure. |
| `quota-exceeded` | item or `bytes` error | Per-napplet Blob quota exceeded. |

## Security Considerations

- The runtime is an SSRF boundary. DNS-time private-IP checks are mandatory.
- Resource bytes are visible to the host page and browser tooling. They are not
  confidential once delivered through this channel.
- Upstream `Content-Type` is attacker-controlled and MUST NOT be trusted.
- Raw SVG is an active XML surface and MUST be rasterized before delivery.
- `htree:` fragment keys are read capabilities. Runtimes MUST keep them
  client-side and MUST NOT forward them to relays, storage servers, FIPS peers,
  or HTTP request bodies.
- Sidecar prefetch can leak user interest to resource hosts. It is optional and
  carrier-policy gated.
- Cache scope comes from runtime-bound napplet identity, never napplet payload.
- Blossom server hints are untrusted input. They MUST NOT weaken SSRF policy or
  disclose whether a blocked origin exists.
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
- `7531258` - Added per-resource Blossom server hints with ordered fallback, runtime-owned network policy, hash verification, miss handling, and URL-only cache identity.
