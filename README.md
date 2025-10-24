# Data Bootcamp Midterm Project – Moving Average Trading Strategy
### Dennis, Eilon, Ivan, Dominik  
October 23, 2025  

------------------------------------------------------------------------------------

##  Data Sources

1. **Yahoo Finance API (via yfinance)**  
We used to collect daily historical price data (Open, High, Low, Close, Volume) for all Nasdaq-100 tickers. Each dataset spans about two years, providing enough data to calculate long-term indicators like the 200-day moving average. 

2. **Tradier Sandbox API**
Used to simulate trade execution in a safe paper-trading environment. Each buy or sell signal triggered a POST request to Tradier, which returned an execution status (“executed,” “rejected,” or “error”). This allowed us to maintain a realistic trade log without financial risk.

3. **Wikipedia (Nasdaq-100 Constituents Table)** 
The ticker list was scraped directly from Wikipedia using `pandas.read_html()`. This method ensures the system always reflects the current Nasdaq-100 composition without manual updates.

4. **Local Portfolio Tracker (custom class)**  
A custom in-memory PortfolioTracker object handled cash balance, recorded trades, and tracked realized and unrealized PnL. This local component served as the data sink that connected all trading, signal, and portfolio modules.

**Collected Variables:**

Each stock’s dataset included:
-OHLCV data: Open, High, Low, Close, Volume
-Indicators: EMA9, EMA21, MA50, MA200
-Derived features: trend classification and buy/sell/hold signals

------------------------------------------------------------------------------------

## Process Overview   

------------------------------------------------------------------------------------

### Data Retrieval (`fetch_data`)  
*Goal: Collect clean historical price data for testing indicators.*

-Uses `yfinance` to download about 730 days of OHLCV data for each ticker.
-Cleans the dataset by flattening columns, removing missing values, and converting timestamps for consistency.
-Keeps only the relevant variables: Open, High, Low, Close, and Volume.
-Includes error handling to skip incomplete or empty data.
-A 730-day window ensures enough data to calculate long-term indicators like the 200-day moving average.

**Visual:**  
![Data Retrieval Chart](charts/data_fetch_example.png)

------------------------------------------------------------------------------------

### Indicator Calculation (`indicators`)  
*Goal: Compute key moving averages used in trading signals.*

-Calculates four core indicators:
  -EMA9: short-term momentum
  -EMA21: medium-term momentum
  -MA50: intermediate trend
  -MA200: long-term trend stability
-Uses a layered approach: shorter EMAs respond quickly to price shifts, while MAs define the broader trend.
-Handles missing values and ensures results remain consistent across tickers.

**Visual:**  
![Indicator Chart](charts/QQQ_Indicators.png)

------------------------------------------------------------------------------------

### Trend Classification (`classify_trend`)  
*Goal: Identify whether the market is trending up, down, or sideways.*

-Uptrend: Price above MA50/MA200 and near EMA21.
-Downtrend: Price below MA50/MA200 and near EMA21.
-Choppy: Unclear movement within a ±1% buffer to avoid false reversals.
-Each classification is stored in the DataFrame for signal generation.

**Visual:**  
![Trend Classification Example](charts/trend_classification.png)

------------------------------------------------------------------------------------

### Signal Generation (`generate_recent_signal`)  
*Goal: Determine whether to buy, sell, or hold.*

-Buy Signals (Uptrend):
  -Pullback Buy: Price within 2% of EMA9.
  -Breakout Buy: Previous close below EMA9, current close crosses above.
-Sell Signals (Downtrend):
  -Pullback Sell: Price within 2% of EMA9.
  -Breakout Sell: Price crosses below EMA9 from above.
-A 2% tolerance prevents noise-triggered signals.
-Signals are tagged with entry price and stop-loss levels based on EMA21 or 93%/107% thresholds.

**Visual:**  
![Signal Generation Example](charts/signal_generation.png)

------------------------------------------------------------------------------------

### Trailing Stop & Take-Profit (`update_trailing_stop_and_take_profit`)  
*Goal: Add basic risk management to the strategy.*

-If a trade moves 5% in profit, the stop-loss is raised to breakeven.
-Protects gains while minimizing downside risk on successful trades.
-Keeps the trade lifecycle realistic by mirroring common risk management methods.

**Visual:**  
![Stop Loss Example](charts/trailing_stop_example.png)

------------------------------------------------------------------------------------

### Visualization (`plot_strategy`)  
*Goal: Display price, indicators, and signals clearly.*

