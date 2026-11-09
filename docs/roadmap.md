# Roadmap

Live status of the custody layer: what is decided, what is being built, what is
queued. Design rationale lives in `docs/decisions/`, not here. Nothing here is
built yet (2026-10-24).

## Decided (ADR-001)

The user stories in [`docs/stories.md`](stories.md) are the spec
(owner, 2026-11-05). EVM: one funded Safe per chain (amended 2026-11-04:
the vault Safe is dropped, all on-chain capital trades), three owner keys
at **3 of 3 for the direct path** and a fundless 2-of-3 Safe queuing
every other owner action through the Delay (decided 2026-11-05), Roles on
it, one trader role for
the home Ledger and the app's KMS key, scoped to swaps and CLMM
range-order opens through **the guard** (decided 2026-11-04: an admin-less,
unowned contract that checks a trade's outcome against the caller's
minimum, shown to the owner before signing, and, where the token has a
backstop pool with a free oracle, against its time-weighted price and
depth, and delivers everything to the Safe, through any whitelisted
router, so every AMM is reachable, Uniswap v2, v3 and v4 and Aerodrome
among them; amended 2026-11-09),
range-order closes, bridges, and on HyperEVM
the CoreWriter actions of the Safe's own Hyperliquid account (decided
2026-11-05: the Safe is the account, fed by CCTP with the recipient
pinned, validated on testnet before live funds; the native multi-sig is
out), and the degen allowance, one rolling cooldown window, spent on
non-whitelisted buys or withdrawn to any address with the home key (the
one priced exit, handed over under duress by design; owner, 2026-11-09); the token set, the
per-pool parameters and the per-period notionals in a **registry** the
guard reads, where the home key alone can add a token behind a **whitelisting
Delay** owned by the vault key with a three-day cooldown (amended
2026-11-05; the whitelisting attack accepted), an **approver** module that
grants the guard its token approvals, deposits to the owner's Kraken
account at an address the registry pins per chain (owner, 2026-11-06:
Kraken's global settings lock makes it another vault of the owner's), a
wide (30%) band over a short
window on sales, and a **pause flag** any role member can set; the CoW
presets out; Hyperliquid: the same Safe on HyperEVM, API wallets for
trading; the app's key in Cloud KMS; fork tests before
any policy goes live; agents never hold a key, never touch mainnet.

## Build order (EVM + Hyperliquid)

1. **The fork test harness**: a Safe + Roles + Delay deployment on a fork of
   each chain, a policy applied, the allowed list passing and the must-fail
   list reverting. The acceptance test everything else is measured against.
2. **The guard, the registry, the approver and the policy as code.** The
   guard (swap by outcome check against the caller's minimum through any
   whitelisted router, range-order open and v4 close, bridge, send to
   exchange, withdraw allowance; the backstop's read adapters for the
   free-oracle kinds only: Uniswap v3 and forks, Aerodrome v2-style,
   Algebra; deviation band, tick distance, a depth floor relative to the
   trade and an absolute one through a pricing path), the
   registry (the whitelisted set, per-pool parameters with defaults, the two
   per-period notionals) and the approver, with their fork tests; the
   whitelisting Delay in front of the registry; and the policies for Base,
   Arbitrum, Ethereum and Robinhood Chain: the guard's functions (swap,
   open, bridge) for the trader role, the position manager's close
   functions with the proceeds pinned to the Safe, the registry's remove,
   the CoreWriter policy on HyperEVM, the Kraken deposit address pinned
   per chain, and the must-fail list (ADR-001);
   the
   apply and verify scripts that read the modifier's events (no subgraph
   on Robinhood Chain).
3. **Same-address Safes by script** (the funded Safe at 3 of 3, the
   2-of-3 Safe on its Delay), the Delay cooldowns (the owner path's, owned
   by the funded Safe; the whitelisting Delay's, owned by the vault key), a
   role call and a queued whitelisting through Rabby with the Ledger — on
   testnets first.
4. **Hyperliquid on testnet, the Safe as the account** (decided
   2026-11-05; ADR-001's Verify step 1): the same-address Safe, Roles and
   Delay on HyperEVM testnet; the CoreWriter policy scoped as `Calldata`;
   a CCTP deposit with the recipient pinned in the hook data, credited to
   the Safe; an API wallet approved through CoreWriter by two keys, then
   trading through it; a class transfer and a send asset as role calls
   back to the HyperEVM Safe and a pinned CCTP burn to Arbitrum; every
   action confirmed through the precompiles, the node tracker's
   silent-failure report reproduced or ruled out; the refusals; a soak of
   weeks. Nothing on Hyperliquid goes live before this passes.
5. **The KMS signer service** and the tracker's intent API, whose intents
   include opening, reloading and closing a range order (a reload cycle is
   one MultiSend role call); the executor's bad-trade residual bounded by
   the registry's whitelisted set and notionals and the engine's checks.
6. **The audit** of the guard, the registry and the approver, its findings
   closed in the fork suite, before any real funds pass through them.
7. **Monitoring** (owner, 2026-11-05: yes; its shape to be decided with
   the owner): at the least an alert on every whitelisting queued on the
   Delay, every entry on the owner-path Delay and every role execution,
   since a coerced whitelisting is delayed and visible rather than refused, and
   the veto is worth nothing unseen.
8. **Go live in stages**: the funded Safe at threshold 1 for a week, then
   the three owners at 3 of 3 with the 2-of-3 Safe on the Delay;
   Hyperliquid last, after step 4's soak, with the balance at trading
   size.

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
  whitelist on `withdraw3`; it would give the home-key-only withdrawal the
  design cannot offer today.
- **UniswapX intents through the Safe** once Roles v3 ships its typed-message
  signing library, as a second swap path on Robinhood Chain that needs no
  router conditions.
- **External feeds on the guard** (Chainlink, Pyth) as a sanity check on a
  tokenized stock's pricing hop, where a feed is deployed on the chain;
  never a dependency (ADR-001, the guard).
- **An off-site guardian** as the whitelisting Delay's owner, a third party who
  can veto a queued whitelisting alone while the owner is held; today the
  owner holds all three keys and the Delay's owner is the vault key
  (ADR-001).
- **Keepers for reference prices without a free oracle** (owner,
  2026-11-09: only if practice shows tokens of interest with no
  free-oracle pool): Uniswap v2 pairs through their cumulative
  accumulators against registry snapshots, Uniswap v4 pools through
  sampled observations by designated keepers (ADR-001, the guard).
- **Probation for new whitelistings**: a token whitelisted from the home key keeps
  counting against the degen allowance for its first days, closing the
  whitelisting attack the owner accepted (ADR-001); a timestamp in the
  registry if ever wanted.
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
