NAP-RELAY
=========

Relay Proxy
-----------

`draft`

**NAP ID:** NAP-RELAY
**Domain:** `relay`
**Depends:**
- `resource` — wire · optional — `relay.event.resources?` carries `ResourceSidecarEntry[]`, a type owned by the `resource` domain and imported here (see Sidecar Pre-Resolution).
**Web binding (NIP-5D):** `window.napplet.relay` · `shell.supports("relay")`

## Description

NAP-RELAY provides napplets with relay access through the shell. Sandboxed iframes cannot open WebSocket connections directly because the `allow-same-origin` sandbox token is absent. The shell acts as a relay proxy, forwarding typed relay messages to its connected relay pool and delivering matching events back to the napplet. This is the most fundamental shell capability -- without it, napplets have no way to read or write Nostr events.

## API Surface

```typescript
interface NappletRelay {
  subscribe(                                         // via relay.subscribe
    filters: NostrFilter | NostrFilter[],
    options?: { relay?: string },
  ): Subscription;

  publish(template: EventTemplate): Promise<NostrEvent>;  // via relay.publish

  publishEncrypted(                                  // via relay.publishEncrypted
    template: EventTemplate,
    recipient: string,
    encryption?: 'nip44' | 'nip04',
  ): Promise<NostrEvent>;

  query(                                             // via relay.query
    filters: NostrFilter | NostrFilter[],
  ): Promise<NostrEvent[]>;
}

interface Subscription {
  on(event: 'event', cb: (event: NostrEvent) => void): void;
  on(event: 'eose', cb: () => void): void;
  close(): void;                                     // via relay.close
}
```

**`subscribe(filters, options?)`** -- Opens a live subscription. The shell queries its relay pool, streaming matching events back as `relay.event` messages. Returns a handle to listen for events and EOSE, and to close the subscription. When `options.relay` is provided, the subscription targets a specific relay (e.g., for NIP-29 group relays) instead of the shared pool.

**`publish(template)`** -- Publishes a Nostr event to the shell's relay pool. The shell signs the event template and broadcasts it. Returns the signed event. Napplets do not have direct access to signing keys.

**`publishEncrypted(template, recipient, encryption?)`** -- Publishes an encrypted Nostr event. The shell encrypts the event content using the specified scheme (NIP-44 by default, NIP-04 as fallback), signs the event, and broadcasts it. The napplet provides plaintext content; the shell handles all cryptographic operations. This ensures the shell can inspect content before encryption, preventing exfiltration of encrypted data.

**`query(filters)`** -- Convenience wrapper: subscribes, collects events until EOSE, then closes the subscription and resolves the Promise with the collected events.

All methods are async because they cross the postMessage boundary. Requests include correlation IDs; the shell responds with matching IDs so the shim can resolve the correct Promise or route events to the correct subscription.

## Wire Protocol

Relay operations use the NIP-5D wire format. Requests include an `id` field for correlation. Subscription-scoped messages use a `subId` field to identify which subscription they belong to.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `relay.subscribe` | napplet -> shell | `id`, `subId`, `filters` (NostrFilter array), `relay`? (string) |
| `relay.close` | napplet -> shell | `id`, `subId` |
| `relay.publish` | napplet -> shell | `id`, `event` (EventTemplate) |
| `relay.publishEncrypted` | napplet -> shell | `id`, `event` (EventTemplate), `recipient` (hex pubkey), `encryption`? ('nip44' or 'nip04') |
| `relay.query` | napplet -> shell | `id`, `filters` (NostrFilter array) |
| `relay.event` | shell -> napplet | `subId`, `event` (NostrEvent), `resources`? (ResourceSidecarEntry[]) |
| `relay.eose` | shell -> napplet | `subId` |
| `relay.closed` | shell -> napplet | `subId`, `reason`? (string) |
| `relay.publish.result` | shell -> napplet | `id`, `ok` (boolean), `event`? (NostrEvent), `eventId`? (string), `error`? (string) |
| `relay.publishEncrypted.result` | shell -> napplet | `id`, `ok` (boolean), `event`? (NostrEvent), `eventId`? (string), `error`? (string) |
| `relay.query.result` | shell -> napplet | `id`, `events` (NostrEvent array) |

