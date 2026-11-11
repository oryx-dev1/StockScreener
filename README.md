# Automated Value-Investing Screener

A Python command-line tool that pulls live financial metrics for an entire stock index and flags companies trading below fundamental value relative to their sector peers.

**Supported indices:** Nifty 50 (NSE, India) and S&P 500 (US)
**Metrics:** P/E, P/B, EV/EBITDA, Gross Profit Margin
**Data source:** Yahoo Finance via [`yfinance`](https://github.com/ranaroussi/yfinance)

> **Disclaimer:** This is an educational screening aid, not investment advice. Yahoo Finance data can be delayed, incomplete or wrong. Always verify against company filings before making decisions.

---

## Features

- Fetches live fundamentals for every constituent in parallel (8 threads)
- Compares each stock against its **sector median**, not a one-size-fits-all cutoff
- Combines three valuation multiples with a gross-margin quality filter
- Handles missing data gracefully (e.g. banks with no EV/EBITDA or gross margin)
- Prints a ranked table and exports full results to CSV
- Adjustable strictness through command-line flags

---

## Installation

Requires Python 3.10+.

```bash
pip install yfinance pandas numpy lxml
```

(`lxml` is only needed for scraping the S&P 500 list from Wikipedia.)

---

## Usage

```bash
# Screen the Nifty 50 with default settings
python value_screener.py

# Screen the S&P 500
python value_screener.py --index sp500

# Stricter: 25% discount to peers, must pass all 3 valuation metrics
python value_screener.py --index sp500 --discount 0.25 --min-pass 3

# Custom output file
python value_screener.py --index nifty50 --out nifty_results.csv
```

### Options

| Flag | Default | Description |
|------|---------|-------------|
| `--index` | `nifty50` | Index to screen: `nifty50` or `sp500` |
| `--discount` | `0.20` | Required discount to sector median per metric (0.20 = 20% cheaper) |
| `--min-pass` | `2` | Minimum number of valuation metrics (of 3) a stock must pass |
| `--out` | `screener_results.csv` | Path for the CSV export |

---

## How the screen works

1. **Fetch:** pull P/E, P/B, EV/EBITDA, gross margin, sector and price for each ticker.
2. **Clean:** negative or zero multiples (losses, negative equity) are treated as missing rather than "cheap".
3. **Benchmark:** for each multiple, compute the sector median. If a sector has fewer than 3 stocks with data, fall back to the whole-index median.
4. **Score:** a stock *passes* a metric if it trades at least `--discount` below the benchmark. Count passes across P/E, P/B and EV/EBITDA.
5. **Quality filter:** gross margin must be at or above the sector median. If gross margin isn't reported (common for financials), the filter is skipped.
6. **Flag:** a stock is marked `UNDERVALUED` when:
   - passes ≥ `--min-pass` valuation metrics, **and**
   - has at least 2 valuation metrics available, **and**
   - clears the gross-margin filter.
7. **Rank:** results are sorted by flag, number of passes, then average discount.

### Output columns

| Column | Meaning |
|--------|---------|
| `symbol`, `name`, `sector`, `price` | Basic identifiers |
| `pe`, `pb`, `ev_ebitda` | Valuation multiples |
| `gross_margin` | Gross profit margin, in % |
| `val_passes` | Number of valuation metrics that passed |
| `metrics_available` | How many of the 3 multiples had usable data |
| `avg_discount_%` | Average discount to peer median (positive = cheaper) |
| `UNDERVALUED` | Final flag |

---

## Example output

```
Fetching 50 tickers...

6 of 50 stocks flagged as undervalued:

symbol         name      sector  price   pe   pb  ev_ebitda  gross_margin  val_passes  avg_discount_%
...
Full results saved to screener_results.csv
```

*(Illustrative only. Actual results depend on live market data.)*

---

## Limitations

- **Data quality:** Yahoo Finance is free and unofficial; fields may be missing, stale or occasionally wrong.
- **Trailing metrics only:** P/E uses trailing earnings, which can mislead for cyclical or fast-changing businesses.
- **Relative, not absolute, value:** a stock can be cheap versus its sector and still be expensive in absolute terms, or cheap for good reason (a value trap).
- **Sector comparability:** banks, insurers and asset-light firms don't map neatly onto EV/EBITDA or gross margin.
- **Hardcoded Nifty 50 list:** update the `NIFTY50` list in the script after index rebalances or corporate actions (mergers, demergers, renames).
- **Rate limits:** screening the full S&P 500 makes ~500 requests; Yahoo may throttle. If you see many skipped tickers, lower `max_workers` or retry later.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Many tickers skipped | Likely rate-limiting; reduce `max_workers` in the script or wait and retry |
| `ImportError: lxml` | `pip install lxml` |
| Wikipedia scrape fails | Check your connection, or paste a ticker list into the script manually |
| A Nifty ticker returns no data | Confirm the Yahoo symbol (NSE tickers use the `.NS` suffix) |

---

## Ideas for extension

- Streamlit dashboard with sortable tables and sector charts
- Add ROE, debt-to-equity, free-cash-flow yield or PEG for a richer quality score
- Support other indices (FTSE 100, Nifty 500, DAX)
- Schedule daily runs (cron / GitHub Actions) and email newly flagged stocks
- Cache fetched data to avoid repeated API calls

---

## License

MIT. Use freely, at your own risk.
