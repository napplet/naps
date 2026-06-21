NAP-WEBRTC
==========

Runtime-Mediated WebRTC Signaling
---------------------------------

`draft`

**NAP ID:** NAP-WEBRTC
**Domain:** `webrtc`
**Web binding (NIP-5D):** `window.napplet.webrtc` · `shell.supports("webrtc")`

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

```typescript
interface NappletWebrtc {
  open(request: WebrtcOpenRequest): Promise<WebrtcOpenResult>;
  send(sessionId: string, payload: unknown): Promise<void>;
  close(sessionId: string, reason?: string): Promise<void>;
  onEvent(handler: (event: WebrtcEvent) => void): Subscription;
}

type WebrtcScope =
  | { type: "direct"; pubkey: string }
  | { type: "room"; room: string; peers?: string[] };

interface WebrtcOpenRequest {
  scope: WebrtcScope;
  channel?: string;
  protocol?: string;
}

interface WebrtcOpenResult {
  session: WebrtcSession;
}

interface WebrtcSession {
  id: string;
  scope: WebrtcScope;
  channel: string;
  protocol?: string;
  state: "connecting" | "open" | "closed";
}

type WebrtcEvent =
  | { type: "state"; sessionId: string; state: WebrtcSession["state"] }
  | { type: "peer"; sessionId: string; pubkey: string; state: "joined" | "left" }
  | { type: "message"; sessionId: string; from: string; payload: unknown }
  | { type: "closed"; sessionId: string; reason?: string };
```

**`open(request)`** — Requests a WebRTC session. For `direct`, the shell connects
to one peer pubkey. For `room`, the shell announces presence to the room and may
connect to discovered peers or to the requested `peers` subset. `channel` names
the WebRTC data channel; when omitted, the shell chooses a default. `protocol`
is an application-level label for the payloads carried after connection and has
no NAP-WEBRTC semantics.

**`send(sessionId, payload)`** — Sends an opaque payload over the established
session. The shell serializes and delivers it over the WebRTC data channel. This
method does not publish the payload to Nostr.

**`close(sessionId, reason?)`** — Closes the session and asks the shell to publish
any corresponding disconnect signal where appropriate.

**`onEvent(handler)`** — Registers for shell-pushed session events. Message
payloads are untrusted input from peers.

## Wire Protocol

`webrtc.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `webrtc.open` | napplet -> shell | `id`, `request` |
| `webrtc.open.result` | shell -> napplet | `id`, `session`?, `error`? |
| `webrtc.send` | napplet -> shell | `id`, `sessionId`, `payload` |
| `webrtc.send.result` | shell -> napplet | `id`, `error`? |
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
`"payload too large"`, and `"policy denied"`.

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
- Peer payloads are untrusted. Napplets MUST validate every `message` payload
  against their own application protocol before acting on it.
- The shell owns encryption and signing. Napplets MUST NOT receive private keys,
  raw signaling payloads, or decrypted signaling secrets through this interface.
- The shell SHOULD rate-limit `open()` and `send()` to prevent relay spam,
  connection floods, and peer abuse.
- A runtime that supports media streams through a future extension MUST keep
  camera, microphone, and screen capture under user control. NAP-WEBRTC alone
  does not grant media capture authority.

## Implementations

- (none yet)
