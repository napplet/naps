NAP-COMMON
==========

Common Social Actions
---------------------

`draft`

**NAP ID:** NAP-COMMON
**Domain:** `common`
**Depends:**
- `identity` — capability · required — user-authoritative actions use the current shell-user identity as the event author.
- `relay` — capability · required — fetches kind 0 profiles and publishes kind 3 follow lists, kind 7 reactions, and kind 1984 reports.
**Web binding (NIP-5D):** `window.napplet.common` · `shell.supports("common")`

## Description

NAP-COMMON gives napplets a small shell-mediated surface for common Nostr
identifier helpers, profile lookup, and social actions: encodeNip19,
decodeNip19, getProfile, follow, unfollow, react, and report. The modifying
actions are user identity operations, not napplet-owned event construction. The
napplet supplies intent and minimal targets; the shell owns identity, consent,
event construction, signing, replacement-list merging, and publishing.

This NAP is deliberately narrow. It does not expose a generic event builder, a
follow-list editor, moderation policy, or relay selection API. Napplets that
need full event control use NAP-RELAY. Napplets that need outbox-aware fanout
use NAP-OUTBOX when available. NAP-COMMON is the simple path for user-visible
identifier displays, profile cards, and social buttons that should behave
consistently across napplets.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `encodeNip19` | `input` (`CommonNip19EncodeInput`) | `CommonNip19EncodeResult` | `common.encodeNip19` / `common.encodeNip19.result` |
| `decodeNip19` | `value` (`tstr`) | `CommonNip19DecodeResult` | `common.decodeNip19` / `common.decodeNip19.result` |
| `getProfile` | `target` (`CommonProfileTarget`) | `CommonProfileResult` | `common.getProfile` / `common.getProfile.result` |
| `follow` | one or more `npub` targets | `CommonActionResult` | `common.follow` / `common.follow.result` |
| `unfollow` | one or more `npub` targets | `CommonActionResult` | `common.unfollow` / `common.unfollow.result` |
| `react` | `id` (`tstr`), `reaction` (`CommonReaction`), optional `customEmojiHref` (`tstr`) | `CommonActionResult` | `common.react` / `common.react.result` |
| `report` | `target` (`CommonReportTarget`), `reason` (`CommonReportReason`), `text` (`tstr`) | `CommonActionResult` | `common.report` / `common.report.result` |

### Schemas

```cddl
Npub = tstr
Nprofile = tstr
HexPubkey = tstr
NostrEventId = tstr
NostrEvent = { * tstr => any }
CommonPubkeyList = [Npub, * Npub]
CommonProfileTarget = HexPubkey / Npub / Nprofile

CommonNip19Type = "npub" / "note" / "nprofile" / "nevent" / "naddr" / "nrelay"

CommonNip19EncodeInput =
  CommonNip19BareEncodeInput /
  CommonNip19ProfileEncodeInput /
  CommonNip19EventEncodeInput /
  CommonNip19AddressEncodeInput /
  CommonNip19RelayEncodeInput

CommonNip19BareEncodeInput = {
  type: "npub" / "note",
  hex: tstr,
}

CommonNip19ProfileEncodeInput = {
  type: "nprofile",
  pubkey: HexPubkey,
  ? relays: [* tstr],
}

CommonNip19EventEncodeInput = {
  type: "nevent",
  eventId: NostrEventId,
  ? relays: [* tstr],
  ? author: HexPubkey,
  ? kind: uint,
}

CommonNip19AddressEncodeInput = {
  type: "naddr",
  identifier: tstr,
  pubkey: HexPubkey,
  kind: uint,
  ? relays: [* tstr],
}

CommonNip19RelayEncodeInput = {
  type: "nrelay",
  relay: tstr,
}

CommonNip19EncodeResult = {
  ok: bool,
  ? value: tstr,
  ? type: CommonNip19Type,
  ? error: tstr,
}

CommonNip19DecodeResult = {
  ok: bool,
  ? type: CommonNip19Type,
  ? hex: tstr,
  ? pubkey: HexPubkey,
  ? eventId: NostrEventId,
  ? identifier: tstr,
  ? relays: [* tstr],
  ? author: HexPubkey,
  ? kind: uint,
  ? relay: tstr,
  ? error: tstr,
}

CommonProfileData = {
  ? name: tstr,
  ? displayName: tstr,
  ? about: tstr,
  ? picture: tstr,
  ? banner: tstr,
  ? nip05: tstr,
  ? lud16: tstr,
  ? website: tstr,
  * tstr => any,
}

CommonProfileResult = {
  ok: bool,
  pubkey: HexPubkey,
  ? profile: CommonProfileData / null,
  ? event: NostrEvent,
  ? relays: [* tstr],
  ? error: tstr,
}

CommonActionResult = {
  ok: bool,
  ? eventId: tstr,
  ? event: NostrEvent,
  ? error: tstr,
}

CommonReaction = "+" / "-" / tstr

CommonReportReason =
  "nudity" /
  "malware" /
  "profanity" /
  "illegal" /
  "spam" /
  "impersonation" /
  "other"

CommonReportTarget = CommonEventReportTarget / CommonPubkeyReportTarget

CommonEventReportTarget = {
  type: "event",
  id: tstr,
  ? pubkey: tstr,
  ? relay: tstr,
}

CommonPubkeyReportTarget = {
  type: "pubkey",
  pubkey: Npub / HexPubkey,
  ? relay: tstr,
}

CommonGetProfileRequest = {
  id: tstr,
  target: CommonProfileTarget,
}

CommonFollowRequest = {
  id: tstr,
  pubkeys: CommonPubkeyList,
}

CommonReactRequest = {
  id: tstr,
  targetEventId: NostrEventId,
  reaction: CommonReaction,
  ? customEmojiHref: tstr,
}

CommonReportRequest = {
  id: tstr,
  target: CommonReportTarget,
  reason: CommonReportReason,
  text: tstr,
}
```

