NAP-CONFIG
==========

Per-Napplet Declarative Configuration
-------------------------------------

`draft`

**NAP ID:** NAP-CONFIG
**Domain:** `config`
**Web binding (NIP-5D):** `window.napplet.config` · `shell.supports("config")`
**Parent:** NIP-5D

## Description

NAP-CONFIG provides per-napplet declarative configuration. A napplet declares a JSON Schema (draft-07 or later) describing its user-facing settings surface; the shell renders a settings UI, validates user input against the schema, persists values in a napplet-scoped store, and delivers live values back to the napplet via snapshot + push.

The shell is the sole writer of configuration values. Napplets cannot mutate configuration over the wire -- they may only read (`config.get`), subscribe to live updates (`config.subscribe`), register a schema (`config.registerSchema`), and request that the shell open its settings UI (`config.openSettings`). All values a napplet sees have been validated by the shell against the currently-registered schema, with declared defaults applied.

## API Surface

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `registerSchema` | `schema` (`ConfigSchema`), optional `version` (`uint`) | none; errors arrive through `onSchemaError` | `config.registerSchema` / `config.registerSchema.result` |
| `get` | none | `ConfigValues` | `config.get` / `config.values` |
| `subscribe` | handler for `ConfigValues` | `Subscription` handle | `config.subscribe` / `config.values` |
| `onSchemaError` | handler for `ConfigSchemaError` | `Subscription` handle | `config.schemaError` |
| `openSettings` | optional `section` (`tstr`) | none | `config.openSettings` |
| `schema` | none | `ConfigSchema` or `null` | local read of currently registered schema |

### Schemas

Primitive references:

| Name | Meaning |
|------|---------|
| `ConfigSchema` | JSON Schema draft-07 or later, restricted below. |
| `ConfigValues` | Shell-validated JSON-serializable configuration values. |

Enumerations:

| Name | Values |
|------|--------|
| `ConfigSchemaErrorCode` | `invalid-schema`, `version-conflict`, `unsupported-draft`, `ref-not-allowed`, `pattern-not-allowed`, `secret-with-default` |

`ConfigSchemaError`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `code` | `ConfigSchemaErrorCode` | yes | Machine-readable schema error code. |
| `message` | `tstr` | yes | Human-readable error message. |

**`registerSchema(schema, version?)`** -- Runtime escape hatch for declaring a configuration schema. The preferred path is manifest-declared schema at napplet build time; `registerSchema` is for napplets whose schema genuinely cannot be static. The optional `version` parameter is the `$version` migration hint. Shell responds asynchronously via a positive-ACK (`config.registerSchema.result`); errors surface via the `onSchemaError` subscription.

**`get()`** -- Returns a one-shot snapshot of the current configuration values. Returned values are always validated by the shell and defaulted per the registered schema. Resolves with a `ConfigValues` object.

**`subscribe(callback)`** -- Subscribes to live configuration updates. The shell MUST deliver an immediate initial `config.values` push on subscription (snapshot delivery), followed by pushes whenever the shell's settings UI commits a change. Returns a `Subscription` with a `close()` method that detaches the callback. The reference shim implementation SHOULD ref-count local subscribers and only send a wire-level `config.unsubscribe` when the last local subscriber detaches.

**`onSchemaError(callback)`** -- Subscribes to schema-registration errors. The shell pushes `config.schemaError` messages when it rejects a `registerSchema` call or cannot parse a manifest-declared schema. Returns a `Subscription` with `close()`.

**`openSettings(options?)`** -- Requests that the shell open its settings UI for this napplet. The optional `options.section` string deep-links the UI to a named section; the section MUST be declared via the `x-napplet-section` extension somewhere in the registered schema. Fire-and-forget -- the shell decides how to render the UI (modal, panel, new tab) and whether to honor the request.

**`schema`** -- Read-only accessor for the currently-registered schema. Returns the schema last registered (via manifest or `registerSchema`), or `null` if none has been declared. Useful for napplets that render capability-dependent UI based on which settings exist.

## Wire Protocol

