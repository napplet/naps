# Implement one NAP domain

**Known contract conflict:** NAP-INTENT's current fragment normalization and
recommendation request schemas disagree. If implementing those paths, start with
[reconciliation PR #106](https://github.com/napplet/naps/pull/106) and record which
contract revision you target. This guide does not resolve the conflict.

This guide takes an implementer from one chosen domain to a reviewable result.
Read the [glossary](README.md#glossary), the owning NAP, and the host projection
first. The specifications define requirements; this guide explains a workflow.
Examples are wire illustrations, not a replacement SDK or manifest format.

## 1. Choose a surface with an owner

Choose a domain in the [registry](README.md#nap-word--interfaces-what-the-runtime-offers)
and record its spec revision and status. Read its operations, schemas, error
rules, dependencies, lifecycle, and security requirements together. The registry
includes draft proposals; a linked PR does not mean a contract is stabilized or
present in every runtime.

For a compact first surface, consider the existing
[NAP-THEME](naps/NAP-THEME.md) contract. It has a request/result operation and an
automatic push, with no declared domain dependencies. Implement its requirements
as a whole, including minimal colors, optional media behavior, and scoped pushes.

## 2. Identify your responsibilities

| Actor | Responsibility |
|-------|----------------|
| Napplet | Probe availability, request operations, consume results, validate received data, and handle absence or errors. |
| Projection/binding | Expose the host's domain surface and carry operations faithfully, including correlation and domain-specific listener behavior. |
| Shell | Mediate grants, resource access, handler selection, and user preferences. |
| Runtime | Authenticate the endpoint, execute the owning domain, enforce policy, and deliver its results and pushes. |

These responsibilities can share a process. Their authority remains distinct.
User identity, verified napplet identity, live endpoint identifiers, and catalog
identifiers cannot be substituted for one another.

## 3. Bind to the host and expose availability

For the [web projection](projections/web.md), follow the living
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) source for manifest and
artifact verification, sandboxing, authenticated browser sources, and namespace
injection. Record the upstream revision you implement. Do not reproduce that
specification in this repository.

Expose `window.napplet.theme` only where runtime policy permits `theme`. Domain
presence is the web availability signal. The napplet checks it before use. A
`shell` domain and its readiness exchange are optional and do not gate other
exposed domains.

Manifests refer to bare domains. Required and optional capability declarations
inform compatibility; they do not grant authority. A theme update goes to a
napplet with an exposed `theme` domain even without a theme advertisement, and
never to a napplet without the domain.

## 4. Implement the operation and push

From [NAP-THEME wire protocol](naps/NAP-THEME.md#wire-protocol):

```json
{"type":"theme.get","id":"theme-1"}
```

For this request, return the same correlation ID and the minimal valid theme:

```json
{"type":"theme.get.result","id":"theme-1","theme":{"colors":{"background":"#000000","text":"#ffffff","primary":"#3366ff"}}}
```

When the active theme changes, push the new theme to every endpoint with the
domain exposed. A push has no request correlation ID:

```json
{"type":"theme.changed","theme":{"colors":{"background":"#ffffff","text":"#000000","primary":"#6633ff"}}}
```

The binding's `onChanged` registers a local handler. Closing its handle removes
that handler. There is no theme wire subscribe/unsubscribe operation. Follow
the owning spec's error behavior and optional fields, including its restricted
environment policy; do not invent another domain's result shape.

## 5. Keep resources and payload semantics with their owners

Font and background URLs do not grant direct network access. Follow the
[theme security guidance](naps/NAP-THEME.md#security-considerations), runtime
resource policy, and local fallback when optional media cannot be used. Do not
turn optional media into an undeclared prerequisite for a basic color theme.

If implementing a domain with dependencies, classify them as wire, capability,
or layering and required or optional. Import shared records by reference from
their one owner. See [dependency rules](AGENTS.md#dependencies--declare-dont-imply).

For inter-napplet behavior, discover stable conventions rather than teaching the
router to infer payload meaning. Follow the owning INC or intent contract for
URI transposition and exact routing. A NAAT names a role; it does not own the
payload or grant a capability.

## 6. Verify the observable boundary

For the theme example, exercise:

| Scenario | Expected observation from the owning contract |
|----------|------------------------------------------------|
| `theme` exposed, `shell` absent | The theme operation works without shell readiness. |
| Theme query | Result carries the request ID and all required color fields. |
| Minimal theme | Missing optional fonts, background, and title remain valid. |
| Active theme changes | Every exposed endpoint receives the push; unexposed endpoints receive none. |
| Listener closes | The closed local handler stops receiving updates; no wire unsubscribe is introduced. |
| Remote optional media unavailable | The napplet uses colors and local fonts without assuming network authority. |

Reuse [the existing conformance engine](https://github.com/napplet/web/tree/main/packages/conformance)
and [browser CLI](https://github.com/napplet/web/tree/main/packages/conformance-cli)
for their supported napplet checks. Runtime behavior needs runtime tests; a
schema validator cannot prove delivery policy. Use the owning project's runner
and record commands, revisions, denied-domain cases, failures, and gaps.

Exchange messages with an independently written counterpart. Identify both
implementations and their contract revisions. A single SDK with its reference
mock establishes useful integration evidence, not independent interoperability.

## 7. Submit a result people can review

Include the implementation link and immutable revision, selected spec and
projection revisions, supported operations, test evidence, known gaps, and
policy choices. Keep the scope on one domain. If the implementation reveals an
ambiguous requirement, surface it in the owning spec review before coding a
private interpretation into the seam.

For specification changes, follow [repository isolation](AGENTS.md#isolation),
[doc synchronization](AGENTS.md#keep-adjacent-docs-in-sync--in-the-same-commit),
and [semantic history](AGENTS.md#changelog-discipline). Draft proposals remain
draft until the maintainer accepts the implemented, defended, stabilized contract
under the repository's informal governance.