**`encodeNip19(input)`** -- Encodes public Nostr identifiers into NIP-19 display
strings. The shell accepts structured inputs for `npub`, `note`, `nprofile`,
`nevent`, `naddr`, and `nrelay`, validates their fields, and returns the encoded
bech32 string. `nprofile`, `nevent`, and `naddr` encode relay hints as NIP-19
TLV `relay` entries when supplied.

**`decodeNip19(value)`** -- Decodes a public NIP-19 display string into a
structured result. The shell supports `npub`, `note`, `nprofile`, `nevent`,
`naddr`, and `nrelay`. It MUST reject `nsec` because private-key material is not
a public common helper and MUST NOT cross the napplet/shell seam. Unknown TLV
entries are ignored, as required by NIP-19.

**`getProfile(hexPubkey|npub|nprofile)`** -- Resolves a public profile for one
Nostr public key. The shell accepts a 32-byte hex public key, a NIP-19 `npub`, or
a NIP-19 `nprofile`. The shell normalizes the target to a hex public key, uses
any `nprofile` relay TLVs as relay hints, fetches the latest kind 0 metadata
event it can find, parses its JSON content, and returns the normalized pubkey,
profile data, optional source event, and optional relays used. If no profile is
found, the shell returns `ok: true` with `profile: null`.

**`follow(...npub)`** -- Adds one or more public keys to the shell-user's NIP-02
follow list. The shell decodes each NIP-19 `npub` to a hex public key, loads the
current kind 3 follow list if available, appends new `p` tags in chronological
order, preserves existing tags it does not need to change, signs the replacement
kind 3 event, and publishes it. Duplicate follows are idempotent.

**`unfollow(...npub)`** -- Removes one or more public keys from the shell-user's
NIP-02 follow list. The shell decodes each `npub`, removes matching `p` tags
from the current kind 3 follow list, preserves existing tags it does not need to
change, signs the replacement kind 3 event, and publishes it. Removing a key
that is not followed is idempotent.

**`react(id, reaction, customEmojiHref?)`** -- Publishes a NIP-25 reaction to a
Nostr event id. `reaction` is normally `"+"`, `"-"`, a single Unicode emoji, or a
NIP-30 custom emoji shortcode such as `":soapbox:"`. When `customEmojiHref` is
present, `reaction` MUST be one custom emoji shortcode, and the shell includes
one `emoji` tag using that shortcode and URL. The shell SHOULD add `e`, `p`, and
`k` tags when it can resolve the target event metadata. NAP-COMMON reacts only
to native Nostr events, so it always publishes kind 7 rather than NIP-25
external-content kind 17 reactions.

**`report(target, reason, text)`** -- Publishes a NIP-56 kind 1984 report. For a
profile report, `target.type` is `"pubkey"` and the shell emits a `p` tag whose
third entry is `reason`. For an event report, `target.type` is `"event"` and the
shell emits an `e` tag whose third entry is `reason` and a `p` tag for the event
author. If the event author is not supplied, the shell MUST resolve it before
publishing or reject the request. `text` becomes the report event content.

SDKs MAY expose shorthand signatures matching the operation names above, but
wire messages MUST use the structured payloads below so event reports and
profile reports are unambiguous.

