# Options Volatility Analysis

Python analysis of option pricing, implied versus realised volatility and dynamically delta-hedged gamma exposure. The project develops a Black-Scholes based straddle model and evaluates discrete delta hedging and measured sensitivity to volatility assumptions, hedge frequency and transaction costs.

## Question

How does a delta-hedged long-option position behave when implied volatility differs from subsequently realised volatility and how much of the resulting P&L can be explained by gamma, theta and transaction costs.

## Model

### Instrument Choice 

The strategy models a European-style SPX straddle and dynamically hedges its directional exposure using E-mini S&P 500 futures. SPX was selected as the option underlying because it provides direct S&P 500 index-option exposure, while ES futures provide a liquid, tradable instrument for managing changes in index delta. The implementation therefore prices the option from the SPX index using a dividend-adjusted Black–Scholes–Merton framework and calculates hedge P&L from movements in ES futures. Because SPX and ES are related but non-identical instruments, the hedge ratio explicitly accounts for their respective $100 and $50 contract multipliers and the futures-to-spot relationship. An alternative formulation would model options directly on ES futures using Black-76. This was not chosen because the objective is to analyse hedging of index-option volatility exposure rather than futures-option pricing.
