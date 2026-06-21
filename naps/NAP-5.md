NAP-5
======

Feed Topics
-----------

`draft`

**NAP ID:** NAP-5

**Domain:** feed topic coordination

**Depends:**
- `inc` — capability · required — rides NAP-INC transport (`inc.subscribe` / `inc.event`)

**Serves:** `feed/open`

**Discovery:** `shell.supports("inc", "NAP-5")`

## Description

This protocol defines a coherent `feed:*` topic family for napplets that hand a
feed napplet the criteria for *what to show* — a set of [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md)
filters and, optionally, an **origin** — a fixed relay set or outbox routing over
authors. Origin is a field, not a topic: a relay feed and an outbox feed are one
request differing only by `origin.type`, and new origin kinds never fork the
contract. The current revision defines:

- `feed:open`

`feed:open` is the recommended open contract for the [`feed`](../naat/feed.md)
archetype: a producer (a profile view, a search box, a hashtag chip, a relay
browser, a saved-search list, …) describes a feed by its **filter**, and a feed
napplet renders the matching stream of events. Future compatible revisions may
define additional `feed:*` topics — refresh, scroll-to, append-filter, report
current criteria — without requiring a separate numbered NAP for every small
feed action.

This protocol specifies topic strings, payload shapes, producer behavior, and
consumer behavior. It does not redefine NAP-INC transport methods, and it does
not define how a feed napplet fetches, deduplicates, orders, or renders events.

## Message Protocol

Napplets coordinate through NAP-INC. The INC event topic is carried by the
event's `t` tag and the payload is carried as JSON in the event content. This
NAP defines the meaning of selected `feed:*` topics; `inc.emit`,
`inc.subscribe`, `inc.unsubscribe`, and `inc.event` remain generic NAP-INC
transport.

Topics in this family MUST use the `feed:` prefix and MUST describe feed
criteria, feed navigation, or feed companion behavior. A topic that belongs
primarily to single-event display (→ `note:*`), profile display, stream
playback, relay administration, or shell window management should use its own
domain family instead of being placed here.

### `feed:open`

Producer intent: ask a feed napplet to present a list of events described by one
or more NIP-01 filters, optionally with an `origin` naming where to source them.

Transport:

```
inc.emit("feed:open", payload)
```

`feed:open` may reach a consumer two ways, both carrying the identical payload:

1. **Directly over NAP-INC** — a running producer emits the topic to any
   subscribed feed napplet (peer-to-peer, no shell routing). Use this when the
   producer already knows a feed surface is listening.
2. **Via [NAP-INTENT](NAP-INTENT.md)** — a caller invokes the `feed` archetype
   (`intent.open("feed", payload)`); the shell resolves the user's default
   handler, creates or focuses its window, and delivers the same payload — as a
   `feed:open` INC event to a running handler, or as initial state to a
   cold-started one.

Consumers MUST accept the payload identically regardless of which path delivered
it; this NAP defines the payload, not the choice of delivery path.

Payload:

```cddl
FeedOpenPayload = {
  filters: [1* NostrFilter], ; one or more NIP-01 filters, OR-combined
  ? origin: FeedOrigin,      ; absent means consumer policy
  ? title: tstr,             ; human-readable label hint
  ? source: FeedSource,
  ? behavior: FeedBehavior,
}

FeedSource = {
  ? napplet: tstr,   ; producer dTag, advisory
  ? windowId: tstr,  ; producer window, advisory
  ? requestId: tstr, ; opaque correlation id
}

FeedBehavior = {
  ? focus: bool,     ; default true
  ? newWindow: bool, ; default shell policy
}

FeedOrigin = FeedRelayOrigin / FeedOutboxOrigin

FeedRelayOrigin = {
  type: "relay",
  relays: [* tstr], ; query these relays
}

FeedOutboxOrigin = {
  type: "outbox",
  ? authors: [* tstr],
  ? strategy: "outbox" / "inbox" / "auto",
}

NostrFilter = {
  ? ids: [* tstr],
  ? authors: [* tstr],
  ? kinds: [* uint],
  ? since: uint,
  ? until: uint,
  ? limit: uint,
  ? search: tstr,
  * tstr => [* tstr], ; single-letter tag filters, e.g. "#t" or "#p"
}
```

Fields:

| Field | Type | Required | Semantics |
|-------|------|----------|-----------|
| `filters` | `NostrFilter[]` | yes | One or more NIP-01 filters describing the feed. Multiple filters are combined as a union (the same OR semantics as the elements of a single NIP-01 `REQ`). Producers send a single-element array for the common single-filter case. |
| `origin` | `FeedOrigin` | no | The routing *method* for sourcing events, not the interface that fulfills it. `{type:"relay",relays}` → query those relays. `{type:"outbox",authors?,strategy?}` → the outbox model: resolve each author's relays from their published relay list and query there (authors default to those derived from `filters`). Absent → consumer policy. Hints, not an access grant. |
| `origin.strategy` | `string` | no | Outbox origin only: `"outbox"` (authors' write relays, default), `"inbox"` (authors' read relays), or `"auto"`. |
| `title` | `string` | no | Human-readable label hint for the feed (e.g. `"#nostr"`, `"Ada's notes"`). For optimistic UI only; not authoritative. |
| `source` | `object` | no | Advisory provenance: the producing napplet, its window, and an opaque correlation id. |
| `source.requestId` | `string` | no | Opaque correlation id a consumer MAY echo on a future `feed:*` response topic. |
| `behavior.focus` | `boolean` | no | Whether the feed should take focus. Defaults to `true`. |
| `behavior.newWindow` | `boolean` | no | Request a new feed surface rather than reusing the current one. Defaults to shell policy; producers SHOULD leave it unset unless the user explicitly asked for a new feed. |