-Plots Close, EMA9/21, and MA50/200 together to show crossovers and signals.
-Marks all Buy/Sell/Exit points for clarity.
-Gives a visual sense of timing accuracy and overall trend-following behavior.

**Visual:**  
![Strategy Plot](charts/strategy_plot.png)

------------------------------------------------------------------------------------

### Trade Execution System (`execute`)  
*Goal: Simulate realistic trade execution using the generated buy/sell signals.*

-The `execute()` function converts trade signals into simulated orders and updates the portfolio records.
-Each order includes the symbol, action (buy or sell), quantity, and entry price, which are passed to the Tradier Sandbox API for paper execution.
-The API returns an execution status such as executed, rejected, or error, which is logged and displayed in the console for confirmation.
-All trade attempts are stored in `trade_log.json` with key details like timestamps, price, and status.
-The system checks basic conditions before execution:
  -Sufficient cash available for buys
  -Existing position required for sells
-Invalid trades are automatically skipped and marked as rejected to avoid interruptions.
-Console output clearly reports results (e.g., “Executed BUY 3 shares of AAPL”) for transparency.
-The combination of automated logging and live feedback makes it easy to monitor trade flow and debug performance.

**Visual:**  
![Execution Log Example](charts/trade_execution_log.png)

------------------------------------------------------------------------------------

### Full Strategy Integration (`run_strategy`)  
*Goal: Combine all major functions into a single workflow that handles the complete strategy for one ticker.*

-The `run_strategy()` function acts as the engine that ties every component together for a single stock.
-Workflow steps:
  -Fetch: Collect and clean historical data with `fetch_data()`.
  -Indicators: Calculate moving averages through `indicators()`.
  -Trend: Identify the market direction with `classify_trend()`.
  -Signal: Generate buy or sell opportunities using `generate_recent_signal()`.
  -Trailing Stop: Update stop-loss and take-profit levels with `update_trailing_stop_and_take_profit()`.
  -Execute: Pass signals to the `execute()` function for simulated trade placement.
  -Plot: Visualize strategy results through `plot_strategy()`.
-The function outputs a cleaned DataFrame containing OHLCV data, calculated indicators, and generated signals.
-This DataFrame, along with the visual chart, provides a full view of the strategy’s performance on that ticker.
-The single-ticker workflow serves as the foundation for scaling up to multiple stocks in later stages.

**Visual:**  
![Run Strategy Output](charts/run_strategy_output.png)

------------------------------------------------------------------------------------

## Expanding to Multiple Stocks

------------------------------------------------------------------------------------

### Nasdaq-100 Scraper (`load_QQQ_tickers`)   
This section transitions the system from single-ticker testing to full market-level automation. Rather than analyzing one stock at a time, the program now scans the entire Nasdaq-100, executes strategy logic for each company, and updates the portfolio accordingly.

Ticker Scraping:
-The process starts by calling `load_QQQ_tickers()`, which uses `pandas.read_html` to scrape the most recent list of Nasdaq-100 companies directly from Wikipedia.
-This approach keeps the system up to date automatically, eliminating the need to manually maintain ticker lists.
-The tickers are cleaned by replacing periods with hyphens and removing extra spaces to make sure each one works properly when downloading data through `yfinance`.

Market Scanning & Execution:
-The `scan_and_execute_QQQ()` function loops through every ticker returned by `load_QQQ_tickers()` and runs the main strategy (`run_strategy`) on each.
-For every stock, the system checks for the latest signal, price, and position size.
-If a valid “buy” or “sell” signal is detected, it executes a simulated order through the `execute()` function and records it.

Portfolio Integration
-All trades feed into the `PortfolioTracker` class, which keeps track of cash balance, open positions, and overall portfolio value.
-This setup allows the portfolio to update in real time as trades occur across different tickers.

Error Handling:
-Try/except blocks prevent a single bad ticker or API failure from stopping the full scan.
-Empty or invalid data frames are skipped automatically to keep the run smooth and uninterrupted.


**Visual:**  
![Ticker Scraper Example](charts/load_QQQ_tickers.png)

------------------------------------------------------------------------------------

### Portfolio System (`PortfolioTracker` Class)  
This part of the project introduces the `PortfolioTracker` class, which acts as the backbone of the trading system. It’s responsible for tracking cash, recording trades, storing open positions, and keeping the entire simulation persistent across sessions. The goal is to create a realistic structure that mirrors how an actual brokerage account or portfolio management system operates

