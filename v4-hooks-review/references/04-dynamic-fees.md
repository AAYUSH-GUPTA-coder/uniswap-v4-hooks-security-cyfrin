# 04 — Dynamic Fees

How v4 dynamic fees work, the two ways they're updated, and what breaks when either path is wired incorrectly.

## Background

A v4 pool can opt into dynamic fees by setting `LPFeeLibrary.DYNAMIC_FEE_FLAG` in `PoolKey.fee` at initialization. After that, the LP fee can change without re-initialising the pool. There are two ways to change it:

1. **Stateful update** — the hook (or a permissioned address) calls `PoolManager.updateDynamicLPFee(key, newFee)`. The new fee is stored and used on every subsequent swap until updated again.
2. **Per-swap override** — `beforeSwap` returns `fee | LPFeeLibrary.OVERRIDE_FEE_FLAG`. The override applies only to the current swap, leaving stored fee unchanged.

Pools initialised with the dynamic flag start at **0% fee**. If `afterInitialize` doesn't immediately set a sensible default via `updateDynamicLPFee`, the pool is initially free-to-swap.

## Heuristics — wiring

- **Is `DYNAMIC_FEE_FLAG` set in `PoolKey.fee` at initialization?** If a hook intends to charge variable fees but the flag is missing, the stored fee is locked and the hook's fee logic does nothing.
- **Is `afterInitialize` setting a default fee?** Otherwise the pool ships at 0% until someone manually fixes it.
- **For per-swap overrides, is `OVERRIDE_FEE_FLAG` OR'd into the returned fee value?** Without it, the returned fee is treated as informational and the stored fee is used.
- **Are the fee values within `LPFeeLibrary`'s valid range** (≤ 1,000,000 = 100%)? Out-of-range overrides revert in `beforeSwap`.
- **If the hook returns a fee from `beforeSwap`, is the `BEFORE_SWAP_RETURNS_DELTA_FLAG` permission set?** Mixed up flags here are surprisingly common.

## Heuristics — accounting

- **Where do dynamic fees accrue?** Standard LP fees flow to in-range LPs. Hook-managed fees (where the hook overrides delta accounting in `beforeSwap` / `afterSwap`) flow wherever the hook directs them. Confirm the intended destination matches the implementation.
- **Are stale fees handled on swap-direction changes?** A common pattern: fee A on zero-for-one, fee B on one-for-zero. If the override picks the wrong one, the protocol charges the wrong side.
- **Are fee transitions atomic?** A fee that updates partway through a swap (e.g. `beforeSwap` updates state then `afterSwap` reads it) can have surprising semantics. Update only on lifecycle boundaries.

## Heuristics — abuse

When fees are settable by a non-protocol party (an auction winner, a manager role, a DAO), confirm there are bounds:

- **Is there a max-fee cap?** A malicious or compromised setter can otherwise push fees to 100% and capture all swap value.
- **Is there a min-fee floor?** A malicious setter who wants to game TWAP or fee-share rewards might set the fee to zero.
- **How quickly can fees change?** A setter who can flip the fee per block can avoid taxes on their own swaps and apply them to everyone else's.
- **If the hook drives a TWAP that's used as an external oracle**, can a setter swing the fee to manipulate the implied price without paying arbitrage cost? This is the Bunni-style attack: cheap on-chain price manipulation that's externalized to other protocols.

## What to grep for

```bash
# Dynamic flag
grep -RE "DYNAMIC_FEE_FLAG|isDynamicFee" .

# Override flag
grep -RE "OVERRIDE_FEE_FLAG" .

# Direct fee updates
grep -RE "updateDynamicLPFee" .

# afterInitialize default-setting
grep -RE "function afterInitialize" -A 30 .

# Who can set fees?
grep -RE "setFee|updateFee|setLPFee" .
```

## Real-world examples

- **Bunni V2 (Trail of Bits, TOB-BUNNI-11)** — auction-managed AMM let a manager manipulate the on-chain TWAP without arbitrage cost; the manipulated price was consumed by an external lending protocol's oracle, enabling collateral abuse.
- **Bunni V2 (High)** — manager could capture fees at the protocol's expense.
- **Various hook audits** — `DYNAMIC_FEE_FLAG` set but `afterInitialize` never set a default, leaving the pool at 0% LP fee.

## Primary sources

- Uniswap Labs — [`LPFeeLibrary`](https://github.com/Uniswap/v4-core/blob/main/src/libraries/LPFeeLibrary.sol)
- Uniswap Labs — [Dynamic fees docs](https://docs.uniswap.org/contracts/v4/concepts/dynamic-fees)
- Trail of Bits — Bunni V2 audit report
- Cyfrin — [Uniswap V4 Hooks Security Deep Dive](https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive)
