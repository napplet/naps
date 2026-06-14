NAP: Nostr Applet Protocol
===============================

NAPs extend [NIP-5D](../NIP-5D.md) with interface and protocol specifications
for the napplet ecosystem. The core NIP defines transport, identity, and
security. Everything else -- relay access, storage, identity queries, IFC,
message protocols -- is a NAP.

The ecosystem has three orthogonal axes: **NAP-WORD** interfaces (what the runtime
offers), **NAP-N** wire formats (what napplets say to each other), and **NAAT**
archetypes (what kind of napplet this is). The first two are NAPs; the third is a
role registry, documented in [ARCHETYPES.md](ARCHETYPES.md).

## Tracks

### NAP-WORD (Interface Specs)

Named by a single uppercase word. One canonical spec per name. Defines
shell-provided API contracts, what shows up on `window.napplet.*` or
`window.nostrdb`. The shell implements these; napplets consume them.
Discovery: `shell.supports("relay")`.

| NAP ID | Namespace | Description | Status |
|--------|-----------|-------------|--------|
| [NAP-RELAY](https://github.com/napplet/naps/pull/2) | `window.napplet.relay` | Relay proxy (subscribe, publish, query, publishEncrypted) | Draft |
| [NAP-IDENTITY](https://github.com/napplet/naps/pull/12) | `window.napplet.identity` | Read-only user identity queries | Draft |
| [NAP-STORAGE](https://github.com/napplet/naps/pull/3) | `window.napplet.storage` | Scoped key-value storage | Draft |
| [NAP-IFC](https://github.com/napplet/naps/pull/5) | `window.napplet.ifc` | Inter-frame communication | Draft |
| [NAP-THEME](https://github.com/napplet/naps/pull/8) | `window.napplet.theme` | Shell-provided theming | Draft |
| [NAP-KEYS](https://github.com/napplet/naps/pull/9) | `window.napplet.keys` | Keyboard forwarding and action keybindings | Draft |
| [NAP-MEDIA](https://github.com/napplet/naps/pull/10) | `window.napplet.media` | Media session control and playback | Draft |
| [NAP-NOTIFY](https://github.com/napplet/naps/pull/11) | `window.napplet.notify` | Shell-rendered notifications | Draft |
| [NAP-INTENT](NAP-INTENT.md) | `window.napplet.intent` | Invoke a napplet by archetype (default-handler dispatch) | Draft |

### NAP-N (Message Protocol Specs)

Numbered sequentially (NAP-1, NAP-2, etc.). Multiple competing specs allowed
per domain. Defines event semantics - what napplets agree on with each other.
Napplets negotiate via `shell.supports("relay", "NAP-2")`. Example domains:
feed rendering, chat, collaborative editing. A NAP-N that shapes an archetype
open payload declares `Serves: <slug>/<action>` so it self-registers against a
[NAAT](ARCHETYPES.md) without editing the registry.

### NAAT (Archetypes)

Canonical napplet *roles* — `note`, `feed`, `profile`, `emoji-list`. Not a NAP:
neither an interface nor a wire format, just a name and a boundary. Archetypes
are rows in the [ARCHETYPES.md](ARCHETYPES.md) registry, each linking to a thin
file under [`naat/`](naat/). A napplet declares the roles it fulfills with a
`["archetype", "<slug>", "<NAP-N>"]` manifest tag, and napplets invoke each other
by role through [NAP-INTENT](NAP-INTENT.md). Slugs are first-come-first-served.

## Boundary Rule

An interface (NAP-WORD) is **shell-provided** AND defines an **API surface**. A
protocol (NAP-N) is **napplet-agreed** AND defines **message semantics**. An
archetype (NAAT) is a **canonical role name** with a **boundary**, owning neither
an API nor a payload — it only *recommends* a NAP-N as its default open contract.
Both NAP criteria must apply for a NAP; edge cases are judged pragmatically by the
maintainer.

## Governance

NIP-style informal process:

- Fork this repo, add a markdown file following the appropriate template, open a
  PR.
- Community discusses via PR comments.
- Maintainer (dskvr) merges when the spec makes sense and has at least one
  implementation.
- No formal stages, review committees, or voting.
- NAP-WORD names are first-come-first-served but must be approved by the
  maintainer.
- NAP-N numbers are assigned sequentially on merge.

## Templates

- Interface proposals: Use [NAP-WORD-TEMPLATE.md](NAP-WORD-TEMPLATE.md)
- Protocol proposals: Use [NAP-N-TEMPLATE.md](NAP-N-TEMPLATE.md)
- Archetype proposals: Use [naat/TEMPLATE.md](naat/TEMPLATE.md) and add a row to [ARCHETYPES.md](ARCHETYPES.md)

## References

- Core protocol: [NIP-5D](../NIP-5D.md)
