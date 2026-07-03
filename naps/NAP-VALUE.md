NAP-VALUE
=========

Value Transfer
--------------

`draft`

**NAP ID:** NAP-VALUE
**Domain:** `value`
**Depends:**
- `relay` — layering · optional — the shell MAY publish / query zap receipts via `relay` or an internal pool
**Web binding (NIP-5D):** `window.napplet.value` · `shell.supports("value")`

## Description

NAP-VALUE provides napplets with a shell-mediated interface for requesting value transfer. The first concrete use case is sending Nostr zaps, but the interface is named around value instead of payment so shells can support additional value rails without changing the napplet API. Napplets describe the intended transfer; the shell handles wallet integration, consent, zap request construction, signing, invoice payment, relay publishing, and result tracking.

Napplets never receive wallet credentials, NWC secrets, signing keys, or direct network access to payment backends. The shell is the policy and consent boundary.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `send` | `request` (`ValueRequest`) | `ValueResult` | `value.send` / `value.send.result` |
| `quote` | `request` (`ValueRequest`) | `ValueQuote` | `value.quote` / `value.quote.result` |
| `status` | `transferId` (`tstr`) | `ValueStatus` | `value.status` / `value.status.result` |
| `onStatus` | handler for `ValueStatus` | `Subscription` handle | `value.status.changed` |

### Schemas

Primitive references:

| Name | Meaning |
|------|---------|
| `NostrEvent` | NIP-01 signed event object. |
| `ValueTarget` | One of `ZapTarget`, `LnurlTarget`, or `ValueExtensionTarget`. |

Enumerations:

| Name | Values |
|------|--------|
| `ValueRail` | `zap`, `lnurl`, or another rail name string. |
| `ValueTransferState` | `pending`, `settled`, `failed`, `cancelled` |
| `ZapTarget.type` | `zap` |
| `LnurlTarget.type` | `lnurl` |

`ValueRequest`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `rail` | `ValueRail` | yes | Value rail to use. |
| `amountMsat` | `uint` | yes | Transfer amount in millisatoshis for Lightning-compatible rails. |
| `comment` | `tstr` | no | User-visible transfer comment. |
| `target` | `ValueTarget` | yes | Transfer target. |
| `metadata` | map of `tstr` to any JSON value | no | Rail-specific metadata. |

`ZapTarget`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | `zap` | yes | Target discriminator. |
| `pubkey` | `tstr` | no | Recipient pubkey. |
| `eventId` | `tstr` | no | Event being zapped. |
| `address` | `tstr` | no | Addressable event being zapped. |
| `relays` | list of `tstr` | no | Relay hints for zap request or receipt handling. |

`LnurlTarget`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | `lnurl` | yes | Target discriminator. |
| `lnurl` | `tstr` | yes | LNURL target. |

`ValueExtensionTarget`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | `tstr` | yes | Extension rail target discriminator. |
| other `tstr` keys | any JSON value | no | Extension-specific target fields. |

`ValueQuote`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ok` | `bool` | yes | Whether the quote succeeded. |
| `amountMsat` | `uint` | yes | Quoted amount in millisatoshis. |
| `rail` | `ValueRail` | yes | Quoted rail. |
| `feesMsat` | `uint` | no | Estimated fees in millisatoshis. |
| `expiresAt` | `uint` | no | Expiration timestamp. |
| `error` | `tstr` | no | Error reason when quote failed. |

`ValueResult`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ok` | `bool` | yes | Whether the transfer request was accepted or completed. |
| `transferId` | `tstr` | yes | Shell-generated transfer identifier. |
| `status` | `ValueTransferState` | yes | Current transfer state. |
| `rail` | `ValueRail` | yes | Transfer rail. |
| `amountMsat` | `uint` | yes | Transfer amount in millisatoshis. |
| `event` | `NostrEvent` | no | Related Nostr event, such as a zap request or receipt. |
| `preimage` | `tstr` | no | Payment preimage when available and safe to disclose. |
| `error` | `tstr` | no | Error reason when transfer failed. |

`ValueStatus`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ok` | `bool` | yes | Whether the status lookup succeeded. |
| `transferId` | `tstr` | yes | Shell-generated transfer identifier. |
| `status` | `ValueTransferState` | yes | Current transfer state. |
| `rail` | `ValueRail` | yes | Transfer rail. |
| `amountMsat` | `uint` | yes | Transfer amount in millisatoshis. |
| `event` | `NostrEvent` | no | Related Nostr event, such as a zap request or receipt. |
| `preimage` | `tstr` | no | Payment preimage when available and safe to disclose. |
| `error` | `tstr` | no | Error reason when transfer failed. |
| `updatedAt` | `uint` | yes | Timestamp of the latest status update. |

**`send(request)`** -- Requests a value transfer. The shell presents any required consent UI, performs the transfer through its configured backend, and returns the initial result. For zap requests, the shell constructs and signs the kind 9734 zap request, pays the Lightning invoice, publishes any required Nostr event, and reports the resulting status.

**`quote(request)`** -- Returns a best-effort quote before transfer. Shells MAY use this to resolve an LNURL-pay endpoint, estimate fees, or validate that the request is payable. A quote is advisory; shells MUST revalidate during `send`.

**`status(transferId)`** -- Returns the latest known status for a prior transfer.

**`onStatus(handler)`** -- Registers for shell-pushed status updates, such as `pending` becoming `settled` after a zap receipt is observed.

## Wire Protocol

`value.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `value.quote` | napplet -> shell | `id`, `request` |
| `value.quote.result` | shell -> napplet | `id`, `quote`, `error?` |
| `value.send` | napplet -> shell | `id`, `request` |
| `value.send.result` | shell -> napplet | `id`, `result`, `error?` |
| `value.status` | napplet -> shell | `id`, `transferId` |
| `value.status.result` | shell -> napplet | `id`, `status`, `error?` |
| `value.status.changed` | shell -> napplet | `status` |

