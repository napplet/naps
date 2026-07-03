NAP-LINK
========

Shell-Mediated Link Opening
---------------------------

`draft`

**NAP ID:** NAP-LINK
**Domain:** `link`
**Web binding (NIP-5D):** `window.napplet.link` · `shell.supports("link")`

## Description

NAP-LINK lets a sandboxed napplet ask the shell to open an external URL for the user. The shell owns the navigation, policy, prompting, and browser context. The napplet never gets direct navigation authority, network access, opener access, or fetched bytes.

This is for user navigation, including links to pages that cannot be opened correctly inside the napplet sandbox because of Cross-Origin-Opener-Policy (COOP), frame, popup, or cross-origin restrictions. It is not a fetch API. Use NAP-RESOURCE when the napplet needs bytes returned. Parsers that turn external URLs into clickable UI SHOULD route those clicks to `link.open`.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `open` | `url` (`tstr`), optional `options` (`LinkOpenOptions`) | `LinkOpenResult` | `link.open` / `link.open.result` |

### Schemas

`LinkOpenOptions`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `label` | `tstr` | no | Human-readable prompt text supplied by the napplet. |

`LinkOpenResult`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | `opened` or `denied` | yes | Whether the shell accepted or refused the navigation request. |

**`open(url, options?)`** -- Requests that the shell open `url` for the user. `url` MUST be an absolute URL. `label` is optional human-readable text for the shell prompt; it is not trusted policy input. Resolves with `"opened"` when the shell accepted the request and handed it to the user agent, or `"denied"` when user or shell policy refused it.

## Wire Protocol

`link.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `link.open` | napplet -> shell | `id`, `url`, `options?` |
| `link.open.result` | shell -> napplet | `id`, `status`, `error?` |

Key design notes:
- `id` correlates the request and result.
- `link.open` is user navigation, not resource retrieval.
- The shell chooses the final target: new tab, external browser, in-shell browser, native handler, or denial.
- The shell MUST NOT expose `window.opener` or equivalent authority back to the napplet.

### Examples

**Open an external page:**
```
-> { "type": "link.open", "id": "l1", "url": "https://example.com/post/123", "options": { "label": "Read post" } }
<- { "type": "link.open.result", "id": "l1", "status": "opened" }
```

**Denied by policy:**
```
-> { "type": "link.open", "id": "l2", "url": "file:///etc/passwd" }
<- { "type": "link.open.result", "id": "l2", "status": "denied", "error": "unsupported-scheme" }
```

### Error Handling

Result messages MAY include `error` when `status` is `"denied"`. Common errors are `"invalid-url"`, `"unsupported-scheme"`, `"blocked-by-policy"`, and `"user-denied"`.

## Shell Behavior

- The shell MUST validate `url` before opening it.
- The shell MUST reject relative URLs.
- The shell MUST reject `javascript:`, `data:`, `blob:`, `file:`, and other local or code-bearing schemes.
- The shell MUST respond to every request with a result message carrying the same `id`.
- The shell SHOULD open web links in a context outside the napplet sandbox.
- The shell SHOULD use `noopener` or an equivalent opener-isolation mechanism.
- The shell MAY prompt the user before opening a link.
- The shell MAY persist user choices per napplet, per hostname, or per URL.
- The shell MAY support non-web schemes such as `nostr:` or `mailto:` by policy.

Prompt choices MAY include:
- allow once
- allow all links from this napplet
- allow links to this hostname from this napplet
- deny once
- deny all links from this napplet

## Security Considerations

- Opening a link is a user-visible action. Shells SHOULD require a user gesture or prompt unless policy already allows the request.
- Prompts SHOULD show the napplet name, stable napplet identifiers, author when known, and destination URL or hostname.
- Shells SHOULD rate-limit denied or repeated link requests to prevent prompt spam.
- Shells SHOULD treat IDN, punycode, redirects, and lookalike hostnames as phishing risks in prompt UI.
- `label` is napplet-supplied text. Shells MUST NOT use it as proof of destination or intent.
- NAP-LINK does not grant direct network access. A napplet that needs direct fetch, WebSocket, SSE, or custom headers needs a different capability.

## Implementations

- (none yet)
