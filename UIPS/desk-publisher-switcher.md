---
title: desk publisher switcher
description: Allow desk publishers to tell subscribers to switch to a new source for updates.
author: ~tinnus-napbus
status: Draft
type: Standards Track
category: Kernel
created: 2023-07-28
---

## Abstract

Add a `%kiln-change-source` poke to `%kiln` that changes all active syncs from
`old=[ship desk]` to `new=[ship desk]`. This poke can be done from the local
ship or from a remote ship iff that ship is the current update source.

Add a `%kiln-change-provider` poke to `%kiln` that scries out the subscribers
for the given `desk`, filters them for `%sing` `%w` subs to `let+1` (what
`%kiln` uses), then sends them all a `%kiln-change-source` poke asking them to
switch to a new `[ship desk]`. This allows a provider to migrate app
distribution to a new ship/desk.

## Motivation

If you're currently distributing an app from ship A and want to change to ship
B, you have to try and tell everyone to manually switch. Alternatively, you can
push an update to your app that pokes kiln to perform this action in
`++on-load`. Both are inconvenient for the app publisher.

For example, the Foundation wants to consolidate app distribution to a star
from the multiple ships that Foundation-developed apps are currently
distributed from. Or, an app developer started off distributing apps from their
personal ship or moon but want to switch to a star.

## Specification

Kiln gets two new pokes: `%kiln-change-source` of `[old=dock new=dock]` and
`%kiln-change-provider` of `[syd=desk her=ship sud=desk]`. The former can be
done from the local ship or a remote ship iff the remote ship is the current
update source.

When `%kiln` gets a `%kiln-change-provider` poke, it scries out the subscribers
for the given `desk` from Clay, filters them for `%sing` `%w` subs to `let+1`
(what `%kiln` uses), and pokes them all with `%kiln-change-source` the new
`[ship desk]` source.

The `%kiln-change-source` poke doesn't change the `zest` of the desk, it just
changes the update source, so it won't do things like reactivate suspended
desks, unlike `|install`.

A `+change-provider` generator is added to `gen/hood` to do this.

It's theoretically possible for a small number of subscribers to not receive
the poke if, due to something like networking issues, they received the
previous update but then didn't manage to resubscribe for the next one. Even
so, switching all but a couple of subscribers is still very useful & such cases
can fairly easily be handled by ordinary technical support.

## Rationale

Kiln indirectly handles subscriptions via Clay, so it's not possible to simply
get a list of kiln subscribers. Scrying Clay for `%w` `%sing` subs to `let+1`
could theoretically capture subs that aren't from kiln, but that shouldn't be a
problem because if those ships don't have kiln subs the poke just won't do
anything.

## Backwards Compatibility

If ships are behind on OTAs and don't support `%kiln-change-source`, they just
won't be switched, so it's no worse than the current situation. This would only
be a problem after initial release, eventually all ships will catch up.

## Reference Implementation

[https://github.com/urbit/urbit/tree/desk-publisher-switcher](https://github.com/urbit/urbit/tree/desk-publisher-switcher)

## Security Considerations

There is a security question of letting app publishers change update sources
without user input, but it doesn't seem worse than what they can already do & I
think it's reasonable to transitively trust the new source based on trust of
the existing publisher.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
