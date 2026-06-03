# Finance Dashboard — CLAUDE.md

> Notes for AI coding assistants and human contributors. End users only need the README.

## What this project is

A single-file personal finance dashboard for Indian users. Tracks bank
accounts, a crypto exchange account, and ETH wallets, all in INR with live
rates. No build step, no framework, no server. Open `Finance_Dashboard.html`
directly in a browser.

## File structure

```
finance-dashboard/
├── Finance_Dashboard.html   ← everything lives here
├── README.md                ← user-facing onboarding
├── CLAUDE.md                ← this file
├── LICENSE
└── .gitignore
```

All CSS, HTML, and JS are inline in `Finance_Dashboard.html`. Do not split
into separate files unless explicitly asked.

## Architecture

### Page system

Twelve tabs, each a `<div class="page" id="page-{id}">`. Switching calls
`switchPage(id)` which:

1. Hides all `.page` divs, shows the target.
2. Calls `PAGE_INIT[id]()`. By default this runs **every visit** (most
   init functions re-read live state). Tabs listed in `PAGE_INIT_ONCE`
   are init'd only on first visit, tracked via the `initializedPages`
   Set — currently just the Markets tab, whose iframes are too heavy
   to re-mount on every click.
3. Updates the page heading (`#pageTitle` element) via `PAGE_TITLES`.

### Chart registry

All Chart.js instances are stored in `CR = {}`. Always use
`mkChart(id, cfg)` — it destroys the old instance before creating the new
one, preventing canvas-reuse errors.

### Settings tab

The user's single in-app surface for editing every value previously
hardcoded in `CONFIG` (plus the `IDX` index-return assumptions).
Persists to `localStorage['user_settings_v1']`. On page load, an IIFE
declared immediately after `CONFIG` (and the moved-up `IDX`) deep-merges
the saved settings into both objects BEFORE any aliases (`HOLDINGS`,
`BANK_Bank1`, etc.) derive their values — so all downstream code reads
post-merge values without modification. After Save, the user is asked
to reload; live in-place mutation of every chart/KPI was avoided in
favor of reload-once simplicity.

The form covers: identity, the two primary/secondary banks, monthly
salary, exchange-side crypto holdings, monthly staking. An Advanced
section (collapsed by default) covers the index comparison baseline,
Nifty/S&P assumed returns, the full bank-interest schedule (slab caps +
rates), and the tax presumptive profit rate.

Decimal storage vs. percent display: `bankInterest.tier*.rate` and
`tax.presumptiveProfitRate` are stored as decimals (0.025); the form
shows percentages and converts at the read/write boundary.

`initSettings` is intentionally NOT in `PAGE_INIT_ONCE` — it re-runs on
every tab visit so inputs always reflect current effective values.

### Reset everything

The Settings tab's "Reset everything" button clears every known
`localStorage` key (see the table below — all of them, plus
`user_settings_v1` and `tv_state_v1`) and reloads. Returns the dashboard
to its ship-empty default state.

### Markets tab (TradingView widgets)

Embeds free TradingView widgets (no API key). State lives in
`CONFIG.tradingview` (defaults) and `tvState` (in-memory, persisted to
`localStorage['tv_state_v1']`). `mountTVWidget(containerId, scriptSrc, config)`
is the destroy-and-recreate primitive — same role `mkChart()` plays for
Chart.js. Watch out for TradingView's inconsistent theme property: the
Advanced Chart uses `theme`, the other widgets use `colorTheme`.

`initTradingView()` is registered in `PAGE_INIT_ONCE` so it runs only on the
first visit to the tab — re-mounting four iframes on every click is too slow.
`PAGE_INIT_ONCE` is opt-in; all other tabs continue to re-init on each visit
because they read live state (`LIVE`, `getMonthlyExpenses()`, localStorage).

### Live rates

`LIVE` object holds current prices. `fetchLiveRates()` runs on load and
every 5 minutes.

**USD/INR sources** (tried in order, first success wins):
1. `cdn.jsdelivr.net/npm/@fawazahmed0/currency-api` — permissive CORS,
   works from `file://`.
2. `api.frankfurter.app`.
3. `api.exchangerate-api.com`.

**Crypto sources** (tried in order):
1. Binance (`api.binance.com`).
2. CoinGecko.
3. CoinCap.

After fetch: calls `updateLiveValues()` → `fetchInflation()` →
`fetchWalletBalances()`.

## The CONFIG block

The first thing inside `<script>` is a `CONFIG` object literal — the single
source of truth for personalization.

```js
const CONFIG = {
  ownerName: "Your Name",
  banks: { primary: {...}, secondary: {...} },
  monthlyNetSalary, defaultMonthlyExpenses,
  holdings: { eth, sol, usdCash, usdc, usdt },
  monthlyStakingUSD,
  bankInterest: { tier1, tier2, tier3 },
  comparisonOpeningBalance,
  tax: { defaultRegime, presumptiveProfitRate },
};
```

