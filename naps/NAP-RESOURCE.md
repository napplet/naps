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

- `resource.bytes(url) -> Blob`
- `resource.bytesMany(urls) -> ResourceBytesItem[]`

The napplet supplies URLs. The runtime owns fetch, scheme dispatch, policy,
MIME classification, SVG rasterization, caching, quotas, and errors. Napplets
MUST NOT receive raw network access or upstream `Content-Type`.

NAP-RESOURCE is the fetch path for the NIP-5D web sandbox. The contract is
projection-neutral; `Blob` is the web projection result type.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `bytes` | `url`, `opts?` | one Blob result | `resource.bytes` |
| `bytesMany` | non-empty `urls`, `opts?` | ordered per-URL results | `resource.bytesMany` |
| `bytesAsObjectURL` | `url` | `{ url, revoke }` helper | helper over `bytes` |

`opts.signal` MAY abort `bytes` or `bytesMany`. Abort sends `resource.cancel`
for the request `id`. Late terminal envelopes for cancelled IDs MUST be dropped.

`bytesMany` reduces envelope count only. Each URL MUST be processed as if it
were an independent `bytes(url)` request. One failed URL MUST NOT discard
successful siblings.

All resource state is scoped to the napplet's `(dTag, aggregateHash)` identity.
A napplet MUST NOT read another napplet's resource cache.

## Wire Protocol

`resource.*` messages use NIP-5D wire format:
`{ "type": "domain.action", ...payload }`.

| Type | Direction | Payload |
|------|-----------|---------|
| `resource.bytes` | napplet -> runtime | `id`, `url` |
| `resource.bytesMany` | napplet -> runtime | `id`, `urls` |
| `resource.cancel` | napplet -> runtime | `id` |
| `resource.bytes.result` | runtime -> napplet | `id`, `blob`, `mime` |
| `resource.bytes.error` | runtime -> napplet | `id`, `error`, `message?` |
| `resource.bytesMany.result` | runtime -> napplet | `id`, `items` |
| `resource.bytesMany.error` | runtime -> napplet | `id`, `error`, `message?` |

```cddl
ResourceBytesItem = {
  url: tstr,
  ok: bool,
  ? blob: bstr,
  ? mime: tstr,
  ? error: tstr,
  ? message: tstr,
}
```

Rules:

- Every request gets one terminal result or error envelope.
- `bytesMany.result.items` MUST preserve input order and length.
- `ok: true` items MUST include `blob` and `mime`.
- `ok: false` items MUST include `error` and MUST NOT include `blob`.
- `mime` MUST be runtime-classified by byte sniffing, never upstream header.
- Successful results deliver complete Blobs only. No streaming, chunks, ranges,
  or progress fields are defined by this NAP.
- `message?` is diagnostic only. Programmatic handling uses `error`.

## Schemes

| Scheme | Rules |
|--------|-------|
| `data:` | MAY decode in the napplet shim. If sent to the runtime, it MUST be decoded and policy-checked. No network access. |
| `https:` | Runtime fetch. Full Default Resource Policy applies. Returned `mime` is sniffed, not upstream `Content-Type`. |
| `blossom:` | Canonical form `blossom:sha256:<hex>`. Runtime MUST verify SHA-256 before delivery. Upstream hosts use `https:` policy. |
| `nostr:` | NIP-19 bech32. Runtime resolves one hop and returns the referenced bytes. MUST NOT recursively follow URLs or `nostr:` references in the result. |

Unknown schemes MUST return `unsupported-scheme`. `http:` is not canonical and
MUST NOT be enabled by default.

## Default Resource Policy

| Policy | Level | Rule |
|--------|-------|------|
| Private IP block | MUST | Enforce after DNS resolution and before connection. Re-check every redirect. Block RFC1918, loopback, link-local, ULA, and `169.254.169.254`. |
| MIME sniffing | MUST | Classify bytes by sniffing. Enforce scheme-appropriate allowlists. Never pass upstream `Content-Type` through. |
| SVG rasterization | MUST | Raw `image/svg+xml` MUST NOT be delivered. Rasterize to PNG/WebP in a no-network sandboxed Worker. |
| Blossom hash check | MUST | Hash mismatch returns `decode-failed`. |
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
domain, such as `relay.event.resources?`.

```cddl
ResourceSidecarEntry = {
  url: tstr,
  blob: bstr,
  mime: tstr,
}
```

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

## Coexistence

- NAP-RELAY MAY carry `ResourceSidecarEntry[]` on `relay.event.resources?`.
  NAP-RELAY owns that field and its default-off privacy policy.
- NAP-IDENTITY profile `picture` and `banner` URLs are fetched through
  `resource.bytes` or `resource.bytesMany`.
- NAP-MEDIA artwork URLs are fetched through `resource.bytes` or
  `resource.bytesMany`.

## Error Codes

| Code | Emitted by | Meaning |
|------|------------|---------|
| `invalid-request` | top-level errors | Malformed payload, missing field, or empty `urls`. |
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
- Sidecar prefetch can leak user interest to resource hosts. It is optional and
  carrier-policy gated.
- Cache scope comes from runtime-bound napplet identity, never napplet payload.

## Implementations

- (none yet)
