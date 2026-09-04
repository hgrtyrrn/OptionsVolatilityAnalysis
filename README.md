# Options Volatility Analysis

Python analysis of option pricing, implied versus realised volatility and dynamically delta-hedged gamma exposure. The project develops a Black-Scholes based straddle model and evaluates discrete delta hedging and measured sensitivity to volatility assumptions, hedge frequency and transaction costs.

## Question

How does a delta-hedged long-option position behave when implied volatility differs from subsequently realised volatility and how much of the resulting P&L can be explained by gamma, theta and transaction costs.

## Model

### Instrument Choice 

The strategy models a European-style SPX straddle and dynamically hedges its directional exposure using E-mini S&P 500 futures. SPX was selected as the option underlying because it provides direct S&P 500 index-option exposure, while ES futures provide a liquid, tradable instrument for managing changes in index delta. The implementation therefore prices the option from the SPX index using a dividend-adjusted Black–Scholes–Merton framework and calculates hedge P&L from movements in ES futures. Because SPX and ES are related but non-identical instruments, the hedge ratio explicitly accounts for their respective $100 and $50 contract multipliers and the futures-to-spot relationship. An alternative formulation would model options directly on ES futures using Black-76. This was not chosen because the objective is to analyse hedging of index-option volatility exposure rather than futures-option pricing.


### ES hedge-price convention

A data issue arose when constructing the E-mini S&P 500 futures hedge around the March 2020 contract roll. The initial implementation intended to use CME fixing prices for both the March (`ESH0`) and June (`ESM0`) contracts. However, the Databento CME statistics data do not provide a continuous daily fixing series for the June contract over the backtest window; a June fixing is only available on 20 March.

Rather than interpolate missing observations, forward-fill another contract's fixing, or construct an arbitrary intraday proxy, the backtest uses the official CME daily settlement price as the hedge mark for both contracts. Settlement observations are available consistently across the required period and provide a common, reproducible marking convention for a daily-frequency futures hedge.

March and June settlement changes are calculated separately within each contract before the active hedge contract is selected around the roll. This prevents the price difference between two distinct futures contracts from being incorrectly recognised as hedge P&L when the strategy rolls from the March to the June contract.

This convention means that hedge P&L represents settlement-to-settlement futures performance rather than realised intraday execution P&L. Transaction timing, bid-ask effects and intraday basis movements are therefore outside the scope of the daily backtest.
