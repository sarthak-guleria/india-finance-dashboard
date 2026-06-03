# Ship Empty + Settings Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the dashboard ship blank (zeroed demo numbers, empty sample transactions) and add a single in-app **Settings** tab that lets a user enter and edit every value currently hardcoded in `CONFIG` (plus the inline `IDX` index-return assumptions), with a Reset-everything button that wipes all `localStorage`.

**Architecture:** All work happens in `Finance_Dashboard.html`. User-entered values persist to a new `localStorage` key `user_settings_v1`. On page load, an IIFE deep-merges the stored settings into `CONFIG` (and `IDX`) *before* any downstream code derives aliases or renders. All existing references to `CONFIG.*` keep working unchanged.

**Tech Stack:** Vanilla HTML/CSS/JS inline in `Finance_Dashboard.html`. Browser `localStorage` for persistence.

**Project context for the executing engineer:**
- Spec: `docs/superpowers/specs/2026-06-03-ship-empty-and-settings-design.md`
- Project rules: `CLAUDE.md` — single-file architecture, do NOT split HTML/CSS/JS into separate files.
- No test harness exists. Verification is **manual in a browser**: open `Finance_Dashboard.html` directly via `file://`, perform the action, observe the result. State "Expected" explicitly so reviewers know what to look for.
- Dashboard uses a **light theme** (cream `#F7F4EE` background, white cards). Reuse existing CSS variables (`--card`, `--border`, `--text`, `--muted`, `--bg`, `--s2`, `--blue`, `--red`, `--faint`, `--shadow-sm`).
- Commit style matches existing repo: short imperative subject, body explains why. Always include `Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>`.

**Critical implementation notes:**
- **CONFIG keys read order:** `const CONFIG = {...}` declaration is at line 1276. The aliases at lines 1340-1349 (`HOLDINGS`, `BANK_HDFC`, `BANK_Canara`, `ACTUAL_STAKING_USD`, `COMPARE_OPENING`, `SALARY_INR`, `EXPENSES_INR`) derive from CONFIG synchronously after the literal closes. The user-settings deep-merge IIFE **must run between** the CONFIG declaration (line 1335 closing brace) and the alias derivations (line 1340).
- **IDX init order:** `IDX` is currently declared at line 1355 — AFTER the aliases. Per the spec's three options, this plan uses option (a): move the `IDX` declaration to immediately AFTER `CONFIG` (and before the IIFE), so the same IIFE can merge into IDX. No downstream code references IDX between CONFIG and where it currently lives — verified by grep on the line range 1336-1354.
- **Percentage display vs. decimal storage:** `bankInterest.tierN.rate` and `tax.presumptiveProfitRate` are stored as decimals (0.025 = 2.5%). The Settings form shows percentages. Conversion happens at form read/write boundary — see Tasks 6 and 7.
- **IDX values are already in percentage points** (12, 10), not decimals — pass through without conversion.

---

## File Structure

| File | Change | Why |
|------|--------|-----|
| `Finance_Dashboard.html` | Modified throughout | Single-file project — all HTML, CSS, JS lives here. |
| `CLAUDE.md` | Modified | Document Settings tab, the deep-merge loader, `user_settings_v1` key, updated tab count. |
| `README.md` | Modified | Add Settings to tab list, update Quickstart (CONFIG-editing is now optional), update tab count. |

Anchors in `Finance_Dashboard.html` (line numbers reflect state after Markets-tab work, commit `4499d2b`):

- `</style>`: line 349 (insert Settings CSS just before).
- Invoices nav-item: lines 400-403 (insert Settings nav-item between Invoices and Markets at line 404).
- Markets page `</div>` + `</main>`: lines 1268-1271 (insert Settings page block before `</main>`).
- `const CONFIG = {`: line 1276 (closes at line 1335).
- CONFIG-derived aliases: lines 1340-1349.
- `const IDX = ...`: line 1355 (will be moved up to ~line 1336).
- `const TRANSACTIONS = [`: line 1636 (closes at line 1668).
- `function initInvoices(){...}`: line 3134 (insert `initSettings` plus helpers after this).
- `PAGE_TITLES` / `PAGE_INIT`: lines 3293-3294 (add Settings entries).
- `PAGE_INIT_ONCE`: line 3300 (do NOT add Settings — its init reads live state and is cheap; should re-run on every visit so it picks up changes from elsewhere).

---

## Task 1: Zero out CONFIG demo values

**Files:**
- Modify: `Finance_Dashboard.html:1276-1335`

Pure data change. Strip user-specific demo numbers (bank balances, salary, holdings, staking, comparison baseline, defaultMonthlyExpenses) to 0. Keep `ownerName` placeholder, `bankInterest` policy defaults, `tax` defaults, `tradingview` defaults.

- [ ] **Step 1: Confirm current CONFIG layout**

Run: `sed -n '1276,1335p' Finance_Dashboard.html`
Expected: the CONFIG literal as currently committed.

- [ ] **Step 2: Replace demo values with zeros**

Use Edit. Replace:

```js
const CONFIG = {
  // --- Identity (optional, shown in page header) ---
  ownerName: "Your Name",

  // --- Bank balances (₹, snapshot) ---
  banks: {
    primary:   { name: "Primary Bank",   balance: 500000  },
    secondary: { name: "Secondary Bank", balance: 0       },
  },

  // --- Salary & expenses ---
  monthlyNetSalary: 80000,
  defaultMonthlyExpenses: 30000,   // user can override in UI; persists to localStorage

  // --- Crypto holdings (Exchange-style spot; works with any exchange that supports these tokens) ---
  holdings: {
    eth:    0.5,
    sol:    2,
    usdCash:1000,
    usdc:   5000,
    usdt:   0,
  },
  monthlyStakingUSD: 25,

  // --- HDFC-style tiered interest (edit if your bank uses different tiers) ---
  bankInterest: {
    tier1: { upTo: 300000,    rate: 0.025 },  // 2.5% on first ₹3L
    tier2: { upTo: 25000000,  rate: 0.065 },  // 6.5% ₹3L–₹25Cr
    tier3: { rate: 0.05 },                    // 5% above ₹25Cr
  },

  // --- Index comparison baseline (₹ value at start of comparison window) ---
  comparisonOpeningBalance: 500000,
```

With:

```js
const CONFIG = {
  // --- Identity (optional, shown in page header) ---
  ownerName: "Your Name",

  // --- Bank balances (₹, snapshot). Edit in the Settings tab. ---
  banks: {
    primary:   { name: "Primary Bank",   balance: 0 },
    secondary: { name: "Secondary Bank", balance: 0 },
  },

  // --- Salary & expenses. Edit salary in Settings; expenses on Projections. ---
  monthlyNetSalary: 0,
  defaultMonthlyExpenses: 0,   // user enters real value via Projections inline editor

  // --- Crypto holdings (exchange-side defaults). Edit in the Settings tab. ---
  holdings: {
    eth:    0,
    sol:    0,
    usdCash:0,
    usdc:   0,
    usdt:   0,
  },
  monthlyStakingUSD: 0,

  // --- HDFC-style tiered interest. Edit in the Settings tab (Advanced section). ---
  bankInterest: {
    tier1: { upTo: 300000,    rate: 0.025 },  // 2.5% on first ₹3L
    tier2: { upTo: 25000000,  rate: 0.065 },  // 6.5% ₹3L–₹25Cr
    tier3: { rate: 0.05 },                    // 5% above ₹25Cr
  },

  // --- Index comparison baseline (₹). Edit in the Settings tab. ---
  comparisonOpeningBalance: 0,
```

