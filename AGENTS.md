# AGENTS.md

**`napplet/naps`** — registry and governance for **NAPs** (Nostr Applet Protocol):
the capability **seam** between a napplet and its runtime. **This repository is public.**

This file is the entry point and the source of truth for working here. Read it,
then fan out via the links below. `CLAUDE.md` is a symlink to this file.

## Map

| Path | What |
|------|------|
| [README.md](README.md) | Concepts, **glossary (canonical terms)**, NAP registry, archetype registry |
| [naps/](naps/) | Runtime-provided NAP interface definitions — `NAP-<WORD>.md` |
| [projections/](projections/) | The seam mapped to a host (e.g. [web](projections/web.md)) |
| [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) | The web projection — **normative, living, upstream**. Link it; never mirror it here |
| [ARCHETYPES.md](ARCHETYPES.md) + [naat/](naat/) | Archetype roles |
| `*-TEMPLATE.md` | Start here for a new spec |

## Goal — never compromise

**Interoperability through one clear seam.** Four roles, one verb each:

| napplet | shell | runtime | user |
|---------|-------|---------|------|
| **designs** | **mediates** | **executes** | **composes** |

Every change must sharpen this seam. A change that blurs who-does-what is wrong,
however useful it seems.

## Boundaries — exacting

| Kind | Is | Owns | Discovery |
|------|-----|------|-----------|
| **NAP-WORD** | runtime-provided | an **API surface** | `shell.supports("<domain>")` |
| **Convention** | napplet-agreed | **message semantics** | queryless `napplet:<archetype>/<intent>` identity in handler metadata |
| **NAAT** | a **role name + boundary** | nothing (not a NAP); may advertise conventions | manifest `["archetype", …]` |
| **Projection** | a host binding | how the seam maps to a host — **contracts are unchanged** | — |

- A NAP is **runtime-provided AND an API** (NAP-WORD). Napplet-agreed message
  semantics are conventions, not NAPs.
- Transport and host detail (`postMessage`, iframes, `window.napplet.*`) live in a
  **projection**, never in a NAP.
- Manifests reference bare **domains** (`relay`), never spec ids (`NAP-RELAY`).
- Cannot place a change cleanly on one side? **Stop and surface it.** Do not guess.

### Convention URI invariant

Developers invoke a convention as
`napplet:<archetype>/<intent>[...?params]`. The queryless path is the stable
convention identity. Handler metadata and subscriptions MUST use that stable
identity.

The query is shallow payload sugar. A runtime-provided binding MUST transpose
each unique, percent-decoded `name=value` pair into a text payload field before
routing or handler resolution. It MUST NOT coerce scalar types. `+` is a literal
plus sign. Fragments, malformed percent-encoding, repeated names, and mixing
query parameters with an explicit payload are invalid. Structured or non-text
data uses the explicit payload.

Transposition precedes routing. Routers match the resulting stable identity by
exact equality. They MUST NOT parse, normalize, prefix-match, or wildcard-match
convention identities.

### Two leaks to catch on sight

These recur. When you touch a spec, grep for them and rip them out — they are
always wrong, never "useful for now."

- **A deferred spec has zero live surface.** `Deferred` means dormant: the spec
  keeps **only** its italic registry row. It MUST NOT be wired into any active
  spec — no field, no wire-payload key, no parameter, no prose that treats it as
  real. A mandatory NAP (e.g. NAP-SHELL) carrying a `class` field for the
  deferred NAP-CLASS track is exactly this leak. **Test:** `grep -ri <domain>`
  across `naps/`, `projections/`, `README.md` should hit only the deferred row.
  Anything else, delete it.
- **A living external document is linked, never mirrored.** Upstream NIPs (NIP-5D,
  NIP-5A, …) are the source of truth and they move. Copying one into this repo
  forks it: the copy drifts, then silently contradicts its source. There MUST be
  **no local mirror file** of an external spec — reference the upstream link
  directly. **Test:** no repo file reproduces a NIP's body; every NIP mention is a
  link to its canonical/PR URL.

## Dependencies — declare, don't imply

NAPs legitimately rest on other NAPs: a miner publishes through `relay`; an
identity surface points its byte fields at `resource`. These edges are fine — but
they MUST be **declared**, never left implicit in prose. Every NAP names its
dependencies in a **`Depends:`** preamble block (alongside `NAP ID` / `Domain`),
**by domain** — lined up with `shell.supports("<domain>")` and the manifest
`["requires", …]` tag, never a bare spec id buried mid-paragraph. Each entry
carries a **kind** and a **strength**:

    **Depends:**
    - `resource` — wire · optional — `relay.event.resources?` carries
      `ResourceSidecarEntry`, a type owned by the `resource` domain.
    - `relay` — capability · required — publishes mined events via `relay.publish`.

**Kind** — what the edge couples:

| Kind | Means | Obligation |
|------|-------|------------|
| **wire** | this NAP's wire format embeds a type **owned by** another domain | one **owner** defines the type; **importers** reference it and MUST NOT redefine it; both specs cross-link |
| **capability** | this NAP's behavior calls another domain's surface (`resource.bytes`, `relay.publish`) — no shared type | name the method; the other domain owns its own contract |
| **layering** | shell-internal composition, invisible to the napplet (`outbox` builds on `relay`) | declared for transparency; imposes nothing on the napplet |

