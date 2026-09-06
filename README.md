# Options Volatility Analysis

Options Volatility Analysis is a Python-based derivatives research project examining the economics of a dynamically delta-hedged long SPX straddle when implied volatility differs from realised volatility. The analysis uses Black-Scholes option pricing and Greeks with SPX, VIX9D and E-mini S&P 500 futures data to study how convexity, time decay, dynamic hedge P&L and transaction costs interact over the life of a short-dated option position.

The project is structured as a progression from a controlled theoretical benchmark: `OptionsVolatilityAnalysis_Theoretical_FractionalES.ipynb` to an executable base-case notebook: `OptionsVolatilityAnalysis_ExecutableBacktest.ipynb` and then to formal robustness testing: ``. The theoretical benchmark uses fractional ES contract equivalents to isolate the mechanics of dynamic delta hedging. Due to the fractional ES contracts, a fully delta-neutral position can be created at the time of the rehedge. This leaves no first-order directional exposure to changes in the price of the underlying asset (in this case the S&P 500 index).

It models contract-specific futures P&L across the March 2020 ES roll, a residual-delta rehedging rule (10% SPX-equivalent delta of residual exposure), option repricing through expiry, futures commissions, Greek P&L attribution, terminal hedge closure and reconciliation controls.

The executable backtest relaxes these simplifying assumptions where possible. Fractional futures are replaced with integer ES contracts, more realistic commissions are introduced and bid-ask spreads and slippage are implemented. The March 2020 ES roll is represented as separate closing and opening transactions and transaction costs are applied to both legs. Limitations that remain are: executable hedge timing and realistic treatment of the SPX-ES basis.

Further extensions to this project could add actual option prices and strike-specific implied volatility (current data constraints prevent this for the chosen backtest window). Intraday hedge execution could also be added in future, with greater data access. These are therefore not included in the notebook and if implemented, would have to be applied to both the theoretical (control) and executable backtest notebooks to ensure a valid comparison to the benchmark (theoretical) model.

A separate sensitivity framework tests the robustness of results to the delta-rehedge band, transaction costs, volatility assumptions, fractional versus integer hedge granularity and hedge frequency at frequencies supported by the available data. The outputs include total P&L, turnover, number of rehedges, transaction costs, residual delta exposure and comparative sensitivity plots and tables.

## Research Question

How does a dynamically delta-hedged long SPX straddle perform when implied volatility differs from realised volatility and how do ES hedging constraints affect P&L, hedge effectiveness and its decomposition inot option Greeks, futures hedge P&L, transaction costs and residual repricing effects?

## Project Architecture

| Notebook                                   | Purpose                                                                                    | Status         |
| ------------------------------------------ | ------------------------------------------------------------------------------------------ | -------------- |
| Theoretical Fractional-ES Benchmark        | Isolates dynamic gamma-scalping mechanics under controlled hedge assumptions               | Complete       |
| Practical Integer-ES / Executable Backtest | Introduces discrete contract sizing and more realistic execution constraints | Complete |
| Sensitivity Analysis                       | Tests robustness to modelling, volatility, cost and hedge assumptions                      | |

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

A data limitation arose when constructing the E-mini S&P 500 futures hedge around the March 2020 contract roll. Databento CME statistics data do not provide a continuous daily fixing series for the June contract (`ESM20`) over the backtest window. Therefore, the the official CME daily settlement prices for March (`ESH20`) and June are used instead. This creates a consistent marking convention without unneccessary intepolation or introducing another proxy.

The hedge is held in March ES before **12 March 2020**. After this date, it is held in June ES (post-roll date). Settlement changes are calculated separately within each contract. P&L for the interval ending on the roll date is genreated by the March contract held over the interval. After this, P&L is recognised, the March position is closed and the June contract hedge is established.

Price-levle difference between the two contracts is therefore not recognised as P&L, ensuring correct accounting. This also ensures that the hedge remains contract-consistent. 

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

The residual is intentionally **not** labelled vega P&L. Due to the fact that the option is repriced each day using a changing VIX9D input, the residual can contain the effect of **volatility-input changes, higher-order Greeks, interaction terms and error from the discrete second-order approximation.**

