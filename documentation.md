# StockSight Version 2 Documentation

This document serves as the comprehensive guide for StockSight Version 2, encompassing its Product Requirements Document (PRD), downstream dependencies, database architecture, and a user manual.

---

## 1. Product Requirements Document (PRD)

### 1.1 Overview
StockSight Version 2 is a personal portfolio intelligence web application designed to help users track, analyze, and optimize their investments. It provides deterministic, rule-based scoring for stocks, mutual funds, and ETFs, alongside an integrated AI assistant ("Stocky") for conversational insights into portfolio health, risk, and asset allocation.

### 1.2 Target Audience
Individual retail investors looking for objective, data-driven analysis of their existing holdings and potential investments without the noise of speculative forecasting.

### 1.3 Key Features
- **Deterministic Analytics Engine:** Evaluates assets based on fixed rules for business quality, economic moat, management efficiency, and risk (Beta, Market Cap).
- **Consolidated Dashboard:** A single terminal view displaying the portfolio grid, a secondary watchlist, and aggregate analytics (returns, health score, risk distribution).
- **Moreshwar Levels (Price Targets):** Automatically generated target and stop-loss levels based on fundamental scoring and current holding status, designed using an internal logical model.
- **Stocky AI Assistant:** A local conversational interface that parses natural language queries to explain stock scores, simulate capital allocation, and assess portfolio efficiency traps.
- **Cloud Synchronization:** An offline-capable data persistence layer that seamlessly synchronizes the user's localized portfolio state to a remote Google Sheet.

### 1.4 Non-Functional Requirements
- **Performance:** Lightweight, client-side rendering utilizing Vanilla JavaScript and TailwindCSS for rapid load times.
- **Resilience:** Implements a multi-proxy fallback mechanism to aggressively fetch screen-scraped financial data when direct APIs fail or block requests.
- **Storage:** LocalStorage first design, allowing complete application usability offline, with deferred cloud syncing.

---

## 2. Downstream Dependencies & Third-Party APIs

The application is built entirely on client-side technologies (HTML, CSS, JS) without a traditional backend framework, heavily relying on external data sources and proxies.

### 2.1 UI Libraries
- **TailwindCSS (via CDN):** Used for all component styling, layouts, and responsive design.
- **Google Fonts (Inter):** Typography.

### 2.2 Financial Data APIs & Scraping Targets
The system aggressively attempts to fetch data using a cascading fallback approach across several sources:
- **Screener.in:** Primary source for Indian equities fundamentals (P/E, ROE, ROCE, Sales Growth, OPM, Market Cap). Scraped via DOM parsing of the HTML response.
- **Yahoo Finance (`query1.finance.yahoo.com`):** Secondary source for global equities, ETFs, historical charts (for calculating 1Y/3Y/5Y returns), and Beta.
- **MFAPI.in (`api.mfapi.in`):** Used exclusively for fetching the latest NAV and historical data points for Indian Mutual Funds.
- **Google Finance:** Tertiary fallback for basic price extraction if Screener and Yahoo fail.

### 2.3 CORS Proxies
Because client-side JS cannot bypass CORS to scrape Screener.in or Yahoo Finance directly, the `app.js` network layer loops through these open proxies until a successful response is received:
1. `api.codetabs.com`
2. `api.allorigins.win`
3. `corsproxy.io`
4. `thingproxy.freeboard.io`

---

## 3. Database Schema & Connection

StockSight v2 does not use a traditional SQL/NoSQL database. It utilizes a combination of browser `localStorage` and a Google Apps Script deployment as a remote JSON store.

### 3.1 Local Storage State (`stockSightData`)
The primary source of truth is maintained in the browser.
```json
{
  "portfolio": {
    "INFY": {
      "qty": 50,
      "avg": 1400.50,
      "analyzedPrice": 1500.00, // Frozen price at time of analysis
      "analyzedLevels": { "target": 1550, "sl": 1400, "entry": null },
      "analyzedAction": "HOLD",
      "notified": { "target": false, "sl": false, "entry": false }
    }
  },
  "activeTab": "portfolio"
}
```

