# 02 — Hook Callbacks

Callback functions exposed to the `PoolManager`: who's allowed to call them, what they're allowed to return, and where in the lifecycle they execute.

## Background

The v4 `PoolManager` is the only address that should ever call most hook callbacks. Like every callback in DeFi — flash loan callbacks, ERC-721 `onERC721Received`, ERC-4626 hooks — calls originating from anyone else are an attacker holding the steering wheel.

Three things go wrong here regularly:

1. **Callbacks have no access control**, so anyone can call them and force the hook to act on attacker-supplied data.
2. **Callbacks return the wrong number of bytes**, so the `PoolManager` reverts when decoding the result.
3. **Logic lives in the wrong half of the lifecycle** (before vs. after), so it executes against state that hasn't been updated yet, or against state that's already been torn down.

## Heuristics — access control

- **Does every hook callback have `onlyPoolManager` (or equivalent)?** `unlockCallback`, `beforeSwap`, `afterSwap`, `beforeAddLiquidity`, `afterAddLiquidity`, `beforeRemoveLiquidity`, `afterRemoveLiquidity`, `beforeInitialize`, `afterInitialize`, `beforeDonate`, `afterDonate` — every one of these should reject calls from any address other than the manager.
- **Does the hook extend `SafeCallback` and override `_unlockCallback`?** The pattern in v4-periphery is to expose `unlockCallback` publicly with `onlyPoolManager`, then dispatch to a protected `_unlockCallback` that subclasses override. If the hook overrides `unlockCallback` directly and forgets the modifier, it's exploitable.
- **For periphery contracts (routers, lockers, accounting hubs)** that themselves expose callback-shaped functions, apply the same rule: only the `PoolManager` (or whichever specific upstream contract) should be authorized.

## Heuristics — return data encoding

The `Hooks` library enforces strict return-data lengths:

| Callback class | Required return length | Contents |
|---|---|---|
| Most callbacks | 32 bytes | `bytes4` selector (left-padded) |
| `afterSwap` / `afterAddLiquidity` / `afterRemoveLiquidity` (when `RETURNS_DELTA` flag is set) | 64 bytes | selector + `int256` delta |
| `beforeSwap` | 96 bytes | selector + `BeforeSwapDelta` + `uint24` LP fee override |

- **Does the hook return the right thing?** Returning `bytes("")` or omitting a `return` statement gives length 0 → revert. Returning `(0)` as `uint256` gives length 32 but might not contain the correct selector → revert.
- **For `beforeSwap`, is the LP fee override flag (`LPFeeLibrary.OVERRIDE_FEE_FLAG`) OR'd into the returned fee?** Without the flag, the returned fee is ignored and the pool's stored fee is used instead. See `04-dynamic-fees.md` for more.

## Heuristics — before vs. after placement

This is the most subtle category. Two questions to ask of every piece of hook logic:

- **Does this logic depend on state that the core `PoolManager` operation will mutate?** If yes, it matters which side of the operation it runs on.
- **Will the logic still produce the intended effect if state is read in its current placement?**

Common foot-guns:

- **Tick or position initialization logic in `afterAddLiquidity`** — by the time `afterAddLiquidity` runs, the position is already initialized. Conditional logic gated on "is this the first deposit" never fires. Put it in `beforeAddLiquidity`.
- **Cleanup of liquidity-position-keyed state in `beforeRemoveLiquidity`** — the position still exists at this point. If you wipe state here and then the manager removes the liquidity, you've prematurely cleared state for a position that's about to legitimately exist with reduced liquidity. Put cleanup in `afterRemoveLiquidity`.
- **Snapshotting in the wrong half** — TWAP / price snapshots taken in `beforeSwap` see the pre-swap price; in `afterSwap` the post-swap price. Make sure that's the intent.
- **Fee accrual in `beforeAddLiquidity` vs. `afterAddLiquidity`** — fees rebalance to the position on liquidity modification. Reading "fees owed" before vs. after the modification gives different answers.

## What to grep for

```bash
# Missing access control
grep -RE "function (unlockCallback|beforeSwap|afterSwap|beforeAddLiquidity|afterAddLiquidity|beforeRemoveLiquidity|afterRemoveLiquidity|beforeInitialize|afterInitialize|beforeDonate|afterDonate)" .
# Then visually verify each has onlyPoolManager / equivalent

# SafeCallback usage
grep -RE "SafeCallback|onlyPoolManager" .

# LP fee override flag
grep -RE "OVERRIDE_FEE_FLAG|LPFeeLibrary" .
```

## Real-world examples

- **Cork Protocol exploit (~$12M)** — `beforeSwap` was callable directly by the attacker with malicious hook data; the protocol assumed the manager was the only caller.
- **Doppler (Certora 2024, High)** — `beforeInitialize` callable by anyone, allowing pool-key hijacking.
- **Gamma UniV4 Limit Orders (Guardian Audits, Critical)** — `beforeSwap` and `afterSwap` callable by anyone, breaking limit-order semantics.
- **Various v4 example hooks** — `afterSwap` and other callbacks shipped without `onlyPoolManager`.

## Primary sources

- Uniswap Labs — [`v4-periphery` `BaseHook`](https://github.com/Uniswap/v4-periphery/blob/main/src/utils/BaseHook.sol) and [`SafeCallback`](https://github.com/Uniswap/v4-periphery/blob/main/src/base/SafeCallback.sol)
- [`Hooks.sol` library — return length checks](https://github.com/Uniswap/v4-core/blob/main/src/libraries/Hooks.sol)
- Cork Protocol post-mortem (search: "Cork Protocol exploit beforeSwap")
- Cyfrin — [Uniswap V4 Hooks Security Deep Dive](https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive)
