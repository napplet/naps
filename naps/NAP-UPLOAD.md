NAP-UPLOAD
==========

Media and Blob Upload
---------------------

`draft`

**NAP ID:** NAP-UPLOAD
**Domain:** `upload`
**Depends:**
- `relay` — capability · optional — a napplet may publish the resulting file event via `relay`
**Web binding (NIP-5D):** `window.napplet.upload` · `shell.supports("upload")`

## Description

NAP-UPLOAD provides napplets with a shell-mediated interface for uploading files, media, and arbitrary blobs to Nostr-aware storage backends. A napplet hands the shell raw bytes plus a description of the intended upload; the shell selects a server, constructs and signs any required authorization, performs the HTTP upload, and returns a stable URL together with the integrity metadata needed to attach the result to a Nostr event.

The interface is named around uploading rather than any single protocol so shells can support multiple storage rails — [NIP-96](https://github.com/nostr-protocol/nips/blob/master/96.md) HTTP file storage and [Blossom](https://github.com/hzrd149/blossom) blob storage are the first concrete backends — without changing the napplet API. Result metadata follows [NIP-94](https://github.com/nostr-protocol/nips/blob/master/94.md) so a napplet can publish a file event or build an `imeta` tag through NAP-RELAY.

Napplets never receive signing keys, server credentials, or direct network access to storage backends. The shell signs the [NIP-98](https://github.com/nostr-protocol/nips/blob/master/98.md) HTTP auth event (for NIP-96) or the kind 24242 authorization event (for Blossom) on the napplet's behalf. The shell is the policy and consent boundary.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `info` | none | `UploadInfo` | `upload.info` / `upload.info.result` |
| `upload` | `request` (`UploadRequest`) | `UploadResult` | `upload.upload` / `upload.upload.result` |
| `status` | `uploadId` (`tstr`) | `UploadStatus` | `upload.status` / `upload.status.result` |
| `onStatus` | handler for `UploadStatus` | `Subscription` handle | `upload.status.changed` |

### Schemas

Primitive references:

| Name | Meaning |
|------|---------|
| `NostrTag` | NIP-01 event tag list. |

Enumerations:

| Name | Values |
|------|--------|
| `UploadRail` | `nip96`, `blossom`, or another rail name string. |
| `UploadState` | `pending`, `uploading`, `complete`, `failed`, `cancelled` |

`UploadRailInfo`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `rail` | `UploadRail` | yes | Upload rail. |
| `enabled` | `bool` | yes | Whether this rail is available. |
| `returns` | list of `tstr` | no | URL or result forms, such as `https` or `blossom`. |

`UploadInfo`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `rails` | list of `UploadRailInfo` | yes | Upload rails the runtime is willing to disclose. |
| `maxBytes` | `uint` | no | Maximum accepted upload size in bytes. |
| `mimeTypes` | list of `tstr` | no | Accepted MIME types. |

`UploadRequest`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `rail` | `UploadRail` | no | Requested rail; omitted lets the shell pick a configured default. |
| `data` | `bstr` | yes | Upload bytes; Blob or ArrayBuffer in the web projection. |
| `mimeType` | `tstr` | no | MIME type; inferred from data when omitted. |
| `filename` | `tstr` | no | Suggested filename. |
| `caption` | `tstr` | no | Alt text or file-event description. |
| `noTransform` | `bool` | no | Request that the server not transform the upload. |
| `metadata` | map of `tstr` to any JSON value | no | Rail-specific metadata. |

`UploadDimensions`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `width` | `uint` | yes | Width in pixels. |
| `height` | `uint` | yes | Height in pixels. |

`UploadResult`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ok` | `bool` | yes | Whether the upload request succeeded. |
| `uploadId` | `tstr` | yes | Shell-generated upload identifier. |
| `status` | `UploadState` | yes | Current upload state. |
| `rail` | `UploadRail` | yes | Upload rail used. |
| `url` | `tstr` | no | Primary download URL. |
| `fallbackUrls` | list of `tstr` | no | Additional download URLs. |
| `sha256` | `tstr` | no | Hash of the stored blob, NIP-94 `x`. |
| `originalSha256` | `tstr` | no | Hash before server transforms, NIP-94 `ox`. |
| `size` | `uint` | no | Stored size in bytes. |
| `mimeType` | `tstr` | no | Stored MIME type. |
| `dimensions` | `UploadDimensions` | no | Media dimensions. |
| `blurhash` | `tstr` | no | Blurhash preview. |
| `nip94` | list of `NostrTag` | no | NIP-94 tags for the uploaded file. |
| `error` | `tstr` | no | Error reason when upload failed. |

`UploadStatus`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ok` | `bool` | yes | Whether the status lookup succeeded. |
| `uploadId` | `tstr` | yes | Shell-generated upload identifier. |
| `status` | `UploadState` | yes | Current upload state. |
| `rail` | `UploadRail` | yes | Upload rail used. |
| `url` | `tstr` | no | Primary download URL. |
| `fallbackUrls` | list of `tstr` | no | Additional download URLs. |
| `sha256` | `tstr` | no | Hash of the stored blob, NIP-94 `x`. |
| `originalSha256` | `tstr` | no | Hash before server transforms, NIP-94 `ox`. |
| `size` | `uint` | no | Stored size in bytes. |
| `mimeType` | `tstr` | no | Stored MIME type. |
| `dimensions` | `UploadDimensions` | no | Media dimensions. |
| `blurhash` | `tstr` | no | Blurhash preview. |
| `nip94` | list of `NostrTag` | no | NIP-94 tags for the uploaded file. |
| `error` | `tstr` | no | Error reason when upload failed. |
| `bytesSent` | `uint` | no | Bytes uploaded so far. |
| `bytesTotal` | `uint` | no | Total bytes expected. |
| `updatedAt` | `uint` | yes | Timestamp of the latest status update. |

**`info()`** — Returns the upload rails and coarse limits the runtime is willing to expose to the napplet. Napplets MAY call this to adapt UI or intentionally request a specific rail, but they MUST NOT be required to call it before `upload(request)`.

**`upload(request)`** — Uploads the supplied bytes. The shell presents any required consent UI, selects a backend (honoring `rail` when given), constructs and signs the rail's authorization, performs the upload, and returns the initial result. When the upload completes synchronously the result carries `status: "complete"` and a `url`; when it is large or asynchronous the shell returns `status: "uploading"` and reports progress through `upload.status.changed`.

**`status(uploadId)`** — Returns the latest known status for a prior upload, including progress counters while an upload is in flight.

**`onStatus(handler)`** — Registers for shell-pushed status updates, such as `uploading` progress and the transition to `complete` or `failed`.

## Wire Protocol

`upload.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `upload.info` | napplet -> shell | `id` |
| `upload.info.result` | shell -> napplet | `id`, `info`, `error?` |
| `upload.upload` | napplet -> shell | `id`, `request` |
| `upload.upload.result` | shell -> napplet | `id`, `result`, `error?` |
| `upload.status` | napplet -> shell | `id`, `uploadId` |
| `upload.status.result` | shell -> napplet | `id`, `status`, `error?` |
| `upload.status.changed` | shell -> napplet | `status` |

Key design notes:
- Request/result pairs use `id` for correlation.
- `upload.info` is optional introspection. It is not a required preflight for `upload.upload`.
- `uploadId` is generated by the shell and is scoped to the requesting napplet.
- `request.data` is a `Blob` or `ArrayBuffer` and crosses the boundary by structured clone; shells MUST NOT require napplets to base64-encode payloads.
- Omitting `request.rail` lets the shell choose the best configured rail according to user and runtime policy.
- `upload.status.changed` is a shell push message and has no `id`.
- The shell SHOULD populate `nip94` so a napplet can attach the file to a Nostr event without recomputing hashes it cannot verify.

### Examples

**Inspect available upload rails:**
```
-> { "type": "upload.info", "id": "i1" }
<- {
     "type": "upload.info.result",
     "id": "i1",
     "info": {
       "rails": [
         { "rail": "nip96", "enabled": true, "returns": ["https"] },
         { "rail": "blossom", "enabled": true, "returns": ["https", "blossom"] }
       ],
       "maxBytes": 104857600,
       "mimeTypes": ["image/png", "image/jpeg", "application/pdf"]
     }
   }
```

**Upload an image:**
```
-> {
     "type": "upload.upload",
     "id": "u1",
     "request": {
       "rail": "nip96",
       "data": <Blob image/png 18234 bytes>,
       "filename": "diagram.png",
       "caption": "architecture diagram",
       "mimeType": "image/png"
     }
   }
<- {
     "type": "upload.upload.result",
     "id": "u1",
     "result": {
       "ok": true,
       "uploadId": "upload-1",
       "status": "uploading",
       "rail": "nip96"
     }
   }
<- {
     "type": "upload.status.changed",
     "status": {
       "ok": true,
       "uploadId": "upload-1",
       "status": "complete",
       "rail": "nip96",
       "url": "https://files.example.com/abc123.png",
       "sha256": "abc123...",
       "originalSha256": "def456...",
       "size": 18234,
       "mimeType": "image/png",
       "dimensions": { "width": 1280, "height": 720 },
       "blurhash": "L6PZfSi_.AyE_3t7t7R**0o#DgR4",
       "nip94": [
         ["url", "https://files.example.com/abc123.png"],
         ["m", "image/png"],
         ["x", "abc123..."],
         ["ox", "def456..."],
         ["size", "18234"],
         ["dim", "1280x720"]
       ],
       "updatedAt": 1234567901
     }
   }
```

**Upload a blob to Blossom:**
```
-> {
     "type": "upload.upload",
     "id": "u2",
     "request": { "rail": "blossom", "data": <Blob application/pdf 90211 bytes>, "filename": "notes.pdf" }
   }
<- {
     "type": "upload.upload.result",
     "id": "u2",
     "result": {
       "ok": true,
       "uploadId": "upload-2",
       "status": "complete",
       "rail": "blossom",
       "url": "https://blossom.example.com/sha256hash.pdf",
       "sha256": "sha256hash...",
       "size": 90211,
       "mimeType": "application/pdf"
     }
   }
```

**User cancelled:**
```
<- {
     "type": "upload.upload.result",
     "id": "u3",
     "result": {
       "ok": false,
       "uploadId": "upload-3",
       "status": "cancelled",
       "rail": "nip96",
       "error": "user cancelled"
     }
   }
```

### Error Handling

Result messages MAY include `error` when the request cannot be fulfilled. Common errors include `"unsupported rail"`, `"policy denied"`, `"no server configured"`, `"file too large"`, `"unsupported media type"`, `"user cancelled"`, `"server rejected"`, `"upload failed"`, and `"quota exceeded"`.

The shell SHOULD return a structured `result` with `ok: false` when an upload was created and then failed or was cancelled. The shell SHOULD return a top-level `error` when no upload was created.

## Shell Behavior

- The shell MUST keep `upload.info` advisory. A napplet that skips `info()` and calls `upload(request)` directly remains valid.
- If `request.rail` is supplied, the shell SHOULD honor it when supported and allowed; otherwise it SHOULD fail with `"unsupported rail"` or `"policy denied"`.
- If `request.rail` is omitted, the shell MUST select the storage rail according to user and runtime policy.
- The shell MUST construct and sign any authorization required by the chosen rail — the NIP-98 HTTP auth event for NIP-96, or the kind 24242 authorization event for Blossom — without exposing signing keys to the napplet.
- The shell MUST select the storage server. Server URLs, credentials, and rotation are shell configuration, not napplet input.
- The shell MUST enforce per-napplet policy for allowed rails, maximum file size, and allowed MIME types.
- The shell MUST respond to every request with a result message carrying the same `id`.
- The shell MUST report the integrity hashes (`sha256`, and `originalSha256` when the server transforms the file) so napplets can attach verifiable NIP-94 metadata.
- The shell SHOULD surface upload progress through `upload.status.changed` for large or asynchronous uploads.
- The shell SHOULD obtain user consent before the first upload from a napplet, and MAY apply a per-napplet allowance for subsequent uploads.
- The shell MAY upload to multiple mirrors and return them in `fallbackUrls`.
- The shell MAY strip or rewrite metadata (such as EXIF) before upload as a privacy measure, and SHOULD reflect this in the returned hashes.

## Security Considerations

- Uploading is a network egress and an identity-linking action. Shells MUST treat every `upload.upload` as an untrusted request until policy checks pass.
- `upload.info` can reveal configured storage rails, limits, and MIME policy. Shells MAY redact unavailable rails or coarse-grain limits for untrusted napplets.
- Napplets can attempt to exfiltrate data by uploading arbitrary bytes. Shell consent UI SHOULD show the file type, size, target server, and requesting napplet identity, and SHOULD allow the user to inspect or preview the payload before the first upload.
- The shell signs the rail authorization on behalf of the user. Napplets MUST NOT be given signing or encryption primitives through this interface; NIP-98 and Blossom auth events are constructed and signed by the shell only.
- Metadata leakage is a real risk. Files commonly carry EXIF GPS, device identifiers, and timestamps. Shells SHOULD offer metadata stripping and SHOULD make the policy visible to the user.
- Returned URLs are public and durable. Shells SHOULD make clear to the user that an upload is published, not private, unless the rail provides encryption.
- Hashes are integrity claims. A `complete` status with a `url` MUST reflect bytes the shell actually stored; the shell MUST NOT report a success the server did not confirm.
- Quotas and abuse: shells SHOULD rate-limit per-napplet uploads to prevent a napplet from exhausting server quota or using the user's identity to spam storage providers.

## Implementations

- (none yet)
