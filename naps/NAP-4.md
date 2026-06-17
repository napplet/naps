NAP-4
======

Note Viewer Open Protocol
-------------------------

`draft`

**NAP ID:** NAP-4
**Domain:** note viewing / cross-napplet navigation
**Depends:**
- `inc` — capability · required — rides NAP-INC transport (`inc.subscribe` / `inc.event`)
- `relay` — capability · required — consumers retrieve the referenced note via `relay`
**Discovery:** `shell.supports("inc", "NAP-4")`

## Description

NAP-4 defines an inter-napplet message for asking a shell or peer napplet to
open a specific Nostr event in a note viewer. It standardizes the target
reference shape, relay hints, and compatibility behavior so feed, profile,
search, notification, and thread napplets can hand off note/event inspection to
a dedicated viewer without inventing per-client topic payloads.

## Message Protocol

Napplets coordinate using NAP-INC messages. The producer emits a `note:open`
topic with a structured payload. Consumers MAY be a shell-owned note viewer, a
third-party note viewer napplet, or a shell router that opens/focuses an
appropriate viewer.

### `note:open`

Producer intent: open one target Nostr event or addressable event for detailed
viewing.

Transport:

```ts
inc.emit("note:open", payload)
```

Payload:

```ts
type NoteOpenPayload = {
  target: NoteOpenTarget;
  relays?: string[];
  source?: {
    napplet?: string;
    windowId?: string;
    requestId?: string;
  };
  behavior?: {
    focus?: boolean;
    newWindow?: boolean;
  };
};

type NoteOpenTarget =
  | {
      type: "event";
      id: string;        // 32-byte lowercase hex event id
      kind?: number;
      pubkey?: string;   // lowercase hex pubkey when known
      nip19?: string;    // note1/nevent1 reference when available
    }
  | {
      type: "address";
      kind: number;
      pubkey: string;    // lowercase hex pubkey
      identifier: string;
      nip19?: string;    // naddr1 reference when available
    };
```

Requirements:

- `target.type` MUST be either `event` or `address`.
- `event.id` MUST be a 64-character lowercase hex event id.
- `address.pubkey` MUST be a 64-character lowercase hex pubkey.
- `relays` are advisory relay hints. Consumers MAY use them to prioritize relay
  selection, but MUST NOT require them.
- `kind` and `pubkey` on event targets are optional metadata used to reduce
  lookup ambiguity and improve UI before the full event is fetched.
- `nip19` is optional compatibility metadata. Consumers MUST parse structured
  fields first and MUST NOT require `nip19`.
- `behavior.focus` defaults to `true`.
- `behavior.newWindow` defaults to shell policy. Producers SHOULD leave it
  unset unless the user explicitly requested a new viewer.

## Compatibility

Consumers that do not support NAP-4 MUST ignore the message. Producers SHOULD
check `shell.supports("inc", "NAP-4")` before relying on the protocol. If the
check is unavailable or false, producers MAY fall back to local navigation or a
legacy client-specific topic.

Consumers SHOULD accept `note1`, `nevent1`, and `naddr1` values when provided
as `target.nip19`, but the canonical wire shape is the structured `target`
object above.

## Security Considerations

The message is a navigation request, not a relay access grant. Consumers MUST
still retrieve events through their own authorized NAP-RELAY surface and shell
policy. Relay hints MUST be treated as untrusted input. Consumers MUST validate
event ids, pubkeys, kinds, and address identifiers before using them in relay
filters.

## Non-Goals

- Defining reply, reaction, repost, or quote rendering.
- Defining shell window-manager APIs.
- Defining relay selection policy.
- Defining a mandatory note viewer implementation.

## Implementations

- Hyprgate Note Viewer napplet milestone v2.8 draft implementation:
  `napps/note-viewer` and `packages/utils/src/note-viewer-protocol.ts`
