# RFP — Fixed-Term Lending on Flare

A request for proposal for composing **Morpho Midnight** (fixed-rate, fixed-maturity lending), **Spectra** (PT/YT yield tokenization), and **Firelight** (coverage) into a native fixed-term lending market — and the term structure it produces: a yield curve with a priced credit dimension.

**Author:** Janus the Watcher · [@XRPWatcherJanus](https://x.com/XRPWatcherJanus)<br>
**Status:** Draft 3 — open for community review · see [CHANGELOG](CHANGELOG.md)<br>
**Companion:** [RFP — Native Options Trading on Flare](https://github.com/janus-watcher/rfp-options-flare-native) (prices volatility; this one prices time and credit)

---

## TL;DR

DeFi has rates. It does not have a curve. Every on-chain rate today is either a floating spot rate (Aave/Compound utilisation) or an unstructured incentive yield. Neither is a term structure, so nobody can separate the price of time from the price of risk, or signal from noise.

Three primitives now live on Flare can be composed into a real yield curve for the first time: **Midnight** gives a collateralised fixed-rate, fixed-maturity backbone (zero-coupon credit/debt units, one market per maturity); **Spectra** gives an implied-yield curve for productive assets (PT = fixed leg, YT = variable leg, live on sFLR); **Firelight** prices the credit/tail-risk spread that turns a riskless curve into a credit curve.

The demand anchor is ~25% consumer-credit APR — the borrower's pain and the lender's illusion at once. But the deeper diagnosis is that the borrow side of on-chain lending is empty because of **liquidation risk, not the rate**: a loan repriced every block and liquidatable by an oracle glitch at 3am (see October 2025, ~$19bn cascade) is not a loan a serious borrower takes. The evidence is recent and hard. MoreMarkets wound down a $40M, 4,000-depositor base in December 2025 — organic, zero incentives — for "non-existent borrower demand"; Kinetic on Flare shows the same shape live (sFLR supply rate 0.05% vs. 2.26% borrow, utilisation in the low single digits). Both prove the lender side and leave the borrow side as the open question. The product that targets it is fixed rate + fixed term + **deterministic, non-exploitable liquidation** (Midnight's bounded, TWAP-read path) + a priced tail (FTSO + Firelight). The RFP is honest that this remains a bet: both data points prove deposit demand, not borrow demand — which is why a paid pilot precedes any build (Section 4, Failure Mode 10.1).

The build is one integration layer and one curve oracle on top of three live protocols — not three new ones. It is intentionally falsifiable: Section 10 lists the scenarios that would kill it, starting with the hardest (does the refinance borrower actually exist).

## What's here

| File | Contents |
|---|---|
| [`RFP-Fixed-Term-Lending-Flare.md`](RFP-Fixed-Term-Lending-Flare.md) | Full RFP, readable on GitHub |
| [`RFP-Fixed-Term-Lending-Flare.pdf`](RFP-Fixed-Term-Lending-Flare.pdf) | PDF version for download / print |
| [`CHANGELOG.md`](CHANGELOG.md) | Version history |

## Structure

1. Executive Summary
2. The Missing Curve — what a yield curve does, why DeFi never built one, what changed
3. The Three Primitives — Midnight (collateralised term structure), Spectra (yield-bearing curve), Firelight (credit spread)
4. The Demand Case — ~25% consumer credit; liquidation risk as the real constraint (Oct 2025 cascade); the MoreMarkets autopsy ($40M death case) and Kinetic (live case); borrower/lender economics; escaping the BTC cycle
5. Architecture — composing the curve: Midnight strip, Spectra PT/YT, Firelight overlay, the curve oracle/terminal, the fixed-term product, the curator layer, PT-as-collateral (denomination sets the risk)
6. Stakeholder Economics — borrower, fixed-income lender, YT speculator, maker/LP, underwriter, protocol
7. The Credit-Spread Layer — Firelight as core, construction, and honest limits
8. Capital Requirements
9. Open Questions — Midnight-on-Flare, continuous curve vs. dots, the borrower overlap, Firelight capacity, governance, tokenomics, metrics, the Kinetic incumbent path
10. Failure Modes — eight scenarios, hardest first (the refinance borrower may not exist)
11. Sources

## Contributing

Issues are open for discussion. Pull requests are welcome **as proposals** — reviewed and merged at the author's discretion to keep the document coherent. This is a curated RFP, not a wiki.

## License

[CC BY 4.0](LICENSE) — free to share and adapt with attribution.