- [ ] **Step 3: Verify CONFIG still parses**

Run from `/Users/sid/Desktop/Finance-Template`:
```
node -e "$(sed -n '/^const CONFIG = {/,/^};/p' Finance_Dashboard.html); console.log(JSON.stringify({sal:CONFIG.monthlyNetSalary, bal:CONFIG.banks.primary.balance, eth:CONFIG.holdings.eth, cmp:CONFIG.comparisonOpeningBalance, dme:CONFIG.defaultMonthlyExpenses, biR:CONFIG.bankInterest.tier1.rate, owner:CONFIG.ownerName}));"
```
Expected: `{"sal":0,"bal":0,"eth":0,"cmp":0,"dme":0,"biR":0.025,"owner":"Your Name"}` — confirms zeroed user values + preserved policy defaults.

- [ ] **Step 4: Open the dashboard and visually confirm**

Open `Finance_Dashboard.html` in a browser. Overview KPIs should show ₹0 for bank totals, salary, holdings. No console errors. Sample transactions are still visible (Task 2 empties them next).

- [ ] **Step 5: Commit**

```bash
cd /Users/sid/Desktop/Finance-Template
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Zero out user-specific demo values in CONFIG

Bank balances, salary, holdings, staking, comparison baseline, and
defaultMonthlyExpenses all set to 0 so the dashboard ships in a
ship-empty state. Owner name placeholder, bankInterest tiers, tax
defaults, and Markets-tab defaults are kept (policy / UI defaults,
not user data).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Empty the sample TRANSACTIONS array

**Files:**
- Modify: `Finance_Dashboard.html:1636-1668`

Replace the ~30-line sample data array with `const TRANSACTIONS = [];`. Verify Cash Flow / Transactions tabs render without crashing on the empty array.

- [ ] **Step 1: Confirm current array bounds**

Run: `sed -n '1635,1670p' Finance_Dashboard.html`
Expected: the array opens at line 1636 with sample data and closes with `];` at line 1668.

- [ ] **Step 2: Replace the entire array with an empty literal**

Use Edit. Replace:

```js
const TRANSACTIONS = [
  ["01-Apr-26","Salary Credit","Salary","Primary Bank","Income",80000],
```

With:

```js
const TRANSACTIONS = [
  // Sample data removed for ship-empty. User adds real transactions via
  // the "Add Transaction" UI on the Transactions tab (persisted to
  // localStorage['manual_entries']).
  ["01-Apr-26","Salary Credit","Salary","Primary Bank","Income",80000],
```

Then use Edit again to remove ALL the sample rows. Easiest path: use a multi-line Edit that replaces the entire block from the first sample row through the closing `];`. Concretely, run this command to confirm the exact bounds:

```
awk '/^const TRANSACTIONS = \[/{p=1} p{print NR":"$0} /^\];/{if(p)exit}' Finance_Dashboard.html
```

Then use a single Edit replacing the block. The final result must be:

```js
const TRANSACTIONS = [
  // Sample data removed for ship-empty. User adds real transactions via
  // the "Add Transaction" UI on the Transactions tab (persisted to
  // localStorage['manual_entries']).
];
```

- [ ] **Step 3: Verify file still parses**

Run: `node --check <(sed -n '/^<script>/,/^<\/script>/p' /Users/sid/Desktop/Finance-Template/Finance_Dashboard.html | sed '1d;$d') 2>&1 | head`
Expected: no output (clean parse).

- [ ] **Step 4: Open dashboard and walk every tab**

Open `Finance_Dashboard.html`. Click each tab in order: Overview, Cash Flow, Portfolio, Projections, Inflight, Transactions, Goals, FIRC, Taxes, Invoices, Markets.

Expected per tab:
- **Overview:** loads, KPIs show ₹0 / placeholder values. No console errors.
- **Cash Flow:** loads. Charts may render with empty datasets / zero bars. Critically — no exceptions thrown. If a chart throws because it indexed into an empty array, note the exact error and stop — fix the chart's empty-array handling as part of this task.
- **Transactions:** loads. Transaction list shows zero rows (or an empty-state message if one exists). Filtering controls still render.
- **Other tabs:** all load without throwing.

If any tab throws, the fix is to add an empty-array guard at the point of the throw. Add the guard, commit it as part of this task, re-verify.

- [ ] **Step 5: Commit**

```bash
cd /Users/sid/Desktop/Finance-Template
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Empty the sample TRANSACTIONS array for ship-empty

The hardcoded sample transactions were illustrative demo data. With
the ship-empty design, the dashboard's transaction history is the
user's manual_entries (already user-editable via Add Transaction UI).
TRANSACTIONS is now [] with a comment explaining the rename.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Reorder IDX above the aliases, then add `applyUserSettings` IIFE

**Files:**
- Modify: `Finance_Dashboard.html:1335-1356`

Two coupled changes that must ship in one commit: (a) move `const IDX` declaration to immediately after CONFIG (before the aliases), (b) insert the `applyUserSettings` IIFE between CONFIG/IDX and the aliases. The IIFE no-ops until Task 7 writes a `user_settings_v1` value — but the load-time infrastructure must exist now so Task 7 can rely on it.

- [ ] **Step 1: Confirm current layout**

Run: `sed -n '1335,1360p' Finance_Dashboard.html`
Expected:
```
1335 };
1336
1337 // ══════════════════════════════════════════
1338 // CONSTANTS — derived from CONFIG (do not edit; edit CONFIG above)
1339 // ══════════════════════════════════════════
1340 const HOLDINGS           = CONFIG.holdings;
1341 const BANK_HDFC          = CONFIG.banks.primary.balance;
... (aliases)
1352 const LIVE = { usdInr:84.0, eth:2017.04, sol:82.33, loaded:false, lastFetch:null };
1353
1354 // Index rates (editable)
1355 const IDX = { nifty:12, sp:10 };
```

- [ ] **Step 2: Replace the block to (a) move IDX up, (b) insert the IIFE**

Use Edit. Replace:

```js
};

// ══════════════════════════════════════════
// CONSTANTS — derived from CONFIG (do not edit; edit CONFIG above)
// ══════════════════════════════════════════
const HOLDINGS           = CONFIG.holdings;
const BANK_HDFC          = CONFIG.banks.primary.balance;
const BANK_Canara         = CONFIG.banks.secondary.balance;
const ACTUAL_STAKING_USD = CONFIG.monthlyStakingUSD;
const COMPARE_OPENING    = CONFIG.comparisonOpeningBalance;
const SALARY_INR         = CONFIG.monthlyNetSalary;
// Monthly expenses — editable, persisted to localStorage
function getMonthlyExpenses(){return parseFloat(localStorage.getItem('monthly_expenses'))||CONFIG.defaultMonthlyExpenses;}
function saveMonthlyExpenses(v){localStorage.setItem('monthly_expenses',v);}
const EXPENSES_INR = getMonthlyExpenses(); // kept for initial projection table; live reads use getMonthlyExpenses()

// Live state (defaults, updated by API)
const LIVE = { usdInr:84.0, eth:2017.04, sol:82.33, loaded:false, lastFetch:null };

// Index rates (editable)
const IDX = { nifty:12, sp:10 };
```

With:

```js
};

// Index returns (assumed annual %). Declared before aliases so the
// applyUserSettings IIFE below can merge user overrides into both
// CONFIG and IDX in one place.
const IDX = { nifty:12, sp:10 };

// ══════════════════════════════════════════
// Apply user_settings_v1 overrides
// On page load, deep-merge any saved settings into CONFIG and IDX
// BEFORE the legacy aliases below derive their values. The Settings
// tab writes to localStorage['user_settings_v1']; if absent, the IIFE
// is a no-op and the ship-empty defaults remain.
// ══════════════════════════════════════════
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
  // CONFIG keys (everything except idx)
  const { idx, ...configOverrides } = stored;
  deepMerge(CONFIG, configOverrides);
  if (idx) Object.assign(IDX, idx);
})();

// ══════════════════════════════════════════
// CONSTANTS — derived from CONFIG (do not edit; edit CONFIG above)
// ══════════════════════════════════════════
const HOLDINGS           = CONFIG.holdings;
const BANK_HDFC          = CONFIG.banks.primary.balance;
const BANK_Canara         = CONFIG.banks.secondary.balance;
const ACTUAL_STAKING_USD = CONFIG.monthlyStakingUSD;
const COMPARE_OPENING    = CONFIG.comparisonOpeningBalance;
const SALARY_INR         = CONFIG.monthlyNetSalary;
// Monthly expenses — editable, persisted to localStorage
function getMonthlyExpenses(){return parseFloat(localStorage.getItem('monthly_expenses'))||CONFIG.defaultMonthlyExpenses;}
function saveMonthlyExpenses(v){localStorage.setItem('monthly_expenses',v);}
const EXPENSES_INR = getMonthlyExpenses(); // kept for initial projection table; live reads use getMonthlyExpenses()

// Live state (defaults, updated by API)
const LIVE = { usdInr:84.0, eth:2017.04, sol:82.33, loaded:false, lastFetch:null };
```

- [ ] **Step 3: Verify behavior in browser**

Open `Finance_Dashboard.html`. Without any `user_settings_v1` set, the IIFE early-returns. Dashboard should look identical to its post-Task-2 state — all zeros, no console errors.

Then in devtools console, simulate a saved setting:
```js
localStorage.setItem('user_settings_v1', JSON.stringify({
  ownerName: "Test User",
  monthlyNetSalary: 75000,
  banks: { primary: { name: "Test Bank", balance: 100000 } },
  idx: { nifty: 15, sp: 11 }
}));
location.reload();
```
Expected after reload:
- Console: `CONFIG.monthlyNetSalary` → `75000`, `CONFIG.ownerName` → `"Test User"`, `CONFIG.banks.primary.balance` → `100000`, `CONFIG.banks.primary.name` → `"Test Bank"`, `CONFIG.banks.secondary.balance` → `0` (untouched), `IDX.nifty` → `15`, `IDX.sp` → `11`.
- Brand sub-title shows "Test User" (existing initOverview line 1892 handles this).

Then clean up:
```js
localStorage.removeItem('user_settings_v1');
location.reload();
```
Expected: back to ship-empty zeros.

- [ ] **Step 4: Commit**

```bash
cd /Users/sid/Desktop/Finance-Template
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Add applyUserSettings IIFE; move IDX above CONFIG aliases

The IIFE runs immediately after CONFIG (and the moved-up IDX) and
before the legacy aliases (HOLDINGS, BANK_HDFC, etc.) derive. If a
user_settings_v1 value exists in localStorage, it's deep-merged into
CONFIG and IDX so all downstream code reads the user's values.
Empty/missing returns a no-op so defaults stay in place.

IDX moved above the IIFE so the same loader can populate it; no
downstream code references IDX before its previous location.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Add Settings tab scaffolding

**Files:**
- Modify: `Finance_Dashboard.html:400-407` (nav item insert)
- Modify: `Finance_Dashboard.html:1268-1271` (page div insert)
- Modify: `Finance_Dashboard.html:3134` (stub init function)
- Modify: `Finance_Dashboard.html:3293-3294` (PAGE_TITLES, PAGE_INIT)

Five edits in one commit — produces a clickable empty Settings tab. Mirrors the pattern used by the Markets tab work (commit `23e365d`).

- [ ] **Step 1: Add Settings nav item between Invoices and Markets**

Use Edit. Replace:

```html
    <div class="nav-item" onclick="switchPage('invoices',this)">
      <svg viewBox="0 0 24 24"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="9" y1="15" x2="15" y2="15"/><line x1="9" y1="11" x2="15" y2="11"/><line x1="9" y1="19" x2="11" y2="19"/></svg>
      Invoices
    </div>
    <div class="nav-item" onclick="switchPage('tradingview',this)">
```

With:

```html
    <div class="nav-item" onclick="switchPage('invoices',this)">
      <svg viewBox="0 0 24 24"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="9" y1="15" x2="15" y2="15"/><line x1="9" y1="11" x2="15" y2="11"/><line x1="9" y1="19" x2="11" y2="19"/></svg>
      Invoices
    </div>
    <div class="nav-item" onclick="switchPage('settings',this)">
      <svg viewBox="0 0 24 24"><circle cx="12" cy="12" r="3"/><path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"/></svg>
      Settings
    </div>
    <div class="nav-item" onclick="switchPage('tradingview',this)">
```

- [ ] **Step 2: Add empty Settings page block before `</main>`**

Use Edit. Replace:

```html
    <div class="tv-widget-card tv-calendar" id="tvCalendar"></div>

  </div>

</main>
```

With:

```html
    <div class="tv-widget-card tv-calendar" id="tvCalendar"></div>

  </div>

  <div class="page" id="page-settings">
    <!-- Settings form, populated and styled in subsequent tasks. -->
  </div>

</main>
```

- [ ] **Step 3: Add stub `initSettings` immediately after `initInvoices`**

Use Edit. Replace:

```js
function initInvoices(){renderInvoices();}
```

With:

```js
function initInvoices(){renderInvoices();}

// ══════════════════════════════════════════
// SETTINGS TAB
// ══════════════════════════════════════════
function initSettings() {
  // Form population, save, reset wired in subsequent tasks.
}
```

- [ ] **Step 4: Register Settings in PAGE_TITLES and PAGE_INIT**

Use Edit. Replace:

```js
const PAGE_TITLES = {overview:'Overview',cashflow:'Cash Flow',portfolio:'Portfolio',projections:'Projections',inflight:'Inflight',transactions:'Transactions',goals:'Goals',firc:'FIRC / FX Tracker',taxes:'Taxes — FY 2026-27',invoices:'Invoices',tradingview:'Markets'};
const PAGE_INIT   = {overview:initOverview,cashflow:initCashFlow,portfolio:initPortfolio,projections:initProjections,goals:initGoals,firc:initFirc,taxes:initTaxes,invoices:initInvoices,tradingview:initTradingView};
```

With:

```js
const PAGE_TITLES = {overview:'Overview',cashflow:'Cash Flow',portfolio:'Portfolio',projections:'Projections',inflight:'Inflight',transactions:'Transactions',goals:'Goals',firc:'FIRC / FX Tracker',taxes:'Taxes — FY 2026-27',invoices:'Invoices',settings:'Settings',tradingview:'Markets'};
const PAGE_INIT   = {overview:initOverview,cashflow:initCashFlow,portfolio:initPortfolio,projections:initProjections,goals:initGoals,firc:initFirc,taxes:initTaxes,invoices:initInvoices,settings:initSettings,tradingview:initTradingView};
```

Note: Settings is intentionally NOT added to `PAGE_INIT_ONCE`. The Settings page should re-init on every visit so it picks up current effective values (in case the user opened it, looked elsewhere, came back). Its init is cheap — no iframes.

- [ ] **Step 5: Verify in browser**

Open `Finance_Dashboard.html`. Click the new **Settings** nav item.
Expected: page area becomes empty (the placeholder comment), header reads "Settings". Click another tab and back — no console errors. Click Markets — Markets tab still works (no regressions from inserting Settings before it in the registry order).

- [ ] **Step 6: Commit**

```bash
cd /Users/sid/Desktop/Finance-Template
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Add Settings tab scaffolding: nav item, page div, registry, init stub

Wires up the Settings tab end-to-end with no UI yet. Clicking the nav
item shows an empty page titled "Settings" and calls a stub
initSettings(). Settings is NOT in PAGE_INIT_ONCE — it must re-init
each visit so it shows current effective values.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Add Settings page HTML + CSS

**Files:**
- Modify: `Finance_Dashboard.html:349` (CSS insert before `</style>`)
- Modify: `Finance_Dashboard.html` — replace the empty Settings page div with the full form.

The form is static HTML — input IDs are stable so Tasks 6/7/8 can wire them. CSS uses existing color variables.

- [ ] **Step 1: Add CSS rules immediately before `</style>`**

Use Edit. Replace:

```css

/* MARKETS TAB */
```

With:

```css

/* SETTINGS TAB */
.set-page{max-width:760px;margin:0 auto;padding-bottom:60px}
.set-section{background:var(--card);border:1px solid var(--border);border-radius:12px;padding:18px 20px;margin-bottom:16px;box-shadow:var(--shadow-sm)}
.set-section-title{font-size:13px;font-weight:700;color:var(--text);margin-bottom:14px;text-transform:uppercase;letter-spacing:0.8px}
.set-row{display:flex;gap:12px;flex-wrap:wrap;align-items:flex-end;margin-bottom:12px}
.set-row:last-child{margin-bottom:0}
.set-field{display:flex;flex-direction:column;flex:1;min-width:160px}
.set-field-label{font-size:11.5px;font-weight:500;color:var(--muted);margin-bottom:5px}
.set-field-suffix{font-size:11px;color:var(--faint);margin-left:6px}
.set-input{padding:9px 11px;border:1px solid var(--border);border-radius:8px;font-size:13px;font-family:inherit;background:var(--bg);color:var(--text);width:100%}
.set-input:focus{outline:none;border-color:var(--blue);background:var(--card)}
.set-note{font-size:12px;color:var(--muted);margin-top:4px;font-style:italic}
.set-advanced{margin-bottom:16px}
.set-advanced-head{background:var(--card);border:1px solid var(--border);border-radius:12px;padding:14px 18px;display:flex;align-items:center;justify-content:space-between;cursor:pointer;user-select:none;box-shadow:var(--shadow-sm)}
.set-advanced-title{font-size:13px;font-weight:700;color:var(--text);text-transform:uppercase;letter-spacing:0.8px}
.set-advanced-caret{transition:transform 0.15s;color:var(--muted)}
.set-advanced.open .set-advanced-caret{transform:rotate(90deg)}
.set-advanced-body{display:none;margin-top:12px}
.set-advanced.open .set-advanced-body{display:block}
.set-actions{display:flex;gap:10px;align-items:center;margin-top:18px}
.set-btn{padding:10px 18px;border:1px solid var(--border);border-radius:8px;background:var(--card);color:var(--text);font-size:13px;font-weight:500;cursor:pointer;transition:background 0.1s}
.set-btn:hover{background:var(--s2)}
.set-btn-primary{background:var(--blue);color:#fff;border-color:var(--blue)}
.set-btn-primary:hover{background:#1E40AF}
.set-btn-danger{background:var(--card);color:var(--red);border-color:var(--red)}
.set-btn-danger:hover{background:var(--red);color:#fff}
.set-status{font-size:12.5px;color:var(--muted);margin-left:8px}
.set-status.ok{color:var(--green)}
.set-status.err{color:var(--red)}
.set-danger-zone{background:var(--card);border:1px solid var(--red);border-radius:12px;padding:18px 20px;margin-top:30px;box-shadow:var(--shadow-sm)}
.set-danger-title{font-size:13px;font-weight:700;color:var(--red);margin-bottom:6px;text-transform:uppercase;letter-spacing:0.8px}
.set-danger-body{font-size:12.5px;color:var(--muted);margin-bottom:14px;line-height:1.5}

/* MARKETS TAB */
```

- [ ] **Step 2: Replace the empty Settings page with the full form**

Use Edit. Replace:

```html
  <div class="page" id="page-settings">
    <!-- Settings form, populated and styled in subsequent tasks. -->
  </div>
```

With:

```html
  <div class="page" id="page-settings">
   <div class="set-page">

    <div class="set-section">
      <div class="set-section-title">Identity</div>
      <div class="set-row">
        <div class="set-field">
          <label class="set-field-label" for="setOwnerName">Owner name</label>
          <input type="text" class="set-input" id="setOwnerName" placeholder="Your Name">
        </div>
      </div>
    </div>

    <div class="set-section">
      <div class="set-section-title">Banks</div>
      <div class="set-row">
        <div class="set-field">
          <label class="set-field-label" for="setPrimaryBankName">Primary bank name</label>
          <input type="text" class="set-input" id="setPrimaryBankName">
        </div>
        <div class="set-field">
          <label class="set-field-label" for="setPrimaryBankBal">Primary balance (₹)</label>
          <input type="number" class="set-input" id="setPrimaryBankBal" min="0" step="1">
        </div>
      </div>
      <div class="set-row">
        <div class="set-field">
          <label class="set-field-label" for="setSecondaryBankName">Secondary bank name</label>
          <input type="text" class="set-input" id="setSecondaryBankName">
        </div>
        <div class="set-field">
          <label class="set-field-label" for="setSecondaryBankBal">Secondary balance (₹)</label>
          <input type="number" class="set-input" id="setSecondaryBankBal" min="0" step="1">
        </div>
      </div>
      <div class="set-note">Add more accounts via "+ Add Account" on the Overview tab.</div>
    </div>

    <div class="set-section">
      <div class="set-section-title">Income</div>
      <div class="set-row">
        <div class="set-field">
          <label class="set-field-label" for="setSalary">Monthly net salary (₹)</label>
          <input type="number" class="set-input" id="setSalary" min="0" step="1">
        </div>
      </div>
      <div class="set-note">Monthly expenses are edited on the Projections tab.</div>
    </div>

    <div class="set-section">
      <div class="set-section-title">Crypto Holdings (exchange-side defaults)</div>
      <div class="set-row">
        <div class="set-field">
          <label class="set-field-label" for="setEth">ETH</label>
          <input type="number" class="set-input" id="setEth" min="0" step="0.0001">
        </div>
        <div class="set-field">
          <label class="set-field-label" for="setSol">SOL</label>
          <input type="number" class="set-input" id="setSol" min="0" step="0.0001">
        </div>
      </div>
      <div class="set-row">
        <div class="set-field">
          <label class="set-field-label" for="setUsdc">USDC</label>
          <input type="number" class="set-input" id="setUsdc" min="0" step="0.01">
        </div>
        <div class="set-field">
          <label class="set-field-label" for="setUsdt">USDT</label>
          <input type="number" class="set-input" id="setUsdt" min="0" step="0.01">
        </div>
      </div>
      <div class="set-row">
        <div class="set-field">
          <label class="set-field-label" for="setUsdCash">USD cash</label>
          <input type="number" class="set-input" id="setUsdCash" min="0" step="0.01">
        </div>
        <div class="set-field">
          <label class="set-field-label" for="setStakingUsd">Monthly staking (USD)</label>
          <input type="number" class="set-input" id="setStakingUsd" min="0" step="0.01">
        </div>
      </div>
    </div>

    <div class="set-advanced" id="setAdvanced">
      <div class="set-advanced-head" onclick="document.getElementById('setAdvanced').classList.toggle('open')">
        <div class="set-advanced-title">Advanced</div>
        <svg class="set-advanced-caret" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="9 18 15 12 9 6"/></svg>
      </div>
      <div class="set-advanced-body">

        <div class="set-section">
          <div class="set-section-title">Index baseline & returns</div>
          <div class="set-row">
            <div class="set-field">
              <label class="set-field-label" for="setComparisonBal">Comparison opening balance (₹)</label>
              <input type="number" class="set-input" id="setComparisonBal" min="0" step="1">
            </div>
          </div>
          <div class="set-row">
            <div class="set-field">
              <label class="set-field-label" for="setIdxNifty">Nifty assumed annual return (%)</label>
              <input type="number" class="set-input" id="setIdxNifty" min="0" step="0.1">
            </div>
            <div class="set-field">
              <label class="set-field-label" for="setIdxSp">S&amp;P assumed annual return (%)</label>
              <input type="number" class="set-input" id="setIdxSp" min="0" step="0.1">
            </div>
          </div>
        </div>

        <div class="set-section">
          <div class="set-section-title">Bank interest schedule</div>
          <div class="set-row">
            <div class="set-field">
              <label class="set-field-label" for="setBiTier1Cap">Tier 1: up to (₹)</label>
              <input type="number" class="set-input" id="setBiTier1Cap" min="0" step="100">
            </div>
            <div class="set-field">
              <label class="set-field-label" for="setBiTier1Rate">Tier 1: rate (%)</label>
              <input type="number" class="set-input" id="setBiTier1Rate" min="0" step="0.1">
            </div>
          </div>
          <div class="set-row">
            <div class="set-field">
              <label class="set-field-label" for="setBiTier2Cap">Tier 2: up to (₹)</label>
              <input type="number" class="set-input" id="setBiTier2Cap" min="0" step="100">
            </div>
            <div class="set-field">
              <label class="set-field-label" for="setBiTier2Rate">Tier 2: rate (%)</label>
              <input type="number" class="set-input" id="setBiTier2Rate" min="0" step="0.1">
            </div>
          </div>
          <div class="set-row">
            <div class="set-field">
              <label class="set-field-label" for="setBiTier3Rate">Tier 3 (above tier 2 cap): rate (%)</label>
              <input type="number" class="set-input" id="setBiTier3Rate" min="0" step="0.1">
            </div>
          </div>
        </div>

        <div class="set-section">
          <div class="set-section-title">Tax</div>
          <div class="set-row">
            <div class="set-field">
              <label class="set-field-label" for="setTaxPresumptive">Presumptive profit rate (44AD, %)</label>
              <input type="number" class="set-input" id="setTaxPresumptive" min="0" max="100" step="0.1">
            </div>
          </div>
        </div>

      </div>
    </div>

    <div class="set-actions">
      <button class="set-btn set-btn-primary" id="setSaveBtn" onclick="settingsSave()">Save changes</button>
      <button class="set-btn" id="setCancelBtn" onclick="settingsCancel()">Cancel</button>
      <span class="set-status" id="setStatus"></span>
    </div>

    <div class="set-danger-zone">
      <div class="set-danger-title">Danger zone</div>
      <div class="set-danger-body">
        Wipe all locally-stored data — Settings, bank accounts, transactions, goals, FIRC entries, tax inputs, invoices, and Markets watchlist. The page will reload.
      </div>
      <button class="set-btn set-btn-danger" id="setResetBtn" onclick="settingsReset()">Reset everything</button>
    </div>

   </div>
  </div>
```

- [ ] **Step 3: Verify in browser**

Open `Finance_Dashboard.html`. Click Settings.
Expected: form sections render (Identity, Banks, Income, Crypto Holdings, collapsed "Advanced" panel, Save/Cancel/Status row, red Danger zone with Reset button). All inputs are empty (Task 6 populates them next). Click "Advanced" header — section expands and shows Index baseline, Bank interest schedule, Tax sub-sections. Clicking Save, Cancel, or Reset throws ReferenceError in console (intentional — those handlers ship in Tasks 7/8). Layout looks consistent with other dashboard tabs.

- [ ] **Step 4: Commit**

```bash
cd /Users/sid/Desktop/Finance-Template
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Add Settings page HTML and CSS

Static form with sections for Identity, Banks, Income, Crypto Holdings,
plus a collapsible Advanced section (Index baseline & returns, Bank
interest schedule, Tax presumptive rate). Save/Cancel/Reset buttons
are present but un-wired — handlers come in subsequent tasks.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Populate form inputs from current effective values

**Files:**
- Modify: `Finance_Dashboard.html` — body of `initSettings()` plus a helper.

`initSettings` reads from the merged `CONFIG` and `IDX` (post-IIFE) and sets each input's `.value`. Rates/percentages convert decimal → display percent.

- [ ] **Step 1: Replace `initSettings` stub with populator**

Use Edit. Replace:

```js
// ══════════════════════════════════════════
// SETTINGS TAB
// ══════════════════════════════════════════
function initSettings() {
  // Form population, save, reset wired in subsequent tasks.
}
```

With:

```js
// ══════════════════════════════════════════
// SETTINGS TAB
// ══════════════════════════════════════════
function initSettings() {
  settingsPopulate();
  // Clear any stale save status from a previous visit.
  const s = document.getElementById('setStatus');
  if (s) { s.textContent = ''; s.className = 'set-status'; }
}

// Read current effective values (post-IIFE CONFIG + IDX) into the form inputs.
// Rates stored as decimals (0.025) are shown as percentages (2.5).
function settingsPopulate() {
  const v = (id, val) => { const el = document.getElementById(id); if (el != null) el.value = val; };

  v('setOwnerName',         CONFIG.ownerName);
  v('setPrimaryBankName',   CONFIG.banks.primary.name);
  v('setPrimaryBankBal',    CONFIG.banks.primary.balance);
  v('setSecondaryBankName', CONFIG.banks.secondary.name);
  v('setSecondaryBankBal',  CONFIG.banks.secondary.balance);
  v('setSalary',            CONFIG.monthlyNetSalary);
  v('setEth',               CONFIG.holdings.eth);
  v('setSol',               CONFIG.holdings.sol);
  v('setUsdc',              CONFIG.holdings.usdc);
  v('setUsdt',              CONFIG.holdings.usdt);
  v('setUsdCash',           CONFIG.holdings.usdCash);
  v('setStakingUsd',        CONFIG.monthlyStakingUSD);

  // Advanced
  v('setComparisonBal',     CONFIG.comparisonOpeningBalance);
  v('setIdxNifty',          IDX.nifty);
  v('setIdxSp',             IDX.sp);
  v('setBiTier1Cap',        CONFIG.bankInterest.tier1.upTo);
  v('setBiTier1Rate',       +(CONFIG.bankInterest.tier1.rate * 100).toFixed(4));
  v('setBiTier2Cap',        CONFIG.bankInterest.tier2.upTo);
  v('setBiTier2Rate',       +(CONFIG.bankInterest.tier2.rate * 100).toFixed(4));
  v('setBiTier3Rate',       +(CONFIG.bankInterest.tier3.rate * 100).toFixed(4));
  v('setTaxPresumptive',    +(CONFIG.tax.presumptiveProfitRate * 100).toFixed(4));
}
```

- [ ] **Step 2: Verify in browser**

Hard-refresh (Cmd+Shift+R). Click Settings.
Expected:
- Owner name input shows "Your Name".
- Primary bank name shows "Primary Bank", balance shows 0.
- Secondary bank name shows "Secondary Bank", balance 0.
- Salary 0. All holdings 0. Staking 0.
- Open Advanced — Comparison balance 0. Nifty 12. S&P 10. Tier 1 cap 300000, rate 2.5. Tier 2 cap 25000000, rate 6.5. Tier 3 rate 5. Tax presumptive rate 6.

Then in console:
```js
localStorage.setItem('user_settings_v1', JSON.stringify({
  ownerName: "Real User",
  banks: { primary: { name: "HDFC", balance: 250000 }, secondary: { name: "Canara", balance: 50000 } },
  monthlyNetSalary: 90000,
  holdings: { eth: 1.2, sol: 5, usdc: 2000, usdt: 0, usdCash: 500 },
  monthlyStakingUSD: 40,
  comparisonOpeningBalance: 300000,
  bankInterest: { tier1: { upTo: 500000, rate: 0.03 }, tier2: { upTo: 10000000, rate: 0.07 }, tier3: { rate: 0.04 } },
  tax: { presumptiveProfitRate: 0.08 },
  idx: { nifty: 14, sp: 10.5 }
}));
location.reload();
```
Click Settings. Expected: every input reflects the user-overridden value. Tier 1 rate shows `3` (not `0.03`). Tax presumptive shows `8`. Nifty shows `14`. Then clean up:
```js
localStorage.removeItem('user_settings_v1');
location.reload();
```

- [ ] **Step 3: Commit**

```bash
cd /Users/sid/Desktop/Finance-Template
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Populate Settings form inputs from current effective values

initSettings calls settingsPopulate which reads merged CONFIG and IDX
and writes to each input's .value. Decimal rates (bankInterest,
tax.presumptiveProfitRate) are converted to display percentages.
toFixed(4) trims float artifacts without losing precision.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Wire Save and Cancel

