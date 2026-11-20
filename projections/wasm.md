WASM Projection — Design Proposal
=================================

`draft` · ideation · no implementation or stable ABI specified

**Interoperability blockers:** standalone WASM artifacts have no agreed manifest
binding here, and the carrier has no agreed byte encoding or application binary
interface (ABI). This proposal MUST NOT be advertised as implemented support.
Arbitrary host imports would bypass shell mediation and break the seam.

This projection would run a napplet as an isolated WebAssembly guest. It changes
how the guest reaches the runtime, not the operations, schemas, errors, or trust
boundaries of any [NAP](../README.md#glossary). It introduces no capability domain.
The napplet designs, the shell mediates, the runtime executes, and the user
composes.

The proposed requirements below constrain an experiment. They are not a complete
interoperability specification. The [open decisions](#open-decisions) must be
resolved before independent implementations can claim compatibility.

## Why this projection

A small event-processing napplet could run without a browser, communicate with
other napplets through the same mediated contracts, and leave presentation to a
separate napplet chosen by the user. The runtime would supply capabilities; the
guest would supply computation. This is a use case to test, not a new convention
or a promise of smaller artifacts or faster execution.

WASM embedded in an HTML napplet already belongs to the
[web projection](web.md). Its surface remains `window.napplet.<domain>` under
[NIP-5D](https://github.com/nostr-protocol/nips/pull/2303). A standalone WASM guest
has no `window`, iframe, or DOM implied by this proposal.

## At a glance

| Concern | Proposed WASM binding |
|---------|-----------------------|
| Host | One isolated guest instance per live napplet endpoint; runtime-owned scheduling and limits |
| Carrier | Existing `domain.action` envelopes through a bounded host message queue |
| Surface | A runtime-provided binding presents the owning NAP's operations in the guest language |
| Discovery | A local availability query for bare domains, usable before any domain request and without `shell` |
| Identity | Runtime-owned instance context bound to verified manifest and artifact before guest execution |
| Authority | Only explicitly permitted host imports; owning NAP authorization rules remain authoritative |
| Presentation | No implicit rendering API; a host exposes a UI-related domain only if it can honor that domain's contract |

## Candidate bindings

| Candidate | Benefit | Cost |
|-----------|---------|------|
| Core WASM with a message queue | Reuses envelopes directly; isolates transport questions from domain changes | Requires a custom memory ABI, codec, allocation rules, and scheduling agreement |
| Component Model carrying the same envelopes | Reuses envelopes while gaining standard cross-language calling and ownership machinery | Still needs an envelope codec, queue lifecycle, and qualified toolchains |
| Component Model with typed domain imports | Makes records and resource lifetimes explicit; enables generated bindings | Must map evolving NAP schemas, optional domains, and errors without creating a second source of truth |
| WASM inside a web napplet | Reuses current HTML distribution and browser mediation | Remains the web projection; does not establish a standalone host binding |

**Recommendation:** start with a core-WASM message-queue experiment and compare
it with a Component Model envelope carrier before freezing an ABI. A small
transport surface does not imply a small implementation: the custom codec and
memory ownership may make the component alternative cheaper overall.

The [Component Model rationale](https://component-model.bytecodealliance.org/design/why-component-model.html)
and [WIT reference](https://component-model.bytecodealliance.org/design/wit.html)
describe its cross-language contracts. WIT is an interface-description language,
not a replacement for the NAP contract. Its
[async model](https://component-model.bytecodealliance.org/design/async.html)
includes asynchronous functions, streams, and futures; engine and toolchain
versions still need qualification. This proposal does not reject components on
the premise that asynchronous delivery is unavailable.

## Artifact and identity boundary

The current [NIP-5D manifest](https://github.com/nostr-protocol/nips/pull/2303)
describes a self-contained `/index.html`, its artifact hash, and browser loading.
It does not authorize substituting a standalone module or component for those
HTML bytes. This proposal does not assign a new event kind, manifest tag, or
artifact selector. A prototype MUST label any local packaging agreement as
experimental and MUST NOT publish it as existing manifest compatibility.

A standalone binding needs agreement on artifact selection, the bytes covered
by the signed hash, supported WASM features, and binding version selection.
Unknown formats or versions MUST fail before guest execution; the loader MUST
NOT guess from a filename or retry a rejected artifact under a weaker binding.
The first experiment should use one self-contained module. Linked guest code,
adapters, and additional artifacts need an explicit verification rule before
they become part of a portable distribution format.

After that packaging agreement, the runtime MUST verify the signed manifest and
artifact bytes and create an instance context before any guest code executes.
It MUST bind imports to that context, not accept an identity argument from the
guest. This includes code executed during
[WASM instantiation](https://webassembly.github.io/spec/core/exec/modules.html),
which can invoke a start function. Required-domain checks, import validation,
and execution limits must already be in place.

The context MUST distinguish publishers, manifests, and simultaneous instances.
A bare `d` tag is not an identity, and manifests without `d` must remain
representable. Persistent manifest identity and a live endpoint identifier are
different things. [NAP-INC](../naps/NAP-INC.md) owns its endpoint and channel
identifiers; [NAP-INTENT](../naps/NAP-INTENT.md) owns its catalog identifiers.
Neither may be substituted for the other. User identity reported by
[NAP-IDENTITY](../naps/NAP-IDENTITY.md) does not authenticate the guest.

## Proposed transport shape

These are conceptual binding primitives, not new NAP operations, wire messages,
or finalized import/export names. Exact signatures and numeric status values
remain an ABI decision.

| Primitive | Direction | Input | Outcome |
|-----------|-----------|-------|---------|
| Availability | guest to host | Bare domain (`text`) | `boolean` for this instance's current exposure; unknown domains return false |
| Submit | guest to host | One encoded NAP envelope | Transport accepted or rejected before dispatch; never the NAP operation's result |
| Receive | guest to host | Guest-owned output capacity | One complete queued envelope, empty queue, or required capacity without consuming the envelope |
| Step | host to guest | Scheduling turn | Guest drains deliveries and performs bounded work, then returns control |

Availability MUST work without [NAP-SHELL](../naps/NAP-SHELL.md). Before running
the guest, the runtime MUST check its complete required-domain set and the
required dependencies of exposed domains. Missing optional domains alone MUST
NOT prevent loading. Manifest declarations are not grants; a linked transport
import also grants no domain. Availability MUST reflect exposure to this
instance, not everything the host implements.

Every submitted message MUST remain associated with the calling instance.
Unknown or unexposed domains MUST be rejected before effects. For exposed
domains, the runtime MUST preserve the owning NAP's authorization lifetime and
failure contract. For example, INC channels authorize at open; this projection
does not add per-message channel authorization. Revocation follows the owning
contract and removes the affected authority even if the guest cached availability.

The binding MUST preserve `type`, correlation `id`, absent fields, unsolicited
messages, and fire-and-forget behavior. Request IDs MUST be scoped to the live
endpoint; two guests using `q1` cannot share results. Domain-owned handles remain
bound to their owning instance. Transport acceptance means the host accepted an
envelope for processing, not that an operation succeeded or that a target acted.

NAP errors MUST keep their owning schemas. Transport rejection, invalid memory,
or instance termination MUST be observable as binding failures, not fabricated
`*.result` messages. A timeout does not establish whether an external effect
occurred. The binding MUST NOT automatically retry an accepted side effect.

### Values and memory

An encoded envelope is a representation of the existing contract, not a new
payload schema. A codec MUST preserve the values required by every exposed
domain, including bytes, numeric ranges, and missing versus `null` fields.
Arbitrary `any` payloads need an agreed portable value model. JSON-only examples
do not settle that model. An experiment that only carries text records MUST NOT
claim full domain compatibility where other values are permitted.

For the core candidate, the runtime MUST validate pointer/length ranges with
overflow-safe bounds checks before reading or writing guest memory. Accepted submission
MUST copy the complete envelope into host-owned memory before returning. The
host MUST NOT retain guest pointers across asynchronous work or memory growth.
Receive MUST copy into guest-owned memory and consume a queued envelope only
after a complete successful copy. Insufficient capacity MUST leave it queued.
The ABI must still settle memory selection, pointer width, encoding, capacity
reporting, allocation ownership, and maximum frame size.

The first experiment excludes shared guest memory and concurrent calls into one
instance. Supporting either later requires a separate race and ownership model.

### Scheduling and teardown

Submit MUST NOT wait for network I/O, user approval, or a domain result. Results
and unsolicited messages enter the instance's receive queue. The runtime wakes
an idle guest by scheduling a step; the guest need not spin on an empty queue.
The host MUST NOT re-enter a guest already executing. A reply produced during
submission remains queued until the guest can receive it.

Each endpoint queue MUST preserve enqueue order and the ordering required by its
NAPs. This does not impose an order on independent asynchronous completions.
Local callbacks and handles MUST preserve their owning contract's retention
rules; draining the transport queue does not authorize dropping an undelivered
intent or an unclaimed INC channel message. Transport readiness MUST NOT become
a mandatory `shell.ready` exchange.

The runtime MUST bound memory, execution time, queued bytes, and outstanding
work. It MUST reject overload before transport acceptance where possible.
After acceptance, it MUST follow domain-defined failure and overflow rules;
if transport delivery becomes impossible, it MUST terminate the endpoint and
report the binding failure to the shell. Reporting MUST NOT depend on a final
guest callback or silently lose required peer notifications.
For INC channel overflow, its channel closure and peer notification rules apply.
Concrete limits and how the guest learns them remain an ABI decision.

On termination, the runtime MUST release instance-owned queues and subscriptions,
close channels under their owning rules, and prevent late results from reaching
a replacement instance. Cleanup MUST NOT depend on a cooperative guest shutdown
callback. Work already accepted under a domain's independent lifetime remains
owned by that domain: accepted intent delivery, for example, survives source
termination. Termination does not undo completed external effects.

## Host imports and presentation

The runtime MUST allowlist imports before execution and reject unknown imports.
It MUST NOT supply ambient sockets, HTTP clients, filesystem access, process
execution, environment secrets, signing keys, or wallet credentials. Adding
WASI (WebAssembly System Interface) imports or a language runtime adapter must
not create an alternate route around the NAP seam. Host references, memories,
and tables MUST NOT expose another napplet's state or a privileged host object.

WASM isolation alone does not establish these properties; host functions are
part of the boundary. See [Wasmtime's security model](https://docs.wasmtime.dev/security.html).
The runtime also needs a way to interrupt a guest that never returns. Wasmtime's
[fuel and epoch mechanisms](https://docs.wasmtime.dev/examples-interrupting-wasm.html)
are examples, not a portable engine requirement. Host calls need their own
deadlines and bounds; interrupting WASM instructions does not bound host I/O.

The first experiment is headless. A later presentation binding must identify
which existing domain owns each capability. It MUST NOT disguise DOM access,
arbitrary graphics APIs, or raw device access as transport helpers. A
[theme](../naps/NAP-THEME.md) value remains the same record; its presence alone
does not promise a renderer or permission to fetch referenced URLs.

## Convention URI binding

The guest-facing binding MUST apply the
[convention URI rules](web.md#convention-uri-binding) before routing: stable
queryless and fragment-free identity, unique percent-decoded text fields,
literal `+`, no scalar coercion, and no query combined with explicit payload.
Routers match the resulting identity by exact equality.

Only intent URI invocation accepts a bare `naddr` fragment as a handler
recommendation. The binding extracts it into `handlerHint`; user defaults and
shell policy retain precedence. INC and other convention-URI operations reject
fragments. A guest bypassing a convenience wrapper must not bypass host
validation of the owning domain's normalized message.

**Existing contract conflict:** the current NAP-INTENT text has duplicate options
schemas and a blanket fragment rejection that conflicts with the web projection
and the repository's handler-hint rule. Its
[reconciliation PR](https://github.com/napplet/naps/pull/106) addresses this.
Resolve that owning contract before freezing or claiming intent interoperability
for WASM; this projection does not repair or redefine it.

## Illustrative exchange

Assume `theme` is exposed and `shell` is absent. Availability for `theme` returns
true. The guest submits the following envelope; transport acceptance alone does
not fulfill the request:

```json
{ "type": "theme.get", "id": "q1" }
```

A later receive yields the owning NAP's result:

```json
{ "type": "theme.get.result", "id": "q1", "theme": { "colors": { "background": "#ffffff", "text": "#111111", "primary": "#663399" } } }
```

An unsolicited update uses the same queue and has no request ID:

```json
{ "type": "theme.changed", "theme": { "colors": { "background": "#111111", "text": "#ffffff", "primary": "#9966cc" } } }
```

These JSON renderings show semantic envelopes, not the proposed byte encoding.
They neither add a wire subscription nor require a browser or a renderer.

## Open decisions

| Decision | Evidence needed before freezing the projection |
|----------|------------------------------------------------|
| Standalone artifact and manifest binding | An agreed signed-artifact selection and verification rule, including adapters and version rejection; no silent reinterpretation of HTML manifests |
| Core versus Component Model carrier | Equivalent guest/host experiments measuring implementation complexity, ownership safety, toolchain portability, and asynchronous delivery |
| Value model and codec | Round trips for every supported domain schema, byte-bearing data, numeric limits, omitted/null fields, and convention payloads; resolve gaps in the owning contract |
| Exact ABI and feature set | Published signatures, import/export namespace and version, memory rules, frame limits, status codes, and startup/step scheduling fixtures |
| Revocation and overload | Domain-preserving denial/closure traces, bounded queue behavior, observable termination, and late-result isolation |
| Intent alignment | Resolution of the linked contract conflict and matching URI/handler-selection fixtures |
| Presentation | A separately reviewed mapping for any UI domains claimed by a host; no implicit UI support from executing WASM |

## Future experiment acceptance criteria

These checks have **not been run**. They define follow-up implementation evidence,
separate from documentation validation of this proposal.

| Case | Required observation |
|------|----------------------|
| Discovery without `shell` | Request/result and push work; unknown/absent optional domains report false; a missing required domain prevents guest execution |
| Pre-execution identity | Bad signature/hash or forbidden import prevents start-function effects; guest-supplied identity cannot change authority |
| Isolation | Two instances reuse `q1` and attempt cross-instance handles; results and state remain confined to their owners |
| Memory ownership | Invalid ranges, overflow, insufficient receive capacity, and memory growth cause no host corruption, queue loss, or stale pointer use |
| Complete values | Byte and numeric boundary fixtures round-trip losslessly; omitted and null fields remain distinct |
| Async and pressure | Delayed approval and unsolicited delivery do not block the host or re-enter the guest; queue limits produce documented rejection/closure/termination |
| Lifecycle | Restart receives no old results; peers receive required closure; accepted intent delivery survives source exit; side effects are not replayed |
| URI parity | Literal plus and text-only queries survive; duplicate decoded names, malformed escapes, query-plus-payload, and INC fragments fail; intent hint precedence follows the reconciled owner |
| Resource limits | An infinite guest loop and a stalled host call cannot exhaust the runtime indefinitely; cleanup completes without guest cooperation |
| Independent implementations | The same guest artifact runs on two independently written host bindings with matching semantic transcripts; record exact engine, toolchain, and binding versions |

No runtime, guest SDK, benchmark, or cross-host conformance result accompanies
this proposal. Promotion requires implementation evidence and resolution of the
open decisions above.
