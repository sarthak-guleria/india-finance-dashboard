# Ship Empty + Settings Panel — Design

**Date:** 2026-06-03
**Status:** Approved, pending implementation plan
**Owner:** sarthakguleria002@gmail.com

## Problem

The dashboard ships with specific demo numbers hardcoded in `CONFIG` (₹500,000 primary-bank balance, ₹80,000 monthly salary, 0.5 ETH, 2 SOL, etc.) and a sample `TRANSACTIONS` array. Users who want to use the dashboard for their own finances must either edit the HTML directly (the current "edit CONFIG and refresh" flow) or live with someone else's numbers showing on every screen. Some values (monthly expenses, custom accounts, goals) are already user-editable via in-app UI; many are not.

Goal: make the dashboard usable by a non-coder with zero numbers in it by default, and provide a single in-app surface for setting their real values without touching code.

## Goals

- Ship the dashboard in a blank state — no specific demo balances, salary, holdings, or sample transactions.
- Add a single in-app **Settings** tab that lets the user enter and edit all values currently hardcoded in `CONFIG` (plus the inline `IDX` index-return assumptions).
- Add a **Reset everything** button that clears all `localStorage`-backed user data and reloads the page.
- Preserve all existing CONFIG-reading code paths unchanged.

## Non-Goals

- A first-run onboarding wizard (user picked "ship empty + edit-in-place").
- Inline edit affordances on each tab (user picked the unified Settings panel).
- Migration of `CONFIG.banks.primary`/`secondary` into the existing `custom_accounts_v1` system. The two paths coexist as today — Settings edits the two hardcoded banks; "+ Add Account" on Overview adds more.
- A CSV importer for transactions. Out of scope; CLAUDE.md already flags it as future work.
- Editable tax slabs or 87A thresholds. These are policy values intentionally hardcoded per CLAUDE.md.
- Live update of every chart/KPI on save. We force a page reload instead.

## Architecture

### CONFIG cleanup (ship-empty defaults)

Zero out user-specific demo values in the `CONFIG` literal. Keep policy/UI defaults intact.

**Zeroed:**

```js
banks.primary    = { name: "Primary Bank",   balance: 0 }
banks.secondary  = { name: "Secondary Bank", balance: 0 }
monthlyNetSalary           = 0
holdings.{eth,sol,usdCash,usdc,usdt} = 0
monthlyStakingUSD          = 0
comparisonOpeningBalance   = 0
```

**Also zeroed:**

```js
defaultMonthlyExpenses: 30000 → 0
```

This is already user-editable on the Projections tab via the `monthly_expenses` localStorage key. Settings does NOT duplicate that input — there's one edit surface for monthly expenses, on Projections, where it's always been.

**Kept (policy / sensible defaults):**

```js
ownerName: "Your Name"                          // placeholder string
bankInterest.{tier1,tier2,tier3}                // IDFC-style policy defaults; user can override
tax.{defaultRegime, presumptiveProfitRate}      // policy defaults
tradingview.*                                   // UI defaults for the Markets tab
IDX = { nifty: 12, sp: 10 }                     // assumed index returns
```

**Sample TRANSACTIONS:** the hardcoded `TRANSACTIONS` array is set to `[]`. The dashboard's Cash Flow and Transactions tabs derive from `TRANSACTIONS` joined with `manual_entries` (localStorage). With `TRANSACTIONS` empty, the user's manual entries are their entire transaction history. Any chart that currently reads `TRANSACTIONS` must handle the empty case without crashing (most do already because they `forEach`/`reduce` over the array; verify during implementation).

### Settings tab

A new 12th tab in the sidebar nav. Position: **after Invoices**, **before Markets** (Markets stays last so visually-heavy embeds remain at the bottom of the list). Internal id: `settings`. Page title: `Settings`.

Layout — one form, grouped into sections:

```
┌──────────────────────────────────────────────┐
│  Settings                                    │
│                                              │
│  Identity                                    │
│    Owner name [____________________]         │
│                                              │
│  Banks                                       │
│    Primary    [Name______] [Balance ₹___]   │
│    Secondary  [Name______] [Balance ₹___]   │
│    Note: Add more accounts via "+ Add        │
│    Account" on Overview.                     │
│                                              │
│  Income                                      │
│    Monthly net salary (₹) [______]           │
│    (Monthly expenses are edited on the       │
│     Projections tab.)                        │
│                                              │
│  Crypto Holdings (exchange-side defaults)    │
│    ETH [_____]   SOL [_____]                 │
│    USDC [_____]  USDT [_____]                │
│    USD cash [_____]                          │
│    Monthly staking (USD) [_____]             │
│                                              │
│  ▶ Advanced (collapsed by default)           │
│    Comparison opening balance (₹) [______]   │
│    Nifty assumed return (%) [______]         │
│    S&P assumed return (%) [______]           │
│                                              │
│    Bank interest schedule                    │
│      Tier 1:  up to ₹ [______]  at [___] %   │
│      Tier 2:  up to ₹ [______]  at [___] %   │
│      Tier 3:  above (no cap)    at [___] %   │
│                                              │
│    Tax presumptive profit rate (%) [______]  │
│                                              │
│  [ Save changes ]   [ Cancel ]               │
│                                              │
│  ─────────────────────────────────────────  │
│  Danger zone                                 │
│  [ Reset everything ] (clears all data)      │
└──────────────────────────────────────────────┘
```

