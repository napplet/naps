# NAP implementation evidence

This is an evidence index, not certification or a second NAP registry. The
[README](README.md#the-two-axes) owns domain and archetype registration. Evidence
here concerns concrete implementation revisions, tested scopes, and exchanges.

Source inventory checked on 2026-10-12 against naps
`e5308ac92cfd49e4fb026812c60886571515e219`. No runtime conformance or independent
interoperability run was performed for this initial inventory. An implementation
link is not a passing result.

## Evidence levels

| Label | What the entry establishes |
|-------|----------------------------|
| source inspected | The named package or test path exists at the linked revision. |
| claimed support | Its maintainers document the scope; no execution evidence is attached here. |
| tested | An attached report shows observed behavior at stated implementation and contract revisions. |
| interoperated | A report identifies independent counterparts and an observed compatible exchange. |

These describe evidence, not formal protocol stages. Preserve failed, blocked,
and missing results in reports. A shared SDK or mock runtime is useful integration
evidence but does not by itself establish independent implementation agreement.

## Initial source inventory

| Project | Role and source revision | Scope established by source | Execution evidence |
|---------|--------------------------|-----------------------------|--------------------|
| [napplet/web SDK](https://github.com/napplet/web/tree/10fd4e618ad4ae742105ca52fd79bf9d03ffc6e1/packages/sdk) | Web SDK; `10fd4e618ad4ae742105ca52fd79bf9d03ffc6e1`; package metadata names `@napplet/sdk` version `0.29.0`. | Package exists. A domain-by-domain implementation audit is not attached. | not-run |
| [napplet conformance engine](https://github.com/napplet/web/tree/10fd4e618ad4ae742105ca52fd79bf9d03ffc6e1/packages/conformance) | Development/test engine; same revision; `@napplet/conformance` version `0.19.0`. | [Documentation](https://github.com/napplet/web/blob/10fd4e618ad4ae742105ca52fd79bf9d03ffc6e1/apps/docs/packages/conformance.md) describes outbound envelope checks, manifest-event shape checks, and a reference mock shell. | not-run; documentation establishes claimed scope only |
| [napplet conformance CLI](https://github.com/napplet/web/tree/10fd4e618ad4ae742105ca52fd79bf9d03ffc6e1/packages/conformance-cli) | Browser test runner; same revision; `@napplet/conformance-cli` version `0.3.3`. | [Documentation](https://github.com/napplet/web/blob/10fd4e618ad4ae742105ca52fd79bf9d03ffc6e1/apps/docs/packages/conformance-cli.md) describes headless Chromium and the `napplet-conformance` executable. | not-run |
| [kehto/web](https://github.com/kehto/web/tree/49658b2d5458e47f5cbdfb24b30c49dbc44be2e9) | Runtime project; `49658b2d5458e47f5cbdfb24b30c49dbc44be2e9`. | Repository has `packages/` and `tests/`; this entry records a runtime candidate, not verified domain coverage. | not-run; no compatibility matrix attached |

Repository renames and redirects do not create separate independent
implementations. Record a project's canonical URL and preserve immutable source
links. Extend this inventory when evidence is available, without inventing
support from a repository description or a list of message types.

## Domain report format

One implementation report can contain several domain rows, each with its own
revision and limitations. Include binding and runtime evidence separately when
they are implemented in different projects.

| Field | Record |
|-------|--------|
| Implementation | Repository URL, exact commit or released artifact hash |
| Role and projection | Napplet, SDK/binding, or runtime; host projection |
| Contract | Naps commit and spec path; upstream projection revision |
| Domain | Bare domain, exposed operations, optional dependency scenarios |
| Coverage | Applicable clauses/case IDs; pass, fail, blocked, not-run, not-applicable |
| Environment and policy | Host version, exposed/denied capabilities, defaults, limits |
| Evidence | Reproducible command, tool/corpus revisions, public sanitized report |
| Limitations | Missing operations, source conflicts, unsupported semantic features |
| Checked date | Observation date, not a promise of ongoing compatibility |

A domain object or manifest advertisement establishes neither complete operation
coverage nor compatible semantics. Root and snapshot manifests, absent optional
domains, denial paths, and endpoint replacement need explicit scenarios.

## Interoperability report format

Record the caller, binding/runtime, and target handler with independent project
names and immutable revisions. Name the convention, payload scenario, exposed
domains, policy, and expected behavior. Attach the actual exchange and negative
cases. Explain shared dependencies that limit the independence of the result.

Useful initial exchanges include profile opening between independent napplets,
INC exact-topic delivery, identity changes, and theme updates across independently
written bindings. A successful UI demonstration without traces is a useful demo,
but does not establish all routing or authorization checks.

## Updating evidence

Keep historical reports pinned; add new observations rather than changing an old
report's revision underneath it. Mark evidence superseded when the contract or
implementation changes. Unknown coverage remains unknown. Proposed domains in
open PRs are identified as proposals and do not become registered or mandatory
through this file.

For adoption, measure independent implementations, reproduced exchanges,
useful deployed napplets, and the effort needed to implement a domain. A growing
count of domains is not evidence that interoperability improved. The
[IAB protocol-success analysis](https://www.rfc-editor.org/rfc/rfc5218.html)
provides background on utility, deployment, and open implementation practices.

The normative web projection remains
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303).
