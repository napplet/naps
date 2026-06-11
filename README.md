NAP: Nostr Applet Protocol
===============================

NAPs extend [NIP-5D](../NIP-5D.md) with interface and protocol specifications
for the napplet ecosystem. The core NIP defines transport, identity, and
security. Everything else -- relay access, storage, identity queries, IFC,
message protocols -- is a NAP.

## Two Tracks

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

### NAP-NN (Message Protocol Specs)

Numbered sequentially (NAP-01, NAP-02, etc.). Multiple competing specs allowed
per domain. Defines event semantics - what napplets agree on with each other.
Napplets negotiate via `shell.supports("relay", "NAP-02")`. Example domains:
feed rendering, chat, collaborative editing.

## Boundary Rule

An interface (NAP-WORD) is **shell-provided** AND defines an **API surface**. A
protocol (NAP-NN) is **napplet-agreed** AND defines **message semantics**. Both
criteria must apply. Edge cases are judged pragmatically by the maintainer.

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
- NAP-NN numbers are assigned sequentially on merge.

## Templates

- Interface proposals: Use [NAP-WORD-TEMPLATE.md](NAP-WORD-TEMPLATE.md)
- Protocol proposals: Use [NAP-NN-TEMPLATE.md](NAP-NN-TEMPLATE.md)

## References

- Core protocol: [NIP-5D](../NIP-5D.md)
