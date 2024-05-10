---
title: Computation Timeout Hint
description: Introduce a `%jinx` hint to permit timeout of a computation which may not terminate.
author: ~lagrev-nocfep
status: Draft
type: Standards Track
category: Kernel
created: 2024-05-09
---

## Abstract

We propose adding a `%jinx` hint to terminate computations automatically from the runtime.

```
> ~>  %jinx.[~s5]  (add 1 3)
4

> ~>  %jinx.[~s5]  (infinite-loop)
bail: timed out at s/5.001.000
```

## Motivation

As a personal server, an Urbit instance may be called upon to evaluate arbitrary code.  Per the halting problem, aside from trivial infinite loops we cannot conclude how long an arbitrary expression will take to evaluate—or if it will never complete.  In certain environements, it is impossible or inconvenient to interrupt the runtime process.  (In particular, interfaces using `%eyre`/HTTP or `%lick` may not be able to send a `SIGINT` to break execution.)

While the subject-oriented programming model provides some security, and userspace permissions will provide more, arbitrary code may result in intentional or inadvertent evaluation of long-running code or non-terminating code.

## Specification

The `%jinx` hint is a dynamic hint accepting a timeout value and an expression.  If the expression does not complete within the span of the timeout value, then the runtime should interrupt the process with a `bail` and slog the elapsed time to the console.

No changes need to be made to `/sys/hoon` or Arvo.  Vere needs to be modified in `nock.c` to handle the hint.  The currently unused timeout mechanism in `u3m_soft` will be reactivated with the head of the hint for the timeout and the tail of the hint for the product.

An implementation has been begun in `sigilante/jinx`.

## Backwards Compatibility

This is a new runtime hint.  No backward compatibility issues found.

## Security Considerations

This should improve Urbit security for any instance in which arbitrary eval is allowed.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
