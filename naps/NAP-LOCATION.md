NAP-LOCATION
============

Shell-Mediated Geolocation
--------------------------

`draft`

**NAP ID:** NAP-LOCATION
**Domain:** `location`
**Depends:** none.
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.location`; domain presence signals availability.

## Description

NAP-LOCATION gives napplets the device position without direct geolocation
access. Position belongs to the user, not to one napplet: it reveals home,
work and routine. A napplet that reads geolocation natively bypasses every
per-napplet decision the shell could make. Under this NAP the runtime owns the
only native reader. The user grants each napplet coarse position by default.
Precise position and background delivery each need their own grant. The shell
delivers positions to a granted napplet while it is visible, or while hidden
when it holds a background grant.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `requestPermission` | `precise` (boolean, optional), `background` (boolean, optional) | `LocationPermissionResult` | `location.permission` / `location.permission.result` |
| `current` | `maximumAge` (number, optional), `timeout` (number, optional), `highAccuracy` (boolean, optional) | `LocationResult` | `location.current` / `location.current.result` |
| `state` | none | `LocationState` | latest `location.state` |
| `latest` | none | `Position` or `null` | latest `location.position` |
| `onState` | handler for `LocationState` | `Subscription` handle | `location.state` |
| `onPosition` | handler for `Position` | `Subscription` handle | `location.position` |

### Schemas

`LocationPermission` accepts `"prompt"`, `"granted"` or `"denied"`.

`LocationUnavailableReason` accepts `"blocked"` or `"unavailable"`.

`LocationError` accepts `"denied"`, `"unavailable"`, `"timeout"` or `"inactive"`.

`LocationState` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `available` | yes | boolean | `false` when the runtime cannot read position. |
| `reason` | no | `LocationUnavailableReason` | Present only when `available` is `false`. `"blocked"` means host policy denied the runtime; `"unavailable"` means the host has no positioning. |
| `permission` | yes | `LocationPermission` | This napplet's grant. `"prompt"` means the user has not decided. |
| `precise` | yes | boolean | `true` when this napplet holds a precise grant. |
| `background` | yes | boolean | `true` when this napplet holds a background grant. |
| `active` | yes | boolean | `true` when this napplet currently receives positions. |

`LocationPermissionResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `permission` | yes | `LocationPermission` | Outcome of the request. Never `"prompt"` unless `error` is present. |
| `precise` | yes | boolean | `true` when precise position is granted. |
| `background` | yes | boolean | `true` when background delivery is granted. |
| `error` | no | text | Present when the request could not be processed. |

`Position` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `latitude` | yes | number | WGS 84 degrees. |
| `longitude` | yes | number | WGS 84 degrees. |
| `accuracy` | yes | number | Radius in metres around the point that contains the true position. |
| `altitude` | yes | number or `null` | Metres above the WGS 84 ellipsoid. |
| `altitudeAccuracy` | yes | number or `null` | Metres. |
| `heading` | yes | number or `null` | Direction of travel, in degrees clockwise from true north. `null` when stationary or unknown. |
| `speed` | yes | number or `null` | Metres per second. |
| `timestamp` | yes | number | Runtime time of the fix, in milliseconds since the Unix epoch. |
| `precise` | yes | boolean | `false` when the shell coarsened this position. |

`LocationResult` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `position` | no | `Position` | Present on success. |
| `error` | no | `LocationError` | Present on failure. Exactly one of `position` and `error` is present. |

