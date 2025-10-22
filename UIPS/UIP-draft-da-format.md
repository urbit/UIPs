---
title: Adjust @da format
description: Use leading zeroes in @da more consistently.
author: ~palfun-foslup
status: Review
type: Standards Track
category: Hoon
created: 2025-10-22
---


## Abstract

At present, the `@da` literal syntax never allows leading zeroes in its month
and day segments. In its hour, minute and second segments however, it allows
leading zeroes during parsing, and actively includes them during rendering to
ensure consistent double-digit representations.

Here, we propose extending the leading zero behavior of the time segments into
the format's month and day segments for both parsing and rendering, accepting
and displaying `~2025.1.1` as `~2025.01.01`.


## Motivation

Datetime timestamps (`@da` and its representations) show up in many places.
Whether used as simple timestamps or as part of resource identifiers, it's not
uncommon during debugging to encounter lists of `@da`s or structures that
contain them, or to be presented with string representations of timestamps in
the `@da` format.

Consider now the following two pretty-printed `(list @da)`, both containing the
same five timestamps:

```hoon
~[
  ~2023.3.8..20.41.01..e65b
  ~2023.11.31..20.55.58..c71d
  ~2024.8.13..21.05.30..4d73
  ~2025.4.7..20.05.53..5048
  ~2025.10.22..20.38.15..5181
]
```

```hoon
~[
  ~2023.03.08..20.41.01..e65b
  ~2023.11.31..20.55.58..c71d
  ~2024.08.13..21.05.30..4d73
  ~2025.04.07..20.05.53..5048
  ~2025.10.22..20.38.15..5181
]
```

Which of these two representations of the data sparks more joy?


## Specification

Rendering of `@da` values is to pad the month and day segments with leading
zeroes to make them at least two digits wide.

Parsing of `@da` literals is to accept optional leading zeroes in the month and
day segments, as the parsing of its time segments presently does.

(This UIP considers resolving any pre-existing `@da` parsing oddities (see
[Appendix A: `@da` parsing oddities](#appendix-a-da-parsing-oddities)) to be
firmly out of scope.)


## Rationale

The change as described is the smallest change that achieves the following two
goals:

1. Standardize on zero-padded date representations.
2. Maintain compatibility with existing `@da` literals.


## Backwards Compatibility

`@da` values will not change. Existing string representations of `@da`s will
continue to parse into their original `@da` values.

However, when operating on the string representation exclusively, old and new
strings may differ while referring to the same underlying `@da` value. Agents
and clients that store the string representations (as opposed to handling them
transiently or working with their underlying integer values) should take
particular care to either re-format their stored strings, or make sure the
strings they ingest continue using the old format.

To help with the latter, we could consider introducing a `@dad` (**DA**te
**D**eprecated) or similar which would retain the current `@da` rendering
behavior without introducing any additional literal parsing syntax. Whether we
would accept an aura for which there is rendering but no distinct parsing
syntax is up for debate.


## Reference Implementation

For an implementation example that achieves a superset of the specified
behavior, see
[the hoon.hoon changes in urbit/arvo#1065](https://github.com/urbit/arvo/pull/1065/files#diff-16921564ac83642f49026067dc653b1539a5fdf6517ce94baef8aff869e81a36).
The diff for this UIP could likely be smaller.

(The description of the change given there and the subsequent discussion are
available as historical context but are not intended to be part of this UIP.)


## Security Considerations

None known beyond anything caused by the pitfalls described in
[Backwards Compatibility](#backwards-compatibility).


## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).


## Appendix A: `@da` parsing oddities

The following is a list of `@da` parsing behavior that may be considered
unexpected or undesirable. It is provided here to inform the reader about
existing `@da` parsing behavior, which may come up in discussion of this UIP.
It is not, however, intended to be included in this UIP's scope in any way.

- Allowing unlimited leading zeroes in time segments.
  ```hoon
  > ~1111.1.1..001.0001.0000000001
  ~1111.1.1..01.01.01
  ```
- Allowing day segments greater than their month logically allows.
  ```hoon
  > ~1111.2.31
  ~1111.3.3
  ```
- Allowing day segments of unlimited size.
  ```hoon
  > ~1111.1.62
  ~1111.3.3
  ```
- Allowing time segments of unlimited size.
  ```hoon
  > ~1111.1.1..987.654321.111
  ~1112.5.10..12.22.51
  ```
- Parses (really, `+yule`) crashes when presented with "oversized" sub-second
  precision.
  ```hoon
  > (rash '~1111.1.1..1.1.1..1122.3344.5566.7788' nuck:so)
  [%$ p=[p=~.da q=170.141.183.975.107.459.934.277.014.437.665.077.128]]
  > (rash '~1111.1.1..1.1.1..1122.3344.5566.7788.9900' nuck:so)
  decrement-underflow
  dojo: hoon expression failed
  ```
