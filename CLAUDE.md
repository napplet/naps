# CLAUDE.md

## What This Is

**NAPs** (Nostr Applet Protocol) — the proposal system for the napplet **capability seam**. A NAP is a transport-agnostic contract for what a runtime provides to a napplet and how the napplet requests it. [NIP-5D](https://github.com/nostr-protocol/nips/blob/master/5D.md) (Nostr Web Applets) is the seam's **web binding**, not what NAPs extend — the same contracts can be implemented by other runtimes (native, WASM, …).

NIP-5D *realizes* the seam for the web: the carrier (`postMessage`), the iframe sandbox model, napplet identity assigned at iframe creation from the NIP-5A manifest, capability discovery (`shell.supports()`), and the security/mediation model. The capabilities themselves — relay proxy, storage, signing, IFC, intents, message protocols — are NAPs, defined independent of any single binding. Napplets are Nostr-native: the seam is transport-agnostic, not Nostr-agnostic.

**Remote:** `git@github.com:napplet/naps.git`

## Repo Structure

```
README.md           — Governance doc, dual-track overview, interface registry table
ARCHETYPES.md       — NAAT archetype registry
NAP-WORD-TEMPLATE.md — Template for interface proposals (NAP-WORD)
NAP-N-TEMPLATE.md   — Template for message protocol proposals (NAP-N)
naps/               — All NAP spec definitions (NAP-WORD interfaces + NAP-N wire formats)
naps/NAP-INTENT.md  — e.g. the archetype intent dispatcher interface
naat/               — Per-archetype thin files (NAAT track)
```

All NAP spec definitions live under `naps/`. The templates and the registries
(`README.md`, `ARCHETYPES.md`) stay at the repo root.

### What gets committed directly

- `README.md` — governance, registry table, track descriptions
- `NAP-WORD-TEMPLATE.md` — interface proposal template
- `NAP-N-TEMPLATE.md` — message protocol proposal template

### What gets submitted as PRs

- `naps/NAP-RELAY.md`, `naps/NAP-STORAGE.md`, `naps/NAP-INC.md`, etc. — interface specs are **opened as PRs**, not committed to `master` directly. They follow the NIP-style informal process: fork, add spec under `naps/`, open PR, community reviews, maintainer merges.
- `naps/NAP-1.md`, `naps/NAP-2.md`, etc. — message protocol specs, same PR process.

## Two Tracks

### NAP-WORD (Interface Specs)

- Named by a single uppercase word: `NAP-RELAY`, `NAP-STORAGE`, `NAP-NOSTRDB`, `NAP-IFC`, `NAP-PIPES`
- One canonical spec per name — no competing interface specs
- Defines shell-provided API contracts on `window.napplet.*` namespaces
- Discovery: `shell.supports("NAP-RELAY")`
- Use `NAP-WORD-TEMPLATE.md` as the starting point

### NAP-N (Message Protocol Specs)

- Numbered: `NAP-1`, `NAP-2`, etc.
- Multiple competing specs allowed per domain (e.g., two different feed protocols)
- Defines event semantics napplets agree on with each other
- Napplets negotiate via `shell.supports("NAP-RELAY", "NAP-2")`
- Use `NAP-N-TEMPLATE.md` as the starting point

## Governance

NIP-style informal. Fork repo, add markdown spec, open PR. Community comments on PRs. Maintainer (dskvr) merges when it makes sense. No formal stages or review committee.

## Preparing PRs

Each NAP spec should be opened as its own PR:

1. Create a branch: `nap-relay`, `nap-storage`, etc.
2. Add the spec file under `naps/`: `naps/NAP-RELAY.md`
3. Update the registry table in `README.md` (add the row with link to the spec)
4. Open PR with title: `NAP-RELAY: NIP-01 relay proxy interface`
5. PR body should include: one-line summary, namespace (`window.napplet.relay`), status (`draft`), and link to NIP-5D

The 6 initial interface specs to submit as PRs (source files in `~/Develop/napplet/specs/naps/`):

| Spec | Source | Branch |
|------|--------|--------|
| NAP-RELAY | `~/Develop/napplet/specs/naps/NAP-RELAY.md` | `nap-relay` |
| NAP-STORAGE | `~/Develop/napplet/specs/naps/NAP-STORAGE.md` | `nap-storage` |
| NAP-NOSTRDB | `~/Develop/napplet/specs/naps/NAP-NOSTRDB.md` | `nap-nostrdb` |
| NAP-IFC | `~/Develop/napplet/specs/naps/NAP-IFC.md` | `nap-ifc` |
| NAP-PIPES | `~/Develop/napplet/specs/naps/NAP-PIPES.md` | `nap-pipes` |

## Checklist: When Modifying a NAP

Every time you create or modify a NAP spec, you MUST:

1. **Update README.md registry table** — add new NAPs, update descriptions for modified ones
2. **Update the PR body** — if the NAP already has an open PR, update its body to reflect changes (use `gh pr edit <number> --body`)
3. **No private references** — NEVER mention `@napplet/*` packages, the napplet/napplet repo, or any private implementation in specs, commits, or PR bodies. This repo is PUBLIC.
4. **Implementations section** — always `- (none yet)` until a public implementation exists
5. **Commit messages** — describe the protocol change only, never reference private packages

## Related Repos

- [`nostr-protocol/nips`](https://github.com/nostr-protocol/nips) — NIP-5D lives here
