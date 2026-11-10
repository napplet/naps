NAP-LLM
========

User-selected language models
-----------------------------

`draft`

**NAP ID:** NAP-LLM
**Domain:** `llm`
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.llm`; domain presence signals availability.

## Description

NAP-LLM lets a napplet request text generation, optionally with images, through a
user-selected model connection owned by the runtime. The napplet owns its prompts,
conversation and result validation. The runtime owns connection selection,
credentials, disclosure consent and provider access. A connection may use the
user's API key, a local model or a runtime-provided service. None grants the
napplet direct network access or access to other napplets' conversations.

This interface has no dependency on another NAP. A purpose-built remote service
can instead expose an application contract through [NAP-CVM](https://github.com/napplet/naps/pull/31).
Service checkout, payments, embeddings, audio and video generation are outside
this proposal. Tool **calls and results** are part of the wire so napplets and
runtimes can represent agentic turns honestly. The runtime MUST NOT execute a
provider-returned tool call through this domain. The napplet owns tool execution
and may use a separately authorized capability.
The reference to NAP-CVM above is an alternative application contract, not a
dependency of this domain.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `select` | optional `requirements` (Requirements) | Model | `llm.select` / `llm.select.result` |
| `describe` | `handle` (text) | Model | `llm.describe` / `llm.describe.result` |
| `generate` | `request` (GenerationRequest) | Generation; optional Delta updates | `llm.generate` / `llm.generate.result`, `llm.delta` |
| `cancel` | `requestId` (text) | boolean | `llm.cancel` / `llm.cancel.result` |

`select` asks the user to choose a connection meeting the requirements. The
runtime MUST NOT silently substitute a different requested model or downgrade a
required capability. Selection does not authorize sending a prompt. `describe`
revalidates a selection without generating output or showing another chooser.
A handle becomes invalid when its connection, model, credentials, account or
endpoint lifetime changes. The napplet must select again after invalidation.

`generate` is one stateless request: the supplied messages are the entire
conversation for that operation. The runtime MUST NOT append another conversation
or silently discard supplied messages, roles or image parts. The display-only
`thinking` exception below does not permit discarding other conversation content. Provider-specific
input restrictions MUST produce `unsupported` before the generation request is
sent. Mapping roles and parts into the provider's equivalent format is permitted.

### Schemas

Unknown fields in these input records, explicit `null`, invalid types and invalid
encodings MUST produce `invalid-request`. Optional fields may be omitted.

**Requirements**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `model` | no | non-empty text | Exact provider model identifier; omission allows user choice |
| `images` | no | boolean | `true` requires image input; omitted or `false` imposes no requirement |
| `json` | no | boolean | `true` requires JSON-object output; omitted or `false` imposes no requirement |
| `tools` | no | boolean | `true` requires tool-calling support; omitted or `false` imposes no requirement |

**Model**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `handle` | yes | non-empty text | Opaque reference scoped to this authenticated endpoint and user account |
| `name` | yes | text | User-facing connection label |
| `model` | yes | non-empty text | Selected model identifier |
| `route` | yes | text | Human-readable disclosure route, including intermediaries and destination; never credentials |
| `images` | yes | boolean | Whether the connection supports image input |
| `json` | yes | boolean | Whether it supports JSON-object output |
| `tools` | yes | boolean | Whether it supports tool calling |
| `toolChoiceModes` | yes | list of text | Supported modes: `auto`, `none`, `required`, `tool`; empty when `tools` is false |
| `sessions` | no | boolean | Whether the runtime can offer a session grant for this handle; omitted means `false` |

The handle is not a URL, credential or transferable capability. The runtime MUST
NOT accept a napplet-supplied provider URL, key or identity as selection authority.
`images` does not promise every provider supports every placement of an image
within a conversation; unsupported combinations must be rejected explicitly.

`tools: true` MUST support `auto` and `none`. `toolChoiceModes` MUST list
exactly the supported modes, without duplicates. `tool` means named-tool forcing.
A valid request for an unlisted mode MUST produce `unsupported` before provider
submission; the runtime MUST NOT substitute another mode.

**GenerationRequest**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `handle` | yes | non-empty text | A live Model handle |
| `messages` | yes | non-empty list of Message | Ordered conversation |
| `format` | no | `text` or `json` | Defaults to `text`; `json` requires an object, not an array or scalar |
| `maxTokens` | no | positive integer | Output token limit; omission requests no napplet-defined limit. Runtime policy and model limits still apply; runtime policy MAY cap it |
| `temperature` | no | number | Sampling temperature; unsupported values MUST produce `unsupported` |
| `tools` | no | list of Tool | Tools the model may call |
| `toolChoice` | no | `auto`, `none`, `required` or ToolChoice | How the model selects tools; defaults to `auto` when tools are present |
| `authorization` | no | `generation` or `session` | Kind of authorization to ask for when none covers this request; defaults to `generation` |

**Tool**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `name` | yes | text | Portable tool name matching `^[A-Za-z0-9_-]{1,64}$` |
| `description` | no | text | What the tool does |
| `parameters` | no | map of text to any | JSON Schema with root `type: "object"`; omission means `{"type":"object","properties":{},"additionalProperties":false}` |

`parameters` uses [JSON Schema draft 2020-12](https://json-schema.org/draft/2020-12/json-schema-core).
Malformed schemas MUST produce `invalid-request`.

Tool names MUST be unique within `tools`. An empty list is equivalent to omission.
An explicit `toolChoice` requires a non-empty `tools` list; a forced name MUST
appear in that list. Violations, invalid names and a `parameters` value without
root `type: "object"` MUST produce `invalid-request`. A structurally valid schema
or choice unsupported by the connection MUST produce `unsupported` before
provider submission. JSON Schema keywords are not unknown record fields.

`auto` permits zero or more calls; `none` prohibits calls; `required` requires at
least one call; a named choice requires at least one call, all to that name.
A successfully completed turn violating the selected mode MUST produce
`provider-error`, not a fallback. Length and refusal remain partial outcomes.
Historical calls and results MAY be supplied without current `tools`; this allows
continuing a completed exchange without enabling new calls. Without `tools`, the
runtime MUST NOT accept new provider tool calls.

**ToolChoice**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | yes | `tool` | Forces one named tool |
| `name` | yes | non-empty text | Tool name to force |

**Message**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `role` | yes | `system`, `user`, `assistant` or `toolResult` | Role of this message |
| `content` | yes | non-empty list of Part | Ordered content parts |
| `toolCallId` | for `toolResult` only | non-empty text | Matches the `toolCall` part this message answers |
| `toolName` | for `toolResult` only | non-empty text | Name of the tool that produced the result |
| `isError` | no | boolean | For `toolResult`: whether the tool failed; defaults to `false` |

A `toolResult` message MUST answer exactly one call in the immediately preceding
assistant turn. Results MUST immediately follow that assistant message, one per
call, in any order, before any other message. Missing, duplicate, orphaned or
name-mismatched results MUST produce `invalid-request`. The runtime MAY regroup
results for its provider without changing their meaning. Call ids MUST be unique
within the supplied conversation. When a provider omits an id, the runtime MUST
synthesize one that remains unchanged in deltas and the terminal result; the
napplet MUST reuse it in the subsequent assistant history and matching result.
Ids are not credentials.

`toolResult` content permits only `text` and `image`. `toolCall` and `thinking`
parts are permitted only in assistant messages; `isError` is permitted only on
`toolResult`. Invalid placements MUST produce `invalid-request`. A connection
that cannot accept images in tool results MUST produce `unsupported` before
submission. If the provider has no native error flag, the runtime MUST represent
`isError: true` by prefixing the result with a text part containing
`Tool execution failed.` followed by the supplied content. It MUST NOT silently
discard the error indication.

**Part** is one of these records:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | yes | `text`, `image`, `thinking` or `toolCall` | Selects the record variant |
| `text` | for `text`/`thinking` only | text | Text content |
| `mimeType` | for `image` only | `image/png`, `image/jpeg` or `image/webp` | Declared image encoding |
| `data` | for `image` only | non-empty text | Standard padded base64 bytes, no whitespace, data-URI prefix or remote URL |
| `toolCallId` | for `toolCall` only | non-empty text | Stable id matching the later `toolResult` |
| `name` | for `toolCall` only | non-empty text | Tool name |
| `arguments` | for `toolCall` only | map of text to any | Tool arguments as JSON |

`thinking` is display-only, not reasoning continuation state. The runtime MUST
omit it from provider input when assistant history is replayed. If continuing
that history requires opaque or signed provider state that this contract cannot
represent, the runtime MUST return `unsupported` before provider submission.
It MUST NOT silently restart or degrade the reasoning loop. Providers that
return only redacted reasoning MAY omit a thinking part; no placeholder text or
provider blob is required. Supporting opaque continuation is outside this draft.

**Generation**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `content` | yes | list of Part | Complete accumulated output; only text, thinking and tool-call parts; may be empty for length or refusal |
| `finish` | yes | `stop`, `length`, `refusal` or `toolUse` | Normal completion, output limit reached, provider refusal, or a tool call |
| `usage` | no | Usage | Provider-reported token counts, when available |

**Usage**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `inputTokens` | yes | non-negative integer | Reported input tokens |
| `outputTokens` | yes | non-negative integer | Reported output tokens |
| `cacheReadTokens` | no | non-negative integer | Tokens read from a provider cache |
| `cacheWriteTokens` | no | non-negative integer | Tokens written to a provider cache |
| `reasoningTokens` | no | non-negative integer | Tokens attributed to reasoning |

Only `finish: "stop"` indicates normal completed output. `length` and `refusal`
are not completed output. A successfully completed turn containing any `toolCall`
MUST have `finish: "toolUse"`, regardless of the provider's normal stop label;
conversely, `toolUse` MUST contain at least one call. Tool execution MUST wait
for an authoritative `toolUse` result, never a delta. Truncation and refusal MUST
NOT be relabelled `toolUse`: such results MUST contain no `toolCall` parts,
even if some calls completed before another was truncated. Previously streamed call fragments remain provisional and are discarded.

Every completed call's arguments MUST parse as a JSON object, and its name MUST
appear in the current `tools`. Malformed arguments, non-object arguments, unknown
names or missing completion markers MUST produce `provider-error` with no raw
provider body echoed. Incomplete arguments on an explicitly truncated/refused
turn are discarded under the rule above, not treated as completed calls.

For `format: "json"`, concatenate the text parts of a `stop` result in order;
the concatenation MUST parse as a JSON object. This does not validate the
application schema, factual accuracy or suitability. Usage is not a price or
settlement receipt; cache and reasoning token counts do not imply a cost.

**Delta**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `requestId` | yes | non-empty text | The originating generation's `id` |
| `sequence` | yes | non-negative integer | Contiguous sequence starting at zero for this generation |
| `event` | yes | `start`, `delta` or `end` | Block lifecycle event |
| `block` | yes | `text`, `thinking` or `toolCall` | Kind of content block |
| `blockIndex` | yes | non-negative integer | Zero-based index of the block within the generation's content |
| `text` | for `delta` only | text | Text fragment, or JSON argument fragment for `toolCall` |
| `toolCallId` | for `toolCall` start only | non-empty text | Id used by the terminal call |
| `name` | for `toolCall` start only | text | Tool name used by the terminal call |

Each streamed block MUST have one `start`, zero or more `delta` events and one
`end`, in that order. Blocks MAY interleave; indices and kinds MUST remain stable.
Concatenated text fragments MUST equal the final block text. For a completed
`toolCall`, concatenated argument fragments MUST parse to the final `arguments`
object. A runtime MAY serialize a buffered argument object to produce fragments.
A block's `end` does not authorize execution. Error, cancellation, length or
refusal MAY terminate a provisional block without `end`; only successfully
completed blocks are required to match the terminal content indices.

## Wire Protocol

Messages use the registry's `domain.action` envelopes. Delivery and endpoint
binding belong to the projection; see the [web projection](https://github.com/napplet/naps/blob/master/projections/web.md).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `llm.select` | napplet → runtime | `id`, optional `requirements` |
| `llm.select.result` | runtime → napplet | `id`, `model` or `error` |
| `llm.describe` | napplet → runtime | `id`, `handle` |
| `llm.describe.result` | runtime → napplet | `id`, `model` or `error` |
| `llm.generate` | napplet → runtime | `id`, `request` |
| `llm.delta` | runtime → napplet | Delta fields; no separate `id` |
| `llm.generate.result` | runtime → napplet | `id`, `result` (Generation) or `error` |
| `llm.cancel` | napplet → runtime | `id`, `requestId` |
| `llm.cancel.result` | runtime → napplet | `id`, `cancelled` (boolean) or `error` |

Request IDs are non-empty text, scoped to the authenticated endpoint. Callers MUST
use a fresh ID for every request, including cancellation. A routable, validly
correlated request receives exactly one terminal result with its ID and matching
result type, unless the endpoint is destroyed. Error results contain `error`
and no success payload. An active duplicate generation ID MUST NOT start another
provider request; it receives `invalid-request`. This does not cancel the original
request, whose eventual result may still arrive. IDs are correlation identifiers,
not an exactly-once or retry guarantee.

Deltas are optional: a runtime MAY buffer the complete output and send no deltas.
When sent, they MUST be ordered and routed only to the requesting endpoint. The
terminal result is authoritative; the caller replaces provisional content with
the result's `content`, rather than appending it. No deltas may follow the
terminal result. On an error, any provisional content remains incomplete. Slow
consumers may receive fewer, larger deltas; the runtime MUST NOT silently drop
text or arguments between sequence numbers.

### Cancellation and deadlines

`cancelled: true` means the runtime found an active generation belonging to this
endpoint and terminated its delivery with a `cancelled` error. It also requests
provider cancellation, on a best-effort basis. Cancellation while awaiting
consent MUST prevent provider submission. An unknown, already terminal or other
endpoint's `requestId` returns `cancelled: false`, without revealing its existence.

If completion wins the race, its result stands and cancellation returns `false`.
If cancellation wins, the generation receives only the cancellation error; late
provider output is discarded. The cancel result and generation result need not
arrive in a particular relative order. Account changes, endpoint destruction and
revoked authorization MUST cancel affected work and invalidate its handles.

The runtime MUST bound generation time under its policy, including consent wait,
and report `timeout` when that deadline expires. It MUST settle cancellation or
timeout without waiting indefinitely for the provider. Neither proves that the
provider stopped computing or charging. A caller MUST NOT automatically retry an
ambiguous timed-out or cancelled generation. A user-initiated retry uses a new ID
and new disclosure authorization, which a live session grant may provide; it may
incur another charge.

### Examples

Choose and inspect a model:

```json
{"type":"llm.select","id":"s1","requirements":{"json":true}}
{"type":"llm.select.result","id":"s1","model":{"handle":"h1","name":"My connection","model":"chosen-model","route":"Local shell → inference provider","images":false,"json":true,"tools":true,"toolChoiceModes":["auto","none","required","tool"]}}
{"type":"llm.describe","id":"d1","handle":"h1"}
{"type":"llm.describe.result","id":"d1","model":{"handle":"h1","name":"My connection","model":"chosen-model","route":"Local shell → inference provider","images":false,"json":true,"tools":true,"toolChoiceModes":["auto","none","required","tool"]}}
```

Generate, optionally stream, then finish (a buffered runtime omits all deltas):

```json
{"type":"llm.generate","id":"g1","request":{"handle":"h1","messages":[{"role":"user","content":[{"type":"text","text":"Return a JSON object with a greeting."}]}],"format":"json","maxTokens":128,"temperature":0.7}}
{"type":"llm.delta","requestId":"g1","sequence":0,"event":"start","block":"text","blockIndex":0}
{"type":"llm.delta","requestId":"g1","sequence":1,"event":"delta","block":"text","blockIndex":0,"text":"{\"greeting\":\"Hello\"}"}
{"type":"llm.delta","requestId":"g1","sequence":2,"event":"end","block":"text","blockIndex":0}
{"type":"llm.generate.result","id":"g1","result":{"content":[{"type":"text","text":"{\"greeting\":\"Hello\"}"}],"finish":"stop","usage":{"inputTokens":12,"outputTokens":7}}}
```

Call a tool, answer it, then continue. The assistant turn carries a `toolCall`
part; the next request answers it with a `toolResult` message. Multiple parallel
calls are allowed and MUST each be answered:

```json
{"type":"llm.generate","id":"g2","request":{"handle":"h1","messages":[{"role":"user","content":[{"type":"text","text":"What is the weather in Berlin?"}]}],"tools":[{"name":"get_weather","description":"Look up current weather","parameters":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}}],"toolChoice":"auto"}}
{"type":"llm.generate.result","id":"g2","result":{"content":[{"type":"toolCall","toolCallId":"call_1","name":"get_weather","arguments":{"city":"Berlin"}}],"finish":"toolUse","usage":{"inputTokens":31,"outputTokens":12}}}
{"type":"llm.generate","id":"g3","request":{"handle":"h1","messages":[{"role":"user","content":[{"type":"text","text":"What is the weather in Berlin?"}]},{"role":"assistant","content":[{"type":"toolCall","toolCallId":"call_1","name":"get_weather","arguments":{"city":"Berlin"}}]},{"role":"toolResult","toolCallId":"call_1","toolName":"get_weather","content":[{"type":"text","text":"18°C and clear"}]}],"tools":[{"name":"get_weather","description":"Look up current weather","parameters":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}}],"toolChoice":"auto"}}
{"type":"llm.generate.result","id":"g3","result":{"content":[{"type":"text","text":"It is 18°C and clear in Berlin."}],"finish":"stop","usage":{"inputTokens":58,"outputTokens":15}}}
```

A streamed tool call identifies the call before its argument fragments:

```json
{"type":"llm.delta","requestId":"g2","sequence":0,"event":"start","block":"toolCall","blockIndex":0,"toolCallId":"call_1","name":"get_weather"}
{"type":"llm.delta","requestId":"g2","sequence":1,"event":"delta","block":"toolCall","blockIndex":0,"text":"{\"city\":\"Berlin\"}"}
{"type":"llm.delta","requestId":"g2","sequence":2,"event":"end","block":"toolCall","blockIndex":0}
```

These events precede the `g2` terminal result above; they are not a second call.

Force one named tool with `toolChoice`:

```json
{"type":"llm.generate","id":"g4","request":{"handle":"h1","messages":[{"role":"user","content":[{"type":"text","text":"Check the weather."}]}],"tools":[{"name":"get_weather","description":"Look up current weather","parameters":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}}],"toolChoice":{"type":"tool","name":"get_weather"}}}
```

Cancel a separate active generation, then try again after it has ended:

```json
{"type":"llm.cancel","id":"c1","requestId":"g2"}
{"type":"llm.generate.result","id":"g2","error":{"code":"cancelled","message":"Generation cancelled; provider charges may still apply."}}
{"type":"llm.cancel.result","id":"c1","cancelled":true}
{"type":"llm.cancel","id":"c2","requestId":"g2"}
{"type":"llm.cancel.result","id":"c2","cancelled":false}
```

Explicit failure and partial completion:

```json
{"type":"llm.generate.result","id":"g5","error":{"code":"unsupported","message":"This connection does not accept image input."}}
{"type":"llm.generate.result","id":"g6","result":{"content":[{"type":"text","text":"Incomplete answer"}],"finish":"length"}}
{"type":"llm.generate.result","id":"g7","error":{"code":"timeout","message":"Generation deadline reached; provider charges may still apply."}}
```

### Error Handling

**Error**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `code` | yes | text | Stable failure category from the table below |
| `message` | yes | text | Safe human-readable diagnostic, never a raw provider body or credential-store error |

| Code | Meaning |
|------|---------|
| `unsupported` | Domain unavailable, no compatible connection, or unsupported input/format |
| `invalid-request` | Malformed input or active duplicate generation ID |
| `selection-expired` | Unknown, revoked, changed or foreign handle; select again |
| `user-cancelled` | The user declined selection or disclosure authorization |
| `cancelled` | Generation cancelled or its endpoint/account authorization revoked |
| `timeout` | Runtime generation deadline expired; provider effects may be uncertain |
| `quota-exceeded` | Runtime size, concurrency, rate or usage policy prevented the request |
| `provider-error` | Provider/transport failure, missing completion marker, or invalid completed output |

Unknown error codes MUST be treated as failures. A declared capability does not
guarantee a request will be authorized. Runtime limits are policy, not fixed
protocol maxima; requests exceeding those limits MUST fail explicitly rather
than truncating messages, dropping images or silently changing models.

## Shell Behavior

- The runtime MUST bind each request to the authenticated endpoint and verified
  napplet identity. It MUST scope handles, results and cancellation to that endpoint
  and the current user account. Shared backend connections do not weaken this rule.
- Before submitting each generation, the runtime MUST obtain user authorization
  identifying the requesting napplet, selected route/model, data being disclosed
  (including tool definitions and tool results) and potential provider charges.
  It MUST NOT send prompts, images, tool definitions or tool results before
  that authorization. A stored API key or model selection is not authorization.
  A live session grant is that authorization for the generations it covers.
- Changing the selection while consent is pending requires fresh selection and
  consent; the runtime MUST NOT silently use the replacement connection.
- The runtime MUST keep credentials and private connection identifiers outside
  the napplet, preserve the sandbox, and perform provider communication itself.
- Domain absence MUST be discoverable without a `shell` handshake. Napplets with
  an optional `llm` integration must retain their other functionality when absent.

### Session authorization

A multi-step exchange, such as an agent loop that calls a tool and feeds its
result back, submits many generations for one user request. Authorizing each one
separately means a prompt per step. A session grant lets the user authorize the
exchange once.

A runtime that offers session grants sets `sessions: true` in the Model it
returns. A napplet MUST NOT send `authorization` unless the current Model has
`sessions: true`: a runtime without session grants treats it as an unknown
field and returns `invalid-request`. A runtime that recognizes the field but
cannot offer a grant for the handle MUST treat `session` as `generation`. When
a generation is not covered by a live grant and its request has
`authorization: "session"`, the runtime MAY ask the user for a session grant
instead of authorization for that one generation. The user may decline the grant
and still authorize the single generation, or decline both, which produces
`user-cancelled`.

A session grant:

- is bound to one handle, and through it to one endpoint, napplet identity and
  account. It ends when the handle becomes invalid, the user revokes it, or a
  runtime limit is reached. It is never transferred to another handle.
- covers a generation on its handle only when the generation's `tools` list is
  the same set of tools, with the same names, descriptions and parameters, as
  the generation that obtained the grant. Any other tool set requires fresh
  authorization. `toolChoice` may vary.
- MUST be obtained with a disclosure that names, beyond what a single
  generation's disclosure names, that later prompts, images and tool results
  will be sent without further review, the tool definitions it covers, and the
  limits that end it, such as a duration, a generation count or a charge ceiling.
- MUST stay visible to the user while it is live, and the user MUST be able to
  revoke it at any time. Revocation cancels the handle's active generations with
  `user-cancelled`; later generations need fresh authorization.

A runtime MAY end a grant earlier than its stated limits. It MUST NOT extend
one without fresh authorization. A session grant changes only how generations are
authorized: deadlines, cancellation, error handling and the rule against
automatically retrying an ambiguous generation still apply to each generation.

## Security Considerations

Prompts, images, tool definitions, tool results and output can be sensitive.
The runtime MUST NOT broadcast them,
include them in unrelated errors, or disclose them to another endpoint/account.
It MUST disclose intermediaries as well as the ultimate provider; "local model"
does not imply data stays on the user's device when the runtime is hosted.

Tool results re-enter the conversation as model input and MUST be treated as
untrusted data, including instructions embedded in them. A tool call is a request,
not authorization; the napplet must validate it and obtain any separately required
authority before execution.

A session grant sends data the user has not reviewed one by one. Tool results
can carry content from files, other services or other people. Runtimes should
keep grant limits short, express charge ceilings in the units the provider
bills, and make revocation one action away.

Model output is untrusted data, never an authorization to execute tools, sign,
pay, fetch URLs or change connections. This contract grants none of those powers.
A runtime MUST NOT execute provider-returned tool calls through this domain.
Provider-side retention and billing remain subject to the selected service's
terms; cancellation is not a deletion or refund guarantee.

## Conformance Cases

These cases define observable checks, not claims that an implementation has passed.
Tests use a controllable provider fixture and isolated authenticated endpoints.

| ID | Exercise | Required observation |
|----|----------|----------------------|
| LLM-C01 | Select, describe, generate; also omit the optional domain | Matching correlation/results; no handshake dependency; absent-domain caller fallback |
| LLM-C02 | Malformed record, base64, enum or explicit null | `invalid-request`; no provider generation |
| LLM-C03 | Unsupported images/JSON/tools/exact model or provider-specific input | Explicit `unsupported`; no silent downgrade or provider generation |
| LLM-C04 | Decline consent after selecting a valid model | `user-cancelled`; zero provider generations |
| LLM-C05 | Two endpoints reuse IDs and attempt each other's handles/cancel | Handles rejected; foreign cancellation false; no cross-delivery |
| LLM-C06 | Duplicate an active generation ID | No second provider generation; explicit failure |
| LLM-C07 | Cancel or time out during revalidation, consent or a stalled provider | Prompt settlement; no later submission from pending consent; late output suppressed |
| LLM-C08 | Switch account, revoke endpoint or change connection during consent | Cancel affected work; invalidate selections; no delivery to the new account |
| LLM-C09 | Buffered and streamed responses; cancellation/completion race | Zero-based ordered deltas if any; one terminal result; none after termination |
| LLM-C10 | Limit/refusal, missing finish marker or invalid JSON-object completion | Partial results remain partial; malformed completion is `provider-error` |
| LLM-C11 | Provider failure containing input or credentials | Safe typed error; no body, credential or other endpoint data leakage |
| LLM-C12 | Assistant turn with multiple `toolCall` parts; next request omits one result | Request rejected as `invalid-request`; no provider generation |
| LLM-C13 | `toolChoice` forcing a named tool; provider ignores it | `provider-error`; no silent substitution or plain-text fallback |
| LLM-C14 | Provider returns no tool call id | Runtime synthesizes a stable id; matching `toolResult` round-trips |
| LLM-C15 | Completed calls with different normal provider stop labels; also truncate/refuse a parallel turn after one call completes | Completed calls normalize to `toolUse`; truncation/refusal preserves finish, no executable calls |
| LLM-C16 | Completed arguments malformed or not an object; unknown tool name | `provider-error`; no raw provider body echoed |
| LLM-C17 | Replay thinking; connection requires opaque reasoning state | Thinking omitted from provider input; required unrepresentable state produces `unsupported`, no generation |
| LLM-C18 | Stream interleaved tool-call blocks | Start carries id/name; ordered lifecycle; fragments parse to final arguments; no execution from provisional blocks |
| LLM-C19 | Request an unlisted tool-choice mode | `unsupported`; no substitution or provider generation |
| LLM-C20 | Duplicate/invalid tool names, missing tools for explicit choice, unknown forced name, non-object schema, zero maxTokens | `invalid-request`; no generation |
| LLM-C21 | Misplaced/duplicate/orphaned results; history without current tools | Invalid histories rejected; completed history without tools accepted if connection supports it; no new calls |
| LLM-C22 | Image tool result; error result with no provider-native flag | Unsupported image placement rejected before submission; error flag mapped to specified text prefix |
| LLM-C23 | Consent declined with sensitive tool definitions/results | Disclosure identifies both; no provider submission |
| LLM-C24 | Obtain a session grant, then generate again with the same tools, with a different tool set, after revocation and after handle invalidation | Same tools submit without a prompt; a different tool set, revocation and invalidation each require fresh authorization; revocation cancels active work with `user-cancelled` |
| LLM-C25 | Decline a session grant but authorize the single generation; also send `session` on a handle the runtime cannot grant | One generation submitted and the next asks again; the ungrantable request gets single-generation authorization, not an error |

## Implementations

- [napplet.soy / soyLI experimental reference](https://github.com/soytec/napplet-soy/tree/0200e7c82675a781ea11292a57801f16a93d3101):
  [implementation guide](https://github.com/soytec/napplet-soy/blob/0200e7c82675a781ea11292a57801f16a93d3101/docs/LLM.md),
  [optional consumer](https://github.com/soytec/napplet-soy/blob/0200e7c82675a781ea11292a57801f16a93d3101/examples/nap-llm.ts)
  and executable conformance fixtures. It is not deployed and does not claim
  upstream acceptance or independent-shell interoperability. Provider responses
  are buffered; synthetic stream fixtures exercise delta and termination rules.

## Changelog

- `4bb12f3` - Define user-selected inference, endpoint-scoped selections, disclosure consent, generation/cancellation lifecycles, typed errors and conformance cases; add the draft registry entry and reference links.
- `8f74132` - Widen the wire subset to the common denominator of chat-completions and messages APIs: generation controls (`maxTokens`, `temperature`, `samplingParams`), tool definitions and choice, `thinking` and `toolCall` parts, the `toolResult` role with an error flag, block-aware deltas, cache and reasoning token counts without cost, and `Generation.content` as a part list; answer parallel-tool-call and provider-synthesized-id questions inline.
- `8c354ce` - Define portable tool names and schemas, supported choice modes, adjacent result placement and error mapping; remove the open sampling bag, require positive maxTokens, make thinking display-only with explicit continuation failure, define block lifecycle events and argument validation, preserve truncation/refusal, and extend consent, security and conformance coverage.
- `d41a1ba` - Add opt-in session authorization: the Model's `sessions` flag and the request's `authorization` field let one user grant cover a multi-step exchange on one handle with the same tool set, disclosed with its limits, visible while live and revocable; per-generation authorization stays the default; add security guidance and conformance cases C24 and C25.
