NAP-CLASS-2
===========

User-Approved Explicit-Origin Class
-----------------------------------

`draft`

**NAP ID:** NAP-CLASS-2
**Parent:** NAP-CLASS
**Class number:** 2

## Description

NAP-CLASS-2 is the posture for napplets that declare explicit direct-network origins and have received user consent for those origins. Shells emit a runtime CSP with `connect-src <granted-origins>`, prompt the user on first load, persist the decision keyed on `(dTag, aggregateHash)`, and send `class.assigned` with `class: 2` when approved. If the user denies the prompt, the napplet is served under the NAP-CLASS-1 posture instead (`class: 1`, `connect-src 'none'`) — denied napplets are indistinguishable from napplets that never declared origins.

## CSP Posture

```
connect-src <space-separated-list-of-granted-origins>
```

Shells emitting the NAP-CLASS-2 posture MUST include `connect-src <granted-origins>` in the runtime CSP, where `<granted-origins>` is the user-approved origin list for the napplet identified by `(dTag, aggregateHash)`. The origin list MUST be exactly the set the user approved — shells MUST NOT add, remove, or normalize origins between the approved set and the emitted directive. Other CSP directives in the baseline are shell-policy concerns; only the `connect-src` value is the class's defining characteristic.

If the approved set is empty (the user approved zero origins on the prompt), the shell MUST emit `connect-src 'none'` and SHOULD downgrade the assignment to NAP-CLASS-1 — an empty-set NAP-CLASS-2 is behaviorally indistinguishable from NAP-CLASS-1 and introduces no value. Shells MAY retain the NAP-CLASS-2 assignment for an empty approved set if doing so materially affects downstream behavior (for example, to record that the user did review a prompt), but the CSP surface is identical to NAP-CLASS-1 in that case.

## Manifest Prerequisites

A napplet manifest reaches NAP-CLASS-2 when it declares at least one tag from a class-contributing NAP whose trigger condition for NAP-CLASS-2 is met, AND the user has approved the napplet's request at first load. The specific manifest tag shape and origin format are defined by the individual class-contributing NAP (see that NAP's spec for the manifest-tag surface). NAP-CLASS-2 itself does not define any manifest tags — it defines the posture.

Validation of the origin set produced by the class-contributing NAP (syntactic shape, canonicalization, scheme whitelist, and similar rules) is likewise that NAP's responsibility. NAP-CLASS-2 consumes only the user-approved output of that process, emits it as `connect-src`, and enforces the grant's persistence rules; it does not re-validate the origins.

## Shell Responsibilities

- Shell MUST prompt the user at first load of any napplet matching `(dTag, aggregateHash)` that is not already in an approved-or-denied state. The prompt MUST identify the napplet and MUST enumerate the concrete origins under consideration literally, without summarization or first-presentation elision.
- Grant persistence MUST be keyed on the exact `(dTag, aggregateHash)` composite; shells MUST NOT key on `dTag` alone. Keying on `dTag` alone would allow a rebuilt napplet (new `aggregateHash`) to inherit a prior approval silently — this is a security bug, not a UX optimization.
- Shell MUST emit a runtime CSP header containing `connect-src <approved-origins>` when serving an approved NAP-CLASS-2 napplet.
- Shell MUST send exactly one `class.assigned` envelope with payload `{ class: 2 }` at iframe-ready time for approved napplets. For denied napplets, the shell MUST send `class.assigned` with `{ class: 1 }` — denied napplets are served under the NAP-CLASS-1 posture, and the wire must reflect that.
- Shell MUST refuse-to-serve any NAP-CLASS-2 napplet whose HTML contains `<meta http-equiv="Content-Security-Policy">`. Browser CSP intersection silently reduces header `connect-src <granted>` AND meta `connect-src 'none'` to `'none'`, suppressing the grant without any diagnostic to the user or the napplet author. The refuse-to-serve diagnostic MUST identify the offending meta element and point the napplet author at rebuild guidance.
- Shell MUST expose a user-facing revocation affordance for approved grants. Revoked grants MUST move to a DENIED state (not deleted) so shells retain historical knowledge and can offer a re-approve path without re-soliciting a fresh prompt from scratch.
- Shell SHOULD reload the napplet's iframe on revocation — the next load will emit `class.assigned` with `class: 1` and restrictive CSP. In-session mid-lifecycle class re-assignment is out of scope (enforced by NAP-CLASS's at-most-one-envelope rule); revocation during an active lifecycle is allowed to take effect only at the next frame creation.
- Shell MUST NOT rewrite, reorder, or merge origin tokens between the consent prompt the user saw and the CSP it emits. The approved set is byte-for-byte the set the user read.

