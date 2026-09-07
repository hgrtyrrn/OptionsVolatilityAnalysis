# Options Volatility Analysis

Options Volatility Analysis is a Python-based derivatives research project examining the economics of a dynamically delta-hedged long SPX straddle when implied volatility differs from realised volatility. The analysis uses Black-Scholes option pricing and Greeks with SPX, VIX9D and E-mini S&P 500 futures data to study how convexity, time decay, dynamic hedge P&L and transaction costs interact over the life of a short-dated option position.

The project is structured as a progression from a controlled theoretical benchmark: `OptionsVolatilityAnalysis_Theoretical_FractionalES.ipynb` to an executable base-case notebook: `OptionsVolatilityAnalysis_ExecutableBacktest.ipynb` and then to formal robustness testing: `Sensitivity_Robustness_Analysis.ipynb`. The theoretical benchmark uses fractional ES contract equivalents to isolate the mechanics of dynamic delta hedging. Due to the fractional ES contracts, a fully delta-neutral position can be created at the time of the rehedge. This leaves no first-order directional exposure to changes in the price of the underlying asset (in this case the S&P 500 index).

It models contract-specific futures P&L across the March 2020 ES roll, a residual-delta rehedging rule (10% SPX-equivalent delta of residual exposure), option repricing through expiry, futures commissions, Greek P&L attribution, terminal hedge closure and reconciliation controls.

The executable backtest relaxes these simplifying assumptions where possible. Fractional futures are replaced with integer ES contracts, more realistic commissions are introduced and bid-ask spreads and slippage are implemented. The March 2020 ES roll is represented as separate closing and opening transactions and transaction costs are applied to both legs. Limitations that remain include timestamped intraday hedge execution and more realistic treatment of the SPX-ES basis.

Further extensions to this project could add actual option prices and strike-specific implied volatility (current data constraints prevent this for the chosen backtest window). Intraday hedge execution could also be added in future, with greater data access. These are therefore not included in the notebook and if implemented, would have to be applied to both the theoretical (control) and executable backtest notebooks to ensure a valid comparison to the benchmark (theoretical) model.

A separate sensitivity framework tests the robustness of results to the delta-rehedge band, transaction costs, volatility assumptions, and fractional versus integer hedge granularity. The outputs include total P&L, turnover, number of rehedges, transaction costs, residual delta exposure and comparative sensitivity plots and tables.

## Research Question

How does a dynamically delta-hedged long SPX straddle perform when implied volatility differs from realised volatility and how do ES hedging constraints affect P&L, hedge effectiveness and its decomposition into option Greeks, futures hedge P&L, transaction costs and residual repricing effects?

## Project Architecture

| Notebook                                   | Purpose                                                                                    | 
| ------------------------------------------ | ------------------------------------------------------------------------------------------ | 
| Theoretical Fractional-ES Benchmark        | Isolates dynamic gamma-scalping mechanics under controlled hedge assumptions               |
| Practical Integer-ES / Executable Backtest | Introduces discrete contract sizing and more realistic execution constraints               |
| Sensitivity Analysis                       | Tests robustness to modelling, volatility, cost and hedge assumptions                      |

## 1. Theoretical Benchmark

### Objective

The theoretical benchmark isolates the economics of a dynamically delta-hedged long-option position before introducing the full set of constraints faced by an executable strategy.

The notebook constructs one short-dated SPX straddle and follows the position from 10 March 2020 to expiry on 20 March 2020, a period containing exceptionally large movements in the S&P 500 and short-dated implied volatility. The option position is repriced daily using Black-Scholes with VIX9D as the volatility input, while its changing directional exposure is hedged using E-mini S&P 500 futures.

Fractional ES contract equivalents are deliberately permitted. This removes integer-contract rounding as a source of residual exposure and creates a cleaner benchmark against which the later executable version can be compared. However, hedging remains discrete: the futures position is changed only when residual SPX-equivalent delta exceeds a defined threshold (10% of the value of an SPX contract), when the active futures contract rolls or when the option expires.

The benchmark's purpose is to establish a controlled P&L and risk-accounting framework in which option repricing, hedge P&L, convexity, time decay, volatility-input changes and transaction costs can be examined separately.

### Instrument Choice

The option position is a long SPX straddle consisting of one European call and one European put with the same strike and expiry. A long straddle provides positive gamma and negative theta and limits the initial directional exposure of the combined option position, making it suitable for studying the relationship between realised underlying movement, option convexity and the cost of carrying long volatility exposure.

