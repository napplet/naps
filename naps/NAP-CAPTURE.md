NAP-CAPTURE
===========

Shell-Mediated Microphone Recording
-----------------------------------

`draft`

**NAP ID:** NAP-CAPTURE
**Domain:** `capture`
**Depends:** none.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.capture`; domain presence signals availability.

## Description

NAP-CAPTURE lets a napplet ask its runtime to create a finite, user-approved
microphone recording. The runtime owns consent, device selection, platform
permission, capture, encoding, limits, indication, retention, and teardown.

The napplet receives encoded audio bytes, not microphone authority. It never
receives a device handle, stable device identifier, raw stream, recorder object,
or platform permission state.

NAP-CAPTURE has no dependency on another NAP. Passing a completed artifact to
`upload`, `storage`, `relay`, or another domain is a separate napplet action.

## Discovery

In the web projection, a napplet detects the capability by checking whether
`window.napplet.capture` is present. No shell handshake is required.

A runtime that reports support MUST implement every baseline operation in this
document.

`info()` reports output formats and finite policy limits. It MUST NOT reveal
stable device identifiers, device labels, device counts, or platform permission
state.

## API Surface

| Operation | Parameters | Result | Wire |
|---|---|---|---|
| `info` | — | `CaptureInfo` | `capture.info` → `capture.info.result` |
| `start` | `CaptureRequest` | `CaptureSession` | `capture.start` → `capture.start.result` |
| `status` | `captureId` | `CaptureStatus` | `capture.status` → `capture.status.result` |
| `stop` | `captureId` | `CaptureArtifact` | `capture.stop` → `capture.stop.result` |
| `cancel` | `captureId` | `CaptureStatus` | `capture.cancel` → `capture.cancel.result` |
| `release` | `captureId` | `captureId` | `capture.release` → `capture.release.result` |

`status` is authoritative. `capture.changed` is advisory; a missed event never
loses a terminal artifact or terminal error before its disclosed expiry.

### `CaptureInfo`

| Field | Required | Type | Notes |
|---|---|---|---|
| `sources` | yes | list of text | MUST contain only `"microphone"` in this version. |
| `mimeTypes` | yes | list of text | MIME types the runtime is prepared to attempt, in preference order. |
| `maxDurationMs` | yes | positive integer | Maximum interval before the runtime initiates terminalization. |
| `maxBytes` | yes | positive integer | Hard maximum deliverable artifact size. |
| `retentionMs` | yes | positive integer | Time a terminal status remains recoverable unless explicitly erased. |

`CaptureInfo` is advisory. Policy, device availability, encoder startup, and
platform permission can still make a later `start` fail. All three limits MUST
be finite; a runtime MUST NOT advertise `null` or an unbounded value.

### `CaptureRequest`

| Field | Required | Type | Notes |
|---|---|---|---|
| `source` | yes | text | MUST be `"microphone"`. |
| `mimeTypes` | no | list of text | Acceptable output MIME types in caller preference order. |

An omitted or empty `mimeTypes` list lets the runtime choose any advertised
format. Each supplied item MUST be a valid MIME media type or `start` fails with
`invalid-request` before consent UI. MIME compatibility is evaluated by parsing
media types. Type, subtype, and parameter names are ASCII case-insensitive.
Duplicate parameter names make a request item invalid. Parsing removes optional
whitespace outside a parameter value. Quoted-string escaping is removed, but
whitespace inside the quoted value is preserved. Parameter values are otherwise
compared byte-for-byte and case-sensitively. For the `codecs` parameter, split the
unquoted comma-separated value and remove optional whitespace around each token.
Compare the resulting ordered token list case-sensitively by default. A
registered codec namespace MAY define a different comparison rule; apply it only
when both tokens belong to that namespace. Every
parameter requested by the caller MUST be present and equal under this algorithm
in the actual output; additional actual parameters are allowed. Parameter order
does not affect compatibility. The runtime MUST apply this same algorithm
wherever this document says MIME-compatible.

The runtime MUST choose an advertised format compatible with at least one
requested item or fail with `unsupported-format` before consent UI. The returned
`mimeType` is the normalized actual output type, not necessarily the caller's
original string. An advertised format is one the runtime is prepared to attempt,
not a guarantee that the platform encoder will later start successfully.

The request has no device selector or low-level audio constraints. A runtime MAY
offer device and quality choices in trusted shell UI.

### `CaptureSession`

| Field | Required | Type | Notes |
|---|---|---|---|
| `captureId` | yes | text | Opaque identifier bound to the authenticated requesting endpoint generation. |
| `source` | yes | text | `"microphone"`. |
| `mimeType` | yes | text | Actual output MIME type selected by the runtime. |
| `maxDurationMs` | yes | positive integer | Effective terminalization deadline interval for this session. |
| `maxBytes` | yes | positive integer | Effective hard artifact-size limit for this session. |
| `retentionMs` | yes | positive integer | Effective terminal-status retention interval for this session. |

A successful `start` result means consent succeeded and recording is active. A
runtime MUST NOT return a `CaptureSession` while consent is pending.

### `CaptureArtifact`

| Field | Required | Type | Notes |
|---|---|---|---|
| `captureId` | yes | text | Identifier of the completed capture. |
| `data` | yes | any | Projection-defined carrier for one immutable, complete encoded audio byte sequence. It is never text, base64, a live stream, or a device/recorder handle. |
| `mimeType` | yes | text | Normalized actual MIME type of `data`. |
| `size` | yes | non-negative integer | Exact byte length of `data`. |
| `durationMs` | yes | non-negative integer | Runtime-measured active capture interval. |
| `truncated` | yes | boolean | `true` when a limit or interruption ended capture. |
| `reason` | yes | `"requested"`, `"user"`, `"limit"`, or `"interrupted"` | Terminal cause. |

`durationMs` is monotonic elapsed time from recorder activation to the terminal
decision, excluding consent and backend finalization time; it is not parsed
container duration. `maxDurationMs` requires the runtime to initiate the terminal
decision by that deadline, but scheduler and backend latency MAY make reported
`durationMs` larger. `maxBytes` is a hard upper bound on a deliverable artifact.
`size` MUST equal the binary byte length and MUST NOT exceed the session's
`maxBytes`. The runtime MUST report the actual container and codec
in `mimeType`; it MUST NOT merely echo the request.

### `CaptureStatus`

| Field | Required | Type | Notes |
|---|---|---|---|
| `captureId` | yes | text | Affected capture. |
| `state` | yes | `"recording"`, `"completed"`, `"cancelled"`, or `"failed"` | Current observable state. |
| `artifact` | completed only | `CaptureArtifact` | Retained terminal artifact. |
| `reason` | cancelled only | `"requested"`, `"user"`, or `"revoked"` | Stable cancellation reason. |
| `error` | failed only | `CaptureError` | Retained normalized failure. |
| `expiresAt` | terminal only | non-negative integer | Unix time in milliseconds when retained terminal state is automatically erased. |

A `recording` status MUST omit `artifact`, `reason`, `error`, and `expiresAt`.
A `completed` status MUST contain only `artifact` and `expiresAt` among those
conditional fields; `cancelled` MUST contain only `reason` and `expiresAt`; and
`failed` MUST contain only `error` and `expiresAt`.

A terminal `CaptureStatus` remains available until `release`, trusted discard,
napplet unload, identity revocation, or `expiresAt`, whichever comes first. The
runtime MUST set `expiresAt` to the terminal commit time plus the session's
`retentionMs` and automatically erase retained runtime state at that time. A new
`start` for the identity fails with `busy` while a pending, recording, finalizing,
or retained capture occupies its slot.

### `CaptureEvent`

| Field | Required | Type | Notes |
|---|---|---|---|
| `captureId` | yes | text | Affected capture. |
| `state` | yes | `"completed"`, `"cancelled"`, or `"failed"` | New terminal state. |
| `reason` | cancelled only | `"requested"`, `"user"`, or `"revoked"` | Stable cancellation reason. |
| `error` | failed only | `CaptureError` | Normalized failure. |

`capture.changed` carries `CaptureEvent`. It is a wake-up hint only. The napplet
uses `status` to retrieve the authoritative terminal state and artifact.

There is no subscribe or unsubscribe operation. The runtime automatically pushes
`capture.changed` to the bound napplet endpoint while that endpoint generation
remains present and authorized. A projection MAY expose an idiomatic
event-listener shape as guidance, but that shape is not part of this NAP contract.

### `CaptureError`

| Field | Required | Type | Notes |
|---|---|---|---|
| `code` | yes | text | Stable error code from the table below. |
| `message` | no | text | Safe diagnostic text; MUST NOT expose device identifiers or platform internals. |

## Lifecycle

An **endpoint generation** is one authenticated lifetime of a projection
endpoint. Each projection MUST define when a generation begins and ends.

Each runtime MUST reserve at most one pending, recording, finalizing, or retained
capture slot per verified napplet identity. The slot's owner is the authenticated
endpoint generation that issued `start`; identity is the scope for attribution,
rate limits, and concurrency policy, not authorization to operate another
endpoint's capture. The runtime MUST also enforce finite runtime-wide limits on
active captures and retained encoded bytes. The identity slot is reserved before
trusted consent UI opens. A concurrent `start` for that identity fails with
`busy`. Denial, cancellation, or start failure releases a pending reservation.

```text
                start approved
     absent  -------------------->  recording
                                        |
                           stop/runtime | cancel
                           decide/finalize| discard
                                        v
                    completed / cancelled / failed
                                        |
                        release / expiry / teardown
                                        v
                                      absent
