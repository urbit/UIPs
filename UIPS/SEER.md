---
title: Seer
description: A monadic scry interface.
author: ~fodwyt-ragful, ~mastyr-bottec
status: Draft
type: Standards Track
category: Kernel, Hoon
created: 2023-10-04
---

## Abstract

<!--
  The Abstract is a multi-sentence (short paragraph) technical summary. This should be a very terse and human-readable version of the specification section. Someone should be able to read only the abstract to get the gist of what this specification does.

  TODO: Remove this comment before submitting
-->

Seer is a monadic scry interface which allows application developers to
wrap read from the namespace without using dotket (`.^`) or its virtual
Nock operator 12.

## Motivation

<!--
  This section is optional.

  The motivation section should include a description of any nontrivial problems the UIP solves. It should not describe how the UIP solves those problems, unless it is not immediately obvious. It should not describe why the UIP should be made into a standard, unless it is not immediately obvious.

  With a few exceptions, external links are not allowed. If you feel that a particular resource would demonstrate a compelling case for your UIP, then save it as a printer-friendly PDF, put it in the assets folder, and link to that copy.

  TODO: Remove this comment before submitting
-->

dotket and Nock 12, which dotket compiles to, makes Nock _not_ referentially
transparent. By consequence, memoization is less effective.

In the interest of removing dotket and Nock 12 in the future, a replacement must
be supplied first. The replacement should attempt to be as ergonomic as dotket
to avoid widespread boilerplate churn.

## Specification

<!--
  The technical specification should describe the syntax, interface and semantics of any new feature. The specification should be detailed enough to allow for implementation in the OS and possible competing, interoperable implementations for any of the current runtimes (vere, ares).

  It is recommended to follow RFC 2119 and RFC 8170. Do not remove the key word definitions if RFC 2119 and RFC 8170 are followed.

  TODO: Remove this comment before submitting
-->

Here's a Gall agent snippet that scries with `.^`:
```hoon
```

Here's the same Gall agent snippet that scries with `seer`:
```hoon
```

A `seer` is the type of programs that scry...
```hoon
++  seer
  |*  a=mold
  %+  each  a
  %+  pair  path
  $-  (unit (unit *))
  (seer a)
```

and `++seer-bind` is used to combine `seer` values:
```
++  seer-bind
  |*  [a=(seer *) f=gate]
  ?-  -.a
    %&  (f p.a)
    %|  :-  %|
        :-  p.p.a             ::  path
        |=  r=(unit (unit *))
        (seer-bind (q.p.a r) f)
  ==
```

We also implement the remaining arms to complete the monad:
- `++seer-ap`
- `++seer-map`
- `++seer-join`

## Rationale

<!--
  The rationale fleshes out the specification by describing what motivated the design and why particular design decisions were made. It should describe alternate designs that were considered and related work.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

## Backwards Compatibility

<!--

  This section is optional.

  All UIPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their consequences. The UIP must explain how the author proposes to deal with these incompatibilities, and how developers can migrate the applications. This section may be omitted if the proposal does not introduce any backwards incompatibilities, but this section must be included if backward incompatibilities exist.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

No backward compatibility issues found.

## Reference Implementation

<!--
  This section is optional.

  The Reference Implementation section should include a minimal implementation that assists in understanding or implementing this specification. It should not include project build files. The reference implementation is not a replacement for the Specification section, and the proposal should still be understandable without it.

  If the reference implementation is too large to reasonably be included inline, then external links to the urbit/urbit and/or urbit/vere repositories are allowed.

  TODO: Remove this comment before submitting
-->

## Security Considerations

<!--

  UIPs SHOULD contain a section that discusses the security implications/considerations relevant to the proposed change. Include information that might be important for security discussions, surfaces risks and can be used throughout the life-cycle of the proposal. E.g. include security-relevant design decisions, concerns, important discussions, implementation-specific guidance and pitfalls, an outline of threats and risks and how they are being addressed.

  The current placeholder is acceptable for a draft.

  TODO: Remove this comment before submitting
-->

Needs discussion.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
