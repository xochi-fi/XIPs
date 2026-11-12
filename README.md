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

## Process

[XIP-0](XIPS/xip-0.md) is the single source for statuses, who approves each transition, required sections, and frontmatter. In short: open a discussion issue, open a pull request from the template, get one approving review from someone who is not an author, and the editor assigns a number on merge. Authors do not merge their own XIPs; branch protection enforces this.

## Writing a XIP

1. Open a discussion issue titled `XIP-draft: Your Title`.
2. Fork this repo and copy `xip-template.md` to `XIPS/xip-draft_your-title.md`.
3. Fill in the template with `xip: TBD` and `discussions-to` set to the issue. Read [XIP-0](XIPS/xip-0.md) for the rules.
4. Open a pull request titled `XIP-draft: Your Title`.

## Directory structure

```
/
  README.md
  xip-template.md                # Template for new proposals
  XIPS/
    xip-0.md                     # Meta: the XIP process itself
    xip-1.md                     # First protocol proposal
    ...
```

## XIPs index

| Number                 | Title                                 | Type     | Status      |
| ---------------------- | ------------------------------------- | -------- | ----------- |
| [XIP-0](XIPS/xip-0.md) | The XIP Process                       | Meta     | Implemented |
| [XIP-1](XIPS/xip-1.md) | Settlement Splitting for Large Trades | Protocol | Review      |
| [XIP-2](XIPS/xip-2.md) | Adaptive Settlement Controls          | Protocol | Draft       |

## License

All XIPs are [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
