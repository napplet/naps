NAP-INTENT
==========

Archetype Intent Dispatcher
---------------------------

`draft`

**NAP ID:** NAP-INTENT
**Domain:** `intent`
**Depends:** none.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.intent` · `shell.supports("intent")`

## Description

NAP-INTENT provides napplets with a shell-mediated interface for invoking *another* napplet by its **archetype** — a shared role name such as `note`, `profile`, or `emoji-list` (see [ARCHETYPES.md](../ARCHETYPES.md)). A napplet describes *what role* it wants, *what action* to perform, and *what payload* to deliver. It MAY recommend a handler. The shell resolves the role to an installed napplet, applies the user's default-handler preference, creates or focuses the window, and delivers the payload. A recommendation does not authorize targeting or override the user's default.

The **archetype** names the role, the **action** names the intent, and the
**payload** carries convention data. NAP-INTENT standardizes the envelope, not
the payload. The optional `convention` field names a stable, queryless and
fragment-free `napplet:<archetype>/<intent>` identity, such as
`napplet:note/open` or `napplet:profile/open`:

- **archetype** — *routing*: which napplet should handle this, and whose default applies.
- **convention** — *parsing*: what payload shape the handler expects.

One archetype may accept several conventions. A convention's archetype segment
MUST equal the requested archetype. Resolution to a concrete napplet remains
shell policy. A recommended handler is metadata for that resolution, not a
direct message destination.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `invoke` | `request` (`IntentRequest`), or convention URI (`text`) with optional options (`IntentInvokeOptions`) | `IntentResult` | `intent.invoke` / `intent.invoke.result` |
| `open` | `archetype` (`text`), optional `payload` (`any`), optional `opts` (`IntentOpenOptions`) | `IntentResult` | sugar over `intent.invoke` with action `"open"` |
| `available` | `archetype` (`text`) | `IntentAvailability` | `intent.available` / `intent.available.result` |
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
| `convention` | no | text | Stable, queryless and fragment-free convention identity. |
| `handler` | no | text | `default`, `choose`, or an authorized napplet address. |
| `handlerHint` | no | `IntentHandlerHint` | Recommended handler; does not override user selection. |
| `behavior` | no | `IntentBehavior` | Window/focus hints. |

`IntentInvokeOptions` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `payload` | no | any | Structured or non-text payload; cannot accompany URI query parameters. |
| `handler` | no | text | `default`, `choose`, or an authorized napplet address. |
| `handlerHint` | no | `IntentHandlerHint` | Explicit recommendation; cannot accompany a URI fragment. |
| `behavior` | no | `IntentBehavior` | Window/focus hints. |

`IntentHandlerHint` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `address` | yes | text | `35129:<pubkey>:<d>` coordinate decoded from the recommended napplet's address. |
| `relays` | no | list of text | Optional discovery hints; not part of the napplet identity. |

A napplet address is a [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md)
coordinate, `<kind>:<pubkey>:<d>`. The pubkey is exactly 64 lowercase hex digits; the
non-empty `d` value is opaque and case-sensitive. Clients MUST NOT normalize
`d` or identify a handler by `d` alone. `IntentHandlerHint` accepts kind `35129`
only. Existing catalog entries MAY use other napplet manifest kinds; a hint
does not change which manifest formats the runtime supports.

`IntentRequest` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `archetype` | yes | text | Role slug, e.g. `note`. |
| `action` | no | text | Defaults to `open`. |
| `convention` | no | text | Stable, queryless and fragment-free convention identity. |
| `payload` | no | any | Opaque; typed by `convention`. |
| `handler` | no | text | `default`, `choose`, or an authorized napplet address. |
| `handlerHint` | no | `IntentHandlerHint` | Normalized recommendation, separate from the payload. |
| `behavior` | no | `IntentBehavior` | Window/focus hints. |

`IntentCandidate` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `address` | yes | text | Publisher-scoped address of the napplet that can fulfill the archetype. |
| `title` | no | text | Human-readable handler label. |
| `actions` | yes | list of text | Supported actions. |
| `conventions` | yes | list of text | Supported payload conventions. |
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
| `handler` | no | text | Publisher-scoped address of the napplet that handled it. |
| `windowId` | no | text | Shell-assigned window id. |
| `convention` | no | text | Payload convention used for delivery. |
| `error` | no | text | Failure reason. |

`action` defaults to `open` when omitted. `payload` is opaque and shaped by
`convention` when present.

### Convention URI normalization

The URI invocation form is:

```text
napplet:<archetype>/<intent>[?params][#<naddr>]
```

`<naddr>` is a placeholder for one complete, bare `naddr1…` identifier defined
by [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md).
The angle brackets are not literal. The fragment MUST NOT contain a key such as
`handler=`, a `nostr:` prefix, percent-encoding, or additional data.

Before sending `intent.invoke`, the runtime-provided binding MUST:

1. Decode any fragment as a NIP-19 `naddr`. Validate its checksum and required
   coordinate fields. Require kind `35129` and a non-empty identifier. Ignore
   unsupported TLV fields as NIP-19 specifies.
2. Put the decoded coordinate in `request.handlerHint.address` and any relay
   hints in `request.handlerHint.relays`. Strip the fragment from the URI.
3. Derive `archetype` and `action` from the path. Set `convention` to the exact
   queryless, fragment-free identity. Require one non-empty archetype segment
   and one non-empty intent segment separated by `/`.
4. Transpose each unique, percent-decoded `name=value` query pair into a text
   payload field. Without a query, use the explicit options payload, if present.
5. Copy any explicit `handler` and `behavior` options into the request.

The binding MUST NOT coerce scalar types. A `+` is a literal plus sign, not a
space. It MUST reject an empty or invalid fragment, malformed percent-encoding,
repeated query names, query parameters with an explicit payload, or a fragment
with an explicit `options.handlerHint`. A fragment MAY accompany an explicit
structured payload when there is no query.

When no fragment is present, the binding MUST copy an explicit
`options.handlerHint` into the request after validating it. If neither source
supplies a recommendation, it MUST omit `handlerHint`. A hint's relay list contains text values only; unsupported
relay URLs MAY be ignored under network policy without invalidating its address.

The shell MUST validate direct wire requests by the same normalized schema.
It MUST reject a `convention` containing a query or fragment, a mismatch between
its archetype or intent and the request fields, or a malformed `handlerHint`.
Routers MUST match the normalized convention identity by exact equality.
They MUST NOT parse fragments or queries during handler resolution.

This fragment exception applies only to the NAP-INTENT URI invocation form.
Handler metadata and subscriptions MUST remain queryless and fragment-free.
`handlerHint` MUST NOT become a payload field or be delivered to the target.

**`invoke(request)`** — Asks the shell to dispatch `action` (default `"open"`) to a napplet of `archetype` with `payload`. The shell resolves the archetype to a handler (the user's default, the napplet named in `handler`, or a user choice when `handler: "choose"`), creates or focuses its window, and delivers the payload using the named `convention` (or the archetype's recommended default when `convention` is omitted). The `action` is carried as a field, not encoded into the message type, so new actions never expand the wire surface. Returns once the handler has been resolved and the window created; delivery to the handler MAY complete asynchronously.

**`invoke(uri, options?)`** — Normalizes the convention URI into `IntentRequest`
and invokes the same dispatch operation. A fragment recommends a handler under
the selection rules below; it does not require that handler to receive the intent.

**`open(archetype, payload?, opts?)`** — Convenience sugar for `invoke({ archetype, action: "open", payload, ...opts })`, the common case.

**`available(archetype)`** — Returns whether the runtime can currently satisfy `archetype`, the candidate napplets that fulfill it, and the actions and conventions each supports. This is the pre-flight guardrail: a caller checks availability before showing an affordance, so a missing handler fails loudly at the call site instead of silently at delivery. Availability is sourced from the **installed-napplet catalog** (the manifests the runtime knows about), so it reports `true` for an installed handler that is not yet running.

**`handlers()`** — Returns availability for every archetype the runtime can currently satisfy. Useful for menus and capability surfaces.

**`onChanged(handler)`** — Registers for shell-pushed availability updates, fired when a napplet is installed or removed, or a default handler changes.

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
- The shell delivers `payload` to the resolved handler using the named convention's delivery mechanism. Internal delivery choices impose no capability requirement on the caller or target. NAP-INTENT governs resolution, default handling, and window lifecycle; the convention governs the payload shape.
- `handlerHint` is shell-only selection metadata. Delivery carries the normalized convention and payload, not the URI fragment or recommendation.

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
         { "address": "35128:<publisher-pubkey>:emojilistr", "title": "Emoji List Maker", "actions": ["open"], "conventions": ["napplet:emoji-list/open"], "isDefault": true }
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
       "handler": "35128:<publisher-pubkey>:emojilistr",
       "windowId": "win-12",
       "convention": "napplet:emoji-list/open"
     }
   }
```

**Open a note using a specific convention:**
```
-> {
     "type": "intent.invoke",
     "id": "i2",
     "request": {
       "archetype": "note",
       "action": "open",
       "convention": "napplet:note/open",
       "payload": { "target": { "type": "event", "id": "abc..." } }
     }
   }
<- { "type": "intent.invoke.result", "id": "i2",
     "result": { "ok": true, "archetype": "note", "action": "open", "handled": true, "handler": "35128:<publisher-pubkey>:noteview", "windowId": "win-13", "convention": "napplet:note/open" } }
```

**Recommend a profile handler:**

```text
invoke("napplet:profile/open?pubkey=abc123…#<naddr>")
```

Replace `<naddr>` with a complete `naddr1…` for kind `35129`. If it decodes to
`35129:<publisher-pubkey>:profile-viewer` with one relay hint, the normalized
request is:

```json
{
  "type": "intent.invoke",
  "id": "i-hint",
  "request": {
    "archetype": "profile",
    "action": "open",
    "convention": "napplet:profile/open",
    "payload": { "pubkey": "abc123…" },
    "handlerHint": {
      "address": "35129:<publisher-pubkey>:profile-viewer",
      "relays": ["wss://relay.example.com"]
    }
  }
}
```

An applicable user default wins even if it has a different address. Without a
default, the shell SHOULD consider the recommendation. If it is not installed,
the shell MAY resolve its event and offer installation. A declined offer or an
unavailable, incompatible, or policy-denied recommendation falls back to normal
resolution. A valid recommendation's failure is not an `"invalid handler hint"`
error.

**Recommend a handler with structured data:**

```text
invoke("napplet:note/open#<naddr>", {
  "payload": { "target": { "type": "event", "id": "abc…" } }
})
```

**No handler installed:**
```
-> { "type": "intent.invoke", "id": "i3", "request": { "archetype": "emoji-list", "payload": {} } }
<- { "type": "intent.invoke.result", "id": "i3",
     "result": { "ok": false, "archetype": "emoji-list", "action": "open", "handled": false, "error": "no handler" } }
```

### Error Handling

Result messages MAY include `error` when the request cannot be fulfilled. Common errors include `"unknown archetype"`, `"no handler"`, `"unsupported action"` (the resolved handler does not support the requested `action`), `"unsupported convention"` (the resolved handler does not accept the requested `convention`), `"user cancelled"` (during an "open with…" prompt), and `"invoke failed"`.

The shell SHOULD return a structured `result` with `ok: false` and `handled: false` when resolution or delivery fails. A caller that wants to avoid the failure path SHOULD call `available()` first.

Malformed URI fragments or normalized hints MUST be rejected before resolution
with `"invalid handler hint"`. Valid hints that cannot be used MUST fall back to
normal resolution. If that also fails, return the ordinary resolution error,
such as `"no handler"`.

## Shell Behavior

- The shell MUST resolve an `archetype` to a handler using its catalog of installed napplets and the user's default-handler preference for that archetype.
- The shell MUST keep a user-overridable default per archetype. An applicable default MUST take precedence over `handlerHint`.
- The shell SHOULD offer an "open with…" chooser when `handler: "choose"`, or when normal resolution reaches multiple candidates without an applicable default or usable recommendation.
- The shell MUST source `available()` / `handlers()` from its verified installed-napplet catalog, not from currently-running instances, so not-yet-running handlers are discoverable.
- The shell MUST respond to every request with a result message carrying the same `id`.
- The shell MUST deliver `payload` to the resolved handler only after that handler is ready to receive it.
- The shell MUST NOT let a napplet address a target instance directly. An explicit `handler` address requires user authorization. `handlerHint` is a recommendation only.
- The shell MUST NOT route an explicitly requested convention to a handler that does not advertise it. When `convention` is absent, the shell MAY use the archetype's recommended default convention.
- The shell SHOULD emit `intent.changed` when the catalog or a default changes.

### Recommended handler resolution

The shell MUST apply this precedence:

1. An explicit authorized `handler` address or `handler: "choose"` selection.
2. The user's applicable default handler.
3. The recommended handler, if policy permits and it supports the requested
   archetype, action, and convention.
4. Normal installed-handler resolution or user choice.

Absent `handler` and `handler: "default"` both use steps 2–4. An explicit
selection MUST NOT silently fall back to the recommendation. An applicable
handler supports the requested contract and is permitted by shell policy.
Support MUST come from its verified catalog metadata, not from the hint itself.
Convention matching uses exact stable identities.

At step 3 the shell SHOULD consider a compatible installed recommendation. If
it is not installed, the shell MAY resolve its signed event using the coordinate
and relay hints and offer installation under its policy. The shell MUST verify
the resolved event's signature and exact coordinate before considering it. It
MUST apply its ordinary installation, artifact verification, and execution rules.
Discovery does not authorize installation or execution. The hint MUST NOT alter
the user's default. An `naddr` identifies a coordinate, not an artifact version.

An unavailable, unsupported, declined, or policy-denied recommendation MUST
fall back to step 4. A shell MAY ignore the recommendation under its policy.
It need not perform discovery or offer installation. Relay hints are untrusted
discovery inputs and MUST NOT bypass the shell's network policy. Discovery is
shell-internal; it imposes no additional capability requirement on the caller.

## Security Considerations

- Dispatching an intent is a navigation and focus-stealing action. Shells SHOULD treat `invoke` as an untrusted request and MAY rate-limit or require a user gesture, especially for `behavior.newWindow` or `behavior.focus`.
- Archetype resolution is a trust boundary. A recommendation MUST NOT bypass user defaults, handler compatibility checks, or shell policy. An explicit `handler` address MUST require user authorization for the caller.
- Payloads cross a napplet boundary. The shell relays `payload` opaquely; the receiving napplet MUST treat it as untrusted input and validate it against the named `convention`. The shell SHOULD NOT inspect or mutate payload contents beyond what routing requires.
- `available()` reveals which napplets are installed, which is a fingerprinting surface. Shells MAY scope or redact candidate details (e.g., return an empty candidate list) per policy while still answering `available`.
- Default-handler settings are user state. Shells MUST NOT let a napplet silently set or change a default; changing a default is a user action.
- Cold-start delivery (passing initial payload at instantiation) MUST NOT leak the payload to napplets other than the resolved handler.

## References

- [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) — shareable address identifiers and relay hints.
- [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) — addressable event coordinates and resolution.
- [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) — normative web binding.
- [NIP-5A](https://github.com/nostr-protocol/nips/blob/master/5A.md) — existing napplet manifests.

## Implementations

- (none yet)

## Changelog

- `ad0847b` - Introduced archetype-based intent dispatch with action as data.
- `6461e4b` - Adopted unnumbered convention identities for payload shapes.
- `efd51ef` - Added bare naddr URI-fragment recommendations, normalized handler hints, publisher-scoped handler addresses, and user-default-first resolution with discovery and fallback.