Immediately after CONFIG, the legacy identifiers (`HOLDINGS`, `BANK_Bank1`,
`BANK_bank2`, `SALARY_INR`, `ACTUAL_STAKING_USD`, `COMPARE_OPENING`) are
re-declared as aliases reading from CONFIG. This means downstream code
(~230 references) keeps working unchanged.

## Tiered bank interest

`calcBank1Interest(balance)` is the canonical interest calculator. The
slab boundaries and rates read from `CONFIG.bankInterest`. The function
name retains "Bank1" for historical reasons but is fully generic — any
tiered savings rate can be expressed by editing CONFIG.

## Dynamic / user-editable data (localStorage)

| Key | Description |
|-----|-------------|
| `monthly_expenses` | Monthly budget (₹). Editable on Projections tab. |
| `tx_overrides` | User-edited transaction categories. |
| `manual_entries` | Manually added transactions. |
| `monthly_memos` | Cash-flow memo per month. |
| `inflight_entries` | Crypto-to-INR offramp entries. |
| `custom_accounts_v1` | Dynamic bank accounts + ETH wallets. |
| `firc_entries` | FIRC remittance log (includes base64 cert). |
| `user_goals` | Financial goals. |
| `tax_regime` | `'new'` \| `'old'`. |
| `cgt_entries` | Crypto capital gains entries. |
| `adv_tax_v1` | Advance tax paid per quarter + base64 receipts. |
| `invoices_v1` | Invoice records. |
| `tv_state_v1` | Markets tab state (ticker tape + watchlist grid + initial main-chart symbol). |
| `user_settings_v1` | Settings-tab overrides for every overrideable CONFIG value plus IDX. Deep-merged into CONFIG/IDX at page load. |

Base64 storage of attachments shares the ~5MB localStorage cap.

## Dynamic accounts (`CUSTOM_ACCS`)

Loaded from localStorage, defaults to empty array. Each account is one
of:

```js
{ id, name, short, type: 'wallet'|'bank', address?, balanceEth?,
  tokensUSD?, tokens:[], balanceINR?, color, note }
```

- Wallet balances fetched via Ethplorer freekey API.
- Token spam filter `isSpamToken(sym, name)` rejects common spam
  patterns and oversized symbols.

## Taxes — 44AD Presumptive

- **Regime:** Default `CONFIG.tax.defaultRegime`. Persisted to
  `tax_regime` localStorage key.
- **Gross receipts:** Auto-pulled from Paid invoices on tab open.
- **Profit rate:** Editable input, default `CONFIG.tax.presumptiveProfitRate`.
- **87A rebate:** New regime — full rebate if slab tax ≤ ₹60,000. Old
  regime — rebate if income ≤ ₹5L and tax ≤ ₹12,500.
- **Crypto CGT:** 30% flat + 4% cess (115BBH). TDS at 1% (194S).
  Losses cannot be set off.

### FY 2026-27 new regime slabs

| Income | Rate |
|---|---|
| 0 – 4L | 0% |
| 4L – 8L | 5% |
| 8L – 12L | 10% |
| 12L – 16L | 15% |
| 16L – 20L | 20% |
| 20L – 24L | 25% |
| > 24L | 30% |

## Projection model

`buildProjection()` runs a 13-month forward simulation:

- **Income:** `SALARY_INR + calcBank1Interest(bank) + stakingINR()`.
- **Expense:** `getMonthlyExpenses()` (reads localStorage).
- **Exchange portfolio:** accumulates staking separately.

Always call `getMonthlyExpenses()` inside loops, never `EXPENSES_INR`
(which is a startup snapshot).

## Common patterns

- `setText(id, val)` — null-safe wrapper used everywhere instead of
  `getElementById().textContent =`.
- `INR(n)` — rounds and locale-formats with the ₹ symbol.
- Adding a new tab: add `<div class="page">`, add nav item, add entry
  to `PAGE_TITLES` + `PAGE_INIT`, write the init function.

## What is hardcoded (intentionally)

- Tax slabs and rules (India FY 2026-27).
- `TRANSACTIONS` sample data — illustrative starter rows.
- `IDX = { nifty: 12, sp: 10 }` — assumed index returns for the
  comparison chart. Editable on Projections.

## Known limitations / future work

- `TRANSACTIONS` is a hardcoded array. A CSV importer would make it
  truly dynamic.
- Some chart datasets in the Cash Flow init still use literal zero
  defaults rather than reading from TRANSACTIONS. A future pass should
  derive these from the array.
- localStorage ~5MB cap can constrain attachment storage.
- Portfolio breakdown and alloc chart use the `HOLDINGS` aliases —
  wallet token breakdowns show up in the KPI total but not itemised
  in the pie chart.
- Tax engine is India-only.
- Display assumes two bank rows; supporting more requires code edits.
