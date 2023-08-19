---
uip: 0111
title: Ulam
description: Self-Describing, Dynamically Typed Nouns
author: ~rovnys-ricfer
status: Draft
type: Standards Track
category: Kernel
created: 2023-07-31
---

## Abstract

This proposal defines a noun datastructure and accompanying human-readable syntax for encoding self-describing nouns.  This is intended as Urbit's answer to JSON: a portable text-based format for encoding arbitrary data in such a way that a consumer of the data can query and operate on it without prior knowledge of its contents.

The format also has a natural bijection into and out of JSON to ease interopability with external systems that want to consume Urbit data.

## Motivation

There are a few intended use cases for this format:
- configuration files (`sys.kelvin`, `desk.bill`, docketfiles, etc.)
- userspace permission data
- agent state backups
- text encoding of a file with a foreign mark

Any mark definition could optionally specify conversions to and from ulam.  Conversion to ulam on the publisher could then be used to fetch data over the wire if the requesting ship doesn't know the type of the data it's fetching.

## Specification



## Rationale


## Backward Compatibility


## Test Cases

TODO

## Reference Implementation

TODO

## Security Considerations


## Copyright Waiver

Copyright and related rights waived via [CC0](../LICENSE.md).
