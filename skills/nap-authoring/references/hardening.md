# Hardening a NAP contract

Use these questions to find concrete ambiguities in the proposed surface. Apply
only relevant rows. Record each finding with the conflicting or missing rule,
an observable failure, and the smallest contract correction. This is authoring
guidance, not a new set of protocol requirements.

| Concern | Settle in the owning contract |
|---------|-------------------------------|
| Authority | Which authenticated endpoint may request this? Which user or napplet owns the affected data? What does shell policy decide? Never trust a payload's claimed identity or infer permission from manifest declarations. |
| Input | Required fields, omission versus null, enums, encoding, units, ranges, malformed input, and unknown-field behavior where it affects compatibility. Define validation before effects. |
| Results | Exact success and failure shapes; whether partial success exists; whether a result means acceptance, delivery, or completion. No contradictory success fields on error. |
| Correlation | Request/result `id`, caller scoping, and routing identifiers. Concurrent callers using the same id must not receive each other's data or results. |
| Failure | Invalid requests, denied policy, absent optional dependencies, and backend failures have defined observable outcomes. Do not promise a response to an unauthenticated or unrouteable message. |
| Lifetime | For stateful work: creation, ownership, release, caller disconnect, revocation, and late results. One endpoint must not cancel or release another's resources. |
| Repetition | For effects: duplicate requests, retries, and ambiguous timeout outcomes. Correlation ids do not automatically provide deduplication or exactly-once execution. |
| Streams | When present: initial state versus updates, ordering, termination, unsubscribe, races, and behavior under slow consumers. Do not invent wire subscribe/unsubscribe for a purely local listener. |
| Resource use | Relevant payload sizes, outstanding work, queue or rate bounds, and observable overload behavior. State which limits are contract requirements and which are runtime policy. |
| Data exposure | Scope stored data and results to their owner; avoid revealing secrets or other napplets' state through errors, enumeration, or broadcasts. |
| External effects | URLs, relay targets, signed data, payments, or file access remain mediated. Link the owning domain and upstream rules rather than silently granting new authority. |
| Compatibility | Reuse the existing domain and type owners. For revisions, identify old/new behavior and whether existing callers can still operate. Availability alone does not prove a new operation is supported. |

## Trace examples through the contract

Write concrete input and observable output for the applicable cases below.
For a documentation-only task, these are consistency checks, not runtime proof.
If an implementation is in scope, turn them into executable conformance cases
using that project's existing tooling.

1. Minimum valid request and a successful result with the same correlation id.
2. Invalid field or range, with a specified failure and no forbidden effect.
3. Policy denial despite a declared domain or caller-supplied identity.
4. Missing optional dependency and the documented fallback or unavailable feature.
5. Two endpoints with overlapping request/resource identifiers and isolated results.
6. For ongoing work, disconnect or cancellation followed by a late event.
7. For retryable effects, a duplicate request or timeout without an invented
   exactly-once guarantee.

Do not manufacture unresolved numeric limits or error codes. Propose the
smallest defensible rule, label it as a new design choice, and make any remaining
interoperability blocker explicit before calling the draft ready.

## Final consistency pass

- Each operation has one meaning across tables, prose, examples, and errors.
- Every record is defined once or imported from its owner. Every dependency has
  a kind, strength, concrete reason, and matching registry Deps entry.
- Security obligations name their enforcing actor. Cryptographic trust and
  privileged execution remain with the shell/runtime.
- Host mechanics stay in projection guidance; no normative SDK-language API.
- No deferred domain leaks into active schemas, behavior, or dependencies.
  Search current deferred domain names across `naps/`, `projections/`, and
  `README.md`; inspect matches for live surface beyond their italic registry rows.
- External specifications are linked at their canonical or living PR URLs.
  No local mirror reproduces their bodies.
- No placeholder, unverified implementation claim, or illustrative example is
  presented as an established protocol guarantee.
- Remove duplicated requirements; preserve all decisions needed by a second
  implementation. Report any unresolved contradiction instead of hiding it.
