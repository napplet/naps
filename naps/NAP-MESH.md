NAP-MESH
========

Hop-Limited Circle Publish and Subscribe
----------------------------------------

`draft`

**NAP ID:** NAP-MESH
**Domain:** `mesh`
**Depends:**
- `relay` — wire · required — imports `EventTemplate` for publish input and `RelayEventResult` for every delivered event.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.mesh`; domain presence signals availability.

> **First draft, for discussion.** This spec describes one working
> implementation so there is something concrete to argue with. It introduces
> the **circle**, a term that is not in the glossary yet and needs refining
> before anything here is settled. See [Open Questions](#open-questions).

## Description

NAP-MESH gives napplets publish and subscribe over the user's **circle**: the devices of people the user has deliberately paired with, reached peer to peer, without a relay. Reach is measured in **hops** along circle edges. A relay publish (NAP-RELAY) answers "put this on my relays"; a mesh publish answers "send this to my people, this far". The shell owns the circle: who is in it, how an event travels between members, how a copy arriving by a second path is deduplicated, and how far any flood may travel. The napplet chooses only the hop budget, within a cap the user sets.

This interface is for napplets whose data belongs to the people around the user — a doorbell, a household chat, a game between friends' devices — and which must work with no internet. It does not replace relay access. An event published here reaches only as far as its hops carry it. A napplet that also wants the open network publishes there separately.

### The circle is not the transport

Two meshes are involved. This interface exposes only one of them.

| | Transport mesh | Circle |
|---|---|---|
| Nodes | Devices | People (one key each) |
| Edges | Links: radio, LAN, any transport | Pairings, made on purpose by both sides |
| Decides | **Reachability**: how bytes get there | **Trust**: whether they are sent at all |
| Membership | Whatever is in range or routable | Mutual, signed, kept while no link exists |
| Strangers | Routinely a link | Never a member |

Every `mesh` operation travels along circle edges only. "Members" below are the user's paired peers that are reachable right now, wherever the transport puts them. A member several transport hops away is one circle hop away. A stranger in the same room is no hop at all. A hop budget counts circle edges: `ttl: 2` means the user's members, and their members.

A circle is **per user**, not shared. Alice's circle and Bob's circle are two lists that happen to contain each other. There is no group, roster or admin.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `info` | none | `MeshInfo` | `mesh.info` / `mesh.info.result` |
| `publish` | `template` (`EventTemplate`), optional `options` (`MeshPublishOptions`) | `MeshPublishResult` | `mesh.publish` / `mesh.publish.result` |
| `subscribe` | `filters` (`NostrFilter` or list of `NostrFilter`), optional `options` (`MeshSubscribeOptions`) | `MeshSubscription` | `mesh.subscribe` plus deliveries |
| `blobs` | `hashes` (list of text, at most 64) | `MeshBlobsResult` | `mesh.blobs` / `mesh.blobs.result` |

### Schemas

Primitive references:

| Name | Meaning |
|------|---------|
| `NostrFilter` | NIP-01 filter object. |
| `NostrEvent` | NIP-01 signed event object. |
| `EventTemplate` | Owned by NAP-RELAY. Unsigned event template for shell signing. |
| `RelayEventResult` | Owned by NAP-RELAY. |

`MeshInfo` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `online` | yes | boolean | The circle transport is running on this device. |
| `peers` | yes | integer | Circle members reachable right now. Reported, not promised. |
| `limits` | yes | `MeshLimits` | The user's caps. |

`MeshLimits` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `publishTtl` | yes | integer | Most hops a publish may ask for. |
| `subscribeTtl` | yes | integer | Most hops a subscribe backlog request may ask for. |

`MeshPublishOptions` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ttl` | no | integer | Hops the event may travel beyond this device. `0` publishes on this device only. Omitted means the user's cap. |

`MeshPublishResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | True when the event was signed, kept and handed to the circle. |
| `event` | no | `NostrEvent` | The signed event. |
| `eventId` | no | text | Its id. |
| `ttl` | no | integer | The hop budget actually used, after the shell's clamp. |
| `error` | no | text | Reason when `ok` is false. Every field but `ok` is then undefined. |

`MeshSubscribeOptions` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ttl` | no | integer | Hops the backlog request may travel beyond this device. `0` asks nobody. Omitted means the user's cap. |

`MeshBlobsResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `peers` | yes | integer | Members that were asked: every circle member reachable right now. |
| `blobs` | yes | map of text to integer | Each asked hash (lowercase hex) → how many of those members hold it. `0` when none do. |

`MeshSubscription` deliveries:

| Delivery | Carries | Notes |
|----------|---------|-------|
| event | `RelayEventResult` | One matching event, from this device or a member. |
| eose | integer | The effective backlog `ttl`. Marks the end of **this device's** backlog. |
| closed | optional text | The subscription ended or was refused; carries the reason. |

A subscription is closed by the napplet with `mesh.close`.

**`info`** — Reports whether the circle transport is up, how many members are reachable, and the user's hop caps. A napplet reads the caps to tell the user what "everyone" means here. It never needs them to make a call: a budget above the cap is clamped, not refused.

**`publish`** — Signs `template` as the user under NAP-RELAY's rules. The shell owns `pubkey`, `created_at`, `id` and `sig`; the napplet owns `kind`, `content` and `tags`. The shell keeps the event on this device, delivers it to this device's live subscriptions, and floods it to reachable members with `min(ttl, publishTtl)` hops of budget. Each member that receives it keeps it, delivers it to its own live subscriptions, and passes it on to its own members with one hop less until the budget is spent. An ephemeral event (NIP-01 kinds 20000–29999) travels and is delivered the same way, but is kept nowhere.

**`subscribe`** — Opens a live subscription. The shell registers the filters, delivers matching events already kept on this device, asks members up to `min(ttl, subscribeTtl)` hops out for theirs, and sends `eose` once this device's backlog is done. Members' backlog arrives as events afterwards, usually after `eose` but not necessarily. Every later matching event, published here or carried in by a member, is delivered until the napplet closes the subscription or the shell does. Ephemeral events are never backlog.

**`blobs`** — Asks the reachable members whether they hold each blob (by sha256), and answers with a count per blob. It serves a napplet that wants to know what the user's people can supply right now, such as an app store showing only apps a member's device can serve. It asks; it downloads nothing. It asks only the circle: never the internet, never a stranger. A member that does not answer in time counts as not holding the blob.

## Wire Protocol

`mesh.*` messages use the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `mesh.info` | napplet -> shell | `id` |
| `mesh.info.result` | shell -> napplet | `id`, `online`, `peers`, `limits`, `error?` |
| `mesh.publish` | napplet -> shell | `id`, `event` (`EventTemplate`), `ttl?` |
| `mesh.publish.result` | shell -> napplet | `id`, `ok`, `event?`, `eventId?`, `ttl?`, `error?` |
| `mesh.subscribe` | napplet -> shell | `id`, `subId`, `filters`, `ttl?` |
| `mesh.subscribe.result` | shell -> napplet | `id`, `error` — only when the subscription is refused |
| `mesh.event` | shell -> napplet | `subId`, `result` (`RelayEventResult`) |
| `mesh.eose` | shell -> napplet | `subId`, `ttl` |
| `mesh.close` | napplet -> shell | `subId` |
| `mesh.closed` | shell -> napplet | `subId`, `reason?` |
| `mesh.blobs` | napplet -> shell | `id`, `hashes` |
| `mesh.blobs.result` | shell -> napplet | `id`, `peers`, `blobs`, `error?` |

Key design notes:
- Request/result pairs use `id` for correlation. `mesh.subscribe` carries an `id` that only a refusal answers; an opened subscription answers with its deliveries.
- `mesh.close` is fire-and-forget and has no `id`.
- Deliveries are routed by `subId`. Subscription ids are scoped to this domain: a `mesh` subscription and a `relay` subscription may share an id.
- `ttl` counts circle hops **beyond the requesting device**. A hop budget never rides inside the event. The shell carries it between members, so the signed event is byte-identical everywhere it lands.
- The shell does not wait for members before `mesh.eose`. The circle is asynchronous; the wire says so rather than hiding it behind a long wait.

### Examples

**Ring the doorbell two hops out (ephemeral: delivered, kept nowhere):**
```
-> { "type": "mesh.publish", "id": "p1", "event": { "kind": 20666, "content": "ding", "tags": [["t", "doorbell"]] }, "ttl": 2 }
<- { "type": "mesh.publish.result", "id": "p1", "ok": true, "event": { "id": "ev1…", "pubkey": "ab12…", "kind": 20666, "content": "ding", "tags": [["t", "doorbell"]], "created_at": 1234567890, "sig": "…" }, "eventId": "ev1…", "ttl": 2 }
```

**Publish above the cap (cap is 3):**
```
-> { "type": "mesh.publish", "id": "p2", "event": { "kind": 1, "content": "hello" }, "ttl": 10 }
<- { "type": "mesh.publish.result", "id": "p2", "ok": true, "event": { … }, "eventId": "ev2…", "ttl": 3 }
```

**Subscribe to a chat, one hop of backlog:**
```
-> { "type": "mesh.subscribe", "id": "s1", "subId": "room", "filters": [{ "kinds": [9], "#h": ["kitchen"] }], "ttl": 1 }
<- { "type": "mesh.event", "subId": "room", "result": { "event": { … } } }   // kept on this device
<- { "type": "mesh.eose", "subId": "room", "ttl": 1 }
<- { "type": "mesh.event", "subId": "room", "result": { "event": { … } } }   // a member's backlog
<- { "type": "mesh.event", "subId": "room", "result": { "event": { … } } }   // a live event
-> { "type": "mesh.close", "subId": "room" }
```

**Subscribe without the capability:**
```
-> { "type": "mesh.subscribe", "id": "s2", "subId": "bell", "filters": [{ "kinds": [20666] }] }
<- { "type": "mesh.subscribe.result", "id": "s2", "error": "mesh not granted" }
```

**Info:**
```
-> { "type": "mesh.info", "id": "i1" }
<- { "type": "mesh.info.result", "id": "i1", "online": true, "peers": 4, "limits": { "publishTtl": 3, "subscribeTtl": 2 } }
```

**Who can serve two files:**
```
-> { "type": "mesh.blobs", "id": "b1", "hashes": ["b167…f553", "0000…0001"] }
<- { "type": "mesh.blobs.result", "id": "b1", "peers": 3, "blobs": { "b167…f553": 2, "0000…0001": 0 } }
```

### Error Handling

`mesh.publish.result` carries `ok: false` and `error` when the publish is refused (not granted, malformed `ttl`) or the template cannot be signed or the event kept. Every field but `ok` is then undefined. `ok` stays, as NAP-RELAY requires of every publish result.

`mesh.subscribe` fails in one of two ways. A projection SHOULD surface both on the subscription's `closed` delivery:
- **Refused before it opens** — not granted, no `subId`, unreadable filters, malformed `ttl`: `mesh.subscribe.result` with `id` and `error`. No `mesh.closed` follows.
- **Ended after it opened** — the local backlog cannot be read, or the shell ends it: `mesh.closed` with `subId` and `reason`.

A `ttl` that is not a whole number from 0 to 255 is an error, not a zero. A napplet that asked for reach and silently got none could not tell. A `ttl` above the cap is **not** an error: it is clamped and the effective value reported.

`mesh.info.result` carries `error` only when the circle cannot be queried at all. An offline circle is `online: false`, not an error.

`mesh.blobs.result` carries `error` when `hashes` is missing, holds something that is not a sha256 in hex, or holds more than 64. Too many is refused whole, not cut short: a napplet that got answers for the first 64 of 100 would read the rest as "nobody has it". An offline circle is `peers: 0` with every count `0`, not an error. A shell that cannot ask about blobs at all says so with `error` rather than answering zeros.

## Shell Behavior

- The shell MUST send every `mesh` operation along circle edges only, to members the user paired with. It MUST NOT treat a device as a member because the transport can reach it.
- The shell MUST sign publish templates under NAP-RELAY's rules.
- The shell MUST keep a published non-ephemeral event on this device before or regardless of forwarding it, and MUST deliver any published event to this device's live subscriptions, so `ttl: 0` is a working local publish.
- The shell MUST clamp a requested `ttl` to the user's cap and MUST report the effective budget in `mesh.publish.result.ttl` and `mesh.eose.ttl`.
- The shell MUST NOT let a napplet raise the shell's own forwarding clamp for events arriving from members, whatever budget those events carry.
- The shell MAY decline, per member, to pass a member's events further. Declined events are still kept and delivered on this device.
- The shell MUST forward a given event id at most once, so a flood terminates whatever budget it carried.
- The shell MUST respond to every `mesh.info`, `mesh.publish` and `mesh.blobs` request with a result carrying the same `id`.
- The shell MUST deliver matching events as `mesh.event` until the napplet sends `mesh.close` or the shell sends `mesh.closed`.
- The shell MUST send `mesh.eose` once per subscription, after this device's backlog and without waiting for members. Members' backlog MAY arrive on either side of it.
- The shell SHOULD deliver a member's backlog to the subscription that asked even when this device has seen the event before.
- The shell MUST bound a backlog request in time.
- The shell SHOULD deduplicate by event id so a copy arriving by a second path is delivered at most once per subscription. A napplet MUST still tolerate a duplicate.
- The shell MAY keep separate caps for publish and subscribe. A flooded read costs more than a flooded write, since every hop answers as well as forwards, so a lower subscribe cap is the expected default.
- The shell MUST answer `mesh.blobs` from the circle alone, with requests that fetch no body. It MUST NOT ask the internet or follow a member's redirect off the circle.
- The shell MUST bound `mesh.blobs` itself, whatever the napplet's call rate: concurrency, per-request and per-call time, and a short memory of recent answers.
- The shell MAY enforce ACL checks on `mesh` capabilities and MAY revoke the grant at any time. Revocation takes effect on the next call and the next live delivery.

## Security Considerations

- **Trust, not proximity.** Nothing here reaches a device the user has not paired with, however close it is. A stranger on the transport may carry encrypted packets between members. It never receives a `mesh` publish, is never asked for backlog, and is never asked about a blob.
- **Amplification.** The hop budget is the only per-message cost control a napplet holds. A cap the napplet cannot change bounds it. The forwarding clamp for members' events is a separate, shell-owned number that nothing in this interface reaches.
- **Acting as the user.** A `mesh` grant lets a napplet publish as the user to all their members, repeatedly, with no per-event prompt. Runtimes SHOULD say so in words at grant time, as for `relay`.
- **What the circle reveals.** `mesh.info.peers` is a count, and so is each `mesh.blobs` answer. This interface exposes no member identities, addresses, transports or signal data. A napplet learns that some of the user's people are reachable, not who or where.
- **Untrusted input.** Filters and templates come from untrusted code. The shell parses them fully and refuses anything unreadable rather than guessing. Members' backlog answers are untrusted too; the shell checks them against the subscription's filters before delivery.
- **Asking is not free.** Every hash in `mesh.blobs` costs one request per member, often over a slow link. The 64-hash cap and the shell's own bounds keep a napplet polling on a timer from turning members' links into its cache check.

## Open Questions

This draft exists to start these discussions.

- **What is a circle, exactly?** Here it is the set of people the user paired with, one key per person, mutual and signed. Open: how pairing is expressed on Nostr (if at all), whether a circle can be partitioned (family, work), and whether the term belongs in the glossary or in a separate convention.
- **Who may forward what.** Hop budgets beyond 1 need members to pass on each other's events. Should per-member forwarding permission be part of this contract, or stay shell policy as drafted?
- **Transport neutrality.** The draft assumes a peer-to-peer transport with no relay. Should a circle member reached through a relay count as a hop?
- **Presence.** `peers` is a count by design. Is there a case for member identities behind a separate NAP, and what would it cost in privacy?
- **Pass-on.** Re-flooding an event the napplet received, rather than signed, is useful (relaying a neighbour's alert) and is left out of this draft. Should it be here, and with what bounds?
- **`blobs` placement.** Asking who holds a blob may belong with NAP-RESOURCE or NAP-BLOSSOM rather than here.
- **Defaults.** Should `mesh` ever be granted by default? The one implementation does so today and intends to make it opt-in.

## Implementations

- [Myco](https://github.com/Origami74/myco) (Android) — circle over the FIPS mesh (BLE, Wi-Fi Aware, LAN, USB). Caps default to publish 3, subscribe 2. Backlog requests are bounded at 8 s per member asked. `mesh.blobs` sends BUD-01 `HEAD /<sha256>` to each member's Blossom: six at a time, 4 s each, 12 s per call, hits remembered 60 s and misses 20 s. Up to 4096 ids are remembered per subscription for deduplication.
