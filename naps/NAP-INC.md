NAP-INC
========

Inter-Napplet Communication
---------------------------

`draft`

**NAP ID:** NAP-INC
**Domain:** `inc`
**Web binding (NIP-5D):** `window.napplet.inc` · `shell.supports("inc")`

## Description

NAP-INC provides topic-based publish/subscribe and point-to-point channels for communication between napplets. The contract is inter-napplet rather than inter-frame: NIP-5D uses sandboxed iframes and `postMessage` today, but the INC API describes shell-mediated communication between napplet endpoints and does not require every implementation target to be modeled as a browser frame.

Under the NIP-5D iframe transport, sandboxed napplets cannot communicate directly because the `allow-same-origin` sandbox token is absent -- they have opaque origins with no shared context. The shell routes messages between napplets using typed `inc.*` messages over the NIP-5D wire format. Topic-based INC is loose coupling: the sender does not know who (if anyone) receives the message. Channel-based INC is tight coupling: a napplet opens a named connection to a specific peer and the shell validates the target once on open. The shell MAY also use INC topics for internal coordination (state operations, service commands, configuration).

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `emit` | `topic` or convention URI (`tstr`), optional `payload` (`any`) | none | `inc.emit` |
| `on` | `topic` (`tstr`), event handler for `IncEvent` | `Subscription` handle | `inc.subscribe` / `inc.subscribe.result` |
| `channel.open` | `target` (`tstr`, peer dTag) | `ChannelHandle` | `inc.channel.open` / `inc.channel.open.result` |
| `channel.onOpened` | handler for inbound `ChannelHandle` | `Subscription` handle | `inc.channel.opened` |
| `channel.list` | none | list of `ChannelInfo` | `inc.channel.list` / `inc.channel.list.result` |
| `channel.broadcast` | `payload` (`any`) | none | `inc.channel.broadcast` |
| `ChannelHandle.emit` | `payload` (`any`) | none | `inc.channel.emit` |
| `ChannelHandle.on` | event handler for `ChannelEvent` | `Subscription` handle | `inc.channel.event` |
| `ChannelHandle.onClosed` | handler for `ChannelClosed` | `Subscription` handle | `inc.channel.closed` |
| `ChannelHandle.close` | none | none | `inc.channel.close` |

### Schemas

`IncEvent` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `topic` | yes | text | Opaque topic name. Matched by exact string equality. |
| `sender` | yes | text | Runtime-attested emitting napplet dTag. |
| `payload` | no | any | Per-message data. |

`ChannelHandle` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | text | Shell-assigned channel id. |
| `peer` | yes | text | Peer dTag. |

`ChannelEvent` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `channelId` | yes | text | Shell-assigned channel id. |
| `sender` | yes | text | Sender dTag. |
| `payload` | no | any | Channel payload. |

`ChannelClosed` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `channelId` | yes | text | Shell-assigned channel id. |
| `reason` | no | text | Runtime-supplied terminal reason. |

`ChannelInfo` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | text | Shell-assigned channel id. |
| `peer` | yes | text | Peer dTag. |

`sender` and `peer` are napplet `dTag` values. `id` and `channelId` are
shell-assigned opaque identifiers.

**`emit(topic, payload?)`** — Broadcasts a message to all napplets subscribed to
the given topic. A napplet MAY supply a convention URI as `topic`; the runtime
transposes its query parameters into `payload` before routing. Fire-and-forget —
there is no delivery confirmation. The runtime derives `sender` from the
authenticated emitting endpoint and includes its dTag in delivered events. The
emitter cannot set or override `sender`.

**`on(topic, callback)`** — Subscribes to messages on a topic. The callback receives an `IncEvent` with the topic, sender `dTag`, and payload. Returns a `Subscription` handle with a `close()` method to unsubscribe. Multiple subscriptions to the same topic are independent.

### Convention URI transposition

The developer-facing convention URI is
`napplet:<archetype>/<intent>[...?params]`. Its path is the stable topic. Its
query is shallow payload sugar.

When `emit` receives a convention URI with a query, the runtime MUST:

1. Remove the query from the routed topic.
2. Percent-decode each unique `name=value` pair as text.
3. Place those pairs in a map of text to text and use that map as `payload`.

The runtime MUST NOT coerce query values to boolean, number, or null. A `+` is a
literal plus sign, not a space. A convention URI with a fragment, malformed
percent-encoding, a repeated name, or an explicit `payload` argument alongside
query parameters MUST be rejected before emission. Structured or non-text data
MUST use the explicit `payload` argument with a queryless topic.

This transposition is part of the `emit` operation. Topic routing still uses
exact equality over the resulting stable topic.

