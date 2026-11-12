NAP-NEIGHBOR
============

Neighborhood Membership and Addresses
-------------------------------------

`draft`

**NAP ID:** NAP-NEIGHBOR
**Domain:** `neighbor`
**Depends:**
- `webrtc` — wire · optional — `Neighborhood.room` and `Member.direct` carry `WebrtcScope`, a type owned by the `webrtc` domain; `serve` routes `webrtc.open` sessions.
- `relay` — capability · optional — `neighborhood:` addresses are accepted as the `relay` of `relay.subscribe`.
- `outbox` — capability · optional — the same addresses are accepted in `OutboxPublishOptions.relays` and the read-option `relays` of `outbox.*`.
- `resource` — capability · optional — the same addresses are honored as a BUD-10 `xs` hint in `blossom:` URIs passed to `resource.bytes`.
- `upload` — capability · optional — the same addresses are accepted as an upload destination where the `upload` domain offers one.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.neighbor`; domain presence signals availability.

## Description

NAP-NEIGHBOR tells a napplet which **neighborhoods** the user belongs to, who
is in them, and how to reach each one. A neighborhood is an open-ended group of
people. Any member may invite others, so no participant holds the full member
list. The user decides which neighborhoods each napplet may see.

NAP-NEIGHBOR carries no data. For each neighborhood and each member it returns
an **address** (a `neighborhood:` URI) and a WebRTC scope. The napplet
passes them to the domains it already uses (`relay`, `outbox`, `resource`,
`upload`, `webrtc`). The runtime recognizes the addresses it minted, keeps the
traffic inside that neighborhood, and carries it however it chooses. The
contract names no transport.

A napplet consumes membership, reachability and addresses, and may ask the
user to found, join, invite to or leave a neighborhood. The shell mediates
every such request. The runtime owns membership proof, scoping and carriage.

A runtime MAY implement the **proximity extension** (see below): a `nearby`
neighborhood of devices physically close to the user, and in-person ways to
invite. Not every runtime has radios; the core NAP works without it.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `info` | none | `NeighborInfo` | `neighbor.info` / `neighbor.info.result` |
| `list` | none | list of `Neighborhood` | `neighbor.list` / `neighbor.list.result` |
| `members` | `neighborhood` (text) | list of `Member` | `neighbor.members` / `neighbor.members.result` |
| `create` | `request` (`CreateRequest`) | `Neighborhood` | `neighbor.create` / `neighbor.create.result` |
| `invite` | `neighborhood` (text), optional `candidates` (list of text) | `InviteResult` | `neighbor.invite` / `neighbor.invite.result` |
| `leave` | `neighborhood` (text) | none | `neighbor.leave` / `neighbor.leave.result` |
| `setOffer` | `neighborhood` (text), `offer` (`Offer`) | `Offer` | `neighbor.setOffer` / `neighbor.setOffer.result` |
| `mute` | `neighborhood` (text), `member` (text), `muted` (boolean) | none | `neighbor.mute` / `neighbor.mute.result` |
| `serve` | `protocol` (text), optional `neighborhoods` (list of text) | none | `neighbor.serve` / `neighbor.serve.result` |
| `unserve` | `protocol` (text) | none | `neighbor.unserve` / `neighbor.unserve.result` |
| `onEvent` | handler for `NeighborEvent` | `Subscription` handle | `neighbor.event` |

### Schemas

External types:

| Name | Meaning |
|------|---------|
| `WebrtcScope` | Owned by the `webrtc` domain. Imported by reference. |

Enums:

| Name | Values |
|------|--------|
| `NeighborhoodKind` | `group`, `list`, `nearby` |
| `Feature` | `nearby`, `create`, `invite`, `leave`, `offer`, `mute`, `serve` |
| `LinkClass` | `proximity` (proximity extension only), `local`, `internet` |

`NeighborInfo` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `features` | yes | list of `Feature` | Operations and kinds this runtime offers. |
| `services` | yes | list of text | Service names members may offer (see `Offer.services`). |

`Neighborhood` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | text | Opaque, minted per napplet. |
| `name` | yes | text | Display name. Not unique. |
| `kind` | yes | `NeighborhoodKind` | |
| `founder` | yes | boolean | The user founded it. |
| `canInvite` | yes | boolean | The user may invite others into it. |
| `reachable` | yes | integer | Members reachable now. Reported, not promised. |
| `address` | no | text | `neighborhood:` URI for the whole neighborhood. |
| `room` | no | `WebrtcScope` | Room scope for the whole neighborhood. |
| `offer` | no | `Offer` | What the user offers its members. Present only with the `offer` feature. |

`Member` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | text | Opaque, minted per napplet and neighborhood. |
| `name` | no | text | Display name, if known. |
| `pubkey` | no | text | The member's public key, when the runtime discloses it. |
| `reachable` | yes | boolean | |
| `link` | no | `LinkClass` | Coarse class of the current path. |
| `services` | yes | list of text | Services this member offers in this neighborhood. |
| `address` | no | text | `neighborhood:` URI for this member, scoped to this neighborhood. |
| `direct` | no | `WebrtcScope` | Direct scope for this member. |

`Offer` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `relayRead` | yes | boolean | Members may read this neighborhood's events from the user. |
| `relayWrite` | yes | boolean | Members may send events to the user in this neighborhood. |
| `relayOwn` | yes | boolean | Members may also read events the user keeps outside any neighborhood. |
| `blobRead` | yes | boolean | |
| `blobWrite` | yes | boolean | |
| `sessions` | yes | boolean | Members may open `webrtc` sessions to served protocols. |
| `services` | no | list of text | Further runtime-defined service names. |

`CreateRequest` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `name` | yes | text | |
| `kind` | no | `"group"` or `"list"` | Omitted means `group`. In a `list`, only the founder may invite. |

`InviteResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `status` | yes | `"sent"` or `"cancelled"` | `sent` means the runtime took delivery responsibility, not that anyone accepted. |
| `count` | no | integer | Invitations sent. |

