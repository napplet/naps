NAP-IDENTITY
============

Read-Only User Identity Queries
--------------------------------

`draft`

**NAP ID:** NAP-IDENTITY
**Domain:** `identity`
**Depends:**
- `resource` — capability · optional — `picture` / `banner` URL bytes are fetched via `resource.bytes`
**Web binding (NIP-5D):** `window.napplet.identity` · `shell.supports("identity")`

## Description

NAP-IDENTITY provides read-only access to the shell-user identity: the currently connected user/signer pubkey and related public identity data. Napplets that need to display the user's profile, check their follow list, or read their relay preferences query the shell for this data. The shell resolves these queries against its local cache or connected relays and returns the results.

Napplets do not have direct access to the user's private key. They cannot sign events, encrypt, or decrypt. Identity queries are strictly read-only -- napplets learn *about* the user but cannot act *as* the user. Signing is delegated to the shell via `relay.publish()` (NAP-RELAY). Encryption is delegated via `relay.publishEncrypted()`.

This shell-user identity is distinct from the NIP-5D napplet session identity. The session identity is assigned by the shell at iframe creation from the NIP-5A `(dTag, aggregateHash)` / `MessageEvent.source` binding and is never negotiated by the napplet. NAP-IDENTITY only reports the user's connected signer identity.

## API Surface

```typescript
interface NappletIdentity {
  getPublicKey(): Promise<string>;
  onChanged(handler: (pubkey: string) => void): Subscription;
  getRelays(): Promise<Record<string, { read: boolean; write: boolean }>>;
  getProfile(): Promise<ProfileData | null>;
  getFollows(): Promise<string[]>;
  getList(type: string): Promise<string[]>;
  getZaps(): Promise<ZapReceipt[]>;
  getMutes(): Promise<string[]>;
  getBlocked(): Promise<string[]>;
  getBadges(): Promise<Badge[]>;
}

interface ProfileData {
  name?: string;
  displayName?: string;
  about?: string;
  picture?: string;
  banner?: string;
  nip05?: string;
  lud16?: string;
  website?: string;
}
```

**Resource resolution.** The `picture` and `banner` fields are URL strings. Napplets that need the bytes (for example, to render an `<img>` via an object URL) MUST fetch them through NAP-RESOURCE: `window.napplet.resource.bytes(url)`. Napplets MUST NOT attempt direct `<img src="https://...">` loads — sandboxed napplets cannot make direct network requests under the iframe sandbox model defined by NIP-5D (`sandbox="allow-scripts"`, no `allow-same-origin`). Conformant shells expose every external byte resource through NAP-RESOURCE, including profile pictures and banners. The shell applies the standard NAP-RESOURCE policy to these fetches (private-IP block list at DNS-resolution time, MIME byte-sniffing, optional SVG rasterization, etc.).

```typescript

interface ZapReceipt {
  eventId: string;
  sender: string;
  amount: number;
  content?: string;
}

interface Badge {
  id: string;
  name?: string;
  description?: string;
  image?: string;
  thumbs?: string[];
  awardedBy: string;
}

interface Subscription {
  close(): void;
}
```

**`getPublicKey()`** -- Returns the user's hex-encoded public key, or the empty string when no user/signer is connected. This is the most basic identity query. Every shell that implements NAP-IDENTITY MUST support this method.

**`onChanged(handler)`** -- Registers a handler for shell-pushed user identity changes. The handler receives the same public-key string shape as `getPublicKey()`: a non-empty hex pubkey when a user/signer is connected, or `""` when the shell signs out or disconnects the signer.

**`getRelays()`** -- Returns the user's relay list as a record mapping relay URLs to read/write permissions (NIP-65 relay list metadata).

**`getProfile()`** -- Returns the user's kind 0 profile metadata. Returns `null` if no profile is found. The shell resolves this from its cache or relays.

**`getFollows()`** -- Returns an array of hex-encoded public keys from the user's kind 3 contact list.

**`getList(type)`** -- Returns entries from a user's categorized list. The `type` parameter specifies the list kind (e.g., `"bookmarks"`, `"interests"`, `"pins"`). Returns the tag values from the matching parameterized replaceable event.

