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

NAP-RESOURCE provides sandboxed napplets with scheme-pluggable primitives for fetching byte-addressable resources through the host shell: `resource.bytes(url) → Blob` and `resource.bytesMany(urls) → ResourceBytesItem[]`. Napplets address resources by URL only -- they never see a raw `fetch`, never open a socket, and never observe the upstream `Content-Type`. The shell mediates each fetch under a documented policy (private-IP block list at DNS-resolution time, MIME byte-sniffing, response size cap, fetch timeout, per-napplet rate limit, redirect chain cap), classifies the returned bytes by sniffing, optionally rasterizes vector formats to bitmap before delivery, and returns `Blob` payloads plus shell-classified `mime` strings.

The URL space is scheme-pluggable. Four canonical schemes are defined in this NAP: `https:` (shell-side network fetch with policy enforcement), `blossom:` (Blossom hash → bytes resolution), `nostr:` (NIP-19 bech32 single-hop resolution), and `data:` (RFC 2397, decoded inline by the napplet shim). Shells MAY support additional schemes behind explicit shell-administrator policy, but unknown schemes MUST be rejected with `unsupported-scheme`.

This NAP depends on the iframe sandbox model defined by NIP-5D (`sandbox="allow-scripts"`, no `allow-same-origin`) preventing direct network access. NAP-RESOURCE is the canonical fetch path that complements that browser-enforced isolation: napplets cannot fetch directly because the browser blocks it; everything network-sourced flows through the host shell over `postMessage`.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `bytes` | `url` (`tstr`), optional abort signal | byte payload (`Blob` in the web projection) | `resource.bytes` / `resource.bytes.result` or `resource.bytes.error` |
| `bytesMany` | `urls` (`[+ tstr]`), optional abort signal | ordered `ResourceBytesItem[]` | `resource.bytesMany` / `resource.bytesMany.result` or `resource.bytesMany.error` |
| `bytesAsObjectURL` | `url` (`tstr`) | object URL handle with `url` and `revoke` | helper over `bytes` |

**`bytes(url, opts?)`** -- Sends a `resource.bytes` envelope to the host shell with the supplied URL and a freshly minted correlation `id`. Resolves with the `Blob` delivered by the shell on `resource.bytes.result`, or rejects with an error whose code is one of the enumerated Error Codes below. When `opts.signal` is supplied, an already-aborted signal MUST cause the returned Promise to reject synchronously with an `AbortError` and SHOULD cause a `resource.cancel` envelope to be sent so the shell can release any in-flight fetch resources. Aborts after dispatch send `resource.cancel` to the shell and reject the awaiting napplet's Promise locally; the shell MAY emit no further envelopes for that `id`.

The `data:` scheme MAY be decoded entirely inside the napplet shim with zero shell round-trip; it is the only scheme where the result envelope is synthesized client-side. All other schemes traverse `postMessage` to the shell.

**`bytesMany(urls, opts?)`** -- Sends one `resource.bytesMany` envelope for a non-empty URL list. The result preserves input order and contains one item per requested URL. Each item succeeds with `blob` and `mime`, or fails with an `error` code. The shell MUST apply the same scheme, policy, MIME, rasterization, size, timeout, cache, and quota rules that would apply to independent `bytes(url)` calls. Bulk requests reduce envelope count; they do not widen what a napplet may fetch.

Invalid bulk requests reject as `resource.bytesMany.error`. Per-URL fetch failures are reported inside `resource.bytesMany.result` so one failed URL does not discard successful siblings. A bulk abort uses `resource.cancel` with the bulk request `id` and cancels every in-flight item for that request.

