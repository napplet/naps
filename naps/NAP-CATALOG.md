NAP-CATALOG
===========

Runtime Napplet Catalog
-----------------------

`draft`

**NAP ID:** NAP-CATALOG
**Domain:** `catalog`
**Depends:**
- `intent` — layering · optional — when implemented, the runtime shares catalog identifiers and current-handler state with its intent resolver. Catalog queries do not invoke or require the intent capability.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.catalog`; domain presence signals availability.

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
| `kind` | yes | integer | Verified manifest kind: 5129, 15129, or 35129. |
| `pubkey` | yes | text | Manifest publisher's 64-character lowercase-hex public key. |
| `eventId` | yes | text | Verified manifest event id, 64-character lowercase hex. |
| `dTag` | conditional | text | Manifest `d` value; present only for kind 35129. |
| `artifactHash` | yes | text | Verified artifact `x` hash, 64-character lowercase hex. |

`IntentDescriptor` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `intent` | yes | text | Intent name, such as `open`. |
| `convention` | yes | text | Stable queryless convention identity, such as `napplet:note/open`. |
| `parameters` | yes | list of text | Parameter names advertised in the manifest i tag, in order. Empty when none are advertised. |

`ArchetypeDescriptor` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `archetype` | yes | text | NAAT role slug, such as `note`. |
| `intents` | yes | list of `IntentDescriptor` | Accepted intents for this role. |

`NappletDescriptor` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | text | Opaque runtime-assigned napplet selector; see Catalog selectors. |
| `identity` | yes | `NappletIdentity` | Verified napplet identity. |
| `title` | no | text | Human-readable title. |
| `description` | yes | text | Non-empty plain-text manifest content. |
| `requires` | yes | list of text | Bare domains from manifest R tags. |
| `optional` | yes | list of text | Bare domains from manifest O tags. |
| `archetypes` | yes | list of `ArchetypeDescriptor` | Advertised roles. Empty for a napplet with no archetype. |

`ArchetypeHandler` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `archetype` | yes | text | NAAT role slug. |
| `currentHandler` | yes | text or null | `id` of a returned napplet selected for implicit dispatch, or null when no such target is available to report. |

`CatalogSnapshot` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `napplets` | yes | list of `NappletDescriptor` | Napplets available to the caller. |
| `handlers` | yes | list of `ArchetypeHandler` | Current handler state for every archetype advertised by the returned napplets. |

### Catalog selectors

Every returned napplet MUST have a unique `id` within the runtime's catalog.
The runtime MUST NOT reuse an identifier for a different napplet during the
runtime session. The identifier is opaque: consumers MUST preserve it exactly
and MUST NOT derive a selector from `identity.dTag` or any other identity field.
The structured `identity` describes the verified version; it is not a selector.

When the runtime implements `intent`, selectors MUST use the
[NAP-INTENT catalog identifier contract](NAP-INTENT.md#catalog-identifiers).
For each napplet returned by both domains, the catalog `id` MUST equal its
`IntentCandidate.id`. This does not require role-less entries to appear in
intent discovery.
A caller MAY pass it unchanged as an explicit `handler` to `intent.invoke`
when that capability is exposed and the user authorizes the selection.
Receiving a catalog entry or `currentHandler` MUST NOT grant that authorization.
Catalog snapshots do not reserve a target, pin its artifact version, or exempt
later invocations from convention compatibility and policy checks.

A non-null `currentHandler` MUST equal exactly one returned entry's `id`.
If the selected entry is omitted, the runtime MUST report null, not substitute
another napplet. Without an intent resolver, handler values MUST be null;
napplet metadata remains queryable. A caller that needs to preserve the reported
target MUST use the explicit selector, not repeat implicit dispatch or fall
back to a `d` tag if selection fails.

### Manifest mapping

The catalog MUST use verified [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)
manifests of kinds 5129, 15129, and 35129. It MUST derive roles from `z` tags and
accepted conventions and parameter names from `i` tags. Each intent is associated
with the role matching its archetype segment on the same manifest, independent
of tag order. Invalid or unmatched intents contribute no dispatch contract;
roles with no matching intent have an empty `intents` list. Convention identities
remain queryless and fragment-free. Legacy combined archetype tags do not
substitute for these advertisements.

Trailing `i` elements are names only. The runtime MUST NOT infer parameter types,
requiredness, descriptions, or event-kind constraints from them. Payload semantics
belong to the named convention. An empty parameter list does not prohibit an
explicit payload. Titles come from `title`; descriptions come from plain-text
manifest `content`. Required and optional domain lists come from `R` and `O`.
Advertisements and capability declarations MUST NOT widen runtime grants. The
runtime MUST inspect the complete required set when assessing compatibility;
missing optional domains alone MUST NOT exclude a napplet.

Identity MUST distinguish publishers and manifest kinds. Named identities are
keyed by publisher, kind, and `d`; root identities by publisher and kind;
snapshot identities by signed event id. `eventId` and `artifactHash` describe the
resolved version. Catalogs MUST NOT require `dTag` on root or snapshot entries.

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
           "id": "catalog-noteview",
           "identity": {
             "kind": 35129,
             "pubkey": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
             "eventId": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
             "dTag": "noteview",
             "artifactHash": "cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc"
           },
           "title": "Note View",
           "description": "Displays a Nostr note.",
           "requires": ["relay"],
           "optional": ["theme"],
           "archetypes": [
             {
               "archetype": "note",
               "intents": [
                 {
                   "intent": "open",
                   "convention": "napplet:note/open",
                   "parameters": ["id"]
                 }
               ]
             }
           ]
         }
       ],
       "handlers": [
         {
           "archetype": "note",
           "currentHandler": "catalog-noteview"
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
  normalize, prefix-match, or wildcard-match them. Parsing an intent to validate
  its archetype segment MUST NOT change its stored identity.
- The shell MUST keep parameter metadata separate from convention identity. It
  MUST NOT report structured payload fields as query parameters.
- The shell MUST include one `ArchetypeHandler` for every archetype advertised
  by the returned napplets. `currentHandler` MUST reference a returned
  napplet's `id` or be null, as specified in Catalog selectors.
- The shell MUST respond to every request with a result carrying the same `id`.
- The shell MUST return every required field of each included record. When a
  required field cannot be verified or disclosed under caller policy, it MUST
  omit the entire napplet entry. It MUST NOT return a partial descriptor or
  fabricate a required value. This includes `dTag` for kind 35129; `dTag` MUST
  be absent for kinds 5129 and 15129.
- The shell MAY omit optional fields it cannot verify or disclose. Absence of
  optional manifest declarations produces the specified empty lists, not omitted
  fields or an invalid entry.
- The shell MAY scope or redact the catalog according to caller policy, subject
  to the required-field and handler-reference rules above. It MUST construct
  `handlers` from the entries that remain after filtering.
- The shell MAY enforce ACL checks on catalog access.

## Security Considerations

- A napplet catalog is a fingerprinting surface. The shell SHOULD expose only
  entries and metadata the caller needs.
- Titles, descriptions, intent names, and parameter names are untrusted display
  text. Consumers MUST escape them before
  rendering.
- Handler preferences are user state. NAP-CATALOG is read-only and MUST NOT let
  a napplet set or change them.
- Catalog identity comes from verified manifests. A running napplet cannot
  alter its verified manifest fields or `artifactHash` through this interface.
- Parameter metadata does not validate a delivered payload. A receiving
  napplet MUST validate payload data according to the named convention.

## Implementations

- (none yet)

## Changelog

- `5711b2e` - Defined read-only napplet discovery, convention parameter metadata, and current archetype handler state.

- `1c076ce` - Aligned catalog identities, z/i advertisements, parameter-name metadata, content descriptions, and R/O capabilities with current manifests.
- `b696bef` - Added opaque selectors shared with intent resolution, changed currentHandler to reference entry IDs, declared optional intent layering, and required complete descriptors with consistent filtering.
