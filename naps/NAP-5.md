NAP-5
======

Feed Topics
-----------

`draft`

**NAP ID:** NAP-5

**Domain:** feed topic coordination

**Requires:** NAP-INC, NAP-RELAY

**Serves:** `feed/open`

**Discovery:** `shell.supports("inc", "NAP-5")`

## Description

This protocol defines a coherent `feed:*` topic family for napplets that hand a
feed napplet the criteria for *what to show* — a set of [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md)
filters and, optionally, the relays to source those events from. The current
revision defines:

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
or more NIP-01 filters, optionally sourced from a specific set of relays.

Transport:

```ts
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

```ts
type FeedOpenPayload = {
  filters: NostrFilter[];   // one or more NIP-01 filters, OR-combined (REQ semantics)
  relays?: string[];        // optional relay URL hints to source the feed from
  title?: string;           // optional human-readable label hint for the feed
  source?: {
    napplet?: string;       // dTag of the producer, advisory
    windowId?: string;      // producer window, advisory
    requestId?: string;     // opaque correlation id
  };
  behavior?: {
    focus?: boolean;        // default true
    newWindow?: boolean;    // default shell policy
  };
};

// A standard NIP-01 filter. Every member is optional; an empty filter matches
// everything and SHOULD be paired with `limit` or a `relays` hint.
type NostrFilter = {
  ids?: string[];           // 64-char lowercase hex event ids
  authors?: string[];       // 64-char lowercase hex pubkeys
  kinds?: number[];
  since?: number;           // unix seconds, inclusive lower bound on created_at
  until?: number;           // unix seconds, inclusive upper bound on created_at
  limit?: number;           // maximum events for the initial page
  search?: string;          // NIP-50 full-text query, where supported
  // single-letter tag filters, e.g. "#t": ["nostr"], "#p": ["<hex>"]
  [singleLetterTag: `#${string}`]: string[] | undefined;
};
```

Fields:

| Field | Type | Required | Semantics |
|-------|------|----------|-----------|
| `filters` | `NostrFilter[]` | yes | One or more NIP-01 filters describing the feed. Multiple filters are combined as a union (the same OR semantics as the elements of a single NIP-01 `REQ`). Producers send a single-element array for the common single-filter case. |
| `relays` | `string[]` | no | Relay URL hints from which to source the feed. When present, the consumer SHOULD prefer them; when absent, the consumer selects relays by its own policy (e.g. the user's configured relays or outbox routing). Advisory only. |
| `title` | `string` | no | Human-readable label hint for the feed (e.g. `"#nostr"`, `"Ada's notes"`). For optimistic UI only; not authoritative. |
| `source` | `object` | no | Advisory provenance: the producing napplet, its window, and an opaque correlation id. |
| `source.requestId` | `string` | no | Opaque correlation id a consumer MAY echo on a future `feed:*` response topic. |
| `behavior.focus` | `boolean` | no | Whether the feed should take focus. Defaults to `true`. |
| `behavior.newWindow` | `boolean` | no | Request a new feed surface rather than reusing the current one. Defaults to shell policy; producers SHOULD leave it unset unless the user explicitly asked for a new feed. |

Producer requirements:

- Producers MUST include a non-empty `filters` array.
- Producers SHOULD use canonical lowercase hexadecimal values for `ids`,
  `authors`, and `#p`/`#e` tag filters.
- Producers MAY include `relays`, `title`, `source`, and `behavior` as hints.
- Producers MUST treat `relays` and `title` as advisory and MUST NOT require a
  specific feed napplet implementation or shell windowing policy.

Consumer requirements:

- Consumers MUST ignore payloads whose `filters` is missing, not an array, or
  empty.
- Consumers SHOULD treat each member of `filters` as a NIP-01 filter and render
  the union of matching events.
- Consumers MUST validate filter contents (hex lengths, kind/`limit`/`since`/
  `until` types) before passing them to a NAP-RELAY subscription, and MAY
  reject or clamp a filter that exceeds their resource policy (e.g. cap an
  absent or oversized `limit`).
- Consumers SHOULD prefer `relays` when present and otherwise apply their own
  relay-selection policy; relay hints are untrusted input.
- Consumers MAY show `title` while loading, but MUST treat fetched events as the
  authoritative content.
- Consumers MAY ignore the message when feed presentation is unsupported in the
  current context.

## Relationship to NAP-INTENT and the `feed` archetype

`feed:open` is the wire format a caller uses to open a feed napplet by role
through [NAP-INTENT](NAP-INTENT.md):

```js
napplet.intent.open("feed", {
  filters: [{ kinds: [1], "#t": ["nostr"], limit: 50 }],
  relays: ["wss://relay.example"],
  title: "#nostr"
});
```

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
  retrieve events through their own authorized NAP-RELAY surface and shell
  policy, never by treating the message as permission to reach arbitrary relays.
- `relays` and all filter contents are untrusted input. Consumers MUST validate
  ids, pubkeys, kinds, and tag values before placing them in a relay
  subscription, and SHOULD bound the cost of a request (filter count, absent or
  oversized `limit`, unbounded time ranges) to avoid a producer driving
  expensive queries.
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
