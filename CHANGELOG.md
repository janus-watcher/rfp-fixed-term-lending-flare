# Changelog — RFP: Fixed-Term Lending on Flare

All notable changes to this RFP. Versioned snapshots live in [`drafts/`](drafts/); the push-ready set lives in [`github_upload/`](github_upload/).

## Draft 5 — 2026-05-31

Named the second demand driver as what it is: on-chain SBLOC.

- **Section 4.6** rewritten around securities-backed lending (SBLOC) and *buy, borrow, die* — the established TradFi wealth strategy (hold appreciating asset, borrow against it tax-free since borrowing isn't a realisation event, step-up at death). The RFP is the on-chain port: SBLOC against FXRP/FLR/FBTC. Anchored on a ~$522bn (2024) → >$1tn (2033) market, so the demand pattern is proven, not hypothetical.
- Added two honest qualifications: it is *living off debt, not yield* (carries the reflexive/liquidation fragility), and the *volatility gap* — crypto collateral runs multiples of equity vol (BTC ~3–4× S&P), so on-chain SBLOC is structurally more liquidation-prone than equity SBLOC; deterministic liquidation + fixed term narrow the gap, don't close it.
- **Failure Mode 10.4** reinforced: vol gap made operational — an LLTV safe for equity SBLOC is reckless against FLR.
- Sources + README updated.

Commit: `Draft 5: frame second demand driver as on-chain SBLOC (buy-borrow-die) + volatility gap`

## Draft 4 — 2026-05-31

Separated two risks that "liquidation risk" was bundling, and named the second demand driver.

- **Section 4.2:** distinguished *mechanism* risk (oracle glitch, 3am bot — solved by deterministic/TWAP liquidation) from *collateral-value* risk (the hard asset crashing — only managed, never removed). Stops the product overselling itself.
- **New Section 4.6:** the second demand driver — conviction holders borrowing fiat against hard money they refuse to sell (liquidity without disposal). Structurally larger than the consumer-credit anchor, and explicitly the same flow that creates the fragility below.
- **New Failure Mode 10.4 — collateral-value risk and the correlation cascade:** the demand unlocked is one-directional, pro-cyclical, and short the Phase-2 "correlations → 1" crash; orderly liquidation still dumps correlated collateral into a falling market, so the cascade returns through the collateral door with a flawless oracle. The underwriter is short the same event. Mitigation: conservative LLTV, exposure caps, correlated-drawdown stress as a mainnet precondition. (Renumbered former 10.4–10.8 to 10.5–10.9.)
- Credit to a sparring review for forcing the mechanism-vs-value split.
- Status bumped to Draft 4.

## Draft 3 — 2026-05-31

Reframed the demand thesis around liquidation risk and added the strongest evidence.

- **Reframe:** the binding constraint on borrow demand is *liquidation risk, not the rate* (Section 4.2), anchored on the October 2025 cascade (~$19–20bn, USDe oracle glitch).
- **Added** the MoreMarkets autopsy as keystone demand evidence (Section 4.3): $40M TVL, ~4,000 depositors, zero incentives, shut Dec 2025 for "non-existent borrower demand." Treated as a near-controlled experiment — lender side proven, borrow side the open variable.
- **Added** PT-as-collateral architecture (Section 5.7): PT-sFLR borrowed against sFLR (near-pure rate market, high LLTV, γ=0.25) vs. against USDT0 (price+rate market, lower LLTV, γ=0.5, Firelight-relevant). The curve oracle must not blend them.
- **Added** Kinetic as the incumbent-path Open Question 9, and positioned it in Section 4.4 as the natural party to confront PT/YT-based fixed-term lending.
- **Deepened** Firelight (Section 7.1) and Midnight liquidation mechanics (Section 5.1) as the deterministic-liquidation answer to Section 4.2.
- **Sharpened** Failure Mode 10.1: both MoreMarkets and Kinetic prove deposit demand, not borrow demand — the load-bearing assumption stays unproven pending a pilot.
- Status bumped to Draft 3; added versioning, `drafts/`, and `github_upload/`.

## Draft 2 — 2026-05-31

- **Renamed** from "Native Yield-Curve Infrastructure" to **"Fixed-Term Lending on Flare"** (functional title; yield and duration are contained in the term). Repo: `rfp-fixed-term-lending-flare`.
- **Added** Kinetic as local demand evidence (variable-rate lending exists on Flare, ~$15M TVL, but liquidity nobody borrows except to lever/hedge) and its honest double edge.

## Draft 1 — 2026-05-31

- Initial RFP. Composes Morpho Midnight (fixed-rate, fixed-maturity lending) × Spectra (PT/YT yield tokenization) × Firelight (coverage) into a native fixed-term lending market and the yield curve it produces.
- Demand anchor: ~25% consumer-credit APR. Sister document to [RFP — Native Options Trading on Flare](https://github.com/janus-watcher/rfp-options-flare-native).
- Intentionally falsifiable: Section 10 lists the scenarios that would kill it.
