# Systematic Factor Portfolio

A quantitative long-only portfolio managing CAD 100M across North American equities and bonds. The system captures six academic risk premia through a fully automated pipeline — from data ingestion to position sizing — with an adaptive regime detection engine and a crash-protection overlay designed for institutional capital preservation.

**[Live Technical Overview](https://lfiset1111.github.io/systematic-factor-portfolio/)** · Built at Université Laval (GSF-3120, Winter 2026)

---

## Performance Summary

| Metric | Gross | Net of Fees |
|---|---|---|
| Annualized Return | 14.0% | 7.23% |
| Sharpe Ratio | 0.79 | — |
| Calmar Ratio | 0.99 | 0.51 |
| Max Drawdown | -14.1% | -14.1% |
| COVID-19 Drawdown | -8.7% | -8.7% |
| Volatility | 11.45% | 11.45% |
| Daily VaR (95%) | -1.10% | -1.10% |

The strategy was validated out-of-sample on 2020–2026 data (COVID-19, 2022 rate hikes, SVB crisis, 2025 tariff shock) with all hyperparameters frozen from the 2012–2019 training period. Gross OOS return: ~15% annualized.

---

## Architecture

```
753 Securities ──→ 6 Factors ──→ Regime Detection ──→ MV Optimizer ──→ ~13 Positions
  (S&P 500,        (Composite     (6 macro signals,    (Ledoit-Wolf      (Long-only,
   TSX, ETFs)       score)         binary decision)     shrinkage)        constrained)
```

### Factor Model

Six factors selected from peer-reviewed academic literature. Weights reflect published evidence, not in-sample optimization.

| Factor | Weight | Definition | Reference |
|---|---|---|---|
| Value | 35% | Earnings Yield (NI TTM / Mkt Cap) | Fama & French 1992 |
| Momentum | 24% | 12-month return, skip 1 month | Jegadeesh & Titman 1993 |
| Profitability | 15% | Gross Profit / Total Assets | Novy-Marx 2013 |
| PEAD | 10% | Standardized Unexpected Earnings | Bernard & Thomas 1989 |
| Investment | 8% | Inverse YoY asset growth | Fama & French 2015 |
| Low Volatility | 8% | Inverse 12-month realized vol | van Vliet 2024 |

Factor weights shift dynamically based on the detected market regime. In RISK_OFF, the model reduces momentum exposure (28% → 12%) and increases defensive factors (Low Volatility 2% → 15%, PEAD 12% → 20%).

A 45,801-combination grid search suggested alternative weights (PEAD at 37%, Value at 1%). This was deliberately not followed — Harvey, Liu & Zhu (2016) demonstrate that the majority of published factors are false positives when tested at appropriate significance thresholds.

### Regime Detection

A composite score built from six macro/technical signals, each producing a value between -1 (bearish) and +1 (bullish):

| Signal | Weight | Mechanism |
|---|---|---|
| VIX | 25% | RISK_OFF above 30, RISK_ON below 18 |
| Trend (SMA 50/200) | 25% | Death cross detection + 126-day momentum |
| Yield Curve | 15% | 10Y-2Y spread inversion |
| Credit Spreads | 10% | HY OAS percentile rank (252-day window) |
| Market Breadth | 10% | Percentage of stocks above SMA 200 |
| HMM (2-state) | 15% | Gaussian Hidden Markov Model, annual retraining |

The system operates in binary mode (RISK_ON / RISK_OFF) with a 5-day confirmation window to filter false signals. Binary mode was selected over continuous after controlled testing showed it reduced max drawdown from -17.6% to -14.1%.

### Three-Layer Risk Management

The strategy layers three independent protection mechanisms during market stress:

**Layer 1 — Factor Weight Adjustment.** When the regime switches to RISK_OFF, factor weights shift toward defensive premia (low volatility, PEAD) and away from momentum.

**Layer 2 — Risk-Managed Momentum (Barroso & Santa-Clara, 2015).** In RISK_OFF, the momentum calculation switches from the standard 12-month return to a 52-week high ratio (George & Hwang, 2004). The 52-week high signal is psychologically anchored and does not exhibit the crash behavior of classical momentum during market reversals.

**Layer 3 — Crash-Only Overlay.** An additional protection layer that activates only during severe stress. Entry requires RISK_OFF confirmation, at least 2 of 4 stress flags active (VIX, trend breakdown, realized volatility, credit spreads), and regime confidence above 70%. When active, the overlay applies a 0.70x multiplier to equity exposure (58.5% → ~41%), redistributing toward bonds. A 10-day cooldown prevents repeated activation/deactivation.

During COVID-19, this three-layer system limited the portfolio drawdown to -8.7% while the S&P 500 fell -34%.

### Optimization

Mean-variance optimization using Ledoit-Wolf shrinkage covariance (252-day lookback, annualized). The optimizer maximizes the Sharpe ratio subject to:

- Long only (w ≥ 0)
- Position bounds: 1% – 10%
- Bond floor: ≥ 25%
- Sector cap: ≤ 25% (GICS)
- Liquidity: ≥ $1M daily ADV
- Market impact: ≤ 10% of ADV

An adaptive smoothing mechanism balances stability and reactivity:
- Normal regime: λ = 0.40 (gradual adjustment)
- Regime change: λ = 0.85 (fast reaction)
- Degenerate state (>50% cash): λ = 1.00 (full reset)

---

## Out-of-Sample Validation

All hyperparameters — factor weights, regime thresholds, overlay rules, allocation splits, smoothing parameters, optimization constraints — were frozen using 2012–2019 data. The model was then run on 2020–2026 without modification.

| Metric | In-Sample | Out-of-Sample | Ratio |
|---|---|---|---|
| Gross Ann. Return | ~14% | ~15% | 1.07x |
| Net Ann. Return | 7.11% | 5.57% | 0.78x |
| Calmar Ratio | 0.399 | 0.213 | 0.53x |

The OOS period includes COVID-19, the fastest rate-hiking cycle in 40 years, the SVB banking crisis, and the 2025 tariff shock. The regime detector spent 36.6% of the OOS period in RISK_OFF (vs. 17.6% in-sample), activating correctly during each major stress event.

Adaptive components (covariance matrix, factor scores, regime state) update with new market data as they would in live deployment. Only the structural parameters are frozen.

---

## Data

| Source | Data | Access |
|---|---|---|
| Yahoo Finance | Prices, volumes, VIX, FX rates | `yfinance` Python package |
| Compustat via WRDS | Quarterly fundamentals (income, assets, earnings dates) | Institutional subscription |
| FRED | Treasury yields (DGS10, DGS2), HY OAS, risk-free rate | Public API |

All fundamental data respects strict point-in-time constraints with a 45-day SEC reporting lag. Raw data files are not included in this repository per WRDS Terms of Use. To reproduce results, obtain WRDS access through your institution.

> *Wharton Research Data Services (WRDS) was used in preparing this research project. This service and the data available thereon constitute valuable intellectual property and trade secrets of WRDS and/or its third-party suppliers.*

---

## Tech Stack

- **Python 3.10+** — core language
- **cvxpy** — convex optimization (mean-variance)
- **hmmlearn** — Gaussian Hidden Markov Model (regime detection)
- **scikit-learn** — Ledoit-Wolf shrinkage estimator
- **pandas / numpy / scipy** — data processing and statistical analysis
- **yfinance** — market data ingestion
- **plotly** — interactive visualization
- **pandas_market_calendars** — trading calendar validation (NYSE, TSX)

---

## Repository Structure

```
├── index.html       # Interactive technical overview (deployed via GitHub Pages)
├── README.md        # This file
```

Full source code (Jupyter notebook with ~11,000 lines across 15+ modules) is available upon request.

---

## Known Limitations

**Positive equity-bond correlation (2022).** When equities and bonds decline simultaneously, the defensive rotation toward bonds becomes counterproductive. The portfolio lost -12.1% vs. -8.0% for a static allocation during this period. This is a structural limitation of any risk-on/risk-off framework and is documented as a known risk.

**Benchmark selection.** The current benchmark (S&P 500) does not reflect the portfolio's mixed equity-bond allocation. A composite benchmark (e.g., 60/40 equity-bond blend) would provide a fairer comparison and isolate alpha from factor selection vs. asset allocation.

**Survivorship bias.** The backtest universe includes 21 historical tickers but is missing hundreds of delisted S&P 500 constituents. Integration of CRSP data via WRDS would address this limitation.

**Turnover.** Annual turnover of 284% remains elevated despite smoothing and no-trade bands. This generates transaction costs and requires institutional-grade execution infrastructure.

**Factor decay.** McLean & Pontiff (2016) document a 26% average decline in factor premia after academic publication. Diversification across six low-correlation factors mitigates but does not eliminate this risk.

**ESG integration.** The current ESG tilt uses a sector-level proxy rather than security-level scores, providing limited differentiation within sectors.

---

## Planned Improvements

### Execution & Trading Infrastructure
- **Market impact model (Almgren-Chriss):** Estimate execution costs as a function of order size, ADV, and volatility using TAQ or Bloomberg data
- **VWAP/TWAP execution algorithms:** Slice block orders according to historical intraday volume profiles to minimize market impact
- **Transaction Cost Analysis (TCA):** Post-trade reporting module measuring implementation shortfall, timing cost, and opportunity cost per rebalancement

### Bond Factor Model (BondSelector v2)
The current bond scoring model (momentum + volatility) would be replaced by a multi-factor approach using Bloomberg/Refinitiv data:

| Factor | Weight | Source | Expected Impact |
|---|---|---|---|
| Carry (YTM) | 30% | Bloomberg `YAS_YLD_SPREAD` | +30–50 bps |
| Rate Momentum | 20% | Bloomberg `USGG10YR` / `USGG2YR` | +15–25 bps |
| Spread Momentum | 25% | Bloomberg `OAS_SPREAD_BID` | +20–40 bps |
| Rolldown | 10% | Bloomberg yield curves `YCGT0025` / `YCGT0016` | +10–20 bps |
| Duration Timing | 15% | Yield curve slope + VIX + regime | +20–30 bps |

### Data & Methodology
- **Composite benchmark:** Add a 60/40 blended benchmark (40% SPY + 20% XIU.TO + 25% AGG + 15% XBB.TO) to isolate factor alpha from asset allocation effects
- **Variable risk-free rate:** Replace fixed 5% Rf with FRED DTB3 series
- **Survivorship bias correction:** Integrate CRSP delisted tickers via WRDS
- **Granular ESG:** Replace sector-level proxy with security-level Sustainalytics scores via Bloomberg
- **Dynamic liquidity constraints:** Replace static $1M/day floor with ADV-based constraints (max 10–15% of 20-day ADV per position)
- **Alternative safe havens:** Add gold and commodity ETFs (GLD, IAU, DJP) to address the 2022 equity-bond correlation breakdown
- **HMM/VIX redundancy analysis:** Quantify signal overlap via Spearman correlation and adjust weights accordingly

---

## Author

**Laurie Fiset** — Université Laval, BSc Quantitative Finance

Developed as part of a four-person team in GSF-3120 (Quantitative Finance Seminar), Winter 2026. The project involved designing, implementing, backtesting, and presenting a systematic portfolio strategy to a simulated investment committee.

---

## License

This project was developed for academic purposes. The code and methodology are shared for educational and portfolio demonstration purposes only. Nothing in this repository constitutes investment advice.
