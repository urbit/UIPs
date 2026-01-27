---
title: Userspace Permissions
description: Limiting capabilities of gall agents, and related work.
author: ~palfun-foslup
status: Draft
type: Standards Track
category: Kernel
created: 2026-01-27
---


## Abstract

Gall agents have always had access to the entirety of the kernel API, as well as all other gall agents. Here we propose a permissions system, restricting the capabilities of agents on a given desk until a user grants corresponding permissions. We distinguish between static permissions, required by the desk prior to installation, and dynamic permissions, requested by a desk's agents at runtime.


## Status

xx working group, subject to model changes there
xx draft, more detail forthcoming after discussion and gall model solidification
xx no specifics about agent effects, subject to gall model changes


## Motivation

Gall implements a userspace model wherein "agents" can emit effects when they are evoked. While the shape and scope of these effects is dictated by gall, they map very closely onto the kernel API, with some additions for agent-to-agent interactions. Gall makes no effort to control, restrict or even monitor the effects emitted by the agents it runs.

In 2021, "software distribution" made it easy to install third-party userspace software. That work punted on most security-related questions, including ways to protect both kernel- and userspace against malicious third-party software.

This particular known security hole has been a big factor in preventing development and adoption of "high-stakes" software like secrets storage and cryptocurrency wallets.


## Specification

We first give an overview of the behavior of permissions. Then we describe the requirements made of the API, the interplay between permissions and desk and lifecycles, we make recommendations about permission presentation and management, and describe best practices for userspace developers regarding permission requirements and requests.

This is not a capabilities-based model. This does not aim to solve the "confused deputy" problem. Local provenance is utilized to realize this model. This does not aim to extend provenance over the network in any way.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Overview

All `%pass` cards emitted by agents and all `.^` namespace reads performed by agents MUST be subject to permission control by gall. Permissions MUST be granted at the desk level in clay. Permissions on a desk MUST apply to all agents running from that desk.

Agents on the `%base` desk MUST be exempted from this permission control ("run as root"), and any agent MUST always be allowed to issue cards which would delete/revoke one of its created resources/subscriptions. Outside of that, there are no implicit permissions. Even interactions or reads to an agent running on the same desk MUST require a corresponding permission.

Desks MAY statically specify required permissions, in a `/desk.seal` file. The desk MUST NOT run (install, revive) until the specified permissions are granted.

Agents MAY dynamically request optional permissions for their desk at runtime. These permissions can be granted and revoked at any time. Clay MUST track requested permissions and notify about additions (see below).

The `$bowl:gall` MUST include the permissions available to the agent at the time of its invocation. The standard library SHOULD provide helpers for checking a card or scry path against those permissions.

When an agent emits a `%pass` or performs a `.^` scry for which it doesn't have a corresponding permission, the agent invocation MUST be treated as having crashed. During `+on-init` and `+on-load`, this crashes the whole event. During all other invocations, this calls the agent's `+on-fail` with `%not-permitted`.

Both gall and clay need to know the current permission state (required, requested, granted) for a desk. Because permissions apply at the desk level and exist even when no agents are running, clay MUST be treated as the canonical store, with gall subscribing to clay for permission updates.

### Interface and Types

