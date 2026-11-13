NAP-CVM
========

ContextVM Bridge
----------------

`draft`

**NAP ID:** NAP-CVM
**Domain:** `cvm`
**Depends:**
- `value` — layering · optional — the shell MAY reuse its `value` payment backend for supported ContextVM payment methods; no `value` wire types cross the `cvm` seam.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.cvm`; domain presence signals availability.

## Description

NAP-CVM provides napplets with native access to ContextVM servers through the shell. ContextVM transports Model Context Protocol (MCP) messages over Nostr relays using public-key addressing and encrypted relay messages. A napplet could build this itself on top of NAP-RELAY, but doing so would force every napplet to implement ContextVM discovery, initialization, JSON-RPC correlation, relay routing, encryption, signing, and optional payment handling. NAP-CVM moves that legwork into the runtime.

The napplet supplies the server identity and the MCP operation it wants. The shell handles ContextVM transport details, user policy, signing, encryption, relay access, and value-transfer prompts.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `discover` | optional `query` (`CvmDiscoverQuery`) | list of `CvmServer` | `cvm.discover` / `cvm.discover.result` |
| `request` | `server` (`CvmServerRef`), `message` (`McpMessage`), optional `options` (`CvmRequestOptions`) | `McpMessage` or `null` | `cvm.request` / `cvm.request.result` |
| `listTools` | `server` (`CvmServerRef`), optional `options` (`CvmRequestOptions`) | list of `McpTool` | wrapper over `request` |
| `callTool` | `server` (`CvmServerRef`), `name` (text), optional `args` (`JsonObject`), optional `options` (`CvmRequestOptions`) | `McpToolResult` | wrapper over `request` |
| `listResources` | `server` (`CvmServerRef`), optional `options` (`CvmRequestOptions`) | list of `McpResource` | wrapper over `request` |
| `readResource` | `server` (`CvmServerRef`), `uri` (text), optional `options` (`CvmRequestOptions`) | list of `McpResourceContent` | wrapper over `request` |
| `close` | `server` (`CvmServerRef`) | none | `cvm.close` / `cvm.close.result` |
| `onEvent` | handler for `CvmEvent` | `Subscription` handle | `cvm.event` |
| `registry.list` | optional `query` (`CvmRegistryQuery`) | list of `CvmRegistryEntry` | `cvm.registry.list` / result |
| `registry.has` | `selector` (`CvmRegistrySelector`), optional `options` (`CvmRegistryOptions`) | boolean | `cvm.registry.has` / result |
| `registry.describe` | `selector` (`CvmRegistrySelector`), optional `options` (`CvmRegistryOptions`) | `CvmRegistryEntry` | `cvm.registry.describe` / result |
| `registry.call` | `selector` (`CvmRegistrySelector`), `tool` (text), optional `args` (`JsonObject`), optional `options` (`CvmRegistryCallOptions`) | `McpToolResult` | `cvm.registry.call` / result |

### Schemas

Primitive references:

| Name | Meaning |
|------|---------|
| `JsonObject` | JSON object. |
| `JsonSchema` | JSON Schema object. |
| `McpTool` | MCP tool object. |
| `McpToolResult` | MCP tool result object. |
| `McpResource` | MCP resource object. |
| `McpResourceContent` | MCP resource content object. |
| `Subscription` | Handle whose `close()` operation removes the local event handler; it does not close a server session. |

Enumerations:

| Name | Values |
|------|--------|
| `CvmRequestOptions.payment` | `deny`, `prompt`, `allow` |
| `CvmRegistryCallOptions.payment` | `deny`, `prompt`, `allow` |
| `CvmRegistryCallOptions.cache` | `default`, `reload`, `no-store` |
| `McpMessage.jsonrpc` | `"2.0"` |

`CvmServerRef`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `pubkey` | text | yes | ContextVM server pubkey as 64 lowercase hexadecimal characters. |
| `relays` | list of text | no | Relay hints for the server. |

`CvmDiscoverQuery`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `search` | text | no | Search string. |
| `kinds` | list of integer | no | Nonnegative announcement kinds to inspect; omitted means all supported announcement kinds. |
| `relays` | list of text | no | Relay hints for discovery. |
| `limit` | integer | no | Positive maximum number of servers to return. |

`CvmServer`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `pubkey` | text | yes | ContextVM server pubkey. |
| `relays` | list of text | no | Relay hints for the server. |
| `name` | text | no | Human-readable server name. |
| `description` | text | no | Human-readable server description. |
| `capabilities` | list of text | no | MCP server capability names, such as `tools`, `resources`, or `prompts`; never tool names or common-schema hashes. Omit when unknown. |
| `paymentRequired` | boolean | no | Advisory shell-derived pricing hint; see Payment handling. Omit when unknown. |

`CvmRequestOptions`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `timeoutMs` | integer | no | Positive timeout in milliseconds, at most 300000; default 30000. See Timeouts. |
| `initialize` | boolean | no | Whether the shell should initialize the MCP session if needed. |
| `payment` | `deny`, `prompt`, or `allow` | no | Requested payment behavior; default `prompt`. Never grants spending authority. See Payment handling. |

`CvmRegistryQuery`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `search` | text | no | Search string. |
| `family` | text | no | Shell-local family label. |
| `schemaHash` | text | no | Required CEP-15 tool schema hash; portable matching key. |
| `category` | text | no | Best-effort CEP-15 `t` category hint. |
| `limit` | integer | no | Positive maximum number of entries to return. |

`CvmRegistrySelector`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `schemaHash` | text | no | CEP-15 tool identity; authoritative when present. |
| `family` | text | no | Shell-local label used only when `schemaHash` is absent. |

At least one selector field MUST be present. A supplied `schemaHash` MUST be a
64-character lowercase hexadecimal SHA-256 digest. A family label MUST be
nonempty. Hash and label value constraints also apply when those fields are
supplied in registry queries.

`CvmRegistryOptions`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `server` | `CvmServerRef` | no | Required provider server. |

`CvmRegistryCallOptions`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `server` | `CvmServerRef` | no | Required provider server. |
| `timeoutMs` | integer | no | Positive timeout in milliseconds, at most 300000; default 30000. See Timeouts. |
| `initialize` | boolean | no | Whether the shell should initialize the MCP session if needed. |
| `payment` | `deny`, `prompt`, or `allow` | no | Requested payment behavior; default `prompt`. Never grants spending authority. See Payment handling. |
| `cache` | `default`, `reload`, or `no-store` | no | Registry cache policy. |

`CvmRegistryEntry`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `family` | text | no | Shell-local display label; not portable identity. |
| `description` | text | no | Human-readable family description. |
| `schemaHashes` | list of text | yes | Distinct CEP-15 hashes of all entries in `tools`; order has no meaning. |
| `selected` | `CvmServerRef` | no | Selected provider implementing the complete tool set, if verified and reachable. |
| `providers` | list of `CvmServerRef` | no | Candidate servers advertising the complete tool set; not proof of liveness. |
| `tools` | list of `CvmRegistryTool` | yes | Nonempty tool set described by this entry. |

`CvmRegistryTool`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | text | yes | Tool name. |
| `description` | text | no | Human-readable tool description. |
| `inputSchema` | `JsonSchema` | yes | Tool input schema. |
| `outputSchema` | `JsonSchema` | no | Tool output schema. |
| `schemaHash` | text | yes | Verified CEP-15 hash for this tool, including its name. |

`McpMessage`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `jsonrpc` | `"2.0"` | yes | JSON-RPC version. |
| `id` | text or integer | no | JSON-RPC correlation ID; never `null`. See Message handling. |
| `method` | text | no | JSON-RPC method name. |
| `params` | `JsonObject` | no | MCP parameters. |
| `result` | `JsonObject` | no | MCP result. |
| `error` | `JsonObject` | no | MCP error with integer `code`, text `message`, and optional `data`. |

`CvmEvent`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `server` | `CvmServerRef` | yes | Server that emitted the event. |
| `message` | `McpMessage` | yes | MCP message delivered by the server. |

### Discovery

`discover(query?)` returns public ContextVM server announcements known to the
shell. Sources MAY include configured relays, cached announcements, and curated
lists. Discovery is not proof of reachability or permission to call a server.

The shell MUST honor supplied `kinds` and `limit` filters, including for cached
results. An empty `kinds` list returns no servers; an unsupported kind returns
`"unsupported kind"`, rather than silently broadening the search. `search` is a
best-effort text filter. Relay hints remain subject to shell policy.

`CvmServer.capabilities` names MCP capabilities learned from initialization or
[CEP-6 announcements](https://docs.contextvm.org/reference/ceps/cep-6/), including
capability-list content. A tools announcement can establish `tools` even without
CEP-15 tags. The shell MUST NOT substitute `i`-tag hashes or tool names for MCP
capability names. It SHOULD resolve verified
[CEP-17 relay lists](https://docs.contextvm.org/reference/ceps/cep-17/) when
available, respecting their direction markers and local relay policy. Discovery
relay locations alone do not establish a server's operational relay set.

### Message handling

`request(server, message, options?)` sends one MCP JSON-RPC message. The shell
MUST validate it before publishing. Missing messages, batches, invalid IDs, and
invalid field combinations return `"invalid message"`. The forms follow
[MCP](https://modelcontextprotocol.io/specification/2025-11-25/basic):

| MCP form | Fields | Completion of the enclosing `cvm.request` |
|----------|--------|-------------------------------------------|
| Request | `method` and `id`; no `result` or `error` | Matching MCP response in `message`. |
| Notification | `method`, no `id`, `result`, or `error` | After transport publication, result without `message`; API returns `null`. No MCP response is awaited. |
| Response | `id`, exactly one of `result` or `error`; no `method` or `params` | After transport publication, result without `message`; API returns `null`. No further MCP response is awaited. |

The shell MUST preserve the supplied MCP message's form and payload. It MUST
NOT add an ID to a notification or rewrite the ID of a napplet's response to a
server request. For outgoing requests only, it MAY map IDs to unique transport
IDs if it restores the caller's ID in the returned MCP response and consistently
maps request references such as cancellation. Publication completion does not
prove remote processing. The envelope ID correlates the napplet-to-shell
exchange only.

Correlations MUST include the authenticated napplet endpoint, server, session,
and direction. Identical IDs from different napplets MUST NOT collide. An ID
already used for a client request in the same MCP session returns
`"invalid message"`; a response must match an outstanding server request
delivered to that napplet. An unmatched response returns `"invalid message"`.
`notifications/cancelled` may refer only to this napplet's requests. Shell-owned
initialization, payment messages, and internal retries MUST use isolated
correlation state; raw messages MUST NOT bypass shell policy.

`listTools`, `callTool`, `listResources`, and `readResource` wrap the corresponding
MCP operations using the same request, timeout, and payment rules. `readResource`
MUST return the complete ordered `contents` list from
[MCP resources/read](https://modelcontextprotocol.io/specification/2025-11-25/server/resources),
including every text or blob entry. It MUST NOT select only the first entry.

### Sessions and events

Session state is scoped to an authenticated napplet endpoint and server. An
accepted outbound MCP request or notification establishes that endpoint's
participation; `discover` and `onEvent` alone do not open a session. The shell
MAY initialize automatically and SHOULD initialize when requested or required
by the server. Shared transport resources MUST preserve this isolation.

`close(server)` releases only the calling napplet endpoint's subscriptions,
initialization state, pending correlations, and session authorizations. It MUST
complete its pending operations with `"session closed"` before acknowledging the
close. It MUST NOT interrupt another napplet's requests or subscriptions. Shared
transport teardown is permitted only when no other endpoint needs it. Closing an
already closed session succeeds. Endpoint destruction performs the same cleanup;
local cleanup does not promise remote cancellation or a payment refund.

`onEvent(handler)` registers for MCP notifications and server-initiated requests
from this endpoint's open sessions. The shell MUST deliver each `cvm.event` only
to its owning endpoint; the binding invokes that endpoint's registered handlers.
It MUST NOT broadcast session messages to unrelated napplets. Notifications
correlated to a request, such as progress, use that request's owner. Payment
control messages are handled by the shell under Payment handling.

A napplet SHOULD register its handler before opening a session. Registration
does not create transport state. Without a handler, the binding MAY discard
notifications; it MUST NOT redirect messages to another napplet. Server requests
still require a response or expire under session policy. Removing a handler via
`Subscription.close()` does not close the server session.

### Client identity

The shell selects and controls the ContextVM signing key. Fresh ephemeral keys
per napplet endpoint and shell session are RECOMMENDED. The ContextVM client
pubkey is not the user's Nostr identity and MUST NOT be presented as such.
[CEP-16 clientPubkey](https://docs.contextvm.org/reference/ceps/informational/cep-16/)
identifies this transport signer, not the human user.

The shell MUST isolate session negotiation, inbound server requests, paid
execution authorizations, and allowance accounting between napplets. If remote
state is keyed only by client pubkey, the shell MUST use distinct client keys
where sharing would let one napplet consume another's authorization or receive
its messages. Sharing a relay connection does not authorize sharing that state.
Closing a session MUST prevent its remaining authorization from being reused by
another napplet, even if the server retains state after local closure.

### Timeouts

`timeoutMs` defaults to 30000 and MUST be an integer from 1 through 300000.
Out-of-range or malformed values return `"invalid timeout"` before publication.
The shell MAY impose a lower policy maximum, but MUST reject a requested duration
above that maximum with `"timeout out of policy"`; it MUST NOT silently clamp it.

The shell MUST enforce the accepted duration from receipt through queueing,
initialization, consent, payment, and final response or publication. Retries and
progress events MUST NOT reset the deadline. Expiry returns `"relay timeout"`,
releases the pending correlation, and prevents further automatic spending or
retries for that operation. A late response MUST NOT produce a second result or
be redelivered as `cvm.event`. Timeout does not prove remote cancellation or undo
an already initiated transfer.

Projection bindings MUST honor the requested duration and permit delivery of the
shell's final result after its deadline, accounting for projection delivery
latency. In particular, a fixed 30-second binding timer MUST NOT cut off an
accepted 120-second request. This applies equally to convenience wrappers and
`registry.call`; other request/result operations use the default duration.

### Payment handling

Payment processing is optional; payment policy enforcement is mandatory.
[CEP-8](https://docs.contextvm.org/reference/ceps/cep-8/) owns pricing, payment
methods, negotiation, correlation, and settlement messages. The shell derives
`paymentRequired` from verified pricing information: `true` when a positive
`cap` price is advertised, `false` only when known advertised capabilities are
explicitly free, and omitted when pricing is unknown. It is an advisory hint,
not a quote or guarantee about any invocation. `pmi` advertises methods, not a
price; missing pricing or a `pmi` tag alone MUST NOT imply free service.

`payment` defaults to `prompt`. It constrains the shell as follows:

| Option | Settlement authority | Requested CEP-8 session mode |
|--------|----------------------|-----------------------------|
| `deny` | No new payment or consumption of paid authorization for this invocation. Free calls remain usable. | `explicit_gating` preferred. |
| `prompt` | Explicit user consent for this invocation, with server, operation, amount, and payment method shown by the shell. | `explicit_gating` preferred. |
| `allow` | Only a standing user allowance configured outside the napplet, covering this napplet, server, operation, method, amount, and remaining budget. No allowance means `"payment denied"` when payment is needed. | `transparent` permitted; `explicit_gating` also permitted. |

The shell MUST negotiate and track the effective `payment_interaction` at the
session level, following CEP-8 and
[CEP-35](https://docs.contextvm.org/reference/ceps/informational/cep-35/). A request
option is not a per-message transport mode. Mode changes MUST follow upstream
negotiation and MUST NOT change the lifecycle of another in-flight operation.
The shell MAY retain the current mode while enforcing each invocation's policy.
If explicit gating is rejected, transparent fallback is permitted only when the
shell can enforce the same consent and budget constraints. Otherwise the call
fails with `"payment required"`; fallback MUST NOT grant spending authority.

The shell MUST intercept and correlate `-32042` payment gates and
`notifications/payment_required` before delivering application results. It MUST
verify the server and the owning invocation, validate the offered amount and
method, and apply the table above. These transport messages MUST NOT become a
napplet-controlled payment path. A shell without a compatible payment handler
returns `"payment required"` promptly; a policy prohibition or user refusal
returns `"payment denied"`. Neither outcome may hang awaiting a payment.
The same policy applies to already-paid execution authorizations: the shell
MUST check ownership and consent before sending an invocation that could consume
one, not just before making a new transfer.

In transparent mode the shell settles an authorized offer and awaits the
original response. In explicit gating it settles one authorized offer, then
retries the same method and params under the same client identity as prescribed
by CEP-8, using a fresh internal request ID and restoring the caller's ID in the
final response. It MUST track payment-pending responses and paid authorizations
without duplicate settlement. It MUST NOT blindly retry after an uncertain execution:
a paid authorization does not guarantee replay of a lost result. Every retry
remains within the original timeout and consent or allowance. Concurrent calls
MUST reserve allowance budget atomically. A napplet MUST NOT borrow another
napplet's paid authorization, including through an otherwise identical call.

The shell MAY reuse a compatible `value` backend internally. Exposing `value` to
the napplet is not required, and its presence does not guarantee support for a
server's payment method. Unsupported payment methods fail as above; this NAP
does not define new value rails. Signing keys and settlement credentials remain
outside the napplet.

### Registry identity and matching

A registry family is a set of CEP-15 tool schema hashes. Its `family` name is a
shell-local label, not portable identity. `CvmRegistryEntry.schemaHashes` MUST
equal the distinct hashes of its `tools`. There is no aggregate family
`schemaHash`. Hashes are computed and verified by the shell according to
[CEP-15](https://docs.contextvm.org/reference/ceps/cep-15/); NAP-CVM defines no
alternative hashing or schema-version algorithm.

`registry.list(query?)` returns known entries. `category` uses CEP-15 `t` tags as
best-effort discovery hints. A `schemaHash` query finds the matching tool across
local family names and returns singleton tool entries. Without a hash, `family`
filters by local label. Search and category filters further narrow candidates;
the positive `limit`, when present, bounds the returned count.

`registry.has(selector, options?)` returns whether a permitted, verified,
recently reachable provider satisfies the selector and optional server pin.
It returns `false` for no match or an unavailable registry implementation.
`registry.describe` returns the matching entry and schemas; no match is
`"schema mismatch"` for a hash selector or `"family not found"` for a label.

A selector's `schemaHash` is authoritative even if its optional `family` label
is unknown or different on this shell. Hash selection returns a singleton tool
entry and MUST NOT require registration under any local family name. Without a
hash, the shell selects the complete tool set for the named family. `selected`
and `providers` refer to providers of that complete returned set. Shells MUST
verify tool hashes before exposing them as matching schemas; announcements are
candidates, not proof of schema compatibility or liveness.

`registry.call(selector, tool, args?, options?)` invokes MCP `tools/call` under the
same policy as `callTool`. The requested tool name MUST match the selected tool's
verified schema hash; mismatch returns `"schema mismatch"` before publication.
A supplied server pin MUST be honored; no other provider may substitute. The
shell MAY select a different provider only before dispatch and only with the
same verified tool hash and no server pin. An uncertain outcome after dispatch
MUST NOT trigger automatic replay on another provider.

The shell MAY cache discovery, schemas, and results under user policy. Result
cache keys MUST include the tool schema hash, arguments, selected server, and
napplet policy scope. Cache hits MUST pass the current policy checks. The cache
option is `default` (local policy), `reload` (bypass stored results, may store the
fresh result), or `no-store` (neither read nor write result cache). Discovery and
schema caches are independent. A prohibited cache choice returns
`"cache policy denied"`.

## Wire Protocol

`cvm.*` messages use the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `cvm.discover` | napplet -> shell | `id`, `query?` |
| `cvm.discover.result` | shell -> napplet | `id`, `servers?`, `error?` |
| `cvm.request` | napplet -> shell | `id`, `server`, `message`, `options?` |
| `cvm.request.result` | shell -> napplet | `id`, `message?`, `error?` |
| `cvm.close` | napplet -> shell | `id`, `server` |
| `cvm.close.result` | shell -> napplet | `id`, `error?` |
| `cvm.event` | shell -> napplet | `server`, `message` |
| `cvm.registry.list` | napplet -> shell | `id`, `query?` |
| `cvm.registry.list.result` | shell -> napplet | `id`, `entries?`, `error?` |
| `cvm.registry.has` | napplet -> shell | `id`, `selector`, `options?` |
| `cvm.registry.has.result` | shell -> napplet | `id`, `has?`, `error?` |
| `cvm.registry.describe` | napplet -> shell | `id`, `selector`, `options?` |
| `cvm.registry.describe.result` | shell -> napplet | `id`, `entry?`, `error?` |
| `cvm.registry.call` | napplet -> shell | `id`, `selector`, `tool`, `args?`, `options?` |
| `cvm.registry.call.result` | shell -> napplet | `id`, `result?`, `error?` |

Key design notes:
- `cvm.request.result` correlates to the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) request `id`; the embedded MCP message retains its own JSON-RPC `id`.
- `cvm.event` carries no envelope `id`. Delivery is scoped to the owning napplet endpoint and open session as defined above.
- The shell owns all ContextVM event IDs, relay subscriptions, `p` tags, `e` tags, encryption state, and signing.
- The shell MAY initialize a ContextVM session automatically before the first capability request.
- Registry messages are shell-level conveniences. They do not define new ContextVM server behavior.
- Registry selectors pin a tool with `selector.schemaHash`; returned entries enumerate the tool set in `schemaHashes`. Family labels do not override hash matching.

### Examples

Hashes and pubkeys in these examples are abbreviated for readability; actual
requests use complete values.

**Discover servers:**
```
-> { "type": "cvm.discover", "id": "d1", "query": { "search": "relay", "limit": 5 } }
<- {
     "type": "cvm.discover.result",
     "id": "d1",
     "servers": [
       {
         "pubkey": "65a334...",
         "relays": ["wss://relay.example.com"],
         "name": "RelayVM",
         "description": "Relay intelligence exposed through MCP over Nostr"
       }
     ]
   }
