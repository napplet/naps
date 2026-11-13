NAP-WORKER
==========

Runtime-owned Workers
---------------------

`draft`

**NAP ID:** NAP-WORKER
**Domain:** `worker`
**Depends:** none.
**Projection:** [Web worker binding](../projections/web.md#worker-binding-draft).

## Description

NAP-WORKER lets a napplet request one or more dedicated workers from the
runtime. A worker is an isolated execution context for a napplet-supplied
program. The napplet designs the computation; the shell mediates creation,
messages, and termination; the runtime executes the program. Each worker
belongs to the requesting napplet endpoint. Workers communicate only with that
endpoint through the shell.

The domain provides background computation, not additional authority. Worker
programs remain untrusted. A runtime MUST NOT expose `worker` unless it can
enforce the isolation and lifecycle requirements below.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `info` | none | `WorkerLimits` | `worker.info` / `worker.info.result` |
| `create` | `source` (text) | `WorkerHandle` | `worker.create` / `worker.create.result` |
| `post` | `workerId` (text), `data` (`WorkerData`) | acknowledgment | `worker.post` / `worker.post.result` |
| `terminate` | `workerId` (text) | acknowledgment | `worker.terminate` / `worker.terminate.result` |
| `onEvent` | handler for `WorkerEvent` | local unsubscribe function | receives `worker.message`, `worker.error`, `worker.closed` |

`onEvent` installs a local listener. It sends no subscription request. The
napplet SHOULD install its listener before creating workers. Removing a listener
does not terminate a worker. The binding dispatches events to currently installed
listeners and does not replay events to later listeners.

### Schemas

`WorkerLimits` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `maxWorkers` | yes | integer | Positive maximum live workers for this endpoint. |
| `maxSourceBytes` | yes | integer | Positive maximum UTF-8 byte length of `source`. |
| `maxMessageBytes` | yes | integer | Positive maximum encoded `WorkerData` size in either direction. |
| `maxQueuedMessages` | yes | integer | Positive maximum pending messages per worker, per direction. |

Limits are a policy snapshot, not reserved capacity. The shell MAY apply lower
available capacity due to aggregate resource use. It MUST enforce finite limits
and MUST NOT silently truncate source or messages. A projection defines the
message encoding used to measure `maxMessageBytes`.

`WorkerHandle` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `workerId` | yes | text | Non-empty, opaque runtime-assigned identifier scoped to the owning endpoint's lifetime. |

`WorkerData` is a JSON-compatible value: null, boolean, text, finite number,
list of WorkerData, or map of text to WorkerData. Cycles, executable values,
shared memory, and transferable handles are excluded. Values MUST be copied
without coercion or dropped fields. A binding MUST reject values outside this
set before sending them. The shell MUST independently validate inbound data.

`WorkerError` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `code` | yes | text | One of the codes in Error Handling. |
| `message` | yes | text | Human-readable diagnostic; not a machine-readable discriminator. |

`WorkerEvent` fields:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | yes | text | `"worker.message"`, `"worker.error"`, or `"worker.closed"`. |
| `workerId` | yes | text | Assigned from the runtime's worker ownership record. |
| `data` | message only | `WorkerData` | Program output, passed as data without interpreting its contents. |
| `error` | error only | `WorkerError` | Fatal execution or output failure. |
| `reason` | closed only | text | `"terminated"`, `"selfClosed"`, `"failed"`, `"limitExceeded"`, or `"revoked"`. |

### Creation and execution

`create` accepts a non-empty, self-contained program as source text, not a
location to fetch. The projection defines its execution language and the
worker-side receive, send, and close primitives. A program MUST NOT load external
code or create child workers. The runtime MUST establish isolation and ownership
before executing any supplied source.

A successful result allocates a worker and returns its identifier. It does not
assert that the program has finished initialization. Syntax errors and uncaught
execution errors after allocation produce `worker.error`, then `worker.closed`
with reason `failed`. Failure before allocation returns an error result and
MUST leave no worker running. Runtime startup MUST have a finite deadline;
failure to start MUST resolve through one of these failure paths.

The runtime MUST order the successful creation result before any event for that
worker. The binding MUST expose the handle before dispatching those events.
Programs that need application readiness signal it through ordinary output.
Each call creates a separate worker; repeated source does not imply sharing.

### Messaging

`post` queues one copy of `data` for the worker. Success means acceptance into
the runtime's delivery queue, not processing by the program. A rejected post
MUST NOT deliver its data. Output from the worker becomes `worker.message`.

The runtime MUST preserve acceptance order for messages to each worker and
emission order for output from each worker. There is no ordering across workers
or between the two directions. Delivery is at most once. The shell MUST NOT
automatically retry or restart workers. Closing a worker MAY discard pending
messages; an acknowledgment is not a guarantee of processing before closure.

When the inbound queue is full, `post` fails with `LIMIT_EXCEEDED`. Invalid or
oversized worker output, or a full outbound queue, MUST fail the worker: emit
`worker.error`, then `worker.closed` with reason `failed` for invalid output or
`limitExceeded` for a resource limit. The shell MUST retain enough control
capacity to deliver these terminal events independently of the data queue.

### Termination and ownership

`terminate` stops execution without requiring program cooperation, discards
pending messages, and releases execution resources. For a live worker, the
runtime MUST emit `worker.closed` with reason `terminated` before acknowledging
termination. No worker event may follow its `worker.closed` event.

Termination is idempotent for identifiers previously issued to the same endpoint,
including workers that already closed. A closed identifier MUST NOT be reused
during that endpoint's lifetime. The runtime retains enough ownership information
to distinguish a closed worker from an identifier it never issued to this
endpoint. Posting to an owned closed worker fails with `WORKER_CLOSED`.

An identifier belonging to another endpoint MUST be treated exactly like an
identifier never issued to the caller. Two endpoints running the same verified
artifact do not share workers. A reload or replacement creates a new endpoint
lifetime; old handles MUST NOT regain authority.

Each allocated worker has exactly one terminal `worker.closed` event while its
owner remains reachable. A worker's own close primitive uses `selfClosed`.
Policy revocation uses `revoked`; enforced execution or resource budgets use
`limitExceeded`. Concurrent closure causes are serialized: the first closure
wins and later termination requests acknowledge the already closed worker.

## Wire Protocol

Requests carry a non-empty text `id`, unique among that endpoint's outstanding
requests. Each recognized, valid request receives exactly one corresponding
result with the same `id` while the endpoint remains reachable. Results carry
`ok` (boolean). Success includes the fields below; failure includes only
`error` (`WorkerError`) in addition to `type`, `id`, and `ok: false`.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `worker.info` | napplet -> shell | `id` |
| `worker.info.result` | shell -> napplet | `id`, `ok`, `limits` (`WorkerLimits`) on success |
| `worker.create` | napplet -> shell | `id`, `source` (text) |
| `worker.create.result` | shell -> napplet | `id`, `ok`, `workerId` on success |
| `worker.post` | napplet -> shell | `id`, `workerId`, `data` |
| `worker.post.result` | shell -> napplet | `id`, `ok` |
| `worker.terminate` | napplet -> shell | `id`, `workerId` |
| `worker.terminate.result` | shell -> napplet | `id`, `ok` |
| `worker.message` | shell -> napplet | `workerId`, `data` |
| `worker.error` | shell -> napplet | `workerId`, `error` |
| `worker.closed` | shell -> napplet | `workerId`, `reason` |

Worker events carry no request `id`. The shell supplies their routing fields;
program output cannot supply an envelope or override ownership.

### Examples

After creating a worker, a napplet exchanges application data and terminates it:

```json
{ "type": "worker.post", "id": "p1", "workerId": "w1", "data": { "task": "sum", "values": [2, 3] } }
{ "type": "worker.post.result", "id": "p1", "ok": true }
{ "type": "worker.message", "workerId": "w1", "data": { "sum": 5 } }
{ "type": "worker.terminate", "id": "t1", "workerId": "w1" }
{ "type": "worker.closed", "workerId": "w1", "reason": "terminated" }
{ "type": "worker.terminate.result", "id": "t1", "ok": true }
```

### Error Handling

| Code | Meaning |
|------|---------|
| `INVALID_REQUEST` | Malformed fields, empty source, or data outside `WorkerData`. |
| `NOT_ALLOWED` | Shell policy denies the operation. |
| `NOT_FOUND` | Worker identifier was never issued to this endpoint. |
| `WORKER_CLOSED` | Posting to a worker that has closed. |
| `LIMIT_EXCEEDED` | Source, message, queue, worker count, or execution budget exceeded. |
| `START_FAILED` | The runtime could not start the execution context. |
| `EXECUTION_FAILED` | Syntax error or uncaught program failure after allocation. |
| `INTERNAL_ERROR` | Runtime failure that does not fit a more specific code. |

Recognized malformed requests with a usable `id` receive `INVALID_REQUEST`.
Requests without a usable `id` and unrecognized message types are silently
ignored. These rules do not override projection authentication requirements.
Error messages MUST NOT expose shell secrets, another endpoint's data, or
internal resource locations. A napplet MUST branch on `code`, not `message`.

## Shell Behavior

- The shell MUST authenticate the napplet endpoint and check ownership and policy
  for every request. A worker identifier alone is not authorization.
- The runtime MUST terminate all of an endpoint's workers when the endpoint is
  destroyed, replaced, or disconnected. Cleanup MUST NOT depend on delivery of
  a final event or on cooperative worker code.
- Revoking `worker` MUST prevent new creation and stop all affected workers.
- The shell MUST bound source size, message size, queues, concurrent workers,
  and execution lifetime. It SHOULD account for aggregate use under the verified
  napplet identity so opening more endpoints cannot evade resource policy.
- The runtime MUST keep the shell responsive to termination even if a program
  never yields. Workers MUST NOT run on the shell's interactive execution path.

## Security Considerations

**Executing napplet code with shell authority breaks the seam.** A worker MUST
NOT inherit shell credentials, privileged storage, signing material, direct
network access, or communication channels to other napplets. Isolation MUST be
enforced before source executes. Source inspection or an agreement by the
program to avoid privileged operations is insufficient.

Workers have no NAP grants or independent napplet identity. Their only external
input and output is the owning napplet's mediated data channel. A napplet may
interpret output and make a separately authorized NAP request itself; the shell
MUST NOT dispatch worker output as a capability request. A value containing
`type`, `id`, or `workerId` remains ordinary data.

Worker source is untrusted even when supplied by a verified napplet artifact.
Accepting source MUST NOT change that artifact's identity or authorize dynamic
code to act as a different napplet. The shell MUST enforce isolation against
malicious source, including attempts to create nested execution contexts or
bypass message and resource limits.

## Implementations

- None yet.

## Changelog

- `aa858e5` - Drafted dedicated worker creation, copied messaging, termination, ownership, limits, isolation, and the web projection binding.
