---
uip:
title: Auras Renovation
description: Address inconsistencies in the aura type system
author: ~ponmep-litsem
status: Draft
type: Standards Track
category: Hoon
created: 2023-06-30
---

## Abstract

The aura type system provides a way to indicate a particular interperation of an atom in Hoon. An aura of an atom influences both how it is presented to the user, as well as specifies the parsing routine. Currently, auras are used in an inconsistent manner, thus preventing enforcement of atom sanity at the compiler level. In this proposal, we address these inconsistencies, together with related problems. With the implementation of this proposal, auras become a correct type system tool, dispensing with the need for manual atom sanity checks.

## Motivation

There are several cases in which auras are used in an inconsistent manner.

1. The `aura` type itself, defined as `@ta`, violates the constraint on knots, which do not permit uppercase letters.

2. Use of a `term`, which is too restrictive, for an aura variable, which admits uppercase bit-width annotations.
	An example of this inconsistency is in the Hoon atom definition,
	
	```
	+$  hoon
	...
	[%rock p=term q=*]
	[%sand p=term q=*]
	```
as well as in many functions throughout `sys/hoon.hoon` which accept `term` for an `aura`. This is seen in 		practice 	with tapes, defined as `(list @tD)`. Another example is `+fitz` which accepts a `term` for a general aura.

3. Further, there are two Hoon literals which have uppercase letters in their digit set: `@uc` and `@uw`. The `scot` atom printer returns a `@ta`, and the `slaw` family of atom parsers similarly violates this same constraint, accepting a `@ta` as their string argument.

4. The `+slaw` family of atom parsers accepts a `@tas` for an aura, thus restricting valid auras with bit-width extensions.

5. Clay persists invalid paths according to the path specification as a list of `@ta`. These can either be some valid `@uc`, `@uw` atoms, or, more generally, any path addressing a file whose name contains uppercase letters.

These violations of the aura type system make it impossible to enforce sanity of atoms at the compiler level, as originally envisioned with the `+sane` and `+ruth` arms. Fixing the inconsistencies will empower the compiler to check atom sanity at the compile time, and free the programmer from the need to perform manual atom sanity checks with `+sane`.

Apart from the inconsistent use of the auras, there are also several related problems.

- (A) `+sane`, so far not used systematically, is underspecified and performs only some atom sanity checks
- (B) The double sig cord syntax appears outdated and its character restriction is inconsistent
- (C) The pretty-printer is unable to correctly print arbitrary `@t` constants
- (D) Default printing of user-defined unnested auras could become a source of unforseen to detect bugs

## Specification
- The knot aura `@ta` denotes a string composed of allowed characters: `[0-9], [a-z], [A-Z], cab, hep, sig, dot`. It is considered URL-safe, and allows the encoding of all Hoon constants
- A valid aura is composed of a prefix pat, a following string of zero or more lowercase letters, with an optional trailing uppercase bit-width annotation
- (To be determined) The `@t` aura denotes a UTF-8 string composed of such and such characters
- (To be determined) The `@c` aura denotes a UTF-32 string composed of such and such characters
- An atom violating its aura constraint, as defined by `+sane`, is considered _mad_
- Production of mad atoms is guarded against by the Hoon compiler. An attempt to produce a mad atom results in a `cast-fail` error
- The double sig cord syntax is considered obsolete
- Auras are printed by the `+scot` arm.  `+scot` crashes on unnested user-defined auras.

## Rationale

### `@ta` to allow uppercase letters

The exclusion of uppercase letters from the knot aura `@ta` specification seems unnecessary. Relaxation of this condition alone solves problems 1--5. There seems to be little justification for it at present, beyond enforcing strict URL safety. It is clear that such strictness can no longer be maintained.

Hoon literals can not be considered URL-safe unless uppercase letters are permitted. Overwhelming majority of websites is served by UNIX systems, which are naturally case sensitive. Clay too has to allow uppercase filenames. Both Eyre and Iris kernel vanes use `@t` for URLs.