```

### Terminal decisions

A terminal decision selects the intended outcome before asynchronous recorder
finalization. The first accepted caller or runtime terminal decision wins, except
that trusted **Discard**, permission or policy revocation, identity revocation,
and owner-endpoint unload MUST supersede an uncommitted completion decision and
erase runtime-held bytes. A runtime MAY use an internal `finalizing` state. While
completion is finalizing, `status` reports `recording`; a second `stop`, `cancel`,
or `release` fails with `invalid-state`. No artifact or terminal event exists
until terminal commit.

Terminal commit atomically stores the terminal `CaptureStatus`, makes any
completed artifact eligible for delivery, sets `expiresAt`, frees active device
resources, and emits the one advisory event. Backend callbacks after commit MUST
NOT alter the status, create another artifact, or deliver more bytes.

Rules:

1. `start` validates the request and reserves policy capacity before opening
   trusted UI. It creates a session only after consent succeeds, microphone input
   is active, and the encoder has started. A pre-session rejection creates no
   `captureId`, status, or event.
2. `status` returns the observable state. For `completed`, it returns the retained
   artifact. For `cancelled` or `failed`, it returns the retained reason or error.
3. `stop` on `recording` claims a completion decision and remains pending until
   finalization commits or fails. On success it retains and returns one artifact
   with `reason: "requested"`. `stop` on `completed` returns the same retained
   artifact. `stop` on `cancelled` or `failed` fails with `invalid-state`.
4. `cancel` on `recording` claims a destructive decision, stops input, erases
   runtime-held bytes, commits `cancelled` with `reason: "requested"`, and returns
   that status. `cancel` on a terminal capture returns the retained status without
   altering or erasing it.
5. `release` is valid only for a committed terminal capture. It erases retained
   runtime status and bytes. Later operations, including a repeated `release`,
   fail with `capture-not-found`.
6. At `expiresAt`, the runtime performs the same erasure as `release` without a
   request or event. The identity slot then becomes available.
7. A duration limit requires the runtime to initiate terminalization no later
   than `maxDurationMs` after activation. A byte limit requires the runtime to
   stop before or when it determines the next complete valid artifact may exceed
   policy. If a complete valid artifact is available within `maxBytes`, commit
   `completed` with `truncated: true` and `reason: "limit"`. The runtime MUST NOT
   byte-truncate an encoded container. If a complete valid artifact cannot fit,
   erase it and commit `failed` with `quota-exceeded`.
8. Device loss MAY commit a complete recoverable artifact within `maxBytes` as
   `completed`, `truncated: true`, `reason: "interrupted"` while recording
   authority remains valid. Otherwise it commits `failed` with
   `device-unavailable` and erases bytes. Device loss MUST NOT leave recording
   active.
9. Permission, policy, or identity revocation stops input and prevents any
   uncommitted artifact delivery. Permission or policy revocation while the owner
   endpoint remains valid commits `cancelled` with `reason: "revoked"`; identity
   revocation erases all matching runtime state without further delivery.
10. Owner-endpoint unload cancels pending consent, stops input, and erases that
    endpoint generation's runtime state without a result or event. It MUST NOT
    erase a capture merely because another live endpoint of the same identity
    unloads, and it MUST NOT continue recording in the background.

### Trusted user controls

| Trusted action | Required outcome |
|---|---|
| Decline or dismiss consent before activation | `capture.start.result` with `user-cancelled`; no `captureId`, status, artifact, or event. |
| trusted **Stop** during recording | Complete and retain one artifact with `reason: "user"`. |
| trusted **Discard** during recording or finalization | Stop input, erase runtime-held bytes, and commit `cancelled` with `reason: "user"`. |
| trusted **Discard** after completion | Erase retained runtime state immediately; later operations return `capture-not-found`. |

Every terminal commit to `completed`, `cancelled`, or `failed` emits exactly one
`capture.changed` event to the owning endpoint generation while it remains
present and authorized. Event loss is harmless before expiry because `status` is
authoritative.

A napplet cannot programmatically cancel consent before `start` returns because
no `captureId` exists yet. The runtime MUST cancel pending consent when the owner
endpoint unloads; trusted shell UI MUST let the user cancel it directly.

Pause/resume is not a baseline v1 operation.

## Wire Protocol

All request messages carry a caller-generated correlation `id`. While the source
endpoint generation remains present and authorized, each processed request MUST
settle exactly once with its matching result type, which echoes that `id`.
Endpoint unload or revocation cancels undeliverable pending responses.
Request-ID replay behavior is owned by the common transport, not this domain, and
callers MUST use a fresh `id` for each operation while an earlier request is
pending.

The projection authenticates both the source endpoint generation and its verified
manifest and artifact identity before dispatch. A `captureId` is generated by
the runtime and authorized only for the authenticated endpoint generation that
created it. The verified identity remains attribution and policy scope. A request
from a foreign identity, a stale/reloaded endpoint generation, or another live
endpoint of the same identity MUST fail uniformly with `capture-not-found`.
Responses and events MUST be delivered only to the owning endpoint generation.

| Type | Direction | Payload fields |
|---|---|---|
| `capture.info` | napplet → runtime | `id` |
| `capture.info.result` | runtime → napplet | `id`, exactly one of `info` or `error` |
| `capture.start` | napplet → runtime | `id`, `request` |
| `capture.start.result` | runtime → napplet | `id`, exactly one of `session` or `error` |
| `capture.status` | napplet → runtime | `id`, `captureId` |
| `capture.status.result` | runtime → napplet | `id`, exactly one of `status` or `error` |
| `capture.stop` | napplet → runtime | `id`, `captureId` |
| `capture.stop.result` | runtime → napplet | `id`, exactly one of `artifact` or `error` |
| `capture.cancel` | napplet → runtime | `id`, `captureId` |
| `capture.cancel.result` | runtime → napplet | `id`, exactly one of `status` or `error` |
| `capture.release` | napplet → runtime | `id`, `captureId` |
| `capture.release.result` | runtime → napplet | `id`, exactly one of `captureId` or `error` |
| `capture.changed` | runtime → napplet | `event` |

### Result and error routing

| Situation | Correlated request outcome | Retained status / event |
|---|---|---|
| Malformed or unsupported request, policy denial, consent decline, or failure before a session exists | Matching `*.result` with `error` | None |
| Unknown, foreign, stale, or wrong-state `captureId` | Matching `*.result` with `error` | No transition and no event |
| Successful operation | One matching `*.result` | If and only if it causes the initial terminal commit, retain status and emit one event |
| Failure while a requested `stop` is finalizing | `capture.stop.result` with the terminal error | Commit `failed` and emit one event |
| Pending `stop` superseded by trusted Discard | `capture.stop.result` with `user-cancelled` | Commit `cancelled` with `reason: "user"` and emit one event |
| Pending `stop` superseded by platform-permission or shell-policy revocation | `capture.stop.result` with `permission-denied` or `policy-denied`, respectively | Commit `cancelled` with `reason: "revoked"` and emit one event |
| An asynchronous terminal failure after successful `start`, with no operation awaiting completion | No request response | Commit `failed` and emit one event; never emit an uncorrelated result |

A pre-session failure and an asynchronous terminal failure are distinct. A
`start` remains pending until trusted consent, platform permission, and encoder
startup settle. If it fails, only its correlated error is sent. Once a
`CaptureSession` exists, a later recorder or device failure is authoritative
through retained `status` and `capture.changed`.

### Examples

Discover formats and limits:

```json
{ "type": "capture.info", "id": "cap-info-1" }
```

```json
{
  "type": "capture.info.result",
  "id": "cap-info-1",
  "info": {
    "sources": ["microphone"],
    "mimeTypes": ["audio/webm;codecs=opus", "audio/mp4"],
    "maxDurationMs": 600000,
    "maxBytes": 52428800,
    "retentionMs": 300000
  }
}
```

Start recording:

```json
{
  "type": "capture.start",
  "id": "cap-start-1",
  "request": {
    "source": "microphone",
    "mimeTypes": ["audio/webm;codecs=opus", "audio/mp4"]
  }
}
```

```json
{
  "type": "capture.start.result",
  "id": "cap-start-1",
  "session": {
    "captureId": "capture_7f2a",
    "source": "microphone",
    "mimeType": "audio/webm;codecs=opus",
    "maxDurationMs": 600000,
    "maxBytes": 52428800,
    "retentionMs": 300000
  }
}
```

Stop and retain the artifact:

```json
{
  "type": "capture.stop",
  "id": "cap-stop-1",
  "captureId": "capture_7f2a"
}
```

```text
{
  "type": "capture.stop.result",
  "id": "cap-stop-1",
  "artifact": {
    "captureId": "capture_7f2a",
    "data": <binary audio/webm 18234 bytes>,
    "mimeType": "audio/webm;codecs=opus",
    "size": 18234,
    "durationMs": 2410,
    "truncated": false,
    "reason": "requested"
  }
}
```

Runtime-driven completion emits a hint:

```json
{
  "type": "capture.changed",
  "event": {
    "captureId": "capture_7f2a",
    "state": "completed"
  }
}
```

The napplet then calls `capture.status`; the completed result contains the
retained `artifact`.

## Runtime Behavior

A conforming runtime:

- MUST own microphone access, device selection, encoding, retention, and teardown.
- MUST authenticate the source endpoint generation, bind each session and
  `captureId` to it, and separately retain the verified napplet identity for
  attribution and policy.
- MUST obtain a fresh capture-specific confirmation immediately before every
  activation. Trusted UI MUST identify the verified requesting napplet using a
  shell-controlled human label, make its publisher, manifest reference, and verified artifact hash
  available, name the microphone source, and state that complete encoded audio
  will be delivered to that napplet and may be copied or sent elsewhere by it.
  A manifest declaration, prior approval, platform permission grant, or unrelated
  user gesture MUST NOT substitute for this confirmation.
- MUST render trusted shell-controlled indication while microphone capture is
  active, including the requesting napplet identity and distinct shell-level
  **Stop** and **Discard** controls with the outcomes defined above.
- MUST keep the indicator active while the platform microphone remains active.
- MUST normalize platform errors and keep low-level device diagnostics local.
- MUST enforce finite duration, artifact-byte, retained-byte, prompt-rate, and
  concurrency policy independently of caller input.
- MUST key prompt-rate policy at least to verified napplet identity so endpoint
  reload does not reset it. The policy MUST define a finite prompt budget within
  a positive time window and MUST be checked before opening or refreshing trusted UI;
  over-budget calls fail with `policy-denied` without showing another capture prompt.
- MUST stop capture promptly when the user, platform, shell policy, or owning
  endpoint lifecycle revokes authority.
- MUST erase runtime-held cancelled, released, expired, failed, and unloaded
  capture bytes and prevent future delivery from those states.
- MUST NOT upload, publish, sign, relay, transcribe, or persist beyond this
  lifecycle as a side effect of capture.

The runtime MAY provide trusted device selection, quality controls, or previews
in shell UI. Those controls do not alter this contract.

## Errors

| Code | Meaning |
|---|---|
| `invalid-request` | The envelope, required field, value type, or MIME syntax is malformed. |
| `policy-denied` | Runtime policy rejected the operation, including prompt/capacity denial or revocation. |
| `user-cancelled` | Trusted UI declined consent or superseded a pending operation with Discard. |
| `permission-denied` | Platform microphone permission was denied or revoked. |
| `device-unavailable` | No usable microphone exists, or an active device was lost without a recoverable artifact. |
| `unsupported-source` | `source` is not `"microphone"`. |
| `unsupported-format` | No advertised output is compatible with an acceptable requested MIME type. |
| `busy` | A pending, recording, finalizing, or retained capture reserves the identity slot, or a runtime-wide limit was reached. |
| `capture-not-found` | The identifier is unknown, expired, stale, or not owned by the authenticated endpoint generation. |
| `invalid-state` | The requested operation is invalid for the observable state or an internal finalization is pending. |
| `quota-exceeded` | No complete valid artifact can be retained within byte policy. |
| `capture-failed` | Capture or encoding failed for another normalized reason. |

A runtime MUST avoid exposing whether a foreign `captureId` exists; use
`capture-not-found` for unknown, expired, stale, and foreign identifiers. Request
validation MUST occur before consent UI. Failures after a session exists follow
the result-and-error routing table and MUST NOT leave input active.

## Security Considerations

Microphone access can expose private speech and ambient audio. This contract
assumes a conforming trusted runtime: a malicious runtime already holds platform
microphone authority and cannot be constrained by napplet messages. Within that
trust model, the runtime is the security boundary and MUST mediate capture as a
visible, revocable capability.

- Platform permission is not napplet consent. A prior platform grant MUST NOT
  authorize silent capture by a newly loaded napplet.
- Consent UI and active-capture indication MUST be rendered by trusted shell UI,
  not supplied solely by the napplet.
- Device identifiers, labels, counts, and permission state MUST NOT cross this API.
- Capture identifiers MUST be unguessable and bound to the authenticated owner
  endpoint generation. Identity equality alone MUST NOT authorize a sibling
  instance, stale frame, or replacement endpoint.
- Runtime limits MUST prevent unbounded recording, retained-byte growth,
  repeated prompt abuse, and concurrent microphone monopolization.
- Completed bytes are sensitive. They MUST be delivered only to the owning
  endpoint generation. `release`, trusted discard, expiry, unload, and revocation
  erase runtime-held copies and prevent future runtime delivery; they cannot
  revoke or erase bytes already delivered to the napplet, copied by it, or sent
  elsewhere.
- Live audio, waveform, and level streaming are excluded. They add continuous
  listening and high-frequency transport semantics not needed here.

## Projection

Each projection MUST define its endpoint-generation lifecycle and the normative
carrier and byte-length operation for `CaptureArtifact.data`. The web binding is
defined by the [web projection](../projections/web.md#capture).

## Non-Goals

NAP-CAPTURE v1 does not define:

- camera, screen, window, tab, system-audio, or mixed-source capture;
- device enumeration, device selection APIs, or persistent device preferences;
- raw audio streams, chunks, waveform data, level meters, or WebRTC tracks;
- pause/resume;
- editing, transcoding, transcription, upload, signing, publishing, or relay use;
- durable recovery after napplet unload;
- background recording after napplet unload; or
- any change to a projection's sandbox policy.

## Implementations

None yet. A runtime prototype demonstrating consent, trusted indication,
binary delivery, terminal retention, limit handling, and unload cleanup is
required before this draft should be considered mergeable.

## Changelog

- `8f69d1e` - Define runtime-owned microphone consent, capture lifecycle, retained artifacts, wire messages, limits, and security rules.
- `3b83f79` - Define automatic change delivery and a language-neutral release result.
- `88b8b49` - Bind captures to endpoint generations and make consent, limits, retention, terminal races, binary delivery, and errors deterministic.
- `ad1c293` - Move web carriers to the projection, preserve codec identifier case, use matching result envelopes, and avoid introducing a canonical byte type.

- `bddbafd` - Adopted injected-domain discovery and verified manifest/artifact attribution while preserving endpoint-generation ownership.
