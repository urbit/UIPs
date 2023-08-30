# Urbit Improvement Proposals (UIPs)

The goals of the UIP project are to:

- Make core development decision-making visible and open to entry.
- Standardize and provide high-quality documentation for Urbit itself and conventions built upon it.

This repository tracks past and ongoing improvements to Urbit in the form of Urbit Improvement Proposals (UIPs). [UIP-0001](./UIPS/UIP-0001.md) governs how UIPs are published.

UIPs can be divided into the following categories:

- **Standards Track UIPs** are improvements to Urbit that require a Kelvin version decrement to any of the following components: the machine code specification, Nock; the programming language and its standard library, Hoon; the kernel, kernel modules, its standard library, or the base distribution &mdash; Arvo, vanes, zuse or %base, respectively.
- **Informational UIPs** describe an Urbit design issue, or provide general guidelines or information to the Urbit community, but do not propose new features. Informational UIPs do not necessarily represent an Urbit community consensus or recommendation, so users and implementors are free to ignore Informational UIPs or follow their advice.
- **Process UIPs** describe a process surrounding Urbit, or proposes a change to (or an event in) a process. Process UIPs are like Standards Track UIPs but apply to areas other than Urbit itself. They may propose an implementation, but not to Urbit's codebase; they often require community consensus; unlike Informational UIPs, they are more than recommendations, and users are typically not free to ignore them. Examples include procedures, guidelines, changes to the decision-making process, and changes to the tools or environment used in Urbit development. Any meta-UIP is also considered a Process UIP. 

## UIP

| Number                     | Title                     | Owner                          | Status    | Type      |
|:---------------------------|:--------------------------|:-------------------------------|:----------|:----------|
| [0001](./UIPS/UIP-0001.md) | Purpose and Guidelines    | ~wolref-podlex                 | Living    | Process   |
| [0100](./UIPS/UIP-0100.md) | Sticky Scry               | ~rovnys-ricfer                 | Draft     | Standards |
| [0101](./UIPS/UIP-0101.md) | %lick                     | ~mopfel-winrux                 | Final     | Standards |
| [0102](./UIPS/UIP-0102.md) | Symmetric Routing         | ~rovnys-ricfer                 | Draft     | Standards |
| [0103](./UIPS/UIP-0103.md) | Persistent Nock Caching   | ~rovnys-ricfer                 | Draft     | Standards |
| [0104](./UIPS/UIP-0104.md) | Scry Store                | ~rovnys-ricfer                 | Withdrawn | Standards |
| [0105](./UIPS/UIP-0105.md) | Drop Pokes to Dead Agents | ~rovnys-ricfer                 | Final     | Standards |
| [0106](./UIPS/UIP-0106.md) | Scry over HTTP            | ~watter-parter                 | Last Call | Standards |
| [0107](./UIPS/UIP-0107.md) | Auras Renovation          | ~ponmep-litsem                 | Draft     | Standards |
| [0108](./UIPS/UIP-0108.md) | %yard                     | ~lagrev-nocfep                 | Draft     | Standards |
| [0109](./UIPS/UIP-0109.md) | Essential Desks           | ~wicdev-wisryt                 | Draft     | Standards |
| [0110](./UIPS/UIP-0110.md) | Gall Agent Backups        | ~midden-fabler, ~mopfel-winrux | Draft     | Standards |
| [0111](./UIPS/UIP-0111.md) | Desk Publisher Switcher   | ~tinnus-napbus                 | Draft     | Standards |
| [0112](./UIPS/UIP-0112.md) | Informal Ping             | ~master-morzod, ~norsyr-torryn | Draft     | Standards |
| [0113](./UIPS/UIP-0113.md) | %ames: Directed Messaging | ~master-morzod                 | Draft     | Standards |
| [0114](./UIPS/UIP-0114.md) | OTA Approval              | ~tinnus-napbus                 | Draft     | Standards |
| [0115](./UIPS/UIP-0115.md) | Breadth-First Arvo        | ~wicdev-wicryt, ~rovnys-ricfer | Draft     | Standards |

## Background

Urbit has been in development for over a decade at time of writing, and until June of 2023 was developed without the use of this UIP process. New UIPs after UIP-0001 will begin their numbering at UIP-0100 to leave room for retroactive improvements that should have canonical documentation for reference.
