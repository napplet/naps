NAP-CATALOG
===========

Runtime Napplet Catalog
-----------------------

`draft`

**NAP ID:** NAP-CATALOG
**Domain:** `catalog`
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.catalog` · `shell.supports("catalog")`

## Description

NAP-CATALOG provides read-only access to the napplets a runtime can currently
resolve and load. A napplet receives verified napplet identities, display
metadata, advertised archetypes, accepted convention intents and their
parameters, and the runtime's current handler for each archetype.

The catalog describes availability, not lifecycle. A catalog entry MAY describe
a napplet that is not running. A current handler is the napplet the runtime
would select for an implicit archetype dispatch at the time of the request. It
does not mean that the napplet is running, focused, or visible.

NAP-CATALOG does not install, launch, focus, or dispatch to napplets. It does not
set handler preferences. It does not define convention semantics. NAP-INTENT
owns archetype dispatch. Conventions own their message semantics. See
[NAP-INTENT](NAP-INTENT.md).

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `get` | none | `CatalogSnapshot` | `catalog.get` / `catalog.get.result` |

### Schemas

`NappletIdentity` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `dTag` | yes | text | Napplet identifier from the verified manifest. |
| `aggregateHash` | yes | text | Lowercase hexadecimal aggregate hash of the resolved napplet version. |

`IntentParameter` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `name` | yes | text | Query parameter name. |
| `required` | yes | boolean | Whether the handler requires the parameter. |
| `description` | no | text | Human-readable parameter purpose. |

`IntentDescriptor` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `intent` | yes | text | Intent name, such as `open`. |
| `convention` | yes | text | Stable queryless convention identity, such as `napplet:note/open`. |
| `parameters` | yes | list of `IntentParameter` | Parameters advertised for this convention. Empty when none are advertised. |

`ArchetypeDescriptor` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `archetype` | yes | text | NAAT role slug, such as `note`. |
| `intents` | yes | list of `IntentDescriptor` | Accepted intents for this role. |

`NappletDescriptor` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `identity` | yes | `NappletIdentity` | Verified napplet identity. |
| `title` | no | text | Human-readable title. |
| `description` | no | text | Human-readable description. |
| `requires` | yes | list of text | Bare NAP domains required by the napplet. |
| `archetypes` | yes | list of `ArchetypeDescriptor` | Advertised roles. Empty for a napplet with no archetype. |

`ArchetypeHandler` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `archetype` | yes | text | NAAT role slug. |
| `currentHandler` | yes | `NappletIdentity` or null | Current implicit dispatch target, or null when the runtime has none. |

`CatalogSnapshot` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `napplets` | yes | list of `NappletDescriptor` | Napplets available to the caller. |
| `handlers` | yes | list of `ArchetypeHandler` | Current handler state for every archetype advertised by the returned napplets. |

Parameter metadata describes only the shallow query parameters accepted by a
developer-facing convention URI. Values remain text after transposition. The
named convention remains authoritative for payload semantics. Structured or
non-text input uses the explicit payload and is outside this catalog schema.

**`get()`** — Returns one point-in-time snapshot. The runtime applies caller
policy before constructing the result. An empty `napplets` list is a successful
result when no napplets are available to the caller.

## Wire Protocol

`catalog.*` messages use the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `catalog.get` | napplet -> shell | `id` |
| `catalog.get.result` | shell -> napplet | `id`, `snapshot`?, `error`? |

Key design notes:

- The request and result use `id` for correlation.
- The snapshot is complete for the caller at the time the runtime handles the
  request.
- Convention identities MUST be queryless. Parameter descriptors are separate
  metadata and MUST NOT be appended to a convention identity.
- A `currentHandler` identifies the implicit dispatch target. It is not a
  window, process, or running-instance identifier.

### Examples

**Get the current catalog:**

```
-> { "type": "catalog.get", "id": "c1" }
<- {
     "type": "catalog.get.result",
     "id": "c1",
     "snapshot": {
       "napplets": [
         {
           "identity": {
             "dTag": "noteview",
             "aggregateHash": "4de8c5..."
           },
           "title": "Note View",
           "description": "Displays a Nostr note.",
           "requires": ["relay"],
           "archetypes": [
             {
               "archetype": "note",
               "intents": [
                 {
                   "intent": "open",
                   "convention": "napplet:note/open",
                   "parameters": [
                     {
                       "name": "id",
                       "required": true,
                       "description": "Nostr event id."
                     }
                   ]
                 }
               ]
             }
           ]
         }
       ],
       "handlers": [
         {
           "archetype": "note",
           "currentHandler": {
             "dTag": "noteview",
             "aggregateHash": "4de8c5..."
           }
         }
       ]
     }
   }
```

### Error Handling

`catalog.get.result` MAY include `error` when the runtime cannot construct the
catalog. When `error` is present, `snapshot` MUST be omitted.

```
<- { "type": "catalog.get.result", "id": "c1", "error": "catalog unavailable" }
```

## Shell Behavior

- The shell MUST source the catalog from napplet manifests it has resolved and
  verified. It MUST NOT trust identity fields supplied by a running napplet.
- The shell MUST report availability from its catalog, not from currently
  running instances.
- The shell MUST include napplets with no archetype when they are available to
  the caller.
- The shell MUST preserve queryless convention identities exactly. It MUST NOT
  parse, normalize, prefix-match, or wildcard-match them while constructing the
  catalog.
- The shell MUST keep parameter metadata separate from convention identity. It
  MUST NOT report structured payload fields as query parameters.
- The shell MUST include one `ArchetypeHandler` for every archetype advertised
  by the returned napplets. `currentHandler` MUST reference a returned
  `NappletIdentity` or be null.
- The shell MUST respond to every request with a result carrying the same `id`.
- The shell MAY omit metadata it cannot verify.
- The shell MAY scope or redact the catalog according to caller policy.
- The shell MAY enforce ACL checks on catalog access.

## Security Considerations

- A napplet catalog is a fingerprinting surface. The shell SHOULD expose only
  entries and metadata the caller needs.
- Titles, descriptions, intent names, parameter names, and parameter
  descriptions are untrusted display text. Consumers MUST escape them before
  rendering.
- Handler preferences are user state. NAP-CATALOG is read-only and MUST NOT let
  a napplet set or change them.
- Catalog identity comes from verified manifests. A running napplet cannot
  alter its own `dTag` or `aggregateHash` through this interface.
- Parameter metadata does not validate a delivered payload. A receiving
  napplet MUST validate payload data according to the named convention.

## Implementations

- (none yet)

## Changelog
