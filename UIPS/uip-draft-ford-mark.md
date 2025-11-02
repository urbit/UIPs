---
title: Ford rune for +grad arm inheritance in marks
description: Delegation of mark revision control is done with a new Ford rune, making the dependency tree discoverable by parsing only Ford headers.
author: ~dozreg-toplud (@quodss), ~midden-fabler (@midden-fabler)
status: Draft
type: Standards Track
category: Kernel
created: 2025-10-21
---

## Abstract

We propose moving revision control core delegation in marks from the product of a given arm of a mark core to the Ford header by declaring the delegation with a new Ford rune. Doing so would allow the build system to discover the dependency graph of a file by parsing static Ford declarations, without having to build the file or its dependencies. We also propose to remove some unused mark definition features to simplify the build system.

## Motivation

Current work on Ford refactoring to use persistent memoization instead of the explicit Ford cache (Ford Lightning) hit a roadblock in mark build process. In the current Ford Lightning design when a file is built its Ford header is parsed first, finding its dependencies. If the file has no dependencies, it forms a leaf of a dependency graph, otherwise the process is applied recursively to all dependencies while checking for recursion. What is left is a dependency graph with Hoon content, which would be then built into a vase with a persistently memoized function.

When a mark is built as a mark core, and not as a file, this design breaks down due to the treatment of some arms by the build system, e.g. `+grad` arm. If the product of that arm is an atom, Ford treats it as a mark name, and tries to build that mark together with mark conversion gates between the marks. To know the dependency graph of a mark we would have to build it as a file, and doing so requires having to build all its dependencies, ruining the one directional `dependency graph -> vase` approach. This may cause an infinite loop if a file that specifies a mark also has the same mark, a case which would be correctly recognized as an infinite loop in the current design of Ford. If the `+grad` arm delegation was declared statically in the Ford header instead of dynamically as a product of `+grad` arm, this would not be an issue.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Delegation rune

The UIP proposes a new Ford rune (provisionally `/#`) to declare `+grad` arm delegation in a mark file.

`/#` rune MUST be followed by Hoon gap, followed by a term to indicate which arm is being delegated, followed by a gap, `%` and a term to indicate to which mark the revision control is delegated:

```hoon
::  example: +grad arm delegation
/#  grad  %mime
```

There MUST NOT be two or more `/#` runes in the Ford header of a Hoon file with the same arm specifier.

If the file with `/#` rune is built as a regular file, `/#` rune MUST NOT have any effect.

If the file with `/#` rune is built as a mark file, the core to which the Hoon contents of the file is evaluated MUST NOT have an arm specified in the first child of the `/#` rune.

The set of arm names that MUST be recognized by `/#` rune are:
- `grad`

### Mark specification changes

In addition to the proposal above, this UIP also proposes changing the mark core definition interface:

- The product of `+grab` arm MUST NOT be an atom
- `+jump` arm SHOULD NOT be present in the definition

## Rationale

In addition to simply removing arm delegation via mark arm magic, the proposal also adds `/#` rune for ergonomics.  It is possible to use existing Ford runes in order to achieve the same effect as arm delegation, but it requires nontrivial amount of boilerplate, for example in `/mar/atom/hoon`:

```hoon
::
::::  /hoon/atom/mar
  ::
/?    310
/%  deg  %mime                          ::  import the mark to delegate to
/$  tub  %atom  %mime                   ::  mark conversion to delegated mark
/$  but  %mime  %atom                   ::  mark conversion from delegated mark
::
::::  A minimal atom mark
  ::
=,  mimes:html
|_  ato=@
++  grab  |%
          ++  noun  @
          ++  mime  |=([* p=octs] q.p)
          --
++  grow  |%
          ++  mime  [/application/x-urb-unknown (as-octs ato)]
          --
++  grad
  |%
  ++  form  form:grad:deg               ::  simply delegated: diff definition
  ++  join  join:grad:deg               ::                    diff merge
  ++  mash  mash:grad:deg               ::                    diff force merge
  ::
  ++  diff                              ::  diff computation: convert operands
    |=  ota=atom                        ::  to delegated mark
    (diff:~(grad deg (tub ato)) (tub ota))
  ::
  ++  pact                              ::  diff application: convert operand
    |=  dif=mime                        ::  to delegated mark, convert result
    (but (pact:~(grad deg (tub ato)) dif))
  ::
  --
--
```

`/#` rune not having an effect if the file is built as a regular Hoon file is necessary for mark-conversion gate builds, which build mark files as regular files, avoiding a cycle in mark file builds with arm delegation.

The purpose of the [Mark specification changes](#mark-specification-changes) proposal is to simplify the build process of marks by removing mark build features that are not used anywhere to the authors' knowledge.

Together with the [Mark specification changes](#mark-specification-changes) proposal only `+grad` arm can be recognized by `/#` rune. The arm specifier argument of the rune is added for forward-compatibility purposes.

## Backwards Compatibility

This UIP would require a change to all mark files that use arm delegation. Even though neither Clay nor Ford are Kelvin-versioned, the change is disruptive enough to be included into a Kelvin-versioned update.

## Security Considerations

Needs discussion.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
