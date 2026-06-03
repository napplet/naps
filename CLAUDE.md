# CLAUDE.md

## What This Is

**NAPs** (Nostr Applet Protocol) — the extension proposal system for the napplet protocol. This repo hosts interface and message protocol specs that extend [NIP-5D](https://github.com/nostr-protocol/nips/blob/master/5D.md) (Nostr Web Applets).

NIP-5D defines the core: transport (`postMessage`), authentication (`REGISTER` → `IDENTITY` → `AUTH`), extension discovery (`shell.supports()`), and security model. Everything else — relay proxy, storage, signing, IFC, pipes, message protocols — is a NAP.

**Remote:** `git@github.com:napplet/naps.git`

## Repo Structure

```
README.md           — Governance doc, dual-track overview, interface registry table
NAP-WORD-TEMPLATE.md — Template for interface proposals (NAP-WORD)
NAP-NN-TEMPLATE.md   — Template for message protocol proposals (NAP-NN)
```

### What gets committed directly

- `README.md` — governance, registry table, track descriptions
- `NAP-WORD-TEMPLATE.md` — interface proposal template
- `NAP-NN-TEMPLATE.md` — message protocol proposal template

### What gets submitted as PRs

- `NAP-RELAY.md`, `NAP-STORAGE.md`, `NAP-IFC.md`, etc. — interface specs are **opened as PRs**, not committed to `master` directly. They follow the NIP-style informal process: fork, add spec, open PR, community reviews, maintainer merges.
- `NAP-01.md`, `NAP-02.md`, etc. — message protocol specs, same PR process.

## Two Tracks

### NAP-WORD (Interface Specs)

- Named by a single uppercase word: `NAP-RELAY`, `NAP-STORAGE`, `NAP-NOSTRDB`, `NAP-IFC`, `NAP-PIPES`
- One canonical spec per name — no competing interface specs
- Defines shell-provided API contracts on `window.napplet.*` namespaces
- Discovery: `shell.supports("NAP-RELAY")`
- Use `NAP-WORD-TEMPLATE.md` as the starting point

### NAP-NN (Message Protocol Specs)

- Numbered: `NAP-01`, `NAP-02`, etc.
- Multiple competing specs allowed per domain (e.g., two different feed protocols)
- Defines event semantics napplets agree on with each other
- Napplets negotiate via `shell.supports("NAP-RELAY", "NAP-02")`
- Use `NAP-NN-TEMPLATE.md` as the starting point

## Governance

NIP-style informal. Fork repo, add markdown spec, open PR. Community comments on PRs. Maintainer (dskvr) merges when it makes sense. No formal stages or review committee.

## Preparing PRs

Each NAP spec should be opened as its own PR:

1. Create a branch: `nap-relay`, `nap-storage`, etc.
2. Add the spec file: `NAP-RELAY.md`
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
