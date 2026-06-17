NAP-1
======

Profile Topics
--------------

`draft`

**NAP ID:** NAP-1

**Domain:** profile topic coordination

**Depends:**
- `inc` — capability · required — rides NAP-INC transport (`inc.subscribe` / `inc.event`)

**Discovery:** `shell.supports("inc", "NAP-1")`

## Description

This protocol defines the `profile:*` topic family for napplets that coordinate
profile-related actions through NAP-INC. The current revision defines
`profile:open`. Future compatible revisions may define additional `profile:*`
topics without requiring a separate numbered NAP for every small profile action.

This protocol specifies topic strings, payload shapes, producer behavior, and
consumer behavior. It does not redefine NAP-INC transport methods.

## Message Protocol

Napplets coordinate through NAP-INC. The INC event topic is carried by the
event's `t` tag and the payload is carried as JSON in the event content. This
NAP defines the meaning of selected `profile:*` topics; `inc.emit`,
`inc.subscribe`, `inc.unsubscribe`, and `inc.event` remain generic NAP-INC
transport.

Topics in this family MUST use the `profile:` prefix and MUST describe profile
identity, profile display, or profile navigation behavior. A topic that belongs
to another product domain, even when it references a public key, should use its
own domain family instead of being placed here.

### `profile:open`

Producers send `profile:open` when a user action selects a profile identity,
for example from an author name, avatar, mention, contact list, or similar
profile affordance.

Consumers receive `profile:open` when they can open, focus, or otherwise
present a profile view for the requested public key.

Payload:

```json
{
  "pubkey": "<hex-pubkey>"
}
```

Fields:

| Field | Type | Required | Semantics |
|-------|------|----------|-----------|
| `pubkey` | string | yes | 64-character lowercase hexadecimal Nostr public key for the profile to open. |

Producer requirements:

- Producers MUST include `pubkey`.
- Producers SHOULD use canonical lowercase hexadecimal public keys.
- Producers MUST NOT require a specific shell windowing policy or profile
  napplet implementation.

Consumer requirements:

- Consumers MUST ignore payloads without a valid `pubkey`.
- Consumers SHOULD open or focus a profile view for `pubkey`.
- Consumers MAY use local profile caches or relay lookups to render metadata.
- Consumers MAY ignore the message when profile navigation is unsupported in
  the current context.

## Negotiation

Napplets discover support for this protocol with:

```ts
window.napplet.shell.supports("inc", "NAP-1");
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
- Defining shell window placement, focus, or routing policy.
- Defining profile metadata fetching, caching, or rendering.
- Defining direct-message, chat, relay, or stream coordination topics.

## Implementations

- (none yet)
