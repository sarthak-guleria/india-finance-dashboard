# Markets Tab (TradingView Widgets) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a "Markets" tab to `Finance_Dashboard.html` that embeds free TradingView widgets (ticker tape, advanced chart, mini-chart grid, economic calendar) driven by a user-editable watchlist persisted to `localStorage`.

**Architecture:** All work happens in the single file `Finance_Dashboard.html`. A new opt-in `PAGE_INIT_ONCE` Set guards `initTradingView` so the heavy iframes mount only once per session. A `mountTVWidget(containerId, scriptSrc, config)` helper plays the same destroy-and-recreate role for TradingView iframes that the existing `mkChart()` plays for Chart.js. The watchlist lives in `CONFIG.tradingview` (defaults) and `localStorage['tv_state_v1']` (user edits).

**Tech Stack:** Vanilla HTML/CSS/JS inline in `Finance_Dashboard.html`. TradingView embed widget scripts loaded from `https://s3.tradingview.com/external-embedding/embed-widget-<name>.js`. Browser `localStorage` for persistence.

**Project context for the executing engineer:**
- Spec: `docs/superpowers/specs/2026-06-03-tradingview-markets-tab-design.md`
- Project rules: `CLAUDE.md` — single-file architecture, do NOT split CSS/HTML/JS into separate files.
- No test harness exists. Verification is **manual in a browser**: open `Finance_Dashboard.html` directly via `file://`, perform the action, observe the result. State "Expected" explicitly so reviewers know what to look for.
- Dashboard uses a **light theme** (cream `#F7F4EE` background, white cards). Widget configs use `colorTheme: "light"` for ticker-tape/symbol-overview/events and `theme: "light"` for the advanced chart — TradingView is inconsistent here, don't unify.
- Commit style matches existing repo: short imperative subject, body explains why. No conventional-commit prefixes. Always include `Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>`.

---

## File Structure

Only one source file is modified, plus two documentation files:

| File | Change | Why |
|------|--------|-----|
| `Finance_Dashboard.html` | Modified throughout | Single-file project — all HTML, CSS, JS lives here. |
| `CLAUDE.md` | Modified | Document the new tab, the `tv_state_v1` key, the `mountTVWidget` helper, and the `PAGE_INIT_ONCE` mechanism. |
| `README.md` | Modified (small addition) | One-line user note about the Markets tab. |

Anchors in `Finance_Dashboard.html` the engineer will touch:
- `CONFIG` object literal: lines 1189–1228.
- Sidebar nav items: lines 332–371 (insert new item).
- Page `<div>` blocks: last one ends at line 1182 (insert new page before `</main>` at line 1184).
- `mkChart` helper: lines 1881–1886 (insert `mountTVWidget` near here).
- `PAGE_TITLES` / `PAGE_INIT`: lines 3000–3001.
- `switchPage` function: lines 3003–3014 (add init-once guard).

---

## Task 1: Add `tradingview` block to `CONFIG`

**Files:**
- Modify: `Finance_Dashboard.html:1223-1228`

- [ ] **Step 1: Read the CONFIG block to confirm current state**

Run: `sed -n '1220,1230p' Finance_Dashboard.html`
Expected: see the `tax:` block ending with `};` on line 1228.

- [ ] **Step 2: Insert `tradingview` block before the closing `};`**

Use Edit to change this exact block:

```js
  // --- Tax defaults (India FY 2026-27) ---
  tax: {
    defaultRegime: "new",          // "new" | "old"
    presumptiveProfitRate: 0.06,   // 6% digital, 8% cash (44AD)
  },
};
```

To:

```js
  // --- Tax defaults (India FY 2026-27) ---
  tax: {
    defaultRegime: "new",          // "new" | "old"
    presumptiveProfitRate: 0.06,   // 6% digital, 8% cash (44AD)
  },

  // --- Markets tab (TradingView widgets) ---
  // Use TradingView's EXCHANGE:SYMBOL format. Find symbols at
  // https://www.tradingview.com/symbols/  (edit these, or use the in-app editor)
  tradingview: {
    tickerTape: [
      "BINANCE:BTCUSDT",
      "BINANCE:ETHUSDT",
      "FX_IDC:USDINR",
      "NSE:NIFTY",
      "FOREXCOM:SPXUSD"
    ],
    mainChart: "BINANCE:BTCUSDT",
    watchlistGrid: [
      "BINANCE:BTCUSDT",
      "BINANCE:ETHUSDT",
      "BINANCE:SOLUSDT",
      "NSE:NIFTY"
    ]
  },
};
```

- [ ] **Step 3: Verify in browser**

Open `Finance_Dashboard.html`. Open devtools console. Run: `CONFIG.tradingview.mainChart`
Expected: `"BINANCE:BTCUSDT"`
Also verify the existing dashboard still loads with no console errors.

- [ ] **Step 4: Commit**

```bash
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Add tradingview config block to CONFIG

Defines default ticker tape, main chart symbol, and watchlist-grid
symbols for the upcoming Markets tab. No behavior change yet.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Add opt-in `PAGE_INIT_ONCE` guard to `switchPage`

**Files:**
- Modify: `Finance_Dashboard.html:3000-3014`

**Why:** Existing 8 init functions intentionally re-run on each tab visit (they re-read `LIVE`, `getMonthlyExpenses()`, localStorage). We only want once-per-session init for the Markets tab so 4+ TradingView iframes don't re-mount on every click. Opt-in Set leaves other tabs unchanged.

- [ ] **Step 1: Read the current switchPage block**

Run: `sed -n '3000,3014p' Finance_Dashboard.html`
Expected output:
```js
const PAGE_TITLES = {overview:'Overview', ... };
const PAGE_INIT   = {overview:initOverview, ... };