**`bytesAsObjectURL(url)`** -- Synchronous helper for napplets that need a stable object URL handle (e.g., to set on an `<img src>` attribute) without awaiting the underlying fetch. Returns `{ url, revoke }` immediately; the `url` field is populated with `URL.createObjectURL(blob)` once the underlying `bytes(url)` resolves. The napplet MUST call `revoke()` when the resource is no longer needed -- the napplet owns the object URL lifetime, not the shell. Multiple `revoke()` calls are idempotent. Calling `revoke()` before the underlying fetch resolves cancels the cache write so the eventual Blob is not retained against an already-dead object URL.

All methods are scoped to the napplet's `(dTag, aggregateHash)` identity per NIP-5D. Napplets cannot read another napplet's cached resources; the shell-side cache is partitioned by source identity.

## Wire Protocol

`resource.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `resource.bytes`        | napplet -> shell | `id`, `url`              |
| `resource.bytesMany`    | napplet -> shell | `id`, `urls`             |
| `resource.cancel`       | napplet -> shell | `id`                     |
| `resource.bytes.result` | shell -> napplet | `id`, `blob`, `mime`     |
| `resource.bytes.error`  | shell -> napplet | `id`, `error`, `message?` |
| `resource.bytesMany.result` | shell -> napplet | `id`, `items` |
| `resource.bytesMany.error`  | shell -> napplet | `id`, `error`, `message?` |

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

Key design notes:

- All correlation uses `id`. Each `resource.bytes` request gets exactly one terminal envelope: either `resource.bytes.result` or `resource.bytes.error`.
- Each `resource.bytesMany` request gets exactly one terminal envelope: either `resource.bytesMany.result` or `resource.bytesMany.error`.
- `resource.bytesMany.result.items` MUST preserve `urls` order and MUST contain exactly one `ResourceBytesItem` per input URL.
- A successful bulk item (`ok: true`) MUST include `blob` and `mime`. A failed bulk item (`ok: false`) MUST include `error` and MUST NOT include `blob`.
- The `mime` field on `resource.bytes.result` is **shell-classified by byte-sniffing** -- never passed through from the upstream `Content-Type` header. See Default Resource Policy § MIME byte-sniffing for the rationale.
- `resource.cancel` is fire-and-forget; shells SHOULD release any in-flight fetch resources for the correlated `id` and SHOULD NOT emit a terminal envelope for that `id` after receiving the cancel. Late-arriving result envelopes for cancelled `id`s MUST be silently dropped by conformant napplet shims.
- **Blob delivery contract**: each successful `resource.bytes` result or `resource.bytesMany` item delivers a complete `Blob` -- there is no streaming, chunking, range, or progress field anywhere in this NAP. Streaming media (audio, video) is reserved for a future NAP with a different delivery model.
- The `message?` field on errors is a human-readable string for diagnostics; programmatic dispatch MUST be driven by the `error` field, never by string-matching `message`.

### Examples

**Successful `data:` fetch (decoded in-shim, no shell round-trip):**
```
napplet: napplet.resource.bytes('data:image/png;base64,iVBORw0KGgo...')
            // resolves with a Blob; no envelope is sent.
```

**Successful `https:` fetch:**
```
-> { "type": "resource.bytes", "id": "r1", "url": "https://example.com/avatar.png" }
<- { "type": "resource.bytes.result", "id": "r1", "blob": <Blob 4321 bytes>, "mime": "image/png" }
```

**Successful `blossom:` fetch:**
```
-> { "type": "resource.bytes", "id": "r2", "url": "blossom:sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855" }
<- { "type": "resource.bytes.result", "id": "r2", "blob": <Blob 9001 bytes>, "mime": "image/webp" }
```

**Error response (private-IP rejection at DNS-resolution time):**
```
-> { "type": "resource.bytes", "id": "r3", "url": "https://internal.local/secret" }
<- { "type": "resource.bytes.error", "id": "r3", "error": "blocked-by-policy", "message": "resolved IP 10.0.0.5 is in the RFC1918 private range" }
```

**Cancellation (napplet aborts an in-flight fetch):**
```
-> { "type": "resource.bytes", "id": "r4", "url": "https://example.com/large.bin" }
-> { "type": "resource.cancel", "id": "r4" }
   // No further envelopes expected for id "r4".
