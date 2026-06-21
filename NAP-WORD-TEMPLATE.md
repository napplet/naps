NAP-{NAME}
==========

{Title}
-------

`draft`

**NAP ID:** NAP-{NAME}
**Domain:** `{name}`
**Depends:** {omit this line entirely if this NAP depends on no other; otherwise a bulleted list directly beneath it — see AGENTS.md → Dependencies}
- `<domain>` — wire|capability|layering · required|optional — the concrete field or method this rests on
**Web binding (NIP-5D):** `window.napplet.{name}` · `shell.supports("{name}")`

## Description

{One paragraph: what this interface provides and why a napplet needs it. State what the shell provides and what the napplet consumes.}

## API Surface

{Language-neutral operation table. These are the high-level operations napplets call. Each method corresponds to one or more wire protocol messages.}

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `method` | `param` (`tstr`) | `{ResultType}` | `{name}.action` / `{name}.action.result` |

### Schemas

Use CDDL-style notation. See AGENTS.md -> Interface schema format.

```cddl
ResultType = {
  field: tstr,
  ? optionalField: bool,
}
```

{Brief description of each method: what it does, what it returns, error conditions.}

## Wire Protocol

{name}.* messages use the NIP-5D wire format (`{ "type": "domain.action", ...payload }`).

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `{name}.action` | napplet -> shell | `id`, {fields} |
| `{name}.action.result` | shell -> napplet | `id`, {fields} |

Key design notes:
- Request/result pairs use `id` for correlation.
- Fire-and-forget messages have no `id` field.
- Shell-initiated deliveries use domain-specific routing fields.

### Examples

**{Action name}:**
```
-> { "type": "{name}.action", "id": "a1", {fields} }
<- { "type": "{name}.action.result", "id": "a1", {fields} }
```

### Error Handling

{How errors are reported -- typically an `error` field in result messages.}

## Shell Behavior

- The shell MUST {primary obligation}.
- The shell MUST respond to every request with a result message carrying the same `id`.
- The shell MAY {optional behavior}.
- The shell MAY enforce ACL checks on {name} capabilities.

## Security Considerations

{Interface-specific security requirements. The shell/runtime is the subject of MUST clauses for any crypto or trust operations -- napplets do not sign or verify.}

- {Isolation guarantees}
- {ACL considerations}
- {Trust boundaries}

## Implementations

- {links to implementations}