The SPX level at inception is approximately **2,882.23**. The model selects the nearest five-point strike and fixes it for the life of the trade, producing a **2,880 strike** straddle expiring on **20 March 2020**. The strike remains unchanged as the index moves.

E-mini S&P 500 futures are used as the delta hedge. The model applies:

* An SPX option multiplier of **$100 per index point**.
* An ES futures value of **$50 per index point**.
* One long SPX straddle.
* A residual-delta rehedging band of **0.10 SPX-equivalent delta**.
* A theoretical commission of **$1.25 per ES contract equivalent traded**.
* A zero risk-free rate for the benchmark.

The conversion between option delta and the required ES hedge is:

$$
q_t^{ES} = -\Delta_t^{option}\frac{100}{50}
$$

where $$\(q^{ES}_t\)$$ is the target number of ES contract equivalents. Because the theoretical benchmark permits fractional futures, the required hedge can be represented without integer rounding.

### Strategy Specification

At inception, the Black-Scholes delta of the straddle is calculated and converted into the exact fractional ES position required to offset its dollar delta.

For every subsequent market observation:

1. The SPX straddle is repriced using the current SPX level, VIX9D volatility input and remaining time to expiry.
2. The ES position held at the start of the interval earns settlement-to-settlement futures P&L.
3. The option-price change is attributed using the previous observation's delta, gamma and theta.
4. Residual SPX-equivalent delta is measured before any new hedge trade.
5. The hedge is left unchanged unless the residual delta exceeds the configured 0.10 band.
6. If the band is breached, the position is rebalanced to the exact fractional delta-neutral target.
7. If the active ES contract changes, the old futures position is closed and the new contract is established explicitly.
8. Transaction costs are charged on the contract-equivalent quantity traded.
9. At option expiry, the straddle settles at intrinsic value and the remaining futures hedge is explicitly closed.

This sequencing ensures that the hedge held during an interval earns that interval's futures P&L before any end-of-period rehedge is applied. It also prevents current-period information from being used retrospectively to change the hedge exposure that generated the preceding interval's P&L.

### Data and Hedge Construction

#### Market Data

The theoretical benchmark covers **10–20 March 2020** and contains nine aligned trading-date observations.

SPX and VIX9D closing data are obtained using `yfinance`:

* `^GSPC` supplies the SPX level used to mark the option position.
* `^VIX9D` supplies the short-dated implied-volatility proxy.

VIX9D is converted from percentage points into decimal volatility before entering the Black-Scholes model. It should be interpreted as a short-dated market volatility proxy rather than the strike-specific implied volatility of the exact 2,880 call and put.

Contract-specific E-mini S&P 500 settlement observations are held locally for:

* March 2020 ES (`ESH20`).
* June 2020 ES (`ESM20`).

The datasets are aligned on common trading dates. The notebook validates that required observations exist at both trade inception and option expiry, checks for non-positive SPX or volatility inputs, rejects missing or duplicate ES settlement observations and prevents data extending beyond expiry.

The strike is determined once from the initial SPX level and remains fixed. Time to expiry is calculated using calendar time:

$$
T_t = \frac{\text{calendar days to expiry}}{365}
$$

SPX log returns are retained separately for the subsequent realised-volatility calculation.

#### ES Hedge-Price Convention

A data limitation arose when constructing the E-mini S&P 500 futures hedge around the March 2020 contract roll. Databento CME statistics data do not provide a continuous daily fixing series for the June contract (`ESM20`) over the backtest window. Therefore, the official CME daily settlement prices for March (`ESH20`) and June are used instead. This creates a consistent marking convention without unnecessary interpolation or introducing another proxy.

The hedge is held in March ES before **12 March 2020**. After this date, it is held in June ES (post-roll date). Settlement changes are calculated separately within each contract. P&L for the interval ending on the roll date is generated by the March contract held over the interval. After this, P&L is recognised, the March position is closed and the June contract hedge is established.

Price-level difference between the two contracts is therefore not recognised as P&L, ensuring correct accounting. This also ensures that the hedge remains contract-consistent. 

Therefore, the theoretical notebook's hedge P&L represents settlement-to-settlement futures performance.

### Methodology

The model separates three related but distinct accounting problems:

