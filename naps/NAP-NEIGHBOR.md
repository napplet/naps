NAP-NEIGHBOR
============

Neighborhoods and Addresses
---------------------------

`draft`

**NAP ID:** NAP-NEIGHBOR
**Domain:** `neighbor`
**Depends:**
- `relay` — capability · optional — `neighborhood:` addresses are accepted as the `relay` of `relay.subscribe`.
- `outbox` — capability · optional — the same addresses are accepted in `OutboxPublishOptions.relays` and the read-option `relays` of `outbox.*`.
- `resource` — capability · optional — the same addresses are honored as a BUD-10 `xs` hint in `blossom:` URIs passed to `resource.bytes`.
- `upload` — capability · optional — the same addresses are accepted as an upload destination where the `upload` domain offers one.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.neighbor`; domain presence signals availability.

## Description

A **neighborhood** is a propagation channel with a meaning ("home", "sports
club", "festival"). The user's runtime has **peers**: people it set up a
direct connection with. Each connection carries some of the user's
neighborhoods. An event published into a neighborhood goes to the peers that
carry it, who may pass it on to their own peers in that neighborhood, and so
on, within limits each runtime sets.

Nobody controls who ends up in a neighborhood. The user decides only which of
their own connections carry it; the rest is social. A neighborhood may reach
any number of people, and no participant can list them.

NAP-NEIGHBOR tells a napplet which neighborhoods the user carries, which of
the user's peers carry each one, and how to reach them. It carries no data.
For each neighborhood and each peer it returns an **address** (a
`neighborhood:` URI). The napplet passes it to the domains it already uses
(`relay`, `outbox`, `resource`, `upload`).
The runtime recognizes the addresses it minted and carries the traffic however
it chooses. The contract names no transport.

A napplet may also ask the user to start, join, share or leave a
neighborhood. The shell mediates every such request.

A runtime MAY implement the **proximity extension** (see below): a `nearby`
neighborhood of devices physically close to the user. Not every runtime has
radios; the core NAP works without it.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `info` | none | `NeighborInfo` | `neighbor.info` / `neighbor.info.result` |
| `list` | none | list of `Neighborhood` | `neighbor.list` / `neighbor.list.result` |
| `peers` | `neighborhood` (text) | list of `Peer` | `neighbor.peers` / `neighbor.peers.result` |
| `create` | `request` (`CreateRequest`) | `Neighborhood` | `neighbor.create` / `neighbor.create.result` |
| `invite` | `neighborhood` (text), optional `candidates` (list of text) | `InviteResult` | `neighbor.invite` / `neighbor.invite.result` |
| `leave` | `neighborhood` (text) | none | `neighbor.leave` / `neighbor.leave.result` |
| `setOffer` | `neighborhood` (text), `offer` (`Offer`) | `Offer` | `neighbor.setOffer` / `neighbor.setOffer.result` |
| `mute` | `neighborhood` (text), `author` (text), `muted` (boolean) | none | `neighbor.mute` / `neighbor.mute.result` |
| `onEvent` | handler for `NeighborEvent` | `Subscription` handle | `neighbor.event` |

### Schemas

Enums:

| Name | Values |
|------|--------|
| `NeighborhoodKind` | `private`, `public`, `nearby` |
| `Feature` | `create`, `public`, `invite`, `leave`, `offer`, `mute`, `nearby` |
| `LinkClass` | `proximity` (proximity extension only), `local`, `internet` |

`NeighborInfo` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `features` | yes | list of `Feature` | Operations and kinds this runtime offers. |
| `services` | yes | list of text | Service names peers may offer (see `Offer.services`). |

`Neighborhood` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | text | Opaque, minted per napplet. |
| `name` | yes | text | Display name. Not unique. |
| `kind` | yes | `NeighborhoodKind` | `public`: anyone using the same name joins the same neighborhood. |
| `peers` | yes | integer | The user's peers that carry it. |
| `reachable` | yes | integer | Of those, reachable now. Reported, not promised. |
| `address` | no | text | `neighborhood:` URI for the whole neighborhood. |
| `offer` | no | `Offer` | What the user offers peers in it. Present only with the `offer` feature. |

`Peer` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | text | Opaque, minted per napplet and neighborhood. Not a social identity. |
| `name` | no | text | Display name the user gave this peer, if any. |
| `reachable` | yes | boolean | |
| `link` | no | `LinkClass` | Coarse class of the current path. |
| `services` | yes | list of text | Services this peer offers in this neighborhood. |
| `address` | no | text | `neighborhood:` URI for this peer, scoped to this neighborhood. |

`Offer` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `relayRead` | yes | boolean | Peers may read this neighborhood's events from the user. |
| `relayWrite` | yes | boolean | Peers may send events to the user in this neighborhood. |
| `relayOwn` | yes | boolean | Peers may also read events the user keeps outside any neighborhood. |
| `blobRead` | yes | boolean | |
| `blobWrite` | yes | boolean | |
| `services` | no | list of text | Further runtime-defined service names. |

`CreateRequest` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `name` | yes | text | |
| `kind` | no | `"private"` or `"public"` | Omitted means `private`. `public` needs the `public` feature. |

`InviteResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `status` | yes | `"sent"` or `"cancelled"` | `sent` means the runtime took delivery responsibility, not that anyone accepted. |
| `count` | no | integer | Invitations sent. |