Fields, units and ranges follow
[W3C Geolocation](https://www.w3.org/TR/geolocation/). `heading` is the
direction of travel derived from position. It is not where the device points;
device orientation belongs to the `motion` domain.

**`requestPermission(precise, background)`** asks the shell for a grant. Coarse
position is always requested. `precise` and `background` request the extra
tiers. It resolves with the outcome. It returns an existing decision without
prompting.

**`current(maximumAge, timeout, highAccuracy)`** returns one position. The shell
MAY return a cached position no older than `maximumAge` milliseconds. It
resolves with `"timeout"` when no position arrives within `timeout`
milliseconds. It resolves with `"inactive"` when the napplet is hidden and holds
no background grant. It does not prompt.

**`state()`** returns the latest `LocationState`. Before the first snapshot it
returns `available` `true`, `permission` `"prompt"`, and `precise`,
`background` and `active` `false`. It never waits for the wire.

**`latest()`** returns the latest delivered `Position`, or `null` when none has
arrived.

**`onState(handler)`** and **`onPosition(handler)`** call `handler` with each new
`LocationState` or `Position` and return a `Subscription` handle. Closing the
handle removes the local handler.

The runtime-provided binding sends `location.subscribe` before or on first use
of `state`, `latest`, `onState` or `onPosition`. It MAY send
`location.unsubscribe` when the napplet no longer needs updates.

## Wire Protocol

`location.*` messages use the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `location.subscribe` | napplet -> shell | `highAccuracy`? |
| `location.unsubscribe` | napplet -> shell | none |
| `location.permission` | napplet -> shell | `id`, `precise`?, `background`? |
| `location.permission.result` | shell -> napplet | `id`, `permission`, `precise`, `background`, `error`? |
| `location.current` | napplet -> shell | `id`, `maximumAge`?, `timeout`?, `highAccuracy`? |
| `location.current.result` | shell -> napplet | `id`, `position`?, `error`? |
| `location.state` | shell -> napplet | `available`, `reason`?, `permission`, `precise`, `background`, `active` |
| `location.position` | shell -> napplet | `latitude`, `longitude`, `accuracy`, `altitude`, `altitudeAccuracy`, `heading`, `speed`, `timestamp`, `precise` |

Key design notes:
- `location.permission` and `location.current` are request/result pairs. They
  correlate by `id`.
- `location.subscribe`, `location.unsubscribe`, `location.state` and
  `location.position` are fire-and-forget.
- `highAccuracy` is a power hint. It asks the runtime to prefer a more accurate
  source. It never raises a coarse grant to precise.
- `location.state` is a complete snapshot, sent right after
  `location.subscribe` and then whenever the napplet's view changes.
- An inactive napplet receives no `location.position`. The shell never sends a
  placeholder position.

### Examples

**Subscribe before any grant:**
```
-> { "type": "location.subscribe" }
<- { "type": "location.state", "available": true, "permission": "prompt",
     "precise": false, "background": false, "active": false }
```

**Request precise position; the user allows coarse only:**
```
-> { "type": "location.permission", "id": "l1", "precise": true }
<- { "type": "location.permission.result", "id": "l1", "permission": "granted",
     "precise": false, "background": false }
<- { "type": "location.state", "available": true, "permission": "granted",
     "precise": false, "background": false, "active": true }
<- { "type": "location.position", "latitude": 52.53, "longitude": 13.40,
     "accuracy": 3500, "altitude": null, "altitudeAccuracy": null,
     "heading": null, "speed": null, "timestamp": 1792160400000, "precise": false }
```

**The napplet is hidden and holds no background grant:**
```
<- { "type": "location.state", "available": true, "permission": "granted",
     "precise": false, "background": false, "active": false }
```

**One position, without a subscription:**
```
-> { "type": "location.current", "id": "l2", "maximumAge": 60000, "timeout": 10000 }
<- { "type": "location.current.result", "id": "l2", "position": {
     "latitude": 52.53, "longitude": 13.40, "accuracy": 3500,
     "altitude": null, "altitudeAccuracy": null, "heading": null, "speed": null,
     "timestamp": 1792160412000, "precise": false } }
```

**The user denies:**
```
-> { "type": "location.permission", "id": "l3" }
<- { "type": "location.permission.result", "id": "l3", "permission": "denied",
     "precise": false, "background": false }
-> { "type": "location.current", "id": "l4" }
<- { "type": "location.current.result", "id": "l4", "error": "denied" }
```

**Host policy denies geolocation to the runtime:**
```
<- { "type": "location.state", "available": false, "reason": "blocked",
     "permission": "prompt", "precise": false, "background": false, "active": false }
```

**Stop receiving updates:**
```
-> { "type": "location.unsubscribe" }
```

### Error Handling

`location.permission.result` carries `error` when the shell cannot process the
request, for example a malformed payload or `available` `false`; the other
fields then report the current grant. `location.current.result` carries a
`LocationError`: `"denied"` without a grant, `"unavailable"` when no position
can be obtained, `"timeout"` when `timeout` elapses, and `"inactive"` when the
napplet is not eligible for delivery. Unavailability is also reported in state.
The shell silently ignores malformed or excessive subscription messages.

## Shell Behavior

- The runtime MUST be the only reader of device position on behalf of
  napplets. Where the host offers a native geolocation API, the runtime MUST
  deny it to napplets.
- Where the host cannot deny native access, the shell MUST NOT present
  `location` as isolated and SHOULD NOT offer the `location` domain.
- The shell MUST respond to every `location.permission` and `location.current`
  with a result carrying the same `id`.
- The shell MUST NOT prompt the user except in response to
  `location.permission`. A prompt MUST identify the napplet by its verified
  identity.
- `precise` and `background` MUST be grants distinct from coarse position. The
  user MAY grant any subset of the requested tiers.
- The shell MUST key grants to the verified napplet identity. It MAY persist
  them under its own policy and MUST let the user revoke them. On revocation
  the shell MUST send a new `location.state` and stop positions.
- After a denial the shell SHOULD NOT prompt again until the user changes the
  decision. It SHOULD rate-limit `location.permission`.
- Without a precise grant, the shell MUST coarsen every position:
  - snap it to the centre of a fixed grid cell no smaller than 1 km on any
    side, for example a geohash of 5 characters;
  - set `accuracy` to at least the distance from that centre to the cell's
    farthest corner;
  - set `altitude`, `altitudeAccuracy`, `heading` and `speed` to `null`; and
  - set `precise` to `false`.
- The shell MUST coarsen by snapping, not by random noise. Noise averages out
  over repeated reads.
- The shell MUST send `location.position` only to a napplet that holds a grant
  and either is visible on a visible host surface or holds a background grant.
- When a napplet stops being eligible, the shell MUST stop positions at once and
  send `location.state` with `active` `false`.
- The shell MUST show a persistent indicator while any napplet receives
  positions in the background. It SHOULD show an indicator whenever any napplet
  is active.
- The shell SHOULD send at most one `location.position` per second per napplet
  and MAY skip unchanged positions.
- The shell MUST send a `location.state` snapshot after each
  `location.subscribe`, then whenever that napplet's view changes. It SHOULD
  NOT resend an unchanged snapshot.
- The shell MUST report `available` `false` with a `reason` when the runtime
  itself cannot read position.
- The shell MUST stop sending `location.state` and `location.position` after
  `location.unsubscribe` and when the napplet's session ends.
- The shell SHOULD stop reading position when no napplet is eligible for
  delivery.
- The shell MAY enforce ACL checks on `location` capabilities.

## Napplet Guidance

- A napplet SHOULD call `requestPermission` from a user action, and SHOULD
  explain why it needs position before calling it.
- A napplet SHOULD request `precise` and `background` only when it cannot work
  without them.
- A napplet SHOULD use `current` for one-off needs, such as tagging a note,
  instead of a subscription.
- A napplet that derives a Nostr `g` (geohash) tag from a `Position` SHOULD NOT
  use more geohash characters than `accuracy` supports.
- A napplet SHOULD treat `active` `false` as a pause.
- The domain is optional. A napplet SHOULD NOT declare `location` as required
  unless it cannot run without position.

## Web Projection Guidance

This section is projection guidance only. The contract above does not change.

- The shell denies native geolocation to napplet frames with the
  Permissions-Policy feature `geolocation`, for example
  `allow="geolocation 'none'"` on each napplet `iframe`. CSP cannot govern
  geolocation.
- `geolocation` defaults to `'self'`. Sandboxed `srcdoc` frames have an opaque
  origin, so they are already denied; the explicit `'none'` also covers
  same-origin frames.
- [MDN compatibility data](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe#browser_compatibility)
  lists the `allow` attribute's `geolocation` feature in Chromium 64+ and
  Firefox 74+, and lists Safari as unsupported. Where an engine does not
  enforce it, native position reaches the napplet's realm and no binding
  replacement can block it. A shell SHOULD verify enforcement per engine before
  offering `location`.
- Under denial, `getCurrentPosition` and `watchPosition` remain present in the
  frame and invoke their error callback with `PERMISSION_DENIED`.
- A binding MAY then replace those operations with ones served from
  `location.*`. Existing libraries then keep working unchanged. Such a
  replacement SHOULD:
  - serve `getCurrentPosition` from `location.current`, and `watchPosition` /
    `clearWatch` from `location.subscribe` and `location.position` with local
    watch ids;
  - keep the native property attributes, name, length and receiver check;
  - deliver objects with the native `GeolocationPosition`,
    `GeolocationCoordinates` and `GeolocationPositionError` prototypes and own
    value fields, since script cannot construct those types;
  - map `"denied"` to `PERMISSION_DENIED`, `"unavailable"` and `"inactive"` to
    `POSITION_UNAVAILABLE`, and `"timeout"` to `TIMEOUT`;
  - send `location.permission` on the first call while permission is
    `"prompt"`, as browsers prompt on first use; and
  - leave the native API untouched when it is not denied.
- The binding MAY also answer `navigator.permissions.query` for `geolocation`
  from `location.state`.
- Hosts throttle or stop geolocation in hidden pages. Background delivery is
  bounded by the host. The shell reports `active` `false` when the host stops
  delivering.

## Security Considerations

- Position identifies people. A trace of coarse positions still reveals home,
  work and routine. Background delivery therefore needs its own grant and a
  visible indicator.
- Grid snapping keeps repeated coarse reads at one value. Coarse positions omit
  `heading` and `speed`, which would otherwise reveal motion within a cell.
- Other domains leak position. Motion traces and compass heading allow route
  reconstruction. A shell SHOULD treat a `motion` grant as partial position
  exposure when it presents prompts.
- The isolation guarantee depends on the runtime denying every native
  geolocation path to napplets. A shell that leaves native geolocation reachable
  provides no isolation, whatever this domain reports.
- Geofencing, reverse geocoding and address lookup are out of scope. This
  domain never publishes position; a napplet that shares position does so
  itself, through `relay`.

## Implementations

- None yet.