1. **Option valuation**, using Black-Scholes.
2. **Option P&L attribution**, using previous-period Greeks. 
3. **Strategy P&L**, combining the actual theoretical option repricing with ES hedge P&L and futures transaction costs.

The Greek attribution is diagnostic rather than a substitute for actual model repricing. Total option P&L is always calculated from the change in the theoretical straddle value. Delta, gamma and theta are then used to explain that change. Any unexplained amount is assigned to a residual.

Similarly, the strategy's hedge P&L is calculated directly from the futures position and contract-specific settlement movement rather than inferred from option delta attribution.

This distinction allows the notebook to reconcile the economic intuition of gamma scalping with the actual accounting mechanics of the simulated position.

#### Option Pricing

The call and put are valued using the Black-Scholes European-option framework.

For the call:

$$
C_t = S_tN(d_1) - Ke^{-rT_t}N(d_2)
$$

and for the put:

$$
P_t = Ke^{-rT_t}N(-d_2) - S_tN(-d_1)
$$

where

$$
d_1 = \frac{\ln(S_t/K) + (r+\frac{1}{2}\sigma_t^2)T_t}{\sigma_t\sqrt{T_t}}
$$

and

$$
d_2 = d_1-\sigma_t\sqrt{T_t}
$$

with:

* $$\(S_t\)$$: SPX level.
* $$\(K\)$$: fixed 2,880 strike.
* $$\(r\)$$: benchmark risk-free rate, set to zero.
* $$\(\sigma_t\)$$: VIX9D divided by 100.
* $$\(T_t\)$$: remaining calendar time to expiry.

The theoretical straddle value is:

$$
V_t=C_t+P_t
$$

and dollar option P&L over an interval is:

$$
\text{P\\&L}^{option}_t = (V_t-V_{t-1})\times100
$$

At expiry, the call and put are valued at intrinsic value rather than evaluating the Black-Scholes expressions as \(T\rightarrow0\). Gamma and theta are also set to zero after expiry. This avoids numerical instability around zero time to maturity.

The initial theoretical straddle value is approximately **218.29 SPX points**, equivalent to approximately **$21,829** using the $100 SPX option multiplier.

#### Greeks

The notebook calculates call and put delta, gamma and annualised theta directly from the Black-Scholes framework.

Straddle delta is:

$$
\Delta^{straddle}_t =\Delta^{call}_t+\Delta^{put}_t
$$

and the same-strike call and put gamma combine to:

$$
\Gamma^{straddle}_t = 2\Gamma_t
$$

Straddle theta is:

$$
\Theta^{straddle}_t = \Theta^{call}_t+\Theta^{put}_t
$$

For each interval, the previous observation's Greeks are used to construct a second-order approximation of the option-price change:

$$
\text{P\\&L}^{\Delta}_t = \Delta_{t-1}\times100\times\Delta S_t
$$

$$
\text{P\\&L}^{\Gamma}_t = \frac{1}{2}\Gamma_{t-1}\times100\times(\Delta S_t)^2
$$

$$
\text{P\\&L}^{\Theta}_t = \Theta_{t-1}\times100\times\Delta t
$$

where $$\(\Delta t\)$$ is measured in calendar years.

The residual is then defined as:

$$
Residual_t = \text{P\\&L}^{option}_t-\text{P\\&L}^{\Delta}_t-\text{P\\&L}^{\Gamma}_t-\text{P\\&L}^{\Theta}_t
$$

The residual is intentionally **not** labelled vega P&L. Due to the fact that the option is repriced each day using a changing VIX9D input, the residual can contain the effect of **volatility-input changes (vega-related effects), higher-order Greeks (measuring the change of Greeks themselves), interaction terms and error from the discrete second-order approximation.**

Futures hedge P&L is calculated separately:

$$
\text{P\\&L}^{ES}_t = q^{ES}_{t-1}\times50\times\Delta F_t
$$

using the futures quantity actually held during the interval and the settlement change of the contract held over that interval.

Total strategy P&L is therefore:

$$
\text{P\\&L}^{strategy}_t = \text{P\\&L}^{option}_t + \text{P\\&L}^{ES}_t - Cost_t
$$

This separation is important: Greek attribution explains the **theoretical option-price change** - this is shown in the decomposition of option P&L, whereas ES P&L records the performance of the **actual hedge instrument** used by the strategy.

### Validation and Backtest Controls

The notebook contains controls intended to prevent mechanically plausible but economically incorrect backtest results.

