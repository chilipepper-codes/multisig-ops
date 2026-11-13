# User stories

The spec the custody layer is measured against (owner, 2026-11-05). Each
story names the mechanism in [ADR-001](decisions/001-custody-design.md)
that satisfies it and the residual the owner accepted. The fork suite's
allowed list and must-fail list are derived from these; a story without a
test is not satisfied.

**Keys assumed.** Three owner keys, all the owner's: the **home key** (a
Ledger, used through Rabby), the **vault key** (in a bank vault) and a
**third key** kept off-site. The **app key** is the execution engine's
Cloud KMS key. Home key and app key are interchangeable below, both being
members of the trader role, with two exceptions: only the home key can
queue a whitelisting (story 4), and the app key never signs anything outside
the signer service. The owner keeps all three keys at home through the testing period and moves them to their locations before real money goes in (owner, 2026-11-12). Agents never hold any key and never touch mainnet.

## Funds and keys

1. **If any of our custom code, or a Zodiac module, is buggy, I can always
   withdraw all funds to another address with my keys.** All three keys,
   at once: the funded Safe's own threshold is 3 of 3 and no module sits
   between the three keys and the funds. Residual: none. A lost key
   leaves story 10.
2. **Withdrawing to another address needs two keys plus a configurable
   delay, three days by default.** Two keys sign on the fundless 2-of-3
   Safe, which queues on the funded Safe's Delay; the cooldown passes;
   anyone executes. The same cooldown sets the degen allowance's window
   (story 7). The Delay is
   owned by the funded Safe, so shortening the cooldown or skipping an
   entry takes all three keys. Residual: a coercer who holds the owner
   through the whole cooldown, visible on chain (story 12).
3. **If I lose any one key, I can still withdraw everything.** The other
   two keys use story 2's path after the cooldown. Residual: the direct
   3-of-3 path is gone until the lost key is replaced, which is itself a
   two-key, delayed change.

## Trading with the home key alone

4. **When I want to sell a token I hold, I can always do so at once with
   the home key.** Through the guard, with the approver granting the
   approval at sale time, through any whitelisted router; my own minimum,
   shown before I sign, always applies, and where the token has a backstop
   pool a wide band (30% by default) over a short window against its
   time-weighted price applies too; a degen token, which has no backstop,
   sells on my minimum and the floors. Residual (accepted): a crash faster than the
   window holds a sale back for at most the window; a thief with the home
   key and the capital to move the pool can sandwich the Safe's own sale
   for up to the band where there is a backstop, and for the position
   where there is none.
5. **When I want to buy a token I already bought in the past, I can do it
   at once with the home key.** A whitelisted token: the whitelisted path. A token
   bought under the degen allowance and never whitelisted: the degen path
   again, under the allowance, until it is whitelisted. Residual: none.
6. **I can swap any whitelisted token for any other whitelisted token at once with
   the home key.** The whitelisted path, debiting the whitelisted notional with the
   value bought; buys of the reserve set (stablecoins, WETH) count for
   nothing. Residual: the whitelisted notional per period, set generously.
7. **When I want to buy a new token, or move money out to any address for
   a degen play, I can do so at once with the home key, up to the
   allowance per window; lowering the allowance is a one-key action for
   the trader role, at once; raising it needs two keys plus the delay.** One bucket in the registry: buys of non-whitelisted tokens
   inside the Safe (only in pools quoted in a token with a pricing path
   that clear the absolute floor), a trader-role call, and plain withdrawals to
   any address through the guard, one signature, no other condition, the
   home key's alone through a role of its own, which the app key is not
   in and the signer refuses outright. The window is rolling and
   equal to the owner path's cooldown, so any window of that length yields
   at most one allowance. The registry is owned by the funded Safe, so the
   allowance changes through story 2's path. Residual, by design: the
   allowance per window is what a coercion or a theft yields (story 10),
   and under duress I hand it over without stalling; it is the published
   ceiling.
8. **I can bridge any held token to my own Safe on another chain with the
   home key.** The guard's bridge: any token, whitelisted or not, the
   destination pinned to the Safe's own address on the other chain, no
   cap. The destination is a parameter of the bridge call, so it is in
   the transaction the Safe signs and the guard sets it to its caller;
   bridges whose recipient lives only in an off-chain quote are not role calls. A bridge that changes the token on the way, USDG off Robinhood Chain into USDC on the Base Safe through Across, carries an output bound the guard checks near one to one, and refunds to the Safe (2026-11-13); a bundled swap-and-bridge like Jumper is not a role call. Residual: none.

## Whitelisting