Inputs are pre-populated from current effective values (merged CONFIG + saved overrides). All number inputs use `type="number"` with `step` appropriate to the field (1 for currency, 0.0001 for crypto quantities, 0.1 for percentages, 100 for slab boundaries). Negative numbers blocked via `min="0"`.

### State & persistence

One new `localStorage` key: `user_settings_v1`. Shape:

```js
{
  ownerName: string,
  banks: { primary: {name, balance}, secondary: {name, balance} },
  monthlyNetSalary: number,
  holdings: { eth, sol, usdCash, usdc, usdt },
  monthlyStakingUSD: number,
  comparisonOpeningBalance: number,
  bankInterest: {
    tier1: { upTo: number, rate: number },   // rate stored as decimal (0.025 = 2.5%)
    tier2: { upTo: number, rate: number },
    tier3: { rate: number }
  },
  tax: { presumptiveProfitRate: number },
  idx: { nifty: number, sp: number }
}
```

`tax.defaultRegime` is intentionally NOT in the settings shape — the user toggles regime per-session via the existing `tax_regime` localStorage key on the Taxes tab.

### Loading and applying settings

Immediately after the `CONFIG` literal is declared (line ~1228) and before the legacy aliases are derived (line ~1233), inject:

```js
(function applyUserSettings() {
  let stored;
  try { stored = JSON.parse(localStorage.getItem('user_settings_v1') || 'null'); }
  catch (e) { stored = null; }
  if (!stored) return;
  function deepMerge(target, source) {
    for (const k in source) {
      if (source[k] && typeof source[k] === 'object' && !Array.isArray(source[k])) {
        if (!target[k]) target[k] = {};
        deepMerge(target[k], source[k]);
      } else {
        target[k] = source[k];
      }
    }
  }
  deepMerge(CONFIG, stored);
  if (stored.idx) Object.assign(IDX, stored.idx);  // IDX is declared later; assign after IDX init
})();
```

The `IDX` assignment is split out because `IDX` is declared further down the script (line ~1248) — the IIFE runs at parse time but `IDX` isn't yet defined. Solution: the IIFE schedules an `IDX` merge for later, or simpler — move the `IDX` declaration above the IIFE, or attach IDX values into a `pendingIdx` variable that gets consumed when IDX is initialized. The plan task picks the simplest variant during implementation; the spec requirement is just "IDX gets the user's saved nifty/sp values."

The merge mutates `CONFIG` and `IDX` in place before any downstream code reads them. All existing aliases (`BANK_IDFC`, `SALARY_INR`, `HOLDINGS`, `ACTUAL_STAKING_USD`, `COMPARE_OPENING`) and all 230+ references to `CONFIG.*` keep working unchanged.

### Save behavior

Click **Save changes**:

1. Read every form value into a settings object matching the shape above. Rates are converted from percent display (e.g., `2.5`) to decimals (`0.025`).
2. `localStorage.setItem('user_settings_v1', JSON.stringify(settings))`.
3. Replace the action area with an inline confirmation banner: "Saved. Reload to apply changes." with a **Reload now** button (`onclick="location.reload()"`).
4. The **Cancel** button discards form edits without saving: re-populates inputs from the current effective values.

Why reload instead of live-update: dozens of charts, KPIs, projection rows, and tax calculations read from `CONFIG.*` at function call time. Forcing a reload is the smallest correct way to apply the new values across the whole dashboard. Personal-use, infrequent edit — acceptable UX.

### Reset behavior

Click **Reset everything** → native `confirm()` dialog:

> "This clears all your data — settings, accounts, transactions, goals, FIRC entries, tax inputs, invoices, and Markets tab watchlist. The page will reload. Continue?"

On confirm, run:

```js
const KEYS = [
  'user_settings_v1',
  'monthly_expenses',
  'tx_overrides',
  'manual_entries',
  'monthly_memos',
  'inflight_entries',
  'custom_accounts_v1',
  'firc_entries',
  'user_goals',
  'tax_regime',
  'cgt_entries',
  'adv_tax_v1',
  'invoices_v1',
  'tv_state_v1'
];
KEYS.forEach(k => { try { localStorage.removeItem(k); } catch(e) {} });
location.reload();
```

The list comes straight from CLAUDE.md's localStorage table plus `tv_state_v1` (Markets tab). Page reload returns the dashboard to the ship-empty defaults.

## Data flow

```
CONFIG (zeroed defaults) ──┐
                            ├──► deepMerge ──► effective CONFIG ──► all downstream code
localStorage                ──┘                       ▲
(user_settings_v1)                                    │
                                                      │
Settings form ──► serialize ──► localStorage ──► reload triggers the merge again
                                      ▲
            Reset button ──► clear all localStorage keys ──► reload
```

