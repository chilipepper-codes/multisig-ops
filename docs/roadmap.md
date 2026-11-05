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
unowned contract that checks the pool's time-weighted price and depth and
delivers everything to the Safe; v3-shaped pools only until it has a
reference price on v4), range-order closes, bridges and the Hyperliquid
deposit (recommended, to confirm on testnet: the Safe itself as the
Hyperliquid account, fed by CCTP with the recipient pinned, its Roles
scoping CoreWriter directly), and buys of unlisted tokens under a
per-period degen allowance
(the one priced exit, shaped by the wrench attack); the token set, the
per-pool parameters and the per-period notionals in a **registry** the
guard reads, where the home key alone can add a token behind a **listing
Delay** owned by the vault key with a three-day cooldown (amended
2026-11-05; the listing attack accepted), an **approver** module that
grants the guard its token approvals, a wide (30%) band over a short
window on sales, and a **pause flag** any role member can set; the CoW
presets out; Hyperliquid: native
multi-sig with API wallets; the app's key in Cloud KMS; fork tests before
any policy goes live; agents never hold a key, never touch mainnet.

## Build order (EVM + Hyperliquid)

1. **The fork test harness**: a Safe + Roles + Delay deployment on a fork of
   each chain, a policy applied, the allowed list passing and the must-fail
   list reverting. The acceptance test everything else is measured against.
2. **The guard, the registry, the approver and the policy as code.** The
   guard (swap and range-order open; time-weighted reference price,
   deviation band, tick distance, a depth floor relative to the trade and
   an absolute one through a pricing path over the chain's own pools), the
   registry (the listed set, per-pool parameters with defaults, the two
   per-period notionals) and the approver, with their fork tests; the
   listing Delay in front of the registry; and the policies for Base,
   Arbitrum, Ethereum and Robinhood Chain: the guard's functions (swap,
   open, bridge) for the trader role, the position manager's close
   functions with the proceeds pinned to the Safe, the registry's remove,
   the Hyperliquid deposit transfer under its allowance, and the must-fail
   list (ADR-001); the
   apply and verify scripts that read the modifier's events (no subgraph
   on Robinhood Chain).
3. **Same-address Safes by script** (the funded Safe at 3 of 3, the
   2-of-3 Safe on its Delay), the Delay cooldowns (the owner path's, owned
   by the funded Safe; the listing Delay's, owned by the vault key), a
   role call and a queued listing through Rabby with the Ledger — on
   testnets first.
4. **Hyperliquid on testnet**: first, the Safe itself as the account
   (ADR-001: a same-address Safe on HyperEVM whose Roles scope CoreWriter
   as `Calldata`; the whole Zodiac stack is deployed there, read from the
   chain; a CCTP deposit with the recipient pinned in the hook data,
   credited to the Safe; API wallets approved through CoreWriter by two
   keys; funds moved back to the HyperEVM Safe and bridged to Arbitrum
   over CCTP with the recipient pinned; the fire-and-forget actions
   confirmed through the precompiles), which would retire the hop, its
   allowance and the native multi-sig and give the home key the
   Hyperliquid withdrawal; otherwise the native multi-sig account fed by
   the same CCTP rail with the recipient pinned to it, and then the multi-sig conversion, agent approval
   through the multi-sig, trading through the agent, a withdrawal refusing
   one signature, and the multisig UI (Hyperliquid's own, or the
   open-source signer).
5. **The KMS signer service** and the tracker's intent API, whose intents
   include opening, reloading and closing a range order (a reload cycle is
   one MultiSend role call); the executor's bad-trade residual bounded by
   the registry's listed set and notionals and the engine's checks.
6. **The audit** of the guard, the registry and the approver, its findings
   closed in the fork suite, before any real funds pass through them.
7. **Monitoring** (owner, 2026-11-05: yes; its shape to be decided with
   the owner): at the least an alert on every listing queued on the
   Delay, every entry on the owner-path Delay and every role execution,
   since a coerced listing is delayed and visible rather than refused, and
   the veto is worth nothing unseen.
8. **Go live in stages**: the funded Safe at threshold 1 for a week, then
   the three owners at 3 of 3 with the 2-of-3 Safe on the Delay;
   Hyperliquid last, since conversion is one-way.

## Queued, no decision made

### v4 pools through the guard
The guard's price and depth checks rest on v3's `observe`; a v4 pool has
no oracle of its own, so v4 pools are outside the role (ADR-001). The open
question is the guard's reference price there: a v3 pool of the same pair
with enough depth, or a hook oracle listed pool by pool. Decided when the
engine's v3-versus-v4 measurement (portfolio-tools ADR-014, paper mode)
shows that v4-only pairs matter.

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
- **External feeds on the guard** (Chainlink, Pyth) as a sanity check on a
  tokenized stock's pricing hop, where a feed is deployed on the chain;
  never a dependency (ADR-001, the guard).
- **An off-site guardian** as the listing Delay's owner, a third party who
  can veto a queued listing alone while the owner is held; today the
  owner holds all three keys and the Delay's owner is the vault key
  (ADR-001).
- **Probation for new listings**: a token listed from the home key keeps
  counting against the degen allowance for its first days, closing the
  listing attack the owner accepted (ADR-001); a timestamp in the
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
