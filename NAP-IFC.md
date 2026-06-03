NAP-IFC
=======

Inter-Frame Communication
-------------------------

`draft`

**NAP ID:** NAP-IFC
**Namespace:** `window.napplet.ifc`
**Discovery:** `shell.supports("ifc")`

## Description

NAP-IFC provides topic-based publish/subscribe and point-to-point channels for communication between napplets. Sandboxed iframes cannot communicate directly because the `allow-same-origin` sandbox token is absent — they have opaque origins with no shared context. The shell routes messages between napplets using typed `ifc.*` messages over the NIP-5D wire format. Topic-based IFC is loose coupling: the sender does not know who (if anyone) receives the message. Channel-based IFC is tight coupling: a napplet opens a named connection to a specific peer and the shell validates the target once on open. The shell also uses IFC topics for internal coordination (state operations, service commands, configuration).

## API Surface

```typescript
interface NappletIfc {
  // Topic-based pub/sub
  emit(topic: string, payload?: unknown): void;         // via ifc.emit
  on(topic: string, callback: (event: IfcEvent) => void): Subscription;  // via ifc.subscribe

  // Point-to-point channels
  channel: {
    open(target: string): Promise<ChannelHandle>;     // via ifc.channel.open
    list(): Promise<ChannelInfo[]>;                    // via ifc.channel.list
    broadcast(payload: unknown): void;                 // via ifc.channel.broadcast
  };
}

interface IfcEvent {
  topic: string;
  sender: string;    // sender dTag (napplet type identifier, per NIP-5D)
  payload: unknown;
}

interface Subscription {
  close(): void;     // via ifc.unsubscribe
}

interface ChannelHandle {
  readonly id: string;           // shell-assigned channel ID
  readonly peer: string;         // peer dTag
  emit(payload: unknown): void;  // via ifc.channel.emit
  on(callback: (event: ChannelEvent) => void): Subscription;
  close(): void;                 // via ifc.channel.close
}

interface ChannelEvent {
  channelId: string;
  sender: string;    // sender dTag
  payload: unknown;
}

interface ChannelInfo {
  id: string;
  peer: string;      // peer dTag
}
```

**`emit(topic, payload?)`** — Broadcasts a message to all napplets subscribed to the given topic. Fire-and-forget — there is no delivery confirmation. The shell identifies the sender via `MessageEvent.source` (per NIP-5D) and includes the sender's `dTag` in delivered events.

**`on(topic, callback)`** — Subscribes to messages on a topic. The callback receives an `IfcEvent` with the topic, sender `dTag`, and payload. Returns a `Subscription` handle with a `close()` method to unsubscribe. Multiple subscriptions to the same topic are independent.

**`channel.open(target)`** — Opens a point-to-point channel to a napplet identified by its dTag. The shell validates the target and checks ACL on open. Returns a `ChannelHandle` on success. If the target is not found or ACL-denied, the promise rejects. The shell validates once on open — subsequent messages flow without per-message checking.

**`channel.list()`** — Returns the list of active channels for this napplet.

**`channel.broadcast(payload)`** — Sends a message to all open channel peers at once. Fire-and-forget.

**`ChannelHandle.emit(payload)`** — Sends a message to the channel peer. Fire-and-forget.

**`ChannelHandle.on(callback)`** — Receives messages from the channel peer. The callback receives a `ChannelEvent` with `channelId`, sender `dTag`, and `payload`.

**`ChannelHandle.close()`** — Tears down the channel. Both sides are notified via `ifc.channel.closed`.

## Wire Protocol

IFC operations use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

### Topics

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `ifc.emit` | napplet -> shell | `topic`, `payload`? |
| `ifc.subscribe` | napplet -> shell | `id`, `topic` |
| `ifc.subscribe.result` | shell -> napplet | `id` |
| `ifc.unsubscribe` | napplet -> shell | `topic` |
| `ifc.event` | shell -> napplet | `topic`, `sender` (dTag), `payload`? |

Key design notes:

