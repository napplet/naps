NAP-CVM
========

ContextVM Bridge
----------------

`draft`

**NAP ID:** NAP-CVM
**Namespace:** `window.napplet.cvm`
**Discovery:** `shell.supports("cvm")`

## Description

NAP-CVM provides napplets with native access to ContextVM servers through the shell. ContextVM transports Model Context Protocol (MCP) messages over Nostr relays using public-key addressing and encrypted relay messages. A napplet could build this itself on top of NAP-RELAY, but doing so would force every napplet to implement ContextVM discovery, initialization, JSON-RPC correlation, relay routing, encryption, signing, and optional payment handling. NAP-CVM moves that legwork into the runtime.

The napplet supplies the server identity and the MCP operation it wants. The shell handles ContextVM transport details, user policy, signing, encryption, relay access, and value-transfer prompts.

## API Surface

```typescript
interface NappletCvm {
  discover(query?: CvmDiscoverQuery): Promise<CvmServer[]>;
  request(server: CvmServerRef, message: McpMessage, options?: CvmRequestOptions): Promise<McpMessage>;
  listTools(server: CvmServerRef, options?: CvmRequestOptions): Promise<McpTool[]>;
  callTool(server: CvmServerRef, name: string, args?: Record<string, unknown>, options?: CvmRequestOptions): Promise<McpToolResult>;
  listResources(server: CvmServerRef, options?: CvmRequestOptions): Promise<McpResource[]>;
  readResource(server: CvmServerRef, uri: string, options?: CvmRequestOptions): Promise<McpResourceContent>;
  close(server: CvmServerRef): Promise<void>;
}

interface CvmServerRef {
  pubkey: string;
  relays?: string[];
}

interface CvmDiscoverQuery {
  search?: string;
  kinds?: number[];
  relays?: string[];
  limit?: number;
}

interface CvmServer extends CvmServerRef {
  name?: string;
  description?: string;
  capabilities?: string[];
  paymentRequired?: boolean;
}

interface CvmRequestOptions {
  timeoutMs?: number;
  initialize?: boolean;
  payment?: "deny" | "prompt" | "allow";
}

interface McpMessage {
  jsonrpc: "2.0";
  id?: string | number;
  method?: string;
  params?: unknown;
  result?: unknown;
  error?: unknown;
}
```

**`discover(query?)`** -- Returns public ContextVM server announcements known to the shell. The shell MAY resolve discovery from local cache, configured relays, or user-curated lists.

**`request(server, message, options?)`** -- Sends a raw MCP JSON-RPC message to a ContextVM server and resolves with the matching MCP response. The shell wraps the MCP message in ContextVM/Nostr transport events.

**`listTools`, `callTool`, `listResources`, `readResource`** -- Convenience wrappers for common MCP operations. They use `request` semantics but spare napplets from writing repetitive JSON-RPC envelopes.

**`close(server)`** -- Closes shell-maintained session state for a server, including subscriptions, cached initialization state, and pending correlation records.

## Wire Protocol

`cvm.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `cvm.discover` | napplet -> shell | `id`, `query?` |
| `cvm.discover.result` | shell -> napplet | `id`, `servers`, `error?` |
| `cvm.request` | napplet -> shell | `id`, `server`, `message`, `options?` |
| `cvm.request.result` | shell -> napplet | `id`, `message?`, `error?` |
| `cvm.close` | napplet -> shell | `id`, `server` |
| `cvm.close.result` | shell -> napplet | `id`, `error?` |
| `cvm.event` | shell -> napplet | `server`, `message` |

Key design notes:
- `cvm.request.result` correlates to the NIP-5D request `id`; the embedded MCP message retains its own JSON-RPC `id`.
- `cvm.event` is for MCP notifications or server messages not directly correlated to a single request.
- The shell owns all ContextVM event IDs, relay subscriptions, `p` tags, `e` tags, encryption state, and signing.
- The shell MAY initialize a ContextVM session automatically before the first capability request.

### Examples

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
       "params": { "name": "get_relay", "arguments": { "url": "wss://relay.example.com" } }
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
     "message": { "jsonrpc": "2.0", "method": "notifications/progress", "params": { "progress": 50 } }
   }
```

### Error Handling

Result messages MAY include `error` when the shell cannot complete the request. Common errors include `"server not found"`, `"relay timeout"`, `"initialization failed"`, `"payment required"`, `"payment denied"`, `"unsupported method"`, and `"policy denied"`.

MCP-level errors from the remote server SHOULD be returned inside the embedded MCP `message.error` field. Transport or shell-policy failures SHOULD be returned in the NIP-5D result `error` field.

## Shell Behavior

- The shell MUST perform ContextVM signing and encryption outside the napplet sandbox.
- The shell MUST route ContextVM messages through its relay policy and MUST NOT let napplets open direct relay sockets.
- The shell MUST correlate ContextVM request and response events before resolving `cvm.request`.
- The shell MUST validate that responses are signed by the expected server pubkey before delivering them to the napplet.
- The shell MUST respond to every request with a result message carrying the same `id`.
- The shell SHOULD perform MCP initialization when `options.initialize` is true or when a server requires it.
- The shell MAY cache server capabilities and initialization state.
- The shell MAY expose discovery from public server announcements, local configuration, or user-curated lists.
- The shell MAY coordinate payment prompts through NAP-VALUE when a ContextVM server requires value exchange.

## Security Considerations

- ContextVM servers are remote programs. Shells SHOULD treat tool calls as untrusted remote execution and enforce per-napplet policy for which servers and methods may be called.
- MCP tool arguments can contain sensitive user data. Shells SHOULD expose consent or ACL controls before allowing a napplet to send data to a new server.
- Shells MUST verify server identity by public key. A relay delivering a response is not proof that the response came from the requested server.
- Shells SHOULD apply request timeouts and concurrency limits to prevent napplets from exhausting relay or ContextVM session resources.
- Payment requests from ContextVM servers MUST NOT be auto-approved unless covered by an explicit user allowance.
- Napplets receive MCP results but not ContextVM private keys, encryption keys, NWC secrets, relay credentials, or direct socket access.

## References

- [ContextVM](https://contextvm.org/)
- [ContextVM protocol specification](https://docs.contextvm.org/spec/ctxvm-draft-spec/)
- [Model Context Protocol](https://modelcontextprotocol.io/)

## Implementations

- (none yet)
