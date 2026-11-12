---
xip: 0
title: The XIP Process
description: Defines how Xochi Improvement Proposals are written, reviewed, and accepted
author: DROO (@DROOdotFOO)
discussions-to: https://github.com/xochi-fi/XIPs/issues/5
status: Implemented
type: Meta
created: 2026-04-07
implementation: https://github.com/xochi-fi/XIPs
---

## Abstract

This XIP defines the process for proposing, discussing, and accepting changes to the Xochi protocol. It establishes proposal types, required sections, the status lifecycle, who approves each transition, and editorial responsibilities.

## Motivation

Xochi needs a transparent, structured process for protocol changes. Ad-hoc decisions in chat don't scale, aren't auditable, and exclude the community. A formal proposal process creates a public record of what was decided, why, and what alternatives were considered.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they appear in all capitals, as shown here.

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
3. **Specification**: Precise technical description. Protocol and Interface XIPs MUST use the BCP 14 key words above and MUST include a **Trust boundaries** subsection: for each MUST, the component that enforces it (contract, Oracle, circuit, server, SDK), and every caller- or prover-supplied input with what binds it.
4. **Security Considerations**: Threats and mitigations. A proposal without this section MUST be Rejected regardless of merit.
5. **Failure Modes**: Degradation scenarios and blast radius.
6. **History**: Once merged, one line per status change or revert: date, from -> to, PR, reason.

Protocol XIPs MUST also include Test Cases (they may link to test files). Rationale, Backwards Compatibility, and Reference Implementation are RECOMMENDED.

### Frontmatter

Every XIP MUST include YAML frontmatter with these fields:

| Field            | Required | Format                                                      |
| ---------------- | -------- | ----------------------------------------------------------- |
| `xip`            | Yes      | Integer assigned by the editor on merge; `TBD` before that  |
| `title`          | Yes      | Max 44 characters, not a sentence                           |
| `description`    | Yes      | Max 140 characters, one sentence                            |
| `author`         | Yes      | `Name (@github), ...`                                       |
| `discussions-to` | Yes      | URL of this XIP's own discussion issue in this repository   |
| `status`         | Yes      | One of the statuses below                                   |
| `type`           | Yes      | Protocol, Interface, Meta, Informational                    |
| `created`        | Yes      | ISO 8601 date                                               |
| `requires`       | Optional | Comma-separated XIP numbers this XIP technically depends on |
| `implementation` | Optional | URL; REQUIRED once status is Implemented                    |
| `supersedes`     | Optional | XIP number(s) this XIP replaces                             |
| `superseded-by`  | Optional | XIP number that replaced this one                           |

`requires` lists technical dependencies only. Every XIP follows XIP-0; it MUST NOT be listed.

### Statuses

- **Draft**: Work in progress.
- **Review**: Complete and seeking review.
- **Last Call**: Final comment period of 14 days.
- **Approved**: Accepted. Implementation may proceed.
- **Implemented**: Deployed. `implementation` link set.
- **Deferred**: Parked. Can return to Draft.
- **Withdrawn**: Pulled by the author. Terminal.
- **Rejected**: Refused by the editor with a recorded reason. Terminal.
- **Superseded**: Replaced by a later XIP named in `superseded-by`. Terminal.

### Status transitions

| From                  | To          | Actor  | Condition                                                                                       |
| --------------------- | ----------- | ------ | ----------------------------------------------------------------------------------------------- |
| Draft                 | Review      | Author | Spec complete; for Protocol and Interface XIPs, matches any merged implementation (see below)   |
| Review                | Last Call   | Editor | No open blocking issues; deadline set to 14 days from the transition date                       |
| Last Call             | Review      | Editor | A blocking issue is raised                                                                      |
| Last Call             | Approved    | Editor | Deadline passed with no open blocking issues                                                    |
| Approved              | Implemented | Editor | Code deployed and `implementation` set                                                          |
| Draft, Review         | Deferred    | Editor | 90 days (Draft) or 60 days (Review) without a commit to the XIP file                            |
| Deferred              | Draft       | Author | Author resumes work                                                                             |
| Any non-terminal      | Withdrawn   | Author | Author pulls the proposal; the editor MAY act if the author is unresponsive for 90 days         |
| Review, Last Call     | Rejected    | Editor | Missing Security Considerations, or an unresolved blocking issue the author declines to address |
| Approved, Implemented | Superseded  | Editor | A later XIP with `supersedes` reaches Approved                                                  |

