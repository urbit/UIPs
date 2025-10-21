---
title: Ford rune for +grad arm inheritance in marks
description: Delegation of mark revision control is done with a new Ford rune, making the dependency tree discoverable by parsing only Ford headers.
author: ~dozreg-toplud (@quodss), ~midden-fabler (midden-fabler)
status: Draft
type: Standards Track
category: Kernel
created: 2025-10-21
---

## Abstract

We propose moving revision control core delegation in marks from the product of the `+grad` arm of a mark core to the Ford header by declaring the delegation with a new Ford rune. Doing so would allow the build system to discover the dependency graph of a file by parsing static Ford declarations, without having to build the file or its dependencies.

## Motivation

Current work on Ford refactoring to use persistent memoization instead of the explicit Ford cache (Ford Lightning) hit a roadblock in mark build process. In the current Ford Lightning design when a file is built its Ford header is parsed first, finding its dependencies. If the file has no dependencies, it forms a leaf of a dependency graph, otherwise the process is applied recursively to all dependencies while checking for recursion. What is left is a dependency graph with Hoon content, which would be then built into a vase with a persistently memoized function.

When a mark is built as a mark core, and not as a file, this design breaks down due to `+grad` arm treatment by the build system. If the product of that arm is an atom, Ford treats it as a mark name, and tries to build that mark together with mark conversion gates between the marks. To know the dependency graph of a mark we would have to build it as a file, and doing so requires having to build all its dependencies, ruining the one directional dependency graph -> vase approach. If the `+grad` arm delegation was declared statically in the Ford header instead of dynamically as a product of `+grad` arm, this would not be an issue.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

The UIP proposes a new Ford rune (provisionally `/#`) to declare `+grad` arm delegation in a mark file.

`/#` rune MUST be followed by Hoon gap, followed by a term to indicate to which mark the revision control is delegated.

If the file with `/#` rune is built as a regular file, `/#` rune MUST NOT have any effect.

If the file with `/#` rune is built as a mark file, the core which is produced by the Hoon contents of the file MUST NOT have a `++grad` arm.

If a file is built as a mark file and it does not have a `/#` rune, the core which is produced by the Hoon contents of the file MUST have a `++grad` arm, and the product of that arm MUST NOT be an atom.

## Rationale

`/#` rune not having an effect is necessary for mark-conversion gate builds, which build mark files as regular files, avoiding a cycle in mark file builds with `+grad` delegation.

## Backwards Compatibility

This UIP would require a change to all mark files that use `+grad` delegation. Even though neither Clay nor Ford are Kelvin-versioned, the change is disruptive enough to be included into a Kelvin-versioned update.


## Security Considerations

Needs discussion.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
