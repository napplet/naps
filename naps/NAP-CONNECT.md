NAP-CONNECT
===========

User-Gated Direct Network Access
--------------------------------

`draft`

**NAP ID:** NAP-CONNECT
**Domain:** `connect`
**Web binding (NIP-5D):** `window.napplet.connect` · `shell.supports("connect")`
**Parent:** NIP-5D

## Description

NAP-CONNECT provides sandboxed napplets with user-gated direct network access (`fetch`, `WebSocket`, `SSE`, `EventSource`) to a pre-declared list of origins. Origins are declared at build time via `["connect", "<origin>"]` tags in the NIP-5A manifest. The host shell reads those tags, prompts the user for approval at first load of a given `(dTag, aggregateHash)`, persists the decision, and emits a runtime Content Security Policy whose `connect-src` directive contains the approved origin list. The user's decision is expressed through the HTTP-response CSP the shell serves with the napplet's HTML — not through any postMessage traffic.

A napplet that declares any `connect` tag takes on the posture defined in `NAP-CLASS-2.md`. A napplet that declares no `connect` tags takes on the default posture defined in `NAP-CLASS-1.md`. This NAP does not redefine those class postures — see `NAP-CLASS-1.md` and `NAP-CLASS-2.md` for CSP shapes, consent-flow MUSTs, grant-persistence semantics, residual-meta-CSP refusal requirements, and revocation responsibilities. NAP-CONNECT defines the manifest-tag shape, origin format, canonical aggregateHash fold, runtime API, and capability advertisement that feed into the class determination; the postures themselves are owned by the sub-track documents.

NAP-CONNECT has **no postMessage wire protocol**. Grants are expressed entirely through the runtime CSP the shell emits in the HTTP response for the napplet's HTML, plus a shell-injected discovery meta tag (`<meta name="napplet-connect-granted" content="...">`) read synchronously by the napplet shim at install time. There are no `connect.*` envelopes in either direction. Implementers and readers looking for a wire message in this spec will not find one — the absence is deliberate.

## Motivation

The sandbox model inherited from NIP-5D (`sandbox="allow-scripts"`, no `allow-same-origin`) combined with a restrictive baseline CSP means napplets cannot, by default, reach the network directly. NAP-RESOURCE partially fills the gap with a shell-mediated single-`Blob` primitive (`resource.bytes(url) → Blob`): read-only, shell-visible, policy-enforced at the URL layer, with MIME byte-sniffing and SVG rasterization baked in. That primitive covers avatars, static assets, one-shot byte fetches, and bech32 resolution — a large fraction of napplet use cases.

What NAP-RESOURCE cannot express: `POST` / `PUT` / `PATCH` methods, WebSocket, Server-Sent Events, streaming responses, custom request headers, long-lived connections, and any third-party library that calls `fetch` or opens a socket directly. For those use cases, the napplet needs direct browser-level network access to a specific known host. Direct network access in a sandboxed iframe requires explicit CSP relaxation, which requires user consent — because the shell cannot see or filter traffic that flows directly from the iframe to an approved origin. NAP-CONNECT is the minimum-viable protocol for that consent.

Authors SHOULD default to NAP-RESOURCE for everything it can express; NAP-CONNECT exists specifically for the cases where the resource NAP cannot. The two NAPs are complementary, not competing — most non-trivial napplets will use both.

## Non-Goals

- **Per-origin partial grants.** v1 is all-or-nothing: approving the prompt approves every declared origin. A future revision MAY add partial grants.
- **Wildcard subdomains.** `https://*.example.com` is not supported. Each subdomain is a separate origin requiring its own `connect` tag and its own line in the consent prompt.
- **Quota or rate-limiting on granted traffic.** The browser provides no hook for a shell running in a parent frame to inspect or rate-limit traffic from a sandboxed child frame's direct network I/O. A service worker could mediate the traffic, but the napplet's `sandbox="allow-scripts"` iframe forbids service-worker registration.
- **Audit logging of individual network calls.** Same reason: the browser enforces CSP transparently to the shell. The shell sees no per-request trace.
- **A postMessage wire protocol.** NAP-CONNECT is expressed through manifest tags, CSP, and a discovery meta tag. No `connect.request`, no `connect.approve`, no `connect.granted` envelope exists. Adding one would contradict the architectural decision documented here and in NAP-CLASS-2.md.
- **Shell visibility into post-grant traffic.** Once the user approves an origin, the shell has no browser-level mechanism to observe, filter, or intercept subsequent traffic between the napplet and that origin. This is the fundamental tradeoff of NAP-CONNECT versus NAP-RESOURCE; it is not a defect to be mitigated but a property to be disclosed clearly in the consent prompt.

