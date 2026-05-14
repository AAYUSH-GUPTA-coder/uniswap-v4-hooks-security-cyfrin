# 05 — Custom Accounting

The most dangerous category. Hooks that override delta accounting take custody of pool value; bugs here are typically critical.

## Background

Custom accounting in v4 means: instead of letting the underlying concentrated-liquidity AMM compute the deltas for a swap or liquidity action, the hook returns its own deltas (via `BeforeSwapDelta` from `beforeSwap`, or `int128` from `afterSwap` / `afterAddLiquidity` / `afterRemoveLiquidity` when the corresponding `RETURNS_DELTA` flag is set). The hook is also free to mint, burn, or transfer ERC-6909 claim tokens to settle those deltas internally.

This unlocks designs that aren't possible in v3 (oracle-backed AMMs, fixed-rate AMMs, request-for-quote, intent-based execution) but it also moves the AMM's core invariants into hook code, which is *not* battle-tested. Most live-exploit critical findings on v4 hooks land in this category.

## Heuristics — input validation

When a hook exposes externally-callable functions that touch its accounting (rebalance triggers, settlement helpers, pre/post-trade hooks for off-chain order systems), every input matters:

- **Who's allowed to call?** "Permissionless" rebalance functions can be invoked outside the intended flow, draining vault balances.
- **Are paired operations actually paired?** If `preFulfill` and `postFulfill` can be called independently, accounting becomes asymmetric — the protocol pays out without taking the matching input, or vice versa.
- **Are amounts / recipients caller-controlled or protocol-derived?** Anything caller-controlled needs allowlisting or invariant checks.
- **Can the same call be replayed?** Nonces, one-shot flags, monotonic counters — any of these missing is an obvious replay vector.

## Heuristics — fund segregation

Hooks typically hold several balance types at once: pool reserves, protocol fees, LP fees, donation residues, incentive tokens, fee-share buffers. Each needs to be accounted for separately.

- **Can one beneficiary's balance be paid from another's pool?** E.g. user A's deposit being usable to pay user B's withdrawal, or LP fees being drained as protocol fees.
- **Are ERC-20 and ERC-6909 representations kept consistent?** Mixed accounting is a known critical surface — the manager's ERC-6909 ledger and the contract's ERC-20 balance can be made to disagree by abusing settlement order.
- **What happens with recursive LP tokens** — the LP token of pool A being used as a currency in pool B? Yield accrued to the LP token can sometimes be stolen by abusing flash accounting (sync → claim-on-behalf-of-manager → settle → take).
- **Are fee / donation residues handled distinctly from principal?** Mixing them is a frequent source of dust-induced revert DoS (see `03-unhandled-reverts.md`).

## Heuristics — swap math invariants

If the hook overrides swap pricing, the burden of preserving AMM invariants moves onto hook code. Things that should hold:

- **Zero input → zero output.** A swap specifying zero `amountSpecified` (or specifying zero on the side actually being paid) should not produce non-zero output. "Free swap" bugs come from this category.
- **Round-trip ≤ 0.** Selling X for Y, then Y for X, should not result in more X than you started with. Round-trip-positive bugs are the classic price-curve break.
- **Same amount, different specification, same result.** An exact-input swap of X tokens A should produce the same output as an exact-output swap requesting that same output amount of B. If these diverge, an attacker picks whichever side wins.
- **Liquidity ≥ 0, well-defined at boundaries.** Operations at zero or near-infinite liquidity must not panic, underflow, or return garbage.

## Heuristics — delta sign conventions

The hardest bugs in this category are sign errors. v4's sign convention is consistent (negative deltas are "owed to the pool"; positive deltas are "owed to the user") but easy to invert by accident.

For `beforeSwap`:
- The returned `BeforeSwapDelta` is decoded as `(int128 hookDeltaSpecified, int128 hookDeltaUnspecified)`. Specified = the currency named in `params.amountSpecified`; unspecified = the other one.
- Core `Hooks.callBeforeSwap` adds `hookDeltaSpecified` to `amountToSwap`. If you get the sign wrong, you can flip exact-input swaps into exact-output (the manager will revert on this, but only after you've burned gas) or shrink/grow the swap unintentionally.

For `afterSwap`:
- The returned `int128` adjusts the unspecified currency's delta. Wrong sign here directly inverts payment direction.

Common patterns to check:

- **Does the hook subtract when it should add, or vice versa?** Often the bug is a single `-` that should be `+`.
- **Is the same delta credited in both `beforeSwap` and `afterSwap`?** Double counting.
- **Does an exact-input intent silently turn into something else?** Validate that the swap type is preserved.

## Heuristics — caller delta vs hook delta

The `swapDelta` returned by `PoolManager.swap` is the *caller's* perspective: positive means the caller is owed tokens, negative means the caller owes tokens. The hook's `afterSwap` return is the hook's perspective. Mixing these up in periphery code that bridges between the two — routers, settlement contracts, periphery accounting — produces silent value leakage.

## What to grep for

```bash
# Custom delta returns
grep -RE "BeforeSwapDelta|toBeforeSwapDelta" .
grep -RE "function afterSwap" -A 30 .

# ERC-6909 mints/burns/transfers from hook
grep -RE "(mint|burn|transferFrom|safeTransferFrom)\s*\(" .

# Sign manipulation
grep -RE "amount0|amount1|delta0|delta1" .

# Pre/post hook pairing
grep -RE "(pre|post)(Hook|Trade|Order|Fulfill)" .
```

## Real-world examples

- **Bunni V2 (Trail of Bits, multiple)** — free swaps, round-trip-positive swaps, exact-input/exact-output asymmetries, arithmetic panics, and liquidity underestimates all surfaced from custom swap math.
- **Bunni V2 (TOB-BUNNI-6/7/8)** — permissionless callable rebalance pre/post hooks let an attacker drain the central hub by triggering asymmetric flows.
- **Various** — recursive LP token abuse where one pool's LP token, used as currency in another pool, lets an attacker drain rewards by abusing flash accounting settlement ordering.
- **C-05 (Pashov-style finding)** — mismatched raw ERC-20 vs ERC-6909 accounting allowed draining of underlying pool currency via a maliciously-recursive pool.

## Primary sources

- Uniswap Labs — [Custom accounting docs](https://docs.uniswap.org/contracts/v4/guides/hooks/custom-accounting)
- Uniswap Labs — [`BeforeSwapDelta` library](https://github.com/Uniswap/v4-core/blob/main/src/types/BeforeSwapDelta.sol)
- Trail of Bits — Bunni V2 audit report (most thorough public reference for this category)
- Cyfrin — [Uniswap V4 Hooks Security Deep Dive](https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive)
