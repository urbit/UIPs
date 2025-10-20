---
uip: "XXXX"
title: Comet Attestation
description: Allow for rotatable comet networking keys
author: ~hanfel-dovned, ~tinnus-napbus, ~bonbud-macryg
status: Draft
type: Standards Track
category: Kernel
created: 2025-10-20
---

## Abstract

This UIP proposes three coordinated changes for comet identity and networking:

1) Allow Ames to query Jael for comet keys instead of requiring self-attestation from comets.
2) Introduce a cryptosuite that supports tweaked keys.
3) Allow comets to include routing information in `$point:jael` and allow Ames routing logic to avoid assuming Azimuth sponsorship hierarchies.

Collectively, these changes make comet identity cryptographically coupled to the PKI substrate it attests to, while remaining substrate-agnostic at the protocol level.

## Motivation

Groundwire’s thesis is pragmatic: over the long term, Bitcoin is the only chain that provides an unambiguous, protocol-level canonical history (the heaviest valid proof-of-work chain) that a Kelvin 0 Urbit could resolve without social coordination or trusted checkpoints. This suggests a clean long-term anchor for the Urbit PKI.

It is too early to propose that that belief be enforced by kernel policy. Instead, Urbit should provide a general mechanism whereby comets can attest keys to any PKI source (chosen by a Jael `%listen` task) and peers can verify these attestations.

The vast majority of Urbit's 128-bit address space is allotted to ephemeral identities. The 32 bits under Azimuth's domain constituting a DNS-centralized hierarchical network of memorable names will be untouched. We seek to upgrade the rest of the Urbit universe to allow for self-sovereign networking.

## Specification

There are three main changes we propose to Arvo:

### 1. Allow Ames to query Jael for comet keys

- Replace `++request-attestation` with `++fetch-comet-pki`:
    - Ames **MUST** query Jael for the comet’s latest `life` and corresponding networking keys record.
    - If Jael has a record, Ames **MUST** use it to validate the sender’s keys. If not present, Ames **MAY** fall back to the self-attestation request.
- The `$point:jael` map from `life` to `[crypto-suite=@ud =pass]` remains the authoritative store.

### 2. Suite C Tweaked keys

- Replace the crypto Suite B core `+crub`, whose payload is: 
  ```
  pub=[cry=@ sgn=@]
  sek=(unit [cry=@ sgn=@])
  ```

  with a "Suite C" core `+cric` whose payload is:
  ```
  $%  $:  suite=%b
          pub=[cry=@ sgn=@ ~]
          sek=$@(~ [sed=@ cry=@ sgn=@])
      ==
      $:  suite=%c
          pub=[cry=@ sgn=@ tw=[ugn=@ dat=@ xtr=@]]
          sek=$@(~ [sed=@ cry=@ sgn=@])
      ==
  ==
  ```
  The first character of `pass`, being `b` or `c`, encodes which `+cric` case to use. In the `%c` case:
    - `sgn.pub` and `sgn.sek` are tweaked keys.
    - `ugn` is the untweaked signing public key.
    - `dat` is the tweak data, typically attesting to PKI choice (e.g., concatenation of `%btc` and a Bitcoin ordinal number).
    - `xtr` is an additional data field not included in the tweak.
    - `sed` is the random seed used to generate the untweaked keypair (so that it can be re-generated)

- Tweaked keys provide a pointer to PKI truth by baking that pointer directly into the networking key itself, ensuring that the two can never drift apart. 

### 3. Fief-based comet routing

- **Jael extension:** Comet PKI entries **MAY** include a `$fief` encoding *either* a static IP/port (`%if` for IPV4 or `%is` for IPV6) or a domain (`%turf`):

    ```
    +$  fief  
        $%  [%turf p=(list turf) q=@udE]
            [%if   p=@ifF        q=@udE]
            [%is   p=@isH        q=@udE]
        ==
    ```

- **Ames routing changes:**
    - If a comet has a `$fief`, Ames **SHOULD** attempt routing according to `$fief` before falling back to sponsor-chain routing.
    - Sponsor-chain termination at galaxies **MUST NOT** be assumed.
      - Therefore, Ames **MUST** detect and avoid routing cycles.

## Backwards Compatibility

- All ships will continue to subscribe to `%azimuth` for galaxy, star, and planet networking keys.
- A peer that is unaware of a comet's networking keys in Jael (due to either party not adopting a new PKI) will continue to request a self-attestation, and the comet will continue to fulfill it.
- Suite B functionality is preserved within Suite C.
- Existing Azimuth-based routing remains; `$fief` is additive.

## Security Considerations

- As is already the case, Jael will accept PKI records from configured watchers via the `%listen` task. Operators **MUST** vet watchers they run; the kernel does not enforce a single source of truth. Misconfigured or malicious watchers could inject bogus records.
- The introduction of Suite C tweaked keys increases the cryptographic surface area. Implementations **MUST** take care to correctly derive, serialize, and verify tweak data and untweaked keys, as misuse or mis-binding could compromise key provenance or create non-interoperable signatures.

## Status

A prototype exists in the Groundwire organization's forks of `urbit/urbit` and `urbit/vere`, including Ames `+fetch-comet-pki`, Suite C `+cric`, and `$fief` handling.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).