Initialization (__init__):
-Starts with a default cash balance of $100,000.
-Creates two main data structures:
-positions: a dictionary that holds all active positions (symbol, qty, avg_price, and entry_time).
-history: a list that records every executed or rejected trade for transparency.

Buy Logic (buy method):
-Each buy uses a dynamic allocation rule: it calculates the maximum allocation per trade based on available cash, ensuring position sizes scale realistically as capital fluctuates.
-If the portfolio doesn’t have enough cash, the order is rejected and logged.
-If accepted, it deducts the cost, updates the average entry price for existing positions, or creates a new one.
-Every transaction (executed or rejected) is logged with a timestamp for later review.

Sell Logic (sell method):
-Checks if the symbol exists in the portfolio before selling.
-Calculates PnL (profit or loss) based on the difference between the sell price and the average purchase price.
-Updates cash and reduces or removes the position if it’s fully sold.
-Each sale is recorded in history with timestamp and realized PnL.

Position Tracking (get_positions):
-Returns a clean DataFrame showing all active holdings.
-Displays key information like quantity, average cost, and when each trade was opened.
-If no open positions exist, it returns an empty table with headers for consistency.

Trade History (get_history):
-Outputs a DataFrame containing all trade actions: buys, sells, and rejected orders — including timestamps, quantities, and PnL.
-Serves as a built-in ledger for reviewing performance and debugging.

Portfolio Value (total_value):
-Calculates the combined value of all positions plus remaining cash.
-Gives a quick way to assess total equity at any given time.

Saving and Loading:
-The `save()` and `load()` methods store and restore portfolio data using a JSON file.
-This makes it possible to pause and resume simulations without losing progress.
-When loading, the system automatically detects whether a previous portfolio exists or starts fresh if none is found.


**Visual:**  
![Portfolio Tracker Output](charts/portfolio_tracker_output.png)

------------------------------------------------------------------------------------

### Portfolio Dashboard (`portfolio_dashboard`)  
The dashboard serves as the visual summary of the entire system, combining key portfolio metrics and performance charts into one output. It’s designed to show cash balance, portfolio value, trading activity, and realized/unrealized profits all in a single, concise view. This section turns raw trade data into something that actually looks and feels like a professional portfolio tracker.

Core Metrics Calculation:
-The function starts by pulling data from the portfolio’s trade history and open positions.
-Calculates total trades, number of buys and sells, average trade size, and total realized profit/loss.
-Loops through all open positions to fetch live prices using yfinance and compute unrealized PnL.

Summary Printout:
-Displays a compact summary block showing key statistics:
  -Cash balance
  -Total portfolio value
  -Number of open positions
  -Total trades, buys, and sells
  -Realized and unrealized PnL
-The printout uses simple formatting and emojis for quick readability, giving it a clean, terminal-based “dashboard” appearance.

PnL Visualization:
-Plots a cumulative PnL chart using `matplotlib`, showing the growth of realized profits over time.
-Unrealized PnL is displayed as a dotted line to show how current holdings are performing relative to closed trades.
-Grid lines, titles, and labels make the chart easy to interpret even at a glance.


**Visual:**  
![Dashboard](charts/portfolio_dashboard.png)

------------------------------------------------------------------------------------

### Full Market Scan (`run_QQQ_full`)  
This section brings every component together into one complete system. The full market scan runs the trading strategy across all Nasdaq-100 stocks, logs executions through the `PortfolioTracker`, and then displays the results in the dashboard. In short, it’s the stage where the project transitions from isolated modules into a fully automated trading simulation capable of scanning, executing, tracking, and visualizing portfolio performance end-to-end.

Automated Market Loop:
-The system begins by calling `scan_and_execute_QQQ()`, which iterates through all Nasdaq-100 tickers.
-For each stock, it runs `run_strategy()` to generate signals and determines whether to buy, sell, or hold.
-Valid trades are executed and automatically logged into the portfolio through the `PortfolioTracker`.

Centralized Portfolio Management:
-Every transaction: including cash adjustments, position updates, and realized PnL, is recorded in real time.
-The portfolio’s structure continuously evolves as trades occur across multiple symbols, simulating live market behavior.
-By consolidating all executions, the system maintains a single source of truth for portfolio state and performance.


Performance Visualization:
-Once the scan completes, the `portfolio_dashboard()` function is called to summarize the final results.
-Displays cash balance, total value, trade statistics, and cumulative PnL (realized and unrealized).
-The chart provides a quick visual on how the strategy performed across the full market run.

