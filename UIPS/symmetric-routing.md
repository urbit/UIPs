---
title: <Symmetric Routing>
description: <Next-Generation Network Routing and Peer Discovery>
author: <~rovnys-ricfer>
discussions-to: <TODO>
status: Draft
type: <Standards Track>
category: <Kernel>
created: <2023-06-13>
---

## Abstract

This proposal defines a routing system for Ames and Fine, Urbit's network protocols, which is "symmetric": the trajectory of a response packet through the network will be the exact reverse of the trajectory of the request packet to which this packet is responding.

Symmetric routing has better properties for NAT traversal than the current asymmetric routing, it promises to be simpler and easier to reason about, it lets stars participate in packet forwarding, and it moves routing responsibility away from publisher ships toward subscribers, increasing scalability.

## Motivation

Current Ames routing is hard to reason about, does not handle NAT traversal elegantly, and does not support stars participating in packet forwarding.  By constraining all routes to be symmetric, the problem space is shrunk, and some NAT traversal problems are defined away.

## Specification

When a ship receives a request packet, it figures out where the next hop in the forwarding chain for that packet should be, forwards the packet to that location, and remembers the IP and port of the incoming request packet for thirty seconds, so if it hears a response packet, it can pass that response back to the previous relay.

This means that every request must be satisfied within thirty seconds, but that's a workable constraint.  Note that many HTTP systems operate under this constraint.  This arrangement is also similar to NDN's "pending interests" table.  An NDN "interest" packet corresponds almost exactly to a Fine scry request packet, and stateful Ames request packets can behave the same with respect to routing.

In order for this to work, each node has to have some idea of the next node to try.  If the node has a direct route to the receiver ship, it should send the packet there.  Otherwise, if it knows the ship's sponsor, it should try that, falling back up the sponsorship chain potentially all the way to the receiver's galaxy.

The next question is how routes tighten, which requires information about lanes for ships later in the relay chain to propagate backward to ships earlier in the relay chain.  A maximally tight route is a direct route between the sender ship and the receiver ship.

Each ship in the relay chain, when forwarding a response packet backward through the route, can overwrite a "next" field in the response packet to contain the relay just past it.  This enables the previous relay to try the "next" relay on the next request packet, skipping the relay in between.

One simple rubric is that a ship should try the tightest known-good route and the next tightest route simultaneously until the tighter route has been demonstrated to be viable.  This way, in case the tighter route turns out not to work (usually because it's behind a firewall and not publicly addressable), the looser route will still get the packet through.

A ship should only overwrite the "next" field when forwarding a response back to a ship that is not one of its direct sponsors (this precludes fraternization from galaxy directly to planet or moon, but I think that's ok).  The following example shows how a route tightens over multiple packet roundtrips.

```
request:  ~rovnys -> ~syl -> ~sipsyl -> ~ritpub-sipsyl -> ~mister-ritpub-sipsyl
response: ~rovnys <- ~syl <- ~sipsyl <- ~ritpub-sipsyl <- ~mister-ritpub-sipsyl
                  ::
                  :: next: ~sipsyl@123.456.789:1234

request:  ~rovnys -> ~sipsyl -> ~ritpub-sipsyl -> ~mister-ritpub-sipsyl
response: ~rovnys <- ~sipsyl <- ~ritpub-sipsyl <- ~mister-ritpub-sipsyl
                  ::
                  :: next: ~ritpub-sipsyl@234.567.890:2345

request:  ~rovnys -> ~ritpub-sipsyl -> ~mister-ritpub-sipsyl
response: ~rovnys <- ~ritpub-sipsyl <- ~mister-ritpub-sipsyl
                  ::
                  :: next: ~mister-ritpub-sipsyl@345.678.901:3456

request:  ~rovnys -> ~mister-ritpub-sipsyl
response: ~rovnys <- ~mister-ritpub-sipsyl
                  ::
                  :: next: ~
```

As an aside, note that while this example was written using full Urbit ships as relays, this routing procedure would also allow any kind of program that speaks the routing protocol and knows the lanes of ships to act as a relay node, even if it does not have an Urbit address (although the system would need a new way to learn about these non-ship relays).

In order for this procedure to work, each relay must maintain a data structure:

```
+$  state
  $:  pending=(map request [=lane expiry=@da])
      sponsees=(map ship lane)
  ==
+$  request
  $%  [%ames sndr=ship rcvr=ship]
      [%scry rcvr=ship =path]
  ==
```

The `expiry` field is set to three minutes from now whenever we hear a request, since the packet re-send backoff interval is two minutes.  Repeated requests bump the timeout, keeping the request alive.

## Rationale

Other routing proposals, for comparison:
- [Earlier Symmetric Routing](https://gist.github.com/belisarius222/3b808fc3fe6d9aa622cc87c9bf6a9a86)
- [Nan Madol](https://gist.github.com/belisarius222/4ae249c07d9e169b38b4e9f57e0eced4): wireguard-based secure channels
- [Ad Fontes](https://gist.github.com/belisarius222/7f8452bfea9b199c0ed717ab1778f35b): scry-maximalist networking with prefix-dependent routing

Compared to the earlier symmetric routing proposal, this self-assembles better and requires less state on each node.  Compared to Nan Madol and Ad Fontes, this is a much more incremental change, and Ad Fontes could use this routing system for its default transport protocol.

This routing system is inspired by the "pending interest table" in the Named Data Networking project, started by Van Jacobson.

## Backward Compatibility

Symmetric routing should be added alongside the older asymmetric routing system to prevent ships from losing communication with each other.  At some later date, the old routing protocol could be sunsetted and eventually removed, or at least removed from any ship other than ships who are sponsoring ships that are multiple years out of date.

A Kelvin should be burned for this update, as well as an Ames and Fine protocol version.

TODO protocol negotiation: send both a new and old packet simultaneously and see which ones work?

## Security Considerations

TODO DoS protection

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
