NAP-INTENT
==========

Archetype Intent Dispatcher
---------------------------

`draft`

**NAP ID:** NAP-INTENT
**Domain:** `intent`
**Depends:** none.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.intent`; domain presence signals availability.

## Description

NAP-INTENT lets a napplet invoke another napplet by convention URI. The runtime
derives the requested archetype and action, resolves an installed handler,
accepts responsibility for delivery, and mediates the target lifecycle. The
**source** is the invoking napplet. The **target** is the resolved handler. The
source names a role and intent, never a target instance unless the user has
explicitly authorized one.

The developer-facing URI is `napplet:<archetype>/<intent>[?params][#<naddr>]`. Its queryless, fragment-free path is the stable convention identity. The runtime-provided binding derives the normalized `archetype`, `action`, `convention`, and `payload` fields before sending the wire request. The URI is authoritative.

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
| `payload` | no | any | Structured or non-text payload; cannot accompany URI query parameters. |
| `handler` | no | text | `default`, `choose`, or an authorized catalog identifier; see Catalog identifiers. |
| `handlerHint` | no | `IntentHandlerHint` | Explicit recommendation; cannot accompany a URI fragment. |
| `behavior` | no | `IntentBehavior` | Lifecycle and focus hints. |

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
| `archetype` | yes | text | Derived from the convention URI. |
| `action` | yes | text | Derived from the convention URI intent. |
| `convention` | yes | text | Stable, queryless, fragment-free convention identity. |
| `payload` | no | any | Query-derived text map or explicit payload. |
| `handler` | no | text | `default`, `choose`, or an authorized catalog identifier. |
| `handlerHint` | no | `IntentHandlerHint` | Recommendation decoded from the URI fragment or supplied in options. |
| `behavior` | no | `IntentBehavior` | Lifecycle and focus hints. |

`IntentContract` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `convention` | yes | text | Stable, queryless, fragment-free convention identity. |
| `params` | yes | list of text | Parameter names advertised by the manifest's `i` tag; empty when none are advertised. |

`IntentCandidate` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | text | Runtime-assigned catalog identifier for the napplet; see Catalog identifiers. |
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
| `handler` | no | text | Resolved handler's catalog identifier. Required when `ok` is true. |
| `error` | no | text | Pre-acceptance failure reason. Required when `ok` is false. |

`IntentDelivery` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `sender` | yes | text | Runtime-attested source catalog identifier. |
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
3. The queryless, fragment-free URI becomes `convention`.
4. Each unique, percent-decoded `name=value` query pair becomes a text payload
   field.

A URI fragment MUST be a bare [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) `naddr` for a named napplet matching the `IntentHandlerHint` address constraints. The binding MUST decode its coordinate and optional relay hints into `request.handlerHint` before sending `intent.invoke`. It MUST reject an invalid fragment or a fragment combined with `options.handlerHint`. Without a fragment, an explicit `options.handlerHint` becomes `request.handlerHint`. The recommendation is separate from explicit `handler` selection and MUST NOT appear in `convention` or `payload`.

The binding MUST NOT coerce query values to boolean, number, or null. A `+` is a literal plus sign, not a space. Malformed percent-encoding, a repeated query name, or an explicit payload alongside query parameters MUST be rejected before invocation. Structured or non-text data MUST use `options.payload` with a queryless URI; that URI MAY include a valid handler fragment.

This fragment exception applies only to NAP-INTENT invocation. Other convention-URI operations, including NAP-INC, MUST reject fragments.

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

The manifest schema and verification rules belong to
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303).
Catalogs MUST recognize its napplet manifest kinds `5129` (snapshot), `15129`
(root), and `35129` (named). Only named manifests carry a `d` tag.
NAP-INTENT defines how their advertisements become dispatch contracts; it does
not define another manifest format.

A napplet advertises a role with `z` and each accepted convention with `i`:

```
["z", "<slug>"]
["i", "napplet:<slug>/<action>", "<param>", ...]
```

For each valid `i` tag, the runtime MUST expose an `IntentContract` whose
`convention` is the tag's second element and whose `params` are the remaining
elements, in order. With no trailing elements, `params` is empty. The convention
MUST be a stable, queryless, fragment-free identity. Its archetype segment MUST
match a `z` value on the same manifest to be eligible for dispatch by that role.
Its intent segment defines the action. Tag order is irrelevant; a runtime MUST
NOT associate `i` with the nearest preceding `z` tag.

A napplet repeats `z` for multiple roles and `i` for multiple accepted intents:

```
["z", "feed"]
["i", "napplet:feed/open"]
["i", "napplet:feed/edit", "filters", "relays"]
```

Trailing `i` values advertise parameter names, not event-kind constraints,
types, required fields, or parameter values. They MUST NOT be interpreted as
`kind:<number>` restrictions. An empty `params` list does not prohibit an
explicit payload. Payload requirements and validation belong to the convention
and the target. The runtime MUST NOT inspect payload content to infer an event
kind or select a handler.