```

**Bulk image fetch (single envelope pair):**
```
-> { "type": "resource.bytesMany", "id": "r5", "urls": ["https://example.com/a.png", "https://example.com/b.svg"] }
<- { "type": "resource.bytesMany.result", "id": "r5", "items": [
     { "url": "https://example.com/a.png", "ok": true, "blob": <Blob 1200 bytes>, "mime": "image/png" },
     { "url": "https://example.com/b.svg", "ok": false, "error": "blocked-by-policy" }
   ] }
```

## Scheme Protocol Surfaces

NAP-RESOURCE defines four canonical schemes. Shells MUST implement scheme dispatch as a **whitelist**: only the canonical schemes (and any explicitly opt-in shell-extension schemes) MAY produce a fetch attempt. Smuggling-prone schemes (`file:`, `gopher:`, `dict:`, `ftp:`, `tftp:`, etc.) MUST NOT be supported under any default policy. Unknown schemes MUST result in `error: "unsupported-scheme"`.

### `data:`

RFC 2397 per normal. The napplet shim MAY decode `data:` URLs locally without a shell round-trip; conformant shells that nonetheless receive a `resource.bytes` envelope with a `data:` URL MUST also handle it (decode and return the `Blob` with the data URL's declared MIME, byte-sniff-validated against scheme-appropriate allowlist). No network access is performed for `data:` URLs. No SSRF policy applies. The Default Resource Policy size cap on the decoded `Blob` still applies.

### `https:`

The shell performs the network fetch. This scheme is subject to the **full Default Resource Policy** below: the private-IP block list MUST be enforced at DNS-resolution time, MIME byte-sniffing MUST be applied, response size cap SHOULD be enforced, fetch timeout SHOULD be enforced, per-napplet concurrency / rate limit SHOULD be enforced, and redirect chain cap SHOULD be enforced with per-hop re-validation. The `mime` returned to the napplet is the shell's byte-sniffed classification -- never the upstream `Content-Type`.

`http:` (cleartext) is NOT a canonical scheme and MUST NOT be enabled by default. Shells MAY opt in to `http:` behind explicit shell-administrator policy (e.g., enterprise on-prem services), but the default for community-deployed shells MUST NOT include `http:`.

### `blossom:`

Blossom hash → bytes resolution. Canonical URL form: `blossom:sha256:<hex>` where `<hex>` is the lowercase 64-character SHA-256 hex digest of the resource bytes. Shells MAY consult any number of Blossom servers per their server-selection policy (typically a user-configured list of preferred Blossom hosts). The shell MUST verify the returned bytes hash against the URL's declared digest **before** delivering them to the napplet; mismatch MUST result in `error: "decode-failed"`. The `mime` field is byte-sniffed by the shell after hash verification, not before.

When fetching from upstream Blossom hosts, shells MUST apply the same `https:` policy (private-IP block list at DNS-resolution time, etc.) to each candidate host. Blossom-host selection MUST NOT bypass DNS-pinning.

### `nostr:`

NIP-19 bech32 input form (e.g., `nostr:nprofile1...`, `nostr:naddr1...`, `nostr:nevent1...`). The shell decodes the bech32 to its referenced Nostr resource (profile metadata, addressable event content, event JSON) and returns those bytes as a `Blob` -- typically with `mime: "application/json"`, or another shell-classified MIME if the resolved resource is byte-typed.

**Single-hop resolution semantics**: the shell resolves the bech32 to its immediately-referenced Nostr resource and returns those bytes. Shells MUST NOT recursively follow URLs, hashtags, or `nostr:` references found *within* the resolved bytes -- recursive resolution is the napplet's job via subsequent `resource.bytes()` calls. Single-hop semantics prevent shell-side fan-out attacks, recursive amplification, and deeply nested traversal-cost surprises.

If the bech32 cannot be decoded or the referenced resource cannot be found in the shell's relay pool, the result MUST be `error: "not-found"`.

## Default Resource Policy

The shell is the irreducible attack surface for the resource NAP. Conformant shells MUST enforce the following defaults; deviations are a deployment policy decision and SHOULD be surfaced in the shell's documentation. These defaults apply to the `https:` scheme and to upstream Blossom-host fetches (`blossom:`); they do not apply to `data:` (no network) or to `nostr:` resolution against the shell's own relay pool.

### Private-IP block list (MUST, at DNS-resolution time)

Shells implementing the `https:` scheme handler MUST, **at DNS resolution time** (not URL-parse time), reject URLs whose resolved IP falls in any of the following ranges. The check MUST happen after the DNS resolver returns an address and BEFORE the HTTP connection is opened (DNS pinning); each redirect MUST be re-validated independently.

| Range | RFC / source |
|-------|--------------|
| `10/8`, `172.16/12`, `192.168/16` | RFC1918 private IPv4 |
| `127/8`, `::1` | Loopback |
| `169.254/16`, `fe80::/10` | Link-local |
| `fc00::/7` | Unique-local IPv6 |
| `169.254.169.254` | Cloud metadata service (AWS, GCP, Azure, DigitalOcean, etc.) |

Rejection MUST emit `error: "blocked-by-policy"`. The `message` string SHOULD identify which range matched so deployers can debug policy. Shells MAY allow additional addresses behind explicit shell-administrator policy (e.g., enterprise on-prem services), but the default for community-deployed shells MUST be restrictive.

URL-parse-time checks (e.g., looking at the literal hostname in the URL) are NOT sufficient. An attacker-controlled DNS record can resolve `attacker.example.com` to `127.0.0.1` and bypass any check that does not actually inspect the resolved address. The DNS pinning requirement here exists specifically to defeat DNS-rebinding and TOCTOU attacks.

### MIME byte-sniffing (MUST)

Shells MUST classify the returned bytes via a byte-sniffing strategy (e.g., the [WHATWG MIME Sniffing Standard](https://mimesniff.spec.whatwg.org/) or equivalent). The upstream `Content-Type` header MUST NOT be passed through to the napplet -- that header is attacker-controlled, and the entire trust model breaks if a napplet renders `text/html` because an attacker said so. Shells MUST enforce a scheme-appropriate MIME allowlist (e.g., for image rendering: `image/png`, `image/jpeg`, `image/webp`, `image/gif`, plus `image/svg+xml` subject to SVG handling below). Bytes whose sniffed MIME falls outside the allowlist MUST be rejected with `error: "blocked-by-policy"` rather than delivered as an unknown MIME.

### Response size cap (SHOULD, recommended ~10 MiB)

Shells SHOULD cap individual response bodies at a reasonable upper bound; recommended default is 10 MiB. Excess MUST result in `error: "too-large"`. The cap MAY be configurable per-shell deployment; community-deployed shells SHOULD NOT raise the cap above ~50 MiB without explicit operator opt-in.

### Fetch timeout (SHOULD, recommended ~30s)

Shells SHOULD enforce a per-URL fetch timeout; recommended default is 30 seconds, measured wall-clock from request dispatch to last byte received. Timeout MUST result in `error: "timeout"`. The timeout MAY be configurable per-shell deployment.

### Per-napplet concurrency + rate limit (SHOULD)

Shells SHOULD enforce a per-napplet limit on concurrent in-flight `resource.bytes` calls (recommended: 10) and a per-napplet rate limit (recommended: 60 calls per minute, sliding window). Rate-limit and concurrency-limit rejections MUST emit `error: "blocked-by-policy"` with a `message` string identifying which limit was hit.

Bulk requests count per URL, not per envelope. A `resource.bytesMany` request with 20 URLs consumes 20 rate-limit units and 20 concurrency slots unless the shell coalesces duplicates from cache.

### Bulk request size cap (SHOULD, recommended ≤ 100 URLs)

Shells SHOULD cap `resource.bytesMany.urls` length. Recommended default is 100 URLs per bulk request. Excess MUST result in `resource.bytesMany.error` with `error: "too-large"`.

### Redirect chain cap (SHOULD, recommended ≤ 5)

Shells SHOULD cap the redirect chain at 5 hops. **Each hop MUST be re-validated against the private-IP block list** (DNS pinning per hop). Excess hops or a redirect to a blocked address MUST result in `error: "blocked-by-policy"`. This per-hop re-validation is the mitigation for redirect-amplification attacks where a public host 302s to an internal address.

## SVG Handling

Vector graphics carry an entire scriptable XML execution surface (`<script>`, `<foreignObject>`, `<image href>` external references, `<use href>` recursion, DOCTYPE entity expansion). Delivering raw SVG bytes to a sandboxed napplet recreates every attack surface the sandbox was designed to eliminate. NAP-RESOURCE addresses this by mandating shell-side rasterization.

### Shell-side rasterization (MUST)

When the shell's MIME byte-sniffer identifies a fetched resource as `image/svg+xml`, the shell MUST rasterize it to a bitmap format (PNG or WebP) before delivering the result envelope to the napplet. **Shells MUST NOT deliver raw `image/svg+xml` bytes to napplets.** The `mime` field on the result envelope for any SVG-source-input MUST be `image/png` or `image/webp` -- never `image/svg+xml`.

Rationale: SVG `<foreignObject>` (which can host HTML and script), `<script>` elements, `<image href>` external references, `<use href>` recursion, and XML DOCTYPE entity expansion (the "billion laughs" pattern) are documented attack surfaces. Rasterization to a flat bitmap eliminates the entire class -- the napplet receives pixels, not parseable XML.

### Rasterization isolation (MUST)

Rasterizers MUST run in a **sandboxed Worker** with **no network** access. The rasterizer MUST NOT issue any network request, including SVG-internal `<image href>` URLs, `@font-face` `src:` URLs, `<use href>` external references, or DOCTYPE entity URLs. Any SVG-internal reference that would require network access MUST resolve to local-only data or be omitted from the rasterized output. The Worker MUST NOT have access to `XMLHttpRequest`, `fetch`, `WebSocket`, `EventSource`, or any other network primitive; conformant implementations typically achieve this via a Worker with a CSP that sets `connect-src 'none'` and by stripping network-capable globals before evaluating the SVG.

### Rasterization caps (SHOULD)

| Cap | Recommended default | On exceed |
|-----|---------------------|-----------|
| Max input bytes | 5 MiB | `error: "too-large"` |
| Max output dimensions | 4096 × 4096 pixels | `error: "too-large"` |
| Wall-clock rasterization budget | 2 s | `error: "timeout"` |

These caps mitigate the billion-laughs entity-expansion attack (input is bounded), recursive-`<use>` rendering bombs (output dimensions are bounded), and `<foreignObject>` script-driven CPU exhaustion (wall-clock budget bounds the worst case). All three caps SHOULD be enforced together; relaxing any one undermines the others.

## Sidecar Pre-Resolution

A shell MAY deliver resources it has already fetched as **sidecar entries** alongside a napplet-bound event, so a later `resource.bytes(url)` call resolves from cache without a `postMessage` round-trip. This NAP owns the **entry type** (`ResourceSidecarEntry`) and the **hydration contract** below. The *delivery envelope* that carries the entries — and the policy deciding whether they are sent at all — is owned by the **carrier domain**: `relay` carries them on `relay.event.resources?` (see NAP-RELAY for that field, its **default-OFF** policy, and the per-event-kind allowlist). This NAP defines the type, never the carrier's wire:

```cddl
ResourceSidecarEntry = {
  url: tstr,  ; canonical URL form for this resource
  blob: bstr, ; pre-fetched bytes; Blob in the web projection
  mime: tstr, ; shell-classified by byte-sniffing -- NEVER upstream Content-Type
}
```

When a carrier envelope arrives with a non-empty `resources` array, conformant napplet shims MUST hydrate their `resource.bytes` cache from the entries BEFORE invoking the napplet's event handler. This ordering is load-bearing: it allows a synchronous `napplet.resource.bytes(sidecarUrl)` call inside the napplet's event handler to resolve from cache without a `postMessage` round-trip. See the carrier domain (e.g. NAP-RELAY) for the wire-level field and its **default-OFF** privacy rationale -- pre-resolution is opt-in per shell policy, not on by default.

The `mime` field on each sidecar entry MUST be shell-classified by byte-sniffing per the same rules as `resource.bytes.result`. Shells MUST NOT populate sidecar `mime` from the upstream `Content-Type` header. SVG entries appearing in a sidecar MUST be rasterized to PNG/WebP per the SVG Handling rules before being placed on the wire -- the sidecar is not a bypass for the rasterization MUST.

## Canonical URL Form

Cache lookups, sidecar hydration, and single-flight coalescing all key on the URL string as supplied by the napplet. This NAP does NOT mandate URL canonicalization across calls -- two URLs that differ only in case, percent-encoding, default port, or trailing slash MAY resolve to two distinct cache entries. Napplets that want guaranteed deduplication SHOULD pass an already-canonical URL form. Future revisions of this NAP MAY tighten this guarantee; for v1, byte-equal URL strings are the cache key.

Shells SHOULD treat the URL string opaquely after scheme dispatch -- the parsed scheme drives handler selection, but the rest of the URL is forwarded to the scheme handler intact. This avoids surprising napplet authors with shell-side URL rewriting.

## Shell Guarantees

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY in this document are to be interpreted as described in RFC 2119.

### MUST

| Behavior | Details |
|----------|---------|
| Whitelist scheme dispatch | Only the canonical schemes (`https:`, `blossom:`, `nostr:`, `data:`) plus shell-extension schemes the shell explicitly opts in to. Unknown schemes MUST return `error: "unsupported-scheme"`. |
| Enforce private-IP block list at DNS-resolution time | Per the Default Resource Policy. Each redirect hop re-validated independently. |
| Byte-sniff MIME | Never honor upstream `Content-Type` for the value delivered to the napplet. Sniffed MIME drives scheme-appropriate allowlist enforcement. |
| Rasterize SVG in a sandboxed Worker with no network | Per the SVG Handling section. `image/svg+xml` MUST NOT be delivered to napplets. |
| Complete Blob delivery | Each successful result item contains exactly one complete `Blob`; no streaming, no chunked, no range, no progress. |
| Preserve bulk result order | `resource.bytesMany.result.items` has exactly one item per input URL, in input order. |
| Apply policy per bulk URL | Bulk requests do not bypass scheme, MIME, rasterization, size, timeout, cache, quota, rate-limit, or private-IP rules. |
| Verify Blossom hash before delivery | For `blossom:sha256:<hex>` URLs, the returned bytes MUST hash to the declared digest. Mismatch MUST result in `error: "decode-failed"`. |
| Single-hop `nostr:` resolution | Do NOT recursively follow URLs found within resolved Nostr resources. |
| Scope cache per `(dTag, aggregateHash)` | Per NIP-5D identity binding. Napplets MUST NOT see another napplet's cached resources. |
| Drop late envelopes after cancel | After receiving `resource.cancel` for an `id`, shells SHOULD NOT emit a terminal envelope for that `id`; if one is emitted (race), conformant napplet shims MUST silently drop it. |

### SHOULD

| Behavior | Details |
|----------|---------|
| Enforce response size cap | Recommended 10 MiB per response. Excess MUST result in `error: "too-large"`. |
| Enforce fetch timeout | Recommended 30 s per URL. Excess MUST result in `error: "timeout"`. |
| Enforce per-napplet concurrency / rate limit | Recommended 10 concurrent / 60 per minute. Excess MUST result in `error: "blocked-by-policy"`. |
| Cap redirect chain | Recommended ≤ 5 hops, re-validating against IP block list per hop. |
| Enforce per-napplet outstanding-Blob quota | Recommended ~50 MiB outstanding per napplet; over-quota MUST emit `error: "quota-exceeded"`. Existing Blobs unaffected. |
| Coalesce concurrent same-URL fetches | Single-flight cache keyed on canonical URL form; N concurrent calls for the same URL share one in-flight fetch. |
| Surface deviations in shell documentation | When the operator overrides any default policy, the shell SHOULD document the override so napplet authors can discover and adapt. |

### MAY

| Behavior | Details |
|----------|---------|
| Cache resources beyond a single fetch | LRU / TTL eviction policy per shell. Cache MUST still be scoped per `(dTag, aggregateHash)`. |
| Pre-resolve resources via sidecar | Carrier-owned (e.g. NAP-RELAY); **default OFF** per the carrier's privacy rationale. This NAP owns only the entry type and hydration contract. |
| Allow additional schemes behind shell-administrator policy | Enterprise / advanced deployments. |
| Use Transferable ArrayBuffer for large delivery | Optimization for resources `> 256 KiB` where the shell does not need to retain the bytes after delivery. |
| Negotiate transform hints in a future NAP revision | This NAP ships byte-faithful primitives only; transform hints (max width, preferred format, priority) are explicitly deferred. |

## Coexistence with NAP-RELAY / NAP-IDENTITY / NAP-MEDIA

The resource NAP is the canonical fetch path for every external byte resource referenced by another NAP. These domains depend on `resource`; the edges are directional and `resource` defines none of their wire:

- **NAP-RELAY** (wire importer): carries `ResourceSidecarEntry[]` on its own `relay.event.resources?` field. NAP-RELAY owns that field, its default-OFF policy, and the per-event-kind allowlist; this NAP owns only the entry type and the hydration contract.
- **NAP-IDENTITY**: `ProfileData.picture` and `ProfileData.banner` are URL strings. Napplets fetch them via `resource.bytes(url)`. No wire change to NAP-IDENTITY -- only a documentation clarification noting that direct `<img src="https://...">` does not work under the iframe sandbox model.
- **NAP-MEDIA**: `MediaMetadata.artwork.url` is a URL string. Napplets fetch artwork bytes via `resource.bytes(url)`. No wire change to NAP-MEDIA -- only a documentation clarification.

## Error Codes

| Code | Emitted by | Meaning |
|------|------------|---------|
| `invalid-request`    | `resource.bytes.error`, `resource.bytesMany.error` | Request payload is malformed, missing required fields, or has an empty `urls` list. |
| `not-found`          | `resource.bytes.error`, `resource.bytesMany.result` item | The resource does not exist (e.g., HTTP 404, Blossom not-found across all consulted hosts, unresolvable `nostr:` bech32). |
| `blocked-by-policy`  | `resource.bytes.error`, `resource.bytesMany.result` item | Shell policy rejected the fetch (private-IP block, MIME-not-allowed, rate limit exceeded, concurrency limit exceeded, redirect-to-blocked, scheme-disallowed). |
| `timeout`            | `resource.bytes.error`, `resource.bytesMany.result` item | Fetch exceeded the per-URL fetch timeout, or rasterization exceeded the wall-clock budget. |
| `too-large`          | `resource.bytes.error`, `resource.bytesMany.result` item, `resource.bytesMany.error` | Response body exceeded the configured size cap, rasterization input/output exceeded the configured caps, or a bulk request exceeded the URL count cap. |
| `unsupported-scheme` | `resource.bytes.error`, `resource.bytesMany.result` item | URL scheme is not in the shell's whitelist (canonical or opt-in). |
| `decode-failed`      | `resource.bytes.error`, `resource.bytesMany.result` item | MIME byte-sniff failed to classify, Blossom hash verification failed, or rasterization aborted on malformed input. |
| `network-error`      | `resource.bytes.error`, `resource.bytesMany.result` item | Generic upstream network failure (DNS resolution failure, TCP reset, TLS handshake failure, connection refused). |
| `quota-exceeded`     | `resource.bytes.error`, `resource.bytesMany.result` item | Per-napplet outstanding-Blob quota exceeded; existing delivered Blobs are unaffected. |

## Security Considerations

### Source-identity scope binding

Cache scope is derived from the napplet's `MessageEvent.source` per NIP-5D. The shell resolves `(dTag, aggregateHash)` from the source -- never from napplet payload. A napplet cannot read another napplet's cached resources, request resources scoped to another napplet, or fingerprint another napplet's recent fetches.

### SSRF as a primary attack surface

The shell-as-fetch-proxy model accepts that the shell is making outbound HTTP requests on the napplet's behalf. An unmitigated implementation is an SSRF gadget: a napplet could probe internal addresses, exfiltrate cloud-metadata credentials, or scan the deployer's intranet. The DNS-pinning + private-IP block list MUST exists to neutralize this. The check happens at DNS-resolution time, after the resolver returns an address and before the connection is opened, specifically to defeat DNS-rebinding (the attacker controls the DNS record for `attacker.example.com` and rotates the answer between a public address at validation time and `127.0.0.1` at connect time -- pinning the resolved address to the validated one defeats this).

### SVG attack surface

`image/svg+xml` is a parseable XML execution environment, not a static image. `<foreignObject>` hosts arbitrary HTML and JavaScript. `<script>` runs in the SVG document. `<image href>` and `<use href>` can issue cross-origin network requests during rendering. DOCTYPE entity expansion enables the billion-laughs DoS. Recursive `<use>` references generate unbounded render trees. Each of these is a documented attack surface against image renderers historically. The sandboxed-Worker-with-no-network rasterization MUST eliminates the network surfaces; the input/output caps eliminate the resource-exhaustion surfaces; the rasterize-to-PNG/WebP MUST eliminates the parseable-XML-reaches-napplet surface. All three MUSTs work together; partial implementations leave attack surface open.

### Upstream Content-Type passthrough is unsafe

If the shell honored upstream `Content-Type` for the value delivered to the napplet, an attacker who controls a content host could declare `text/html` for what is actually `image/png` (or vice-versa) and coerce a napplet into a confused-render attack. MIME byte-sniffing on the actual response bytes is the only safe classification; the upstream header is hint-only and MUST NOT be propagated.

### Sidecar pre-resolution is opt-in for privacy reasons

The carrier domain (e.g. NAP-RELAY) defines its `resources?` sidecar field as **default OFF**. Pre-fetching resources from URLs in events reveals user activity to upstream hosts before the user has chosen to render the event; an avatar URL on every event in a 1000-event timeline becomes 1000 HTTP requests, each one a fingerprint visible to the operator of the upstream avatar host. See the carrier's Privacy Rationale for the full discussion and the per-event-kind allowlist guidance.

### Cleartext bytes over postMessage

`resource.bytes.result` payloads are transmitted cleartext over `postMessage`. Browser extensions with script access can observe the event, developer-tools inspection exposes the payload, and crash reporters or analytics scripts may capture it. Napplets SHOULD treat resource bytes as observable and avoid reading sensitive byte payloads through this channel where confidentiality matters. NAP-RESOURCE does not claim resource bytes are "secure" -- the honest framing is that the shell decides what to deliver and the bytes are visible to any code with access to the napplet's host page.

## Implementations

- (none yet)