9. **When I plan to buy a new token, I can whitelist it with the home key alone
   from the app's UI, and it takes effect after a delay.** The app builds
   the call, the home Ledger signs it through Rabby, the whitelisting Delay
   holds it for three days, the app's worker executes it; the vault key
   can skip it. A whitelisting applies default parameters and the backstop
   pool the app proposed and I confirmed, the deepest pool with a free
   oracle that clears the absolute floor; a token with no such pool
   cannot be whitelisted and stays under the degen allowance, and the
   registry checks the pool on chain when the whitelisting executes;
   re-pointing the backstop is mine with the home key after a short
   delay (a per-chain parameter, one hour by default), the registry
   checking on chain that the new pool holds the token against a priceable
   quote, clears the depth floor and has a full window of oracle history;
   loosening parameters is two-key through the delay; removal is
   immediate. Residual (accepted):
   whoever holds the home key can whitelist a coin of their own and, after the
   cooldown, buy it on the whitelisted path up to that path's per-period
   notional, unless the whitelisting is skipped first (story 12).

## Attack bounds

10. **An attacker with the home key can take out the degen allowance once
    per cooldown window, by a plain withdrawal, and nothing else.** That
    is the published ceiling and it is easy to pay. The role path has no
    other exit: every swap,
    open, bridge and Hyperliquid transfer delivers to the Safe's own
    address, approvals go only to the guard, and the one external
    destination, the owner's Kraken account (story 17), returns funds only to the bank or the Safes; the app key's gas (story 20) goes to the app key's own address, a week's allowance at most. Residuals, each accepted: story 9's
    whitelisting attack after three days; story 4's sandwich on sales.
    Story 11 ends both once noticed.
11. **I can halt all trading at once with the home key; resuming needs two
    keys plus the delay.** The registry's pause flag, set by any role member, cleared through story 2's path. Setting it also sets the degen and the Hyperliquid deposit allowances to zero (owner, 2026-11-13), so a thief with the home key takes nothing more and a wrench attacker sees the legible zero of story 7; the guard refuses every call while it is set except the close of a range order, so a filled range can still be collected before the price comes back. Raising the allowances again rides the entry that clears the pause. Residual: none.
12. **I hear within minutes of any whitelisting queued, any Delay entry and any trade I did not make.** The app's (owner, 2026-11-13): a scheduled job reads the Delays, Roles, the registry and the guard every few minutes and alerts on the dashboard and by push for anything I did not start; one independent channel outside the app, set up once by me on the Safe and module addresses, backs it, since a compromised app hides its own alerts. This repository keeps the event catalogue the job reads and the runbook for the independent channel. Stories 2, 9 and 10 depend on it.

## Hyperliquid

13. **I can deposit to Hyperliquid with the home key, with no hop, under
    an allowance per window.** The Safe itself is the Hyperliquid account: the same Safe at
    the same address on HyperEVM, where the whole Zodiac stack is
    deployed, controls its HyperCore account through CoreWriter, scoped
    by Roles with no custom code. A deposit is the guard's `depositToHyperliquid`: a CCTP burn from the Base Safe with the recipient pinned to the Safe by the guard, which debits the allowance and refuses while paused (amended 2026-11-13), so it is a transfer between the owner's own addresses that the registry sees. Decided 2026-11-05,
    validated thoroughly on testnet before any live funds (ADR-001,
    Hyperliquid). The deposit allowance has the same window and controls
    as the degen one, because Hyperliquid trading has no guard: on a thin
    market a sale into a bid the attacker placed from their own account
    would turn the balance into an exit, under a wrench or a remote
    compromise alike. Residual: the Hyperliquid balance plus one allowance
    per window; story 14 carries the account's exposure.
14. **Trading on Hyperliquid needs no owner key; withdrawing back to my
    wallet needs the home key alone; leaving the Safe system needs two
    keys plus the delay.** API wallets (the owner's browser agent, the
    app key), approved through CoreWriter by two keys, trade and cannot
    withdraw by Hyperliquid's own rules. A withdrawal back to the wallet
    is a class transfer, a send asset to the USDC system address and a
    CCTP burn pinned to the Base Safe, all role calls between the
    owner's own addresses; after that, story 2. The account is never
    converted into a native multi-sig (Hyperliquid's rule: CoreWriter
    does not work for multi-sig users), and it has no key for the 2025
    conversion phishing to sign with. Residual: the account's funds depend
    on CoreWriter and on the HyperCore-to-HyperEVM USDC link, both
    Hyperliquid-operated, so the balance there is kept at trading size;
    CoreWriter actions are fire-and-forget, so the engine confirms each
    one through the precompiles and treats "not landed" as retryable.

