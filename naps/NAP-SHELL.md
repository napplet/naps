NAP-SHELL
=========

Shell Environment and Compatibility
-----------------------------------

`draft`

**NAP ID:** NAP-SHELL
**Domain:** `shell`
**Depends:** none.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.shell`; domain presence signals availability.

## Description

NAP-SHELL exposes optional shell environment information, compatibility checks,
and readiness notifications. It is not a prerequisite for other domains. In the
[web projection](../projections/web.md), the runtime injects granted domain
objects before napplet scripts run. A napplet discovers a domain by its presence;
no handshake establishes identity or grants capabilities.

When `shell` is exposed, the runtime-provided binding signals that its receiver
is installed with `shell.ready`. The runtime replies with `shell.init`, containing
an environment snapshot. This exchange serves the `ready`, `onReady`, and
`services` operations only. Other exposed domains remain usable before, during,
and after it.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `supports` | `domain` (`text`) | `boolean` | local projection availability query |
| `domains` | none | list of text | local projection availability query |
| `check` | `manifest` (`NappletManifest`) | `ShellCompatibility` | `shell.check` / `shell.check.result` |
| `services` | none | list of text service names | local read of `shell.init` |
| `ready` | none | `ShellEnvironment` | resolves after `shell.ready` / `shell.init` |
| `onReady` | handler for `ShellEnvironment` | `Subscription` handle | fires after `shell.init` |

### Schemas

`ShellCapabilities` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `domains` | yes | list of text | Bare NAP domains exposed to this napplet at snapshot time. |

`ShellEnvironment` fields:

| Field | Required | Type |
|-------|----------|------|
| `capabilities` | yes | `ShellCapabilities` |
| `services` | yes | list of text |

The environment is descriptive. It MUST NOT grant domains or override the
projection's availability signal. Service names do not grant capabilities or
introduce operations outside their owning NAP contracts.

`NappletManifest` is a signed napplet manifest event defined by
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303). Its schema is not redefined here.

`ShellCompatibility` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `status` | yes | "compatible", "incompatible", or "unknown" | Required-domain assessment under current shell policy. |
| `missingRequired` | yes | list of text | Required domains known to be unavailable to the target. |
| `missingOptional` | yes | list of text | Optional domains known to be unavailable to the target. |

**`supports(domain)`** — Returns whether the projection currently exposes the
bare domain to this napplet. It is synchronous and local, and MUST work before
`shell.init`. In the web projection it reflects the presence of the corresponding
injected domain object. Unknown or unexposed domains return `false`. This
convenience operation does not make `shell` a dependency of domain discovery.

**`domains()`** — Returns a fresh, duplicate-free list of bare domains currently
exposed to the calling napplet. It is synchronous and local, MUST work before
`shell.init`, and MUST agree with `supports(domain)`. Ordering is unspecified.

**`check(manifest)`** — Assesses the target napplet's declared domain requirements.
The runtime MUST validate the manifest and its signature, then assess its `R`
and `O` domains using the same capability and policy rules used when loading
that target, independently of the caller's exposed domains. Availability that
cannot be determined or disclosed is unknown.

The result lists each known unavailable bare domain once in its corresponding list.
`status` MUST be `incompatible` if any required domain is known unavailable;
otherwise `compatible` if every required domain is known available, or `unknown`.
Missing optional domains do not affect `status`; no required domains means
`compatible`.

The check MUST work without `shell.init`. It MUST NOT fetch artifacts, install,
launch, request consent, or grant capabilities. A result describes current
declared-domain compatibility, not successful launch. The runtime MUST recheck
requirements at load time; artifact verification and user consent remain separate.

**`services`** — A read-only list from the delivered environment. It is empty
until `shell.init` is received.

**`ready()`** — Resolves with the retained environment once it is received. The
runtime-provided binding sends the readiness signal automatically after
installing its receiver. Repeated calls MUST NOT send additional signals.

**`onReady(handler)`** — Registers a one-shot callback for environment delivery.
A handler registered after delivery MUST receive the retained environment.

## Wire Protocol

`shell.*` messages use the
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format
(`{ "type": "domain.action", ...payload }`). Neither readiness message carries
an `id`; the environment is delivered once per napplet endpoint lifecycle.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `shell.ready` | napplet -> runtime | none |
| `shell.init` | runtime -> napplet | `capabilities`, `services` |
| `shell.check` | napplet -> runtime | `id`, `manifest` |
| `shell.check.result` | runtime -> napplet | `id`, `compatibility` or `error` |

For each `shell.check` request from an authenticated endpoint with a text `id`,
the runtime MUST reply to that endpoint with the same `id` and exactly one of
`compatibility` (`ShellCompatibility`) or `error` (`text`).

`shell.ready` carries no identity or capability claims. The runtime obtains
identity from the projection's already authenticated endpoint. `shell.init`
reports only this napplet's environment.

### Examples

**Optional environment delivery:**

```
-> { "type": "shell.ready" }
<- {
     "type": "shell.init",
     "capabilities": { "domains": ["shell", "theme"] },
     "services": []
   }
