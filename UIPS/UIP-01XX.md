---
uip: "01XX"
title: "%loop hint: reify infinite loops as crashes"
description: "A simple hint allows the interpreter to reify \"obvious\" non-terminating loops as Nock crashes."
author: ~ritpub-sipsyl (eamsden) 
status: Draft
type: Standards Track
category: Nock # Only required for Standards Track. Otherwise, remove this field.
created: 2024-03-25
---

## Abstract

A simple static hint can signal to the runtime to check for nontermination by checking if the current subject and hinted formula have a circular evaluation dependency. Since Nock crashes and non-termination are definitionally semantically equivalent, the interpreter can crash deterministically if this heuristic matches. 

Implementing this hint would remove the need to manually implement such checks within Nock, enabling usecases of the `%memo` hint previously prevented by the need to track such state in the subject.

## Motivation

Some Nock programs such as the Hoon compiler must run complex recursive functions over arbitrary user input. Algorithms which rely on memoization to be feasible can leverage the `%memo` hint supported by both Vere and Ares.

However, if it is necessary to deduce non-termination and crash with an error, the necessary state in the subject prevents the use of the %memo hint. While this is not the only reason for custom memoizing jets in the Hoon compiler, it is a major contributor. Non-termination and crashes are equivalent in Nock, and an interpreter choosing to report a crash is reporting that the computation would not terminate if continued.

Thus, if the interpreter makes this determination, it is correct for it to crash with a stack trace. However, we do not want to check every subject-formula pair against all prior pairs in the call stack at every evaluation site (Nock 2 or Nock 9).

## Specification

Nock interpreters SHOULD recognize the `%loop` static hint as follows (or extensionally equivalent behavior):

- Maintain a stack of subject and formula pairs
- On entry into the hinted computation
  - check whether the current subject and hinted formula are already on the stack
  - if so, crash
  - if not, push the current subject and formula onto the stack
- On exit from the hinted computation, pop the stack.


## Rationale

Any algorithm which maintains a stack-like structure in order to catch input on which it would naively nonterminate can simply be hinted by `%loop` instead. This removes state from the subject and allows such algorithms to also leverage the `%memo` hint.

## Backwards Compatibility

If `%loop` was used as described, algorithms which depend on it to detect obvious nontermination would instead actually spin forever in interpreters which do not support the hint. Existing Kelvin-version guarding should suffice to prevent this case.

## Reference Implementation

TBD

## Security Considerations

Semantically: this proposal presents no security concerns as the kelvin-guarding mechanisms already in place should prevent differential event replay from algorithms relying on its function. Further, crashes and infinite loops are definitionally equivalent in Nock.

Operationally: programs relying on this hint could be made to consume large amounts of memory in a manner not obvious by examining the type or contents of the subject, as the memory would have no direct corresponding noun in the subject.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