**`getZaps()`** -- Returns zap receipts (kind 9735) sent to the user. Each receipt includes the sender, amount in millisats, and optional content.

**`getMutes()`** -- Returns the user's mute list (kind 10000) as an array of hex-encoded public keys.

**`getBlocked()`** -- Returns the user's block list as an array of hex-encoded public keys. Semantically distinct from mutes -- blocks are hard exclusions.

**`getBadges()`** -- Returns badges awarded to the user (NIP-58). Each badge includes the badge definition ID, metadata, and the awarding pubkey.

## Wire Protocol

`identity.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `identity.getPublicKey` | napplet -> shell | `id` |
| `identity.getPublicKey.result` | shell -> napplet | `id`, `pubkey` |
| `identity.changed` | shell -> napplet | `pubkey` |
| `identity.getRelays` | napplet -> shell | `id` |
| `identity.getRelays.result` | shell -> napplet | `id`, `relays`, `error?` |
| `identity.getProfile` | napplet -> shell | `id` |
| `identity.getProfile.result` | shell -> napplet | `id`, `profile?`, `error?` |
| `identity.getFollows` | napplet -> shell | `id` |
| `identity.getFollows.result` | shell -> napplet | `id`, `pubkeys`, `error?` |
| `identity.getList` | napplet -> shell | `id`, `listType` |
| `identity.getList.result` | shell -> napplet | `id`, `entries`, `error?` |
| `identity.getZaps` | napplet -> shell | `id` |
| `identity.getZaps.result` | shell -> napplet | `id`, `zaps`, `error?` |
| `identity.getMutes` | napplet -> shell | `id` |
| `identity.getMutes.result` | shell -> napplet | `id`, `pubkeys`, `error?` |
| `identity.getBlocked` | napplet -> shell | `id` |
| `identity.getBlocked.result` | shell -> napplet | `id`, `pubkeys`, `error?` |
| `identity.getBadges` | napplet -> shell | `id` |
| `identity.getBadges.result` | shell -> napplet | `id`, `badges`, `error?` |

Key design notes:
- All methods are request/result pairs using `id` for correlation.
- `identity.changed` is a push message. It has no `id` and is not a response to a request.
- All results include an optional `error` field for failure cases.
- `getPublicKey` is the simplest query and MUST always succeed (no `error` field). It returns `pubkey: ""` when no shell-user identity is currently connected.
- `getProfile` returns `profile: null` if no kind 0 is found -- this is not an error.
- `getList` includes a `listType` field to specify which categorized list to query.

Recommended handshake:

1. A napplet that needs user identity calls `identity.getPublicKey` once on startup to get a snapshot.
2. The shell returns `identity.getPublicKey.result` with `pubkey` set to either a hex pubkey or `""`.
3. The napplet then listens for `identity.changed` and re-runs identity-dependent work when the pushed pubkey changes.

Napplets SHOULD NOT poll `identity.getPublicKey` waiting for a signer to appear. Shells MUST use `identity.changed` to notify already-loaded napplets when the shell-user identity changes.

### Examples

**Get public key:**
```
-> { "type": "identity.getPublicKey", "id": "q1" }
<- { "type": "identity.getPublicKey.result", "id": "q1", "pubkey": "ab12cd..." }
```

**No user connected:**
```
-> { "type": "identity.getPublicKey", "id": "q1b" }
<- { "type": "identity.getPublicKey.result", "id": "q1b", "pubkey": "" }
```

**User identity changed:**
```
<- { "type": "identity.changed", "pubkey": "ab12cd..." }
<- { "type": "identity.changed", "pubkey": "" }
```

**Get profile:**
```
-> { "type": "identity.getProfile", "id": "q2" }
<- { "type": "identity.getProfile.result", "id": "q2", "profile": { "name": "Alice", "about": "Nostr dev", "picture": "https://..." } }
```

**Profile not found:**
```
-> { "type": "identity.getProfile", "id": "q3" }
<- { "type": "identity.getProfile.result", "id": "q3", "profile": null }
```

**Get follows:**
```
-> { "type": "identity.getFollows", "id": "q4" }
<- { "type": "identity.getFollows.result", "id": "q4", "pubkeys": ["ab12...", "cd34...", "ef56..."] }
```

**Get relay list:**
```
-> { "type": "identity.getRelays", "id": "q5" }
<- { "type": "identity.getRelays.result", "id": "q5", "relays": { "wss://relay.example.com": { "read": true, "write": true }, "wss://readonly.example.com": { "read": true, "write": false } } }
```

**Get categorized list:**
```
-> { "type": "identity.getList", "id": "q6", "listType": "bookmarks" }
<- { "type": "identity.getList.result", "id": "q6", "entries": ["note1abc...", "note1def..."] }
```

**Get mutes:**
```
-> { "type": "identity.getMutes", "id": "q7" }
<- { "type": "identity.getMutes.result", "id": "q7", "pubkeys": ["spam1...", "spam2..."] }
```

**Get zap receipts:**
```
-> { "type": "identity.getZaps", "id": "q8" }
<- { "type": "identity.getZaps.result", "id": "q8", "zaps": [{ "eventId": "ev1...", "sender": "ab12...", "amount": 21000, "content": "Great post!" }] }
```

**Get badges:**
```
-> { "type": "identity.getBadges", "id": "q9" }
<- { "type": "identity.getBadges.result", "id": "q9", "badges": [{ "id": "badge1...", "name": "Early Adopter", "awardedBy": "cd34..." }] }
```

**Error case:**
```
-> { "type": "identity.getFollows", "id": "q10" }
<- { "type": "identity.getFollows.result", "id": "q10", "pubkeys": [], "error": "relay timeout" }
```

### Error Handling

Result messages include an `error` field (string) when the shell cannot fulfill the request. Common errors: `"relay timeout"`, `"not found"`, `"unsupported list type"`. When `error` is present, the primary result field contains a sensible default (empty array, null, etc.).

`identity.getPublicKey.result` MUST NOT include an `error` field. The shell returns `pubkey: ""` when no user/signer is connected.

## Shell Behavior

- The shell MUST respond to every `identity.*` request with the corresponding `*.result` message carrying the same `id`.
- The shell MUST return `pubkey` for `identity.getPublicKey` without error. If no user/signer is connected, `pubkey` MUST be the empty string.
- The shell MUST emit `identity.changed` to loaded napplets whenever the shell-user identity changes.
- The shell MUST emit `identity.changed` with `pubkey: ""` when the shell-user identity is cleared, the signer disconnects, or the user signs out.
- The shell MAY emit an initial `identity.changed` after napplet load, but napplets MUST NOT depend on that initial push and SHOULD use `identity.getPublicKey` for the startup snapshot.
- The shell SHOULD resolve identity queries from its local cache when available.
- The shell MAY fetch data from relays if not cached.
- The shell MAY return stale data with a best-effort freshness policy.
- The shell MAY enforce ACL checks on identity capabilities (e.g., restricting which napplets can query follows or mutes).
- The shell MAY return empty results for queries it does not support (e.g., badges) rather than an error.
- The shell MUST NOT expose the user's private key or any signing capability through this interface.
- The shell MUST NOT provide encrypt or decrypt operations through this interface.

## Security Considerations

- NAP-IDENTITY is strictly read-only. No method modifies user state, signs events, or performs cryptographic operations.
- `identity.changed` reports only the shell-user identity. It does not change or renegotiate the napplet's NIP-5D session identity.
- The user's public key is not secret, but follow lists, mute lists, and block lists reveal social graph information. Shells MAY restrict access to sensitive lists based on napplet trust level.
- Zap receipts reveal financial information. Shells SHOULD consider whether to expose zap data to all napplets or restrict it to trusted napplets.
- Profile data may contain URLs (picture, banner, website). Napplets that render these URLs should sanitize them. The shell is not responsible for sanitizing profile content.
- This interface intentionally excludes signing, encryption, and decryption. These operations are security-critical and belong to the shell. See NAP-RELAY for shell-mediated publish and encrypted publish.
- Block lists and mute lists are sensitive. A malicious napplet could use this data to target blocked users through side channels. Shells MAY redact or restrict access to these lists.

## Implementations

- (none yet)