function switchPage(id, el) {
  document.querySelectorAll('.nav-item').forEach(n=>n.classList.remove('active'));
  document.querySelectorAll('.page').forEach(p=>p.classList.remove('active'));
  el.classList.add('active');
  document.getElementById('page-'+id).classList.add('active');
  document.getElementById('pageTitle').textContent = PAGE_TITLES[id];
  window.scrollTo({top:0,behavior:'smooth'});
  if (PAGE_INIT[id]) PAGE_INIT[id]();
  setTimeout(()=>Object.values(CR).forEach(c=>{try{c.resize()}catch(e){}}),80);
  if (id==='transactions') filterTxns();
  if (id==='inflight')     renderInflight();
}
```

- [ ] **Step 2: Replace with guarded version**

Use Edit. Replace:

```js
const PAGE_TITLES = {overview:'Overview',cashflow:'Cash Flow',portfolio:'Portfolio',projections:'Projections',inflight:'Inflight',transactions:'Transactions',goals:'Goals',firc:'FIRC / FX Tracker',taxes:'Taxes — FY 2026-27',invoices:'Invoices'};
const PAGE_INIT   = {overview:initOverview,cashflow:initCashFlow,portfolio:initPortfolio,projections:initProjections,goals:initGoals,firc:initFirc,taxes:initTaxes,invoices:initInvoices};

function switchPage(id, el) {
  document.querySelectorAll('.nav-item').forEach(n=>n.classList.remove('active'));
  document.querySelectorAll('.page').forEach(p=>p.classList.remove('active'));
  el.classList.add('active');
  document.getElementById('page-'+id).classList.add('active');
  document.getElementById('pageTitle').textContent = PAGE_TITLES[id];
  window.scrollTo({top:0,behavior:'smooth'});
  if (PAGE_INIT[id]) PAGE_INIT[id]();
  setTimeout(()=>Object.values(CR).forEach(c=>{try{c.resize()}catch(e){}}),80);
  if (id==='transactions') filterTxns();
  if (id==='inflight')     renderInflight();
}
```

With:

```js
const PAGE_TITLES = {overview:'Overview',cashflow:'Cash Flow',portfolio:'Portfolio',projections:'Projections',inflight:'Inflight',transactions:'Transactions',goals:'Goals',firc:'FIRC / FX Tracker',taxes:'Taxes — FY 2026-27',invoices:'Invoices'};
const PAGE_INIT   = {overview:initOverview,cashflow:initCashFlow,portfolio:initPortfolio,projections:initProjections,goals:initGoals,firc:initFirc,taxes:initTaxes,invoices:initInvoices};

// Tabs in PAGE_INIT_ONCE get their init function called only on FIRST visit.
// All other tabs re-init on every visit (they re-read live state like LIVE,
// getMonthlyExpenses(), localStorage). Use this set sparingly — only for tabs
// whose init is genuinely expensive (e.g., mounts iframes).
const PAGE_INIT_ONCE = new Set([]);
const initializedPages = new Set();

function switchPage(id, el) {
  document.querySelectorAll('.nav-item').forEach(n=>n.classList.remove('active'));
  document.querySelectorAll('.page').forEach(p=>p.classList.remove('active'));
  el.classList.add('active');
  document.getElementById('page-'+id).classList.add('active');
  document.getElementById('pageTitle').textContent = PAGE_TITLES[id];
  window.scrollTo({top:0,behavior:'smooth'});
  if (PAGE_INIT[id]) {
    if (PAGE_INIT_ONCE.has(id)) {
      if (!initializedPages.has(id)) {
        PAGE_INIT[id]();
        initializedPages.add(id);
      }
    } else {
      PAGE_INIT[id]();
    }
  }
  setTimeout(()=>Object.values(CR).forEach(c=>{try{c.resize()}catch(e){}}),80);
  if (id==='transactions') filterTxns();
  if (id==='inflight')     renderInflight();
}
```

- [ ] **Step 3: Verify existing tabs still re-init**

Open `Finance_Dashboard.html`. Open devtools console. Click Cash Flow → click Overview → click Cash Flow again.
Expected: no console errors. Charts on each tab still render correctly each visit. (`PAGE_INIT_ONCE` is empty so behavior is unchanged for all 8 existing tabs.)

- [ ] **Step 4: Commit**

```bash
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Add opt-in PAGE_INIT_ONCE guard in switchPage

Adds a runtime initializedPages Set consulted only for tabs listed in
PAGE_INIT_ONCE. All existing tabs remain in PAGE_INIT (re-init on every
visit, which they rely on for live state). The Markets tab will opt
into one-shot init so its TradingView iframes don't re-mount on click.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Add `mountTVWidget` helper + `tvState` loader

**Files:**
- Modify: `Finance_Dashboard.html:1881-1886` (insert after `mkChart`)

- [ ] **Step 1: Read the current `mkChart` location**

Run: `sed -n '1880,1890p' Finance_Dashboard.html`
Expected:
```js
const CR = {};
function mkChart(id, cfg) {
  if (CR[id]) { CR[id].destroy(); }
  CR[id] = new Chart(document.getElementById(id), cfg);
  return CR[id];
}
```

- [ ] **Step 2: Add `mountTVWidget` and `tvState` immediately after `mkChart`**

Use Edit. Replace:

```js
const CR = {};
function mkChart(id, cfg) {
  if (CR[id]) { CR[id].destroy(); }
  CR[id] = new Chart(document.getElementById(id), cfg);
  return CR[id];
}
```

With:

```js
const CR = {};
function mkChart(id, cfg) {
  if (CR[id]) { CR[id].destroy(); }
  CR[id] = new Chart(document.getElementById(id), cfg);
  return CR[id];
}

// ══════════════════════════════════════════
// TRADINGVIEW WIDGETS — mount helper + state
// ══════════════════════════════════════════
// Destroy-and-recreate primitive for TradingView embed widgets — same role
// mkChart() plays for Chart.js. Each widget is a <script> tag that
// self-initializes into an iframe inside the given container.
function mountTVWidget(containerId, scriptSrc, config) {
  const el = document.getElementById(containerId);
  if (!el) return;
  el.innerHTML = "";  // tear down any existing iframe
  const script = document.createElement("script");
  script.src = scriptSrc;
  script.async = true;
  script.innerHTML = JSON.stringify(config);
  el.appendChild(script);
}

// Markets tab state: defaults from CONFIG.tradingview, user edits in localStorage.
// mainChart is the INITIAL symbol for the advanced chart only — once mounted,
// the user navigating inside the widget is handled by TradingView's iframe storage
// and is NOT persisted back into tvState.
let tvState = (function loadTVState() {
  try {
    const raw = localStorage.getItem('tv_state_v1');
    if (raw) return JSON.parse(raw);
  } catch (e) { /* fall through to defaults */ }
  return JSON.parse(JSON.stringify(CONFIG.tradingview));  // deep copy
})();
function saveTVState() {
  try { localStorage.setItem('tv_state_v1', JSON.stringify(tvState)); } catch (e) {}
}
```

- [ ] **Step 3: Verify helper is reachable**

Open `Finance_Dashboard.html`. Open devtools console. Run: `typeof mountTVWidget`
Expected: `"function"`. Run: `tvState.mainChart`. Expected: `"BINANCE:BTCUSDT"`.

- [ ] **Step 4: Commit**

```bash
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Add mountTVWidget helper and tvState loader

mountTVWidget mirrors mkChart's destroy-and-recreate pattern: clears
the container, then appends a TradingView embed <script> with inline
JSON config. tvState loads from localStorage 'tv_state_v1' with a
deep-copy fallback to CONFIG.tradingview defaults.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Add sidebar nav item, empty page, PAGE_TITLES/PAGE_INIT entry, opt into init-once

**Files:**
- Modify: `Finance_Dashboard.html:368-371` (nav item)
- Modify: `Finance_Dashboard.html:1182-1184` (page div)
- Modify: `Finance_Dashboard.html:3000-3001` (PAGE_TITLES, PAGE_INIT)
- Modify: `Finance_Dashboard.html` `PAGE_INIT_ONCE` Set (added in Task 2)

**Why bundled:** these four edits together produce one shippable, clickable (if empty) tab. Splitting would leave a half-wired nav item.

- [ ] **Step 1: Add nav item before Invoices**

Read lines 368–371 to confirm structure, then use Edit. Replace:

```html
    <div class="nav-item" onclick="switchPage('invoices',this)">
      <svg viewBox="0 0 24 24"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="9" y1="15" x2="15" y2="15"/><line x1="9" y1="11" x2="15" y2="11"/><line x1="9" y1="19" x2="11" y2="19"/></svg>
      Invoices
    </div>
```

With:

```html
    <div class="nav-item" onclick="switchPage('invoices',this)">
      <svg viewBox="0 0 24 24"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="9" y1="15" x2="15" y2="15"/><line x1="9" y1="11" x2="15" y2="11"/><line x1="9" y1="19" x2="11" y2="19"/></svg>
      Invoices
    </div>
    <div class="nav-item" onclick="switchPage('tradingview',this)">
      <svg viewBox="0 0 24 24"><polyline points="22 12 18 12 15 21 9 3 6 12 2 12"/></svg>
      Markets
    </div>
```

- [ ] **Step 2: Add empty page block before `</main>`**

Read lines 1180–1185 to confirm the closing structure, then use Edit. Replace:

```html
    <div id="goalsEmpty" style="text-align:center;padding:60px 20px;color:var(--faint);display:none">
      <div style="font-size:40px;margin-bottom:12px">🎯</div>
      <div style="font-size:14px;font-weight:600;margin-bottom:6px">No goals yet</div>
      <div style="font-size:12px">Add your first financial goal above — retirement, travel, emergency fund…</div>
    </div>
  </div>

</main>
```

With:

```html
    <div id="goalsEmpty" style="text-align:center;padding:60px 20px;color:var(--faint);display:none">
      <div style="font-size:40px;margin-bottom:12px">🎯</div>
      <div style="font-size:14px;font-weight:600;margin-bottom:6px">No goals yet</div>
      <div style="font-size:12px">Add your first financial goal above — retirement, travel, emergency fund…</div>
    </div>
  </div>

  <div class="page" id="page-tradingview">
    <!-- Symbol manager, ticker tape, advanced chart, mini-chart grid, and economic calendar are added in later tasks. -->
  </div>

