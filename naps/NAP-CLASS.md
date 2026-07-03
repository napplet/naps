NAP-CLASS
=========

Napplet Class Authority
-----------------------

`draft`

**NAP ID:** NAP-CLASS
**Domain:** `class`
**Web binding (NIP-5D):** `window.napplet.class` · `shell.supports("class")`
**Parent:** NIP-5D

## Description

NAP-CLASS establishes an abstract "napplet class" concept — an integer number assigned by the shell at iframe-ready time, delivered over the wire, and readable by the napplet at `window.napplet.class`. The class captures the shell's determination of the napplet's security posture: CSP shape, consent state, any additional shell-enforced constraints a concrete posture cares to name. This NAP OWNS the class-track machinery; concrete class definitions live in sibling `NAP-CLASS-$N` documents.

The shell is the sole authority on class. Class is determined by which class-contributing NAPs are declared in the napplet manifest combined with any user-consent outcomes. The shell MUST send exactly one `class.assigned` envelope per napplet lifecycle, after the iframe signals readiness and before the napplet has an opportunity to meaningfully branch on class. Dynamic mid-session re-classification is out of scope for v1; shells that need to change class for an already-running napplet MUST tear down the frame and create a new one.

This NAP is voluntary. Napplets MUST gracefully degrade when `window.napplet.class` is `undefined` — which happens either because the shell does not implement `nap:class`, or because the `class.assigned` envelope has not yet arrived. Cross-NAP invariants (for example, in a shell implementing both NAP-CLASS and some class-contributing NAP, that the advertised class matches the contributing NAP's runtime state) are shell responsibilities documented by the NAPs that participate. NAP-CLASS itself has no opinion on how a given integer is derived; it only specifies how that integer is communicated.

## API Surface

`NappletClassState`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `class` | `uint` | no | Shell-assigned class number; absent until assigned or unsupported. |

Napplets read `window.napplet.class`. Before depending on the value, napplets SHOULD check `shell.supports('nap:class')`; a shell that does not advertise this capability will never populate the field. Once the field has been populated by the shell's `class.assigned` envelope, it MUST NOT change for the remainder of the napplet's lifetime.

## Wire Protocol

`class.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `class.assigned` | shell -> napplet | `id`, `class` |

Normative constraints on `class.assigned`:

- The shell MUST send `class.assigned` after the iframe signals readiness (shim bootstrap complete) and before any other napplet-bound envelope for that iframe. The napplet's first observation of a shell-originated envelope for the current lifecycle SHOULD be `class.assigned`.
- The shell MUST send AT MOST ONE `class.assigned` envelope per napplet lifecycle. Dynamic re-classification is out of scope for v1. A second `class.assigned` for the same napplet is a protocol violation; conformant napplet shims MAY silently drop it, and SHOULD surface a diagnostic.
- The `id` field is a freshly minted correlation identifier per NIP-5D envelope conventions. There is no napplet→shell response envelope for `class.assigned`; the correlation id exists for consistency with other NIP-5D envelopes, not because a response is expected.
- The `class` field is a non-negative integer. The integer value selects a concrete class definition from the `NAP-CLASS-$N` sub-track (e.g., `class: 1` selects `NAP-CLASS-1`). No further payload fields are defined; shells MUST NOT attach class-specific policy configuration to this envelope.
- The shell MUST NOT use `class.assigned` as a configuration channel. Policy semantics belong to the corresponding track member; the integer is the only wire-level signal.

### Example

```
shell -> napplet: { "type": "class.assigned", "id": "c1", "class": 2 }
```

The napplet shim writes `2` into `window.napplet.class` upon receipt and consults `NAP-CLASS-2.md` for the semantics of that class.

## NAP-CLASS-$N Sub-Track

The NAP-CLASS track is the set of documents named `NAP-CLASS-$N.md`, where `$N` is the non-negative integer carried by `class.assigned`. Each track member is a standalone normative spec that fully defines one security posture. A shell implementer implementing class `N` correctly MUST be able to do so by reading only `NAP-CLASS.md` and `NAP-CLASS-$N.md`.

### Naming rules

- Class numbers are non-negative integers.
- Integers ascend (1, 2, 3, ...). The next published class is one greater than the largest previously published class.
- A class number MUST NOT be reused once published to the public NAPs repo. A published class may be superseded by a later class, but its integer remains retired.
- A class number MUST NOT be skipped without spec-level justification. If a gap is intentional, the gap number MUST exist as a `NAP-CLASS-$N.md` document marked `reserved` in its status banner, explaining why the number is held.
- File names are always `NAP-CLASS-$N.md` (e.g., `NAP-CLASS-1.md`, `NAP-CLASS-2.md`). Everything participating in the track is under the `NAP-` namespace.

### Required content structure for each sub-track member

Every `NAP-CLASS-$N.md` MUST include, in order:

1. **Setext heading + subtitle + status banner** matching the shape of NAP-CLASS.md (setext `=` underline for the title, setext `-` underline for the subtitle, backticked status banner, metadata block).
2. **Description** — at minimum a single paragraph naming the posture in one sentence and identifying its role in the track.
3. **CSP posture** — the exact CSP directive shape the shell MUST emit for napplets assigned to this class, stated as wire shape rather than as prose. At minimum, the `connect-src` shape; other directives are optional for the class to name.
4. **Manifest prerequisites** — the conditions a napplet manifest MUST satisfy to be assigned to this class (what class-contributing NAPs or tags trigger the class, and any user-consent prerequisites).
5. **Shell responsibilities** — enumerated MUSTs the shell MUST honor at iframe serve time (consent prompt? emit grant-derived header? persist state? refuse-to-serve conditions?).
6. **Security considerations** — the threat model the posture defeats and the attacks it explicitly does not defeat.
7. **References** — cites `NAP-CLASS.md` as parent, and any additional context the track member relies on.

A track member SHOULD additionally include worked example CSP strings, compatibility notes (how napplets authored against other classes interoperate), and a "Assignment" subsection reiterating that class communication is exclusively via `class.assigned`.

### Citation conventions

NAPs and other specs MUST cite class documents by file name (for example, "see `NAP-CLASS-2.md`"). They MUST NOT use the abstract phrase "Class 2" or "class 2" as the primary reference. File-name citations prevent prose drift when a class document is renamed, retired, or superseded; abstract phrasings outlive the document they nominally refer to.

A spec that needs to discuss the conceptual integer without pointing at a specific member (for example, when discussing the track as a whole) SHOULD use the phrase "the class number" or "the integer carried by `class.assigned`" rather than "Class N". Specific members are always cited by file name.

### Adding a new class

Authors proposing a new class MUST open a PR to the public `napplet/naps` repo containing (a) a new `NAP-CLASS-$N.md` file following the required content structure above, and (b) an update to this section documenting the new entry in the track. The class number is assigned at merge time by the NAPs-track maintainers, following the ascending-integer rule. A proposed class whose integer is already in use MUST be renumbered before merge.

## Capability Advertisement

Shells implementing this NAP MUST advertise `shell.supports('nap:class') === true`. Napplets MUST NOT depend on `window.napplet.class` having a value; they SHOULD check `shell.supports('nap:class')` first. If the shell does not advertise the capability, the napplet MUST fall back to treating itself as unclassified — assuming the most restrictive defaults the napplet can function under. A shell that advertises `nap:class` but fails to deliver `class.assigned` is non-conformant; a shell that simply does not advertise `nap:class` is conformantly unclassified.

## Security Considerations

### Shell is sole authority

Napplet code MUST NOT attempt to infer its own class from the environment — from the CSP it observes, from the capabilities the shell advertises, from the presence or absence of other NAPs. The shell's `class.assigned` envelope is the only authoritative signal. A napplet that ignores `class.assigned` and assumes a class risks being confused between a strict-posture shell and a permissive-posture shell, and may make unsafe assumptions about capabilities the shell has not granted.

The integer carried on the wire is a reflection of the shell's enforcement decision, not a substitute for it. Sandboxing, CSP, and consent MUST be applied at frame creation time, before any napplet code runs. A malicious or buggy napplet that ignores `window.napplet.class` still runs under whatever posture the shell enforced; the integer is informational to the napplet, not instructive to the shell.

### At-most-one envelope

The at-most-one-terminal-envelope-per-lifecycle constraint prevents races where the napplet's behavior depends on late-arriving class signals, and prevents a compromised napplet context from prompting the shell for a re-assignment mid-session. Shells MUST NOT emit a second `class.assigned` to "upgrade" or "downgrade" a class mid-session. If a shell wishes to change posture, it MUST tear down the current napplet iframe and create a new one, at which point the new iframe's lifecycle begins with its own `class.assigned` envelope.

### Cross-NAP invariants are shell responsibilities

NAPs that contribute to class determination document their own invariants (for example, "a napplet with this NAP's tags and granted consent takes on class `N`"). Enforcing those invariants is a shell obligation, not a NAP-CLASS obligation. If a shell implements a class-contributing NAP but advertises a class integer inconsistent with that NAP's runtime state, the shell is non-conformant to the class-contributing NAP — NAP-CLASS itself has no opinion on whether any particular integer is "correct" for a given napplet. NAP-CLASS only specifies how the integer is communicated.

### Graceful degradation

`window.napplet.class === undefined` is a valid and well-defined state. It means one of: the shell does not implement `nap:class`; the shell implements it but has not yet delivered `class.assigned`; the napplet is running in an environment that does not implement the protocol at all. Napplets that branch on class MUST handle the undefined case as "unclassified — assume the most restrictive defaults the napplet can function under." Napplets MUST NOT assume that a missing class means any particular posture is active; the correct assumption is always the most restrictive one the napplet can support.

## References

- NIP-5D — Napplet wire format (JSON envelope over `postMessage`); parent spec for this NAP.
- `NAP-CLASS-1.md` — strict baseline posture; track member for `class: 1`.
- `NAP-CLASS-2.md` — user-approved explicit-origin posture; track member for `class: 2`.

## Changelog

- `78f9157` - Introduced NAP-CLASS as the napplet class authority sub-track root.
