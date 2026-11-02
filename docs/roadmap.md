# Roadmap

Live status of the custody layer: what is decided, what is being built, what is
queued. Design rationale lives in `docs/decisions/`, not here. Nothing here is
built yet (2026-10-24).

## Decided (ADR-001)

EVM: two Safes per chain, Roles + Delay on the trading one, one trader role
for the home Ledger and the app's KMS key, scoped to swaps, bridges, the
Hyperliquid deposit and (amended 2026-11-02) CLMM range orders, the
execution engine's resting limit orders as single-sided positions owned by
the Safe; Hyperliquid: native multi-sig with API wallets; the app's key in
Cloud KMS; fork tests before any policy goes live; agents never hold a key,
never touch mainnet.

## Build order (EVM + Hyperliquid)

1. **The fork test harness**: a Safe + Roles + Delay deployment on a fork of
   each chain, a policy applied, the allowed list passing and the must-fail
   list reverting. The acceptance test everything else is measured against.
2. **Policy as code** for Base, Arbitrum, Ethereum (CoW presets, for the
   owner's own swaps through Pilot) and Robinhood Chain (SwapRouter02 with
   the recipient pinned, the native bridge, the Hyperliquid deposit transfer
   under its allowance), plus, on every chain, the **position manager**
   target for the engine's CLMM range orders (ADR-001: mint and collect
   pinned to the Safe, the NFT's transfers and approvals, `multicall` and
   the sweeps on the must-fail list; v3-shaped managers only, v4 waits on
   step 7); the apply and verify scripts that read the modifier's events
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
6. **Go live in stages**: the trading Safe with threshold 1 for a week,
   then the vault key and threshold 2; Hyperliquid last, since conversion is
   one-way.
7. **Measure the v4-only share** of the Robinhood Chain flow for a few
   weeks, swaps and the pairs the engine would rest range orders in, then
   decide between Universal Router conditions and the guarded swapper
   (ADR-001); a guarded contract, if built, gets the range-order variant
   that checks the range against the pool price.

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
- **The guarded swapper** as a product-grade component (audited, any router,
  any chain) if the measurement in step 7 says v4-only pools matter.
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
