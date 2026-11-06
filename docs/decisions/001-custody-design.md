# ADR-001: Wrench-resistant self-custody for a trading book — Safe + Roles on EVM and on HyperEVM as the Hyperliquid account, a hardware key for the app

**Status:** accepted as the design (owner, 2026-10-24); amended 2026-11-02 to
scope CLMM range orders into the trader role; amended 2026-11-04 (owner):
one funded Safe per chain, the guard contract decided in front of every swap and
range-order open, the CoW presets out of the role; amended 2026-11-05 (owner):
the token registry behind a whitelisting Delay the home key alone can queue on,
and the approver module; amended again 2026-11-05 after the owner's user
stories (`docs/stories.md`, the spec this design is measured against):
three owner keys, 3 of 3 direct and 2 of 3 through the Delay, a wide
band on sales, the whitelisting attack accepted at a three-day cooldown, a
pause flag; decided 2026-11-05 (owner): the Safe is the Hyperliquid
account, validated on testnet before live funds, the native multi-sig out;
nothing built yet. Solana is
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
  actions (ecrecover; a Safe cannot). Amended 2026-11-05: since 2025
  HyperEVM's CoreWriter system contract lets a contract act for its own
  address on HyperCore, so a Safe at the same address on HyperEVM holds and
  controls that account; the Zodiac stack, Safe 1.4.1 and CoreWriter have
  bytecode on HyperEVM at their canonical addresses (read from the chain).
  Hyperliquid's own docs: CoreWriter does not work for native multi-sig
  users, and multi-sig users should not interact with HyperEVM.
- Uniswap v2, v3, v4 and UniswapX are live on Robinhood Chain; the v3
  SwapRouter02 takes the recipient as a plain struct field; the Universal
  Router (v4) takes nested command bytes. The canonical Universal Router
  address there has changed three times this year.
- Cloud KMS signs secp256k1 (EVM, Hyperliquid) and Ed25519 (Solana) in an
  HSM; the private key never leaves it.

## Decision

### EVM chains (Ethereum, Arbitrum, Base, Robinhood Chain)

- **One funded Safe per chain, at the same address on every chain** (amended
  2026-11-04: the earlier vault Safe is dropped; all on-chain capital is
  trading capital, so there was nothing for it to hold apart, and its
  isolation from module bugs protected an empty account. Under construction
  (b) below a second Safe still exists per chain as the Delay's queuing
  module, but it holds nothing, so the isolation argument does not come
  back with it: one Safe holds funds either way). Three owners, all the
  owner's: the home Ledger, the vault key (the one in the bank vault) and
  a third key kept **off-site** (never a second home device: home alone
  must not reach any threshold). **Threshold 3 of 3** on this Safe: the
  direct path is the break-glass, with no module between the three keys
  and the funds (decided 2026-11-05, the owner-path bullet below). The
  Zodiac **Roles** modifier and the **Delay** modifier are enabled on it;
  the everyday two-key path goes through the Delay. The whole book
  sits behind the modules, so a Roles bug reaches all of it; the
  mitigations are the pinned versions, members that are EOAs never Safes
  (the June 2026 bug's shape), the security page watched, the per-period
  allowances, and the guard below in front of every trade. "The trading
  Safe" below means this one Safe.
- **One role, "trader", two members:** the home Ledger's EOA and the app's
  KMS key. What the role allows, and nothing else: calls to the guard
  (swaps and range-order opens, on the whitelisted path or the degen path as
  the registry says, and bridges of any held token to the Safe's own
  address on the other chain; the guard delivers everything to the Safe,
  next bullets; its approvals are the approver module's and need no role
  call); closing a range position on the position manager with the
  proceeds pinned to the Safe; removing a token from the registry; on
  HyperEVM, the CoreWriter actions the Hyperliquid section lists, scoped
  as calldata. Nothing the role allows sends funds to an address that is
  not the Safe's own (amended 2026-11-05: the Hyperliquid deposit hop, the
  one such transfer, is gone with the native multi-sig). The token set, the per-pool parameters and
  the per-period notionals are the registry's, not Roles conditions
  (amended 2026-11-05: Roles scopes calls per target address and cannot
  scope "approve on any token", so a token set in Roles would make every
  whitelisting, and every sale of a degen token, a Roles change). An approval
  to any other spender, a call on a token contract outside those, and a
  transfer to any other address, is not a role call: it needs two owners.
