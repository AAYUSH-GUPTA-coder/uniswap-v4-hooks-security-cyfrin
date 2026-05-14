# 07 — Rounding & Precision

The category where a one-`wei` error in the right place compounds into a multi-million-dollar exploit.

## Background

Solidity has no native fixed-point or fractional numbers. Every fraction in DeFi is implemented as a numerator-divided-by-denominator and rounded at the moment of division. The direction of that rounding — toward zero (floor), toward infinity (ceil), toward nearest, toward the protocol's favor, toward the user's favor — is a security parameter, not a stylistic choice.

The Bunni V2 $8.4M exploit was a rounding bug. The attacker:

1. Drained almost all of the pool's reserves via a swap, putting the active balance into a regime where rounding errors became a significant fraction of the balance.
2. Repeatedly withdrew tiny share amounts, exploiting floor rounding on the idle-balance side of the share calculation to inflate share price while shrinking the active-balance liquidity disproportionately.
3. Swapped back, locking in atomic liquidity inflation, then drained the pool at the new inflated price.

This is the prototype for the rounding attack: find a place where rounding goes one way for one side of the calculation and the opposite way for the other, then push the system into a regime where the asymmetry is large.

## Heuristics — find the rounding sites

Every place `/` (or `mulDiv`, or shift-divide) appears in code that computes:

- Share price (shares ↔ underlying)
- LP token amounts on deposit / withdrawal
- Fee growth per unit liquidity
- Tick-based price conversions
- Slippage / bound calculations

…is a candidate site. List every one of them. For each:

- **Which direction does it round?** Read the code, don't assume.
- **Who benefits if it rounds down? Who benefits if it rounds up?** "The protocol" and "the user" are not interchangeable.
- **Can the user choose inputs that maximize the magnitude of the rounding error?** Tiny amounts, large amounts, amounts that produce specific remainders.
- **Can the user repeat the operation?** A 1-wei-per-call gain is unprofitable; ten million 1-wei-per-call gains in a single transaction is not.

## Heuristics — share-price inflation patterns

The ERC-4626-style "donate inflation" attack is the most-replicated version of this. v4 has its own variants:

- **First-depositor share-price inflation** — depositing 1 wei to mint 1 share, then "donating" a huge amount to make 1 share worth millions. Subsequent depositors round their share amount down to zero and lose their deposit. Mitigation: dead-shares minted at initialization to a burn address.
- **Idle-vs-active balance asymmetry** — hooks that split liquidity between an active (in-AMM) and idle (in-vault) pool can have separate rounding policies on each side. If the policies disagree, an atomic withdraw-then-deposit cycle can extract value.
- **Fee growth quantization** — fee per share is computed by accumulating into a shared counter. If fee accumulation rounds down per swap, many small swaps add nothing; one large swap adds the full amount. Attackers can grief or exploit depending on direction.

## Heuristics — Q-format arithmetic

v4 inherits v3's Q96-format square-root price math. Bugs in that math:

- **`sqrtPriceX96` boundaries** — operations near `TickMath.MIN_SQRT_RATIO` or `MAX_SQRT_RATIO` are prone to overflow / underflow if a custom hook mutates them.
- **Tick-to-price conversion errors** — small ticks have proportionally larger rounding error in the resulting price. Hooks that compare ticks for equality after a round-trip price conversion are buggy.
- **mulDiv direction selection** — `FullMath.mulDiv` rounds down, `mulDivRoundingUp` rounds up. Picking the wrong one in custom invariant math is a classic bug.

## Heuristics — protecting against the atomic-manipulation pattern

The Bunni-style attack requires the attacker to make many small operations in a single transaction to compound the rounding edge. Defenses include:

- **Minimum operation size** — refuse withdraws / deposits below a threshold where rounding error is negligible relative to the operation.
- **Slippage / share-price bounds** — require the share price implied by the operation stays within bounds.
- **Per-block rate limits** — a single-tx attack is harder if the operation can only be done once per block.
- **Symmetric rounding** — round both the active and idle sides in the user's disfavor, removing the asymmetry.

## What to grep for

```bash
# Division sites
grep -RE "/[^/]" --include="*.sol" . | grep -v "//"

# mulDiv / FullMath
grep -RE "mulDiv|FullMath|UnsafeMath" .

# Round direction
grep -RE "round(ing)?(Up|Down)" .

# Q96 / sqrt price math
grep -RE "sqrtPriceX96|Q96|TickMath" .

# Share math
grep -RE "(toShares|toAssets|previewDeposit|previewWithdraw|convertTo)" .
```

## Real-world examples

- **Bunni V2 exploit (~$8.4M)** — atomic share-price inflation via floor-rounding asymmetry on the idle/active balance split. Despite the auditor disclosure of the related Bunni V2 critical, this exploit still landed — a sobering data point on how subtle this category is.
- **Various ERC-4626 vault exploits** — the same family of bugs that earlier hit Cream, Hundred Finance, and others, now ported into AMM hooks.

## Primary sources

- Cyfrin / Bunni — Bunni V2 exploit post-mortem
- [Bunni V2 attack technical writeup](https://medium.com/) (search "Bunni v2 exploit post-mortem")
- [`FullMath.mulDiv`](https://github.com/Uniswap/v4-core/blob/main/src/libraries/FullMath.sol)
- Cyfrin — [Uniswap V4 Hooks Security Deep Dive](https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive)