**Files:**
- Modify: `Finance_Dashboard.html` — add `settingsSave()` and `settingsCancel()` immediately after `settingsPopulate()`.

Save reads every input, builds the `user_settings_v1` shape, writes to localStorage, replaces the actions row with a confirmation + Reload button. Cancel calls `settingsPopulate()` to re-fill from current effective values.

- [ ] **Step 1: Add save and cancel handlers**

Use Edit. Replace:

```js
  v('setTaxPresumptive',    +(CONFIG.tax.presumptiveProfitRate * 100).toFixed(4));
}
```

With:

```js
  v('setTaxPresumptive',    +(CONFIG.tax.presumptiveProfitRate * 100).toFixed(4));
}

// Read current form values, persist to localStorage['user_settings_v1'],
// and prompt the user to reload (which re-runs applyUserSettings to apply
// the saved values across the whole dashboard).
function settingsSave() {
  const n  = id => parseFloat(document.getElementById(id).value) || 0;        // number, default 0
  const s  = id => (document.getElementById(id).value || '').trim();          // string, trimmed
  const p  = id => (parseFloat(document.getElementById(id).value) || 0) / 100; // percent → decimal

  const settings = {
    ownerName: s('setOwnerName') || 'Your Name',
    banks: {
      primary:   { name: s('setPrimaryBankName')   || 'Primary Bank',   balance: n('setPrimaryBankBal') },
      secondary: { name: s('setSecondaryBankName') || 'Secondary Bank', balance: n('setSecondaryBankBal') }
    },
    monthlyNetSalary: n('setSalary'),
    holdings: {
      eth:     n('setEth'),
      sol:     n('setSol'),
      usdc:    n('setUsdc'),
      usdt:    n('setUsdt'),
      usdCash: n('setUsdCash')
    },
    monthlyStakingUSD: n('setStakingUsd'),
    comparisonOpeningBalance: n('setComparisonBal'),
    bankInterest: {
      tier1: { upTo: n('setBiTier1Cap'), rate: p('setBiTier1Rate') },
      tier2: { upTo: n('setBiTier2Cap'), rate: p('setBiTier2Rate') },
      tier3: {                            rate: p('setBiTier3Rate') }
    },
    tax: { presumptiveProfitRate: p('setTaxPresumptive') },
    idx: { nifty: n('setIdxNifty'), sp: n('setIdxSp') }
  };

  const status = document.getElementById('setStatus');
  try {
    localStorage.setItem('user_settings_v1', JSON.stringify(settings));
    status.className = 'set-status ok';
    status.innerHTML = '✓ Saved. <a href="#" onclick="location.reload();return false;" style="color:var(--blue);text-decoration:none;font-weight:600">Reload now</a> to apply.';
  } catch (e) {
    status.className = 'set-status err';
    status.textContent = "Couldn't save — your browser blocked storage.";
  }
}

// Discard unsaved edits and re-populate inputs from current effective values.
function settingsCancel() {
  settingsPopulate();
  const s = document.getElementById('setStatus');
  if (s) { s.textContent = 'Changes discarded.'; s.className = 'set-status'; }
}
```

