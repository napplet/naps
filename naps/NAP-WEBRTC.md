NAP-WEBRTC
==========

Runtime-Mediated WebRTC Signaling
---------------------------------

`draft`

**NAP ID:** NAP-WEBRTC
**Domain:** `webrtc`
**Depends:** none.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.webrtc`; domain presence signals availability.

## Description

NAP-WEBRTC lets a napplet ask the shell to establish a WebRTC peer session whose
signaling is carried over Nostr. The shell owns the signaling mechanism, relay
subscriptions, signing, encryption, SDP offers and answers, ICE candidates, and
`RTCPeerConnection` lifecycle. The napplet names the peer or room it wants to
join and sends or receives opaque application payloads once the session is open.

This is a transport capability, not a call protocol. It does not standardize
voice, video, mute state, chat messages, or application commands. Those semantics
belong in a NAP-N or an application protocol carried over the established
channel.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `open` | `request` (`WebrtcOpenRequest`) | `WebrtcOpenResult` | `webrtc.open` / `webrtc.open.result` |
| `send` | `sessionId` (`tstr`), `payload` (`any`) | none | `webrtc.send` / `webrtc.send.result` |
| `sendBytes` | `sessionId` (`tstr`), `bytes` (`WebrtcBytes`) | none | `webrtc.sendBytes` / `webrtc.sendBytes.result` |
| `close` | `sessionId` (`tstr`), optional `reason` (`tstr`) | none | `webrtc.close` / `webrtc.close.result` |
| `onEvent` | handler for `WebrtcEvent` | `Subscription` handle | `webrtc.event` |

### Schemas

Enums:

| Name | Values |
|------|--------|
| `WebrtcState` | `connecting`, `open`, `closed` |
| `WebrtcPeerState` | `joined`, `left` |

`WebrtcScope` variants:

| Variant | Required fields | Optional fields |
|---------|-----------------|-----------------|
| direct | `type: "direct"`, `pubkey: tstr` | none |
| room | `type: "room"`, `room: tstr` | `peers: list of tstr` |

`WebrtcOpenRequest` fields:

| Field | Required | Type |
|-------|----------|------|
| `scope` | yes | `WebrtcScope` |
| `channel` | no | text |
| `protocol` | no | text |
| `ordered` | no | boolean |
| `maxRetransmits` | no | unsigned integer |

`WebrtcOpenResult` fields:

| Field | Required | Type |
|-------|----------|------|
| `session` | yes | `WebrtcSession` |

`WebrtcSession` fields:

| Field | Required | Type |
|-------|----------|------|
| `id` | yes | text |
| `scope` | yes | `WebrtcScope` |
| `channel` | yes | text |
| `protocol` | no | text |
| `state` | yes | `WebrtcState` |
| `ordered` | no | boolean |
| `maxRetransmits` | no | unsigned integer |
| `maxPayloadBytes` | no | unsigned integer |

`WebrtcEvent` variants:

| Variant | Required fields | Optional fields |
|---------|-----------------|-----------------|
| state | `type: "state"`, `sessionId: tstr`, `state: WebrtcState` | none |
| peer | `type: "peer"`, `sessionId: tstr`, `pubkey: tstr`, `state: WebrtcPeerState` | none |
| message | `type: "message"`, `sessionId: tstr`, `from: tstr`, `payload: any` | none |
| bytes | `type: "bytes"`, `sessionId: tstr`, `from: tstr`, `bytes: WebrtcBytes` | none |
| stats | `type: "stats"`, `sessionId: tstr` | `pubkey: tstr`, `rttMs: number`, `lossRatio: number` |
| closed | `type: "closed"`, `sessionId: tstr` | `reason: tstr` |

**`open(request)`** — Requests a WebRTC session. For `direct`, the shell connects
to one peer pubkey. For `room`, the shell announces presence to the room and may
connect to discovered peers or to the requested `peers` subset. `channel` names
the WebRTC data channel; when omitted, the shell chooses a default. `protocol`
is an application-level label for the payloads carried after connection and has
no NAP-WEBRTC semantics. `ordered` and `maxRetransmits` request delivery
semantics; see [Delivery](#delivery).

**`send(sessionId, payload)`** — Sends an opaque payload over the established
session. The shell serializes and delivers it over the WebRTC data channel. This
method does not publish the payload to Nostr.

**`sendBytes(sessionId, bytes)`** — Sends one opaque byte sequence over the
established session; see [Byte payloads](#byte-payloads). This method does not
publish the bytes to Nostr.

**`close(sessionId, reason?)`** — Closes the session and asks the shell to publish
any corresponding disconnect signal where appropriate.

**`onEvent(handler)`** — Registers for shell-pushed session events. Message
and byte payloads are untrusted input from peers.

### Delivery

`ordered` and `maxRetransmits` let a napplet trade reliability for timeliness.
A napplet that streams short-lived state can ask that a lost payload be skipped
instead of delaying every later one.

| Field | Omitted means | Meaning when present |
|-------|---------------|----------------------|
| `ordered` | `true` | `false` permits delivery out of send order. |
| `maxRetransmits` | reliable delivery | Upper bound on retransmissions of a lost payload. `0` means never retransmit. |

- `WebrtcSession.ordered` and `WebrtcSession.maxRetransmits` report the
  semantics the shell established, not the semantics requested.
- A shell that cannot honor a requested value MUST still open the session with
  the semantics it supports and MUST report them. It MUST NOT fail `open` for
  that reason alone.
- A napplet MUST treat an absent `ordered` as `true` and an absent
  `maxRetransmits` as reliable delivery.
- Delivery semantics are fixed for the life of a session. A napplet that needs
  both reliable and lossy delivery opens two sessions with distinct `channel`
  values.
- The reported semantics apply to the whole session: to every peer and in both
  directions, including peers that join after `open` returns. They do not
  change when peers join or leave.
- The shell MUST convey a session's delivery semantics to peers during
  signaling and MUST connect only peers whose shells established the same
  semantics for that session. A peer whose shell did not is not part of the
  session; the shell MUST NOT emit `peer`, `message`, `bytes`, or `stats`
  events for it.
- A napplet whose reported semantics differ from those it requested can
  therefore reach fewer peers than it expected. Comparing the two is how it
  detects this.

### Byte payloads

`WebrtcBytes` is text containing canonical padded base64 as defined by
[RFC 4648, section 4](https://www.rfc-editor.org/rfc/rfc4648.html#section-4):
standard alphabet, required `=` padding, zero pad bits, no whitespace, and no
data-URL prefix. Empty bytes encode as `""`. Receivers MUST reject non-canonical
encodings.

- The shell MUST deliver the decoded bytes to the peer as one binary payload.
  It MUST NOT send the base64 text.
- A payload sent with `send` MUST be received as a `message` event. A payload
  sent with `sendBytes` MUST be received as a `bytes` event. The shell MUST NOT
  convert between them.
- A shell that does not support byte payloads returns the error
  `"unsupported"`. A shell that predates this operation ignores the request, as
  [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) requires for an
  unrecognized `type`; a napplet SHOULD treat a missing result as unsupported
  and fall back to `send`.

### Limits

`WebrtcSession.maxPayloadBytes`, when present, is the largest single payload
the shell accepts on that session. It counts decoded bytes for `sendBytes` and
serialized payload bytes for `send`. A larger payload fails with
`"payload too large"`.

### Stats

A shell MAY push `stats` events while a session is open, at most once per
second per peer.

| Field | Meaning |
|-------|---------|
| `pubkey` | The peer the values describe. Omitted for a `direct` session. |
| `rttMs` | Current round-trip time estimate to that peer, in milliseconds. |
| `lossRatio` | Fraction of payloads lost over the last reporting interval, `0` to `1`. |

Stats are advisory. A napplet MUST NOT require them.

## Wire Protocol

`webrtc.*` messages use the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `webrtc.open` | napplet -> shell | `id`, `request` |
| `webrtc.open.result` | shell -> napplet | `id`, `session`?, `error`? |
| `webrtc.send` | napplet -> shell | `id`, `sessionId`, `payload` |
| `webrtc.send.result` | shell -> napplet | `id`, `error`? |
| `webrtc.sendBytes` | napplet -> shell | `id`, `sessionId`, `bytes` |
| `webrtc.sendBytes.result` | shell -> napplet | `id`, `error`? |
| `webrtc.close` | napplet -> shell | `id`, `sessionId`, `reason`? |
| `webrtc.close.result` | shell -> napplet | `id`, `error`? |
| `webrtc.event` | shell -> napplet | `event` |

Key design notes:
- Request/result pairs use `id` for correlation.
- `webrtc.event` has no `id`; it is shell-initiated delivery.
- Napplet-facing messages never carry raw SDP, ICE candidates, or raw signaling
  payloads.
- The shell may deliver `webrtc.event` state updates before `open()` resolves if
  the underlying connection progresses quickly, but it MUST NOT deliver messages
  for an unknown `sessionId`.

### Examples

**Open a direct peer channel:**
```
-> {
     "type": "webrtc.open",
     "id": "w1",
     "request": {
       "scope": { "type": "direct", "pubkey": "abc123..." },
       "channel": "chat",
       "protocol": "chat:live"
     }
   }
