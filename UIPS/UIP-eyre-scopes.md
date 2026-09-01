---
title: Eyre Scopes
description: Desk provenance for eyre authentication, for userspace permissions.
author: ~palfun-foslup
status: Draft
type: Standards Track
category: Kernel
created: 2026-02-25
---


## Abstract

In order to extend userspace permissions into HTTP contexts, eyre must provide secure client provenance. We propose assigning desk scopes to authentication sessions, and isolating these sessions in browser contexts by assigning each desk its own subdomain. We reduce UX impact by automating subdomain authentication for browsers, and support OAuth-style authentication for standalone clients.


## Status

The Eyre Security Working Group is actively refining the implementation as specified here. Core mechanism has been implemented, big remaining task is supporting multiple domains and their certificates. Work on ancillary services has not yet started.


## Motivation

Userspace permissions ([UIP-userperms](./UIP-userperms.md)) restricts the capabilities of agents per desk. However, agent code isn't the only place from which system interactions originate: eyre exposes HTTP endpoints that allow interacting with userspace. Eyre's authentication distinguishes between different ship identities, but otherwise treats every session as "root": any requested interaction gets performed unconditionally.  
Giving eyre's authentication some form of desk provenance lets userspace permissions apply to HTTP interactions.

Client code is most commonly accessed in the browser, and served by desks themselves. This access usually happens at the same domain, letting client code from different desks access the same cookies. (Even with `Http-Only` cookies, that don't allow the _value_ to be read out, those cookies can still be _sent_ with requests.) Simply giving each desk a unique cookie provides provenance but is not secure. We must ensure those cookies remain isolated and cannot be leaked or extracted across desk boundaries.


## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Request handling

#### Domains

Eyre MUST store a set of "known domains", for which it expects to handle incoming requests. Eyre MUST support adding and removing domains from this set, and setting the set wholesale, through a `%turf` task. This set MUST contain `localhost` by default.

Per the RFC2616 specification, HTTP requests must have a `Host` header. Eyre already reads this and, for requests coming from the localhost, respects the `Forwarded` header's `Host` field if present. If this process does not deliver a target hostname for the request, eyre MUST serve a `400` bad request response, again per RFC2616. Eyre MUST support cases where the `Host` field contains an IP address rather than a domain name. The bulk of this specification assumes a domain name, for the special handling of IP addresses see [the IP address access section](#ip-address-access).

Eyre MUST define a "**scope**" as `(unit desk)`, where `~` represents "root", and a value represents a specific desk scope. Eyre MUST deduce the target scope for a request based on the domain name provided in its `Host` header in the following way:
1. If the domain name is a known domain, the target scope is root.
2. If the domain name is more than one segment long, and removing the smallest segment (i.e. `ex.ample.org` -> `ample.org`) results in a known domain, the target scope is the smallest segment interpreted as a desk name (i.e. `ex`).
3. Else, there is no target scope.

If a request does not have a target scope, eyre MUST serve a `421` "misdirected request" response.

Eyre MUST define a "**path owner**" as `(unit desk)`, where `~` represents "eyre", and a value represents a specific desk. Eyre MUST deduce the path owner for a request based on the path of its URL in the following way:
1. Resolve the URL to its binding.
2. If the binding maps to a `%gen` action, the path owner is the desk the generator comes from.
3. If the binding maps to an `%app` action, the path owner is the desk the agent is running from.
4. Else, the path owner is eyre.

If the path owner is not eyre, and the path owner does not match the scope, eyre MUST redirect the request to the domain name which would resolve to the appropriate scope using a `307` "temporary redirect" response.  
(For example, `ship.io/myapp` to `mydesk.ship.io/myapp`, or `mydesk.ship.io/~/login` to `ship.io/~/login`.)

If the previous checks have not served a response, eyre MUST proceed to handle the request's authentication.

#### Authentication

The authentication `$session` type MUST include a `scopes=(each (set @uv) @uv)`, where the `%&` case lets root sessions track their corresponding child sessions, and the `%|` case lets child sessions point back to their parent session. More on this in [subdomain authentication](#subdomain-authentication).

The `$identity` associated with an authentication `$session` MUST, in addition to its authentication method and corresponding `@p`, contain a `scope=(unit desk)` to indicate the scope of the session.

When sending interactions to gall, eyre MUST include the scope in its provenance path (in `$sack`, in the `%deal` task). Gall MUST check and pass along this provenance as appropriate. Spider MUST do the same.

```hoon
::  $session: server side data about a session
::
+$  session
  $:  ::  identity: authentication level & id of this session
      ::
      =identity
      ::  scopes:
      ::    %&  this is a root session, and these are its child tokens
      ::    %|  this is a child session, this is its parent
      ::
      ::NOTE  ?=(~ scope.identity) should be %& here, and the inverse.
      ::      if that's not the case, that's a bug!
      scopes=(each (set @uv) @uv)
      ::
      ...
  ==
::  $identity: authentication method & @p, w/ scope
::
+$  identity
  $:  $=  who
      $~  [%ours ~]
      $%  [%ours ~]                                   ::  local, root
          [%fake who=@p]                              ::  guest id
          [%real who=@p]                              ::  authed cross-ship
      ==
    ::
      scope=(unit desk)
  ==
```

If the scope of the request's authentication doesn't match the request's scope, the authentication MUST be considered invalid.  
If invalid or no authentication is provided on a desk scope request, eyre MUST initiate [subdomain authentication](#subdomain-authentication) in response to the request by serving a `307` "temporary redirect". Eyre MUST NOT mint guest sessions for desk scope requests.

If invalid authentication is provided on a root scope request, eyre MUST (as was already the case) serve a `401` "unauthorized" with a `Set-Cookie` header that expires the provided cookie.

### Subdomain authentication

Eyre MUST serve the `/~/login` page exclusively on the root scope. To obtain a cookie for desk scopes, eyre MUST implement a `/~/holm/...` endpoint and use it in the following flow.

1. A request to `/~/holm` comes in on the root scope. The URL has the shape of `/~/holm/sink/[scope](target-url)`.
2. Eyre generates a random temporary token, `tmp-token`, stores it in state along with the requester's session identifier, and sets a 30-second expiry timer. (If the requester did not provide a session identifier, mint a new guest session.)
3. Eyre serves a `307` "temporary redirect" to `//[scope].[hostname]/~/holm/gain/[tmp-token][target-url]`.
4. That request to `/~/holm/...` comes in on the desk scope.
5. Eyre checks the `tmp-token` from the request URL against its state. If a match exists, it removes the token from state and mints a new child session with the matching session as its parent.
6. Eyre serves a `307` "temporary redirect" to the `target-url`, including a `set-cookie` header for the newly minted session.

For requests with out-of-spec URL shapes, eyre MUST serve a `400` "bad request" response.

Eyre MAY implement a `/~/holm/jump(target-url)` endpoint. If a request to this endpoint comes in on a desk scope, eyre MUST redirect to `/~/holm/sink/[scope](target-url)` on the root scope. If a request to this endpoint comes in on the root scope, eyre SHOULD redirect to the `/~/login` page.  
This endpoint enables the runtime to initiate subdomain authentication without needing to parse the current/target scope out of the request hostname, for which knowledge of known domains would be required.

When a session with the root scope gets expired, all its "child" sessions MUST be expired along with it.

### Scoped OAuth-style authentication

Eyre MUST support OAuth-style authentication at the `/~/auth` endpoint. This endpoint MUST only be accessible on the root scope. If the request is not authenticated as the host ship, eyre MUST redirect to `/~/login`.

On GET requests, the endpoint MUST read the following query parameters from the URL:
- `scope`: desk scope being requested,
- `return`: URI to redirect to upon completion,
- `client`: optional, name of the client requesting authentication.

Eyre MUST serve a dialog page describing the authentication request and present the user with "approve" and "deny" buttons. Clicking either MUST submit a POST request containing the `scope` and `return` values.

On POST requests, eyre MUST retrieve `scope` and `return` values from the request body.  
If an `approve` value is present, eyre MUST mint a new session and serve a `303` "see other" redirect to the `return` value's URI, appending `?token=` followed by the minted session's identifier.  
Else, eyre MUST serve a `303` "see other" redirect to the `return` value's URI, appending `?error=rejected`.

Standalone clients SHOULD use `/~/auth` in place of `/~/login` whenever possible.

### IP address access

Eyre MUST store a flag indicating whether it will respond to requests that have an IP address (rather than a domain name) in their `Host` header. This flag MUST be disabled by default.

If the flag is disabled, eyre MUST serve a `421` "misdirected request" responses to requests on IP addresses.

If the flag is enabled, eyre MUST handle requests on IP addresses as per normal, with the following behavioral changes:
- Every request MUST be treated as targetting the root scope.
- Every minted authentication session MUST have the root scope.
- Requests to paths owned by a desk scope MUST respond directly, instead of redirecting to a subdomain.

Regardless of flag state, eyre MUST continue handling requests on domain as normal without any behavioral changes.

Users SHOULD be warned that enabling the flag and accessing third party apps over an IP address makes them exceedingly vulnerable to malicious software. Setup documentation SHOULD NOT recommend enabling the flag as part of normal setup procedure. For local machine setups, documentation SHOULD always instruct to visit `localhost`, never `127.0.0.1`.

### SSL certificates

Eyre MUST support storing SSL certificates for multiple domains. Eyre MUST communicate these to the runtime in its `%set-config` gift. The runtime's `http.c` MUST apply the appropriate certificate based on the request's hostname.

Eyre SHOULD implement a subscription endpoint for listening to changes to the set of known domains.

### Runtime authentication checks

Eyre MUST include the scope of session tokens in its `%sessions` gift to the runtime. The runtime MUST require an appropriate scope when handling requests directly. To this end, eyre MUST include the originating desk alongside `$cache-entry`s and send them to the runtime in `/cache/[aeon]/[url]` scry responses.

If scoped authentication is required, but the requester provides no authentication whatsoever, the runtime MAY redirect to `/~/holm/jump(request-url)` in an attempt to have the client obtain appropriate authentication.

### Ancillary services

ACME agent SHOULD be updated to support negotiating SSL certificates for IP addresses.

The `arvo.network` DNS service provider SHOULD be updated to set wildcard subdomain DNS entries for each ship (`*.sampel.arvo.network`), in addition to the existing ship domain entries. If it does so, the wildcard MUST point to the same location as the ship domain.

The `arvo.network` DNS service provider SHOULD be updated to facilitate setting records for the DNS-01 ACME challenge type. DNS and ACME agents SHOULD be updated to make use of this.

Hosting services SHOULD ensure they handle requests to subdomains appropriately. They SHOULD NOT rewrite URLs in ways that result in all requests to a ship having the same origin from a client's perspective.


## Rationale

The approach taken here has its origin in [`origins.txt`](https://gist.github.com/Fang-/18ce946a2bb33bada3f10bdb3546bb55). That description of the problem and reasoning towards this solution still applies. We opted for subdomains instead of ports to differentiate origins because outward-facing ports are limited in number, very difficult to support in hosting environments, and would require very extensive runtime changes.

Setting cookies from distinct origins is the only way to ensure browsers isolate the cookies in a secure way. `Set-Cookie` supports a number of attributes that provide different degrees of control over when and how cookies can be sent or read, but no combination of them is sufficient for isolation without origin separation.

Cookies need to be served from their respective subdomains, but we do not want to make scopes authoritative over authentication sessions. Making them defer to the root scope's session gives us a clean hierarchy of sessions and enables logging out of all scopes.  
The root scope needs to "legitimize" a subdomain cookie request. Passing a temporary authentication token in the URL is easy, reliable and effective. The redirects are transparent to the user and browsers do not store them in history. Even if they did, the short lifetimes and single-use nature of the temporary tokens help prevent exploitation.

Adding an endpoint for OAuth-style authentication is an accommodation for standalone clients. Authenticating into root with `/~/login` continues to work but is ill-advised, and users should become wary of entering their `+code` anywhere outside of their own ship. Proper OAuth has many features which we do not need or are difficult to support in a personal server context, so we forego real compatibility.

Eyre considers `localhost` known by default because accessing a ship locally should always be valid. Firefox, Chrome and Brave seem to support subdomains for localhost without OS-side configuration.

We keep the eyre binding namespace as a flat path-prefix mapping, instead of giving each desk its own namespace, to ensure that navigating to `/some-other-app-endpoint` continues to work as-is. Though it's uncommon, installing and runnnig desk distributions under non-standard names is possible, so namespacing the bindings would require cross-desk linking to do lookups and construct the target URL dynamically.

### Attempts at hiding subdomains

Because of the UX impact of making subdomains visible to the user, we explored different approached for hiding the subdomains from the user. None of them worked out in practice. We briefly describe them below.

It is possible to hide the subdomains from the user by hiding them inside an iframe served at the root domain. While message passing between cross-origin frames is possible, some browsers (Brave, Chrome) treat the origin of the inner frame as that of the outer page for security checks. As a consequence, the inner frame cannot dynamicall set cookies for the subdomain it's being served from.  
While the iframes approach would have delivered a convenient place to put "userchrome" or "launcher" UI, this presented additional compatibility challenges. For example, link preview fetchers would only retrieve the metadata of the outer page, not the inner frame.

We briefly considered utilizing a browser plugin to manage authentication tokens, rewriting requests to the user's ship on the fly. While this approach seemed technically possible, mobile browsers offer very little plugin support. Supporting mobile web access is important, so we were forced to drop this approach.


## Backwards compatibility

A common pattern for serving files from a desk to the client relies on the docket agent. This agent runs from the landscape desk. As a result, in the new model, all client files would be served under the landscape scope.  
Desks will need to take responsibility for serving their own files. In addition to bespoke one-off solutions, generic file-serving software could be developed to facilitate this. (For example, [foo-fileserver](https://github.com/Fang-/suite/blob/wip/owntracks/app/foo-fileserver.hoon).) This is a good opportunity to return from glob-based file-serving to clay-based file-serving, now that tombstoning is real.

The setup process changes slightly, so documentation will need to be updated. Localhost access continues to work for some browsers, but not all of them. If the user's browser doesn't support localhost subdomains, or if their ship is on a different machine on the local network, IP access mode is available. For setups on internet-accessible machines, documentation should recommend using the `*.arvo.network` service, and describe the required steps from configuring your own domain name if you have one.

For standalone clients, authentication using `/~/login` on the root scope continues to work as it has been, and userspace can be accessed through that scope. But requests to desk-specific resources require following redirects, and will always be behind three redirects if subdomain authentication cookies are not remembered. As such, it is strongly recommended to switch to using scoped authentication using credentials obtained through `/~/auth`.

Hosting environments may need to update their infrastructure to support wildcard subdomains. Self-hosting software may need to do the same.


## Security considerations

In the common case, we depend on the browser's ability to keep cookies isolated between different origins. Luckily browsers have years of experience with this. Care must still be taken around cross-site requests, and the `SameSite` cookie attribute should be considered carefully, but that is unchanged from the status quo.

There is nothing to be gained from "faking" the `/~/auth` page in phishing attempts. As is the status quo, users should take care to only type their `+code` into pages served by their ship.

Using the new authentication endpoints over plain HTTP is vulnerable to man-in-the-middle attacks. But this holds for any and every endpoint and authentication method. We are no worse off, and there is nothing to be done about this. The (unrelated to this UIP) possibility of SSL certs for IP addresses should help improve this situation.

The design provided here and its implementation should get subjected to red-teaming before release.


## Appendices

### Appendix A: eyre endpoints

url | session required? | root or all scopes? | description
--- | --- | --- | ---
`/~/holm`    | y | all  | scoped auth flow
`/~/auth`    | y | root | oauth-like flow
`/~/host`    | n | all  | host ship
`/~/name`    | y | all  | authenticated identity
`/~/login`   | n | root | login form & login handling
`/~/logout`  | n | all  | logout handling
`/~/eauth`   | y | root | cross-ship authentication
`/~/channel` | y | all  | userspace interactions
`/~/scry`    | y | all  | userspace reads
`/~/ip`      | n | all  | requester ip
`/~/boot`    | n | all  | jam of network state of ship in url (or 404)
`/~/sponsor` | n | all  | galaxy-level sponsor of ship in url

### Appendix B: livenet /~/login POST request handling

- 400 responses serve the `/~/login` page as body
- no or invalid body (query/form data encoding): **400**
- `eauth` present in body:
  - no or invalid `@p` `name` present in body: **400**
  - delegate to eauth, server side flow (NOTE: even if `name` is `our`!)
- no or invalid `password` in body: **400**
- close session that made the request
- start `%local` session
- re-associate the connection (for `+handle-response`)
- potentially overwrite public hostname
- put session token into response body
  - response header *always* gets `set-cookie` with session token
- if no redirect: **200**
- if redirect: **303** with `location` header

### Appendix C: livenet /~/logout request handling

- any method goes
- every response is a **303** redirect to `/~/login`
- body parsed as query/form data args, fallback to empty list
- if `sid` not present, fallback to requester's session id
- if `host` present, it's the eauth host, `sid` is the remote session nonce:
  - send an eauth `%shut` to the host, put that into the host's pending msgs queue
  - serve response
- if `all` present, will close all sessions with the same identity
- if `sid` is the requester's own session id, include a `set-cookie` header in the response
- close the requested session(s) in state as requested
- serve response
