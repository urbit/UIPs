---
uip: "XXXX"
title: %mach
description: TCP vane
author: ~bonbud-macryg
status: Draft
type: Standards Track
category: Kernel
created: 2026-10-15
---

## Abstract

A new vane, Mach, serves as a Transmission Control Protocol (TCP) client for Gall agents. Mach is a TCP client in its first release, and may be upgraded to function as a TCP server after its first release. Gall agents can use Mach to open and close communication channels with servers such as Bitcoin nodes, running locally or remotely, without need for a complex sidecar integration. A general-purpose TCP client/server vane completes the system of Urbit networking: where Ames is ship-to-ship and [Lick](https://github.com/urbit/UIPs/blob/main/UIPS/UIP-0101.md) ship-to-host, Mach opens the way to ship-to-server communication across any application-level protocol built on top of TCP/IP.

## Status

Mach is planned for release on Groundwire’s Arvo distribution in a 407k version bump. Mach PRs will target the upstream `urbit/urbit` and `urbit/vere` repos, to be merged as the Core Guild sees fit.

## Motivation

Groundwire needs to read the Bitcoin blockchain in order to populate its Urbit PKI. This requires at minimum a Bitcoin light client running on the ship, which we have at the `gwbtc/node` repo. In our current implementation this desk’s `%bitcoin-client` agent talks to another agent, `%tcp-sidecar`, which is itself just a passthrough via Lick to a TCP sidecar written in C. In practice getting this sidecar set up for end-users, keeping it alive across Vere lifecycles, and securing this pipeline is unacceptably complex and unreliable. This approach is a non-starter if we want end-users' Urbit ships to be portable across host machines.

Urbit has depended on sidecars to communicate with two kinds of thing: an external process such as a local LLM, which was obviated by Lick, and any kind of crypto wallet software. Urbit's original Bitcoin wallet relied on a three-part stack of `%btc-wallet` (the logical agent) `%btc-provider` (an interface to the sidecar) and `bitcoin-rpc`, which caused issues like [subscribed ships slowing down](https://github.com/urbit/urbit/issues/5072) to [becoming inoperable](https://github.com/urbit/landscape-archive/issues/792#issuecomment-870107217) because of bad transactions between ships and their sidecars.

A TCP client vane will mitigate against these kinds of issues in two ways: allowing Gall agents to interact directly with remote peers over TCP, not a brittle sidecar process, and setting sensible constraints on how those agents and peers can interact with eachother.

Few Bitcoin nodes support JSON RPC and there is no reliable oracle to discover such nodes. An Urbit that does not support TCP natively forces us to take one of two options: either hard-code one or more Bitcoin RPC nodes that everyone contacts (much the same situation we are in now with Tlon's Infura endpoint for Ethereum) or force some/all users to run sidecars to enable RPC, which is what `bitcoin-rpc` was for. The only reliable, non-centralizing approach is for Urbit to be able to speak the languages of the networks we want it to talk to.

### TCP Server

Mach may later be extended to serve incoming connections, which would let TCP-based protocols (SSH, IMAP/SMTP, WebSockets) be implemented as ordinary Gall agents. A server would introduce port binding to Vere and create a significant new attack surface to Vere, and it’s beyond what Groundwire needs from Arvo right now.

## Specification

In its first release, Mach SHALL be a minimal interface to a Mach driver in the runtime.

### Interface

Agents name their connections, as they name Lick ports. A `$name` is a path, and Gall prefixes it with the agent's name, so two agents can use the same name without clashing. Mach addresses a connection's peer with a `$addr`, an IPv4 or IPv6 address and a port:

```hoon
+$  addr
  $%  [%if p=@if q=@ud]                                 ::  ipv4
      [%is p=@is q=@ud]                                 ::  ipv6
  ==
```

Mach does not resolve domain names. An agent must resolve a domain to an address first (see Rationale).

#### Tasks, gifts

An outgoing TCP connection MAY close for a number of reasons. A `$wye` namespaces those reasons, and Gall agents MAY branch on error conditions with `?-(-.wye ...)` or `?=([%peer %dead] wye)`.

```hoon
++  mach  ^?
  |%
  +$  addr
    $%  [%if p=@if q=@ud]                               ::  ipv4
        [%is p=@is q=@ud]                               ::  ipv6
    ==
  +$  name  path
  ::  $wye: why a connection closed
  ::  by category, then cause
  ::
  +$  wye
    $%  $:  %connect                                    ::  never opened
            $=  p
            $?  %invalid                                ::  malformed $addr
                %taken                                  ::  name in use
                %refused                                ::  connect failed
                %timeout                                ::  not open in time
        ==  ==
        $:  %peer                                       ::  peer ended it
            $=  p
            $?  %done                                   ::  peer closed cleanly
                %reset                                  ::  reset or r/w failed
                %dead                                   ::  peer unresponsive
        ==  ==
        $:  %close                                      ::  userspace ended it
            $=  p
            $?  %done                                   ::  flushed outgoing
                %timeout                                ::  unsent after timeout
        ==  ==
        $:  %local                                      ::  arvo/vere ended it
            $=  p
            $?  %reborn                                 ::  vere restarted
                %trimmed                                ::  dropped on %trim
                %dropped                                ::  %received failed
                %overflowed                             ::  send queue full
        ==  ==
    ==
  ::
  +$  task
    $~  [%vega ~]
    $%  $>(%born vane-task)                             ::  new unix process
        $>(%trim vane-task)                             ::  trim state
        $>(%vega vane-task)                             ::  report upgrade
        [%close =name]                                  ::  close connection
        [%send =name =octs]                             ::  send bytes
        [%connect =name =addr]                          ::  connect to address
        [%opened id=@ud]                                ::  vere opens
        [%closed id=@ud p=wye]                          ::  vere closes
        [%received id=@ud =octs]                        ::  vere receives bytes
    ==
  ::
  +$  gift
    $%  [%connected =name]                              ::  opened
        [%received =name =octs]                         ::  received bytes
        [%closed =name p=wye]                           ::  connection closed
        [%connect id=@ud =addr]                         ::  vere, open connection
        [%send id=@ud =octs]                            ::  vere, send bytes
        [%close id=@ud]                                 ::  vere, close connection
    ==
  --
```

Every gift to an agent carries the connection's name. Only the duct that opened a connection MAY `%send` on it or `%close` it; Mach MUST ignore these tasks from any other duct. A `%connect` on a name already open MUST close with `[%connect %taken]` and leave the connection open. Mach maps each name to a numeric id for the driver, so the driver never sees names.

#### Scries

* `/ma/connections`: list the names of open connections
* `/md/owner/<name>`: the duct that opened this connection
* `/mu/<name>`: does this connection exist

### Lifecycle

Upon first installation, Mach MUST initialize a default `//mach` duct as bunt `unix-duct=_[//mach ~]`, as Lick does.

On `%born`, Mach MUST drop all outgoing connections, as Iris and Lick do, and tell the Gall agents that own them about these dropped connections with `[%local %reborn]`. On `%trim`, Mach MUST close every connection, telling each owner with `[%local %trimmed]` and the driver with `%close`. Once Gall resource tracking is implemented, Gall SHOULD tell Mach to clean up an agent’s outgoing connections when the agent is nuked.

### Runtime

In its first release the Mach vane is a minimal interface with the Mach driver. Mach's driver MUST defer to Gall agents to frame incoming messages according to the protocol they implement. As such, Mach cannot frame incoming packets into complete messages on behalf of the agent. All congestion control, bindings, timeouts, etc. MUST be specified as hard-coded constants in the driver.

The driver MUST limit the size of each receive event, and maintain per-connection queues (one incoming, one outgoing) where one `%received` gift is in transit per connection until the agent has processed it, and up to 16 MiB is queued to go out. These queues abstract over host platforms' own TCP buffers, which vary per platform, per configuration, and may dynamically resize per-connection.

If Vere crashes before the event is written to disk, the event is lost, and on restart `%born` closes the connection with `[%local %reborn]`, so an agent never sees an interrupted bytestream.

The driver MUST enforce the following constraints. Most of these were taken from the TCP sidecar that Mach replaces, and others were picked to be generous enough for userspace to implement protocols within. These constraints are simple constants that MAY be tweaked in response to the needs of userspace developers.

* `%received` events MUST carry no more than 1 MiB
* When the incoming queue reaches 1 MiB, the driver MUST stop reading from the socket until a `%received` event takes the queue, leaving further flow control to TCP
* The outgoing queue MUST be capped at 16 MiB, closing the connection with `[%local %overflowed]` on overflow
* Outgoing connection attempts MUST time out after 30 seconds, closing with `[%connect %timeout]`
* On `%close`, the driver SHALL stop delivering received bytes, flush the outgoing queue, and close with `[%close %done]`, or with `[%close %timeout]` after a 30 second drain limit
* The driver MUST specify TCP keepalive with hard-coded timing applied consistently on every host platform: 120 seconds idle before the first probe, 10 seconds between probes, and 6 unanswered probes before closing with `[%peer %dead]`
* The driver MUST close with `[%peer %dead]` when sent data stays unacknowledged for 180 seconds, on every host platform; the host platform MAY close such a connection sooner
* The driver MUST close with `[%peer %dead]` when it holds unsent bytes and none of them reach the host OS for 180 seconds, so a peer that stops reading cannot hold a connection open

The keepalive timing is an upper bound. If a Gall agent needs faster detection, or its protocol requires its own ping (Bitcoin P2P, MQTT, IRC, WebSocket), the agent MAY send pings over the connection and `%close` it early when they go unanswered.

The above constraints protect from resource exhaustion, make Arvo and Vere behavior consistent across host platforms, and place constraints on what Gall agents can do with TCP. Conversely, the driver SHALL run with `TCP_NODELAY` on, such that batching is left to the agents and peers that understand whichever protocol they’re communicating over.

The driver SHALL connect with libuv to the one address in the `$addr`. If the connect fails before the timeout, the driver closes with `[%connect %refused]`.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

## Rationale

WebSocket support was added to Groundwire’s distribution in March 2026 following ~[UIP-0125](https://github.com/urbit/UIPs/blob/main/UIPS/UIP-0125.md)~: the client in Iris and `cttp.c`, the server in Eyre and `http.c` (~[gwbtc/urbit\#23](https://github.com/gwbtc/urbit/pull/23)~, ~[gwbtc/vere\#4](https://github.com/gwbtc/vere/pull/4)~). That split a single protocol across two vanes and two drivers. A TCP vane would have contained it in a userspace.

This vane should not set a precedent that unsupported protocols should be candidates for support in either the kernel or the `%base` desk. The entire class of protocols that builds on top of TCP should be, by default, excluded from those codebases *because* this vane enables them to be implemented in userspace. HTTP/HTTPS is unique in that Arvo builds on it for user authentication in the browser, the `/~debug` dashboard, and streaming logs to the runtime.

## Backwards Compatibility

A Kelvin decrement will be necessary.

The main backwards compatibility issue is between the `gwbtc/{urbit,vere}` repos and upstream `urbit/{urbit, vere}` repos.

This will require some careful scoping of branches and upstream PRs on Groundwire’s part, but releasing a 407k to Groundwire users while mainstream Urbit remains on 408k is no stranger than the “alt-408k” we’ve already been distributing for around eight months. A prerequisite for this will be rebasing Groundwire’s release from `urbit/urbit:next/kelvin/407`.

Our goals are to ensure as much continuity as possible between Groundwire Alpha releases, and to make it as easy as possible to eventually upstream Groundwire’s kernel changes to `urbit/urbit`. Network compatibility between Groundwire comets and Azimuth ships is already achieved by the Suite C changes outlined in [UIP-0136](https://github.com/urbit/UIPs/blob/main/UIPS/UIP-0136.md). We have explicitly disclaimed continuity of Urbit data and Groundwire identity between our Alpha release and a future Beta / production release.

## Reference Implementation

See pull requests [`urbit/urbit#7437`](https://github.com/urbit/urbit/pull/7437) and [`urbit/vere#1149`](https://github.com/urbit/vere/pull/1149) for the current implementation.

## Security Considerations

Mach’s initial release will only allow Gall agents to connect to outside servers, never the other way around. The runtime driver’s tasks are only accepted from the `//mach` duct. We defer to the Gall Working Group’s userspace permissions system to manage which agents and desks can pass tasks to Mach.

An early draft of Mach supported TLS in a case where the agent tried to `%connect` to a domain, but domain support was removed as it was not strictly necessary for our intended use-cases. TLS support would require the Mach driver to either depend on h2o or run OpenSSL over libuv, which would introduce more security considerations.

As with Iris, any Gall agent can reach any TCP endpoint the host OS can, including loopback. This is a userspace security question and out-of-scope for this UIP.

The Mach driver's hard-coded constraints protect from incoming DoS attacks by limiting how much memory a remote peer can allocate on the host OS: around 2 MiB per connection (1 MiB in the driver, 1 MiB in the host kernel's TCP module) with a 180-second timeout. Mach also limits outgoing connections to 16 MiB queues, but like Iris it does not prevent a Gall agent from opening as many connections as it likes. The 16 MiB limit is generous enough to enable many use-cases (e.g. Bitcoin blocks) but will overflow in the case where a malicious or broken agent spams the connection with outgoing messages. Once userspace permissions are implemented, it is up to the user to be careful about which desks they allow to use Mach to allocate resources on their machine. These kinds of questions around resource allocation on the host machine will be the main concern in any effort to upgrade Mach to a general-purpose TCP client/server vane.

## Copyright

Copyright and related rights waived via [CC0](/urbit/UIPs/blob/main/LICENSE.md).
