Web Projection — NIP-5D
=======================

The **web projection** of the NAP capability seam. It maps the binding-neutral
contracts in the [registry](../README.md) onto the browser, and is defined
normatively by [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) — the
living, upstream document.

A *projection* answers four binding-specific questions for one host environment:
where napplets run, how messages travel, how a napplet's identity is bound, and
how each NAP **domain** is surfaced. The contracts themselves (operations,
schemas, error models, trust boundaries) do not change between projections.

## At a glance

| Concern | Web projection |
|---------|----------------|
| Host | Verified napplet artifacts run in `srcdoc` iframes with `sandbox="allow-scripts"` |
| Carrier | Messages travel over `postMessage` |
| Surface | Capabilities and convention URI transposition appear on `window.napplet.*` |
| Discovery | Presence of `window.napplet.<domain>`, injected before napplet scripts run |
| Identity | Runtime verifies `MessageEvent.source` and binds each message to a napplet |

## Domain surfacing

A NAP is named in the registry by its **domain** (`relay`, `intent`, …). In the
web projection, domain `X`:

- surfaces as the object `window.napplet.X`, and
- is available when that object is present. No `shell` handshake is required.

So `NAP-RELAY` (domain `relay`) is reached at `window.napplet.relay` and probed
by checking whether that object is present. Other projections map the same
domains into their own host idiom.

## Message delivery

Request/result objects (the `domain.action` envelopes described in the registry)
are delivered by `postMessage`:

```
-> { "type": "relay.publish", "id": "a1", "event": { … } }   // napplet → shell
<- { "type": "relay.publish.result", "id": "a1", "ok": true } // shell → napplet
```

## Convention URI binding

A developer MAY pass
`napplet:<archetype>/<intent>[...?params]` to a `window.napplet.*` operation that
accepts a convention URI. Before `postMessage`, the web binding:

1. extracts any NAP-INTENT handler fragment into `handlerHint` as specified below,
2. removes the query from the stable convention identity,
3. percent-decodes each unique `name=value` pair as text, and
4. places those pairs in the operation's payload object.

The binding MUST NOT coerce scalar types or apply form-encoding semantics: `+`
is a literal plus sign. It MUST reject malformed percent-encoding,
repeated names, and a query combined with an explicit payload before sending a
message. The shell receives normalized identity and payload fields. Routing and
handler resolution use exact equality over the queryless, fragment-free identity.

### Intent handler fragment

The URI form of `window.napplet.intent.invoke` additionally accepts:

```text
napplet:<archetype>/<intent>[?params][#<naddr>]
```

The web binding MUST implement the normalization and validation in
[NAP-INTENT](../naps/NAP-INTENT.md#convention-uri-normalization). It decodes the
bare `naddr` into `request.handlerHint.address` and optional
`request.handlerHint.relays` before `postMessage`. The raw fragment MUST NOT
appear in the wire convention or payload. A queryless invocation MAY combine
the fragment with an explicit structured payload.

The fragment is a recommendation, not an explicit handler selection. The shell
applies NAP-INTENT's user-default precedence, discovery policy, and fallback.
Every other convention-URI operation, including NAP-INC, MUST reject fragments.

## Internal navigation binding (draft)

[NAP-NAVIGATION](../naps/NAP-NAVIGATION.md) exposes `report` and `get` through
`window.napplet.navigation`. It requires `intent`; domain presence still follows
the ordinary injected-namespace rules, without a `shell` handshake.

A router adapter reports committed views as advertised convention identities
and payload maps containing only advertised parameter names. It reports the
complete retained history and current index. Unadvertised internal routes MUST
NOT be retained as locations. Query sugar is transposed before reporting; stored
convention identities are queryless and fragment-free.

Navigation requests use `window.napplet.intent` delivery handling. There is no
separate navigation listener or outcome binding. Intent-based history traversal
is not yet fully specified; see the draft's
[intent integration questions](../naps/NAP-NAVIGATION.md#intent-integration-still-to-specify).

Native `window.location`, `history.pushState`, `history.replaceState`, and hash
changes do not report navigation by themselves. The parent MUST NOT assume it
can inspect an opaque-origin iframe's internal router. No automatic replacement
of browser APIs is required.

The shell's top-level URL and browser Back button remain shell policy. A stored
invocation MUST NOT replace the verified `srcdoc` artifact or be assigned to an
iframe's `src`. Reporting it does not dispatch it.

## Identity & trust

The shell is the policy boundary. For every inbound message it verifies
`MessageEvent.source` to bind the message to a napplet identity — the
verified artifact identity defined by
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303). The runtime verifies the signed
manifest and its artifact hash before execution; it does not trust an identity
supplied by the napplet or a gateway. Napplets are untrusted: they never receive signing
keys, wallet credentials, or raw network access. Security-critical operations are
performed by the shell on the napplet's behalf, gated by per-napplet capability
policy.

## References

- [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) — normative web binding (living, upstream document)
- [Registry & governance](../README.md)