### NostrFilter

Filters follow the NIP-01 filter format:

```json
{ "kinds": [1], "authors": ["<hex>"], "#e": ["<hex>"], "#p": ["<hex>"], "since": 1234, "until": 5678, "limit": 50 }
```

All filter fields are optional. Multiple filters in the `filters` array are OR-combined.

### Examples

**Subscribe:**
```
-> { "type": "relay.subscribe", "id": "a1", "subId": "sub-1", "filters": [{ "kinds": [1], "limit": 50 }] }
<- { "type": "relay.event", "subId": "sub-1", "event": { "id": "abc...", "pubkey": "def...", "kind": 1, "content": "hello", "tags": [], "created_at": 1234567890, "sig": "..." } }
<- { "type": "relay.eose", "subId": "sub-1" }
```

**Publish (shell signs the event template):**
```
-> { "type": "relay.publish", "id": "b2", "event": { "kind": 1, "content": "hello world", "tags": [], "created_at": 1234567890 } }
<- { "type": "relay.publish.result", "id": "b2", "ok": true, "event": { "id": "abc...", "pubkey": "def...", "kind": 1, "content": "hello world", "tags": [], "created_at": 1234567890, "sig": "ghi..." }, "eventId": "abc..." }
```

**Publish encrypted (shell encrypts and signs):**
```
-> { "type": "relay.publishEncrypted", "id": "f6", "event": { "kind": 4, "content": "secret message", "tags": [["p", "recipient..."]], "created_at": 1234567890 }, "recipient": "recipient...", "encryption": "nip44" }
<- { "type": "relay.publishEncrypted.result", "id": "f6", "ok": true, "event": { "id": "enc...", "pubkey": "def...", "kind": 4, "content": "encrypted...", "tags": [["p", "recipient..."]], "created_at": 1234567890, "sig": "..." }, "eventId": "enc..." }
```

**Query:**
```
-> { "type": "relay.query", "id": "c3", "filters": [{ "authors": ["def..."], "kinds": [0], "limit": 1 }] }
<- { "type": "relay.query.result", "id": "c3", "events": [{ "id": "xyz...", "pubkey": "def...", "kind": 0, "content": "{...}", "tags": [], "created_at": 1234567890, "sig": "..." }] }
```

**Close subscription:**
```
-> { "type": "relay.close", "id": "d4", "subId": "sub-1" }
```

**Scoped relay:**
```
-> { "type": "relay.subscribe", "id": "e5", "subId": "sub-2", "filters": [{ "kinds": [9, 10, 11, 12] }], "relay": "wss://groups.example.com" }
```

**Error in publish:**
```
<- { "type": "relay.publish.result", "id": "b2", "ok": false, "error": "blocked: content policy violation" }
```

### Error Handling

`relay.publish.result` includes `ok: false` and an `error` string on failure. `relay.closed` indicates the shell terminated a subscription, with an optional `reason`. `relay.query.result` MAY include an `error` field instead of `events` if the query fails.

## Sidecar Pre-Resolution

Shells MAY opportunistically pre-resolve byte resources referenced by an event before delivering the event to the subscribing napplet. When a shell has pre-fetched such resources, it MAY include them on the `relay.event` envelope in an optional `resources?: ResourceSidecarEntry[]` field. The napplet's subsequent `resource.bytes(url)` call (per NAP-RESOURCE) resolves from cache without a `postMessage` round-trip when the URL matches a sidecar entry.

Pre-resolution is **OPTIONAL** with **default OFF** for privacy reasons documented below. Conformant shells MUST NOT enable sidecar pre-resolution by default.

### `ResourceSidecarEntry` shape

`ResourceSidecarEntry` is owned by the `resource` domain (NAP-RESOURCE). `relay`
**imports** the type to type its own `resources?` field; it MUST NOT redefine it.
The owning shape is reproduced here for reference only — `resource` is normative:

```typescript
interface ResourceSidecarEntry {
  url: string;       // canonical URL form for this resource
  blob: Blob;        // pre-fetched bytes
  mime: string;      // shell-classified by byte-sniffing -- NEVER upstream Content-Type
}
```