A `z` tag alone does not declare an accepted intent; an `i` tag alone does not
advertise its role. Missing or unusable advertisements MUST NOT invalidate an
otherwise valid napplet manifest. They contribute no matching dispatch contract.
Legacy combined `archetype` tags MUST NOT substitute for `z` and `i`.

Runtimes MUST build `available()` and `handlers()` from these manifest tags.
For each role, candidates MUST have at least one eligible contract, and their
`contracts` MUST contain only contracts for that role. Runtimes MUST derive the
candidate's `actions` and `conventions` from those contracts, removing duplicates.
Handler resolution MUST match the requested stable convention by exact equality;
it MUST NOT prefix-match, wildcard-match, or normalize that identity.

Advertisements are routing hints, never capability grants. The runtime MUST
apply the manifest capability requirements and its own policy before accepting
delivery. Declaring a role or intent MUST NOT widen the domains exposed to the
napplet. In the web projection, domain availability comes from the injected
namespace as defined by [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303);
neither catalog discovery nor invocation requires a `shell` domain or handshake.

### Catalog identifiers

`IntentCandidate.id`, an explicit `handler`, a successful result's `handler`,
and `IntentDelivery.sender` use the same runtime-assigned identifier space.
Identifiers are opaque text: callers MUST preserve them exactly and MUST NOT
interpret them as a `d` tag, event address, or running-instance identifier.

`handlerHint.address` is a recommendation coordinate, not a catalog identifier. The runtime resolves a permitted recommendation to a verified catalog entry before selecting its identifier; callers MUST NOT substitute a hint coordinate for an explicit `handler`.

The runtime MUST reserve `default` and `choose` for handler selection.

Identifiers MUST distinguish publishers and manifest kinds. Named napplets are
keyed by publisher, kind, and `d`; root napplets by publisher and kind; snapshots
by signed event id. Updating a root or named manifest MUST preserve its catalog
identifier. Distinct snapshots MUST have distinct identifiers. A bare `d` tag
MUST NOT serve as catalog identity. Identifier encoding and persistence across
runtime restarts are runtime policy; identifiers are not portable between
runtimes.

The runtime MUST also assign an identifier to the source, even when it has no
dispatch advertisements. It MUST derive `sender` from the source's verified
manifest bound to its authenticated endpoint. Catalog identity does not replace
the web projection's artifact verification or endpoint binding.

## Wire Protocol

`intent.*` messages use the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format
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

For an installed manifest advertising `["z", "profile"]` and
`["i", "napplet:profile/open", "pubkey"]`:

```
-> { "type": "intent.available", "id": "a1", "archetype": "profile" }
<- {
     "type": "intent.available.result",
     "id": "a1",
     "availability": {
       "archetype": "profile",
       "available": true,
       "candidates": [{
         "id": "catalog-profile-viewer",
         "title": "Profile Viewer",
         "actions": ["open"],
         "conventions": ["napplet:profile/open"],
         "contracts": [{
           "convention": "napplet:profile/open",
           "params": ["pubkey"]
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
       "handler": "catalog-profile-viewer"
     }
   }
```

After acceptance, the runtime may leave the source open or close it. Once the
target is ready, it receives:

```
<- {
     "type": "intent.deliver",
     "delivery": {
       "sender": "catalog-social-feed",
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
- The runtime MUST source `available()` and `handlers()` from verified installed
  [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) manifests, not only
  running instances.
- The runtime MUST NOT let a caller address a handler instance unless the user
  explicitly authorized that handler.
- The runtime MUST reject an unsupported convention before acceptance.
- The runtime SHOULD emit `intent.changed` when catalog or default-handler state
  changes.

### Recommended handler resolution

The shell MUST apply this precedence:

1. An explicit authorized `handler` catalog identifier or `handler: "choose"` selection.
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

- [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) — shareable address identifiers and relay hints.
- [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) — addressable event coordinates and resolution.
- [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) — napplet manifests, artifact verification, identity, domain availability, and normative web binding.
- [NIP-5A](https://github.com/nostr-protocol/nips/blob/master/5A.md) — existing napplet manifests.

## Implementations

- (none yet)

## Changelog

- `ad0847b` - Introduced archetype-based intent dispatch with action as data.
- `6461e4b` - Adopted unnumbered convention identities for payload shapes.
- `6c0d731` - Made convention URIs authoritative and delivery lifecycle-independent while preserving manifest-derived handler discovery.
- `3dc945f` - Aligned manifest discovery with z/i advertisements and parameter names, supported all napplet manifest kinds with publisher-safe catalog identifiers, and adopted injected-domain availability.

- `f89efbe` - Reconciled handler recommendation normalization and schemas with opaque catalog selection; kept fragments outside convention identity and payload.
