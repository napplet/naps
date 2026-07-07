# NAP-BLE

## Runtime-Mediated Bluetooth Low Energy

`draft`

**NAP ID:** NAP-BLE
**Domain:** `ble`
**Web binding (NIP-5D):** `window.napplet.ble` · `shell.supports("ble")`

## Description

NAP-BLE lets a napplet communicate with user-approved Bluetooth Low Energy
devices through GATT. The runtime owns discovery, chooser UI, permission,
connection lifecycle, notification setup, disconnect handling, and backend
policy. Napplets receive only opaque sessions and GATT attribute operations.

The contract is projection-neutral. Web runtimes MAY use Web Bluetooth; native
runtimes MAY use Core Bluetooth, BlueZ, Android Bluetooth APIs, or another BLE
backend. Backend handles never cross the seam.

NAP-BLE does not define device protocols. Byte layout, framing, checksums, and
commands belong to the napplet or a higher-level protocol.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `open` | `request` (`BleOpenRequest`) | `BleOpenResult` | `ble.open` / `.result` |
| `services` | `sessionId` | list of `BleService` | `ble.services` / `.result` |
| `read` | `sessionId`, `target` | `bstr` | `ble.read` / `.result` |
| `write` | `sessionId`, `target`, `data`, `opts?` | none | `ble.write` / `.result` |
| `subscribe` | `sessionId`, `target` | none | `ble.subscribe` / `.result` |
| `unsubscribe` | `sessionId`, `target` | none | `ble.unsubscribe` / `.result` |
| `close` | `sessionId`, `reason?` | none | `ble.close` / `.result` |
| `onEvent` | handler for `BleEvent` | `Subscription` | `ble.event` |

### Schemas

Primitive aliases and enums:

| Name | Definition |
|------|------------|
| `BleUuid` | text or unsigned integer UUID reference |
| `BleSessionState` | `opening`, `open`, `closed` |
| `BleWriteResponse` | `with-response`, `without-response`, `auto` |

`BleOpenRequest` fields:

| Field | Required | Type |
|-------|----------|------|
| `filters` | no | list of `BleDeviceFilter` |
| `exclusionFilters` | no | list of `BleDeviceFilter` |
| `acceptAllDevices` | no | boolean |
| `optionalServices` | no | list of `BleUuid` |
| `label` | no | text |

`BleDeviceFilter` fields:

| Field | Required | Type |
|-------|----------|------|
| `services` | no | list of `BleUuid` |
| `name` | no | text |
| `namePrefix` | no | text |
| `manufacturerData` | no | list of `BleManufacturerDataFilter` |
| `serviceData` | no | list of `BleServiceDataFilter` |

`BleManufacturerDataFilter` fields:

| Field | Required | Type |
|-------|----------|------|
| `companyIdentifier` | yes | unsigned integer |
| `dataPrefix` | no | bytes |
| `mask` | no | bytes |

`BleServiceDataFilter` fields:

| Field | Required | Type |
|-------|----------|------|
| `service` | yes | `BleUuid` |
| `dataPrefix` | no | bytes |
| `mask` | no | bytes |

`BleOpenResult` fields:

| Field | Required | Type |
|-------|----------|------|
| `session` | yes | `BleSession` |

`BleSession` fields:

| Field | Required | Type |
|-------|----------|------|
| `id` | yes | text |
| `state` | yes | `BleSessionState` |
| `device` | yes | `BleDeviceInfo` |

`BleDeviceInfo` fields:

| Field | Required | Type |
|-------|----------|------|
| `id` | yes | runtime-scoped opaque text |
| `name` | no | text |
| `services` | no | list of text |

`BleService` fields:

| Field | Required | Type |
|-------|----------|------|
| `uuid` | yes | text |
| `characteristics` | yes | list of `BleCharacteristic` |

`BleCharacteristic` fields:

| Field | Required | Type |
|-------|----------|------|
| `uuid` | yes | text |
| `properties` | yes | `BleCharacteristicProperties` |

`BleCharacteristicProperties` fields:

| Field | Required | Type |
|-------|----------|------|
| `read` | no | boolean |
| `write` | no | boolean |
| `writeWithoutResponse` | no | boolean |
| `notify` | no | boolean |
| `indicate` | no | boolean |

`BleAttribute` fields:

| Field | Required | Type |
|-------|----------|------|
| `service` | yes | `BleUuid` |
| `characteristic` | yes | `BleUuid` |
| `descriptor` | no | `BleUuid` |

