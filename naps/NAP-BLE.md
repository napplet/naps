# NAP-BLE

## Runtime-Mediated Bluetooth Low Energy

`draft`

**NAP ID:** NAP-BLE
**Domain:** `ble`
**Web binding (NIP-5D):** `window.napplet.ble` · `shell.supports("ble")`

## Description

NAP-BLE lets a napplet communicate with user-approved Bluetooth Low Energy
devices through GATT. The runtime owns device discovery, chooser UI, permission,
GATT connection lifecycle, notification setup, disconnect handling, and backend
policy. The napplet gets an opaque session and can list services, read, write,
and subscribe to GATT attributes by UUID.

This NAP defines the runtime capability, not a backend. A web projection can
implement it with the Web Bluetooth API. A native projection can implement it
with Core Bluetooth, BlueZ, Android Bluetooth APIs, or another native BLE stack.
The napplet-facing contract stays the same.

This NAP does not define a device protocol. Byte layout, framing, checksums, and
command semantics belong to the napplet or to higher-level protocols.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `open` | `request` (`BleOpenRequest`) | `BleOpenResult` | `ble.open` / `ble.open.result` |
| `services` | `sessionId` (`tstr`) | list of `BleService` | `ble.services` / `ble.services.result` |
| `read` | `sessionId` (`tstr`), `target` (`BleAttribute`) | `bstr` | `ble.read` / `ble.read.result` |
| `write` | `sessionId` (`tstr`), `target` (`BleAttribute`), `data` (`bstr`), optional `opts` (`BleWriteOptions`) | none | `ble.write` / `ble.write.result` |
| `subscribe` | `sessionId` (`tstr`), `target` (`BleAttribute`) | none | `ble.subscribe` / `ble.subscribe.result` |
| `unsubscribe` | `sessionId` (`tstr`), `target` (`BleAttribute`) | none | `ble.unsubscribe` / `ble.unsubscribe.result` |
| `close` | `sessionId` (`tstr`), optional `reason` (`tstr`) | none | `ble.close` / `ble.close.result` |
| `onEvent` | handler for `BleEvent` | `Subscription` handle | `ble.event` |

### Schemas

```cddl
BleUuid = tstr / uint
BleSessionState = "opening" / "open" / "closed"

BleOpenRequest = {
  ? filters: [* BleDeviceFilter],
  ? exclusionFilters: [* BleDeviceFilter],
  ? acceptAllDevices: bool,
  ? optionalServices: [* BleUuid],
  ? label: tstr,
}

BleDeviceFilter = {
  ? services: [* BleUuid],
  ? name: tstr,
  ? namePrefix: tstr,
  ? manufacturerData: [* BleManufacturerDataFilter],
  ? serviceData: [* BleServiceDataFilter],
}

BleManufacturerDataFilter = {
  companyIdentifier: uint,
  ? dataPrefix: bstr,
  ? mask: bstr,
}

BleServiceDataFilter = {
  service: BleUuid,
  ? dataPrefix: bstr,
  ? mask: bstr,
}

BleOpenResult = {
  session: BleSession,
}

BleSession = {
  id: tstr,
  state: BleSessionState,
  device: BleDeviceInfo,
}

BleDeviceInfo = {
  id: tstr, ; runtime-scoped, opaque
  ? name: tstr,
  ? services: [* tstr],
}

BleService = {
  uuid: tstr,
  characteristics: [* BleCharacteristic],
}

BleCharacteristic = {
  uuid: tstr,
  properties: BleCharacteristicProperties,
}

BleCharacteristicProperties = {
  ? read: bool,
  ? write: bool,
  ? writeWithoutResponse: bool,
  ? notify: bool,
  ? indicate: bool,
}

BleAttribute = {
  service: BleUuid,
  characteristic: BleUuid,
  ? descriptor: BleUuid,
}

BleWriteOptions = {
  ? response: "with-response" / "without-response" / "auto",
}

BleEvent = BleStateEvent / BleNotificationEvent / BleClosedEvent

BleStateEvent = {
  type: "state",
  sessionId: tstr,
  state: BleSessionState,
}

BleNotificationEvent = {
  type: "notification",
  sessionId: tstr,
  target: BleAttribute,
  data: bstr,
}

BleClosedEvent = {
  type: "closed",
  sessionId: tstr,
  ? reason: tstr,
}
```

