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

NAP-COMMON is a small shell-mediated surface for public Nostr identifier helpers,
profile lookup, and common social actions. It does not expose generic event
construction, relay selection, moderation policy, or private-key material.

The napplet supplies intent and minimal targets. The shell owns user identity,
consent, signing, replacement-list merging, publishing, lookup policy, and NIP
normalization.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `encodeNip19` | `input` (`CommonNip19EncodeInput`) | `CommonNip19EncodeResult` | `common.encodeNip19` / `.result` |
| `decodeNip19` | `value` (`tstr`) | `CommonNip19DecodeResult` | `common.decodeNip19` / `.result` |
| `getProfile` | `target` (`CommonProfileTarget`) | `CommonProfileResult` | `common.getProfile` / `.result` |
| `follow` | one or more `npub` targets | `CommonActionResult` | `common.follow` / `.result` |
| `unfollow` | one or more `npub` targets | `CommonActionResult` | `common.unfollow` / `.result` |
| `react` | `id`, `reaction`, `customEmojiHref?` | `CommonActionResult` | `common.react` / `.result` |
| `report` | `target`, `reason`, `text` | `CommonActionResult` | `common.report` / `.result` |

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
  { type: "npub" / "note", hex: tstr } /
  { type: "nprofile", pubkey: HexPubkey, ? relays: [* tstr] } /
  { type: "nevent", eventId: NostrEventId, ? relays: [* tstr], ? author: HexPubkey, ? kind: uint } /
  { type: "naddr", identifier: tstr, pubkey: HexPubkey, kind: uint, ? relays: [* tstr] } /
  { type: "nrelay", relay: tstr }

CommonNip19EncodeResult = {
  ok: bool,
  ? value: tstr,
  ? nip19Type: CommonNip19Type,
  ? error: tstr,
}

CommonNip19DecodeResult = {
  ok: bool,
  ? nip19Type: CommonNip19Type,
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
CommonReportReason = "nudity" / "malware" / "profanity" / "illegal" / "spam" / "impersonation" / "other"
CommonReportTarget =
  { type: "event", id: tstr, ? pubkey: tstr, ? relay: tstr } /
  { type: "pubkey", pubkey: Npub / HexPubkey, ? relay: tstr }
```

### Operation Rules

| Operation | Rules |
|-----------|-------|
| `encodeNip19` | Encodes public NIP-19 identifiers: `npub`, `note`, `nprofile`, `nevent`, `naddr`, `nrelay`. MUST reject `nsec`. |
| `decodeNip19` | Decodes the same public types. MUST reject `nsec`, ignore unknown TLVs, and SHOULD reject strings longer than 5000 chars. |
| `getProfile` | Accepts hex pubkey, `npub`, or `nprofile`. Normalizes to hex, treats `nprofile` relay TLVs as hints, queries latest kind 0, returns `profile: null` if not found. |
| `follow` | Decodes each `npub`, merges new `p` tags into the current NIP-02 kind 3 list, preserves unrelated tags, signs, publishes. Idempotent. |
| `unfollow` | Decodes each `npub`, removes matching `p` tags from the current kind 3 list, preserves unrelated tags, signs, publishes. Idempotent. |
| `react` | Publishes NIP-25 kind 7 for native Nostr events only. `reaction` is `+`, `-`, one Unicode emoji, or one NIP-30 shortcode. `customEmojiHref` requires one matching `emoji` tag. |
| `report` | Publishes NIP-56 kind 1984. Pubkey reports use `p` with reason. Event reports use `e` with reason plus author `p`; shell MUST resolve or reject missing authors. |

SDKs MAY expose `report(id|pubkey, reason, text)` shorthand, but wire messages
MUST use `CommonReportTarget`.

## NIP Mapping

| Operation | NIP |
|-----------|-----|
| `encodeNip19`, `decodeNip19` | NIP-19 public bech32 / TLV identifiers |
| `getProfile` | NIP-01 latest kind 0 by pubkey |
| `follow`, `unfollow` | NIP-02 replacement kind 3 follow list |
| `react` | NIP-25 kind 7; optional NIP-30 `emoji` tag |
| `report` | NIP-56 kind 1984 |

## Wire Protocol

`common.*` messages use NIP-5D wire format: `{ "type": "domain.action", ...payload }`.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `common.encodeNip19` | napplet -> shell | `id`, `input` |
| `common.encodeNip19.result` | shell -> napplet | `id`, `ok`, `value?`, `nip19Type?`, `error?` |
| `common.decodeNip19` | napplet -> shell | `id`, `value` |
| `common.decodeNip19.result` | shell -> napplet | `id`, `ok`, `nip19Type?`, `hex?`, `pubkey?`, `eventId?`, `identifier?`, `relays?`, `author?`, `kind?`, `relay?`, `error?` |
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

## Error Handling

Result messages use `ok: false` plus `error`. Common errors:
`"not-signed-in"`, `"invalid-pubkey"`, `"invalid-nip19"`,
`"unsupported-nip19-type"`, `"invalid-profile-target"`, `"invalid-target"`,
`"invalid-reaction"`, `"invalid-report-reason"`, `"author-unresolved"`,
`"user-denied"`, `"relay-timeout"`, `"publish-failed"`, `"unsupported"`.

## Shell Behavior

- MUST reject NIP-19 helper requests that encode or decode `nsec`.
- MUST normalize NIP-19 helper results to hex fields for keys and event ids.
- MUST resolve `getProfile` targets to hex before querying relays.
- MUST query kind 0 replaceable metadata for `getProfile`.
- MUST perform all signing and never expose signing keys.
- MUST reject modifying operations when no shell-user signer is connected; `getProfile` MAY work signed out.
- MUST publish follow changes as NIP-02 replacement kind 3 events.
- MUST publish reactions as kind 7 events for native Nostr event targets.
- MUST publish reports as kind 1984 events with NIP-56 reasons.
- MAY prompt, rate-limit, deny, redact, or route by runtime policy.

## Security Considerations

- Treat NIP-19 values, relay hints, report text, custom emoji URLs, and profile metadata as untrusted.
- Treat `nprofile` relays as advisory; napplets can choose hostile relays.
- Prompts SHOULD show requesting napplet, action, stable targets, and report text.
- Success means the shell published or returned data. It does not prove relay, client, or user acceptance.

## References

- [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md) — Basic protocol flow and kind 0 metadata
- [NIP-02](https://github.com/nostr-protocol/nips/blob/master/02.md) — Follow List
- [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) — bech32-encoded entities
- [NIP-25](https://github.com/nostr-protocol/nips/blob/master/25.md) — Reactions
- [NIP-30](https://github.com/nostr-protocol/nips/blob/master/30.md) — Custom Emoji
- [NIP-56](https://github.com/nostr-protocol/nips/blob/master/56.md) — Reporting

## Implementations

- None yet.