<- {
     "type": "webrtc.open.result",
     "id": "w1",
     "session": {
       "id": "sess-1",
       "scope": { "type": "direct", "pubkey": "abc123..." },
       "channel": "chat",
       "protocol": "chat:live",
       "state": "connecting"
     }
   }
<- { "type": "webrtc.event", "event": { "type": "state", "sessionId": "sess-1", "state": "open" } }
```

**Send an application payload:**
```
-> {
     "type": "webrtc.send",
     "id": "s1",
     "sessionId": "sess-1",
     "payload": { "body": "hello" }
   }
<- { "type": "webrtc.send.result", "id": "s1" }
```

**Receive a peer payload:**
```
<- {
     "type": "webrtc.event",
     "event": {
       "type": "message",
       "sessionId": "sess-1",
       "from": "abc123...",
       "payload": { "body": "hi" }
     }
   }
```

**Open a lossy, unordered room channel:**
```
-> {
     "type": "webrtc.open",
     "id": "w2",
     "request": {
       "scope": { "type": "room", "room": "k3f9..." },
       "channel": "state",
       "ordered": false,
       "maxRetransmits": 0
     }
   }
<- {
     "type": "webrtc.open.result",
     "id": "w2",
     "session": {
       "id": "sess-2",
       "scope": { "type": "room", "room": "k3f9..." },
       "channel": "state",
       "state": "connecting",
       "ordered": false,
       "maxRetransmits": 0,
       "maxPayloadBytes": 16384
     }
   }
