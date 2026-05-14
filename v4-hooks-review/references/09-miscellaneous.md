# 09 — Miscellaneous

The categories that don't slot neatly into the other eight, but bite often enough to warrant their own pass.

## Native ETH handling

v4 represents native ETH as `Currency.wrap(address(0))`. This is a UX improvement over v3 (no mandatory `WETH9` wrapping) and a footgun.

### Heuristics

- **Is the hook (and its periphery) able to receive ETH?** A `receive()` function is required if the contract is expected to accept ETH transfers. Without it, refunds, dust returns, and router-mediated value flows revert.
- **Where ETH is transferred to the hook or its periphery, is `msg.value` validated?** Sending more ETH than the swap or deposit requires can leave the excess stuck. Sending less can break the custom delta accounting.
- **Native transfers are external calls** — they're a reentrancy entrypoint. Anything that calls `(bool ok,) = recipient.call{value: amount}("")` is a potential reentry site. See `06-reentrancy.md`.
- **Currencies are ordered by address; `address(0)` sorts first.** Native ETH is always `currency0`, never `currency1`. No need to validate that `currency1 != address(0)` — but every other native-token assumption (input parsing, recipient handling) should be verified.

### Real-world

- **Various** — periphery contracts that don't implement `receive()` and silently DoS native-token paths via router refunds.
- **L-22 (general v4 hook audit pattern)** — `msg.value` not validated; excess ETH locked in the hook.
- **M-19** — `msg.value` mismatch breaks custom swap delta accounting.

## Just-in-time (JIT) liquidity

JIT is when an LP adds liquidity within the same transaction as a swap, captures fees from that swap, and removes the liquidity again — usually atomically, often through a bot watching the mempool.

### Heuristics

- **Is JIT desirable here?** Some protocols (Euler v2's EulerSwap, intent-based AMMs) deliberately use JIT for capital efficiency. Others (LP rewards programs, fee-share schemes) get drained by it.
- **If JIT is undesirable, how is it disincentivized?** Common patterns: minimum holding time before fee accrual; fee penalty on removal within N blocks of addition; fee accrual that's linear in time-held rather than just present-at-swap.
- **Can JIT be used to manipulate any state the hook reads?** Rebalance triggers, oracle snapshots, fee-tier votes, reward weights — anything weighted by current liquidity composition is a JIT target.
- **Are withdrawal-queue interactions JIT-safe?** A withdrawal queued in block N, processed in block N+K, can be sandwiched by JIT in block N+K.

### Real-world

- **TOB-BUNNI-9 (Trail of Bits)** — withdrawal queue logic combined with JIT liquidity provision let an attacker manipulate the rebalance order computation, degrading pool state.
- **General front-running of custom liquidity / rewards mechanisms** — multiple findings in audits of incentive-bearing hooks.

## Router parameter mistakes

The `V4Router` is peripheral, so bugs here don't compromise the core protocol, but they can lose user funds. The same patterns appear inside hook logic that does its own routing.

### Heuristics

- **Recipient addresses** — are they `msg.sender`, the hook itself, the manager, or a caller-supplied address? Be explicit. "Default to `msg.sender`" is wrong when the caller is a contract that wants funds delivered elsewhere.
- **Token / amount pairing** — `amount0` and `token0` belong together; `amount1` and `token1` belong together. Mixing them is a copy-paste bug that's easy to make and hard to notice.
- **Swap direction (`zeroForOne`)** is the sign convention for the entire call. Flipping it inverts the meaning of `amountSpecified`.
- **`amountSpecified` sign** — positive for exact-output, negative for exact-input. The wrong sign produces a swap that succeeds but in the wrong mode.

### Real-world

- **C-03 (v4 router audit)** — wrong recipient address on settlement.
- **C-04 / H-07** — `amount0` used in place of `amount1` and vice versa.

## Custom incentives

The catch-all for fee distributions, reward emissions, vote-escrow systems, gauge weights, and anything else that gets bolted onto a hook to do off-AMM accounting.

### Heuristics

- **If you've reimplemented v3 swap math, tick crossing, or fee growth logic** — you've inherited every v3 bug class plus the v4-specific ones. Treat reimplementations as suspect by default.
- **Test the flow end-to-end** — fees / rewards typically pass through 3–5 contracts between accrual and withdrawal. Each hop is a potential value leak.
- **Are tokens stuck recoverable?** A `sweepDust` or `recoverERC20` function with appropriate access control is cheap insurance against misaccounted residues.
- **Are reward distributions front-runnable?** A common pattern: rewards accrue over time, the user claims, the claim resets the accrual. Anything observable on-chain that triggers a claim makes the claim front-runnable.

### Real-world

This is the most common category by raw count of findings. The Cyfrin deep-dive lists 27+ separate examples. Patterns repeat:

- LP-fee-per-share computations that floor-round at low fee tiers, stranding small accruals.
- Reward emission rates that don't account for liquidity changing mid-period.
- Vote-escrow / locked-position implementations that don't handle position transfer or burn correctly.

## What to grep for

```bash
# Native ETH handling
grep -RE "receive\(\)|fallback\(\)" .
grep -RE "msg\.value" .
grep -RE "CurrencyLibrary\.NATIVE|Currency\.wrap\s*\(\s*address\s*\(\s*0" .

# Routing
grep -RE "amount0|amount1|token0|token1|currency0|currency1" .
grep -RE "zeroForOne" .

# Recoverability
grep -RE "rescue|sweep|recover|salvage" .
```

## Primary sources

- Uniswap Labs — [`Currency` type and native ETH handling](https://github.com/Uniswap/v4-core/blob/main/src/types/Currency.sol)
- [`V4Router`](https://github.com/Uniswap/v4-periphery/blob/main/src/V4Router.sol)
- Trail of Bits — Bunni V2 audit report
- Cyfrin — [Uniswap V4 Hooks Security Deep Dive](https://www.cyfrin.io/blog/uniswap-v4-hooks-security-deep-dive)