- [ ] **Step 2: Verify save round-trip in browser**

Hard-refresh. Click Settings. Edit values: salary 100000, ETH 2.5, Tier 1 cap 500000, Tier 1 rate 3, Nifty 15. Click **Save changes**.
Expected: status text shows "✓ Saved. Reload now to apply.". Click **Reload now**.
After reload: console `JSON.parse(localStorage.getItem('user_settings_v1'))` returns the saved object; `CONFIG.monthlyNetSalary` is 100000; `CONFIG.holdings.eth` is 2.5; `CONFIG.bankInterest.tier1.rate` is 0.03 (decimal); `IDX.nifty` is 15. Click Settings — inputs show the saved values (salary 100000, ETH 2.5, Tier 1 rate 3, etc.). Overview KPIs reflect the new values.

- [ ] **Step 3: Verify Cancel discards**

Click Settings. Change salary to 99999. Click **Cancel**.
Expected: salary input reverts to the previously-saved value (100000). Status text shows "Changes discarded." `localStorage['user_settings_v1']` is unchanged.

- [ ] **Step 4: Verify empty-input safety**

Click Settings. Clear the Owner name input entirely. Save. Reload.
Expected: ownerName falls back to "Your Name" (the `|| 'Your Name'` fallback in `settingsSave`). Bank names similarly fall back to their string defaults. Numeric inputs fall back to 0 via `parseFloat(...) || 0`. No crashes.

