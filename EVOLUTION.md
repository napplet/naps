# Evolving NAP

This is review guidance for changing contracts while preserving one clear seam.
Requirements remain in the owning NAP and projection. It adds no global version
field, mandatory domain set, negotiation API, or formal acceptance stage.

The [registry](README.md#governance) supplies the informal process: implement,
discuss, defend, and stabilize the contract. The maintainer decides when to merge.
Compatibility evidence helps that decision; a checklist is not a substitute for
an implementation.

## Classify the change

| Change | Owning surface | Compatibility question |
|--------|----------------|------------------------|
| New runtime capability | One NAP domain | Can existing napplets continue with their exposed domains and optional behavior? |
| New payload meaning between napplets | Convention | Can callers discover an exact supported convention without teaching the router its payload semantics? |
| New role | NAAT and archetype registry | Is its boundary distinct, and do advertisements remain separate from grants? |
| Host-specific delivery or availability | Projection | Does the binding preserve the same domain contract and trust boundary? |
| New shared wire record | One owning domain plus referencing importers | Is its owner explicit, and can old importers handle the change? |
| Clarification of existing prose | Owning specification | Does it change any compliant observable behavior or settle an ambiguity implementations interpreted differently? |

Treat a clarification as semantic when it changes an implementation's behavior.
If ownership cannot be placed cleanly, surface the conflict before extending it.

## Assess compatibility in both directions

Use exact old and proposed spec revisions and independent implementation
revisions. The table is an assessment format, not an assertion that all versions
must interoperate forever.

| Caller | Runtime/binding | Target handler, when applicable | Evidence to record |
|--------|-----------------|----------------------------------|--------------------|
| Old | Old | Old | Existing behavior and regression baseline |
| Old | New | Old | Preserved operation, fields, errors, authorization, and delivery |
| New | Old | Old | Defined detection, allowed fallback, or explicit unavailability |
| New | New | Old | Convention compatibility and target payload validation |
| Old | New | New | Compatibility of existing requests with the updated handler |
| New | New | New | Intended behavior and new negative cases |

Record an unavailable capability as such. Do not label silent loss of authority
checks or changed payload meaning a graceful fallback. Runtime-assigned catalog
and endpoint identifiers remain local to their documented scope.

## Changes that require careful evidence

| Proposal | Risk to evaluate |
|----------|------------------|
| Add an optional field | Older parsers may reject, reinterpret, or accidentally grant authority from it. Optional syntax alone does not prove compatibility. |
| Add a required field or change a type | Existing callers or runtimes may no longer produce or accept valid messages. |
| Rename an operation, domain, or enum value | Availability probes, routing, SDKs, and existing wire traffic can diverge. |
| Change a default, error, or result meaning | Unchanged message shapes can conceal changed semantics. |
| Add a required dependency | Previously usable deployments may no longer satisfy the complete required set. |
| Change identity, routing, or handler selection | Two implementations can select different principals or targets from the same input. |
| Change buffering or lifecycle | Requests accepted under one revision can lose delivery or require a live source under another. |
| Promote deferred work | Dormant concepts can leak into existing surfaces before a concrete contract is reviewed. |

The authority for handling unknown message types remains the
[upstream web binding](https://github.com/nostr-protocol/nips/pull/2303).
Unknown fields and semantic feature support are assessed against their owning
contracts; do not extrapolate one unknown-type rule into every parser.

## Describe an intentional break

If the evidence shows an intentional break, put that fact first in the PR.
Name the affected deployed behavior, why an extension cannot preserve it, and
the old and new peers that will disagree. Describe a migration sequence with:

1. An observable way to identify the supported contract, defined by its owner.
2. The order in which callers, bindings, runtimes, and handlers can update.
3. The period and limits of any compatibility behavior.
4. Verification against actual old and new implementations.
5. A rollback or retirement plan, including affected state and retained work.

Domain presence signals availability, not semantic version support. If a domain
needs feature checks, diagnostics, or negotiation, design them in that domain's
contract. Do not make `shell` a prerequisite for every domain.

## Keep sources aligned

Follow [adjacent doc synchronization](AGENTS.md#keep-adjacent-docs-in-sync--in-the-same-commit)
and [changelog discipline](AGENTS.md#changelog-discipline). Update declared
dependencies, owner/importer references, registry pointers, and the living PR
body when their semantics change. Supporting indexes link requirements instead
of defining a competing contract. Upstream changes stay upstream.

Use independent branches based on `origin/master` for separate concerns. When
related PRs have dependencies, name them and their merge order in the bodies.
After a squash merge, refresh against the resulting `master` commit; the PR's
original commit hash is not a reliable ancestry marker for later branches.

## Current reconciliation example

At naps `e5308ac92cfd49e4fb026812c60886571515e219`,
[intent normalization](naps/NAP-INTENT.md#convention-uri-normalization),
[recommendation schemas](naps/NAP-INTENT.md#schemas), and the
[web handler binding](projections/web.md#intent-handler-fragment) disagree.
[PR #106](https://github.com/napplet/naps/pull/106) is a reconciliation proposal.
Record conflicting cases as blocked until the contract is reconciled, then check
both interpretations that implementations may already have adopted. Do not
silently redefine the protocol in a conformance fixture or implementation guide.

For background on architectural compatibility and extension cost, see
[RFC 6709](https://www.rfc-editor.org/rfc/rfc6709.html). Its guidance informs
review; it does not add NAP requirements.
