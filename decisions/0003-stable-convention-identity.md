# 0003 — Stable convention identity preserves payload ownership

**Interoperability risk:** the current NAP-INTENT recommendation text conflicts
with its URI normalization and request schemas. This record explains the stable
identity rule; it does not choose a recommendation wire interpretation.

**Recorded:** 2026-10-12
**Status:** existing direction documented; intent recommendation details unresolved
**Source revision:** naps `e5308ac92cfd49e4fb026812c60886571515e219`
**Reviews:** [Convention transposition #90](https://github.com/napplet/naps/pull/90),
[handler recommendations #99](https://github.com/napplet/naps/pull/99),
[reconciliation #106](https://github.com/napplet/naps/pull/106)

## Context

Napplets need a stable message identity for advertisements, subscriptions, and
handler resolution. A convenient invocation may carry per-message query values
or a recommended handler. Those values should not create a different convention
or make a router infer application meaning from payload content.

## Decision

The queryless, fragment-free `napplet:<archetype>/<intent>` path identifies the
convention. The runtime-provided binding transposes invocation query values
before routing, preserving text values and literal `+`. Exact identity matching
keeps producers, consumers, and handlers in agreement.

A convention owns payload semantics. A NAAT names a role. The runtime resolves
eligible handler contracts from verified manifest advertisements, while the
target validates payload content. Recommendations remain separate from identity
and subordinate to user choice and shell policy.

## Alternatives inconsistent with the current boundary

| Alternative | Convenience | Consequence |
|-------------|-------------|------------|
| Advertise or subscribe to full query-bearing URIs | One string for identity and data | Each payload creates another apparent identity; compatible handlers fail to match. |
| Prefix- or wildcard-match convention paths | Broad subscriptions | Different conventions can accidentally share routing and payload interpretation. |
| Coerce query strings to scalar types | Less parsing in the target | Caller and receiver can disagree about the payload before its owner validates it. |
| Infer a handler from payload content | Implicit application dispatch | Routers take ownership of convention semantics and can override advertised compatibility. |

These are consequences of the established contract, not a reconstructed vote
or a claim that every alternative was historically discussed.

## Consequences

Structured and non-text payloads use the explicit payload form. Query plus
explicit payload and invalid query syntax are rejected by the binding. Plain INC
topics remain opaque exact-match strings. The intent acceptance result means
delivery responsibility transferred, not that the handler has processed the
payload.

## Verification and reconsideration

Check encoded names, repeated decoded names, malformed escapes, literal `+`, and
near-match topics. Verify advertisements and subscriptions remain free of query
and fragment data. Test recommendation precedence separately from its currently
unresolved serialization. Record fragment/request-shape cases as blocked until
the owning contract is reconciled.

Revisit convenience syntax only with a precise normalization contract and
evidence from independent bindings. Preserve stable identity and payload ownership
or explicitly review the resulting architectural change.

## References

- [Convention invariant](../AGENTS.md#convention-uri-invariant)
- [INC transposition](../naps/NAP-INC.md#convention-uri-transposition)
- [Intent catalog contract](../naps/NAP-INTENT.md#manifest-catalog-contract)
- [Web handler binding](../projections/web.md#intent-handler-fragment)
- [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)