`NeighborEvent` variants:

| Variant | Required fields | Optional fields |
|---------|-----------------|-----------------|
| neighborhood | `type: "neighborhood"`, `change: "added" \| "removed" \| "changed"`, `neighborhood: text` | none |
| member | `type: "member"`, `neighborhood: text`, `member: text`, `change: "joined" \| "left" \| "reachable" \| "unreachable" \| "changed"` | none |

**`info()`** reports features and service names. A napplet MUST check
`features` before calling an operation it names.

**`list()`** returns only the neighborhoods the user granted this napplet.

**`members(neighborhood)`** returns the members this runtime knows in that
neighborhood. In an open neighborhood that is a subset by design. It is never
the full membership.

**`create(request)`** asks the user to found a neighborhood. It resolves after
the user confirms, and grants it to the calling napplet.

**`invite(neighborhood, candidates?)`** asks the user to invite people.
`candidates` are `Member.id` values from other neighborhoods this napplet can
see. The shell lets the user choose, and MAY offer other ways to reach people.

**`leave(neighborhood)`** asks the user to leave.

**`setOffer(neighborhood, offer)`** asks the user to change what they offer
that neighborhood's members. It resolves to the offer actually in force.

**`mute(neighborhood, member, muted)`** hides and stops passing on that
member's traffic in that neighborhood. This is local; no other participant is
told.

**`serve(protocol, neighborhoods?)`** registers the napplet to receive
`webrtc` sessions opened by members with `protocol`. If `neighborhoods` is
omitted, every granted neighborhood is meant. Nothing is advertised to members.
**`unserve(protocol)`** removes the registration.

**`onEvent(handler)`** receives membership and reachability changes for
granted neighborhoods.

### Addresses

An address is a URI with the scheme `neighborhood:`. It is minted by the
runtime, per napplet, and names a neighborhood, or a member as seen in one
neighborhood. The same address serves relay and blob traffic.

- A napplet MUST NOT parse, construct or modify an address. Only the scheme is
  specified; everything after it is opaque.
- A napplet MUST NOT expect an address to work in another napplet or another
  runtime.
- The scheme is not tied to any network or name system. It never resolves
  through DNS.

| Use | How |
|-----|-----|
| Read a neighborhood | `relay.subscribe(filters, { relay: <address> })` |
| Publish into it | `outbox.publish(template, { relays: [<address>], toOutbox: false })` |
| Fetch a blob from it | `resource.bytes("blossom:<sha256>.<ext>?xs=<address>")` (`xs` percent-encoded) |
| Upload into it | `upload` with `<address>` as destination, where offered |
| Talk live | `webrtc.open({ scope: <room or direct>, protocol })` |

### Proximity extension

Optional. A runtime that can sense nearby devices (radio, local network, tap)
MAY implement it and lists `nearby` in `info.features`. A runtime without it
omits `nearby`, never returns a `nearby` neighborhood, and never reports
`link: "proximity"`. Napplets MUST work without it.

It adds:

- **The `nearby` kind.** One neighborhood, created and maintained by the
  runtime, whose members are devices close to the user right now. It has no
  founder and no invitations; `create`, `invite` and `setOffer` on it fail with
  `"unsupported"`. Members join and leave as they come into and out of range.