```

**Call an MCP tool through ContextVM:**
```
-> {
     "type": "cvm.request",
     "id": "r1",
     "server": { "pubkey": "65a334...", "relays": ["wss://relay.example.com"] },
     "message": {
       "jsonrpc": "2.0",
       "id": 2,
       "method": "tools/call",
       "params": { "name": "get_relay", "arguments": { "url": "wss://relay.example.com" }, "_meta": { "progressToken": "tool-2" } }
     },
     "options": { "initialize": true, "payment": "prompt" }
   }
<- {
     "type": "cvm.request.result",
     "id": "r1",
     "message": {
       "jsonrpc": "2.0",
       "id": 2,
       "result": { "content": [{ "type": "text", "text": "{...}" }], "isError": false }
     }
   }
```

**Server notification:**
```
<- {
     "type": "cvm.event",
     "server": { "pubkey": "65a334..." },
     "message": { "jsonrpc": "2.0", "method": "notifications/progress", "params": { "progressToken": "tool-2", "progress": 50 } }
   }
```

**Browse a local family, then call by portable tool hash:**

```
-> { "type": "cvm.registry.has", "id": "h1", "selector": { "family": "relatr" } }
<- { "type": "cvm.registry.has.result", "id": "h1", "has": true }

-> { "type": "cvm.registry.describe", "id": "s1", "selector": { "family": "relatr" } }
<- {
     "type": "cvm.registry.describe.result",
     "id": "s1",
     "entry": {
       "family": "relatr",
       "schemaHashes": ["a7f3d9..."],
       "selected": { "pubkey": "65a334...", "relays": ["wss://relay.example.com"] },
       "tools": [
         {
           "name": "search_profiles",
           "schemaHash": "a7f3d9...",
           "inputSchema": {
             "type": "object",
             "properties": { "q": { "type": "string" } },
             "required": ["q"]
           }
         }
       ]
     }
   }