`BleWriteOptions` fields:

| Field | Required | Type |
|-------|----------|------|
| `response` | no | `BleWriteResponse` |

`BleEvent` variants:

| Variant | Required fields | Optional fields |
|---------|-----------------|-----------------|
| state | `type: "state"`, `sessionId: tstr`, `state: BleSessionState` | none |
| notification | `type: "notification"`, `sessionId: tstr`, `target: BleAttribute`, `data: bstr` | none |
| closed | `type: "closed"`, `sessionId: tstr` | `reason: tstr` |

`BleUuid` requests MAY use assigned names, 16-bit or 32-bit numbers, or
canonical UUID strings. Results MUST use canonical lowercase UUID strings.
Byte arrays are integers in `0..255`.

## Operation Rules

| Operation | Rules |
|-----------|-------|
| `open` | Runtime selects, authorizes, and connects. Request MUST include at least one filter or `acceptAllDevices: true`. `acceptAllDevices` excludes `filters` and `exclusionFilters`. |
| `services` | Returns granted services and characteristics. Runtime MAY redact outside the grant. |
| `read` | Reads a characteristic or descriptor value. |
| `write` | Writes bytes to a characteristic or descriptor. `response` selects with-response, without-response, or runtime auto. |
| `subscribe` | Starts characteristic notifications or indications. `target.descriptor` MUST be omitted. |
| `unsubscribe` | Stops characteristic notifications or indications. |
| `close` | Disconnects the session. Runtime MAY also close on device disconnect, policy change, or napplet unload. |
| `onEvent` | Receives runtime-pushed state, notification, and close events. |

## Projection Backends

- Web runtimes MAY implement `ble` with Web Bluetooth.
- Native runtimes MAY implement `ble` with Core Bluetooth, BlueZ, Android
  Bluetooth APIs, platform plugins, or another native BLE backend.
- Runtimes MUST NOT expose backend-specific handles to napplets.
- Chooser UX, remembered devices, adapter selection, permission prompts, and OS
  device identifiers stay runtime-owned.

## Wire Protocol

`ble.*` messages use NIP-5D wire format: `{ "type": "domain.action", ...payload }`.

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

`ble.event` has no `id`; it is runtime-pushed. The runtime MUST NOT expose raw
`BluetoothDevice`, GATT objects, `navigator.bluetooth`, native handles, adapter
addresses, stable device addresses, or OS device paths.

## Error Handling

Result messages include `error` when the runtime cannot fulfill the request.
Common errors: `"unsupported"`, `"policy denied"`, `"user cancelled"`,
`"permission denied"`, `"device unavailable"`, `"open failed"`,
`"session not found"`, `"service not found"`, `"characteristic not found"`,
`"descriptor not found"`, `"operation not supported"`, `"invalid data"`,
`"read failed"`, `"write failed"`, `"device disconnected"`.

`ble.open.result` MUST NOT include `session` when it includes `error`.

## Runtime Behavior

- MUST own all Bluetooth backend calls.
- MUST answer every request with the same `id`.
- MUST obtain user approval and satisfy backend activation or permission gates
  before opening a session.
- MUST scope sessions to the requesting napplet and close them on unload or
  permission loss.
- MUST NOT allow access outside the granted device scope.
- MUST deliver notifications through `ble.event`.
- MUST NOT expose raw Bluetooth objects, native handles, stable device
  addresses, adapter addresses, or OS device paths.
- MAY remember devices, enforce allowlists, limit payload size, limit concurrent
  sessions, redact metadata, or require a user gesture for `open()`.

## Security Considerations

BLE devices may control hardware, leak local data, or expose stable identifiers.
The runtime is the policy boundary.

- Require explicit user or policy approval before opening a session.
- Show the requesting napplet and optional `label` when helpful.
- Prefer narrow filters unless policy permits `acceptAllDevices`.
- Expose minimum device metadata.
- Treat incoming bytes as untrusted and apply rate, size, and lifecycle limits.

## References

- Web Bluetooth Community Group specification:
  <https://webbluetoothcg.github.io/web-bluetooth/>
- MDN `Bluetooth.requestDevice()` reference:
  <https://developer.mozilla.org/en-US/docs/Web/API/Bluetooth/requestDevice>

## Implementations

- None yet.

## Changelog

- `5f44ca1` - Introduced NAP-BLE as a portable BLE GATT access surface.