</main>
```

- [ ] **Step 3: Add `tradingview` to `PAGE_TITLES` and `PAGE_INIT`**

Use Edit. Replace:

```js
const PAGE_TITLES = {overview:'Overview',cashflow:'Cash Flow',portfolio:'Portfolio',projections:'Projections',inflight:'Inflight',transactions:'Transactions',goals:'Goals',firc:'FIRC / FX Tracker',taxes:'Taxes — FY 2026-27',invoices:'Invoices'};
const PAGE_INIT   = {overview:initOverview,cashflow:initCashFlow,portfolio:initPortfolio,projections:initProjections,goals:initGoals,firc:initFirc,taxes:initTaxes,invoices:initInvoices};
```

With:

```js
const PAGE_TITLES = {overview:'Overview',cashflow:'Cash Flow',portfolio:'Portfolio',projections:'Projections',inflight:'Inflight',transactions:'Transactions',goals:'Goals',firc:'FIRC / FX Tracker',taxes:'Taxes — FY 2026-27',invoices:'Invoices',tradingview:'Markets'};
const PAGE_INIT   = {overview:initOverview,cashflow:initCashFlow,portfolio:initPortfolio,projections:initProjections,goals:initGoals,firc:initFirc,taxes:initTaxes,invoices:initInvoices,tradingview:initTradingView};
```

- [ ] **Step 4: Add `tradingview` to `PAGE_INIT_ONCE`**

Use Edit. Replace:

```js
const PAGE_INIT_ONCE = new Set([]);
```

With:

```js
const PAGE_INIT_ONCE = new Set(['tradingview']);
```

- [ ] **Step 5: Add stub `initTradingView` function**

Find a place near the other `init*` functions. The simplest spot is immediately after `initInvoices` at line 2995 (`function initInvoices(){renderInvoices();}`). Use Edit. Replace:

```js
function initInvoices(){renderInvoices();}
```

With:

```js
function initInvoices(){renderInvoices();}

// ══════════════════════════════════════════
// MARKETS TAB (TradingView)
// ══════════════════════════════════════════
function initTradingView() {
  // Widget mounts are added in later tasks.
}
```

- [ ] **Step 6: Verify in browser**

Open `Finance_Dashboard.html`. Click the new "Markets" nav item.
Expected: page switches; header reads "Markets"; page area is blank (no errors). Click another tab and back — no console errors. (Stub `initTradingView` runs once.)

- [ ] **Step 7: Commit**

```bash
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Add Markets tab scaffolding: nav item, page div, page registry, init stub

Wires up the Markets tab end-to-end with no widgets yet. Clicking the
nav item switches to an empty page titled "Markets" and calls a stub
initTradingView() once per session via PAGE_INIT_ONCE.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Markets page HTML — symbol manager, widget containers, CSS

**Files:**
- Modify: `Finance_Dashboard.html` — replace the empty `<div class="page" id="page-tradingview">` block with the full structure.
- Modify: `Finance_Dashboard.html` — add a small CSS block in the existing `<style>` section for Markets-specific layout.

- [ ] **Step 1: Add CSS rules**

Find the end of the existing `<style>` block. Run: `grep -n "</style>" Finance_Dashboard.html` (expected: a single line near the top, before the body content). Use Edit to add the Markets CSS immediately before `</style>`. Replace:

```css
</style>
```

With:

```css

/* MARKETS TAB */
.tv-section{margin-bottom:18px}
.tv-section-label{font-size:11px;font-weight:700;color:var(--muted);text-transform:uppercase;letter-spacing:0.8px;margin-bottom:8px}
.tv-manager{background:var(--card);border:1px solid var(--border);border-radius:12px;padding:14px 16px;margin-bottom:18px;box-shadow:var(--shadow-sm)}
.tv-manager-head{display:flex;align-items:center;justify-content:space-between;cursor:pointer;user-select:none}
.tv-manager-title{font-size:13px;font-weight:600;color:var(--text)}
.tv-manager-body{margin-top:14px;display:none}
.tv-manager.open .tv-manager-body{display:block}
.tv-manager.open .tv-manager-caret{transform:rotate(90deg)}
.tv-manager-caret{transition:transform 0.15s;color:var(--muted)}
.tv-row{display:flex;gap:8px;flex-wrap:wrap;align-items:center;margin-bottom:10px}
.tv-input{flex:1;min-width:180px;padding:8px 10px;border:1px solid var(--border);border-radius:8px;font-size:13px;font-family:inherit;background:var(--bg);color:var(--text)}
.tv-btn{padding:8px 14px;border:1px solid var(--border);border-radius:8px;background:var(--card);color:var(--text);font-size:12.5px;font-weight:500;cursor:pointer;transition:background 0.1s}
.tv-btn:hover{background:var(--s2)}
.tv-btn-primary{background:var(--blue);color:#fff;border-color:var(--blue)}
.tv-btn-primary:hover{background:#1E40AF}
.tv-chips{display:flex;flex-wrap:wrap;gap:6px;margin-bottom:6px}
.tv-chip{display:inline-flex;align-items:center;gap:6px;padding:5px 10px;background:var(--s2);border-radius:14px;font-size:12px;color:var(--text)}
.tv-chip-x{cursor:pointer;color:var(--muted);font-weight:700;line-height:1}
.tv-chip-x:hover{color:var(--red)}
.tv-hint{font-size:11.5px;color:var(--muted);margin-top:6px}
.tv-hint a{color:var(--blue);text-decoration:none}
.tv-hint a:hover{text-decoration:underline}
.tv-widget-card{background:var(--card);border:1px solid var(--border);border-radius:12px;padding:0;margin-bottom:18px;overflow:hidden;box-shadow:var(--shadow-sm)}
.tv-ticker-tape{height:46px}
.tv-main-chart{height:560px}
.tv-mini-grid{display:grid;grid-template-columns:repeat(2,1fr);gap:14px;margin-bottom:18px}
@media (min-width:900px){.tv-mini-grid{grid-template-columns:repeat(3,1fr)}}
@media (min-width:1300px){.tv-mini-grid{grid-template-columns:repeat(4,1fr)}}
.tv-mini-tile{background:var(--card);border:1px solid var(--border);border-radius:12px;overflow:hidden;height:220px;box-shadow:var(--shadow-sm)}
.tv-calendar{height:520px}
</style>
```

- [ ] **Step 2: Replace the empty Markets page with the full structure**

Use Edit. Replace:

```html
  <div class="page" id="page-tradingview">
    <!-- Symbol manager, ticker tape, advanced chart, mini-chart grid, and economic calendar are added in later tasks. -->
  </div>
```

With:

```html
  <div class="page" id="page-tradingview">

    <!-- Symbol manager (collapsed by default) -->
    <div class="tv-manager" id="tvManager">
      <div class="tv-manager-head" onclick="document.getElementById('tvManager').classList.toggle('open')">
        <div class="tv-manager-title">Edit watchlist</div>
        <svg class="tv-manager-caret" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="9 18 15 12 9 6"/></svg>
      </div>
      <div class="tv-manager-body">

        <div class="tv-section">
          <div class="tv-section-label">Ticker tape (scrolling banner)</div>
          <div class="tv-chips" id="tvChipsTape"></div>
          <div class="tv-row">
            <input type="text" class="tv-input" id="tvInputTape" placeholder="BINANCE:BTCUSDT or NSE:RELIANCE">
            <button class="tv-btn tv-btn-primary" onclick="tvAddTicker('tickerTape')">Add</button>
          </div>
        </div>

        <div class="tv-section">
          <div class="tv-section-label">Watchlist grid (mini-charts)</div>
          <div class="tv-chips" id="tvChipsGrid"></div>
          <div class="tv-row">
            <input type="text" class="tv-input" id="tvInputGrid" placeholder="BINANCE:BTCUSDT or NSE:RELIANCE">
            <button class="tv-btn tv-btn-primary" onclick="tvAddTicker('watchlistGrid')">Add</button>
          </div>
        </div>

        <div class="tv-row" style="justify-content:flex-end;margin-bottom:0">
          <button class="tv-btn" onclick="tvResetDefaults()">Reset to defaults</button>
        </div>

        <div class="tv-hint">Use TradingView's <code>EXCHANGE:SYMBOL</code> format. Find symbols at <a href="https://www.tradingview.com/symbols/" target="_blank" rel="noopener">tradingview.com/symbols</a>.</div>

      </div>
    </div>

    <!-- Ticker tape -->
    <div class="tv-widget-card tv-ticker-tape" id="tvTickerTape"></div>

    <!-- Main advanced chart -->
    <div class="tv-widget-card tv-main-chart" id="tvMainChart"></div>

    <!-- Mini-chart grid: tiles injected in tvRenderGrid() -->
    <div class="tv-mini-grid" id="tvMiniGrid"></div>

    <!-- Economic calendar -->
    <div class="tv-widget-card tv-calendar" id="tvCalendar"></div>

  </div>
```

- [ ] **Step 3: Verify in browser**