- [ ] **Step 5: Clean up between tests**

Console: `localStorage.removeItem('user_settings_v1'); location.reload();`

- [ ] **Step 6: Commit**

```bash
cd /Users/sid/Desktop/Finance-Template
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Wire Save and Cancel handlers in Settings tab

settingsSave reads every form input, converts percent displays to
decimal storage, writes to localStorage['user_settings_v1'], and
shows an inline 'Reload now' link. Empty strings fall back to safe
defaults (placeholder text, 0 for numbers). settingsCancel discards
unsaved edits by re-populating from current effective values.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Wire Reset everything

**Files:**
- Modify: `Finance_Dashboard.html` — add `settingsReset()` immediately after `settingsCancel()`.

Native confirm dialog → clear all known localStorage keys → reload.

- [ ] **Step 1: Add reset handler**

Use Edit. Replace:

```js
function settingsCancel() {
  settingsPopulate();
  const s = document.getElementById('setStatus');
  if (s) { s.textContent = 'Changes discarded.'; s.className = 'set-status'; }
}
```

With:

```js
function settingsCancel() {
  settingsPopulate();
  const s = document.getElementById('setStatus');
  if (s) { s.textContent = 'Changes discarded.'; s.className = 'set-status'; }
}

// Wipe ALL locally-stored user data and reload to ship-empty defaults.
// Key list mirrors the localStorage table in CLAUDE.md plus tv_state_v1.
function settingsReset() {
  const ok = confirm(
    "This clears all your data — Settings, bank accounts, transactions, " +
    "goals, FIRC entries, tax inputs, invoices, and Markets watchlist. " +
    "The page will reload. Continue?"
  );
  if (!ok) return;
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
  KEYS.forEach(k => { try { localStorage.removeItem(k); } catch (e) {} });
  location.reload();
}
```

- [ ] **Step 2: Verify reset flow in browser**

Hard-refresh. Click Settings and save some values (e.g., salary 50000). Add a transaction on the Transactions tab. Add a custom account via "+ Add Account" on Overview. Add a ticker to the Markets watchlist.

Now click **Reset everything**. Confirm dialog appears with the listed warning.

Click **Cancel** in the dialog.
Expected: nothing happens. Data still intact.

Click **Reset everything** again, then click **OK** in the dialog.
Expected: page reloads. All KPIs back to ₹0. Settings inputs back to defaults. Manual transaction gone. Custom account gone. Markets watchlist back to CONFIG defaults.

Spot-check console:
```js
['user_settings_v1','manual_entries','custom_accounts_v1','tv_state_v1','tax_regime','monthly_expenses']
  .map(k => [k, localStorage.getItem(k)])
