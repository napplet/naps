NAP-OPEN
========

Archetype Open Dispatcher
-------------------------

`draft`

**NAP ID:** NAP-OPEN
**Namespace:** `window.napplet.open`
**Discovery:** `shell.supports("open")`

## Description

NAP-OPEN provides napplets with a shell-mediated interface for opening *another* napplet by its **archetype** — a shared role name such as `note`, `profile`, or `emoji-list` (see [ARCHETYPES.md](ARCHETYPES.md)). A napplet describes *what role* it wants opened and *what payload* to deliver; the shell resolves the role to an installed napplet, applies the user's default-handler preference, creates or focuses the window, and delivers the payload. This is the napplet equivalent of an operating system's "default application" / URI-scheme handler: the caller never names a specific napplet, only the role.

NAP-OPEN standardizes the **envelope**, not the payload. The `payload` is opaque and tagged by a `protocol` field naming the numbered wire format (a NAP-N spec) that shapes it — exactly as an HTTP request body is opaque and tagged by `Content-Type`. This keeps two axes independent and free to evolve at their own speed:

- **archetype** — *routing*: which napplet should handle this, and whose default applies.
- **protocol** — *parsing*: what wire format the payload is in.

The two are orthogonal (N:M): one protocol may serve several archetypes, and one archetype may accept several protocols. Resolution of an archetype to a concrete napplet — including which napplets fulfill which roles and which is the user's default — is shell policy. Napplets never address each other directly.

## API Surface

```typescript
interface NappletOpen {
  open(request: OpenRequest): Promise<OpenResult>;            // via open.open
  available(archetype: string): Promise<OpenAvailability>;    // via open.available
  archetypes(): Promise<OpenAvailability[]>;                  // via open.archetypes
  onAvailabilityChanged(handler: (a: OpenAvailability) => void): Subscription;
}

interface OpenRequest {
  archetype: string;                 // role slug, e.g. "note" (see ARCHETYPES.md)
  action?: string;                   // verb, default "open" (e.g. "open" | "edit" | "pick" | "share")
  protocol?: string;                 // NAP-N id shaping `payload`; omit -> the archetype's recommended default
  payload?: unknown;                 // opaque, typed by `protocol`
  handler?: "default" | "choose" | string;  // user default | prompt "open with…" | a specific napplet dTag
  behavior?: {
    focus?: boolean;
    newWindow?: boolean;
    reuse?: boolean;
  };
}

interface OpenCandidate {
  dTag: string;                      // the napplet that can fulfill the archetype
  title?: string;
  protocols: string[];               // NAP-N ids this candidate accepts for the archetype
  isDefault?: boolean;
}

interface OpenAvailability {
  archetype: string;
  available: boolean;                // at least one installed napplet fulfills it
  candidates: OpenCandidate[];       // sourced from the manifest catalog, not running instances
  hasDefault: boolean;               // a user/runtime default is set for this archetype
}

interface OpenResult {
  ok: boolean;
  archetype: string;
  opened: boolean;
  handler?: string;                  // dTag of the napplet that handled it
  windowId?: string;
  protocol?: string;                 // the wire format actually used
  error?: string;
}
```

**`open(request)`** — Asks the shell to open a napplet of `archetype` and deliver `payload`. The shell resolves the archetype to a handler (the user's default, the napplet named in `handler`, or a user choice when `handler: "choose"`), creates or focuses its window, and delivers the payload using the named `protocol` (or the archetype's recommended default when `protocol` is omitted). Returns once the handler has been resolved and the window created; payload delivery to the handler MAY complete asynchronously.

**`available(archetype)`** — Returns whether the runtime can currently open `archetype`, the candidate napplets that fulfill it, and the protocols each accepts. This is the pre-flight guardrail: a caller checks availability before showing an "open" affordance, so a missing handler fails loudly at the call site instead of silently at delivery. Availability is sourced from the **installed-napplet catalog** (the manifests the runtime knows about), so it reports `true` for an installed handler that is not yet running.

**`archetypes()`** — Returns availability for every archetype the runtime can currently satisfy. Useful for menus and capability surfaces.

**`onAvailabilityChanged(handler)`** — Registers for shell-pushed availability updates, fired when a napplet is installed or removed, or a default handler changes.

## Wire Protocol

