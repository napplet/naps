# AGENTS.md

**`napplet/naps`** — registry and governance for **NAPs** (Nostr Applet Protocol):
the capability **seam** between a napplet and its runtime. **This repository is public.**

This file is the entry point and the source of truth for working here. Read it,
then fan out via the links below. `CLAUDE.md` is a symlink to this file.

## Map

| Path | What |
|------|------|
| [README.md](README.md) | Concepts, **glossary (canonical terms)**, NAP-WORD / NAP-N registries |
| [naps/](naps/) | All spec definitions — `NAP-<WORD>.md` interfaces, `NAP-<N>.md` wire formats |
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
| **NAP-N** | napplet-agreed | **message semantics** | `shell.supports("<domain>", "NAP-N")` |
| **NAAT** | a **role name + boundary** | nothing (not a NAP); *recommends* a NAP-N | manifest `["archetype", …]` |
| **Projection** | a host binding | how the seam maps to a host — **contracts are unchanged** | — |

- A NAP is **runtime-provided AND an API** (NAP-WORD) **or** **napplet-agreed AND
  message semantics** (NAP-N). Never both; never neither.
- Transport and host detail (`postMessage`, iframes, `window.napplet.*`) live in a
  **projection**, never in a NAP.
- Manifests reference bare **domains** (`relay`), never spec ids (`NAP-RELAY`).
- Cannot place a change cleanly on one side? **Stop and surface it.** Do not guess.

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

## Terminology

Use [glossary](README.md#glossary) terms verbatim — *seam, napplet, runtime/shell,
NAP, domain, projection, NAP-WORD, NAP-N, NAAT*. No synonyms, no new coinages. To
rename: change the glossary first, then every use, in one commit.

## Voice

Terse. Exacting. Reader-comprehension first.

- Short declarative sentences. No filler, hedging, or marketing.
- Define an uncommon term on first use, or link the glossary.
- Specs use RFC-2119 **MUST / SHOULD / MAY** — and mean them.
- A table beats a paragraph when it reads faster.

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
| a NAP spec | README registry row + status |
| an archetype | ARCHETYPES.md · `naat/<slug>.md` · README |
| projection semantics | `projections/<host>.md` · README Projections table |
| terminology | README glossary, then all references |

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
slugs are first-come, maintainer-approved; NAP-N numbers are assigned on merge.
