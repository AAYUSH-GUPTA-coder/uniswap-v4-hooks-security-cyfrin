# 08 — Tick Trouble

Concentrated liquidity is precise at the tick level. Off-by-one errors, off-by-tick-spacing errors, and edge cases at usable-tick boundaries cause severe issues — including the KyberSwap Elastic exploit ($56M).

## Background

Three numeric concepts to keep separate:

- **Tick** — an integer in `[MIN_TICK, MAX_TICK]` = `[-887272, 887272]` representing a price level.
- **Usable tick** — for a given `tickSpacing`, the range of ticks that liquidity positions can actually use: `[MIN_USABLE_TICK, MAX_USABLE_TICK]` (e.g. for `tickSpacing = 60`, `[-887220, 887220]`). Positions must align to `tickSpacing`.
- **Tick spacing** — the increment between usable ticks. Set per-pool in `PoolKey.tickSpacing`.

The gap between `MAX_USABLE_TICK` and `MAX_TICK` (52 ticks for `tickSpacing = 60`) is a narrow band where the math is still defined but no liquidity position can exist. The current price *can* enter that band during a swap, but no liquidity exists to support trades there, and special-case logic should prevent the price from getting stuck or producing weird results.

## Heuristics — misaligned ticks

- **For every tick used to construct a liquidity position** (lower or upper), confirm it's a multiple of the pool's `tickSpacing`. The `PoolManager` reverts with `TickMisaligned` if not.
- **If the hook accepts user-supplied ticks**, validate them up front rather than letting the manager revert downstream — better error messages, less wasted gas.
- **For hooks that compute ticks from prices or off-chain inputs**, the result must be snapped to tick-spacing. A `tick - (tick % tickSpacing)` formula is wrong for negative ticks (Solidity `%` returns the wrong sign). Use a proper rounding library.

## Heuristics — usable tick bounds

- **Is every tick bounded against `MIN_USABLE_TICK` / `MAX_USABLE_TICK`?** Computations involving tick arithmetic can land outside the usable range without producing an error — silently undefined behavior.
- **Are price-to-tick conversions clamped?** A square-root price at the boundary of `MIN_SQRT_RATIO` / `MAX_SQRT_RATIO` converts to a tick at the boundary; logic that subtracts a constant or shifts can step outside.
- **Are positions at the extreme ends initializable?** A position whose upper tick is exactly `MAX_USABLE_TICK` should work; a position with upper tick at `MAX_USABLE_TICK + 1` should not. Test both.

## Heuristics — zero-liquidity operations

A pool can legitimately end up with zero liquidity in the current tick — e.g. an unbalanced range that's been exhausted by swaps. What the hook does in that state matters:

- **Can `sqrtPriceX96` be pushed to an extreme value when liquidity is zero?** With no liquidity to anchor the price, swaps (or hook-triggered price updates) can move it freely. This breaks any subsequent assumption about price-as-oracle.
- **Does swapping against zero liquidity DoS the hook?** Some custom math returns NaN-equivalents (division by zero) when liquidity is zero. The pool should either revert cleanly or no-op.
- **Are incentives / fee accruals gated on liquidity > 0?** Distributing rewards proportionally to zero liquidity is undefined; accruing them blindly may strand tokens.
- **Is single-sided liquidity addition handled at the boundaries?** Adding only token0 at `tick > current`, or only token1 at `tick < current`, is a legitimate pattern that some custom AMMs break.

## Heuristics — tick crossings

The most error-prone part of v3/v4 math. The active liquidity is the running sum of all position liquidity-delta contributions for the current tick. When the price crosses a tick:

- The tick's stored `liquidityNet` is added to (one direction) or subtracted from (the other direction) the active liquidity.
- The tick's `feeGrowthOutside` values flip relative to the current `feeGrowthGlobal`.

Bugs here have produced two of the largest AMM exploits in history (KyberSwap Elastic $56M, Raydium $4.4M). Things to check:

- **Are tick crossings symmetric across swap directions?** The "going up" and "going down" paths must give the same active liquidity at the same tick.
- **Is a tick at exactly the upper boundary of an active range counted correctly?** This is the specific class of bug in the [example finding referenced in the Cyfrin deep dive] — a zero-for-one swap whose starting price was exactly at a tick that was the upper bound of a position would skip that position's contribution to active liquidity.
- **Does adding / removing liquidity correctly update tick state without double-counting?** Each position's contribution to a tick's `liquidityNet` should be added once on creation, removed once on removal — never double-applied.
- **Is `feeGrowthOutside` updated atomically with the crossing?** If the order is reversed, fee accounting drifts.

## What to grep for

```bash
# Tick alignment
grep -RE "tickSpacing|TickMisaligned" .

# Usable bounds
grep -RE "(MIN|MAX)_TICK|(MIN|MAX)_USABLE_TICK|(MIN|MAX)_SQRT_RATIO" .

# Active liquidity / tick crossings
grep -RE "liquidityNet|liquidityGross|crossTick|feeGrowthOutside" .

# Zero-liquidity guards
grep -RE "liquidity\s*(==|>|<)\s*0" .
```

## Real-world examples

- **KyberSwap Elastic (~$56M)** — incorrect tick crossing math allowed liquidity to be double-counted on certain swap paths, then drained.
- **Raydium (~$4.4M)** — similar class of tick-state inconsistency on swap.
- **KyberSwap Elastic critical vulnerability disclosure** — pre-exploit responsible disclosure of the same class of bug.
- **Various v4 hook audits** — boundary-tick edge cases in zero-for-one swaps, where the current tick equals the upper tick of an active position; the position's liquidity is dropped from the active sum and fee growth is over-credited to the remaining positions.
- **Various v4 hook audits** — DoS when single-sided liquidity is added at `MIN_USABLE_TICK` or `MAX_USABLE_TICK` boundaries.

## Primary sources

- Uniswap Labs — [v3 whitepaper, especially section 6 (tick math)](https://uniswap.org/whitepaper-v3.pdf)
- [`TickMath` library](https://github.com/Uniswap/v4-core/blob/main/src/libraries/TickMath.sol)
- KyberSwap Elastic post-mortem
- Raydium post-mortem
- Cyfrin — [Uniswap V4 Hooks Security Deep Dive](https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive)