Key design notes:
- Request/result pairs use `id` for correlation.
- `transferId` is generated by the shell and is scoped to the requesting napplet.
- `value.status.changed` is a shell push message and has no `id`.
- `amountMsat` is always millisatoshis for Lightning-compatible rails. Other rails MUST document their unit in `metadata.unit` until a dedicated rail-specific NAP exists.

### Examples

**Quote a zap:**
```
-> {
     "type": "value.quote",
     "id": "q1",
     "request": {
       "rail": "zap",
       "amountMsat": 21000,
       "comment": "great note",
       "target": { "type": "zap", "eventId": "abc...", "pubkey": "def...", "relays": ["wss://relay.example.com"] }
     }
   }
<- {
     "type": "value.quote.result",
     "id": "q1",
     "quote": { "ok": true, "rail": "zap", "amountMsat": 21000, "feesMsat": 10, "expiresAt": 1234567999 }
   }
```

**Send a zap:**
```
-> {
     "type": "value.send",
     "id": "s1",
     "request": {
       "rail": "zap",
       "amountMsat": 21000,
       "comment": "great note",
       "target": { "type": "zap", "eventId": "abc...", "pubkey": "def...", "relays": ["wss://relay.example.com"] }
     }
   }
<- {
     "type": "value.send.result",
     "id": "s1",
     "result": {
       "ok": true,
       "transferId": "value-1",
       "status": "pending",
       "rail": "zap",
       "amountMsat": 21000
     }
   }
<- {
     "type": "value.status.changed",
     "status": {
       "ok": true,
       "transferId": "value-1",
       "status": "settled",
       "rail": "zap",
       "amountMsat": 21000,
       "updatedAt": 1234567901
     }
   }
```

**User cancelled:**
```
<- {
     "type": "value.send.result",
     "id": "s2",
     "result": {
       "ok": false,
       "transferId": "value-2",
       "status": "cancelled",
       "rail": "zap",
       "amountMsat": 21000,
       "error": "user cancelled"
     }
   }
```

### Error Handling

Result messages MAY include `error` when the request cannot be fulfilled. Common errors include `"unsupported rail"`, `"invalid target"`, `"amount out of policy"`, `"user cancelled"`, `"wallet unavailable"`, `"invoice expired"`, and `"payment failed"`.

The shell SHOULD return a structured `result` with `ok: false` when a transfer was created and then failed or was cancelled. The shell SHOULD return a top-level `error` when no transfer was created.

## Shell Behavior

- The shell MUST require explicit user consent before spending value unless the user has configured an allowance that covers the request.
- The shell MUST enforce per-napplet policy for allowed rails, maximum amount, and allowance use.
- The shell MUST construct and sign Nostr events required by a value rail. For zaps, this includes the kind 9734 zap request.
- The shell MUST NOT sign or pay opaque requests that it cannot inspect and present to the user.
- The shell MUST keep wallet credentials, NWC connection strings, LNURL secrets, payment preimages, and private keys out of the napplet sandbox.
- The shell MUST respond to every request with a result message carrying the same `id`.
- The shell SHOULD surface intermediate states through `value.status.changed` when settlement is asynchronous.
- The shell MAY implement rails through NWC, LNURL-pay, an internal wallet service, or another backend. Backend choice is an implementation detail.
- The shell MAY publish or query zap receipts through NAP-RELAY or an internal relay pool.

## Security Considerations

- Spending value is user-visible and security-critical. Shells MUST treat every `value.send` as an untrusted request until consent and policy checks pass.
- Napplets can attempt dark-pattern spending. Shell consent UI SHOULD show amount, target, rail, recipient identity, comment, and requesting napplet identity.
- Allowances are risky. Shells that implement allowances SHOULD bind them to the napplet identity, rail, maximum amount, recurrence window, and target constraints.
- Zap comments and metadata may leak user intent. Shells SHOULD present them before signing and sending.
- The shell signs zap requests and publishes value-related events. Napplets MUST NOT be given signing or encryption primitives through this interface.
- Payment failures MUST NOT be hidden as success. A settled state requires backend confirmation or a rail-specific proof accepted by shell policy.

## Implementations

- (none yet)