`BleUuid` values in requests may be Bluetooth assigned names, 16-bit or 32-bit
numbers, or canonical UUID strings. Runtime results MUST use canonical lowercase
UUID strings. Byte arrays are arrays of integers in the range `0..255`.

**`open(request)`** asks the runtime to select, authorize, and connect to a BLE
device. `filters`, `exclusionFilters`, `acceptAllDevices`, and
`optionalServices` match the Web Bluetooth device-selection model. A request
MUST provide at least one filter or set `acceptAllDevices: true`. When
`acceptAllDevices` is true, `filters` and `exclusionFilters` MUST be omitted.

**`services(sessionId)`** returns the GATT services and characteristics the
runtime exposes for the session. The runtime MAY redact services or
characteristics outside the grant.

**`read(sessionId, target)`** reads a characteristic or descriptor value.

**`write(sessionId, target, data, opts?)`** writes bytes to a characteristic or
descriptor. `response` requests write-with-response, write-without-response, or
lets the runtime choose from characteristic properties.

**`subscribe(sessionId, target)`** starts characteristic notifications or
indications. `target.descriptor` MUST be omitted.

**`unsubscribe(sessionId, target)`** stops notifications or indications for a
characteristic.

**`close(sessionId, reason?)`** disconnects the session. The runtime may also
close sessions when the device disconnects, policy changes, or the napplet
unloads.

**`onEvent(handler)`** subscribes to runtime-pushed session state, notification,
and close events.

## Projection Backends

NAP-BLE is portable across projections:

- Web runtimes MAY implement `ble` with the Web Bluetooth API.
- Native runtimes MAY implement `ble` with Core Bluetooth, BlueZ, Android
  Bluetooth APIs, platform plugins, or another native BLE backend.
- Runtimes MUST NOT expose backend-specific handles to napplets.
- Projection-specific chooser UX, remembered devices, adapter selection,
  permission prompts, and OS device identifiers stay runtime-owned.

## Wire Protocol

`ble.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `ble.open` | napplet -> runtime | `id`, `request` |
| `ble.open.result` | runtime -> napplet | `id`, `session?`, `error?` |
| `ble.services` | napplet -> runtime | `id`, `sessionId` |
| `ble.services.result` | runtime -> napplet | `id`, `services?`, `error?` |
| `ble.read` | napplet -> runtime | `id`, `sessionId`, `target` |
| `ble.read.result` | runtime -> napplet | `id`, `data?`, `error?` |
| `ble.write` | napplet -> runtime | `id`, `sessionId`, `target`, `data`, `options?` |
| `ble.write.result` | runtime -> napplet | `id`, `error?` |
| `ble.subscribe` | napplet -> runtime | `id`, `sessionId`, `target` |
| `ble.subscribe.result` | runtime -> napplet | `id`, `error?` |
| `ble.unsubscribe` | napplet -> runtime | `id`, `sessionId`, `target` |
| `ble.unsubscribe.result` | runtime -> napplet | `id`, `error?` |
| `ble.close` | napplet -> runtime | `id`, `sessionId`, `reason?` |
| `ble.close.result` | runtime -> napplet | `id`, `error?` |
| `ble.event` | runtime -> napplet | `event` |

Key design notes:
- Request/result pairs use `id` for correlation.
- `ble.event` has no `id`; it is runtime-pushed.
- `data`, `dataPrefix`, and `mask` are byte arrays. Values outside `0..255` are
  invalid.
- The runtime never exposes raw `BluetoothDevice`, GATT objects,
  `navigator.bluetooth`, native handles, adapter addresses, or OS device paths
  to the napplet.

### Examples

**Open a BLE session:**

```
-> {
     "type": "ble.open",
     "id": "b1",
     "request": {
       "filters": [{ "services": ["heart_rate"] }],
       "optionalServices": ["battery_service"],
       "label": "heart monitor"
     }
   }
