NAP-INTENT
==========

Archetype Intent Dispatcher
---------------------------

`draft`

**NAP ID:** NAP-INTENT
**Domain:** `intent`
**Web binding (NIP-5D):** `window.napplet.intent` · `shell.supports("intent")`

## Description

NAP-INTENT provides napplets with a shell-mediated interface for invoking *another* napplet by its **archetype** — a shared role name such as `note`, `profile`, or `emoji-list` (see [ARCHETYPES.md](../ARCHETYPES.md)). A napplet describes *what role* it wants, *what action* to perform, and *what payload* to deliver; the shell resolves the role to an installed napplet, applies the user's default-handler preference, creates or focuses the window, and delivers the payload. This is the napplet equivalent of an operating system's implicit intents with a "default application": the caller names a role and an action, never a specific napplet.

The model maps directly onto a proven pattern (Android-style implicit intents): the **archetype** is the intent category, the **action** is the intent action, the **payload** is the extras, and the user's default handler is the default app. NAP-INTENT standardizes the **envelope**, not the payload. The `payload` is opaque and tagged by a `protocol` field naming the numbered wire format (a NAP-N spec) that shapes it — exactly as an HTTP request body is opaque and tagged by `Content-Type`. This keeps the two axes independent and free to evolve at their own speed:

- **archetype** — *routing*: which napplet should handle this, and whose default applies.
- **protocol** — *parsing*: what wire format the payload is in.

The two are orthogonal (N:M): one protocol may serve several archetypes, and one archetype may accept several protocols. Resolution of an archetype to a concrete napplet — including which napplets fulfill which roles and which is the user's default — is shell policy. Napplets never address each other directly.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `invoke` | `request` (`IntentRequest`) | `IntentResult` | `intent.invoke` / `intent.invoke.result` |
| `open` | `archetype` (`tstr`), optional `payload` (`any`), optional `opts` (`IntentOpenOptions`) | `IntentResult` | sugar over `intent.invoke` with action `"open"` |
| `available` | `archetype` (`tstr`) | `IntentAvailability` | `intent.available` / `intent.available.result` |
| `handlers` | none | list of `IntentAvailability` | `intent.handlers` / `intent.handlers.result` |
| `onChanged` | handler for `IntentAvailability` | `Subscription` handle | `intent.changed` |

### Schemas

`IntentBehavior` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `focus` | no | boolean | Focus the target surface. |
| `newWindow` | no | boolean | Request a new window instead of reuse. |
| `reuse` | no | boolean | Permit reuse of an existing matching window. |

`IntentOpenOptions` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `protocol` | no | text | NAP-N id shaping the payload. |
| `handler` | no | text | `default`, `choose`, or a specific napplet dTag. |
| `behavior` | no | `IntentBehavior` | Window/focus hints. |

`IntentRequest` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `archetype` | yes | text | Role slug, e.g. `note`. |
| `action` | no | text | Defaults to `open`. |
| `protocol` | no | text | NAP-N id shaping the payload. |
| `payload` | no | any | Opaque; typed by `protocol`. |
| `handler` | no | text | `default`, `choose`, or a specific napplet dTag. |
| `behavior` | no | `IntentBehavior` | Window/focus hints. |

`IntentContract` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `action` | yes | text | Verb this contract serves; defaults to `open`. |
| `protocol` | yes | text | NAP-N id shaping the payload for this contract. |
| `eventKinds` | no | list of unsigned integers | Optional NIP-01 event-kind constraints. |

`IntentCandidate` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `dTag` | yes | text | Napplet that can fulfill the archetype. |
| `title` | no | text | Human-readable handler label. |
| `actions` | yes | list of text | Supported actions. |
| `protocols` | yes | list of text | Supported payload protocols. |
| `contracts` | yes | list of `IntentContract` | Parsed manifest contracts. |
| `isDefault` | no | boolean | Whether this candidate is the user's default. |

`IntentAvailability` fields:

| Field | Required | Type |
|-------|----------|------|
| `archetype` | yes | text |
| `available` | yes | boolean |
| `candidates` | yes | list of `IntentCandidate` |
| `hasDefault` | yes | boolean |

`IntentResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `ok` | yes | boolean | Whether dispatch completed. |
| `archetype` | yes | text | Requested role slug. |
| `action` | yes | text | Dispatched action. |
| `handled` | yes | boolean | Whether a handler accepted the dispatch. |
| `handler` | no | text | dTag of the napplet that handled it. |
| `windowId` | no | text | Shell-assigned window id. |
| `protocol` | no | text | Payload protocol used for delivery. |
| `error` | no | text | Failure reason. |

**`invoke(request)`** — Asks the shell to dispatch `action` (default `"open"`) to a napplet of `archetype` with `payload`. The shell resolves the archetype to a handler (the user's default, the napplet named in `handler`, or a user choice when `handler: "choose"`), creates or focuses its window, and delivers the payload using the named `protocol` (or the archetype's recommended default when `protocol` is omitted). The `action` is carried as a field, not encoded into the message type, so new actions never expand the wire surface. Returns once the handler has been resolved and the window created; delivery to the handler MAY complete asynchronously.

**`open(archetype, payload?, opts?)`** — Convenience sugar for `invoke({ archetype, action: "open", payload, ...opts })`, the common case.

**`available(archetype)`** — Returns whether the runtime can currently satisfy `archetype`, the candidate napplets that fulfill it, and the actions, protocols, and contracts each supports. This is the pre-flight guardrail: a caller checks availability before showing an affordance, so a missing handler fails loudly at the call site instead of silently at delivery. Availability is sourced from the **installed-napplet catalog** (the manifests the runtime knows about), so it reports `true` for an installed handler that is not yet running.

**`handlers()`** — Returns availability for every archetype the runtime can currently satisfy. Useful for menus and capability surfaces.

**`onChanged(handler)`** — Registers for shell-pushed availability updates, fired when a napplet is installed or removed, or a default handler changes.

## Manifest Catalog Contract

A napplet declares every accepted archetype contract in its manifest with one
`archetype` tag per protocol:

```
["archetype", "<slug>", "<NAP-N>", "kind:<number>", ...]
```

Fields:

| Position | Value | Meaning |
|----------|-------|---------|
| 0 | `archetype` | tag name |
| 1 | `<slug>` | NAAT role slug, e.g. `note` |
| 2 | `<NAP-N>` | one numbered wire protocol accepted for this role |
| 3+ | `kind:<number>` | optional NIP-01 event-kind constraint for this protocol |

Each tag declares one `IntentContract` with `action: "open"` unless the named
NAP-N's `Serves:` metadata scopes it to a different action. A napplet that accepts
several protocols for the same archetype repeats the tag:

```
["archetype", "note", "NAP-4", "kind:1", "kind:30023"]
["archetype", "note", "NAP-8", "kind:30023"]
["archetype", "note", "NAP-12"]
```

The `kind:<number>` tokens apply only to the protocol named in the same tag. A
tag MUST NOT name several NAP-N ids, because that makes later constraints
ambiguous. If no `kind:<number>` token is present, the contract declares no
event-kind restriction.

Runtimes MUST build `available()` / `handlers()` from these manifest tags. They
MUST expose the parsed contracts in `IntentCandidate.contracts`, and MAY derive
the summary `actions` and `protocols` arrays from those contracts for quick
filtering.

## Wire Protocol

`intent.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `intent.invoke` | napplet -> shell | `id`, `request` |
| `intent.invoke.result` | shell -> napplet | `id`, `result`, `error?` |
| `intent.available` | napplet -> shell | `id`, `archetype` |
| `intent.available.result` | shell -> napplet | `id`, `availability`, `error?` |
| `intent.handlers` | napplet -> shell | `id` |
| `intent.handlers.result` | shell -> napplet | `id`, `handlers`, `error?` |
| `intent.changed` | shell -> napplet | `availability` |

Key design notes:
- Request/result pairs use `id` for correlation.
- The **action is a field** (`request.action`), never part of the message type. `intent.invoke` is the single dispatch verb for `open`, `edit`, `pick`, `share`, and any future action.
- `intent.changed` is a shell push message and has no `id`.
- The shell delivers `payload` to the resolved handler using the named `protocol`'s own delivery mechanism — typically a NAP-N INC topic event (e.g., `note:open`), or initial state passed at instantiation for a cold-started handler. NAP-INTENT governs resolution, default handling, and window lifecycle; the NAP-N protocol governs the payload wire and its delivery.
- `protocol` and `archetype` are independent. The shell MUST NOT assume a one-to-one mapping between them.

### Examples

**Conditionally show an "add emoji list" button, then open it:**
```
-> { "type": "intent.available", "id": "a1", "archetype": "emoji-list" }
<- {
     "type": "intent.available.result",
     "id": "a1",
     "availability": {
       "archetype": "emoji-list",
       "available": true,
       "candidates": [
         {
           "dTag": "emojilistr",
           "title": "Emoji List Maker",
           "actions": ["open"],
           "protocols": ["NAP-7"],
           "contracts": [{ "action": "open", "protocol": "NAP-7" }],
           "isDefault": true
         }
       ],
       "hasDefault": true
     }
   }
-> {
     "type": "intent.invoke",
     "id": "i1",
     "request": {
       "archetype": "emoji-list",
       "action": "open",
       "payload": { "seed": ["🤙", "⚡"] },
       "behavior": { "focus": true }
     }
   }
<- {
     "type": "intent.invoke.result",
     "id": "i1",
     "result": {
       "ok": true,
       "archetype": "emoji-list",
       "action": "open",
       "handled": true,
       "handler": "emojilistr",
       "windowId": "win-12",
       "protocol": "NAP-7"
     }
   }
```

