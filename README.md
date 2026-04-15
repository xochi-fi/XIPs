# XIPs

Xochi Improvement Proposals.

XIPs are the primary mechanism for proposing changes to the Xochi protocol -- new features, parameter changes, process updates, and standards. Anyone can write a XIP. The process is modeled on [Ethereum's EIPs](https://eips.ethereum.org/) and [Lido's LIPs](https://github.com/lidofinance/lido-improvement-proposals).

## Proposal types

| Type              | Description                                                                       |
| ----------------- | --------------------------------------------------------------------------------- |
| **Protocol**      | Changes to on-chain contracts, settlement logic, fee structures, or proof systems |
| **Interface**     | Changes to API surfaces, intent schemas, or client-facing behavior                |
| **Meta**          | Process changes, governance rules, or changes to the XIP process itself           |
| **Informational** | Design guidelines, best practices, or context that doesn't propose a change       |

## Status lifecycle

```
Draft --> Review --> Last Call (14 days) --> Approved --> Implemented
                                        \-> Withdrawn
                          \-> Deferred (parked, revisit later)
```

- **Draft**: Work in progress. Not ready for formal review.
- **Review**: Proposal is complete and seeking feedback.
- **Last Call**: Final comment period (14 days minimum). If no blocking issues raised, moves to Approved.
- **Approved**: Accepted by the team. Implementation may proceed.
- **Implemented**: Deployed to production. Implementation link added to frontmatter.
- **Withdrawn**: Author or team has pulled the proposal. Terminal state.
- **Deferred**: Not rejected, but not actively being pursued. Can be reopened.

## Writing a XIP

1. Fork this repo
2. Copy `xip-template.md` to `XIPS/xip-draft_your-title.md`
3. Fill in the template. Read [XIP-0](XIPS/xip-0.md) for rules.
4. Open a PR. The proposal number gets assigned on merge.
5. Discuss in the linked forum thread or PR comments.

### Required sections

Every XIP MUST include:

- **Abstract** -- What this proposal does, in 2-3 sentences.
- **Motivation** -- Why the current state is inadequate.
- **Specification** -- The precise technical change. Use RFC 2119 language for normative requirements.
- **Security Considerations** -- Threats, risks, and mitigations. Proposals without this section will be rejected.
- **Failure Modes** -- What happens if this goes wrong? How does the system degrade?

Optional but recommended: Rationale, Backwards Compatibility, Test Cases, Reference Implementation.

## Directory structure

```
/
  README.md
  LICENSE.md                     # CC0-1.0
  xip-template.md                # Template for new proposals
  XIPS/
    xip-0.md                     # Meta: the XIP process itself
    xip-1.md                     # First protocol proposal
    ...
  assets/
    xip-1/                       # Supporting files per proposal
      diagram.png
      data.json
```

## XIPs index

| Number                 | Title           | Type | Status      |
| ---------------------- | --------------- | ---- | ----------- |
| [XIP-0](XIPS/xip-0.md) | The XIP Process | Meta | Implemented |
| [XIP-1](XIPS/xip-draft_settlement-splitting.md) | Settlement Splitting for Large Trades | Protocol | Draft |
| [XIP-2](XIPS/xip-draft_adaptive-settlement-controls.md) | Adaptive Settlement Controls | Protocol | Draft |

## License

All XIPs are [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
