NAP-MEDIA
=========

Media Session Control
---------------------

`draft`

**NAP ID:** NAP-MEDIA
**Domain:** `media`
**Depends:**
- `resource` — capability · optional — `artwork.url` bytes are fetched via `resource.bytes`
**Web binding (NIP-5D):** `window.napplet.media` · `shell.supports("media")`

## Description

NAP-MEDIA provides media session management between napplets and the shell. Napplets that play audio or video need a way to expose playback state and metadata to the shell so the shell can display media controls (play/pause, skip, volume), aggregate multiple media sources across napplets, and enforce audio policy (e.g., mute on focus loss, duck overlapping sources). Napplets that cannot or should not fetch/play media directly need a way to request shell-owned playback without gaining direct network or audio authority.

The shell provides a media session registry and control surface. Napplets create sessions, report state and metadata, declare dynamic capabilities, and receive commands from the shell when they own playback. For shell-owned playback, the napplet passes a source reference and the shell validates policy, fetches, plays, owns lifecycle, and reports state back to the napplet. Each napplet may have multiple concurrent sessions (e.g., a music player with background playback and a notification sound effect).

## Ownership Model

Every session has a playback owner. The owner is the side that fetches or decodes the media, emits authoritative playback state, and receives playback commands.

- `owner: "shell"` -- The napplet asks the shell to play a source. The shell validates policy, fetches through shell-controlled resource handling, owns the media element/backend, emits state, and may keep playback alive according to its own persistence policy. This is the strict sandbox path: the napplet does not gain direct media or network access.
- `owner: "napplet"` -- The napplet plays media inside its own frame and registers the session so the shell can provide metadata display, media keys, audio focus, OS integration, and commands.

Stronger napplets can still choose shell-owned playback when they want shell policy enforcement, persistent playback, shared player UI, or OS-level integration.

## API Surface

```typescript
interface NappletMedia {
  createSession(options: MediaSessionCreate): Promise<MediaSessionResult>;
  updateSession(sessionId: string, metadata: Partial<MediaMetadata>): void;
  destroySession(sessionId: string): void;
  reportState(sessionId: string, state: MediaState): void;
  reportCapabilities(sessionId: string, actions: MediaAction[]): void;
  onCommand(sessionId: string, cb: (action: MediaAction, value?: number) => void): Subscription;
  onControls(sessionId: string, cb: (controls: MediaAction[]) => void): Subscription;
}

type MediaPlaybackOwner = 'shell' | 'napplet';

interface MediaSessionCreate {
  owner: MediaPlaybackOwner;
  sessionId?: string;
  source?: MediaSourceRef;
  metadata?: MediaMetadata;
  capabilities?: MediaAction[];
  autoplay?: boolean;
  live?: boolean;
}

interface MediaSourceRef {
  url?: string;
  blossomHash?: string;
  nostr?: {
    eventId?: string;
    address?: string;
    relays?: string[];
  };
  mimeType?: string;
}

interface MediaMetadata {
  title?: string;
  artist?: string;
  album?: string;
  artwork?: { url?: string; hash?: string };
  duration?: number;
  mediaType?: 'audio' | 'video';
}
```

**Resource resolution.** The `artwork.url` field is a URL string. Napplets and shells that need the artwork bytes (for example, to render album art on a media controls surface) MUST fetch them through NAP-RESOURCE: `window.napplet.resource.bytes(url)`. The optional `artwork.hash` field, when present, MAY be used by shells as a content-addressed cache key but is not a substitute for the URL fetch — napplets address artwork by URL through the resource NAP. Direct `<img src="https://...">` loads will not work under the iframe sandbox model defined by NIP-5D (`sandbox="allow-scripts"`, no `allow-same-origin`); the shell is the sole network-fetch broker. Standard NAP-RESOURCE policy applies (private-IP block list at DNS-resolution time, MIME byte-sniffing, SVG rasterization, etc.).

```typescript

interface MediaState {
  status: 'playing' | 'paused' | 'stopped' | 'buffering';
  position?: number;
  duration?: number;
  volume?: number;
}

type MediaAction = 'play' | 'pause' | 'stop' | 'next' | 'prev' | 'seek' | 'volume';

interface MediaSessionResult {
  sessionId?: string;
  owner?: MediaPlaybackOwner;
  error?: string;
}

interface Subscription {
  close(): void;
}
```