- **`LinkClass` `proximity`.** A member's current path is a direct, local
  radio or physical link.
- **In-person invitations.** For `invite`, the shell MAY offer tap, scan or
  nearby discovery as ways to reach people. Which ones is runtime choice and
  not visible to the napplet.

How the runtime discovers devices (Bluetooth LE, Wi-Fi Aware, NFC, a LAN, …) is
outside this NAP.

## Wire Protocol

`neighbor.*` messages use the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `neighbor.info` | napplet -> shell | `id` |
| `neighbor.info.result` | shell -> napplet | `id`, `info`?, `error`? |
| `neighbor.list` | napplet -> shell | `id` |
| `neighbor.list.result` | shell -> napplet | `id`, `neighborhoods`?, `error`? |
| `neighbor.members` | napplet -> shell | `id`, `neighborhood` |
| `neighbor.members.result` | shell -> napplet | `id`, `members`?, `error`? |
| `neighbor.create` | napplet -> shell | `id`, `request` |
| `neighbor.create.result` | shell -> napplet | `id`, `neighborhood`?, `error`? |
| `neighbor.invite` | napplet -> shell | `id`, `neighborhood`, `candidates`? |
| `neighbor.invite.result` | shell -> napplet | `id`, `result`?, `error`? |
| `neighbor.leave` | napplet -> shell | `id`, `neighborhood` |
| `neighbor.leave.result` | shell -> napplet | `id`, `error`? |
| `neighbor.setOffer` | napplet -> shell | `id`, `neighborhood`, `offer` |
| `neighbor.setOffer.result` | shell -> napplet | `id`, `offer`?, `error`? |
| `neighbor.mute` | napplet -> shell | `id`, `neighborhood`, `member`, `muted` |
| `neighbor.mute.result` | shell -> napplet | `id`, `error`? |
| `neighbor.serve` | napplet -> shell | `id`, `protocol`, `neighborhoods`? |
| `neighbor.serve.result` | shell -> napplet | `id`, `error`? |
| `neighbor.unserve` | napplet -> shell | `id`, `protocol` |
| `neighbor.unserve.result` | shell -> napplet | `id`, `error`? |
| `neighbor.event` | shell -> napplet | `event` (`NeighborEvent`) |

### Examples

**List, then read one neighborhood:**
```
-> { "type": "neighbor.list", "id": "n1" }
<- { "type": "neighbor.list.result", "id": "n1", "neighborhoods": [
     { "id": "h_2c", "name": "Household", "kind": "group", "founder": false,
       "canInvite": true, "reachable": 3, "address": "neighborhood://h_2c",
       "room": { "type": "room", "room": "h_2c" } } ] }
-> { "type": "relay.subscribe", "id": "r1", "subId": "s1",
     "filters": [{ "kinds": [1], "limit": 50 }], "relay": "neighborhood://h_2c" }
```

**Serve a protocol:**
```
-> { "type": "neighbor.serve", "id": "n2", "protocol": "doodle" }
<- { "type": "neighbor.serve.result", "id": "n2" }
```

**Membership change:**
```
<- { "type": "neighbor.event", "event": { "type": "member",
     "neighborhood": "h_2c", "member": "m_7f", "change": "reachable" } }
```

The address strings above are illustrative. They are not a format.

### Error Handling

Result messages carry `error` on failure. Every other field is then undefined.

| Error | Meaning |
|-------|---------|
| `"unsupported"` | The runtime does not offer this operation or kind (see `info.features`). |
| `"denied"` | Not granted, or refused by policy. |
| `"cancelled"` | The user declined. |
| `"not found"` | Unknown or ungranted neighborhood or member id. |

## Shell Behavior

**Requests**
- The shell MUST respond to every request with a result message carrying the
  same `id`.

**Grants and visibility**
- The shell MUST expose only the neighborhoods the user granted to the napplet,
  and members of those only.
- The shell SHOULD let the user choose neighborhoods per napplet at grant time.
- Revoking a grant, or leaving a neighborhood, MUST end the napplet's
  subscriptions and sessions bound to its addresses. They end with the owning
  domain's close message and reason `"revoked"`.

**Identifiers**
- `Neighborhood.id`, `Member.id` and the addresses MUST be opaque and minted
  per napplet.
- For `group` and `list` neighborhoods they SHOULD stay stable for that napplet
  across restarts.
- With the proximity extension, `nearby` `Member.id` values MAY change
  between encounters.

