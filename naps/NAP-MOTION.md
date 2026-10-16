NAP-MOTION
==========

Shell-Mediated Device Motion and Orientation
--------------------------------------------

`draft`

**NAP ID:** NAP-MOTION
**Domain:** `motion`
**Depends:** none.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.motion`; domain presence signals availability.

## Description

NAP-MOTION gives napplets device motion and orientation through one
shell-mediated contract that behaves the same in every shell. Accelerometer, gyroscope and magnetometer readings belong to the
device, not to one napplet. A napplet that reads them natively can infer what
the user types or taps in another napplet, fingerprint the device and, from
compass heading, narrow the user's location. Under this NAP the runtime reads
the sensors and, where the host allows, denies napplets native access. The
shell asks the user before a napplet receives
motion, then delivers live samples only to a granted napplet that has input
focus. Every subscribed napplet still sees availability, permission and focus
status.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `requestPermission` | `absolute` (boolean, optional) | `MotionPermissionResult` | `motion.permission` / `motion.permission.result` |
| `state` | none | `MotionState` | latest `motion.state` |
| `latest` | none | `MotionSample` or `null` | latest `motion.sample` |
| `onState` | handler for `MotionState` | `Subscription` handle | `motion.state` |
| `onSample` | handler for `MotionSample` | `Subscription` handle | `motion.sample` |

### Schemas

`MotionPermission` accepts `"prompt"`, `"granted"` or `"denied"`.

`MotionUnavailableReason` accepts `"blocked"` or `"unavailable"`.

`MotionState` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `available` | yes | boolean | `false` when the runtime cannot read motion sensors. |
| `reason` | no | `MotionUnavailableReason` | Present only when `available` is `false`. `"blocked"` means host policy denied the runtime; `"unavailable"` means the host has no motion sensors or API. |
| `permission` | yes | `MotionPermission` | This napplet's grant. `"prompt"` means the user has not decided. |
| `absolute` | yes | boolean | `true` when this napplet holds an absolute orientation grant and the host can supply it. |
| `focused` | yes | boolean | `true` when this napplet currently receives samples. |
| `frequency` | yes | number | Samples per second the shell delivers while `focused` is `true`. `0` when no samples flow. |

`MotionPermissionResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `permission` | yes | `MotionPermission` | Outcome of the request. Never `"prompt"` unless `error` is present. |
| `absolute` | yes | boolean | `true` when absolute orientation is granted. |
| `error` | no | text | Present when the request could not be processed. |

`Vector3` fields, in metres per second squared:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `x` | yes | number or `null` | Device x axis. `null` when the host cannot measure it. |
| `y` | yes | number or `null` | Device y axis. |
| `z` | yes | number or `null` | Device z axis. |

`RotationRate` fields, in degrees per second:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `alpha` | yes | number or `null` | Rotation around the z axis. |
| `beta` | yes | number or `null` | Rotation around the x axis. |
| `gamma` | yes | number or `null` | Rotation around the y axis. |

`Orientation` fields, in degrees:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `alpha` | yes | number or `null` | `0` to `360`. Around the z axis. |
| `beta` | yes | number or `null` | `-180` to `180`. Around the x axis. |
| `gamma` | yes | number or `null` | `-90` to `90`. Around the y axis. |
| `absolute` | yes | boolean | `true` when the frame is Earth-referenced (compass north). `false` when the frame is arbitrary. |

`MotionSample` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `timestamp` | yes | number | Runtime time of the reading, in milliseconds. |
| `interval` | yes | number | Milliseconds between delivered samples. |
| `acceleration` | yes | `Vector3` or `null` | Linear acceleration without gravity. |
| `accelerationIncludingGravity` | yes | `Vector3` or `null` | Acceleration including gravity. |
| `rotationRate` | yes | `RotationRate` or `null` | Gyroscope reading. |
| `orientation` | yes | `Orientation` or `null` | Absolute only when `MotionState.absolute` is `true`. |

