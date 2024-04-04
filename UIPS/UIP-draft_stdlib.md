---
title: Standard Library Aliases
description: "Provide alternate names for stdlib gates more intuitive to new developers."
author: ~lagrev-nocfep <neal@urbit.org>
type: "Standards Track"
category: "Hoon"
created: 2024-04-04
---

## Abstract

We propose adding a set of aliasing libraries to `/lib` in `%base` which may be optionally imported by developers into Dojo or their own code without needing any additional installation or setup steps.  These libraries provide an alternative naming scheme to the lapidary form preferred in kernel development.

## Motivation

Developers new to the Urbit ecosystem often find the usage of particular words for gate names in Hoon, Zuse, and elsewhere to be somewhat disorienting.  Although there are good reasons that `/sys` prefers the arm names and faces that it does, it is also straightforward to provide a set of aliases which more or less hew to the expectations of a modal developer new to Urbit.  For instance, we alias `++slag` with `++after`, and `++fand` with `++find-all`.

While this proposal follows the basic inspiration of the [`/lib/sequent`]([https:](https://github.com/jackfoxy/sequent)) library, it hews to only providing aliases for built-in functions in `/sys` (rather than extending functionality).  (`/lib/sequent` provides enhanced list functionality, including `zip3` and similar compound operators.)

(It is possible to provide a wrapper for Gall or Shrubbery agents down the road which automatically imports these libraries; such consideration should be deferred until their practical utility has been established.)

## Specification

The files which are proposed here shall be provided in `/lib` for immediate availability in the Dojo via a single import.

We provide:

- `/lib/map`
- `/lib/set`
- `/lib/list`
- `/lib/bits`
- `/lib/maplist`
- `/lib/mapset`

## Rationale

The objectives of this library scheme are to:

1. Provide alternative names for common library functions in `/sys` which are legible to developers new to Urbit.  (No `/lib` dependencies should be introduced beyond internal references.)
2. Partition `/sys`/`%base` renamings from more complex functionality, even where convenient.  I.e., these libraries should not introduce new functionality over and above what `/sys` already affords.
3. Pass inner functionality (as in `++by`) to outer cores (e.g. `+$jar`s).  This obviates needing to remember small differences in cores, and has precedence in `++in`/`++by` already (cf. some of the `map` and `set` jets).
4. Name nothing clever or twee!  (Be as clear as possible, but no clearer, to immediately break our own rule.)

A few arm names are preserved, e.g. `++spin` and `++rear`, as no clearer expression is liable to be found.

## Backwards Compatibility

The most likely compatibility issues stem from an alias unexpected to the developer.  For example, a supplied library name could mask an expected `/sys` name.  In our testing, we have not discovered an instance of this causing a problem yet.  The subject-oriented programming means that 

It is possible that we should mask, e.g. `++spin`, by its own name to preserve the pattern of `^spin` for an intentional skip.

## Reference Implementation

An implementation is provided at [`tamlut-modnys/userspace-util/)`]([https://github.com/sigilante/libtree](https://github.com/tamlut-modnys/userspace-util/)).  As an excerpt, here are two example snippets:

**`/lib/maplist`**

```hoon
|@
:: +add: [(jar) noun noun] -> (jar)
::
:: Adds a value to the head of the list at key in jar.
:: Examples
:: > =j `(jar @t @ud)`(make ~[['a' ~[1 2 3]] ['b' ~[4 5 6]]])
:: > j
:: {[p='b' q=~[4 5 6]] [p='a' q=~[1 2 3]]}
:: 
:: > `(jar @t @ud)`(add j 'b' 7)
:: {[p='b' q=~[7 4 5 6]] [p='a' q=~[1 2 3]]}
:: 
:: > `(jar @t @ud)`(add j 'c' 8)
:: {[p='b' q=~[4 5 6]] [p='a' q=~[1 2 3]] [p='c' q=~[8]]}
:: Source
++  add
  |*  [j=(jar) k=* v=*]
  (~(add ja j) k v)
--
::  * * *
```

**`/lib/list`**

```hoon
|@
++  after  slag
++  and-each  levy
++  any-each  lien
++  append  snoc
++  apply  turn
++  before  scag
++  concat  weld
++  except  skip
++  filter  skim
::  * * *
--
```

## Security Considerations

Needs discussion.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
