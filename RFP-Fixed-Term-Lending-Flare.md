# RFP — Fixed-Term Lending on Flare

**Composing a Term Structure from Midnight × Spectra × Firelight**

*Phase 1: FXRP, FLR, sFLR · Roadmap: FBTC, stXRP, cross-asset credit*

**Author:** Janus the Watcher ([@XRPWatcherJanus](https://x.com/XRPWatcherJanus))
**Status:** Draft 5 — community review
**Date:** 31 May 2026

---

> This is a request for proposal, not a finished spec. It is intentionally falsifiable. Section 10 lists the scenarios that would kill the project. If they cannot be mitigated, it should not be built.
>
> The companion document is [RFP — Native Options Trading on Flare](https://github.com/janus-watcher/rfp-options-flare-native). That RFP prices *volatility*. This one prices *time and credit*. Together they describe the two halves of a complete on-chain capital market.

---

# 1. Executive Summary

DeFi has rates. It does not have a curve.

Every on-chain interest rate today is one of two things: a floating spot rate discovered through pool utilisation (Aave, Compound), or an unstructured incentive yield denominated in someone's emissions schedule. Neither is a term structure. Neither tells you what the market prices three-month money against twelve-month money. Neither lets a borrower or a lender lock a known rate for a known term and walk away.

A yield curve is the single instrument that lets a market separate the price of *time* from the price of *risk*, and signal from noise. Its absence is why a holder cannot tell whether a 25% yield is a genuine term-and-credit premium or a subsidy about to evaporate — and why a holder who needs liquidity today still chooses between selling the asset (a taxable event, a loss of exposure) and paying a credit-card company roughly 25% a year.

It is also why the borrow side of on-chain lending sits empty. MoreMarkets wound down a $40M, 4,000-depositor base in December 2025 for want of borrowers; Kinetic on Flare today shows the same shape, a large supply pool almost nobody borrows against. The reason is not the rate. It is that a loan repriced every block and liquidatable by an oracle glitch at 3am is not a loan a serious borrower will take. Removing that liquidation risk, and adding a term structure, is what this RFP is for — and the demand evidence is laid out in Section 4.

Three primitives now live on Flare that, composed, produce a real yield curve for the first time on this chain:

- **Morpho Midnight** — non-custodial, fixed-rate, fixed-maturity lending built on isolated immutable markets, where lending and borrowing happen through the trading of zero-coupon credit and debt units. Each market is a discrete maturity point. A strip of Midnight markets across maturities is a collateralised discount curve.
- **Spectra** — permissionless yield tokenization (PT/YT), live on Flare today on sFLR, stXRP planned. Splitting a yield-bearing asset into a fixed leg (PT) and a variable leg (YT) prices the *implied yield* of productive collateral across maturities. That is the curve for assets that already earn.
- **Firelight** — coverage as a first-class DeFi primitive, built on Flare, underwriting smart-contract, oracle, and governance tail risk. A priced, tradeable default-protection layer is what turns a *riskless* curve into a *credit* curve.

Composed: a collateralised term structure (Midnight) plus an implied-yield term structure (Spectra) plus a priced credit spread (Firelight) is a full on-chain yield curve with a credit dimension. No chain has built this deliberately. The points exist; nobody has drawn the line between them, published it as infrastructure, and let the rest of the ecosystem quote against it.

This document lays out why the curve is *necessary* (Section 4, the demand case against ~25% consumer credit), how the three primitives *compose* into one (Section 5), the economics for each participant (Section 6), the credit-spread layer in detail (Section 7), capital requirements (Section 8), the open questions a build must resolve (Section 9), and the failure modes that would kill it (Section 10).

The thesis is deliberately stated as an open RFP. The moat is the curve, and the moat is open for the first credible team. Janus does not need to be that team. The curve needs to exist.

It is a draft for community review. If the failure modes in Section 10 cannot be mitigated, the project should not be built.

# 2. The Missing Curve

## 2.1 What a yield curve does

In any mature credit market, the yield curve is not a chart. It is a coordination device. It lets a saver and a borrower who will never meet agree on the price of a loan that matures in 2031, today, and it lets a third party check whether that price is fair by comparing it to every other maturity. It is the public artifact against which private judgments about the future become legible and contestable.

It does three things at once:

1. It prices *time* — the term premium between short and long money.
2. It isolates *risk* — the credit spread of a borrower over the riskless rate at the same maturity.
3. It separates *signal from noise* — a quoted curve makes it obvious when a yield is a real premium for term and risk, and when it is an unsustainable incentive.

DeFi has none of this. Utilisation-driven variable rates collapse all three dimensions into a single number that moves with pool flow. An incentive APY denominated in emissions is not a rate at all; it is a marketing budget expressed as a percentage.

## 2.2 Why DeFi never built one

Early lending protocols emerged under thin, passive liquidity and high transaction costs. Pool-based variable-rate markets were the right design for that environment: aggregate everyone into one pool, discover the rate through utilisation, let anyone enter and exit at will. It worked. It also made a term structure impossible — there is no maturity in a pool you can leave at any moment, so there is no point to plot.

Fixed-rate lending has been explored on-chain (the Yield protocol, 2020) but never became the general foundation, because the hard problem was always liquidity bootstrapping. Isolated, per-maturity markets fragment liquidity by construction: capital that would happily lend across several maturities gets stranded in one. A curve needs many liquid maturity points simultaneously. Until recently, nothing solved that fragmentation cheaply.

## 2.3 What changed

Morpho Midnight's design directly attacks the fragmentation problem. Its offers do not lock capital and source liquidity only at settlement, so a single maker can quote across many maturities at once, with one signature, keeping capital productive elsewhere until an offer is filled. That is the mechanical precondition for a continuous curve rather than a scatter of disconnected dots: makers can populate the whole term structure without pre-funding every point.

Spectra independently solved the same problem from the asset side. By tokenising the yield of sFLR (and, planned, stXRP) into a fixed leg and a variable leg, it produces a market-clearing *implied fixed yield* for a productive asset at each listed maturity — an observable curve for collateral that earns.

Firelight makes the third dimension priceable. A curve without a credit spread is only a riskless curve. With a coverage market underwriting protocol and oracle failure, the spread over the collateralised rate becomes an explicit, tradeable number rather than an unmodelled tail.

Three teams solved three independent pieces of the same instrument. Nobody has composed them into the instrument itself.

# 3. The Three Primitives

The build is not three new protocols. It is one integration layer and one curve oracle on top of three existing ones. This section states precisely what each contributes; Section 5 composes them.

## 3.1 Morpho Midnight — the collateralised term structure

Midnight is a non-custodial fixed-rate lending protocol for the EVM, organised around isolated, immutable, permissionlessly created markets, each with a fixed maturity ([whitepaper, May 2026](https://github.com/morpho-org/midnight)). The mechanics that matter for a curve:

- **Zero-coupon units.** Within a market, positions are accounted in units. One debt unit is an obligation to repay one loan token before maturity; one credit unit is a claim on those repaid tokens. Buying units increases credit; selling increases debt. The rate is implied directly from the discount: for a traded price `P`, the simple rate over the remaining term is `r = 1/P − 1`. Each market is, by construction, one point on a discount curve.
- **Fixed calendar maturities, fungible positions.** Markets mature on fixed dates, not rolling tenors. Positions opened at different times into the same maturity are fungible. A strip of markets at 1, 3, 6, 12 months *is* the term structure for that collateral pair.
- **Offer-based, no locked capital.** Makers post offers that source liquidity only at settlement, via callbacks, and can quote across many markets with a single signature (Merkle-rooted offer sets, shared consumption-group budgets). This is what lets one maker populate the entire curve without fragmenting capital across every maturity.
- **Tick grid in rate space.** Ticks are defined so each step is a constant relative change in implied return (δ starts at 2%, can tighten to 0.5%). Quoting is in rate, which is exactly the unit a curve is drawn in.
- **Risk parameters per market.** Multi-collateral configurations, liquidation loan-to-value, a liquidation cursor `γ ∈ {0.25, 0.5}`, a recovery close factor, and a Dutch-auction softening of overdue (post-maturity) liquidations. Access-control gates can restrict entry without trapping funds.
- **Bounded protocol fees.** A settlement fee (capped so implied annualised cost stays ≤ 50 bps) and a continuous fee on outstanding credit (capped at 1% annualised).

What Midnight gives the curve: the **collateralised, fixed-rate, fixed-maturity backbone**. This is the closest thing on-chain to a riskless-rate term structure (riskless in the sense of overcollateralised, not risk-free).

## 3.2 Spectra — the implied-yield term structure for productive assets

Spectra is a permissionless yield-tokenization protocol, [live on Flare](https://flare.network/news/spectra-debuts-on-flare-trade-yield-on-sflr-and-soon-on-stxrp) with an sFLR market and stXRP planned. It splits a yield-bearing token into:

- **PT (Principal Token)** — a zero-coupon claim redeemable for the underlying at maturity. Holding PT locks a *fixed* yield, captured as the discount at which PT trades below the underlying.
- **YT (Yield Token)** — the stripped variable yield stream to maturity. Buying YT is a leveraged long on the asset's future yield.

The PT discount at each listed maturity is a directly observable **implied fixed yield** for the productive asset. Spectra is therefore the curve for collateral that already earns (sFLR, stXRP) — structurally distinct from Midnight, which prices credit against static collateral.

What Spectra gives the curve: the **yield-bearing-asset term structure**, and a native venue where a lender's fixed-rate preference (PT) and a speculator's variable-rate appetite (YT) clear against each other.

## 3.3 Firelight — the credit-spread layer

Firelight is building [coverage as a first-class DeFi primitive on Flare](https://www.theblock.co/post/381210/firelight-xrp-staking-flare-stxrp-defi-insurance), chain-agnostic by design, with claims submitted by an appointed agent and reviewed by an independent consortium, payouts executed on-chain. It covers smart-contract exploits, reentrancy, oracle manipulation, and malicious governance.

What Firelight gives the curve: the **default/tail-risk spread**. The difference between Midnight's overcollateralised rate and a rate a lender would accept *net of protocol risk* is a credit spread. Today that spread is unpriced — lenders bear smart-contract and oracle tail risk without compensation they can see. A coverage market makes the spread explicit: the cost of Firelight cover on a Midnight position is, to a first approximation, the credit spread of that position. The curve gains its third dimension.

## 3.4 The division of labour

| Primitive | Prices | Curve contribution | Status on Flare |
|---|---|---|---|
| Midnight | Fixed credit on static collateral | Collateralised spot/discount curve | EVM-generic; Flare deployment proposed |
| Spectra | Implied yield on productive assets | Yield-bearing term structure | Live (sFLR), stXRP planned |
| Firelight | Protocol/oracle/governance tail risk | Credit spread over the riskless curve | Live, Phase 2 coverage rewards 2026 |

The build composes them. It does not rebuild any of them.

# 4. The Demand Case

## 4.1 The anchor: ~25% consumer credit

In Q1 2026 the average APR across all US credit-card accounts was about 21%; the average *new-card offer* was about 23.79%, with a typical range of roughly 20% to 27%, and borrowers with fair credit routinely paid 24–28% ([LendingTree](https://www.lendingtree.com/credit-cards/study/average-credit-card-interest-rate-in-america/), [Bankrate](https://www.bankrate.com/credit-cards/advice/current-interest-rates/)). Call it 25%. People are actively looking for alternatives — that is the demand signal, and it is not a crypto-native one.

The 25% number does double duty in this thesis. It is the **borrower's pain** and the **lender's illusion** at the same time.

## 4.2 The real constraint is liquidation risk, not the rate

The borrower side of on-chain lending is empty, and the reflex is to blame the rate. The rate is not the problem. At 2% fixed for a year the yen carry trade moved trillions for three decades; price money cheaply enough and someone borrows it. Borrowers avoid on-chain credit for a different reason: the loan is not a loan.

In traditional finance a loan has term structure — a fixed rate, a fixed maturity, and a clean separation between the cost of money and the risk of the position. On-chain, a "loan" is an open position against the whole market: solvency repriced every block, collateral marked every second against oracle feeds that can glitch. On 10–11 October 2025 a single mis-reported price on one venue — USDe quoted at $0.65 on Binance — set off the largest liquidation cascade in crypto history: roughly $19–20bn of positions force-closed within hours while exchanges buckled and traders could not post collateral to defend themselves ([CoinGecko](https://www.coingecko.com/learn/october-10-crypto-crash-explained)). That was not a malfunction. That is the system working as designed.

No treasurer leverages a balance sheet against an instrument a bot can liquidate at 3am on a feed error. Liquidation risk, not the interest rate, is the binding constraint on borrow demand — and a fixed rate alone does not remove it. The product that actually unlocks the borrower is fixed rate **and** fixed term **and** deterministic, non-exploitable liquidation **and** a priced tail. Midnight supplies the first two and a predictable liquidation path (Section 5.1); FTSO and Firelight supply the last two (Section 7). Remove any one and the borrower stays home.

One distinction has to be drawn precisely, because conflating it is how a product like this oversells itself. "Liquidation risk" is two risks wearing one name. The first is *mechanism* risk — the oracle glitch, the 3am bot, the cascade triggered by a bad print on one venue. Deterministic, TWAP-read liquidation and a decentralised oracle remove that, and the removal is real. The second is *collateral-value* risk — the hard asset itself falling. No liquidation design removes that; it only makes the liquidation orderly rather than chaotic. A borrower against FXRP or FLR is still short the drawdown. This RFP solves the mechanism risk and only *manages* the value risk — through overcollateralisation, through Firelight, and by stating it plainly as a failure mode in its own right (Failure Mode 10.4), not folding it into the win.

## 4.3 The MoreMarkets autopsy: a $40M proof of the missing machine

The sharpest evidence is a recent death. In December 2025 MoreMarkets shut its Earn product with more than $40M in TVL and roughly 4,000 depositors — built with no points, no LP program, no incentives at all ([closure notice](https://www.moremarkets.xyz/blog/moremarkets-earn-accounts-closure)). Organic deposits at that scale, in a market of mercenary capital, are rare. The founder's public diagnosis was blunt: the borrower market was non-existent.

Read carelessly, that says "altcoin yield is dead." Read carefully, it says the opposite. The deposit demand was real, organic, and unbought — the lender side showed up in force. What collapsed was the borrow side, and it collapsed under exactly the architecture Section 4.2 describes: variable-rate, liquidate-every-block loans no serious borrower will take. MoreMarkets did not fail for lack of demand for on-chain credit. It failed because the machine that makes on-chain credit safe to borrow had not been built. (The full argument is Janus, *The Death of MoreMarkets.xyz — A Da Vinci Autopsy*, December 2025.)

This is close to a controlled experiment for this RFP. It holds the lender side fixed — proven, organic, $40M — and isolates the borrow side as the variable. It moves the question from the vague "is there demand for on-chain credit" to the sharp, testable "does removing liquidation risk and adding term structure convert standing deposit demand into actual borrow demand." That is precisely the hypothesis the build exists to test, and the pilot in Open Question 3 is designed to answer it before any capital commits.

The limit, kept in view rather than buried: MoreMarkets proves deposit demand, not borrow demand. It is equally consistent with "fix the architecture and borrowers appear" and with "the borrow side is empty for reasons no architecture repairs." The autopsy makes the bet legible. It does not win it. Failure Mode 10.1 carries that forward.

## 4.4 Kinetic: the same pattern, still live

MoreMarkets is the death case; Kinetic is the live one. Kinetic is the premier lending and borrowing protocol on Flare — overcollateralised, rates set dynamically by supply and demand each block, roughly $15M TVL as of February 2026 ([Flare](https://flare.network/news/kinetic-to-introduce-lending-and-borrowing-to-flare-ecosystem)). The plumbing already exists. What it lacks is genuine credit demand. The screen states it directly: the sFLR market shows a supply rate of 0.05% against a borrow rate of 2.26%, with a pool of roughly 434M sFLR available and almost none of it borrowed — utilisation in the low single digits. The borrowing that does occur is leverage and spot-hedging, not anyone financing a real obligation.

This is the thesis as a live reading rather than an abstraction. Variable rates select for speculative borrowing and against credit borrowing, because no one refinancing a real obligation accepts a cost that can reprice every block. The same double edge as MoreMarkets applies, and is stated rather than hidden: idle Kinetic supply is consistent with "wrong architecture" and with "no borrow demand at all." Kinetic cannot separate the two. That separation is the single most important thing the pilot must establish before a build (Open Question 3, Failure Mode 10.1).

Kinetic is also the natural incumbent to confront this directly. It already runs the lending rails and holds the supply; the part it lacks is the fixed-term, PT/YT-based layer on top — not the lending engine underneath. A venue that already has variable-rate lending and a standing deposit base, sitting on the exact demand gap this RFP describes, has the strongest reason of anyone to reckon with fixed-term, deterministically-liquidated lending. Either it adds that layer, or a venue that does displaces it. The choice between those paths is Open Question 9.

## 4.5 The borrower side: refinancing time, not chasing yield

A holder of FXRP, FLR, or sFLR who needs cash today has three options. Sell the asset — a taxable event in most jurisdictions and a forfeit of exposure. Borrow at a variable on-chain rate — but a borrower refinancing 25% card debt cannot rationally accept a rate that floats to 40% next month, defended by a liquidation bot; the whole point of refinancing is payment certainty. Or pay the card company 25%.

A *fixed-rate, fixed-maturity* loan against crypto collateral, with a predictable liquidation path, is the only on-chain product that actually competes with consumer credit, because it is the only one that offers the borrower the thing they are buying: a known cost for a known term that a feed glitch cannot revoke. Midnight provides exactly this. But a single rate at a single maturity is not enough — the borrower needs to see the price of 3-month versus 12-month credit to choose a term they can service. The curve is the precondition for the product, not a decoration on top of it.

Best case: crypto-collateralised fixed-term credit clears materially below 25% (overcollateralised lending should), and the refinance use case pulls genuinely external, non-crypto-native demand on-chain. Base case: the product wins crypto-native borrowers who currently use variable-rate lending and want term certainty, a real but smaller market. Worst case: the person paying 25% on a Visa and the person holding $50k of FXRP are nearly disjoint sets, and the refinance narrative is a story the collateral can't actually serve (see Failure Mode 10.1).

## 4.6 The second demand driver: on-chain SBLOC (buy, borrow, die)

The ~25% anchor (4.1) is the external, consumer-credit case. The second driver is native to the holders this chain attracts, structurally larger, and it already has a name in traditional wealth. A conviction holder of an appreciating asset does not sell — selling ends the thesis and realises the tax. Instead they take a securities-backed line of credit (SBLOC) against the portfolio, spend or redeploy the borrowed cash, and never touch the underlying. Because borrowing is not a realisation event, no tax is due; the asset is held to death, where heirs receive a stepped-up basis. *Buy, borrow, die.* It is the oldest move in private wealth, and it is not niche: securities-backed lending was roughly a $522bn market in 2024, projected past $1tn by 2033 ([Dataintelo](https://dataintelo.com/report/securities-backed-lending-market)).

This RFP is the on-chain port of that strategy: an SBLOC against FXRP, FLR, or FBTC instead of against a brokerage account of equities. The demand pattern is therefore not a crypto hypothesis waiting to be discovered — it is a proven, half-trillion-dollar TradFi behaviour, looking for collateral it already holds on this chain. Fixed term and a known cost are not optional here; they are the point. A conviction holder will not pledge an asset they refuse to sell against a debt a feed glitch can liquidate — that is forced selling at the worst price, on a bot's schedule. Remove the mechanism risk (4.2), give them a term they can plan around, and the trade becomes rational on-chain for the first time.

Two honest qualifications, both carried into Failure Mode 10.4. First, this is *living off debt, not off yield*: unlike the lender buying PT for income, the SBLOC borrower services debt against a static asset, so this demand carries the full reflexive and liquidation fragility, and it is one-directional and pro-cyclical — everyone long the same hard money, short the same crash. The driver that makes the product viable is the driver that makes it fragile. Second, the *volatility gap*: equity SBLOC survives on collateral whose annualised volatility runs near 15%, while crypto runs multiples of that — Bitcoin three to four times equity vol across 2020–2025, FXRP and FLR more ([CoinDesk](https://www.coindesk.com/markets/2025/04/11/s-and-p-500-more-volatile-than-bitcoin-as-u-s-assets-lose-investor-favor)). The same loan-to-value that is conservative against a stock portfolio is reckless against FLR. Deterministic liquidation and fixed term narrow the gap between on-chain and TradFi SBLOC; they do not close it. On-chain SBLOC is structurally more liquidation-prone than its equity parent, and the LLTVs have to say so.

## 4.7 The lender side: converting noise yield into signal yield

The lender earning a 25%-ish floating yield somewhere in DeFi today cannot decompose it. How much is term premium? How much is credit spread? How much is an emissions subsidy that ends next quarter? Without a curve, the answer is unknowable, so the yield is indistinguishable from noise.

A curve lets a lender lock a fixed yield for a fixed term — buy PT on Spectra, or take the credit side on Midnight — and read off exactly what they are being paid for. With Firelight cover priced alongside, the lender can split the quoted yield into riskless term premium and credit spread, and decide whether the spread compensates the tail. That is the difference between earning 25% and *understanding* 25%.

## 4.8 Two-sided demand, and escaping the cycle

The curve is necessary precisely because the two sides want opposite things and currently have no venue to meet across maturities. Borrowers want payment certainty (fixed cost, chosen term). Fixed-income lenders want yield certainty (PT, Midnight credit). YT speculators want the variable leg. The curve is the price at which these preferences clear. Build it and the demand becomes visible; leave it unbuilt and the demand stays latent, mispriced as a flat 25% that nobody can interrogate.

There is a larger reason the curve matters, beyond any single borrower. Without term structure, on-chain yield is a derivative of the Bitcoin cycle: leverage enters in the bull, yields spike, "DeFi works"; leverage exits in the bear, yields collapse, "DeFi is dead"; repeat. That is a sentiment gauge, not a credit market. A yield curve — fixed rates over fixed terms — is the instrument that lets capital price and hold duration through the cycle instead of being repriced by it every block. Building the curve is building the capacity to allocate across time rather than across mood.

# 5. Architecture — Composing the Curve

Five domains. The integration layer and the curve oracle are the new build; the three protocols underneath are integrations.

## 5.1 The collateralised curve (Midnight strip)

Deploy or integrate a strip of Midnight fixed-maturity markets for each approved collateral/loan pair, at a standard maturity calendar (proposal: 1, 3, 6, 12 months, monthly roll). Each market's implied rate `r = 1/P − 1` is one point. The curve oracle (5.4) reads the strip and publishes the collateralised term structure per pair.

Phase 1 pairs: FXRP-collateral / USDT0-loan, FLR-collateral / USDT0-loan, sFLR-collateral / USDT0-loan. USDT0 is the live, liquid base unit on Flare today; RLUSD is the ecosystem-aligned alternative and activates as an approved loan token if and when it bridges to Flare.

Midnight's liquidation design is the deterministic-liquidation half of the answer to Section 4.2. A recovery close factor caps each liquidation at the amount needed to restore health rather than wiping the whole position; overdue (post-maturity) liquidations are softened through a Dutch auction that starts at no incentive and ramps over a short window; and strike/health checks read a TWAP rather than a single-block snapshot. The effect is a liquidation path a borrower can model in advance — predictable, bounded, and far harder for a bot to weaponise on a momentary feed error than a variable-rate pool's instant full liquidation. That predictability is as much the product as the fixed rate.

## 5.2 The yield-bearing curve (Spectra PT/YT)

Integrate Spectra's PT/YT markets for sFLR (live) and stXRP (on listing). The PT discount per maturity publishes the implied-yield curve for each productive asset. This curve is *compared against* the Midnight collateralised curve for the same underlying: the gap between "fixed yield from holding the productive asset" (Spectra PT) and "fixed cost of borrowing against it" (Midnight) is the on-chain carry, and a tradeable one.

## 5.3 The credit spread (Firelight overlay)

Offer Firelight coverage as an optional, priced overlay on lender positions (Midnight credit, Spectra PT). The premium for cover at a given maturity is published as the credit spread for that position. A lender can hold an uncovered position (full yield, full tail risk) or a covered one (yield minus premium, tail capped). The difference is the market's price of the protocol/oracle/governance risk — the curve's third dimension, made explicit.

## 5.4 The curve oracle and terminal

The new core contract and data layer. It:

- reads the Midnight strip, Spectra PTs, and Firelight premiums per asset and maturity;
- publishes a unified, machine-readable yield curve per asset (riskless term structure + credit spread), via an open subgraph, free WebSocket feed, and a public terminal;
- exposes the curve as an on-chain oracle other Flare protocols can quote against (structured-product builders, the options venue's IV-vs-rate models, curated vaults).

Following the companion options RFP, all derived curve data is published openly. The moat is liquidity depth at each point and being the venue others quote against — not data licensing. Open data lowers integration cost, more integrations route more flow, flow deepens the curve.

## 5.5 The fixed-term credit product (user-facing)

The borrower- and lender-facing application that turns the plumbing into the product from Section 4: a borrower posts collateral, sees the curve, picks a term and a fixed rate, takes a Midnight offer; a lender supplies at a chosen maturity, optionally buys Firelight cover, sees the decomposed yield. Curated vaults (5.6) aggregate the lender side at scale.

## 5.6 The curator layer

As in the options RFP, the venue is infrastructure for curated vaults. Curators aggregate retail and mid-cap lender capital into single deposits and run defined strategies across the curve: a "fixed-income ladder" vault (PT + Midnight credit across maturities), a "covered-credit" vault (Midnight credit with Firelight overlay), a "carry" vault (long Spectra PT, short via Midnight). Curators own the user relationship; the venue stays neutral and asset-agnostic. Separation of concerns holds: Spectra/the venue curates strategy, Firelight underwrites — a curator must not insure its own book.

## 5.7 Spectra PTs as Midnight collateral: denomination sets the risk

The richest composition point, and the one most easily got wrong: a Spectra PT can serve as collateral in a Midnight market, but the loan token it is borrowed against changes the instrument entirely. A PT-sFLR (a principal token on sFLR) is a zero-coupon claim that pulls to par — one sFLR — at its maturity; before then it trades at a discount that is the locked fixed yield. The same collateral, in two markets, is two different risks.

**PT-sFLR borrowed against sFLR.** Collateral and loan are the same underlying family. The only thing that can diverge between them is the PT's own discount, which is bounded and converges monotonically to zero at the PT's maturity, absent an sFLR exchange-rate break or a Spectra-specific failure. There is almost no directional price gap to liquidate against. This is a near-pure rate-and-duration market: the borrower trades the PT discount against the sFLR borrow rate, not FLR/USD direction. It justifies a high LLTV and a low liquidation cursor (Midnight's γ = 0.25), and the liquidation risk that Section 4.2 identifies as the binding constraint is structurally small here. This is the safe, boring, and most important market to launch first.

**PT-sFLR borrowed against USDT0.** Now the collateral carries the full sFLR → FLR/USD price exposure while the debt is fixed in dollars. A FLR drawdown cuts the USD value of the collateral while the USDT0 debt does not move — genuine, directional liquidation risk, layered on top of the PT's pull-to-par and any sFLR exchange-rate drift. This is a price-and-rate market, not a pure rate market. It needs a lower LLTV, a higher liquidation cursor (γ = 0.5), and it is where Firelight cover earns its place.

The architectural consequence is firm: these are separate Midnight markets with separate risk parameters, and the curve oracle (5.4) must never blend their rates into a single point — one is a clean term-structure observation, the other is contaminated by FLR/USD volatility. This mirrors the degeneracy rule from the options RFP: an asset priced against itself has no volatility surface. PT-sFLR against sFLR is near-degenerate as a *price* market, which is exactly why it is excellent as a *rate* market; PT-sFLR against USDT0 is a real price market and must be margined like one.

One further subtlety, flagged for the build: a PT has its own maturity, and so does the Midnight market. If the PT matures before the loan it becomes plain sFLR mid-loan and the collateral's character changes; if it matures after, it still carries discount risk at the loan's settlement. Maturity-matching, or explicit bounding of the mismatch, is a design requirement, not a detail.

## 5.8 Why composition beats a monolith

A single protocol that re-implemented fixed lending, yield-stripping, and coverage would need three audits' worth of novel surface and would compete with three live teams. Composing inherits their audit heritage, their liquidity, and their maintenance. The new surface is the integration layer and the curve oracle — small, focused, independently auditable. The risk concentrates exactly where the novelty is, which is the honest place for it to sit.

# 6. Stakeholder Economics

Every credit market lives or dies on whether each side has a clean reason to show up.

## 6.1 The borrower

Posts crypto collateral, borrows a stable loan token at a fixed rate for a fixed term (Midnight debt units).

- Gains: payment certainty, retained asset exposure, no taxable disposal, a cost that should clear below unsecured consumer credit.
- Risks: liquidation if collateral falls below the maintenance threshold (Midnight's LLTV and recovery close factor); must repay or roll before maturity or face Dutch-auction overdue liquidation.
- Best / base / worst: refinances 25% external debt cheaply / gains term certainty over variable on-chain rates / gets liquidated in a collateral crash precisely when refinancing options elsewhere also vanish.

## 6.2 The fixed-income lender

Supplies the loan token at a chosen maturity (Midnight credit units) or buys PT (Spectra).

- Gains: a known yield to a known date, decomposable into term premium and (with cover) credit spread.
- Risks: protocol/oracle/governance tail (covered or not), opportunity cost if rates rise after locking, early-exit at a worse price if they need liquidity before maturity.
- Best / base / worst: locks a real premium over the riskless rate / earns a clean fixed yield / a covered tail event still pays slowly through a discretionary claims process.

## 6.3 The YT speculator

Buys the variable yield leg (Spectra YT).

- Gains: leveraged long on an asset's future yield; the natural counterparty that lets the PT lender lock fixed.
- Risks: YT decays to zero at maturity; if realised yield underperforms the implied yield paid, the position loses.
- Why it matters: YT demand is what makes PT (the fixed leg) liquid. As on Pendle, speculative YT flow tends to drive volume while PT is the quieter side. Thin YT demand means a thin fixed curve (Failure Mode 10.2).

## 6.4 The maker / LP

Quotes across the Midnight strip (offer-based, capital productive elsewhere until filled) and provides Spectra pool liquidity.

- Gains: spread and fees across many maturities from one capital base; the offer/callback model means capital stays deployed until an offer is taken.
- Risks: adverse selection from informed takers; inventory and duration risk across the curve; correlated drawdown when a collateral crash hits multiple maturities at once.
- Critical constraint: makers need rate-move circuit breakers and per-maturity exposure caps. A maker quoting the whole curve is short convexity across it (Taleb's point — the smooth curve hides the tail).

## 6.5 The underwriter (Firelight)

Sells coverage on lender positions; earns premium, bears claim payouts.

- Gains: premium income, a new product surface (covering a credit market, not just a staking protocol).
- Risks: correlated claims — a single oracle or contract failure hits every covered position at once; capacity limits; the discretion of the claims consortium is itself an attack surface (Failure Mode 10.6).

## 6.6 The protocol

Two revenue surfaces: integration/routing fees on flow it directs into Midnight and Spectra, and a configurable cut of the curve-terminal's value-added services (e.g. structured-product routing). Bounded by Midnight's own capped fees underneath. Token-design questions in Section 9.

# 7. The Credit-Spread Layer (Firelight)

Treated as core, not an afterthought, because a curve without a priced credit spread is only half an instrument.

## 7.1 Why the spread must be explicit

A lender on any overcollateralised protocol bears risks the headline APY ignores: a contract exploit, an oracle manipulation at settlement, a malicious governance action. These are real, correlated, and currently uncompensated in any visible way — they are folded silently into a flat yield. The lender cannot tell whether 8% fixed is generous or a thin pad over a fat tail.

Pricing cover on the position makes the tail a number. The premium Firelight charges to cover a Midnight credit position at maturity *is* that position's credit spread, observable and tradeable. Subtract it from the gross fixed yield and the lender sees the true riskless-equivalent return.

The deeper role, beyond pricing the spread, is removing the constraint of Section 4.2. The reason borrowers stay away is not an unpriced spread; it is liquidation risk — the 3am bot, the feed glitch, the cascade. Three pieces in combination attack it: Flare's FTSO, a decentralised oracle sourcing prices from a large set of independent providers, removes the single-venue glitch that detonated October 2025; Midnight's bounded, TWAP-read, Dutch-auctioned liquidation path (Section 5.1) makes the liquidation itself predictable rather than a race; and Firelight covers the residual tail the first two cannot design away. Pricing the spread is the visible output. Making the loan safe enough to borrow in the first place is the point. A credit market does not form because the spread is published; it forms because a borrower can take a position without betting against the infrastructure.

## 7.2 Construction

- For each covered (asset, maturity) position, publish the Firelight premium as an annualised spread.
- The curve terminal (5.4) publishes two curves per asset: gross collateralised yield (Midnight) and net-of-cover yield (Midnight minus Firelight premium). The gap is the credit spread term structure.
- Coverage is opt-in per position. The market decides how much of the curve trades covered.

## 7.3 The honest limits

Firelight's claims process is not parametric: an appointed agent files, an independent consortium reviews, payout executes on-chain only if approved. That review is discretion, and discretion is both a feature (it can judge novel exploits) and an attack surface (it can be captured, delayed, or disputed). A credit spread derived from a discretionary-payout product is only as credible as the consortium's independence and the coverage pool's capacity. Section 10.6 treats this as a first-order failure mode, not a footnote.

Coverage capacity is finite. If the pool cannot cover the notional that wants covering, the published spread understates true risk (it is the price of cover that exists, not cover that is needed). The terminal must publish remaining capacity alongside the spread, or the spread misleads.

# 8. Capital Requirements

| Item | Range (USD) | Note |
|---|---|---|
| Integration layer + curve oracle (dev, 9–15 mo) | 350K – 1.2M | The genuinely new surface; Midnight/Spectra/Firelight are integrations |
| Audits (integration + oracle) | < 150K (composed) | Narrower than greenfield: inherits underlying audit heritage; risk concentrates on the integration surface |
| Maker / LP bootstrap across the strip | 1M – 5M | Seed enough maturity points for a continuous curve, not isolated dots |
| Borrower-side acquisition (the hard side) | 100K – 400K | Refinance demand is non-crypto-native; acquisition is the binding constraint |
| Curated reference vaults | 75K – 200K | Ladder, covered-credit, carry archetypes (5.6) |
| Curve terminal + open data infra | 75K – 200K | Subgraph, WebSocket feed, public terminal |
| Total ex-LP (Phase 1) | ~0.75M – 2.15M | |

Funding sources to evaluate: a Flare Foundation grant against the remaining incentive pool; private LP from Flare-native treasuries; the Foundation as anchor maker across the strip (productive, durable, but puts the Foundation on the principal-risk side and raises the governor-and-LP conflict flagged in the options RFP); an optional protocol token (Section 9). The litmus from "Rented Deflation" applies unchanged: own the productive curve, do not pay mercenaries to rent its TVL — and if the curve needs perpetual subsidy to show liquidity at every point, the loop has only relocated.

# 9. Open Questions

This RFP is incomplete by design.

1. **Midnight on Flare.** Is Midnight deploying to Flare natively, or does the build deploy the open-source Midnight contracts ([github.com/morpho-org/midnight](https://github.com/morpho-org/midnight)) under its licence? The whole collateralised backbone depends on the answer. This is the first thing to resolve with the Morpho Association.
2. **Continuous curve vs. isolated dots.** How many maturity points, with how much maker depth each, before the strip reads as a curve rather than scattered quotes? What is the minimum viable calendar? Midnight's multi-market offers help, but the empirical threshold is unknown.
3. **The borrower overlap.** Is there measurable overlap between people paying ~25% consumer credit and people holding Flare collateral? This is the load-bearing demand assumption and it should be tested with a small paid pilot before the full build, not assumed.
4. **Firelight capacity and claims credibility.** What notional can Firelight actually cover at Phase 1, and how is consortium independence guaranteed and disclosed? Without a credible answer the credit-spread layer is decorative.
5. **Settlement and yardstick governance.** USDT0-settled only at launch, or RLUSD-ready? Who governs the approved collateral/loan/maturity lists — Foundation, DAO, builder, no-curation?
6. **Tokenomics.** Is there a protocol token, and if so does it touch LP capital (reflexivity risk, as Lyra in 2022) or stay purely governance/fee-discount? Fee-reinforced liquidity with no token is the cleanest answer to reflexivity, at the cost of slower bootstrap.
7. **Relationship to the options RFP.** The options venue prices volatility; this prices time and credit. Should they share a curve oracle, a curator layer, a token? A shared term-structure-and-vol surface is a strictly richer instrument than either alone.
8. **Success metrics.** Proposed: number of liquid maturity points per asset; borrower count and fixed-loan notional; covered vs. uncovered share of lender positions; the basis between Spectra PT yield and Midnight borrow cost (carry); time-to-fill on offers across the strip.
9. **Incumbent path: Kinetic.** Kinetic already operates variable-rate lending on Flare with standing supply but thin borrow demand (Section 4.4). It is the natural party to add a fixed-term, PT/YT-based layer — or the natural incumbent to be displaced by one. Is the build a collaboration with Kinetic (fastest path to existing liquidity and users), a Midnight-native venue running alongside it, or a direct competitor? Each path changes the liquidity-bootstrap, governance, and go-to-market calculus.

# 10. Failure Modes

Scenarios that would kill this. Mitigations are first drafts, not solutions. Listed in descending order of how much they threaten the thesis.

## 10.1 The refinance borrower doesn't exist

The strongest objection, and both MoreMarkets and Kinetic cut both ways on it (Sections 4.3, 4.4). The optimistic reading is that MoreMarkets' $40M of organic deposits and Kinetic's idle supply prove latent demand that the right architecture unlocks. The pessimistic reading is that the borrow side is empty for reasons no architecture repairs — that the person paying 25% on a Visa and the person holding $50k of FXRP are nearly disjoint sets, and a fixed-term product inherits the same emptiness. Both protocols show the lender side; neither proves the borrow side appears once liquidation risk is removed. That is the load-bearing assumption of the whole RFP, and it remains unproven. Mitigation: a small paid pilot measuring real, non-speculative borrower demand once a deterministic-liquidation, fixed-term product exists — before committing build capital (Open Question 3). Falsification: if a credible pilot shows no external borrower demand even with liquidation risk removed, the thesis collapses to "term certainty for existing DeFi borrowers" — a real but far smaller market, and the RFP should be rewritten around it rather than the 25% anchor.

## 10.2 Thin fixed-side demand — no curve, just one point

If DeFi participants structurally prefer variable yield and leverage (YT, not PT), the fixed leg stays thin and the curve has one liquid maturity and noise elsewhere. Pendle's history is the warning: YT speculation drives volume, PT is the quiet side. A curve with one liquid point is not a curve. Mitigation: seed maker depth at multiple maturities (Section 8), use Midnight's multi-market offers to populate the strip cheaply, treat number-of-liquid-points as the primary health metric. Falsification: if depth refuses to spread across maturities even when seeded, the term structure won't form on this chain yet.

## 10.3 Fixed-rate tail transfer (Taleb)

A fixed rate does not remove rate-and-default risk; it transfers it to the maker/LP and the underwriter, who are short convexity across the curve. The curve looks smooth in calm regimes and breaks in a vol spike — borrowers default exactly when collateral craters and exactly when cover is most needed. Mitigation: rate-move circuit breakers, per-maturity exposure caps, Firelight overlay sized for correlated drawdown, stress simulation (e.g. collateral −40% in 48h) as a mainnet precondition. The via-negativa framing: the product's value is the rate uncertainty it removes for users, paid for by concentrating tail risk in parties equipped to price it — only true if they actually are.

## 10.4 Collateral-value risk and the correlation cascade

The distinction from Section 4.2, given its own failure mode because it is the one the architecture cannot engineer away. Deterministic liquidation solves *mechanism* risk — the oracle glitch, the 3am bot. It does nothing about *collateral-value* risk: the hard asset itself crashing. Orderly liquidation is still liquidation; it makes the event clean, not absent.

The teeth are reflexive. The demand this RFP unlocks — conviction holders borrowing against FXRP, FLR, FBTC they refuse to sell (Section 4.6) — is one-directional and correlated. In a "correlations → 1" event, every borrower's collateral craters together, every position breaches at once, and even perfectly orderly liquidations dump correlated collateral into a falling market. The cascade returns through the collateral door, with a flawless oracle. The product is structurally short exactly the macro event the broader Janus framework treats as the one that matters — and by making the borrow-against-hard-money trade easy and rational (4.6), it *concentrates* that fragility rather than dispersing it. The underwriter does not escape it either: a Firelight pool covering these positions is short the same correlated event its policyholders are. This is the Section 4.6 volatility gap made operational — an LLTV that is conservative for an equity SBLOC is reckless against FLR, and the gap cannot be engineered away, only priced into a lower ceiling.

Mitigation: conservative LLTV on volatile collateral and the safe same-asset markets first (Section 5.7); per-asset and aggregate exposure caps; Firelight sized for correlated rather than idiosyncratic drawdown, with that limitation disclosed; a correlated −50%-across-all-collateral stress simulation as a mainnet precondition, not a later audit. Falsification: if the only LLTVs that survive a correlated crash are so conservative that the borrow stops being competitive with the 25% it was meant to beat, the value proposition is hollow and the product should not ship at scale.

## 10.5 Liquidity fragmentation across maturities

The classic isolated-market problem. Capital that would lend across the curve gets stranded per maturity, and no point is deep enough to quote tightly. Mitigation: lean entirely on Midnight's offer/callback and multi-market design, which exists precisely to let one maker quote the whole strip without pre-funding each point. Falsification: if fragmentation persists despite the offer model, the curve is uneconomic to maintain.

## 10.6 Firelight discretion and capacity

The credit spread is only as credible as the coverage behind it. A discretionary claims consortium can be slow, captured, or disputed; finite capacity means the published spread prices cover that exists, not cover that is needed. Mitigation: publish remaining capacity beside every spread; disclose consortium composition and independence; treat covered yield as an upper bound on safety, never a guarantee. Falsification: if the spread cannot be made credible, publish only the gross collateralised curve and drop the credit-spread claim rather than mislead.

## 10.7 Midnight licence / deployment blocker

If the Morpho Association does not deploy Midnight to Flare and the licence does not permit an independent deployment, the collateralised backbone is missing and the composition fails. Mitigation: resolve this first (Open Question 1); evaluate a fallback fixed-maturity lending base only if Midnight is genuinely unavailable. This is a precondition, not a risk to manage mid-build.

## 10.8 Reflexive tokenomics

A protocol token coupled to LP capital risks the Lyra-2022 death spiral: token down → maker TVL down → spreads widen → volume falls → token down. Mitigation: fee-reinforced liquidity with no token, or a strict separation of token (governance/fee-discount) from maker capital. Falsification: if no token design avoids reflexivity, ship without one.

## 10.9 Regulatory reclassification

Fixed-term lending against collateral, marketed as a consumer-credit alternative, is closer to a regulated lending and securities perimeter than variable DeFi yield — precisely *because* the consumer-credit framing invites the comparison. Note that Midnight's own whitepaper goes to unusual lengths to disclaim that its "credit", "debt", "obligation", and "maturity" terms denote any regulated instrument. Mitigation: permissionless, non-upgradeable settlement core; geo-block at the frontend; no KYC dependency that can be weaponised; careful that marketing copy does not itself create the regulated characterisation the contracts avoid.

# 11. Sources and Context

- Morpho Midnight Whitepaper, May 2026 — [github.com/morpho-org/midnight](https://github.com/morpho-org/midnight)
- Spectra debuts on Flare (sFLR live, stXRP planned) — [flare.network](https://flare.network/news/spectra-debuts-on-flare-trade-yield-on-sflr-and-soon-on-stxrp); [Spectra developer docs](https://dev.spectra.finance/guides/tokenizing-yield)
- Firelight XRP staking + DeFi insurance on Flare — [The Block](https://www.theblock.co/post/381210/firelight-xrp-staking-flare-stxrp-defi-insurance); [firelight.finance](https://firelight.finance/)
- US credit-card APR, 2026 — [LendingTree](https://www.lendingtree.com/credit-cards/study/average-credit-card-interest-rate-in-america/); [Bankrate](https://www.bankrate.com/credit-cards/advice/current-interest-rates/)
- Securities-backed lending (SBLOC) market (~$522bn 2024 → >$1tn by 2033) — [Dataintelo](https://dataintelo.com/report/securities-backed-lending-market); buy-borrow-die / SBLOC mechanics — [Fidelity](https://www.fidelity.com/lending/securities-backed-line-of-credit)
- Crypto vs. equity volatility (BTC ~3–4× S&P, 2020–2025) — [CoinDesk](https://www.coindesk.com/markets/2025/04/11/s-and-p-500-more-volatile-than-bitcoin-as-u-s-assets-lose-investor-favor)
- Kinetic — variable-rate lending on Flare (~$15M TVL, Feb 2026) — [Flare](https://flare.network/news/kinetic-to-introduce-lending-and-borrowing-to-flare-ecosystem); [Kinetic litepaper](https://medium.com/@socials.kinetic/the-kinetic-litepaper-a97bf58030ed)
- MoreMarkets Earn shutdown, Dec 2025 ($40M+ TVL, no incentives, "borrower market non-existent") — [closure notice](https://www.moremarkets.xyz/blog/moremarkets-earn-accounts-closure); Janus, *The Death of MoreMarkets.xyz — A Da Vinci Autopsy* (Dec 2025), [janusthewatcher.substack.com](https://janusthewatcher.substack.com)
- October 10–11, 2025 liquidation cascade (~$19–20bn, USDe oracle glitch to $0.65 on Binance) — [CoinGecko](https://www.coingecko.com/learn/october-10-crypto-crash-explained)
- Companion: [RFP — Native Options Trading on Flare](https://github.com/janus-watcher/rfp-options-flare-native)
- Lineage: the Yield protocol (Robinson & Niemerg, 2020); Morpho Blue (2023); Pendle (PT/YT precedent)

---

*This document is published under [CC BY 4.0](LICENSE). It is a thesis offered for the ecosystem to build, not a claim on the building. The curve needs to exist. Whoever draws it first, and deepest, owns the point everyone else quotes against.*

*— Janus the Watcher*
