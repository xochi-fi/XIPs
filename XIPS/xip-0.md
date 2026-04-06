---
xip: 0
title: The XIP Process
description: Defines how Xochi Improvement Proposals are written, reviewed, and accepted
author: DROO (@Hydepwns)
discussions-to: https://github.com/xochi-fi/XIPs/issues/1
status: Implemented
type: Meta
created: 2026-04-07
---

## Abstract

This XIP defines the process for proposing, discussing, and accepting changes to the Xochi protocol. It establishes proposal types, required sections, the status lifecycle, and editorial responsibilities.

## Motivation

Xochi needs a transparent, structured process for protocol changes. Ad-hoc decisions in chat don't scale, aren't auditable, and exclude the community. A formal proposal process creates a public record of what was decided, why, and what alternatives were considered.

## Specification

### What is a XIP?

A XIP is a design document providing information to the Xochi community or describing a change to the protocol. Each XIP MUST contain a concise technical specification and rationale for the proposed change.

### Proposal types

- **Protocol**: Changes to on-chain contracts, settlement logic, fee structures, proof systems, or trust scoring. These affect the core protocol.
- **Interface**: Changes to API surfaces, intent schemas, widget SDK, or client-facing behavior. These affect integrators.
- **Meta**: Changes to the XIP process, governance rules, or team structure. These affect how decisions are made.
- **Informational**: Design guidelines, best practices, or ecosystem context. These don't propose changes.

### Required sections

All XIPs MUST include:

1. **Abstract**: 2-3 sentence summary.
2. **Motivation**: Why the change is needed.
3. **Specification**: Precise technical description. Protocol and Interface types MUST use RFC 2119 normative language.
4. **Security Considerations**: Threats and mitigations. Proposals without this section MUST be rejected regardless of merit.
5. **Failure Modes**: Degradation scenarios and blast radius.

Protocol type XIPs MUST also include Test Cases (may be links to test files).

### Frontmatter

Every XIP MUST include YAML frontmatter with these fields:

| Field            | Required | Format                                   |
| ---------------- | -------- | ---------------------------------------- |
| `xip`            | Yes      | Integer (assigned by editor)             |
| `title`          | Yes      | Max 44 characters, not a sentence        |
| `description`    | Yes      | Max 140 characters, one sentence         |
| `author`         | Yes      | `Name (@github), ...`                    |
| `discussions-to` | Yes      | URL to forum thread or issue             |
| `status`         | Yes      | One of the defined statuses              |
| `type`           | Yes      | Protocol, Interface, Meta, Informational |
| `created`        | Yes      | ISO 8601 date                            |
| `requires`       | Optional | Comma-separated XIP numbers              |
| `implementation` | Optional | URL, added after status = Implemented    |

### Status transitions

```
Draft -> Review:       Author believes proposal is complete
Review -> Last Call:   Editor moves to Last Call, sets 14-day deadline
Last Call -> Approved: No blocking issues after deadline
Approved -> Implemented: Code deployed, implementation link added
Any -> Withdrawn:      Author or team pulls the proposal
Any -> Deferred:       Team parks the proposal for later
```

A proposal in Last Call MUST wait the full 14 days. If blocking issues are raised, it returns to Review.

A proposal SHOULD NOT stay in Draft for more than 90 days. After 90 days of inactivity, the editor MAY move it to Deferred.

### Editorial process

The editor (currently the core team) is responsible for:

1. Assigning proposal numbers on merge
2. Checking frontmatter format
3. Moving proposals through status transitions
4. Maintaining the XIPs index in README.md

The editor does not judge technical merit. Review and approval are separate from editorial process.

### Submission process

1. Fork the XIPs repository.
2. Copy `xip-template.md` to `XIPS/xip-draft_your-title.md`.
3. Write the proposal. Follow the template structure.
4. Open a pull request. Title format: `XIP-draft: Your Title`.
5. The editor reviews formatting and assigns a number on merge.
6. Open a discussion thread (GitHub issue or forum) and add the URL to `discussions-to`.

### Amendments

Substantive changes to an Approved or Implemented XIP require a new XIP that explicitly supersedes the original. Non-substantive changes (typos, clarifications that don't change meaning) can be made via PR to the original.

## Security Considerations

The XIP process itself has no direct security implications. However, proposals that pass through this process affect protocol security. The mandatory Security Considerations and Failure Modes sections exist to ensure security review is not optional.

A risk: social engineering via well-written but subtly flawed proposals. Mitigated by requiring discussion periods and multiple reviewers for Protocol type XIPs.

## Failure Modes

If the XIP process becomes too bureaucratic, contributors will route around it and make changes informally. The process should remain lightweight enough that writing a XIP is faster than having the same discussion repeatedly in chat.

If the process is too permissive, low-quality proposals consume review bandwidth. The editor role and required sections help filter.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
