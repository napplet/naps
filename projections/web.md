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
| Host | Napplets run as `sandbox="allow-scripts"` iframes |
| Carrier | Messages travel over `postMessage` |
| Surface | Capabilities appear on a `window.napplet.*` object |
| Discovery | `shell.supports("<domain>")`, optionally `shell.supports("<domain>", "NAP-N")` |
| Identity | Runtime verifies `MessageEvent.source` and binds each message to a napplet |

## Domain surfacing

A NAP is named in the registry by its **domain** (`relay`, `intent`, …). In the
web projection, domain `X`:

- surfaces as the object `window.napplet.X`, and
- is discovered via `shell.supports("X")`.

So `NAP-RELAY` (domain `relay`) is reached at `window.napplet.relay` and probed
with `shell.supports("relay")`. Other projections map the same domains into their
own host idiom.

## Message delivery

Request/result objects (the `domain.action` envelopes described in the registry)
are delivered by `postMessage`:

```
-> { "type": "relay.publish", "id": "a1", "event": { … } }   // napplet → shell
<- { "type": "relay.publish.result", "id": "a1", "ok": true } // shell → napplet
```

## Identity & trust

The shell is the policy boundary. For every inbound message it verifies
`MessageEvent.source` to bind the message to a napplet identity — the
`(dTag, aggregateHash)` tuple, assigned by the shell from the napplet's
[NIP-5A](https://github.com/nostr-protocol/nips/blob/master/5A.md) manifest, not
negotiated by the napplet. Napplets are untrusted: they never receive signing
keys, wallet credentials, or raw network access. Security-critical operations are
performed by the shell on the napplet's behalf, gated by per-napplet capability
policy.

## References

- [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) — normative web binding (living, upstream document)
- [NIP-5A](https://github.com/nostr-protocol/nips/blob/master/5A.md) — napplet manifest / identity
- [Registry & governance](../README.md)
