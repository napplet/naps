NAP-DM
======

Direct Messages
---------------

`draft`

**NAP ID:** NAP-DM
**Domain:** `dm`
**Web binding (NIP-5D):** `window.napplet.dm` · `shell.supports("dm")`

## Description

NAP-DM lets a napplet present direct-message UI while the runtime owns the
message implementation. The napplet asks for conversations, message history,
send, and live delivery. The runtime chooses and executes the transport,
encryption, signing, relay routing, storage, and key management.

This interface is generic. It does not require NIP-04, NIP-17, Double Ratchet,
MLS, or any other concrete private-message protocol. A runtime MAY implement one
or more of them behind the same `dm` surface.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `status` | — | `DmStatus` | `dm.status` / `dm.status.result` |
| `conversations` | `DmConversationQuery` | `DmConversationPage` | `dm.conversations` / `dm.conversations.result` |
| `messages` | `DmMessageQuery` | `DmMessagePage` | `dm.messages` / `dm.messages.result` |
| `send` | `DmSendRequest` | `DmSendResult` | `dm.send` / `dm.send.result` |
| `subscribe` | `DmSubscribeRequest` | `DmSubscription` | `dm.subscribe` / `dm.subscribe.result` |
| `unsubscribe` | `subscriptionId` (`tstr`) | `DmOk` | `dm.unsubscribe` / `dm.unsubscribe.result` |

### Schemas

```cddl
HexPubkey = tstr
Timestamp = int

DmStatus = {
  available: bool,
  ? ownerPubkey: HexPubkey,
  implementations: [* tstr],
  capabilities: [* tstr],
}

DmConversationQuery = {
  ? cursor: tstr,
  ? limit: uint,
}

DmPeer = {
  pubkey: HexPubkey,
  ? label: tstr,
  ? avatar: tstr,
}

DmConversation = {
  id: tstr,
  kind: "direct" / "group",
  participants: [* DmPeer],
  ? subject: tstr,
  unread: uint,
  ? updatedAt: Timestamp,
}

DmConversationPage = {
  conversations: [* DmConversation],
  ? cursor: tstr,
}

DmMessageQuery = {
  conversationId: tstr,
  ? cursor: tstr,
  ? limit: uint,
}

DmMessage = {
  id: tstr,
  conversationId: tstr,
  senderPubkey: HexPubkey,
  createdAt: Timestamp,
  content: tstr,
  status: "sent" / "delivered" / "received" / "failed",
}

DmMessagePage = {
  messages: [* DmMessage],
  ? cursor: tstr,
}

DmSendRequest = {
  ? conversationId: tstr,
  recipients: [* HexPubkey],
  content: tstr,
  ? clientMessageId: tstr,
}

DmSendResult = {
  ok: bool,
  message: DmMessage,
}

DmSubscribeRequest = {
  ? conversationId: tstr,
}

DmSubscription = {
  subscriptionId: tstr,
}

DmOk = {
  ok: bool,
}

DmError = {
  error: tstr,
}
```

`status` returns whether the runtime can currently service direct messages for
the active user. `implementations` is advisory. Values are runtime labels, not
protocol negotiation. Napplets MUST NOT require a specific implementation label
unless another spec defines it.

`conversations` returns normalized conversation summaries. Runtime-specific
thread, invite, session, and relay state stays hidden unless another NAP defines
it.

`messages` returns normalized cleartext messages for one conversation. The
runtime MAY redact or omit messages according to policy.

`send` asks the runtime to send a direct message. The runtime MAY create a new
conversation when `conversationId` is omitted. The runtime MUST perform signing,
encryption, routing, persistence, and policy checks outside the napplet.

`subscribe` starts live delivery. When `conversationId` is omitted, the runtime
MAY deliver messages for all conversations visible to the napplet.

`unsubscribe` stops a live subscription.

## Wire Protocol

