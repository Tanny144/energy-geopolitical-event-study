# Energy Commodities & Geopolitical Risk: An Event Study

> How much do oil, gas, freight, tanker and coal prices move when a geopolitical shock hits an energy chokepoint, and is the move statistically real?

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Tannybih/energy-geopolitical-event-study/blob/main/energy_event_study.ipynb)
![Python](https://img.shields.io/badge/python-3.10+-blue) ![License](https://img.shields.io/badge/license-MIT-green)

**Author:** Tanmay Agrawal · BSc Accounting & Finance, Queen Mary University of London

---

## Key findings (16 events, CAR over the 5 trading days after each shock)
| Result | Value |
|---|---|
| Largest average reaction | Crude tankers (Frontline) **+3.2%** (95% CI −2.5% to +9.9%) |
| Statistically significant at 5% | **None**: the average price effect across mixed shocks is not reliable |
| Volatility after shocks | Dry-bulk freight realised volatility **+37%** (median) |
| Chokepoint events (Hormuz, Red Sea) | Dry-bulk freight **+8.4%** on average |
| Safe-haven control | Gold **−0.3%** (no flight-to-safety signal) |

**Interpretation:** markets appear to price much geopolitical risk *before* the headline, and the reaction depends on whether supply is physically disrupted. The clearest effect is on **risk (volatility)** rather than on average price direction. The next step is to separate supply-disrupting events from threat-only events.

![CAR paths](outputs/figures/01_car_paths.png)
![Event heatmap](outputs/figures/03_event_heatmap.png)

## 1. Project overview
This project measures the abnormal price reaction of 8 energy-linked assets to 16 geopolitical events between 2019 and 2025: Hormuz tanker attacks, the Abqaiq strike, Red Sea Houthi attacks, Russia–Ukraine, the Israel–Iran escalations, and OPEC+ shocks. It uses the standard event-study framework from academic finance (MacKinlay, 1997). For each asset it estimates a "normal" return model on clean pre-event data, isolates the abnormal return around each event, and tests whether the average cumulative reaction differs from zero.

**Motivation.** My family runs a coal and energy trading business in India. During the Strait of Hormuz tensions I watched freight and coal costs move overnight on news from thousands of miles away. This project quantifies what I'd seen first-hand.

## 2. Real-world finance use case
| Who | How they use it |
|---|---|
| Commodity trading desks | Size event-risk hedges and set stop levels before known flashpoints |
| Macro hedge funds | Decide whether to fade or follow the initial move (the drift analysis in §11) |
| Bank market risk teams | Calibrate stress scenarios (for example, "Hormuz closure = +X% Brent in 5 days") |
| Shipping and energy corporates | Plan freight and fuel hedging around chokepoint risk |
| Wealth managers | Explain to clients why gold or energy positions moved after a headline |

## 3. System architecture
```
 ┌────────────────┐   ┌──────────────────┐   ┌───────────────────┐   ┌────────────────────┐
 │ Event catalogue │   │ Yahoo Finance API │   │  Cleaning layer    │   │  Event-study engine │
 │ (CSV, 16 events)│──▶│ batched download  │──▶│ calendar align,    │──▶│ OLS market model,   │
 │ date·event·type │   │ retry + cache     │   │ log returns, WTI<0 │   │ AR, CAR, σ_est      │
 └────────────────┘   └──────────────────┘   │ mask, outlier flag │   └─────────┬──────────┘
                                              └───────────────────┘             │
          ┌──────────────────────────────┬───────────────────────────┬──────────┴───────────┐
          ▼                              ▼                           ▼                      ▼
 ┌──────────────────┐        ┌────────────────────┐      ┌──────────────────┐    ┌──────────────────┐
 │ Statistical tests │        │ Robustness checks   │      │ Volatility shift  │    │ Drift / trading   │
 │ t, BMP, sign,     │        │ by category, by     │      │ pre vs post,      │    │ lens: CAR[+1,+5], │
 │ bootstrap CI      │        │ model (market/mean) │      │ Wilcoxon          │    │ hit rate          │
 └────────┬─────────┘        └─────────┬──────────┘      └────────┬─────────┘    └────────┬─────────┘
          └──────────────────────┬─────┴──────────────────────────┴──────────────────────┘
                                 ▼
             5 publication-quality charts · Excel workbook · auto-written findings
```

## 4. APIs and data sources
| Source | Used for | Key needed? |
|---|---|---|
| **Yahoo Finance** (via `yfinance`) | Daily prices: `BZ=F` Brent, `CL=F` WTI, `NG=F` Henry Hub gas, `BDRY` dry-bulk freight ETF, `FRO` Frontline (tankers), `BTU` Peabody (coal), `XLE` energy sector, `GC=F` gold, `SPY` market | No |
| Hand-curated event list | 16 dated geopolitical and supply shocks (editable CSV inside the notebook) | No |
| *Optional upgrades:* FRED, EIA, GDELT | Cleaner Brent spot series, official inventories, automatic news-based events | Free key |

## 5. Python libraries
`yfinance` · `pandas` · `numpy` · `scipy` (t-test, Wilcoxon, binomial) · `matplotlib` · `openpyxl` (Excel export). All except `yfinance` come preinstalled in Colab.

## 6. Folder / file structure
```
energy-geopolitical-event-study/
├── energy_event_study.ipynb     # the full project: run top to bottom in Colab
├── README.md
├── requirements.txt
├── data/
│   └── prices.parquet           # auto-created cache (git-ignored)
└── outputs/
    ├── event_study_results.xlsx # all result tables, one sheet each
    ├── abnormal_returns.csv     # event × day × asset ARs
    └── figures/
        ├── 01_car_paths.png
        ├── 02_mean_car_bars.png
        ├── 03_event_heatmap.png
        ├── 04_brent_timeline.png
        └── 05_vol_shift.png
```

## 7. Step-by-step build guide
1. **Setup:** install `yfinance`, import libraries, fix the random seed, create the output folders.
2. **Config:** put every research choice in one frozen dataclass (windows, model, tickers).
3. **Events:** define the catalogue as CSV, then validate it (dates, duplicates, required columns).
4. **Data:** run a batched download with retries and caching.
5. **Cleaning:** align to the market calendar, mask non-positive prices, compute log returns, flag outliers, snap events to trading days.
6. **Model:** estimate α, β and σ per asset per event on the estimation window, then compute the abnormal returns.
7. **Testing:** compute CARs over 3 windows, then run the cross-sectional t, BMP, sign test and bootstrap CI.
8. **Robustness:** split results by event category and swap the normal-return model.
9. **Volatility:** compare 20-day realised volatility before and after each event.
10. **Charts:** produce 5 figures saved at 200 dpi.
11. **Trading lens:** measure day-0 reaction vs follow-through.
12. **Export:** write the Excel workbook and CSV, and auto-generate the findings text.

## 8. Data collection pipeline
* One `yf.download` call for all 9 tickers (faster than looping, and lighter on rate limits).
* `auto_adjust=True` gives split- and dividend-adjusted closes.
* 3 attempts with exponential backoff (2 s, 4 s, 8 s), then a clear error.
* Tickers that return no data are dropped with a warning rather than crashing the run.
* Results are cached to Parquet, falling back to CSV, so re-runs are instant and work offline.

## 9. Data cleaning & feature engineering
| Step | Why |
|---|---|
| Reindex to the S&P 500 trading calendar | Futures and US equities have different holiday calendars |
| Forward-fill gaps of 3 days or fewer | Fills holiday mismatches without inventing long runs of data |
| Mask prices ≤ 0 | WTI settled at **−$37.63 on 20 Apr 2020**, where log returns are undefined |
| Log returns | Additive over time, so CAR is a plain sum |
| Flag \|z\| > 8 moves (keep them) | Transparency: the extreme days *are* the signal |
| Snap event date to the next trading day | Weekend events (Abqaiq, Hamas 7 Oct, US strikes 22 Jun 2025) map correctly |

## 10. Core models / algorithms
**Market model** (default), estimated by OLS on 120 trading days ending 15 days before the event:

$$R_{i,t} = \alpha_i + \beta_i R_{m,t} + \varepsilon_{i,t}$$

**Abnormal return and CAR** over event-time τ ∈ [−5, +10]:

$$AR_{i,\tau} = R_{i,\tau} - (\hat\alpha_i + \hat\beta_i R_{m,\tau}), \qquad CAR_i(\tau_1,\tau_2)=\sum_{\tau=\tau_1}^{\tau_2} AR_{i,\tau}$$

**Significance tests:**
* *Cross-sectional t-test.* Robust to the variance spikes that shocks cause.
* *BMP standardised test* (Boehmer, Musumeci & Poulsen, 1991). Scales each CAR by σ·√L.
* *Sign test.* Non-parametric, so it isn't driven by a single giant event.
* *Bootstrap 95% CI.* 5,000 resamples of the event-level CARs.

**Robustness:** a constant-mean model (β = 0); results split by event category (chokepoint, conflict, supply); a Wilcoxon signed-rank test on the volatility shift.

## 11. Visualisations & dashboard components
| Figure | Question it answers |
|---|---|
| `01_car_paths`: 6-panel mean CAR with bootstrap bands | How does each asset react day by day around the shock? |
| `02_mean_car_bars`: ranked mean CAR[0,+5] with CI and significance stars | Which assets react most, and is it real? |
| `03_event_heatmap`: event × asset CAR grid | Which specific shocks moved which markets? |
| `04_brent_timeline`: Brent with events colour-coded by type | Where do the events sit in the price history? |
| `05_vol_shift`: before/after volatility dumbbell chart | Does risk rise after the shock? |

## 12. Performance metrics
* Mean and median CAR over [−1,+1], [0,+5] and [−5,+10]
* 95% bootstrap confidence intervals
* t-stat and p-value (cross-sectional), BMP t and p, sign-test p
* Share of events with a positive CAR
* Realised volatility change (pre vs post), with the Wilcoxon p-value
* Drift lens: day-0 AR, CAR[+1,+5], continuation rate, hit rate, best and worst case

## 13. Final deliverables
* A Colab notebook that runs end-to-end with no API keys
* An Excel workbook with every result table
* 5 high-resolution charts for the README or slides
* Auto-generated plain-English findings
* This README, written for GitHub and for discussion in interviews

## 14. Resume description
> **Energy Commodities & Geopolitical Risk Event Study** (Python, pandas, SciPy) · *Oct 2026*
> • Built an event-study engine measuring abnormal returns of 8 energy assets (Brent, gas, freight, tankers, coal) across 16 geopolitical shocks from 2019 to 2025, using market-model OLS and bootstrap confidence intervals.
> • Found no significant average price reaction across shocks, but dry-bulk freight volatility rose 37% and chokepoint events lifted freight 8.4%; validated with 3 statistical tests and 2 return models.

## 15. Potential upgrades
1. **GDELT news feed:** detect events automatically and remove hindsight selection bias.
2. **Clustered standard errors** (Kolari & Pynnönen) for overlapping events, such as June 2019 and June 2025.
3. **GARCH abnormal volatility:** model the variance shock, not just the return.
4. **India lens:** add Coal India (`COALINDIA.NS`), USD/INR and Indian refiners to connect to emerging-market impact.
5. **Freight indices:** the Baltic Dry and tanker rate indices (paid) instead of the ETF proxy.
6. **Streamlit dashboard:** let users pick an event and an asset, and see the CAR live.
7. **Options-implied view:** OVX (oil volatility index) reaction as a forward-looking risk measure.

---

### How to run
1. Open the notebook in Google Colab.
2. Choose **Runtime → Run all**.
3. Collect the results from `outputs/`.

### References
* MacKinlay, A. C. (1997). Event Studies in Economics and Finance. *Journal of Economic Literature*, 35(1).
* Boehmer, E., Musumeci, J., & Poulsen, A. (1991). Event-study methodology under conditions of event-induced variance. *Journal of Financial Economics*, 30(2).
* Kolari, J., & Pynnönen, S. (2010). Event study testing with cross-sectional correlation of abnormal returns. *Review of Financial Studies*, 23(11).

### Disclaimer
This project is for education and research only, and is not investment advice. Event dates were compiled by hand; verify them against primary sources.