- **The guard (decided 2026-11-04): one contract in front of every swap and
  every range-order open.** Roles conditions compare calldata and cannot
  see a price, so a compromised engine could trade through a thin or
  manipulated pool: move the spot price with a flash swap, swap or mint
  single-sided against the moved price, swap back through it, all in one
  bundle. The guard derives the bounds on-chain instead of trusting the
  caller's parameters:
  - **The reference price** is the pool's time-weighted price (`observe`),
    never `slot0`, which is what the attacker moves. The checks below
    apply to buys and to sales alike, with different parameters. **Sales
    get a wide band, 30% by default, over a short window** (owner,
    2026-11-05): a rug must be exitable at once with the home key, and a
    narrow band against a lagging time-weighted price would lock the owner
    in, while no band at all would let a thief with the home key sandwich
    the Safe's own sales in their real pool for the whole position. The
    wide band bounds that loss per sale at the band; the short window
    (minutes, a per-pool parameter) bounds how long a crash faster than
    the window can hold a sale back, at most the window itself. With the
    whitelist and the degen allowance in place the checks need not be
    stringent anywhere, so the buy-side defaults are loose too, tightened
    per pool by the owner. On every sale the guard sells only through a
    pool the registry knows for the token (the whitelisted pool, or the pool a
    degen token was first bought in), never one the caller names, so a
    thief cannot route a sale to a pool of their own; a pool with no
    time-weighted price yet (a young one) passes a sale on that rule
    alone.
  - **A swap** pulls the input from its caller, the Safe, computes the
    minimum output
    from the reference price less a maximum deviation, calls the whitelisted
    router, delivers the whole output to the Safe, and reverts otherwise.
  - **A range open** mints a single-sided position whose range lies on the
    correct side of the reference price and within a few tick spacings of
    it, with the minimum equal to the amount offered (a range the price has
    already entered reverts), the NFT to the Safe.
  - **A bridge** (2026-11-05) pulls any held token, whitelisted or not, and
    calls a whitelisted bridge with the destination pinned to the Safe's own
    address on the other chain; no price check and no cap, since the
    destination is the owner's. It is in the guard because a bridge needs
    an approval on the token, which Roles cannot scope for a token it has
    not whitelisted; the approver grants it. **How the destination is pinned:**
    every bridge worth whitelisting takes the recipient as a parameter of the
    call itself (CCTP's `mintRecipient`, Across's `recipient`, the
    Arbitrum gateway's `to`, a DLN order's `receiverDst`), so the value
    is in the transaction the Safe signs, and Roles compares it to the
    Safe's address before the call runs; the guard does the same for its
    own bridge function by setting the recipient to its caller. Because
    the Safe has the same address on every chain, the pinned value is one
    constant. What cannot be pinned is a bridge whose recipient lives only
    in an off-chain quote (Relay's deposits: the Safe's transaction is a
    transfer to the relayer with a request id, and the request id says
    nothing Roles can read), so such a bridge is not a role call.
  - **A depth floor relative to the trade, per pool, for whitelisted pools.**
    The floor is what makes the reference price worth trusting: moving a
    time-weighted price by the band must cost more than it gains. On a pool
    nobody arbitrages, pushing the price and pulling it back costs only the
    fees of the round trip, so the cost is the depth within the band times
    the fees, and the gain is the trade size times the band. Both are in
    the same token, so the floor needs no valuation of either token and
    holds for any pair, a shitcoin against a tokenized stock included: the
    depth within the band, on the side the trade consumes, must be at
    least N times the trade size, or the call reverts. N is a policy
    parameter per pool, not a rule of the guard. The multiple that makes
    the inequality hold is the band over twice the fee (about one half on
    a 1% pool with a 1% band, about ten on a 0.05% pool), mild at realistic
    settings; the default is looser still or zero (owner, 2026-11-05), set
    per pool where the owner wants it, and zero on the degen path, since a
    conservative multiple would stop the owner aping a fresh launch, which
    is sometimes the trade. Depth can be flash-added to pass the floor, which is harmless
    at a fair price and caught by the price check otherwise; the two
    checks hold together, neither alone.
  - **An absolute floor through a pricing path**, as a sanity bound on top.
    "Never trade a pool with less than this in it" needs a unit, and a
    quote asset can be anything (the current fashion pairs coins with
    tokenized stocks). The guard values the quote through the chain's own
    pools: at most two hops to a stablecoin or WETH (coin to stock token,
    stock token to USDC), each hop a time-weighted price in a pool that
    must clear its own relative floor. The path is a registry parameter
    per pool, like the band and the multiple. For a tokenized stock its USDC pool on the chain is its
    on-chain market and the natural source. External feeds (Chainlink,
    Pyth) exist for ETH and may exist for tokenized stocks on Robinhood
    Chain, never for a coin; on-chain they are at most a sanity check on a
    stock token's hop, where one is deployed, and a stale feed must never
    halt a pool that clears its own floor. The engine, off-chain, is free
    to use any feed, exchange price or the stock's real quote in its own
    checks, with no on-chain dependency.
  - **The degen allowance (owner, 2026-11-04): buys of non-whitelisted tokens,
    capped.** A just-launched coin is not on the token whitelist, and
    whitelisting one takes the home key and the cooldown, so without this the
    Safe could never ape a launch in time. The role may buy a token that is not on the whitelist,
    through the guard, under a per-period allowance stated in stablecoin
    terms through the pricing path, provided the pool is quoted in a
    stablecoin or WETH and clears the absolute floor. This is the one
    priced exit in the design, and it is shaped by the wrench attack: a
    coerced buy of the attacker's coin in the attacker's pool is how the
    role path would be made to pay out, so the absolute floor makes the
    attacker show up with a pool holding real money (a capital barrier,
    not a loss to them, since they take their liquidity back), and the
    allowance is what a coercion yields per period, the number the owner
    is prepared to lose with a wrench at their head. The time-weighted
    price is no defence here, the attacker set it, and neither is the
    Delay, which role calls bypass. Selling any token the Safe holds back
    into the reserve set is never capped and needs no whitelisting, since the
    approver grants the guard its approval at sale time. A token bought
    under the degen allowance stays on the degen path, re-buys included,
    until it is whitelisted: whitelisting on first buy would hand a thief an
    uncapped path for a dollar of their own coin. A young v3 pool has an
    oracle cardinality of one, so `observe` over the window reverts until
    it is grown and filled; the guard lets a buy in such a pool through on
    the absolute floor and the degen allowance alone, without the price
    check, and that is the degen path by construction. The registry
    records the pool of that first buy as the token's pool, which is where
    the guard will sell it. The engine grows the pool's cardinality in the
    same batch as its first buy, so a sell band can be switched on for the
    position later if wanted. Launches on v2 pairs and on v4 are outside
    the guard and so outside the role.
  - **The registry holds the policy the guard reads (option B, owner,
    2026-11-05).** One contract per chain, owned by the funded Safe (so
    two keys through the Delay, or three direct), holding: the whitelisted set,
    and for every token the pool the guard trades it in; per pool, the
    deviation band, the tick distance, the depth multiple, the absolute
    floor, the pricing path and the sell band, each with a default applied
    at whitelisting; two running per-period notionals in stablecoin terms
    through the pricing path, one for the whitelisted path and one for the
    degen path, which the guard debits with the value of what is
    **bought** on every swap and open; and a **pause flag** (owner,
    2026-11-05): any role member sets it with a role call, at once, and
    the guard refuses every call while it is set; clearing it is the
    owner's, two keys through the Delay. It is what bounds a compromised
    app key the moment the owner notices, since removing a role member is
    a two-key change. Buys of the
    reserve set (the stablecoins and WETH, a two-key list) count for
    nothing, so a sale of any token into them is never capped: de-risking
    is not an exit. Whitelisting a token needs no allowance entry: the notionals
    are per path, not per token. The guard itself holds no funds
    and has no admin: routers, position managers, the registry and the
    approver as immutables, no upgrade; the registry is its state. A
    reload cycle re-mints the same capital, so how the notional nets a
    close against the next open is settled when the registry is written.
  - **The whitelisting Delay: the home key alone can add a token, after a
    cooldown.** A Zodiac Delay modifier whose target is the registry and
    whose only module is the home Ledger's EOA. It can queue one thing,
    an add with the default parameters; after the cooldown (**three days**,
    the owner's number, 2026-11-05) anyone executes it. Loosening a pool's
    parameters or raising a notional is two-key, as is any other policy
    change: otherwise a coercer would queue "whitelist my token, multiple
    zero". Removing a token is a role call, immediate, since narrowing is
    always safe. The Delay's owner, who can skip a queued entry
    (`setTxNonce`), is **the vault key**; a guardian who can veto a whitelisting
    alone is left open as an optional improvement. **The whitelisting is queued
    from the tracker's UI**: the app builds the call, the home Ledger
    signs it through Rabby, the app's worker executes the entry once the
    cooldown has passed (executing needs no permission, only gas), and the
    app shows what is queued and when it executes; the vault key's skip is
    a Rabby flow too. No second app: it would be a second thing to secure
    for nothing. **The whitelisting attack is accepted** (owner, 2026-11-05):
    whoever holds the home key can whitelist a coin of their own and, after the
    cooldown, buy it on the whitelisted path up to that path's per-period
    notional. The cooldown, the vault key's skip and the monitoring step
    are the answer, not a probation on new whitelistings (an option, in the
    roadmap). That notional is the second number the owner sizes with a
    wrench in mind, after the degen allowance.
  - **The approver: one module on the Safe, one function.** It makes the
    Safe approve the guard for a token, any token, and nothing else; it
    trusts the guard alone, and the guard invokes it only when the Safe is
    its caller, right before a pull. An approval to the guard is spendable
    only through the guard's own checks, and the guard delivers only to
    the Safe, so the approver opens no exit; what it removes is the
    per-token Roles scoping that would otherwise make every whitelisting, and
    every sale of a degen token, a two-key change. Standing approvals to
    the guard accumulate for every token the Safe has traded, which puts
    the guard squarely in the trust path: a bug in it could misroute what
    it is approved to pull. That is why the audit covers the guard, the
    registry and the approver together, before real funds.
  - **v3-shaped pools only, for now.** The reference price is v3's built-in
    `observe`. A v4 pool has no oracle of its own, only what a hook
    provides, and a hook also changes the swap math, so the guard's price
    and depth checks are not defined there and v4 pools stay outside the
    role. Reaching v4 through the guard (which would spare the role any
    conditions on command bytes) is a queued decision whose open question
    is the reference price: a v3 pool of the same pair with enough depth,
    or a hook oracle whitelisted pool by pool. It is taken when the engine's
    v3-versus-v4 measurement (portfolio-tools ADR-014, paper mode) shows
    that v4-only pairs matter.
  - **Cost:** custom Solidity in the trust path, the guard, the registry
    and the approver together a few hundred lines, so an audit before real
    funds, and the fork suite testing their reverts as it tests the
    policy's. The time-weighted price is itself movable on
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
  always zero (WETH is a whitelisted token). Denied, and on the must-fail list,
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
- **The owner path, decided 2026-11-05 (owner): three keys direct, two
  keys through the Delay.** The Delay modifier delays only what reaches
  the Safe *through it as a module*; owners who sign directly are never
  delayed (Zodiac's own docs). With two keys alone there is no
  construction where a delay binds them and they can still rescue the
  book from a Delay bug, which is why the third key is load-bearing:
  - **Direct, 3 of 3.** The funded Safe's own threshold. The break-glass
    that no module, ours or Zodiac's, can lock: a bug anywhere, the three
    keys withdraw everything at once. Impractical under a wrench, since it
    means home, the bank and the off-site place in one trip.
  - **Everyday, 2 of 3, delayed.** A second Safe holding no funds, the same
    three owners at threshold 2, is the module enabled on the funded
    Safe's Delay. Every non-role action — a withdrawal, a policy change, a
    notional raised, the pause cleared — is queued by two keys, waits the
    cooldown (configurable; a day or two), and anyone executes it after.
    The Delay's owner is the funded Safe itself, so shortening the
    cooldown or skipping an entry takes all three keys. A coerced two-key
    withdrawal needs the attacker to hold the owner through the whole
    cooldown while the entry sits visible on chain (the monitoring step);
    an owner who is free during it moves the funds with three keys first.
    A lost key leaves this path intact with the other two.
  Rehearse the skip and the expiration on a fork. Role calls bypass the
  Delay by design.
- **Swaps go through the guard, on every chain, v3 routers first.** On
  Robinhood Chain the guard calls Uniswap v3's SwapRouter02; v4 pools stay
  outside the role until the queued decision above gives the guard a
  reference price there. Amended 2026-11-04: the CoW presets are out of the role. The
  engine does not use CoW (a solver fills a resting order on its own terms,
  a range order fills from the pool's flow and earns the fee), the presets
  were one more code path on the Safe to fork-test, and the ERC-1271 path
  they rely on is where the June bug lived. They can come back as a policy
  change if the owner wants CoW for manual swaps through Pilot. Before this
  amendment the design kept custom Solidity out until a measurement said
  v4 mattered; the guard is now justified by the residual it closes (the
  app's key, below), and the measurement stays as the trigger for the v4
  decision.
- **Tooling on Robinhood Chain is ours.** Policy as code with the Roles SDK's
  condition utilities, applied through the Safe by script and read back from
  the modifier's events over RPC (no subgraph there); the app builds
  `execTransactionWithRole` calls for the role member to sign with the Ledger
  through Rabby. On the other chains the Zodiac app and Pilot are used as is.
  Ask Gnosis Guild to add the chain; do not depend on it.
- **Same-address Safes by script** (factory, singleton, initializer, salt
  replayed per chain), until Safe{Wallet}'s multi-chain deployment covers
  Robinhood Chain.

### Hyperliquid (decided 2026-11-05: the Safe is the account)

- **The account is the Safe.** The same Safe, at the same address, deployed
  on HyperEVM with the same Roles and Delay as on every other chain: the
  Roles 2.1.1 and Delay 1.1.1 mastercopies, the module proxy factory, the
  MultiSend unwrapper, the Safe 1.4.1 singleton and its factory all have
  bytecode on HyperEVM at their canonical addresses, as does CoreWriter at
  `0x3333…3333` (read from the chain's RPC, 2026-11-05; the official
  Roles v2 deployments list names HyperEVM). Its HyperCore account is a
  normal user that happens to be a contract address, created by its first
  deposit (a 1 USDC activation), holding a little HYPE on Core for
  transfers and on HyperEVM for gas. It is **never converted into a native
  multi-sig user**: Hyperliquid's own multi-sig page says CoreWriter does
  not work for multi-sig users and that multi-sig users should not
  interact with HyperEVM at all. And it has no key on HyperCore, so the
  June 2025 phishing campaign that converted some 1,200 accounts into
  attacker-controlled 1-of-1 multi-sigs with one signature has nothing to
  sign with here. The owner's condition: validated thoroughly on testnet
  before any live funds (build step 4).
- **Control is CoreWriter, scoped by Roles, with no custom code.**
  CoreWriter has one function, `sendRawAction(bytes)`, and the payload is
  one version byte, a three-byte action id, then the ABI-encoded
  parameters: the shape of a function call with a four-byte selector,
  which is Roles' `Calldata` parameter type. So the trader role scopes
  the bytes as it would a call: USD class transfer (action 7) in either
  direction; send asset (action 13) with the destination pinned to the
  USDC system address, the token pinned to USDC and the sub-account to
  zero; orders and cancels (1, 10, 11) if the owner wants the Safe itself
  to trade. Add API wallet (action 9), set abstraction (16) and anything
  else is two-key through the Delay. The Hyperliquid side therefore needs
  no guard, no registry, no custom Solidity: policy only, proved by the
  fork suite like the rest. The `hyper-evm-lib` library (hyperliquid-dev)
  encodes every action and ships a local simulator for Foundry, which is
  what the fork tests use. CoreWriter actions are attributed to the
  calling contract, the Safe; the Zodiac module path leaves the Safe as
  the caller.
- **Trading is by API wallets**, approved through CoreWriter by two keys:
  the owner's browser agent and the app's KMS key. API wallets cannot
  withdraw by Hyperliquid's own rules; the engine signs orders with the
  KMS key off-chain exactly as before.
- **Deposits are role calls with the destination in the signed calldata**,
  never a transfer to an address that is not the Safe's own. Best first:
  (1) **CCTP v2 with Circle's forwarder**: one transaction from the
  Arbitrum Safe, `depositForBurnWithHook` with the mint recipient and
  destination caller pinned to Circle's forwarder on HyperEVM and the
  HyperCore recipient, the Safe, in the hook data (bytes 32 to 52, pinned
  by Roles' bitmask), which the forwarder deposits for the Safe into perp
  or spot; Circle's canonical contracts end to end, no aggregator, no
  solver. (2) **Plain CCTP to the HyperEVM Safe**, then the Safe's own
  deposit for itself through Circle's `CoreDepositWallet.depositFor` with
  the recipient pinned to the Safe: two transactions, same trust. (3)
  **The native bridge from Arbitrum**, credited to the sender: pinned by
  construction. From other chains, first to the Arbitrum Safe over a
  bridge with the destination pinned, then (1). Aggregators (Across's
  `HyperliquidDepositHandler`, LI.FI's Polymer CCTP facet, deBridge's
  orders) also carry the recipient in calldata, but their contracts and
  relayers join the trust path for nothing (LI.FI's facet is route (1)
  with LI.FI in front); a bridge whose recipient lives only in an
  off-chain quote (Relay) cannot be pinned and is not a role call.
- **Withdrawals back to the wallet are a home-key action**, requirement 2
  met for Hyperliquid, which the native multi-sig never could: a USD
  class transfer to spot, a send asset to the USDC system address (paid
  out by an ERC-20 transfer to the sender's own HyperEVM address, the
  Safe), and a CCTP burn from the HyperEVM Safe with the mint recipient
  pinned to the Arbitrum Safe. All transfers between the owner's own
  addresses, all role calls. Leaving the Safe system stays the two-key
  delayed path as everywhere.
- **Exposure and failure modes, stated squarely.** The account's funds
  depend on CoreWriter accepting the Safe's actions and on the USDC link
  between HyperCore and HyperEVM (the system address and Circle's deposit
  wallet), since CoreWriter has no direct withdrawal to Arbitrum. Both are
  Hyperliquid-operated mechanisms that the whole HyperEVM ecosystem
  depends on (Hyperliquid steers vault protocols to exactly this pattern;
  Circle's forwarder, Across's handler and LI.FI's facet run on it in
  production), but they are the exposure, where the native design's was
  the Arbitrum bridge; the mitigation is the one already in force, a
  Hyperliquid balance kept at trading size. The engine is designed around
  the known failure modes: CoreWriter actions are fire-and-forget (the
  EVM transaction succeeds whether or not HyperCore accepts the action; a
  wrongly encoded action, or one from an address with no HyperCore
  account yet, is a silent no-op, not a revert), so every action is
  confirmed by reading HyperCore state through the precompiles and "not
  landed" is retryable, never lost; an open report on the node tracker
  (September 2026, testnet, unresolved) of actions that succeeded on the
  EVM and did nothing on Core is what the testnet soak reproduces or rules
  out before any mainnet step; unified-account mode rejects spot send, so
  the policy uses send asset; order actions from CoreWriter land a few
  seconds after the EVM block. To prove on a fork: Roles' `Calldata`
  typing of the CoreWriter payload and the bitmask on CCTP hook data.
  Circle's forwarder is an upgradeable proxy Circle administers, the one
  party trusted beyond Hyperliquid and Zodiac, and the same party already
  trusted for USDC itself.
- **Rejected (2026-11-05): the native multi-sig account.** A fresh EOA
  converted to a native multi-sig user with the home and vault keys at
  threshold 2 was the design until this amendment. It needed a deposit
  hop through that EOA's own Arbitrum address, whose key sat at home, and
  so a per-period allowance on the one transfer out of the Safe; its
  withdrawal could go to any destination and needed both keys, the one
  requirement the design did not meet; conversion is one-way in practice
  (the documented conversion back is reported broken in the Python SDK's
  tracker, December 2025); the account has a key, which is what the 2025
  conversion phishing signs with; and Hyperliquid's own rules keep a
  multi-sig user off HyperEVM, so it could never have used CoreWriter.
  The Safe account has none of these and the same zero custom code.

### The app's key

- A **Cloud KMS HSM key** (secp256k1 for EVM and Hyperliquid), usable by one
  service account, every signature audit-logged. It is a role member on the
  trading Safe and an API wallet on Hyperliquid. By construction a full
  compromise of the app can only trade inside the whitelists; its residual,
  bad trades, is bounded by the registry's whitelisted set and per-period
  notionals, the guard's price and depth checks on-chain, and the engine's
  own risk checks. Without the guard the residual was the whole book, at whatever
  price a flash swap could move a thin pool to, in one bundle (a swap with
  no trusted minimum output, or a single-sided open just beyond the moved
  price); with it, a bad trade costs the attacker what it takes to move a
  deep pool's time-weighted price out of the band. Never a seed in the
  app.

### Boundaries and rules

- **This repository** holds the custody layer: the user stories
  (`docs/stories.md`) the design is measured against, the policies as
  code, the guard, the registry and the approver, the apply and verify
  scripts, the fork test suite, the
  Safe deployment scripts, the Hyperliquid multi-sig tooling, the KMS
  signer service. The portfolio
  tracker keeps the execution engine and talks to this layer through a
  narrow intent API; it records the boundary in its own ADR.
- **Agents never hold any of these keys and never run against mainnet.**
  Fork and testnet only, as the tracker's IBKR rule already says for
  execution. Nothing in this repository places a real order.
- **The fork test suite comes before any policy goes live**: every allowed
  call passes, a list of must-fail calls (a transfer out, an approval to
  another spender or above the allowance, a router or a `mint` called
  outside the guard, an non-whitelisted token) reverts, and the guard's own reverts hold (a swap past
  the band, an open against a moved spot, a trade larger than the pool's
  depth allows, a pricing hop under its floor, a buy of an non-whitelisted token
  above the degen allowance or in a pool under the absolute floor or
  quoted in another non-whitelisted token),
  on every chain, on every policy change and on a schedule. Policy is code that fails
  silently; this is what makes it safe to change.
- Build on Roles 2.1.1 and Delay 1.1.1 with the current unwrapper; migrate to
  v3 only after it has been in production elsewhere. Subscribe to Zodiac's
  security page.

### Verify before real funds

1. Hyperliquid testnet, the Safe as the account: the same-address Safe,
   Roles and Delay deployed on HyperEVM testnet; a CCTP deposit from the
   Arbitrum testnet Safe with the recipient pinned in the hook data,
   credited to the Safe's HyperCore account (and a native-bridge deposit,
   credited to the sender); an API wallet approved through CoreWriter by
   two keys, then trading through it from the app and from the web UI; a
   class transfer and a send asset to the system address as role calls,
   the USDC landing on the HyperEVM Safe, and a pinned CCTP burn back to
   the Arbitrum Safe; every CoreWriter action confirmed through the
   precompiles, with the silent-failure report on the node tracker
   reproduced or ruled out; a send asset to any other destination, and an
   add API wallet by the role, refused by Roles; a soak of weeks before any
   live funds (owner's condition).
2. A fork of Robinhood Chain: the guard (a swap within the band passes and
   delivers to the Safe; a swap after a flash move of the spot reverts; a
   single-sided open to the Safe; an open against a moved spot reverts; a
   trade larger than the pool's depth allows reverts; a quote priced
   through a hop under its floor reverts; a buy of an non-whitelisted token
   passes under the degen allowance in a young pool over the floor, and
   reverts above the allowance, under the floor, or quoted in another
   non-whitelisted token), the registry and the whitelisting Delay (an add queued by
   the home key executes after the cooldown and not before; the vault key
   skips it; a home-key call that loosens a parameter or raises a notional
   reverts; a removal by the role is immediate), the approver (refuses any
   caller but the guard and any spender but the guard; a degen token
   bought in one block is sold in the next, through the pool it was bought
   in, and a sale routed to another pool reverts), the pause (set by a
   role call, the guard refuses everything; cleared only through the
   Delay), the close permission on the
   position manager (a partial fill decreased and collected to the Safe, a
   reload cycle as one MultiSend), the bridge permission, and the
   must-fail list, with a
   direct router call, a direct `mint`, the position NFT's transfers and
   approvals, an approval to another spender or above the allowance, and a
   `collect` to another recipient among it.
3. Safe deployment at the same address on Robinhood Chain, the funded
   Safe at 3 of 3 and the 2-of-3 Safe on its Delay; a role call through
   Rabby with the Ledger; a two-key withdrawal that waits the cooldown; a
   three-key withdrawal that does not, with a module disabled; the skip
   by three keys and the expiration.
4. The guard's audit, its findings closed in the fork suite, before any real
   funds pass through it.

## Consequences

- **Daily use, one Ledger tap, no vault key:** swaps through Pilot on
  supported chains and through the app's panel on Robinhood Chain; bridges
  the same way; Hyperliquid deposits, and withdrawals back to the Safe,
  from the app; Hyperliquid trading with the browser agent. **Rare, with
  two keys:** moving anything out of the Safe, changing the policy,
  clearing the pause (two owners of the 2-of-3 Safe in Safe{Wallet} or
  Rabby's Safe mode, queued through the Delay, executed after the
  cooldown). **Rarest, with all three keys:** the break-glass withdrawal,
  shortening a cooldown, skipping a queued entry.
- **Changing the policy** is a non-role action with one exception.
  Whitelisting a token is the home key's alone, through the whitelisting Delay and
  its cooldown, with default parameters. Everything else (a new router or
  bridge, a pool's parameters loosened, a notional raised, the Roles
  configuration itself, whose owner is the Safe) is a Safe transaction by
  two owners through the Delay and its cooldown. A role member cannot
  widen its own role beyond that one delayed add — the point of the
  design, and the reason the other changes are batched for the vault
  key's visits.
- Under a wrench attack the home key alone can trade within the whitelists
  and move money between the owner's own addresses, the Safe's
  Hyperliquid account included. The one priced exit is
  the degen allowance: a coerced buy of the attacker's token in a pool the
  attacker has funded past the absolute floor, capped per period. A
  coerced whitelisting costs the attacker the cooldown with the owner in hand,
  visible on chain, and then at most the whitelisted path's per-period
  notional in a pool that clears the floors. It cannot send anything
  elsewhere.
- **A stolen home key**, with no owner to coerce, takes the degen
  allowance per day. The residuals, each accepted by the owner
  (2026-11-05): after the three-day whitelisting cooldown, the whitelisted path's
  per-period notional into a pool of the thief's own, unless the vault key
  skips the whitelisting first; and a sandwich of the Safe's
  own sales in their real pool by a thief who also has the capital to
  move that pool (which they get back), bounded per sale by the 30% sell
  band and by what moving a deep pool's time-weighted price costs. The
  pause flag ends all of it the moment
  the owner notices, with the home key; the monitoring step is what makes
  noticing likely.
- Custom Solidity is the guard, the registry and the approver: holding no
  funds, with no admin beyond the Safe, audited together, their reverts in
  the fork suite. The guard is in the trust path through the approvals it
  accumulates, so a bug in it could misroute what it is approved to pull;
  the registry and the approver can at worst widen the whitelisted set or an
  approval to the guard, never move funds. Custom policy is the other
  risk, and the same suite answers it.
- Costs accepted: the Hyperliquid balance depends on CoreWriter and on
  the HyperCore-to-HyperEVM USDC link, so it stays at trading size; three
  contracts to audit and a dependency of every trade; a coerced whitelisting is
  delayed and visible, not refused; v4-only pools are
  outside the role until the guard has a reference price there; the engine's reload
  cycles are signed role calls, so the KMS key's gas balance and the
  signer's audit volume scale with how actively it trades; the Zodiac
  tooling gap on Robinhood Chain is ours to fill; the design is one
  system (Safe, Roles, Delay, the guard) on every chain, HyperEVM
  included, with HyperCore's contract-account mechanics the one thing not
  of our choosing to rehearse.
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
  destination whitelist — that move funds without a vote. Transfers, not
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
