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
| [NAP-RELAY](NAP-RELAY.md) | `window.napplet.relay` | NIP-01 relay proxy | Draft |
| [NAP-STORAGE](naps/NAP-STORAGE.md) | `window.napplet.storage` | Scoped key-value storage | Draft |
| [NAP-SIGNER](NAP-SIGNER.md) | `window.nostr` | NIP-07 signer proxy | Draft |
| [NAP-NOSTRDB](NAP-NOSTRDB.md) | `window.nostrdb` | Local event database | Draft |
| [NAP-IPC](NAP-IPC.md) | `window.napplet.ipc` | Inter-napplet pub/sub | Draft |
| [NAP-PIPES](NAP-PIPES.md) | `window.napplet.pipes` | Authenticated point-to-point connections | Draft |

### NAP-N (Message Protocol Specs)

Numbered sequentially (NAP-1, NAP-2, etc.). Multiple competing specs allowed
per domain. Defines event semantics -- what napplets agree on with each other.
Napplets negotiate via `shell.supports("NAP-RELAY", "NAP-2")`. Example domains:
feed rendering, chat, collaborative editing.

## Boundary Rule

An interface (NAP-WORD) is **shell-provided** AND defines an **API surface**. A
protocol (NAP-N) is **napplet-agreed** AND defines **event semantics**. Both
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
- NAP-N numbers are assigned sequentially on merge.

## Templates

- Interface proposals: Use [TEMPLATE-WORD.md](TEMPLATE-WORD.md)
- Protocol proposals: Use [TEMPLATE-NN.md](TEMPLATE-NN.md)

## References

- Core protocol: [NIP-5D](../NIP-5D.md)
- Reference implementation: [@napplet packages](https://github.com/sandwichfarm/napplet)
- Reference shell: [hyprgate](https://github.com/sandwichfarm/hyprgate)