`config.*` messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `config.registerSchema` | napplet -> shell | `id`, `schema`, `version?` |
| `config.registerSchema.result` | shell -> napplet | `id`, `ok`, `error?`, `code?` |
| `config.get` | napplet -> shell | `id` |
| `config.subscribe` | napplet -> shell | (none) |
| `config.unsubscribe` | napplet -> shell | (none) |
| `config.openSettings` | napplet -> shell | `section?` |
| `config.values` | shell -> napplet | `id?`, `values` |
| `config.schemaError` | shell -> napplet | `error`, `code` |

Key design notes:
- All correlation uses `id`.
- `config.values` is dual-use: with `id` it answers `config.get`; without `id` it is a subscription push.
- `config.registerSchema` uses positive-ACK -- every call receives a `config.registerSchema.result` with `ok: true`, or `ok: false` accompanied by `code` and `error` describing the rejection reason. It is never error-only.
- `config.subscribe` MUST produce an immediate initial `config.values` push (snapshot delivery) before any value-change pushes.
- `config.unsubscribe` is the clean-teardown path; iframe unmount also implicitly unsubscribes the source.

### Examples

**Register schema (success):**
```
-> { "type": "config.registerSchema", "id": "r1", "schema": { "$schema": "http://json-schema.org/draft-07/schema#", "type": "object", "properties": { "theme": { "type": "string", "enum": ["light", "dark"], "default": "dark" } } }, "version": 1 }
<- { "type": "config.registerSchema.result", "id": "r1", "ok": true }
```

**Register schema (rejection):**
```
-> { "type": "config.registerSchema", "id": "r2", "schema": { "type": "object", "properties": { "username": { "type": "string", "pattern": "^[a-z]+$" } } } }
<- { "type": "config.registerSchema.result", "id": "r2", "ok": false, "code": "pattern-not-allowed", "error": "JSON Schema `pattern` keyword is not permitted in the Core Subset" }
```

**Get values:**
```
-> { "type": "config.get", "id": "g1" }
<- { "type": "config.values", "id": "g1", "values": { "theme": "dark" } }
```

**Subscribe + initial snapshot:**
```
-> { "type": "config.subscribe" }
<- { "type": "config.values", "values": { "theme": "dark" } }
```

**Push update after user changes a value:**
```
<- { "type": "config.values", "values": { "theme": "light" } }
```

**Unsubscribe:**
```
-> { "type": "config.unsubscribe" }
```

**Open settings with section deep-link:**
```
-> { "type": "config.openSettings", "section": "notifications" }
```

**Schema error push (background rejection):**
```
<- { "type": "config.schemaError", "code": "invalid-schema", "error": "schema root must be of type `object`" }
```

## Schema Contract

NAP-CONFIG is a JSON-Schema-driven protocol, but it is NOT all of JSON Schema. Shells and napplets MUST agree on a bounded subset so that the same schema renders and validates consistently across every conformant shell. This section defines that subset (the Core Subset), the standardized annotations shells MAY act on (Potentialities), the migration-version signal (`$version`), and the set of features explicitly excluded from v1. Any schema feature not named in Core Subset or Potentialities is out of scope for NAP-CONFIG v1.

### Core Subset

Every conformant shell MUST accept and render schemas that use only the types, keywords, and constraints enumerated below. Napplets MUST NOT rely on any feature outside this subset for correct behavior.

**Supported types (Core Subset):**

- `object` -- top-level only. The schema root MUST be `{ "type": "object", "properties": { ... } }`.
- `string`
- `number`
- `integer`
- `boolean`
- `array` of primitives -- homogeneous only. Tuple-typed arrays (`items: [schemaA, schemaB]`) are NOT in the Core Subset.
- Nested `object` -- bounded depth; see Depth Limit below. Recommended maximum depth is 4.

**Supported keywords (Core Subset):**

- `type`
- `properties`
- `required` (array form)
- `items` (homogeneous schema only)
- `additionalProperties` (see override below)
- `default`
- `title`
- `description`
- `enum`
- `enumDescriptions` -- parallel-array hint whose entries correspond positionally to `enum` entries; shells SHOULD render each description alongside its enum value.

**Supported constraints (Core Subset):**

