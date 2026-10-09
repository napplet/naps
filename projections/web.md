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
| Surface | Capabilities and convention URI transposition appear on `window.napplet.*` |
| Discovery | `shell.supports("<domain>")` |
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

## Convention URI binding

A developer MAY pass
`napplet:<archetype>/<intent>[...?params]` to a `window.napplet.*` operation that
accepts a convention URI. Before `postMessage`, the web binding:

1. removes the query from the stable convention identity,
2. percent-decodes each unique `name=value` pair as text, and
3. places those pairs in the operation's payload object.

The binding MUST NOT coerce scalar types or apply form-encoding semantics: `+`
is a literal plus sign. It MUST reject fragments, malformed percent-encoding,
repeated names, and a query combined with an explicit payload before sending a
message. The shell receives normalized identity and payload fields. Routing and
handler resolution use exact equality over the queryless identity.

## Identity & trust

The shell is the policy boundary. For every inbound message it verifies
`MessageEvent.source` to bind the message to a napplet identity — the
`(dTag, aggregateHash)` tuple, assigned by the shell from the napplet's
[NIP-5A](https://github.com/nostr-protocol/nips/blob/master/5A.md) manifest, not
negotiated by the napplet. Napplets are untrusted: they never receive signing
keys, wallet credentials, or raw network access. Security-critical operations are
performed by the shell on the napplet's behalf, gated by per-napplet capability
policy.

## Capture

[NAP-CAPTURE](../naps/NAP-CAPTURE.md) does not weaken or relax this projection's
sandbox. Web runtimes remain governed by
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303), including
source-window authentication and `sandbox="allow-scripts"` without
`allow-same-origin`.

For capture authorization, a web shell MUST maintain an opaque registration
generation in addition to each registered iframe `Window` reference and verified
identity. Initial registration creates it. Navigation, reload, unregister, or
replacement MUST invalidate it and tear down its capture state before the shell
accepts messages under a new registration, even if the browser reuses the same
`WindowProxy` and the verified identity is unchanged. The web owner key is the
current `(Window reference, registration generation)` pair. This token is
shell-internal and MUST NOT be accepted from or exposed to the napplet.

Web runtimes MUST represent `CaptureArtifact.data` as a `Blob` carried as a
member of the `postMessage` envelope by structured clone, without a transfer
list. `CaptureArtifact.size` MUST equal `Blob.size`, and the `Blob.type` MUST be
MIME-compatible with `CaptureArtifact.mimeType` under NAP-CAPTURE's comparison
algorithm. Runtimes MUST NOT JSON-stringify this envelope, substitute an
`ArrayBuffer` or typed array, or encode audio as text or base64. Structured
cloning the immutable `Blob` does not transfer ownership; the runtime retains
its reference until the capture lifecycle erases it.

## References

- [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) — normative web binding (living, upstream document)
- [NIP-5A](https://github.com/nostr-protocol/nips/blob/master/5A.md) — napplet manifest / identity
- [Registry & governance](../README.md)
