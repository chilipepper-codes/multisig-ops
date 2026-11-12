# ADR-001: Wrench-resistant self-custody for a trading book — Safe + Roles on EVM and on HyperEVM as the Hyperliquid account, a hardware key for the app

**Status:** accepted as the design (owner, 2026-10-24); amended 2026-11-02 to
scope CLMM range orders into the trader role; amended 2026-11-04 (owner):
one funded Safe per chain, the guard contract decided in front of every swap and
range-order open, the CoW presets out of the role; amended 2026-11-05 (owner):
the token registry behind a whitelisting Delay the home key alone can queue on,
and the approver module; amended again 2026-11-05 after the owner's user
stories (`docs/stories.md`, the spec this design is measured against):
three owner keys, 3 of 3 direct and 2 of 3 through the Delay, a wide
band on sales, the whitelisting attack accepted at a three-day cooldown (both
Delays configurable, three days by default), a pause flag; decided 2026-11-05 (owner): the Safe is the Hyperliquid
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
  as calldata; deposits to the owner's Kraken account at the pinned
  address (the Kraken bullet below). Nothing else the role allows sends
  funds to an address that is not the Safe's own (amended 2026-11-05: the
  Hyperliquid deposit hop, the one such transfer, is gone with the native
  multi-sig; amended 2026-11-06: the Kraken deposit is the one that
  remains, and it is an exit only into another vault of the owner's). The token set, the per-pool parameters and
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
  - **The minimum is the caller's; the reference price is a backstop**
    (owner, 2026-11-09). The app looks prices up off-chain, shows the
    owner the amounts it expects to receive and the tolerance, and passes
    the minimum to the guard, which enforces it on the outcome. That is
    the primary check, and it is strong wherever the caller is honest:
    the owner signing through Rabby sees the amounts (Rabby's pre-sign
    simulation shows the Safe's balance changes, and a Ledger
    clear-signing descriptor for the guard's calls, ERC-7730, would put
    the input and the minimum on the hardware screen, which also defeats
    malware rewriting calldata in the browser, once Ledger's registry
    accepts it; until then the Ledger shows a blind-signing hash, Rabby's
    simulation is the pre-sign defence and the outcome check still catches
    a tampered minimum on chain; low priority, back of the roadmap, owner
    2026-11-10); for the engine's own
    trades the minimum is the engine's own, with no off-chain second line
    (the signer's price check of an earlier draft is dropped as a
    duplicate, 2026-11-10; the signer's section below says what it keeps).
    What a caller-supplied minimum cannot defend against is the caller
    being the attacker: a compromised executor or a stolen home key
    setting it to zero. For that the guard
    keeps an on-chain **backstop**: where the registry names a reference
    pool for the token, the effective minimum is the larger of the
    caller's and the one derived from that pool's time-weighted price less
    the band, read through a small adapter for the pool's kind, never a
    spot price, which is what an attacker moves. The rate is composed
    through the pricing path when the reference pool's quote is not the
    trade's other token (owner's question, 2026-11-10): a coin whose
    reference pool is quoted in a tokenized stock is priced as the coin's
    time-weighted price in the stock times the stock's time-weighted price
    in USDC from the stock's own reference pool, and so on to the trade's
    input token, at most two hops, each hop a free-oracle pool of a
    whitelisted token that clears its own depth floor; the route the
    trade takes is still irrelevant, only the amounts in and out are
    checked. A token whose path cannot be completed cannot be whitelisted
    (next sentence). The band applies to the composed
    rate, so it is set per token with the hops in mind. Only kinds with a **free
    oracle**, a time-weighted price the pool maintains itself as part of
    its normal operation, are built: Uniswap v3 and its forks (Aerodrome
    Slipstream, PancakeSwap v3) through `observe`; Aerodrome and Velodrome
    v2-style pools, volatile and stable, through their built-in
    observations (`quote` over the pool's samples, reserves for the depth,
    the stable curve's math for the band); Algebra pools through their
    oracle plugin. **The backstop pool is chosen at whitelisting**, by the
    owner from the app's proposal: the app scans the token's pools on the
    chain, keeps those of the free-oracle kinds, measures each one's depth
    in stable terms through the pricing path and proposes the deepest that
    clears the absolute floor; the owner confirms it in the whitelisting
    screen, and the pool and its kind are written into the registry entry
    with the whitelisting itself, so the home key sets it after the
    cooldown. **A token with no such pool cannot be whitelisted** (owner,
    2026-11-10): while the only adapters are the free-oracle kinds, a
    whitelisting must carry a backstop pool of a supported kind that
    clears the absolute floor, and the registry's add checks both on
    chain when the Delay executes it, so even a coerced whitelisting
    cannot point at a thin or unsupported pool. Everything else stays on
    the degen path, under the allowance, until it has such a pool or
    until keepers are built; the whitelisted path therefore always has a
    backstop, and the stolen-key sandwich on it is bounded by the band
    everywhere. **Re-pointing** (owner, 2026-11-10 and 2026-11-11): one
    path for every case, the home key alone, after a delay that is a
    per-chain parameter, one hour by default, with three checks the
    registry makes on chain when the re-point activates: the pool holds
    the whitelisted token against a quote with a pricing path; its depth
    clears the absolute floor in stable terms; and its oracle answers for
    a full window, which is the on-chain form of an hour of existence. No
    continuity check against the old backstop's price and no comparison
    with the old pool's depth: two liquid pools of the same token on the
    same chain cannot sit far apart, since arbitrage bots close any gap
    within blocks, so a continuity check adds nothing when both pools are
    liquid and refuses the honest fix in the one case it would bite, an
    old pool that broke; and a thinner pool that still clears the floor is
    a legitimate destination when the old one thinned, which is the usual
    reason to re-point. The delay, not two keys, because the attack a
    re-point invites fails on its own on a chain with bots: a pool of the
    attacker's own, funded past the floor with the price where they want
    it, is drained by the bots within blocks, its time-weighted price ends
    up honest over any window, and the attacker's capital goes to the
    bots, not to them; bundling does not help, since the window itself
    takes longer than a bundle. The hour is for the new pool to have a
    history before its price means anything and for the depth alert to
    reach the owner. The residual, accepted: once the new pool is active,
    an attacker who owns its liquidity can withdraw it and then drag a
    thin pool's price over a window, against bots that take their money
    on every push; the depth alert, the pause and the bots answer it, and
    the per-chain delay is the lever if a chain turns out to have no
    bots. The app alerts when a backstop pool's depth falls under the
    floor, so the owner has the hour in hand; the guard refuses a buy
    whose backstop pool no longer clears the floor until the re-point
    activates, while a sale still goes through on the wide band.
    Kinds that would need a keeper of ours (Uniswap v2 pairs through their
    cumulative accumulators against registry snapshots; Uniswap v4 pools
    through sampled observations by designated keepers) are **not built**:
    the owner expects the free-oracle kinds to cover most or all tokens of
    interest, and the keepers sit at the back of the roadmap as an option
    if practice says otherwise. A degen token has no backstop by
    construction. The checks below apply to buys and to sales alike, with
    different parameters. **Sales
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
    per pool by the owner. On every sale the caller's minimum applies and,
    where the token has a backstop, the wide band against it too, whatever
    route the caller supplies, so a thief gains nothing by routing a sale
    through a pool of their own; a degen token, which has no backstop,
    passes a sale on the caller's minimum and the floors.
  - **A swap checks the outcome, not the route.** The guard pulls exactly
    the input from its caller, the Safe, approves exactly that amount to a
    whitelisted router or aggregator, calls it with the calldata the
    caller supplies, and then requires that its own balance of the output
    token grew by at least the effective minimum (the caller's, or the
    backstop's where that is larger); everything that came back, output
    and unspent input,
    goes to the Safe, and anything short reverts. The route is the
    engine's business, so every AMM a whitelisted router reaches is
    tradable: Uniswap's Universal Router for v2, v3 and v4 (hooked pools
    included, since what matters is what came back), Aerodrome's routers
    on Base, an aggregator (1inch, Odos, KyberSwap, 0x) spanning all of
    them. Routers are a two-key whitelist. The calldata can spend only the
    amount approved for this call, the guard holds nothing between calls,
    and a route that sends the output anywhere but the guard fails the
    check.
  - **A range open** mints a single-sided position whose range lies on the
    correct side of the pool's current price (and of the backstop's
    time-weighted price, where the token has one) and within a few tick
    spacings of it, with the minimum equal to the amount offered (a range the price has
    already entered reverts), the NFT to the Safe, on any
    concentrated-liquidity manager the registry whitelists: Uniswap v3's
    and its forks' position managers (Aerodrome Slipstream among them),
    and Uniswap v4's PositionManager, whose command bytes the guard builds
    itself with the recipient pinned, so Roles never needs conditions on
    them; on v4 the guard also does the close, since a direct role call
    cannot scope those bytes. A hooked v4 pool is whitelisted for range
    orders only by two keys after a look at the hook, since a hook can
    touch position liquidity; for swaps the outcome check makes the hook
    irrelevant. Pools with no concentrated liquidity (v2 pairs, Aerodrome
    volatile and stable) have swaps only.
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
    terms through the pricing path, provided the pool is quoted in a token
    with a pricing path (a stablecoin, WETH, or a whitelisted token that
    reaches one in at most two hops; amended 2026-11-10, since the floor
    and the bucket only need a valuation, not a reserve quote) and clears
    the absolute floor. **The same allowance
    is withdrawable to any address with the home key alone (owner,
    2026-11-09):** one bucket, spent either on buys of non-whitelisted
    tokens inside the Safe or on a plain transfer out through the guard's
    `withdrawAllowance(token, amount, to)`, one Rabby signature and no
    other condition. **The withdrawal is the home key's alone** (owner,
    2026-11-10, after the tracker session's review): it sits behind a
    second role whose only member is the home Ledger, as the whitelisting
    Delay does, and the signer refuses it whatever the role says, so the
    app key has no ability to withdraw, which is requirement 3; the degen
    buy inside the Safe stays a trader-role call. The owner's reasoning: this is the one priced exit in
    the design, the number the owner is prepared to lose with a wrench at
    their head, and an attacker who has read the design knows it exists;
    if the only way to pay it out were to stand up a pool and have the
    victim buy the attacker's coin, the victim would look as if they were
    stalling while doing something slow and technical, and would be hurt
    for it. So the ceiling is easy to hand over: the runbook says pay it,
    the published stories say what the ceiling is, and nothing else is
    reachable. It also covers the degen activity the Safe can never host
    (a new chain, an NFT, an exotic protocol): withdraw the allowance to a
    hot wallet and go, and whatever comes back lands in the Safe like any
    inbound transfer. **The window is rolling and equal to the owner
    path's cooldown** (three days by default, so the allowance is once per
    three days until the cooldown is changed)**:** the registry keeps each debit with its timestamp
    and sums those younger than one cooldown, so a new debit passes only
    if the trailing sum stays under the allowance, and any window of that
    length yields at most one allowance, with no double take at a period
    boundary (a calendar reset would hand an attacker who arrives just
    before the boundary two allowances in a night; a continuously
    refilling bucket would hand them a full take plus a cooldown's refill,
    twice the allowance in a span, so neither is used); an attacker
    holding the owner for a full cooldown gets one allowance or, with the
    vault key too, a delayed withdrawal, never faster. The Delay does not
    apply to the allowance, by design. **What still holds on a degen
    buy**, which loses only the backstop and the whitelisted path's
    notional: the caller's minimum, shown to the owner before signing and
    verified by the signer for the engine; the outcome rule, so the bought
    token lands in the Safe and nothing leaves but the input; the absolute
    floor, so the pool is quoted in a stablecoin or WETH and holds real
    money; the bucket; and the pool of first buy recorded as where the
    token sells. Its remaining value over a withdrawal to a hot wallet is
    custody: the position sits in the Safe from day one, and a later
    whitelisting gives it the full path and a backstop. Not a priority
    (owner, 2026-11-09), built when convenient. The withdrawal needs
    nothing but the bucket. **The allowance is a registry parameter**
    (owner, 2026-11-10): lowering it is a trader-role call, at once, the
    app key included (the registry refuses a value that is not lower);
    raising it is two keys through the Delay. The owner's reasoning for
    letting any role member lower it: a visible zero tells a wrench
    attacker that nothing moves without the vault key and the delay, which
    is legible and safe; the hazard to avoid is an allowance that is not
    zero but cannot be paid. The pause follows the same logic: a
    trader-role call that stops trading but never `withdrawAllowance`, so
    that while the allowance is not zero it stays payable, and never the
    guard's range close either (the tracker session's review, 2026-11-11):
    on v4 the guard does the close, and a filled range that cannot be
    collected un-fills if the price comes back, which would turn a pause
    into a reversed sale; a close has no price exposure and delivers only
    to the Safe, the same argument that exempts the payout. Selling any token the Safe holds back
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
    position later if wanted. Launches on v2 pairs, on v4 (the launchpads'
    hooked pools included) and on other AMMs are reachable like any other
    pool: the route is the engine's, the outcome check is the guard's, and
    a young pool with no reference passes on the floors and the allowance.
  - **The registry holds the policy the guard reads (option B, owner,
    2026-11-05).** One contract per chain, owned by the funded Safe (so
    two keys through the Delay, or three direct), holding: the whitelisted set, and
    for every token its backstop pool and kind, or none, and for a degen
    token the pool of its first buy; per pool, the
    deviation band, the tick distance, the depth multiple, the absolute
    floor, the pricing path and the sell band, each with a default applied
    at whitelisting, the defaults set when the registry is deployed with
    the Safe and changed, like every parameter of a whitelisted pool in
    either direction, only by two keys through the Delay (owner,
    2026-11-10, the bands in particular); two running notionals in stablecoin terms through the pricing
    path, rolling windows of the owner path's cooldown, one for the
    whitelisted path, debited with the value of what is **bought** on
    every swap and open, and one for the degen path, debited with every
    non-whitelisted buy and every `withdrawAllowance`; and a **pause flag** (owner,
    2026-11-05): any role member sets it with a role call, at once, and
    the guard refuses every call but `withdrawAllowance` and the range
    close while it is set (an allowance that is not zero must stay
    payable, and a filled range must stay collectable; owner, 2026-11-10
    and 2026-11-11); clearing it is the
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
    an add with the default parameters; after the cooldown (configurable
    by the Delay's owner; **three days by default**, the owner's number)
    anyone executes it. Loosening a pool's
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
  - **Every AMM, through adapters (amended 2026-11-09).** The guard
    separates where it checks from where it trades. Execution is any
    whitelisted router with an outcome check, so no AMM is out of reach
    and a new one costs a router entry in the policy, not code. The
    backstop needs a read adapter per pool kind with a free oracle, three
    in all, each a few dozen lines; kinds that need a keeper are deferred.
    What the guard cannot do is conjure a trustworthy price for a pool
    with no history: there the caller's minimum is the only price check,
    and the floors and the notionals bound the trade.
  - **Cost:** custom Solidity in the trust path, the guard with its three
    read adapters, the registry and the approver, together a few hundred
    lines, so an audit before real
    funds, and the fork suite testing their reverts as it tests the
    policy's. **Build notes for that audit** (the tracker session's
    review, 2026-11-10): a reentrancy lock around the caller's calldata;
    the router approval reset to zero after every call; native value
    refused unless the route needs it; routers that pull through Permit2
    (the Universal Router does) served by the guard granting Permit2 a
    per-call allowance that expires with the block; the rolling windows
    kept as hourly buckets with a running total and lazy expiry (each call
    expires the buckets that aged out and adjusts the total), so a call
    reads a handful of slots whatever the bucket size, and the hour costs
    nothing over a coarser bucket (owner's question, 2026-11-10). The time-weighted price is itself movable on
    a thin pool over several blocks, so the band is a cost imposed on a
    compromised engine, not a wall; with the depth floor that cost is what
    it takes to move a deep pool.
- **Deposits to the owner's Kraken account are a home-key action (owner,
  2026-11-06).** Kraken's global settings lock is on for the account, and
  the owner has confirmed only two withdrawal destinations there: the
  owner's bank account and the Safes. With the lock on, a new destination
  cannot be added without the lock's own unlock wait, so funds that reach
  Kraken can only come back to the Safes or go to the bank. That makes a
  deposit to Kraken a transfer into another vault of the owner's, with a
  delay the owner configured rather than one of ours, and so a role call:
  the one address outside the Safe system the role may send to, a
  trader-role call, the app key included (owner, 2026-11-10: a deposit
  to Kraken is recoverable, and the engine may want to make one). How:
  the registry holds, per chain, the Kraken deposit address for the
  owner's account, set and changed by two keys through the Delay, and a
  **Kraken token list** per chain, USDC and ETH by default, extended by
  the home key alone and at once, since its purpose is to stop a mistake
  (a token Kraken does not accept on that chain is sent and never
  credited) rather than a theft; the guard gains a `sendToExchange`
  function that pulls a token on that list (the approver grants the
  approval; WETH is unwrapped, since Kraken takes ETH as native on the
  L2s) and transfers it to that address, nothing else; no price check and
  no cap, since the destination is the owner's. Only on chains Kraken
  accepts (USDC and ETH on Ethereum, Arbitrum, Base, Optimism, Polygon at
  the time of writing; nothing on Robinhood Chain, so those funds bridge
  to the Arbitrum Safe first); the engine or the owner checks Kraken's
  current networks before each pin or list entry.
  Operational rules that carry the whole safety of this path: the lock
  stays on, with a long unlock wait; the destination allowlist at Kraken
  holds only the bank and the Safes; a pinned address is proved with a
  small test deposit before the policy change that pins it; and the
  monitoring step alerts on every deposit to it. The must-fail list gains
  a transfer to any other external address. Under a wrench attack the
  attacker can push the book to Kraken and no further than Kraken's own
  controls allow, which is the bank or the Safes; under a stolen home key
  the funds are recoverable by the owner through Kraken.