- `minimum`, `maximum` -- apply to `number` and `integer` values
- `minLength`, `maxLength` -- apply to `string` values
- `minItems`, `maxItems` -- apply to `array` values

**`additionalProperties` override:**

Shells MUST treat `additionalProperties` as `false` at the top-level schema object when the napplet does not specify it explicitly. This overrides the JSON Schema draft-07 default of `true` for NAP-CONFIG scope. Rationale: persisted user settings must not accrete unknown properties silently; schema drift between napplet versions should drop orphaned keys, not preserve them.

**Default-resolution rule (deterministic):**

For every property at `config.values` delivery time, the shell MUST resolve its value in the following order:

1. If a persisted value exists AND validates against the current schema, use it.
2. Else if the property has its own `default`, use that.
3. Else if an ancestor `default` object supplies a value for this property, use the ancestor's value for this property.
4. Else the property is `undefined` in the delivered `config.values`.

### Standardized Extensions (Potentialities)

These are opt-in annotations napplets MAY declare on schema properties. Shells MAY act on them; shells MAY ignore them. Shells that do not recognize an extension MUST treat it as opaque metadata and MUST NOT reject the schema on that basis.

| Keyword | Applies to | Shell MAY |
|---------|------------|-----------|
| `x-napplet-secret` (boolean) | any `string` property | mask input, suppress console/log output, decline to render a default value, route the stored value through a platform keychain or encrypted store |
| `x-napplet-section` (string) | any property | group the property under the declared section heading in the settings UI |
| `x-napplet-order` (non-negative number) | any property | sort properties within a section in ascending `x-napplet-order`; ties broken alphabetically by property key |
| `deprecationMessage` (string) | any property | render the message next to the field as a deprecation notice |
| `markdownDescription` (string) | any property | render the value as markdown; shells MAY fall back to plain text if markdown rendering is not available |

### `$version` Potentiality

A schema MAY carry a top-level `$version` integer field. Shells MAY use `$version` to drive cross-hash migration when the napplet's `aggregateHash` changes; shells MAY ignore it entirely and treat each hash as a fresh scope. NAP-CONFIG does NOT prescribe a migration strategy -- migration is a shell concern, and `$version` is a signal, not a contract.

Napplets MUST NOT receive values that violate the currently-registered schema at `config.values` delivery time, regardless of how the shell reconciles older persisted values. Any clamping, dropping, or user-prompted merging happens entirely shell-side; the napplet only ever sees post-migration values.

### Exclusions (Not in v1 Core Subset)

The following are explicitly excluded from NAP-CONFIG v1:

- **`pattern` keyword** -- excluded due to ReDoS (Regular Expression Denial of Service) risk. JavaScript's built-in `RegExp` engine is backtracking-based; a napplet-declared pattern such as `^(a|a)*$` against a 31-character input can consume tens of seconds of CPU on the shell's main thread. See CVE-2025-69873 (ajv validator) for a concrete real-world precedent of this class of vulnerability in JSON Schema validators. Readmission requires a shell-safe regex strategy (Worker + timeout, RE2-WASM, or equivalent) to be specified in a future revision of this NAP.
- **`$ref`** -- ALL forms forbidden, including same-document (`#/definitions/foo`), external URL (`https://...`), filesystem (`file://`), and cross-document relative references. Rationale: schemas are self-contained; `$ref` resolution is a security surface (exfiltration, DoS via slow URLs, TOCTOU) and a renderer-complexity explosion.
- **`definitions` / `$defs`** -- follows from the `$ref` exclusion; with no `$ref` there is nothing to reference.
- **`oneOf`, `anyOf`, `allOf`, `not`** -- combinatorial schemas are out of Core Subset v1. Shells MAY support them as an Extended Subset, but napplets MUST NOT rely on them for conformant behavior.
- **`if` / `then` / `else`** -- conditional schemas are out of scope for v1.
- **`patternProperties`, `propertyNames`, `dependencies`, `dependentSchemas`, `unevaluatedProperties`** -- draft 2019-09+ features out of scope.
- **Tuple-typed arrays** (`items: [schemaA, schemaB]`) -- only homogeneous arrays are in the Core Subset.
- **Recursive `$ref` of any kind** -- forbidden by the `$ref` exclusion, called out explicitly here so there is no ambiguity.
- **Expression-valued defaults** (`"$now()"`, `"$random()"`, etc.) -- `default` values are literal JSON per JSON Schema semantics. Napplets needing runtime-generated defaults MUST handle the `undefined` case in their own code.
- **`x-napplet-secret: true` combined with a declared `default`** -- a secret with a hardcoded default is not a secret. Shells MUST reject any schema that declares both on the same property with `code: "secret-with-default"`.