Axes, units and ranges follow
[W3C DeviceOrientation Event](https://www.w3.org/TR/orientation-event/).

**`requestPermission(absolute)`** asks the shell for a grant. With `absolute`
`true` it also asks for absolute orientation. It resolves with the outcome. It
returns an existing decision without prompting.

**`state()`** returns the latest `MotionState`. Before the first snapshot it
returns `available` `true`, `permission` `"prompt"`, `absolute` `false`,
`focused` `false` and `frequency` `0`. It never waits for the wire.

**`latest()`** returns the latest `MotionSample`, or `null` when none has
arrived or `focused` is `false`.

**`onState(handler)`** and **`onSample(handler)`** call `handler` with each new
`MotionState` or `MotionSample` and return a `Subscription` handle. Closing the
handle removes the local handler.

The runtime-provided binding sends `motion.subscribe` before or on first use.
It MAY subscribe when the domain is installed. It MAY send `motion.unsubscribe`
when the napplet no longer needs motion.

## Wire Protocol

`motion.*` messages use the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `motion.subscribe` | napplet -> shell | `frequency`? |
| `motion.unsubscribe` | napplet -> shell | none |
| `motion.permission` | napplet -> shell | `id`, `absolute`? |
| `motion.permission.result` | shell -> napplet | `id`, `permission`, `absolute`, `error`? |
| `motion.state` | shell -> napplet | `available`, `reason`?, `permission`, `absolute`, `focused`, `frequency` |
| `motion.sample` | shell -> napplet | `timestamp`, `interval`, `acceleration`, `accelerationIncludingGravity`, `rotationRate`, `orientation` |

Key design notes:
- `motion.permission` is the only request/result pair. It correlates by `id`.
- `motion.subscribe`, `motion.unsubscribe`, `motion.state` and `motion.sample`
  are fire-and-forget.
- `frequency` on `motion.subscribe` is the requested samples per second. The
  shell treats it as an upper bound and reports the delivered rate in
  `motion.state`.
- `motion.state` is a complete snapshot, sent right after `motion.subscribe`
  and then whenever the napplet's view changes.
- A napplet without a grant or without focus receives no `motion.sample`.
  Unlike NAP-GAMEPAD's neutral snapshots, the shell sends no zero samples: zero
  motion is a reading, and sending it would fabricate data.

### Examples

**Subscribe before any grant:**
```
-> { "type": "motion.subscribe", "frequency": 60 }
<- { "type": "motion.state", "available": true, "permission": "prompt",
     "absolute": false, "focused": false, "frequency": 0 }
```

**Request a grant with absolute orientation; the user allows relative only:**
```
-> { "type": "motion.permission", "id": "m1", "absolute": true }
<- { "type": "motion.permission.result", "id": "m1", "permission": "granted", "absolute": false }
<- { "type": "motion.state", "available": true, "permission": "granted",
     "absolute": false, "focused": true, "frequency": 60 }
<- { "type": "motion.sample", "timestamp": 18235.4, "interval": 16.7,
     "acceleration": { "x": 0.1, "y": -0.2, "z": 0.0 },
     "accelerationIncludingGravity": { "x": 0.1, "y": -0.2, "z": 9.8 },
     "rotationRate": { "alpha": 1.5, "beta": -0.3, "gamma": 0.0 },
     "orientation": { "alpha": 112.4, "beta": 3.1, "gamma": -1.2, "absolute": false } }
```

**Focus moves to another napplet; samples stop:**
```
<- { "type": "motion.state", "available": true, "permission": "granted",
     "absolute": false, "focused": false, "frequency": 0 }
```

**The user denies:**
```
-> { "type": "motion.permission", "id": "m2" }
<- { "type": "motion.permission.result", "id": "m2", "permission": "denied", "absolute": false }
```

**Host policy denies sensor access to the runtime:**
```
<- { "type": "motion.state", "available": false, "reason": "blocked",
     "permission": "prompt", "absolute": false, "focused": false, "frequency": 0 }
```

**Stop receiving motion:**
```
-> { "type": "motion.unsubscribe" }
```

### Error Handling

`motion.permission.result` carries `error` when the shell cannot process the
request, for example a malformed payload or `available` `false`; `permission`
then reports the current grant. Unavailability is reported in state through
`available` and `reason`. The shell silently ignores malformed or excessive
subscription messages.

## Shell Behavior

- A shell that offers `motion` MUST implement this whole contract, whether or
  not its host can deny native sensors to napplets. Presence of the domain
  guarantees the operations, schemas and gating in this spec. It does not
  guarantee isolation from native APIs.
- Where the host can deny native motion and orientation APIs to napplets, the
  runtime MUST deny them, and MUST be the only reader of motion sensors on
  their behalf.
- A shell MAY offer `motion` on a host that cannot deny native access. It
  SHOULD then tell the user that napplets can read sensors outside its grants.
- The shell MUST respond to every `motion.permission` with a
  `motion.permission.result` carrying the same `id`.
- The shell MUST NOT prompt the user except in response to `motion.permission`.
  A prompt MUST identify the napplet by its verified identity.
- Absolute orientation MUST require a grant distinct from relative motion. The
  user MAY grant relative motion and refuse absolute orientation.
- The shell MUST key grants to the verified napplet identity. It MAY persist
  them under its own policy and MUST let the user revoke them. On revocation
  the shell MUST send a new `motion.state` and stop samples.
- After a denial the shell SHOULD NOT prompt again until the user changes the
  decision. It SHOULD rate-limit `motion.permission`.
- The shell MUST send `motion.sample` only to a napplet that holds a grant,
  holds input focus, and runs while the host surface is visible and focused.
- When any of those conditions ends, the shell MUST stop samples at once and
  send `motion.state` with `focused` `false` and `frequency` `0`.
- The shell MUST send `orientation` with `absolute` `true` only to a napplet
  holding an absolute grant. Otherwise it MUST send relative orientation or
  `null`.
- The shell MUST cap the sample rate and SHOULD NOT exceed 60 samples per
  second. It MUST report the delivered rate in `frequency`.
- The shell SHOULD quantize values to no finer than 0.1 m/s², 0.1 deg/s and
  0.1 degrees, as W3C DeviceOrientation recommends. It SHOULD coarsen absolute
  `alpha` further.
- The shell MUST send a `motion.state` snapshot after each `motion.subscribe`,
  then whenever that napplet's view changes. It SHOULD NOT resend an unchanged
  snapshot.
- The shell MUST report `available` `false` with a `reason` when the runtime
  itself cannot read motion sensors.
- The shell MUST stop sending `motion.state` and `motion.sample` after
  `motion.unsubscribe` and when the napplet's session ends.
- The shell SHOULD stop reading sensors when no napplet is eligible for samples.
- The shell MAY enforce ACL checks on `motion` capabilities.

## Napplet Guidance

- A napplet SHOULD call `requestPermission` from a user action, and SHOULD
  explain why it needs motion before calling it.
- A napplet SHOULD treat `focused` `false` as a pause. It SHOULD NOT integrate
  acceleration or rotation across a gap in samples.
- A napplet SHOULD request `absolute` only when it needs compass heading.
- A napplet SHOULD keep pointer and keyboard controls.
- A napplet SHOULD use `motion` instead of native sensor APIs. Native access is
  denied in some shells and open in others; `motion` behaves the same wherever
  it is present.
- The domain is optional. A napplet SHOULD NOT declare `motion` as required
  unless it cannot run without motion.

## Web Projection Guidance

This section is projection guidance only. The contract above does not change.

- The shell denies native sensors to napplet frames with the Permissions-Policy
  features `accelerometer`, `gyroscope` and `magnetometer`, for example
  `allow="accelerometer 'none'; gyroscope 'none'; magnetometer 'none'"` on each
  napplet `iframe`. CSP cannot govern sensors.
- Where the engine enforces these features, `devicemotion`,
  `deviceorientation` and `deviceorientationabsolute` stop firing in the frame
  without any error, and Generic Sensor classes (`Accelerometer`, `Gyroscope`,
  `AbsoluteOrientationSensor`, …) report an error on `start()`.
- Where the engine does not enforce these features, native events still reach
  the napplet's realm, and no binding replacement can block them. The shell
  still offers `motion` with the full contract; only isolation from native
  reads is lost. Engine support varies, so a shell SHOULD verify enforcement
  per engine instead of assuming it.
- A binding MAY then replace the event APIs with ones served from `motion.*`.
  Existing libraries then keep working unchanged. Such a replacement SHOULD:
  - replace `DeviceMotionEvent.requestPermission` and
    `DeviceOrientationEvent.requestPermission` with operations that send
    `motion.permission` and resolve to `"granted"` or `"denied"`, keeping the
    native property attributes, name, length and receiver check;
  - dispatch `devicemotion` and `deviceorientation` on `window` for each
    `motion.sample`, built with the native `DeviceMotionEvent` and
    `DeviceOrientationEvent` constructors so `on*` handlers and listeners both
    work;
  - dispatch `deviceorientationabsolute` only when `MotionState.absolute` is
    `true`;
  - never cause a prompt except through the replaced `requestPermission`; and
  - leave the native APIs untouched when they are not denied.
- Dispatched events report `isTrusted` `false`. Code that rejects untrusted
  events will not see them.
- Generic Sensor classes stay denied. The binding does not replace them.
- Hosts that require a user gesture for sensors, such as iOS Safari, accept the
  shell's own prompt as that gesture. The napplet's gesture does not need to
  cross the frame boundary.

## Security Considerations

- Motion data is a side channel on user input. Accelerometer and gyroscope
  readings can reveal taps, keystrokes and PINs entered elsewhere, and can pick
  up nearby speech. Focus gating, the user grant, the rate cap and quantization
  all bound this channel.
- Factory calibration error in motion sensors can identify a device across
  sites. Quantization SHOULD be coarse enough to mask it.
- Absolute orientation exposes compass heading, which aids location inference.
  It requires its own grant, and the shell SHOULD coarsen it.
- Isolation from native reads depends on the host. Where the runtime denies
  every native sensor path, this domain's gating is the only way a napplet reads
  motion. Where the host cannot deny them, the gating governs only `motion`
  itself, and native reads bypass it.
- This domain grants no device commands. Haptics, batched raw readings, ambient
  light, proximity and geolocation are out of scope.

## Implementations

- None yet.

## Changelog

- `e43dd4f` - Introduced NAP-MOTION for grant- and focus-scoped, shell-mediated device motion and orientation samples.