**Input integrity**

* Required ES columns are checked before the strategy runs.
* Duplicate futures dates trigger an error.
* Missing March (`ESH20`) or June (`ESM20`) settlement observations trigger an error.
* SPX and VIX9D observations must remain positive.
* Trade inception and option expiry must both exist in the aligned dataset.
* Negative time to expiry is prohibited.

**Contract-roll integrity**

Settlement differences are calculated separately for `ESH20` and `ESM20` before the active contract is selected. The contract held at the start of each interval decides the settlement change used for hedge P&L. This ensures the roll does not create artificial P&L from the price-level difference between March and June futures.

**P&L sequencing**

The futures quantity held at the start of an interval earns that interval's hedge P&L. Rebalancing occurs only after the current option state and residual delta have been calculated.

**Attribution reconciliation**

For every observation, the notebook verifies numerically that:

$$
Delta + Gamma + Theta + Residual = Option\ \text{P\\&L}
$$

The backtest raises an exception if the attribution does not reconcile.

**Terminal controls**

The notebook verifies that:

* The final observation is the option-expiry date.
* The final futures position is equal to zero. Closing this futures position generates turnover, incurring commission fees. This is reported in the results section.
* The final trade is explicitly recorded as `EXPIRY_CLOSE`.
* On expiry, the option is settled **at intrinsic value**. The Black-Scholes formula is not evaluated at T=0, to avoid numerical instability.

These checks prevent a reported terminal strategy value from containing an unrecognised open hedge position.

### Results

The theoretical benchmark produces the following strategy-level results over 10–20 March 2020:

| Metric                  |                      Result |
| ----------------------- | --------------------------: |
| Total strategy P&L      |             **+$18,375.21** |
| Theoretical option P&L  |                 +$35,678.62 |
| ES hedge P&L            |                 -$17,295.63 |
| Futures commissions     |                      -$7.77 |
| Total ES turnover       | 6.2188 contract equivalents |
| Delta-band rehedges     |                           4 |
| Futures rolls           |                           1 |
| Expiry hedge closes     |                           1 |
| Ending ES position      |                      0.0000 |
| Initial VIX9D proxy     |                      57.39% |
| Realised SPX volatility |                     117.95% |

Realised SPX volatility is calculated from the sample standard deviation of daily SPX log returns over the backtest window and annualised using the number of trading days: $$\(\sqrt{252}\)$$.

The option-price attribution generated by the previous-period Greeks is:

| Option P&L Attribution |             USD |
| ---------------------- | --------------: |
| Delta                  |     +$15,108.67 |
| Gamma                  | **+$30,570.82** |
| Theta                  |  **-$9,133.48** |
| Residual               |        -$867.39 |
| **Total option P&L**   | **+$35,678.62** |

The (Greeks) attribution is checked against the options P&L to ensure components add up correctly.

The notebook also produces daily strategy P&L, cumulative P&L and stacked option/hedge/commission component visualisations.

### Interpretation

The benchmark shows the economics of a long-gamma, delta-hedged options position during an extreme volatility episode.

The initial VIX9D input is approximately **57.4%**, compared with annualised realised SPX volatility of approximately **118.0%** over the backtest. This highlights the unusually large index moves following the trade start date. However, the comparison is only indicative and should not be interpreted as a clean implied-versus-realised volatility trade because VIX9D is a proxy, not strike-specific implied volatility of the straddle, and the model updates the VIX9D input in the daily option repricing.

The most significant attribution term is gamma. The gamma contribution is **+$30,570.82**, compared with **-$9,133.48** of theta decay. This shows the long-gamma trade-off which is key to the strategy: large underlying moves generate a large positive gamma contribution, while the option position incurs theta decay as it approaches expiry.

Delta contributes approximately **+$15,108.67** to option P&L, while the ES hedge loses **-$17,295.63**. These values should be considered together rather than interpreting the positive delta term as intentional directional profit, since the futures hedge is used to reduce the option's delta exposure. The combined contribution of the two terms is **-$2186.96**. The two terms do not offset exactly as hedging is discrete and only occurs when the delta band is breached. There is also a mismatch in the pricing used in the model: option delta uses SPX closing prices, whereas futures P&L is based on ES settlement prices. This results in movements in the SPX-ES basis leaving a residual difference between delta contribution and hedge P&L.