For `report(id|pubkey, reason, text)` shorthands, SDKs MUST map the target into
`CommonReportTarget` before sending the wire message. A Nostr event id maps to
`{ "type": "event", "id": ... }`; an `npub` maps to
`{ "type": "pubkey", "pubkey": ... }`.

## NIP Mapping

| Action | Event shape |
|--------|-------------|
| `encodeNip19` | NIP-19 bech32 display encoding for public identifiers |
| `decodeNip19` | NIP-19 bech32 / TLV decoding for public identifiers |
| `getProfile` | NIP-01 replaceable metadata: latest kind 0 by target pubkey |
| `follow` | NIP-02 replacement follow list: kind 3, `p` tags, empty content |
| `unfollow` | NIP-02 replacement follow list with matching `p` tags removed |
| `react` | NIP-25 reaction: kind 7, `content` set to reaction, target `e` tag; optional NIP-30 `emoji` tag |
| `report` | NIP-56 report: kind 1984, `p` tag, optional `e` tag for event reports, content set to `text` |

## Wire Protocol

`common.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `common.encodeNip19` | napplet -> shell | `id`, `input` |
| `common.encodeNip19.result` | shell -> napplet | `id`, `ok`, `value?`, `type?`, `error?` |
| `common.decodeNip19` | napplet -> shell | `id`, `value` |
| `common.decodeNip19.result` | shell -> napplet | `id`, `ok`, `type?`, `hex?`, `pubkey?`, `eventId?`, `identifier?`, `relays?`, `author?`, `kind?`, `relay?`, `error?` |
| `common.getProfile` | napplet -> shell | `id`, `target` |
| `common.getProfile.result` | shell -> napplet | `id`, `ok`, `pubkey`, `profile?`, `event?`, `relays?`, `error?` |
| `common.follow` | napplet -> shell | `id`, `pubkeys` |
| `common.follow.result` | shell -> napplet | `id`, `ok`, `eventId?`, `event?`, `error?` |
| `common.unfollow` | napplet -> shell | `id`, `pubkeys` |
| `common.unfollow.result` | shell -> napplet | `id`, `ok`, `eventId?`, `event?`, `error?` |
| `common.react` | napplet -> shell | `id`, `targetEventId`, `reaction`, `customEmojiHref?` |
| `common.react.result` | shell -> napplet | `id`, `ok`, `eventId?`, `event?`, `error?` |
| `common.report` | napplet -> shell | `id`, `target`, `reason`, `text` |
| `common.report.result` | shell -> napplet | `id`, `ok`, `eventId?`, `event?`, `error?` |

Key design notes:
- Request/result pairs use `id` for correlation.
- `encodeNip19` and `decodeNip19` are public identifier helpers only. They MUST
  NOT accept or return `nsec`.
- `decodeNip19` MUST ignore unknown TLV entries rather than failing the whole
  decode.
- `decodeNip19` SHOULD reject bech32 strings longer than 5000 characters.
- `target` for `common.getProfile` MUST be a 32-byte hex public key, NIP-19
  `npub`, or NIP-19 `nprofile`. `nprofile` relay TLVs are hints only; the shell
  MAY use, ignore, reorder, or supplement them by policy.
- `common.getProfile.result` returns `profile: null` when no kind 0 profile is
  found. This is not an error.
- `pubkeys` MUST be a non-empty list of NIP-19 `npub` strings. Shells MAY accept
  32-byte hex public keys as a compatibility extension, but MUST publish hex
  keys in Nostr events.
- `reaction` MUST be `"+"`, `"-"`, a single Unicode emoji, or one NIP-30
  shortcode. If `customEmojiHref` is present, `reaction` MUST be exactly one
  `:shortcode:` and the published event MUST contain exactly one matching
  `emoji` tag whose name omits the surrounding colons.
- `target` MUST be structured. Event report shorthands are SDK conveniences, not
  wire format.
- The shell MAY return the signed event in `event` for optimistic UI or audit
  display. Callers MUST key success on `ok` and `eventId`, not on `event`.
- The shell MAY publish through direct relay fanout, outbox-aware fanout, or
  another runtime policy path. That routing choice is not visible to napplets.

### Examples

**Encode npub:**

```
-> { "type": "common.encodeNip19", "id": "n1", "input": { "type": "npub", "hex": "3bf0c63fcb93463407af97a5e5ee64fa883d107ef9e558472c4eb9aaaefa459d" } }
<- { "type": "common.encodeNip19.result", "id": "n1", "ok": true, "type": "npub", "value": "npub1..." }
```

**Decode nprofile:**

```
-> { "type": "common.decodeNip19", "id": "n2", "value": "nprofile1..." }
<- {
     "type": "common.decodeNip19.result",
     "id": "n2",
     "ok": true,
     "type": "nprofile",
     "pubkey": "3bf0c63fcb93463407af97a5e5ee64fa883d107ef9e558472c4eb9aaaefa459d",
     "relays": ["wss://relay.example.com"]
   }
