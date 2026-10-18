# NAP conformance

**Interoperability risk:** current NAP-INTENT normalization rejects fragments
while its recommendation section and web projection permit them. Its repeated
options schemas also disagree and its request record omits recommendation
fields. These checks are **blocked**, not passed or silently waived. See
[PR #106](https://github.com/napplet/naps/pull/106).

Conformance is a claim about a named implementation, projection, domain, and
contract revision. This document describes evidence and reporting. Requirements
remain in the [NAP registry](README.md), owning specs, and projections. It creates
no universal capability set, handshake, certification service, or wire version.

## State the claim

Record the implementation repository, immutable revision or artifact hash,
tested role (napplet, binding, or runtime), projection, and domain set. Include:

| Record | Evidence |
|--------|----------|
| Contract | Naps commit; each spec path/revision; upstream projection revision |
| Environment | Host/browser version, binding and SDK versions, configuration |
| Policy | Exposed and denied domains, resource rules, handler defaults, limits |
| Tests | Tool version/revision, corpus revision, exact command, case IDs |
| Results | Per-case outcomes, traces/logs, exclusions and their reason |
| Interoperability | Counterpart implementation and revision; independently observed exchange |

The same SDK and mock shell on both sides provide useful integration evidence,
but do not establish agreement between independently implemented runtimes.
An unavailable optional domain is an availability scenario, not a failure merely
because the domain is absent. A claimed exposed domain is checked against its
own requirements. Apply `SHOULD` clauses with a documented reason for deviations.

## Separate what the checks prove

| Scope | Positive evidence | Negative evidence | Limit |
|-------|-------------------|-------------------|-------|
| Manifest/artifact | Valid signed manifest and verified executed artifact | Substitution, invalid signatures/hashes, incompatible required domains | Event shape validation alone does not prove signature or byte verification. |
| Projection | Authenticated endpoint, correct availability before code runs | Unknown source, forbidden authority, denied-domain exposure | Web results do not establish native or WASM behavior. |
| Envelope/schema | Direction, discriminant, correlation, required field shapes | Malformed outbound fields and wrong-direction messages | Schema validity does not prove authorization or delivery. |
| Domain behavior | Operations, pushes, lifecycle, and policy boundaries | Spoofing, leakage, late registration, rejection, peer replacement | A happy-path call does not cover the domain. |
| Composition | Independent caller and handler agree on convention semantics | Near-match identities, conflicting advertisements, invalid payload | A single convention exchange does not certify every convention. |

## Existing tooling

Use the [napplet conformance engine](https://github.com/napplet/web/tree/main/packages/conformance)
and its [browser CLI](https://github.com/napplet/web/tree/main/packages/conformance-cli)
for the checks they actually implement. The
[engine documentation](https://github.com/napplet/web/blob/10fd4e618ad4ae742105ca52fd79bf9d03ffc6e1/apps/docs/packages/conformance.md)
describes envelope and manifest-event validation plus a reference mock shell.
The [CLI documentation](https://github.com/napplet/web/blob/10fd4e618ad4ae742105ca52fd79bf9d03ffc6e1/apps/docs/packages/conformance-cli.md)
describes execution in headless Chromium. These are development tools.

For a built napplet, the package metadata at that toolchain revision exposes the
`napplet-conformance` executable from `@napplet/conformance-cli` version `0.3.3`:

```sh
npx --yes --package @napplet/conformance-cli@0.3.3 napplet-conformance ./dist
```

Run this in the napplet project after building its artifact and satisfying the
tool's browser prerequisites. This repository is a specification registry and
has no napplet `dist` to run. Record the package version and its dependency
resolution; the top-level CLI pin alone does not pin every transitive dependency.

Use the implementation's own runtime tests for endpoint authentication, policy,
catalogs, lifecycle, and delivery. Reuse that runner and the existing engine when
adding shared fixtures. This document does not claim automatic ingestion of a
corpus that the tool does not yet support.

## Outcomes

| Outcome | Meaning |
|---------|---------|
| pass | The applicable assertion was observed at the recorded revisions. |
| fail | Observation contradicts an applicable, unambiguous requirement. |
| blocked | Conflicting requirements or missing prerequisites prevent a trustworthy judgment. |
| not-run | The applicable case has not been executed; state why. |
| not-applicable | The case's preconditions do not apply; state the supported scope or policy. |

Never combine blocked, not-run, or not-applicable cases into a pass count. Report
coverage by applicable requirements as well as case counts. A policy may choose
among allowed behaviors, but cannot waive an applicable `MUST`.

## Priority behavior checks

| Contract | Cases to cover |
|----------|----------------|
| [NAP-SHELL](naps/NAP-SHELL.md#shell-behavior) | Other domains without `shell`; calls before environment delivery; repeated readiness; scoped environment. |
| [NAP-INC](naps/NAP-INC.md#shell-behavior) | Exact topic routing and sender exclusion; runtime provenance; ACL-denied channel open; target notification ordering; late listeners; overflow; destroyed peers. |
| [NAP-IDENTITY](naps/NAP-IDENTITY.md#shell-behavior) | Empty pubkey without a signer; startup snapshot; sign-out push; separation from endpoint identity; read-only authority. |
| [NAP-THEME](naps/NAP-THEME.md#shell-behavior) | Complete colors and minimal payload; automatic updates only to exposed domains; no wire subscription; optional media fallback. |
| [NAP-INTENT](naps/NAP-INTENT.md#runtime-behavior) | Manifest-derived contracts; exact matching; authenticated sender; acceptance before source closure; source-independent retained delivery; user selection and policy. |

## Source conflicts and maintenance

Baseline assessment: naps `e5308ac92cfd49e4fb026812c60886571515e219`.
The intent fragment and recommendation schema checks described above are blocked
at this revision. Other cases can still run if their expectations do not depend
on those ambiguities. A green schema validator cannot settle the disagreement.

When a contract changes, map the semantic diff to affected cases. Preserve the
old fixture revision for historical reports; update expectations from the owning
spec and record the changed revision. Re-run affected interoperability exchanges
when serialization, routing, authorization, or lifecycle changes.

Reference reports should include failed and missing checks. Publish only logs
cleared of credentials, private payloads, and personal data. The web authority
remains [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303).