### Format as Hint Only

The JSON Schema `format` keyword (`email`, `uri`, `date`, `date-time`, `color`, `ipv4`, `ipv6`, and others) is annotative in NAP-CONFIG: shells MAY use `format` to pick an input widget but MUST NOT reject a value solely because it fails the `format`. Napplets requiring strict validation MUST use `enum`, `minLength`, `maxLength`, `minimum`, or `maximum` -- NOT `format`.

### Depth Limit

Shells MUST reject schemas nesting `object` properties more than 4 levels deep at `config.registerSchema` time, returning `config.registerSchema.result` with `ok: false` and `code: "schema-too-deep"`. Napplets MUST NOT assume arbitrary nesting is supported.

## Shell Guarantees

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY in this document are to be interpreted as described in RFC 2119.

### MUST

| Behavior | Details |
|----------|---------|
| Validate every value before delivery | Every `config.values` delivery MUST contain only values that validate against the currently-registered schema. Invalid values MUST NOT be delivered. |
| Apply declared defaults | Missing properties MUST be populated from `default` per the deterministic default-resolution rule in Schema Contract. |
| Scope storage by `(dTag, aggregateHash)` | Persisted values MUST be keyed on the napplet's `(dTag, aggregateHash)` identity per NIP-5D. A napplet cannot read or write outside its own scope. |
| Be the sole writer | The shell MUST be the only entity that writes values. No napplet->shell wire message mutates persisted values. |
| Mask `x-napplet-secret: true` fields (Tier 0) | Shells MUST treat properties marked `x-napplet-secret: true` as secrets at the minimum Tier 0 level: mask input in the settings UI, do NOT include the default (napplets MUST NOT set `default` on such fields), and do NOT deliver the property in `config.values` if the value has never been explicitly set by the user. |
| Reject schemas exceeding the depth limit | Return `config.registerSchema.result` with `ok: false, code: "schema-too-deep"`. |
| Reject schemas with forbidden features | `$ref`, `pattern` (in v1), `definitions`, combinatorial schemas (`oneOf`/`anyOf`/`allOf`/`not`), tuple-typed arrays, and conditional schemas (`if`/`then`/`else`) MUST cause `config.registerSchema.result` with `ok: false` and an appropriate code. |
| Reject `x-napplet-secret: true` coexisting with `default` | Return `config.registerSchema.result` with `ok: false, code: "secret-with-default"`. |
| Produce initial snapshot after registerSchema is applied | `config.subscribe` MUST NOT deliver its first `config.values` until the most recent `config.registerSchema` from the same source has been fully applied (defaults resolved, storage scoped). If subscribe arrives before any schema has been registered (no manifest, no runtime registerSchema), shells MUST emit `config.schemaError` with `code: "no-schema"`. |
| Drop orphaned properties | Persisted values for properties not in the currently-registered schema MUST NOT be delivered. For `x-napplet-secret: true` properties, orphaned values MUST be deleted immediately on schema change; non-secret orphans MAY be retained briefly (grace period) but MUST NOT be delivered. |

### SHOULD

| Behavior | Details |
|----------|---------|
| Group by `x-napplet-section` | Shells SHOULD render properties grouped under the declared section heading. |
| Sort within a section by `x-napplet-order` | Ascending; ties broken by property-key alphabetical order. Properties without `x-napplet-order` come after, alphabetically sorted. |
| Surface `deprecationMessage` | Display next to affected fields so users understand impending removal. |
| Render `markdownDescription` as markdown | Falling back to plain text. Never render as HTML. |
| Coalesce rapid value changes | Debounce user-side edits (recommend ~100ms) so napplets receive terminal values, not every keystroke. |
| Drop non-secret orphans after a grace period | One session is the recommended grace window. |
| Honor `config.openSettings` for focused napplets | Only when the calling napplet has (or would have) user focus. Rate-limit repeated requests. |