### Wire example (with sidecar)

Shell pre-resolved the author's avatar before delivering a kind 1 event:

```
<- {
     "type": "relay.event",
     "subId": "sub-1",
     "event": {
       "id": "abc...",
       "pubkey": "def...",
       "kind": 1,
       "content": "hello world",
       "tags": [],
       "created_at": 1234567890,
       "sig": "..."
     },
     "resources": [
       {
         "url": "https://example.com/avatar.png",
         "blob": <Blob 4321 bytes>,
         "mime": "image/png"
       }
     ]
   }
```

The field is additive and backward-compatible: shells that omit it produce envelopes that parse identically; napplets that ignore it behave exactly as before. NIP-5D §Wire Format mandates that unrecognized fields are silently ignored.

### Ordering semantics

When `resources` is present and non-empty, conformant napplet shim implementations MUST hydrate their `resource.bytes` single-flight cache (per NAP-RESOURCE) from the entries BEFORE delivering the event to the subscribing napplet's event handler. This ordering is load-bearing: it allows a synchronous `napplet.resource.bytes(url)` lookup inside the napplet's event handler to resolve from cache without a `postMessage` round-trip.

If a `resources` entry's `url` is already present in the cache (e.g., from a prior fetch or a prior sidecar hydration), the shim MUST treat the existing cache entry as authoritative and discard the duplicate sidecar entry. This makes hydration idempotent and prevents a buggy shell from clobbering an in-flight fetch.

### Default OFF privacy rationale

**Conformant shells MUST default sidecar pre-resolution to OFF.** Opt-in is per-shell-policy and SHOULD be configurable by the user (or by the shell deployer for community-deployed shells).

The privacy concerns motivating default-OFF:

- **Pre-fetching reveals user activity to upstream hosts before the napplet has rendered the event.** When a shell pre-fetches the avatar URL on every event in a 1000-event timeline, that becomes 1000 HTTP requests to upstream avatar hosts -- each one a fingerprint visible to the operator of that host (IP address, user-agent, time-of-fetch). The user has not yet chosen to render any of these events; pre-fetching makes the choice for them.
- **Encrypted DM events with embedded image URLs leak "user is online and got the message" telemetry.** A kind 4 / kind 1059 event with an inline `https://attacker.example.com/track.png` URL would, under naive pre-fetching, trigger a fetch as soon as the event arrives at the shell -- before the user opens the DM, possibly even when the user is AFK and would never have opened it.
- **Pre-fetch failure on host-down events forces the napplet to fall back to `resource.bytes()` anyway**, doubling latency and RAM cost compared to fetching only when the napplet actually needs the bytes.
- **Pre-fetch success on never-rendered events occupies shell content-cache memory speculatively.** A user who scrolls past 90% of their timeline has paid for 90% of those fetches in network, RAM, and (worst case) credentials.

Together these concerns make "always on" sidecar pre-resolution privacy-hostile and resource-wasteful. Default OFF is the safe baseline; opt-in is the considered choice.

### Per-event-kind allowlist guidance

Shells that opt in to sidecar pre-resolution SHOULD only pre-fetch URLs that match a per-event-kind allowlist defined in shell policy. Recommended starting policy:

- **Pre-fetch profile picture URLs** from kind 0 (metadata) events for authors the user follows, where the URL host is on a known-good Blossom or avatar-CDN allowlist.
- **Pre-fetch artwork URLs** from kind 31337 / 31938 (long-form / podcast) events the user has explicitly subscribed to.
- **Do NOT pre-fetch arbitrary `https:` URLs from event content** (kind 1, kind 4, kind 1059, etc.) -- this is the dominant fingerprinting vector and accounts for the bulk of "leaked-by-pre-fetch" attack surface.

Shells SHOULD also provide users with control over:

- Which event kinds receive sidecar pre-resolution (per-kind opt-in toggles).
- Which URL hosts are eligible for pre-resolution (host allowlist).
- Which napplets receive sidecar pre-resolution (per-napplet opt-in).

