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
- [Interfaces and conventions](#interfaces-and-conventions)
- [NAAT deprecation](#naat-deprecation)
- [Boundary rule](#boundary-rule)
- [Governance](#governance)
- [Templates](#templates)
- [Authoring skill](#authoring-skill)
- [License](#license)
- [References](#references)

## Glossary

| Term | Meaning |
|------|---------|
| **Seam** | The boundary between a napplet and its runtime — what's offered, and how it's asked for. Transport-agnostic. |
| **Napplet** | A Nostr applet: a small, single-purpose app. Described by a [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) manifest. |
| **Runtime** (shell) | The host that composes napplets and provides their capabilities. |
| **NAP** | One capability contract in the seam — operations, message schema, error model, and trust boundary. Never the delivery mechanism. |
| **Domain** | A capability's short name (`relay`, `intent`); how a NAP is referenced and discovered. |
| **Projection** (binding) | A mapping of the seam onto one concrete host (web, native, WASM, …). |
| **NAP-WORD** | An interface spec — an API the runtime offers. One canonical spec per name. |
| **Convention** | An unnumbered message shape napplets agree to use. Its stable identity is `napplet:<archetype>/<intent>`; a developer-facing invocation MAY append `?params`, which the runtime binding transposes into payload data. Not a NAP. |
| **Archetype** | A role name used in convention URIs and manifest advertisements. No registry entry or maintainer approval is required. |

## What is a napplet?

A napplet is a Nostr applet — a small app that does one thing well. A chat
widget, a feed viewer, a profile editor, and a relay manager are four napplets,
not one app with four tabs. **The runtime composes napplets; napplets do not
compose themselves.**

A napplet is described and distributed as a
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) manifest: a signed
snapshot, root, or named napplet event describing a
self-contained artifact and its advertised capabilities. The runtime verifies
the artifact against the manifest before running it. See the upstream manifest
and identity rules; a `d` tag alone is not a publisher-scoped identity.

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
Manifest what a napplet IS / how it's described    artifact, identity   (substrate)
  NAP    what a runtime offers a napplet           the capability seam  (this repo)
   └─ projection: web, native, WASM, …             same contracts, different host
```

[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) defines the web napplet manifest.
A NAP defines a capability the napplet can ask a runtime for. A *projection*
implements that seam for a concrete host.

## Projections

A **projection** maps the seam onto a host environment: where napplets run, the
message carrier, identity binding, and how each domain is surfaced. The contracts
are transport-neutral by design — a domain named `relay` in this registry is the
same contract everywhere; only the host idiom changes.

| Projection | Status | Spec |
|------------|--------|------|
| **Web** — iframes + `postMessage`, injected domains and URI-to-payload binding on `window.napplet.*` | In use | [projections/web.md](projections/web.md) ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)) |
| Native (OS process + IPC/FFI) | Possible | — |
| WASM (host imports) | Possible | — |

The web projection is the only implementation today, and the specs are written
against it — but the contracts are shaped so they need not stay web-only.

## How it works

The mechanics below live at the seam and are described projection-neutrally; the
[web projection](projections/web.md) shows how each is realized in the browser.

**Discovery.** A napplet checks domain availability through its projection.
In the [web projection](projections/web.md), the presence of
`window.napplet.relay` means the runtime exposes `relay`. Discovery requires no
`shell` domain or handshake.

A [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) manifest declares required domains
with `R` tags and optional integrations with `O` tags. These are declarations,
not grants. The runtime checks the complete required set before loading; missing
optional domains alone do not prevent loading.

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
The runtime binds each endpoint to the verified manifest and artifact identity,
not to identity claims from the napplet or an untrusted host.

## Interfaces and conventions

This repository governs runtime capability contracts. Cross-napplet message
shapes are conventions. They are not assigned NAP numbers.

### NAP-WORD — interfaces (*what the runtime offers*)

Named by a single uppercase word, one canonical spec per name. Defines a
shell-provided API contract — a capability domain a napplet can call. Discovery:
the projection's availability signal for that domain.

Every NAP-WORD is **optional** — a runtime offers it or it doesn't, and a napplet
checks before using it. In the web projection, domain objects are injected before
napplet scripts run. NAP-SHELL supplies its own shell operations when present;
it is not a prerequisite for domain discovery.

The **Deps** column lists the domains a NAP rests on — declared in each spec's
`Depends:` block, always by domain (never a spec id).

| NAP ID | Domain | req. | Deps | Description | Status |
|--------|--------|------|------|-------------|--------|
| [NAP-SHELL](https://github.com/napplet/naps/pull/101) | `shell` |  | — | Optional shell environment and readiness notifications | Draft |
| [NAP-INTENT](https://github.com/napplet/naps/pull/91) | `intent` |  | — | Accept lifecycle-independent convention delivery from manifest intent advertisements | Draft |
| [NAP-INC](https://github.com/napplet/naps/pull/104) | `inc` |  | — | Inter-napplet communication with authenticated endpoint identifiers | Draft |
| [NAP-THEME](https://github.com/napplet/naps/pull/103) | `theme` |  | — | Shell-provided themes and updates for exposed domains | Draft |
| [NAP-RELAY](https://github.com/napplet/naps/pull/2) | `relay` |  | `resource` | Relay proxy (subscribe, publish, query, publishEncrypted) | Draft |
| [NAP-IDENTITY](https://github.com/napplet/naps/pull/102) | `identity` |  | `resource` | Read-only user identity queries with verified napplet endpoint binding | Draft |
| [NAP-STORAGE](https://github.com/napplet/naps/pull/3) | `storage` |  | — | Scoped key-value storage | Draft |
| [NAP-KEYS](https://github.com/napplet/naps/pull/9) | `keys` |  | — | Keyboard forwarding and action keybindings | Draft |
| [NAP-MEDIA](https://github.com/napplet/naps/pull/10) | `media` |  | `resource` | Media session control and playback | Draft |
| [NAP-NOTIFY](https://github.com/napplet/naps/pull/11) | `notify` |  | — | Shell-rendered notifications | Draft |
| [NAP-RESOURCE](https://github.com/napplet/naps/pull/13) | `resource` |  | — | Sandboxed resource fetching (https / blossom / nostr / data) | Draft |
| [NAP-CONFIG](https://github.com/napplet/naps/pull/14) | `config` |  | — | Per-napplet declarative configuration (JSON Schema-driven) | Draft |
| [NAP-UPLOAD](https://github.com/napplet/naps/pull/33) | `upload` |  | `relay` | Shell-mediated file and blob upload (NIP-96, Blossom) | Draft |
| [NAP-VALUE](https://github.com/napplet/naps/pull/30) | `value` |  | `relay` | Shell-mediated value transfer and zaps | Draft |
| [NAP-OUTBOX](https://github.com/napplet/naps/pull/32) | `outbox` |  | `relay` | Outbox-aware relay routing and queries | Draft |
| [NAP-CVM](https://github.com/napplet/naps/pull/31) | `cvm` |  | `value` | Native ContextVM / MCP-over-Nostr bridge | Draft |
| [NAP-LINK](https://github.com/napplet/naps/pull/53) | `link` |  | — | Shell-mediated external link opening | Draft |
| [NAP-COMMON](https://github.com/napplet/naps/pull/67) | `common` |  | `identity` `relay` `resource` | NIP-19 helpers, profile lookup, follows, and common social actions (follow, unfollow, react, report) | Draft |
| [NAP-POW](https://github.com/napplet/naps/pull/39) | `pow` |  | `identity` `relay` `outbox` | NIP-13 proof-of-work miner (mine, mine-and-publish, queue, progress, hashrate) | Draft |
| [NAP-DISPLAY](https://github.com/napplet/naps/pull/97) | `display` |  | — | Runtime-controlled pixel displays (list, push) | Draft |
| *[NAP-CLASS](https://github.com/napplet/naps/pull/16)* | *`class`* |  | *—* | *Napplet class authority (sub-track root)* | *Deferred* |
| *[NAP-CONNECT](https://github.com/napplet/naps/pull/19)* | *`connect`* |  | *—* | *User-gated direct network access* | *Deferred* |

### Conventions — message shapes (*what napplets say to each other*)

Cross-napplet message shapes are unnumbered conventions. Their stable identities
are named as `napplet:<archetype>/<intent>`, such as `napplet:profile/open`,
`napplet:note/open`, `napplet:dm/open`, `napplet:feed/open`, or
`napplet:stream/switch`. The registry does not assign sequence numbers for them.

A developer invokes a convention as
`napplet:<archetype>/<intent>[...?params]`. The query is shallow payload sugar,
not part of the convention identity. The runtime-provided binding transposes
query parameters into text payload fields before routing or handler resolution.
Structured or non-text data uses the explicit payload. Routers match only the
resulting stable identity, by exact equality.

For [NAP-INTENT](naps/NAP-INTENT.md) URI invocations only, a caller MAY append
`#<naddr>` after the path or query to recommend a handler:

```text
napplet:profile/open?pubkey=abc123…#<naddr>
```

`<naddr>` stands for one bare `naddr1…` identifying a kind `35129` napplet under
[NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md). The binding
extracts its coordinate and relay hints into `handlerHint`; the fragment never
becomes convention identity or payload. An applicable user default wins. The
shell MAY discover an uninstalled recommendation and offer installation under
its policy. An unusable recommendation falls back to normal handler resolution.
Other convention-URI operations reject fragments.

In a convention exchange the **producer** is the napplet that invokes the
convention and the **consumer** is the napplet that receives and acts on it,
reached directly or, by archetype, via the runtime.

A convention that shapes an archetype open payload is advertised by its stable,
queryless identity in an `i` tag and in `intent.available()` handler
metadata. No registry edit is required before two napplets can try a compatible
payload.

## NAAT deprecation

**NAAT (Napplet Archetypes) is deprecated.** The unused role registry, role
definitions, and proposal process are retired. No replacement is defined;
role standardization can be reconsidered when a concrete need arises.

Convention URIs and [NAP-INTENT](naps/NAP-INTENT.md) manifest-based dispatch remain
in use. Their archetype names do not depend on NAAT registration.

## Boundary rule

A NAP is **runtime-provided** AND defines an **API surface**. A convention is
**napplet-agreed** AND defines **message semantics**.
Only runtime-provided API surfaces are NAPs.

## Governance

NIP-style informal process:

- Fork this repo, add a markdown file under [`naps/`](naps/) following the
  interface template, open a PR. Every NAP spec is a named runtime capability
  contract. Templates and the NAP registry (`README.md`) stay at the
  repo root.
- Community discusses via PR comments.
- Maintainers merge when the spec has been implemented, defended and has stabilized.
- No formal stages, review committees, or voting.
- NAP-WORD names are first-come-first-served but must be approved
  by the maintainer.

## Templates

- Interface proposals: [NAP-WORD-TEMPLATE.md](NAP-WORD-TEMPLATE.md)
- Convention notes: [CONVENTION-TEMPLATE.md](CONVENTION-TEMPLATE.md)

## Authoring skill

Install the [NAP authoring skill](skills/nap-authoring/SKILL.md) for an agent that
writes or reviews NAP specifications:

```sh
npx skills add napplet/naps
```

The skill covers format, philosophy, ownership boundaries, and contract
hardening. It reads the current registry and templates rather than bundling a
second copy of the protocol. It is for specification work; napplet application
development uses the implementation toolchain's skills.

## License

The contents of this repository, including specifications, registries, templates,
documentation, and examples, are dedicated to the public domain under
[CC0 1.0 Universal](LICENSE) (`CC0-1.0`). You may copy, modify, and redistribute
them, including for commercial use, without attribution or permission.

By submitting a contribution, you agree to apply CC0-1.0 to it.
Separate implementations choose their own licenses. Linked external documents
remain subject to their own terms. CC0 does not grant patent or trademark rights.

## References

- Web projection: [projections/web.md](projections/web.md) — [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) (living, upstream document)
- Napplet manifest / identity: [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)