<- {
     "type": "ble.open.result",
     "id": "b1",
     "session": {
       "id": "ble-1",
       "state": "open",
       "device": { "id": "dev-1", "name": "HR Monitor" }
     }
   }
```

**Read and subscribe:**

```
-> {
     "type": "ble.read",
     "id": "r1",
     "sessionId": "ble-1",
     "target": { "service": "battery_service", "characteristic": "battery_level" }
   }
<- { "type": "ble.read.result", "id": "r1", "data": [97] }

-> {
     "type": "ble.subscribe",
     "id": "n1",
     "sessionId": "ble-1",
     "target": { "service": "heart_rate", "characteristic": "heart_rate_measurement" }
   }
<- { "type": "ble.subscribe.result", "id": "n1" }
<- {
     "type": "ble.event",
     "event": {
       "type": "notification",
       "sessionId": "ble-1",
       "target": { "service": "heart_rate", "characteristic": "heart_rate_measurement" },
       "data": [6, 72]
     }
   }
```

### Error Handling

Result messages include an `error` string when the runtime cannot fulfill the
request. Common errors: `"unsupported"`, `"policy denied"`, `"user cancelled"`,
`"permission denied"`, `"device unavailable"`, `"open failed"`,
`"session not found"`, `"service not found"`, `"characteristic not found"`,
`"descriptor not found"`, `"operation not supported"`, `"invalid data"`,
`"read failed"`, `"write failed"`, and `"device disconnected"`.

When `ble.open.result` includes `error`, it MUST NOT include `session`.

## Runtime Behavior

- The runtime MUST own all calls to Web Bluetooth, native BLE APIs, or any other
  Bluetooth backend.
- The runtime MUST respond to every request with a result message carrying the
  same `id`.
- The runtime MUST obtain user approval and satisfy backend activation or
  permission requirements before opening a session.
- The runtime MUST scope sessions to the requesting napplet and close them when
  that napplet unloads or loses permission.
- The runtime MUST NOT allow access to services, characteristics, or descriptors
  outside the granted device scope.
- The runtime MUST deliver notifications through `ble.event`.
- The runtime MUST NOT expose raw Bluetooth objects, native handles, adapter
  addresses, stable device addresses, or OS device paths to napplets.
- The runtime MAY remember user-approved devices, enforce per-napplet allowlists,
  limit payload sizes, limit concurrent sessions, redact device metadata, or
  require a user gesture for `open()`.

## Security Considerations

BLE devices may control hardware, leak local data, or expose stable identifiers.
The runtime is the policy boundary.

- Napplets MUST NOT receive raw Bluetooth permissions, raw Bluetooth objects,
  native handles, adapter addresses, stable device addresses, or OS device paths.
- The runtime MUST require explicit user or policy approval before opening a BLE
  session.
- The runtime SHOULD show which napplet is requesting access and why, using the
  optional `label` when helpful.
- The runtime SHOULD require narrow filters unless policy permits
  `acceptAllDevices`.
- The runtime SHOULD expose only the minimum device metadata needed by the
  napplet.
- The runtime SHOULD treat incoming bytes as untrusted input and SHOULD apply
  reasonable rate, size, and lifecycle limits.

## References

- Web Bluetooth Community Group specification:
  <https://webbluetoothcg.github.io/web-bluetooth/>
- MDN `Bluetooth.requestDevice()` reference:
  <https://developer.mozilla.org/en-US/docs/Web/API/Bluetooth/requestDevice>

## Implementations

- None yet.