`NeighborEvent` variants:

| Variant | Required fields | Optional fields |
|---------|-----------------|-----------------|
| neighborhood | `type: "neighborhood"`, `change: "added" \| "removed" \| "changed"`, `neighborhood: text` | none |
| peer | `type: "peer"`, `neighborhood: text`, `peer: text`, `change: "added" \| "removed" \| "reachable" \| "unreachable" \| "changed"` | none |

**`info()`** reports features and service names. A napplet MUST check
`features` before calling an operation it names.

**`list()`** returns only the neighborhoods the user granted this napplet.

**`peers(neighborhood)`** returns the user's own peers that carry that
neighborhood. It never describes anyone further away; nobody knows who that
is.

**`create(request)`** asks the user to start a neighborhood, or for `public`,
to join the public neighborhood of that name. It resolves after the user
confirms, and grants it to the calling napplet.

**`invite(neighborhood, candidates?)`** asks the user to share a neighborhood
with people. `candidates` are `Peer.id` values from other neighborhoods this
napplet can see: the shell asks those peers to carry this neighborhood too.
The shell MAY also offer ways to set up new peers.

**`leave(neighborhood)`** asks the user to stop carrying a neighborhood on all
their connections.

**`setOffer(neighborhood, offer)`** asks the user to change what they offer
peers in that neighborhood. It resolves to the offer actually in force.

**`mute(neighborhood, author, muted)`** hides, and stops passing on, events by
`author` (a Nostr public key, hex) in that neighborhood. This is local; nobody
else is told.

**`onEvent(handler)`** receives neighborhood and peer changes for granted
neighborhoods.

### Addresses

An address is a URI with the scheme `neighborhood:`. It is minted by the
runtime, per napplet, and names a neighborhood, or one peer as seen in one
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

A peer address reaches that peer only. Its traffic is not passed on.

### Proximity extension

Optional. A runtime that can sense nearby devices (radio, local network, tap)
MAY implement it and lists `nearby` in `info.features`. A runtime without it
omits `nearby`, never returns a `nearby` neighborhood, and never reports
`link: "proximity"`. Napplets MUST work without it.

It adds:

- **The `nearby` kind.** One neighborhood, maintained by the runtime, carried
  on short-lived connections to devices close to the user right now. `create`,
  `invite` and `setOffer` on it fail with `"unsupported"`. Peers come and go
  as they move in and out of range, and their ids MAY change between
  encounters.