The aggregate attribution residual is approximately **-$0.9k**. This should not be interpreted as direct vega P&L. Daily changes in VIX9D, higher-order option effects and approximation error all enter this term.

Commission costs are minimal in this version because the theoretical benchmark applies a constant linear charge of $1.25 to fractional ES contract-equivalent turnover. This is a simplifying assumption rather than evidence that execution costs are unimportant to strategy outcome. The practical backtest is designed to test whether the economic viability of the strategy persists given more realistic transaction costs and allowing only integer values of ES contracts to be used as the hedge instrument.

The **+$18.4k** total P&L should therefore be interpreted as the result of a theoretical (control) experiment during the short and unusually volatile historical window of March 10-20 2020. It shows the mechanics of delta hedging, convexity (gamma) capture, theta decay and futures hedge accounting.

### Limitations

This notebook is designed as a theoretical benchmark, not a fully executable trading simulation. The main limitations are as follows:

- **Implied Volatility Proxy:** VIX9D is used as the volatility input to the Black-Scholes model rather than strike-specific implied volatility for the selected SPX call and put. This means the model produces proxy straddle value rather than reproducing the histroical market price of the actual position.

- **Fractional Futures:** The ES hedge can take fractional contract quantities. This isolates the effect of exact delta-hedging (the position is fully delta-neutral when rehedged). However, fractional ES contracts can not be traded in practice.

- **Discrete Hedging:** Hedging only occurs at the available daily observations and when residual SPX-equivalent delta exceeds the rehedge band.

- **Futures Transactions Costs are simplified:** A linear transaction cost of $1.25 per ES contract traded is applied to fractional hedge turnover. This is not realistic of fully executable trading costs. Bid-ask spread, slippage, market impact and liquidity effects are all ignored.

- **SPX and ES price conventions:** The option is marked using SPX closing levels while the hedge is valued using official ES settlement prices. Differences in timing and market conventions may introduce additional basis effects.

- **SPX–ES basis risk:** ES futures are used as a proxy hedge for SPX exposure. Changes in the futures basis can therefore cause hedge P&L to differ from the P&L implied by movements in the spot index alone.

- **Greek attribution:** Delta, gamma and theta are used to attribute the option-price change. The residual therefore contains the effects of changes in implied volatility, higher-order Greeks and approximation error, and should not be interpreted as pure vega P&L.

- **Short stress-period sample:** The analysis covers a small number of observations during the unusually volatile March 2020 market environment. Results therefore illustrate the mechanics of gamma scalping under stressed conditions rather than providing evidence of long-run strategy performance.

## 2. Executable Backtest

### Objective

The executable backtest extends the theoretical fractional-ES benchmark by testing how the strategy behaves when the futures hedge is implemented under more realistic trading constraints.

Its purpose is to measure how much of the theoretical result survives once the idealised hedge is replaced by a whole-contract approximation. The underlying straddle position, historical window, option valuation framework, delta-rehedging band and P&L accounting remain consistent to allow for a meaningful comparison with the theoretical benchmark.

Analysis therefore focuses on the effect of execution realism on strategy P&L, hedge effectiveness and residual directional (delta) exposure - particularly when whole ES contracts prevent each rehedge from returning the position to exact delta neutrality.

This notebook aims to establish a single practical base case, separate to the robustness analysis notebook. 

### Constraints Introduced

Three main changes are introduced to the theoretical hedge:

* **Whole ES contracts:**

Fractional ES contracts are replaced by integer-only positions. The fractional position is kept as a diagnostic, but the actual position held is rounded to the nearest whole contract.

* **Explicit Futures Commissions:**

Futures commissions are charged at **$1.25 per ES contract per side traded.** Therefore, costs are determined by integer contract turnover rather than fractional contract-equivalent turnover. Commission is applied whenever: *a futures transaction occurs (including opening the hedge), rehedging, contract-roll transactions and final hedge closure.*

* **Bid-ask spread and slippage:**

A trading-friction fee is added to the executable model which is not considered in the theoretical model. This is equal to:

- 0.5 ES tick for the assumed half-spread.
- 0.5 ES tick for the adverse execution slippage.

Therefore, combined friction is **1 ES tick per contract-side traded.** The 1-tick execution-friction, together with the ES contract value of $50 per index point determines the monetary cost of execution friction in the model. These costs are added to the commission charge and sum to total transaction fees.

The model does not contain timestamped executable hedge timing.

