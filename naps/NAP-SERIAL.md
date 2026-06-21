NAP-SERIAL
==========

Shell-Mediated Serial Device Access
-----------------------------------

`draft`

**NAP ID:** NAP-SERIAL
**Domain:** `serial`
**Web binding (NIP-5D):** `window.napplet.serial` · `shell.supports("serial")`

## Description

NAP-SERIAL lets a sandboxed napplet communicate with user-approved serial
devices without receiving direct access to browser serial APIs, native handles,
or OS device paths. The runtime owns device selection, permission prompts, port
opening, read loops, write ordering, disconnect handling, and policy. The
napplet gets a small byte-oriented session API.

This NAP defines the runtime capability, not a specific implementation backend.
A web projection can implement it with the Web Serial API. A native projection
can implement it with OS serial APIs, Tauri plugins, or another native backend.
The napplet-facing contract stays the same.

This NAP does not define a device protocol. Text encoding, packet framing,
checksums, and command semantics belong to the napplet or to higher-level
protocols built on top of this byte channel.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `open` | `request` (`SerialOpenRequest`) | `SerialOpenResult` | `serial.open` / `serial.open.result` |
| `write` | `sessionId` (`tstr`), `data` (`bstr`) | none | `serial.write` / `serial.write.result` |
| `close` | `sessionId` (`tstr`), optional `reason` (`tstr`) | none | `serial.close` / `serial.close.result` |
| `onEvent` | handler for `SerialEvent` | `Subscription` handle | `serial.event` |

### Schemas

```cddl
SerialOpenRequest = {
  ? filters: [* SerialPortFilter],
  options: SerialOpenOptions,
  ? label: tstr,
}

SerialPortFilter = {
  ? usbVendorId: uint,
  ? usbProductId: uint,
  ? bluetoothServiceClassId: tstr / uint,
}

SerialOpenOptions = {
  baudRate: uint,
  ? dataBits: 7 / 8,
  ? stopBits: 1 / 2,
  ? parity: "none" / "even" / "odd",
  ? bufferSize: uint,
  ? flowControl: "none" / "hardware",
}

SerialOpenResult = {
  session: SerialSession,
}

SerialState = "opening" / "open" / "closed"

SerialSession = {
  id: tstr,
  state: SerialState,
  ? info: SerialPortInfo,
}

SerialPortInfo = {
  ? usbVendorId: uint,
  ? usbProductId: uint,
  ? bluetoothServiceClassId: tstr / uint,
  ? displayName: tstr,
}

SerialEvent = SerialStateEvent / SerialDataEvent / SerialClosedEvent

SerialStateEvent = {
  type: "state",
  sessionId: tstr,
  state: SerialState,
}

SerialDataEvent = {
  type: "data",
  sessionId: tstr,
  data: bstr,
}

SerialClosedEvent = {
  type: "closed",
  sessionId: tstr,
  ? reason: tstr,
}
```

**`open(request)`** asks the runtime to select and open a serial port. The
runtime may use `filters`, `label`, and napplet policy to explain or narrow the
chooser. It returns a runtime-assigned session id after the port is open.

**`write(sessionId, data)`** writes bytes to an open session. The runtime
preserves write order per session.

**`close(sessionId, reason)`** closes a session. The runtime may also close
sessions when a device disconnects, policy changes, or the napplet unloads.

**`onEvent(handler)`** subscribes to runtime-pushed session state, data, and
close events. Device data is delivered as byte values in the range `0..255`.

## Projection Backends

NAP-SERIAL is portable across projections:

- Web runtimes MAY implement `serial` with the Web Serial API.
- Native runtimes MAY implement `serial` with OS serial APIs, Tauri plugins, or
  another native serial backend.
- Runtimes MUST NOT expose backend-specific handles to napplets.
- Projection-specific chooser UX, remembered devices, device paths, permission
  prompts, and stream objects stay runtime-owned.

## Wire Protocol

`serial.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `serial.open` | napplet -> runtime | `id`, `request` |
| `serial.open.result` | runtime -> napplet | `id`, `session?`, `error?` |
| `serial.write` | napplet -> runtime | `id`, `sessionId`, `data` |
| `serial.write.result` | runtime -> napplet | `id`, `error?` |
| `serial.close` | napplet -> runtime | `id`, `sessionId`, `reason?` |
| `serial.close.result` | runtime -> napplet | `id`, `error?` |
| `serial.event` | runtime -> napplet | `event` |

Key design notes:
- Request/result pairs use `id` for correlation.
- `serial.event` has no `id`; it is runtime-pushed.
- `data` is a byte array. Values outside `0..255` are invalid.
- The runtime never exposes raw `SerialPort`, `ReadableStream`,
  `WritableStream`, `navigator.serial`, native handles, or OS device paths to
  the napplet.

### Examples

**Open a serial session:**

```
-> {
     "type": "serial.open",
     "id": "s1",
     "request": {
       "filters": [{ "usbVendorId": 9025 }],
       "options": { "baudRate": 115200 },
       "label": "controller"
     }
   }
<- {
     "type": "serial.open.result",
     "id": "s1",
     "session": {
       "id": "serial-1",
       "state": "open",
       "info": { "usbVendorId": 9025 }
     }
   }
```

**Receive and write bytes:**

```
<- {
     "type": "serial.event",
     "event": { "type": "data", "sessionId": "serial-1", "data": [79, 75, 10] }
   }
-> {
     "type": "serial.write",
     "id": "w1",
     "sessionId": "serial-1",
     "data": [112, 105, 110, 103, 10]
   }
<- { "type": "serial.write.result", "id": "w1" }
```

**Device closed:**

```
<- {
     "type": "serial.event",
     "event": {
       "type": "closed",
       "sessionId": "serial-1",
       "reason": "device disconnected"
     }
   }
```

### Error Handling

Result messages include an `error` string when the runtime cannot fulfill the
request. Common errors: `"unsupported"`, `"policy denied"`, `"user cancelled"`,
`"permission denied"`, `"port unavailable"`, `"open failed"`,
`"session not found"`, `"invalid data"`, `"write failed"`, and
`"device disconnected"`.

When `serial.open.result` includes `error`, it MUST NOT include `session`.

## Runtime Behavior

- The runtime MUST own all calls to Web Serial, native serial APIs, or any other
  serial backend.
- The runtime MUST respond to every request with a result message carrying the
  same `id`.
- The runtime MUST obtain user approval and satisfy backend activation or
  permission requirements before opening a port.
- The runtime MUST scope sessions to the requesting napplet and close them when
  that napplet unloads or loses permission.
- The runtime MUST preserve write order per session.
- The runtime MUST deliver readable bytes through `serial.event`.
- The runtime MUST NOT expose raw serial objects, browser stream objects, native
  handles, or OS device paths to napplets.
- The runtime MAY remember user-approved ports, enforce per-napplet allowlists,
  limit payload sizes, limit concurrent sessions, or redact `info` fields.

## Security Considerations

Serial devices may control hardware, leak local data, or expose identifying
device information. The runtime is the policy boundary.

- Napplets MUST NOT receive raw serial permissions, raw port objects, native
  handles, or OS device paths.
- The runtime MUST require explicit user or policy approval before opening a
  serial session.
- The runtime SHOULD show which napplet is requesting access and why, using the
  optional `label` when helpful.
- The runtime SHOULD treat incoming bytes as untrusted input and SHOULD apply
  reasonable rate, size, and lifecycle limits.
- The runtime SHOULD expose only the minimum device metadata needed by the
  napplet.

## Implementations

- None yet.
