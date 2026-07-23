NAP-INTENT
==========

Archetype Intent Dispatcher
---------------------------

`draft`

**NAP ID:** NAP-INTENT
**Domain:** `intent`
**Web binding (NIP-5D):** `window.napplet.intent` · `shell.supports("intent")`

## Description

NAP-INTENT lets a napplet invoke another napplet by convention URI. The runtime
derives the requested archetype and action, resolves an installed handler,
accepts responsibility for delivery, and mediates the target lifecycle. The
**source** is the invoking napplet. The **target** is the resolved handler. The
source names a role and intent, never a target instance unless the user has
explicitly authorized one.

The developer-facing URI is
`napplet:<archetype>/<intent>[...?params]`. Its queryless path is the stable
convention identity. The runtime-provided binding derives the normalized
`archetype`, `action`, `convention`, and `payload` fields before sending the wire
request. The URI is authoritative.

A successful invocation transfers delivery responsibility to the runtime. It
does not assert that the target already received or handled the intent. The
runtime may deliver to an existing target, start a target before closing the
source, or close the source before starting the target.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `invoke` | convention URI (`text`), optional options (`IntentInvokeOptions`) | `IntentResult` | `intent.invoke` / `intent.invoke.result` |
| `open` | convention URI whose intent is `open` (`text`), optional options (`IntentInvokeOptions`) | `IntentResult` | sugar over `invoke` |
| `available` | archetype (`text`) | `IntentAvailability` | `intent.available` / `intent.available.result` |
| `handlers` | none | list of `IntentAvailability` | `intent.handlers` / `intent.handlers.result` |
| `onChanged` | handler for `IntentAvailability` | `Subscription` handle | `intent.changed` |
| `onDelivery` | handler for `IntentDelivery` | `Subscription` handle | `intent.deliver` |

### Schemas

`IntentBehavior` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `focus` | no | boolean | Request focus for the target surface. |
| `reuse` | no | boolean | Permit reuse of an existing matching target. |

These fields are hints. Runtime lifecycle and workspace policy remain
authoritative.

`IntentInvokeOptions` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `payload` | no | any | Structured or non-text convention payload. |
| `handler` | no | text | `default`, `choose`, or an authorized napplet dTag. |
| `behavior` | no | `IntentBehavior` | Lifecycle and focus hints. |

`IntentRequest` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `archetype` | yes | text | Derived from the convention URI. |
| `action` | yes | text | Derived from the convention URI intent. |
| `convention` | yes | text | Stable, queryless convention identity. |
| `payload` | no | any | Query-derived text map or explicit payload. |
| `handler` | no | text | `default`, `choose`, or an authorized napplet dTag. |
| `behavior` | no | `IntentBehavior` | Lifecycle and focus hints. |

`IntentContract` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `convention` | yes | text | Stable, queryless convention identity. |
| `eventKinds` | no | list of unsigned integers | Optional discovery metadata; does not authorize runtime payload inspection. |

`IntentCandidate` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `dTag` | yes | text | Napplet that can fulfill the archetype. |
| `title` | no | text | Human-readable handler label. |
| `actions` | yes | list of text | Actions derived from accepted conventions. |
| `conventions` | yes | list of text | Stable convention identities. |
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
| `ok` | yes | boolean | Whether the runtime accepted delivery responsibility. |
| `archetype` | no | text | Normalized requested role. Required when `ok` is true. |
| `action` | no | text | Normalized requested action. Required when `ok` is true. |
| `convention` | no | text | Stable convention identity. Required when `ok` is true. |
| `handler` | no | text | Resolved handler dTag. Required when `ok` is true. |
| `error` | no | text | Pre-acceptance failure reason. Required when `ok` is false. |

`IntentDelivery` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `sender` | yes | text | Runtime-attested source napplet dTag. |
| `archetype` | yes | text | Normalized target role. |
| `action` | yes | text | Normalized action. |
| `convention` | yes | text | Stable convention identity. |
| `payload` | no | any | Normalized convention payload. |

No intent delivery identifier is exposed. The wire request `id` correlates only
`intent.invoke` with its immediate `intent.invoke.result`.

### Convention URI normalization

The runtime-provided binding MUST normalize a convention URI before sending
`intent.invoke`:

1. The first URI component after `napplet:` becomes `archetype`.
2. The path component after `/` becomes `action`.
3. The queryless URI becomes `convention`.
4. Each unique, percent-decoded `name=value` query pair becomes a text payload
   field.

