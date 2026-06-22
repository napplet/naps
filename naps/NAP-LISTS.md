# NAP-LISTS

## Runtime-Mediated NIP-51 List Mutation

`draft`

**NAP ID:** NAP-LISTS
**Domain:** `lists`
**Depends:**
- `identity` — capability · required — list mutations use the current shell-user identity as event author.
- `relay` — capability · required — reads and publishes NIP-51 replaceable and addressable list events.
**Web binding (NIP-5D):** `window.napplet.lists` · `shell.supports("lists")`

## Description

NAP-LISTS lets napplets add and remove items from NIP-51 lists without owning the
dangerous read-modify-sign-publish path for replaceable and addressable events.
The napplet names the list and items. The runtime owns current-event lookup,
kind/type mapping, tag formatting, private item encryption, preservation of
existing content, signing, and publishing.

This NAP does not expose a generic list event editor. It exposes safe mutations
for supported NIP-51 lists only.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `supported` | none | list of `ListSupport` | `lists.supported` / `.result` |
| `add` | `list`, `items`, `options?` | `ListMutationResult` | `lists.add` / `.result` |
| `remove` | `list`, `items`, `options?` | `ListMutationResult` | `lists.remove` / `.result` |

### Schemas

```cddl
ListRef = {
  ? kind: uint,
  ? type: tstr,
  ? identifier: tstr, ; maps to NIP-51 "d" for addressable sets
}

ListItem = {
  itemType: "pubkey" / "event" / "address" / "hashtag" / "word" /
            "relay" / "emoji" / "server" / "url" / "group",
  value: tstr,
  ? relay: tstr,
  ? label: tstr,
  ? visibility: "public" / "private",
}

ListOptions = {
  ? create: bool,
  ? title: tstr,
  ? description: tstr,
  ? image: tstr,
}

ListSupport = {
  kind: uint,
  type: tstr,
  addressable: bool,
  ? supportedItemTypes: [* tstr],
  ? privateItems: bool,
}

ListMutationResult = {
  ok: bool,
  ? eventId: tstr,
  ? event: { * tstr => any },
  ? added: uint,
  ? removed: uint,
  ? skipped: uint,
  ? error: tstr,
  ? reason: tstr,
  ? supported: [* ListSupport],
}
```

Exactly one of `ListRef.kind` or `ListRef.type` MUST be present. Addressable
sets MUST include `identifier`; the runtime writes it as the `"d"` tag.

`type` is derived from the NIP-51 table name: lowercase, punctuation and
whitespace collapsed to hyphens, trimmed. Examples: `Mute list` -> `mute-list`,
`Pinned notes` -> `pinned-notes`, `Read/write relays` -> `read-write-relays`,
`Follow sets` -> `follow-sets`, `Kind mute sets` -> `kind-mute-sets`.
If a derived type maps to more than one NIP-51 kind, the runtime MUST reject the
type-only request with `"ambiguous-list"` and include candidates in `supported`.
The napplet can retry with `kind`.

For `add`, omitted `visibility` means `"public"`.

## Operation Rules

| Operation | Rules |
|-----------|-------|
| `supported` | Returns the NIP-51 list kinds/types this runtime supports by policy and implementation. |
| `add` | Loads the current list event, appends new items in chronological order, preserves unrelated tags/content, signs, publishes. With `create: true`, MAY create a missing list. |
| `remove` | Loads the current list event, removes matching items, preserves unrelated tags/content, signs, publishes. If `visibility` is omitted, removes matching public and private items. |

The runtime maps `ListItem.itemType` to the correct NIP-51 tag for the selected
list. Napplets SHOULD NOT need to know whether the item is encoded as `"p"`,
`"e"`, `"a"`, `"t"`, `"relay"`, or another expected tag.

## NIP-51 Mapping

| Concept | NIP-51 tie |
|---------|------------|
| `kind` | Direct NIP-51 event kind. |
| `type` | Derived from the NIP-51 table `name` cell. |
| `identifier` | NIP-51 `"d"` tag for addressable sets. |
| `visibility: "public"` | Item is written to event `tags`. |
| `visibility: "private"` | Item is written to encrypted `.content` using NIP-44 private items. |

## Wire Protocol

`lists.*` messages use NIP-5D wire format: `{ "type": "domain.action", ...payload }`.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `lists.supported` | napplet -> runtime | `id` |
| `lists.supported.result` | runtime -> napplet | `id`, `lists?`, `error?` |
| `lists.add` | napplet -> runtime | `id`, `list`, `items`, `options?` |
| `lists.add.result` | runtime -> napplet | `id`, `ok`, `eventId?`, `event?`, `added?`, `skipped?`, `error?`, `reason?`, `supported?` |
| `lists.remove` | napplet -> runtime | `id`, `list`, `items`, `options?` |
| `lists.remove.result` | runtime -> napplet | `id`, `ok`, `eventId?`, `event?`, `removed?`, `skipped?`, `error?`, `reason?`, `supported?` |

## Unsupported Lists

When a runtime does not support the requested list kind/type, it MUST return:

```json
{ "ok": false, "error": "unsupported-list", "reason": "...", "supported": [] }
```

`supported` SHOULD include the runtime's supported list set unless policy forbids
revealing it. This is the primary sad path for implementation choice, NIP-51
drift, deprecated list formats, and unknown type names.

## Error Handling

Common errors: `"unsupported-list"`, `"unsupported-item"`, `"invalid-list-ref"`,
`"ambiguous-list"`, `"missing-identifier"`, `"invalid-item"`, `"not-signed-in"`,
`"list-not-found"`, `"list-unavailable"`, `"private-items-unsupported"`,
`"decrypt-failed"`, `"user-denied"`, `"publish-failed"`, `"unsupported"`.

The runtime MUST reject instead of publishing when it cannot read, decrypt, or
preserve existing list state without likely data loss.

## Runtime Behavior

- MUST perform the full read-modify-sign-publish flow.
- MUST preserve unrelated tags, metadata tags, and private items.
- MUST append added items after existing items.
- MUST deduplicate exact item matches.
- MUST validate item types against the selected list's expected NIP-51 tags.
- MUST reject addressable sets without `identifier`.
- MUST use NIP-44 for newly written private items.
- MUST NOT require napplets to provide raw NIP-51 tags.
- MAY prompt, deny, redact returned events, or limit supported lists by policy.

## Security Considerations

- NIP-51 mutations can overwrite replaceable/addressable events. Runtime
  preservation is mandatory because napplet mistakes can lose user data.
- Private items are sensitive. Napplets supply intent; the runtime owns
  encryption, decryption, and failure handling.
- Items, relay hints, URLs, and labels are untrusted input.
- Returning the signed event is optional; success is `ok` plus `eventId`.

## References

- [NIP-51](https://github.com/nostr-protocol/nips/blob/master/51.md) — Lists
- [NIP-44](https://github.com/nostr-protocol/nips/blob/master/44.md) — Versioned Encryption

## Implementations

- None yet.