- `ifc.emit` has no `id` field — it is fire-and-forget with no acknowledgment.
- `ifc.subscribe` uses `id` for correlation so the shim can confirm the subscription was registered.
- `ifc.subscribe.result` confirms registration by echoing the `id`.
- `ifc.unsubscribe` is fire-and-forget (no `id`, no result message).
- `ifc.event` has no `id` field — it is a shell-initiated delivery identified by `topic` and `sender` (the emitting napplet's `dTag` string per NIP-5D).

### Channels

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `ifc.channel.open` | napplet -> shell | `id`, `target` (dTag string) |
| `ifc.channel.open.result` | shell -> napplet | `id`, `channelId`?, `peer`? (dTag), `error`? |
| `ifc.channel.emit` | napplet -> shell | `channelId`, `payload`? |
| `ifc.channel.event` | shell -> napplet | `channelId`, `sender` (dTag), `payload`? |
| `ifc.channel.broadcast` | napplet -> shell | `payload`? |
| `ifc.channel.list` | napplet -> shell | `id` |
| `ifc.channel.list.result` | shell -> napplet | `id`, `channels` (ChannelInfo array) |
| `ifc.channel.close` | napplet -> shell | `channelId` |
| `ifc.channel.closed` | shell -> napplet | `channelId`, `reason`? (string) |

Key design notes:

- `ifc.channel.open` uses `id` for correlation; the result returns a shell-assigned `channelId`.
- `ifc.channel.emit` has no `id` field — fire-and-forget like `ifc.emit`.
- `ifc.channel.event` has no `id` field — shell-initiated delivery, carries `channelId` + `sender`.
- `ifc.channel.broadcast` has no `id` field — fire-and-forget to all open channel peers.
- `ifc.channel.close` has no `id` — the `channelId` identifies the target channel.
- `ifc.channel.closed` is a shell-initiated notification sent on graceful close, peer destruction, or ACL revocation.

### Examples

**Emit:**
```
-> { "type": "ifc.emit", "topic": "profile:open", "payload": { "pubkey": "abc123..." } }
```
No response — fire-and-forget.

**Subscribe:**
```
-> { "type": "ifc.subscribe", "id": "a1", "topic": "profile:open" }
<- { "type": "ifc.subscribe.result", "id": "a1" }
```

**Event delivery:**
```
<- { "type": "ifc.event", "topic": "profile:open", "sender": "social-feed", "payload": { "pubkey": "abc123..." } }
```

**Unsubscribe:**
```
-> { "type": "ifc.unsubscribe", "topic": "profile:open" }
```

**Open channel:**
```
-> { "type": "ifc.channel.open", "id": "ch1", "target": "media-player" }
<- { "type": "ifc.channel.open.result", "id": "ch1", "channelId": "c-abc", "peer": "media-player" }
```

**Channel emit:**
```
-> { "type": "ifc.channel.emit", "channelId": "c-abc", "payload": { "command": "play", "track": 3 } }
```
No response — fire-and-forget.

**Channel event delivery:**
```
<- { "type": "ifc.channel.event", "channelId": "c-abc", "sender": "media-player", "payload": { "status": "playing", "track": 3 } }
```

**Channel broadcast:**
```
-> { "type": "ifc.channel.broadcast", "payload": { "announcement": "shutting down" } }
```

**List channels:**
```
-> { "type": "ifc.channel.list", "id": "l1" }
<- { "type": "ifc.channel.list.result", "id": "l1", "channels": [{ "id": "c-abc", "peer": "media-player" }, { "id": "c-def", "peer": "chat-widget" }] }
```

**Close channel:**
```
-> { "type": "ifc.channel.close", "channelId": "c-abc" }
<- { "type": "ifc.channel.closed", "channelId": "c-abc" }
```

**Channel closed by peer or shell:**
```
<- { "type": "ifc.channel.closed", "channelId": "c-abc", "reason": "peer destroyed" }
```

**Open error:**
```
<- { "type": "ifc.channel.open.result", "id": "ch1", "error": "target not found" }
```

### Error Handling

`ifc.subscribe.result` MAY include an `error` field (string) if the shell rejects the subscription. When `error` is present, the subscription was not registered.

```
<- { "type": "ifc.subscribe.result", "id": "a1", "error": "topic rejected by ACL" }
```

`ifc.channel.open.result` MAY include an `error` field if the channel cannot be opened. When `error` is present, no channel was created.

## Topic Conventions

Topics use a prefix convention to signal direction and scope:

| Prefix | Direction | Meaning |
|--------|-----------|---------|
| `shell:*` | napplet -> shell | Commands sent by a napplet to the shell (e.g., `shell:state-get`) |
| `napplet:*` | shell -> napplet | Responses/notifications from shell to napplet (e.g., `napplet:state-response`) |
| `{domain}:*` | bidirectional | Domain-scoped messages between napplets (e.g., `profile:open`, `chat:open-dm`) |

These conventions are advisory. The shell routes by topic match, not by prefix parsing. A napplet can subscribe to any topic regardless of prefix.

## Shell Behavior

### Topic routing

- The shell MUST route `ifc.emit` messages to all napplets subscribed to the matching topic.
- The shell MUST identify the sender via `MessageEvent.source` and include the sender's `dTag` in delivered `ifc.event` messages (per NIP-5D identity model).
- The shell MUST NOT deliver `ifc.event` back to the emitting napplet (sender exclusion).
- The shell MUST respond to `ifc.subscribe` with `ifc.subscribe.result` carrying the same `id`.
- The shell MUST honor `ifc.unsubscribe` by removing the subscription for that topic.
- The shell MAY intercept specific topic prefixes (e.g., `shell:*`) for internal command handling rather than routing them to other napplets.
- The shell MAY enforce ACL checks on IFC capabilities and reject subscriptions or emits that violate shell policy.

### Channel management

- The shell MUST respond to `ifc.channel.open` with `ifc.channel.open.result` carrying the same `id`.
- The shell MUST assign a `channelId` (opaque identifier) on successful channel open.
- The shell MUST validate that the target dTag exists and has a live iframe before opening a channel.
- The shell MUST forward `ifc.channel.emit` messages to the channel peer as `ifc.channel.event`.
- The shell MUST NOT perform per-message validation on channel messages after open — only the initial open is validated once (auth-on-open model).
- The shell MUST send `ifc.channel.closed` to both sides when a channel is torn down (graceful close, iframe removal, or ACL revocation).
- The shell MUST respond to `ifc.channel.list` with `ifc.channel.list.result` listing the napplet's active channels.
- The shell MUST deliver `ifc.channel.broadcast` to all open channel peers of the sender, excluding the sender itself.
- The shell MUST clean up channel state when a napplet iframe is destroyed, sending `ifc.channel.closed` with `reason: "peer destroyed"` to the surviving endpoint.
- The shell MAY enforce ACL checks on `ifc.channel.open` (e.g., restrict which dTags a napplet can open channels to).
- The shell MAY impose a maximum number of concurrent channels per napplet.

## Channels vs Topics

Topics and channels serve different communication patterns within NAP-IFC:

**Topics** are loose-coupled publish/subscribe. Any napplet can subscribe to any topic; the sender does not know who (if anyone) will receive the message. There is no persistent connection. Topics are well-suited for infrequent coordination — UI commands, state sync, configuration events, and notifications.

**Channels** are pre-authorized point-to-point connections. A napplet calls `channel.open(target)` to establish a channel to a specific peer identified by its dTag. The shell validates the target once on open (auth-on-open model): it checks that the target exists, has a live iframe, and passes ACL. After open, messages flow between the two endpoints without per-message shell validation. Channels are well-suited for sustained data streams, real-time collaboration, and command channels between specific napplets.

Both are part of the same `ifc` namespace and share the NIP-5D wire format. A napplet may use both simultaneously.

## Security Considerations

Sender identity is shell-enforced via `MessageEvent.source` mapping to napplet identity (per NIP-5D). There is no per-message signing — the shell's sender identification is the trust boundary. `MessageEvent.source` is unforgeable within the same browsing context.

### Topics

- Topic namespaces are not enforced — any napplet can emit on any topic. The shell MAY restrict topics via ACL.
- Payloads are opaque to the shell. Receiving napplets are responsible for validating payload content.
- Sender exclusion prevents echo loops but does not prevent a napplet from emitting messages on any topic. Receivers should check the `sender` dTag if sender identity matters for their use case.

### Channels

- Channel authorization is validated once on `ifc.channel.open`. After open, messages flow without per-message shell validation. A compromised browser extension could forge messages on an open channel — this is an accepted trust boundary, the same as topic-based IFC.
- Channel IDs are opaque. Napplets cannot enumerate or guess other channels' IDs. This prevents channel hijacking.
- `ifc.channel.broadcast` reaches all open channel peers. Napplets SHOULD NOT send sensitive data via broadcast. For confidential communication, use a named channel with a specific target.
