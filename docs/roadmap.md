# Roadmap

Live status of the custody layer: what is decided, what is being built, what is
queued. Design rationale lives in `docs/decisions/`, not here. Nothing here is
built yet (2026-10-24).

## Decided (ADR-001)

EVM: one Safe per chain (amended 2026-11-04: the vault Safe is dropped,
all on-chain capital trades), Roles on it and Delay under construction (b),
one trader role for the home Ledger and the app's KMS key, scoped to swaps
and CLMM range-order opens through **the guard** (decided 2026-11-04: a
stateless, unowned contract that checks the pool's time-weighted price and
depth and delivers everything to the Safe), range-order closes, bridges
and the Hyperliquid deposit; the CoW presets out; Hyperliquid: native
multi-sig with API wallets; the app's key in Cloud KMS; fork tests before
any policy goes live; agents never hold a key, never touch mainnet.

## Build order (EVM + Hyperliquid)

1. **The fork test harness**: a Safe + Roles + Delay deployment on a fork of
   each chain, a policy applied, the allowed list passing and the must-fail
   list reverting. The acceptance test everything else is measured against.
2. **The guard and the policy as code.** The guard contract (swap and
   range-order open; time-weighted reference price, deviation band, tick
   distance, depth floor per pool; stateless, unowned) with its fork tests,
   and the policies for Base, Arbitrum, Ethereum and Robinhood Chain:
   approvals pinned to the guard, a bridge or Permit2 and exact, the
   guard's parameters pinned per pool, the position manager's close
   functions with the proceeds pinned to the Safe, the native bridge, the
   Hyperliquid deposit transfer under its allowance, and the must-fail list
   (ADR-001); the apply and verify scripts that read the modifier's events
   (no subgraph on Robinhood Chain).
3. **Same-address Safes by script**, the Delay cooldown, a role call through
   Rabby with the Ledger — on testnets first.
4. **Hyperliquid on testnet**: the multi-sig conversion, agent approval
   through the multi-sig, trading through the agent, a withdrawal refusing
   one signature. The multisig UI: Hyperliquid's own, or the open-source
   signer.
5. **The KMS signer service** and the tracker's intent API, whose intents
   include opening, reloading and closing a range order (a reload cycle is
   one MultiSend role call); the executor's bad-trade residual bounded by
   the token list, the allowances (sized for the churn of reloads) and the
   engine's checks.
6. **The guard's audit**, its findings closed in the fork suite, before any
   real funds pass through it.
7. **Go live in stages**: the Safe with threshold 1 for a week, then the
   vault key and threshold 2; Hyperliquid last, since conversion is
   one-way. (The former step 7, a measurement of the v4-only share to
   decide how the role reaches v4, is retired: the guard reaches v4 by
   construction, listed pool by pool.)

## Queued, no decision made

### Solana expansion
Same requirements, different toolbox (ADR-001, last section). **Ruled out
by the owner (2026-10-24): an on-chain vault with capped hot wallets.** On
Solana there are only trading wallets, so the trading wallet itself must
carry the two-key withdrawal rule. What remains on the table, with nothing
decided and another solution possible when the work is taken up:

- A Turnkey-held trading key with a swap-only policy (program ids, mints,
  amounts), for the owner by passkey and for the app; enforced at signing
  time in a TEE, hosted by Turnkey; Jupiter's dynamic routing limits how
  rigid the policy can be, the CLMM programs (Orca, Raydium, Meteora) do
  not (ADR-001, Solana section).
- A Squads smart account with a swap-only policy program, if Squads' policy
  interface expresses it (their docs to read first; a testnet trial).
- A custom guarded-vault program (a PDA, a trader authority that may call
  Jupiter with the output delivered back, withdrawals through the Squads
  multisig): faithful to the EVM design, custom Rust with an audit.
- Hosting footprint for any Squads-based option: an RPC subscription and
  the executor; Squads' program and app are theirs; a custom policy program
  would be ours to deploy and audit.

### Ideas, not committed
- **Hyperliquid withdrawal whitelist**: ask Hyperliquid for a destination
  allowlist on `withdraw3`; it would give the home-key-only withdrawal the
  design cannot offer today.
- **UniswapX intents through the Safe** once Roles v3 ships its typed-message
  signing library, as a second swap path on Robinhood Chain that needs no
  router conditions.
- **Dollar-denominated depth floors** on the guard through an oracle
  (Chainlink's ETH/USD if deployed on Robinhood Chain, else the chain's
  deepest WETH/stable pool), if floors in quote units prove awkward to
  keep.
- **Monitoring**: an alert on every role execution and every Delay queue
  entry (Tenderly, OpenZeppelin Defender or our own watcher), so a coerced
  or compromised use is seen within minutes.
- **Scheduled policy drift checks**: the fork suite on a schedule against
  the live policy, since routers and protocols move.
- **A wrench-attack and key-loss drill**: a written runbook and a dry run
  for both (the cancel during the Delay window; recovering with the off-site
  key), repeated yearly.
- **The brokerage side**: IBKR's own user permissions (a trading-only
  secondary user; withdrawals only by the primary with its own second
  factor) are the analogue of the role there; worth configuring as part of
  the same posture.
- **Business**: the policy packs with their fork tests, the Robinhood Chain
  tooling, and the Hyperliquid multi-sig operations as an open-source
  library with a hosted layer; the full product (one policy for an
  individual across EVM, Robinhood Chain, Hyperliquid and a brokerage, with
  a bounded execution agent) later, if the own-use version proves itself.
