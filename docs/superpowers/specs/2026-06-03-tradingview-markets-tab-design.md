# TradingView "Markets" Tab — Design

**Date:** 2026-06-03
**Status:** Approved, pending implementation plan
**Owner:** sarthakguleria002@gmail.com

## Problem

The dashboard has no live market view. The user wanted a "TradingView tab where a person can plug in their API key." Clarifying conversation established:

- TradingView does not expose a personal REST API where users plug in a key.
- This project is a single static HTML file opened via `file://` with no backend, so any key would sit unencrypted in `localStorage`.
- The user's actual need is a market-data tab inside the dashboard.

Chosen direction: embed free TradingView widgets. No API key required. Configuration is the watchlist of symbols.

## Goals

- Add a "Markets" tab to `Finance_Dashboard.html` showing live charts via TradingView widgets.
- Let the user manage a watchlist of symbols that drives multiple widgets.
- Match the existing dark theme and single-file architecture.

## Non-Goals

- Broker integration (Zerodha, Binance, Alpaca, etc.) — explicitly out of scope.
- Pulling user data from TradingView (watchlists, alerts, layouts) — not exposed by their API.
- Theme toggle — dashboard is dark-only.
- Mobile optimization beyond what the rest of the dashboard already does.
- Coupling TradingView widgets to the existing `LIVE` INR-conversion logic.
- Tracking the user's in-widget chart navigation. Once the main chart mounts, any symbol the user picks inside it is owned by TradingView's own iframe storage; the dashboard does not observe or persist those changes. The `tvState.mainChart` field controls only the *initial* symbol on first mount and after "Reset to defaults".

## Architecture

### Tab integration

Follows the existing 10-tab pattern in `Finance_Dashboard.html`:

- New `<div class="page" id="page-tradingview">` block.
- New sidebar nav item linking to it.
- New entries in `PAGE_TITLES` (`tradingview: "Markets"`) and `PAGE_INIT` (`tradingview: initTradingView`).
- Init runs once via the existing `initializedPages` Set — important because TradingView widgets are heavy and should lazy-load on first visit only.

All HTML/CSS/JS stays inline in `Finance_Dashboard.html`, per the project's no-file-splitting rule.

### Configuration

Add a `tradingview` block to the `CONFIG` object literal:

```js
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
}
```

User edits persist to a new `localStorage` key `tv_watchlist_v1` (naming matches `custom_accounts_v1`). On init:

```js
const tvState = JSON.parse(localStorage.getItem('tv_watchlist_v1') || 'null')
              || structuredClone(CONFIG.tradingview);
```

### Page layout (stacked sections)

```
┌────────────────────────────────────────────────┐
│  Symbol manager (collapsible, collapsed by    │
│  default): input + Add button, ticker chips,   │
│  Reset-to-defaults button, format hint link    │
├────────────────────────────────────────────────┤
│  Ticker Tape (embed-widget-tickers.js)        │
├────────────────────────────────────────────────┤
│  Main Advanced Chart                           │
│  (embed-widget-advanced-chart.js)              │
│  symbol = tvState.mainChart                    │
├────────────────────────────────────────────────┤
│  Mini-chart grid (CSS grid, responsive         │
│  2/3/4 columns by viewport):                   │
│  one embed-widget-symbol-overview.js per       │
│  ticker in tvState.watchlistGrid               │
├────────────────────────────────────────────────┤
│  Economic Calendar (embed-widget-events.js)    │
└────────────────────────────────────────────────┘
```

All widgets hard-set `theme: "dark"`.

### Widget mount helper

Single helper, modeled on the existing `mkChart()` pattern:

```js
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
```

This is the destroy-and-recreate primitive for re-rendering on watchlist edits — same conceptual role `mkChart()` plays for Chart.js.

### Symbol manager UI

- Collapsed by default. Toggle button: "Edit watchlist".
- Single text input + "Add" button. Placeholder: `BINANCE:BTCUSDT or NSE:RELIANCE`.
- Two chip rows, each labeled, with × on each chip:
  - "Ticker tape" → drives `tvState.tickerTape`
  - "Watchlist grid" → drives `tvState.watchlistGrid`
- "Reset to defaults" button → copies `CONFIG.tradingview` into `tvState`, writes it to `localStorage`, and re-mounts the ticker tape and grid. Also re-mounts the main chart so its initial symbol resets.
- Small help link to `https://www.tradingview.com/symbols/` for users who don't know the format.

**On add/remove of a ticker:**
1. Save updated `tvState` to `localStorage`.
2. Re-mount only the affected widget(s): ticker tape and/or the mini-chart grid.
3. Main chart and economic calendar are not re-mounted — they don't depend on the list.

**On "Reset to defaults":** treated as a global reset (see button description above) — re-mounts the ticker tape, grid, AND the main chart so its initial symbol reverts. Economic calendar is still untouched.

The main chart's symbol is changed inside the TradingView widget itself (it has its own search UI). We do not expose a separate input for it.

## Data flow

```
CONFIG.tradingview  ──┐
                      ├──► tvState ──► mountTVWidget() ──► TradingView iframe
localStorage          ──┘     ▲
(tv_watchlist_v1)             │
                              │
            Symbol manager UI ─┘ (edit → save → re-mount)
```

No coupling to `LIVE`, `HOLDINGS`, `CUSTOM_ACCS`, or any other dashboard state.

## Error handling

- **Widget fails to load (offline / blocked):** container stays blank. Same behavior as existing Binance/CoinGecko fetches when offline. No retry logic, no error UI. Acceptable per project conventions.
- **Invalid symbol entered by user:** TradingView renders an empty chart or an "Invalid symbol" message inside its iframe. We do not validate — TradingView's own search is the better source of truth.
- **localStorage unavailable / quota exceeded:** the watchlist falls back to `CONFIG.tradingview` defaults; edits silently fail to persist. Matches behavior of other `localStorage`-backed features.

## Testing

Manual verification (no test harness exists in this project):

1. Open `Finance_Dashboard.html` in a browser. Click Markets tab.
2. Confirm all four widgets render with the dark theme and default symbols.
3. Open Edit watchlist, add `BINANCE:ADAUSDT` to ticker tape → confirm it appears in the scrolling tape.
4. Remove a ticker from the grid → confirm the corresponding mini-chart disappears.
5. Click Reset to defaults → confirm chips and widgets revert.
6. Reload page → confirm edits persisted (ticker tape and grid match pre-reload state).
7. Switch to another tab and back → confirm init does not re-run (no iframe flicker).
8. Verify no console errors.

## Risks & tradeoffs

| Risk | Mitigation |
|------|------------|
| First-load latency (~2–5s for 4+ iframes) | Lazy init on first tab visit (free via existing pattern). |
| Memory (~100–200MB extra when tab open) | Acceptable on desktop; rest of dashboard is also desktop-focused. |
| Symbol format friction (`BINANCE:BTCUSDT`) | Good defaults + format hint + link to TradingView symbol search. |
| TradingView CDN blocked / down | Blank widgets, no fallback. Acceptable — non-critical feature. |
| Iframe leaks on re-mount | `container.innerHTML = ""` before re-mount; consistent with `mkChart()` pattern. |

## Documentation updates

- **CLAUDE.md:** add a one-paragraph section under "Architecture" describing the Markets tab, the `tvState` / `tv_watchlist_v1` model, and the `mountTVWidget()` helper. Also add `tv_watchlist_v1` to the localStorage key table.
- **README.md:** brief note that the dashboard includes embedded TradingView charts and how to customize the watchlist (via the in-app editor).

## Open questions

None. Design is ready for implementation planning.
