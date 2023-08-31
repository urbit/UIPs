---
uip: 0117
title: Ulam
description: Self-Describing, Dynamically Typed Nouns
author: ~rovnys-ricfer
status: Draft
type: Standards Track
category: Kernel
created: 2023-08-30
---

## Abstract

This proposal defines a noun data structure and accompanying human-readable syntax for encoding self-describing nouns.  This is intended as Urbit's answer to JSON: a portable text-based format for encoding arbitrary data in such a way that a consumer of the data can query and operate on it without prior knowledge of its contents.

The format also has a natural bijection into and out of JSON to ease interopability with external systems that want to consume Urbit data.

## Motivation

There are a few intended use cases for this format:
- configuration files (`sys.kelvin`, `desk.bill`, docketfiles, etc.)
- userspace permission data
- agent state backups
- text encoding of a file with a foreign mark

Any mark definition could optionally specify conversions to and from ulam.  Conversion to ulam on the publisher could then be used to fetch data over the wire if the requesting ship doesn't know the type of the data it's fetching.

## Specification

Here is an example of ulam syntax, showing off some of its features:
```
{
  ~zod: 'foo',
  ~nec: [0x0, 'foo'],
  ~nus: [0x0, 'foo', {~2023.7.12: ~bus}],
  ~bud: /foo/3/~2000.1.1,
  ~wes: {
    'foo': 0x0,
    'bar': %mime|~09jq3e9gm4u0eurr6f05,
    ['foo', 'bar']: 3.000,
    {~zod: 3}: 4
  }
}
```

The outer data structure is an ordered map of ulams.  The ordering is a natural ordering over all ulams.  The keys of this outer map are all ships.  The values have various types.  `'foo'` is a cord (string as atom).  `[0x0, 'foo']` is an ulam "list" with two elements; the first element is a hexadecimal zero.  This list is null-terminated, but the null terminator is not present in the syntax.

All Hoon literal formats are valid ulam syntax.  Atom literals are parsed as atoms tagged with their auras, and other `$coin` literals (`%blob` and `%many`) are also supported, for encoding raw nouns and pairs of coins, respectively.  Hoon path literals are also supported (parsed as typed paths in the current version of this proposal).

Finally, there is also special ulam syntax for a `$page`, i.e. a noun tagged with a mark name (`[p=mark q=noun]`).  The `%mime|~09jq3e9gm4u0eurr6f05` example is a `%mime`-tagged page, whose noun literal expands to `[/foo/bar 3 'foo']`.

The data structure for representing a parsed ulam looks like this:
```hoon
|%
+$  ulam
  $~  [%coin *coin]
  ::  leaves
  ::
  $%  [%coin =coin]                            ::  atom, noun, or compound coin
      [%path =pith]                            ::  hoon path syntax
      [%page =mark noun=*]                     ::  %foo|bar, bar is coin blob
  ::
  ::  containers
  ::
      [%list p=(list ulam)]                ::  [item, item, item]
      [%map p=((mop ulam ulam) lte-ulam)]  ::  {key: value, key: value}
  ==
--
```

Any ulam can be used as a key in an ulam `%map`.

**TODO** *Should commas and colons be required?*
**TODO** *Should we add `::` comment syntax?*
**TODO** *Should we support multi-line strings?*
**TODO** *Should we support Hoon "cube"s, e.g. `%foo`?  If we do, should we also support cube-tagged values?  That would be a natural way to represent a value in a `$%`, which is very common for Hoon data structures.  That might make it easier to convert Hoon data structures into ulams.*

There is a natural bijection into JSON, which should be considered standard.  Here is the above example encoded as JSON:

```json
  {
    "~zod": "'foo'",
    "~nec": ["0x0", "'foo'"],
    "~nus": ["0x0", "'foo'", {"~2023.7.12": "~bus"}],
    "~bud": "/foo/3/~2000.1.1",
    "~wes": {
      "'foo'": "0x0",
      "'bar'": "%mime|~09jq3e9gm4u0eurr6f05",
      "['foo', 'bar']": "3.000",
      "{~zod: 3}": "4"
    }
  }
```

I think including an `%ulam` mark in the `%base` desk, with conversions to and from other standard marks, would be enough to start using the data structure in all the places we need it to be used.

## Rationale

Advantages over JSON:
- standard encoding of arbitrary keys, not just strings
- standard way (`%page`) to extend the syntax with new datatypes
- standard ways to embed binary data (`%page` and `%blob`)
- better atom types and formats -- in my opinion, Hoon's data formats are quite good
- standard ordering for maps

The biggest issue with this data structure, in my opinion, is that it's not that simple to convert a normal Hoon data structure to an ulam and back.  Programmers will likely want some kind of conversion generators, like the JSON reparsers.  This is not a great developer experience.  There is some tension between the normal statically typed way that Hoon code stores data and recursively self-tagging data structures like ulam where each layer down the tree has its own type tag.

A different approach to a self-describing data structure would be a pair of a more normally formatted Hoon noun and a portable type designator.  This would not be recursively self-describing; instead, it would be like a Hoon vase: a pair of a descriptor and value.  One could think of such a data structure as a portable vase (vases can't be effectively serialized over the network).

I prototyped such an approach here, called "cone": https://gist.github.com/belisarius222/066ea3487185b0a8b18b2522526b2585.

I went away from portable vases because the Hoon type system doesn't help you very much with containers.  It can't tell the difference between different treaps, so a map looks just like a set or queue.  This could potentially be alleviated by adding `%hint` type annotations to the type definitions for maps, sets, etc., so a system could use that to auto-generate a portable type descriptor from an annotated Hoon type.

The lack of recursive self-description would make `$cone`s more difficult to manipulate than `$ulam`s.  For instance, any operation on two `$cone`s would require modifying both the types and values, just like with vases.  They would likely be easier to convert from and to Hoon data structures, though.  Their representations might also be slightly more compact, although the deduplication in `+jam` encoding might reduce that to a marginal difference.

Ulam could have a runic representation, but more standard delimiters seemed more aesthetic and readable for this application.

## Backward Compatibility

This is new, but we could consider Kelvin versioning it, ideally to a lower value than the Hoon language, since it has a lot fewer moving parts and will ideally remain unchanged across most Hoon Kelvin changes.

## Test Cases

A unit test suite for parsing, serialization, and JSON conversion should be written.  This should be sufficient testing, since this is a self-contained module.

## Reference Implementation

https://gist.github.com/belisarius222/52680c74205a4401f879651354658e34

## Security Considerations

As long as there aren't parser jets, this should not pose serious security complications beyond normal Hoon atom parsing.

## Copyright Waiver

Copyright and related rights waived via [CC0](../LICENSE.md).