Futures hedge P&L is calculated separately:

$$
\text{P\\&L}^{ES}_t = q^{ES}_{t-1}\times50\times\Delta F_t
$$

using the futures quantity actually held during the interval and the settlement change of the contract held over that interval.

Total strategy P&L is therefore:

$$
\text{P\\&L}^{strategy}_t = \text{P\\&L}^{option}_t + \text{P\\&L}^{ES}_t - Cost_t
$$

This separation is important: Greek attribution explains the theoretical **option-price change**, whereas ES P&L records the performance of the **actual hedge instrument** used by the strategy.

### Validation and Backtest Controls

The notebook contains explicit controls intended to prevent mechanically plausible but economically incorrect backtest results.

**Input integrity**

* Required ES columns are checked before the strategy runs.
* Duplicate futures dates trigger an error.
* Missing March or June settlement observations trigger an error.
* SPX and VIX9D observations must remain positive.
* Trade inception and option expiry must both exist in the aligned dataset.
* Negative time to expiry is prohibited.

**Contract-roll integrity**

Settlement differences are calculated separately for ESH20 and ESM20 before the active contract is selected. The contract held at the start of each interval decides the settlement change used for hedge P&L. This ensures the roll can not create artificial P&L from the price-level difference between March and June futures.

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

* The final observation is the specified option-expiry date.
* The final futures position equals zero.
* The final trade is explicitly recorded as `EXPIRY_CLOSE`.

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

Realised SPX volatility is calculated from the sample standard deviation of daily SPX log returns over the backtest window and annualised using $$\(\sqrt{252}\)$$.

The option-price attribution generated by the previous-period Greeks is:

| Option P&L Attribution |             USD |
| ---------------------- | --------------: |
| Delta                  |     +$15,108.67 |
| Gamma                  | **+$30,570.82** |
| Theta                  |  **-$9,133.48** |
| Residual               |        -$867.39 |
| **Total option P&L**   | **+$35,678.62** |

The attribution therefore reconciles to the theoretical option repricing by construction and validation.

The notebook also produces daily strategy P&L, cumulative P&L and stacked option/hedge/commission component visualisations. Non-trading calendar dates are inserted for presentation purposes only and do not alter the underlying strategy calculations.

### Interpretation

The benchmark illustrates the economics of long convexity during an extreme volatility episode.

The initial VIX9D input is approximately **57.4%**, while annualised realised SPX volatility over the subsequent sample is approximately **118.0%**. This comparison is consistent with a period in which subsequent underlying movement was exceptionally large relative to the short-dated volatility level observed at inception. It should not, however, be interpreted as a clean ex-ante implied-versus-realised volatility trade because VIX9D is updated throughout the backtest and therefore also affects daily theoretical option repricing.

The most economically significant attribution term is gamma. The second-order gamma contribution is approximately **+$30.6k**, compared with approximately **-$9.1k** of theta decay. This demonstrates the core long-gamma trade-off within the benchmark: convexity generated positive attribution from large underlying moves, while the long-option position continuously paid for that convexity through negative theta.

Delta attribution contributes approximately **+$15.1k** to theoretical option P&L, while the ES hedge loses approximately **$17.3k**. These two quantities should be considered jointly rather than interpreting the positive delta term as intentional directional profit. Their combined contribution is approximately **-$2.2k**. The incomplete offset reflects the fact that hedging is discrete and band-based and that option delta is generated from SPX closing levels while the actual hedge P&L is generated from ES settlement changes. SPX-ES basis movement and timing differences therefore prevent the futures hedge from being a mathematically exact offset to the Black-Scholes delta attribution.

The aggregate attribution residual is approximately **-$0.9k**. This relatively small net residual does not imply that volatility changes were unimportant on individual days, nor should it be interpreted as direct vega P&L. Daily changes in VIX9D, higher-order option effects and approximation error all enter this term.

Commission costs are economically negligible in this version because the theoretical benchmark applies a constant linear charge of $1.25 to fractional ES contract-equivalent turnover. This is an intentional simplifying assumption rather than evidence that execution costs are unimportant. The practical backtest is designed specifically to test how much of the theoretical result survives once discrete contract sizing and more realistic execution frictions are introduced.

