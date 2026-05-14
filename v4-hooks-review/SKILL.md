---
name: v4-hooks-review
description: Security review of Uniswap v4 hook contracts. Use this skill whenever the user mentions reviewing, auditing, or checking a Uniswap v4 hook, asks about v4 hook vulnerabilities or attack vectors, references specific hook functions (beforeSwap, afterSwap, beforeAddLiquidity, etc.) in a security context, mentions custom accounting / flash accounting / ERC-6909 deltas in a hook, or asks to walk a hook through known issues. Trigger this even on partial cues like "is my hook safe?" or "look at this hook for bugs" — the skill walks the contract through nine categories of known v4 hook vulnerabilities (permissions, callbacks, unhandled reverts, dynamic fees, custom accounting, reentrancy, rounding, tick handling, and miscellaneous) and produces a structured findings report.
---

# v4-hooks-review

Structured security review for Uniswap v4 hook contracts.

## When this skill runs

The user has either pointed you at a hook contract, a directory containing one, or asked a question about v4 hook safety. Your job is to walk the target through the nine vulnerability categories below, surface concrete findings, and produce a report.

You are **not** running a fuzzer or formal verifier. This is a structured human-style review aided by a checklist — output should look like an audit reviewer's notes, not a static analyzer's dump.

## Workflow

1. **Identify the target.** Locate the hook contract(s). Check for:
   - The hook's deployed (or salt-mined) address — permission flags live in the low 14 bits
   - Inheritance — does it extend `BaseHook` (v4-periphery)? If not, expect more findings under Phase 1
   - Associated contracts — periphery routers, accounting hubs, vaults, incentive distributors

2. **Run each phase in order.** For each phase, read the corresponding reference file under `references/`, then check the heuristics against the target. Phases are independent — if a phase doesn't apply (e.g. the hook has no dynamic fee logic), note that and move on.

3. **For each finding**, record:
   - **Category** — which phase / reference file it came from
   - **Location** — file:line of the problematic code
   - **Severity** — Critical / High / Medium / Low / Informational
   - **Description** — what's wrong, in plain prose
   - **Impact** — what an attacker (or honest user) loses if this is exploited
   - **Fix** — concrete remediation (not just "validate input")

4. **Produce the report.** You MUST write the final report to a new markdown file in the current directory (e.g., `v4-hook-security-review.md`). Do not just print it in the terminal. Group by severity (Critical first), then by category. End with a summary table and a list of categories that were checked but had no findings (so the human reviewer can see coverage).

## Phases and reference files

Load the reference file at the start of each phase. Each one is self-contained and cites the audit reports / post-mortems / spec sections it draws from.

| # | Category | Reference |
|---|----------|-----------|
| 1 | Hooks & Permissions | [`references/01-hooks-and-permissions.md`](references/01-hooks-and-permissions.md) |
| 2 | Hook Callbacks | [`references/02-hook-callbacks.md`](references/02-hook-callbacks.md) |
| 3 | Unhandled Reverts | [`references/03-unhandled-reverts.md`](references/03-unhandled-reverts.md) |
| 4 | Dynamic Fees | [`references/04-dynamic-fees.md`](references/04-dynamic-fees.md) |
| 5 | Custom Accounting | [`references/05-custom-accounting.md`](references/05-custom-accounting.md) |
| 6 | Reentrancy | [`references/06-reentrancy.md`](references/06-reentrancy.md) |
| 7 | Rounding & Precision | [`references/07-rounding-and-precision.md`](references/07-rounding-and-precision.md) |
| 8 | Tick Trouble | [`references/08-tick-trouble.md`](references/08-tick-trouble.md) |
| 9 | Miscellaneous | [`references/09-miscellaneous.md`](references/09-miscellaneous.md) |

## Scoping

- **Full review**: run all nine phases. Default when the user says "review this hook" or similar.
- **Single phase**: if the user names a category ("check tick handling", "audit the custom accounting"), run just that phase and skip the others. You can still mention if neighbouring phases would be worth running.
- **Diff mode**: if the user asks about a specific change ("did this PR introduce any v4 hook bugs?"), restrict each phase to the diff but still load all nine reference files — many bugs manifest in interactions across categories.

## What's out of scope

This skill does not cover:

- **Generic Solidity bugs** unrelated to v4-specific surface area (use a general smart contract review skill for those — e.g. [solidity-common-mistakes-skill](https://github.com/AAYUSH-GUPTA-coder/solidity-common-mistakes-skill))
- **Malicious hooks** — the threat model assumed here is *benign but vulnerable*. For unverified hook integrations the relevant concerns (upgradeability, key compromise, governance) are protocol-level
- **Gas optimisation** — flag obvious DoS-via-gas patterns, but not micro-optimisation

## After the review

If findings exist, offer to:

- Draft remediation diffs for any straightforward fixes (alignment validation, missing access control modifiers, permission flag corrections)
- Write Foundry tests that reproduce the highest-severity findings
- Map the findings to the format used by audit firms the user is preparing to engage with

If the review surfaces nothing significant, say so plainly. Suggest the user still engage a professional auditor before mainnet — this skill is a preparation aid, not a verdict.