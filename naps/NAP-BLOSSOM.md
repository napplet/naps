# NAP-BLOSSOM

## Low-Level Blossom Blob Access

`draft`

**NAP ID:** NAP-BLOSSOM
**Domain:** `blossom`
**Depends:**
- `identity` -- layering · optional -- authenticated Blossom requests use the current shell-user pubkey for BUD-11 authorization tokens.
**Web binding (NIP-5D):** `window.napplet.blossom` · `shell.supports("blossom")`

## Description

NAP-BLOSSOM gives napplets a low-level, shell-mediated interface to
[Blossom](https://github.com/hzrd149/blossom) blob servers. A napplet names the
server, blob hash, bytes, and endpoint intent. The shell performs the HTTP
request, applies runtime network policy, constructs and signs any required
Blossom authorization token, and returns endpoint-shaped results.

This is not a generic network escape hatch. It only covers Blossom server
endpoints and Blossom URI helpers. NAP-RESOURCE remains the generic byte-fetch
surface. NAP-UPLOAD remains the high-level "store these bytes somewhere" surface
where the shell chooses NIP-96, Blossom, or another rail.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `head` | `request` (`BlossomHeadRequest`) | `BlossomHeadResult` | `blossom.head` / `blossom.head.result` |
| `get` | `request` (`BlossomGetRequest`) | `BlossomBytesResult` | `blossom.get` / `blossom.get.result` |
| `checkUpload` | `request` (`BlossomCheckRequest`) | `BlossomCheckResult` | `blossom.checkUpload` / `blossom.checkUpload.result` |
| `upload` | `request` (`BlossomUploadRequest`) | `BlossomDescriptorResult` | `blossom.upload` / `blossom.upload.result` |
| `mirror` | `request` (`BlossomMirrorRequest`) | `BlossomDescriptorResult` | `blossom.mirror` / `blossom.mirror.result` |
| `checkMedia` | `request` (`BlossomCheckRequest`) | `BlossomCheckResult` | `blossom.checkMedia` / `blossom.checkMedia.result` |
| `media` | `request` (`BlossomMediaRequest`) | `BlossomDescriptorResult` | `blossom.media` / `blossom.media.result` |
| `list` | `request` (`BlossomListRequest`) | `BlossomListResult` | `blossom.list` / `blossom.list.result` |
| `delete` | `request` (`BlossomDeleteRequest`) | `BlossomDeleteResult` | `blossom.delete` / `blossom.delete.result` |
| `parseUri` | `uri` (`tstr`) | `BlossomUri` | local helper; no wire message |
| `formatUri` | `uri` (`BlossomUri`) | `tstr` | local helper; no wire message |

### Schemas

```cddl
HexSha256 = tstr
HexPubkey = tstr
ServerUrl = tstr
BlobExtension = tstr
MimeType = tstr
ErrorCode = tstr
NostrTag = [* tstr]

BlossomAuthMode = "auto" / "required" / "none"

BlossomHeadRequest = {
  server: ServerUrl,
  sha256: HexSha256,
  ? extension: BlobExtension,
  ? auth: BlossomAuthMode, ; default "auto"
  ? purpose: tstr,         ; prompt text hint; shell-owned and diagnostic only
}

BlossomGetRequest = {
  server: ServerUrl,
  sha256: HexSha256,
  ? extension: BlobExtension,
  ? range: tstr,
  ? auth: BlossomAuthMode, ; default "auto"
  ? purpose: tstr,
}

BlossomCheckRequest = {
  server: ServerUrl,
  sha256: HexSha256,
  size: uint,
  type: MimeType,
  ? auth: BlossomAuthMode, ; default "auto"
  ? purpose: tstr,
}

BlossomUploadRequest = {
  server: ServerUrl,
  data: bstr,
  ? sha256: HexSha256,
  ? type: MimeType,
  ? size: uint,
  ? auth: BlossomAuthMode, ; default "auto"
  ? purpose: tstr,
}

BlossomMirrorRequest = {
  server: ServerUrl,
  url: tstr,
  ? sha256: HexSha256,
  ? auth: BlossomAuthMode, ; default "auto"
  ? purpose: tstr,
}

BlossomMediaRequest = {
  server: ServerUrl,
  data: bstr,
  ? sha256: HexSha256,
  ? type: MimeType,
  ? size: uint,
  ? auth: BlossomAuthMode, ; default "auto"
  ? purpose: tstr,
}

BlossomListRequest = {
  server: ServerUrl,
  pubkey: HexPubkey,
  ? cursor: HexSha256,
  ? limit: uint,
  ? auth: BlossomAuthMode, ; default "auto"
  ? purpose: tstr,
}

BlossomDeleteRequest = {
  server: ServerUrl,
  sha256: HexSha256,
  ? extension: BlobExtension,
  ? auth: BlossomAuthMode, ; default "auto"
  ? purpose: tstr,
}

BlossomHeaders = {
  ? contentType: MimeType,
  ? contentLength: uint,
  ? acceptRanges: tstr,
  ? sunset: tstr,
  ? reason: tstr,
}

BlossomDescriptor = {
  url: tstr,
  sha256: HexSha256,
  size: uint,
  type: MimeType,
  uploaded: uint,
  ? nip94: [* NostrTag],
  * tstr => any,
}

BlossomHeadResult = {
  ok: bool,
  status: uint,
  headers: BlossomHeaders,
  ? error: ErrorCode,
}

BlossomBytesResult = {
  ok: bool,
  status: uint,
  ? blob: bstr,
  ? headers: BlossomHeaders,
  ? error: ErrorCode,
}

BlossomCheckResult = {
  ok: bool,
  status: uint,
  ? headers: BlossomHeaders,
  ? error: ErrorCode,
}

BlossomDescriptorResult = {
  ok: bool,
  status: uint,
  ? descriptor: BlossomDescriptor,
  ? error: ErrorCode,
}

BlossomListResult = {
  ok: bool,
  status: uint,
  blobs: [* BlossomDescriptor],
  ? cursor: HexSha256,
  ? error: ErrorCode,
}

BlossomDeleteResult = {
  ok: bool,
  status: uint,
  ? error: ErrorCode,
}

BlossomUri = {
  sha256: HexSha256,
  extension: BlobExtension,
  servers: [* ServerUrl],
  authors: [* HexPubkey],
  ? size: uint,
}
```

**`head(request)`** -- Performs BUD-01 `HEAD /<sha256>` against `server`.
Returns status and safe metadata headers. It never returns bytes.

**`get(request)`** -- Performs BUD-01 `GET /<sha256>`. The shell MUST verify
that returned bytes hash to `request.sha256` before delivering `blob`. When
`range` is provided, the result MAY be a partial byte range; the shell MUST NOT
claim whole-blob verification for partial content.

**`checkUpload(request)`** -- Performs BUD-06 `HEAD /upload` using
`X-SHA-256`, `X-Content-Type`, and `X-Content-Length`. The result is advisory.
A later `upload` can still fail.

**`upload(request)`** -- Performs BUD-02 `PUT /upload`. The shell sends the
exact supplied bytes. It MUST NOT transform them. When `sha256` is supplied, the
shell MUST verify it against `data` before sending and use it for preflight or
authorization scoping.

**`mirror(request)`** -- Performs BUD-04 `PUT /mirror`. The shell submits the
remote URL to the target server. When `sha256` is supplied, the shell MUST scope
authorization to that hash and reject a returned descriptor with a different
hash.

**`checkMedia(request)`** -- Performs BUD-05 `HEAD /media`. The result is
advisory. A later `media` can still fail.

**`media(request)`** -- Performs BUD-05 `PUT /media`. The Blossom server may
optimize or transform media. The returned descriptor describes the stored result,
not necessarily the original bytes.

**`list(request)`** -- Performs BUD-12 `GET /list/<pubkey>`. This endpoint is
optional and unrecommended by Blossom. Shells MUST expose server failure as a
normal result, not as proof that the user has no blobs.

**`delete(request)`** -- Performs BUD-12 `DELETE /<sha256>`. A delete request
acts on exactly one hash. Multiple authorization `x` tags, if the shell creates
them, MUST NOT be treated as a multi-delete request.

**`parseUri(uri)`** -- Parses a BUD-10 `blossom:` URI into hash, extension,
server hints, author hints, and size. It performs no network I/O.

**`formatUri(uri)`** -- Produces a BUD-10 `blossom:` URI. It MUST include the
hash and extension, defaulting the extension to `.bin` when no better extension
is known.

## Wire Protocol

`blossom.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `blossom.head` | napplet -> shell | `id`, `request` |
| `blossom.head.result` | shell -> napplet | `id`, `result`, `error?` |
| `blossom.get` | napplet -> shell | `id`, `request` |
| `blossom.get.result` | shell -> napplet | `id`, `result`, `error?` |
| `blossom.checkUpload` | napplet -> shell | `id`, `request` |
| `blossom.checkUpload.result` | shell -> napplet | `id`, `result`, `error?` |
| `blossom.upload` | napplet -> shell | `id`, `request` |
| `blossom.upload.result` | shell -> napplet | `id`, `result`, `error?` |
| `blossom.mirror` | napplet -> shell | `id`, `request` |
| `blossom.mirror.result` | shell -> napplet | `id`, `result`, `error?` |
| `blossom.checkMedia` | napplet -> shell | `id`, `request` |
| `blossom.checkMedia.result` | shell -> napplet | `id`, `result`, `error?` |
| `blossom.media` | napplet -> shell | `id`, `request` |
| `blossom.media.result` | shell -> napplet | `id`, `result`, `error?` |
| `blossom.list` | napplet -> shell | `id`, `request` |
| `blossom.list.result` | shell -> napplet | `id`, `result`, `error?` |
| `blossom.delete` | napplet -> shell | `id`, `request` |
| `blossom.delete.result` | shell -> napplet | `id`, `result`, `error?` |

Key design notes:
- Request/result pairs use `id` for correlation.
- Binary `data` and `blob` fields cross the boundary as projection-native byte
  objects. In the web projection that means `Blob` or `ArrayBuffer`, not
  base64-encoded strings.
- Result `status` is the upstream HTTP status after shell policy checks. Shell
  policy failures MAY use `status: 0` with a stable `error`.
- `headers.reason` carries BUD `X-Reason` diagnostics only. Napplets MUST NOT
  branch on it.
- Local helpers (`parseUri`, `formatUri`) do not send wire messages.

### Examples

**Check, then upload exact bytes:**

```
-> { "type": "blossom.checkUpload", "id": "b1",
     "request": { "server": "https://cdn.example.com", "sha256": "b167...", "size": 184292, "type": "application/pdf" } }
<- { "type": "blossom.checkUpload.result", "id": "b1",
     "result": { "ok": true, "status": 200, "headers": {} } }
-> { "type": "blossom.upload", "id": "b2",
     "request": { "server": "https://cdn.example.com", "data": "<bytes>", "sha256": "b167...", "type": "application/pdf", "size": 184292 } }
<- { "type": "blossom.upload.result", "id": "b2",
     "result": { "ok": true, "status": 201, "descriptor": { "url": "https://cdn.example.com/b167....pdf", "sha256": "b167...", "size": 184292, "type": "application/pdf", "uploaded": 1725105921 } } }
```

**Retrieve by hash:**

```
-> { "type": "blossom.get", "id": "b3",
     "request": { "server": "https://cdn.example.com", "sha256": "b167...", "extension": ".pdf" } }
<- { "type": "blossom.get.result", "id": "b3",
     "result": { "ok": true, "status": 200, "blob": "<bytes>", "headers": { "contentType": "application/pdf", "contentLength": 184292 } } }
```

### Error Handling

All result messages use `result.ok: false` plus `result.error` for programmatic
failure. The top-level `error?` field is reserved for malformed envelopes where
no typed result can be constructed.

Common error codes: `invalid-request`, `invalid-server`, `invalid-hash`,
`not-signed-in`, `user-denied`, `blocked-by-policy`, `not-found`,
`auth-required`, `payment-required`, `too-large`, `unsupported-media-type`,
`hash-mismatch`, `network-error`, `timeout`, `server-error`, `unsupported`.

## Shell Behavior

- The shell MUST treat `server`, `url`, hashes, MIME strings, URI hints, and
  descriptors as untrusted.
- The shell MUST respond to every request with a result message carrying the
  same `id`.
- The shell MUST restrict requests to Blossom endpoint shapes. It MUST NOT turn
  this interface into arbitrary HTTP method, path, or header access.
- The shell MUST apply its normal network policy before connecting to any
  Blossom server or mirror origin.
- The shell MUST perform all BUD-11 authorization token construction, signing,
  encoding, and header attachment. It MUST NOT expose authorization tokens or
  signing keys to the napplet.
- The shell MUST scope authorization tokens with `server` tags derived from the
  lowercase Blossom server domain, not from a full URL.
- The shell MUST scope authorization tokens with `x` tags whenever the target
  endpoint implies a hash.
- The shell MUST reject `auth: "required"` requests and server-authenticated
  operations with `not-signed-in` when no shell-user signer is connected.
- The shell MAY prompt, deny, rate-limit, cache, redact diagnostics, or route by
  napplet policy.
- The shell MAY try unsigned requests for public endpoints when `auth` is
  `"auto"` and no shell-user signer is connected.
- The shell MAY send unsigned requests when `auth: "none"` is supplied, except
  when runtime policy requires user consent.

## Security Considerations

- Blossom server access is a network boundary. Shells MUST enforce SSRF
  protections, redirect limits, size limits, and per-napplet rate limits before
  delivering results.
- BUD-11 tokens are bearer credentials for their lifetime. Shells SHOULD keep
  expirations short, scope tokens to exact server domains, and scope hash-based
  operations with `x` tags.
- `mirror` asks a server to fetch a remote URL. Shells SHOULD apply policy to
  the origin URL before forwarding it and SHOULD make the destination server
  visible in consent UI.
- `media` may transform bytes. Napplets that require exact byte preservation
  MUST use `upload`, not `media`.
- `list` can reveal a user's stored blobs. Shells SHOULD require explicit
  consent when listing a pubkey that belongs to the shell-user.
- Successful upload, mirror, delete, or media results prove only what the
  Blossom server reported. They do not prove future availability or deletion
  across other servers.

## References

- [Blossom](https://github.com/hzrd149/blossom)
- [BUD-01: Server requirements and blob retrieval](https://github.com/hzrd149/blossom/blob/master/buds/01.md)
- [BUD-02: Blob upload](https://github.com/hzrd149/blossom/blob/master/buds/02.md)
- [BUD-03: User Server List](https://github.com/hzrd149/blossom/blob/master/buds/03.md)
- [BUD-04: Mirroring blobs](https://github.com/hzrd149/blossom/blob/master/buds/04.md)
- [BUD-05: Media optimization endpoints](https://github.com/hzrd149/blossom/blob/master/buds/05.md)
- [BUD-06: Upload requirements](https://github.com/hzrd149/blossom/blob/master/buds/06.md)
- [BUD-10: Blossom URI Schema](https://github.com/hzrd149/blossom/blob/master/buds/10.md)
- [BUD-11: Nostr Authorization](https://github.com/hzrd149/blossom/blob/master/buds/11.md)
- [BUD-12: Blob management endpoints](https://github.com/hzrd149/blossom/blob/master/buds/12.md)

## Implementations

- (none yet)
