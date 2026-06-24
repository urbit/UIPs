---
uip: "137"
title: Self-Attestation Packets for Suite C Comets
description: Suite C comet self-attestation for expanded PKI pointers
author: ~hanfel-dovned
status: Draft
type: Standards Track
category: Kernel
created: 2026-06-23
---

## Abstract

UIP-136 introduced Suite C comets as Urbit IDs whose networking keys are tweaked with a pointer to an arbitrary PKI. Upon contact with a Suite C comet, a ship will check Jael for that comet's presence and ack or nack accordingly.

However, this design requires ships to run background watcher agents for each PKI, which depending on PKI implementation may be inefficient or even impossible. To solve this, we introduce a self-attestation packet for Suite C comets, akin to the self-attestation packet for Suite B comets, containing additional PKI pointer data that makes it possible to more generally validate identity on first contact.

## Motivation 

In Groundwire's initial Bitcoin protocol, we expected all ships to index and parse Bitcoin blocks to find Groundwire PKI events encoded as on-chain data. However, we've discovered that most Bitcoin wallets don't allow this on-chain data to be broadcast in the first place, requiring us to either build a bespoke wallet app for comet custody (with inevitably worse security than the state of the art and incompatible UX for existing Bitcoiners) or change how we encode data on-chain.

In a nutshell, we can enforce that all Bitcoin comets do in fact attest to their identity on Bitcoin by secretly encoding PKI operations within Taproot commit transactions and then sharing Merkle proofs of those attestations' presence off-chain without ever revealing the operation on-chain. We can store this sequence of Merkle proofs on the user's ship and send them to new peers on first contact in a self-attestation packet.

## Specification 

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

Upon receiving a Suite C self-attestation packet, Ames MUST pass it to some validation agent to either accept it or reject it as a valid Urbit ID. The sending comet's tweaked networking key MUST contain a `domain=@tas` corresponding to this agent.

A mapping from domain to agent is set by passing an `%anex` task to Jael, which we introduce in this UIP (alongside tasks for deleting and modifying mappings). Upon installing a new PKI handler desk, its main agent SHOULD set this in `++on-init`.

When Ames receives a Suite C self-attestation packet from an alien, it MUST pass it to Jael via the newly-introduced `%writ` task. Jael MUST then either dispatch it to the appropriate agent or nack if the domain is non-existent. The agent SHOULD then perform the validation and give the new point's data to Jael as a `%fact`. When the sending ship tries to contact the receiving ship again, the receiving ship's Ames will see it in Jael and thus will establish normal contact.

## Backwards Compatibility

This update will network breach the existing few dozen Groundwire comets. This is consistent with the premise of the Groundwire alpha launch which specified that identities would not yet be permanent.

## Status

An [in-progress implementation branch](https://github.com/gwbtc/urbit/compare/gw/next/kelvin/408...cyc/cc) exists in the Groundwire Foundation's urbit repo.

## Copyright

Copyright and related rights waived via CC0.

