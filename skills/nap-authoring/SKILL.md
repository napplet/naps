---
name: nap-authoring
description: Write, revise, or review Nostr Applet Protocol (NAP) specifications. Use for runtime capability proposals, NAP format and boundary questions, and hardening a NAP contract for independent implementations. Not for building napplet applications.
---

# NAP authoring

Produce a small, complete contract that an independent napplet author and runtime
author can implement without guessing. NAP means **Nostr Applet Protocol**.

## Read the current contract

This skill is authoring guidance, not a second protocol registry. In a checkout
of `napplet/naps`, read the files below locally. Otherwise retrieve them from
the canonical repository using these links; installed skills do not include the
repository's parent files.

- [AGENTS.md](https://github.com/napplet/naps/blob/master/AGENTS.md): boundaries,
  dependencies, contribution rules, and terminology.
- [SOUL.md](https://github.com/napplet/naps/blob/master/SOUL.md): purpose and values.
- [README.md](https://github.com/napplet/naps/blob/master/README.md): glossary,
  domain registry, status, and dependencies.
- [NAP-WORD-TEMPLATE.md](https://github.com/napplet/naps/blob/master/NAP-WORD-TEMPLATE.md):
  the starting structure for a runtime API.

Read the target spec, its dependency owners, and the relevant projection before
editing. Registry entries may point to living PRs whose specs are absent from
`master`; inspect the linked PR's current files and status. Do not treat a draft,
a registry listing, or an SDK method as proof of implemented support. Check an
implementation separately when making a support claim.

Current repository rules and owning contracts take precedence over this guide
and incidental patterns in older examples. If current sources contradict one
another, identify the conflicting clauses and check the linked living PR;
surface any unresolved conflict before presenting dependent behavior as settled.
Do not silently repair unrelated specs as part of the requested draft.
If sources are unavailable, identify
the missing source and label the output provisional; do not invent its contract.
Link living upstream documents, including
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303), rather than copying them.

## Establish the seam

The goal is **interoperability through one clear seam**: the capability boundary
between a napplet and its runtime. The napplet **designs**, the shell **mediates**,
the runtime **executes**, and the user **composes**. Small tools cooperate under
user choice; runtime convenience must not override user preferences or authority.
The seam is transport-neutral and Nostr-native.

Classify the requested behavior before naming or drafting it:

| Behavior | Owner and next step |
|----------|---------------------|
| An API provided by the runtime | NAP-WORD; inspect existing domains before proposing another. |
| Message semantics agreed by napplets | Convention; use the [convention template](https://github.com/napplet/naps/blob/master/CONVENTION-TEMPLATE.md). Do not register it as a NAP. |
| Host transport, sandbox, injection, or SDK binding | Projection; use the [web projection](https://github.com/napplet/naps/blob/master/projections/web.md) and its upstream contract for browser details. |
| App UI, business logic, or implementation internals | Keep it with the implementation unless it changes the shared capability contract. |

State the capability, its owner, and what remains outside it in a few sentences.
If ownership is ambiguous, surface that precise conflict before drafting the
dependent surface. Do not silently choose a side or expand the user's task.

Keep these boundaries explicit:

- A NAP owns only its runtime-provided API. Dependencies do not transfer ownership.
- Every domain is optional for a runtime to implement. A napplet's required
  domain declarations constrain loading; optional integrations need fallbacks.
  Manifests name bare domains, such as `relay`, not `NAP-RELAY`.
- Availability comes from the projection. Do not introduce a `shell` handshake
  as a prerequisite for discovery. Declarations and advertisements are not grants.
- Napplets are untrusted. The runtime binds authenticated endpoints to verified
  manifests and artifacts. User identity and napplet identity are distinct.
  Napplet-supplied identifiers do not establish authority.
- Signing keys, wallet credentials, raw network access, and privileged execution
  stay behind shell mediation. A URL or payload never grants authority to use it.
- A deferred domain has no live fields, parameters, dependencies, or behavior;
  only its italic registry row remains. Read current status before referencing it.
- Archetype names need no registration. NAAT is deprecated; do not revive its
  registry to describe a convention.

If convention URIs are involved, read
[references/conventions.md](references/conventions.md) before specifying them.

## Draft the minimum complete contract

Start from the current NAP-WORD template. Use a single uppercase word for the
spec (`NAP-WORD`), a bare domain, and `draft` status for a new proposal. A name
is proposed until approved by the maintainer; do not assign a numbered NAP.

Keep the template's web-binding pointer explicitly informational about the
projection. Browser mechanics and `window.napplet.*` SDK shapes must not define
the normative API. Use language-neutral operations and schemas even if an older
template example uses shorthand such as `tstr`.

| Part | What it must settle |
|------|---------------------|
| Preamble | NAP ID, domain, status, dependencies when present, and projection pointer. |
| Description | One paragraph: what the shell provides, what the napplet consumes, and why. |
| API Surface | Operation, parameters, result, and corresponding wire messages. |
| Schemas | Every record and enum; field, required, type, and notes. Distinguish omission, `null`, and empty values. |
| Wire Protocol | `domain.action`, direction, payload fields, request/result correlation, and unsolicited delivery routing. |
| Examples | Concrete request/result and relevant failure or delivery traces that agree with the tables. |
| Error Handling | Failure shape, success/error exclusivity, and observable outcomes for invalid input or denied operations. |
| Shell Behavior | Obligations with a named actor, trigger, and observable result. |
| Security Considerations | Capability-specific trust boundaries and enforceable runtime obligations. |
| Implementations | Verified links and support scope, or an honest statement that none are known. |
| Changelog | Semantic history, following the repository's commit and PR rules. |

Use `text`, `boolean`, `integer`, `number`, `null`, `any`, `list of T`,
`map of text to T`, named records, and quoted enum alternatives. Define units,
ranges, encodings, defaults, and length bounds where they affect interoperability.
Avoid implementation-language declarations, `Promise`, `readonly`, `unknown`,
and language-specific collection syntax. Narrow `any` unless opaque data is
intentional and its validation owner is explicit.

Choose verbs that add meaning: `get`, `query`, `publish`, `open`, `subscribe`.
Reject tautologies such as `count.count`. Requests and results correlate by
`id`; fire-and-forget messages omit it. Define routing for pushed deliveries.
Do not imply that a local callback registration requires a wire subscription.
Distinguish request acceptance, delivery, and completed effect when they differ.

Declare each dependency in the preamble by **domain**, with a concrete reason:

```markdown
**Depends:**
- `resource` — capability · optional — obtains referenced image bytes through `resource.bytes`; use a local placeholder when absent.
```

| Kind | Contract |
|------|----------|
| `wire` | Embeds a type owned by another domain. One owner defines it; importers link it without redefining it; both specs cross-link. |
| `capability` | Calls another domain's operation. Name that operation and leave its contract with its owner. |
| `layering` | Runtime-internal composition, declared for transparency; imposes nothing on the napplet. |

Use `required` when the dependency domain must be present and `optional` when
only some features need it. Describe the affected feature's behavior when an
optional dependency is absent. Omit `Depends:` when there are no dependencies,
as directed by the current template. Never make two specs mutually redefine a
shared type. Update the README Deps column to match declared edges.

## Harden, then shorten

Run [references/hardening.md](references/hardening.md) against the draft. Resolve
choices that would make two implementations disagree. Keep runtime policy
flexible where the outcome is specified; do not add speculative operations,
universal limits, or a new global error protocol just to fill a checklist.

Use RFC-2119 MUST, SHOULD, and MAY intentionally. Put each requirement in one
owning section and cross-reference it elsewhere. Delete repeated rationale,
marketing, and implementation tutorials. Retain details needed to implement
success, failure, authority, and lifecycle behavior. Concise means no redundant
contract text, not missing failure semantics.

## Verify and deliver

Trace each operation through its API row, schemas, wire row, error rules, shell
obligations, and examples. Check that every normative requirement is observable
and every example obeys the contract. Report unverified implementation behavior
separately from document consistency.

For repository changes, follow current AGENTS.md rules and the user's requested
delivery scope. Keep one capability concern per branch/PR; include its required
registry/status/Deps synchronization, not an unrelated README rewrite. A shared
wire-type change requires owner/importer synchronization; surface any conflict
with the one-spec rule and resolve the split before editing both contracts.
Projection changes also require projection and registry synchronization.

Use the repository's semantic changelog format and actual commit references;
never fabricate hashes. For a spec PR, use the spec's kebab branch name and
`NAP-<NAME>: <imperative summary>` title. Include Summary, Changes, and Downstream
when applicable; state draft status, the web namespace as projection guidance,
and link [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303). Mirror the spec's
changelog at the bottom of its living PR body.

Before delivery, inspect the complete diff against the intended base. Keep
private context and `.planning/` out of public artifacts. Lead with unresolved
interoperability threats, then give the draft or changed files, validation
evidence, and remaining gaps. Creating a proposal does not imply implemented
support, permission to publish, or permission to merge.
