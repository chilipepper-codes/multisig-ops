# ADR-001: Wrench-resistant self-custody for a trading book — Safe + Roles on EVM, native multi-sig on Hyperliquid, a hardware key for the app

**Status:** accepted as the design (owner, 2026-10-24); amended 2026-11-02 to
scope CLMM range orders into the trader role; amended 2026-11-04 (owner):
one Safe per chain, the guard contract decided in front of every swap and
range-order open, the CoW presets out of the role; nothing built yet. Solana is
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

- **One Safe per chain, at the same address on every chain** (amended
  2026-11-04: the earlier vault Safe is dropped; all on-chain capital is
  trading capital, so there was nothing for it to hold apart, and its
  isolation from module bugs protected an empty account). Threshold 2 of 3
  owners: the home Ledger, the vault key (the one in the bank vault) and a
  third key kept **off-site** (never a second home device: home alone must
  not reach the threshold). The Zodiac **Roles** modifier is enabled on it,
  and the **Delay** modifier under construction (b) below. The whole book
  sits behind the modules, so a Roles bug reaches all of it; the
  mitigations are the pinned versions, members that are EOAs never Safes
  (the June 2026 bug's shape), the security page watched, the per-period
  allowances, and the guard below in front of every trade. "The trading
  Safe" below means this one Safe.
- **One role, "trader", two members:** the home Ledger's EOA and the app's
  KMS key. What the role allows, and nothing else: ERC-20 approvals on
  listed tokens, the spender pinned to the guard, a listed bridge or
  Permit2, for the exact amount and in the same batch as the call that
  spends it, so no standing approval sits on the Safe; swaps and
  range-order opens through the guard, which delivers everything to the
  Safe (next bullets); closing a range position on the position manager
  with the proceeds pinned to the Safe; bridges whose destination is the
  Safe's address on the other chain; a token allowlist; per-period
  allowances; one transfer target outside the Safe, the Hyperliquid deposit
  account below, under an allowance. An approval to any other spender, and
  a transfer to any other address, is not a role call: it needs two owners.
- **The guard (decided 2026-11-04): one contract in front of every swap and
  every range-order open.** Roles conditions compare calldata and cannot
  see a price, so a compromised engine could trade through a thin or
  manipulated pool: move the spot price with a flash swap, swap or mint
  single-sided against the moved price, swap back through it, all in one
  bundle. The guard derives the bounds on-chain instead of trusting the
  caller's parameters:
  - **The reference price** is the pool's time-weighted price (`observe`),
    never `slot0`, which is what the attacker moves.
  - **A swap** pulls the input from the Safe, computes the minimum output
    from the reference price less a maximum deviation, calls the listed
    router, delivers the whole output to the Safe, and reverts otherwise.
  - **A range open** mints a single-sided position whose range lies on the
    correct side of the reference price and within a few tick spacings of
    it, with the minimum equal to the amount offered (a range the price has
    already entered reverts), the NFT to the Safe.
  - **A depth floor**, per pool, in quote-token units: the quote amount the
    pool's active liquidity holds within a band of the reference price must
    exceed it, or the call reverts. The floor is what makes the reference
    price worth trusting: a thin pool's time-weighted price is cheap to
    move, a deep one's is not. Depth can be flash-added to pass the floor,
    which is harmless at a fair price and caught by the price check
    otherwise; the two checks hold together, neither alone.
    Stablecoin-quoted pools need no conversion; WETH-quoted pools state the
    floor in ETH. An oracle (Chainlink's ETH/USD if it is deployed on
    Robinhood Chain, else the chain's deepest WETH/stable pool) would let
    floors be stated in dollars; it is an option, not a dependency, since a
    stale feed would revert every trade.
  - **The parameters are the policy's.** The deviation, the tick distance
    and the depth floor are call parameters that Roles pins per pool to a
    maximum or minimum, so changing them is a two-key policy change. The
    guard is **stateless and unowned**: routers and position managers as
    immutables, no admin, no upgrade, nothing to govern or compromise. The
    token allowlist and the allowances stay in Roles, as conditions on the
    guard's parameters.
  - **It reaches v4** where the engine wants it: the guard can call the
    Universal Router or the v4 PositionManager internally with recipients
    pinned, so the role never needs conditions on command bytes. Hooked v4
    pools are listed one by one, since a hook changes the swap math.
  - **Cost:** custom Solidity in the trust path, roughly two hundred lines,
    so an audit before real funds, and the fork suite testing its reverts
    as it tests the policy's. The time-weighted price is itself movable on
    a thin pool over several blocks, so the band is a cost imposed on a
    compromised engine, not a wall; with the depth floor that cost is what
    it takes to move a deep pool.
- **CLMM range orders (amended 2026-11-02; opened through the guard since
  2026-11-04).** The execution engine rests its limit orders as
  single-sided concentrated-liquidity positions: a range entirely above the
  price sells through it, one entirely below buys through it, and the
  position earns the pool fee from the flow that fills it. The open goes
  through the guard (above), which mints the position to the Safe on a
  v3-shaped manager (Uniswap v3's NonfungiblePositionManager; Aerodrome
  Slipstream on Base; PancakeSwap v3; Camelot on Arbitrum). The close has
  no price exposure, so it is the Safe's own call on the manager, allowed
  by the role directly: `decreaseLiquidity` (moves nothing); `collect` with
  the recipient pinned to the Safe, `address(0)` rejected (it means
  "collect into the manager" for a later sweep); `burn`; native value
  always zero (WETH is a listed token). Denied, and on the must-fail list,
  because the position NFT is funds: `mint` outside the guard, `approve`,
  `setApprovalForAll`, `transferFrom` and both `safeTransferFrom`
  overloads; `multicall` (Roles cannot see inside it; a cycle of decrease,
  collect, burn and the next open goes through the Safe's MultiSend, which
  Roles unwraps, as one role call); `sweepToken`, `unwrapWETH9`,
  `refundETH` and the permit variants. Role calls bypass the Delay, which
  the mechanism needs: a filled range un-fills if the price crosses back
  before the proceeds are collected. The engine's own rules (rung sizing,
  icebergs, reload cycles) are the tracker's, recorded in its
  execution-engine ADR.
- **A cooldown on withdrawals — open choice, corrected 2026-10-24.** The
  Delay modifier delays only what reaches the Safe *through it as a module*;
  owners who sign directly are never delayed (Zodiac's own docs). So "a
  delay on the two-key path" is one of two constructions, not a setting:
  - **(a) No delay.** The two owners sign directly; the wrench defence is the
    vault key alone. Simplest, nothing can brick the Safe.
  - **(b) The owner path routed through Delay.** A second Safe holding no
    funds, the same 2-of-3 owners, is the module enabled on the trading
    Safe's Delay; the trading Safe's own owner set is made unable to sign
    (its single owner the Delay contract, which cannot produce a
    signature), so every non-role action — a withdrawal, a policy change —
    is queued by that second Safe, waits the
    cooldown (a day or two), and can be cancelled during it by the Delay's
    owner (the home key, or the off-site third party). A coerced two-key
    withdrawal then needs the attacker to hold the owner for the whole
    delay. Cost: the trading Safe has no direct owner path at all, so a
    Delay bug or a stuck queue needs the queue's own skip (`setTxNonce`) or
    the expiration to clear; rehearse both on a fork before choosing it.
  Role calls bypass the delay in both constructions by design.
- **Swaps go through the guard, on every chain, v3 routers first.** On
  Robinhood Chain the guard calls Uniswap v3's SwapRouter02; v4 pools are
  reached through the guard when the engine wants them, hooked pools listed
  one by one. Amended 2026-11-04: the CoW presets are out of the role. The
  engine does not use CoW (a solver fills a resting order on its own terms,
  a range order fills from the pool's flow and earns the fee), the presets
  were one more code path on the Safe to fork-test, and the ERC-1271 path
  they rely on is where the June bug lived. They can come back as a policy
  change if the owner wants CoW for manual swaps through Pilot. Before this
  amendment the design deferred v4 to a measurement and kept custom
  Solidity out; the guard replaces both, justified by the residual it
  closes (the app's key, below) rather than by v4.
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
- **Deposits** go from the Safe to that account's Arbitrum address
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
  bad trades, is bounded by the token allowlist, the allowances, the
  guard's price and depth checks on-chain, and the engine's own risk
  checks. Without the guard the residual was the whole book, at whatever
  price a flash swap could move a thin pool to, in one bundle (a swap with
  no trusted minimum output, or a single-sided open just beyond the moved
  price); with it, a bad trade costs the attacker what it takes to move a
  deep pool's time-weighted price out of the band. One caveat on
  allowances: an allowance on open amounts counts the same capital every
  time a reload cycle re-mints it, so it is sized for churn or left to the
  engine. Never a seed in the app.

### Boundaries and rules

- **This repository** holds the custody layer: the policies as code, the
  guard contract, the apply and verify scripts, the fork test suite, the
  Safe deployment scripts, the Hyperliquid multi-sig tooling, the KMS
  signer service. The portfolio
  tracker keeps the execution engine and talks to this layer through a
  narrow intent API; it records the boundary in its own ADR.
- **Agents never hold any of these keys and never run against mainnet.**
  Fork and testnet only, as the tracker's IBKR rule already says for
  execution. Nothing in this repository places a real order.
- **The fork test suite comes before any policy goes live**: every allowed
  call passes, a list of must-fail calls (a transfer out, an approval to
  another spender, a router or a `mint` called outside the guard, an
  unlisted token) reverts, and the guard's own reverts hold (a swap past
  the band, an open against a moved spot, a pool under the depth floor),
  on every chain, on every policy change and on a schedule. Policy is code that fails
  silently; this is what makes it safe to change.
- Build on Roles 2.1.1 and Delay 1.1.1 with the current unwrapper; migrate to
  v3 only after it has been in production elsewhere. Subscribe to Zodiac's
  security page.

### Verify before real funds

1. Hyperliquid testnet: the multi-sig conversion (one report says converting
   back failed, treat it as one-way); approving an agent through a
   multi-sig-wrapped action; trading through that agent from the app and from
   the web UI; a withdrawal refusing one signature.
2. A fork of Robinhood Chain: the guard (a swap within the band passes and
   delivers to the Safe; a swap after a flash move of the spot reverts; a
   single-sided open to the Safe; an open against a moved spot reverts; a
   pool under the depth floor reverts), the close permission on the
   position manager (a partial fill decreased and collected to the Safe, a
   reload cycle as one MultiSend), the bridge permission, the Hyperliquid
   deposit transfer under its allowance, and the must-fail list, with a
   direct router call, a direct `mint`, the position NFT's transfers and
   approvals, an approval to another spender and a `collect` to another
   recipient among it.
3. Safe deployment at the same address on Robinhood Chain; a role call
   through Rabby with the Ledger; the Delay cooldown and its cancel.
4. The guard's audit, its findings closed in the fork suite, before any real
   funds pass through it.

## Consequences

- **Daily use, one Ledger tap, no vault key:** swaps through Pilot on
  supported chains and through the app's panel on Robinhood Chain; bridges
  the same way; Hyperliquid deposits from the app; Hyperliquid trading with
  the browser agent. **Rare, with the vault key:** withdrawing from
  Hyperliquid, moving anything out of the Safe, changing the policy (two
  owners in Safe{Wallet} or Rabby's Safe mode; under construction (b)
  above, queued through the Delay and executed after the cooldown).
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
- Custom Solidity is the guard and nothing else: stateless, unowned,
  audited, its reverts in the fork suite. A bug in it stops trading and
  never moves funds, since it holds nothing and sends only to the Safe.
  Custom policy is the other risk, and the same suite answers it.
- Costs accepted: the Hyperliquid withdrawal needs both keys; the guard is
  a contract to audit and a dependency of every trade; the engine's reload
  cycles are signed role calls, so the KMS key's gas balance and the
  signer's audit volume scale with how actively it trades; the Zodiac
  tooling gap on Robinhood Chain is ours to fill; the design is three
  systems (Safe, Roles, Hyperliquid multi-sig) with three failure modes to
  rehearse.
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
  time. Range orders are the easier case there: Orca Whirlpools, Raydium
  CLMM and Meteora DLMM (whose single-sided bins are the native limit-order
  idiom) have static instruction and account layouts, and the rule is the
  EVM one (the position owner and every destination token account pinned
  to the trading wallet; transfers of the position mint denied), whichever
  option below is taken.
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