Every transition is made by a pull request that changes `status` and adds a History line. Re-entering Last Call restarts the 14 days.

A **blocking issue** is an unresolved review thread or discussion comment that cites a section of the XIP and states a correctness, security, or compliance defect. The editor decides when it is resolved and records the decision in the thread.

### Approval

Every pull request that adds or changes a file under `XIPS/` MUST have at least one approving review from a person who is not an author of that XIP, and authors MUST NOT merge their own XIP's pull requests. This is enforced by branch protection on `main`: one approving review, stale approvals dismissed on new commits, approval of the last push, resolved conversations, and no administrator bypass.

### Spec and code

A Protocol or Interface XIP whose implementation is already merged MUST match that implementation before it enters Review. Each difference MUST either be fixed in the XIP or listed as "Not yet implemented" next to the requirement. Every XIP listed in `requires` MUST be at the same or a later status than the XIP that requires it.

### Removal and reverts

A merged XIP MUST NOT be deleted. Removal is a transition to Withdrawn or Rejected with the reason in History. If a merge is reverted, the History of the re-merged XIP MUST record the merge, the revert, and the reason. XIP numbers are never reused.

### Editorial process

The editors are the members of the core team with admin rights on this repository. The editor is responsible for:

1. Assigning proposal numbers on merge.
2. Checking frontmatter, including that `discussions-to` resolves to this XIP's own issue.
3. Making the status transitions in the table above.
4. Maintaining the XIPs index in README.md.

The editor does not judge technical merit. Approval of content comes from the non-author review required above.

### Submission process

1. Open a discussion issue in this repository titled `XIP-draft: Your Title`.
2. Fork the repository and copy `xip-template.md` to `XIPS/xip-draft_your-title.md`.
3. Write the proposal following the template, with `xip: TBD` and `discussions-to` set to the issue URL.
4. Open a pull request titled `XIP-draft: Your Title`.
5. After non-author approval the editor assigns a number, renames the file, updates the issue title, and merges.

### Amendments

A substantive change is any change to a line containing a BCP 14 key word, an interface, a constant, a parameter, or a test vector. Substantive changes to an Approved or Implemented XIP require a new XIP that supersedes the original. Other changes (typos, links, wording that does not change meaning) MAY be made by pull request to the original.

### Bootstrap

XIP-0 was adopted as Implemented without Review or Last Call because no process existed to review it. Its 2026-11-12 amendment is the last substantive change made in place; later substantive changes to XIP-0 require a superseding Meta XIP.

## Security Considerations

The XIP process has no direct on-chain effect, but proposals that pass through it shape protocol security.

- **Single-person approval**: mitigated by the non-author approval rule and branch protection without administrator bypass.
- **Subtly flawed proposals**: mitigated by the mandatory Trust boundaries subsection, the spec-and-code rule, and the 14-day Last Call.
- **Silent changes**: the amendment rule treats any change to a normative line as substantive; History records every status change and revert.

## Failure Modes

If the process becomes too bureaucratic, contributors will route around it and make changes informally. The process should stay light enough that writing a XIP is faster than having the same discussion repeatedly in chat.

If the process is too permissive, low-quality proposals consume review bandwidth. The editor role and required sections help filter.

If no non-author reviewer is available, proposals stall in Review rather than merge unreviewed; the 60-day Deferred rule makes the stall visible.

## History

- 2026-04-07: Implemented (bootstrap).
- 2026-11-12: Amended in place under the bootstrap clause: non-author approval, status table with actors, Rejected and Superseded statuses, History requirement, spec-and-code rule, Trust boundaries subsection, `discussions-to` fixed to #5. PR #8.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