No coupling to `LIVE`, `tvState`, `CUSTOM_ACCS`, or other runtime state — the merge happens once at load and everything else proceeds as today.

## Error handling

- **`localStorage` read fails / corrupted JSON:** the IIFE falls through and the dashboard runs on pristine CONFIG defaults. No error UI; subsequent saves overwrite the bad value.
- **`localStorage` write fails (quota / private browsing):** the save handler shows "Couldn't save — your browser blocked storage." in the same banner slot. No reload triggered. Acceptable for a non-critical feature.
- **Empty / negative numbers:** the form's `type="number"` and `min="0"` block negative input. Empty fields submit as `0` (parsed via `parseFloat(val) || 0`). No additional validation — the dashboard's existing math handles zeros without crashing.
- **TRANSACTIONS empty:** all existing reads of `TRANSACTIONS` must handle `[]` gracefully. Implementation task verifies each call site (most use `.forEach`/`.reduce`/`.filter`, which all no-op on empty arrays).

## Testing

Manual verification in a browser (no test harness exists):

1. **Fresh install:** clear `localStorage`, reload. Dashboard shows ₹0 across all KPIs, owner reads "Your Name", no sample transactions in Cash Flow / Transactions tabs. No console errors.
2. **Settings round-trip:** open Settings, enter real values (e.g., primary balance ₹500,000, salary ₹100,000, ETH 1.5), click Save. See confirmation banner. Click Reload now. Verify Overview KPIs now reflect the entered values.
3. **Cancel discards:** open Settings, change a field, click Cancel. Field re-populates with the previously-saved value.
4. **Advanced collapsible:** the Advanced section starts collapsed. Click the chevron — it expands. Reload — it starts collapsed again (no persistence of UI state).
5. **Bank interest schedule:** change Tier 2 cap from ₹2.5Cr to ₹1Cr at 7%, save, reload. The interest calculation on Projections reflects the new schedule.
6. **Reset everything:** add a manual transaction, save a custom account, save Settings values. Click Reset. Confirm dialog appears with the explicit warning. Confirm. Page reloads — all data gone, dashboard back to ship-empty.
7. **Markets tab unaffected:** existing `tv_state_v1` survives Settings save (only writes `user_settings_v1`). Reset DOES wipe `tv_state_v1`.
8. **Regression sweep:** click through all 12 existing tabs (Overview, Cash Flow, Portfolio, Projections, Inflight, Transactions, Goals, FIRC, Taxes, Invoices, Markets). All render without console errors on a freshly-reset dashboard.

## Risks & tradeoffs

| Risk | Mitigation |
|------|------------|
| TRANSACTIONS-reading code crashes on `[]` | Implementation task explicitly walks each call site; empty-array operations are no-ops for `forEach`/`reduce`/`filter` — likely no fixes needed but must verify. |
| Deep-merge drops fields the user didn't override | Merge only writes keys present in `source`. CONFIG keys absent from `user_settings_v1` retain their defaults. Test with a partial settings object. |
| `IDX` is declared after the IIFE — reference-error risk | Plan picks the simplest of three options: (a) move IDX declaration above the IIFE, (b) defer the IDX merge with a `pendingIdx` variable consumed at IDX init, or (c) make IDX an object literal next to CONFIG. Implementation decides. |
| User clicks Reset by accident | Native `confirm()` with explicit list of data categories. No silent destruction. |
| Save form has 15+ inputs — feels heavy | Sectioned layout + collapsed Advanced keeps initial view to 6 visible groups. Acceptable for a one-time-per-major-life-change form. |
| 12-tab sidebar feels crowded | Acceptable — alternative is a gear icon in the sidebar footer next to the eye icon, but a tab is more discoverable. If feedback says crowded later, can move to gear icon without breaking anything. |
| `defaultMonthlyExpenses` zeroed makes Projections show a fresh user a 100%-surplus picture until they set their expenses | Acceptable — consistent with ship-empty; the Projections tab's inline editor is the existing single edit surface for this value, so the user fixes it immediately on first visit. |

## Documentation updates

- **CLAUDE.md:**
  - Update the localStorage table to add `user_settings_v1`.
  - Update the tab count from "Eleven tabs" to "Twelve tabs".
  - Add a "Settings tab" section under Architecture describing the deep-merge loader and the reset semantics.
  - Mention in the CONFIG section that values are now overrideable via `user_settings_v1` and updated through the Settings tab.
  - Note that demo `TRANSACTIONS` is now `[]` by default (update the "Known limitations" line about hardcoded TRANSACTIONS).
- **README.md:**
  - Add Settings to the tab list (between Invoices and Markets).
  - Update the Quickstart: step 2 ("Edit the CONFIG block...") is now optional/legacy — the primary user flow is "open the file → click Settings → enter your numbers → reload."
  - Update the "11 tabs" line to "12 tabs".

## Open questions

None. Design is ready for implementation planning.
