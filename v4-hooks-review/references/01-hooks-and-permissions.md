# 01 — Hooks & Permissions

How a hook advertises which callbacks it implements, and what goes wrong when the advertisement and the implementation disagree.

## Background

A v4 hook's permissions live in the low bits of its deployed address. The `PoolManager` checks each bit before calling the corresponding hook function:

```solidity
function hasPermission(IHooks self, uint160 flag) internal pure returns (bool) {
    return uint160(address(self)) & flag != 0;
}
```

Two consequences fall out of this:

- If a bit is **clear**, the `PoolManager` skips that hook function entirely. The contract behaves as if that function doesn't exist, even if it's defined.
- If a bit is **set** but the function isn't implemented (and there's no fallback), the call reverts.

Hook addresses are derived from `CREATE2(deployer, salt, init_code_hash)` — so the deployer mines a salt that produces an address with the desired bit pattern. Any mismatch between the mined permissions and the actually-implemented functions is a latent bug at best, a DoS at worst.

## Heuristics

- **Does the hook inherit `BaseHook` (v4-periphery) or the OpenZeppelin variant?** If yes, the constructor will run `validateHookPermissions` and reject deployment if the address bits disagree with the declared `Permissions` struct. If no, look for an equivalent check and assume there isn't one until you find it.
- **For every function the hook implements**, confirm the corresponding bit is set in the address. Missing bits mean the function is silently skipped — often catastrophic if it was supposed to take a fee, enforce a constraint, or update accounting.
- **For every permission bit set on the address**, confirm the function is implemented and won't revert under normal conditions.
- **Look for `RETURNS_DELTA` flags.** If the hook returns a non-zero delta from `afterSwap`, `afterAddLiquidity`, or `afterRemoveLiquidity` but doesn't have the corresponding `*_RETURNS_DELTA_FLAG` bit set, the delta is ignored and the protocol leaks value.
- **Can hook-mediated actions be performed directly on the `PoolManager`?** If business logic assumes liquidity additions or donations always flow through the hook, but the corresponding before/after hooks aren't implemented (and bits aren't set), users can bypass the hook by calling `PoolManager.modifyLiquidity` or `PoolManager.donate` directly.
- **Is the pool key validated?** A hook deployed without scoping to specific pools can be wired into an attacker-controlled pool with fake tokens. The attacker then exercises the hook's accounting against a pool they control. Check that hook functions validate `key.currency0`, `key.currency1`, and (where applicable) `key.tickSpacing` and `key.fee` against an allowlist.

## What to grep for

```bash
# Inheritance check
grep -RE "is\s+BaseHook|SafeCallback" .

# Permissions declaration
grep -RE "function getHookPermissions" .

# Pool key scoping
grep -RE "PoolKey|key\.currency0|key\.currency1" .

# Direct PoolManager entrypoints that might be unintentionally exposed
grep -RE "poolManager\.(modifyLiquidity|donate|swap)" .
```

## Real-world examples

- **Doppler (Certora 2024)** — Critical: malicious pool key with attacker-controlled currencies could drain the coordination contract because hook+currency addresses weren't validated.
- **Doppler (Certora 2024)** — High: `beforeInitialize` was callable by anyone and overwrote the stored pool key, hijacking subsequent operations.
- **Gamma UniV4 Limit Orders (Guardian Audits)** — Critical: `beforeSwap` and `afterSwap` had no access control, undermining the entire limit-order mechanism.
- **v4-stoploss example hook** — `afterSwap` callable by anyone with arbitrary parameters.

## Primary sources

- Uniswap Labs — [Known Effects of Hook Permissions](https://github.com/Uniswap/v4-core/blob/main/docs/security/HookPermissions.md)
- Composable Security — [Uniswap v4 Hook Security Checklist](https://composable-security.com/blog/secure-development-series-on-uniswap-v-4-hooks-1/)
- BlockSec — [Thorns in the Rose: Exploring Security Risks in Uniswap v4's Novel Hook Mechanism](https://blocksec.com/blog/thorns-in-the-rose-exploring-security-risks-in-uniswap-v4s-novel-hook-mechanism)
- Cyfrin — [Uniswap V4 Hooks Security Deep Dive](https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive)