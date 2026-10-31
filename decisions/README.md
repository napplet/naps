# NAP decision records

These records explain consequential architectural choices and their evidence.
They preserve reasoning for future collaborators. The owning specifications,
[glossary](../README.md#glossary), and [repository guidance](../AGENTS.md) remain
authoritative; a record does not create an API or silently change a contract.

## Index

| Record | Subject | Status |
|--------|---------|--------|
| [0001](0001-one-seam-many-projections.md) | One capability seam across host projections | Existing direction documented |
| [0002](0002-availability-is-not-authority.md) | Projection availability and runtime authority | Existing direction documented |
| [0003](0003-stable-convention-identity.md) | Stable convention identity and payload ownership | Existing direction documented; intent recommendation details unresolved |

The initial records were written retrospectively on 2026-10-12 from naps commit
`e5308ac92cfd49e4fb026812c60886571515e219`. Their alternatives are comparisons
against the current boundaries, not claims about discussions absent from the
linked review history.

## When to add a record

Use a record when a choice changes architectural ownership, identity scope,
extension design, compatibility, or a trust assumption that later proposals
might revisit. Routine operation details belong in their owning specs and
changelogs. A decision record adds rationale rather than repeating the wire
contract.

Start with [TEMPLATE.md](TEMPLATE.md). Link the governing requirement and review,
describe considered alternatives, and separate evidence from inference. Record
the consequences and the observations that would justify revisiting the choice.
Do not invent consensus, approvals, implementation results, or dates.

## Review and maintenance

A new architectural proposal is marked proposed and reviewed with its owning
contract. Once the maintainer accepts the decision, record the review link and
the resulting merged revision. These statuses describe the record, not a new
formal protocol stage.

For a changed decision, add a successor record and link the old record as
superseded. Preserve the original reasoning. Fix factual errors explicitly, and
identify source conflicts rather than selecting a private interpretation.
Separate concerns into independent PRs. Supporting records never replace the
specification's semantic changelog.

For the web host, link [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)
as the living upstream authority. Do not mirror its body into a decision record.