## Grant Persistence Semantics

Grants are keyed on the composite `(dTag, aggregateHash)`. Rebuilding the napplet with any change that alters `aggregateHash` (including changes to the declared origin set from any class-contributing NAP) produces a new key and auto-invalidates the prior grant — the user is prompted fresh on the next load. This is the primary defense against silent supply-chain upgrade: a prior approval for `(dTag, v1)` does not automatically extend to `(dTag, v2)`.

Shells SHOULD show a diff UI on re-prompt when the same `dTag` has been previously approved but the `aggregateHash` has changed: "previously approved origins X; now requesting origins Y." This is a UX floor for informed re-consent — users who approved v1 should be able to understand what changed in v2 before approving again. Shells MUST NOT present the re-prompt as if it were a first prompt when a prior grant exists under the same `dTag`; the historical context matters.

Shells MAY extend the keying tuple (for example, to include additional attributes such as author pubkey or network partition), but an extended key MUST be a strict superset of `(dTag, aggregateHash)` — the two components MUST remain present and MUST continue to carry the semantics described above — so that grants remain portable across shells that implement this class conformantly. A shell that keys grants on a subset of `(dTag, aggregateHash)` is non-conformant to NAP-CLASS-2.

## Security Considerations

### Post-grant opacity

Once origins are approved, the shell has no browser-level hook to inspect or filter subsequent network traffic between the napplet and the approved origins. This is a fundamental tradeoff of direct-network grants. Consent-prompt language MUST reflect this: the user is approving the napplet to send and receive any data with the listed servers; the shell cannot see or filter that traffic. Consent UI that understates the scope of the grant is misleading the user.

### Cleartext origins

Shells MAY refuse cleartext schemes (`http:`, `ws:`) entirely, and SHOULD document their policy. When cleartext is permitted, the consent prompt MUST visibly mark cleartext origins so the user can evaluate the confidentiality tradeoff. Cleartext origins granted from an HTTPS shell are subject to mixed-content rules imposed by the browser and may silently fail at connect time; that failure is orthogonal to the consent layer but may surprise the napplet author.

### Composite-key hygiene

A shell that keys grants on `dTag` alone would allow a rebuilt napplet (new `aggregateHash`) to inherit a prior approval silently — this is a security bug, specifically a silent-supply-chain-upgrade vector. The `(dTag, aggregateHash)` composite is load-bearing. A shell that elides the hash, or that treats the hash as advisory, is not implementing NAP-CLASS-2 — it is implementing a weaker posture that MUST be assigned its own class number.

### Residual meta-CSP is a Class-2 project-killer

Unlike NAP-CLASS-1, where residual meta-CSP is harmless (intersection of `'none'` and `'none'` is `'none'`), a NAP-CLASS-2 napplet with residual meta-CSP has its granted origins silently suppressed by browser CSP intersection (intersection of `<granted>` and `'none'` is `'none'`). The user sees a prompt, approves origins, and then the napplet still cannot reach those origins — with no error message pointing at the cause. The refuse-to-serve MUST exists specifically to prevent this silent-failure mode. A shell that does not implement the refuse-to-serve is not implementing NAP-CLASS-2 faithfully, even if every other surface is correct.

### Revocation timing

Revocation of an active grant MUST take effect no later than the next time the affected `(dTag, aggregateHash)` is loaded. Shells are not required to tear down an already-running napplet iframe on revocation (NAP-CLASS's at-most-one-envelope rule forbids in-session class re-assignment), but they MUST NOT emit the revoked origins in `connect-src` for any napplet frame created after the revocation is acknowledged. A shell that continues to serve revoked origins to new frames is non-conformant.

### Denied napplets are Class-1 napplets

A user who denies the NAP-CLASS-2 consent prompt produces a napplet that is served identically to a NAP-CLASS-1 napplet: `connect-src 'none'`, `class.assigned` with `class: 1`. This is deliberate. The napplet MUST NOT be able to detect, from its own runtime surface, whether it is running as a "genuine" NAP-CLASS-1 or as a denied NAP-CLASS-2. Allowing the napplet to distinguish would let it pressure the user — for example, by refusing to function meaningfully under the denied-equivalent posture — and would convert the consent prompt from a binary decision into a coercive one.

## References

- `NAP-CLASS.md` — parent spec; defines the `class.assigned` envelope, the `window.napplet.class` runtime surface, and the authoring rules for track members including NAP-CLASS-2.
- `NAP-CLASS-1.md` — sibling track member; the posture a denied NAP-CLASS-2 napplet is served under.
- NIP-5D — Napplet wire format.
