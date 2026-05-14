# 06 — Reentrancy

Classic vulnerability class, new attack surface. v4's hook + ERC-6909 + flash-accounting design adds three or four new ways to reenter that didn't exist in v3.

## Background

The v4 `PoolManager` has a global reentrancy guard on `unlock` — only one lock can be active at a time. This is the foundation everything else assumes. But hooks are not guarded by default; periphery contracts aren't guarded by default; rehypothecation paths and external token callbacks are not under v4's control.

Live exploit: the Bunni V2 critical (reported by Cyfrin) exploited exactly this seam — the global manager-level lock was unlockable from outside its intended single entrypoint, and an attacker-deployed hook could be used to legally hold the lock open while reentering the central hub.

## Heuristics — direct reentrancy

- **Does the hook make external calls?** Token transfers, oracle reads, callback hooks, vault deposit/withdraw — every one is a reentrancy entrypoint if the callee is not fully trusted.
- **Are external addresses caller-supplied?** If the hook ever calls a function on an address that came from a function argument or pool key, treat it as adversarial. ERC-20 callbacks (ERC-777, hooked tokens) extend this even when the address is statically known.
- **Are external calls made before state updates?** Checks-Effects-Interactions still applies. State that's read after the external call is potentially stale; state that's written based on a value read before the external call may be operating on outdated assumptions.
- **Is there a reentrancy guard on every public/external function that touches accounting?** A nonReentrant on `swap` is useless if the hook can reenter via `addLiquidity`, `donate`, or any other entrypoint that shares state.
- **Can the reentrancy guard be bypassed?** If the guard's locked-state can be set by anyone (e.g. by deploying a malicious hook that calls a privileged `unlockGuard` function), the guard is decoration.

## Heuristics — read-only reentrancy

Hooks often expose view functions used by other protocols (oracles, quoters, price feeds). During an active lock, internal state can be in an intermediate inconsistent state. If a view function reads that state and exposes it externally, integrators get bad data.

- **Are quoter / oracle / preview functions guarded against being called from inside a lock?** A check like `require(poolManager.isUnlocked() == false)` (or its inverse, depending on what's safe) prevents read-only reentrancy.
- **Are external integrators known to call these views without protection?** Even if your own code is safe, the integrators may not be — flag this as informational and mention it to them.

## Heuristics — ERC-6909 settlement reentrancy

The flash-accounting unlock cycle gives an attacker who controls *one* token in the swap path a way to re-enter via `transfer` / callback hooks. Specifically:

- `take(currency, recipient, amount)` calls the currency's `transfer` (for ERC-20s) or sends ETH (for native). Either can reenter.
- `settle()` for ERC-20s requires the caller to have already sent tokens; for ERC-777-style or hooked tokens, the `transfer` itself can call back.
- If the hook is in the middle of a multi-currency settlement and an external call re-enters, deltas can be observed in inconsistent states.

## Heuristics — rehypothecation / vault reentrancy

If the hook holds idle liquidity in external vaults (e.g. Aave, Compound, ERC-4626 vaults) to earn yield, every deposit / withdraw / harvest is an external call that can reenter:

- Are vault addresses caller-supplied (e.g. as part of the pool key or initialization data)? If so, an attacker can specify a malicious vault that reenters whenever it's deposited into or withdrawn from.
- Is pool state cached before the vault call and not re-read after? An attacker who reenters during the vault call can mutate state that the hook then ignores when it continues.

## What to grep for

```bash
# External calls inside hook functions
grep -RE "function (beforeSwap|afterSwap|beforeAddLiquidity|afterAddLiquidity|beforeRemoveLiquidity|afterRemoveLiquidity)" -A 60 .
# Then look for .call(, .transfer(, .send(, IERC20(...).transfer(...), and any IFooBar(addr).xxx()

# Reentrancy guards
grep -RE "nonReentrant|ReentrancyGuard|reentrancyLock" .

# Lock state checks
grep -RE "isUnlocked|onlyWhenUnlocked|extsload" .

# View functions that might be read-only-reentered
grep -RE "function (quote|preview|getPrice|getAmount|getReserves) " .

# Caller-supplied addresses used in calls
grep -RE "function .*\(address.*\) " .
# Then see if those addresses are .call()'d
```

## Real-world examples

- **Bunni V2 critical (reported by Cyfrin, full protocol TVL at risk)** — attacker-deployed hook could unlock the global hub-level reentrancy guard, then combined with cached pool state across unsafe external rehypothecation vault calls to recursively drain raw balances and vault reserves across all pools. Most thorough live demonstration of v4 reentrancy compounding.
- **C-03 (related Bunni-style finding)** — intermediate state between pre/post hooks for deposits could be reentered before the post-hook settlement.
- **L-13 (Bunni V2)** — read-only reentrancy through quoter functions affected integrating contracts even when the target itself was safe.

## Primary sources

- Cyfrin — [Bunni V2 critical disclosure write-up](https://www.cyfrin.io/blog/bunni-v2-disclosure)
- Trail of Bits — Bunni V2 audit report
- Cyfrin — [Uniswap V4 Hooks Security Deep Dive](https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive)
- Euler exploit post-mortems (different protocol, same lesson about compounding low-severity issues into critical reentrancy)