## Exchange

17. **I can deposit to my Kraken account with the home key alone, and so
    can the app.** The guard's `sendToExchange` transfers a token on the
    registry's Kraken list for that chain (USDC and ETH by default,
    extended by the home key at once, to stop a mistake rather than a
    theft) to the Kraken deposit address the registry pins for that
    chain, set by two keys through the delay; no cap, since Kraken's global settings lock is on
    and the account's withdrawal destinations are only my bank and the
    Safes, so a deposit is a move into another vault of mine. Only on
    chains and for tokens Kraken accepts. Residual: the path is as safe as
    Kraken's lock and allowlist, which I keep on with a long unlock wait,
    and a pinned address is proved by a small test deposit before it is
    pinned; a wrench attacker can push the book to Kraken and no further.

## Venues

18. **I can trade any pool on any AMM or aggregator the policy
    whitelists, with the home key: Uniswap v2, v3 and v4, Aerodrome on
    Base, others as they come.** The guard checks the outcome of a swap
    against the token's reference pool, not the route, so the route is the
    engine's; a new AMM is a router entry in the policy, a new pool kind a
    small read adapter with its fork tests. Range orders work on every
    concentrated-liquidity manager the policy whitelists, v4's included.
    Residual: my own minimum always applies; a whitelisted token always
    has a backstop, since whitelisting requires a reference pool with a
    free oracle; a token without one stays on the degen path and trades
    on my minimum, the floors and the allowance alone.
19. **Before I sign a trade, I see the amounts I expect to receive and the
    tolerance, and the trade reverts if the Safe gets less.** The app
    looks prices up, shows the amounts, and passes the minimum to the
    guard, which enforces it on the outcome. The amounts I see come from
    the app: a home-key trade is a role call signed by my Ledger's own
    account and the balances change on the Safe, so Rabby's simulation
    shows no amount for me, and with the contracts unverified it decodes
    no parameters. The app simulates the call at the Safe's level and shows me the Safe's balance changes before I sign, as Rabby does for my own wallet; the bar is Rabby with a Ledger, blind-signed, and this meets it (owner, 2026-11-12). I trust the app's amounts after the testnet soak and small live amounts, Zodiac Pilot gives a Safe-level simulation where
    it supports the chain, and a clear-signing descriptor would put the
    input and the minimum on the Ledger's screen once Ledger's registry
    accepts it (low priority). For the engine's own trades the minimum is
    the engine's own; the signer does not second-guess it. Residual: a
    minimum is as honest as whoever sets it, which is why the backstop
    exists for a stolen home key or a compromised executor, and the
    allowances bound what either can do where there is no backstop.

## Operations

15. **I can replace a contract (the guard, the registry, the approver, a
    router that changed) with two keys plus the delay.** The addresses
    are immutables in the guard and conditions in Roles, so a replacement
    is a new deployment plus a policy change through story 2's path. Residual: none.
16. **The policy in force is what the fork suite proved.** Every allowed
    call passes, every must-fail call reverts, on every chain, on every
    change and on a schedule; a policy change ships with its tests.
20. **The app key never runs out of gas unnoticed, and only I can refill
    it.** The Safe holds some native for gas, which I keep topped up by
    hand like any of my wallets. The app warns me when the app key's
    balance is low, when I place an order, on the dashboard and through
    the monitoring channel, and I top it up from the app with the home
    key: a plain native transfer to the app key's address, scoped in my
    own role with a weekly allowance the contract checks (ADR-001, the
    app's key). The app key cannot call it, so the app never pays itself
    (owner, 2026-11-13). Raising or lowering the allowance, or re-pinning
    the address, is story 2's path. Residual: a week's gas allowance per
    chain, added to what a stolen home key takes; nothing for a
    compromised app.
21. **I can create the whole system on a new chain from the app, with my
    three keys at hand, and check it before I fund it.** The app drives
    multisig-ops' versioned setup plan step by step, each step one
    signature with the key it names; the Safes are created at their
    final thresholds, and enabling the modules and applying the policy is
    one 3-of-3 transaction per chain. At the end the app reads the
    deployed state back and runs the must-fail list on a fork of the new
    Safe, and I compare the owners it shows with my own copy of the
    addresses before anything moves in. Residual: a compromised app can
    propose a wrong input, the owner addresses first of all; the read-back
    against my own copy is what catches it, and the privileged setup
    calls go to verified Safe and Zodiac contracts that Rabby and the
    Ledger decode.
