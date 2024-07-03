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
| [0102](./UIPS/UIP-0102.md) | Symmetric Routing         | ~rovnys-ricfer                 | Withdrawn | Standards |
| [0103](./UIPS/UIP-0103.md) | Persistent Nock Caching   | ~rovnys-ricfer                 | Last Call | Standards |
| [0104](./UIPS/UIP-0104.md) | Scry Store                | ~rovnys-ricfer                 | Withdrawn | Standards |
| [0105](./UIPS/UIP-0105.md) | Drop Pokes to Dead Agents | ~rovnys-ricfer                 | Final     | Standards |
| [0106](./UIPS/UIP-0106.md) | Scry over HTTP            | ~watter-parter                 | Last Call | Standards |
| [0107](./UIPS/UIP-0107.md) | Auras Renovation          | ~ponmep-litsem                 | Draft     | Standards |
| [0108](./UIPS/UIP-0108.md) | %yard                     | ~lagrev-nocfep                 | Draft     | Standards |
| [0109](./UIPS/UIP-0109.md) | Essential Desks           | ~wicdev-wisryt                 | Last Call | Standards |
| [0110](./UIPS/UIP-0110.md) | Gall Agent Backups        | ~midden-fabler, ~mopfel-winrux | Final     | Standards |
| [0111](./UIPS/UIP-0111.md) | Desk Publisher Switcher   | ~tinnus-napbus                 | Final     | Standards |
| [0112](./UIPS/UIP-0112.md) | Informal Ping             | ~master-morzod, ~norsyr-torryn | Final     | Standards |
| [0113](./UIPS/UIP-0113.md) | %ames: Directed Messaging | ~master-morzod                 | Last Call | Standards |
| [0114](./UIPS/UIP-0114.md) | OTA Approval              | ~tinnus-napbus                 | Final     | Standards |
| [0115](./UIPS/UIP-0115.md) | Breadth-First Arvo        | ~wicdev-wicryt, ~rovnys-ricfer | Last Call | Standards |
| [0116](./UIPS/UIP-0116.md) | Arvo Ticks                | ~wicdev-wicryt, ~rovnys-ricfer | Last Call | Standards |
| [0117](./UIPS/UIP-0117.md) | Ulam: Self-Describing Nouns | ~rovnys-ricfer               | Draft     | Standards |
| [0118](./UIPS/UIP-0118.md) | Encrypted Remote Scry     | ~hastuc-dibtux                 | Final     | Standards |
| [0119](./UIPS/UIP-0119.md) | Pretty Printer Improvements | ~sidnym-ladrut               | Draft     | Standards |
| [0120](./UIPS/UIP-0120.md) | HTTP Streaming            | ~rovnys-ricfer                 | Draft     | Standards |
| [0121](./UIPS/UIP-0121.md) | %pine Request at Latest   | ~rovnys-ricfer                 | Draft     | Standards |
| [0122](./UIPS/UIP-0122.md) | %wild: Stateless jet registration | ~ritpub-sipsyl         | Draft     | Standards |
| [0123](./UIPS/UIP-0123.md) | %loop hint: reify infinite loops as crashes | ~ritpub-sipsyl | Draft   | Standards |
| [0124](./UIPS/UIP-0124.md) | Computation Timeout Hint  | ~lagrev-nocfep                 | Draft     | Standards |
| [0125](./UIPS/UIP-0125.md) | %eyre/%iris Webssocket Support | ~fidwed-sipwyn            | Draft     | Standards |
| [0126](./UIPS/UIP-0126.md) | Pierport Protocol         | ~littul-pocdev                 | Draft     | Standards |


## Background

Urbit has been in development for over a decade at time of writing, and until June of 2023 was developed without the use of this UIP process. New UIPs after UIP-0001 will begin their numbering at UIP-0100 to leave room for retroactive improvements that should have canonical documentation for reference.