The **+$18.4k** total P&L should therefore be interpreted as the result of a controlled theoretical experiment during a short and unusually volatile historical window. It demonstrates the mechanics of delta hedging, convexity capture, theta decay and futures hedge accounting; it is not evidence of a persistent trading edge or a directly executable historical return.

### Limitations

This notebook is intentionally designed as a theoretical benchmark rather than a fully executable trading simulation.

* **VIX9D is a volatility proxy.** The Black-Scholes volatility input is the observed VIX9D level rather than the strike-specific implied volatility of the exact 2,880 SPX call and put. The theoretical option values therefore do not reproduce historical traded option prices.

* **Volatility surface effects are omitted.** A single volatility input does not capture strike skew, smile dynamics or the full term structure of SPX implied volatility.

* **Fractional futures are not executable.** ES exposure can take non-integer contract quantities. This deliberately removes hedge granularity from the theoretical benchmark but cannot be replicated directly in live futures trading.

* **Hedging is discrete.** The strategy observes the market daily and rebalances only when residual delta exceeds the configured threshold. It therefore represents discrete gamma scalping rather than continuous delta hedging.

* **SPX and ES use different marking conventions.** The option is marked from SPX closes while futures hedge P&L is generated from official ES settlement prices. Differences in observation timing can create additional hedge mismatch.

* **SPX-ES basis risk remains.** ES futures are a proxy hedge for SPX exposure. Futures-basis changes can therefore cause hedge P&L to differ from the offset implied by a pure movement in the spot index.

* **Futures transaction costs are simplified.** A linear transaction cost of $1.25 per ES contract equivalent traded is applied to fractional hedge turnover - this is a modelling assumption rather than an estimate of fully executable trading costs. Bid-ask spread, slippage, market impact and liquidity effects are omitted.

* **Option execution is not modelled.** Option bid-ask spreads, execution costs, liquidity and deviations between traded prices and Black-Scholes theoretical marks are excluded.

* **The Black-Scholes marking framework is simplified.** The benchmark assumes a zero risk-free rate and does not separately model an index dividend yield or a richer cost-of-carry framework.

* **Greek attribution is approximate.** Delta, gamma and theta form a discrete second-order attribution based on previous-period Greeks. The residual includes volatility-input changes, higher-order Greeks, cross-effects and approximation error and must not be interpreted as pure vega.

* **Exchange-specific option settlement is abstracted.** The benchmark terminates the theoretical straddle at intrinsic value using the terminal SPX observation rather than reconstructing the complete settlement mechanics of a specific historically traded option contract.

* **The sample is deliberately short and stressed.** The analysis contains nine observations during the March 2020 market shock. The results illustrate strategy mechanics under unusually high volatility and cannot establish long-run expected returns, statistical significance or robustness across regimes.

These limitations define the purpose of the benchmark rather than invalidate it. They establish a controlled reference case against which the subsequent executable and sensitivity notebooks can measure the effect of progressively more realistic assumptions.

## 2. Executable Backtest

### Objective


The executable backtest extends the theoretical fractional-ES benchmark by testing how the strategy behaves when the futures hedge is implemented under more realistic trading constraints.

Its purpose is to measure how much of the theoretical result survives once the idealised hedge is replaced by an implementable approximation. The underlying option position, historical window and core valuation framework remain sufficiently consistent to preserve a meaningful comparison with the theoretical benchmark.

Analysis therefore focuses on the effect of execution realism on strategy P&L, hedge effectiveness and residual directional (delta) exposure - particularly when whole ES contracts prevent each rehedge from returning the position to exact delta neutrality.

This notebook aims to establish a single practical base case, separate to the robustness analysis notebook. 

### Constraints Introduced

### Methodology Changes

### Results

### Comparison with the Theoretical Benchmark

## 3. Sensitivity and Robustness Analysis

### Objective

### Delta-Band Sensitivity

### Transaction-Cost Sensitivity

### Volatility-Input Sensitivity

### Hedge Granularity and Frequency

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
├── OptionsVolatilityAnalysis_Practical_IntegerES.ipynb      # planned
├── OptionsVolatilityAnalysis_SensitivityAnalysis.ipynb      # planned
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
