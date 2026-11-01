NAP-EMAIL
=========

Shell-Mediated Email
--------------------

`draft`

The key words MUST, MUST NOT, SHOULD, SHOULD NOT, and MAY are interpreted as
described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.html).

**NAP ID:** NAP-EMAIL
**Domain:** `email`
**Web binding ([NIP-5D](https://github.com/nostr-protocol/nips/pull/2303)):** `window.napplet.email`; domain presence signals availability. This is projection guidance.

## Description

NAP-EMAIL lets napplets read email and submit messages using accounts configured
in the shell. The napplet designs the inbox, reader, and composer. The shell
mediates account selection, disclosure, and sending authority. The runtime holds
credentials, retrieves mail through POP3 (Post Office Protocol version 3), and
submits mail through SMTP (Simple Mail Transfer Protocol). The user can grant a
reader access to incoming mail and a separate composer permission to send.

### Boundary

**Account configuration is privileged runtime state.** Returning SMTP or POP3
credentials as napplet settings would give the napplet authority outside the
seam. An email address, account handle, or declared domain grants no authority.

| Concern | Owner |
|---------|-------|
| Account creation, removal, server names, ports, TLS and authentication settings | User, through trusted shell UI |
| Passwords, application passwords, tokens, credential refresh, server connections | Runtime |
| Account and sender selection, read grants, send grants, recipient restrictions and confirmation | Shell, under user control |
| Retrieval, MIME parsing and construction, message storage, submission records | Runtime |
| Inbox presentation, composing, reply and threading choices, local drafts | Napplet |
| Network endpoints, protocol commands, server sessions and message-number mapping | Runtime internals |

SMTP and POP3 are separate account services. An account MAY have either or both.
SMTP supplies no inbox; POP3 supplies no submission service, folders, synchronized
read flags, or sent-mail archive. This draft exposes a runtime-held inbox and
explicit submission records. It does not promise those server features.

Remote deletion, mailbox administration, raw SMTP/POP3 commands, arbitrary
headers, and account-configuration APIs are outside this surface. Email reads
MUST NOT cause the runtime to delete mail from the POP3 server. Any independent
shell retention policy requires separate user authorization.

No other NAP domain is required. Per-napplet configuration APIs are not a channel
for provisioning these accounts or delivering their secrets. This NAP defines
no cross-napplet convention and changes no projection semantics.

MIME means Multipurpose Internet Mail Extensions: the format for email body
parts, alternatives, and attachments.

## API Surface

A runtime exposing `email` MUST implement every operation below. The `receive`
and `send` fields describe the services exposed to the caller for each account;
they do not override current policy. Availability is projection-defined.
Manifests declare the bare domain `email`; declarations are not grants.

| Operation | Parameters | Result | Wire |
|-----------|------------|--------|------|
| `accounts` | none | list of `EmailAccount` | `email.accounts` / `email.accounts.result` |
| `list` | `account` (text), optional `cursor` (text), optional `limit` (integer) | `EmailPage` | `email.list` / `email.list.result` |
| `get` | `account` (text), `message` (text) | `EmailMessage` | `email.get` / `email.get.result` |
| `readPart` | `account` (text), `message` (text), `part` (text), `offset` (integer), `length` (integer) | `EmailBytes` | `email.readPart` / `email.readPart.result` |
| `submit` | `account` (text), `submissionId` (text), `draft` (`EmailDraft`) | `EmailSubmission` | `email.submit` / `email.submit.result` |
| `submission` | `account` (text), `submissionId` (text) | `EmailSubmission` | `email.submission` / `email.submission.result` |

`accounts` returns only accounts the shell has authorized this napplet to
discover. An empty list means none are exposed; it does not reveal whether other
accounts exist. Account setup and grants occur through the shell's own UI.
The shell MAY offer account selection or permission confirmation when handling
a request. The napplet cannot preselect an undisclosed account by address.

### Common values

| Value | Definition |
|-------|------------|
| Handles | Nonempty opaque text issued by the runtime. Account, message, part and cursor handles are not URLs, paths, protocol commands, or permission tokens. |
| `submissionId` | Exactly 32 lowercase hexadecimal characters, generated randomly by the napplet for each deliberate new submission. It is separate from request correlation `id` and the email `Message-ID` header. |
| Timestamp | Integer seconds since the Unix epoch, UTC. |
| Encoded bytes | Canonical padded base64 with no whitespace. Byte counts and offsets refer to decoded bytes. |
| Optional fields | Omission has the stated meaning. `null` is invalid except where explicitly permitted. Empty lists and empty text are valid only as specified. |
| Control characters | Unicode U+0000–U+001F and U+007F–U+009F. |

Requests MUST contain only their defined fields, including nested records;
the runtime MUST reject extra fields with `invalid-request`. Results MAY add
fields in later revisions; napplets MUST ignore unrecognized result fields.

### EmailAccount

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `account` | yes | text | Account handle scoped to the shell user and verified napplet identity. Stable across endpoint reloads while the account and identity remain the same. |
| `label` | yes | text | Shell-selected display label; may be empty. No server or login details are required. |
| `receive` | yes | boolean | Caller may request the runtime-held inbox for this account. |
| `send` | yes | boolean | Caller may request submission through this account. |
| `senders` | yes | list of `EmailAddress` | Approved From identities. Nonempty when `send` is true; empty otherwise. |
| `maxPageSize` | yes | integer | Positive maximum `list.limit`; omission of `limit` uses this value. |
| `maxReadBytes` | yes | integer | Positive maximum `readPart.length`. |
| `maxContentBytes` | yes | integer | Positive maximum sum of UTF-8 body bytes (`text` plus `html`, if present) and decoded attachment bytes per submission. |

`receive` and `send` reflect both configured services and exposed grants. A
send-only grant does not disclose inbox metadata. A read grant does not imply a
send grant. Limits are runtime-selected and MAY change; exceeding a current
limit returns `too-large`. Additional finite resource and rate limits MAY apply.

### EmailAddress

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `address` | yes | text | One mailbox addr-spec under [RFC 5322](https://www.rfc-editor.org/rfc/rfc5322.html); no display-name syntax, groups, comments, or control characters. |
| `name` | no | text | Display name; omission means none. May be empty. MUST NOT contain control characters. |

For outbound addresses the baseline is ASCII addr-spec syntax with an ASCII
domain; internationalized local parts are outside this draft. Display names and
message text MAY contain Unicode. The runtime MUST perform header encoding and
validation; it MUST NOT concatenate napplet text into protocol commands.

### EmailPage

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `messages` | yes | list of `EmailSummary` | At most `limit` items. Empty when the snapshot has no remaining items. |
| `next` | yes | text or null | Opaque cursor for the next page, or `null` at the end. |
| `syncedAt` | yes | integer or null | Completion time of the last successful POP3 retrieval reflected in the snapshot; `null` if none. |

`list` reads the runtime-held inbox. The runtime schedules POP3 retrieval under
shell policy; calling `list` does not force a network poll. A first page captures
an immutable metadata snapshot, ordered by runtime ingestion time descending,
then message handle in ascending Unicode code-point order. Subsequent pages
preserve that order and `syncedAt`. New mail appears in a new first-page query.
Expired cursors return `stale`; callers restart without a cursor. A cursor is
bound to its caller scope, account, and original limit; a supplied continuation
limit MUST equal the original limit, or returns `invalid-request`. Omission on
a continuation uses the original limit.
`limit` MUST be positive; a value above the current `maxPageSize` returns
`too-large`. A reduced limit can require restarting pagination.

### EmailSummary and EmailMessage

`EmailMessage` contains every `EmailSummary` field plus `parts`.

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `message` | yes | text | Opaque runtime message handle; scoped to the account. |
| `messageId` | no | text | Parsed Message-ID including angle brackets; omitted if absent or invalid. Untrusted correspondence metadata, not a lookup key. |
| `from` | yes | list of `EmailAddress` | Parsed From mailboxes; empty if unavailable or invalid. |
| `replyTo` | yes | list of `EmailAddress` | Parsed Reply-To mailboxes; empty if unavailable or invalid. A suggestion, not send authority. |
| `to` | yes | list of `EmailAddress` | Parsed To mailboxes; empty if unavailable or invalid. |
| `cc` | yes | list of `EmailAddress` | Parsed Cc mailboxes; empty if unavailable or invalid. |
| `subject` | yes | text | Decoded subject; empty if unavailable. |
| `date` | yes | integer or null | Parsed message Date; `null` if unavailable or invalid. Not trusted receipt time. |
| `ingestedAt` | yes | integer | Time the runtime first stored this message. |
| `parts` | only on EmailMessage | list of `EmailPart` | Non-multipart MIME entities in document order, including body alternatives and attachments. Multipart containers are traversed recursively; encapsulated messages remain opaque `message/rfc822` parts. |

Header lists preserve mailbox order; groups in received address headers are
flattened to their mailboxes. Invalid headers yield the empty or omitted values
specified above. Header unfolding and encoded-word decoding follow
[RFC 5322](https://www.rfc-editor.org/rfc/rfc5322.html) and
[RFC 2047](https://www.rfc-editor.org/rfc/rfc2047.html). Content-Type defaults
follow MIME; `charset` preserves a declared charset rather than guessing one.

The runtime MUST preserve a handle's association with immutable message content
until that content is removed. Removed content returns `not-found`; a handle
MUST NOT be reassigned to different content. POP3 session message numbers and
email Message-ID values MUST NOT serve as message handles. If persistent POP3
identifiers are unavailable or unreliable, the runtime MUST preserve this
association using its local store; it MAY ingest duplicates but MUST NOT return
different content under an existing handle. See [RFC 1939](https://www.rfc-editor.org/rfc/rfc1939.html).

### EmailPart and EmailBytes

The runtime parses MIME under
[RFC 2045](https://www.rfc-editor.org/rfc/rfc2045.html) and
[RFC 2046](https://www.rfc-editor.org/rfc/rfc2046.html). It returns decoded
content bytes, not transfer-encoded MIME bodies.

| EmailPart field | Required | Type | Notes |
|-----------------|----------|------|-------|
| `part` | yes | text | Handle bound to this immutable message. |
| `mediaType` | yes | text | Lowercase media type without parameters, e.g. `text/plain`. |
| `charset` | no | text | Declared text charset, if present. Omission does not assert UTF-8. |
| `disposition` | yes | "inline" or "attachment" | Missing or unrecognized MIME disposition maps to `inline`; this is a presentation hint. |
| `filename` | no | text | Decoded, untrusted suggested filename. Not a host path. |
| `contentId` | no | text | Parsed Content-ID including angle brackets, if present. Local MIME reference only. |
| `size` | yes | integer | Nonnegative decoded byte count. |

Malformed MIME or transfer encoding returns `malformed-message` from `get` or
`readPart`; it MUST NOT be executed or silently represented as a different
message. Metadata can still appear in `list`. Unsupported text charsets MAY
remain bytes; the napplet decides how to display them.

| EmailBytes field | Required | Type | Notes |
|------------------|----------|------|-------|
| `offset` | yes | integer | Requested zero-based offset, from zero through the part's `size`. |
| `data` | yes | text | Base64 for the next `min(length, size - offset)` bytes. Empty only at end. |
| `eof` | yes | boolean | True exactly when the returned range ends at `size`. |

`length` MUST be positive; exceeding `maxReadBytes` returns `too-large`. An offset
below zero or beyond `size` is invalid. Chunking changes no content or part
handles. If retention removes a message between reads, later reads return
`not-found`.

### EmailDraft and EmailAttachment

| EmailDraft field | Required | Type | Notes |
|------------------|----------|------|-------|
| `from` | yes | text | Exact `address` of one advertised sender; shell supplies its approved display name. |
| `to` | yes | list of `EmailAddress` | May be empty if another recipient list is nonempty. |
| `cc` | no | list of `EmailAddress` | Default empty. |
| `bcc` | no | list of `EmailAddress` | Default empty. Envelope recipients only; MUST NOT appear in submitted message headers. |
| `subject` | yes | text | May be empty. MUST NOT contain control characters. |
| `text` | yes | text | UTF-8 plain-text body; may be empty. |
| `html` | no | text | UTF-8 HTML alternative; omission means none. May be empty. |
| `inReplyTo` | no | text | One valid RFC 5322 Message-ID including angle brackets. |
| `references` | no | list of text | Valid RFC 5322 Message-IDs in caller order. Default empty. |
| `attachments` | no | list of `EmailAttachment` | Default empty; sent as MIME attachments. |

At least one recipient across `to`, `cc`, and `bcc` is required. The SMTP envelope
is their union, deduplicated by exact address text; names do not affect that
union. The runtime MUST validate the whole draft and all recipient permissions
before admitting a submission; a grant MAY still require confirmation before
the SMTP attempt. It MUST use the selected approved sender for From and the
account's shell-configured envelope sender. It constructs Date,
Message-ID, MIME headers, and body encoding. Reply metadata grants no account,
recipient, or conversation authority. Local drafts stay with the napplet until
submitted; this operation does not create a server-side draft or sent copy.
The runtime MUST freeze the constructed message and envelope before requesting
confirmation or starting an attempt. A change requires a new submission identifier.

| EmailAttachment field | Required | Type | Notes |
|-----------------------|----------|------|-------|
| `filename` | yes | text | Nonempty suggested filename; no control characters or path separators. |
| `mediaType` | yes | text | Lowercase valid media type without parameters. |
| `data` | yes | text | Base64 attachment bytes; empty attachments are valid. |

Attachment bytes are supplied explicitly. A path, URL, MIME reference, or received
part handle MUST NOT cause the runtime to fetch or attach content. Forwarding a
received attachment requires independently authorized reads and a new submission.

### EmailSubmission

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `submissionId` | yes | text | Caller-supplied submission identifier. |
| `state` | yes | "pending" or "accepted" or "failed" or "uncertain" | Defined below. |
| `messageId` | yes | text | Runtime-generated RFC 5322 Message-ID, fixed when the record is created. |
| `failure` | only when failed | `EmailError` | Definitive reason the message was not accepted for submission. |

| State | Meaning |
|-------|---------|
| `pending` | Durably admitted; waiting for mediation or for one SMTP attempt to finish. No delivery claim. |
| `accepted` | SMTP server positively acknowledged the complete message for every envelope recipient. No claim of final mailbox delivery or reading. |
| `failed` | Definitively no message accepted for delivery to any recipient. No automatic retry. |
| `uncertain` | An attempt may have succeeded but no definitive completion is available. No automatic retry. |

The runtime MUST durably record a new submission before any SMTP attempt and
before returning success from `submit`. It MAY return `pending` or a state
already reached. `submission` reads the current record without causing an
attempt. `pending` MUST eventually become a terminal state under a finite
runtime-selected deadline. Terminal states MUST NOT change.

Submission identity is scoped to shell user, verified napplet identity, and
account. Repeating `submit` with the same `submissionId` and same draft MUST
return the existing record without another SMTP attempt or confirmation.
Different drafts under an existing identifier return `conflict`. Equality
ignores record-field order and applies the defaults in the schema; list order
and text remain significant. Requests using different correlation ids do not
create different submissions. A new deliberate attempt requires a new
`submissionId` and fresh policy evaluation.

The runtime MUST reserve new identifiers atomically across concurrent requests
in the same scope. Existing records and tombstones are resolved after authority
and schema checks, before applying limits for a new admission. A later limit
change MUST NOT turn an identical repeat into another attempt.

The runtime MUST preserve records across endpoint disconnects and runtime
restarts. It MAY discard terminal record details under shell retention policy,
but MUST retain a tombstone preventing reuse of that identifier for the account's
lifetime. Queries and repeated submits for tombstones return `expired`, with no
SMTP attempt. If it cannot reserve durable state, it MUST reject before effects.
Removing and recreating an account MUST issue a different account handle.

The runtime MUST make at most one SMTP attempt per admitted identifier. It MUST
abort before transmitting the message body if any envelope recipient is
rejected; partial recipient submission is not supported. A positive final
acknowledgment produces `accepted`; a definitive rejection produces `failed`.
Loss of the final response or a restart after an attempt may have begun produces
`uncertain` unless the runtime has definitive evidence. Recovery MUST NOT
resubmit an ambiguous attempt. This prevents runtime retries from duplicating
an uncertain send; SMTP itself offers no exactly-once delivery guarantee.
See [RFC 5321](https://www.rfc-editor.org/rfc/rfc5321.html).

## Wire Protocol

Messages use the projection's carrier. The following logical fields define the
contract; no host transport is required here.

| Type | Direction | Payload fields |
|------|-----------|----------------|
| `email.accounts` | napplet → runtime | `id` |
| `email.accounts.result` | runtime → napplet | `id`, `value` or `error` |
| `email.list` | napplet → runtime | `id`, `account`, `cursor?`, `limit?` |
| `email.list.result` | runtime → napplet | `id`, `value` or `error` |
| `email.get` | napplet → runtime | `id`, `account`, `message` |
| `email.get.result` | runtime → napplet | `id`, `value` or `error` |
| `email.readPart` | napplet → runtime | `id`, `account`, `message`, `part`, `offset`, `length` |
| `email.readPart.result` | runtime → napplet | `id`, `value` or `error` |
| `email.submit` | napplet → runtime | `id`, `account`, `submissionId`, `draft` |
| `email.submit.result` | runtime → napplet | `id`, `value` or `error` |
| `email.submission` | napplet → runtime | `id`, `account`, `submissionId` |
| `email.submission.result` | runtime → napplet | `id`, `value` or `error` |

`id` is nonempty text, unique among the calling endpoint's outstanding requests.
For each authenticated, routable request with a valid correlation id, the
runtime MUST return exactly one corresponding result with the same `id` to
that endpoint. Success contains `value` of the API row's result type and no
`error`; failure contains `error` and no `value`. Correlation is endpoint-scoped.
There are no unsolicited deliveries or wire subscriptions; callers poll inbox
snapshots and submission records subject to shell rate limits.

### Error Handling

`EmailError`:

| Field | Required | Type | Notes |
|-------|----------|------|-------|
| `code` | yes | text | One of the codes below. Callers branch on this value. |
| `message` | yes | text | Safe human-readable explanation. |

| Code | Meaning |
|------|---------|
| `invalid-request` | Invalid schema, field, encoding, syntax, or range. |
| `forbidden` | Current policy denies account discovery or the requested account, data, sender, or recipients. |
| `unsupported` | Account does not expose the required configured service. |
| `not-found` | Message, part, or submission is absent in the caller's authorized scope. |
| `stale` | Cursor snapshot expired. |
| `expired` | Submission details expired; its identifier remains reserved. |
| `conflict` | Existing submission identifier was used with a different draft. |
| `too-large` | Request or result exceeds a runtime limit. |
| `busy` | Finite work, rate, or durable-storage capacity is exhausted. |
| `unavailable` | Required account service or local store is unavailable, including authentication or TLS failure. |
| `malformed-message` | Stored message cannot be safely parsed or decoded. |
| `rejected` | SMTP server definitively rejected the attempt; used in a failed submission's `failure`. |

The runtime MUST validate and authorize before effects. Unknown and unauthorized
account handles both return `forbidden`. Within an authorized account, unknown
and out-of-scope message, part, cursor, and submission identifiers MUST be
indistinguishable from absent identifiers (`not-found`). Policy checks precede
existence disclosure. A missing service returns `unsupported` only after account
access is established. An invalid range on an authorized existing part returns
`invalid-request`. Rate and size errors require no disclosure of hidden data.

After admission, an SMTP failure is a successful API result carrying a `failed`
submission record. API failure MUST NOT report a definitive no-send outcome for
an admitted attempt. If the reply is lost, the napplet SHOULD query `submission`
or repeat the identical `submit`; it MUST NOT infer failure and automatically
choose a new identifier. `not-found` after a timeout is not proof that no earlier
request can still be admitted; repeating the identical request remains safe.

### Examples

**Discover accounts already exposed by shell policy:**

```text
→ { "type": "email.accounts", "id": "a1" }
← { "type": "email.accounts.result", "id": "a1", "value": [{ "account": "a1", "label": "Personal", "receive": true, "send": true, "senders": [{ "address": "bob@example.net" }], "maxPageSize": 100, "maxReadBytes": 65536, "maxContentBytes": 10485760 }] }
```

**Read an authorized account's inbox and a body part:**

```text
→ { "type": "email.list", "id": "l1", "account": "a1", "limit": 10 }
← { "type": "email.list.result", "id": "l1", "value": { "messages": [{ "message": "m1", "from": [{ "address": "alice@example.org" }], "replyTo": [], "to": [{ "address": "bob@example.net" }], "cc": [], "subject": "Hello", "date": null, "ingestedAt": 1793491200 }], "next": null, "syncedAt": 1793491200 } }
→ { "type": "email.get", "id": "g1", "account": "a1", "message": "m1" }
← { "type": "email.get.result", "id": "g1", "value": { "message": "m1", "from": [{ "address": "alice@example.org" }], "replyTo": [], "to": [{ "address": "bob@example.net" }], "cc": [], "subject": "Hello", "date": null, "ingestedAt": 1793491200, "parts": [{ "part": "p1", "mediaType": "text/plain", "charset": "utf-8", "disposition": "inline", "size": 5 }] } }
→ { "type": "email.readPart", "id": "r1", "account": "a1", "message": "m1", "part": "p1", "offset": 0, "length": 5 }
← { "type": "email.readPart.result", "id": "r1", "value": { "offset": 0, "data": "SGVsbG8=", "eof": true } }
```

**Invalid range and denied account; neither grants access or changes mail:**

```text
→ { "type": "email.readPart", "id": "r2", "account": "a1", "message": "m1", "part": "p1", "offset": -1, "length": 5 }
← { "type": "email.readPart.result", "id": "r2", "error": { "code": "invalid-request", "message": "Offset must be nonnegative." } }
→ { "type": "email.list", "id": "l2", "account": "not-granted" }
← { "type": "email.list.result", "id": "l2", "error": { "code": "forbidden", "message": "Account access is not authorized." } }
```

**Admission, retry after a lost reply, and uncertain SMTP completion:**

```text
→ { "type": "email.submit", "id": "s1", "account": "a1", "submissionId": "3ca28f17da2648b3a0ad1e0281c5f647", "draft": { "from": "bob@example.net", "to": [{ "address": "alice@example.org" }], "subject": "Hello", "text": "Hello" } }
← { "type": "email.submit.result", "id": "s1", "value": { "submissionId": "3ca28f17da2648b3a0ad1e0281c5f647", "state": "pending", "messageId": "<s1@mail.example.net>" } }
→ { "type": "email.submit", "id": "s2", "account": "a1", "submissionId": "3ca28f17da2648b3a0ad1e0281c5f647", "draft": { "from": "bob@example.net", "to": [{ "address": "alice@example.org" }], "subject": "Hello", "text": "Hello" } }
← { "type": "email.submit.result", "id": "s2", "value": { "submissionId": "3ca28f17da2648b3a0ad1e0281c5f647", "state": "pending", "messageId": "<s1@mail.example.net>" } }
→ { "type": "email.submission", "id": "s3", "account": "a1", "submissionId": "3ca28f17da2648b3a0ad1e0281c5f647" }
← { "type": "email.submission.result", "id": "s3", "value": { "submissionId": "3ca28f17da2648b3a0ad1e0281c5f647", "state": "uncertain", "messageId": "<s1@mail.example.net>" } }
```

Both submits refer to one durable record and at most one SMTP attempt. With a
definitive positive final SMTP response, the queried state would be `accepted`.
Reusing the identifier with changed text returns `conflict`. If the endpoint
disconnects after admission, the runtime keeps the record; a new authorized
endpoint of the same verified napplet identity can query it. A different napplet
using the same correlation id receives only its own result, and cannot inspect
this record by copying these handles.

## Shell Behavior

- The shell MUST configure and store accounts outside napplet-controlled UI and
  schemas. It MUST NOT accept server, port, credential, TLS, or identity overrides
  from a napplet request. The user can configure SMTP, POP3, or both and manage
  grants through the shell without running a particular email napplet.
- The runtime MUST derive caller scope from the authenticated endpoint's
  verified manifest and artifact identity, as defined by the projection, plus
  the shell user. A payload's address, pubkey, manifest claim, or handle cannot
  replace that binding. Grants MUST distinguish account discovery, receiving
  mail, and sending mail. New artifacts require shell policy evaluation.
- The shell MUST disclose the scope of a read grant: inbox headers, bodies, and
  attachments. The runtime MUST enforce read authority on `list`, `get`, and
  `readPart`, and send authority on `submit` and `submission`. The shell MAY
  require confirmation for each draft or authorize bounded sending in advance.
  Any confirmation MUST cover the sender, every recipient including Bcc, message
  body alternatives, and attachments of the immutable admitted draft.
- The runtime MUST recheck policy before an SMTP attempt. Revocation before the
  attempt or declined confirmation produces `failed` with `forbidden`.
  Revocation or disconnect after an attempt begins cannot promise recall; the
  runtime MUST preserve its outcome internally. Revocation MUST stop later
  disclosures and unstarted attempts.
  In-flight reads MUST recheck authority before releasing their results. On a
  shell-user switch, the runtime MUST NOT deliver the old user's results to the
  new user. Endpoint teardown stops delivery, not durable submission tracking.
- For configured POP3 accounts, the shell MUST provide retrieval controls and
  show the last successful retrieval and current failures, including login or
  TLS failure. The runtime SHOULD retrieve periodically while the account is
  enabled, under user-selected shell policy. Reading a cached inbox MUST NOT
  imply that the POP3 server is reachable or synchronized at request time.
- The runtime MUST bound retrieval, MIME nesting, decoded bytes, pagination,
  message retention, submissions, and request duration. Limit failures use
  `too-large`, `busy`, or `unavailable` as applicable. An admitted submission's
  deadline follows its state rules rather than erasing an ambiguous attempt.
- The runtime MAY use local libraries, native services, or a trusted configured
  backend. Backend selection does not change this contract. A remote backend
  handling credentials or plaintext mail is part of the runtime's trust boundary;
  the shell MUST disclose that handling to the user during account setup.

## Security Considerations

- The runtime MUST keep credentials, authentication transcripts, tokens, server
  settings and login identifiers out of napplet results, logs exposed to
  napplets, and error text. Approved sender addresses are deliberate disclosures;
  an address does not establish a Nostr identity or prove an incoming author.
- The runtime MUST use authenticated TLS for SMTP submission and POP3 access,
  consistent with [RFC 8314](https://www.rfc-editor.org/rfc/rfc8314.html). It MUST
  reject certificate or required TLS negotiation failures and MUST NOT silently
  fall back to plaintext. Credentials MUST only reach the configured authorized
  service. Mail headers, attachments and napplet input MUST NOT redirect account
  connections or choose authentication mechanisms.
- Email content, sender names, MIME metadata, HTML and filenames are untrusted.
  The runtime MUST parse them with bounded work and MUST NOT execute scripts,
  resolve external entities, load remote images, or follow content URLs while
  retrieving or parsing mail. Napplets rendering HTML MUST keep it inert and
  sanitize it; a MIME media type conveys no execution authority. The projection's
  isolation and network restrictions still apply to rendered mail.
- Possession of a read grant permits disclosure of private mail to that napplet.
  The shell MUST NOT describe it as a display-only grant. It does not authorize
  uploading mail, sending replies, or retrieving external tracking content;
  those effects require their own applicable shell mediation.
- The runtime MUST prevent header and command injection, Bcc header disclosure,
  filename traversal, decompression bombs and cross-account cache leakage. It
  MUST NOT save attachment filenames as host paths without independent handling.
  Submission confirmation MUST show the exact authorized draft, not a mutable
  napplet-controlled preview.

## Implementations

No implementation of this NAP is verified. Existing SMTP and POP3 clients do not
establish support for this draft.

## Changelog