**`createSession(options)`** -- Creates a new media session. The napplet MUST set `owner` to `"shell"` or `"napplet"`. For shell-owned sessions, `source` is required and the shell fetches/plays it. For napplet-owned sessions, `source` is optional and advisory; the napplet owns playback inside its frame. The napplet MAY provide a preferred `sessionId`, metadata, initial capabilities, `autoplay`, and `live`. Returns the shell-canonical `sessionId` and owner. The shell MAY reject session creation (e.g., invalid source, unsupported owner mode, or session limit exceeded).

**`updateSession(sessionId, metadata)`** -- Updates metadata for an existing session. Partial updates are supported -- only the fields provided are changed. Fire-and-forget.

**`destroySession(sessionId)`** -- Destroys a session. The shell removes it from the media control surface. Fire-and-forget.

**`reportState(sessionId, state)`** -- Reports the current playback state for a napplet-owned session. The playback owner sends state whenever it changes (play/pause/stop/buffer transitions, position updates, volume changes). Fire-and-forget, high frequency during active playback.

**`reportCapabilities(sessionId, actions)`** -- Declares which media actions a napplet-owned session currently supports. Capabilities are dynamic -- a streaming source may not support `seek` initially but add it once buffered. The shell uses this to enable/disable media control buttons. Fire-and-forget.

**`onCommand(sessionId, callback)`** -- Listens for media commands from the shell for a napplet-owned session. The shell sends commands based on its media control UI (user presses play, adjusts volume slider, etc.). The `value` parameter is used for `seek` (position in seconds) and `volume` (0.0 to 1.0). Returns a Subscription with `close()`.

**`onControls(sessionId, callback)`** -- Listens for the shell's control list. The shell tells the napplet which controls the shell supports, so the napplet can adapt its own UI (e.g., hide a next/prev button if the shell does not support it). Returns a Subscription with `close()`.

## Wire Protocol

`media.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `media.session.create` | napplet -> shell | `id`, `owner`, `sessionId?`, `source?`, `metadata?`, `capabilities?`, `autoplay?`, `live?` |
| `media.session.create.result` | shell -> napplet | `id`, `sessionId?`, `owner?`, `error?` |
| `media.session.update` | napplet -> shell | `sessionId`, `metadata` |
| `media.session.destroy` | napplet -> shell | `sessionId` |
| `media.state` | owner -> peer | `sessionId`, `status`, `position?`, `duration?`, `volume?` |
| `media.capabilities` | owner -> peer | `sessionId`, `actions` |
| `media.command` | controller -> owner | `sessionId`, `action`, `value?` |
| `media.controls` | shell -> napplet | `sessionId`, `controls` |

Key design notes:
- `media.session.create` / `media.session.create.result` use `id` for correlation. The napplet MAY provide `sessionId` as a preferred client identifier, but the shell canonicalizes the session id in `media.session.create.result`.
- `owner` is required. Without it, the shell cannot know who fetches media, emits state, receives commands, or owns audio output.
- `source` is required for shell-owned sessions and optional for napplet-owned sessions. A napplet-owned `source` is advisory metadata; it does not ask the shell to fetch or play media.
- `media.session.update`, `media.session.destroy`, `media.state`, and `media.capabilities` are fire-and-forget (no `id` correlation).
- For napplet-owned sessions, `media.state` and `media.capabilities` are napplet -> shell, and `media.command` is shell -> napplet.
- For shell-owned sessions, `media.state` and `media.capabilities` are shell -> napplet, and `media.command` is napplet -> shell when the napplet requests an allowed playback action.
- `media.controls` is shell-initiated. The shell pushes its supported control list so the napplet can adapt its UI.
- Multiple sessions per napplet are supported. Each session is identified by `sessionId`.
- Volume handling is owner-specific. For napplet-owned sessions, the napplet reports its own volume via `media.state`, and the shell can request volume changes via `media.command` with `action: 'volume'`. The effective volume may be the product of napplet volume and shell policy volume. For shell-owned sessions, the shell owns output volume and reports it in `media.state`.

### Session ID Canonicalization

The napplet MAY include `sessionId` in `media.session.create` as a stable client-generated hint. The shell MUST return the canonical `sessionId` in `media.session.create.result`. The canonical id MAY equal the napplet-supplied value, or it MAY be rewritten to avoid collisions, enforce namespace policy, or bind the id to the napplet identity.

All later messages MUST use the canonical `sessionId`. Messages that use the pre-canonical hint after creation SHOULD be treated as unknown-session messages.

### Source References

`source` identifies media the shell may play for a shell-owned session. It supports direct URLs, Blossom content hashes, and Nostr event/address references. Shells MAY support a subset of source forms and reject unsupported forms during session creation.

For `owner: "shell"`, `source` is required unless the shell defines a separate out-of-band source binding for the napplet. The shell MUST fetch and validate source bytes through shell-controlled policy. The napplet does not gain direct network access.

For `owner: "napplet"`, `source` is optional metadata. The shell MUST NOT treat a napplet-owned `source` as permission to fetch or play media on the napplet's behalf.

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
```

