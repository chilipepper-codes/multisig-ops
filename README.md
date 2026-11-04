# multisig-ops

The custody layer for a trading book: policies as code for Safe + Zodiac Roles
on EVM chains, the Hyperliquid native multi-sig tooling, the hardware-held key
the portfolio tracker's execution engine trades with, and the fork test suite
that proves what each policy allows and refuses.

**Start here:** [`docs/decisions/001-custody-design.md`](docs/decisions/001-custody-design.md)
— the design, what it meets, what it does not, and what is verified before any
real funds move — then [`docs/roadmap.md`](docs/roadmap.md) for the build
order and what is queued without a decision (Solana).

## Rules

- **No secrets in the repo.** Keys live in hardware (Ledgers, Cloud KMS);
  nothing here holds or derives one.
- **Agents never hold any key and never run against mainnet.** Fork and
  testnet only. Nothing in this repository places a real order or moves real
  funds; the owner does that, by hand, with the owner's keys.
- **A policy change ships with its fork tests**: every allowed call passes, the
  must-fail list reverts, before and after.

## Layout (planned)

- `docs/decisions/` — ADRs (this is the canon; design rationale lives here).
- `policies/` — the Roles policies per chain, as code.
- `contracts/` — the guard (ADR-001): the one contract in front of every
  swap and range-order open, stateless and unowned, with its fork tests.
- `scripts/` — Safe deployment at the same address per chain, policy apply
  and verify (reads the modifier's own events; no subgraph needed), the
  Hyperliquid multi-sig actions.
- `signer/` — the KMS signer service the execution engine calls.
- `test/` — the fork suite.