- **`LinkClass` `proximity`.** A peer's current path is a direct, local radio
  or physical link.
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
| `neighbor.peers` | napplet -> shell | `id`, `neighborhood` |
| `neighbor.peers.result` | shell -> napplet | `id`, `peers`?, `error`? |
| `neighbor.create` | napplet -> shell | `id`, `request` |
| `neighbor.create.result` | shell -> napplet | `id`, `neighborhood`?, `error`? |
| `neighbor.invite` | napplet -> shell | `id`, `neighborhood`, `candidates`? |
| `neighbor.invite.result` | shell -> napplet | `id`, `result`?, `error`? |
| `neighbor.leave` | napplet -> shell | `id`, `neighborhood` |
| `neighbor.leave.result` | shell -> napplet | `id`, `error`? |
| `neighbor.setOffer` | napplet -> shell | `id`, `neighborhood`, `offer` |
| `neighbor.setOffer.result` | shell -> napplet | `id`, `offer`?, `error`? |
| `neighbor.mute` | napplet -> shell | `id`, `neighborhood`, `author`, `muted` |
| `neighbor.mute.result` | shell -> napplet | `id`, `error`? |
| `neighbor.event` | shell -> napplet | `event` (`NeighborEvent`) |

### Examples

**List, then read one neighborhood:**
```
-> { "type": "neighbor.list", "id": "n1" }
<- { "type": "neighbor.list.result", "id": "n1", "neighborhoods": [
     { "id": "h_2c", "name": "Household", "kind": "private", "peers": 3,
       "reachable": 2, "address": "neighborhood://h_2c" } ] }
-> { "type": "relay.subscribe", "id": "r1", "subId": "s1",
     "filters": [{ "kinds": [1], "limit": 50 }], "relay": "neighborhood://h_2c" }
```

**Join a public neighborhood:**
```
-> { "type": "neighbor.create", "id": "n2",
     "request": { "name": "Lowlands 2027", "kind": "public" } }
<- { "type": "neighbor.create.result", "id": "n2", "neighborhood": {
     "id": "h_91", "name": "Lowlands 2027", "kind": "public", "peers": 0,
     "reachable": 0, "address": "neighborhood://h_91" } }
```

**Peer change:**
```
<- { "type": "neighbor.event", "event": { "type": "peer",
     "neighborhood": "h_2c", "peer": "p_7f", "change": "reachable" } }
```

The address strings above are illustrative. They are not a format.

### Error Handling

Result messages carry `error` on failure. Every other field is then undefined.

| Error | Meaning |
|-------|---------|
| `"unsupported"` | The runtime does not offer this operation or kind (see `info.features`). |
| `"denied"` | Not granted, or refused by policy. |
| `"cancelled"` | The user declined. |
| `"not found"` | Unknown or ungranted neighborhood or peer id. |

## Shell Behavior

**Requests**
- The shell MUST respond to every request with a result message carrying the
  same `id`.

**Grants and visibility**
- The shell MUST expose only the neighborhoods the user granted to the napplet,
  and peers in those only.
- The shell SHOULD let the user choose neighborhoods per napplet at grant time.
- Revoking a grant, or leaving a neighborhood, MUST end the napplet's
  subscriptions and requests bound to its addresses. They end with the owning
  domain's close message and reason `"revoked"`.

**Identifiers**
- `Neighborhood.id`, `Peer.id` and the addresses MUST be opaque and minted per
  napplet.
- They SHOULD stay stable for that napplet across restarts, except `nearby`
  peer ids.
- The shell MUST NOT expose the keys or transport addresses it uses to reach a
  peer.

**Addresses**
- A runtime that offers `neighbor` and a consuming domain MUST accept, in that
  domain, the addresses it minted for the calling napplet. It MUST refuse them
  from any other napplet.
- A consuming domain that validates URLs (for example `relay` requiring
  `ws:`/`wss:`) MUST accept the `neighborhood:` scheme for addresses this
  runtime minted.