**`channel.open(target)`** — Opens a point-to-point channel to a napplet
identified by its dTag. The shell validates the target and checks ACL on open.
The opener receives a `ChannelHandle`. Before reporting success, the runtime
MUST enqueue `inc.channel.opened` for the target. If the target is not found,
has no live endpoint, cannot receive the notification, or is ACL-denied, the
request fails. The shell validates once on open. Subsequent messages flow
without per-message authorization.

**`channel.onOpened(callback)`** — Receives a `ChannelHandle` when a peer opens
a channel to this napplet. The handle identifies the same channel as the
opener's handle and supports the same `emit`, `on`, `onClosed`, and `close`
operations. Its `peer` is the runtime-attested opener dTag. This notification is
not an authorization prompt. The runtime has already applied channel ACL.

**`channel.list()`** — Returns informational snapshots of this napplet's active
inbound and outbound channels. `ChannelInfo` is not a handle and does not attach
the caller to a channel.

**`channel.broadcast(payload)`** — Sends a message to all open channel peers at once. Fire-and-forget.

**`ChannelHandle.emit(payload)`** — Sends a message to the channel peer. Fire-and-forget.

**`ChannelHandle.on(callback)`** — Receives messages from the channel peer. The callback receives a `ChannelEvent` with `channelId`, sender `dTag`, and `payload`.

**`ChannelHandle.onClosed(callback)`** — Receives the terminal `ChannelClosed`
record when either endpoint closes the channel or the runtime tears it down. A
handler registered after closure MUST be invoked with the retained terminal
record.

**`ChannelHandle.close()`** — Tears down the channel. Either endpoint may close.
Both handles become terminal and their `onClosed` handlers are notified via
`inc.channel.closed`.

## Wire Protocol

INC operations use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

### Topics

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `inc.emit` | napplet -> shell | `topic`, `payload`? |
| `inc.subscribe` | napplet -> shell | `id`, `topic` |
| `inc.subscribe.result` | shell -> napplet | `id` |
| `inc.unsubscribe` | napplet -> shell | `topic` |
| `inc.event` | shell -> napplet | `topic`, `sender` (dTag), `payload`? |

Key design notes:

