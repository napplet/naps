NAP-GAMEPAD
===========

Shell-Mediated Gamepad Input
----------------------------

`draft`

**NAP ID:** NAP-GAMEPAD
**Domain:** `gamepad`
**Depends:** none.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.gamepad`; domain presence signals availability.

## Description

NAP-GAMEPAD gives napplets game controller input without direct device access.
A controller is a user-level device: every napplet that can read it natively
sees every press, so napplets running side by side all react to the same input.
The runtime instead owns the only native reader. The shell delivers controller
snapshots to subscribed napplets and gives live values only to the napplet that
has input focus. Other napplets see which controllers are connected, but their
values are neutral.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `getGamepads` | none | list of `GamepadSlot` | latest `gamepad.state` |
| `available` | none | boolean | latest `gamepad.state` |
| `focused` | none | boolean | latest `gamepad.state` |
| `onChange` | handler for `GamepadState` | `Subscription` handle | `gamepad.state` |

### Schemas

`GamepadSlot` is a `Gamepad` or `null`. `null` marks an empty slot.

`GamepadButton` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `value` | yes | number | `0` to `1`. Analog travel for triggers; `0` or `1` for digital buttons. |
| `pressed` | yes | boolean | Device-reported pressed state. |
| `touched` | yes | boolean | Device-reported touch state; equals `pressed` when the device has no touch sensing. |

`Gamepad` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `index` | yes | integer | Slot position. Stable while the controller stays connected. |
| `id` | yes | text | Runtime-reported controller description. Not an authenticated identity. |
| `mapping` | yes | text | `"standard"` when buttons and axes follow the [W3C standard layout](https://www.w3.org/TR/gamepad/#remapping); `""` otherwise. Other values MAY appear. |
| `connected` | yes | boolean | `true` in every slot of a snapshot. |
| `timestamp` | yes | number | Runtime time of the last input change, in milliseconds. `0` in neutral slots. |
| `axes` | yes | list of number | Each `-1` to `1`. |
| `buttons` | yes | list of `GamepadButton` | Button order follows `mapping`. |

`GamepadUnavailableReason` accepts `"blocked"` or `"unavailable"`.

`GamepadState` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `available` | yes | boolean | `false` when the runtime cannot read controllers. |
| `reason` | no | `GamepadUnavailableReason` | Present only when `available` is `false`. `"blocked"` means host policy denied the runtime; `"unavailable"` means no controller API exists. |
| `focused` | yes | boolean | `true` when this napplet currently receives live values. |
| `pads` | yes | list of `GamepadSlot` | Complete slot list in runtime order. Not a diff. |

**`getGamepads()`** returns the slots from the latest snapshot. Before the first
snapshot arrives it returns only `null` slots, possibly none. It never waits for
the wire.

**`available`** and **`focused`** return those fields from the latest snapshot.
Before the first snapshot, `available` is `true` and `focused` is `false`.

**`onChange(handler)`** calls `handler` with each new `GamepadState` and returns
a `Subscription` handle. Closing the handle removes the local handler.

The runtime-provided binding sends `gamepad.subscribe` before or on first use.
It MAY subscribe when the domain is installed. It MAY send `gamepad.unsubscribe`
when the napplet no longer needs input.

## Wire Protocol

`gamepad.*` messages use the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `gamepad.subscribe` | napplet -> shell | none |
| `gamepad.unsubscribe` | napplet -> shell | none |
| `gamepad.state` | shell -> napplet | `available`, `reason`?, `focused`, `pads` |

Key design notes:
- All three messages are fire-and-forget. None carries `id`.
- `gamepad.state` is a complete snapshot, sent right after `gamepad.subscribe`
  and then whenever the napplet's view changes.
- Snapshots are pushed, not polled. A napplet reads the latest snapshot locally
  as often as it renders.

### Examples

**Subscribe; this napplet has focus:**
```
-> { "type": "gamepad.subscribe" }
<- { "type": "gamepad.state", "available": true, "focused": true, "pads": [
     null,
     { "index": 1, "id": "Xbox Wireless Controller", "mapping": "standard",
       "connected": true, "timestamp": 18235.4,
       "axes": [0.0, -1.0, 0.0, 0.0],
       "buttons": [{ "value": 1, "pressed": true, "touched": true },
                   { "value": 0, "pressed": false, "touched": false }] }
   ]}
```

**Focus moves to another napplet; this napplet receives a neutral snapshot:**
```
<- { "type": "gamepad.state", "available": true, "focused": false, "pads": [
     null,
     { "index": 1, "id": "Xbox Wireless Controller", "mapping": "standard",
       "connected": true, "timestamp": 0,
       "axes": [0, 0, 0, 0],
       "buttons": [{ "value": 0, "pressed": false, "touched": false },
                   { "value": 0, "pressed": false, "touched": false }] }
   ]}
