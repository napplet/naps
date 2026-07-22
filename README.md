NAP: Nostr Applet Protocol
==========================

NAPs define the **capability seam** between a napplet and its runtime: the
contract for what a runtime offers (relay access, storage, intents, …) and how a
napplet asks for it. The contract is fixed; how it is delivered varies by
**projection**. This repository is the registry and governance home for those
contracts.

## Contents

- [Glossary](#glossary)
- [What is a napplet?](#what-is-a-napplet)
- [What is a NAP?](#what-is-a-nap)
- [Layering](#layering)
- [Projections](#projections)
- [How it works](#how-it-works)
- [The two axes](#the-two-axes)
- [Boundary rule](#boundary-rule)
- [Governance](#governance)
- [Templates](#templates)
- [References](#references)

## Glossary

| Term | Meaning |
|------|---------|
| **Seam** | The boundary between a napplet and its runtime — what's offered, and how it's asked for. Transport-agnostic. |
| **Napplet** | A Nostr applet: a small, single-purpose app. Described by a [NIP-5A](https://github.com/nostr-protocol/nips/blob/master/5A.md) manifest. |
| **Runtime** (shell) | The host that composes napplets and provides their capabilities. |
| **NAP** | One capability contract in the seam — operations, message schema, error model, and trust boundary. Never the delivery mechanism. |
| **Domain** | A capability's short name (`relay`, `intent`); how a NAP is referenced and discovered. |
| **Projection** (binding) | A mapping of the seam onto one concrete host (web, native, WASM, …). |
| **NAP-WORD** | An interface spec — an API the runtime offers. One canonical spec per name. |
| **Convention** | An unnumbered message shape napplets agree to use. Usually named as `napplet:<archetype>/<intent>[...?params]`, e.g. `napplet:note/open`. Not a NAP. |
| **NAAT** | A *Napplet Archetype*: a canonical role name (`note`, `feed`) with a boundary. Not a NAP. |

## What is a napplet?

A napplet is a Nostr applet — a small app that does one thing well. A chat
widget, a feed viewer, a profile editor, and a relay manager are four napplets,
not one app with four tabs. **The runtime composes napplets; napplets do not
compose themselves.**

A napplet is described and distributed as a
[NIP-5A](https://github.com/nostr-protocol/nips/blob/master/5A.md) manifest (a
Nostr event, kind 35128): a pubkey-addressed `dTag`, an aggregate hash of its
build, and the capabilities it requires. That manifest is the napplet's identity,
independent of how or where it runs.

## What is a NAP?

A NAP is one capability contract in the seam. `NAP-RELAY` says *a runtime can
proxy relay reads and writes, and here is exactly how a napplet requests them.*
`NAP-INTENT` says *a runtime can open another napplet by role, and here is the
request/result shape.*

A NAP is **not**:

- **the transport.** `postMessage`, iframes, and `window.napplet.*` are
  *projection* details (see [Projections](#projections)), not the NAP. The same
  `NAP-RELAY` contract could be carried by IPC, an FFI call, or WASM imports.
- **Nostr itself.** Napplets are Nostr-native — identities are pubkeys, manifests
  are events, payloads carry Nostr data. NAPs standardize the *runtime contract*
  around that data; they do not redefine Nostr. The seam is transport-agnostic,
  not Nostr-agnostic.

## Layering

```
NIP-5A   what a napplet IS / how it's described    manifest, identity   (substrate)
  NAP    what a runtime offers a napplet           the capability seam  (this repo)
   └─ projection: web, native, WASM, …             same contracts, different host
```

NIP-5A defines the napplet. A NAP defines a capability the napplet can ask a
runtime for. A *projection* implements that seam for a concrete host.

## Projections

A **projection** maps the seam onto a host environment: where napplets run, the
message carrier, identity binding, and how each domain is surfaced. The contracts
are transport-neutral by design — a domain named `relay` in this registry is the
same contract everywhere; only the host idiom changes.

| Projection | Status | Spec |
|------------|--------|------|
| **Web** — iframes + `postMessage`, capabilities on `window.napplet.*` | In use | [projections/web.md](projections/web.md) ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)) |
| Native (OS process + IPC/FFI) | Possible | — |
| WASM (host imports) | Possible | — |

The web projection is the only implementation today, and the specs are written
against it — but the contracts are shaped so they need not stay web-only.

## How it works

The mechanics below live at the seam and are described projection-neutrally; the
[web projection](projections/web.md) shows how each is realized in the browser.

**Discovery.** A runtime advertises the capabilities it provides; a napplet
checks for one before using it by domain:

```
shell.supports("relay")          // is the relay capability available?
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

**Mediation & trust.** The runtime is the policy boundary. Napplets are
untrusted: they never receive signing keys, wallet credentials, or raw network
access. Security-critical operations (signing, payments, uploads) are performed
by the runtime on the napplet's behalf, gated by per-napplet capability policy.
Napplet identity — the `(dTag, aggregateHash)` tuple — is assigned by the runtime
from the manifest, not negotiated by the napplet.

## The two axes

A complete napplet ecosystem needs two governed things: what the runtime
*offers*, and what kind of napplet each *is*. Cross-napplet message shapes are
conventions. They are not assigned NAP numbers.

### NAP-WORD — interfaces (*what the runtime offers*)

Named by a single uppercase word, one canonical spec per name. Defines a
shell-provided API contract — a capability domain a napplet can call. Discovery:
`shell.supports("<domain>")`.

Every NAP-WORD is **optional** — a runtime offers it or it doesn't, and a napplet
checks before using it — except **NAP-SHELL**, which every conformant runtime
MUST implement. NAP-SHELL is the foundational handshake that defines
`shell.supports()` itself, so it is the one capability that cannot be discovered
through it; it is assumed present.

The **Deps** column lists the domains a NAP rests on — declared in each spec's
`Depends:` block, always by domain (never a spec id).

| NAP ID | Domain | req. | Deps | Description | Status |
|--------|--------|------|------|-------------|--------|
| [NAP-SHELL](naps/NAP-SHELL.md) | `shell` | ✓ | — | Bootstrap handshake and capability negotiation (foundational — defines `shell.supports()`) | Active |
| [NAP-INTENT](naps/NAP-INTENT.md) | `intent` |  | — | Invoke a napplet by archetype (default-handler dispatch) | Active |
| [NAP-INC](https://github.com/napplet/naps/pull/5) | `inc` |  | — | Inter-napplet communication | Active |
| [NAP-THEME](https://github.com/napplet/naps/pull/8) | `theme` |  | — | Shell-provided theming | Active |
| [NAP-RELAY](https://github.com/napplet/naps/pull/2) | `relay` |  | `resource` | Relay proxy (subscribe, publish, query, publishEncrypted) | Draft |
| [NAP-IDENTITY](https://github.com/napplet/naps/pull/12) | `identity` |  | `resource` | Read-only user identity queries | Draft |
| [NAP-STORAGE](https://github.com/napplet/naps/pull/3) | `storage` |  | — | Scoped key-value storage | Draft |
| [NAP-KEYS](https://github.com/napplet/naps/pull/9) | `keys` |  | — | Keyboard forwarding and action keybindings | Draft |
| [NAP-MEDIA](https://github.com/napplet/naps/pull/10) | `media` |  | `resource` | Media session control and playback | Draft |
| [NAP-NOTIFY](https://github.com/napplet/naps/pull/11) | `notify` |  | — | Shell-rendered notifications | Draft |
| [NAP-RESOURCE](https://github.com/napplet/naps/pull/13) | `resource` |  | — | Sandboxed resource fetching (https / blossom / nostr / data) | Draft |
| [NAP-TORRENT](naps/NAP-TORRENT.md) | `torrent` |  | `identity` `relay` `resource` | Runtime-mediated NIP-35 torrent indexes and transfer jobs | Draft |
| [NAP-CONFIG](https://github.com/napplet/naps/pull/14) | `config` |  | — | Per-napplet declarative configuration (JSON Schema-driven) | Draft |
| [NAP-UPLOAD](https://github.com/napplet/naps/pull/33) | `upload` |  | `relay` | Shell-mediated file and blob upload (NIP-96, Blossom) | Draft |
| [NAP-VALUE](https://github.com/napplet/naps/pull/30) | `value` |  | `relay` | Shell-mediated value transfer and zaps | Draft |
| [NAP-OUTBOX](https://github.com/napplet/naps/pull/32) | `outbox` |  | `relay` | Outbox-aware relay routing and queries | Draft |
| [NAP-CVM](https://github.com/napplet/naps/pull/31) | `cvm` |  | `value` | Native ContextVM / MCP-over-Nostr bridge | Draft |
| [NAP-LINK](https://github.com/napplet/naps/pull/53) | `link` |  | — | Shell-mediated external link opening | Draft |
| [NAP-POW](https://github.com/napplet/naps/pull/39) | `pow` |  | `identity` `relay` `outbox` | NIP-13 proof-of-work miner (mine, mine-and-publish, queue, progress, hashrate) | Draft |
| *[NAP-CLASS](https://github.com/napplet/naps/pull/16)* | *`class`* |  | *—* | *Napplet class authority (sub-track root)* | *Deferred* |
| *[NAP-CONNECT](https://github.com/napplet/naps/pull/19)* | *`connect`* |  | *—* | *User-gated direct network access* | *Deferred* |

### Conventions — message shapes (*what napplets say to each other*)

Cross-napplet message shapes are unnumbered conventions. They are named as
`napplet:<archetype>/<intent>[...?params]`, such as `napplet:profile/open`,
`napplet:note/open`, `napplet:dm/open`, `napplet:feed/open`, or
`napplet:stream/switch`. Napplets converge by using the same topic names and
payload fields. The registry does not assign sequence numbers for them.

In a convention exchange the **producer** is the napplet that emits a topic and
the **consumer** is the napplet that receives and acts on it, reached directly or,
by archetype, via the runtime.

A convention that shapes an archetype open payload is advertised on the
archetype tag and by `intent.available()` handler metadata. No registry edit is
required before two napplets can try a compatible payload.

### NAAT — archetypes (*what kind of napplet this is*)

A NAAT is neither an interface nor a payload convention, just a name and a boundary.
Archetypes are rows in the [ARCHETYPES.md](ARCHETYPES.md) registry, each linking
to a thin file under [`naat/`](naat/). A napplet declares the roles it fulfills
with a `["archetype", "<slug>", "<convention>"]` manifest tag, and napplets invoke
each other by role through [NAP-INTENT](naps/NAP-INTENT.md). A napplet with no
archetype tag is fully valid — it simply isn't invokable by role.

## Boundary rule

A NAP is **runtime-provided** AND defines an **API surface**. A convention is
**napplet-agreed** AND defines **message semantics**. An archetype (NAAT) is a
**canonical role name** with a **boundary**, owning neither an API nor a payload.
Only runtime-provided API surfaces are NAPs.

## Governance

NIP-style informal process:

- Fork this repo, add a markdown file under [`naps/`](naps/) following the
  interface template, open a PR. Every NAP spec is a named runtime capability
  contract. Templates and registries (`README.md`, `ARCHETYPES.md`) stay at the
  repo root.
- Community discusses via PR comments.
- Maintainer (dskvr) merges when the spec makes sense and has at least one
  implementation.
- No formal stages, review committees, or voting.
- NAP-WORD names and NAAT slugs are first-come-first-served but must be approved
  by the maintainer.

## Templates

- Interface proposals: [NAP-WORD-TEMPLATE.md](NAP-WORD-TEMPLATE.md)
- Convention notes: [CONVENTION-TEMPLATE.md](CONVENTION-TEMPLATE.md)
- Archetype proposals: [naat/TEMPLATE.md](naat/TEMPLATE.md) + a row in [ARCHETYPES.md](ARCHETYPES.md)

## References

- Web projection: [projections/web.md](projections/web.md) — [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) (living, upstream document)
- Napplet manifest / identity: [NIP-5A](https://github.com/nostr-protocol/nips/blob/master/5A.md)
- Archetype registry: [ARCHETYPES.md](ARCHETYPES.md)
