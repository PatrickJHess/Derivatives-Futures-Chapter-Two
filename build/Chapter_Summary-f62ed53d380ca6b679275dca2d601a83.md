# 📝 Chapter Summary
<br>
This chapter bridged the gap between academic theory and actual market microstructure by systematically reverse-engineering the Cost of Carry model. We began by visualizing the mechanical forces connecting futures and spot prices, using Altair to map how uncertainty and trading volume influence the futures basis.

Moving from visual intuition to quantitative implementation, we deployed custom Python pipelines (FredReader and MASSIVEReader) to source historical market data. Using Pandas and NumPy, we constructed a vectorized pricing engine capable of handling real-world market frictions. By systematically aligning business calendars, accounting for discrete dividend cash flows, and applying daily Forward SOFR rates, we successfully extracted the exact Implied Repo Rate embedded in the E-mini S&P 500 contract.

Ultimately, this module provided a hands-on demonstration of why institutional futures prices deviate from continuous textbook formulas, highlighting the balance sheet premiums and estimation risks that drive real trading desks.