```
Expected: every value is `null`.

- [ ] **Step 3: Commit**

```bash
cd /Users/sid/Desktop/Finance-Template
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Wire Reset everything: confirm, clear all localStorage, reload

Native confirm() dialog explicitly lists the data categories that will
be wiped. The KEYS list mirrors CLAUDE.md's localStorage table plus
tv_state_v1. removeItem is wrapped in try/catch so a single failure
doesn't block the rest of the cleanup. Page reloads to the ship-empty
default state.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 9: Update CLAUDE.md and README.md

**Files:**
- Modify: `CLAUDE.md`
- Modify: `README.md`

- [ ] **Step 1: Update tab count in CLAUDE.md page-system section**

Use Edit on `CLAUDE.md`. Replace:

```markdown
Eleven tabs, each a `<div class="page" id="page-{id}">`. Switching calls
```

With:

```markdown
Twelve tabs, each a `<div class="page" id="page-{id}">`. Switching calls
```

- [ ] **Step 2: Add Settings tab section to CLAUDE.md Architecture**

Use Edit on `CLAUDE.md`. Replace:

```markdown
### Markets tab (TradingView widgets)
```

With:

```markdown
### Settings tab

The user's single in-app surface for editing every value previously
hardcoded in `CONFIG` (plus the `IDX` index-return assumptions).
Persists to `localStorage['user_settings_v1']`. On page load, an IIFE
declared immediately after `CONFIG` (and the moved-up `IDX`) deep-merges
the saved settings into both objects BEFORE any aliases (`HOLDINGS`,
`BANK_HDFC`, etc.) derive their values — so all downstream code reads
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
```