```

**Send and receive bytes `00 01 02 ff`:**
```
-> { "type": "webrtc.sendBytes", "id": "s2", "sessionId": "sess-2", "bytes": "AAEC/w==" }
<- { "type": "webrtc.sendBytes.result", "id": "s2" }
<- {
     "type": "webrtc.event",
     "event": { "type": "bytes", "sessionId": "sess-2", "from": "abc123...", "bytes": "AAEC/w==" }
   }
```

**Receive stats for a room peer:**
```
<- {
     "type": "webrtc.event",
     "event": { "type": "stats", "sessionId": "sess-2", "pubkey": "abc123...", "rttMs": 42, "lossRatio": 0.01 }
   }
```

## Signaling

NIP-100 is an unmerged draft at
<https://github.com/nostr-protocol/nips/pull/363>. It is a useful reference for
WebRTC signaling over Nostr, but it is not part of the NAP-WEBRTC interface.

A conformant shell MAY use NIP-100, a later accepted Nostr signaling NIP, or
another Nostr signaling scheme. That choice is a runtime detail. The napplet
observes only the `webrtc.*` interface, session state, peer state, and opaque
peer messages defined here.

Regardless of signaling scheme, the shell MUST treat raw signaling payloads as
shell-internal transport data, not as napplet-visible API data.

## Error Handling

Any result message MAY include an `error` field (string). When `error` is
present, other result fields are undefined.

Common errors include `"unsupported scope"`, `"signaling unavailable"`,
`"peer unavailable"`, `"session not found"`, `"session closed"`,
`"payload too large"`, `"unsupported"`, and `"policy denied"`.

The shell SHOULD send a `webrtc.event` with `type: "closed"` when an established
session ends because a peer disconnects, ICE fails, signaling expires, or policy
revokes the capability.

## Shell Behavior

- The shell MUST respond to every request with a result message carrying the same
  `id`.
- The shell MUST NOT expose raw relay sockets, signing keys, SDP descriptions,
  ICE candidates, or `RTCPeerConnection` objects to napplets through this NAP.
- The shell MUST perform any signaling signing and encryption itself. It MUST
  NOT delegate keys or raw signing material to the napplet.
- The shell MUST scope sessions per napplet and MUST NOT deliver one napplet's
  peer messages to another napplet.
- The shell MUST enforce per-napplet policy before opening a session, publishing
  signaling events, or delivering peer messages.
- The `from` field of a `message` or `bytes` event and the `pubkey` field of a
  `peer` or `stats` event MUST be the public key the shell authenticated for
  that peer during signaling. The shell MUST NOT take either value from data a
  peer supplied over the data channel. The shell MUST NOT deliver payloads from
  a peer it cannot authenticate.
- The shell MAY use a shell-generated temporary key for a session, including
  when no user is signed in. `from` and `pubkey` identify the key authenticated
  during signaling, which is not necessarily the user's usual Nostr profile
  key.
- The shell MAY limit relays, peers, rooms, payload sizes, data-channel labels,
  connection duration, or concurrent session count.
- The shell MAY route a room session through a WebRTC media server or peer mesh.
  That topology is shell policy and is not visible to the napplet.
- The shell SHOULD publish `disconnect` when `close()` succeeds or when it can
  gracefully tear down a session.

## Security Considerations

- WebRTC can reveal network metadata to peers. Shells MUST treat `open()` as an
  untrusted request and SHOULD require user policy or consent before starting a
  session.
- Signaling room identifiers can be sensitive. Shells SHOULD prefer random,
  non-meaningful room ids and SHOULD avoid exposing private room ids to napplets
  that did not create or join them.
- Peer payloads are untrusted. Napplets MUST validate every `message` and
  `bytes` payload against their own application protocol before acting on it.
- `from` and `pubkey` identify the key a peer authenticated during signaling.
  Authentication establishes control of that key; it does not by itself
  establish an association with the user's usual Nostr profile, because a shell
  MAY use a temporary key. Deciding what that peer may do remains the napplet's
  responsibility.
- Stats are network metadata. Shells MAY round or withhold them and MUST NOT
  include addresses, candidate types, or other values that locate a peer's
  network.
- The shell owns encryption and signing. Napplets MUST NOT receive private keys,
  raw signaling payloads, or decrypted signaling secrets through this interface.
- The shell SHOULD rate-limit `open()` and `send()` to prevent relay spam,
  connection floods, and peer abuse.
- A runtime that supports media streams through a future extension MUST keep
  camera, microphone, and screen capture under user control. NAP-WEBRTC alone
  does not grant media capture authority.

## Implementations

- (none yet)

## Changelog

- `1a83117` - Introduced NAP-WEBRTC as a runtime-owned WebRTC signaling surface.

- `394b597` - Adopted injected-domain availability and linked the current upstream web binding.

- `10793e0` - Added optional delivery semantics, byte payloads, an advertised payload limit, advisory stats, and authenticated peer public keys.
