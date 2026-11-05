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
queue a listing (story 4), and the app key never signs anything outside
the signer service. Agents never hold any key and never touch mainnet.

## Funds and keys

1. **If any of our custom code, or a Zodiac module, is buggy, I can always
   withdraw all funds to another address with my keys.** All three keys,
   at once: the funded Safe's own threshold is 3 of 3 and no module sits
   between the three keys and the funds. Residual: none. A lost key
   leaves story 10.
2. **Withdrawing to another address needs two keys plus a configurable
   delay.** Two keys sign on the fundless 2-of-3 Safe, which queues on the
   funded Safe's Delay; the cooldown passes; anyone executes. The Delay is
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
   approval at sale time; a wide band (30% by default) over a short window
   against the pool's time-weighted price; the sale goes through the pool
   the registry knows for the token, and a pool too young to have a
   time-weighted price passes on that rule alone. Residual (accepted): a
   crash faster than the window holds a sale back for at most the window;
   a thief with the home key and the capital to move the pool can
   sandwich the Safe's own sale for up to the band.
5. **When I want to buy a token I already bought in the past, I can do it
   at once with the home key.** A listed token: the listed path. A token
   bought under the degen allowance and never listed: the degen path
   again, under the allowance, until it is listed. Residual: none.
6. **I can swap any listed token for any other listed token at once with
   the home key.** The listed path, debiting the listed notional with the
   value bought; buys of the reserve set (stablecoins, WETH) count for
   nothing. Residual: the listed notional per period, set generously.
7. **When I want to buy a new token, I can do so at once with the home key
   up to a value per day; changing that allowance needs two keys plus the
   delay.** The degen allowance in the registry, only in pools quoted in a
   stablecoin or WETH that clear the absolute floor; the registry is owned
   by the funded Safe, so the allowance changes through story 2's path.
   Residual: the allowance per period is what a coercion or a theft
   yields (story 9).
8. **I can bridge any held token to my own Safe on another chain with the
   home key.** The guard's bridge: any token, listed or not, the
   destination pinned to the Safe's own address on the other chain, no
   cap. The destination is a parameter of the bridge call, so it is in
   the transaction the Safe signs and the guard sets it to its caller;
   bridges whose recipient lives only in an off-chain quote are not role
   calls. Residual: none.

## Listing

9. **When I plan to buy a new token, I can list it with the home key alone
   from the app's UI, and it takes effect after a delay.** The app builds
   the call, the home Ledger signs it through Rabby, the listing Delay
   holds it for three days, the app's worker executes it; the vault key
   can skip it. A listing applies default parameters; loosening them is
   two-key through the delay; removal is immediate. Residual (accepted):
   whoever holds the home key can list a coin of their own and, after the
   cooldown, buy it on the listed path up to that path's per-period
   notional, unless the listing is skipped first (story 12).

## Attack bounds

10. **An attacker with the home key can take out the degen allowance once
    per day and nothing else.** The role path has no exit: every swap,
    open, bridge and Hyperliquid transfer delivers to the Safe's own
    address, and approvals go only to the guard. Residuals, each
    accepted: story 9's listing attack after three days; story 4's
    sandwich on sales. Story 11 ends both once noticed.
11. **I can halt all trading at once with the home key; resuming needs two
    keys plus the delay.** The registry's pause flag, set by any role
    member, cleared through story 2's path. The guard refuses every call
    while it is set. Residual: none.
12. **I hear within minutes of any listing queued, any Delay entry and any
    trade I did not make.** The monitoring step of the roadmap; its shape
    is to be decided with the owner. Stories 2, 9 and 10 depend on it.

## Hyperliquid

13. **I can deposit to Hyperliquid with the home key, with no hop and no
    cap.** The Safe itself is the Hyperliquid account: the same Safe at
    the same address on HyperEVM, where the whole Zodiac stack is
    deployed, controls its HyperCore account through CoreWriter, scoped
    by Roles with no custom code. A deposit is a CCTP burn from the
    Arbitrum Safe with the recipient pinned to the Safe in the signed
    calldata (or a native-bridge deposit, credited to the sender), so it
    is a transfer between the owner's own addresses. Decided 2026-11-05,
    validated thoroughly on testnet before any live funds (ADR-001,
    Hyperliquid). Residual: none on the deposit itself; story 14 carries
    the account's exposure.
14. **Trading on Hyperliquid needs no owner key; withdrawing back to my
    wallet needs the home key alone; leaving the Safe system needs two
    keys plus the delay.** API wallets (the owner's browser agent, the
    app key), approved through CoreWriter by two keys, trade and cannot
    withdraw by Hyperliquid's own rules. A withdrawal back to the wallet
    is a class transfer, a send asset to the USDC system address and a
    CCTP burn pinned to the Arbitrum Safe, all role calls between the
    owner's own addresses; after that, story 2. The account is never
    converted into a native multi-sig (Hyperliquid's rule: CoreWriter
    does not work for multi-sig users), and it has no key for the 2025
    conversion phishing to sign with. Residual: the account's funds depend
    on CoreWriter and on the HyperCore-to-HyperEVM USDC link, both
    Hyperliquid-operated, so the balance there is kept at trading size;
    CoreWriter actions are fire-and-forget, so the engine confirms each
    one through the precompiles and treats "not landed" as retryable.

## Operations

15. **I can replace a contract (the guard, the registry, the approver, a
    router that changed) with two keys plus the delay.** The addresses
    are immutables in the guard and conditions in Roles, so a replacement
    is a new deployment plus a policy change through story 2's path.
    Residual: none.
16. **The policy in force is what the fork suite proved.** Every allowed
    call passes, every must-fail call reverts, on every chain, on every
    change and on a schedule; a policy change ships with its tests.
