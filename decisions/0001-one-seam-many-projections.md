# 0001 — One seam across host projections

**Recorded:** 2026-10-12
**Status:** existing direction documented
**Source revision:** naps `e5308ac92cfd49e4fb026812c60886571515e219`
**Review:** retrospective record of the sources below; no new contract proposed

## Context

NAP defines the capability seam between a napplet and its runtime. The current
host is web, but browser message carriers and object names are projection
details. Binding an operation's meaning to those details would make another host
invent a different contract for the same domain.

## Decision

Keep operations, schemas, errors, and trust boundaries in the owning NAP. Keep
the mapping to a host in a projection. A NAP owns a runtime-provided API; a
convention owns napplet-agreed message semantics; a NAAT names a role and boundary.
The [glossary](../README.md#glossary) and
[boundary rule](../README.md#boundary-rule) define these distinctions.

The napplet designs, the shell mediates, the runtime executes, and the user
composes. Projection mechanisms implement that division without transferring
ownership. The [web projection](../projections/web.md) links its living upstream
contract, [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303).

## Alternatives inconsistent with the current boundary

| Alternative | Convenience | Consequence |
|-------------|-------------|------------|
| Put browser objects and iframe assumptions in every NAP | Reads like the first implementation | Host mechanics become part of the capability contract; another host must reinterpret them. |
| Assign NAP domains to ordinary payload conventions | One registry for every message | Napplet-agreed semantics acquire the appearance of runtime authority. |
| Let archetypes accumulate operations and payload schemas | Everything for a role in one file | Role boundaries become competing interface and convention definitions. |

These comparisons follow from the current ownership rules. They do not assert
that the original maintainers formally reviewed each alternative.

## Consequences

Specs use language-neutral tables. SDKs may expose host idioms as projection
guidance. Shared types have one owner, with declared dependencies for importers.
External living specifications remain upstream. Supporting documents point to
the contract rather than defining another version of it.

## Verification and reconsideration

Describe the proposed operation without browser-specific transport, then map it
to the web host. Check that changing the carrier preserves its observable
results and trust boundary. Native and WASM are possible projections in the
registry, not established interoperating implementations. No such execution
evidence is claimed here.

Revisit a host mapping when a real implementation reveals a missing binding
rule. Change the owning contract only when its semantics need to change across
hosts, with explicit compatibility evidence.

## References

- [AGENTS boundaries](../AGENTS.md#boundaries--exacting)
- [README projections](../README.md#projections)
- [Interface schema rules](../AGENTS.md#interface-schema-format)
- [Archetype entry schema](../ARCHETYPES.md#entry-schema)