```

**Host policy denies controller access to the runtime:**
```
<- { "type": "gamepad.state", "available": false, "reason": "blocked", "focused": false, "pads": [] }
```

**Stop receiving input:**
```
-> { "type": "gamepad.unsubscribe" }
```

### Error Handling

There are no request/result pairs. Unavailability is reported in state through
`available` and `reason`. The shell silently ignores malformed or excessive
subscription messages.

## Shell Behavior

- The runtime MUST be the only reader of physical controllers on behalf of
  napplets. Where the host offers a native controller API, the runtime MUST
  deny it to napplets.
- The shell MUST send a `gamepad.state` snapshot after each `gamepad.subscribe`.
  It MUST then send a snapshot whenever that napplet's view changes, and SHOULD
  NOT resend an unchanged snapshot.
- The shell SHOULD send at most one snapshot per display frame per napplet.
- The shell MUST deliver live `axes`, `buttons` and `timestamp` values only to
  the napplet that holds input focus, and only while the host surface is visible
  and focused.
- For every other subscribed napplet, the shell MUST send each connected slot
  with every axis `0`, every button `value` `0` with `pressed` and `touched`
  `false`, and `timestamp` `0`. It MUST send this neutral snapshot as soon as
  focus leaves a napplet.
- The shell MUST report connection changes to every subscribed napplet,
  including napplets without focus.
- The shell MUST report `available: false` with a `reason` when the runtime
  itself cannot read controllers.
- The shell MUST stop sending `gamepad.state` after `gamepad.unsubscribe` and
  when the napplet's session ends.
- The shell SHOULD clamp axes to `-1`..`1` and button values to `0`..`1`. It
  MAY cap the number of slots, axes and buttons, and SHOULD rate-limit
  subscription changes.
- The shell MAY enforce ACL checks on `gamepad` capabilities.

## Napplet Guidance

- A napplet SHOULD treat `focused: false` as idle input, not as release events
  to act on.
- A button that is already held when `focused` becomes `true` SHOULD NOT be
  treated as a new press.
- A napplet SHOULD route players by `index`, not by list position, and SHOULD
  keep keyboard and pointer controls.
- The domain is optional. A napplet SHOULD NOT declare `gamepad` as required
  unless it cannot run without controllers.

## Web Projection Guidance

This section is projection guidance only. The contract above does not change.

- The shell denies the native API to napplet frames with the Permissions-Policy
  feature `gamepad`, for example `allow="gamepad 'none'"` on each napplet
  `iframe`. CSP cannot govern controller access.
- Denying the feature leaves `navigator.getGamepads` present in the frame, but
  calls to it throw `SecurityError`. A binding MAY then replace only that
  operation with one served from `gamepad.state`. Existing games and libraries
  then keep working unchanged. Such a replacement SHOULD:
  - keep the native property attributes, name, length and receiver check;
  - return the host's native slot shape, including `null` empty slots;
  - give pads and buttons the native `Gamepad` and `GamepadButton` prototypes,
    and dispatch `gamepadconnected` / `gamepaddisconnected` with
    `GamepadEvent`'s prototype;
  - throw `SecurityError` when the shell reports `"blocked"`; and
  - leave the native API untouched when it is not denied.

## Security Considerations

- Controller input is user input. The isolation guarantee depends on the
  runtime denying every native controller path to napplets. A shell that leaves
  the native API reachable provides no isolation.
- Neutral snapshots carry no values or timing, so a napplet without focus
  cannot observe another napplet's input.
- `id` strings can identify controller models and aid fingerprinting. The shell
  MAY generalize them. Napplets MUST NOT treat `id` as an authenticated or
  stable device identity.
- Subscriptions cost the runtime a polling loop. The shell SHOULD bound
  subscription churn and SHOULD stop polling when no napplet is subscribed.
- This domain grants no device commands. Haptics, raw HID access and remapping
  are out of scope.

## Implementations

- [napplet.soy](https://github.com/hzrd149/napplet-soy/tree/feat/gamepad-isolation): shell broker ([`gamepad-session.ts`](https://github.com/hzrd149/napplet-soy/blob/feat/gamepad-isolation/packages/runtime/src/gamepad-session.ts)), web binding with native-API replacement ([`prelude.ts`](https://github.com/hzrd149/napplet-soy/blob/feat/gamepad-isolation/packages/runtime/src/prelude.ts)), and Permissions-Policy denial.

## Changelog

- `31b3613` - Introduced NAP-GAMEPAD for focus-scoped, shell-mediated controller snapshots.