## Architecture Overview

Three moving parts, and nothing else:

- **Napplet build.** The napplet author declares required origins through the napplet build tool's `connect: string[]` option. The build tool normalizes each origin (lowercase scheme and host, Punycode for IDN, strip default ports, reject malformed input), emits one `["connect", "<normalized-origin>"]` tag per origin into the signed NIP-5A manifest, and folds the normalized origin set into `aggregateHash` via a synthetic `connect:origins` xTag entry. Any change to the origin set — addition, removal, reorder after normalization — produces a different `aggregateHash` and therefore a different grant key.

- **Shell runtime.** The shell reads the manifest's `connect` tags, validates each origin against the format rules, and looks up grant state keyed on `(dTag, aggregateHash)`. On first load of a new key: prompt the user, persist the decision. On subsequent loads: reuse the persisted decision. On approval, emit a runtime CSP whose `connect-src` directive contains the approved origins and inject `<meta name="napplet-connect-granted" content="<space-separated-origins>">` into the served HTML. On denial, fall back to the posture defined in `NAP-CLASS-1.md` (the shell emits `connect-src 'none'` and the injected meta tag is empty or absent). Grant-persistence and consent-flow details are owned by `NAP-CLASS-2.md`.

- **Napplet runtime.** The napplet shim reads the discovery meta tag synchronously at install time and populates `window.napplet.connect` with `{ granted, origins }`. Napplet code branches on `window.napplet.connect.granted`: when `true`, call `fetch` / open `WebSocket` / `EventSource` directly against approved origins; when `false`, fall back to NAP-RESOURCE for what the resource NAP can express, or degrade the affected feature gracefully. The shim never waits for a wire message — grant state is known at install time.

## Posture Citation

A napplet declaring any `["connect", "<origin>"]` manifest tag takes on the posture defined in `NAP-CLASS-2.md`. A napplet with no `connect` tags takes on the default posture defined in `NAP-CLASS-1.md`. NAP-CONNECT does NOT redefine those postures: this NAP specifies only the manifest-tag shape, the origin format, the canonical aggregateHash fold, the runtime API, and the capability advertisement that feed the class determination. The concrete CSP directive shapes, the consent-prompt MUSTs, the shell responsibilities at serve time, the grant-persistence semantics, the residual-meta-CSP refuse-to-serve requirement, and the revocation UX are owned by `NAP-CLASS-2.md` — readers looking for any of those details MUST follow the citation rather than treating this section as a substitute. For the default-deny posture that a napplet without `connect` tags receives (and that a napplet whose user denied the consent prompt is downgraded to), see `NAP-CLASS-1.md`.

## Manifest Tag Shape

Origins are declared in the NIP-5A manifest using one tag per origin:

```
["connect", "<origin>"]
```

The `<origin>` value MUST be the lowercase, normalized, Punycode-encoded form defined in **Origin Format** below. Tags are signed as part of the NIP-5A manifest and participate in `aggregateHash` via a synthetic xTag entry defined in **Canonical `connect:origins` aggregateHash Fold** below. One origin, one tag; an origin list of length N produces exactly N `connect` tags in the manifest.

Worked example: a manifest fragment declaring three origins (two ASCII, one IDN converted to Punycode):

```
["connect", "https://api.example.com"]
["connect", "wss://events.example.com"]
["connect", "https://xn--caf-dma.example.com"]
```

The napplet-build tool is the sole author of these tags. Napplet authors do NOT hand-edit the manifest; they supply origins to the build tool's `connect: string[]` option and the build tool performs normalization, validation, and emission.

## Origin Format

Each origin MUST conform to the following rules. The napplet-build tool MUST reject any origin that fails these rules at build time; the shell MUST reject any `connect` tag whose value fails these rules at manifest-load time. Both sides enforce because the build rejection catches author errors early and the shell rejection catches tampering after signing.

- **Scheme whitelist.** The scheme part MUST be one of `https:`, `wss:`, `http:`, or `ws:`. Any other scheme — including `file:`, `ftp:`, `gopher:`, `data:`, `blob:`, and custom schemes — MUST be rejected. This is a closed set; shells MUST NOT accept additional schemes under any policy, and the build tool MUST NOT emit them.
- **Lowercase scheme and host.** The scheme and host parts MUST be lowercase on the wire. `HTTPS://FOO.COM` is invalid as emitted. The build tool MUST lowercase before emission; the shell MUST reject non-lowercase input with a diagnostic. Shells MUST NOT silently lowercase on lookup — doing so would cause two cosmetically different origins to produce two different aggregateHashes while matching the same CSP directive, breaking the determinism rationale for the fold below.
- **Punycode IDN.** A host containing any non-ASCII character MUST be Punycode-encoded (`xn--` form, lowercase) before emission. The build tool is the sole authority for the UTF-8 → Punycode conversion. The shell MUST NOT auto-convert and MUST reject any non-Punycode non-ASCII host with a diagnostic. This direction of authority ensures that whatever Punycode form the napplet author signs into the manifest is the byte-identical form the shell sees on lookup.
- **No wildcards.** The host MUST NOT contain `*`. Wildcard subdomains (for example `https://*.example.com`) MUST be rejected at build time and at manifest-load time. Each subdomain is a separate origin that requires its own `connect` tag and participates independently in the consent prompt.
- **Default ports banned.** If the port component would be the scheme's default (443 for `https:` and `wss:`, 80 for `http:` and `ws:`), the port MUST be omitted from the origin. `https://foo.com:443` MUST be rejected at build time; `https://foo.com` is the correct form. The rationale is aggregateHash determinism: two cosmetically different forms that CSP treats as equivalent origin-matches would otherwise produce different hashes, silently invalidating prior grants on no semantic change. Shells MUST NOT normalize default ports on lookup for the same reason.
- **No path, query, or fragment.** The origin is scheme + host + optional non-default port. `https://foo.com/api`, `https://foo.com?x=1`, and `https://foo.com#section` are all invalid. CSP `connect-src` matches at the origin granularity; path-level constraints are out of scope for NAP-CONNECT.
- **No trailing slash.** The origin MUST NOT end in `/`. `https://foo.com/` is invalid; `https://foo.com` is the correct form. This rule is a consequence of the no-path rule but is worth calling out because many URL-parsing libraries auto-append a trailing slash when stringifying a parsed origin.

### Origin Normalization Conformance Vectors

The table below enumerates canonical input/output pairs a conformant build tool and a conformant shell MUST agree on. Rows marked `reject` indicate inputs that MUST be refused with a diagnostic; rows with a normalized output indicate inputs that are accepted after producing the specified byte-identical string.