- Traffic to an address MUST go only to endpoints that peers carrying that
  neighborhood (or that one peer) declared for it. It MUST NOT also go to the
  user's own relays, or anywhere else, as a side effect of the address. What a
  peer does with traffic it received is outside the shell's control.
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

**Events and limits**
- The shell SHOULD coalesce `neighbor.event` deliveries. It MAY rate-limit
  them.
- `peers`, `reachable`, `link` and `services` are reports, not promises. A
  napplet MUST tolerate an address that reaches nobody.
- The shell MAY enforce ACL checks on `neighbor` capabilities.

## Non-Goals

- **Membership.** There are no members, only the user's own peers. No
  operation returns, or promises, who a neighborhood reaches.
- **Removing someone.** There is no authority. The user may stop carrying a
  neighborhood with a peer, leave, or mute an author.
- **Delivery guarantees.** Addresses are best effort over whatever the runtime
  carries them on.
- **Reach controls.** How far traffic travels is runtime policy, not napplet
  input. Two runtimes may reach differently for the same publish.
- **The peering protocol.** How runtimes set up connections and carry
  neighborhoods between them is outside this NAP.
- **Live sessions.** Real-time connections between peers belong to the
  `webrtc` domain. A napplet can use a neighborhood to find people who want to
  connect (for example by publishing an invitation event into it) and then
  open a session through `webrtc`. This NAP does not route sessions.
- **Media.** Voice and video are not part of this interface.

## Security Considerations

- **No expectation of privacy.** A neighborhood is a propagation path for
  ordinary Nostr events, not a private space. Events are signed by their
  author, may be authored by anyone, and may be passed on, bridged or published
  by any peer. Anonymity is not a goal.
- **Re-sharing is permitted.** A peer holds what they received and may pass it
  on, including outside the neighborhood. This is how events travel offline
  and spread.
- **Bridging is permitted.** A napplet granted two neighborhoods can repost
  from one into the other. Segregation constrains the shell, not the napplet.
  Grant UI SHOULD say when a napplet holds more than one neighborhood.
- **Social map.** Which neighborhoods the user carries, and with which peers,
  maps their social life. Grants are per neighborhood. Ids are per napplet, so
  two napplets cannot correlate them.
- **Presence.** `reachable` and `link` reveal that peers are reachable, and
  how. With the proximity extension they reveal physical closeness.
- **Untrusted input.** Anyone can author events that reach a neighborhood. A
  napplet MUST treat event content as untrusted.
- **Acting as the user.** Publishing to an address signs as the user, as in
  `relay`. Management operations change who the user is connected to; this is
  why they need confirmation.
- **Addresses are capabilities.** A leaked address is inert: it is per napplet,
  and refused for any other napplet.

## Implementations

- [Myco](https://github.com/Origami74/myco) (Android), planned.

## Changelog

- `7698892` - Introduced NAP-NEIGHBOR draft: neighborhood membership, opaque `neighborhood:` addresses consumed by `relay`, `outbox`, `resource`, `upload` and `webrtc`, and an optional proximity extension. Supersedes NAP-MESH (#122).
- `ef04953` - Dropped Nearby anonymity requirements (pubkey disclosure, mandatory id rotation); stated no expectation of privacy and that members may re-share events anywhere; forbade the shell from putting addresses into events.
- `5fc9f63` - Reframed neighborhoods as propagation channels carried over the user's own peer connections: no membership, founders or invite rights. Renamed `members` to `peers` and `Member` to `Peer` (no `pubkey`); added `public` neighborhoods and kinds `private`/`public`/`nearby`; `mute` takes an author key; addresses may reach endpoints peers declare anywhere, not only the runtime's own network.
- `PENDING` - Removed WebRTC: dropped `serve`/`unserve`, `Neighborhood.room`, `Peer.direct`, `Offer.sessions` and the `webrtc` dependency. Live sessions are left to the `webrtc` domain; neighborhoods only help napplets find who to connect to.
