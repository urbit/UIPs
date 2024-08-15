---
uip: "01XX"
title: Subsecond Timing Syntax
description: Add subsecond timing in both decimal and binary sizes.
author: ~lagrev-nocfep
status: Draft
type: Standards Track
category: Kernel
created: 2024-08-15
---

## Abstract

This proposal permits relative time intervals to be specified in subsecond timings, such as milliseconds or nanoseconds.  (It does not change the prettyprinter output.)

## Motivation

In Urbit's 64-bit time scheme, values less than a second are specified as negative powers of two.  For example, one millisecond is `0x41.8937.4bc6.a7ef`.  This is not particularly legible to an observer as a millisecond, nor is the expedient of `(div ~s1 1.000)` always convenient.  Applications such as the `%jinx` hint will be better served by directly providing a subsecond input syntax for time intervals `@dr`.

## Specification

We provide subsecond timing intervals in `/sys/hoon` in `++crub:so` according to the following scheme:

| Interval | Decimal Value | Hexadecimal Value |
| -------- | ------------- | ----------------- |
| `ms`     | 10¯³          | `0x41.8937.4bc6.a7ef` |
| `µs`     | 10¯⁶          | `0x10c6.f7a0.b5ed` |
| `ns`     | 10¯⁹          | `0x4.4b82.fa09` |
| `ps`     | 10¯¹²          | `0x119.7998` |
| `fs`     | 10¯¹⁵          | `0x480e` |
| `as`     | 10¯¹⁸          | `0x12` |

Smaller intervals of a second than 2¯⁶⁴ cannot be represented in native Urbit time relative time `@dr`.

Urbit time can therefore be more exactly expressed in terms of binary fractions of a second rather than decimal fractions of a second.  However, while IEC 80000-13 codified expressions for powers of two that are close to positive powers of ten, there are no expressions for powers of two close to negative powers of ten.  We would like to introduce such a scheme for convenience in Urbit time.

| Interval | Binary Value | Hexadecimal Value | Decimal Approximation | Error |
| -------- | ------------ | ----------------- | --------------------- | ----- |
| `mis`    | 2¯¹⁰      | `0x40.0000.0000.0000` | 10¯³ | 2.4% |
| `uis`    | 2¯²⁰      | `0x10.0000.0000.0000` | 10¯⁶ | 4.9% |
| `nis`    | 2¯³⁰      | `0x4.0000.0000` | 10¯⁹ | 7.4% |
| `pis`    | 2¯⁴⁰      | `0x100.0000` | 10¯¹² | 9.9% |
| `fis`    | 2¯⁵⁰      | `0x4000` | 10¯¹⁵ | 12.6% |
| `ais`    | 2¯⁶⁰      | `0x10` | 10¯¹⁸ | 12.5% (truncation) |

## Rationale

We trust that the ability to more precisely express subsecond intervals in Urbit time will prove useful.

More exposition can be found at [an associated USTJ article](https://github.com/Urbit-Systems-Technical-Journal/ustj-subsecond-timing/blob/master/mss.tex).

## Backward Compatibility

This does not break current time parsing.

## Test Cases

The values should overflow similar to how regular time overflows (e.g. `~s60` = `~m1`).

## Reference Implementation

- [#7057](https://github.com/urbit/urbit/pull/7057)

## Security Considerations

None.

## Copyright Waiver

Copyright and related rights waived via [CC0](../LICENSE.md).