---
uip: "[TBD]"
title: Collaborative Working Groups Framework
description: Framework for establishing cross-organizational technical working groups
author: ~nisfeb, ~tiller-tolbus
status: Draft
type: Process
created: 2025-09-03
---

## Abstract

This UIP establishes a framework for creating and managing cross-organizational technical working groups to advance shared open source initiatives within the Urbit ecosystem. The framework defines working group formation, governance, meeting structure, membership mechanisms, and shared repository management.

## Motivation

Open source development across multiple organizations in the Urbit ecosystem requires structured collaboration to ensure efficient progress toward shared goals. Current ad-hoc collaboration methods lack formal structure for accountability, decision-making, and resource allocation. This proposal provides a standardized framework that enables multiple organizations to collaborate effectively on technical initiatives while maintaining transparency and community involvement.

## Specification

### Working Group Charter

Each working group MUST operate under a charter that defines:
- Scope of work
- Expected deliverables
- Success metrics
- Timeline for completion

Initial working group duration SHALL be 6 months with renewal option based on progress evaluation.

### Working Group Governance

Each working group MUST have one designated chair selected through the following process:

1. Partner organizations MAY nominate candidates
2. Candidates MUST submit qualification statements (maximum 1 page)
3. Each partner organization receives one vote
4. The Core Guild SHALL break ties

Chair responsibilities include:
- Scheduling and facilitating regular meetings
- Tracking deliverables and reporting progress
- Serving as primary point of contact
- Ensuring meeting minutes are recorded and distributed

Chair terms SHALL be 6 months, renewable once (12 months maximum).

### Meeting Structure

Working groups SHALL conduct bi-weekly 60-minute meetings. Such meetings MAY follow this template:

```
WORKING GROUP: [Name]
DATE: [Date]
ATTENDEES: [Names/Organizations]

1. Action Item Review (10 min)
   - [Previous items with status updates]

2. Technical Discussion (35 min)
   - [Topic 1]
   - [Topic 2]

3. Decisions Made (5 min)
   - [Decision 1]
   - [Decision 2]

4. New Action Items (10 min)
   - [Item] - [Owner] - [Due Date]

5. Next Meeting: [Date/Time]
```

All meetings MUST be documented in the shared Git repository.

### Membership Mechanism

#### Composition

Working groups SHALL start with three or more members.

#### Requirements

Members MUST meet the following requirements:
- Technical expertise relevant to working group focus
- Minimum commitment of 1 hour per week
- Formal support from sponsoring organization

#### Addition Process

1. Nomination (self or organization)
2. Application stating qualifications and contribution goals
3. Simple majority vote of existing members required for approval
4. Maximum of 2 representatives per organization per working group

#### Removal Process

Members MAY be removed for:
- Missing 3 consecutive meetings without notice
- Failure to complete assigned tasks for 30+ days

Removal requires chair recommendation and majority vote.

### Shared Repository Structure

A central Git repository named "collaborative-working-groups" SHALL be established with the following structure:

```
/
├── working-groups/
│   ├── <working-group-title>
│   │   ├── charter.md
│   │   ├── meetings/
│   │   ├── proposals/
│   │   └── work-products/
│   ├── <working-group-title>
│   └── <working-group-title>
├── governance/
│   ├── working-group-template.md
│   ├── meeting-template.md
│   └── member-registry.md
├── shared-resources/
└── README.md
```

#### Access Control

- Public read access for transparency
- Write/Admin access limited to working group members

#### Content Requirements

The repository MUST contain:
- Working group charters
- Meeting minutes in markdown format
- Technical proposals
- Decision records
- Work products and deliverables
- Member registry
- Governance documents

#### Pull Request Process

All substantive changes to working group documentation and deliverables MUST utilize pull requests with required review from at least one other working group member.

## Rationale

This framework draws from successful models in other open source communities (IETF, W3C, Linux Foundation) while adapting to Urbit's unique governance structure. The bi-weekly meeting cadence balances progress with participant availability. The 6-month term limits ensure fresh perspectives while allowing continuity through renewal. Public repositories ensure transparency while controlled write access maintains quality.

The three initial working groups address critical areas identified by the community: Bitcoin integration for identity, flexible sponsorship mechanisms, and Eyre security improvements. These topics require coordinated effort across multiple organizations and will benefit from structured collaboration.

## Resource Requirements

### Technical Infrastructure
- Git repository with appropriate permissions granted to members, elders, and editor of the Core Guild.
- Communication platform, meeting scheduling and recording tool.

### Organizational Support
Each sponsoring organization commits to supporting their representatives with adequate time allocation and resources to fulfill working group responsibilities.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
