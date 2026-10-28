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

## Resource binding

### Network-isolation prerequisite

Exposing `window.napplet.resource` requires enforced direct-network denial for
that napplet before any napplet-controlled code or resource is processed, and
throughout its lifetime. An opaque iframe origin is insufficient. This is a
NAP-RESOURCE prerequisite beyond the advisory CSP in
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303), not a claim that the
upstream sandbox already guarantees it.

The shell MUST enforce a browser policy blocking direct `fetch`, XHR,
WebSocket, EventSource, beacon, and external subresource loads. It MUST also
prevent workers, nested contexts, navigation, or other browser facilities from
bypassing that policy. Follow the upstream CSP placement and artifact-verification
rules; replacing JavaScript globals alone is not enforcement. The shell MUST NOT
apply the upstream direct-network opt-in exception to a napplet exposed to this
domain. If the host cannot enforce these restrictions, it MUST NOT expose
`window.napplet.resource` to that napplet.

### Bytes and local helpers

All wire `blob` fields retain NAP-RESOURCE's canonical `ResourceBytes` base64
text across `postMessage`. The binding MUST validate and decode the text, then
construct a `Blob` from the decoded octets with the runtime-classified `mime` as
its media type. `resource.bytes()` returns that Blob. Successful
`resource.bytesMany()` items expose the decoded Blob in `blob`; failed items
remain errors. Imported resource sidecars use the same conversion before cache
hydration. The binding MUST NOT accept Blob, ArrayBuffer, or integer-array wire
alternatives. Invalid encoding is a protocol failure and MUST NOT reach a success
callback or cache entry.

`bytesAsObjectURL(url)` is a local web helper over `bytes`: it creates an object
URL for the returned Blob and returns `{ url, revoke }`, where `revoke` releases
that URL. It adds no wire message. An optional `opts.signal` on `bytes` or
`bytesMany` maps abort to the NAP's `resource.cancel` message and suppresses late
results for the cancelled request.

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

## Identity & trust

The shell is the policy boundary. For every inbound message it verifies
`MessageEvent.source` to bind the message to a napplet identity — the
verified artifact identity defined by
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303). The runtime verifies the signed
manifest and its artifact hash before execution; it does not trust an identity
supplied by the napplet or a gateway. Napplets are untrusted: they never receive signing
keys or wallet credentials. Direct-network denial for resource-enabled napplets
is enforced as specified in [Resource binding](#resource-binding). Security-critical operations are
performed by the shell on the napplet's behalf, gated by per-napplet capability
policy.

## References

- [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) — normative web binding (living, upstream document)
- [Registry & governance](../README.md)