### MAY

| Behavior | Details |
|----------|---------|
| Tier 2+ secret handling | Shells MAY store `x-napplet-secret: true` values encrypted at rest, in an OS keychain, or in another hardened backend. NOT required -- a Tier 0 in-process store is conformant. |
| Render richer `format` widgets | For `format: "email" \| "uri" \| "date" \| "date-time" \| "color" \| "ipv4" \| "ipv6"`, shells MAY render a specialized widget; MUST NOT fail validation on `format` alone. |
| Render nested objects beyond one level | Up to the depth limit (4). JSON fallback UI is acceptable for deep nesting. |
| Back NAP-CONFIG storage with NAP-STORAGE internally | An implementation choice. The NAP-CONFIG wire surface does not depend on NAP-STORAGE. |
| Use `$version` for cross-hash migration | Shells MAY act on `$version` to migrate values across `aggregateHash` changes; MAY ignore it and treat each hash as a fresh scope. |
| Retain a "graveyard" of orphaned non-secret values | For a single session in case the napplet author rolls back a schema change. Graveyard values MUST NOT be delivered. |
| Emit `config.settingsOpened` after openSettings | Optional ack so napplets can fall back when UI is unavailable. Napplets MUST NOT rely on this message. |

## Anti-Features

The following ideas are deliberately excluded from NAP-CONFIG and MUST NOT appear in conformant implementations. They are listed here so that proposals to add them surface the prior rationale rather than re-litigating.

- **`config.set` (napplet->shell write message)** -- violates the sole-writer invariant; defeats shell-UI validation and consent UX; enables silent settings override by compromised napplets.
- **`$ref` / `definitions` / `$defs` in schemas** -- schemas are self-contained; $ref resolution is a security surface (exfiltration, DoS, TOCTOU) and a renderer complexity explosion. Bundle refs at build time.
- **`pattern` keyword in Core Subset v1** -- ReDoS exposure per CVE-2025-69873 class. Readmission requires a shell-safe regex strategy in a future revision.
- **napplet-rendered settings iframe** -- defeats consistent UX across napplets and places the settings surface inside the same sandbox that renders the napplet.
- **napplet-supplied validation code** -- the shell executing napplet code is a sandbox violation and breaks validate-before-delivery.
- **napplet-dictated UI widget hints (`ui:widget`)** -- turns NAP-CONFIG into a UI framework spec.
- **Migration-as-napplet-code** -- shell is the risk carrier; `$version` as a signal suffices.
- **Cross-napplet config sharing** -- violates per-napplet isolation (security property).
- **Live two-way binding / streaming uncommitted keystrokes** -- creates validation flicker and storage churn; shells debounce and push on commit instead.
- **JSON Schema draft 2020-12 conditional features** (`if`/`then`/`else`, `unevaluatedProperties`, `dependentSchemas`, `$dynamicRef`) -- enormous validator/UI complexity, no precedent uses them for extension config.
- **OS-keychain secret handling as MUST** -- unimplementable in browser-only shells; kept as MAY.

## Security Considerations

### Source-identity scope binding

Storage scope is derived from the napplet's `MessageEvent.source` at iframe creation per NIP-5D. The shell resolves `(dTag, aggregateHash)` from that source -- never from napplet-supplied payload. Consequence: NAP-CONFIG wire messages MUST NOT carry `dTag` or `aggregateHash` fields from the napplet side. A napplet cannot read, register, or mutate another napplet's configuration scope.

### Cleartext secrets over postMessage

