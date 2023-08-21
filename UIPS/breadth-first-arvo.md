---
uip: 0114
title: Breadth-First Arvo
description: Breadth-First Move Processing in the Arvo Kernel
author: ~wicdev-wisryt, ~rovnys-ricfer
status: Draft
type: Standards Track
category: Kernel
created: 2023-08-19
---

## Abstract

This proposal modifies the Arvo kernel's move processing to process moves in a breadth-first order rather than the historical depth-first order.  This ensures chronological move processing order, allows for a new concept of simultaneity among vane activations, and should be easier to reason about than depth-first move processing.

The second stage of this work will add a "tick" to Arvo that increments each time it runs a vane.  This tick will be used as the `case` in scry requests to request data at the current tick, replacing date-based requests at `now`, which violate referential transparency when run multiple times within an event.

Large portions of this proposal are adapted or copied verbatim from [`~wicdev-wisryt`'s original pull request from `~2022.10.7`](https://github.com/urbit/urbit/pull/6041).

## Motivation

Breadth-first ordering guarantees that moves will be processed in the order they are emitted.  Depth-first ordering constantly violates this, and in the presence of reentrancy this can cause extremely unexpected results.

The usual argument for depth-first ordering is that it feels like it should allow greater composability. In both cases, if A sends moves to B and C, then A knows that the move to B will happen before the move to C. However, in depth-first ordering, we further know that anything that B causes to happen will also happen before the move to C. This means the move to B will "fully complete", even if it delegates some of the work to another entity. Of course, this doesn't apply if B needs to send something over the wire to fully complete.

Another guarantee that depth-first ordering gives you is that the first move you send will be processed immediately, with no intervening moves.  Breadth-first ordering often will process other moves that were already in the queue.

Overall, the guarantees that depth-first ordering gives are rarely useful because they can be violated in practice, while the guarantees that breadth-first ordering gives are consistent and more useful.  Moves being processed out of chronological order in depth-first processing is particularly difficult to reason about and work with.

One place this comes up in the kernel is in breach handling: when Jael that another ship breached, ideally Ames, Clay, and Gall would all hear about the breach "simultaneoulsy", meaning none of those vanes would be able to emit moves to any of the other ones before they had all heard about the breach.  Otherwise Gall might, for example, try to resynchronize with the breached ship and emit a move to Ames, but if Ames hasn't heard about the breach yet, Ames will emit moves to the stale, pre-breach version of the ship that breached, and the new moves will be lost.

Consider an example Arvo event.  A keypress to Dill triggers a poke to the `%hood` agent, which pokes the `%dojo` agent.  `%dojo` emits four `%fact` moves to `%hood`.  Two of these `%fact` from `%dojo` cause `%hood` to give a `%fact` to Dill.  The first `%fact` to Dill triggers it to give three `%blit` moves to Vere, and the second `%fact` triggers one more `%blit` move.

The current depth-first `|verb` command-line trace for this event looks like this:

```
["" %unix %belt /d/term/1 ~2022.10.27..06.32.09..30db]
["|" %pass [%dill %g] [[%deal [~zod ~zod] %hood %poke] /] ~[//term/1]]
["||" %give %gall [%unto %poke-ack] i=/dill t=~[//term/1]]
["||" %pass [%gall %g] [[%deal [~zod ~zod] %dojo %poke] /use/hood/0w2.efXKi/out/~zod/dojo/drum/phat/~zod/dojo] ~[/dill //term/1]]
["|||" %give %gall [%unto %poke-ack] i=/gall/use/hood/0w2.efXKi/out/~zod/dojo/drum/phat/~zod/dojo t=~[/dill //term/1]]
["|||" %give %gall [%unto %fact] i=/gall/use/hood/0w2.efXKi/out/~zod/dojo/1/drum/phat/~zod/dojo t=~[/dill //term/1]]
["||||" %give %gall [%unto %fact] i=/dill t=~[//term/1]]
["|||||" %give %dill %blit i=/gall/use/herm/0w2.efXKi/~zod/view/ t=~[/dill //term/1]]
["|||||" %give %dill %blit i=/gall/use/herm/0w2.efXKi/~zod/view/ t=~[/dill //term/1]]
["|||||" %give %dill %blit i=/gall/use/herm/0w2.efXKi/~zod/view/ t=~[/dill //term/1]]
["|||" %give %gall [%unto %fact] i=/gall/use/hood/0w2.efXKi/out/~zod/dojo/1/drum/phat/~zod/dojo t=~[/dill //term/1]]
["||||" %give %gall [%unto %fact] i=/dill t=~[//term/1]]
["|||||" %give %dill %blit i=/gall/use/herm/0w2.efXKi/~zod/view/ t=~[/dill //term/1]]
["|||" %give %gall [%unto %fact] i=/gall/use/hood/0w2.efXKi/out/~zod/dojo/1/drum/phat/~zod/dojo t=~[/dill //term/1]]
["|||" %give %gall [%unto %fact] i=/gall/use/hood/0w2.efXKi/out/~zod/dojo/1/drum/phat/~zod/dojo t=~[/dill //term/1]]
```

We can split this up into a few pieces to see it more clearly, adding a blank line befre each time Gall processes a `%fact` from Dojo.

```
["" %unix %belt /d/term/1 ~2022.10.27..06.32.09..30db]
["|" %pass [%dill %g] [[%deal [~zod ~zod] %hood %poke] /] ~[//term/1]]
["||" %give %gall [%unto %poke-ack] i=/dill t=~[//term/1]]
["||" %pass [%gall %g] [[%deal [~zod ~zod] %dojo %poke] /use/hood/0w2.efXKi/out/~zod/dojo/drum/phat/~zod/dojo] ~[/dill //term/1]]
["|||" %give %gall [%unto %poke-ack] i=/gall/use/hood/0w2.efXKi/out/~zod/dojo/drum/phat/~zod/dojo t=~[/dill //term/1]]

["|||" %give %gall [%unto %fact] i=/gall/use/hood/0w2.efXKi/out/~zod/dojo/1/drum/phat/~zod/dojo t=~[/dill //term/1]]
["||||" %give %gall [%unto %fact] i=/dill t=~[//term/1]]
["|||||" %give %dill %blit i=/gall/use/herm/0w2.efXKi/~zod/view/ t=~[/dill //term/1]]
["|||||" %give %dill %blit i=/gall/use/herm/0w2.efXKi/~zod/view/ t=~[/dill //term/1]]
["|||||" %give %dill %blit i=/gall/use/herm/0w2.efXKi/~zod/view/ t=~[/dill //term/1]]

["|||" %give %gall [%unto %fact] i=/gall/use/hood/0w2.efXKi/out/~zod/dojo/1/drum/phat/~zod/dojo t=~[/dill //term/1]]
["||||" %give %gall [%unto %fact] i=/dill t=~[//term/1]]
["|||||" %give %dill %blit i=/gall/use/herm/0w2.efXKi/~zod/view/ t=~[/dill //term/1]]

["|||" %give %gall [%unto %fact] i=/gall/use/hood/0w2.efXKi/out/~zod/dojo/1/drum/phat/~zod/dojo t=~[/dill //term/1]]

["|||" %give %gall [%unto %fact] i=/gall/use/hood/0w2.efXKi/out/~zod/dojo/1/drum/phat/~zod/dojo t=~[/dill //term/1]]
```

We can see that Arvo processes everything downstream of the first `%fact` before processing the next `%fact`.  Contrast this with a trace from a breadth-first version of Arvo:

```
["" %unix %belt /d/term/1 ~2022.10.27..06.32.57..1d2c]
["|" %pass [%dill %g] [[%deal [~zod ~zod] %hood %poke] /] ~[//term/1]]
["||" %give %gall [%unto %poke-ack] i=/dill t=~[//term/1]]
["||" %pass [%gall %g] [[%deal [~zod ~zod] %dojo %poke] /use/hood/0w3.qnAr5/out/~zod/dojo/drum/phat/~zod/dojo] ~[/dill //term/1]]
["|||" %give %gall [%unto %poke-ack] i=/gall/use/hood/0w3.qnAr5/out/~zod/dojo/drum/phat/~zod/dojo t=~[/dill //term/1]]
["|||" %give %gall [%unto %fact] i=/gall/use/hood/0w3.qnAr5/out/~zod/dojo/1/drum/phat/~zod/dojo t=~[/dill //term/1]]
["|||" %give %gall [%unto %fact] i=/gall/use/hood/0w3.qnAr5/out/~zod/dojo/1/drum/phat/~zod/dojo t=~[/dill //term/1]]
["|||" %give %gall [%unto %fact] i=/gall/use/hood/0w3.qnAr5/out/~zod/dojo/1/drum/phat/~zod/dojo t=~[/dill //term/1]]
["|||" %give %gall [%unto %fact] i=/gall/use/hood/0w3.qnAr5/out/~zod/dojo/1/drum/phat/~zod/dojo t=~[/dill //term/1]]
["||||" %give %gall [%unto %fact] i=/dill t=~[//term/1]]
["||||" %give %gall [%unto %fact] i=/dill t=~[//term/1]]
["|||||" %give %dill %blit i=/gall/use/herm/0w3.qnAr5/~zod/view/ t=~[/dill //term/1]]
["|||||" %give %dill %blit i=/gall/use/herm/0w3.qnAr5/~zod/view/ t=~[/dill //term/1]]
["|||||" %give %dill %blit i=/gall/use/herm/0w3.qnAr5/~zod/view/ t=~[/dill //term/1]]
["|||||" %give %dill %blit i=/gall/use/herm/0w3.qnAr5/~zod/view/ t=~[/dill //term/1]]
```

These are the same 15 lines, and they're each at the same depth, but they're in a different order.  Most code we write is agnostic to this order, because there are many circumstances where this order gets inverted compared to our expectations.

The most obvious difference is that the four %facts from dojo to hood happen one right after the other instead of being mingled with other moves.  If some of those intermingled moves invoked dojo (which would be a form of reentrancy) and caused dojo to emit more facts, those facts would be given to hood *before* the %facts which were already on the stack to be sent to hood.  If this is textual output, then it will be in reverse order.  The breadth-first move order fixes this problem completely by running all the facts that were issued at the same time before processing the moves that those themselves produced.

Here we can see the main advantage of depth-first ordering at work: moves are processed in the order they are emitted.

## Specification



## Backward Compatibility


## Test Cases


## Reference Implementation


## Security Considerations


## Copyright Waiver

Copyright and related rights waived via [CC0](../LICENSE.md).
