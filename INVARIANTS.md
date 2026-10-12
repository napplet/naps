# NAP invariants

**Interoperability risk:** the current NAP-INTENT text gives conflicting fragment
rules and incomplete recommendation schemas. See [Unresolved checks](#unresolved-checks).
Supporting documents and tests cannot settle a conflict in the owning contract.

This is a review index for **interoperability through one clear seam**. The
[glossary](README.md#glossary), owning NAPs, and projections remain authoritative.
Each invariant names its source and an observable way to detect a violation.
Check a requirement against the revision being reviewed; this index adds no API.

## Ownership

| ID | Invariant | Authority | Review or verification |
|----|-----------|-----------|------------------------|
| I-01 | The napplet designs, the shell mediates, the runtime executes, and the user composes. | [AGENTS goal](AGENTS.md#goal--never-compromise) | For each proposed behavior, name the actor that requests it, the actor that authorizes it, and the actor that executes it. Flag any transfer of user choice or runtime authority. |
| I-02 | A NAP owns a runtime-provided API; a convention owns napplet-agreed message semantics; a NAAT names a role and boundary. | [Boundary rule](README.md#boundary-rule), [archetype schema](ARCHETYPES.md#entry-schema) | Classify the change before naming it. A payload convention needs no new NAP domain. An archetype file stays within its fixed schema. |
| I-03 | A projection maps contracts to a host without changing their meaning. | [Projections](README.md#projections), [web projection](projections/web.md) | Describe the behavior without browser objects, then check that the host binding preserves operations, errors, and trust boundaries. |
| I-04 | Dependencies are explicit and shared wire types have one defining owner. | [Dependencies](AGENTS.md#dependencies--declare-dont-imply) | Compare each imported field or called capability with `Depends:` and the README Deps column. Reject independently redefined shared types and mutual amendments. |
| I-05 | Operations and schemas are language-neutral contracts. | [Interface schema format](AGENTS.md#interface-schema-format), [operation naming](AGENTS.md#operation-naming) | Review tables independently of an SDK. Check that each `domain.action` action adds meaning beyond the domain name. |

## Authority and identity

| ID | Invariant | Authority | Review or verification |
|----|-----------|-----------|------------------------|
| I-06 | Manifest capability and role advertisements are declarations, not grants. Actual availability comes from the projection. | [Discovery and mediation](README.md#how-it-works), [manifest catalog contract](naps/NAP-INTENT.md#manifest-catalog-contract) | Advertise a policy-denied domain. Confirm it remains unexposed and cannot be exercised through another domain or a routing advertisement. |
| I-07 | A `shell` domain or readiness exchange is not a prerequisite for other exposed domains. | [NAP-SHELL behavior](naps/NAP-SHELL.md#shell-behavior) | Omit `shell`, or delay its environment delivery, while exercising another exposed domain. Its availability and policy enforcement still work. |
| I-08 | Runtime-attested provenance comes from authenticated endpoints bound to verified manifests and artifacts. User identity is distinct from napplet identity. | [Web identity and trust](projections/web.md#identity--trust), [INC schemas](naps/NAP-INC.md#schemas), [identity security](naps/NAP-IDENTITY.md#security-considerations) | Supply a forged sender, use two publishers with the same `d` value, and change the connected user. Confirm provenance is not caller-controlled and endpoint identity does not become user identity. |
| I-09 | Endpoint identifiers, catalog identifiers, artifact identity, and user pubkeys have distinct purposes. | [INC schemas](naps/NAP-INC.md#schemas), [catalog identifiers](naps/NAP-INTENT.md#catalog-identifiers) | Check simultaneous instances, root and snapshot manifests, and a replaced endpoint. Do not substitute a bare `d` value or catalog identifier for a live INC peer. |

## Routing, choice, and lifecycle

| ID | Invariant | Authority | Review or verification |
|----|-----------|-----------|------------------------|
| I-10 | Convention identity is the queryless, fragment-free path. Payload transposition precedes exact routing. | [Convention invariant](AGENTS.md#convention-uri-invariant), [INC transposition](naps/NAP-INC.md#convention-uri-transposition) | Preserve text values and literal `+`; reject malformed encoding, repeated decoded names, and query plus explicit payload. Check near-match topics do not receive delivery. |
| I-11 | Handler recommendations remain subordinate to authorized selection, applicable user defaults, and shell policy. | [Recommended resolution](naps/NAP-INTENT.md#recommended-handler-resolution) | Check precedence and policy denial separately from URI parsing. Discovery does not itself authorize installation or execution. Fragment and request-shape checks remain unresolved below. |
| I-12 | Intent acceptance transfers delivery responsibility to the runtime, independently of source lifetime. | [Intent description](naps/NAP-INTENT.md#description), [runtime behavior](naps/NAP-INTENT.md#runtime-behavior) | Observe the acceptance result before source closure, then destroy the source and start the target. Acceptance is not proof of target handling or persistence across runtime restart. |
| I-13 | Policy decisions and data stay scoped to the intended napplet or authorized peers. | [Theme behavior](naps/NAP-THEME.md#shell-behavior), [intent security](naps/NAP-INTENT.md#security-considerations), [INC management](naps/NAP-INC.md#channel-management) | Ensure an unexposed theme domain receives no updates; unrelated napplets receive no intent payload; replacement endpoints inherit no old channel. |

## Source discipline

| ID | Invariant | Authority | Review or verification |
|----|-----------|-----------|------------------------|
| I-14 | Deferred work has no live surface. Living external specifications are linked to their upstream source. | [Two leaks](AGENTS.md#two-leaks-to-catch-on-sight) | Search active specs, projections, and README for a deferred domain: only its italic registry row should remain. Check that no external specification body has been mirrored locally. |
| I-15 | Adjacent documentation and semantic history agree with the changed contract. | [Doc synchronization](AGENTS.md#keep-adjacent-docs-in-sync--in-the-same-commit), [changelog discipline](AGENTS.md#changelog-discipline) | Verify registry status, dependencies, owner/importer references, and the spec/PR changelog against the final diff. |

## Unresolved checks

Baseline: `napplet/naps` commit `e5308ac92cfd49e4fb026812c60886571515e219`.

| Check | Conflicting evidence | Treatment |
|-------|----------------------|-----------|
| Intent URI handler fragment | [Convention URI normalization](naps/NAP-INTENT.md#convention-uri-normalization) rejects fragments; [web handler binding](projections/web.md#intent-handler-fragment) and recommended resolution describe them. | Report blocked; do not select one rule in a test adapter. |
| Intent recommendation request | [Schemas](naps/NAP-INTENT.md#schemas) repeat `IntentInvokeOptions` with different selection shapes and do not carry the recommendation consistently through `IntentRequest`. | Report blocked for recommendation serialization and explicit handler representation. |

[Reconciliation PR #106](https://github.com/napplet/naps/pull/106) is the existing
review location. Re-evaluate these entries after its contract changes land.
Passing unaffected routing or lifecycle checks does not resolve either conflict.

## Applying this index

For a proposal, list affected invariant IDs, the owning requirements, and the
positive and negative evidence available. For a new requirement, add its proof
to the owning spec or conformance tooling before treating a supporting index as
evidence of implementation. Record absent tests and unresolved text explicitly.

The web binding remains [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303);
its details belong to the projection. Pin its revision in implementation reports.