- **CLMM range orders (amended 2026-11-02; opened through the guard since
  2026-11-04).** The execution engine rests its limit orders as
  single-sided concentrated-liquidity positions: a range entirely above the
  price sells through it, one entirely below buys through it, and the
  position earns the pool fee from the flow that fills it. The open goes
  through the guard (above), which mints the position to the Safe on any
  position manager the registry whitelists (Uniswap v3's
  NonfungiblePositionManager; Aerodrome Slipstream on Base; PancakeSwap
  v3; Camelot on Arbitrum; Uniswap v4's PositionManager, open and close
  both through the guard). The close has
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
    cooldown (configurable by the Delay's owner, so by all three keys;
    **three days by default**, owner, 2026-11-10), and anyone executes it
    after.
    The Delay's owner is the funded Safe itself, so shortening the
    cooldown or skipping an entry takes all three keys. A coerced two-key
    withdrawal needs the attacker to hold the owner through the whole
    cooldown while the entry sits visible on chain (the monitoring step);
    an owner who is free during it moves the funds with three keys first.
    A lost key leaves this path intact with the other two.
  Rehearse the skip and the expiration on a fork. Role calls bypass the
  Delay by design.
- **Swaps go through the guard, on every chain, through any whitelisted
  router.** On Robinhood Chain that is Uniswap's Universal Router (v2, v3
  and v4 pools) and whatever aggregator covers the chain; on Base,
  Uniswap's and Aerodrome's; v4 pools are in from the start, their
  reference price from a sibling pool, from sampled observations or none,
  as the registry says per pool. Amended 2026-11-04: the CoW presets are out of the role. The
  engine does not use CoW (a solver fills a resting order on its own terms,
  a range order fills from the pool's flow and earns the fee), the presets
  were one more code path on the Safe to fork-test, and the ERC-1271 path
  they rely on is where the June bug lived. They can come back as a policy
  change if the owner wants CoW for manual swaps through Pilot. Before this
  amendment the design kept custom Solidity out until a measurement said
  v4 mattered; the guard is justified by the residual it closes (the
  app's key, below), v4 is in through the outcome check, and the engine's
  v3-versus-v4 measurement (portfolio-tools ADR-014) is fill-quality
  information for the engine, with no custody decision waiting on it.
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
  the owner's own agent and the app's KMS key. API wallets cannot
  withdraw by Hyperliquid's own rules; the engine signs orders with the
  KMS key off-chain exactly as before. Three facts bind the owner's agent
  (researched 2026-11-12): an agent is only a signer, and every read needs
  the master's address, so whatever interface the owner trades in must
  read the Safe and sign with the agent; the agent must be a software
  key, held by a frontend, a phone or our app, because Rabby and MetaMask
  reject the typed-data domain Hyperliquid uses for orders, so the Ledger
  cannot be the trading agent; and the unnamed agent slot is the one
  Hyperliquid's own frontend uses, approving a new unnamed agent
  deregisters the previous one, and agents expire, so the CoreWriter
  approval is redone on a schedule and carries a name where the interface
  allows. Which interface the owner trades in is a testnet question
  (roadmap step 4): Hyperliquid's own interfaces are built for an agent
  the master approves in the browser, which a Safe cannot sign, so each
  option has to be tried.
- **Deposits are role calls with the destination in the signed calldata**,
  never a transfer to an address that is not the Safe's own, **under a
  Hyperliquid deposit allowance per window** (owner, 2026-11-10). The
  allowance exists because Hyperliquid trading has no guard: a market
  sell on a thin book (the HIP-3 perps, most spot tokens) fills against
  whatever bid is there, and a bid placed by the attacker's own account
  turns the Safe's Hyperliquid balance into an exit, under a wrench
  (the owner made to deposit and trade) and under a remote compromise of
  the executor alike; on a liquid perp a market order walks the book
  first, so the loss goes to other traders, which is why the thin markets
  are the ones that matter. With the allowance the bound in both cases is
  the Hyperliquid balance plus one allowance per window. Same bookkeeping
  and controls as the degen allowance: a rolling window equal to the owner
  path's cooldown, lowering a trader-role call, raising two keys through
  the Delay; withdrawals back to the Safe uncapped. Rails, best first:
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
- **The signer service** (build step 5; slimmed 2026-11-10 after the
  owner's question of what it buys over the chain's own checks). The
  engine never signs: it sends the signer an intent (chain, account,
  target, calldata, value, gas, fees, nonce, attempt id) and gets back a
  signed transaction or a refusal. The signer is a separate process in
  this repository and the only identity with permission to ask Cloud KMS
  to sign. Its EVM-side checks of the first draft (an allowlist, caps, a
  price check) duplicated Roles, the registry's notionals and the guard's
  backstop and are dropped: on EVM the chain is the policy. What it keeps,
  because nothing else can do it: **never send twice**, one nonce per
  attempt, a replacement only at the same nonce with higher fees, the
  same intent returning the same signature, since the process that holds
  the key is the right place to stop a retrying executor from double
  sending; **`withdrawAllowance` refused outright**, whatever the role
  says; and **the order policy for venues with no on-chain guard**,
  Hyperliquid first: price within a band of the mark or oracle price,
  size caps per order and per window, a market allowlist, reduce-only
  rules, which is what bounds a remote compromise of the executor on
  Hyperliquid, and later Solana. **The freeze needs no feature:** the
  signer can sign only because its service account holds one IAM
  permission on the KMS key, and the owner, with a cloud identity the app
  never has, can revoke that permission or disable the key version from
  the console in seconds, stopping every signature the app's key could
  make on every chain and on Hyperliquid, reversibly; it is a runbook
  entry, the off-chain and instant counterpart of the on-chain pause.
  **The key cannot be extracted** by anyone: Cloud KMS keeps it in a
  hardware security module and signs on request, so what exists outside
  is a permission to ask; under a wrench the owner can be made to ask,
  which is why the deposit allowances, not the signer's policy, are the
  bound in that case, and the policy bounds the remote case. Optional
  hardening, in the roadmap's ideas: run the signer in Confidential Space
  and bind the KMS permission to the attested container image, so that
  even the project's own cloud credentials cannot call the key from
  anywhere else. The owner's own trades through Rabby never touch the
  signer.

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
  outside the guard, a non-whitelisted token) reverts, and the guard's own reverts hold (a swap past
  the band, an open against a moved spot, a trade larger than the pool's
  depth allows, a pricing hop under its floor, a buy of a non-whitelisted token
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
   through a hop under its floor reverts; a buy of a non-whitelisted token
   passes under the degen allowance in a young pool over the floor, and
   reverts above the allowance, under the floor, or quoted in another
   non-whitelisted token; a `withdrawAllowance` to any address passes
   within the bucket and reverts above it, the bucket rolling over one
   cooldown; a buy whose backstop pool is under the floor reverts; a buy
   below the caller's minimum reverts), the registry and the whitelisting Delay (an add queued by
   the home key executes after the cooldown and not before; the vault key
   skips it; a home-key call that loosens a parameter or raises a notional
   reverts; a removal by the role is immediate), the approver (refuses any
   caller but the guard and any spender but the guard; a degen token
   bought in one block is sold in the next, through the pool it was bought
   in, and a sale routed to another pool reverts), the pause (set by a
   role call, the guard refuses everything; cleared only through the
   Delay), the close permission on the
   position manager (a partial fill decreased and collected to the Safe, a
   reload cycle as one MultiSend), the bridge permission, the Kraken
   deposit (a `sendToExchange` to the pinned address passes; to any other
   address reverts; changing the pinned address by the role reverts), and
   the must-fail list, with a
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
  the degen allowance, withdrawable to any address with the home key, one
  allowance per cooldown window, which the owner hands over under duress
  by design. A
  coerced whitelisting costs the attacker the cooldown with the owner in hand,
  visible on chain, and then at most the whitelisted path's per-period
  notional in a pool that clears the floors. It can send the book to the
  owner's Kraken account, where Kraken's global settings lock allows
  withdrawals only to the bank or back to the Safes. It cannot send
  anything elsewhere.
- **A stolen home key**, with no owner to coerce, takes the degen
  allowance per cooldown window, by a plain withdrawal. The residuals, each accepted by the owner
  (2026-11-05): after the three-day whitelisting cooldown, the whitelisted path's
  per-period notional into a pool of the thief's own, unless the vault key
  skips the whitelisting first; and a sandwich of the Safe's
  own sales in their real pool by a thief who also has the capital to
  move that pool (which they get back), bounded per sale by the 30% sell
  band and by what moving a deep pool's time-weighted price costs. A
  deposit to Kraken moves funds the thief cannot reach into the owner's
  own exchange account. The pause flag ends all of it the moment
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
  delayed and visible, not refused; a pool without a trustworthy price history trades with no price check,
  bounded by the floors and the notionals; the engine's reload
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
