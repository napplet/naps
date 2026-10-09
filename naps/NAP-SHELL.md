NAP-SHELL
=========

Bootstrap Handshake & Capability Negotiation
--------------------------------------------

`draft`

**NAP ID:** NAP-SHELL
**Domain:** `shell`
**Required:** Mandatory — every conformant runtime MUST implement NAP-SHELL.
**Web binding (NIP-5D):** `window.napplet.shell`

> NAP-SHELL is the **foundational** capability: it defines `shell.supports()`
> itself and is therefore the one NAP that cannot be discovered through
> `shell.supports()`. Every runtime that implements any NAP implements NAP-SHELL
> unconditionally, and a napplet MAY assume it is present.

## Description

Before a napplet can use any capability, two things must be true that neither
side can know on its own: the runtime must learn **when the napplet is ready to
receive messages**, and the napplet must learn **what the runtime offers**. The
napplet cannot enumerate the runtime's capabilities until told; the runtime
cannot deliver that list until the napplet's receiver is live. NAP-SHELL is the
two-message handshake that resolves this bootstrap dependency.

The napplet signals readiness (`shell.ready`). The runtime replies once with the
**environment** (`shell.init`): the set of capabilities it offers and the named
services it exposes. The napplet caches that
environment, which is what makes `shell.supports()` answerable **synchronously
and locally** thereafter — no round-trip per query. Receipt of the readiness
signal is also the point at which the runtime considers the napplet's **session
established**, so every other capability call is serviceable only after the
handshake completes.

NAP-SHELL standardizes the **handshake and the queryable capability set**, not
the internal representation of that set. A runtime is conformant as long as it
delivers an environment from which `supports(domain)` can be answered for every
capability it offers; the byte layout of the capability object is not normative.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `supports` | `domain` (`tstr`) | `bool` | local query against cached `shell.init` |
| `services` | none | list of `tstr` service names | local read against cached `shell.init` |
| `ready` | none | `ShellEnvironment` | resolves after `shell.ready` / `shell.init` |
| `onReady` | handler for `ShellEnvironment` | `Subscription` handle | fires after `shell.init` |

### Schemas

`ShellCapabilities` is a runtime-internal value with whatever shape is
sufficient to answer `supports(domain)`.

`ShellEnvironment` fields:

| Field | Required | Type |
|-------|----------|------|
| `capabilities` | yes | `ShellCapabilities` |
| `services` | yes | list of text |

**`supports(domain)`** — Returns whether the runtime offers `domain`.
Synchronous: it reads the cached environment and never blocks. Returns `false`
before the environment has been delivered, and `false` for any unknown domain.

**`services`** — A read-only view onto the delivered environment: the named
services the runtime exposes for this napplet.

**`ready()`** — Resolves with the environment once the handshake completes. The
readiness signal is normally emitted automatically by the runtime-provided shim
at load; `ready()` is the napplet-facing await point, not a second signal.

**`onReady(handler)`** — Registers a one-shot callback for environment delivery.

## Wire Protocol

`shell.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`). The handshake is two fire-and-forget
messages; neither carries a correlation `id`, because each occurs exactly once
per napplet lifecycle.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `shell.ready` | napplet -> runtime | *(none)* |
| `shell.init` | runtime -> napplet | `capabilities`, `services` |

Key design notes:
- `shell.ready` carries **no payload**. It is a liveness signal only — "my
  receiver is installed." It MUST NOT carry napplet identity or capability
  claims; identity is assigned by the runtime at napplet creation (NIP-5A), not
  asserted over this channel.
- `shell.init` is sent **exactly once** in response to the first `shell.ready`.
- `shell.supports()` is answered **locally** from the cached `shell.init`
  environment. It is not a wire round-trip.
- The capability object's internal shape is not normative; only that it can
  answer `supports(domain)` for every offered capability.

### Examples

**Handshake:**
```
-> { "type": "shell.ready" }
<- {
     "type": "shell.init",
     "capabilities": {
       "domains": ["<domain>", "<domain>"]
     },
     "services": []
   }
```

**Subsequent local queries (no wire traffic):**
```
shell.supports("<domain>")             // true if the runtime offers that domain
shell.supports("<unknown>")            // false — domain not offered
```

### Error Handling

The handshake has no result envelope and therefore no `error` field. Failure is
expressed by **absence**:

- If `shell.init` never arrives, `supports()` returns `false` for everything and
  no capability is serviceable. A napplet SHOULD treat a missing environment
  after a reasonable timeout as "running outside a conformant runtime" and
  degrade rather than hang.
- A runtime that declines to service a napplet MAY withhold `shell.init`
  entirely; the napplet observes this as a runtime that offers nothing.

## Shell Behavior

- The runtime MUST send `shell.init` in response to the napplet's first
  `shell.ready`, and MUST do so only after the napplet's receiver is live (i.e.
  in response to the signal, never speculatively before it).
- The runtime MUST establish the napplet's session upon receiving the first
  `shell.ready`, binding it to the identity assigned at napplet creation
  (NIP-5A) — never to anything carried in the message.
- The runtime MUST send `shell.init` **exactly once** per napplet lifecycle.
- The runtime's delivered capability set MUST be sufficient to answer
  `supports(domain)` truthfully for every capability it offers, and
  MUST answer `false` for capabilities it does not offer.
- The runtime SHOULD treat a duplicate `shell.ready` as **idempotent**: it MUST
  NOT establish a second session or overwrite the first, and SHOULD NOT resend
  `shell.init`.
- The runtime MUST NOT service capability calls for a napplet whose session has
  not been established by the handshake.

## Security Considerations

- `shell.ready` originates from **untrusted napplet content**. It is a bare
  liveness ping by design: it carries no identity, no capability request, and no
  payload the runtime could be tricked into trusting. A runtime MUST derive the
  napplet's identity from creation-time assignment (NIP-5A), not from
  the handshake channel.
- Session establishment is a privileged side effect. Because a second
  `shell.ready` MUST NOT create a second session or mutate the first, a napplet
  cannot replay the signal to escalate, re-key, or re-scope its session.
- The capability set in `shell.init` is the runtime's **authoritative statement**
  of what this napplet may use. A runtime MUST scope it per napplet and MUST NOT
  leak the capabilities or services granted to other napplets.
- The handshake gates all other capability traffic: by withholding
  `shell.init`, a runtime denies a napplet every capability at once, giving the
  runtime a single, total enforcement point.

## Implementations

- (none yet)
