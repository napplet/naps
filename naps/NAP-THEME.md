NAP-THEME
=========

Shell-Provided Theming
----------------------

`draft`

**NAP ID:** NAP-THEME
**Domain:** `theme`
**Web binding (NIP-5D):** `window.napplet.theme` · `shell.supports("theme")`

## Description

NAP-THEME provides napplets with read-only access to the shell's active theme. The shell owns the theme and delivers it as a typed payload — napplets have no knowledge of where the theme originates or how it is stored. The theme includes colors (required), as well as optional fonts, background media, and a human-readable title. Napplets can query the current theme on demand and receive automatic notifications when the shell's active theme changes.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `get` | none | `Theme` | `theme.get` / `theme.get.result` |

### Schemas

`ThemeColors` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `background` | yes | text | Hex color, e.g. `#1a1a2e`. |
| `text` | yes | text | Hex color. |
| `primary` | yes | text | Hex color. |

`ThemeFont` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `name` | yes | text | Font family name. |
| `url` | yes | text | URL to font file. |

`ThemeBackground` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `url` | yes | text | URL to background image or media. |
| `mode` | yes | text | CSS background-size value, e.g. `cover`. |
| `mime` | yes | text | MIME type, e.g. `image/jpeg`. |

`ThemeFonts` fields:

| Field | Required | Type |
|-------|----------|------|
| `body` | no | `ThemeFont` |
| `title` | no | `ThemeFont` |

`Theme` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `colors` | yes | `ThemeColors` | Active color set. |
| `fonts` | no | `ThemeFonts` | Optional font set. |
| `background` | no | `ThemeBackground` | Optional background media. |
| `title` | no | text | Human-readable theme name. |

`theme.changed` is received as a message event, not via a method call. Napplets listen for it via the standard `postMessage` listener. There is no subscribe or unsubscribe mechanism — change notifications are automatic for all napplets that support NAP-THEME.

## Wire Protocol

`theme.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `theme.get` | napplet -> shell | `id` |
| `theme.get.result` | shell -> napplet | `id`, `theme` (Theme object) |
| `theme.changed` | shell -> napplet | `theme` (Theme object) |

Key design notes:
- `theme.get` / `theme.get.result` use `id` for correlation.
- `theme.changed` has no `id` — it is a shell-initiated push with no napplet request to correlate.
- There is no subscribe or unsubscribe. All napplets that declare `theme` in their manifest `requires` tags automatically receive `theme.changed` when the active theme changes.

### Examples

**Get active theme (full payload):**
```
-> { "type": "theme.get", "id": "t1" }
<- { "type": "theme.get.result", "id": "t1", "theme": {
     "colors": { "background": "#1a1a2e", "text": "#e0e0e0", "primary": "#6c3ce0" },
     "fonts": {
       "body": { "name": "Inter", "url": "https://example.com/inter.woff2" },
       "title": { "name": "Playfair Display", "url": "https://example.com/playfair.woff2" }
     },
     "background": { "url": "https://example.com/bg.jpg", "mode": "cover", "mime": "image/jpeg" },
     "title": "MK Dark Theme"
   }}
```

**Get active theme (minimal — colors only):**
```
-> { "type": "theme.get", "id": "t2" }
<- { "type": "theme.get.result", "id": "t2", "theme": {
     "colors": { "background": "#ffffff", "text": "#333333", "primary": "#0066cc" }
   }}
```

**Theme changed (auto-push from shell, no id):**
```
<- { "type": "theme.changed", "theme": {
     "colors": { "background": "#0d1117", "text": "#c9d1d9", "primary": "#58a6ff" },
     "title": "GitHub Dark"
   }}
```

### Error Handling

Any result message MAY include an `error` field (string). When `error` is present, other result fields are undefined.

```
<- { "type": "theme.get.result", "id": "t1", "error": "no active theme" }
```

## Shell Behavior

- The shell MUST respond to `theme.get` with a `theme.get.result` carrying the same `id`.
- The shell MUST include `colors` with all three fields (`background`, `text`, `primary`) in every theme payload. The `fonts`, `background`, and `title` fields are optional and MAY be omitted.
- The shell MUST broadcast `theme.changed` to all napplets that declare `theme` in their manifest `requires` tags when the active theme changes. The shell MAY also broadcast `theme.changed` to napplets that do not declare it, at the shell's discretion.
- The shell MAY source the theme from any origin: a Nostr kind 16767 event, a user preference, a hardcoded default, or any other mechanism. The theme source is an implementation detail invisible to napplets.

### Kind 16767 Mapping

Shells that source themes from Nostr kind 16767 events (active profile themes, as used by Ditto and compatible clients) SHOULD map event tags to the typed theme payload as follows:

| Kind 16767 Tag | Theme Payload Field |
|----------------|---------------------|
| `["c", "<hex>", "background"]` | `colors.background` |
| `["c", "<hex>", "text"]` | `colors.text` |
| `["c", "<hex>", "primary"]` | `colors.primary` |
| `["f", "<name>", "<url>", "body"]` | `fonts.body.name`, `fonts.body.url` |
| `["f", "<name>", "<url>", "title"]` | `fonts.title.name`, `fonts.title.url` |
| `["bg", "url <url>", "mode <mode>", "m <mime>"]` | `background.url`, `background.mode`, `background.mime` |
| `["title", "<name>"]` | `title` |

The `bg` tag uses positional string prefixes (`url `, `mode `, `m `). Shells MUST strip these prefixes when extracting values before placing them in the payload.

This mapping is informational guidance for shells that choose to source from kind 16767 events. It is not a requirement — shells that source themes from other origins MUST still deliver a valid Theme payload.

## Security Considerations

- Theme data is read-only. Napplets cannot modify the shell's active theme via this interface.
- Font URLs and background media URLs may point to external resources. Shells SHOULD validate these URLs before forwarding them to napplets and MAY restrict loading to trusted origins or proxy the resources.
- The shell controls theme delivery. A napplet cannot influence what theme other napplets receive.
- Color values are strings. Napplets SHOULD validate hex color format (e.g., `#rrggbb`) before applying values to prevent injection via malformed color strings.
- The shell MAY enforce ACL checks before responding to `theme.get` in restricted environments.

## Implementations

- (none yet)