Opt-out at any granularity MUST be honored. A user who opts out of sidecar pre-resolution for a specific napplet, kind, or host MUST receive `relay.event` envelopes with no `resources` field for the matching events; the shell MUST NOT silently downgrade to "we still fetched it but didn't tell you" -- if the user opted out, the fetch MUST NOT have happened.

### Coordination with NAP-RESOURCE

The `mime` field on each sidecar entry MUST be shell-classified by byte-sniffing per the same rules as `resource.bytes.result` in NAP-RESOURCE. Shells MUST NOT populate sidecar `mime` from the upstream `Content-Type` header. SVG entries appearing in a sidecar MUST be rasterized to PNG/WebP per NAP-RESOURCE's SVG Handling rules before being placed on the wire -- the sidecar is not a bypass for the rasterization MUST, the private-IP block list MUST, or any other Default Resource Policy rule. Pre-resolution is "the same fetch, just earlier"; every safety property of `resource.bytes` MUST hold for sidecar entries too.

## Shell Behavior

- The shell MUST forward subscriptions to its relay pool and deliver matching events as `relay.event` messages to the subscribing napplet.
- The shell MUST send `relay.eose` when stored events for a subscription are exhausted.
- The shell MUST honor `relay.close` by tearing down the subscription and ceasing event delivery for that `subId`.
- The shell MUST sign event templates from `relay.publish`, forward the signed event to its relay pool, and respond with `relay.publish.result` containing the signed event.
- The shell MUST encrypt content and sign event templates from `relay.publishEncrypted`, forward the signed encrypted event to its relay pool, and respond with `relay.publishEncrypted.result` containing the signed event.
- The shell MUST decrypt incoming encrypted events (NIP-04/NIP-44) before delivering them to the napplet via `relay.event`. Napplets receive plaintext content.
- The shell MUST respond to `relay.query` by collecting events until EOSE and returning them in `relay.query.result`.
- The shell MUST respond to every request with a result or lifecycle message carrying the same `id` or `subId`.
- The shell MAY send `relay.closed` to indicate a subscription was terminated by the shell (e.g., due to an error, resource limit, or policy).
- The shell MAY enforce ACL checks on relay read and relay write capabilities before processing subscribe or publish messages.
- The shell MAY support scoped relay connections when the `relay` field is present, targeting a specific relay independently from the shared pool.
- The shell MAY manage a relay pool internally -- which relays to connect to, reconnection strategy, and load balancing are implementation details.
- The shell MAY opportunistically pre-resolve byte resources referenced by events and include them in `relay.event.resources`. Sidecar pre-resolution is **default OFF** per the Sidecar Pre-Resolution section; conformant shells MUST NOT enable it by default and SHOULD honor per-napplet / per-kind / per-host opt-out.

## Security Considerations

- The shell controls which relays the napplet can access. Napplets cannot open arbitrary WebSocket connections.
- The shell signs event templates on behalf of the napplet. Napplets do not have direct access to signing keys.
- The shell encrypts content for `publishEncrypted` requests. Napplets provide plaintext content; the shell handles all cryptographic operations. This design prevents a malicious napplet from encrypting exfiltrated data and publishing it to relays where the shell cannot inspect the content.
- The shell MAY inspect event content before signing and publishing. This is the primary security benefit of shell-mediated signing -- the shell can enforce content policies.
- The shell MAY filter or reject subscriptions and publishes based on ACL capabilities.
- For incoming encrypted events (received via subscriptions), the shell decrypts the content before delivering it to the napplet. The napplet never sees ciphertext and never has access to decryption keys.
- Scoped relay URLs SHOULD be validated by the shell to prevent SSRF-like abuse (e.g., napplets targeting internal network addresses via `wss://`).
- Subscription filters SHOULD be validated to prevent resource exhaustion (e.g., unbounded subscriptions without `limit`).
- Sidecar pre-resolution is **default OFF** for privacy reasons (see Sidecar Pre-Resolution § Default OFF privacy rationale). Pre-fetching byte resources referenced by events reveals user activity to upstream hosts before the user has chosen to render the event; this is the dominant fingerprinting vector on a relay-proxy surface and MUST NOT be enabled by default.