`dm.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `dm.status` | napplet -> runtime | `id` |
| `dm.status.result` | runtime -> napplet | `id`, `DmStatus` or `DmError` |
| `dm.conversations` | napplet -> runtime | `id`, `DmConversationQuery` |
| `dm.conversations.result` | runtime -> napplet | `id`, `DmConversationPage` or `DmError` |
| `dm.messages` | napplet -> runtime | `id`, `DmMessageQuery` |
| `dm.messages.result` | runtime -> napplet | `id`, `DmMessagePage` or `DmError` |
| `dm.send` | napplet -> runtime | `id`, `DmSendRequest` |
| `dm.send.result` | runtime -> napplet | `id`, `DmSendResult` or `DmError` |
| `dm.subscribe` | napplet -> runtime | `id`, `DmSubscribeRequest` |
| `dm.subscribe.result` | runtime -> napplet | `id`, `DmSubscription` or `DmError` |
| `dm.unsubscribe` | napplet -> runtime | `id`, `subscriptionId` |
| `dm.unsubscribe.result` | runtime -> napplet | `id`, `DmOk` or `DmError` |
| `dm.message` | runtime -> napplet | `subscriptionId`, `message` (`DmMessage`) |

Request/result pairs use `id` for correlation. `dm.message` is a runtime push
message and has no request `id`.

### Examples

**Status:**
```
-> { "type": "dm.status", "id": "s1" }
<- { "type": "dm.status.result", "id": "s1", "available": true, "ownerPubkey": "<hex>", "implementations": ["nip17"], "capabilities": ["direct"] }
```

**Send:**
```
-> { "type": "dm.send", "id": "m1", "recipients": ["<hex>"], "content": "hello" }
<- { "type": "dm.send.result", "id": "m1", "ok": true, "message": { "id": "msg1", "conversationId": "c1", "senderPubkey": "<hex>", "createdAt": 1790337600, "content": "hello", "status": "sent" } }
```

**Subscribe:**
```
-> { "type": "dm.subscribe", "id": "sub1", "conversationId": "c1" }
<- { "type": "dm.subscribe.result", "id": "sub1", "subscriptionId": "live1" }
<- { "type": "dm.message", "subscriptionId": "live1", "message": { "id": "msg2", "conversationId": "c1", "senderPubkey": "<hex>", "createdAt": 1790337601, "content": "hi", "status": "received" } }
```

### Error Handling

Result messages include `error` when the runtime cannot fulfill a request.
Common errors: `"unavailable"`, `"not signed in"`, `"forbidden"`,
`"not found"`, `"invalid recipient"`, `"send failed"`, `"unsupported"`.

When `error` is present, other result fields are undefined.

## Shell Behavior

- The runtime MUST respond to every request with a result message carrying the
  same `id`.
- The runtime MUST keep signing keys, private message keys, ratchet/session
  state, relay credentials, and storage adapters outside the napplet.
- The runtime MUST enforce per-napplet policy before revealing conversations,
  message history, or live messages.
- The runtime MUST validate recipient public keys before sending.
- The runtime MAY choose any private-message implementation.
- The runtime MAY expose implementation labels in `DmStatus.implementations`,
  but those labels are advisory unless another spec defines their semantics.
- The runtime MAY deny history or live subscriptions while still allowing send.

## Security Considerations

- Napplets receive normalized message data only. They MUST NOT receive signing
  keys, private message keys, raw ratchet state, relay credentials, or storage
  handles.
- The runtime is the policy boundary for private conversation visibility.
- The runtime is responsible for encryption, authentication, replay handling,
  sender validation, and relay routing.
- The runtime SHOULD avoid exposing implementation-specific metadata that lets a
  napplet fingerprint peers, devices, sessions, or relay topology beyond what
  the user-facing DM experience requires.
- Runtimes that support several DM implementations MUST avoid downgrade behavior
  that silently weakens user privacy without policy or user consent.

## Relation to NAP-3 and NAAT-DM

NAP-3 defines `chat:open-dm`, a NAP-N topic for opening or focusing a DM
handler by role. NAP-DM defines the runtime API that a DM handler can use once
it is running. They are complementary.

NAAT-DM names the `dm` handler role. It does not own the NAP-DM payload or API.

## Non-goals

- Defining a specific private-message protocol.
- Defining Double Ratchet session management, invite exchange, or device
  semantics.
- Defining public chat rooms, stream chat, or group membership administration.
- Defining profile metadata lookup.
- Defining shell window placement, default handler resolution, or role dispatch.

## Implementations

- Hyprgate PR #233 uses temporary `him.*` spike wire for a Double Ratchet IM
  prototype. That wire is not NAP-DM and should migrate to this generic `dm`
  interface if this spec is accepted.
