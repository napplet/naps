NAP-DISPLAY
===========

Runtime-Controlled Pixel Displays
---------------------------------

`draft`

**NAP ID:** NAP-DISPLAY
**Domain:** `display`
**Web binding (NIP-5D):** `window.napplet.display` · `shell.supports("display")`

## Description

NAP-DISPLAY lets a napplet list displays that the runtime controls and push
sparse batches of logical RGB pixels. The napplet supplies coordinates and
color bytes. The runtime reports each display's logical geometry and device
type, then maps each request to the device's color depth, orientation, layout,
and refresh model. The shell mediates access and competing writes.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `list` | none | list of `DisplayDevice` | `display.list` / `display.list.result` |
| `push` | `displayId` (text), `pixels` (list of `Pixel`) | `null` | `display.push` / `display.push.result` |

### Schemas

`DisplayType` accepts `"lcd"`, `"eink"`, `"led-matrix"`, or `"other"`.

`DisplayDevice` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `id` | yes | text | Opaque runtime-assigned stable identifier. |
| `width` | yes | integer | Logical width in pixels. Positive. |
| `height` | yes | integer | Logical height in pixels. Positive. |
| `type` | yes | `DisplayType` | Runtime-determined device type. |

The runtime MUST return the same `id` for the same display across list calls,
ordering changes, and runtime restarts. It MUST NOT reuse an `id` for a different
display.

`Pixel` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `x` | yes | integer | Zero-based logical x-coordinate. |
| `y` | yes | integer | Zero-based logical y-coordinate. |
| `bytes` | yes | list of integer | Exactly three sRGB bytes: red, green, blue. |

Each byte MUST fall within `0..255`. Each coordinate MUST satisfy
`0 <= x < width` and `0 <= y < height` for the selected display. A batch MUST
contain each coordinate at most once.

**`list()`** returns the displays that the shell permits the napplet to use. Each
call returns a snapshot and MAY reflect connection or policy changes.

**`push(displayId, pixels)`** submits one non-empty pixel batch. A successful
result means the runtime accepted the entire batch. It does not promise that
the hardware has completed its refresh.

## Wire Protocol

`display.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `display.list` | napplet -> shell | `id` |
| `display.list.result` | shell -> napplet | `id`, `displays`, `error?` |
| `display.push` | napplet -> shell | `id`, `displayId`, `pixels` |
| `display.push.result` | shell -> napplet | `id`, `error?` |

Key design notes:

- `id` correlates each request and result.
- Coordinates use a top-left origin. `x` increases rightward; `y` increases
  downward.
- RGB bytes describe logical color, not a native framebuffer. The runtime never
  exposes device byte order, scan order, or driver commands.
- The runtime validates a whole batch before applying any pixel from it.

### Examples

**List displays:**

```
-> { "type": "display.list", "id": "d1" }
<- { "type": "display.list.result", "id": "d1", "displays": [
     { "id": "desk", "width": 800, "height": 480, "type": "eink" },
     { "id": "badge", "width": 32, "height": 8, "type": "led-matrix" }
   ] }
```

**Push two pixels:**

```
-> { "type": "display.push", "id": "d2", "displayId": "badge", "pixels": [
     { "x": 0, "y": 0, "bytes": [255, 0, 0] },
     { "x": 1, "y": 0, "bytes": [0, 32, 255] }
   ] }
<- { "type": "display.push.result", "id": "d2" }
```

### Error Handling

Result messages MAY include `error`. This NAP defines `"not-found"`,
`"unavailable"`, `"denied"`, `"invalid-pixel"`, and `"too-large"`. A failed
`display.list` returns an empty `displays` list. A failed `display.push` applies
no pixels.

## Runtime and Shell Behavior

- The runtime MUST discover devices and determine their `id`, `width`, `height`,
  and `type`. A napplet cannot supply or alter descriptors.
- The runtime MUST map logical RGB pixels to the selected device. It SHOULD
  preserve position, relative luminance, and color as closely as the device
  allows.
- The runtime MAY rotate coordinates, reorder scan lines, quantize colors,
  dither, adjust brightness, or coalesce refreshes to match the device.
- The runtime MUST process accepted batches in arrival order for each display.
- The shell MUST mediate which displays each napplet can list and write.
- The shell MUST respond to every request with the matching result and `id`.
- The shell MAY reject, rate-limit, or cap batches according to device and
  per-napplet policy.

## Security Considerations

- Display access can reveal physical hardware and affect user-visible output.
  The shell SHOULD require explicit per-napplet permission.
- Stable identifiers can aid fingerprinting. The runtime SHOULD use opaque
  identifiers instead of hardware serial numbers.
- Large or rapid batches can exhaust CPU, bandwidth, power, or device lifetime.
  The shell SHOULD bound batch size and update rate.
- The fixed RGB schema prevents a napplet from sending native driver commands.

## Implementations

- (none yet)

## Changelog

- `62f2d8b` - Introduced NAP-DISPLAY for runtime-controlled device discovery and RGB pixel updates.