- `inc.emit` has no `id` field — it is fire-and-forget with no acknowledgment.
- `inc.emit` carries the stable topic and transposed payload, not the developer-facing convention URI.
- `inc.subscribe` uses `id` for correlation so the shim can confirm the subscription was registered.
- `inc.subscribe.result` confirms registration by echoing the `id`.
- `inc.unsubscribe` is fire-and-forget (no `id`, no result message).
- `inc.event` has no `id` field — it is a shell-initiated delivery identified by `topic` and `sender` (the emitting napplet's `dTag` string per NIP-5D).

### Channels

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `inc.channel.open` | napplet -> shell | `id`, `target` (dTag string) |
| `inc.channel.open.result` | shell -> napplet | `id`, `channelId`?, `peer`? (dTag), `error`? |
| `inc.channel.opened` | shell -> target napplet | `channelId`, `peer` (dTag) |
| `inc.channel.emit` | napplet -> shell | `channelId`, `payload`? |
| `inc.channel.event` | shell -> napplet | `channelId`, `sender` (dTag), `payload`? |
| `inc.channel.broadcast` | napplet -> shell | `payload`? |
| `inc.channel.list` | napplet -> shell | `id` |
| `inc.channel.list.result` | shell -> napplet | `id`, `channels` (ChannelInfo array) |
| `inc.channel.close` | napplet -> shell | `channelId` |
| `inc.channel.closed` | shell -> napplet | `channelId`, `reason`? (string) |

Key design notes:

- `inc.channel.open` uses `id` for correlation; the result returns a shell-assigned `channelId`.
- `inc.channel.opened` is a target-side push with no request `id`. It carries
  the same `channelId`; `peer` is the runtime-attested opener dTag.
- The runtime MUST enqueue `inc.channel.opened` for the target before sending a
  successful `inc.channel.open.result` to the opener.
- For each endpoint, handle creation MUST precede any `inc.channel.event` or
  `inc.channel.closed` for that channel.
- `inc.channel.emit` has no `id` field — fire-and-forget like `inc.emit`.
- `inc.channel.event` has no `id` field — shell-initiated delivery, carries `channelId` + `sender`.
- `inc.channel.broadcast` has no `id` field — fire-and-forget to all open channel peers.
- `inc.channel.close` has no `id` — the `channelId` identifies the target channel.
- `inc.channel.closed` is a shell-initiated notification sent on graceful close, peer destruction, or ACL revocation.

### Examples

**Emit a convention URI:**
```
emit("napplet:profile/open?pubkey=abc123...")
```

The runtime transposes the call before sending the wire message:

```
-> { "type": "inc.emit", "topic": "napplet:profile/open", "payload": { "pubkey": "abc123..." } }
```
No response — fire-and-forget.

**Subscribe:**
```
-> { "type": "inc.subscribe", "id": "a1", "topic": "napplet:profile/open" }
<- { "type": "inc.subscribe.result", "id": "a1" }
```

**Event delivery:**
```
<- { "type": "inc.event", "topic": "napplet:profile/open", "sender": "social-feed", "payload": { "pubkey": "abc123..." } }
```

**Unsubscribe:**
```
-> { "type": "inc.unsubscribe", "topic": "napplet:profile/open" }
```

**Open channel:**

Assume the opener dTag is `music-controller` and the target dTag is
`media-player`:

```
-> { "type": "inc.channel.open", "id": "ch1", "target": "media-player" }
target <- { "type": "inc.channel.opened", "channelId": "c-abc", "peer": "music-controller" }
<- { "type": "inc.channel.open.result", "id": "ch1", "channelId": "c-abc", "peer": "media-player" }
```

The opener and target bindings each materialize a `ChannelHandle` for `c-abc`.

**Channel emit:**
```
-> { "type": "inc.channel.emit", "channelId": "c-abc", "payload": { "command": "play", "track": 3 } }
```
No response — fire-and-forget.

**Channel event delivery:**
```
<- { "type": "inc.channel.event", "channelId": "c-abc", "sender": "media-player", "payload": { "status": "playing", "track": 3 } }
```

**Channel broadcast:**
```
-> { "type": "inc.channel.broadcast", "payload": { "announcement": "shutting down" } }
```

**List channels:**
```
-> { "type": "inc.channel.list", "id": "l1" }
<- { "type": "inc.channel.list.result", "id": "l1", "channels": [{ "id": "c-abc", "peer": "media-player" }, { "id": "c-def", "peer": "chat-widget" }] }
```

**Close channel:**
```
-> { "type": "inc.channel.close", "channelId": "c-abc" }
<- { "type": "inc.channel.closed", "channelId": "c-abc" }
```

**Channel closed by peer or shell:**
```
<- { "type": "inc.channel.closed", "channelId": "c-abc", "reason": "peer destroyed" }
```

**Open error:**
```
<- { "type": "inc.channel.open.result", "id": "ch1", "error": "target not found" }
```

### Error Handling

`inc.subscribe.result` MAY include an `error` field (string) if the shell rejects the subscription. When `error` is present, the subscription was not registered.

```
<- { "type": "inc.subscribe.result", "id": "a1", "error": "topic rejected by ACL" }
```

`inc.channel.open.result` MAY include an `error` field if the channel cannot be opened. When `error` is present, no channel was created.

A successful `inc.channel.open.result` means the runtime created both endpoint
handles and enqueued the target's `inc.channel.opened` notification. It does not
mean the target registered an application handler or accepted the channel's
application semantics.

## Topic Conventions

Topics use a prefix convention to signal direction and scope:

| Prefix | Direction | Meaning |
|--------|-----------|---------|
| `shell:*` | napplet -> shell | Commands sent by a napplet to the shell (e.g., `shell:state-get`) |
| `napplet:<archetype>/<intent>` | bidirectional | Archetype-scoped messages between napplets (e.g., `napplet:profile/open`, `napplet:dm/open`) |

The routed topic is the stable message identity. The payload carries per-message
data. A convention producer MAY use the full convention URI when calling
`emit`; the runtime MUST transpose it before routing. Consumers subscribe to the
stable topic without query parameters.

These prefixes are advisory. Except for an explicitly intercepted shell topic,
the shell treats the complete topic as opaque text. It MUST NOT parse, decode,
normalize, prefix-match, or wildcard-match a topic for routing. Therefore
query transposition happens before topic routing, never as part of matching.

## Shell Behavior

### Topic routing

- The shell MUST route `inc.emit` messages to all napplets subscribed to the exact same complete topic string.
- The shell MUST copy the emitted `topic` unchanged into each delivered `inc.event`.
- The runtime MUST derive `sender` from the authenticated emitting endpoint and
  include its dTag in delivered `inc.event` messages. It MUST ignore or reject
  caller-supplied sender data.
- The shell MUST NOT deliver `inc.event` back to the emitting napplet (sender exclusion).
- The shell MUST respond to `inc.subscribe` with `inc.subscribe.result` carrying the same `id`.
- The shell MUST honor `inc.unsubscribe` by removing the subscription for that topic.
- The shell MAY intercept specific topic prefixes (e.g., `shell:*`) for internal command handling rather than routing them to other napplets.
- The shell MAY enforce ACL checks on INC capabilities and reject subscriptions or emits that violate shell policy.

### Channel management

- The shell MUST respond to `inc.channel.open` with `inc.channel.open.result` carrying the same `id`.
- The shell MUST assign a `channelId` (opaque identifier) on successful channel open.
- The shell MUST validate that the target dTag exists and has a live napplet endpoint before opening a channel.
- The runtime MUST derive the opener dTag from its authenticated endpoint. The
  opener cannot set or override `peer` in `inc.channel.opened`.
- The runtime MUST enqueue `inc.channel.opened` for the target before it sends a
  successful `inc.channel.open.result` to the opener.
- The runtime-provided binding MUST materialize a `ChannelHandle` from
  `inc.channel.opened` and retain it until a `channel.onOpened` handler receives
  it.
- The runtime-provided binding MUST retain incoming `inc.channel.event` messages
  until a `ChannelHandle.on` handler is registered for that channel. It MUST NOT
  silently discard buffered channel messages.
- The runtime-provided binding MUST retain the terminal `inc.channel.closed`
  record. A later `ChannelHandle.onClosed` registration MUST receive it.
- A runtime MAY bound unopened-handle or message buffers. On overflow it MUST
  close the channel and notify both endpoints instead of silently dropping the
  handle or messages.
- The shell MUST forward `inc.channel.emit` messages to the channel peer as `inc.channel.event`.
- The shell MUST NOT perform per-message validation on channel messages after open — only the initial open is validated once (auth-on-open model).
- The shell MUST send `inc.channel.closed` to both sides when a channel is torn down (graceful close, iframe removal, or ACL revocation).
- The shell MUST respond to `inc.channel.list` with `inc.channel.list.result` listing the napplet's active channels.
- The shell MUST deliver `inc.channel.broadcast` to all open channel peers of the sender, excluding the sender itself.
- The shell MUST clean up channel state when a napplet endpoint is destroyed, sending `inc.channel.closed` with `reason: "peer destroyed"` to the surviving endpoint.
- The shell MAY enforce ACL checks on `inc.channel.open` (e.g., restrict which dTags a napplet can open channels to).
- The shell MAY impose a maximum number of concurrent channels per napplet.

## Channels vs Topics

Topics and channels serve different communication patterns within NAP-INC:

**Topics** are loose-coupled publish/subscribe. Any napplet can subscribe to any topic; the sender does not know who (if anyone) will receive the message. There is no persistent connection. Topics are well-suited for infrequent coordination — UI commands, state sync, configuration events, and notifications.

**Channels** are pre-authorized point-to-point connections. A napplet calls `channel.open(target)` to establish a channel to a specific peer identified by its dTag. The shell validates the target once on open (auth-on-open model): it checks that the target exists, has a live napplet endpoint, and passes ACL. After open, messages flow between the two endpoints without per-message shell validation. Channels are well-suited for sustained data streams, real-time collaboration, and command channels between specific napplets.

Both are part of the same `inc` namespace and share the NIP-5D wire format. A napplet may use both simultaneously.

## Security Considerations

Sender identity is runtime-enforced from the authenticated emitting endpoint.
There is no per-message signing. Runtime sender identification is the trust
boundary. A projection defines how its authenticated endpoint is bound.

### Topics

- Topic namespaces are not enforced — any napplet can emit on any topic. The shell MAY restrict topics via ACL.
- After convention URI transposition, payloads are opaque to topic routing. Receiving napplets are responsible for validating payload content.
- Sender exclusion prevents echo loops but does not prevent a napplet from emitting messages on any topic. Receivers should check the `sender` dTag if sender identity matters for their use case.

### Channels

- Channel authorization is validated once on `inc.channel.open`. After open, messages flow without per-message shell validation. A compromised browser extension could forge messages on an open channel — this is an accepted trust boundary, the same as topic-based INC.
- An inbound `ChannelHandle` means runtime ACL permitted the channel. It does
  not imply application-level consent. The target MAY close it immediately.
- Unclaimed inbound handles and buffered messages consume runtime resources.
  The runtime SHOULD rate-limit opens and bound buffers per napplet.
- Channel IDs are opaque. Napplets cannot enumerate or guess other channels' IDs. This prevents channel hijacking.
- `inc.channel.broadcast` reaches all open channel peers. Napplets SHOULD NOT send sensitive data via broadcast. For confidential communication, use a named channel with a specific target.

## Changelog

- `cdeaec3` - Introduced NAP-INC topic and channel communication.
- `6461e4b` - Adopted unnumbered convention topics for napplet messages.
- `f24a708` - Separated stable topic identity from per-message payload data.
- `8782bb1` - Added runtime transposition from convention URI parameters to topic payload data.
- `13fed4b` - Made sender attestation runtime-derived and carrier-neutral.
