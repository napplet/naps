# 0002 — Availability is distinct from authority

**Recorded:** 2026-10-12
**Status:** existing direction documented
**Source revision:** naps `e5308ac92cfd49e4fb026812c60886571515e219`
**Review:** [NAP-SHELL alignment #101](https://github.com/napplet/naps/pull/101)

## Context

A runtime can expose some domains and omit others. A napplet needs to discover
what is exposed without turning discovery into authentication, a capability
grant, or a requirement for a separate bootstrap domain.

## Decision

Each projection defines availability for bare domains. In the web host, the
injected domain object is the signal. [NAP-SHELL](../naps/NAP-SHELL.md) supplies
optional environment and readiness operations; its absence or delayed environment
does not make other exposed domains unusable.

The runtime binds verified napplet identity before servicing domain calls.
Readiness and caller-supplied identity cannot establish or change that identity.
Manifest required/optional declarations, role advertisements, and environment
snapshots describe information; the shell's policy still owns exposure and
authorization.

## Alternatives inconsistent with the current contract

| Alternative | Convenience | Consequence |
|-------------|-------------|------------|
| Require a shell handshake before every capability | A single apparent bootstrap sequence | Adds a prerequisite to unrelated domains and confuses readiness with authority. |
| Treat required declarations as capability grants | Less policy bookkeeping | Untrusted metadata widens access without an authorization decision. |
| Withhold readiness to deny another domain | One control switch | Leaves actual domain enforcement unspecified and can disagree with availability. |

These comparisons explain the current contract. They are not separate newly
adopted policies.

## Consequences

Domain presence does not prove all operations, semantic versions, or compatible
features. Checks beyond availability belong in the owning domain when needed.
User sign-in changes user identity data, not the verified napplet endpoint.
Manifests reference bare domains rather than spec labels.

## Verification and reconsideration

Exercise an exposed domain without `shell`. Delay environment delivery and repeat
readiness. Confirm other operations remain usable, no extra session is created,
and the environment stays scoped to the authenticated napplet. Supply policy-
denied declarations and forged identity claims; neither should widen authority.
This record names checks, not a completed implementation report.

If a domain needs its own readiness or compatibility signal, specify that within
the domain. Revisit the projection mapping when another host supplies concrete
evidence that its availability mechanism needs clarification.

## References

- [NAP-SHELL behavior](../naps/NAP-SHELL.md#shell-behavior)
- [NAP-SHELL security](../naps/NAP-SHELL.md#security-considerations)
- [Web domain surfacing](../projections/web.md#domain-surfacing)
- [NAP-IDENTITY security](../naps/NAP-IDENTITY.md#security-considerations)
- [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)