- [ ] **Step 3: Add `user_settings_v1` to CLAUDE.md localStorage table**

Use Edit on `CLAUDE.md`. Replace:

```markdown
| `tv_state_v1` | Markets tab state (ticker tape + watchlist grid + initial main-chart symbol). |
```

With:

```markdown
| `tv_state_v1` | Markets tab state (ticker tape + watchlist grid + initial main-chart symbol). |
| `user_settings_v1` | Settings-tab overrides for every overrideable CONFIG value plus IDX. Deep-merged into CONFIG/IDX at page load. |
```

- [ ] **Step 4: Add Settings to README tab list**

Use Edit on `README.md`. Replace:

```markdown
- **Markets** — Embedded TradingView widgets (ticker tape, advanced chart, mini-chart grid, economic calendar). Watchlist editable in-app.
```

With:

```markdown
- **Settings** — Edit your real numbers (bank balances, salary, holdings, bank interest schedule, etc.). Saves to your browser. Includes a Reset-everything button.
- **Markets** — Embedded TradingView widgets (ticker tape, advanced chart, mini-chart grid, economic calendar). Watchlist editable in-app.
```

- [ ] **Step 5: Update Quickstart in README — CONFIG-edit is now optional**

Use Edit on `README.md`. Replace:

```markdown
2. **Edit the `CONFIG` block** at the top of the `<script>` tag in
   `Finance_Dashboard.html`. Set your bank balances, salary, holdings,
   and (optionally) your bank's interest tiers.
```