-> {
     "type": "cvm.registry.call",
     "id": "c1",
     "selector": { "schemaHash": "a7f3d9..." },
     "tool": "search_profiles",
     "args": { "q": "jack" }
   }
<- {
     "type": "cvm.registry.call.result",
     "id": "c1",
     "result": { "content": [{ "type": "text", "text": "{...}" }], "isError": false }
   }
```

**Publish a notification without creating an MCP request:**

Here MCP request `2` is still pending for this napplet.

```
-> { "type": "cvm.request", "id": "n1", "server": { "pubkey": "65a334..." }, "message": { "jsonrpc": "2.0", "method": "notifications/cancelled", "params": { "requestId": 2 } } }
<- { "type": "cvm.request.result", "id": "n1" }
```

**Answer a server request, preserving its ID:**
```
<- { "type": "cvm.event", "server": { "pubkey": "65a334..." }, "message": { "jsonrpc": "2.0", "id": "srv-7", "method": "ping" } }
-> { "type": "cvm.request", "id": "answer-1", "server": { "pubkey": "65a334..." }, "message": { "jsonrpc": "2.0", "id": "srv-7", "result": {} } }
<- { "type": "cvm.request.result", "id": "answer-1" }
```

**Unsupported registry and denied request:**
```
-> { "type": "cvm.registry.list", "id": "l1" }
<- { "type": "cvm.registry.list.result", "id": "l1", "error": "unsupported method" }
-> { "type": "cvm.registry.has", "id": "h2", "selector": { "family": "relatr" } }
<- { "type": "cvm.registry.has.result", "id": "h2", "has": false }
-> { "type": "cvm.request", "id": "denied-1", "server": { "pubkey": "65a334..." }, "message": { "jsonrpc": "2.0", "id": 3, "method": "tools/list" } }
<- { "type": "cvm.request.result", "id": "denied-1", "error": "policy denied" }
```

### Error Handling

A shell exposing `cvm` MUST recognize every napplet-to-shell request type in the
wire table and return exactly one corresponding `.result` with the same envelope
`id`, including failures and notification/response publication acknowledgments.
Recognized but unimplemented operations return `"unsupported method"`, except
`registry.has`, which returns `has: false` when registry support is absent.
Truly unknown message types retain the silent-ignore behavior defined by
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303).

Shell and transport failures MUST use the top-level text `error` in that result.
There are no `cvm.*.error` messages; ACL denial returns the matching `.result`
with `error: "policy denied"`. On a shell error, success payload fields MUST be
omitted. On success, the operation's payload is required, except for `close`
and notification/response publication, whose result has only `type` and `id`.

| Error | Meaning |
|-------|---------|
| `invalid message` | Missing or malformed MCP message, invalid field combination, duplicate client request ID, or unmatched server response. |
| `invalid request` | Malformed server reference, query, selector, or other non-message parameter. |
| `invalid timeout` | Timeout is not an integer in the permitted range. |
| `timeout out of policy` | Valid timeout exceeds the shell's configured maximum. |
| `unsupported method` | Recognized NAP operation is not implemented. |
| `unsupported kind` | Requested announcement kind is not supported. |
| `server not found` | Server cannot be resolved. |
| `relay timeout` | Operation deadline expired. |
| `initialization failed` | Required MCP initialization failed. |
| `session closed` | The calling endpoint released the session while work was pending. |
| `payment required` | A payment gate cannot be fulfilled with available payment support or negotiation. |
| `payment denied` | Request policy, allowance, or user decision disallows payment. |
| `policy denied` | Shell ACL disallows the operation. |
| `family not found` | Local family label has no match. |
| `schema mismatch` | Pinned tool hash or tool name cannot be satisfied. |
| `provider unavailable` | Matching providers are not reachable. |
| `cache policy denied` | Requested cache behavior is forbidden. |

Remote MCP errors MUST remain in `message.error` for raw `request`, except for
payment control handled above. Convenience wrappers MUST surface remote errors
as failures, not successful empty results. A registry tool failure uses its
`McpToolResult` when available; a remote JSON-RPC error becomes a top-level text
error describing that failure. Validation errors MUST NOT be reported as
`"unsupported method"`.

## Shell Behavior

- The shell MUST sign, encrypt, route, and verify ContextVM events outside the
  napplet sandbox. Relay delivery alone is not proof of the server's identity.
- The shell MUST enforce the session, message, timeout, payment, registry, and
  result rules above, including when a feature is unsupported.
- The shell MUST NOT expose direct relay sockets, transport tags, or signing
  authority to a napplet through this domain.
- Initialization and cached state MUST preserve per-napplet isolation. Advertised
  MCP client capabilities MUST reflect features the shell and target napplet can
  actually service; forwarding a server request does not itself implement one.
- Transport implementation is runtime-owned. The
  [ContextVM SDK](https://docs.contextvm.org/reference/ts-sdk/transports/nostr-client-transport/)
  or another conforming transport MAY be used; either must preserve these NAP
  contracts and the linked upstream protocols.

## Conformance cases

These cases exercise the contract; they are not implementation test results.

| Case | Required outcome |
|------|------------------|
| A and B call the same server using MCP ID `1`; A closes. | A's pending call fails with `session closed`; B's call and events remain usable. |
| A registers `onEvent` without an open session; B receives a server request. | A receives nothing. Only B can answer the request. |
| A sends a notification, then answers a server request. | Neither awaits an MCP response; notification stays ID-less, response ID is preserved, each envelope receives one acknowledgment. |
| Missing message, `id: null`, or both `result` and `error`. | `invalid message` in `cvm.request.result`, with no publication. |
| Registry is not implemented; an ACL denies a direct request. | Registry operations return `unsupported method` (`has` returns false); ACL denial uses the matching `.result`, never `.error`. |
| A 120000 ms call returns after 45000 ms. | It succeeds if permitted by policy; no fixed 30000 ms binding timeout. |
| Timeout is zero, above 300000, or above a lower shell maximum. | `invalid timeout` for the first two; `timeout out of policy` for the last, before publication. |
| A call expires or closes before its response arrives. | Exactly one result; the late response is not an event or a second result. |
| `deny`, refused `prompt`, or `allow` without sufficient allowance meets a payment gate. | `payment denied`; no new settlement or unauthorized consumption of prior payment. |
| A shell without payment support receives `-32042` or `notifications/payment_required`. | `payment required` promptly, without a stalled MCP call. |
| Two napplets make the same paid call concurrently. | No shared authorization consumption or budget overrun; session modes remain isolated. |
| Two shells use different family labels for the same tool hash. | The same hash selector resolves on both; returned `schemaHashes` equals the set of tool hashes. |
| A tool hash/name mismatches, or a pinned provider is unavailable. | No substitute tool or provider is called; `schema mismatch` or `provider unavailable`. |
| `resources/read` returns several text/blob entries. | `readResource` returns all entries in order. |
| A tools announcement has no CEP-15 `i` tags; query restricts announcement kinds. | Capability names still derive from MCP data; only the requested supported kinds contribute results. |

## Security Considerations

- ContextVM servers are remote programs. Shells SHOULD treat tool calls as untrusted remote execution and enforce per-napplet policy for which servers and methods may be called.
- MCP tool arguments can contain sensitive user data. Shells SHOULD expose consent or ACL controls before allowing a napplet to send data to a new server.
- Shells MUST verify server identity by public key. A relay delivering a response is not proof that the response came from the requested server.
- Shells MUST enforce the timeouts above and SHOULD apply per-napplet concurrency limits to prevent resource exhaustion.
- Payment requests from ContextVM servers MUST NOT be auto-approved unless covered by an explicit user allowance.
- Napplets receive MCP results but not ContextVM private keys, encryption keys, NWC secrets, relay credentials, or direct socket access.
- Registry discovery is not trust. Shells SHOULD apply the same per-napplet policy to `registry.call` that they apply to direct `callTool`.
- Shells SHOULD NOT serve cached registry tool results across napplets when the arguments or selected provider are user-sensitive or policy-scoped.

## References

- [ContextVM](https://contextvm.org/)
- [ContextVM protocol specification](https://docs.contextvm.org/reference/spec/ctxvm-draft-spec/)
- [ContextVM CEP-8 Capability Pricing and Payment Flow](https://docs.contextvm.org/reference/ceps/cep-8/)
- [ContextVM CEP-15 Common Tool Schemas](https://docs.contextvm.org/reference/ceps/cep-15/)
- [Model Context Protocol](https://modelcontextprotocol.io/specification/2025-11-25/basic)

## Implementations

- [kehto/web](https://github.com/kehto/web) — existing partial implementation of
  discovery, raw MCP requests, convenience wrappers, sessions, and events. The
  [2026-10-14 review](https://github.com/napplet/naps/pull/31#issuecomment-6168948959)
  identifies required runtime corrections and reports registry and payment
  handling as unimplemented. This is not a claim of conformance to this revision.

## Changelog

- `e001eaf` - Introduced NAP-CVM as a native ContextVM bridge.
- `1873f7e` - Added a runtime registry facade for discovering CVM surfaces.
- `69d6bbd` - Adopted injected-domain availability and linked the current upstream web binding.
- `12ad986` - Scoped sessions, client identities, events, and payment authorization per napplet; defined MCP message completion, bounded timeouts, result errors, portable tool-hash registry selection, complete resource reads, discovery semantics, and conformance cases.
