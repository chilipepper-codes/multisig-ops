# ADR-001: Wrench-resistant self-custody for a trading book — Safe + Roles on EVM, native multi-sig on Hyperliquid, a hardware key for the app

**Status:** accepted as the design (owner, 2026-10-24); nothing built yet. Solana is
**proposed**, not decided (last section). Every "verify" line below is a
testnet or fork step that precedes real funds.

## Context

The owner trades on-chain (EVM chains including Robinhood Chain, which is a
hard requirement, and Solana), on Hyperliquid, and through a brokerage, from a
Ledger used through Rabby. The threat this design answers is the wrench attack:
someone at the owner's home forces the owner to sign. Today one key at home
moves everything.

The requirements, as stated:

1. Moving funds off the wallet to any other address needs **two keys**: one at
   home, one in a bank vault.
2. Trading on DEXs, bridging, depositing to Hyperliquid and withdrawing from
   Hyperliquid back to the wallet need **the home key only**.
3. The app (the portfolio tracker's execution engine, running on GCP) executes
   stops and strategies **without the owner signing**, with trading permissions
   and **no ability to withdraw**.
4. Robinhood Chain is non-negotiable and the exposure there cannot be capped.
5. First and foremost for the owner's own use; possibly a business later.

Facts the design rests on (checked 2026-10-23/24, sources in the owner's
session notes):

- Safe is live on Robinhood Chain; the Zodiac Roles 2.1.1 and Delay 1.1.1
  mastercopies are deployed there at their canonical addresses (the owner
  confirmed non-empty bytecode). The Zodiac app, its subgraph and the Pilot
  extension do not list the chain yet.
- Zodiac had two bugs this year: the June 2026 ERC-1271 authorization bypass
  (Roles 2.1.0 / Delay 1.1.0; exposed only setups where a *Safe* was a role
  member or an enabled module; patched 2.1.1 / 1.1.1 on June 19) and a
  MultiSend-unwrapper bypass (needed an existing role member; fixed, the app
  migrates). Roles v3 is audited (Oct 2026) but unreleased.
- Hyperliquid has two permission levels only: the account key (everything,
  including `withdraw3` to any destination) and API wallets (trade only, no
  withdrawals or transfers). Its native multi-sig applies the threshold to
  **every** action of the account, trading included. Deposits are credited to
  the sending address, which must be able to sign Hyperliquid's EIP-712
  actions (ecrecover; a Safe cannot).
- Uniswap v2, v3, v4 and UniswapX are live on Robinhood Chain; the v3
  SwapRouter02 takes the recipient as a plain struct field; the Universal
  Router (v4) takes nested command bytes. The canonical Universal Router
  address there has changed three times this year.
- Cloud KMS signs secp256k1 (EVM, Hyperliquid) and Ed25519 (Solana) in an
  HSM; the private key never leaves it.

## Decision

### EVM chains (Ethereum, Arbitrum, Base, Robinhood Chain)

- **Two Safes per chain, at the same address on every chain.** A *vault Safe*
  with **no modules**, threshold 2 of 3 owners: the home Ledger, the vault key,
  and a third key kept **off-site** (never a second home device: home alone
  must not reach the threshold). It holds everything beyond working capital.
  A *trading Safe*, same owners and threshold, with the Zodiac **Roles**
  modifier and the **Delay** modifier enabled. A module bug reaches the
  trading Safe only.
- **One role, "trader", two members:** the home Ledger's EOA and the app's
  KMS key. What the role allows, and nothing else: ERC-20 approvals to listed
  routers, bridges and Permit2; swaps whose recipient is pinned to the Safe
  (`c.avatar`); bridges whose destination is the Safe's address on the other
  chain; a token allowlist; per-period allowances; one transfer target outside
  the Safe, the Hyperliquid deposit account below, under an allowance. A
  transfer to any other address is not a role call: it needs two owners.
- **A cooldown on withdrawals — open choice, corrected 2026-10-24.** The
  Delay modifier delays only what reaches the Safe *through it as a module*;
  owners who sign directly are never delayed (Zodiac's own docs). So "a
  delay on the two-key path" is one of two constructions, not a setting:
  - **(a) No delay.** The two owners sign directly; the wrench defence is the
    vault key alone. Simplest, nothing can brick the Safe.
  - **(b) The owner path routed through Delay.** The vault Safe (2 of 3) is
    the module enabled on the trading Safe's Delay; the trading Safe's own
    owner set is made unable to sign (its single owner the Delay contract,
    which cannot produce a signature), so every non-role action — a
    withdrawal, a policy change — is queued by the vault Safe, waits the
    cooldown (a day or two), and can be cancelled during it by the Delay's
    owner (the home key, or the off-site third party). A coerced two-key
    withdrawal then needs the attacker to hold the owner for the whole
    delay. Cost: the trading Safe has no direct owner path at all, so a
    Delay bug or a stuck queue needs the queue's own skip (`setTxNonce`) or
    the expiration to clear; rehearse both on a fork before choosing it.
  Role calls bypass the delay in both constructions by design.
- **Swaps, v3 first.** On chains with CoW Swap, the CoW presets (Safe signs the
  order, Roles scopes the message). On Robinhood Chain, Uniswap v3's
  SwapRouter02 with the recipient pinned: standard conditions, **no custom
  contract**. Pools that exist only on v4 are not reachable through the role
  until either (a) conditions on the Universal Router's command bytes are
  written and fork-tested, or (b) a small guarded-swap contract (pulls the
  input, calls an allowlisted router, delivers the whole output to the Safe,
  reverts otherwise) is written and audited. Decide (a) or (b) only after
  measuring, over a few weeks, how much of the Robinhood Chain flow needs
  v4-only pools. Until then, v4-only trades are not role trades.
- **Tooling on Robinhood Chain is ours.** Policy as code with the Roles SDK's
  condition utilities, applied through the Safe by script and read back from
  the modifier's events over RPC (no subgraph there); the app builds
  `execTransactionWithRole` calls for the role member to sign with the Ledger
  through Rabby. On the other chains the Zodiac app and Pilot are used as is.
  Ask Gnosis Guild to add the chain; do not depend on it.
- **Same-address Safes by script** (factory, singleton, initializer, salt
  replayed per chain), until Safe{Wallet}'s multi-chain deployment covers
  Robinhood Chain.

### Hyperliquid

- **The account is a fresh EOA**, converted to a **native multi-sig user**
  with the home and vault keys as authorized users, threshold 2. Everything
  the *account* does (withdraw, approve an agent, transfer) needs both keys.
- **Trading is by API wallets**, approved once by the multi-sig: the owner's
  browser agent and the app's KMS key. API wallets cannot withdraw by
  Hyperliquid's own rules.
- **Deposits** go from the trading Safe to that account's Arbitrum address
  (the one transfer-out the role allows, capped by an allowance), then from
  that address into the bridge. The account's Arbitrum key is kept at home.
- **Withdrawing from Hyperliquid is a two-key action.** This is the one
  requirement the design does not meet: whoever can withdraw on Hyperliquid
  can choose any destination, and the multi-sig threshold is the only lever.
  Accepted: the Hyperliquid balance is kept at trading size, and withdrawals
  are planned. The alternative, the account on the home key, leaves the whole
  balance wrench-exposed.

### The app's key

- A **Cloud KMS HSM key** (secp256k1 for EVM and Hyperliquid), usable by one
  service account, every signature audit-logged. It is a role member on the
  trading Safe and an API wallet on Hyperliquid. By construction a full
  compromise of the app can only trade inside the allowlists; its residual,
  bad trades, is bounded by the token allowlist, the allowances and the
  engine's own risk checks. Never a seed in the app.

### Boundaries and rules

- **This repository** holds the custody layer: the policies as code, the
  apply and verify scripts, the fork test suite, the Safe deployment scripts,
  the Hyperliquid multi-sig tooling, the KMS signer service. The portfolio
  tracker keeps the execution engine and talks to this layer through a
  narrow intent API; it records the boundary in its own ADR.
- **Agents never hold any of these keys and never run against mainnet.**
  Fork and testnet only, as the tracker's IBKR rule already says for
  execution. Nothing in this repository places a real order.
- **The fork test suite comes before any policy goes live**: every allowed
  call passes, a list of must-fail calls (a transfer out, a swap to another
  recipient, an unlisted token, a router outside the list) reverts, on every
  chain, on every policy change and on a schedule. Policy is code that fails
  silently; this is what makes it safe to change.
- Build on Roles 2.1.1 and Delay 1.1.1 with the current unwrapper; migrate to
  v3 only after it has been in production elsewhere. Subscribe to Zodiac's
  security page.

### Verify before real funds

1. Hyperliquid testnet: the multi-sig conversion (one report says converting
   back failed, treat it as one-way); approving an agent through a
   multi-sig-wrapped action; trading through that agent from the app and from
   the web UI; a withdrawal refusing one signature.
2. A fork of Robinhood Chain: the SwapRouter02 permission (recipient pinned,
   token list, approvals), the bridge permission, the Hyperliquid deposit
   transfer under its allowance, and the must-fail list.
3. Safe deployment at the same address on Robinhood Chain; a role call
   through Rabby with the Ledger; the Delay cooldown and its cancel.

## Consequences

- **Daily use, one Ledger tap, no vault key:** swaps through Pilot on
  supported chains and through the app's panel on Robinhood Chain; bridges
  the same way; Hyperliquid deposits from the app; Hyperliquid trading with
  the browser agent. **Rare, with the vault key:** withdrawing from
  Hyperliquid, topping up the trading Safe, moving anything out of either
  Safe (two owners in Safe{Wallet} or Rabby's Safe mode; under construction
  (b) above, queued through the Delay and executed after the cooldown).
- **Changing the policy** (a new router or bridge on the allowlist, a
  token, an allowance) is itself a non-role action: the Roles modifier's
  configuration is owner-only and its owner is the trading Safe, so a
  change is a Safe transaction by two owners (and, under (b), through the
  cooldown). A role member cannot widen its own role — the point of the
  design, and the reason allowlist changes are batched for the vault key's
  visits.
- Under a wrench attack the home key alone can trade within the allowlists
  and move money between the owner's own addresses, including the
  Hyperliquid deposit account up to its allowance. It cannot send anything
  elsewhere.
- Custom code is confined to scripts and the app's execution path; the only
  candidate for custom Solidity is the guarded swapper, and it is optional
  until measured. Custom policy is the real risk and is answered by the fork
  suite.
- Costs accepted: the Hyperliquid withdrawal needs both keys; v4-only pools
  on Robinhood Chain are outside the role for now; the Zodiac tooling gap on
  Robinhood Chain is ours to fill; the design is three systems (Safe, Roles,
  Hyperliquid multi-sig) with three failure modes to rehearse.
- Products that do parts of this for teams exist (karpatkey's Roles
  policies, Cobo Argus, Turnkey's policy engine, Squads' smart accounts).
  None covers Robinhood Chain, Hyperliquid and a brokerage under one policy
  for an individual; that integration, with the fork suite and the
  execution engine, is the thing a business could be built on.

## Solana — proposed, not decided

The same requirements, different toolbox; no Zodiac-Roles equivalent for
swaps is established there. What exists:

- **Squads v4** (audited by OtterSec, Neodyme, Certora, Trail of Bits): a
  multisig with member permissions (initiate, vote, execute), time locks, and
  **spending limits** — per member, per mint, per period, with an optional
  destination allowlist — that move funds without a vote. Transfers, not
  swaps: a swap is a vault transaction executed at threshold.
- **Squads' Smart Account Program** (mainnet 2025): admin keys, permissions,
  session keys, custom policy programs. Whether an off-the-shelf
  "swap-only through Jupiter, output to the vault" policy exists is
  unconfirmed; its docs were not reachable from the session.
- **Turnkey's policy engine**: a TEE-held key whose signing is gated by
  policies over program ids, token mints and amounts; its own write-up
  shows a swap-only Jupiter policy and notes that Jupiter's dynamic routing
  defeats rigid instruction patterns. Enforcement is off-chain, at signing
  time.
- **Cloud KMS signs Ed25519**, so the app's Solana key can be hardware-held
  like the EVM one.

Options, in increasing custom code:

1. **Vault plus capped hot wallet.** A Squads v4 vault (2 of 3, Ledger owners,
   time lock) holds the book; a small Ledger trading wallet and the app's
   KMS key each receive a spending limit from the vault (per day, to their
   own addresses only). Trading happens in those wallets with full control
   of what they hold. Exposure under a wrench attack is the hot balance plus
   the daily limit; it is a cap, which the owner has ruled out for Robinhood
   Chain and may rule out here.
2. **Turnkey-held trading key with a swap-only policy** for both the owner
   (passkey) and the app, the vault in Squads. Off the shelf, hosted,
   enforced off-chain; trust moves to Turnkey's TEE and policy engine.
3. **A Squads smart account with a swap-only policy program**, if Squads ships
   one or once their policy interface is documented: the on-chain analogue of
   Roles. To verify first.
4. **A custom guarded-vault program** (a PDA vault; a trader authority may
   CPI into Jupiter with the output delivered to the vault; withdrawals need
   the Squads multisig): the most faithful to the EVM design, and custom
   Rust with an audit.

**Owner's call, 2026-10-24: option 1 is out.** There is no on-chain vault on
Solana, only trading wallets, so the trading wallet itself must carry the
two-key withdrawal rule, which leaves options 2, 3 and 4 — or another
solution found when the work is taken up. No decision yet; the roadmap
carries the open question.
