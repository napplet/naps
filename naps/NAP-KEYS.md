NAP-KEYS
========

Keyboard Forwarding and Action Keybindings
------------------------------------------

`draft`

**NAP ID:** NAP-KEYS
**Domain:** `keys`
**Web binding (NIP-5D):** `window.napplet.keys` · `shell.supports("keys")`

## Description

NAP-KEYS provides bidirectional keyboard interaction between napplets and the shell. Sandboxed iframes capture keyboard events and prevent them from reaching the parent window, so shell-level hotkeys (workspace switching, command palette, etc.) do not work when a napplet has focus. NAP-KEYS solves this with two mechanisms: keyboard forwarding (napplet sends keystrokes to the shell) and action registration (napplet declares named actions that the shell can bind to keys).

When a napplet registers an action and the shell binds it to a key, the shell pushes binding updates to the napplet. The napplet then suppresses forwarding for bound keys and triggers the action locally. This eliminates dual-fire (both napplet and shell reacting to the same keystroke) and gives the shell full control over the keymap.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `registerAction` | `action` (`Action`) | `RegisterResult` | `keys.registerAction` / `keys.registerAction.result` |
| `unregisterAction` | `actionId` (`tstr`) | none | `keys.unregisterAction` |
| `forward` | keyboard event fields | none | `keys.forward` |
| `onAction` | `actionId` (`tstr`), local callback | `Subscription` handle | local helper; not a wire message |

### Schemas

```cddl
Action = {
  id: tstr,    ; unique action identifier, e.g. "editor.save"
  label: tstr, ; human-readable label for shell keybinding UI
  ? defaultKey: tstr, ; suggested binding hint, e.g. "Ctrl+S"
}

RegisterResult = {
  actionId: tstr,
  ? binding: tstr, ; key combo assigned by the shell, absent when unbound
}
```

**`registerAction(action)`** -- Declares a named action that the shell can bind to a key. The `defaultKey` is a hint only — the shell decides the actual binding. Returns the assigned binding if one was made immediately. The shell MAY defer binding until the user configures it.

**`unregisterAction(actionId)`** -- Removes a previously registered action. The shell removes any binding for it and updates the napplet's suppress list.

**`forward(event)`** -- Sends a keystroke to the shell. The shim calls this automatically for keydown events that are not in the suppress list. Napplets do not call this directly under normal use.

**`onAction(actionId, callback)`** -- Registers a local handler for when a bound key is pressed. This is NOT a wire message — it is SDK convenience that listens for the key locally and invokes the callback. Returns a Subscription with `close()` to stop listening.

## Wire Protocol

`keys.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `keys.forward` | napplet -> shell | `key`, `code`, `ctrl`, `alt`, `shift`, `meta` |
| `keys.registerAction` | napplet -> shell | `id` (correlation), `action` (Action object) |
| `keys.registerAction.result` | shell -> napplet | `id`, `actionId`, `binding`? |
| `keys.unregisterAction` | napplet -> shell | `actionId` |
| `keys.bindings` | shell -> napplet | `bindings` (array of `{ actionId, key }`) |
| `keys.action` | shell -> napplet | `actionId` |

Key design notes:
- `keys.forward` is fire-and-forget (no `id`). High frequency — one per keystroke.
- `keys.registerAction` / `keys.registerAction.result` use `id` for correlation.
- `keys.unregisterAction` is fire-and-forget.
- `keys.bindings` is a shell-initiated push. Sent whenever bindings change (initial load, user rebind, action unregister). Contains the complete binding list — not a diff.
- `keys.action` is a shell-initiated push when the shell wants to trigger an action in the napplet (e.g., the user pressed a globally-bound key while a different napplet has focus, but the action belongs to this napplet).

### Key Combo Format

Key combinations use the format `Modifier+Key` where modifiers are `Ctrl`, `Alt`, `Shift`, `Meta` and the key is the `KeyboardEvent.key` value. Examples: `Ctrl+S`, `Alt+Shift+P`, `F5`, `Escape`.

Shells MUST normalize combos to alphabetical modifier order: `Alt+Ctrl+Shift+Meta+Key`. Napplets SHOULD use this order in `defaultKey` hints.

### Forwarding Fields

The `keys.forward` message carries the essential `KeyboardEvent` fields:

| Field | Type | Source |
|-------|------|--------|
| `key` | string | `KeyboardEvent.key` — character intent (e.g., `"s"`, `"Enter"`, `"ArrowDown"`) |
| `code` | string | `KeyboardEvent.code` — physical key position (e.g., `"KeyS"`, `"Enter"`, `"ArrowDown"`) |
| `ctrl` | boolean | `KeyboardEvent.ctrlKey` |
| `alt` | boolean | `KeyboardEvent.altKey` |
| `shift` | boolean | `KeyboardEvent.shiftKey` |
| `meta` | boolean | `KeyboardEvent.metaKey` |

Both `key` and `code` are included. `key` is used for command shortcut matching (intent-based, respects keyboard layout). `code` is available for shells that need position-based matching (e.g., WASD game controls). The shell decides which field to match against.

### Smart Forwarding

The napplet shim maintains a local suppress list derived from `keys.bindings` messages. On each keydown event:

1. If the event target is a text input (`<input>`, `<textarea>`, `contenteditable`), do not forward. Let the napplet handle it normally.
2. If the key is a bare modifier (`Control`, `Alt`, `Shift`, `Meta`), do not forward.
3. If `event.isComposing` is true (IME active), do not forward.
4. Normalize the keystroke to a combo string (e.g., `Ctrl+s`).
5. If the combo matches an entry in the suppress list:
   - Call `event.preventDefault()` to prevent the napplet's default handler.
   - Trigger the local action handler (registered via `onAction()`).
   - Do NOT send `keys.forward`.
6. If the combo does not match:
   - Send `keys.forward` to the shell.

This ensures bound keys are handled locally with zero latency (no postMessage round-trip), while unbound keys are forwarded to the shell for global hotkey processing.

### Reserved Keys

The following keys MUST NOT be suppressed by smart forwarding, even if they appear in the suppress list. These are essential for accessibility and browser function:

- `Tab` and `Shift+Tab` — focus navigation (WCAG 2.1.2)
- `Escape` — close dialogs, exit modes
- Browser-reserved shortcuts (e.g., `Ctrl+W`, `Ctrl+T`, `Ctrl+L`) — these are intercepted by the browser before JavaScript and cannot be suppressed anyway

Shells MUST NOT bind actions to reserved keys. Napplets MUST NOT use reserved keys in `defaultKey` hints.

### Examples

**Register an action:**
```
-> { "type": "keys.registerAction", "id": "r1", "action": { "id": "editor.save", "label": "Save", "defaultKey": "Ctrl+S" } }
<- { "type": "keys.registerAction.result", "id": "r1", "actionId": "editor.save", "binding": "Ctrl+S" }
```

**Shell pushes binding updates:**
```
<- { "type": "keys.bindings", "bindings": [
     { "actionId": "editor.save", "key": "Ctrl+S" },
     { "actionId": "editor.undo", "key": "Ctrl+Z" }
   ]}