| Input | Normalized | Reason |
|-------|------------|--------|
| `HTTPS://api.example.com` | `https://api.example.com` | Uppercase scheme lowercased by build tool |
| `https://API.example.com` | `https://api.example.com` | Uppercase host lowercased by build tool |
| `https://foo.com/` | _reject_ | Trailing slash forbidden |
| `https://foo.com:443` | _reject_ | Default port `:443` for `https:` MUST NOT be written |
| `wss://events.example.com:443` | _reject_ | Default port `:443` for `wss:` MUST NOT be written |
| `http://foo.com:80` | _reject_ | Default port `:80` for `http:` MUST NOT be written |
| `https://api.example.com:8443` | `https://api.example.com:8443` | Non-default port accepted verbatim |
| `https://*.example.com` | _reject_ | Wildcard host forbidden |
| `https://foo.com/api` | _reject_ | Path not permitted in origin |
| `https://foo.com?x=1` | _reject_ | Query not permitted in origin |
| `https://foo.com#top` | _reject_ | Fragment not permitted in origin |
| `ftp://files.example.com` | _reject_ | Scheme not in whitelist |
| `data:text/plain,hello` | _reject_ | Scheme not in whitelist |
| `https://café.example.com` | `https://xn--caf-dma.example.com` | UTF-8 IDN converted to Punycode by build tool |
| `https://xn--caf-dma.example.com` | `https://xn--caf-dma.example.com` | Already Punycode, accepted verbatim |
| `https://CAFÉ.example.com` | _reject_ | Build tool rejects uppercase non-ASCII; author MUST supply lowercase or ASCII |
| `https://日本.example.com` | `https://xn--wgv71a.example.com` | Multi-byte UTF-8 IDN converted to Punycode by build tool |
| `https://api.example.com` | `https://api.example.com` | Already canonical, accepted verbatim (VALID accept case) |

Shell implementers MUST produce byte-identical output to the build tool for every accepted row. A disagreement on any row is a conformance bug.

## Canonical `connect:origins` aggregateHash Fold

The set of accepted origins MUST be folded into `aggregateHash` via a synthetic xTag entry so that any change to the origin set invalidates prior grants keyed on `(dTag, aggregateHash)`. The fold is defined as normative pseudocode; the output is a 64-character lowercase hexadecimal SHA-256 digest.

```
# Input:  origins = list[str]
#         Each origin has already been validated and normalized per the
#         Origin Format rules above (scheme lowercased, host lowercased,
#         Punycode-encoded, no default port, no path / query / fragment,
#         no trailing slash).
# Output: a 64-character lowercase hexadecimal SHA-256 digest.

def connect_origins_fold(origins: list[str]) -> str:
    normalized = [o.lower() for o in origins]   # 1. lowercase (idempotent after normalization)
    normalized.sort()                            # 2. ASCII-ascending sort
    joined = "\n".join(normalized)               # 3. join with single LF, NO trailing newline
    encoded = joined.encode("utf-8")             # 4. UTF-8 encode
    digest = sha256(encoded).hexdigest()         # 5. SHA-256
    return digest.lower()                        # 6. lowercase hex (already lowercase from hexdigest)
```

The resulting digest is pushed as a synthetic xTag entry `[<hex-digest>, 'connect:origins']` into the manifest's xTag array **before** `aggregateHash` is computed. This synthetic entry participates in `aggregateHash` but is filtered out of the `['x', ...]` tag projection in the signed manifest — the public manifest surfaces origins as their own `['connect', ...]` tags, not as pseudo-file x-tag entries.

Shells performing grant lookups MUST recompute the fold from the manifest's `['connect', ...]` tags (after normalization-equivalence verification) and MUST verify that the recomputed digest matches the synthetic `connect:origins` xTag's hash. A mismatch MUST result in manifest rejection with a diagnostic; a mismatch indicates either a build-tool bug or post-signing tampering, and in neither case is it safe to proceed. This double-entry — origin tags plus folded hash — is the guard against silent origin-list tampering.

### Conformance Fixture

The following fixture is normative for interop. A conformant build tool and a conformant shell MUST both produce the specified hexadecimal digest from the specified input origins.

```
Input origins (post-normalization, as supplied by the napplet author):
  https://api.example.com
  https://xn--caf-dma.example.com
  wss://events.example.com

Sorted (ASCII-ascending):
  https://api.example.com
  https://xn--caf-dma.example.com
  wss://events.example.com

Joined with literal LF (\n), no trailing newline:
  https://api.example.com\nhttps://xn--caf-dma.example.com\nwss://events.example.com

UTF-8 byte length of the joined string: 80

Expected SHA-256 lowercase hex digest:
  cc7c1b1903fb23ecb909d2427e1dccd7d398a5c63dd65160edb0bb8b231aa742
```

