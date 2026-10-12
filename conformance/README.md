# Shared NAP conformance fixtures

**Blocked contract checks:** the current intent text conflicts on handler
fragments and recommendation serialization. `blocked.json` records these cases
without inventing an expected wire shape. See
[reconciliation PR #106](https://github.com/napplet/naps/pull/106).

This corpus supplies language-neutral input and observation fixtures. It is test
data, not a new NAP API, normative wire schema, or runner. Requirements remain
in the linked owning specs. Baseline: naps
`e5308ac92cfd49e4fb026812c60886571515e219`.

## Files

| File | Scope |
|------|-------|
| [fixture.schema.json](fixture.schema.json) | JSON Schema for corpus metadata and case records; not a protocol-message schema |
| [bindings.json](bindings.json) | INC convention URI transposition, exact text values, and pre-emission rejection |
| [behavior.json](behavior.json) | Runtime/binding scenarios for provenance, availability, routing, and lifecycle |
| [blocked.json](blocked.json) | Conflicting intent requirements awaiting owning-contract reconciliation |

## Case format

Each file records `format_version`, the naps `contract_revision`, and `cases`.
Each case has an ID, domain, layer, requirement references, status, preconditions,
input, and expected observations. IDs are stable report references. The format
version describes this test-data format only and never appears on the NAP wire.

`input` uses literal wire objects or scenario descriptions. `expected.assertions`
names independently observable outcomes. A binding case may additionally supply
an exact `wire` object or `reject_before_emission: true`. Rejection cases do not
mandate an error string or a new wire response for a fire-and-forget operation.

Status `ready` means the expected observations derive from unambiguous source
requirements. It does not mean the case has run or passed. Status `blocked`
requires a reason and empty assertions; adapters report it as blocked, never pass.

Fixture names such as `endpoint-a`, `catalog-source`, and `channel-1` are aliases
for identities a harness creates. They do not specify identifier encoding. Map
them to actual authenticated endpoints/catalog entries, and compare attested
identity rather than testing a literal runtime-generated string.

## Adapting to existing tools

1. Pin the implementation, corpus, naps contract, and upstream projection
   revisions. Confirm each requirement still applies at that revision.
2. Use the case's layer: `binding` operates on the developer-facing call;
   `runtime` authenticates real endpoints and observes policy/routing;
   `composition` exercises source/target lifecycle. Do not send a binding URI
   directly as already-normalized wire input.
3. Configure exactly the listed preconditions. Establish identities through
   runtime verification, not by placing fixture aliases in an untrusted payload.
4. Capture emitted requests, results, target deliveries, local handler calls,
   and teardown in the owning project's test harness. Match the assertions.
5. Report pass, fail, blocked, not-run, or not-applicable with reasons and traces.
   A timeout follows the configured test environment and does not invent a
   protocol deadline.

Reuse [the conformance engine](https://github.com/napplet/web/tree/main/packages/conformance)
for supported schema checks and [its browser CLI](https://github.com/napplet/web/tree/main/packages/conformance-cli)
for supported napplet checks. Their existing interfaces do not promise automatic
ingestion of these files. Runtime and composition cases require adapters in the
implementation's existing tests. Do not equate corpus JSON/schema validation
with protocol conformance.

## Coverage limits

The initial corpus samples boundaries in `inc`, `shell`, `identity`, `theme`, and
`intent`. It does not cover every operation, error, optional dependency, browser
sandbox property, manifest signature, or artifact verification rule. Root,
snapshot, and named manifest verification belong to the
[upstream web binding](https://github.com/nostr-protocol/nips/pull/2303).
No execution evidence or independent interoperability result is bundled here.

When extending the corpus, reference the owning requirement and include a
negative case when the change touches authority, identity, routing, or lifecycle.
Preserve historical revisions for reports. Resolve disputed requirements in the
owning spec before changing a blocked case into an executable expectation.
