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
Service checkout, payments, tool execution, embeddings, audio and video generation
are outside this proposal.

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
or silently discard supplied messages, roles or image parts. Provider-specific
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

**Model**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `handle` | yes | non-empty text | Opaque reference scoped to this authenticated endpoint and user account |
| `name` | yes | text | User-facing connection label |
| `model` | yes | non-empty text | Selected model identifier |
| `route` | yes | text | Human-readable disclosure route, including intermediaries and destination; never credentials |
| `images` | yes | boolean | Whether the connection supports image input |
| `json` | yes | boolean | Whether it supports JSON-object output |

The handle is not a URL, credential or transferable capability. The runtime MUST
NOT accept a napplet-supplied provider URL, key or identity as selection authority.
`images` does not promise every provider supports every placement of an image
within a conversation; unsupported combinations must be rejected explicitly.

**GenerationRequest**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `handle` | yes | non-empty text | A live Model handle |
| `messages` | yes | non-empty list of Message | Ordered conversation |
| `format` | no | `text` or `json` | Defaults to `text`; `json` requires an object, not an array or scalar |

**Message**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `role` | yes | `system`, `user` or `assistant` | Role of this message |
| `content` | yes | non-empty list of Part | Ordered content parts |

**Part** is one of these records:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `type` | yes | `text` or `image` | Selects the record variant |
| `text` | for `text` only | text | Text content |
| `mimeType` | for `image` only | `image/png`, `image/jpeg` or `image/webp` | Declared image encoding |
| `data` | for `image` only | non-empty text | Standard padded base64 bytes, no whitespace, data-URI prefix or remote URL |

**Generation**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `text` | yes | text | Complete accumulated output, including any streamed portion |
| `finish` | yes | `stop`, `length` or `refusal` | Normal completion, output limit reached, or provider refusal |
| `usage` | no | Usage | Provider-reported token counts, when available |

**Usage**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `inputTokens` | yes | non-negative integer | Reported input tokens |
| `outputTokens` | yes | non-negative integer | Reported output tokens |

Only `finish: "stop"` indicates normal completion. `length` and `refusal` may carry
partial or empty text; a napplet MUST NOT present them as completed output. For
`format: "json"`, a `stop` result MUST parse as a JSON object. This says nothing
about its application schema, factual accuracy or suitability. Napplets remain
responsible for those checks. Usage is not a price or settlement receipt.

**Delta**

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `requestId` | yes | non-empty text | The originating generation's `id` |
| `sequence` | yes | non-negative integer | Contiguous sequence starting at zero for this generation |
| `text` | yes | text | Text to append to the provisional display |

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
terminal result is authoritative; the caller replaces provisional text with its
`text`, rather than appending it. No deltas may follow the terminal result. On an
error, any provisional text remains incomplete. Slow consumers may receive fewer,
larger deltas; the runtime MUST NOT silently drop text between sequence numbers.

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
and new disclosure authorization; it may incur another charge.

### Examples

Choose and inspect a model:

```json
{"type":"llm.select","id":"s1","requirements":{"json":true}}
{"type":"llm.select.result","id":"s1","model":{"handle":"h1","name":"My connection","model":"chosen-model","route":"Local shell → inference provider","images":false,"json":true}}
{"type":"llm.describe","id":"d1","handle":"h1"}
{"type":"llm.describe.result","id":"d1","model":{"handle":"h1","name":"My connection","model":"chosen-model","route":"Local shell → inference provider","images":false,"json":true}}
```

Generate, optionally stream, then finish (a buffered runtime omits the delta):

```json
{"type":"llm.generate","id":"g1","request":{"handle":"h1","messages":[{"role":"user","content":[{"type":"text","text":"Return a JSON object with a greeting."}]}],"format":"json"}}
{"type":"llm.delta","requestId":"g1","sequence":0,"text":"{\"greeting\":"}
{"type":"llm.generate.result","id":"g1","result":{"text":"{\"greeting\":\"Hello\"}","finish":"stop","usage":{"inputTokens":12,"outputTokens":7}}}
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
{"type":"llm.generate.result","id":"g3","error":{"code":"unsupported","message":"This connection does not accept image input."}}
{"type":"llm.generate.result","id":"g4","result":{"text":"Incomplete answer","finish":"length"}}
{"type":"llm.generate.result","id":"g5","error":{"code":"timeout","message":"Generation deadline reached; provider charges may still apply."}}
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
  and potential provider charges. It MUST NOT send prompt/image content before
  that authorization. A stored API key or model selection is not authorization.
- Changing the selection while consent is pending requires fresh selection and
  consent; the runtime MUST NOT silently use the replacement connection.
- The runtime MUST keep credentials and private connection identifiers outside
  the napplet, preserve the sandbox, and perform provider communication itself.
- Domain absence MUST be discoverable without a `shell` handshake. Napplets with
  an optional `llm` integration must retain their other functionality when absent.

## Security Considerations

Prompts, images and output can be sensitive. The runtime MUST NOT broadcast them,
include them in unrelated errors, or disclose them to another endpoint/account.
It MUST disclose intermediaries as well as the ultimate provider; "local model"
does not imply data stays on the user's device when the runtime is hosted.

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
| LLM-C03 | Unsupported images/JSON/exact model or provider-specific input | Explicit `unsupported`; no silent downgrade or provider generation |
| LLM-C04 | Decline consent after selecting a valid model | `user-cancelled`; zero provider generations |
| LLM-C05 | Two endpoints reuse IDs and attempt each other's handles/cancel | Handles rejected; foreign cancellation false; no cross-delivery |
| LLM-C06 | Duplicate an active generation ID | No second provider generation; explicit failure |
| LLM-C07 | Cancel or time out during revalidation, consent or a stalled provider | Prompt settlement; no later submission from pending consent; late output suppressed |
| LLM-C08 | Switch account, revoke endpoint or change connection during consent | Cancel affected work; invalidate selections; no delivery to the new account |
| LLM-C09 | Buffered and streamed responses; cancellation/completion race | Zero-based ordered deltas if any; one terminal result; none after termination |
| LLM-C10 | Limit/refusal, missing finish marker or invalid JSON-object completion | Partial results remain partial; malformed completion is `provider-error` |
| LLM-C11 | Provider failure containing input or credentials | Safe typed error; no body, credential or other endpoint data leakage |

## Implementations

- [napplet.soy / soyLI experimental reference](https://github.com/soytec/napplet-soy/tree/0200e7c82675a781ea11292a57801f16a93d3101):
  [implementation guide](https://github.com/soytec/napplet-soy/blob/0200e7c82675a781ea11292a57801f16a93d3101/docs/LLM.md),
  [optional consumer](https://github.com/soytec/napplet-soy/blob/0200e7c82675a781ea11292a57801f16a93d3101/examples/nap-llm.ts)
  and executable conformance fixtures. It is not deployed and does not claim
  upstream acceptance or independent-shell interoperability. Provider responses
  are buffered; synthetic stream fixtures exercise delta and termination rules.

## Changelog

- `4bb12f3` - Define user-selected inference, endpoint-scoped selections, disclosure consent, generation/cancellation lifecycles, typed errors and conformance cases; add the draft registry entry and reference links.