**Open a note using a specific wire format (negotiated upgrade):**
```
-> {
     "type": "intent.invoke",
     "id": "i2",
     "request": {
       "archetype": "note",
       "action": "open",
       "protocol": "NAP-4",
       "payload": { "target": { "type": "event", "id": "abc..." } }
     }
   }
<- { "type": "intent.invoke.result", "id": "i2",
     "result": { "ok": true, "archetype": "note", "action": "open", "handled": true, "handler": "noteview", "windowId": "win-13", "protocol": "NAP-4" } }
```

**Discover note handlers by event kind:**
```
-> { "type": "intent.available", "id": "a2", "archetype": "note" }
<- {
     "type": "intent.available.result",
     "id": "a2",
     "availability": {
       "archetype": "note",
       "available": true,
       "candidates": [
         {
           "dTag": "noteview",
           "title": "Note Viewer",
           "actions": ["open"],
           "protocols": ["NAP-4", "NAP-8"],
           "contracts": [
             { "action": "open", "protocol": "NAP-4", "eventKinds": [1, 30023] },
             { "action": "open", "protocol": "NAP-8", "eventKinds": [30023] }
           ],
           "isDefault": true
         }
       ],
       "hasDefault": true
     }
   }
```

**No handler installed:**
```
-> { "type": "intent.invoke", "id": "i3", "request": { "archetype": "emoji-list", "payload": {} } }
<- { "type": "intent.invoke.result", "id": "i3",
     "result": { "ok": false, "archetype": "emoji-list", "action": "open", "handled": false, "error": "no handler" } }
```

### Error Handling

Result messages MAY include `error` when the request cannot be fulfilled. Common errors include `"unknown archetype"`, `"no handler"`, `"unsupported action"` (the resolved handler does not support the requested `action`), `"unsupported protocol"` (the resolved handler does not accept the requested `protocol`), `"user cancelled"` (during an "open with…" prompt), and `"invoke failed"`.

The shell SHOULD return a structured `result` with `ok: false` and `handled: false` when resolution or delivery fails. A caller that wants to avoid the failure path SHOULD call `available()` first.

## Shell Behavior

- The shell MUST resolve an `archetype` to a handler using its catalog of installed napplets and the user's default-handler preference for that archetype.
- The shell MUST parse one `IntentContract` from each valid `archetype` manifest tag, and MUST scope `kind:<number>` constraints to the protocol named in the same tag.
- The shell MUST keep a user-overridable default per archetype. When a default exists, `invoke` without an explicit `handler` MUST route to it.
- The shell SHOULD offer an "open with…" chooser when `handler: "choose"`, or when no default exists and more than one candidate is available.
- The shell MUST source `available()` / `handlers()` from the installed-napplet catalog (signed NIP-5A manifests), not from currently-running instances, so not-yet-running handlers are discoverable.
- The shell MUST respond to every request with a result message carrying the same `id`.
- The shell MUST deliver `payload` to the resolved handler only after that handler is ready to receive it.
- The shell MUST NOT let a napplet learn the identity of, or address, another napplet except through this resolution. Callers name roles, never instances — unless the user has explicitly granted a specific `handler` dTag.
- The shell MAY reject an `invoke` whose `action` or `protocol` the resolved handler does not support, or MAY fall back to the archetype's recommended default protocol.
- The shell SHOULD emit `intent.changed` when the catalog or a default changes.

## Security Considerations

- Dispatching an intent is a navigation and focus-stealing action. Shells SHOULD treat `invoke` as an untrusted request and MAY rate-limit or require a user gesture, especially for `behavior.newWindow` or `behavior.focus`.
- Archetype resolution is a trust boundary. A napplet asking for `archetype: "note"` MUST NOT be able to coerce the shell into routing to an arbitrary napplet; only the user's default or an explicit user choice decides the handler. The `handler: "<dTag>"` form SHOULD require that the user has authorized cross-napplet targeting for the caller.
- Payloads cross a napplet boundary. The shell relays `payload` opaquely; the receiving napplet MUST treat it as untrusted input and validate it against the named `protocol`. The shell SHOULD NOT inspect or mutate payload contents beyond what routing requires.
- `available()` reveals which napplets are installed, which is a fingerprinting surface. Shells MAY scope or redact candidate details (e.g., omit `dTag`/`title`) per policy while still answering `available`.
- Default-handler settings are user state. Shells MUST NOT let a napplet silently set or change a default; changing a default is a user action.
- Cold-start delivery (passing initial payload at instantiation) MUST NOT leak the payload to napplets other than the resolved handler.

## Implementations

- (none yet)
