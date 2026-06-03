NAP-NN-TOPIC-PAYLOADS
=====================

Topic Payload Semantics
-----------------------

`draft`

**NAP ID:** NAP-NN-TOPIC-PAYLOADS (final number assigned on merge)
**Domain:** Napplet-to-napplet navigation and context topics
**Requires:** NAP-IFC
**Discovery:** `shell.supports("ifc", "NAP-NN-TOPIC-PAYLOADS")`

## Description

This NAP defines a small set of topic payload semantics for napplets that
coordinate navigation and contextual views through NAP-IFC topic pub/sub. NAP-IFC
defines how topic messages move between napplets; this NAP defines what selected
topic names mean and what payloads producers and consumers agree to exchange.

## Boundary

NAP-IFC owns the generic transport messages:

- `ifc.emit`
- `ifc.subscribe`
- `ifc.unsubscribe`
- `ifc.event`

This NAP does not redefine those envelopes. It defines semantic contracts for
specific `topic` values carried by NAP-IFC. A shell can support NAP-IFC without
supporting this protocol, and a napplet can use NAP-IFC topics not listed here
for private or application-specific behavior.

## Message Protocol

Napplets coordinate through NAP-IFC topic messages. The transport envelope is:

```json
{ "type": "ifc.emit", "topic": "<topic>", "payload": { "...": "..." } }
```

Consumers receive:

```json
{ "type": "ifc.event", "topic": "<topic>", "sender": "<sender>", "payload": { "...": "..." } }
```

Producers SHOULD send JSON-serializable object payloads. Consumers SHOULD ignore
malformed payloads, unknown fields, and unsupported topics. Consumers MUST NOT
throw protocol errors across the IFC dispatch path for malformed payloads.

Unless a topic says otherwise, public keys are 64-character lowercase hex Nostr
public keys.

### `profile:open`

Requests that a profile-capable napplet load a user profile.

Payload:

```json
{ "pubkey": "<hex-pubkey>" }
```

Producer behavior:

- Send when the user activates an author, avatar, mention, or profile affordance.
- Include exactly one `pubkey`.

Consumer behavior:

- If the consumer can show profiles, load the requested profile.
- If the requested profile is already open, the consumer MAY focus or refresh it.
- If the payload is missing `pubkey` or the pubkey is invalid, ignore it.

### `chat:open-dm`

Requests that a chat-capable napplet open a direct message view with a user.

Payload:

```json
{
  "pubkey": "<hex-pubkey>",
  "displayName": "optional display name"
}
```

Producer behavior:

- Send when the user chooses to start or continue a direct conversation.
- Include `displayName` only as a display hint; it is not authoritative identity.

Consumer behavior:

- Open or focus a direct-message view for `pubkey`.
- Use `displayName` only as a temporary label until trusted metadata is loaded.
- Ignore malformed payloads.

### `livestream:channel-switch`

Announces that a stream-capable napplet selected a livestream channel and gives
chat-capable napplets enough context to open the related stream chat.

Payload:

```json
{
  "streamId": "<stream-coordinate>",
  "streamUrl": "optional media URL",
  "metadata": {
    "title": "optional title",
    "chatRelays": ["wss://relay.example"],
    "image": "optional image URL",
    "hostPubkey": "optional hex pubkey"
  }
}
```

Producer behavior:

- Send when the active livestream selection changes.
- Set `streamId` to the stream address or other stable stream coordinate.
- Include `metadata.chatRelays` when known.

Consumer behavior:

- Open, focus, or update the stream chat context for `streamId`.
- Treat `metadata` fields as hints.
- Ignore messages without `streamId`.

### `stream:current-context-get`

Requests the active stream context from stream-capable napplets.

Payload:

```json
{ "requestId": "optional correlation id" }
```

Producer behavior:

- A requester SHOULD subscribe to `stream:current-context` before emitting this
  topic.
- `requestId` is optional. If present, responders SHOULD echo it.

Consumer behavior:

- A stream-capable napplet SHOULD respond by emitting `stream:current-context`.
- If no stream is active, respond with an empty context payload rather than
  omitting the response.

### `stream:current-context`

Reports the active stream context.

Payload:

```json
{
  "streamAddr": "<stream-coordinate-or-null>",
  "title": "optional title or null",
  "chatRelays": ["wss://relay.example"],
  "requestId": "optional echoed correlation id"
}
```

Producer behavior:

- Send in response to `stream:current-context-get`.
- Set `streamAddr` to `null` when no stream is active.
- Set `chatRelays` to an empty array when no relay hints are available.

Consumer behavior:

- Use `streamAddr` and `chatRelays` to open, focus, or update stream chat.
- If `streamAddr` is `null`, treat the response as "no active stream."
- If the requester supplied `requestId`, ignore responses with a different
  `requestId`.

## Non-Goals

This NAP does not define:

- Shell window management, workspace placement, or focus policy.
- `wm:*` shell telemetry topics.
- `shell:*` host command topics.
- Deprecated identity topics such as `auth:identity-changed`.
- The stale `stream:channel-switch` alias. New implementations SHOULD use
  `livestream:channel-switch` for the channel-switch behavior above.
- Media playback handoff. Use NAP-MEDIA or NAP-PLAYER as appropriate.

## Negotiation

A napplet that requires these topic semantics declares NAP-IFC in its manifest:

```
["requires", "ifc"]
```

At runtime it checks for this protocol:

```js
window.napplet.shell.supports("ifc", "NAP-NN-TOPIC-PAYLOADS")
```

After this draft receives a final number, implementations should check that
assigned number instead of the provisional identifier.

## Implementations

- (none yet)