The digest above can be verified independently with a one-liner such as `python3 -c "import hashlib; print(hashlib.sha256(b'https://api.example.com\nhttps://xn--caf-dma.example.com\nwss://events.example.com').hexdigest())"`. Implementations whose output differs from `cc7c1b1903fb23ecb909d2427e1dccd7d398a5c63dd65160edb0bb8b231aa742` for the specified inputs are non-conformant.

## Runtime API

The napplet-side runtime surface is a single property `connect` mounted at `window.napplet.connect`:

```cddl
NappletConnect = {
  granted: bool, ; true when all declared origins are granted for this (dTag, aggregateHash)
  origins: [* tstr], ; user-approved origins; empty on denial or unsupported shells
}
```

The shim populates `window.napplet.connect` synchronously at install time from a shell-injected `<meta name="napplet-connect-granted" content="...">` element in the served HTML. The meta tag's `content` attribute is a space-separated list of approved origins. An empty `content` value or an absent meta tag indicates `{ granted: false, origins: [] }`.

Shells implementing this NAP MUST inject the discovery meta tag when serving the napplet's HTML — even on denial, in which case the tag is emitted with an empty `content` attribute. The presence of the meta tag is the signal that the shell implements NAP-CONNECT; absence is indistinguishable from a shell that does not advertise `nap:connect`. Shells MUST NOT rely on postMessage traffic to convey grant state; `window.napplet.connect` is populated before any napplet code runs.

`window.napplet.connect` MUST default to `{ granted: false, origins: [] }` on shells that do not implement `nap:connect`, on shells that implement `nap:connect` but denied the prompt, and on shells that have not yet injected the meta tag at shim install. The property MUST NEVER be `undefined`. Napplets gracefully degrade by checking `shell.supports('nap:connect')` before depending on `window.napplet.connect.granted === true`; a napplet that branches solely on `.granted` without checking `shell.supports(...)` will produce correct (degraded) behavior but will not be able to distinguish "shell does not implement the NAP" from "user denied the prompt".

The `NappletConnect` object and its `origins` array SHOULD be frozen via `Object.freeze` at shim install so that napplet code cannot tamper with the state after install. Freezing is a best-effort integrity signal, not a security control — the napplet runs in the same execution context as the shim and can always replace `window.napplet.connect` wholesale. The shell's CSP header, not the shim object, is the authoritative enforcement.

## Capability Advertisement

Shells advertise NAP-CONNECT support through three capability strings on the NIP-5D `shell.supports()` surface:

- `shell.supports('nap:connect')` — primary capability. Returns `true` when the shell implements this NAP: honors `connect` manifest tags, injects the discovery meta tag, performs the origin-format validation and the `(dTag, aggregateHash)` grant-persistence model, and emits the runtime CSP described in `NAP-CLASS-2.md`. A napplet that branches on `.granted` SHOULD first check this capability to distinguish "shell denied" from "shell does not implement".
- `shell.supports('connect:scheme:http')` — secondary, operator policy. Returns `true` if the shell permits cleartext `http:` origins in `connect` tags. Operator policy MAY refuse cleartext entirely — some deployments require HTTPS end-to-end and advertise `false` for this capability.
- `shell.supports('connect:scheme:ws')` — secondary, parallel to `connect:scheme:http`. Returns `true` if the shell permits cleartext `ws:` origins.

Napplets declaring cleartext (`http:` or `ws:`) origins SHOULD check the corresponding `connect:scheme:*` capability before relying on a grant being obtainable and SHOULD gracefully degrade when the shell refuses cleartext. When a shell refuses to serve a napplet solely on the basis of a cleartext origin, the refuse-to-serve diagnostic MUST identify the offending origin and the operator-policy reason, so that the napplet author or a sophisticated end user can understand why the napplet cannot load.

## Shell Consent Flow

The shell consent flow is owned by the posture defined in `NAP-CLASS-2.md`. NAP-CONNECT specifies only the triggering condition and the semantic content of the prompt; the UX wording, layout, and revocation machinery are posture concerns.

