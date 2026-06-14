NAP: Nostr Applet Protocol
==========================

NAPs define the **capability seam** between a napplet and its runtime — the
contract for what a runtime provides (relay access, storage, intents, …) and how
a napplet asks for it. The seam is **transport-agnostic**:
[NIP-5D](https://github.com/nostr-protocol/nips/blob/master/5D.md) is its
*binding* for the web — napplets as sandboxed iframes communicating over
`postMessage` — but the same NAP contracts can be implemented by other runtimes.
What's fixed is the contract; what varies per binding is the host environment and
the message carrier.

This repository is the registry and governance home for those contracts. By the
end of this document you should understand what a napplet is, what a NAP is (and
is not), how the pieces layer together, and how to read or propose a spec.

## What is a napplet?

A napplet is a Nostr applet — a small, single-purpose application that does one
thing well. A chat widget, a feed viewer, a profile editor, and a relay manager
are four napplets, not one app with four tabs. The **runtime composes napplets;
napplets do not compose themselves.**

A napplet is described and distributed as a
[NIP-5A](https://github.com/nostr-protocol/nips/blob/master/5A.md) manifest (a
Nostr event, kind 35128): a pubkey-addressed `dTag`, an aggregate hash of its
build, and the capabilities it requires. That manifest is the napplet's identity,
independent of how or where it runs.

## What is a NAP?

A NAP is one capability contract in that seam. `NAP-RELAY` says *a runtime can
proxy relay reads and writes, and here is exactly how a napplet requests them.*
`NAP-INTENT` says *a runtime can open another napplet by role, and here is the
request/result shape.* Each NAP standardizes the **operations, their semantics,
the message schema, the error model, and the trust boundary** — never the
delivery mechanism.

A NAP is therefore **not**:

- **the transport.** `postMessage`, iframes, and the `window.napplet.*` object
  are web-*binding* details defined by NIP-5D, not by the NAP. The same
  `NAP-RELAY` contract could be carried by IPC, an FFI call, or WASM host
  imports.
- **Nostr itself.** Napplets are Nostr-native — identities are pubkeys,
  manifests are events, payloads carry Nostr data. NAPs standardize the *runtime
  contract* around that data; they do not redefine Nostr. The seam is
  transport-agnostic, not Nostr-agnostic.

## Layering

```
NIP-5A   what a napplet IS / how it's described      manifest, identity   (substrate)
  NAP    what a runtime offers a napplet             the capability seam  (this repo)
   └─ binding: NIP-5D — web (iframe + postMessage)   the only binding today
   └─ binding: native, WASM, …                       same contracts, different host
```

NIP-5A defines the napplet. A NAP defines a capability the napplet can ask a
runtime for. A *binding* implements that seam for a concrete host.

## Bindings

A **binding** implements the NAP seam for a host environment: it provides
capability discovery, napplet identity binding, a request/result channel, and
mediation of sensitive operations.

- **Web — [NIP-5D](https://github.com/nostr-protocol/nips/blob/master/5D.md)**,
  the binding in use today. Napplets run as `sandbox="allow-scripts"` iframes;
  messages travel over `postMessage`; capabilities surface on a
  `window.napplet.*` object. The local copy of this spec is [SPEC.md](SPEC.md).

The contracts are transport-neutral by design, so a native host (OS process +
IPC/FFI) or a WASM runtime (host imports) could provide the same seam. None exist
yet — the web binding is the only implementation, and the specs are written
against it — but the contracts are shaped so they need not stay web-only.

**Web projection.** Throughout the registry a NAP is named by its **domain**
(`relay`, `intent`). In the web binding, domain `X` surfaces as
`window.napplet.X` and is discovered via `shell.supports("X")`. Other bindings
project the same domains into their own host idiom.

## How it works

The mechanics below are described at the seam (binding-neutral); the web
projection is shown inline.

**Discovery.** A runtime advertises the capabilities it provides. A napplet
checks for one before using it — by domain, optionally with a numbered protocol:

```
shell.supports("relay")          // is the relay capability available?
shell.supports("inc", "NAP-2")   // …and does it speak the NAP-2 wire format?
```

A napplet also declares the capabilities it needs in its NIP-5A manifest
(`["requires", "relay"]`); a runtime that lacks one may refuse to load it.

**Request / result.** Messages are objects with a `type` discriminant in
`domain.action` form. Request/result pairs correlate by `id`; fire-and-forget
messages omit it; runtimes may push unsolicited messages.

```
-> { "type": "relay.publish", "id": "a1", "event": { … } }   // napplet → runtime
<- { "type": "relay.publish.result", "id": "a1", "ok": true } // runtime → napplet
```

In the web binding these objects are delivered by `postMessage`, and the runtime
verifies `MessageEvent.source` to bind every message to a napplet identity.

**Mediation & trust.** The runtime is the policy boundary. Napplets are
untrusted: they never receive signing keys, wallet credentials, or raw network
access; security-critical operations (signing, payments, uploads) are performed
by the runtime on the napplet's behalf, gated by per-napplet capability policy.
Napplet identity — the `(dTag, aggregateHash)` tuple — is assigned by the runtime
from the manifest, not negotiated by the napplet.

## The three axes

A complete picture of a napplet ecosystem needs three orthogonal things: what the
runtime *offers*, what napplets *say to each other*, and what kind of napplet
each *is*.

### NAP-WORD — interfaces (*what the runtime offers*)

Named by a single uppercase word, one canonical spec per name. Defines a
shell-provided API contract — a capability domain a napplet can call. Discovery:
`shell.supports("<domain>")`.

| NAP ID | Domain | Description | Status |
|--------|--------|-------------|--------|
| [NAP-RELAY](https://github.com/napplet/naps/pull/2) | `relay` | Relay proxy (subscribe, publish, query, publishEncrypted) | Draft |
| [NAP-IDENTITY](https://github.com/napplet/naps/pull/12) | `identity` | Read-only user identity queries | Draft |
| [NAP-STORAGE](https://github.com/napplet/naps/pull/3) | `storage` | Scoped key-value storage | Draft |
| [NAP-INC](https://github.com/napplet/naps/pull/5) | `inc` | Inter-napplet communication | Draft |
| [NAP-THEME](https://github.com/napplet/naps/pull/8) | `theme` | Shell-provided theming | Draft |
| [NAP-KEYS](https://github.com/napplet/naps/pull/9) | `keys` | Keyboard forwarding and action keybindings | Draft |
| [NAP-MEDIA](https://github.com/napplet/naps/pull/10) | `media` | Media session control and playback | Draft |
| [NAP-NOTIFY](https://github.com/napplet/naps/pull/11) | `notify` | Shell-rendered notifications | Draft |
| [NAP-INTENT](naps/NAP-INTENT.md) | `intent` | Invoke a napplet by archetype (default-handler dispatch) | Draft |

### NAP-N — wire formats (*what napplets say to each other*)

Numbered sequentially (`NAP-1`, `NAP-2`, …). Multiple competing specs are allowed
per domain — they define the *semantics of messages* napplets exchange, not an
API the runtime provides. Napplets negotiate them at runtime via
`shell.supports("<domain>", "NAP-N")`. Existing protocols cover cross-napplet
navigation (opening a profile, a note, a DM) and stream coordination; example
domains include feed rendering, chat, and collaborative editing.

A NAP-N that shapes an archetype open payload declares `Serves: <slug>/<action>`,
so it self-registers against a [NAAT](ARCHETYPES.md) without editing the registry.

### NAAT — archetypes (*what kind of napplet this is*)

Canonical napplet *roles* — `note`, `feed`, `profile`, `emoji-list`. A NAAT is
**not** a NAP: neither an interface nor a wire format, just a name and a boundary.
Archetypes are rows in the [ARCHETYPES.md](ARCHETYPES.md) registry, each linking
to a thin file under [`naat/`](naat/). A napplet declares the roles it fulfills
with a `["archetype", "<slug>", "<NAP-N>"]` manifest tag, and napplets invoke
each other by role through [NAP-INTENT](naps/NAP-INTENT.md). A napplet with no
archetype tag is fully valid — it simply isn't invokable by role.

## Boundary rule

An interface (NAP-WORD) is **runtime-provided** AND defines an **API surface**. A
protocol (NAP-N) is **napplet-agreed** AND defines **message semantics**. An
archetype (NAAT) is a **canonical role name** with a **boundary**, owning neither
an API nor a payload — it only *recommends* a NAP-N as its default open contract.
Both NAP criteria must apply for a NAP; edge cases are judged pragmatically by the
maintainer.

## Governance

NIP-style informal process:

- Fork this repo, add a markdown file under [`naps/`](naps/) following the
  appropriate template, open a PR. Every NAP spec — both NAP-WORD interfaces and
  numbered NAP-N wire formats — lives in the `naps/` directory; the templates
  and registries (`README.md`, `ARCHETYPES.md`) stay at the repo root.
- Community discusses via PR comments.
- Maintainer (dskvr) merges when the spec makes sense and has at least one
  implementation.
- No formal stages, review committees, or voting.
- NAP-WORD names and NAAT slugs are first-come-first-served but must be approved
  by the maintainer.
- NAP-N numbers are assigned sequentially on merge.

## Templates

- Interface proposals: [NAP-WORD-TEMPLATE.md](NAP-WORD-TEMPLATE.md)
- Protocol proposals: [NAP-N-TEMPLATE.md](NAP-N-TEMPLATE.md)
- Archetype proposals: [naat/TEMPLATE.md](naat/TEMPLATE.md) + a row in [ARCHETYPES.md](ARCHETYPES.md)

## References

- Web binding: [NIP-5D](https://github.com/nostr-protocol/nips/blob/master/5D.md) — local mirror [SPEC.md](SPEC.md)
- Napplet manifest / identity: [NIP-5A](https://github.com/nostr-protocol/nips/blob/master/5A.md)
- Archetype registry: [ARCHETYPES.md](ARCHETYPES.md)
