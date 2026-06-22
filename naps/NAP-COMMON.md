NAP-COMMON
==========

Common Social Actions
---------------------

`draft`

**NAP ID:** NAP-COMMON
**Domain:** `common`
**Depends:**
- `identity` — capability · required — actions use the current shell-user identity as the event author.
- `relay` — capability · required — publishes kind 3 follow lists, kind 7 reactions, and kind 1984 reports.
**Web binding (NIP-5D):** `window.napplet.common` · `shell.supports("common")`

## Description

NAP-COMMON gives napplets a small shell-mediated surface for common Nostr social
actions: follow, unfollow, react, and report. These actions are user identity
operations, not napplet-owned event construction. The napplet supplies intent
and minimal targets; the shell owns identity, consent, event construction,
signing, replacement-list merging, and publishing.

This NAP is deliberately narrow. It does not expose a generic event builder, a
follow-list editor, moderation policy, or relay selection API. Napplets that need
full event control use NAP-RELAY. Napplets that need outbox-aware fanout use
NAP-OUTBOX when available. NAP-COMMON is the simple path for user-visible social
buttons that should behave consistently across napplets.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `follow` | one or more `npub` targets | `CommonActionResult` | `common.follow` / `common.follow.result` |
| `unfollow` | one or more `npub` targets | `CommonActionResult` | `common.unfollow` / `common.unfollow.result` |
| `react` | `id` (`tstr`), `reaction` (`CommonReaction`), optional `customEmojiHref` (`tstr`) | `CommonActionResult` | `common.react` / `common.react.result` |
| `report` | `target` (`CommonReportTarget`), `reason` (`CommonReportReason`), `text` (`tstr`) | `CommonActionResult` | `common.report` / `common.report.result` |

### Schemas

```cddl
Npub = tstr
NostrEventId = tstr
NostrEvent = { * tstr => any }
CommonPubkeyList = [Npub, * Npub]

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
  pubkey: Npub,
  ? relay: tstr,
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
| `follow` | NIP-02 replacement follow list: kind 3, `p` tags, empty content |
| `unfollow` | NIP-02 replacement follow list with matching `p` tags removed |
| `react` | NIP-25 reaction: kind 7, `content` set to reaction, target `e` tag; optional NIP-30 `emoji` tag |
| `report` | NIP-56 report: kind 1984, `p` tag, optional `e` tag for event reports, content set to `text` |

## Wire Protocol

`common.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
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
`"invalid-target"`, `"invalid-reaction"`, `"invalid-report-reason"`,
`"author-unresolved"`, `"user-denied"`, `"publish-failed"`, and
`"unsupported"`.

For `report` event targets, `"author-unresolved"` means the shell could not
determine the `p` tag required by NIP-56. The shell MUST NOT publish an event
report without the required `p` tag.

## Shell Behavior

- The shell MUST perform all signing. Napplets MUST NOT receive signing keys.
- The shell MUST reject every operation when no shell-user signer is connected.
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
- Napplets MUST NOT treat a successful NAP-COMMON action as proof that relays,
  clients, or users accepted the social meaning of the event.

## References

- [NIP-02](https://github.com/nostr-protocol/nips/blob/master/02.md) — Follow List
- [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) — bech32-encoded entities
- [NIP-25](https://github.com/nostr-protocol/nips/blob/master/25.md) — Reactions
- [NIP-30](https://github.com/nostr-protocol/nips/blob/master/30.md) — Custom Emoji
- [NIP-56](https://github.com/nostr-protocol/nips/blob/master/56.md) — Reporting

## Implementations

- None yet.