**Addresses**
- A runtime that offers `neighbor` and a consuming domain MUST accept, in that
  domain, the addresses it minted for the calling napplet. It MUST refuse them
  from any other napplet.
- A consuming domain that validates URLs (for example `relay` requiring
  `ws:`/`wss:`) MUST accept the `neighborhood:` scheme for addresses this
  runtime minted.
- Traffic to an address MUST reach only members of that neighborhood (or that
  one member), as the runtime knows them. It MUST NOT reach any other relay,
  blob server or network as a side effect of the address.
- A subscription on a neighborhood address MUST return only events that
  arrived through, or were published into, that neighborhood.
- A blob fetch through an address MUST verify the SHA-256 before delivery.
- The shell MUST NOT put an address into any event it signs or carries,
  including relay hints in tags. Addresses are local to the napplet.

**Segregation**
- The shell MUST NOT, by its own action, make an event or blob received in one
  neighborhood available through another neighborhood's address, or through
  the user's own relays.
- The same event received in two neighborhoods is available through both.
- Segregation bounds what the shell does on its own. It is not a privacy
  guarantee (see Security Considerations).

**Requests to the user**
- `create`, `invite`, `leave` and any `setOffer` that widens access MUST be
  confirmed by the user.
- The user MAY pre-authorize a napplet they chose as their neighborhood
  manager.

**Serving**
- The shell MUST route an incoming `webrtc` session for a served protocol only
  to a napplet that serves it in that neighborhood.
- It SHOULD route only to the same napplet identity that opened the session on
  the other side.
- A session to a protocol nobody serves, to a napplet that isn't present, or
  one the user declined, MUST fail identically, so that a member cannot learn
  what is installed.
- The shell MAY ask the user to start a napplet that is present but not
  running.

**Events and limits**
- The shell SHOULD coalesce `neighbor.event` deliveries. It MAY rate-limit
  them.
- `reachable`, `link` and `services` are reports, not promises. A napplet
  MUST tolerate an address that reaches nobody.
- The shell MAY enforce ACL checks on `neighbor` capabilities.

## Non-Goals

- **A member list.** Neighborhoods are open-ended. No operation returns, or
  promises, full membership.
- **Removing someone.** There is no authority. The user may leave or mute.
- **Delivery guarantees.** Addresses are best effort over whatever the runtime
  carries them on.
- **Reach controls.** How far traffic travels inside a neighborhood is runtime
  policy, not napplet input.
- **The membership protocol.** How runtimes prove membership to each other is
  outside this NAP.
- **Media.** Voice and video are not part of this interface.

## Security Considerations

- **Membership is sensitive.** Which neighborhoods the user is in, and who else
  is, can map their social life. Grants are per neighborhood. Ids are per
  napplet, so two napplets cannot correlate them.
- **Presence.** `reachable` and `link` reveal that people are reachable, and
  how. With the proximity extension they reveal physical closeness.
- **Open growth.** A neighborhood may contain people the user never met.
  Events from them are signed. A napplet MUST treat member content and
  `webrtc` payloads as untrusted input.
- **Bridging is permitted.** A napplet granted two neighborhoods can repost
  from one into the other. Segregation constrains the shell, not the napplet.
  Grant UI SHOULD say when a napplet holds more than one neighborhood.
- **No expectation of privacy.** A neighborhood is a propagation path for
  ordinary Nostr events, not a private space. Events are signed by their
  author, may be authored by non-members, and may be passed on, bridged or
  published by any member. Anonymity is not a goal.
- **Re-sharing is permitted.** A member holds what they received and may pass
  it on, including outside the neighborhood. This is how events travel
  offline and spread.
- **Acting as the user.** Publishing to an address signs as the user, as in
  `relay`. Management operations change who the user is connected to; this is
  why they need confirmation.
- **Probing.** Uniform `serve` failures stop members from probing for
  installed napplets. Rate limits on `webrtc.open` per member SHOULD apply.
- **Addresses are capabilities.** A leaked address is inert: it is per napplet,
  and refused for any other napplet.

## Implementations

- [Myco](https://github.com/Origami74/myco) (Android), planned.

## Changelog

- `7698892` - Introduced NAP-NEIGHBOR draft: neighborhood membership, opaque `neighborhood:` addresses consumed by `relay`, `outbox`, `resource`, `upload` and `webrtc`, and an optional proximity extension. Supersedes NAP-MESH (#122).
- `ef04953` - Dropped Nearby anonymity requirements (pubkey disclosure, mandatory id rotation); stated no expectation of privacy and that members may re-share events anywhere; forbade the shell from putting addresses into events.
