# Uniswap v4 Hooks Security Skills

A collection of open-source [Claude Code](https://claude.ai/claude-code) skills focused on security review of Uniswap v4 hooks and the protocols built on top of them.

These skills package the public corpus of v4 hook security knowledge — known vulnerability classes, heuristics from audit reports, post-mortems from live exploits — into a structured review workflow that Claude Code can walk through against a target hook.

## Why

Hooks are powerful. They also unlock a large, nuanced attack surface that didn't exist in v3: misconfigured permissions, callback access control, custom accounting math, flash accounting deltas, dynamic fee abuse, tick crossing bugs, JIT manipulation, and more. Builders shipping hooks today have to internalize lessons from a fast-growing pile of public audit findings to ship safely.

These skills exist to make that easier. Point Claude Code at a hook contract and the skill walks through the known vulnerability classes systematically, flagging issues for human review.

## Skills

### v4-hooks-review

**Review a Uniswap v4 hook contract for known security issues.**

Walks the hook (and any associated periphery / accounting contracts) through nine categories of known v4 hook vulnerabilities, producing a structured report with findings, severity, and remediation pointers:

| Phase | What it checks |
| --- | --- |
| 1. Hooks & Permissions | Permission flag bits vs. implemented functions, address validation, bypasses via direct `PoolManager` access, pool key validation |
| 2. Hook Callbacks | `unlockCallback` access control, return data encoding, before-vs-after placement of logic |
| 3. Unhandled Reverts | `sync` before `unlock`, uncleared dust, reverts on liquidity modification, reverts in peripheral logic |
| 4. Dynamic Fees | Dynamic fee flag handling, override flag, `afterInitialize` setup, fee abuse by privileged callers |
| 5. Custom Accounting | Input validation, fund segregation, swap logic invariants, delta sign conventions, recursive LP tokens |
| 6. Reentrancy | Unsafe external calls in hook + periphery, reentrancy guards, read-only reentrancy through quoters |
| 7. Rounding & Precision | Floor/ceil rounding direction, share-price inflation surfaces, atomic liquidity manipulation |
| 8. Tick Trouble | Tick alignment to spacing, usable tick bounds, operations at zero liquidity, tick crossing math |
| 9. Miscellaneous | Native ETH handling, JIT liquidity, router parameter mistakes, custom incentive logic |

**Features:**

- Phase-by-phase walkthrough — each category is a self-contained reference file the skill loads on demand
- Heuristic-driven — every check is a concrete question a reviewer can answer against the code
- Links to primary sources — every category cites the audit reports, post-mortems, or spec sections it derives from
- Works on individual contracts or full repositories (Foundry, Hardhat)

## Disclaimer

**This is not a substitute for a professional security audit.** The skill is a structured review aid — it helps a human reviewer (or Claude) walk through known vulnerability classes systematically. It does not catch novel bugs, complex multi-contract interactions, or anything outside the categories encoded in its reference files. Use it before sending code to auditors, not instead of sending it to auditors.

## Prerequisites

- [Claude Code](https://claude.ai/claude-code) installed and authenticated

## Install

Clone the repo and symlink the skill into your Claude Code skills directory:

```bash
git clone https://github.com/AAYUSH-GUPTA-coder/uniswap-v4-hooks-security-cyfrin.git ~/uniswap-v4-hooks-security-cyfrin
ln -s ~/uniswap-v4-hooks-security-cyfrin/v4-hooks-review ~/.claude/skills/v4-hooks-review
```

Verify the skill is discoverable:

```bash
ls ~/.claude/skills/v4-hooks-review/SKILL.md
```

To update later:

```bash
cd ~/uniswap-v4-hooks-security-cyfrin && git pull
```

## Run

Navigate to a project containing a v4 hook and start Claude Code:

```bash
cd /path/to/your/project
claude
```

Run the full review:

```bash
review this v4 hook
```

Or scope to a specific category:

```bash
check v4 hook permissions
check v4 hook custom accounting for delta sign bugs
check v4 tick handling
```

Or point at a specific file:

```bash
review src/MyHook.sol for known v4 hook vulnerabilities
```

## Contributing

PRs and issues welcome. New categories, new heuristics, or new examples from recent audit reports — all of it makes the skill more useful. See [`CLAUDE.md`](./CLAUDE.md) for skill authoring guidelines.

## License

[MIT](./LICENSE) © Aayush Gupta

## Acknowledgements

The category structure and many of the heuristics in this skill are adapted from Cyfrin's deep dive [_Uniswap V4 Hooks Security Deep Dive_](https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive), which itself synthesizes findings from public audit reports by Trail of Bits, Pashov Audit Group, Certora, Guardian Audits, and others. Each reference file cites the specific findings and post-mortems it draws from.

If you find this useful and want to support the people doing the underlying research: [Cyfrin](https://www.cyfrin.io/), [Trail of Bits](https://www.trailofbits.com/), [Pashov Audit Group](https://www.pashov.net/), [Certora](https://www.certora.com/), [Guardian Audits](https://guardianaudits.com/), [BlockSec](https://blocksec.com/), and [Composable Security](https://composable-security.com/).