**Strength** — `required` (the dep domain MUST be present; a napplet SHOULD gate
on `shell.supports("<dep>")`) or `optional` (only some features need it; degrade
gracefully when absent).

**The owner/importer rule kills wire entanglement.** A `wire` dependency is
**directional**: exactly one domain owns the shared type, every other spec imports
it by reference. Two specs that *mutually amend* each other are a bug — collapse
them to one owner + one importer (`resource` owns `ResourceSidecarEntry`; `relay`
imports it — not the reverse, never both).

A dependency never moves the boundary. A NAP still owns only its own surface;
`Depends:` documents the edge, it does not license reaching across the seam.

## Terminology

Use [glossary](README.md#glossary) terms verbatim — *seam, napplet, runtime/shell,
NAP, domain, projection, NAP-WORD, convention, NAAT*. No synonyms, no new coinages. To
rename: change the glossary first, then every use, in one commit.

## Voice

Terse. Exacting. Reader-comprehension first.

- Short declarative sentences. No filler, hedging, or marketing.
- Define an uncommon term on first use, or link the glossary.
- Specs use RFC-2119 **MUST / SHOULD / MAY** — and mean them.
- A table beats a paragraph when it reads faster.

## Interface schema format

NAP specs are language-neutral contracts. Do not express normative surfaces as
TypeScript, Rust, Go, or any other implementation language.

Use this format instead:

- **Operations table** for the napplet-facing API: operation, parameters,
  result, and corresponding wire messages.
- **Schema tables** for records and enums: field, required, type, and notes.
- Use language-neutral types: `text`, `boolean`, `integer`, `number`, `null`,
  `any`, `list of T`, `map of text to T`, and quoted string alternatives.
- Use record names like `IntentRequest` or `Theme`, but do not use `interface`,
  `type`, `Promise`, `readonly`, `unknown`, or language-specific collection
  syntax.
- If a projection exposes an idiomatic SDK shape, describe it as projection
  guidance only. The NAP contract remains the operations table plus schemas.

## Operation naming

Domain repetition is a code smell. A wire type is `domain.action`; the action MUST
add meaning beyond the domain name.

- Avoid tautologies like `count.count`, `relay.relay`, `storage.storage`, or
  `<domain>.<domain>`.
- Prefer the operation the napplet is asking the runtime to perform: `query`,
  `publish`, `open`, `get`, `set`, `subscribe`, `info`, `check`.
- If the best verb equals the domain, the domain or operation is probably wrong.
  Stop and rename one side before the smell enters a spec.

## Isolation

One concern per branch, commit, and PR. Never tangle.

- **Specs** (`naps/`, `naat/<slug>.md`) → branch + PR, **one spec per PR**.
- **Governance / meta** (README.md, ARCHETYPES.md, `*-TEMPLATE.md`, this file) →
  committed **directly to `master`**.
- Never mix a spec with a README rework, or two specs in one branch.
- `.planning/` is local-only — never push.
- Before claiming done: `git diff origin/master...<branch>` must show **only** the
  intended files.

## Keep adjacent docs in sync — in the same commit

| Change | Also update |
|--------|-------------|
| a NAP spec | README registry row + status + **Deps column** |
| a NAP's `Depends:` block | README **Deps column** for that row |
| a **wire** dependency | both specs — owner defines the type, importer references it (never redefines) — plus each `Depends:` block |
| an archetype | ARCHETYPES.md · `naat/<slug>.md` · README |
| projection semantics | `projections/<host>.md` · README Projections table |
| terminology | README glossary, then all references |

## Changelog discipline

Every NAP spec carries its history at the bottom under `## Changelog`.

- Add one bullet per semantic commit-change: ``- `<short-sha>` - <summary>``.
- Summarize all semantic changes from one commit in one bullet, even when the
  commit touched multiple fields or operations.
- Include only changes that affect the spec contract, boundary, dependencies,
  operation names, wire shape, policy, or normative guidance.
- Skip formatting-only, table/schema-notation-only, spelling-only,
  changelog-only, and registry-pointer-only commits.
- For a living PR NAP, keep the bottom of the PR body in the same `## Changelog`
  format with the same bullets as the spec.
- For a merged NAP edited in a later PR, append changelog bullets to the spec in
  the same commit that changes the spec; mirror them in the PR body when the
  change is under review.

## PR format — identical every time

- **Branch:** kebab of the spec — `nap-relay`, `nap-storage`.
- **Title:** `NAP-<NAME>: <imperative one-line summary>`
- **Body**, these sections, in order:

      ## Summary      one paragraph — what changed and why
      ## Changes      specific bullets
      ## Downstream   boundary / doc-sync impact (omit if none)

  State `draft`; give the namespace (`window.napplet.<domain>`); link NIP-5D.

## Surface threats first

If a change risks interoperability or erodes the seam, say so **loudly and first**,
in plain words, in the PR or an issue. Never hide, defer, or quietly work around an
existential threat to the goal.

## Governance

NIP-style informal. Fork → add a spec under `naps/` from its template → open a PR.
dskvr merges when it makes sense and has ≥1 implementation. NAP-WORD names and NAAT
slugs are first-come, maintainer-approved. Numbered NAPs are not assigned.
