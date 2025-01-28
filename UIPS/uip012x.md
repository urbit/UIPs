---
uip: "012X"
title: Remove `/===`
description: Alter Hoon syntax such that /=== is a Dojo affordance only.
author: ~lagrev-nocfep
status: Draft
type: Standards Track
category: Kernel, Hoon
created: 2024-07-15
---

## Abstract

The syntax `/===` prefills in Hoon to the current `beak` (in Dojo) or its bunt
(in `/app`).  This syntax should be made Dojo-only and removed from the Hoon
core language.

## Motivation

The syntax `/===` prefills in the Dojo to the current `beak`.  This is fast
and convenient, and at least in the Dojo, like `%` should be preserved.

However, `/===` is error-prone in actual files:  because many develop on ~zod
and the bunt of `/===` is to ~zod, the error may not be caught during
development and must be hotfixed on deployment.

By making `/===` a Dojo-only syntax, we can still gain the programming
benefits of the short syntax without incidentally introducing an entire
vector of bugs.

## Specification

The main thing to be done is to move `++gasp:vast` and/or `++poon:vast`
from `/sys/hoon` to the Dojo parser.  `++gasp` itself is simple:

```
  ++  gasp  ;~  pose                                    ::  parse =path= etc.
              %+  cook
                |=([a=tyke b=tyke c=tyke] :(weld a b c))
              ;~  plug
                (cook |=(a=(list) (turn a |=(b=* ~))) (star tis))
                (cook |=(a=hoon [[~ a] ~]) hasp)
                (cook |=(a=(list) (turn a |=(b=* ~))) (star tis))
              ==
              (cook |=(a=(list) (turn a |=(b=* ~))) (plus tis))
            ==
```

as is `++poon`:

```
  ::
  ::  tyke is =foo== as ~[~ `foo ~ ~]
  ::  interpolate '=' path components
  ++  poon                                              ::  try to replace '='s
    |=  [pag=(list hoon) goo=tyke]                      ::    default to pag
    ^-  (unit (list hoon))                              ::    for null goo's
    ?~  goo  `~                                         ::  keep empty goo
    %+  both                                            ::  otherwise head comes
      ?^(i.goo i.goo ?~(pag ~ `u=i.pag))                ::    from goo or pag
    $(goo t.goo, pag ?~(pag ~ t.pag))                   ::  recurse on tails
```

The call chain to `++poon:vast` is 

Unfortunately, since this occurs inside of Hoon expressions currently, we
have to alter permissible expressions slightly.  The `/===` syntax will not
be permitted inside of Hoon code at all, but treated as a top-level path in
Dojo similar to how `%` is used now.  That is supplied by `++parse-rood`
in `/app/dojo`, which calls out to the `/sys/hoon` `++rood:vast` parsers
(`++rood`, `++posh`, `++porc` all ultimately reference `++poon`).

```
  ++  parse-rood
    ::  XX should this use +hoon-parser instead to normalize the case?
    ::
    =>  (vang | (en-beam dir))
    ;~  pose
      rood
    ::
      ::  XX refactor ++scat
      ::
      =-  ;~(pfix cen (stag %clsg -))
      %+  sear  |=([a=@ud b=tyke] (posh ~ ~ a b))
      ;~  pose
        porc
        (cook |=(a=(list) [(lent a) ~]) (star cen))
      ==
    ==
```

Another possibility is that `++ford` could "sanitize" the `/app` file somehow
to prevent `/===` syntax, but it's less clear what the right approach there
would be.

## Backwards Compatibility

The change in `/===` may affect existing code, but since  `/===` in `/app` code
is an antipattern it seems more likely to shake out bugs than introduce them.

## Security Considerations

This change is more likely to increase security than diminish it.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