The binding MUST NOT coerce query values to boolean, number, or null. A `+` is a
literal plus sign, not a space. A URI with a fragment, malformed
percent-encoding, a repeated query name, or an explicit payload alongside query
parameters MUST be rejected before invocation. Structured or non-text data MUST
use `options.payload` with a queryless URI.

The runtime MUST reject a normalized wire request when:

- `convention` contains a query or fragment;
- the convention archetype differs from `request.archetype`; or
- the convention intent differs from `request.action`.

**`invoke(uri, options?)`** — Normalizes the URI, resolves a handler, and asks
the runtime to accept delivery responsibility. A successful result may precede
target startup and delivery.

**`open(uri, options?)`** — Convenience form of `invoke`. The URI intent MUST be
`open`.

**`available(archetype)`** — Returns installed candidates and their stable
convention contracts. A candidate need not be running.

**`handlers()`** — Returns availability for every archetype known to the
runtime.

**`onChanged(handler)`** — Receives availability changes after catalog or
default-handler updates.

**`onDelivery(handler)`** — Receives an `IntentDelivery` after the target is ready.
The runtime supplies `sender`; the source napplet cannot set or override it. The
runtime-provided binding MUST retain an incoming delivery until an `onDelivery`
handler is registered.

## Manifest Catalog Contract

A napplet declares each accepted convention in its NIP-5A manifest with one
`archetype` tag:

```
["archetype", "<slug>", "napplet:<slug>/<action>", "kind:<number>", ...]
```

| Position | Value | Meaning |
|----------|-------|---------|
| 0 | `archetype` | Tag name. |
| 1 | `<slug>` | NAAT role slug. |
| 2 | convention | One stable, queryless convention identity. |
| 3+ | `kind:<number>` | Optional NIP-01 event-kind constraint. |

The convention archetype MUST equal `<slug>`. Its intent defines the accepted
action. Query parameters MUST NOT appear in handler metadata. A napplet repeats
the tag when it accepts several conventions:

```
["archetype", "note", "napplet:note/open", "kind:1", "kind:30023"]
["archetype", "note", "napplet:note/edit", "kind:30023"]
```

Each tag produces one `IntentContract`. `kind:<number>` values apply only to the
convention in the same tag. Absence of `kind:<number>` declares no event-kind
restriction. Event kinds help callers choose a compatible contract. The runtime
MUST NOT inspect payload content to infer an event kind during resolution.

Runtimes MUST build `available()` and `handlers()` from these manifest tags.
They MUST expose parsed `contracts`. They MAY derive `actions` and `conventions`
from those contracts for quick filtering.

## Wire Protocol

`intent.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `intent.invoke` | napplet -> runtime | `id`, `request` |
| `intent.invoke.result` | runtime -> napplet | `id`, `result` |
| `intent.available` | napplet -> runtime | `id`, `archetype` |
| `intent.available.result` | runtime -> napplet | `id`, `availability`, `error`? |
| `intent.handlers` | napplet -> runtime | `id` |
| `intent.handlers.result` | runtime -> napplet | `id`, `handlers`, `error`? |
| `intent.changed` | runtime -> napplet | `availability` |
| `intent.deliver` | runtime -> napplet | `delivery` |

Key design notes:

- Request/result pairs use `id` only for immediate correlation.
- `intent.deliver` is a runtime push and has no request or delivery id.
- The runtime-provided binding buffers `intent.deliver` when no `onDelivery`
  handler is registered yet.
- Handler discovery uses stable, queryless convention identities.
- Delivery uses the same `IntentDelivery` shape whether the target was already
  running or started for the invocation.
- NAP-INTENT does not depend on NAP-INC. A runtime MAY use INC internally for a
  live target, but that is not visible to the target contract.

### Examples

**Discover profile handlers:**

```
-> { "type": "intent.available", "id": "a1", "archetype": "profile" }
<- {
     "type": "intent.available.result",
     "id": "a1",
     "availability": {
       "archetype": "profile",
       "available": true,
       "candidates": [{
         "dTag": "profile-viewer",
         "title": "Profile Viewer",
         "actions": ["open"],
         "conventions": ["napplet:profile/open"],
         "contracts": [{
           "convention": "napplet:profile/open"
         }],
         "isDefault": true
       }],
       "hasDefault": true
     }
   }
```

**Invoke a convention URI:**

```
invoke("napplet:profile/open?pubkey=abc123...")
```

The runtime binding sends the normalized request:

```
-> {
     "type": "intent.invoke",
     "id": "i1",
     "request": {
       "archetype": "profile",
       "action": "open",
       "convention": "napplet:profile/open",
       "payload": { "pubkey": "abc123..." }
     }
   }
