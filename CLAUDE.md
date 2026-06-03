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

Ten tabs, each a `<div class="page" id="page-{id}">`. Switching calls
`switchPage(id)` which:

1. Hides all `.page` divs, shows the target.
2. Calls `PAGE_INIT[id]()` **once** on first visit (tracked via the
   `initializedPages` Set).
3. Updates `document.title` and `.page-title` via `PAGE_TITLES`.

### Chart registry

All Chart.js instances are stored in `CR = {}`. Always use
`mkChart(id, cfg)` — it destroys the old instance before creating the new
one, preventing canvas-reuse errors.

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

Immediately after CONFIG, the legacy identifiers (`HOLDINGS`, `BANK_IDFC`,
`BANK_KOTAK`, `SALARY_INR`, `ACTUAL_STAKING_USD`, `COMPARE_OPENING`) are
re-declared as aliases reading from CONFIG. This means downstream code
(~230 references) keeps working unchanged.

## Tiered bank interest

`calcIDFCInterest(balance)` is the canonical interest calculator. The
slab boundaries and rates read from `CONFIG.bankInterest`. The function
name retains "IDFC" for historical reasons but is fully generic — any
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

- **Income:** `SALARY_INR + calcIDFCInterest(bank) + stakingINR()`.
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
