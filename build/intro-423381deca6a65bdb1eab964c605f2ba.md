## **Financial Python**

## **📚 Volume: Futures Markets**

### 📈 Visualizing the Concepts and Applying the Model: Arbitrage and the Cost of Carry ⚙️


#### **⚖️ Conceptualizing Pricing: The Transmission of Uncertainty and the Power of Arbitrage**
It is often said that seeing is believing, but visualization only creates understanding when it points to underlying principles. Staring at the night sky yields little astronomical knowledge without a map of the constellations. Once that organizing principle is provided, you have the foundation for true insight.

The first notebook in our exploration adopts this philosophy. We begin with a 30,000-foot macro view of how market uncertainty and arbitrage express themselves through trading volume and prices. The core objective is to demonstrate how these dynamics shift based on the specific attributes of a futures contract. To that end, we examine three contracts covering distinctly different assets:

*  **📈 The S&P 500 E-mini (CME):** A purely financial construct that utilizes cash settlement.

*  **🛢️ West Texas Intermediate (WTI) Crude Oil (CME):** A commodity with a continuous, non-seasonal production cycle, settled via physical delivery.

*  **🌽 Corn (CBOT):** A commodity tied to a strict seasonal production cycle, also settled via physical delivery.

Despite the fundamental differences of the underlying assets, common threads bind them. Each is driven by the evolution of uncertainty, and each is subjected—to varying degrees—to the strict discipline of arbitrage.

Our first goal is to illustrate these concepts by visualizing trading volume and the difference between spot and futures prices (the basis). These visualizations naturally point to the core components of the cost of carry model. The model's inputs can feel abstract—but they leave massive footprints in the data, making it easy to map the visuals directly to the fundamental pricing equation:

$$F= S + \text{Storage} - \text{Convenience or Dividend Yield} + \text{Interest}$$


* **F and S:** The two lines you'll see stretched and compressed on your dashboard.

* **Storage:** The massive steel tanks in Cushing and the squeezed logistical pipelines on the Mississippi River.

* **Convenience or Dividend Yield:** The massive premium paid during the 2026 Iran panic, the value of having immediate grain at the export terminal, or the actual cash dividends paid by the S&P 500.

* **Interest:** The capital efficiency of keeping your cash in the bank while still getting full price exposure.

#### **⚙️ Building the Machinery: Pinning Down the Implied Repo Rate**
Looking at historical graphs and tracing large visual footprints is the easy part. The underlying concepts make intuitive sense; applying them requires that we get dirt under our fingernails. That is exactly what the second phase of this chapter does: it actually builds the machinery.

In finance textbooks, this relationship is often neatly summarized by a continuous compounding formula:

$$F=S \times e^{(r-dy)\times t}$$

Where $r$ is the continuous rate of interest, $dy$ is the continuous dividend yield, and $t$ is the time until the contract expires. The equation gives you the general idea, but ignores messy but necessary details. If your knowledge stops here, you won't be able to sort out how to estimate the cost of carry model and the 'fair' basis.
You'll know the destination, but not how to get there.

The second notebook focuses on the exact mechanics of getting from point A to point B. We will use the September 2026 S&P 500 E-mini contract to illustrate the intricacies of real-world pricing. Because it is an equity index, we can drop storage costs and convenience yields from the cost of carry model. This strips the equation to its core, allowing us to focus on the precise, moving parts of a purely financial transaction:

* **Dividend Yield:** The future value of dividends paid out before expiration.
* **Interest:** The financing cost—tied to benchmark rates like SOFR—that compensates the seller for delayed payment.
  
Constructing this model from the ground up reveals a persistent, practical gap between theoretical benchmark rates and the actual implied repo rate traded by the market. A gap reflecting the real-world frictions and costs of executing arbitrage. If you have ever wondered why those fancy Wall Street banks exist, you now know one reason.
