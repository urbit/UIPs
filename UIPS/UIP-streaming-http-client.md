---
title: Streaming HTTP Client Responses
description: Opt-in incremental delivery of HTTP client response headers and body chunks
author: ~niblyx-malnus
status: Draft
type: Standards Track
category: Kernel
created: 2026-03-16
---

## Abstract

This UIP adds an opt-in streaming mode to the HTTP client driver (cttp). When a request includes an `x-urbit-stream` header, the runtime delivers response headers and body chunks incrementally as `%start` / `%continue` events instead of buffering the entire response into a single `%finished` event. On connection failure, streaming requests receive a `%cancel` event instead of a fabricated 504 response.

## Motivation

The current HTTP client implementation in vere buffers the entire response body before delivering anything to Arvo. This is incompatible with streaming protocols like Server-Sent Events (SSE), where the connection stays open indefinitely and data arrives incrementally. Under the buffered model, zero data reaches Arvo until the connection closes — which may be never.

Additionally, when HTTP requests fail (DNS failure, connection refused, TLS error, timeout), vere fabricates a `[%start [504 ~] ~ %.y]` response. This is misleading — it is not an actual HTTP 504 from a gateway. Once streaming delivers `[%start [200 headers] ~ %.n]`, we have committed to status 200. There is no way to subsequently send a different status code. The existing `%cancel` variant of `http-event` — defined in `lull.hoon` but never sent by the runtime — is the correct mechanism for signaling connection failures.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Opt-in via Request Header

Streaming mode is opt-in. An agent requests streaming by including the `x-urbit-stream` header in its `%request` card. The header value is ignored; presence is sufficient.

The runtime MUST detect this header, record the request as streaming, and strip the header before sending the HTTP request to the remote server.

The runtime MUST scan all headers in the request, not just the first. The `x-urbit-stream` header MAY appear at any position.

Requests without this header MUST continue to receive fully-buffered responses exactly as they do today.

### Streaming Delivery Sequence

When streaming is enabled, the runtime MUST deliver HTTP events incrementally using the existing `http-event` variants:

**Headers arrive (`on_head` callback):**

```
[%start [status-code headers] ~ complete=%.n]
```

The `%start` event is sent immediately when response headers arrive, with no body and `complete=%.n`. If the response has no body (end-of-stream on head), `complete=%.y` is sent instead.

**Body chunk arrives (`on_body` callback):**

```
[%continue body=`(unit octs) complete=?]
```

Each body chunk is dispatched immediately as a `%continue` event. The `body` field is `~` when no data is present (e.g. a final empty chunk). The `complete` flag is `%.y` on the final chunk (end-of-stream) and `%.n` otherwise.

The runtime MUST send `%continue` when either data is available or the stream has ended (or both). The runtime MUST NOT suppress a final `%continue` with `complete=%.y` even if the data field is empty.

**Connection error (`on_head` or `on_body` error):**

```
[%cancel ~]
```

When an HTTP connection fails at any point — before or after headers have been delivered — the runtime MUST send `[%cancel ~]` for streaming requests. The error string is logged to the terminal.

### Non-streaming Behavior

Without the `x-urbit-stream` header, the runtime MUST buffer the complete response and deliver it as a single `[%start [status headers] body %.y]`, preserving current behavior. On failure, the runtime continues to send a fabricated `[%start [504 ~] ~ %.y]`.

### No Type Changes

This proposal requires no changes to `lull.hoon` or any Hoon-side type definitions. The `%start`, `%continue`, and `%cancel` variants of `http-event` already exist. The runtime simply uses them in a new sequence. `%cancel` in particular is already defined as `[%cancel ~]` in `lull.hoon` — this proposal causes the runtime to actually send it.

## Rationale

**Why opt-in?** Streaming changes the delivery model from a single synchronous response to an asynchronous sequence of events. Agents written for the buffered model expect a single `%start` with `complete=%.y` containing the full body. Sending a partial `%start` followed by `%continue` chunks would break them. Opt-in via a request header ensures zero disruption to existing agents. Additionally, some use cases still prefer buffering, such as short-lived requests where the agent only cares about the final result, or large file downloads where hitting the event loop on every chunk would be inefficient.

**Why a request header?** The requesting agent is the right place to decide whether it wants streaming. A header on the outbound request is the natural mechanism — it requires no new types or API changes and mirrors HTTP conventions (like `Accept: text/event-stream`). The runtime checks for the header's presence and strips it before sending.

**Why `%cancel` instead of fixing 504?** A fabricated 504 is misleading — no gateway actually returned it. More critically, once a streaming `%start` delivers status 200, there is no HTTP-legal way to retract it. `%cancel` is the only honest signal for mid-stream failure. Using it uniformly for all streaming errors (pre- and post-header) is simpler than maintaining two error paths.

**Why strip the header?** The `x-urbit-stream` header is a directive to the runtime, not to the remote server. Sending it would be confusing at best and could cause unexpected behavior if a remote server happens to interpret it.

## Backwards Compatibility

This proposal is fully backward compatible:

- **Existing agents** that do not send `x-urbit-stream` see no change in behavior. Responses continue to be fully buffered with fabricated 504 on failure.
- **Existing `%cancel` handling** continues to work. `[%cancel ~]` matches the existing type in `lull.hoon`.
- **No kelvin change.** No Hoon types are modified. The runtime uses existing event variants in a new delivery pattern, gated entirely by the presence of a request header.

## Reference Implementation

[urbit/vere#976](https://github.com/urbit/vere/pull/976) — runtime changes in `pkg/vere/io/cttp.c`:

- `_cttp_creq_is_streaming()` — scans request headers for `x-urbit-stream`, strips it, returns flag
- `_cttp_creq_send_start()` — dispatches `%start` event with headers and status
- `_cttp_creq_send_continue()` — dispatches `%continue` event with optional body chunk and fin flag
- `_cttp_creq_send_cancel()` — dispatches `[%cancel ~]` event
- `_cttp_creq_fail()` — routes to `%cancel` (streaming) or fabricated 504 (buffered)
- `_cttp_creq_on_head()` — sends `%start` immediately for streaming, buffers for non-streaming
- `_cttp_creq_on_body()` — sends `%continue` immediately for streaming, accumulates for non-streaming

## Security Considerations

- **No new attack surface.** Streaming does not change what data is transmitted, only when it is delivered to Arvo. The same HTTP request/response semantics apply.
- **Denial of service.** Agents receiving an excessive volume of chunks can send `%cancel-request` to terminate the connection at any time.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
