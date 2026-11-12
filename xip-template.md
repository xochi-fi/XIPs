---
xip: TBD
title: <short title, max 44 characters>
description: <one sentence, max 140 characters>
author: <FirstName LastName (@GitHubUsername), ...>
discussions-to: <URL of this XIP's discussion issue, opened before the PR>
status: Draft
type: <Protocol | Interface | Meta | Informational>
created: <yyyy-mm-dd>
# requires: <XIP number(s) this XIP technically depends on; never 0>
# implementation: <URL, required once Implemented>
# supersedes: <XIP number(s)>
---

## Abstract

2-3 sentence technical summary. What does this proposal do?

## Motivation

Why is the current state inadequate? What problem does this solve? Include concrete examples where possible.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they appear in all capitals, as shown here.

The precise technical change. For protocol changes, include interface definitions, parameter values with units, and state transitions. For interface changes, include request/response schemas. For meta changes, describe the process modification.

If an implementation is already merged, the specification MUST match it; mark anything not yet built as "Not yet implemented".

### Trust boundaries

Required for Protocol and Interface XIPs.

| Requirement        | Enforced by                                  |
| ------------------ | -------------------------------------------- |
| <each MUST, short> | <contract / Oracle / circuit / server / SDK> |

List every input supplied by a caller or prover and what binds it (a signature, a public input checked on-chain, a server lookup). Say which requirements nothing enforces.

## Rationale

Why this design over alternatives? What tradeoffs were made? What was considered and rejected?

## Backwards Compatibility

Does this break existing behavior? If so, what's the migration path?

## Test Cases

Input/output pairs or scenarios demonstrating correct behavior, with units. Required for Protocol XIPs. Every vector MUST satisfy every requirement in the Specification.

## Reference Implementation

Minimal working code, or links to the implementation pinned to a commit.

## Security Considerations

Threats, attack vectors, and mitigations. This section is mandatory. Proposals without it are Rejected.

Think about: proof soundness, who chooses each input, economic incentives, griefing vectors, MEV, privacy leakage, trust assumptions, centralization risks.

## Failure Modes

What happens if this goes wrong? How does the system degrade? What's the blast radius?

Consider: partial failures, dependency failures, adversarial conditions, unexpected load.

## History

One line per status change or revert after merge: date, from -> to, PR, reason.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
