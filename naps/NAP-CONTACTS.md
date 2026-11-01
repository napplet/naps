NAP-CONTACTS
============

Shell-Mediated Contact Selection and Queries
-------------------------------------------

`draft`

**NAP ID:** NAP-CONTACTS
**Domain:** `contacts`
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.contacts`; domain presence signals availability. This is projection guidance.

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are interpreted as
described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.html).

## Description

NAP-CONTACTS lets a napplet request contacts from the shell. The napplet designs
recipient selection, address-book views, and autocomplete. The shell mediates
which contact data the user discloses. The runtime holds the contact sources,
credentials, and query state. A composer can request a trusted picker without
receiving permission to search or enumerate the user's address book.

### Boundary

**Selecting a recipient must not silently grant address-book access.** Permission
to read mail, send a message, or query another domain is not a contact grant.
The shell MUST distinguish per-request selection from continuing browse access.

| Concern | Owner |
|---------|-------|
| Contact sources, accounts, synchronization and credentials | Runtime, configured by the user through the shell |
| Picker UI, disclosure confirmation, browse grants and their source/kind scope | Shell |
| Normalized contact records, validation, filtering and pagination | Runtime |
| Recipient fields, address-book presentation, autocomplete and messaging choices | Napplet |

This draft provides read-only email endpoints and optional Nostr public-key
endpoints. A contact endpoint is an address at which a person or organization
may be reached; it is not an authenticated identity or an authorization token.
Contact editing, raw address-book files, provider-specific queries, phone
endpoints, and executing a messaging action are outside this surface.

No other NAP domain is required. This domain neither provisions email accounts
nor borrows another domain's grants. Source selection and synchronization remain
runtime internals; no server, credential, or source URL is accepted from a napplet.

## API Surface

A runtime exposing `contacts` MUST implement every operation below and support
the `email` endpoint kind. It MAY also support `nostr`, reported by `info`.
Availability is projection-defined. Manifests declare the bare domain `contacts`;
domain declarations and availability are not grants.

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `info` | none | `ContactsInfo` | `contacts.info` / `contacts.info.result` |
| `pick` | `kind` (text), optional `multiple` (boolean) | `ContactSelection` | `contacts.pick` / `contacts.pick.result` |
| `query` | `kind` (text), optional `search` (text), optional `cursor` (text), optional `limit` (integer) | `ContactPage` | `contacts.query` / `contacts.query.result` |
| `get` | `kind` (text), `contact` (text) | `Contact` | `contacts.get` / `contacts.get.result` |

`info` describes the contract supported by this runtime, not its contact data or
the caller's grants. `pick` discloses only explicit selections made in trusted
shell UI for that request. `query` and `get` require a separate current browse
grant covering the requested endpoint kind and shell-authorized contact sources.
Neither a previous selection nor a contact handle can satisfy that requirement.

### Common values

| Value | Definition |
|-------|------------|
| Endpoint kind | `"email"` or `"nostr"`. A defined but unimplemented kind returns `unsupported`; any other request value is invalid. |
| Handle | Nonempty opaque text issued by the runtime. Contact and cursor handles are scoped to the shell user and verified napplet identity. They are not paths, URLs, or permission tokens. |
| Control characters | Unicode U+0000–U+001F and U+007F–U+009F. |
| Optional fields | Omission has the stated default. `null` is invalid except for a page's `next` field. Empty text and lists are valid only as specified. |

Requests MUST contain only their defined fields, including nested records;
the runtime MUST reject extra fields with `invalid-request`. Napplets MUST
ignore unrecognized result fields added in later revisions.

### ContactsInfo

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `kinds` | yes | list of text | Unique supported endpoint kinds; contains `"email"` and optionally `"nostr"`. No contact counts or source identifiers. |
| `maxPageSize` | yes | integer | Positive maximum `query.limit`; also the default for a first page. |
| `maxSelections` | yes | integer | Positive maximum number of endpoints returned by one multiple-selection picker. |

Limits are runtime-selected and MAY change. A request exceeding a current limit
returns `too-large`. The runtime MAY also enforce finite rate and resource limits.

### ContactEndpoint

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `kind` | yes | "email" or "nostr" | Address representation. |
| `value` | yes | text | Valid endpoint of this kind, as defined below. |
| `label` | no | text | Untrusted display label, such as `Work`; may be empty. |

| Kind | Value |
|------|-------|
| `email` | One ASCII mailbox addr-spec under [RFC 5322](https://www.rfc-editor.org/rfc/rfc5322.html), with an ASCII domain and no display-name syntax, groups, comments, or control characters. Internationalized local parts are outside this draft. |
| `nostr` | One public key encoded as exactly 64 lowercase hexadecimal characters under [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md). No private key, URI, relay list, or signing authority. |

The runtime MUST validate endpoints before returning them. Invalid or
unrepresentable source endpoints MUST NOT be disclosed as valid values. Names
and labels MUST contain no control characters; the runtime MUST sanitize source
display text to satisfy that constraint. Display text is not markup. An endpoint
value MUST NOT be repaired by interpreting it as a command, URI, or credential.

### ContactSelection and ContactChoice

| ContactSelection field | Required | Type | Notes |
|------------------------|----------|------|-------|
| `choices` | yes | list of `ContactChoice` | Only confirmed selections; empty on cancellation or dismissal. |

| ContactChoice field | Required | Type | Notes |
|---------------------|----------|------|-------|
| `name` | yes | text | Selected contact's display name; may be empty. |
| `endpoint` | yes | `ContactEndpoint` | One explicitly selected endpoint of the requested kind. |

`multiple` defaults to false. A single-selection picker returns at most one
choice; a multiple-selection picker returns at most `maxSelections`. The shell
MUST confirm the exact names and endpoints being disclosed, including labels
when present. Selecting a contact with several addresses MUST NOT disclose its
other addresses. Returned choices contain no persistent contact handles, notes,
photos, hidden fields, or source metadata. They grant no later record access.

The shell MAY refuse to open the picker under focus, interaction, or rate policy.
Each accepted picker request requires fresh user confirmation; the runtime
MUST NOT reuse an earlier selection as automatic confirmation for a later call.
Cancellation, dismissal, or expiry of an open picker returns an empty selection.
The runtime MUST NOT return partial unconfirmed choices. Concurrent picker
requests remain separate or return `busy`; they MUST NOT share a confirmation.

### Contact and ContactPage

| Contact field | Required | Type | Notes |
|---------------|----------|------|-------|
| `contact` | yes | text | Caller-scoped handle for this source entry. Stable across endpoint reloads while the entry and verified napplet identity remain the same. |
| `name` | yes | text | Untrusted display name; may be empty. |
| `endpoints` | yes | list of `ContactEndpoint` | Nonempty; only valid endpoints of the requested kind from authorized sources. |

A contact handle MUST NOT be reassigned to a different source entry. Removed
and out-of-scope entries return `not-found`. Duplicate names or endpoint values
MUST NOT merge source entries or establish identity. Handles MUST NOT be used
to associate records across napplets or shell users.

| ContactPage field | Required | Type | Notes |
|-------------------|----------|------|-------|
| `contacts` | yes | list of `Contact` | At most `limit` records. May be empty. |
| `next` | yes | text or null | Cursor for the next page, or `null` at the end. |

`query` applies the current browse grant's source and kind scope before
matching. Entries with no valid authorized endpoint of the requested kind are
excluded. Optional `search` defaults to empty text, which matches every such
entry. A nonempty search matches a case-sensitive substring of the disclosed
name or an endpoint value of the requested kind. It MUST NOT search hidden
fields or other endpoint kinds. Comparison uses the returned Unicode code
points without normalization or case folding. Other kinds MUST NOT appear in
the returned records, even when the same contact contains them.

A first page captures an immutable snapshot, sorted by name and then contact
handle in ascending Unicode code-point order. New or changed source records
appear in a new first-page query. The cursor is bound to the caller scope,
source grant scope, kind, search, and original limit. Continuations MUST still
supply the same `kind`; explicitly supplied search or limit MUST match the
original values, while omitted values inherit them. A mismatch is invalid.
`limit` MUST be positive; exceeding `maxPageSize` returns `too-large`.

The runtime MUST recheck browse authority on each continuation. Expiry or a
change to the authorized source scope returns `stale`; loss of the required
browse grant returns `forbidden`. Callers restart pagination without a cursor.
`get` returns a current record under the same source and kind restrictions;
it MAY differ from an earlier snapshot. An entry with no valid authorized
endpoint of the requested kind returns `not-found`. Large records or pages return
`too-large`, rather than silently truncate endpoints or skip matching records.

## Wire Protocol

Messages use the projection's carrier. These logical fields define the contract;
no host transport is required here.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `contacts.info` | napplet → runtime | `id` |
| `contacts.info.result` | runtime → napplet | `id`, `value` or `error` |
| `contacts.pick` | napplet → runtime | `id`, `kind`, `multiple?` |
| `contacts.pick.result` | runtime → napplet | `id`, `value` or `error` |
| `contacts.query` | napplet → runtime | `id`, `kind`, `search?`, `cursor?`, `limit?` |
| `contacts.query.result` | runtime → napplet | `id`, `value` or `error` |
| `contacts.get` | napplet → runtime | `id`, `kind`, `contact` |
| `contacts.get.result` | runtime → napplet | `id`, `value` or `error` |

`id` is nonempty text, unique among the calling endpoint's outstanding requests.
For each authenticated, routable request with a valid correlation id, the
runtime MUST return exactly one corresponding result with the same `id` to
that endpoint. Success contains `value` of the API row's result type and no
`error`; failure contains `error` and no `value`. Correlation is endpoint-scoped.
There are no unsolicited deliveries or subscriptions.

### Error Handling

`ContactsError`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `code` | yes | text | One code below; callers branch on this value. |
| `message` | yes | text | Safe human-readable explanation. |

| Code | Meaning |
|------|---------|
| `invalid-request` | Invalid field, kind, type, range, or continuation parameters. |
| `unsupported` | Requested defined endpoint kind is not implemented. |
| `forbidden` | Shell policy denies opening the picker or the required browse grant. |
| `not-found` | Contact or cursor is absent in the caller's authorized scope. |
| `stale` | Cursor expired or its authorized source scope changed. |
| `too-large` | Request, record, selection, or page exceeds a runtime limit. |
| `busy` | Finite work, picker, rate, or storage capacity is exhausted. |
| `unavailable` | Required contact data or source is unavailable. |

The runtime MUST validate and authorize before querying or disclosing contact
data. Without a browse grant, `query` and `get` return `forbidden` regardless of
whether any contact or supplied handle exists. With that grant, missing and
out-of-scope contact and cursor handles both return `not-found`. Error text
MUST NOT reveal hidden source names, contact details, login identifiers, or
credentials. `info` MUST NOT reveal whether the address book is empty.

A repeated query is a new read unless it carries a valid snapshot cursor.
Retrying `pick` with a new correlation id is a new picker request, not a replay
of an earlier disclosure. A lost result does not grant browse access.

### Examples

**Discover the available representations and limits:**

```text
→ { "type": "contacts.info", "id": "i1" }
← { "type": "contacts.info.result", "id": "i1", "value": { "kinds": ["email", "nostr"], "maxPageSize": 100, "maxSelections": 20 } }
```

**A composer receives only the email endpoint selected by the user:**

```text
→ { "type": "contacts.pick", "id": "p1", "kind": "email" }
← { "type": "contacts.pick.result", "id": "p1", "value": { "choices": [{ "name": "Alice", "endpoint": { "kind": "email", "value": "alice@example.org", "label": "Work" } }] } }
```

**Selection does not grant enumeration; cancellation discloses nothing:**

```text
→ { "type": "contacts.query", "id": "q0", "kind": "email" }
← { "type": "contacts.query.result", "id": "q0", "error": { "code": "forbidden", "message": "Contact browsing is not authorized." } }
→ { "type": "contacts.pick", "id": "p2", "kind": "email", "multiple": true }
← { "type": "contacts.pick.result", "id": "p2", "value": { "choices": [] } }
```

**A separately granted browser queries and reads current contact data:**

```text
→ { "type": "contacts.query", "id": "q1", "kind": "email", "search": "alice", "limit": 1 }
← { "type": "contacts.query.result", "id": "q1", "value": { "contacts": [{ "contact": "c1", "name": "Alice", "endpoints": [{ "kind": "email", "value": "alice@example.org", "label": "Work" }] }], "next": null } }
→ { "type": "contacts.get", "id": "g1", "kind": "email", "contact": "c1" }
← { "type": "contacts.get.result", "id": "g1", "value": { "contact": "c1", "name": "Alice", "endpoints": [{ "kind": "email", "value": "alice@example.org", "label": "Work" }] } }
```

**Invalid range and a source-scope change:**

The final request uses a previously issued cursor whose source grant scope has
changed while the caller retains browse permission for the requested kind.

```text
→ { "type": "contacts.query", "id": "q2", "kind": "email", "limit": 0 }
← { "type": "contacts.query.result", "id": "q2", "error": { "code": "invalid-request", "message": "Limit must be positive." } }
→ { "type": "contacts.query", "id": "q3", "kind": "email", "cursor": "expired-snapshot" }
← { "type": "contacts.query.result", "id": "q3", "error": { "code": "stale", "message": "Restart the contact query." } }
```

A second endpoint may use the same correlation id without receiving the first
endpoint's result. Another napplet copying `c1` still needs its own browse grant
and its own handles. An unsupported `nostr` request on an email-only runtime
returns `unsupported`; callers can use `info` before offering that feature.

## Shell Behavior

- The runtime MUST bind every request to the authenticated endpoint's verified
  manifest and artifact identity, as defined by the projection, plus the shell
  user. Napplet-supplied identity claims do not establish that scope. New
  artifacts require shell policy evaluation.
- The shell MUST provide trusted picker UI and disclose the requesting napplet
  and exact data being released. It MUST NOT delegate that confirmation to
  napplet-controlled markup. Querying visible picker results on the napplet's
  behalf before confirmation is forbidden.
- The shell MUST keep picker permission and browse permission separate. Browse
  grants MUST state their contact-source and endpoint-kind scope. The shell
  MUST disclose that browse access permits contact data to be read by the
  napplet, not merely displayed, and allow the user to revoke it.
- The runtime MUST recheck authority and the current shell user before releasing
  any result containing contact data. Revocation stops later disclosures and
  invalidates affected cursors. Revocation cannot recall previously disclosed
  values from a napplet.
- Endpoint teardown or a shell-user switch MUST close that endpoint's open
  pickers and discard unreturned selections. Late confirmations MUST NOT reach
  another endpoint, another napplet, or the new shell user. The runtime MUST
  bound picker duration, cursor lifetime, source work, record sizes, and request
  duration; expired open pickers follow cancellation semantics.
- The runtime MAY use local stores or trusted configured providers. The shell
  MUST disclose providers that handle contact data or search text when granting
  access. Query semantics and disclosure scope remain the same across backends.

## Security Considerations

- Contact contents are untrusted data. Names and labels MUST NOT execute markup
  or code. Email syntax or a public key does not prove control, identity, trust,
  or permission to send. The runtime MUST NOT expose private keys, tokens,
  provider credentials, source URLs, notes, or raw source records.
- Picker output MUST be limited to confirmed endpoints and their disclosed
  display text. A selected address MUST NOT become a query for related contact
  records, a permission to enumerate them, or an automatic messaging action.
- The runtime MUST enforce source and kind restrictions before filtering,
  paging, cache reuse, or `get`. A match in a hidden field MUST NOT reveal the
  existence of a contact. It MUST isolate cached data and cursors across shell
  users and verified napplet scopes.
- A contact value MUST NOT select credentials, a synchronization server, a file,
  or a network destination for contact retrieval. Any later messaging or
  publication remains governed by its owning domain and shell policy.

## Implementations

No implementation of this NAP is verified. Existing address books or contact
pickers do not establish support for this draft.

## Changelog