The trigger for the consent flow is: the napplet's manifest contains at least one `['connect', '<origin>']` tag AND no prior approve-or-deny decision exists for the composite `(dTag, aggregateHash)` key. Shells encountering this condition MUST present a consent prompt before the iframe is served; shells MUST NOT silently approve or silently deny.

The prompt MUST capture the following semantic elements. Exact wording is a shell-UX decision, but these MUSTs are load-bearing regardless of copy:

1. Napplet name as declared in the NIP-5A manifest.
2. The complete list of requested origins (verbatim as declared in the `connect` tags — shells MUST NOT summarize, truncate, or first-presentation-elide the list).
3. An explicit statement that approval allows the napplet to **send AND receive** any data with the listed origins — the grant is not read-only and is not shell-mediated.
4. An explicit statement that the shell cannot see or filter post-grant traffic between the napplet and the listed origins.
5. Visible marking of any cleartext (`http:` or `ws:`) origins alongside the confidentiality tradeoff: the user is approving traffic that is not encrypted in transit.

Shells MUST NOT use diminutive language ('just', 'only', 'simply') that understates the trust cost of approval. Consent prompts written to minimize user friction at the cost of informed consent are misleading the user. See `NAP-CLASS-2.md` for the complete consent-flow MUSTs — including refuse-to-serve behavior on residual meta-CSP, revocation UX, at-next-load-only effective-revocation timing, and the full set of shell-side responsibilities.

## Grant Persistence

Grants are keyed on the composite `(dTag, aggregateHash)`. The `aggregateHash` input includes the `connect:origins` synthetic xTag entry defined in the **Canonical `connect:origins` aggregateHash Fold** section above. Any change to the napplet's origin set (addition, removal, or reorder after normalization) produces a new `aggregateHash`, which auto-invalidates the prior grant and triggers a fresh consent prompt on next load.

Shells MUST key grants on the exact composite. Keying on `dTag` alone is a security bug: a rebuilt napplet with a changed origin set would inherit the prior approval silently, giving the attacker-in-the-rebuild a free pass to reach newly-added origins the user never approved. This is the silent-supply-chain-upgrade vector, and the composite-key rule is the direct mitigation.

See `NAP-CLASS-2.md`'s Grant Persistence Semantics section for the full posture-level MUSTs — including the diff UX requirement on re-prompt when a prior approved grant existed under the same `dTag` but a different `aggregateHash`, and the extended-key hygiene rule (shells MAY extend the key tuple but the extension MUST be a strict superset of `(dTag, aggregateHash)` to preserve portability across conformant shells).

## Security Considerations

### Post-grant opacity

Once origins are approved and the runtime CSP is emitted, the shell has zero browser-level hook to inspect or filter subsequent network traffic between the napplet and the approved origins. This is a fundamental tradeoff and cannot be papered over. Consent-prompt language MUST reflect this reality — see **Shell Consent Flow** above. Mechanisms that would enable post-grant traffic inspection — audit logging, per-request quotas, traffic filtering — all require a service worker intercepting the napplet's network layer, and the napplet's `sandbox="allow-scripts"` iframe forbids service-worker registration. NAP-CONNECT does not attempt to provide what the browser's security model does not allow.

### Mixed-content silent failure

Browsers block `http:` and `ws:` fetches from napplets running in shells served over `https:`, regardless of the CSP header's `connect-src` value. A napplet declaring `http:` origins approved by the user will silently fail to fetch when the shell is served over `https:`, except for the `localhost` / `127.0.0.1` secure-context exceptions. The napplet-build tool SHOULD emit a build-time warning when `http:` or `ws:` origins appear in `connect`; shells SHOULD surface this reality in the consent prompt so the user is not surprised by an apparently-approved grant that produces no traffic in practice. Mixed-content is a browser-level rule enforced below the CSP layer; NAP-CONNECT cannot override it.

### Sandbox preservation

