NAP-WEBMCP
==========

Expose live napplet operations to authorized agents
-------------------------------------------------

`draft`

**NAP ID:** NAP-WEBMCP  
**Domain:** `webmcp`  
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.webmcp`; domain presence signals availability.

**Depends:** none.

## Description

The runtime accepts typed tools from a running napplet and mediates their
discovery and execution by authorized agents. The napplet implements each tool
against its current application state. The shell controls who can invoke it.
Registration grants neither agent access nor additional host capabilities.

The shell MAY project approved tools into browser [WebMCP](https://webmachinelearning.github.io/webmcp/),
offer them to a shell-owned agent, or do both. Native browser support, an LLM,
a chat interface and a remote MCP server are not prerequisites. This contract
defines live tool registration and execution, not inference or agent planning.
Ordinary napplet interaction SHOULD remain usable when this domain is absent
or tool exposure is denied.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `registerTool` | `tool` (Tool), local executor | Registration | `webmcp.register` / `webmcp.register.result` |
| `unregister` | `registrationId` (text) | success or Error | `webmcp.unregister` / `webmcp.unregister.result` |
| `reset` | none | success or Error | `webmcp.reset` / `webmcp.reset.result` |
| `onExposureChanged` | local listener | unsubscribe operation | `webmcp.changed` |

The executor and listener are local binding constructs, never serialized code.
An executor receives validated arguments and a context for cooperative
cancellation and progress. It returns JSON output or an Error. A web binding
MAY expose an asynchronous callback with an abort signal and a progress function.

`registerTool` accepts a descriptor without exposing it. Adding a registration
MUST revoke existing exposure grants for that instance, so changed operations
cannot inherit approval. Duplicate local names MUST be rejected; replacement
requires withdrawal first. `reset` retains descriptors but revokes exposure and
cancels outstanding calls. A napplet MUST reset before replacing the private
document or equivalent context to which an exposure grant applied. It MUST await
reset acknowledgement and prevent old executors from reading or changing the new
context. A successful reset acknowledges grant revocation and cancellation, not
executor settlement or rollback.

### Schemas

**Tool**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `name` | yes | text | Local name matching `[a-z][a-z0-9_]{0,63}`; unique within this instance |
| `title` | no | text | Human-readable label |
| `description` | yes | text | Purpose, argument units, preconditions and side effects |
| `inputSchema` | yes | Schema | Root type MUST be `object` |
| `outputSchema` | no | Schema | Validates successful output when supplied |
| `readOnly` | no | boolean | Advisory claim, default false; never authorization |

**Registration**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `registrationId` | yes | text | Runtime-assigned, scoped to this instance and registration lifetime |
| `name` | yes | text | Runtime-assigned consumer name; cannot collide with another instance |
| `exposed` | yes | boolean | Initially false; true means at least one authorized consumer can invoke tools |

Exposure is an aggregate availability notification. It neither identifies an
agent nor authorizes the napplet to contact one. Grants for different consumers
remain shell policy and MUST NOT be inferred from this boolean.

**Schema** is the following closed subset of
[JSON Schema 2020-12](https://json-schema.org/draft/2020-12).
Keywords use that dialect's meanings. Each schema MUST specify one `type`.
Unsupported keywords MUST fail registration, not be silently ignored.

| Keyword | Applicable type | Value |
|---------|-----------------|-------|
| `type` | all | `"object"`, `"array"`, `"string"`, `"number"`, `"integer"`, `"boolean"` or `"null"` |
| `description` | all | text |
| `enum` | all | non-empty list of JSON values matching the declared type |
| `properties` | object | map of text to Schema |
| `required` | object | list of distinct property names |
| `additionalProperties` | object | boolean; defaults to true |
| `items` | array | Schema for every item |
| `minItems`, `maxItems` | array | non-negative integer |
| `uniqueItems` | array | boolean |
| `minLength`, `maxLength` | string | non-negative integer |
| `minimum`, `maximum` | number, integer | finite number |

Inconsistent bounds or malformed keyword values MUST fail registration.
String lengths count Unicode code points, not encoding units. Equality for
`enum` and `uniqueItems` is structural: object member order does not matter;
array order does. Keywords outside their applicable type MUST fail registration.
References, remote schema loading, executable validators, composition and type
coercion are outside this contract. JSON data MUST contain only finite numbers
and ordinary JSON values. Bindings MUST prevent prototype-key assignment from
altering their own objects.

**Error**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `code` | yes | text | Machine-readable category |
| `message` | yes | text | Safe, actionable explanation |

Codes include `unsupported`, `invalid-tool`, `invalid-schema`, `invalid-request`,
`invalid-arguments`, `invalid-data`, `duplicate-tool`, `unknown-tool`,
`permission-denied`, `quota-exceeded`, `busy`, `cancelled`, `timeout` and
`execution-error`. Application failures, including stale document revisions,
SHOULD describe the recoverable cause without exposing credentials or stack dumps.

## Wire Protocol

Messages describe the seam independently of transport. The web projection carries
them using the [NIP-5D](https://github.com/nostr-protocol/nips/pull/2303) wire format.
Request IDs are non-empty text, unique among that instance's pending requests.
Every valid request MUST receive one matching result or error while the instance
remains connected. Success fields and `error` are mutually exclusive.
Requests and results correlate by `id`. Executor traffic correlates by both
`callId` and `registrationId`, scoped to the authenticated napplet instance.
Both identifiers are non-empty text. Arguments are a JSON object; progress
`text` is text. Successful completion MUST contain `result`, even when it is null.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `webmcp.register` | napplet → shell | `id`, `tool` |
| `webmcp.register.result` | shell → napplet | `id`, Registration fields or `error` |
| `webmcp.unregister` | napplet → shell | `id`, `registrationId` |
| `webmcp.unregister.result` | shell → napplet | `id`, `ok: true` or `error` |
| `webmcp.reset` | napplet → shell | `id` |
| `webmcp.reset.result` | shell → napplet | `id`, `ok: true` or `error` |
| `webmcp.changed` | shell → napplet | `exposed` (boolean) |
| `webmcp.invoke` | shell → napplet | `callId`, `registrationId`, `sequence`, `arguments` |
| `webmcp.progress` | napplet → shell | `callId`, `registrationId`, `text` |
| `webmcp.complete` | napplet → shell | `callId`, `registrationId`, exactly one of `result` or `error` |
| `webmcp.cancel` | shell → napplet | `callId`, `registrationId` |

`sequence` is an integer from 1 through 9007199254740991, strictly increasing
per registration. The binding MUST reject repeated or older sequences without
executing the callback. The shell MUST NOT reuse a call ID during an instance's
lifetime. Late, cross-instance and withdrawn-registration messages MUST NOT
complete or revive a call. Progress is optional text, not an execution receipt.

### Example

```json
{"type":"webmcp.register","id":"r1","tool":{"name":"count_faces","description":"Count faces in the open document.","inputSchema":{"type":"object","additionalProperties":false},"outputSchema":{"type":"integer","minimum":0},"readOnly":true}}
{"type":"webmcp.register.result","id":"r1","registrationId":"t1","name":"instance7_count_faces","exposed":false}
```

After shell authorization:

```json
{"type":"webmcp.changed","exposed":true}
{"type":"webmcp.invoke","callId":"c1","registrationId":"t1","sequence":1,"arguments":{}}
{"type":"webmcp.complete","callId":"c1","registrationId":"t1","result":24}
```

## Execution and lifecycle

- The shell MUST validate descriptors and arguments before dispatch and output
  against `outputSchema`, when present, before delivery. The napplet MUST also
  validate arguments and application-specific preconditions.
- The shell MUST bound registration count, descriptor/data size, queued work and
  execution time, and document these limits, including how approval waits count
  toward deadlines. Registration overload fails with
  `quota-exceeded`; execution overload fails with `busy`. Queues MUST be bounded.
- At most one executor per instance may run at once. Implementations MAY queue
  calls, rechecking authorization and registration immediately before dispatch.
  This serialization does not prevent concurrent manual edits; napplets SHOULD
  use revision checks or equivalent application-level conflict handling.
- Cancellation is cooperative. On revocation, timeout or caller cancellation,
  the shell MUST reject the caller, cancel dispatched work, discard queued work
  and ignore late output. The instance MUST remain busy until its executor
  settles or the instance ends. A rejected caller does not prove work stopped.
  A late terminal response MAY establish settlement and release the execution
  slot, but MUST NOT be delivered as a new success to the rejected caller.
- Withdrawal cancels calls to that registration. Reset revokes all consumer
  grants and cancels all outstanding calls for the instance.
- Reload, teardown, loss of the instance channel and shell account changes MUST
  withdraw registrations and grants. No registration survives by name alone.
  This contract does not resume calls across instances or document replacement;
  a consumer's continuation mechanism MUST NOT transfer the old authority.
- A timeout after dispatch can leave effects uncertain. The shell MUST NOT
  automatically retry an invocation. Cancellation cannot undo completed effects;
  napplets SHOULD make mutations visible and offer undo where appropriate.

## Shell behavior and security

The runtime MUST bind each registration and executor message to the verified
napplet instance, rather than trusting a supplied name or currently focused app.
Consumer names MUST be namespaced by the runtime. The shell MUST preserve clear
napplet attribution when projecting tools under its own origin or agent identity.
It SHOULD make active work and pending host approval visible and provide a way
to stop work or revoke access. Presentation remains shell policy, including
keeping approval controls reachable in expanded or fullscreen views.

Registration MUST NOT grant exposure. Authorization is shell-controlled and
scoped to the instance and current document context. Read-only tools disclose
data too: the shell SHOULD explain that outputs can reach an agent or its
provider. Behavior hints are untrusted claims, not proof of safety or user intent.
Consumers MUST NOT gain access to another instance through a tool's arguments.

Descriptions, guidance, progress and results are untrusted application content.
They MUST NOT become privileged shell instructions, expand grants or trigger
additional tools merely by being returned. Tools MUST NOT expose shell secrets.
The runtime SHOULD avoid logging arguments and outputs by default.

Tool execution MUST NOT grant additional network, signing, payment, file or
resource authority, bypass their separate approvals, or manufacture user
activation. A browser adapter belongs to the shell; it MUST NOT weaken the
napplet sandbox or expose tools to additional origins without authorization.
The domain's presence does not imply browser WebMCP support or external exposure.

An application may use other available domains while implementing a tool. Those
domains retain their own contracts; none is required by this interface. For
example, an export can report a file's name, media type and readiness after an
independently authorized file operation. A virtual path is not a public URL or
proof of download. This NAP introduces neither public artifact hosting nor an
agent file-download permission.

Accepting a call, granting a permission and completing its requested operation are
distinct. The runtime MUST NOT report tool success merely because a user approved
a host request. A tool returning a prepared artifact SHOULD state what is ready
and what remains for the user or consumer to do. The shell SHOULD offer an
accessible download action when it provides user-download delivery. No download
receipt may be inferred from the tool's successful return.

## Authoring guidance (informative)

Expose deliberate application operations, not arbitrary code execution or an
automatic reflection of every internal method. Reuse the same validated commands
and state as the human UI. Describe units, identifiers and side effects.

Domain guidance can be an ordinary read-only tool returning concise advice for a
requested topic. Essential preconditions still belong in operation descriptions
and executable validation. There is no skill registry or guarantee an agent will
read such guidance. Shell-owned chats and their context-loading policy remain
implementation choices.

NAP-KEYS actions cover named shortcut activation; NAP-INTENT covers convention
delivery. This interface adds typed arguments, correlated execution results and
agent authorization for a particular live instance. NAP-CVM's remote-service
direction and NAP-LLM's inference exchange do not register live application tools.

## Changelog

- `5d0f26a` - Propose permissioned live tool registration, typed execution, cancellation and context-scoped exposure.