```

**Web projection availability, without waiting for readiness:**

```js
if (window.napplet.theme) {
  const theme = await window.napplet.theme.get();
}
```

**Target compatibility** (`manifest` denotes a complete signed event requiring `relay`):

```text
-> { "type": "shell.check", "id": "c1", "manifest": <manifest> }
<- { "type": "shell.check.result", "id": "c1", "compatibility": {
     "status": "incompatible", "missingRequired": ["relay"], "missingOptional": []
   } }
```

### Error Handling

A missing, malformed, or invalidly signed manifest MUST return `error`:
`"invalid manifest"`. Other check failures return an explanatory `error`.
An unavailable operation, timeout, or error MUST NOT be interpreted as
incompatibility. Indeterminate assessments use `status: "unknown"`.

The readiness exchange has no result envelope or `error` field. If the environment
never arrives, a napplet MAY stop waiting under its own timeout policy. This
means shell environment information is unavailable; it MUST NOT be interpreted
as absence of other exposed domains or failure to establish napplet identity.
A napplet MUST NOT gate unrelated domain calls on `ready()` or `onReady`.

## Shell Behavior

- The runtime MAY omit `shell` while exposing other domains.
- The runtime MUST bind the endpoint to verified napplet identity before
  servicing any domain. It MUST NOT derive or alter identity from `shell.ready`.
- When `shell` is exposed, its binding MUST install the receiver before sending
  `shell.ready`.
- The runtime MUST send `shell.init` exactly once in response to the first
  `shell.ready` for that authenticated endpoint.
- Duplicate `shell.ready` messages MUST NOT establish another session, change
  policy, or cause another `shell.init`.
- The environment MUST describe only domains and services exposed to that
  napplet. It MUST NOT expose another napplet's grants.
- The binding MUST retain the environment for `ready`, `onReady`, and `services`.
- The runtime MUST NOT gate other domain calls or their availability on this
  readiness exchange. Each domain owns its own readiness and delivery contract.

## Security Considerations

- The readiness signal is untrusted input, not authentication or authorization.
- Manifest capability declarations and environment snapshots are not grants.
  The runtime enforces per-napplet policy independently of discovery helpers.
- Compatibility checks MUST NOT reveal another napplet's existing grants or
  private state. Unavailable policy information MUST remain undisclosed.
- Withholding `shell.init` MUST NOT serve as a substitute for enforcing policy
  on another domain. A runtime denies a domain through the projection's
  availability and enforcement mechanisms.
- Web identity verification, namespace injection, and transport authentication
  remain defined by [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303).

## Implementations

- (none yet)

## Changelog

- `7ea6cd3` - Introduced the shell bootstrap handshake and capability query.
- `f86fe4b` - Made the shell contract self-contained and mandatory.
- `c616fbb` - Removed deferred class support and linked the upstream web binding.
- `a31573e` - Removed mandatory shell bootstrap, made availability independent of readiness, and defined the optional environment snapshot.
- `89d0066` - Added immediate domain enumeration and target-manifest compatibility checks without granting capabilities or launching napplets.
