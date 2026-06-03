NAP: Nostr Applet Protocol
===============================

NAPs extend [NIP-5D](../NIP-5D.md) with interface and protocol specifications
for the napplet ecosystem. The core NIP defines transport, authentication, and
security. Everything else -- relay access, storage, signing, IPC, pipes, message
protocols -- is a NAP.

## Two Tracks

### NAP-WORD (Interface Specs)

Named by a single uppercase word. One canonical spec per name. Defines
shell-provided API contracts -- what shows up on `window.napplet.*` or
`window.nostr` or `window.nostrdb`. The shell implements these; napplets
consume them. Discovery: `shell.supports("NAP-RELAY")`.

| NAP ID | Namespace | Description | Status |
|--------|-----------|-------------|--------|
| [NAP-RELAY](https://github.com/napplet/naps/pull/2) | `window.napplet.relay` | NIP-01 relay proxy | Draft |
| [NAP-STORAGE](https://github.com/napplet/naps/pull/3) | `window.napplet.storage` | Scoped key-value storage | Draft |
| [NAP-SIGNER](https://github.com/napplet/naps/pull/1) | `window.nostr` | NIP-07 signer proxy | Draft |
| [NAP-NOSTRDB](https://github.com/napplet/naps/pull/4) | `window.nostrdb` | Local event database | Draft |
| [NAP-IFC](https://github.com/napplet/naps/pull/5) | `window.napplet.ifc` | Inter-frame communication | Draft |
| [NAP-PIPES](https://github.com/napplet/naps/pull/6) | `window.napplet.pipes` | Authenticated point-to-point connections | Draft |

### NAP-NN (Message Protocol Specs)

Numbered sequentially (NAP-01, NAP-02, etc.). Multiple competing specs allowed
per domain. Defines event semantics -- what napplets agree on with each other.
Napplets negotiate via `shell.supports("NAP-RELAY", "NAP-02")`. Example domains:
feed rendering, chat, collaborative editing.

## Boundary Rule

An interface (NAP-WORD) is **shell-provided** AND defines an **API surface**. A
protocol (NAP-NN) is **napplet-agreed** AND defines **event semantics**. Both
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

- Interface proposals: Use [TEMPLATE-WORD.md](TEMPLATE-WORD.md)
- Protocol proposals: Use [TEMPLATE-NN.md](TEMPLATE-NN.md)

## References

- Core protocol: [NIP-5D](../NIP-5D.md)
- Reference implementation: [@napplet packages](https://github.com/sandwichfarm/napplet)
- Reference shell: [hyprgate](https://github.com/sandwichfarm/hyprgate)