Producer requirements:

- Producers MUST include a non-empty `filters` array.
- Producers SHOULD use canonical lowercase hexadecimal values for `ids`,
  `authors`, and `#p`/`#e` tag filters.
- Producers MAY include `origin`, `title`, `source`, and `behavior` as hints.
- Producers name an author-routed feed with `origin: { type: "outbox", authors }`
  and a relay-pinned feed with `origin: { type: "relay", relays }`.
- Producers MUST treat `origin` and `title` as advisory and MUST NOT require a
  specific feed implementation or windowing policy.

Consumer requirements:

- Consumers MUST ignore payloads whose `filters` is missing, not an array, or
  empty.
- Consumers SHOULD treat each member of `filters` as a NIP-01 filter and render
  the union of matching events.
- Consumers MUST validate filter contents (hex lengths, kind/`since`/`until`
  types) before use, and MAY reject a filter that exceeds their resource policy.
- `limit` is not authoritative: the runtime owns fetch policy (page size,
  pagination, cost bounds) and MAY ignore a producer-supplied `limit`.
- Consumers SHOULD resolve `origin` as a *method*, fulfilled through whatever
  relay access they have — this NAP names the method, not the interface:
  - `relay` — query the named relays.
  - `outbox` — query each author's relays, resolved from their published relay
    lists (`strategy` selects the authors' write or read relays).
- If `origin` is absent, the consumer applies its own relay-selection policy.
- A consumer that cannot perform a given method MAY fall back to its own relay
  policy, or ignore the request.
- Consumers MUST treat `origin` (relay URLs and author pubkeys) as untrusted and
  validate it before use; it is a hint, never a relay-access grant.
- Consumers MAY show `title` while loading, but MUST treat fetched events as the
  authoritative content.
- Consumers MAY ignore the message when feed presentation is unsupported in the
  current context.

## Relationship to NAP-INTENT and the `feed` archetype

`feed:open` is the wire format a caller uses to open a feed napplet by role
through [NAP-INTENT](NAP-INTENT.md):

```js
// A hashtag chip opens a relay-sourced feed:
napplet.intent.open("feed", {
  filters: [{ kinds: [1], "#t": ["nostr"] }],
  origin: { type: "relay", relays: ["wss://relay.example"] },
  title: "#nostr"
});

// A profile view opens an outbox-routed feed of that author's notes:
napplet.intent.open("feed", {
  filters: [{ kinds: [1] }],
  origin: { type: "outbox", authors: ["<hex pubkey>"] },
  title: "Ada's notes"
});
```

A runtime opening a feed with no producer typically defaults to the user's own
notes (`origin: { type: "outbox", authors: [<user>] }`, `kinds: [1]`) and extends
it as the user edits. This default is the runtime's, not part of the wire format
(non-normative).

The `Serves: feed/open` header self-registers this protocol as the recommended
open contract for the [`feed`](../naat/feed.md) archetype, without editing the
ARCHETYPES registry. NAP-INTENT governs role resolution, default-handler
selection, and window lifecycle; this NAP governs only the payload and its
delivery over NAP-INC.

## Growth and Competing Drafts

This NAP defines one feed-criteria coordination surface. Future numbered NAP-N
drafts MAY define alternate or expanded `feed:*` topic sets — for example a
richer feed-description format, paged-cursor coordination, or a
`feed:current-criteria` report topic. Napplets discover the specific numbered
protocol with `shell.supports("inc", "NAP-5")`; ecosystem preference between
competing feed-topic NAPs is determined by implementation adoption.

## Negotiation

Napplets discover support for this protocol with:

```ts
window.napplet.shell.supports("inc", "NAP-5");
```

A napplet that requires INC transport declares the interface dependency in its
manifest:

```
["requires", "inc"]
```

A feed napplet that fulfills the `feed` archetype via this protocol declares:

```
["archetype", "feed", "NAP-5"]
```

The manifest declares only the interface dependency and the archetype it
fulfills. The numbered protocol is negotiated at runtime with
`shell.supports()`.

## Security Considerations

- `feed:open` is a navigation request, not a relay-access grant. Consumers MUST
  retrieve events through their own authorized relay access and shell policy,
  never by treating the message as permission to reach arbitrary relays.
- `origin` (relay URLs and author pubkeys) and all filter contents are untrusted
  input. Consumers MUST validate ids, pubkeys, kinds, and tag values before
  placing them in a subscription, and SHOULD bound the cost of a request (filter
  count, unbounded time ranges, the number of authors driving outbox routing)
  to avoid a producer driving expensive queries.
- A `search` filter may be forwarded to a relay; consumers SHOULD treat it as an
  opaque user string and MUST NOT interpolate it into any other context.

## Non-goals

- Defining generic INC transport.
- Defining how a feed fetches, deduplicates, orders, paginates, or renders
  events.
- Defining relay-selection, outbox, or NIP-50 search-relay policy.
- Defining shell window placement, focus, or lifecycle policy.
- Reserving non-feed topic families such as `note:*`, `profile:*`, `stream:*`,
  `chat:*`, `wm:*`, or `shell:*`.

## Implementations

- (none yet)