We therefore propose to relax this restriction and allow uppercase letters in `@ta`, still declaring it to be URL-safe.

### Only a single, trailing uppercase bit-width annotation to be allowed

Currently, only the last uppercase letter is treated as a bit-width extension by the aura compatibility check arm `+fitz`. This means that an aura like `@uDE` denotes an aura of type `@uD` with the bit-width extension of `E` -- 16 bits. We propose to simplify things by further restricting the aura syntax: only a single, trailing uppercase letter is allowed in the aura specification. Thus  an aura `@uDE` is invalid, while `@uD` and `@uE` are allowed.

### Atom sanity to be enforced at compile-time

After restrictions placed upon each aura are examined and encoded in `+sane`, `+ruth` arm can be enabled. Henceforth, violations of aura constraints are going to be caught at compile-time. The are three families of restricted auras: `@c, @i, @t`. In addition, any aura can be further size-restricted by a trailing bit-width annotation. The size restriction on atoms is enforced.

#### Sanity of @c atoms

`@c` needs to be a valid UTF-32 string. _Precise constraints to be determined._

#### Sanity of @i atoms

The internet address atoms `@i` possess an implicit size restrictions. `@if` should not exceed 32 bits, and `@is` should not exceed 128 bits.
Attempts to cast an oversize atoms will result in a `cast-fail` error.

```
> `@if`0xbe.1234.cafe
cast-fail

> `@is`(bex 128)
cast-fail

> `@uxD`0x1ff
cast-fail
```

#### Sanity of @t atoms

`@t` family of auras specify progressively restricted string types. _Precise constraints to be determined._

```
> `@tas`'Madness !'
cast-fail
```

`+scot` would catch mad atoms via `+sane`, what now works
```
(scot %tas '11 Hello!')
~.11 Hello!
```
would fail as
```
(scot %tas '11 Hello!')
cast-fail
```
on the grounds that the semantics of `+scot`**w** is to interpret an arbitrary atom with the given aura and print it.

### Outdated cord `~~` syntax

A Hoon literal beginning with `~~` parses with `;~(pfix sig (stag %t urx:ab))`, that is, it parses to a cord `@t`. However, `urx:ab` refuses to parse uppercase letters, while allowing the uppercase letters to be parsed using the escape mechanism

```
> (slaw %t '~~Haha')
~
> !>('Haha')
[#t/@t q=1.634.230.600]
> !>(~~~48.aha)
[#t/@t q=1.634.230.600]
```
.
Given that the `~~` strings parse to a cord, uppercase letters should be allowed. There is also no actual use across Arvo codebase and standard desks. The only place where this notation appears is in comments, in `sys/hoon.hoon`,
```
:-  [%ktts %i [%sand 'tD' *@]]                  ::  :-  i=~~
[[%sand 'tD' i.p.gen] res]                      ::  [~~{i.p.gen} {res}]
```
and in `lib/pprint.hoon`
```
::  XX Output for cords is ugly. Why ~~a instead of 'a'?
```

. We propose this cord notation be considered obsolete and removed.

### Change in default printing of atoms

The default printing of user-defined auras as hexadecimal without separator invites abuses.
There is already an instance of code in `%base` using presently undefined aura `@x` to print a hash value as a hex string without separators, instead of using appropriate arm, in this case the `+x-co:co` hex encoder. Such code works, until one day the aura `@x` suddenly becomes defined. `+scot` should therefore crash on those user-defined auras which do not nest with any of the auras presently defined in `sys/hoon.hoon`.

## Backward Compatibility

- The stricter aura syntax, with only single trailing uppercase bit-width annotation allowed, is currently obeyed across all standard desks
- The redefined `@ta` aura will cover a wider range of strings and is thus naturally backward-compatible
- With the enabling of `+ruth`, code relying on aura violations will crash. All aura violations across kernel components need to be tracked down and fixed
- There is no instance of the double sig cord notation in standard desks
- With the removal of default printing of unnested user-defined auras, the code relying on it will crash

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).

