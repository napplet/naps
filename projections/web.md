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

## Worker binding (draft)

[NAP-WORKER](../naps/NAP-WORKER.md) proposes `window.napplet.worker` for
runtime-owned dedicated Web Workers. This section binds that draft's operations;
the core transport and identity rules remain defined by
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303).

**A Worker created by the parent frame is not automatically a napplet sandbox.**
The runtime MUST enforce NAP-WORKER isolation before exposing this domain.
[Worker origin rules](https://html.spec.whatwg.org/multipage/workers.html#script-settings-for-workers)
can associate a worker with its creator's origin. Creating a Blob URL in the
shell is not sufficient isolation. Blocking network requests alone does not
isolate origin-scoped storage or communication channels.

The shell in the parent frame owns the Worker objects and proxies messages
between each napplet iframe and its workers. It MUST NOT expose a Worker object
or transfer a direct communication port to the napplet. The napplet iframe's
sandbox and network restrictions remain in force.

| Contract | Web binding |
|----------|-------------|
| Domain availability | Presence of `window.napplet.worker`, injected before napplet scripts run. |
| `info`, `create`, `post`, `terminate` | Asynchronous methods on that object; resolve to their contract result, or reject with `WorkerError`. Acknowledgments resolve without a value. |
| `create(source)` | Source is a self-contained classic JavaScript worker program, passed as text. No URL argument, module imports, or external script loading. |
| Program receives data | Standard worker `message` event; input is `event.data`. |
| Program sends data | Standard worker `postMessage(data)`; only `WorkerData` is accepted. |
| Program closes | Standard worker `close()`; the binding reports `worker.closed` with reason `selfClosed`. |
| `onEvent(handler)` | Receives the NAP's event record and returns a local unsubscribe function. |
| Message size | UTF-8 byte length of compact JSON serialization of `data`, without an envelope. |

The shell MUST validate both directions as `WorkerData`, despite the broader
values accepted by browser structured cloning. Transfer lists, MessagePorts,
ArrayBuffers, and SharedArrayBuffers are outside this draft. Serialization MUST
NOT silently discard unsupported values or turn non-finite numbers into null.

For each Worker, the runtime MUST install its receiver before running supplied
source and retain the authenticated owning iframe endpoint. Worker output is
wrapped as `worker.message.data`; an output object resembling a NAP envelope
MUST NOT enter the shell's request dispatcher. Worker events are bound through
the actual Worker object, never through an identity claimed inside its output.
The binding MUST observe program closure and execution failures, order events
after the creation result, and preserve the NAP's terminal-event rules.

The runtime MUST prevent access to shell-origin storage, credentials, network,
cross-context channels, and nested workers. Removing selected globals or
rewriting supplied source alone is not a security boundary. The execution
environment MUST enforce these restrictions independently of the program. A
runtime unable to do so MUST leave `window.napplet.worker` absent.

Illustrative napplet-side use, with a listener installed before creation:

```js
const stopListening = window.napplet.worker.onEvent(event => {
  if (event.type === 'worker.message') console.log(event.workerId, event.data);
});
const { workerId } = await window.napplet.worker.create(
  'self.onmessage = event => self.postMessage(event.data);'
);
await window.napplet.worker.post(workerId, { hello: 'worker' });
// Later, when computation is no longer needed:
await window.napplet.worker.terminate(workerId);
stopListening();
```

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
