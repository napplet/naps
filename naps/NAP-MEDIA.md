NAP-MEDIA
=========

Media Session Control
---------------------

`draft`

**NAP ID:** NAP-MEDIA
**Domain:** `media`
**Depends:**
- `resource` — capability · optional — `artwork.url` bytes are fetched via `resource.bytes`
- `torrent` — layering · optional — shell-owned playback resolves an authorized job file through the runtime's torrent engine, preserving [NAP-TORRENT](https://github.com/napplet/naps/pull/72) verification, selection, quotas, and retention rules.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.media`; domain presence signals availability.

## Description

NAP-MEDIA provides media session management between napplets and the shell. Napplets that play audio or video need a way to expose playback state and metadata to the shell so the shell can display media controls (play/pause, skip, volume), aggregate multiple media sources across napplets, and enforce audio policy (e.g., mute on focus loss, duck overlapping sources). Napplets that cannot or should not fetch/play media directly need a way to request shell-owned playback without gaining direct network or audio authority.

The shell provides a media session registry and control surface. Napplets create sessions, report state and metadata, declare dynamic capabilities, and receive commands from the shell when they own playback. For shell-owned playback, the napplet passes a source reference and the shell validates policy, fetches, plays, owns lifecycle, and reports state back to the napplet. Each napplet may have multiple concurrent sessions (e.g., a music player with background playback and a notification sound effect).

## Ownership Model

Every session has a playback owner. The owner is the side that fetches or decodes the media, emits authoritative playback state, and receives playback commands.

- `owner: "shell"` -- The napplet asks the shell to play a source. The shell validates policy, fetches through shell-controlled resource handling, owns the media element/backend, emits state, and may keep playback alive according to its own persistence policy. This is the strict sandbox path: the napplet does not gain direct media or network access.
- `owner: "napplet"` -- The napplet plays media inside its own frame and registers the session so the shell can provide metadata display, media keys, audio focus, OS integration, and commands.

Stronger napplets can still choose shell-owned playback when they want shell policy enforcement, persistent playback, shared player UI, or OS-level integration.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `createSession` | `options` (`MediaSessionCreate`) | `MediaSessionResult` | `media.session.create` / `media.session.create.result` |
| `updateSession` | `sessionId` (`text`), partial `metadata` (`MediaMetadata`) | none | `media.session.update` |
| `destroySession` | `sessionId` (`text`) | none | `media.session.destroy` |
| `reportState` | `sessionId` (`text`), `state` (`MediaState`) | none | `media.state` |
| `reportCapabilities` | `sessionId` (`text`), `actions` (list of `MediaAction`) | none | `media.capabilities` |
| `command` | `sessionId` (`text`), command (`MediaCommand`) | none | `media.command` |
| `onState` | `sessionId` (`text`), handler for `MediaState` | `Subscription` handle | `media.state` |
| `onCapabilities` | `sessionId` (`text`), handler for list of `MediaAction` | `Subscription` handle | `media.capabilities` |
| `onCommand` | `sessionId` (`text`), handler for `MediaCommand` | `Subscription` handle | `media.command` |
| `onControls` | `sessionId` (`text`), handler for list of `MediaAction` | `Subscription` handle | `media.controls` |
| `onEnded` | `sessionId` (`text`), handler for `MediaSessionEnded` | `Subscription` handle | `media.session.ended` |

### Schemas

| Name | Values |
|------|--------|
| `MediaPlaybackOwner` | "shell" or "napplet" |
| `MediaAction` | "play", "pause", "stop", "next", "prev", "seek", or "volume" |

`MediaSessionCreate` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `owner` | yes | MediaPlaybackOwner | — |
| `sessionId` | no | text | — |
| `source` | no | MediaSourceRef | — |
| `metadata` | no | MediaMetadata | — |
| `context` | no | MediaSessionContext | — |
| `capabilities` | no | list of MediaAction | — |
| `autoplay` | no | boolean | — |
| `live` | no | boolean | — |

`MediaSourceRef` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `url` | no | text | — |
| `blossomHash` | no | text | — |
| `nostr` | no | MediaNostrRef | — |
| `mimeType` | no | text | Advisory hint; the shell validates the actual media format. |
| `torrent` | no | MediaTorrentSource | Optional source form; exclusive of url, blossomHash, and nostr. |

`MediaTorrentSource` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `jobId` | yes | text | Existing job identifier resolved within the caller's verified napplet principal and user profile. |
| `fileIndex` | yes | integer | Non-negative, stable index from the job's verified file metadata. |

`MediaNostrRef` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `eventId` | no | text | — |
| `address` | no | text | — |
| `relays` | no | list of text | — |

`MediaArtwork` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `url` | no | text | — |
| `hash` | no | text | — |