### 3.2 Google Sheets App Script Connection
- **Endpoint:** `https://script.google.com/macros/s/AKfycbyScFGyxqSwudrvbCngaPtlKxtHlS4O8Q7bj4FcQEESir3LFSlnHquPxskKsBKC9kS1VQ/exec`
- **Mechanism:** The application sends a POST request with the stringified `portfolio` object whenever the user makes an update, ensuring an eventual-consistency backup. On app initialization, a GET request retrieves the cloud state. If cloud data exists, it prompts the user to overwrite local data.
- **Data Sanitization:** Before syncing, the payload is scrubbed of runtime metadata (e.g., `status`, `sync_ts`) via the `sanitizePortfolio()` function to keep the JSON payload pure to the asset holdings.

---

## 4. User Manual

Welcome to StockSight Cloud Terminal. This manual will guide you through managing your portfolio and analyzing assets.

### Getting Started
1. Open `welcome.html` in your browser and click **Enter Terminal** to proceed to the main dashboard.
2. The dashboard consists of a Sidebar for adding assets and a Main View for tracking them.

### Adding Assets to Watchlist or Portfolio
1. Locate the input field in the left sidebar under "Terminal".
2. Type a stock ticker symbol (e.g., `RELIANCE`, `TCS`, `INFY`) or a Mutual Fund code (e.g., `120503`).
3. Press `Enter` or click **Add**. 
4. The system will fetch the data and display a new card under the Watchlist tab.
5. **Moving to Portfolio:** To graduate a stock from the Watchlist to your active Portfolio, simply enter a quantity (`Qty`) and an average buy price (`Avg`) in the input fields provided on the stock's card.

### Understanding the Stock Card
- **Scores:** You will see a fundamental score (out of 100) combining Business, Moat, Management, and Risk.
- **Action Signal:** A clear verdict (BUY NOW, ADD, HOLD, REDUCE, EXIT, WATCH).
- **Moreshwar Levels:** 
  - If you hold the stock, it shows a **Target** and **Stop-loss**.
  - If you do not hold it, it shows a **Target Entry** or **Avoid Till** level.
- **Locked Analysis:** When a stock is first fetched, its price and target levels are "Locked" to prevent them from wildly fluctuating intraday. 
  - To refresh the analysis with the live spot price, click the specific **✖** button next to the "Locked @ ₹X" badge on the top right of the card.
- **Tabs:** Switch between the `Fundamental` tab and the `Porter's 5` tab inside the equity card for different analysis perspectives.

### Reviewing Portfolio Analytics
1. Click the **Analytics** tab at the top of the main window.
2. View your **Asset Allocation** split (Equity vs Cash vs Mutual Funds) and your top cyclical sector exposures.
3. Review your **Portfolio Health** score out of 100, which grades your holdings' foundational quality.
4. Check the **Efficiency Check** box. The system will alert you to "Capital Traps" (heavy allocations in fundamentally weak assets) or "Under-allocated Winners" (tiny allocations in high-scoring assets).

### Using Stocky AI
Stocky is your built-in analytical assistant.
1. Click the floating **Stocky** icon in the bottom right corner of the screen.
2. Type queries in natural language, such as:
   - *"Explain TCS"*
   - *"Compare HDFC and ICICI"*
   - *"Is my portfolio safe?"*
   - *"Allocate 2 Lakh"* (Stocky will simulate buying your top-conviction watchlist items with that capital).

### Cloud Sync
StockSight automatically syncs your `Qty` and `Avg` data to the cloud in the background. If you see a red dot in the bottom left corner, you are offline. You can manually force a sync by clicking the "Force Cloud Sync" icon next to the "Terminal" header.
