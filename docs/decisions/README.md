# Architecture Decision Records

Short records of decisions that are expensive to revisit — the "why" that stops
future contributors (human or agent) from relitigating settled questions. Add one
when a change makes a choice that constrains future work; amend (don't rewrite)
when a decision is superseded. Format: Context / Decision / Consequences.

| ADR | Decision |
|---|---|
| [001](001-custody-design.md) | Wrench-resistant self-custody for the trading book: one funded Safe per EVM chain with Zodiac Roles (and Delay, if chosen) on it, a stateless guard contract in front of every swap and range-order open, Hyperliquid's native multi-sig with API wallets for trading, a Cloud KMS key for the app's executions; Solana proposed, not decided |