**Visual:**  
![Full Market Scan Output](charts/run_QQQ_full_output.png)

------------------------------------------------------------------------------------

## Results and Observations         `
The full market simulation was executed using the run_QQQ_full function over roughly 700 trading days, starting with an initial balance of $100,000. The system ran multiple times across all Nasdaq-100 tickers, automatically generating signals, executing trades, and updating portfolio performance through the integrated dashboard.

Trade Frequency:
-Each run generated an average of 75–85 trades, showing that the EMA crossover logic reacts actively to day-to-day price movement.
-This confirmed that the signal engine and execution pipeline were functioning as intended across a large ticker universe.


Signal Distribution:
-All trades during testing were buy signals, as the timeframe wasn’t long enough to trigger sell conditions.
-This highlighted the need for longer simulation periods or adjusted EMA parameters to allow for both entry and exit cycles.


Portfolio Behavior:
-During typical runs, the system purchased around $16,000 worth of stocks in total.
-Each position represented about 2–3% of total portfolio weight, following the built-in buy limit rules that capped allocations to $500 per share or 1/200th of portfolio value.
-Stocks exceeding these limits were automatically rejected, ensuring the allocation logic stayed consistent with our intended risk controls.


API Limitation & Sandbox Constraint:
-When testing with the Tradier API, we found that it ignored our share size and allocation parameters, defaulting to buying roughly $10,000 worth of the top 10 tickers, then rejecting all subsequent orders due to -insufficient capital.
-After researching this behavior, we confirmed that the sandbox environment imposes a $10,000 hard cap on simulated accounts, making it impossible to fully test portfolio-wide allocation logic in a live API context.
-Despite this limitation, the core strategy and portfolio management functions performed correctly when simulated locally.


Performance Metrics:
-Unrealized PnL fluctuated between roughly –$100 and +$100 across runs, showing moderate variance and confirming that position value tracked correctly with market movements.
-Realized PnL remained $0.00 due to the absence of sell triggers, keeping the Total Portfolio Value stable near the $100,000 starting point.

Key Takeaways:
-The system successfully executed full-cycle simulations: scraping live tickers, generating signals, enforcing allocation limits, managing portfolio states, and visualizing results.
-While API restrictions prevented live execution testing beyond the sandbox limits, all internal components behaved as expected, validating the system’s structure and scalability.
-Future iterations could incorporate alternate strategies, longer testing periods, and alternate APIs to better mirror real trading environments.

**Visual:**  
![Results Example](charts/results_observations.png)

------------------------------------------------------------------------------------

## Conclusions and Takeaways
Summary of Demonstration:
The project successfully developed a systematic trading framework capable of automating the entire process from data collection and technical signal generation to portfolio tracking and performance visualization. The system demonstrated consistent functionality across each module, including ticker scraping, execution logic, dynamic position sizing, and real-time portfolio monitoring.

What Worked Well and What Could Be Improved:
-Worked Well:
  -The end-to-end pipeline ran smoothly and produced diversified portfolio allocations across the Nasdaq-100.
  -Each subsystem such as the strategy logic, portfolio tracker, and dashboard visualization integrated effectively, confirming system stability and scalability.

-Could Be Improved:
  -The backtest period was too short to capture sell signals or evaluate long-term strategy performance.
  -Future tests should include multiple market environments such as bull, bear, and sideways cycles to better assess profitability, drawdowns, and overall risk-adjusted returns.
  -The current EMA-based trend detection may be too simple for stable or range-bound markets, which could limit signal accuracy.

Future Steps for Refinement:
-Add More Indicators:
  -Include additional indicators like the Relative Strength Index (RSI), volume filters, or MACD to validate signals and improve accuracy.
-Refine Position Sizing:
  -Shift toward volatility-based sizing using ATR and implement adaptive trailing stops instead of fixed percentage thresholds to improve risk management.

-Expand Capabilities:
  -Add short selling functionality to generate returns in downtrending markets.
  -Experiment with intraday data to generate faster and more frequent trading signals.
  -Transition from the Tradier sandbox to a live or paper-trading API to remove capital restrictions and enable complete strategy testing.

Key Takeaway:
The project achieved its main objective of building a functional automated trading and portfolio management system. The next phase should focus on improving signal precision, enhancing execution logic, and extending backtesting coverage to better evaluate long-term performance.

------------------------------------------------------------------------------------