```

**Forward an unbound keystroke:**
```
-> { "type": "keys.forward", "key": "j", "code": "KeyJ", "ctrl": false, "alt": false, "shift": false, "meta": false }
```

**Shell triggers an action in the napplet:**
```
<- { "type": "keys.action", "actionId": "editor.save" }
```

**Unregister an action:**
```
-> { "type": "keys.unregisterAction", "actionId": "editor.save" }
<- { "type": "keys.bindings", "bindings": [
     { "actionId": "editor.undo", "key": "Ctrl+Z" }
   ]}
```

**Error in registration:**
```
<- { "type": "keys.registerAction.result", "id": "r1", "actionId": "editor.save", "error": "duplicate action ID" }
```

### Error Handling

`keys.registerAction.result` includes an `error` field (string) on failure. Common errors: `"duplicate action ID"`, `"invalid action"`. When `error` is present, `binding` is undefined.

## Shell Behavior

- The shell MUST accept `keys.forward` messages and process the forwarded keystroke as if it occurred in the shell's own context (for global hotkey matching, command palette, workspace switching, etc.).
- The shell MUST respond to `keys.registerAction` with a `keys.registerAction.result` carrying the same `id`.
- The shell MUST push `keys.bindings` whenever the binding set for a napplet changes (initial load, user rebind, action register/unregister). The message contains the complete list, not a diff.
- The shell MAY assign a binding immediately using the napplet's `defaultKey` hint, or MAY leave the action unbound until the user configures it.
- The shell MAY send `keys.action` to trigger a napplet's registered action from outside (e.g., a global keybinding targets a specific napplet's action even when that napplet does not have focus).
- The shell MAY enforce ACL checks on `keys` capabilities (e.g., restricting which napplets can register actions or receive forwarded keystrokes).
- The shell MUST NOT bind actions to reserved keys (Tab, Shift+Tab, Escape, browser-reserved shortcuts).
- The shell MUST silently ignore `keys.forward` messages from napplets it has not mapped to an identity.

## Security Considerations

- Keystroke forwarding exposes user input. The shell SHOULD NOT forward keystrokes to other napplets — `keys.forward` is strictly napplet-to-shell.
- Action registration is a trust operation. A napplet that registers too many actions or deceptive labels could confuse the user. Shells MAY cap the number of actions per napplet and SHOULD display action labels in a shell-controlled UI (not napplet-provided UI).
- The suppress list is delivered by the shell and applied locally. A compromised napplet could ignore the suppress list and forward all keys anyway — the shell MUST NOT rely on the napplet to suppress correctly for security-critical bindings.
- `keys.action` (shell-triggered) allows the shell to invoke actions in the napplet. The napplet trusts the shell (NIP-5D security model) but SHOULD validate the `actionId` against its own registered actions.
- Keyboard event data (`key`, `code`, modifier flags) does not contain sensitive information beyond what the user is typing. However, forwarding from text inputs is suppressed to prevent accidental password/credential leakage.

## Implementations

- (none yet)

## Changelog

- `c7a7a34` - Introduced NAP-KEYS for keyboard forwarding and action keybindings.