**Session creation rejected:**
```
<- { "type": "media.session.create.result", "id": "m1", "sessionId": "s1", "error": "session limit exceeded" }
```

### Error Handling

`media.session.create.result` includes an `error` field (string) on failure. Common errors: `"session limit exceeded"`, `"invalid session ID"`. When `error` is present, the session was not created.

For shell-owned sessions, common creation errors also include `"missing source"`, `"unsupported source"`, `"source blocked"`, and `"autoplay denied"`.

Messages referencing an unknown `sessionId` (e.g., `media.state` for a destroyed session) MUST be silently ignored by both napplet and shell.

Commands that are not valid for the current owner mode, source, or capabilities SHOULD be silently ignored or rejected using an implementation-defined diagnostic channel. They MUST NOT cause a session to switch owner.

## Shell Behavior

- The shell MUST accept `media.session.create` messages and respond with `media.session.create.result` carrying the same `id`.
- The shell MUST reject `media.session.create` messages without `owner`.
- The shell MUST reject `owner: "shell"` creation when `source` is missing or blocked by policy.
- The shell MUST return the canonical `sessionId` in `media.session.create.result`.
- The shell MUST track active sessions per napplet and remove them on `media.session.destroy` or when the napplet iframe is removed.
- The shell SHOULD display media controls (play/pause, skip, volume) for active sessions based on the napplet's reported capabilities.
- The shell SHOULD update its media control UI in response to `media.state` messages.
- The shell MAY send `media.command` messages to control napplet playback based on user interaction with the shell's media controls.
- The shell MAY send `media.controls` to inform the napplet which controls the shell supports. The napplet can adapt its own UI accordingly.
- The shell MAY enforce audio policy (e.g., mute sessions on focus loss, duck overlapping sources, enforce a single-active-output policy).
- The shell SHOULD make audio focus decisions across all sessions, regardless of owner.
- For shell-owned sessions, the shell MUST own fetching, playback lifecycle, state emission, and policy enforcement.
- For napplet-owned sessions, the shell MUST NOT fetch/play `source` on the napplet's behalf unless the napplet creates a separate shell-owned session.
- When the owning napplet/window closes, the shell MUST destroy napplet-owned sessions and SHOULD stop shell-owned sessions unless the shell explicitly promotes the session to persistent playback.
- If shell-promoted persistent playback is allowed, the shell MUST detach the persistent session from the closed iframe before destroying the napplet's message endpoint, and MUST define how the user can stop or revoke that playback.
- The shell MAY limit the number of concurrent sessions per napplet.
- The shell MAY enforce ACL checks on `media` capabilities (e.g., restricting which napplets can create media sessions).
- The shell MUST silently ignore messages from unknown session IDs.
- The shell MUST clean up all sessions for a napplet when the napplet's iframe is removed.

## Command Validity

For napplet-owned sessions:

- The shell SHOULD only send commands listed in the napplet's current `media.capabilities`.
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
- Artwork URLs are fetched by the shell, not the napplet (the napplet is sandboxed with no network access). The shell SHOULD validate URLs and enforce a content security policy for fetched resources.
- Shell-owned `source` URLs are fetched and played by the shell, not the napplet. The shell MUST apply the same resource safety policy it applies to artwork and other external bytes.
- Blossom artwork hashes allow the shell to resolve artwork through its own Blossom infrastructure without the napplet needing network access. The shell controls which Blossom servers are queried.
- For napplet-owned sessions, volume control is advisory -- the napplet controls actual audio output. A malicious napplet could ignore volume commands. The shell MAY enforce volume limits at the iframe level using the Web Audio API or iframe attribute policies if available. For shell-owned sessions, the shell controls actual output volume.
- Session creation is rate-limited by the shell. A napplet that creates excessive sessions can be throttled or denied.
- The `media.command` message allows the shell to control napplet behavior. The napplet trusts the shell (NIP-5D security model) but SHOULD validate the `action` against its declared capabilities.

## Implementations

- (none yet)
