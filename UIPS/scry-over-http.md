---
title: <Scry over HTTP>
description: <Map URLs to Scry Requests>
author: <~rovnys-ricfer, ~watter-parter>
discussions-to: <TODO>
status: Draft
type: <Standards Track>
category: <Kernel>
created: <2023-06-20>
---

## Abstract

Vere will interpret certain HTTP requests as scry requests and call into Arvo's `+peek` arm to perform the query statelessly, rather than injecting an event using Arvo's `+poke` arm.  Vere could also maintain a cache for such HTTP requests, but that is not a requirement for a practical runtime.

## Motivation

- We want to support requests for fully qualified scry paths, in case a ship is re-publishing someone else's content
- We want to support `%pine`-style requests at the ship's current date, without the client knowing that date ahead of time
- We need to have a coherent story for mark-converting scry responses
  - We could always jam the result of a scry request as either a default or fallback
  - We need to support arbitrary mark conversion, e.g. conversion from some mark to `%json`, or `%html` to `%mime` -- `%mime` is probably the default

## Specification

- URL for a Fully Qualified Scry Path
  - `/scry/<ship>/<rift>/<life>/<vane>/<care>/<case>/<spur>` or
  - `/scry/<old-path>` the old scry format that started with either vane or vane+care, e.g. `cx/~zod/...`
- URL for a Partial Scry Path at Latest:
  - `/pine/<vane>/<care>/<spur>` or
  - `/pine/<vane>/<care>/<spur>?urbit-redirect=true`
  - If the URL includes the `urbit-redirect=true` query string, then Vere will handle this request by first resolving the request to a fully qualified scry path, then redirecting to that path.
  - Without that header, Vere will handle the request directly.  This distinction is to protect the behavior of a user refreshing in their browser to see the latest version of some file.  If Vere redirects by default, the user would have to understand what was going on and also click "back" each time they wanted to refresh, which would be annoying and counterintuitive.
- URL for mark conversions:
  - Append a `.new-mark` to the URL, e.g. `/c/x/sys/kelvin.html`
  - This converts from the original mark of the scry result, to the specified mark (`html` here), to `%mime`.

## Rationale

TODO

## Backward Compatibility

TODO Make sure this doesn't conflict with any existing uses of the URL namespace.

## Security Considerations

TODO This should use the same authentication as other Eyre requests, and then Eyre should use that to populate the security mask on the internal Arvo scry request.

TODO Ideally you could share one of these URLs with someone and they could auto-log-in as their Urbit to see the data.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

