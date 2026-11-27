# **🎓 Capstone: Mapping the Arbitrage Corridor on the December 2026 Contract**

In this Chapter, we built an empirical pricing engine from the ground up—aligning exchange settlement dates, deriving implied forward rates, valuing a shrinking pool of dividends, and backing out the daily **Implied Repo Rate** for the September 2026 E-mini S&P 500 contract (`ESU26`). We also saw why theoretical arbitrage lines bend into dynamic **"Express Lane" arbitrage bands** once prime broker balance-sheet costs, stock-borrow fees, and annualization noise enter the picture.

Once you know how to align multi-source futures, rate, and dividend series, you can apply this exact pipeline to evaluate pricing efficiency across other contract cycles—testing how real-world institutional frictions twist observed prices away from textbook carry.

Now, it is time to put that workflow into practice.

### **🎯 The Mission**

Grab a partner or work solo—in this capstone challenge, your objective is to roll the pipeline forward one quarter to the **December 2026 E-mini S\&P 500 contract (`ESZ26`)** and:

- Compute its daily Implied Repo Rate.
- Compare it against benchmark measures of institutional friction.

:::{admonition} 📚 Learning Objectives
:class: tip

By the end of this challenge, you will be able to:

**Data Pipeline & Calendar Alignment**
- **Acquire and align** multi-source time-series data for December E-mini S\&P 500 futures (`ESZ26`), September SOFR futures (`SR3U26`), spot index levels, and Q4 dividend distributions.
- **Resolve** exchange holiday discrepancies and maturity mismatches—re-indexing futures to `session_end_date` and bridging the short date gap between the SOFR reference end date and the December 18, 2026 S\&P expiration.

**Dynamic Dividend & Repo Valuation**
- **Implement** the shrinking-pool daily dividend valuation loop using point-in-time forward rates and the Actual/360 day-count convention.
- **Compute** the daily Implied Repo Rate across Q4 2026 and isolate genuine financing spreads from end-of-contract annualization noise.

**Arbitrage Band Modeling**
- **Construct and visualize** asymmetric upper and lower no-arbitrage corridors incorporating prime broker spreads, stock-borrow haircuts, and amortized execution slippage.
:::

---

*Ready to build your model? Set up your environment, and let's get to work\!*

### **🛠️ The Starting Point**

To complete this challenge, you will use the modules and helper functions introduced earlier in this notebook.

**1. Imports and Instantiations**
- Import `financial_quant` as `fq`
- Import `pandas` as `pd` and `matplotlib.pyplot` as `plt`
- Instantiate your data readers:
  - `fred_data` using `fq.FredReader()`
  - `massive_data` using `fq.MASSIVEReader(calls_per_minute=5)`

**2. Required Functions**
Ensure the custom functions defined earlier in this chapter are loaded into your environment:
- `fq.futures_session_end_index`
- `forward_curve`
- Daily shrinking-pool `Dividend FV` valuation logic

*Tip: Run all setup cells earlier in the notebook before attempting the steps below.*

**3. Get and Organize Data**
- Fetch the Series:
  - December 2026 E-mini S\&P 500 contract (`ESZ26`)
  - September 2026 SOFR futures contract (`SR3U26`)
  - S\&P 500 spot index
  - Q4 S\&P 500 dividend schedule
- Re-index futures contracts to `session_end_date`
- Calculate forward rates and bridge the gap between `SR3U26` and `ESZ26` expiration dates

**4. Compute Daily Dividend FV and Implied Repo Rates**
- Filter strictly future ex-dates (`index > current_date`)
- Compound remaining dividends via Actual/360
- Solve for the daily Implied Repo Rate

**5. Define Arbitrage Bands**

> Real-world index arbitrage faces asymmetric financing costs plus fixed execution slippage that scales inversely with time to expiration. Construct three benchmark series around your daily SOFR benchmark:

- **Amortized Slippage:** Annualize `0.0005` (5 bps) of round-trip execution friction over the remaining days to expiration ($0.0005 \times \frac{360}{\text{Days to Expiration}}$).
- **Upper Band (Cash & Carry Ceiling):** Add `0.0060` (60 bps) prime broker equity financing spread plus amortized slippage to SOFR.
- **Lower Band (Reverse Cash & Carry Floor):** Subtract `0.0045` (45 bps) stock-borrow/reinvestment haircut and amortized slippage from SOFR.

**6. Visualize the Arbitrage Corridor**

> Plot the Implied Repo Rate, SOFR, and the Upper and Lower Bands, excluding the final 10 trading days so expiration noise does not flatten the corridor.

:::{dropdown} 🔑 Here Is A Solution (Click to Expand)

```{include} Chapter_Two_Capstone_Solution.md
