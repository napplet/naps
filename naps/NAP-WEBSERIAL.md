NAP-WEBSERIAL
=============

Shell-Mediated Serial Device Access
-----------------------------------

`draft`

**NAP ID:** NAP-WEBSERIAL
**Domain:** `webserial`
**Web binding (NIP-5D):** `window.napplet.webserial` · `shell.supports("webserial")`

## Description

NAP-WEBSERIAL lets a sandboxed napplet communicate with user-approved serial
devices without receiving direct access to `navigator.serial`, `SerialPort`, or
browser streams. The shell owns device selection, permission prompts, port
opening, read loops, write ordering, disconnect handling, and policy. The
napplet gets a small byte-oriented session API.

This NAP does not define a device protocol. Text encoding, packet framing,
checksums, and command semantics belong to the napplet or to higher-level
protocols built on top of this byte channel.

## API Surface

```typescript
interface NappletWebserial {
  open(request: WebserialOpenRequest): Promise<WebserialOpenResult>;
  write(sessionId: string, data: Uint8Array | number[]): Promise<void>;
  close(sessionId: string, reason?: string): Promise<void>;
  onEvent(handler: (event: WebserialEvent) => void): Subscription;
}

interface WebserialOpenRequest {
  filters?: WebserialPortFilter[];
  options: WebserialOpenOptions;
  label?: string;
}

interface WebserialPortFilter {
  usbVendorId?: number;
  usbProductId?: number;
  bluetoothServiceClassId?: string | number;
}

interface WebserialOpenOptions {
  baudRate: number;
  dataBits?: 7 | 8;
  stopBits?: 1 | 2;
  parity?: "none" | "even" | "odd";
  bufferSize?: number;
  flowControl?: "none" | "hardware";
}

interface WebserialOpenResult {
  session: WebserialSession;
}

interface WebserialSession {
  id: string;
  state: "opening" | "open" | "closed";
  info?: WebserialPortInfo;
}

interface WebserialPortInfo {
  usbVendorId?: number;
  usbProductId?: number;
  bluetoothServiceClassId?: string | number;
}

type WebserialEvent =
  | { type: "state"; sessionId: string; state: WebserialSession["state"] }
  | { type: "data"; sessionId: string; data: number[] }
  | { type: "closed"; sessionId: string; reason?: string };

interface Subscription {
  close(): void;
}
```

**`open(request)`** asks the shell to select and open a serial port. The shell
may use `filters`, `label`, and napplet policy to explain or narrow the chooser.
It returns a shell-assigned session id after the port is open.

**`write(sessionId, data)`** writes bytes to an open session. The shell preserves
write order per session.

**`close(sessionId, reason)`** closes a session. The shell may also close
sessions when a device disconnects, policy changes, or the napplet unloads.

**`onEvent(handler)`** subscribes to shell-pushed session state, data, and close
events. Device data is delivered as byte values in the range `0..255`.

## Wire Protocol

`webserial.*` messages use the NIP-5D wire format
(`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `webserial.open` | napplet -> shell | `id`, `request` |
| `webserial.open.result` | shell -> napplet | `id`, `session?`, `error?` |
| `webserial.write` | napplet -> shell | `id`, `sessionId`, `data` |
| `webserial.write.result` | shell -> napplet | `id`, `error?` |
| `webserial.close` | napplet -> shell | `id`, `sessionId`, `reason?` |
| `webserial.close.result` | shell -> napplet | `id`, `error?` |
| `webserial.event` | shell -> napplet | `event` |

Key design notes:
- Request/result pairs use `id` for correlation.
- `webserial.event` has no `id`; it is shell-pushed.
- `data` is a byte array. Values outside `0..255` are invalid.
- The shell never exposes raw `SerialPort`, `ReadableStream`, `WritableStream`,
  `navigator.serial`, or OS device paths to the napplet.

### Examples

**Open a serial session:**

```
-> {
     "type": "webserial.open",
     "id": "s1",
     "request": {
       "filters": [{ "usbVendorId": 9025 }],
       "options": { "baudRate": 115200 },
       "label": "controller"
     }
   }
<- {
     "type": "webserial.open.result",
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
     "type": "webserial.event",
     "event": { "type": "data", "sessionId": "serial-1", "data": [79, 75, 10] }
   }
-> {
     "type": "webserial.write",
     "id": "w1",
     "sessionId": "serial-1",
     "data": [112, 105, 110, 103, 10]
   }
<- { "type": "webserial.write.result", "id": "w1" }
```

**Device closed:**

```
<- {
     "type": "webserial.event",
     "event": {
       "type": "closed",
       "sessionId": "serial-1",
       "reason": "device disconnected"
     }
   }
```

### Error Handling

Result messages include an `error` string when the shell cannot fulfill the
request. Common errors: `"unsupported"`, `"policy denied"`, `"user cancelled"`,
`"permission denied"`, `"port unavailable"`, `"open failed"`,
`"session not found"`, `"invalid data"`, `"write failed"`, and
`"device disconnected"`.

When `webserial.open.result` includes `error`, it MUST NOT include `session`.

## Shell Behavior

- The shell MUST own all calls to Web Serial or to any native serial backend.
- The shell MUST respond to every request with a result message carrying the
  same `id`.
- The shell MUST obtain user approval and satisfy browser activation
  requirements before opening a port.
- The shell MUST scope sessions to the requesting napplet and close them when
  that napplet unloads or loses permission.
- The shell MUST preserve write order per session.
- The shell MUST deliver readable bytes through `webserial.event`.
- The shell MUST NOT expose raw serial objects, browser stream objects, or OS
  device paths to napplets.
- The shell MAY remember user-approved ports, enforce per-napplet allowlists,
  limit payload sizes, limit concurrent sessions, or redact `info` fields.

## Security Considerations

Serial devices may control hardware, leak local data, or expose identifying
device information. The shell is the policy boundary.

- Napplets MUST NOT receive raw Web Serial permission or raw port objects.
- The shell MUST require explicit user or policy approval before opening a
  serial session.
- The shell SHOULD show which napplet is requesting access and why, using the
  optional `label` when helpful.
- The shell SHOULD treat incoming bytes as untrusted input and SHOULD apply
  reasonable rate, size, and lifecycle limits.
- The shell SHOULD expose only the minimum device metadata needed by the
  napplet.

## Implementations

- None yet.
