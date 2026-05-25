NUB: Napplet Unified Blueprints
===============================

NUBs extend [NIP-5D](../NIP-5D.md) with interface and protocol specifications
for the napplet ecosystem. The core NIP defines transport, identity, and
security. Everything else -- relay access, storage, identity queries, IFC,
message protocols -- is a NUB.

## Two Tracks

### NUB-WORD (Interface Specs)

Named by a single uppercase word. One canonical spec per name. Defines
shell-provided API contracts, what shows up on `window.napplet.*` or
`window.nostrdb`. The shell implements these; napplets consume them.
Discovery: `shell.supports("relay")`.

| NUB ID | Namespace | Description | Status |
|--------|-----------|-------------|--------|
| [NUB-RELAY](https://github.com/napplet/nubs/pull/2) | `window.napplet.relay` | Relay proxy (subscribe, publish, query, publishEncrypted) | Draft |
| [NUB-IDENTITY](https://github.com/napplet/nubs/pull/12) | `window.napplet.identity` | Read-only user identity queries | Draft |
| [NUB-STORAGE](https://github.com/napplet/nubs/pull/3) | `window.napplet.storage` | Scoped key-value storage | Draft |
| [NUB-IFC](https://github.com/napplet/nubs/pull/5) | `window.napplet.ifc` | Inter-frame communication | Draft |
| [NUB-THEME](https://github.com/napplet/nubs/pull/8) | `window.napplet.theme` | Shell-provided theming | Draft |
| [NUB-KEYS](https://github.com/napplet/nubs/pull/9) | `window.napplet.keys` | Keyboard forwarding and action keybindings | Draft |
| [NUB-MEDIA](https://github.com/napplet/nubs/pull/10) | `window.napplet.media` | Media session control and playback | Draft |
| [NUB-NOTIFY](https://github.com/napplet/nubs/pull/11) | `window.napplet.notify` | Shell-rendered notifications | Draft |

### NUB-NN (Message Protocol Specs)

Numbered sequentially (NUB-01, NUB-02, etc.). Multiple competing specs allowed
per domain. Defines event semantics - what napplets agree on with each other.
Napplets negotiate via `shell.supports("relay", "NUB-02")`. Example domains:
feed rendering, chat, collaborative editing.

| NUB ID | Domain | Description | Status |
|--------|--------|-------------|--------|
| [NUB-02](NUB-02.md) | chat navigation | `chat:open-dm` topic payload and behavior over NUB-IFC | Draft |

## Boundary Rule

An interface (NUB-WORD) is **shell-provided** AND defines an **API surface**. A
protocol (NUB-NN) is **napplet-agreed** AND defines **message semantics**. Both
criteria must apply. Edge cases are judged pragmatically by the maintainer.

## Governance

NIP-style informal process:

- Fork this repo, add a markdown file following the appropriate template, open a
  PR.
- Community discusses via PR comments.
- Maintainer (dskvr) merges when the spec makes sense and has at least one
  implementation.
- No formal stages, review committees, or voting.
- NUB-WORD names are first-come-first-served but must be approved by the
  maintainer.
- NUB-NN numbers are assigned sequentially on merge.

## Templates

- Interface proposals: Use [TEMPLATE-WORD.md](TEMPLATE-WORD.md)
- Protocol proposals: Use [TEMPLATE-NN.md](TEMPLATE-NN.md)

## References

- Core protocol: [NIP-5D](../NIP-5D.md)
