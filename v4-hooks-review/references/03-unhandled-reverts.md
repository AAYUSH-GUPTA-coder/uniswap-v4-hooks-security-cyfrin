# 03 — Unhandled Reverts

When a hook reverts in the wrong place, users lose access to their own liquidity. This is the category most likely to convert a "minor bug" into a permanent loss.

## Background

Flash accounting binds the v4 lifecycle into a single `unlock` → ... → settle-all-deltas → return loop. The invariant is: at the end of the callback, every `currencyDelta` for every currency the lock touched must be zero. Hooks that revert in the middle of this loop — for any reason — break the invariant and abort the entire transaction.

A revert during a swap or liquidity add is annoying. A revert during a liquidity *remove* can lock LP funds permanently if the only path out goes through the failing code.

## Heuristics — sync before unlock

- `PoolManager.sync(currency)` requires the manager to be unlocked. Calling it from a hook callback that *isn't* inside an active unlock cycle reverts.
- Pattern: if your hook needs to settle a delta, it must be inside an `unlockCallback`, and the `sync` (if you're using the donate-style accounting path) must follow the unlock.
- Recent `PoolManager` versions have relaxed this — `sync` can now be called without an active unlock. Confirm against the deployed version before flagging.

## Heuristics — uncleared / unsynchronised dust

- Token balances accumulated as side effects (rounding leftover, donation residues) can create non-zero deltas that the hook doesn't actively settle. If those deltas survive into the unlock-return check, the whole transaction reverts.
- Two ways out:
  - Call `clear(currency, amount)` to explicitly write off small balances the protocol is willing to forfeit.
  - Settle them via `take` / `settle` to either pay them out or absorb them into the pool.
- Look at every code path that touches `currencyDelta` and ask: "if this leaves dust, where does it go?"

## Heuristics — reverts on liquidity modification

The highest-severity sub-category. Walk both `beforeRemoveLiquidity` and `afterRemoveLiquidity` looking for any line that can revert:

- External calls (oracles, vaults, other protocols) that can be paused / made to revert by an adversary
- Math that can underflow when a position is removed in full
- Access-control checks that fail under conditions an LP can be pushed into (e.g. expired permissions)
- Loops over user-controlled data that can be bloated to hit the block gas limit
- Storage writes that depend on external state (paused tokens, frozen accounts)

If any revert source exists with no escape hatch, LP funds are locked.

`beforeAddLiquidity` / `afterAddLiquidity` reverts are less severe (LP keeps their tokens, just can't deposit) but still represent denied core functionality.

Also worth flagging: in v4, fee growth is settled as a delta on *every* liquidity modification (unlike v3 where fees were collected separately). A hook that reverts on liquidity modification therefore also blocks fee collection — even if the LP doesn't care about adding or removing liquidity, they can't pull their accrued fees out.

## Heuristics — reverts in peripheral logic

Custom incentives, rebalancing, top-of-block fees, MEV protection, and any other "bolted-on" logic that runs from inside a hook callback is a source of reverts that can DoS the whole pool.

Specific cases to look for:

- **Per-block state machines** that assume only one update per block, then break when called twice
- **Multi-pool hooks** that don't key state by `PoolId`, so operations on pool A clobber state used by pool B
- **Permissionless reward distribution** loops that can be filled with attacker-supplied entries to blow gas
- **Math (taxes, prices, rebases)** with edge cases at extreme inputs that revert via overflow or division by zero
- **Cross-chain quirks** — gas limits, opcode availability, block timing differing between L1 and L2s

## What to grep for

```bash
# sync calls
grep -RE "poolManager\.sync\(" .

# clear/take/settle — make sure deltas are reconciled
grep -RE "\.(clear|take|settle|sync)\s*\(" .

# Reverts in liquidity hooks
grep -RE "function (beforeAddLiquidity|afterAddLiquidity|beforeRemoveLiquidity|afterRemoveLiquidity)" -A 50 .
# Then visually scan for require/revert/assert and external calls

# Loops over user data
grep -RE "for\s*\(.*\.length" .
```

## Real-world examples

- **Gamma UniV4 Limit Orders (Guardian Audits, High)** — dust balances accumulated in transient storage; the hook reverted on settlement because deltas couldn't be cleared.
- **Gamma UniV4 Limit Orders (Guardian Audits, Low)** — donations made without a prior `sync` reverted on settle.
- **Bunni V2 (Pashov Audit Group, High)** — multi-pool rebalance logic didn't key by pool ID; rebalance succeeded on the first pool of the block and reverted on every subsequent one.
- **Various** — top-of-block tax logic that was supposed to apply only to the first swap of a block but in fact applied (and reverted) on every swap after the first.

## Primary sources

- Uniswap Labs — [Flash accounting and the unlock pattern](https://docs.uniswap.org/contracts/v4/concepts/lock-mechanism)
- Guardian Audits — Gamma UniV4 Limit Order review
- Pashov Audit Group — Bunni V2 review
- Cyfrin — [Uniswap V4 Hooks Security Deep Dive](https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive)