`open.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `open.open` | napplet -> shell | `id`, `request` |
| `open.open.result` | shell -> napplet | `id`, `result`, `error?` |
| `open.available` | napplet -> shell | `id`, `archetype` |
| `open.available.result` | shell -> napplet | `id`, `availability`, `error?` |
| `open.archetypes` | napplet -> shell | `id` |
| `open.archetypes.result` | shell -> napplet | `id`, `archetypes`, `error?` |
| `open.availability.changed` | shell -> napplet | `availability` |

Key design notes:
- Request/result pairs use `id` for correlation.
- `open.availability.changed` is a shell push message and has no `id`.
- The shell delivers `payload` to the resolved handler using the named `protocol`'s own delivery mechanism — typically a NAP-N IFC topic event (e.g., `note:open`), or initial state passed at instantiation for a cold-started handler. NAP-OPEN governs resolution, default handling, and window lifecycle; the NAP-N protocol governs the payload wire and its delivery.
- `protocol` and `archetype` are independent. The shell MUST NOT assume a one-to-one mapping between them.

### Examples

**Conditionally show an "add emoji list" button, then open it:**
```
-> { "type": "open.available", "id": "a1", "archetype": "emoji-list" }
<- {
     "type": "open.available.result",
     "id": "a1",
     "availability": {
       "archetype": "emoji-list",
       "available": true,
       "candidates": [
         { "dTag": "emojilistr", "title": "Emoji List Maker", "protocols": ["NAP-7"], "isDefault": true }
       ],
       "hasDefault": true
     }
   }
-> {
     "type": "open.open",
     "id": "o1",
     "request": {
       "archetype": "emoji-list",
       "action": "open",
       "payload": { "seed": ["🤙", "⚡"] },
       "behavior": { "focus": true }
     }
   }
<- {
     "type": "open.open.result",
     "id": "o1",
     "result": {
       "ok": true,
       "archetype": "emoji-list",
       "opened": true,
       "handler": "emojilistr",
       "windowId": "win-12",
       "protocol": "NAP-7"
     }
   }
```

**Open a note using a specific wire format (negotiated upgrade):**
```
-> {
     "type": "open.open",
     "id": "o2",
     "request": {
       "archetype": "note",
       "protocol": "NAP-4",
       "payload": { "target": { "type": "event", "id": "abc..." } }
     }
   }
<- { "type": "open.open.result", "id": "o2",
     "result": { "ok": true, "archetype": "note", "opened": true, "handler": "noteview", "windowId": "win-13", "protocol": "NAP-4" } }
```

**No handler installed:**
```
-> { "type": "open.open", "id": "o3", "request": { "archetype": "emoji-list", "payload": {} } }
<- { "type": "open.open.result", "id": "o3",
     "result": { "ok": false, "archetype": "emoji-list", "opened": false, "error": "no handler" } }
```

### Error Handling

Result messages MAY include `error` when the request cannot be fulfilled. Common errors include `"unknown archetype"`, `"no handler"`, `"unsupported protocol"` (the resolved handler does not accept the requested `protocol`), `"user cancelled"` (during an "open with…" prompt), and `"open failed"`.

The shell SHOULD return a structured `result` with `ok: false` and `opened: false` when resolution or delivery fails. A caller that wants to avoid the failure path SHOULD call `available()` first.

## Shell Behavior

- The shell MUST resolve an `archetype` to a handler using its catalog of installed napplets and the user's default-handler preference for that archetype.
- The shell MUST keep a user-overridable default per archetype. When a default exists, `open` without an explicit `handler` MUST route to it.
- The shell SHOULD offer an "open with…" chooser when `handler: "choose"`, or when no default exists and more than one candidate is available.
- The shell MUST source `available()` / `archetypes()` from the installed-napplet catalog (signed NIP-5A manifests), not from currently-running instances, so not-yet-running handlers are discoverable.
- The shell MUST respond to every request with a result message carrying the same `id`.
- The shell MUST deliver `payload` to the resolved handler only after that handler is ready to receive it.
- The shell MUST NOT let a napplet learn the identity of, or address, another napplet except through this resolution. Callers name roles, never instances — unless the user has explicitly granted a specific `handler` dTag.
- The shell MAY reject an `open` whose `protocol` the resolved handler does not accept, or MAY fall back to the archetype's recommended default protocol.
- The shell SHOULD emit `open.availability.changed` when the catalog or a default changes.

## Security Considerations

- Opening a napplet is a navigation and focus-stealing action. Shells SHOULD treat `open` as an untrusted request and MAY rate-limit or require a user gesture, especially for `behavior.newWindow` or `behavior.focus`.
- Archetype resolution is a trust boundary. A napplet asking for `archetype: "note"` MUST NOT be able to coerce the shell into routing to an arbitrary napplet; only the user's default or an explicit user choice decides the handler. The `handler: "<dTag>"` form SHOULD require that the user has authorized cross-napplet targeting for the caller.
- Payloads cross a napplet boundary. The shell relays `payload` opaquely; the receiving napplet MUST treat it as untrusted input and validate it against the named `protocol`. The shell SHOULD NOT inspect or mutate payload contents beyond what routing requires.
- `available()` reveals which napplets are installed, which is a fingerprinting surface. Shells MAY scope or redact candidate details (e.g., omit `dTag`/`title`) per policy while still answering `available`.
- Default-handler settings are user state. Shells MUST NOT let a napplet silently set or change a default; changing a default is a user action.
- Cold-start delivery (passing initial payload at instantiation) MUST NOT leak the payload to napplets other than the resolved handler.

## Implementations

- (none yet)