Open `Finance_Dashboard.html`. Click Markets.
Expected: see the "Edit watchlist" collapsed panel at top, then four empty card-shaped containers (one tall, one slim, then a grid area, then another tall). No widgets render yet (functions called by buttons don't exist) — that's fine for this task. Click the "Edit watchlist" header — panel expands and shows two empty chip rows + inputs + reset button. Console may warn about undefined `tvAddTicker` only if clicked — don't click yet.

- [ ] **Step 4: Commit**

```bash
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Add Markets page layout: symbol manager UI, widget containers, CSS

Static HTML and CSS for the Markets tab — collapsible watchlist editor
at top, then four empty containers for the ticker tape, main chart,
mini-chart grid, and economic calendar. Widget mounts and editor
behavior are wired up in subsequent tasks.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Mount the ticker tape

**Files:**
- Modify: `Finance_Dashboard.html` — body of `initTradingView()`

- [ ] **Step 1: Add ticker-tape mount call**

Use Edit. Replace:

```js
function initTradingView() {
  // Widget mounts are added in later tasks.
}
```

With:

```js
function initTradingView() {
  tvMountTickerTape();
}

function tvMountTickerTape() {
  mountTVWidget('tvTickerTape',
    'https://s3.tradingview.com/external-embedding/embed-widget-ticker-tape.js',
    {
      symbols: tvState.tickerTape.map(s => ({ proName: s, title: s.split(':').pop() })),
      showSymbolLogo: true,
      isTransparent: false,
      displayMode: "adaptive",
      colorTheme: "light",
      locale: "en"
    }
  );
}
```

- [ ] **Step 2: Verify in browser**

Hard-refresh `Finance_Dashboard.html` (Cmd+Shift+R). Click Markets.
Expected: scrolling ticker tape appears at the top of the page with BTC, ETH, USD/INR, NIFTY, SPX. The other three containers stay empty. No console errors. Click another tab and back — the ticker tape does NOT re-mount (init-once is working).

- [ ] **Step 3: Commit**

```bash
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Mount TradingView ticker tape on Markets tab

Adds tvMountTickerTape() and calls it from initTradingView(). Uses
colorTheme: "light" (ticker-tape uses colorTheme, not theme).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Mount the main advanced chart

**Files:**
- Modify: `Finance_Dashboard.html` — `initTradingView()` + new `tvMountMainChart()`

- [ ] **Step 1: Add main-chart mount**

Use Edit. Replace:

```js
function initTradingView() {
  tvMountTickerTape();
}
```

With:

```js
function initTradingView() {
  tvMountTickerTape();
  tvMountMainChart();
}

function tvMountMainChart() {
  mountTVWidget('tvMainChart',
    'https://s3.tradingview.com/external-embedding/embed-widget-advanced-chart.js',
    {
      autosize: true,
      symbol: tvState.mainChart,
      interval: "D",
      timezone: "Asia/Kolkata",
      theme: "light",
      style: "1",
      locale: "en",
      enable_publishing: false,
      withdateranges: true,
      hide_side_toolbar: false,
      allow_symbol_change: true,
      details: false,
      hotlist: false,
      calendar: false,
      support_host: "https://www.tradingview.com"
    }
  );
}
```

- [ ] **Step 2: Verify in browser**

Hard-refresh. Click Markets.
Expected: a large interactive chart appears in the second container showing BTCUSDT daily candles in light theme. Symbol search inside the chart (top-left icon) works — try switching to ETHUSDT. No console errors.

- [ ] **Step 3: Commit**

```bash
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Mount TradingView advanced chart on Markets tab

Uses theme: "light" (advanced chart uses theme, not colorTheme).
Timezone set to Asia/Kolkata; allow_symbol_change enables in-widget
symbol search so the user navigates inside the iframe rather than via
the dashboard's editor.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Mount the mini-chart grid

**Files:**
- Modify: `Finance_Dashboard.html` — `initTradingView()` + new `tvRenderGrid()`

- [ ] **Step 1: Add grid render**

Use Edit. Replace:

```js
function initTradingView() {
  tvMountTickerTape();
  tvMountMainChart();
}
```

With:

```js
function initTradingView() {
  tvMountTickerTape();
  tvMountMainChart();
  tvRenderGrid();
}

function tvRenderGrid() {
  const host = document.getElementById('tvMiniGrid');
  if (!host) return;
  host.innerHTML = '';
  tvState.watchlistGrid.forEach((sym, i) => {
    const tileId = 'tvMiniTile_' + i;
    const tile = document.createElement('div');
    tile.className = 'tv-mini-tile';
    tile.id = tileId;
    host.appendChild(tile);
    mountTVWidget(tileId,
      'https://s3.tradingview.com/external-embedding/embed-widget-symbol-overview.js',
      {
        symbols: [[sym, sym + "|1D"]],
        chartOnly: false,
        width: "100%",
        height: "100%",
        locale: "en",
        colorTheme: "light",
        autosize: true,
        showVolume: false,
        showMA: false,
        hideDateRanges: false,
        hideMarketStatus: false,
        hideSymbolLogo: false,
        scalePosition: "right",
        scaleMode: "Normal",
        fontFamily: "Inter, -apple-system, BlinkMacSystemFont, sans-serif",
        fontSize: "10",
        noTimeScale: false,
        valuesTracking: "1",
        changeMode: "price-and-percent",
        chartType: "area",
        dateFormat: "yyyy-MM-dd"
      }
    );
  });
}
```

- [ ] **Step 2: Verify in browser**

Hard-refresh. Click Markets.
Expected: below the main chart, a responsive grid of 4 small Symbol Overview tiles (BTC, ETH, SOL, NIFTY), each showing a small line chart with price and change. On a wide window, see up to 4 columns. No console errors.

- [ ] **Step 3: Commit**

```bash
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Mount mini-chart grid on Markets tab

tvRenderGrid() creates one Symbol Overview widget per ticker in
tvState.watchlistGrid. Tile container is cleared and rebuilt on each
call so future watchlist edits are a single function call.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 9: Mount the economic calendar

**Files:**
- Modify: `Finance_Dashboard.html` — `initTradingView()` + new `tvMountCalendar()`

- [ ] **Step 1: Add calendar mount**

Use Edit. Replace:

```js
function initTradingView() {
  tvMountTickerTape();
  tvMountMainChart();
  tvRenderGrid();
}
```

With:

```js
function initTradingView() {
  tvMountTickerTape();
  tvMountMainChart();
  tvRenderGrid();
  tvMountCalendar();
}

function tvMountCalendar() {
  mountTVWidget('tvCalendar',
    'https://s3.tradingview.com/external-embedding/embed-widget-events.js',
    {
      width: "100%",
      height: "100%",
      colorTheme: "light",
      isTransparent: false,
      locale: "en",
      importanceFilter: "0,1",  // Medium and High importance only
      countryFilter: "in,us,eu,gb"
    }
  );
}
```

- [ ] **Step 2: Verify in browser**

Hard-refresh. Click Markets.
Expected: economic calendar appears at the bottom showing upcoming events filtered to India / US / EU / UK with medium and high importance. No console errors.

- [ ] **Step 3: Commit**

```bash
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Mount TradingView economic calendar on Markets tab

Filters events to India / US / EU / UK at medium/high importance to
keep the calendar relevant and reduce visual noise.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 10: Wire the symbol manager — add / remove / persist / re-render

**Files:**
- Modify: `Finance_Dashboard.html` — add chip renderer + add/remove handlers; call from `initTradingView()`.

- [ ] **Step 1: Add chip rendering and editor handlers**

Use Edit. Replace:

```js
function tvMountCalendar() {
  mountTVWidget('tvCalendar',
    'https://s3.tradingview.com/external-embedding/embed-widget-events.js',
    {
      width: "100%",
      height: "100%",
      colorTheme: "light",
      isTransparent: false,
      locale: "en",
      importanceFilter: "0,1",  // Medium and High importance only
      countryFilter: "in,us,eu,gb"
    }
  );
}
```

With:

```js
function tvMountCalendar() {
  mountTVWidget('tvCalendar',
    'https://s3.tradingview.com/external-embedding/embed-widget-events.js',
    {
      width: "100%",
      height: "100%",
      colorTheme: "light",
      isTransparent: false,
      locale: "en",
      importanceFilter: "0,1",  // Medium and High importance only
      countryFilter: "in,us,eu,gb"
    }
  );
}

// ---- Symbol manager (chips + add/remove + persist) ----
function tvRenderChips() {
  const fields = [
    { listKey: 'tickerTape',    chipsId: 'tvChipsTape' },
    { listKey: 'watchlistGrid', chipsId: 'tvChipsGrid' }
  ];
  fields.forEach(({ listKey, chipsId }) => {
    const host = document.getElementById(chipsId);
    if (!host) return;
    host.innerHTML = '';
    tvState[listKey].forEach((sym, i) => {
      const chip = document.createElement('span');
      chip.className = 'tv-chip';
      chip.innerHTML = sym + ' <span class="tv-chip-x" title="Remove">&times;</span>';
      chip.querySelector('.tv-chip-x').addEventListener('click', () => tvRemoveTicker(listKey, i));
      host.appendChild(chip);
    });
  });
}

function tvAddTicker(listKey) {
  const inputId = listKey === 'tickerTape' ? 'tvInputTape' : 'tvInputGrid';
  const input = document.getElementById(inputId);
  const sym = (input.value || '').trim().toUpperCase();
  if (!sym) return;
  if (tvState[listKey].includes(sym)) { input.value = ''; return; }
  tvState[listKey].push(sym);
  saveTVState();
  input.value = '';
  tvRenderChips();
  if (listKey === 'tickerTape')    tvMountTickerTape();
  if (listKey === 'watchlistGrid') tvRenderGrid();
}

function tvRemoveTicker(listKey, index) {
  tvState[listKey].splice(index, 1);
  saveTVState();
  tvRenderChips();
  if (listKey === 'tickerTape')    tvMountTickerTape();
  if (listKey === 'watchlistGrid') tvRenderGrid();
}
```

- [ ] **Step 2: Call `tvRenderChips()` from `initTradingView`**

Use Edit. Replace:

```js
function initTradingView() {
  tvMountTickerTape();
  tvMountMainChart();
  tvRenderGrid();
  tvMountCalendar();
}
```

With:

```js
function initTradingView() {
  tvRenderChips();
  tvMountTickerTape();
  tvMountMainChart();
  tvRenderGrid();
  tvMountCalendar();
}
```

- [ ] **Step 3: Verify in browser**

Hard-refresh. Click Markets → Edit watchlist (expand). You should see chips for each ticker in both rows. Click an × on the watchlist-grid `SOL` chip.
Expected: SOL chip disappears; the SOL mini-tile disappears from the grid (re-mount happened). Type `BINANCE:ADAUSDT` into the ticker tape input and click Add.
Expected: chip appears, ticker tape re-mounts with ADA scrolling. Reload the page → click Markets → Edit watchlist.
Expected: edits persisted (no SOL, ADA present in tape).

- [ ] **Step 4: Commit**

```bash
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Wire Markets watchlist editor: add, remove, persist, re-render

tvRenderChips() draws both chip rows from tvState. tvAddTicker /
tvRemoveTicker mutate state, persist to localStorage, and re-mount
only the affected widget (ticker tape OR mini-chart grid). Duplicates
are silently rejected.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 11: Wire "Reset to defaults"

**Files:**
- Modify: `Finance_Dashboard.html` — add `tvResetDefaults()`

- [ ] **Step 1: Add reset handler**

Use Edit. Insert immediately after `tvRemoveTicker`. Replace:

```js
function tvRemoveTicker(listKey, index) {
  tvState[listKey].splice(index, 1);
  saveTVState();
  tvRenderChips();
  if (listKey === 'tickerTape')    tvMountTickerTape();
  if (listKey === 'watchlistGrid') tvRenderGrid();
}
```

With:

```js
function tvRemoveTicker(listKey, index) {
  tvState[listKey].splice(index, 1);
  saveTVState();
  tvRenderChips();
  if (listKey === 'tickerTape')    tvMountTickerTape();
  if (listKey === 'watchlistGrid') tvRenderGrid();
}

function tvResetDefaults() {
  tvState = JSON.parse(JSON.stringify(CONFIG.tradingview));  // deep copy
  saveTVState();
  tvRenderChips();
  tvMountTickerTape();
  tvRenderGrid();
  tvMountMainChart();  // re-mount so its initial symbol resets
  // Economic calendar untouched — does not depend on tvState.
}
```

- [ ] **Step 2: Verify in browser**

Open Markets → Edit watchlist. Remove a few chips, add a custom one. Click Reset to defaults.
Expected: chips revert to the CONFIG defaults; ticker tape, grid, AND main chart re-mount (main chart will scroll back to its top to redraw). Reload the page — defaults are still in effect (localStorage was overwritten with defaults).

- [ ] **Step 3: Commit**

```bash
git add Finance_Dashboard.html
git commit -m "$(cat <<'EOF'
Add 'Reset to defaults' for Markets watchlist

Replaces tvState with a deep copy of CONFIG.tradingview, persists,
re-renders chips, and re-mounts the ticker tape, grid, and main chart.
Economic calendar is untouched (independent of tvState).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 12: Update `CLAUDE.md` and `README.md`

**Files:**
- Modify: `CLAUDE.md` — add Markets tab description under Architecture; add `tv_state_v1` to the localStorage table.
- Modify: `README.md` — add a one-line note about the Markets tab.

- [ ] **Step 1: Add `tv_state_v1` to CLAUDE.md localStorage table**

Use Edit on `CLAUDE.md`. Replace:

```markdown
| `invoices_v1` | Invoice records. |

Base64 storage of attachments shares the ~5MB localStorage cap.
```

With:

```markdown
| `invoices_v1` | Invoice records. |
| `tv_state_v1` | Markets tab state (ticker tape + watchlist grid + initial main-chart symbol). |

Base64 storage of attachments shares the ~5MB localStorage cap.
```

- [ ] **Step 1b: Update tab count and init description in CLAUDE.md**

Use Edit on `CLAUDE.md`. Replace:

```markdown
Ten tabs, each a `<div class="page" id="page-{id}">`. Switching calls
`switchPage(id)` which:

1. Hides all `.page` divs, shows the target.
2. Calls `PAGE_INIT[id]()` **once** on first visit (tracked via the
   `initializedPages` Set).
3. Updates `document.title` and `.page-title` via `PAGE_TITLES`.
```

With:

```markdown
Eleven tabs, each a `<div class="page" id="page-{id}">`. Switching calls
`switchPage(id)` which:

1. Hides all `.page` divs, shows the target.
2. Calls `PAGE_INIT[id]()`. By default this runs **every visit** (most
   init functions re-read live state). Tabs listed in `PAGE_INIT_ONCE`
   are init'd only on first visit, tracked via the `initializedPages`
   Set — currently just the Markets tab, whose iframes are too heavy
   to re-mount on every click.
3. Updates `document.title` and `.page-title` via `PAGE_TITLES`.
```

- [ ] **Step 2: Add Markets tab section to CLAUDE.md Architecture**

Use Edit on `CLAUDE.md`. Replace:

```markdown
### Live rates
```

With:

```markdown
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
```

- [ ] **Step 3: Add Markets to the README tab list**

Use Edit on `README.md`. Replace:

```markdown
- **Invoices** — Issue and track invoices; gross receipts feed into tax.
```

With:

```markdown
- **Invoices** — Issue and track invoices; gross receipts feed into tax.
- **Markets** — Embedded TradingView widgets (ticker tape, advanced chart, mini-chart grid, economic calendar). Watchlist editable in-app.
```

- [ ] **Step 3b: Update the tab count in README architecture section**

Use Edit on `README.md`. Replace:

```markdown
- 10 tabs, each a `<div class="page">`. Switching is handled by `switchPage(id)`.
```

With:

```markdown
- 11 tabs, each a `<div class="page">`. Switching is handled by `switchPage(id)`.
```

- [ ] **Step 4: Verify**

Open `CLAUDE.md` in a viewer. Confirm: `tv_state_v1` row is present in the table; Markets tab section appears between the existing Chart registry and Live rates sections. Open `README.md` — confirm the Markets line is visible and matches the surrounding style.

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md README.md
git commit -m "$(cat <<'EOF'
Document the Markets tab in CLAUDE.md and README.md

CLAUDE.md gains a Markets architecture section and a tv_state_v1 row
in the localStorage table. README.md gets a one-line user-facing note.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 13: End-to-end manual verification

**Files:** (none modified)

This task is the full acceptance pass against the spec's testing section.

- [ ] **Step 1: Cold-start verification**

Close all dashboard tabs. Clear `tv_state_v1`: in devtools console run `localStorage.removeItem('tv_state_v1'); location.reload();`. Click Markets.
Expected: ticker tape, main chart, grid (4 default tiles), and economic calendar all render in light theme. Page header reads "Markets". No console errors.

- [ ] **Step 2: Editor flows**

Open Edit watchlist. Add `BINANCE:ADAUSDT` to ticker tape → tape re-mounts with ADA. Add `BINANCE:LINKUSDT` to grid → 5th mini-tile appears. Remove `BINANCE:SOLUSDT` from grid → 4 tiles again, no SOL.
Expected: each action immediate, no flicker outside the affected widget, no console errors.

- [ ] **Step 3: Persistence**

Reload page → click Markets → Edit watchlist.
Expected: edits from step 2 are still present.

- [ ] **Step 4: Reset**

Click Reset to defaults.
Expected: chips revert to CONFIG values, ticker tape and grid re-mount, main chart re-mounts and reverts to BINANCE:BTCUSDT. Reload page — defaults persist (state was rewritten to defaults).

- [ ] **Step 5: Once-per-session init**

In console, run: `console.log('init count check');`. Switch from Markets to Overview, then back to Markets twice. Watch the network panel.
Expected: no new requests to `s3.tradingview.com/external-embedding/*` after the first Markets visit. Iframes survive tab switches without re-mounting.

- [ ] **Step 6: Existing tabs unaffected**

Click through Overview, Cash Flow, Portfolio, Projections, Inflight, Transactions, Goals, FIRC, Taxes, Invoices.
Expected: every tab still renders charts/tables correctly, no console errors, no visual regressions.

- [ ] **Step 7: Offline check (optional but recommended)**

Block `s3.tradingview.com` in devtools Network → throttle to Offline. Reload, click Markets.
Expected: containers stay blank, no JS errors thrown. Page does not crash. Restoring network and reloading recovers the widgets.

- [ ] **Step 8: Final commit (only if any tweaks were needed)**

If steps 1–7 surfaced any fixes, commit them with a clear message. Otherwise, no commit needed for this task — it's verification only.

---

## Self-Review Summary (already performed by plan author)

- **Spec coverage:** Tab integration → Task 4. CONFIG block → Task 1. tvState + persistence → Task 3. Page layout → Task 5. Per-widget mounts → Tasks 6–9. Per-widget theme props (theme vs colorTheme) → spelled out in Tasks 6 (colorTheme), 7 (theme), 8 (colorTheme), 9 (colorTheme). Symbol manager → Task 10. Reset to defaults → Task 11. Documentation updates → Task 12. Manual verification matches spec's testing section → Task 13.
- **Function/symbol consistency:** `mountTVWidget`, `tvState`, `saveTVState`, `tvMountTickerTape`, `tvMountMainChart`, `tvRenderGrid`, `tvMountCalendar`, `tvRenderChips`, `tvAddTicker`, `tvRemoveTicker`, `tvResetDefaults`, `initTradingView`, `PAGE_INIT_ONCE`, `initializedPages` — names match across every task that references them.
- **No placeholders:** All code blocks are complete and runnable. No "TODO" or "implement later" steps.