`MediaMetadata` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `title` | no | text | — |
| `artist` | no | text | — |
| `album` | no | text | — |
| `artwork` | no | MediaArtwork | — |
| `duration` | no | number | — |
| `mediaType` | no | "audio" or "video" | — |

`MediaSessionContext` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `label` | no | text | — |
| `detail` | no | text | — |
| `index` | no | integer | Non-negative. |
| `total` | no | integer | Non-negative. |
| `links` | no | list of MediaContextLink | — |

`MediaContextLink` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `rel` | yes | text | — |
| `title` | no | text | — |
| `nostr` | no | MediaNostrRef | — |

**Resource resolution.** The `artwork.url` field is a URL string. Napplets and shells that need the artwork bytes (for example, to render album art on a media controls surface) MUST fetch them through NAP-RESOURCE: `window.napplet.resource.bytes(url)`. The optional `artwork.hash` field, when present, MAY be used by shells as a content-addressed cache key but is not a substitute for the URL fetch — napplets address artwork by URL through the resource NAP. Artwork URLs do not grant direct network access. The web projection's runtime
network policy governs external loads; an opaque sandbox origin alone does not
block network requests. If the optional `resource` domain is unavailable,
napplets SHOULD omit remote artwork or use local placeholders. Standard NAP-RESOURCE policy applies (private-IP block list at DNS-resolution time, MIME byte-sniffing, SVG rasterization, etc.).

`MediaState` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `status` | yes | "playing", "paused", "stopped", or "buffering" | Playback state within an active session. |
| `position` | no | number | Playback position in seconds. |
| `duration` | no | number | Duration in seconds. |
| `volume` | no | number | Volume from 0.0 to 1.0. |
| `reason` | no | text | Cause of this transition; see Transition reasons. |

`MediaCommand` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `action` | yes | `MediaAction` | Requested playback action. |
| `value` | no | number | Seek position or volume, as required by the action. |
| `reason` | no | text | Cause of the command; does not grant authority. |

`MediaSessionEnded` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `sessionId` | yes | text | Canonical identifier of the retired session. |
| `reason` | yes | text | Why the shell retired the session. |

`MediaSessionResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `sessionId` | conditional | text | Required on successful creation. |
| `owner` | conditional | `MediaPlaybackOwner` | Required on successful creation. |
| `error` | no | text | Creation failure; no session was created. |

**`createSession(options)`** -- Creates a new media session. The napplet MUST set `owner` to `"shell"` or `"napplet"`. For shell-owned sessions, `source` is required and the shell fetches/plays it. For napplet-owned sessions, `source` is optional and advisory; the napplet owns playback inside its frame. The napplet MAY provide a preferred `sessionId`, metadata, context, initial capabilities, `autoplay`, and `live`. Returns the shell-canonical `sessionId` and owner. The shell MAY reject session creation (e.g., invalid source, unsupported owner mode, or session limit exceeded).

**`updateSession(sessionId, metadata)`** -- Updates metadata for an existing session. Partial updates are supported -- only the fields provided are changed. Fire-and-forget.

**`destroySession(sessionId)`** -- Requests session retirement. The shell removes
it from the media control surface and sends `media.session.ended` with reason
`destroyed`. The request is fire-and-forget; the terminal notification has no
request correlation id.

**`reportState(sessionId, state)`** -- Reports the current playback state for a napplet-owned session. The playback owner sends state whenever it changes, including transition reasons when known (see Transition reasons). Fire-and-forget, high frequency during active playback.

**`reportCapabilities(sessionId, actions)`** -- Declares which media actions a napplet-owned session currently supports. Capabilities are dynamic -- a streaming source may not support `seek` initially but add it once buffered. The shell uses this to enable/disable media control buttons. Fire-and-forget.

**`onCommand(sessionId, callback)`** -- Listens for media commands from the shell for a napplet-owned session. The shell sends commands based on user interaction or audio policy; `reason` distinguishes their causes. The `value` parameter is used for `seek` (position in seconds) and `volume` (0.0 to 1.0). Returns a Subscription with `close()`.

**`onControls(sessionId, callback)`** -- Listens for the shell's control list. The shell tells the napplet which controls the shell supports, so the napplet can adapt its own UI (e.g., hide a next/prev button if the shell does not support it). Returns a Subscription with `close()`.

**`command(sessionId, command)`** — Requests an allowed action on shell-owned
playback. It cannot revive a retired session or change playback ownership.

**`onState(sessionId, callback)`** and **`onCapabilities(sessionId, callback)`** —
Receive authoritative state and supported actions for shell-owned playback.

**`onEnded(sessionId, callback)`** — Receives terminal retirement for either
owner mode. The binding MUST retain the terminal record until the napplet
endpoint is destroyed. A handler registered after retirement MUST receive it
once. Retirement MUST stop further state, capability, control, and command
callbacks for that session. A napplet-owned player MUST stop its output and
release session resources when it receives retirement.

## Wire Protocol

`media.*` messages use the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `media.session.create` | napplet -> shell | `id`, `owner`, `sessionId?`, `source?`, `metadata?`, `context?`, `capabilities?`, `autoplay?`, `live?` |
| `media.session.create.result` | shell -> napplet | `id`, `sessionId?`, `owner?`, `error?` |
| `media.session.update` | napplet -> shell | `sessionId`, `metadata` |
| `media.session.destroy` | napplet -> shell | `sessionId` |
| `media.session.ended` | shell -> napplet | `sessionId`, `reason` |
| `media.state` | owner -> peer | `sessionId`, `status`, `position?`, `duration?`, `volume?`, `reason?` |
| `media.capabilities` | owner -> peer | `sessionId`, `actions` |
| `media.command` | controller -> owner | `sessionId`, `action`, `value?`, `reason?` |
| `media.controls` | shell -> napplet | `sessionId`, `controls` |

Key design notes:
- `media.session.create` / `media.session.create.result` use `id` for correlation. The napplet MAY provide `sessionId` as a preferred client identifier, but the shell canonicalizes the session id in `media.session.create.result`.
- `owner` is required. Without it, the shell cannot know who fetches media, emits state, receives commands, or owns audio output.
- `source` is required for shell-owned sessions and optional for napplet-owned sessions. A napplet-owned `source` is advisory metadata; it does not ask the shell to fetch or play media.
- `context` is optional session association metadata. It can describe where this media item sits in a list (`label`, `detail`, `index`, `total`) and can link related resources such as a Nostr event, address, or live-chat target. Context does not grant fetch authority and MUST NOT change session ownership.
- `media.session.update`, `media.session.destroy`, `media.state`, and `media.capabilities` are fire-and-forget (no `id` correlation).
- For napplet-owned sessions, `media.state` and `media.capabilities` are napplet -> shell, and `media.command` is shell -> napplet.
- For shell-owned sessions, `media.state` and `media.capabilities` are shell -> napplet, and `media.command` is napplet -> shell when the napplet requests an allowed playback action.
- `media.session.ended` is a shell-initiated terminal notification with no `id` correlation.
- `media.controls` is shell-initiated. The shell pushes its supported control list so the napplet can adapt its UI.
- Multiple sessions per napplet are supported. Each session is identified by `sessionId`.
- Volume handling is owner-specific. For napplet-owned sessions, the napplet reports its own volume via `media.state`, and the shell can request volume changes via `media.command` with `action: 'volume'`. The effective volume may be the product of napplet volume and shell policy volume. For shell-owned sessions, the shell owns output volume and reports it in `media.state`.

### Session ID Canonicalization

The napplet MAY include `sessionId` in `media.session.create` as a stable client-generated hint. The shell MUST return the canonical `sessionId` in `media.session.create.result`. The canonical id MAY equal the napplet-supplied value, or it MAY be rewritten to avoid collisions, enforce namespace policy, or bind the id to the napplet identity.

All later messages MUST use the canonical `sessionId`. Messages that use the pre-canonical hint after creation SHOULD be treated as unknown-session messages.

### Transition reasons

`reason` explains a transition or command; it does not determine session
existence. Only `media.session.ended` retires a session. Missing or unrecognized
reasons MUST be treated as unspecified causes, while still processing the
message's status, action, or terminal meaning. Receivers MUST NOT infer natural
completion or permission to resume from `stopped` or `paused` alone.

| Reason | Meaning |
|--------|---------|
| `user` | The user acted in the shell, player, or napplet UI. |
| `ended` | The current source completed naturally. |
| `error` | Playback failed. |
| `preempted` | Another session permanently took this session's output. |
| `focus-lost` | The shell or platform temporarily withdrew audio focus. |
| `focus-gained` | The shell permits playback to resume after focus returns. |
| `destroyed` | The napplet requested session destruction. |
| `backend-exited` | The playback backend exited without a more specific known cause. |
| `policy` | The shell revoked the session under its policy. |
| `source-unavailable` | The bound source was removed or permanently became unusable; for torrent sources this retires the session. |

Playback owners SHOULD include the known reason on state transitions. A natural
completion that leaves the session usable MUST be reported as `status: "stopped"`
with `reason: "ended"`; user stop and playback failure use `user` and `error`
respectively when known. An ended track does not by itself retire the session.
If the shell retires the session for that completion instead, it MUST send
`media.session.ended` with reason `ended` as the sole notification of that
completion. Consumers MUST NOT interpret other reasons as natural completion.

For an active session, a later `play` remains subject to advertised capabilities
and shell policy. An empty capability list disables actions; it MUST NOT be used
as a substitute for session retirement.

### Session retirement and preemption

The shell MAY retire either owner mode under policy. It MUST retire a session
when it permanently reallocates that session's output to another session, or
when its playback backend exits and it will not preserve a usable session.
Permanent output replacement uses reason `preempted`. Backend exit uses the
known cause (`user`, `ended`, or `error`), or `backend-exited` when unknown.

Before notifying retirement, the shell MUST remove the session from dispatch
and stop or revoke its output authority. It MUST send exactly one
`media.session.ended` for that session to its still-live napplet endpoint.
Any already-enqueued state, capability, control, or command messages MUST
precede the terminal notification. The shell MUST NOT send further state, capabilities, controls,
or commands for the retired session. The binding MUST discard late messages
for it and retain the terminal record for `onEnded`.

For a newly created session, a successful creation result MUST precede any
terminal notification. Failed creation produces no retirement notification.
Retired identifiers MUST NOT be reused during the napplet endpoint's lifetime,
even when a later creation requests the same preferred identifier.

Messages targeting retired sessions, including `play`, MUST be silently ignored
as unknown-session messages. They MUST NOT reacquire output or preempt another
session. Playing again requires successful creation of a new session. The shell
MUST continue enforcing consent, autoplay, and audio-focus policy on creation.

Endpoint destruction requires cleanup but no notification to the dead endpoint.
Shell-promoted persistent playback MUST first detach from the destroyed endpoint;
its former session identifier MUST NOT become usable by a replacement endpoint.

### Temporary audio-focus loss

Temporary focus loss keeps the session alive. For shell-owned playback, the
shell MUST report a focus-induced pause as `media.state` with `status: "paused"`
and `reason: "focus-lost"`. For napplet-owned playback, the shell MUST identify
a focus-induced pause command with `reason: "focus-lost"`. The napplet MUST pause
and report its actual state. The reason does not transfer playback ownership.
A napplet-owned session MUST support `pause` if it produces pausable output;
otherwise the shell MUST revoke output and retire the session under policy.

A napplet MUST NOT resume automatically merely because its window regains focus.
For napplet-owned playback, automatic resumption requires a shell `play` command
with `reason: "focus-gained"`. The shell MAY issue that command only when the
session was playing before the focus-induced pause and no later user pause,
stop, retirement, or playback failure superseded it. The napplet MUST ignore
that automatic-resume command if its own subsequent user action or playback
failure superseded the focus pause. For shell-owned playback, the shell applies
the same conditions and reports resumed playback with reason `focus-gained`.
User-requested play remains subject to ordinary shell policy. A focus transition
MUST NOT revive a retired session.

### Source References

`source` identifies media the shell may play for a shell-owned session. It supports direct URLs, Blossom content hashes, Nostr event/address references, and optional torrent job-file selectors. Shells MAY support a subset of source forms and reject unsupported forms during session creation.

For `owner: "shell"`, `source` is required unless the shell defines a separate out-of-band source binding for the napplet. The shell MUST fetch and validate source bytes through shell-controlled policy. The napplet does not gain direct network access.

For `owner: "napplet"`, `source` is optional metadata. The shell MUST NOT treat a napplet-owned `source` as permission to fetch or play media on the napplet's behalf.

### Torrent file sources

For shell-owned playback, `source.torrent` selects one file in an existing
[NAP-TORRENT](https://github.com/napplet/naps/pull/72) job. `MediaTorrentSource`
is owned by this NAP. It embeds no external wire record and carries no grant.
The `torrent` dependency is optional shell-internal composition. Supporting
this source does not require exposing torrent operations to every media caller.
A caller that creates or manages its own jobs separately declares and uses the
`torrent` capability under its own policy.

A source containing `torrent` MUST NOT also contain `url`, `blossomHash`, or
`nostr`. `mimeType` remains an advisory format hint. The shell MUST reject
ambiguous selectors with `invalid source`. Existing source forms keep their
contracts. For `owner: "napplet"`, the selector is advisory metadata only:
it MUST NOT cause shell job resolution, reading, or playback.

**Admission and authority.** The shell MUST resolve the job using the original
requesting endpoint's verified napplet principal and user profile. It MUST NOT
accept caller-supplied ownership or resolve the identifier under an ambient
shell/service identity with wider access. A user-approved session-scoped job
alias may be used only within its existing delegated authority. Copying a job
identifier between napplets MUST NOT transfer access. Unknown or inaccessible
jobs return the same `source not found` error.

The shell MUST independently authorize playback of the selected content.
Permission to seed local files is not permission to play them. A playback
grant MUST NOT grant raw-byte access to the napplet, expand file selection,
change ownership, or authorize a different job. The session binds to the
resolved job identity and stable file index; it MUST NOT silently rebind if
an alias disappears or another job uses a similar identifier.

The shell MUST validate verified torrent metadata and an existing selected
file before successful creation. Missing metadata returns `source not ready`;
an invalid index returns `invalid source`; an unselected file or denied
playback grant returns `source blocked`. The caller may retry after resolving
the condition through the torrent domain. Creating a media session MUST NOT
create, resume, reconfigure, or select files in a torrent job.

Torrent files are finite sources; `live: true` is invalid and returns
`invalid source`. Complete file data is not required for creation. A successful
creation result confirms an authorized binding, not successful decoding. It
MUST precede all state and capability notifications for that session. For this
source, omitted `autoplay` means false: the initial state is `paused`. With
permitted autoplay, report `buffering` until enough verified media data is
available, then `playing`. A shell lacking this source resolver returns
`unsupported source`; domain presence alone does not promise source support.
No new discovery operation is required before attempting creation.

**Playback and seeking.** The runtime's internal reader MUST preserve torrent
integrity checks, selection, paused-job restrictions, bandwidth limits, and
per-principal quotas. It may consume verified cached data from a paused job,
but MUST NOT resume its network activity. This is a shell-internal reader;
it need not pass encoded byte messages through the napplet or use the
endpoint-scoped public `torrent.read` request IDs. Any quota accounting MUST
remain attached to the originating principal, including detached playback.

The shell owns media probing, decoding, and mapping a seek in seconds to byte
ranges. It MUST validate the actual container and codecs; `mimeType` cannot
bypass format checks. Unsupported media known before creation returns
`unsupported media`. An unrecoverable format or source-integrity failure
discovered after creation retires the session with reason `error` under the
terminal-ordering rules. A corrupt peer piece rejected and retried by the
torrent engine is not by itself a terminal playback failure.
Implementations MUST bound probing, buffering, and outstanding range demand
under runtime policy. While playback is requested, temporary starvation uses
`buffering`; exhaustion of the shell's bounded stall/probe policy retires the
session with reason `error`. The stall timer MUST NOT count time spent in an
explicit media pause or stop. Data arrival, transfer pause/resume, or a seek
MUST NOT override the user's media pause/stop or restart output automatically.

The shell MUST advertise only playback actions it can perform for this source.
An advertised seek may need to buffer while verified data arrives; it need
not wait until the whole file is downloaded to be advertised. Each accepted
seek MUST cancel obsolete demand and replace only this session's temporary
range priorities. It MUST NOT alter persistent file selection, another
session's reader, or the torrent's configured priorities. Pausing/stopping
playback, focus-induced pause, or retirement releases that temporary demand.
Pausing playback MUST NOT implicitly pause the torrent or stop seeding.
`play` after a pause or stop remains subject to normal capabilities and policy.

**Source and session lifetime.** Destroying or retiring a media session releases
only its reader, temporary priorities, and playback grant. It MUST NOT remove
the job, delete its data, or revoke another reader. Torrent download completion
is not playback completion. Reaching media end-of-file uses the existing
natural-completion rules; it does not stop seeding or remove the torrent.

Removing the job, losing its source permanently, losing the delegated alias,
or deselecting the selected file MUST retire dependent media sessions with
reason `source-unavailable`. Revoking playback or source authorization MUST
stop output, cancel reads, discard buffered output, and retire the session
with reason `policy`. Previously buffered bytes MUST NOT continue playing
after revocation or source retirement. During requested playback, transient
lack of peers or a paused transfer is a buffering condition, not natural
completion. An explicitly paused or stopped media session keeps its state. Notification
ordering, late-message suppression, and stale-command rejection follow
Session retirement and preemption.

Shell promotion to persistent playback requires an already-persistent source
job and explicit authorization to retain the playback binding beyond endpoint
closure. A session-retained job or endpoint-scoped delegated alias cannot
support that promotion. The shell MUST NOT silently change job retention or
preserve an expiring grant. If the prerequisites are absent, playback stops
and the session is retired on endpoint closure. If promotion is admitted,
the shell detaches the playback binding before destroying the endpoint,
retains its original principal/profile restrictions, and rechecks revocation.
Recovering a retained torrent job does not restore an old media session ID;
a replacement napplet endpoint must create a new media session.

**Composition boundary.** This source is resolved inside the runtime; it does
not introduce a `torrent:` URL scheme, extend `resource.bytes` to support
ranges, or return a transferable cross-napplet file reference. The existing
resource dependency remains for artwork. Browser/native streaming mechanisms
are implementation choices; support for torrent files does not imply support
for every media container or codec.

### Session Context

`context` describes the media session around the source. It is for UI, queue position, and related-resource affordances. A shell MAY ignore any context field it does not understand.

`context.links[]` entries identify related resources by relation name. Relation names are lower-case strings. Implementations MAY define relation names such as `live-chat` or `source-event`, but unknown relation names MUST be ignored.

Nostr links use `MediaNostrRef`. `nostr.address` MAY be an address coordinate such as `30311:<pubkey>:<d-tag>`. `nostr.eventId` MAY identify an event. `nostr.relays` are optional routing hints. Absence of `nostr.relays` MUST NOT make the link invalid; the runtime resolves the Nostr reference through its normal routing policy.

### Metadata

All metadata fields are optional. The `artwork` field supports two forms:

- `url` -- A direct URL to the artwork image.
- `hash` -- A Blossom hash (SHA-256) that the shell can resolve via its configured Blossom servers. If both are provided, the shell MAY prefer either.

```json
{
  "title": "Song Title",
  "artist": "Artist Name",
  "album": "Album Name",
  "artwork": { "url": "https://example.com/cover.jpg", "hash": "abc123..." },
  "duration": 240,
  "mediaType": "audio"
}
```

### Examples

**Create a session:**
```
-> { "type": "media.session.create", "id": "m1", "owner": "napplet", "sessionId": "s1", "metadata": { "title": "My Song", "artist": "The Artist" } }
<- { "type": "media.session.create.result", "id": "m1", "sessionId": "s1", "owner": "napplet" }
```

**Create a shell-owned playback session:**
```
-> { "type": "media.session.create", "id": "m2", "owner": "shell", "source": { "url": "https://example.com/live.mp3", "mimeType": "audio/mpeg" }, "metadata": { "title": "Live Stream" }, "autoplay": true, "live": true }
<- { "type": "media.session.create.result", "id": "m2", "sessionId": "shell-7", "owner": "shell" }
```

**Create a shell-owned live stream with a chat association:**
```
-> { "type": "media.session.create", "id": "m3", "owner": "shell", "source": { "url": "https://example.com/live.mp3", "mimeType": "audio/mpeg" }, "metadata": { "title": "Live Stream" }, "context": { "links": [{ "rel": "live-chat", "title": "Live Stream", "nostr": { "address": "30311:pubkey:d-tag" } }] }, "autoplay": true, "live": true }
<- { "type": "media.session.create.result", "id": "m3", "sessionId": "shell-8", "owner": "shell" }
```

**Update metadata:**
```
-> { "type": "media.session.update", "sessionId": "s1", "metadata": { "title": "Updated Title" } }
```

**Report playback state:**
```
-> { "type": "media.state", "sessionId": "s1", "status": "playing", "position": 42.5, "duration": 240, "volume": 0.8 }
```

**Report capabilities:**
```
-> { "type": "media.capabilities", "sessionId": "s1", "actions": ["play", "pause", "seek", "volume"] }
```

**Shell sends a command:**
```
<- { "type": "media.command", "sessionId": "s1", "action": "pause" }
<- { "type": "media.command", "sessionId": "s1", "action": "seek", "value": 120 }
<- { "type": "media.command", "sessionId": "s1", "action": "volume", "value": 0.5 }
```

**Napplet requests a command on shell-owned playback:**
```
-> { "type": "media.command", "sessionId": "shell-7", "action": "pause" }
```

**Shell reports state for shell-owned playback:**
```
<- { "type": "media.state", "sessionId": "shell-7", "status": "playing", "position": 12.0, "volume": 0.7 }
```

**Shell sends control list:**
```
<- { "type": "media.controls", "sessionId": "s1", "controls": ["play", "pause", "stop", "next", "prev", "seek", "volume"] }
```

**Destroy a session:**
```
-> { "type": "media.session.destroy", "sessionId": "s1" }
<- { "type": "media.session.ended", "sessionId": "s1", "reason": "destroyed" }
```

**Track completes; the session remains usable:**
```
<- { "type": "media.state", "sessionId": "shell-7", "status": "stopped", "reason": "ended" }
```

**Another session takes the shared player; the old session is retired:**
```
<- { "type": "media.session.ended", "sessionId": "shell-7", "reason": "preempted" }
-> { "type": "media.command", "sessionId": "shell-7", "action": "play" }
```
The stale command is silently ignored. A new session is required to play again.

**Temporary focus loss and permitted resumption of napplet-owned playback:**
```
<- { "type": "media.command", "sessionId": "s2", "action": "pause", "reason": "focus-lost" }
-> { "type": "media.state", "sessionId": "s2", "status": "paused", "reason": "focus-lost" }
<- { "type": "media.command", "sessionId": "s2", "action": "play", "reason": "focus-gained" }
```
The resume command is valid only if no intervening user action or failure
superseded the focus pause.

**Session creation rejected:**
```
<- { "type": "media.session.create.result", "id": "m1", "sessionId": "s1", "error": "session limit exceeded" }
```

**Create shell-owned torrent playback and seek into missing data:**

The job is already authorized, its metadata is verified, and file 0 is selected. The shell supports its container/codecs. Pausing and destroying playback do not pause or remove the torrent job.

```
-> {"type": "media.session.create", "id": "tm1", "owner": "shell", "source": {"torrent": {"jobId": "torrent-7", "fileIndex": 0}}, "autoplay": true}
<- {"type": "media.session.create.result", "id": "tm1", "sessionId": "torrent-player-1", "owner": "shell"}
<- {"type": "media.capabilities", "sessionId": "torrent-player-1", "actions": ["play", "pause", "stop", "seek", "volume"]}
<- {"type": "media.state", "sessionId": "torrent-player-1", "status": "buffering"}
<- {"type": "media.state", "sessionId": "torrent-player-1", "status": "playing", "position": 0}
-> {"type": "media.command", "sessionId": "torrent-player-1", "action": "seek", "value": 120}
<- {"type": "media.state", "sessionId": "torrent-player-1", "status": "buffering", "position": 120}
<- {"type": "media.state", "sessionId": "torrent-player-1", "status": "playing", "position": 120}
-> {"type": "media.command", "sessionId": "torrent-player-1", "action": "pause"}
<- {"type": "media.state", "sessionId": "torrent-player-1", "status": "paused", "reason": "user"}
-> {"type": "media.session.destroy", "sessionId": "torrent-player-1"}
<- {"type": "media.session.ended", "sessionId": "torrent-player-1", "reason": "destroyed"}
```

**Wait for metadata before retrying creation:**

The caller retries with a new request ID after metadata becomes available. Omitted autoplay leaves the admitted session paused.

```
-> {"type": "media.session.create", "id": "tm2", "owner": "shell", "source": {"torrent": {"jobId": "torrent-7", "fileIndex": 0}}}
<- {"type": "media.session.create.result", "id": "tm2", "error": "source not ready"}
-> {"type": "media.session.create", "id": "tm3", "owner": "shell", "source": {"torrent": {"jobId": "torrent-7", "fileIndex": 0}}}
<- {"type": "media.session.create.result", "id": "tm3", "sessionId": "torrent-player-2", "owner": "shell"}
<- {"type": "media.state", "sessionId": "torrent-player-2", "status": "paused"}
```

**Revoke source authorization during buffered playback:**

The shell stops output and discards buffered content before retirement. The subsequent stale command is ignored.

```
-> {"type": "media.command", "sessionId": "torrent-player-2", "action": "play"}
<- {"type": "media.state", "sessionId": "torrent-player-2", "status": "buffering"}
<- {"type": "media.session.ended", "sessionId": "torrent-player-2", "reason": "policy"}
-> {"type": "media.command", "sessionId": "torrent-player-2", "action": "play"}
```

**Reject an inaccessible job without revealing ownership:**

```
-> {"type": "media.session.create", "id": "tm4", "owner": "shell", "source": {"torrent": {"jobId": "other-job", "fileIndex": 0}}}
<- {"type": "media.session.create.result", "id": "tm4", "error": "source not found"}
```

**Reject an unsupported resolver:**

```
-> {"type": "media.session.create", "id": "tm5", "owner": "shell", "source": {"torrent": {"jobId": "torrent-7", "fileIndex": 0}}}
<- {"type": "media.session.create.result", "id": "tm5", "error": "unsupported source"}
```

**Retire playback when the torrent job is removed:**

This session had already been admitted. Job removal terminates its binding; buffered bytes do not keep it alive.

```
<- {"type": "media.session.ended", "sessionId": "torrent-player-3", "reason": "source-unavailable"}
```

### Error Handling

`media.session.create.result` includes an `error` field (string) on failure. Common errors: `"session limit exceeded"`, `"invalid session ID"`. When `error` is present, the session was not created.

For shell-owned sessions, common creation errors also include `"missing source"`, `"unsupported source"`, `"source blocked"`, and `"autoplay denied"`.

Torrent-source creation also defines `"invalid source"`, `"source not found"`,
`"source not ready"`, and `"unsupported media"`. These are machine-readable
values, not arbitrary diagnostics. An error creates no session and MUST NOT
be followed by state, capability, or retirement notifications for that attempt.

Messages referencing an unknown `sessionId` (e.g., `media.state` for a destroyed session) MUST be silently ignored by both napplet and shell.

Commands that are not valid for the current owner mode, source, or capabilities SHOULD be silently ignored or rejected using an implementation-defined diagnostic channel. They MUST NOT cause a session to switch owner.

## Shell Behavior

- The shell MUST accept `media.session.create` messages and respond with `media.session.create.result` carrying the same `id`.
- The shell MUST reject `media.session.create` messages without `owner`.
- The shell MUST reject `owner: "shell"` creation when `source` is missing or blocked by policy.
- The shell MUST return the canonical `sessionId` in `media.session.create.result`.
- The shell MUST track active sessions per napplet and apply Session retirement
  and preemption on destruction, output replacement, and terminal backend exit.
- The shell SHOULD display media controls (play/pause, skip, volume) for active sessions based on the napplet's reported capabilities.
- The shell SHOULD update its media control UI in response to `media.state` messages.
- The shell MAY send `media.command` messages to control napplet playback based on user interaction with the shell's media controls.
- The shell MAY send `media.controls` to inform the napplet which controls the shell supports. The napplet can adapt its own UI accordingly.
- The shell MAY enforce audio policy (e.g., mute sessions on focus loss, duck overlapping sources, enforce a single-active-output policy).
- The shell SHOULD make audio focus decisions across all sessions, regardless of owner.
- For shell-owned sessions, the shell MUST own fetching, playback lifecycle, state emission, and policy enforcement.
- Torrent sources MUST follow Torrent file sources for verified reading,
  independent playback permission, source failure, and persistence admission.
- For napplet-owned sessions, the shell MUST NOT fetch/play `source` on the napplet's behalf unless the napplet creates a separate shell-owned session.
- When the owning napplet/window closes, the shell MUST destroy napplet-owned sessions and SHOULD stop shell-owned sessions unless the shell explicitly promotes the session to persistent playback.
- If shell-promoted persistent playback is allowed, the shell MUST detach the persistent session from the closed iframe before destroying the napplet's message endpoint, and MUST define how the user can stop or revoke that playback.
- The shell MAY limit the number of concurrent sessions per napplet.
- The shell MAY enforce ACL checks on `media` capabilities (e.g., restricting which napplets can create media sessions).
- The shell MUST silently ignore messages from unknown session IDs.
- On endpoint removal, the shell MUST clean up all sessions still bound to it.
  Explicitly detached persistent playback follows the persistence policy above.

## Command Validity

For napplet-owned sessions:

- The shell SHOULD only send commands listed in the napplet's current
  `media.capabilities`. Focus withdrawal follows Temporary audio-focus loss;
  unsupported pause MUST NOT leave revoked output playing.
- The napplet SHOULD ignore commands it does not currently support.
- `seek` MUST include `value` as a position in seconds.
- `volume` MUST include `value` from `0.0` to `1.0`.

For shell-owned sessions:

- The napplet MAY send `media.command` only for actions exposed by the shell through `media.controls` or `media.capabilities`.
- The shell MUST enforce source policy, autoplay policy, audio focus, and output volume.
- The shell MAY ignore `seek` when the source is live or not seekable.
- The shell MAY ignore `next` and `prev` unless the source represents a playlist or queue.
- The shell MUST NOT expose raw source bytes to the napplet as part of command handling.

## Security Considerations

- Media sessions expose metadata and playback state to the shell. Napplets SHOULD NOT include sensitive information in metadata fields.
- Session context is napplet-supplied metadata. Shells SHOULD validate Nostr references before using them for UI or routing and MUST ignore unknown context links.
- Artwork URLs are fetched by the shell, not the napplet (the napplet is sandboxed with no network access). The shell SHOULD validate URLs and enforce a content security policy for fetched resources.
- Shell-owned `source` URLs are fetched and played by the shell, not the napplet. The shell MUST apply the same resource safety policy it applies to artwork and other external bytes.
- Torrent selectors MUST be resolved under the original caller's authority.
  They MUST NOT bypass content verification, expose local filesystem paths,
  or turn seeding permission into playback or raw-byte permission.
- Blossom artwork hashes allow the shell to resolve artwork through its own Blossom infrastructure without the napplet needing network access. The shell controls which Blossom servers are queried.
- For napplet-owned sessions, volume control is advisory -- the napplet controls actual audio output. A malicious napplet could ignore volume commands. The shell MAY enforce volume limits at the iframe level using the Web Audio API or iframe attribute policies if available. For shell-owned sessions, the shell controls actual output volume.
- Session creation is rate-limited by the shell. A napplet that creates excessive sessions can be throttled or denied.
- Transition reasons are explanatory metadata, not authorization. The shell
  MUST NOT let a caller-supplied reason bypass ownership, capability, autoplay,
  or audio-focus checks.
- The `media.command` message allows the shell to control napplet behavior. The napplet trusts the shell ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) security model) but SHOULD validate the `action` against its declared capabilities.

## Implementations

- (none yet)

## Changelog

- `d3adc87` - Introduced NAP-MEDIA for media session control.
- `7eeac9f` - Added media session context associations for linking playback state to external context.

- `df41999` - Adopted injected-domain availability and clarified mediated artwork loading and optional-resource fallback.
- `bd521c6` - Added playback and command reasons, terminal session notifications, stale-command rejection, ordered retirement delivery, and audio-focus resumption rules; aligned the operations table with the wire contract.
- `451cc68` - Added optional torrent job-file sources for shell-owned playback with scoped authorization, verified buffering and seeking, source retirement, and persistence admission independent of torrent transfer control.