### Methodology Changes

The core option-pricing and attribution framework remains the same as the theoretical backtest. The 2880 strike price SPX straddle is still repriced using Black-Scholes, with VIX9D as the implied volatility input and ES provides the delta hedge.

The theoretical ES target remains in the notebook, as such:

$$
q_t^{ES,*}=-\Delta_t^{option}\frac{100}{50}
$$

but the executable target is obtained by rounding this value to the nearest whole ES contract. The notebook stores both values so the difference between the ideal and actual implemented hedge can be observed.

The 0.10 SPX-equivalent residual-delta band is also kept in the executable backtest. At each observation, the following occurs:

1. The futures position held at the start of the interval earns settlement-to-settlement P&L for that interval.
2. The current SPX straddle is valued using Black-Scholes and delta is calculated.
3. Residual delta is measured using the currently held futures position.
4. No transaction occurs where delta stays within the 0.10 band.
5. If the band is breached, the fractional ES target for the hedge is converted to the nearest integer-value number of contracts. 
6. If this integer differs from the current number of futures contracts held, the rehedge is executed.
7. If band is breached, but integer rounding keeps the target unchanged, the observation is recorded as a `GRANULARITY_HOLD`.

`GRANULARITY_HOLD` was introduced for situations where the delta remains outside the target delta band, but the position can not be improved through rehedging of ES contracts to a different integer value. The introduction of this allows distinction between situations such as these and a standard `HOLD` scenario.

Futures-roll accounting remains contract-specific. At the roll date, the existing contract is closed and the new contract is opened in two separate transactions. (Commissions and execution friction is applied to both legs).

At expiry, the ES position is closed incurring associated transaction costs.

## Validation and Backtest Controls

The executable model keeps all controls previously used in the theoretical model. It adds checks to:

* Ensure ES positions are integer values.
* Transaction costs must be calculated from number of contracts traded.
* Observations defined as `GRANULARITY_HOLD` are distinguished from actual rehedges.
* Ensure no commission or execution friction is incurred where no ES rehedge occurs.
* Futures position entering the interval is checked against the position carried forward to ensure consistency.

### Results

| **Metric**              | **Result**           |
|-------------------------|---------------------:|
| Option P&L              | **+$35,678.62**      |
| ES Hedge P&L            | **-$27,775.00**      |
| Futures Commissions     | **-$7.50**           |       
| Spread and Slippage     | **-$75.00**          |
| Total Execution Cost    | **-$82.50**          |
| **Total strategy P&L**  | **+$7,821.12**       |

The executable hedge generates **6 contracts of total turnover.** The residual-delta band produces **2 rehedges** and **1 `GRANULARITY_HOLD`** occurs where the band is breached but rounding to whole contracts leaves the implementable target unchanged. 

| **Hedge Metric**                | **Result**           |
|---------------------------------|---------------------:|
| Total ES contract turnover      | 6                    |
| Delta-band rehedges             | 2                    |
|`GRANULARITY_HOLD` events        | 1                    |
| Mean absolute residual delta    | 0.0747               |
| Maximum absolute residual delta | 0.1840               |
| Ending ES position              | 0                    |

The remaining futures hedge is closed at expiry - confirmed by the final ES position being equal to  **zero.**

Direct futures execution costs sum to **$82.50**. This can be decomposed into its constituent parts: **$7.50 of commission and $75.00 of bid-ask spread and slippage costs.**


### Comparison with the Theoretical Benchmark

The executable backtest produces a meaningfully lower strategy result than the theoretical benchmark. This is despite leaving the underlying option position and valuation framework (and accompanying inputs) unchanged.

| **Metric**          | **Theoretical Benchmark**       | **Executable Backtest**      | **Difference**        |
|---------------------|--------------------------------:|-----------------------------:|----------------------:|
| Total Strategy P&L  | **+$18,375.21**                 | **+$7,821.12**               | **-$10,554.09**       |
| Option P&L          | +$35,678.62                     | +$35,678.62                  | $0.00                 |
| ES hedge P&L        | **-$17,295.63**                 | **-$27,775.00**              | **-$10,479.37**       |
| Futures Commissions | -$7.77                          | -$7.50                       | +$0.27                |
| Spread and Slippage | $0.00                           | -$75.00                      | -$75.00               |
| Total Futures Costs | **-$7.77**                      | **-$82.50**                  | **-$74.73**           |
| ES turnover         | 6.2188 contract equivalents     | 6 contracts                  | -                     | 
| Delta-band rehedges | 4                               | 2                            | -2                    |
| Ending ES position  | 0                               | 0                            | -                     |

