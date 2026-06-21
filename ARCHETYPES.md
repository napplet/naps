NAAT: Napplet Archetypes
========================

A **NAAT** (Nostr Applet ArcheType) is a shared *role* a napplet can fulfill — `note`, `feed`, `profile`, `pet`. It is the third axis of the napplet ecosystem, orthogonal to the two NAP tracks:

- **NAP-WORD** — shell-provided API surfaces (`window.napplet.*`). *What the runtime offers.*
- **NAP-N** — numbered, competing wire formats. *What napplets say to each other.*
- **NAAT** — canonical role identities. *What kind of napplet this is.*

A NAAT is deliberately **not** a NAP: it is neither an interface nor a wire protocol. It is a name and a boundary.

This file is the **registry** — the index of every archetype. Each archetype's prose lives in its own thin file under [`naat/`](naat/), following [`naat/TEMPLATE.md`](naat/TEMPLATE.md). The registry is the source of truth for which archetypes exist; every row links to its file.

## How a NAAT is used

1. A napplet **declares the roles it fulfills** in its NIP-5A manifest:
   ```
   ["archetype", "note", "NAP-4"]      // role slug, then the NAP-N wire format(s) it accepts for that role
   ```
   A napplet may declare several archetype tags. A napplet with **no** archetype tag is fully valid — it simply cannot be opened *by role*. "Weird" single-purpose napplets are first-class.

2. A napplet **opens another by role** via [NAP-INTENT](naps/NAP-INTENT.md):
   ```js
   if (napplet.shell.supports("intent")) {
     const { available } = await napplet.intent.available("pet");
     if (available) showButton();
   }
   napplet.intent.open("pet", { /* payload, if a handler advertises one */ });
   ```
   The runtime resolves the role to the user's **default** handler (like an OS "default app"), creates or focuses its window, and delivers the payload.

3. The **slug** (`note`) is the identifier used everywhere — in the manifest tag, in `intent.open(archetype)`, and in a NAP-N's `Serves:` field. The `NAAT-NOTE` id is a display/cross-reference label only, mirroring the `NAP-RELAY` / `relay` split.

## Archetype vs. wire format

A NAAT names a role and **recommends** one NAP-N as its default open contract — the answer to "what do I send to open this?" for the common case. It does **not** own the payload. New and richer wire formats are added as numbered NAP-N specs that declare `Serves: <slug>/<action>`; they self-register against the archetype **without editing the registry**. The recommendation is a convenience and an interop floor, not a mandate — callers can negotiate any protocol a handler advertises via `available()`.

## Entry schema

Each archetype is one thin file in [`naat/`](naat/) in a fixed shape, so roles stay comparable and the vocabulary stays disjoint:

- **Definition** — one sentence: what a napplet of this role does.
- **Boundary (IS / IS NOT)** — the scope line. This is the "precise but not too precise" knob, and the surface a maintainer reviews a new archetype against.
- **Distinct from** — sibling roles it is most likely confused with. This is the anti-drift mechanism that prevents `note` / `note-view` / `event-viewer` from all appearing over time.
- **Actions** — the verbs it supports (`open` by default).

**Invariant:** a NAAT file is the fixed schema and nothing else. Anything that wants to grow — payload detail, lifecycle, action-specific wire — overflows into a NAP-N spec, never into the archetype file. An entry is structurally incapable of becoming a spec.

## Governance

Same informal process as the NAP tracks: open a PR that adds a row to the registry below **and** the matching `naat/<slug>.md` file. Slugs are first-come-first-served and must be approved by the maintainer (dskvr). A proposal is judged on whether its **Boundary** is genuinely disjoint from existing archetypes. Status is `draft` until at least one public implementation exists.

## Registry

| NAAT ID | Slug | Definition | Recommended open contract | Status |
|---------|------|------------|---------------------------|--------|
| [NAAT-NOTE](naat/note.md) | `note` | Displays a single Nostr event in focused detail | NAP-4 (`note:open`) | Draft |
| [NAAT-PROFILE](naat/profile.md) | `profile` | Displays a user's profile and activity | NAP-1 (`profile:open`) | Draft |
| [NAAT-DM](naat/dm.md) | `dm` | Opens a direct-message conversation with a person | NAP-3 (`chat:open-dm`) | Draft |
| [NAAT-FEED](naat/feed.md) | `feed` | A scrolling list of many events by some criteria | *(none yet)* | Draft |
| [NAAT-FEED-MANAGER](naat/feed-manager.md) | `feed-manager` | Creates, edits, saves, imports, exports, or organizes feed definitions | *(none yet)* | Draft |
| [NAAT-COMPOSER](naat/composer.md) | `composer` | Creates and publishes a new Nostr event | *(none yet)* | Draft |
| [NAAT-PET](naat/pet.md) | `pet` | Presents and manages a virtual pet or companion | *(none yet)* | Draft |