```

**Get profile from nprofile:**

```
-> { "type": "common.getProfile", "id": "p1", "target": "nprofile1..." }
<- {
     "type": "common.getProfile.result",
     "id": "p1",
     "ok": true,
     "pubkey": "3bf0c63fcb93463407af97a5e5ee64fa883d107ef9e558472c4eb9aaaefa459d",
     "profile": { "name": "Alice", "picture": "https://example.com/alice.png" },
     "relays": ["wss://relay.example.com"]
   }
```

**Follow:**

```
-> { "type": "common.follow", "id": "f1", "pubkeys": ["npub1..."] }
<- { "type": "common.follow.result", "id": "f1", "ok": true, "eventId": "abc..." }
```

**React with custom emoji:**

```
-> {
     "type": "common.react",
     "id": "r1",
     "targetEventId": "abc...",
     "reaction": ":soapbox:",
     "customEmojiHref": "https://example.com/soapbox.png"
   }
<- { "type": "common.react.result", "id": "r1", "ok": true, "eventId": "def..." }
```

**Report an event:**

```
-> {
     "type": "common.report",
     "id": "x1",
     "target": { "type": "event", "id": "abc...", "pubkey": "def..." },
     "reason": "spam",
     "text": "Repeated unsolicited replies"
   }
<- { "type": "common.report.result", "id": "x1", "ok": true, "eventId": "987..." }
```

### Error Handling

Result messages include `ok: false` and an `error` string when the shell cannot
complete the action. Common errors are `"not-signed-in"`, `"invalid-pubkey"`,
`"invalid-nip19"`, `"unsupported-nip19-type"`, `"invalid-profile-target"`,
`"invalid-target"`, `"invalid-reaction"`, `"invalid-report-reason"`,
`"author-unresolved"`, `"user-denied"`, `"relay-timeout"`, `"publish-failed"`, and
`"unsupported"`.

For `report` event targets, `"author-unresolved"` means the shell could not
determine the `p` tag required by NIP-56. The shell MUST NOT publish an event
report without the required `p` tag.

## Shell Behavior

- The shell MUST reject NIP-19 helper requests that encode or decode `nsec`.
- The shell MUST normalize NIP-19 helper results to hex fields for keys and
  event ids.
- The shell MUST resolve `getProfile` targets to hex public keys before querying
  relays.
- The shell MUST query kind 0 replaceable metadata for `getProfile`.
- The shell MUST perform all signing. Napplets MUST NOT receive signing keys.
- The shell MUST reject modifying operations when no shell-user signer is
  connected. `getProfile` MAY succeed without a connected signer.
- The shell MUST publish NIP-02 follow-list updates as replacement kind 3 events.
- The shell SHOULD preserve existing follow-list `p` tag relay and petname
  fields when it can.
- The shell MUST publish reactions as kind 7 events for native Nostr event
  targets.
- The shell MUST publish reports as kind 1984 events and MUST use only NIP-56
  report reasons.
- The shell MAY prompt the user before any action.
- The shell MAY rate-limit or deny repeated actions by policy.

## Security Considerations

Common social actions are user-authoritative. A malicious napplet could attempt
to trick the user into following accounts, reacting to spam, or reporting
legitimate users.

- Shell prompts SHOULD show the requesting napplet, stable target identifiers,
  action, and any human-readable report text.
- Shells SHOULD apply anti-spam and duplicate-action policy before publishing.
- Shells SHOULD treat napplet-supplied report text, custom emoji URLs, and relay
  hints as untrusted input.
- Shells SHOULD treat `nprofile` relay hints as advisory. A malicious napplet can
  choose slow, hostile, or privacy-invasive relays.
- Napplets SHOULD treat profile metadata as untrusted display data.
- Napplets MUST NOT treat a successful NAP-COMMON action as proof that relays,
  clients, or users accepted the social meaning of the event.

## References

- [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) — Basic protocol flow and kind 0 metadata
- [NIP-02](https://github.com/nostr-protocol/nips/blob/master/02.md) — Follow List
- [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) — bech32-encoded entities
- [NIP-25](https://github.com/nostr-protocol/nips/blob/master/25.md) — Reactions
- [NIP-30](https://github.com/nostr-protocol/nips/blob/master/30.md) — Custom Emoji
- [NIP-56](https://github.com/nostr-protocol/nips/blob/master/56.md) — Reporting

## Implementations

- None yet.