The identical option P&L of **+$35,678.62** confirms that the overall P&L difference of **-$10,554.09** is due to hedge implementation differences between the theoretical and executable notebooks. **-$10,479.37** of this difference comes from changes in the ES (futures) hedge P&L. Additional costs modelled account for a further **-$74.73** of the loss in profit. Therefore, the results show that the effect of enforcing realistic hedge rules (preventing fractional ES positions) is more important than commissions, spread and slippage in terms of the overall economic effect on strategy P&L.

Enforcing integer sized positions only also changes the outcome of the hedge itself. The executable model records **2** rehedge events compared with **four** in the theoretical model with fractional shares allowed. There is also one `GRANULARITY_HOLD` in the executable backtest where the band is breached but a move to a different integer-contract position does not reduce the delta of the position. This leaves a less precise hedge during market moves and alters futures P&L significantly. 

However, the results show that the same expiry date, roll accounting and hedge closure rules are applied across both notebooks. Therefore, the comparison isolates the effect of the lesser precision of the executable hedge relative to the theoretical hedge, combined with a minor contribution from increased total futures costs accumulated.


## 3. Sensitivity and Robustness Analysis

### Objective

### Delta-Band Sensitivity

### Transaction-Cost Sensitivity

### Volatility-Input Sensitivity

### Hedge Granularity

## Reproducing the Analysis

The current theoretical benchmark requires Python, access to the SPX and VIX9D historical series and the locally stored contract-specific ES dataset.

The notebook:

1. downloads daily `^GSPC` and `^VIX9D` observations for the configured backtest period using `yfinance`;
2. loads `Data/es_futures_mar2020.csv`;
3. validates and aligns the market datasets;
4. constructs the fixed-strike straddle and declining time-to-expiry series;
5. calculates Black-Scholes prices and Greeks;
6. runs the fractional-ES delta-hedging loop;
7. reconciles option attribution and terminal hedge state;
8. calculates realised volatility and summary statistics; and
9. generates the strategy P&L visualisations.

The local ES file must contain, at minimum:

```text
Date
ESH20_Settlement
ESM20_Settlement
```

The notebook will raise an error if these required settlement fields contain missing values or duplicate dates.

SPX and VIX9D are currently retrieved at run time rather than frozen locally. Exact reproducibility is therefore partially dependent on the continued availability and consistency of the upstream `yfinance` historical series. A fully frozen research dataset would remove this external dependency if exact archival reproducibility becomes necessary.

## Repository Structure

The project follows the planned three-stage research structure:

```text
OptionsVolatilityAnalysis/
│
├── OptionsVolatilityAnalysis_Theoretical_FractionalES.ipynb
│
├── Data/
│   └── es_futures_mar2020.csv
│
├── OptionsVolatilityAnalysis_Practical_IntegerES.ipynb      
├── OptionsVolatilityAnalysis_SensitivityAnalysis.ipynb      
│
├── README.md
└── requirements.txt
```

The theoretical notebook remains the controlled benchmark. The practical and sensitivity notebooks are deliberately separated so that execution realism and robustness analysis do not alter the assumptions of the reference model.

The practical engine is intended to be refactored into a callable backtest function before systematic sensitivity testing is introduced. Shared source modules can be added later if duplication across notebooks becomes sufficiently large to justify further abstraction.

## Requirements

Core Python dependencies used by the theoretical notebook are:

```text
yfinance
pandas
numpy
matplotlib
```

The notebook also uses Python's standard-library `statistics.NormalDist` implementation for the normal cumulative distribution function.

A working Jupyter environment is required to execute the notebook interactively.

The theoretical benchmark additionally requires the locally stored ES settlement dataset at:

```text
Data/es_futures_mar2020.csv
```

Internet access is required when rerunning the current implementation because SPX and VIX9D observations are downloaded using `yfinance`.

## Disclaimer

This repository is a research and educational project and does not constitute investment advice, a trading recommendation or evidence of expected future performance. Results from the theoretical benchmark depend on model assumptions, historical data, simplified transaction-cost treatment and a short stressed-market sample. The theoretical strategy permits fractional futures positions and does not represent a directly executable historical trading strategy.