`config.values` payloads are transmitted cleartext over `postMessage` to the napplet iframe, including fields marked `x-napplet-secret: true`. This is a property of the sandbox model (no secure key-exchange channel exists between a sandboxed iframe and its host that isn't itself `postMessage`). Implementers MUST be aware that:

- Browser extensions with script access can observe the event
- Developer-tools inspection exposes the payload
- Crash reporters and analytics may capture it

Shells SHOULD suppress console logging of `config.values` deliveries containing `x-napplet-secret: true` fields. Napplets SHOULD read secrets on demand and avoid holding them in long-lived memory. NAP-CONFIG does NOT claim secrets are "secure" -- the honest framing is that the shell decides when to deliver.

### `additionalProperties: false` override

JSON Schema draft-07 defaults `additionalProperties` to `true`. NAP-CONFIG overrides this: the top-level schema object is treated as if `additionalProperties: false` when the napplet does not specify otherwise. This prevents silent persisted-data accretion and makes schema evolution deterministic (orphaned properties drop out of delivery). Napplets SHOULD still declare `additionalProperties: false` explicitly for robustness across shell implementations.

### External `$ref` forbidden

All `$ref` forms are forbidden in v1 (see Schema Contract Exclusions). The ban is particularly important for external references (`http://`, `https://`, `file://`, `ftp://`): they enable data exfiltration (the request URL can carry stolen values), denial-of-service (slow or hanging URLs stall validation), and privacy leaks (every validation pings the remote). Shells MUST reject any schema containing `$ref` at `config.registerSchema` time.

## Error Envelopes

Every napplet->shell request type either returns a correlated result message with an `ok: boolean` field (positive ACK) or fires a matching push error. Result messages follow the NIP-5D envelope format `{ "type": "domain.action.result", "id", "ok", "code"?, "error"? }`. Error `code` values are drawn from the catalogue below; `error` is a human-readable explanation.

### Error codes

| Code | Emitted by | Meaning |
|------|------------|---------|
| `invalid-schema` | `config.registerSchema.result`, `config.schemaError` | Napplet-supplied schema is not parseable as JSON Schema draft-07 or later. |
| `unsupported-draft` | `config.registerSchema.result`, `config.schemaError` | Schema declares a JSON Schema draft the shell does not support. |
| `ref-not-allowed` | `config.registerSchema.result`, `config.schemaError` | Schema uses `$ref`, `definitions`, or `$defs`. |
| `pattern-not-allowed` | `config.registerSchema.result`, `config.schemaError` | Schema uses the `pattern` keyword (excluded in Core Subset v1). |
| `secret-with-default` | `config.registerSchema.result`, `config.schemaError` | A property marked `x-napplet-secret: true` declares a `default`. |
| `schema-too-deep` | `config.registerSchema.result`, `config.schemaError` | Schema nesting exceeds the depth limit (4). |
| `version-conflict` | `config.registerSchema.result`, `config.schemaError` | Re-registration declares a `$version` the shell does not accept. |
| `no-schema` | `config.schemaError` | `config.subscribe` or `config.get` received before any schema has been registered for this source (no manifest, no runtime registerSchema). |
| `unknown-section` | (see below -- non-normative) | `config.openSettings` referenced a section that does not exist on the currently-registered schema. SHOULD NOT be surfaced as an error envelope; shells SHOULD silently ignore (see Shell Guarantees SHOULD row for openSettings). |

### Error cases

**Malformed schema at registerSchema time.** A schema that is not parseable, declares an unsupported draft, uses forbidden features, or violates the depth limit is rejected. The shell emits `config.registerSchema.result` with `ok: false` and the appropriate `code`. Napplets that registered via `window.napplet.config.registerSchema()` observe the failure via the `onSchemaError` listener; the shim correlates the `.result` via `id`.

**Undeclared section in openSettings.** When `config.openSettings({ section })` references a `section` not declared by any property's `x-napplet-section` in the currently-registered schema, shells SHOULD silently ignore the request (opening the settings UI to its default entry is acceptable). Shells MUST NOT return a wire error for this case -- that would leak shell internals. The section string is case-sensitive and compared as an opaque identifier.

**Subscribe-before-schema.** `config.subscribe` or `config.get` that arrives before any schema has been registered (neither a manifest-declared schema nor a runtime `config.registerSchema`) receives a `config.schemaError` push with `code: "no-schema"`. No `config.values` is emitted until a valid schema has been registered.

## Implementations

- (none yet)
