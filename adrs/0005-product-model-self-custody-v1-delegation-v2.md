# Architecture Decision Record (ADR) 0005 -- Product model: self-custody v1 (moneymentum) and trust-minimized delegation v2 (fund)

- Status: **proposed** (awaiting owner ratification)
- Date: 2026-07-06
- Issue: none yet -- this ADR sets the framing that later issues hang off; open
  a tracking issue in `dataclique/fund` if the owner ratifies the direction.

Reframes [ADR 0002](0002-tiered-off-solana-nav-inclusion.md),
[ADR 0003](0003-permissionless-nav-attestation.md), and
[ADR 0004](0004-nav-accounting-architecture.md): those design the
trust-minimized inclusion of off-Solana venue value into a redeemable share
price. This ADR places that whole body of work in **v2** (the
trust-minimized-delegation product this repo exists to build) and records that
it is **not a launch blocker** for the near-term product, which ships without a
vault. It does not modify their content; it fixes where they sit in the
sequence.

## Context

`moneymentum` is a derivatives-first portfolio tool: the factor analytics, the
screener, and the rebalancer are designed to be used **with** perpetual futures
and options (Hyperliquid perps, Derive options are the deepest venues). Any
design that drops derivatives to make some other property easier defeats the
product's reason to exist. An earlier framing in this repo's roadmap discussion
proposed moving perps to a Solana-native venue (Drift / Jupiter) so a pooled
vault could be fully trust-minimized; that trades away the derivatives depth
that is the product, so it is rejected here.

Three properties were being conflated as if they were one "decentralization"
axis. They are independent:

- **A -- execution authority.** Who signs the trade? Either **self-signed** (a
  wallet connected to the frontend; no automation) or **delegated** (a Turnkey
  policy wallet or an on-chain keeper signs within bounds; this is what enables
  auto-rebalancing).
- **B -- whose money.** Either the signer's **own** capital (a separately
  managed account; no shares, no pooling) or **pooled** third-party capital
  (limited partners deposit, receive share tokens, redeem -- this is what needs
  the vault program, or a custodial ledger).
- **C -- venue / tier.** Where the position lives and how verifiable it is:
  Tier-1 Solana-native (natively readable, oracle-priced), Tier-2 off-Solana
  with a consensus read (Hyperliquid via HyperEVM precompiles + Wormhole
  Queries), Tier-3 attested-only (Derive, any centralized exchange (CEX)).

Two facts fall directly out of separating the axes:

1. **The vault program does not provide auto-rebalancing.** Auto-rebalancing is
   axis A (delegated execution); the vault is axis B (pooled custody and share
   accounting). Even in the vault's own flow the pooled capital is forwarded to
   a Turnkey wallet to trade -- the vault never automates anything.
2. **Self-custody trading is venue-unconstrained.** When the signer is the owner
   (A coupled to B), there is no redeemable share price to protect, so no tier
   restriction applies: a connected wallet can trade Hyperliquid perps and
   Derive options directly, exactly as the current rebalancer does. The Tier-1
   restriction only ever came from wanting a **pooled** fund's redemption
   trust-minimized.

The consequence: for the near-term product a user connects a wallet to the
frontend and builds the multi-instruction transactions the rebalance needs, on
whatever venue. That is fully non-custodial and needs neither Turnkey nor the
vault. The vault program earns its complexity only when the goal is to let a
portfolio manager (PM) run **other people's** capital without those people
trusting the PM -- and that is precisely the property this repo (`fund`) is for.

## Decision

Build the product in two stages defined by which axes are coupled, and let each
axis migrate independently over time.

### v1 -- couple A to B (self-custody), documented and built in `moneymentum`

The signer is the owner. Two variants ship, differing only on axis A:

- **Non-custodial frontend (self-signed).** Wallet connected to the frontend,
  every trade signed by the user, all venues (Hyperliquid, Derive,
  Solana-native) reachable, no auto-rebalancing, no vault, no Turnkey. This is
  approximately the rebalancer that exists today, made client-side. Zero custody
  trust.
- **Turnkey tier (delegated).** The same capability plus auto-rebalancing, by
  accepting Turnkey policy-bounded delegated custody. The convenience variant;
  honestly centralized on axis A, disclosed as such.

v1 is a tool, not a pooled fund. It monetizes the user's own trading (and, if
the owner chooses, custodial managed accounts with explicit disclosure), not
trust-minimized third-party money.

### v2 -- decouple A from B (trust-minimized delegation), built in `fund`

Separate who executes from whose money: a PM gains **policy-scoped execution
authority** over **pooled** capital **without depositors surrendering custody**.
The one-line test: _trust funds to a PM without trusting funds to a wallet._
Shares and redemption are trust-minimized on-chain (the vault program plus the
NAV / redemption machinery of ADRs 0001-0004); execution authority is bounded by
on-chain policy rather than by a custodian's honesty. This is the differentiator
and the only thing that justifies this repo existing -- a manager-attested,
custodial fund would be redundant with the v1 Turnkey tier.

Crucially, decoupling A from B decentralizes the **authority and custody**
around the venues, not the venue choice, so **v2 does not have to sacrifice C
(derivatives) to be trust-minimized.** Hyperliquid and Derive stay in scope;
what gets trust-minimized is the delegation and the redeemable claim.

### The transition is a two-axis, gradual migration

The two variants converge over time along two orthogonal axes; progress on
either is independent and incremental:

| Axis  | v1 (now)                              | migration                                                         | v2 (target)                         |
| ----- | ------------------------------------- | ----------------------------------------------------------------- | ----------------------------------- |
| Trust | Turnkey delegated custody             | more execution / authority moves to permissionless on-chain infra | trust-minimized on-chain delegation |
| UX    | many wallets, one signature per venue | account abstraction / intents batch and unify signing             | few-signature, unified multichain   |

"Converged" means permissionless **and** few-signature **and** all-venue **and**
open to other people's money at once. Until then each axis advances on its own
schedule.

## Alternatives considered

### Move perps to a Solana-native venue (Drift / Jupiter) to trust-minimize a pooled fund now

- Pros: collapses the entire Tier-2 / Tier-3 apparatus (ADRs 0002-0004); the
  fund's redemption becomes fully trust-minimized with only on-chain reads; no
  cross-chain machinery.
- Cons: guts the derivatives depth that is the product; Solana-native perp /
  options liquidity is far thinner than Hyperliquid / Derive; and it still buys
  nothing on axis A (no auto-rebalancing).
- Rejected because: it optimizes axis C to trust-minimize axis B at the cost of
  the product's core (derivatives), and the same trust-minimization is
  achievable in v2 by decoupling A from B while keeping the venues.

### Ship the pooled fund first, on Hyperliquid, via the full Tier-2 model

- Pros: pooled OPM plus deep derivatives from day one; directly serves the
  stated commercialization (PM fees on other people's money).
- Cons: the hardest, most external-dependency-laden, most audit-heavy path (ADR
  0002 has multiple unpinned external facts and blocking owner decisions); the
  fund cannot process a redemption until a large body of cross-chain
  infrastructure exists; the redeemable Tier-2 claim is only ever capped /
  floored partial-trust anyway.
- Rejected because: it puts the frontier work on the critical path before any
  product ships, and the v1 self-custody tool delivers derivatives-first value
  now with zero of that risk.

### Make the fund honestly custodial (manager-attested NAV) short-term

- Pros: fastest path to pooled OPM and fees; no cross-chain verification needed.
- Cons: it is the Stream-Finance trust model the security design exists to
  reject; an on-chain vault adds cost and audit surface for near-zero trust
  reduction over the v1 Turnkey tier.
- Rejected because: if the fund is custodial anyway, it is redundant with the v1
  Turnkey tier -- the trust-minimization is the entire point of this repo.

## Consequences

- **v1 documentation and code live in `moneymentum`.** The near-term product
  ships there with no dependency on this repo. `moneymentum`'s SPEC should
  record the self-custody / Turnkey split and that the vault is a v2 concern.
- **This repo (`fund`) is scoped to v2:** the pooled vault (already begun --
  `create_fund`, `deposit`) plus the on-chain infrastructure for trust-minimized
  delegation. Withdrawal, fees, and the NAV / redemption machinery remain the
  fund's work regardless of stage.
- **ADRs 0002-0004 are v2 research, not launch blockers.** Their design stands
  and their Hyperliquid upgrade path stays open; they simply do not gate the v1
  product. The prior "fund-first, Hyperliquid-central" reading is demoted.
- **Commercialization note:** the v1 self-custody frontend is a tool, not a
  fee-earning managed-money product. Fees on other people's money still require
  either v2 (trust-minimized) or custodial managed accounts (trusted,
  disclosed). v2 is therefore the business's differentiator, correctly sequenced
  after v1 has users, not abandoned.
- If this framing turns out wrong, the blast radius is a mis-sequenced roadmap,
  not shipped code: no v1 code depends on the vault, and no v2 code has been
  written against the old framing.

## Open questions -- the v2 on-chain infrastructure research agenda

This repo's next research task. Each is a "what on-chain primitive does v2 need,
and does one exist that is trust-minimized enough" question.

1. **Policy-scoped delegated execution on Solana.** How does a PM or keeper
   execute trades over pooled capital within on-chain bounds (allowlisted
   venues, parameter ranges, rate limits) without holding custody? Survey
   Squads, native account delegation, session keys, and purpose-built
   permissioned programs. This is the axis-A trust migration in concrete terms.
2. **Trust-minimized cross-chain custody and execution.** How does pooled
   capital reach Hyperliquid / Derive and come back without a trusted bridge or
   custodian (the ADR 0002 open decision on off-exchange-settlement (OES)
   custody)? Survey intent-based bridges, canonical bridges, and their per-venue
   loss-sizing.
3. **Account abstraction and unified multichain signing.** What reduces "connect
   many wallets, sign many transactions" to few-signature, unified UX across
   Solana and the EVM venues? Survey smart accounts, session keys, and
   intent-execution networks -- the axis-UX migration.
4. **Trust-minimized NAV for off-Solana derivative legs.** Revisit the ADR 0002
   / 0003 tier ladder under the v2 framing: which legs can be priced into a
   redeemable claim, and which stay side-pocketed.
5. **Pooled vault and share accounting completion.** The remaining fund
   fundamentals independent of the above: withdrawal (signal / claim), fee
   accrual (management + performance / high-water mark), on-chain
   crystallization on realized proceeds only.
6. **Which venues v2 targets first, and whether v2 launches Tier-1-only
   trust-minimized or accepts Tier-2 partial-trust** for derivatives from the
   start -- the same tradeoff ADR 0002 surfaces, now a v2 sequencing call.
