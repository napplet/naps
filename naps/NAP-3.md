NAP-3
======

Chat Topics
-----------

`draft`

**NAP ID:** NAP-3

**Domain:** chat topic coordination

**Depends:**
- `inc` — capability · required — rides NAP-INC transport (`inc.subscribe` / `inc.event`)

**Discovery:** `shell.supports("inc", "NAP-3")`

## Description

This protocol defines a coherent `chat:*` topic family for napplets that
coordinate chat-related navigation and conversation context through NAP-INC. The
current revision defines:

- `chat:open-dm`

Future compatible revisions may define additional `chat:*` topics, such as
group, room, or thread navigation, without requiring a separate numbered NAP for
every small chat action.

This protocol specifies topic strings, payload shapes, producer behavior, and
consumer behavior. It does not redefine NAP-INC transport methods.

## Message Protocol

Napplets coordinate through NAP-INC. The INC event topic is carried by the
event's `t` tag and the payload is carried as JSON in the event content. This
NAP defines the meaning of selected `chat:*` topics; `inc.emit`,
`inc.subscribe`, `inc.unsubscribe`, and `inc.event` remain generic NAP-INC
transport.

Topics in this family MUST use the `chat:` prefix and MUST describe chat
navigation, conversation targeting, chat context, or chat companion behavior. A
topic that belongs primarily to profile display, stream playback, relay
administration, or shell window management should use its own domain family
instead of being placed here.

### `chat:open-dm`

Producers send `chat:open-dm` when a user action selects a person or identity
for direct conversation, for example from a profile view, contact list, mention,
message action, or other person affordance.

Consumers receive `chat:open-dm` when they can open, focus, or otherwise present
a direct-message conversation for the requested public key.

Payload:

```json
{
  "pubkey": "<hex-pubkey>",
  "displayName": "Ada"
}
```

Fields:

| Field | Type | Required | Semantics |
|-------|------|----------|-----------|
| `pubkey` | string | yes | 64-character lowercase hexadecimal Nostr public key for the conversation peer. |
| `displayName` | string | no | Human-readable label hint for optimistic UI. Consumers MUST NOT treat it as authoritative identity metadata. |

Producer requirements:

- Producers MUST include `pubkey`.
- Producers SHOULD use canonical lowercase hexadecimal public keys.
- Producers MAY include `displayName` as a display hint.
- Producers MUST NOT require a specific shell windowing policy or chat napplet
  implementation.

Consumer requirements:

- Consumers MUST ignore payloads without a valid `pubkey`.
- Consumers SHOULD open or focus a direct-message conversation for `pubkey`.
- Consumers MAY show `displayName` while resolving profile metadata, but MUST
  treat profile events or local identity state as authoritative.
- Consumers MAY ignore the message when direct-message navigation is unsupported
  in the current context.

## Growth and Competing Drafts

This NAP defines one chat-topic coordination surface. Future numbered NAP-N
drafts MAY define alternate or expanded `chat:*` topic sets. Napplets discover
the specific numbered protocol with `shell.supports("inc", "NAP-3")`; ecosystem
preference between competing chat-topic NAPs is determined by implementation
adoption.

## Negotiation

Napplets discover support for this protocol with:

```
window.napplet.shell.supports("inc", "NAP-3");
```

A napplet that requires INC transport declares the interface dependency in its
manifest:

```
["requires", "inc"]
```

The manifest declares only the interface dependency. The numbered protocol is
negotiated at runtime with `shell.supports()`.

## Non-goals

- Defining generic INC transport.
- Defining message encryption, gift wrap, signing, relay selection, or chat
  history retrieval.
- Defining shell window placement, focus, or routing policy.
- Defining profile, stream, relay, or shell administration topics.

## Implementations

- (none yet)