With:

```markdown
2. **Open the file in a browser and enter your numbers via the
   Settings tab** (bank balances, salary, holdings, bank interest
   schedule, index assumptions). Saves automatically to your browser.
   Alternatively, you can pre-fill defaults by editing the `CONFIG`
   block at the top of the `<script>` tag in `Finance_Dashboard.html`.
```

- [ ] **Step 6: Update tab count in README architecture section**

Use Edit on `README.md`. Replace:

```markdown
- 11 tabs, each a `<div class="page">`. Switching is handled by `switchPage(id)`.
```

With:

```markdown
- 12 tabs, each a `<div class="page">`. Switching is handled by `switchPage(id)`.
```

- [ ] **Step 7: Update the Known limitations section in README**

The current TRANSACTIONS hardcoded-array note is now stale because TRANSACTIONS is `[]`. Use Edit on `README.md`. Replace:

```markdown
- `TRANSACTIONS` is a hardcoded array. To bring in your real transactions,
  edit the array directly or extend the dashboard with a CSV importer
  (PRs welcome).
```

With:

```markdown
- The sample `TRANSACTIONS` array is empty by default. To bring in your
  real transactions, add them via the Transactions tab "Add Transaction"
  UI (stored in `localStorage['manual_entries']`), edit the
  `TRANSACTIONS` array directly, or extend the dashboard with a CSV
  importer (PRs welcome).
```

- [ ] **Step 8: Verify**

`grep -n "Twelve\|user_settings_v1\|Settings tab" /Users/sid/Desktop/Finance-Template/CLAUDE.md`
`grep -n "Settings\|12 tabs" /Users/sid/Desktop/Finance-Template/README.md`
Each grep should show the new lines.

- [ ] **Step 9: Commit**

```bash
cd /Users/sid/Desktop/Finance-Template
git add CLAUDE.md README.md
git commit -m "$(cat <<'EOF'
Document Settings tab + ship-empty defaults in CLAUDE.md and README.md

CLAUDE.md gains a Settings tab architecture section and a
user_settings_v1 row in the localStorage table; tab count updated to
12; new Reset-everything subsection. README.md gets a Settings bullet,
a Quickstart rewrite that puts the in-app Settings tab first (CONFIG
editing is now the alternative), tab count updated to 12, and a
refreshed Known-limitations entry that reflects the empty default
TRANSACTIONS array.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 10: End-to-end manual verification

**Files:** (none modified)

Full acceptance pass against the spec's testing section. This task is for the human in a browser; no commits expected unless something is fixed.

- [ ] **Step 1: Fresh install**

Open devtools console. Run `Object.keys(localStorage).forEach(k=>localStorage.removeItem(k)); location.reload();`. After reload: every Overview KPI shows ₹0 (or "Your Name" for the owner). Cash Flow and Transactions tabs show empty/zero state. No console errors anywhere.

- [ ] **Step 2: Settings round-trip**

Click Settings. Enter:
- Owner: "Real Name"
- Primary: "HDFC", 250000
- Secondary: "Canara", 50000
- Salary: 100000
- ETH 1.5, SOL 3, USDC 1500, USDT 0, USD cash 200
- Monthly staking 35
- (Advanced) Comparison balance 200000, Nifty 14, S&P 11
- (Advanced) Tier 1 cap 400000, rate 2.75; Tier 2 cap 20000000, rate 6.75; Tier 3 rate 4.5
- (Advanced) Tax presumptive 7

Click **Save changes** → status "✓ Saved. Reload now to apply." → click **Reload now**.

Expected after reload:
- Overview brand sub-title reads "Real Name · FY 2026".
- Overview bank KPIs sum to ₹300,000.
- Projections monthly net income reflects ₹100,000 salary.
- Portfolio shows ETH holding 1.5, SOL holding 3.
- Console: `CONFIG.bankInterest.tier1.rate` returns 0.0275, `IDX.nifty` returns 14.

- [ ] **Step 3: Cancel discards**

Click Settings. Change salary to 999999. Click **Cancel**.
Expected: salary input snaps back to 100000. Status shows "Changes discarded." `localStorage['user_settings_v1']` still has the Step-2 saved object (verify in console).

- [ ] **Step 4: Advanced collapsible**

Reload. Click Settings. Advanced section should start collapsed. Click its header — expands. Click again — collapses. Switch to another tab and back — starts collapsed again (no persistence of UI state).

- [ ] **Step 5: Bank interest schedule round-trip**

In Settings → Advanced, change Tier 2 rate from 6.75 to 8.0. Save. Reload. Go to Projections — the projection table's per-month interest column should reflect the new 8% Tier 2 rate (compare against a month where the running balance is in the Tier 2 range).

- [ ] **Step 6: Reset everything**

On Transactions, add a manual transaction. On Goals, add a goal. On Markets, add a ticker. On Settings → Reset everything → confirm.
Expected: page reloads. All custom data gone. Settings inputs back to defaults (owner "Your Name", salaries 0, etc.). Manual transaction gone. Goal gone. Markets watchlist back to CONFIG defaults. `Object.keys(localStorage)` returns `[]` (or only the items the dashboard hasn't yet touched).

- [ ] **Step 7: Markets tab regression**

Reload (clean state). Click Markets. All four widgets (ticker tape, main chart, mini-grid, calendar) render. Verify in Network tab — TradingView script URLs load successfully.

- [ ] **Step 8: All-tabs regression sweep**

Click through Overview, Cash Flow, Portfolio, Projections, Inflight, Transactions, Goals, FIRC, Taxes, Invoices, Settings, Markets — every tab renders without console errors on a freshly-reset dashboard.

- [ ] **Step 9: Final commit (only if a fix was needed during verification)**

If any of the above surfaced a bug that wasn't anticipated, commit the fix with a clear message. Otherwise this task is verification-only — no commit needed.

---

## Self-Review Summary (already performed by plan author)

- **Spec coverage:** CONFIG cleanup → Task 1. Empty TRANSACTIONS → Task 2. IDX reorder + IIFE → Task 3. Tab scaffolding → Task 4. Page HTML/CSS → Task 5. Form population → Task 6. Save/Cancel → Task 7. Reset → Task 8. Documentation → Task 9. Manual verification → Task 10. The spec's three IDX-init options are resolved in Task 3 (option a, move IDX above the IIFE).
- **Function/symbol consistency:** `settingsPopulate`, `settingsSave`, `settingsCancel`, `settingsReset`, `initSettings`, `applyUserSettings`, `user_settings_v1` — names match across every task that references them. Input IDs (`setOwnerName`, `setPrimaryBankName`, etc.) used in Task 5 HTML match Task 6 population and Task 7 read.
- **No placeholders:** All code blocks are complete and runnable. Task 2's array-replacement step uses an `awk` line to locate exact bounds rather than a generic "remove all rows" instruction.
- **Reset key list:** Cross-checked against CLAUDE.md's localStorage table (12 keys) + `tv_state_v1` + `user_settings_v1` = 14 keys total in `settingsReset`.