<- {
     "type": "intent.invoke.result",
     "id": "i1",
     "result": {
       "ok": true,
       "archetype": "profile",
       "action": "open",
       "convention": "napplet:profile/open",
       "handler": "profile-viewer"
     }
   }
```

After acceptance, the runtime may leave the source open or close it. Once the
target is ready, it receives:

```
<- {
     "type": "intent.deliver",
     "delivery": {
       "sender": "social-feed",
       "archetype": "profile",
       "action": "open",
       "convention": "napplet:profile/open",
       "payload": { "pubkey": "abc123..." }
     }
   }
```

**Invoke with a structured payload:**

```
invoke("napplet:note/open", {
  "payload": { "target": { "type": "event", "id": "abc..." } }
})
```

**No handler installed:**

```
-> {
     "type": "intent.invoke",
     "id": "i2",
     "request": {
       "archetype": "emoji-list",
       "action": "open",
       "convention": "napplet:emoji-list/open"
     }
   }
<- {
     "type": "intent.invoke.result",
     "id": "i2",
     "result": { "ok": false, "error": "no handler" }
   }
```

### Error Handling

The runtime MUST reject an invocation before acceptance when the convention URI
or normalized request is invalid, no compatible handler exists, policy denies
the request, or the user cancels handler selection. Common errors include
`"invalid convention"`, `"no handler"`, `"unsupported convention"`,
`"user cancelled"`, and `"invoke rejected"`.

An `ok: true` result transfers responsibility to the runtime. Target startup,
delivery timing, retries, terminal failure handling, and persistence across
runtime restart are runtime policy. A post-acceptance failure is not reported as
a second result to the source.

## Runtime Behavior

- The runtime MUST derive the source `sender` from the authenticated endpoint.
  It MUST ignore or reject caller-supplied sender data.
- The runtime MUST validate normalized `archetype`, `action`, and `convention`
  consistency before handler resolution.
- The runtime MUST keep a user-overridable default per archetype. A napplet
  cannot set or change it.
- The runtime SHOULD offer a chooser when `handler` is `choose`, or when several
  candidates exist and no default is set.
- The runtime MUST resolve an installed handler using manifest contracts and the
  user's default-handler preference.
- The runtime MUST respond to `intent.invoke` with the same request `id`.
- An `ok: true` result MUST mean the runtime accepted delivery responsibility.
- Before closing the source because of an invocation, the runtime MUST send
  its `intent.invoke.result`.
- After acceptance, the runtime MUST retain the normalized delivery independently
  of the source until runtime policy determines delivery succeeded or failed
  terminally.
- After acceptance, delivery MUST NOT depend on the source remaining alive.
- The runtime MUST deliver `IntentDelivery` only after the target is ready.
- The runtime MAY reuse an existing target, briefly overlap source and target,
  or close the source before starting the target.
- Retry behavior, target replacement, overlap, terminal failure handling, and
  persistence across runtime restart are runtime policy.
- `behavior` fields are hints. Runtime workspace and lifecycle policy remain
  authoritative.
- The runtime MUST source `available()` and `handlers()` from installed NIP-5A
  manifests, not only running instances.
- The runtime MUST NOT let a caller address a handler instance unless the user
  explicitly authorized that handler.
- The runtime MUST reject an unsupported convention before acceptance.
- The runtime SHOULD emit `intent.changed` when catalog or default-handler state
  changes.

## Security Considerations

- Invocation may navigate, focus, start, or close napplet surfaces. The runtime
  SHOULD rate-limit or require a user gesture according to policy.
- The runtime MUST derive `sender`; a napplet cannot impersonate another source.
- Query transposition interprets URI syntax only. After normalization, payload
  semantics remain opaque to the runtime.
- The target MUST validate `payload` against `convention` and treat `sender` as
  provenance, not proof that payload content is safe.
- Handler selection is a trust boundary. Only installed manifest contracts,
  user defaults, or explicit user choice may select a target.
- Default-handler state belongs to the user. A napplet MUST NOT change it.
- Availability can reveal installed napplets. The runtime MAY redact candidate
  details according to policy.
- Delivery MUST NOT leak payload or sender metadata to any napplet other than
  the resolved target.

## References

- [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) — normative web binding.
- [NIP-5A](https://github.com/nostr-protocol/nips/blob/master/5A.md) — napplet manifest and identity.
- [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) — event kinds.

## Implementations

- (none yet)

## Changelog

- `ad0847b` - Introduced archetype-based intent dispatch with action as data.
- `6461e4b` - Adopted unnumbered convention identities for payload shapes.
