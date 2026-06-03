# Finance Dashboard

A single-file personal finance dashboard for Indian users. Tracks bank
balances, crypto holdings on an exchange, ETH wallets, projections, India
tax (44AD presumptive + new/old regime), FIRC remittances, invoices, and
financial goals. All in INR with live FX and crypto rates. No build step,
no server, no account. Open the HTML file in any modern browser.

## What you get

- **Overview** — Net-worth KPI, allocation, live FX/crypto/inflation.
- **Cash Flow** — Monthly cash-flow over the projection window.
- **Portfolio** — Allocation breakdown across banks, crypto, wallets.
- **Projections** — 13-month forward simulation, FIRE projection.
- **In-Flight** — Track crypto-to-INR offramps in transit.
- **Transactions** — Categorize, override, and add manual entries.
- **Goals** — Track savings targets.
- **FIRC** — Log foreign-remittance certificates (with file attachments).
- **Taxes** — India FY 2026-27 slabs, 44AD presumptive, advance tax, CGT.
- **Invoices** — Issue and track invoices; gross receipts feed into tax.
- **Settings** — Edit your real numbers (bank balances, salary, holdings, bank interest schedule, etc.). Saves to your browser. Includes a Reset-everything button.
- **Markets** — Embedded TradingView widgets (ticker tape, advanced chart, mini-chart grid, economic calendar). Watchlist editable in-app.

## Quickstart

1. **Clone the repo**
   ```
   git clone <your-fork-url> finance-dashboard
   cd finance-dashboard
   ```

2. **Open the file in a browser and enter your numbers via the
   Settings tab** (bank balances, salary, holdings, bank interest
   schedule, index assumptions). Saves automatically to your browser.
   Alternatively, you can pre-fill defaults by editing the `CONFIG`
   block at the top of the `<script>` tag in `Finance_Dashboard.html`.

3. **Open the file in a browser**
   ```
   open Finance_Dashboard.html        # macOS
   xdg-open Finance_Dashboard.html    # Linux
   start Finance_Dashboard.html       # Windows
   ```

That's it. Everything else is editable in the UI — monthly expenses,
invoices, goals, FIRC entries, advance tax, custom accounts/wallets.

## Architecture (short)

- Single HTML file with inline CSS and JS. No build, no framework.
- 12 tabs, each a `<div class="page">`. Switching is handled by `switchPage(id)`.
- All Chart.js instances live in a `CR = {}` registry, recreated via `mkChart`.
- Live rates fetched on load and every 5 minutes from multiple
  CORS-friendly endpoints (jsdelivr currency-api, Frankfurter, Binance,
  CoinGecko, CoinCap) with fallback chains.
- User data persists in browser `localStorage` — invoices, goals, FIRC,
  transactions, advance tax, custom accounts.

See `CLAUDE.md` for a deeper architectural dive.

## Live rate sources

| Data | Primary | Fallbacks |
|---|---|---|
| USD/INR | jsdelivr currency-api | Frankfurter, exchangerate-api |
| ETH, SOL price | Binance | CoinGecko, CoinCap |
| Inflation (India + Global) | World Bank API | (none, falls back silently) |
| ETH wallet balances + tokens | Ethplorer freekey | (none) |

## Data privacy

Everything is in your browser's `localStorage`. **Nothing leaves your
machine** except outbound calls to the public rate/balance APIs listed
above. No server, no telemetry, no analytics.

If you attach files in the FIRC or advance-tax tabs, they're stored as
base64 strings in localStorage. The browser cap is ~5MB total; large
attachments can push you over.

## Customizing further

- **Different bank tiers?** Edit `CONFIG.bankInterest`.
- **Different exchange?** The `holdings` shape (eth/sol/usdCash/usdc/usdt)
  matches what Exchange exposes, but any exchange with these tokens works.
  Update the quantities in `CONFIG.holdings`.
- **More wallets / sub-accounts?** Use the "+ Add Account" button on
  the Overview tab — saved to localStorage, no code edit needed.
- **Different tax year?** Slabs are hardcoded for India FY 2026-27.
  Open the Taxes tab init code and update.

## Known limitations

- The sample `TRANSACTIONS` array is empty by default. To bring in your
  real transactions, add them via the Transactions tab "Add Transaction"
  UI (stored in `localStorage['manual_entries']`), edit the
  `TRANSACTIONS` array directly, or extend the dashboard with a CSV
  importer (PRs welcome).
- localStorage 5MB cap can be hit if you attach many large file
  certificates.
- Tax engine is India-only.
- Display labels assume two bank accounts; adding more requires
  light code edits.

## Contributing

Issues and PRs welcome. Keep the project single-file — no build step.

## License

MIT. See `LICENSE`.