Approving `connect` origins does NOT relax the iframe sandbox. The napplet remains at an opaque origin under `sandbox="allow-scripts"`. Grants relax only the `connect-src` CSP directive; all other browser protections (Same-Origin Policy against the shell's origin, absence of cookies on napplet-origin requests, no service-worker registration, no shared-storage access, no direct access to the shell's DOM) remain in force. A NAP-CONNECT grant is a network-egress permission scoped to specific origins — it is not a sandbox escape, it is not a capability-expansion beyond network, and it does not interact with NAP-RESOURCE's own enforcement layer.

### Weaker posture than NAP-RESOURCE

NAP-CONNECT is substantially weaker than NAP-RESOURCE in terms of shell-side visibility and policy enforcement. NAP-RESOURCE proxies all fetches through the shell, which can MIME-sniff response bytes, enforce a private-IP block list at DNS-resolution time, rasterize SVG in a sandboxed worker, cap response size, limit redirect chains, and re-validate per hop. NAP-CONNECT does none of those things — the grant is a full trust vote for the listed origins, and every byte flowing through a NAP-CONNECT grant is outside the shell's view. Napplet authors SHOULD default to NAP-RESOURCE for everything NAP-RESOURCE can express, and reach for NAP-CONNECT only when the use case genuinely requires direct `POST`, WebSocket, SSE, streaming responses, custom headers, or long-lived connections.

## Graceful Degradation

Napplets that might want network access SHOULD branch on four states, in priority order:

1. `shell.supports('nap:connect') === true` AND `window.napplet.connect.granted === true` — the user approved the manifest's `connect` origins. Use direct `fetch` / `WebSocket` / `EventSource` against `window.napplet.connect.origins`.
2. `shell.supports('nap:connect') === true` AND `window.napplet.connect.granted === false` — user denied the prompt, the prompt has not yet been presented, or the shell implements the NAP but chose not to advertise grants for this napplet. Fall back to NAP-RESOURCE for anything the resource NAP can express; otherwise degrade the affected feature gracefully. Optionally display a napplet-side affordance prompting the user to re-approve through the shell's revocation UI.
3. `shell.supports('nap:connect') === false` AND `shell.supports('nap:resource') === true` — shell does not implement NAP-CONNECT at all; use NAP-RESOURCE for read-only byte fetches. POST / WebSocket / SSE features are unavailable in this environment and MUST be degraded gracefully; the napplet MUST NOT pressure the user to switch shells.
4. Neither `nap:connect` nor `nap:resource` is advertised — the napplet cannot reach the network at all. Display appropriate UX (offline mode, read-only-from-cache, or a clear explanation that the feature requires a shell implementing one of the network NAPs) and avoid silent failure.

States (1) and (2) are the common paths in modern shells. States (3) and (4) are graceful-shutdown paths that napplets targeting restrictive or minimal shells MUST handle without crashing, hanging, or presenting confusing error messages.

## References

- `NIP-5D` — Parent transport spec: JSON envelope wire format, iframe sandbox model, capability advertisement surface.
- `NAP-CLASS.md` — Parent class track: defines the `class.assigned` envelope, the `window.napplet.class` runtime surface, and the authoring rules for track members.
- `NAP-CLASS-1.md` — Default strict-baseline posture. Triggered when a napplet declares no `connect` tags, and also when a user denies the NAP-CLASS-2 consent prompt (denied napplets are served under the NAP-CLASS-1 posture).
- `NAP-CLASS-2.md` — User-approved explicit-origin posture triggered by presence of `connect` tags. Owns the CSP shape, consent-flow MUSTs, grant-persistence semantics, residual-meta-CSP refuse-to-serve requirement, and revocation UX.
- `NAP-RESOURCE` — Sibling NAP providing shell-mediated read-only byte fetching. Authors SHOULD default to NAP-RESOURCE for everything it can express and reach for NAP-CONNECT only when direct network access is genuinely required.
- WHATWG URL — Origin format and Punycode IDN conversion rules.
- WHATWG Fetch, Mixed Content — Browser-level mixed-content enforcement (HTTPS shells cannot fetch HTTP origins; `localhost` / `127.0.0.1` secure-context exceptions).
