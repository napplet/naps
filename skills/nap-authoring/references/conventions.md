# Convention boundaries

Read this when a proposed NAP accepts convention URIs or routes between napplets.
Use the current [NAP-INTENT](https://github.com/napplet/naps/blob/master/naps/NAP-INTENT.md),
[NAP-INC](https://github.com/napplet/naps/blob/master/naps/NAP-INC.md), and
[web projection](https://github.com/napplet/naps/blob/master/projections/web.md)
as the owning contracts. This guide does not define another carrier.

- Stable identity is the queryless, fragment-free `napplet:<archetype>/<intent>`.
  Handler metadata and subscriptions use that exact identity.
- Invocation queries are shallow payload sugar. The runtime-provided binding
  transposes unique, percent-decoded `name=value` pairs into text fields before
  routing or handler resolution. It does not coerce booleans or numbers, and
  `+` remains a literal plus sign.
- Reject malformed percent-encoding, repeated decoded names, and a query mixed
  with an explicit payload. Use explicit payloads for structured or non-text data.
- Only the NAP-INTENT URI invocation form accepts `#<naddr>`. The binding extracts
  the bare [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) value
  into `handlerHint` before transposition and routing. Follow NAP-INTENT's
  validation rules; do not generalize fragment support to other operations.
- The hint recommends a handler under user defaults and shell policy. It is
  never payload, stable identity, handler metadata, or an authority grant.
- Other operations, including NAP-INC, reject fragments.
- Routers compare the resulting stable identity by exact equality. They do not
  parse, normalize, prefix-match, or wildcard-match it.

For example, `napplet:profile/open?label=A+B` yields stable identity
`napplet:profile/open` and text payload field `label` equal to `A+B`. It does not
declare that `label` is supported by a particular convention; the convention's
payload schema must name supported query fields.

A request to standardize a profile-opening payload belongs in a convention.
A request to change how the runtime selects and delivers to a handler belongs
in the owning runtime API. Keep those contracts separate.