Clay MUST support granting and revoking permissions through a `%seal` task and requesting permissions through a `%pine` task.
To support userspace reactivity (and gall's syncing), clay MUST expose a kernel-style subscription endpoint for notifications about permission requests and updates through a `%ward` task, and closing of that subscription through a `%wink` task. Agents that want to use this MUST be granted the corresponding permission.

The type that describes a permission MUST allow for "scoping" if possible. That is to say, for example, a permission for a resource at a path must apply to resources at _and below_ the given path. xx example of (unit resource-id)

xx concrete type description once gall api settles
xx include difference between local vs remote vs both comms

xx (request) notification type/api. should we differentiate between runtime-requested (optional) and update-requested (required)?

xx upgrade behavior:
xx new permissions types MUST be introduced across kelvin updates, and base's perm manager MUST receive appetizer that provides new perm's rendering?
xx don't use perms within same kelvin release that introduces them. should have different types for "latest known" and "latest usable"

### Permissions and Desk Lifecycles

A desk MUST be granted its required permissions before being set to live. (That is, before clay signals to gall that the agents on that desk should run.)
Once live, a desk's required permissions MUST NOT be able to be revoked. To do so, the desk must first be suspended.
A commit to a live desk MUST fail to apply if doing so would add required permissions that have not yet been granted.

xx this failure MUST register the to-be-required permissions and send a perm request notification?

Desks that are installed as part of the boot sequence MUST have all their required permissions granted automatically.

Desks that are installed at the time of the upgrade which introduces userspace permissions MUST have all their required permissions granted automatically.

xx possibility to mark desk as "trusted", auto-granting all perms requests?

### Presentation and Management

The `%base` desk SHOULD ship with a permission manager for each interface modality.
- Generators SHOULD be available for granting and revoking permissions on a given desk, either case-by-case or "all requested".
- `|install` SHOULD support granting whatever permissions are required.
- Eyre SHOULD serve a primitive permission management web interface.

When presenting permissions to the user, if the agent affected by a permission is locally known, it SHOULD be presented as or alongside the name or description of the desk it is running from. A "poweruser" SHOULD be empowered to view the "raw" underlying permission.

When an agent affected by a permission is not locally known, a permission manager MAY prevent the granting of the permission, even if this would prevent installation.

The recommendations around "known" agents SHOULD only apply to local agent interactions. Interactions that go over the network SHOULD be presented generically as "sharing data" or "reading".
xx does that imply "poke over the network" and "watch over the network" perms shouldn't specify agent name? can we meaningfully narrow by mark?

When an agent affected by a permission is on the desk requesting the permission, a permission manager MAY collate it with similar permissions, or hide them completely.

Permissions that are functionally equivalent to root access (see Security Considerations below) SHOULD be presented as such.

### Requesting Permissions

Userspace developers SHOULD require (through the `/desk.seal` file) permissions that are essential to the core functioning of their desks.
For permissions without which the desk can reasonably function, these should be requested and checked at runtime (on-init/load, or on-demand). Developers SHOULD use the standard library-provided utilities for checking permission availability.

Developers SHOULD request permissions at the smallest reasonable scope. For example, when subscribing at paths of the shape `/data/[some-id]`, request permissions for `/data`, not for `/` (too broad), and not for each individual `/data/whatever` (too narrow).


## Rationale

Permissions apply at the desk level, not at the agent level, because desks map more closely to the concept of "apps" than agents do. A desk may contain many agents, but they're commonly intended to function together as a pre-packaged suite of software.
Desks are less volatile than agents. Agents within a desk may or may not be running, and an agent may move from running on one desk, to running on a different desk.
Lastly, desks are more closely tied to specific publishers than agents are. That tie is not absolute though (only "running foreign desks" would be), so switching sources for a desk remains a risk. See also security considerations below.

The `%base` desk is exempt from permission checking because it runs the kernel. Any code that makes it onto the `%base` desk is implicitly trusted. If the `%base` desk code cannot be trusted, neither can the permissions implementation.

There are no implicit permissions granted, even for interactions between agents from the same desk, because this complicates the model and implementation, and raises additional security concerns. Hiding or auto-granting self-referential permissions can be done at the display or userspace level if desired.

There is separation between required/static and optional/dynamic permissions for a couple of reasons.
- Strictly requiring permissions improves the ergonomics of writing agents. No matter how good the helpers are, a check is a check, necessitates code branching, increases complexity.
- Specifying permissions ahead-of-time helps users make more-informed decisions about whether to install any given software.
- Necessary permissions are not always knowable ahead of time. This is almost necessarily true for software that intends to interact with other apps in a generic way: it cannot know what apps the user will make it interact with.

Treating agent invocations from which the agent tries to do something it does not have permission for as crashes matches the behavior of `.^` on non-existent paths and makes it obvious that the entire invocation is null and void. The latter is important to avoid internal inconsistencies in agents. xx better phrasing
The alternative would be to inject a "permission nack" into `+on-agent` or `+on-arvo` to notify them that their effect was prevented from executing. This cannot be done for `.^`, and results in more "loose ends" for developers to handle. xx phrasing
Developers should be encouraged to check permissions prior to emitting effects so they can handle failure cases eagerly/synchronously, rather than firing them off indiscriminately and handling failures lazily/asynchronously.
(To improve developers' ability to handle resulting `+on-fail` calls appropriately, that interface should be expanded to provide more details about the failed invocation. However, doing so is out of scope for this UIP.)

### Interface and Types

Considering the possibility of building "app managers" in userspace, it is important for the kernel to expose permission information and management capabilities.

xx kernel-style subscriptions follow established pattern, see examples

xx scoping

xx kelvin upgrade / changed permission types discussion

### Permissions and Desk Lifecycles

Clay already manages desk "liveness" status and transitions. Permissions, applying at the desk level, overlap with this nicely.
xx desk liveness requirements

Automatically granting required permissions for desks present/installed during the boot sequence ensures the immediate post-boot state is "complete" according to the sequence's intent, without requiring additional permission-granting events to be formalized into the boot sequence.

Automatically granting required permissions for desks present/installed during the permissions upgrade ensures the upgrade can happen without user intervention.

### Presentation and Management

Permission managers are advised to prevent the granting of permissions affecting unknown agents. Disregarding this advice saddles the user with an unanswerable question: what does it mean to read from or write to an app I have no awareness of?
"I trust it will be fine" can be used as an answer, but doing so is not risk-free. It is possible to lower that risk by, when installing a new app, showing all permissions granted to other desks that affect the to-be-installed app.

All of that assumes the `/desk.bill` file is a complete list of all agents that will be running on a desk. For scenarios where there are "optional" agents on a desk, the above becomes more nuanced. Arguably, in those cases, the relevant permissions should be optional, requested at runtime only once the optional agent has been observed to be running.

Permissions concerning agents on other ships should be presented generically, because there can be no guarantee about what software is running under what name on remote ships.

### Requesting Permissions

Requesting permissions ahead-of-time improves both developer and user ergonomics: the developer never has to check permission status, and the user does not need to be prompted for those permissions after installing the app.
Requesting permissions dynamically grants users meaningful control over the behavior of the software they run, in practice letting them en- or disable features according to their level of trust in the software.


## Backwards Compatibility

Userspace developers will need to specify the permissions which their agents require. Failure to do so will result in runtime crashes.

xx summarize upgrade instructions for userspace devs


## Security Considerations

### Eyre Security

This UIP describes a permission system for gall-level interactions. Other vanes also interact with gall's userspace. In general, the kernel is trusted software, so does not need to be checked for permissions. Eyre is an exception to this: it allows outside callers (anything capable of making HTTP requests) to interact with gall. To make userspace permissions hold water in real-world scenarios, permission checks must be extended to these outside callers.

A detailed specification for that work is outside of the scope of this UIP. In summary: eyre will append a desk to its provenance path when interacting with gall. Gall will check for permission to perform that interaction, as if it had originated from that desk. Eyre determines that desk based on the scope of the authenticating cookie. Eyre safeguards scoped cookies against trivial theft by serving desks on their own subdomains.

At the time of writing, that line of work is being pursued, and a working prototype is [available on the `eswg/wip` branch of urbit/urbit](https://github.com/urbit/urbit/compare/develop...eswg/wip). Formal UIP to follow.

### Implicit Root

There are many seemingly-narrow permissions that are functionally equivalent to root, usually by interacting with something on the base desk. Care should be taken to enumerate all of these and inform the user about them appropriately whenever possible. A non-exhaustive list of such permissions is given below.

- pass `%dill` a `%belt`
- pass `%clay` a `[%park %base ...]`
- poke `%hood` with `&helm-pass`
- poke `%herm` with `%herm-task`

### Desk Sources

Changing the installation source of a desk for which permissions have been granted implies trusting the new source with all of the granted permissions. Revoking permissions when changing installation source may be excessive, but warning about this in installation UI could be sensible.
Updating clay to support running off source desks directly (that is, running agents from a `[=ship =desk]` instead of a necessarily-local `desk`) would mitigate this, but such a change is outside the scope of this UIP.


## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
